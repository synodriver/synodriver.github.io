---
title: 'Home Assistant 核心架构与实现深度解析'
date: '2026-06-05T00:41:47+08:00'
draft: false
tags: ['python', 'asyncio', 'home-assistant', 'iot']
author: 'synodriver'
---

# Home Assistant 核心架构与实现深度解析

> 基于源码版本 2026.7.0.dev0，Python 3.14.2+

## 目录

- [1. 项目总体结构](#1-项目总体结构)
- [2. 核心类与基类体系](#2-核心类与基类体系)
- [3. 启动流程详解](#3-启动流程详解)
- [4. 集成加载机制](#4-集成加载机制)
- [5. 配置流(ConfigFlow)机制](#5-配置流configflow机制)
- [6. 实体(Entity)体系](#6-实体entity体系)
- [7. 服务(Service)机制](#7-服务service机制)
- [8. 平台(Platform)与 EntityComponent 编排](#8-平台platform与-entitycomponent-编排)
- [9. 集成实现完整示例：Hue](#9-集成实现完整示例hue)
- [10. 如何实现自己的集成](#10-如何实现自己的集成)
- [11. 关键设计模式总结](#11-关键设计模式总结)

---

## 1. 项目总体结构

### 1.1 顶层目录

```
E:\pyproject\core\
├── homeassistant/          # 核心源代码主目录
│   ├── core.py             # HomeAssistant 主类、EventBus、StateMachine、ServiceRegistry
│   ├── config_entries.py   # ConfigEntry、ConfigFlow、ConfigEntries 管理器
│   ├── bootstrap.py        # 启动引导，分阶段加载
│   ├── loader.py           # Integration 类，manifest 解析，自定义集成管理
│   ├── setup.py            # async_setup_component，依赖解析
│   ├── data_entry_flow.py  # FlowHandler/FlowManager 基类
│   ├── config.py           # 配置文件处理
│   ├── const.py            # 全局常量、Platform 枚举
│   ├── runner.py           # 运行器，事件循环设置
│   ├── __main__.py         # CLI 入口
│   ├── auth/               # 认证系统
│   ├── components/         # 所有内置集成（2000+个）
│   ├── generated/          # 自动生成的代码（蓝牙、DHCP、SSDP 数据等）
│   ├── helpers/            # 辅助模块（entity、entity_registry、template 等）
│   └── util/               # 工具模块
├── tests/                  # 测试代码
├── custom_components/      # 自定义集成目录
└── pyproject.toml          # 项目配置
```

### 1.2 集成组织方式

所有集成都以子目录形式存放在 `homeassistant/components/` 下，目录名即域名。每个集成至少包含：

```
components/<domain>/
├── manifest.json       # 集成清单（必需）
├── __init__.py         # 集成入口
├── config_flow.py      # 配置流（可选，支持 UI 配置时需要）
├── light.py            # 向 light 域提供实体（可选平台文件）
├── sensor.py           # 向 sensor 域提供实体（可选平台文件）
├── strings.json        # 前端字符串
└── services.yaml       # 服务定义
```

### 1.3 manifest.json 关键字段

```json
{
  "domain": "hue",
  "name": "Philips Hue",
  "integration_type": "hub",
  "config_flow": true,
  "dependencies": [],
  "after_dependencies": [],
  "requirements": ["aiohue==4.8.1"],
  "iot_class": "local_push",
  "zeroconf": ["_hue._tcp.local."],
  "codeowners": ["@marcelveldt"]
}
```

| 字段 | 说明 |
|------|------|
| `domain` | 集成域名，必须等于目录名 |
| `name` | 人类可读名称 |
| `integration_type` | `entity`(平台型)、`hub`(集线器型)、`system`(系统型)、`virtual`(虚拟型)、`helper`(辅助型)、`device`(设备型) |
| `config_flow` | 是否支持 UI 配置流 |
| `dependencies` | 前置依赖集成（必须先加载） |
| `after_dependencies` | 后置依赖（仅影响加载顺序，不强制加载） |
| `requirements` | pip 依赖包 |
| `iot_class` | `local_push`、`local_polling`、`cloud_push`、`cloud_polling`、`assumed_state` |
| `zeroconf`/`homekit`/`ssdp`/`dhcp`/`usb`/`bluetooth` | 自动发现配置 |

---

## 2. 核心类与基类体系

### 2.1 HomeAssistant 类

**源码**: `homeassistant/core.py`

`HomeAssistant` 是整个系统的根对象，几乎一切皆通过它访问。

```python
class HomeAssistant:
    def __init__(self, config_dir: str):
        self.data = HassDict()                    # 全局数据存储字典
        self.loop = asyncio.get_running_loop()    # 事件循环
        self.bus = EventBus(self)                 # 事件总线
        self.services = ServiceRegistry(self)     # 服务注册表
        self.states = StateMachine(self.bus, self.loop)  # 状态机
        self.config = Config(self, config_dir)     # 配置
        self.config_entries: ConfigEntries         # 配置条目管理器
        self.auth: AuthManager                     # 认证管理器
        self.state: CoreState = CoreState.not_running
        self.exit_code: int = 0
        self.timeout = TimeoutManager()
        self.import_executor = InterruptibleThreadPoolExecutor(...)
```

**核心方法**:

| 方法 | 说明 |
|------|------|
| `async_run()` | 主入口，设置信号处理，等待停止 |
| `async_start()` | 发射 `EVENT_HOMEASSISTANT_START` → `CoreState.running` → 发射 `EVENT_HOMEASSISTANT_STARTED` |
| `async_create_task()` | 创建异步任务 |
| `async_add_job()` / `async_add_executor_job()` | 添加任务到事件循环或线程池 |

**CoreState 生命周期**:

```
NOT_RUNNING → STARTING → RUNNING → STOPPING → FINAL_WRITE → STOPPED
```

### 2.2 EventBus — 事件总线

**源码**: `homeassistant/core.py`

事件驱动架构的核心，所有组件间通信都通过事件总线。

```python
class EventBus:
    def async_listen(self, event_type, listener) -> Callable:  # 注册监听器
    def async_fire(self, event_type, event_data) -> None:       # 发射事件
```

关键事件：
- `EVENT_HOMEASSISTANT_START` — HA 开始运行
- `EVENT_HOMEASSISTANT_STARTED` — HA 完全启动
- `EVENT_HOMEASSISTANT_STOP` — HA 停止
- `EVENT_STATE_CHANGED` — 状态变更
- `EVENT_CALL_SERVICE` — 服务调用
- `EVENT_COMPONENT_LOADED` — 组件加载完成

### 2.3 StateMachine — 状态机

**源码**: `homeassistant/core.py`

管理所有实体的状态，状态以 `(entity_id, state, attributes)` 三元组存储。

```python
class StateMachine:
    def async_set(self, entity_id, state, attributes) -> None  # 设置状态
    def async_get(self, entity_id) -> State | None              # 获取状态
    def async_all(self) -> list[State]                          # 获取所有状态
    def async_remove(self, entity_id) -> bool                   # 移除状态
```

状态变更会触发 `EVENT_STATE_CHANGED` 事件。

### 2.4 ServiceRegistry — 服务注册表

**源码**: `homeassistant/core.py`

管理所有已注册的服务，两层字典结构 `_services[domain][service_name]`。

```python
class ServiceRegistry:
    def async_register(self, domain, service, service_func, schema) -> None
    def async_call(self, domain, service, service_data, blocking) -> Any
    def async_remove(self, domain, service) -> None
```

详见[第 7 节](#7-服务service机制)。

### 2.5 Entity 类 — 实体基类

**源码**: `homeassistant/helpers/entity.py`

所有实体的抽象基类，使用 `ABCCachedProperties` 元类缓存高频属性。

```python
class Entity(metaclass=ABCCachedProperties, cached_properties=...):
    entity_id: str = None
    hass: HomeAssistant = None
    platform: EntityPlatform = None
    entity_description: EntityDescription

    # 核心属性
    @property
    def state(self) -> StateType                        # 实体状态值
    @property
    def capability_attributes(self) -> dict             # 能力属性
    @property
    def name(self) -> str | None                        # 名称
    @property
    def icon(self) -> str | None                        # 图标
    @property
    def device_class(self) -> str | None                # 设备类
    @property
    def unit_of_measurement(self) -> str | None         # 计量单位
    @property
    def supported_features(self) -> int                 # 支持的特性标志位
    @property
    def available(self) -> bool                         # 是否可用
    @property
    def should_poll(self) -> bool                       # 是否需要轮询（默认 True）

    # 核心方法
    def async_write_ha_state(self) -> None              # 写入状态到状态机（推荐）
    def async_update_ha_state(self, force_refresh) -> None  # 更新状态
    def async_device_update(self) -> None               # 从设备拉取最新数据
    def async_added_to_hass(self) -> None               # 添加到 HA 后的钩子
    def async_will_remove_from_hass(self) -> None       # 从 HA 移除前的钩子
    def async_on_remove(self, func) -> None             # 注册移除时的清理回调
```

**关键属性设置模式**：Entity 大量使用 `_attr_*` 类属性来提供默认值，子类可以直接设置类属性而不需要定义 property：

```python
class MyEntity(Entity):
    _attr_should_poll = False     # 等价于 should_poll = False
    _attr_icon = "mdi:lightbulb"  # 等价于 icon = "mdi:lightbulb"
```

### 2.6 ToggleEntity — 开关实体基类

**源码**: `homeassistant/helpers/entity.py`

继承自 `Entity`，添加开/关语义。

```python
class ToggleEntity(Entity):
    _attr_is_on: bool | None = None

    @property
    @final
    def state(self) -> Literal["on", "off"] | None:
        if (is_on := self.is_on) is None: return None
        return STATE_ON if is_on else STATE_OFF

    async def async_turn_on(self, **kwargs) -> None: ...   # 子类必须实现
    async def async_turn_off(self, **kwargs) -> None: ...   # 子类必须实现
    async def async_toggle(self, **kwargs) -> None: ...     # 默认实现调用 turn_on/off
```

### 2.7 ConfigEntry 类 — 配置条目

**源码**: `homeassistant/config_entries.py`

每个用户配置的集成实例对应一个 `ConfigEntry`，是 Config Entry 驱动架构的核心数据结构。

```python
class ConfigEntry[_DataT = Any]:
    entry_id: str           # 唯一 ID (ULID)
    domain: str             # 所属集成域名
    title: str              # 显示标题
    data: MappingProxyType  # 配置数据（如 host, api_key）
    runtime_data: _DataT    # 运行时数据（泛型，集成可自定义类型）
    options: MappingProxyType  # 用户选项
    unique_id: str | None   # 去重 ID
    state: ConfigEntryState # 状态
    version: int            # 配置迁移版本
    minor_version: int      # 配置迁移次版本号
    source: str             # 来源: user, bluetooth, dhcp, zeroconf, ...
    subentries: MappingProxyType[str, ConfigSubentry]  # 子条目
    disabled_by: ConfigEntryDisabler | None
```

**ConfigEntryState 生命周期**:

```
NOT_LOADED → SETUP_IN_PROGRESS → LOADED
                              → SETUP_ERROR
                              → SETUP_RETRY
                              → MIGRATION_ERROR
LOADED → UNLOAD_IN_PROGRESS → NOT_LOADED
                            → FAILED_UNLOAD
```

**runtime_data 模式**：集成通过泛型参数声明运行时数据类型：

```python
type HueConfigEntry = ConfigEntry[HueBridge]

# 在 async_setup_entry 中：
entry.runtime_data = bridge

# 在平台文件中：
bridge = config_entry.runtime_data  # 自动具有 HueBridge 类型
```

### 2.8 Integration 类 — 集成定义

**源码**: `homeassistant/loader.py`

代表一个已解析的集成，包含 manifest 信息和模块引用。

```python
class Integration:
    domain: str
    name: str
    integration_type: str
    dependencies: list[str]
    after_dependencies: list[str]
    requirements: list[str]
    disabled: str | None
    manifest: dict

    @classmethod
    def resolve_from_root(cls, hass, root_module, domain) -> Integration | None

    async def async_get_component(self) -> ModuleType     # 获取 __init__.py 模块
    async def async_get_platform(self, platform) -> ModuleType  # 获取平台模块
    async def resolve_dependencies(self) -> bool | None
```

关键数据键：
- `DATA_COMPONENTS` — 已加载的组件模块缓存
- `DATA_INTEGRATIONS` — 已解析的 Integration 对象缓存
- `DATA_CUSTOM_COMPONENTS` — 自定义集成缓存

---

## 3. 启动流程详解

### 3.1 完整启动链

```
__main__.py::main()
  ├── validate_python()              # 校验 Python 版本
  ├── validate_os()                 # 校验操作系统
  ├── ensure_config_path()          # 确保配置目录存在
  ├── restore_backup()              # 检查是否需要从备份恢复
  └── runner.run(runtime_config)
        ├── HassEventLoopPolicy 设置  # 自定义事件循环策略
        │     ├── 64 workers 线程池
        │     ├── 调试模式
        │     └── loop.time = monotonic
        ├── asyncio.new_event_loop()
        └── loop.run_until_complete(setup_and_run_hass())
              ├── bootstrap.async_setup_hass()
              │     ├── create_hass()
              │     │     ├── core.HomeAssistant(config_dir)
              │     │     │     ├── EventBus
              │     │     │     ├── ServiceRegistry
              │     │     │     └── StateMachine
              │     │     ├── loader.async_setup(hass)
              │     │     └── async_enable_logging()
              │     ├── conf_util.async_ensure_config_exists()
              │     ├── conf_util.process_ha_config_upgrade()
              │     ├── conf_util.async_hass_config_yaml()
              │     └── async_from_config_dict(config_dict, hass)
              │           ├── ConfigEntries(hass, config)
              │           ├── loader.async_get_custom_components()
              │           ├── async_load_base_functionality()
              │           ├── async_setup_component("homeassistant")
              │           ├── async_setup_component("persistent_notification")
              │           ├── async_process_ha_core_config()
              │           └── _async_set_up_integrations()
              │                 ├── 解析依赖关系
              │                 ├── Stage 0: 核心基础设施
              │                 ├── Stage 1: 发现和云服务
              │                 └── Stage 2: 所有其余集成
              └── hass.async_run()
                    ├── hass.async_start()
                    │     ├── CoreState → STARTING
                    │     ├── fire EVENT_HOMEASSISTANT_START
                    │     ├── CoreState → RUNNING
                    │     └── fire EVENT_HOMEASSISTANT_STARTED
                    └── await _stopped.wait()
```

### 3.2 启动阶段

**Stage 0 — 核心基础设施**（分多个子阶段）：

| 子阶段 | 集成 | 说明 |
|--------|------|------|
| logging, http deps | isal, logger, network, system_log, sentry | 日志和网络基础 |
| labs | labs | 实验性功能 |
| frontend | frontend | 前端 UI |
| recorder | recorder | 状态历史记录 |
| debugger | debugpy | 调试器 |
| zeroconf | zeroconf | mDNS 发现 |

**Stage 1 — 发现和云服务**（超时 120s）：

- bluetooth, dhcp, ssdp, usb — 发现集成
- mqtt_eventstream, cloud, hassio

**Stage 2 — 所有其余集成**（超时 300s）

### 3.3 async_load_base_functionality — 注册表并行加载

启动时并行加载以下核心模块：

```python
await asyncio.gather(
    entity.async_setup(hass),        # 实体源
    frame.async_setup(hass),          # 帧追踪
    template.async_setup(hass),       # 模板引擎
    translation.async_setup(hass),    # 翻译
    device_registry.async_load(),     # 设备注册表
    area_registry.async_load(),       # 区域注册表
    entity_registry.async_load(),     # 实体注册表
    floor_registry.async_load(),      # 楼层注册表
    issue_registry.async_load(),      # 问题注册表
    label_registry.async_load(),      # 标签注册表
    restore_state.async_load(),       # 状态恢复
    hass.config_entries.async_initialize(),  # 配置条目
    ...
)
```

### 3.4 恢复模式

如果核心集成（如 frontend）加载失败或配置解析出错：
1. 停止当前 hass 实例
2. 重新创建 hass 实例
3. 仅加载最小集：`backup`、`cloud`、`frontend` + http 配置

---

## 4. 集成加载机制

### 4.1 async_setup_component 完整流程

**源码**: `homeassistant/setup.py`

```
async_setup_component(hass, domain, config)
  │
  ├── 1. 检查 hass.config.components — 已加载则返回 True
  ├── 2. 检查 _DATA_SETUP — 正在加载则 await 已有 Future
  ├── 3. 创建新 Future 注册到 _DATA_SETUP
  │
  └── 4. 调用 _async_setup_component(hass, domain, config)
        │
        ├── 阶段1: 获取 Integration 定义
        │     └── loader.async_get_integration(hass, domain)
        │           ├── 先查 DATA_INTEGRATIONS 缓存
        │           ├── 内置集成: Integration.resolve_from_root(hass, components, domain)
        │           └── 自定义集成: 从 custom_components 包解析
        │
        ├── 阶段2: 检查集成是否被禁用
        │
        ├── 阶段3: 预加载翻译（异步启动，稍后等待）
        │
        ├── 阶段4: 解析依赖关系
        │     └── integration.resolve_dependencies()
        │           检查依赖是否存在 + 循环依赖检测
        │
        ├── 阶段5: 处理依赖和需求
        │     └── async_process_deps_reqs(hass, config, integration)
        │           ├── _async_process_dependencies() — 并行加载 dependencies + after_dependencies
        │           │     ├── dependencies: 强制加载（async_setup_component）
        │           │     └── after_dependencies: 仅等待已计划安装的依赖
        │           └── requirements.async_get_integration_with_requirements() — 安装 pip 包
        │
        ├── 阶段6: 导入组件模块
        │     └── integration.async_get_component() → importlib.import_module()
        │
        ├── 阶段7: 校验配置
        │     ├── conf_util.async_process_component_config() — 用 CONFIG_SCHEMA 校验
        │     ├── conf_util.async_handle_component_errors()
        │     └── conf_util.async_drop_config_annotations()
        │
        ├── 阶段8: 执行安装
        │     ├── 如果有 component.async_setup → task = component.async_setup(hass, config)
        │     ├── 否则如果有 component.setup → 线程池执行 component.setup(hass, config)
        │     ├── 否则如果没有 async_setup_entry → 记录错误返回 False
        │     ├── async_timeout(SLOW_SETUP_MAX_WAIT=300s) 等待结果
        │     └── 等待翻译加载完成
        │
        ├── 阶段9: 等待 Config Flow 导入完成
        │     └── hass.config_entries.flow.async_wait_import_flow_initialized(domain)
        │
        └── 阶段10: 设置 Config Entry + 标记完成
              ├── hass.config.components.add(domain)  ← 先标记，防止死锁
              ├── 并行安装所有 ConfigEntry:
              │     await asyncio.gather(*(
              │       entry.async_setup_locked(hass, integration=integration)
              │       for entry in hass.config_entries.async_entries(domain)
              │     ))
              └── 触发 EVENT_COMPONENT_LOADED 事件
```

### 4.2 依赖解析细节

**`_async_process_dependencies()`** 的行为：

1. **`dependencies`（前置依赖）**：强制加载。如果依赖未安装，调用 `async_setup_component` 安装。
2. **`after_dependencies`（后置依赖）**：仅等待已计划安装的依赖，不主动触发安装，避免死锁。

```python
# 伪代码
for dep in integration.dependencies:
    if dep not in setup and dep not in components:
        create_task(async_setup_component(hass, dep, config))  # 强制安装

for dep in integration.after_dependencies:
    if dep in setup_done_tasks:  # 只等待已计划的
        wait_tasks.append(setup_done_tasks[dep])

await asyncio.gather(*all_tasks)
```

### 4.3 ConfigEntry 的 setup 流程

```
entry.async_setup_locked(hass, integration=integration)
  │
  ├── 检查 entry.state (必须是 NOT_LOADED)
  ├── entry.state = SETUP_IN_PROGRESS
  │
  └── component.async_setup_entry(hass, entry)
        │  ← 集成 __init__.py 中定义的 async_setup_entry 函数
        │
        ├── 初始化连接/API
        ├── 创建运行时对象（如 HueBridge）
        ├── entry.runtime_data = 运行时对象
        ├── hass.config_entries.async_forward_entry_setups(entry, platforms)
        │     └── 转发到各平台设置
        └── 注册更新监听器等
```

### 4.4 错误处理与重试

| 错误类型 | 处理方式 |
|----------|----------|
| `IntegrationNotFound` | 记录错误 + 创建 repair issue |
| 集成被禁用 | 记录错误 + 持久通知 |
| 依赖解析失败 | 记录错误 |
| `ImportError` | 记录错误 + 持久通知 |
| 配置校验失败 | 记录错误 + 持久通知 |
| `TimeoutError`（超 300 秒） | 记录错误 |
| `PlatformNotReady` | 线性递增退避重试（基础等待 30s，`min(tries,6)*30` 秒后重试） |
| `ConfigEntryNotReady` | 标记为 SETUP_RETRY，自动重试 |

---

## 5. 配置流(ConfigFlow)机制

### 5.1 类继承链

```
FlowHandler (data_entry_flow.py)
  └── ConfigEntryBaseFlow (config_entries.py)
        ├── ConfigFlow   — 新设备配置流程
        └── OptionsFlow  — 已配置设备的选项流程
```

### 5.2 FlowResultType — 流程结果类型

| 类型 | 说明 |
|------|------|
| `FORM` | 显示表单，等待用户输入 |
| `CREATE_ENTRY` | 创建条目，流程成功结束 |
| `ABORT` | 中止流程 |
| `EXTERNAL_STEP` | 外部步骤（如 OAuth 跳转） |
| `EXTERNAL_STEP_DONE` | 外部步骤完成 |
| `SHOW_PROGRESS` | 显示进度 |
| `SHOW_PROGRESS_DONE` | 进度完成 |
| `MENU` | 显示导航菜单 |

### 5.3 FlowHandler 基类核心方法

```python
class FlowHandler:
    # 结果返回方法
    def async_show_form(self, *, step_id, data_schema, errors, ...) → FORM
    def async_create_entry(self, *, title, data, ...) → CREATE_ENTRY
    def async_abort(self, *, reason, ...) → ABORT
    def async_external_step(self, *, step_id, url, ...) → EXTERNAL_STEP
    def async_external_step_done(self, *, next_step_id) → EXTERNAL_STEP_DONE
    def async_show_progress(self, *, step_id, progress_action, progress_task, ...) → SHOW_PROGRESS
    def async_show_progress_done(self, *, next_step_id) → SHOW_PROGRESS_DONE
    def async_show_menu(self, *, step_id, menu_options, ...) → MENU
```

### 5.4 FlowManager 核心流程

```
FlowManager.async_init(handler, *, context, data)
  │
  ├── async_create_flow(handler, context, data) → 创建 FlowHandler 实例
  ├── 分配 flow_id (UUID)
  ├── 注册到 _progress 索引
  └── _async_handle_step(flow, flow.init_step, data)
        │  ← init_step 默认为 "init"，但 ConfigFlow 中被覆盖为 context["source"]
        │
        ├── getattr(flow, f"async_step_{step_id}")(user_input)
        │     ← 反射调用步骤方法
        │
        ├── 如果结果在 FLOW_NOT_COMPLETE_STEPS 中 → 设置 cur_step，流程继续
        └── 否则 → async_finish_flow()，流程结束
```

### 5.5 ConfigFlow 类详解

**源码**: `homeassistant/config_entries.py`

```python
class ConfigFlow(ConfigEntryBaseFlow):
    # 自动注册：子类声明 domain 时自动注册到 HANDLERS 注册表
    def __init_subclass__(cls, *, domain=None, **kwargs):
        if domain is not None:
            HANDLERS.register(domain)(cls)

    # unique_id 管理
    async def async_set_unique_id(self, unique_id, *, raise_on_progress=True) -> ConfigEntry | None
    def _abort_if_unique_id_configured(self, updates=None, *, error="already_configured") -> None

    # 发现步骤方法（全部默认委托到 _async_step_discovery_without_unique_id → async_step_user）
    async def async_step_user(self, user_input)               # 用户手动添加
    async def async_step_bluetooth(self, discovery_info)      # 蓝牙发现
    async def async_step_dhcp(self, discovery_info)           # DHCP 发现
    async def async_step_zeroconf(self, discovery_info)       # mDNS/Zeroconf 发现
    async def async_step_homekit(self, discovery_info)        # HomeKit 发现
    async def async_step_hassio(self, discovery_info)         # Hass.io 发现
    async def async_step_ssdp(self, discovery_info)           # SSDP 发现
    async def async_step_usb(self, discovery_info)            # USB 发现
    async def async_step_mqtt(self, discovery_info)            # MQTT 发现
    async def async_step_integration_discovery(self, discovery_info)  # 集成发现
    async def async_step_import(self, import_data)            # YAML 导入
    async def async_step_ignore(self, user_input)             # 忽略发现
    async def async_step_reauth(self, user_input)             # 重新认证
    async def async_step_reconfigure(self, user_input)        # 重新配置

    # 选项流支持
    @staticmethod
    def async_get_options_flow(config_entry) -> OptionsFlow  # 子类覆盖以支持选项

    # 结果方法
    def async_create_entry(self, *, title, data, options, ...) -> ConfigFlowResult
    def async_abort(self, *, reason, next_flow, ...) -> ConfigFlowResult
    def async_update_and_abort(self, entry, *, unique_id, title, data, ...) -> ConfigFlowResult
    def async_update_reload_and_abort(self, entry, ...) -> ConfigFlowResult
```

### 5.6 配置流程完整示例

**用户手动添加集成**：

```
1. 用户点击"添加集成"
   ↓
2. ConfigEntriesFlowManager.async_init("my_integration", context={"source": "user"})
   ↓
3. async_create_flow() → 实例化 MyConfigFlow, 设置 init_step="user"
   ↓
4. _async_handle_step(flow, "user", None)
   ↓
5. MyConfigFlow.async_step_user(None)
   → 返回 async_show_form(step_id="user", data_schema=SCHEMA)
   ↓
6. 用户填写表单，前端调用 async_configure(flow_id, user_input)
   ↓
7. Schema 校验 user_input
   ↓
8. _async_handle_step(flow, "user", validated_input)
   ↓
9. MyConfigFlow.async_step_user(validated_input)
   → 调用 async_set_unique_id(device_id)
   → 返回 async_create_entry(title="My Device", data={...})
   ↓
10. async_finish_flow() → 创建 ConfigEntry → async_add() → 完成
```

**自动发现流程**：

```
1. 蓝牙发现设备
   → async_init("my_integration", context={"source": "bluetooth"}, data=BluetoothServiceInfoBleak(...))
   ↓
2. init_step="bluetooth" → 调用 async_step_bluetooth(discovery_info)
   ↓
3. 如果未覆盖: 默认实现调用 _async_step_discovery_without_unique_id() → async_step_user()
   如果已覆盖: 通常先 async_set_unique_id()，再返回表单或直接创建条目
```

### 5.7 OptionsFlow — 选项流

```python
class OptionsFlow(ConfigEntryBaseFlow):
    @property
    def config_entry(self) -> ConfigEntry: ...  # 关联的 ConfigEntry

# 常用变体
class OptionsFlowWithReload(OptionsFlow):
    automatic_reload: bool = True  # 选项变更后自动重新加载 ConfigEntry
```

集成通过在 ConfigFlow 中覆盖 `async_get_options_flow` 支持选项流：

```python
class MyConfigFlow(ConfigFlow, domain="my_integration"):
    @staticmethod
    @callback
    def async_get_options_flow(config_entry) -> OptionsFlow:
        return MyOptionsFlow(config_entry)
```

---

## 6. 实体(Entity)体系

### 6.1 Entity 生命周期

```
创建实体实例
  │
  ↓ EntityPlatform.async_add_entities() → _async_add_entity()
  │
  ├── entity.add_to_platform_start()
  │     ├── 设置 entity.hass = hass
  │     ├── 设置 entity.platform = entity_platform
  │     └── 设置 parallel_updates 信号量
  │
  ├── 可选: entity.async_device_update()  (update_before_add=True 时)
  │
  ├── 生成/查找 entity_id
  │     ├── 有 unique_id → 通过 entity_registry 查找或创建
  │     └── 无 unique_id → 根据 suggested_object_id 生成
  │
  ├── hass.states.async_reserve(entity_id)  防止并发冲突
  │
  ├── 注册移除回调
  │
  └── entity.add_to_platform_finish()
        ├── 调用 async_internal_added_to_hass()
        ├── 调用 async_added_to_hass()  ← 子类覆盖的钩子
        └── async_write_ha_state()  写入初始状态
```

### 6.2 状态更新机制

```python
# 推荐方式 — 立即写入状态
entity.async_write_ha_state()

# 传统方式 — 先更新再写入
entity.async_update_ha_state(force_refresh=False)

# 从设备拉取最新数据
entity.async_device_update()
```

`async_write_ha_state()` 的内部流程：
1. 收集实体的 state、capability_attributes、state_attributes
2. 构造 `State` 对象
3. 调用 `hass.states.async_set_internal(entity_id, state, attr, force_update, context, state_info, time_now)`
4. 触发 `EVENT_STATE_CHANGED` 事件

### 6.3 平台特定实体类

#### LightEntity

**继承链**: `Entity → ToggleEntity → LightEntity`

```python
class LightEntity(ToggleEntity):
    # 核心属性
    brightness: int | None                      # 亮度 0..255
    color_mode: ColorMode | None                # 当前颜色模式
    supported_color_modes: set[ColorMode] | None # 支持的颜色模式
    hs_color: tuple[float, float] | None        # HS 颜色
    rgb_color: tuple[int, int, int] | None      # RGB 颜色
    color_temp_kelvin: int | None               # 色温(开尔文)
    min_color_temp_kelvin: int                  # 最小色温
    max_color_temp_kelvin: int                  # 最大色温
    effect: str | None                          # 当前效果
    effect_list: list[str] | None               # 效果列表

    # 核心方法
    async def async_turn_on(self, **kwargs)     # 开灯
    async def async_turn_off(self, **kwargs)     # 关灯
    async def async_toggle(self, **kwargs)       # 切换（重写自 ToggleEntity）

    # 色彩模式 ColorMode
    # ONOFF, BRIGHTNESS, HS, XY, RGB, RGBW, RGBWW, COLOR_TEMP, WHITE, UNKNOWN

    # 特性标志 LightEntityFeature (IntFlag)
    # EFFECT = 4, FLASH = 8, TRANSITION = 32
```

**async_setup 模式**：

```python
# light/__init__.py
async def async_setup(hass, config):
    component = EntityComponent[LightEntity](_LOGGER, DOMAIN, hass, SCAN_INTERVAL)
    await component.async_setup(config)
    # 注册实体服务
    component.async_register_entity_service(SERVICE_TURN_ON, ..., async_handle_light_on_service)
    component.async_register_entity_service(SERVICE_TURN_OFF, ..., async_handle_light_off_service)
    component.async_register_entity_service(SERVICE_TOGGLE, ..., async_handle_toggle_service)

async def async_setup_entry(hass, entry):
    return await hass.data[DATA_COMPONENT].async_setup_entry(entry)
```

#### SensorEntity

**继承链**: `Entity → SensorEntity`

```python
class SensorEntity(Entity):
    native_value: StateType | date | datetime | Decimal  # 传感器原始值
    native_unit_of_measurement: str | None               # 原始计量单位
    device_class: SensorDeviceClass                       # 设备类
    state_class: SensorStateClass                         # 状态类 (MEASUREMENT/TOTAL/TOTAL_INCREASING)
    suggested_display_precision: int | None               # 建议显示精度
    suggested_unit_of_measurement: str | None              # 建议显示单位
    options: list[str] | None                              # 枚举选项

    # state 属性是 @final，不可重写
    # 内部自动处理: 单位转换、精度、校验
```

**SensorEntityDescription** 额外字段：`device_class`, `native_unit_of_measurement`, `options`, `state_class`, `suggested_display_precision`, `suggested_unit_of_measurement`

#### BinarySensorEntity

**继承链**: `Entity → BinarySensorEntity`

```python
class BinarySensorEntity(Entity):
    is_on: bool | None              # 核心属性
    device_class: BinarySensorDeviceClass  # 设备类 (battery, motion, door, ...)

    # state 属性是 @final，根据 is_on 返回 "on" 或 "off"
```

这是最简单的实体类之一，只需实现 `is_on`。

### 6.4 EntityDescription — 实体描述模式

用于将实体属性从子类移到描述对象，减少子类数量：

```python
from homeassistant.helpers.entity import EntityDescription

# 定义描述
SENSOR_DESCRIPTION = SensorEntityDescription(
    key="temperature",
    device_class=SensorDeviceClass.TEMPERATURE,
    native_unit_of_measurement="°C",
    state_class=SensorStateClass.MEASUREMENT,
)

# 使用描述
class MySensor(SensorEntity):
    def __init__(self, description):
        self.entity_description = description
        self._attr_unique_id = f"{description.key}_sensor"
        self._native_value = None

    @property
    def native_value(self):
        return self._native_value
```

### 6.5 CoordinatorEntity — 协调器实体

用于配合 `DataUpdateCoordinator` 实现轮询式更新：

```python
from homeassistant.helpers.update_coordinator import CoordinatorEntity, DataUpdateCoordinator

coordinator = DataUpdateCoordinator(
    hass, LOGGER,
    update_method=async_fetch_data,
    update_interval=timedelta(seconds=30),
)

class MyEntity(CoordinatorEntity, LightEntity):
    def __init__(self, coordinator, light_data):
        super().__init__(coordinator)
        self.light_data = light_data

    @property
    def brightness(self):
        return self.coordinator.data[self.light_data.id]["brightness"]

    # CoordinatorEntity 自动在 coordinator 刷新时调用 async_write_ha_state()
```

---

## 7. 服务(Service)机制

### 7.1 服务注册

```python
# 域级服务注册
hass.services.async_register(
    domain="light",
    service="turn_on",
    service_func=async_handle_turn_on,
    schema=TURN_ON_SCHEMA,
)

# 实体服务注册（更常用）
component.async_register_entity_service(
    SERVICE_TURN_ON,
    TURN_ON_SCHEMA,
    "async_turn_on",  # 字符串，映射到实体方法名
)

# 管理员服务注册
async_register_admin_service(hass, domain, service, service_func, schema)
```

### 7.2 服务调用流程

```
用户/自动化调用 light.turn_on(entity_id="light.living_room", brightness=255)
  │
  ├── ServiceRegistry.async_call("light", "turn_on", service_data)
  │     ├── 查找 handler = _services["light"]["turn_on"]
  │     ├── 校验 return_response 与 supports_response
  │     ├── Schema 校验 service_data
  │     ├── 创建 ServiceCall 对象
  │     ├── bus.fire(EVENT_CALL_SERVICE)
  │     └── 执行服务
  │
  └── 实体服务调用链:
        entity_service_call(hass, entities, service_func, call)
          ├── _resolve_entity_service_call_entities()  # 解析目标实体
          │     ├── 根据 entity_id/area_id/device_id 解析
          │     ├── 过滤不可用的实体
          │     ├── 过滤不满足 required_features 的实体
          │     └── 过滤不满足 entity_device_classes 的实体
          │
          └── _async_handle_entity_calls([(entity, _handle_single_entity_call(...))])
                ├── 单实体: entity.async_request_call(coro)
                ├── 多实体: asyncio.gather 并发调用
                └── _handle_single_entity_call()
                      ├── func 是字符串 → getattr(entity, func)(**data)
                      │                   如 entity.async_turn_on(brightness=255)
                      └── func 是 HassJob → hass.async_run_hass_job(func, entity, data)
```

### 7.3 批量实体服务

`async_register_batched_entity_service` 注册的服务将所有匹配实体一次性传给服务函数：

```python
component.async_register_batched_entity_service(
    SERVICE_GET_FORECASTS,
    GET_FORECASTS_SCHEMA,
    async_get_forecasts,  # func(entities, service_call)
)
```

---

## 8. 平台(Platform)与 EntityComponent 编排

### 8.1 EntityComponent — 域级编排器

**源码**: `homeassistant/helpers/entity_component.py`

`EntityComponent` 是域级组件（如 light、sensor）的核心编排器，管理该域下所有平台和实体。

```python
class EntityComponent[_EntityT = Entity]:
    def __init__(self, logger, domain, hass, scan_interval):
        # 创建根平台（与域同名的 "catch-all" 平台）
        self._platforms: dict[str | tuple, EntityPlatform] = {}
        self._entities: dict[str, Entity]  # 引用根平台的 domain_entities
        hass.data[DATA_INSTANCES][domain] = self  # 全局注册

    async def async_setup(self, config):
        # 遍历配置中的平台，为每个平台创建异步任务

    async def async_setup_entry(self, config_entry):
        # 导入平台模块，创建 EntityPlatform，调用 EntityPlatform.async_setup_entry()

    async def async_unload_entry(self, config_entry):
        # 弹出平台，调用 EntityPlatform.async_reset()

    async_register_entity_service(...)     # 注册逐实体服务
    async_register_batched_entity_service(...)  # 注册批量服务
```

### 8.2 EntityPlatform — 平台与实体的桥梁

**源码**: `homeassistant/helpers/entity_platform.py`

每个 EntityPlatform 实例负责一种集成在一种域下的所有实体。

```python
class EntityPlatform:
    def __init__(self, *, hass, logger, domain, platform_name, platform, scan_interval, entity_namespace):
        self.entities: dict[str, Entity] = {}        # 当前平台的实体
        self.domain_entities: dict[str, Entity]        # 按域索引的所有实体
        self.domain_platform_entities: dict[str, Entity]  # 按(域,平台名)索引的实体
        self.config_entry: ConfigEntry | None = None
        self._setup_complete: bool = False
```

**两条设置路径**：

```
路径 A: YAML 配置
  EntityPlatform.async_setup(platform_config, discovery_info)
    ├── 检查平台模块是否有 async_setup_platform / setup_platform
    └── 调用 platform.async_setup_platform(hass, config, async_add_entities, discovery_info)

路径 B: ConfigEntry
  EntityPlatform.async_setup_entry(config_entry)
    ├── 保存 self.config_entry = config_entry
    └── 调用 platform.async_setup_entry(hass, config_entry, async_add_entities)
```

**共享核心 `_async_setup_platform()`**：

1. 设置 `current_platform` 上下文变量
2. 加载翻译
3. 设置慢启动警告（10秒）
4. 执行 setup awaitable（60秒超时）
5. 等待所有实体添加任务完成
6. 异常处理：`PlatformNotReady` → 线性递增退避重试（`min(tries,6)*30` 秒）
7. 成功：`hass.config.components.add(full_name)`, `_setup_complete = True`

### 8.3 实体添加流程

```
集成代码调用 async_add_entities(entities)
  │  ← 这是 EntityPlatform 传给平台的回调
  │
  ├── _async_schedule_add_entities() / _async_schedule_add_entities_for_entry()
  │     └── 创建 task → async_add_entities()
  │
  └── async_add_entities(new_entities, update_before_add, *, config_subentry_id=None)
        ├── 验证 config_subentry_id
        ├── update_before_add=True → _async_add_and_update_entities() (并发)
        └── update_before_add=False → _async_add_entities() (顺序)
              │
              └── 逐个调用 _async_add_entity(entity, ...)
                    ├── entity.add_to_platform_start()  # 设置 hass/platform 引用
                    ├── 可选: entity.async_device_update()
                    ├── 生成/查找 entity_id
                    │     ├── 有 unique_id → entity_registry 查找/创建
                    │     └── 无 unique_id → suggested_object_id 生成
                    ├── 禁用检查 → 已禁用则跳过
                    ├── 注册到三个字典:
                    │     self.entities[entity_id] = entity
                    │     self.domain_entities[entity_id] = entity
                    │     self.domain_platform_entities[entity_id] = entity
                    ├── hass.states.async_reserve(entity_id)
                    └── entity.add_to_platform_finish()
                          ├── async_internal_added_to_hass()
                          ├── async_added_to_hass()  ← 子类钩子
                          └── async_write_ha_state()
```

### 8.4 轮询机制

当有 `should_poll=True` 的实体时，EntityPlatform 启动轮询定时器：

```
_async_handle_interval_callback()
  ├── 重新调度自身（scan_interval 秒后）
  └── 创建后台任务 → _async_update_entity_states()
                        ├── 少量实体: 顺序更新
                        └── 多量实体: 并发 gather 更新

并行度控制: PARALLEL_UPDATES 常量
  ├── 平台模块可定义 PARALLEL_UPDATES 常量
  ├── 未定义且有同步 update 方法 → 默认为 1
  └── 设为 0 则不限制并行
```

### 8.5 架构层次总览

```
HomeAssistant (hass)
  └─ EntityComponent (如 "light")
       │  管理 domain 级别的逻辑
       │
       ├─ EntityPlatform (platform_name="light")  ← 根平台，catch-all
       ├─ EntityPlatform (platform_name="hue")    ← hue 集成的 light 平台
       ├─ EntityPlatform (platform_name="tplink") ← tplink 集成的 light 平台
       └─ EntityPlatform (platform_name="zwave")  ← zwave 集成的 light 平台
            │
            └─ entities: {entity_id: Entity}  ← 该平台管理的所有实体
```

`hass.data` 中的关键数据结构：

| Key | 类型 | 说明 |
|-----|------|------|
| `DATA_ENTITY_PLATFORM` | `dict[str, list[EntityPlatform]]` | 按集成名索引的所有 EntityPlatform |
| `DATA_DOMAIN_ENTITIES` | `dict[str, dict[str, Entity]]` | 按域索引的所有实体 |
| `DATA_DOMAIN_PLATFORM_ENTITIES` | `dict[tuple, dict[str, Entity]]` | 按(域, 平台名)索引的实体 |
| `DATA_INSTANCES` | `dict[str, EntityComponent]` | 按域索引的 EntityComponent 实例 |

---

## 9. 集成实现完整示例：Hue

### 9.1 整体架构

```
用户发现/配置桥接器
    → ConfigFlow 创建 ConfigEntry
    → async_setup_entry 创建 HueBridge 实例
    → HueBridge 初始化 API 并转发平台设置
    → 各平台 (light/sensor/...) 的 async_setup_entry 创建实体
```

### 9.2 `__init__.py` — 集成入口

```python
DOMAIN = "hue"
type HueConfigEntry = ConfigEntry[HueBridge]

async def async_setup(hass: HomeAssistant, config: ConfigType) -> bool:
    async_setup_services(hass)  # 注册全局服务
    return True

async def async_setup_entry(hass: HomeAssistant, entry: HueConfigEntry) -> bool:
    await check_migration(hass, entry)
    bridge = HueBridge(hass, entry)
    if not await bridge.async_initialize_bridge():
        return False
    # unique_id 修正 & 设备注册
    return True

async def async_unload_entry(hass: HomeAssistant, entry: HueConfigEntry) -> bool:
    return await entry.runtime_data.async_reset()
```

### 9.3 `bridge.py` — 桥接器管理

```python
class HueBridge:
    def __init__(self, hass, config_entry):
        self.config_entry = config_entry
        if self.api_version == 1:
            self.api = HueBridgeV1(self.host, app_key, session)
        else:
            self.api = HueBridgeV2(self.host, app_key)
        self.config_entry.runtime_data = self  # 自注册

    async def async_initialize_bridge(self) -> bool:
        # 1. 连接 API
        try:
            async with asyncio.timeout(10):
                await self.api.initialize()
        except Unauthorized:
            create_config_flow(self.hass, self.host)  # 重新触发配对
            return False
        except (TimeoutError, ...):
            raise ConfigEntryNotReady(...)  # HA 会自动重试

        # 2. 根据版本转发平台设置
        if self.api_version == 1:
            await self.hass.config_entries.async_forward_entry_setups(
                self.config_entry, PLATFORMS_v1
            )  # [BINARY_SENSOR, LIGHT, SENSOR]
        else:
            await self.hass.config_entries.async_forward_entry_setups(
                self.config_entry, PLATFORMS_v2
            )  # [BINARY_SENSOR, EVENT, LIGHT, SCENE, SENSOR, SWITCH]

        # 3. 注册选项更新监听器
        self.reset_jobs.append(
            self.config_entry.add_update_listener(_update_listener)
        )
        return True

    async def async_reset(self):
        # 卸载所有平台 + 清理
        while self.reset_jobs:
            self.reset_jobs.pop()()
        unload_success = await self.hass.config_entries.async_unload_platforms(
            self.config_entry, PLATFORMS_v1 if ... else PLATFORMS_v2
        )
        if unload_success:
            delattr(self.config_entry, "runtime_data")
        return unload_success
```

### 9.4 `config_flow.py` — 配置流

```python
class HueFlowHandler(ConfigFlow, domain=DOMAIN):
    VERSION = 1

    async def async_step_user(self, user_input=None):
        # 展示桥接器列表或手动输入
        if user_input is None:
            bridges = await discover_nupnp()
            return self.async_show_form(step_id="init", ...)

    async def async_step_link(self, user_input=None):
        # 请求用户按 Link 按钮
        if user_input is None:
            return self.async_show_form(step_id="link")

        try:
            app_key = await create_app_key(bridge.host, ...)
        except LinkButtonNotPressed:
            errors["base"] = "register_failed"

        return self.async_create_entry(
            title=f"Hue Bridge {bridge.id}",
            data={CONF_HOST: bridge.host, CONF_API_KEY: app_key, CONF_API_VERSION: 2},
        )

    # 三种发现入口
    async def async_step_zeroconf(self, discovery_info): ...  # mDNS 发现
    async def async_step_homekit(self, discovery_info): ...  # HomeKit 发现
    async def async_step_import(self, import_data): ...      # YAML 导入
```

### 9.5 `light.py` — 平台文件

```python
async def async_setup_entry(hass, config_entry, async_add_entities):
    bridge = config_entry.runtime_data
    if bridge.api_version == 1:
        await setup_entry_v1(hass, config_entry, async_add_entities)
    else:
        await setup_entry_v2(hass, config_entry, async_add_entities)
```

**V1 路径 — 轮询模式**：

```python
async def setup_entry_v1(hass, config_entry, async_add_entities):
    bridge = config_entry.runtime_data

    # 创建 DataUpdateCoordinator（5秒轮询）
    coordinator = DataUpdateCoordinator(
        hass, LOGGER, name="light",
        update_method=partial(async_safe_fetch, bridge, bridge.api.lights.update),
        update_interval=timedelta(seconds=5),
    )
    await coordinator.async_refresh()
    if not coordinator.last_update_success:
        raise PlatformNotReady

    # 创建实体
    async_add_entities(HueLight(coordinator, light) for light in bridge.api.lights.values())

class HueLight(CoordinatorEntity, LightEntity):
    _attr_should_poll = False  # 由 Coordinator 管理更新

    async def async_turn_on(self, **kwargs):
        await bridge.async_request_call(self.light.set_state, **command)
```

**V2 路径 — 事件驱动模式**：

```python
async def setup_entry_v2(hass, config_entry, async_add_entities):
    bridge = config_entry.runtime_data
    controller = bridge.api.lights

    # 立即添加所有当前灯
    async_add_entities(make_light_entity(light) for light in controller)

    # 注册事件监听器，动态添加新灯
    config_entry.async_on_unload(
        controller.subscribe(async_add_light, event_filter=EventType.RESOURCE_ADDED)
    )

class HueLight(HueBaseEntity, LightEntity):
    _attr_should_poll = False  # V2 事件驱动，不需要轮询

class HueBaseEntity(Entity):
    _attr_should_poll = False

    async def async_added_to_hass(self):
        # 订阅资源更新和删除事件
        self.async_on_remove(
            self.controller.subscribe(
                self._handle_event, self.resource.id,
                (EventType.RESOURCE_UPDATED, EventType.RESOURCE_DELETED),
            )
        )

    def _handle_event(self, event_type, resource):
        if event_type == EventType.RESOURCE_DELETED:
            # 自动移除实体
        else:
            self.async_write_ha_state()  # 推送状态更新
```

### 9.6 完整流程图

```
┌─────────────────────────────────────────────────────────────────┐
│  1. 发现阶段 (config_flow.py)                                    │
│                                                                 │
│  Zeroconf/HomeKit/手动 → HueFlowHandler                          │
│    → async_step_link() 请求用户按 Link 按钮                     │
│    → create_app_key() 获取 API 密钥                             │
│    → async_create_entry(data={host, api_key, api_version})      │
└───────────────────────────┬─────────────────────────────────────┘
                            │ 创建 ConfigEntry
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. 集成初始化 (__init__.py)                                     │
│                                                                 │
│  async_setup_entry(hass, entry):                                │
│    → bridge = HueBridge(hass, entry)                            │
│    → bridge.async_initialize_bridge()                           │
│        → api.initialize()  连接 API                              │
│        → async_forward_entry_setups()  转发到各平台               │
│    → entry.runtime_data = bridge                                │
└───────────────────────────┬─────────────────────────────────────┘
                            │ 转发到平台
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. 平台初始化 (light.py → v1/light.py 或 v2/light.py)         │
│                                                                 │
│  V1: DataUpdateCoordinator(5秒轮询) → HueLight(CoordinatorEntity)│
│  V2: 事件驱动 → HueLight(HueBaseEntity) → async_write_ha_state  │
└───────────────────────────┬─────────────────────────────────────┤
                            │ 实体运行
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. 实体运行时                                                   │
│                                                                 │
│  V1: Coordinator 定时轮询 → CoordinatorEntity 自动更新状态      │
│  V2: SSE 事件推送 → HueBaseEntity._handle_event → async_write   │
│  控制: bridge.async_request_call(api_method, **params)          │
│  卸载: bridge.async_reset() → unload_platforms → 清理          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 10. 如何实现自己的集成

### 10.1 最小集成结构

```
custom_components/my_integration/
├── manifest.json
├── __init__.py
└── sensor.py          # 向 sensor 域提供实体（可选）
```

### 10.2 manifest.json

```json
{
  "domain": "my_integration",
  "name": "My Integration",
  "codeowners": ["@your_github"],
  "config_flow": true,
  "documentation": "https://www.example.com",
  "integration_type": "device",
  "iot_class": "local_polling",
  "requirements": ["my-library==1.0.0"],
  "zeroconf": ["_myservice._tcp.local."]
}
```

### 10.3 `__init__.py` — 集成入口

```python
from homeassistant.config_entries import ConfigEntry
from homeassistant.core import HomeAssistant
from homeassistant.const import Platform

DOMAIN = "my_integration"
type MyConfigEntry = ConfigEntry[MyRuntimeData]

PLATFORMS = [Platform.SENSOR, Platform.BINARY_SENSOR]


async def async_setup_entry(hass: HomeAssistant, entry: MyConfigEntry) -> bool:
    """从 ConfigEntry 设置集成。"""
    # 1. 初始化运行时数据
    runtime_data = MyRuntimeData(hass, entry)
    entry.runtime_data = runtime_data

    # 2. 初始化连接
    try:
        await runtime_data.async_connect()
    except ConnectionError as err:
        raise ConfigEntryNotReady(f"Connection failed: {err}") from err

    # 3. 转发平台设置
    await hass.config_entries.async_forward_entry_setups(entry, PLATFORMS)

    # 4. 注册选项更新监听器
    entry.async_on_unload(entry.add_update_listener(_async_update_listener))

    return True


async def async_unload_entry(hass: HomeAssistant, entry: MyConfigEntry) -> bool:
    """卸载 ConfigEntry。"""
    # 卸载所有平台
    unload_ok = await hass.config_entries.async_unload_platforms(entry, PLATFORMS)
    if unload_ok:
        await entry.runtime_data.async_disconnect()
    return unload_ok


async def _async_update_listener(hass: HomeAssistant, entry: MyConfigEntry) -> None:
    """选项变更时重新加载。"""
    await hass.config_entries.async_reload(entry.entry_id)
```

### 10.4 `config_flow.py` — 配置流

```python
from homeassistant import config_entries
from homeassistant.core import callback
import voluptuous as vol

from .const import DOMAIN, CONF_HOST, CONF_API_KEY


class MyConfigFlow(config_entries.ConfigFlow, domain=DOMAIN):
    """My Integration 配置流。"""

    VERSION = 1

    async def async_step_user(self, user_input=None):
        """用户手动配置步骤。"""
        errors = {}

        if user_input is not None:
            # 验证连接
            try:
                await validate_input(self.hass, user_input)
            except CannotConnect:
                errors["base"] = "cannot_connect"
            except InvalidAuth:
                errors["base"] = "invalid_auth"
            else:
                # 设置唯一 ID 防止重复配置
                await self.async_set_unique_id(user_input[CONF_HOST])
                self._abort_if_unique_id_configured()
                return self.async_create_entry(
                    title=f"My Device ({user_input[CONF_HOST]})",
                    data=user_input,
                )

        return self.async_show_form(
            step_id="user",
            data_schema=vol.Schema({
                vol.Required(CONF_HOST): str,
                vol.Required(CONF_API_KEY): str,
            }),
            errors=errors,
        )

    async def async_step_zeroconf(self, discovery_info):
        """Zeroconf 自动发现步骤。"""
        host = discovery_info.host
        await self.async_set_unique_id(host)
        self._abort_if_unique_id_configured()

        # 展示确认表单
        return self.async_show_form(
            step_id="user",
            data_schema=vol.Schema({
                vol.Required(CONF_HOST, default=host): str,
                vol.Required(CONF_API_KEY): str,
            }),
        )

    @staticmethod
    @callback
    def async_get_options_flow(config_entry):
        """选项流。"""
        return MyOptionsFlow(config_entry)


class MyOptionsFlow(config_entries.OptionsFlowWithReload):
    """选项流。"""

    def __init__(self, config_entry):
        self.config_entry = config_entry

    async def async_step_init(self, user_input=None):
        if user_input is not None:
            return self.async_create_entry(data=user_input)

        return self.async_show_form(
            step_id="init",
            data_schema=vol.Schema({
                vol.Required("poll_interval", default=30): int,
            }),
        )
```

### 10.5 `sensor.py` — 向 sensor 域提供实体

```python
from homeassistant.components.sensor import SensorEntity, SensorEntityDescription
from homeassistant.config_entries import ConfigEntry
from homeassistant.core import HomeAssistant
from homeassistant.helpers.entity_platform import AddEntitiesCallback
from homeassistant.helpers.update_coordinator import (
    CoordinatorEntity,
    DataUpdateCoordinator,
)

from .const import DOMAIN


async def async_setup_entry(
    hass: HomeAssistant,
    config_entry: ConfigEntry,
    async_add_entities: AddEntitiesCallback,
) -> None:
    """设置传感器平台。"""
    runtime_data = config_entry.runtime_data

    # 创建协调器（轮询式）
    coordinator = DataUpdateCoordinator(
        hass,
        __name__,
        update_method=runtime_data.async_fetch_data,
        update_interval=timedelta(seconds=30),
    )
    await coordinator.async_config_entry_first_refresh()

    # 添加实体
    async_add_entities(
        MySensor(coordinator, description)
        for description in SENSORS
    )


SENSORS = [
    SensorEntityDescription(
        key="temperature",
        name="Temperature",
        native_unit_of_measurement="°C",
        device_class=SensorDeviceClass.TEMPERATURE,
        state_class=SensorStateClass.MEASUREMENT,
    ),
]


class MySensor(CoordinatorEntity, SensorEntity):
    """My Integration 传感器实体。"""

    _attr_has_entity_name = True

    def __init__(self, coordinator, description):
        super().__init__(coordinator)
        self.entity_description = description
        self._attr_unique_id = description.key

    @property
    def native_value(self):
        """返回传感器值。"""
        return self.coordinator.data.get(self.entity_description.key)
```

### 10.6 关键实现要点

1. **runtime_data 模式**：使用 `ConfigEntry[MyData]` 泛型，在 `async_setup_entry` 中设置 `entry.runtime_data`，平台通过 `config_entry.runtime_data` 获取，类型安全。

2. **ConfigEntryNotReady**：连接失败时抛出此异常，HA 会自动重试设置。

3. **async_forward_entry_setups**：在集成入口的 `async_setup_entry` 中调用，将配置条目转发到各平台。

4. **async_unload_platforms**：在 `async_unload_entry` 中调用，卸载所有平台。

5. **async_on_unload**：注册卸载时的清理回调，如取消订阅、关闭连接。

6. **DataUpdateCoordinator**：轮询式更新的推荐方式，自动处理轮询间隔、错误重试、状态更新。

7. **CoordinatorEntity**：与 Coordinator 配合的实体基类，自动在 Coordinator 刷新时更新状态。

8. **EntityDescription**：将实体属性从子类移到描述对象，支持一个实体类多个实例。

9. **should_poll**：事件驱动集成的实体应设 `_attr_should_poll = False`，通过 `async_write_ha_state()` 主动推送更新。

---

## 11. 关键设计模式总结

### 11.1 架构模式

| 模式 | 实现 |
|------|------|
| **事件驱动** | EventBus + StateMachine，所有组件间通信通过事件 |
| **分层架构** | 入口层 → 核心层 → 加载层 → 实体层 → 集成层 |
| **泛型约束** | `EntityComponent[LightEntity]`, `ConfigEntry[HueBridge]` |
| **注册表模式** | ServiceRegistry, EntityRegistry, DeviceRegistry, HANDLERS 等 |
| **观察者模式** | EventBus 监听器, Coordinator 订阅, ConfigEntry 更新监听 |

### 11.2 加载模式

| 模式 | 实现 |
|------|------|
| **分阶段启动** | Stage 0/1/2，基础设施先行 |
| **并行加载** | `asyncio.gather` 并行设置 ConfigEntry 和平台 |
| **依赖注入** | `hass.data` 全局字典 + `DATA_INSTANCES` |
| **惰性加载** | 自定义集成按需解析，平台模块按需导入 |
| **线性递增退避重试** | `PlatformNotReady` → `min(tries,6)*30` 秒递增等待（30, 60, 90, ... 180s） |

### 11.3 实体模式

| 模式 | 实现 |
|------|------|
| **轮询式** | `should_poll=True` + EntityPlatform 定时轮询 |
| **协调器式** | `DataUpdateCoordinator` + `CoordinatorEntity` |
| **事件驱动式** | `should_poll=False` + `async_write_ha_state()` 主动推送 |
| **描述模式** | `EntityDescription` 将属性从子类移到描述对象 |
| **_attr_* 模式** | 类属性默认值，减少 property 定义 |

### 11.4 集成模式

| 模式 | 实现 |
|------|------|
| **runtime_data 模式** | `entry.runtime_data` 存储运行时对象，类型安全 |
| **平台转发模式** | `async_forward_entry_setups` 将设置传播到各平台 |
| **配置流模式** | `ConfigFlow` + 步骤方法，支持 UI 配置 |
| **统一错误处理** | `async_request_call` 包装 API 调用，统一异常转换 |
| **选项 = 重载** | 选项变更触发 `async_reload`，重新初始化整个集成 |

### 11.5 @final 保护

以下关键属性/方法标记为 `@final`，子类不可覆盖：

- `LightEntity.state_attributes` — 灯光状态序列化逻辑
- `SensorEntity.state` — 包含单位转换、精度处理的完整校验
- `SensorEntity.unit_of_measurement` — 单位转换管道
- `BinarySensorEntity.state` — 从 `is_on` 自动派生
- `ToggleEntity.state` — 从 `is_on` 自动派生

---

## 附录：关键源码文件索引

| 文件 | 行数 | 说明 |
|------|------|------|
| `homeassistant/core.py` | 2878 | HomeAssistant 主类、EventBus、StateMachine、ServiceRegistry |
| `homeassistant/config_entries.py` | 4180 | ConfigEntry、ConfigFlow、ConfigEntries 管理器 |
| `homeassistant/bootstrap.py` | 1084 | 启动引导 |
| `homeassistant/loader.py` | 1789 | Integration 类、manifest 解析 |
| `homeassistant/setup.py` | 842 | async_setup_component |
| `homeassistant/data_entry_flow.py` | 940 | FlowHandler/FlowManager 基类 |
| `homeassistant/helpers/entity.py` | 1780 | Entity 基类 |
| `homeassistant/helpers/entity_platform.py` | 1348 | EntityPlatform |
| `homeassistant/helpers/entity_component.py` | 398 | EntityComponent |
| `homeassistant/helpers/service.py` | 1408 | 服务辅助函数 |
| `homeassistant/const.py` | 1011 | 全局常量 |
| `homeassistant/runner.py` | 330 | 运行器 |
