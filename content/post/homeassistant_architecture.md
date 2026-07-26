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
  - [5.7 ConfigFlow step 方法职责与调用时机](#57-configflow-step-方法职责与调用时机)
  - [5.8 ConfigEntry 创建、更新与 `unique_id`](#58-configentry-创建更新与-unique_id)
  - [5.9 ConfigFlow 实例生命周期与 BLE 发现去重](#59-configflow-实例生命周期与-ble-发现去重)
  - [5.10 OptionsFlow — 选项流](#510-optionsflow--选项流)
- [6. 实体(Entity)体系](#6-实体entity体系)
- [7. 服务(Service)机制](#7-服务service机制)
- [8. 平台(Platform)与 EntityComponent 编排](#8-平台platform与-entitycomponent-编排)
- [9. 集成实现完整示例：Hue](#9-集成实现完整示例hue)
- [10. 如何实现自己的集成](#10-如何实现自己的集成)
  - [10.1 最小集成结构](#101-最小集成结构)
  - [10.2 manifest.json](#102-manifestjson)
  - [10.3 `__init__.py` — 集成入口](#103-__init__py--集成入口)
  - [10.4 `config_flow.py` — 配置流](#104-config_flowpy--配置流)
  - [10.5 `sensor.py` — 向 sensor 域提供实体（CoordinatorEntity 模式）](#105-sensorpy--向-sensor-域提供实体coordinatorentity-模式)
  - [10.6 从 YAML 配置加载集成](#106-从-yaml-配置加载集成)
  - [10.7 不使用 Coordinator 的 SensorEntity 实现模式](#107-不使用-coordinator-的-sensorentity-实现模式)
    - [10.7.1 轮询模式 — 纯 SensorEntity + async_update](#1071-轮询模式--纯-sensorentity--asyncupdate)
    - [10.7.2 推送模式 — SensorEntity + dispatcher + async_write_ha_state](#1072-推送模式--sensorentity--dispatcher--async_write_ha_state)
  - [10.8 关键实现要点](#108-关键实现要点)
- [11. 蓝图(Blueprint)机制](#11-蓝图blueprint机制)
  - [11.1 蓝图概述与使用方法](#111-蓝图概述与使用方法)
  - [11.2 蓝图 YAML 格式与自定义蓝图](#112-蓝图-yaml-格式与自定义蓝图)
  - [11.3 蓝图源码解析：导入、校验与替换](#113-蓝图源码解析导入校验与替换)
  - [11.4 蓝图与自动化/脚本的集成](#114-蓝图与自动化脚本的集成)
- [12. 关键设计模式总结](#12-关键设计模式总结)

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
├── strings.json        # Core 集成英文翻译源；自定义集成通常使用 translations/en.json
├── icons.json          # 前端图标资源（可选）
├── translations/       # 自定义集成运行时翻译目录（可选）
│   └── en.json
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

**`entry_id` 与 `unique_id` 的区别**：

| 字段 | 谁生成/设置 | 是否必须 | 主要用途 | 典型值 |
|------|-------------|----------|----------|--------|
| `entry_id` | HA core 在 `ConfigEntry.__init__` 中生成（未传入时使用 ULID） | 必有 | Core 内部主键：setup/unload/reload/remove、存储、设备/实体注册表关联 | `01J...` 形式的 ULID |
| `unique_id` | 集成在 `ConfigFlow` 中调用 `async_set_unique_id()` 设置 | 可选但强烈建议 | 识别同一个真实设备/账号/服务，防止重复配置；可用于发现流匹配已有条目 | MAC、序列号、云账号 ID、网关 ID |

二者不是同一个概念：`entry_id` 是 HA 分配给“这一条配置记录”的不可变内部 ID；`unique_id` 是集成提供给 HA 的“真实对象身份”。删除后重新添加同一设备时，通常会得到新的 `entry_id`，但应继续使用相同的 `unique_id`，这样 HA 可以在发现阶段判断设备是否已经配置过。

**ConfigEntry 的创建路径**：自定义集成通常不直接实例化 `ConfigEntry`。在 `config_flow.py` 中，集成只返回 `self.async_create_entry(title=..., data=...)` 这样的 `CREATE_ENTRY` 结果；流程结束时由 `ConfigEntriesFlowManager.async_finish_flow()` 根据结果和 `flow.unique_id` 构造真正的 `ConfigEntry`，并调用 `hass.config_entries.async_add(entry)`。此外，HA 启动时还会从 `.storage/core.config_entries` 反序列化已有条目并重建 `ConfigEntry` 对象。因此，“新条目”主要来自 config flow 完成阶段，但 `ConfigEntry` 对象并不只会在 config flow 中被实例化。

**`unique_id` 的设置时机**：通常在某个 `async_step_*` 中拿到可稳定识别设备的信息后立即调用：

```python
await self.async_set_unique_id(device_serial_or_mac)
self._abort_if_unique_id_configured()
```

`async_set_unique_id()` 会把值记录到 flow context，并检查同 domain 下是否已有相同 `unique_id` 的条目或正在进行的 flow；真正创建 `ConfigEntry` 时，core 再把 `flow.unique_id` 写入 `ConfigEntry.unique_id`。如果设备还无法稳定识别，不应使用临时 host/IP 作为 `unique_id`；更好的做法是先探测设备，拿到序列号、MAC、云端账号 ID 等稳定标识后再设置。

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

### 5.7 ConfigFlow step 方法职责与调用时机

`ConfigFlow` 的每个 `async_step_*` 方法都不是随便命名的回调，而是由 `FlowManager` 根据当前 step id 反射调用：`_async_handle_step(flow, step_id, data)` 会执行 `flow.async_step_{step_id}(user_input)`。对 `ConfigFlow` 来说，初始 step 来自 `context["source"]`，所以不同来源会进入不同方法：

| step 方法 | 何时被调用 | 主要职责 | 常见返回 |
|-----------|------------|----------|----------|
| `async_step_user` | 用户在 UI 点击“添加集成”；或默认发现流程转入手动确认 | 展示表单、校验用户输入、连接设备/服务、设置 `unique_id`、创建条目 | `async_show_form` / `async_create_entry` / `async_abort` |
| `async_step_import` | YAML 配置通过 `hass.config_entries.flow.async_init(..., context={"source": SOURCE_IMPORT})` 导入 | 把 YAML 配置迁移成 config entry；避免和 UI 配置重复 | `async_create_entry` / `async_abort` |
| `async_step_bluetooth` | Bluetooth 集成发现到匹配 manifest matcher 的 BLE 设备 | 从 `BluetoothServiceInfoBleak` 提取 MAC/名称/service data，设置稳定 `unique_id`，必要时探测设备，进入确认或密钥步骤 | `async_show_form` / `async_create_entry` / `async_abort` |
| `async_step_zeroconf` / `dhcp` / `ssdp` / `usb` / `homekit` / `mqtt` | 对应发现源产生匹配结果 | 从发现数据提取 host、serial、MAC 等，设置 `unique_id`，提示用户确认或补充认证 | `async_show_form` / `async_create_entry` |
| `async_step_reauth` | 已有条目认证失败，集成调用 reauth flow | 找到原 `ConfigEntry`，重新收集 token/password，成功后更新并 reload 原条目 | `async_update_reload_and_abort` |
| `async_step_reconfigure` | 用户对已有条目发起重新配置 | 修改连接参数、账号、host 等核心配置；不应创建新条目 | `async_update_reload_and_abort` / `async_abort` |
| `async_step_ignore` | 用户忽略某个发现项 | 记录忽略状态，阻止同一发现项反复弹出 | `async_abort` |

典型 step 的工作顺序是：

```
收到 user_input / discovery_info
  ├── 如果没有输入：返回 async_show_form()
  ├── 校验 schema 和连接能力
  ├── 获取稳定身份：serial / MAC / account id
  ├── await async_set_unique_id(stable_id)
  ├── _abort_if_unique_id_configured()
  └── async_create_entry(...) 或进入下一个 async_step_xxx
```

需要注意：`async_create_entry()` 只是返回 `CREATE_ENTRY` 类型的 flow result，它本身不是 `ConfigEntry` 构造器。真正的 `ConfigEntry` 由 core 在 flow 完成时创建。

### 5.8 ConfigEntry 创建、更新与 `unique_id`

`ConfigEntriesFlowManager.async_finish_flow()` 处理 `CREATE_ENTRY` 结果时，会把 flow result 中的 `data/title/options/version` 与 `flow.unique_id` 合并，实例化 `ConfigEntry`，再加入 `ConfigEntries` 管理器。伪代码如下：

```python
entry = ConfigEntry(
    data=result["data"],
    domain=result["handler"],
    source=flow.context["source"],
    title=result["title"],
    unique_id=flow.unique_id,
    version=result["version"],
    minor_version=result["minor_version"],
    options=result["options"],
    subentries_data=result["subentries"],
    discovery_keys=discovery_keys,
)
await self.config_entries.async_add(entry)
```

因此集成作者在 `config_flow.py` 中应关注“返回正确的 flow result”，而不是手工 new 一个 `ConfigEntry`。已有条目的修改也不应直接改属性，core 对 `entry_id`、`unique_id`、`data`、`options` 等字段有冻结/更新管理逻辑；需要通过 `hass.config_entries.async_update_entry(...)`、`async_update_reload_and_abort(...)` 等 API 更新。

`async_set_unique_id()` 的作用包括：

1. 把 `unique_id` 写入当前 flow 的 context。
2. 检查同一 domain 是否已有相同 `unique_id` 的 `ConfigEntry`。
3. 检查是否已有相同 `unique_id` 的 flow 正在进行，避免并发重复配置。
4. 对发现流，把默认 discovery unique id 替换为真实设备 id 后，终止重复的默认发现 flow。

所以实践中应尽早设置真实 `unique_id`，并紧跟 `_abort_if_unique_id_configured()`：

```python
await self.async_set_unique_id(format_mac(discovery_info.address))
self._abort_if_unique_id_configured()
```

### 5.9 ConfigFlow 实例生命周期与 BLE 发现去重

`ConfigFlow` 实例代表“一次正在进行的配置流程”，不是集成生命周期内的单例。每次调用 `hass.config_entries.flow.async_init(domain, context=..., data=...)`，core 都会通过该 domain 注册的 handler class 创建一个新的 flow 实例，分配新的 `flow_id`，并记录在 `_progress` 中。用户手动添加一次、一次 YAML import、一次 BLE discovery、一次 reauth，通常都是不同的 flow 实例。

但“会多次实例化”不等于“每收到一次 BLE 广播都调用一次 `async_step_bluetooth`”。Bluetooth 路径有多层去重：

```
BLE advertisement
  ↓
bluetooth.manager._discover_service_info()
  ├── IntegrationMatcher.match_domains(service_info)
  │     └── 若同 address 的 name / manufacturer data / service data / service UUIDs 等字段此前都见过，返回空集合
  ↓
helpers.discovery_flow.async_create_flow(..., source="bluetooth", data=service_info)
  └── 若已有相同 domain/source/data 的发现 flow 正在进行，则不再 async_init
  ↓
ConfigEntriesFlowManager.async_init(...)
  └── 创建新的 ConfigFlow 实例，init_step="bluetooth"
        ↓
      async_step_bluetooth(discovery_info)
```

因此更准确的说法是：**每个“匹配且未被去重的发现事件”可能启动一个新的 config flow，并调用一次该 flow 实例的 `async_step_bluetooth`；重复广播如果没有新增可匹配字段，通常在 Bluetooth matcher 层就被抑制，不会进入 config flow。** 如果设备消失、历史被清理、已有 config entry 被删除，或同一地址广播出现新的 service data/manufacturer data 等字段，HA 可能重新触发发现。

真实 BLE 集成常见写法类似：

```python
async def async_step_bluetooth(self, discovery_info: BluetoothServiceInfoBleak):
    await self.async_set_unique_id(format_mac(discovery_info.address))
    self._abort_if_unique_id_configured()

    self._discovered = {
        CONF_ADDRESS: discovery_info.address,
        CONF_NAME: discovery_info.name,
    }

    # 可选：探测设备能力、检查是否支持、读取加密要求
    if not await device_is_supported(discovery_info):
        return self.async_abort(reason="not_supported")

    self.context["title_placeholders"] = {"name": discovery_info.name}
    self._set_confirm_only()
    return self.async_show_form(step_id="bluetooth_confirm")

async def async_step_bluetooth_confirm(self, user_input=None):
    if user_input is not None:
        return self.async_create_entry(
            title=self._discovered[CONF_NAME],
            data={CONF_ADDRESS: self._discovered[CONF_ADDRESS]},
        )
    return self.async_show_form(step_id="bluetooth_confirm")
```

BTHome、Acaia 等内置集成都遵循这个模式：`async_step_bluetooth()` 先用 BLE address 或格式化 MAC 设置 `unique_id`，检测是否已经配置，然后保存 discovery info，最后进入确认表单、加密密钥表单或直接创建 entry。

### 5.10 OptionsFlow — 选项流

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

### 7.4 services.yaml — 服务说明文件

`services.yaml` **不是服务注册入口**。真正让服务可以被调用的是 Python 代码里的 `hass.services.async_register(...)`、`component.async_register_entity_service(...)`、`component.async_register_batched_entity_service(...)` 或 `async_register_admin_service(...)`；`services.yaml` 只是为这些已经注册的服务提供“用户文档”：前端服务面板、开发者工具、自动化动作编辑器、LLM action 描述等地方会用它展示字段、目标对象、选择器、示例和分组。

源码里有两个关键边界：

1. `loader.Integration.has_services` 只检查集成目录顶层是否存在 `services.yaml`。
2. `homeassistant.helpers.service.async_get_all_descriptions()` 会遍历当前已经注册到 `ServiceRegistry` 的服务；如果某个 domain 的服务缺少描述，并且该集成 `has_services=True`，才从 `integration.file_path / "services.yaml"` 加载描述并合并到返回结果。

也就是说：**只写 `services.yaml` 不会注册任何服务；只注册服务但不写 `services.yaml`，服务仍可被调用，但前端缺少友好的字段说明。** hassfest 也会扫描集成源码，如果发现注册服务的代码却没有 `services.yaml`，会报 `Registers services but has no services.yaml`。

一个服务描述文件的结构大致如下：

```yaml
refresh:
  target:
    entity:
      domain: sensor
      integration: my_integration
  fields:
    force:
      required: false
      default: false
      example: true
      selector:
        boolean:
    options:
      collapsed: true
      fields:
        timeout:
          default: 10
          selector:
            number:
              min: 1
              max: 60
              unit_of_measurement: seconds
```

| 键 | 作用 |
|----|------|
| 顶层 `<service>` | 服务名，必须是 slug；最终完整服务名是 `<domain>.<service>`，例如 `my_integration.refresh`。 |
| `target` | 目标选择器，用于实体服务常见的 `entity_id` / `area_id` / `device_id` 选择 UI。常见写法是 `target.entity.domain` 限制实体域，`target.entity.integration` 限制集成来源。 |
| `fields` | 服务参数定义。每个字段名应与 Python 服务 schema / handler 使用的字段一致。 |
| `required` | 该字段在 UI 和 schema 说明中是否必填；真正的强校验仍应由 Python 的 voluptuous schema 完成。 |
| `default` | UI 默认值或说明默认值；应和 Python 端默认值保持一致。 |
| `example` | 开发者工具里展示的示例值。 |
| `advanced` | 标记为高级字段，前端可以折叠或弱化展示。 |
| `selector` | 前端输入控件，例如 `text`、`number`、`boolean`、`select`、`template`、`duration`、`entity`、`device` 等。 |
| `filter` | 仅 targeted service 可用，用于按实体 `supported_features` 或属性能力过滤字段显示；light 的 `transition`、`rgb_color` 等字段大量使用这种模式。 |
| 字段分区 `<section>` | `fields` 下的某个 key 如果自身包含 `fields`，它就是 section，而不是普通字段。section 可写 `collapsed: true` 并嵌套一组字段。 |
| `name` / `description` | 自定义集成 schema 允许在 `services.yaml` 内直接写名称和说明；更推荐放到 `translations/en.json` 的 `services` 分类，便于本地化。 |

注意几个源码约束：

- `services.yaml` 的顶层服务值可以为空，例如 MQTT 的 `reload:`，表示该服务没有字段。
- core 集成可以用 YAML anchor 复用片段，例如 light 的 `.brightness_support: &brightness_support`；hassfest core schema 会移除 `.` 开头的内部 anchor key。自定义集成 schema 顶层只接受 slug 服务名，不建议写 `.` 开头的辅助 key。
- `target` 不支持写 device filter；hassfest 会报错并提示“Services do not support device filters on target, use a device selector instead”。如果服务需要选择设备，应在 `fields` 里放一个 `device` selector。
- `services.yaml` 只描述 UI；Python 端仍然要注册服务、定义 schema、处理权限和实际执行逻辑。
- 服务名称、描述、字段名、section 名和服务图标应与 `translations/en.json`、`icons.json` 对齐。core 集成强制要求服务 icon；自定义集成不一定强制，但写上体验更好。

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

## 10. 如何实现自己的集成

### 10.1 最小集成结构

```
custom_components/my_integration/
├── manifest.json
├── __init__.py
├── config_flow.py              # 支持 UI 配置流时需要
├── sensor.py                   # 向 sensor 域提供实体（可选）
├── translations/
│   └── en.json                 # 自定义集成运行时翻译（config/options/services/entity 等）
├── icons.json                  # 前端图标资源（可选）
└── services.yaml               # 服务定义（可选）
```

### 10.2 manifest.json

```json
{
  "domain": "my_integration",
  "name": "My Integration",
  "version": "1.0.0",
  "codeowners": ["@your_github"],
  "config_flow": true,
  "documentation": "https://www.example.com",
  "issue_tracker": "https://www.example.com/issues",
  "integration_type": "device",
  "iot_class": "local_polling",
  "requirements": ["my-library==1.0.0"],
  "zeroconf": ["_myservice._tcp.local."]
}
```

`manifest.json` 是集成加载器读取的元数据入口。HA 在 `loader.py` 中解析每个集成目录下的 `manifest.json`，生成 `Integration` 对象；自定义集成还需要声明 `version`，否则会被标记为无效集成。

| 键 | 作用 |
|----|------|
| `domain` | 集成域名，必须等于目录名，例如 `custom_components/my_integration/` 对应 `my_integration`。它也是配置条目、实体平台、服务、翻译 key 的命名空间。 |
| `name` | 前端展示的集成名称；如果翻译资源没有提供根级 `title`，翻译加载器会回退到 manifest 的 `name`。 |
| `version` | 自定义集成版本号。Core 内置集成通常不写；custom integration 应写，便于 HA 校验、诊断和兼容性判断。 |
| `documentation` | 集成文档 URL。内置集成要求指向 Home Assistant 官方集成文档；自定义集成可以指向自己的文档页。 |
| `issue_tracker` | 自定义集成的问题反馈 URL，通常指向 GitHub Issues。 |
| `codeowners` | 维护者列表，常写 GitHub 用户名；用于源码维护、诊断展示和 hassfest 校验。 |
| `integration_type` | 集成类型，默认是 `hub`。常见值包括 `device`、`hub`、`service`、`helper`、`entity`、`system`、`hardware`、`virtual`。它影响质量规则、实体平台预期和图标/翻译 schema。 |
| `iot_class` | IoT 通信模式，例如 `local_push`、`local_polling`、`cloud_push`、`cloud_polling`、`assumed_state`、`calculated`。用于告诉用户集成依赖本地网络、云端还是计算值。 |
| `config_flow` | 是否支持 UI 配置流。为 `true` 时，集成目录必须提供 `config_flow.py`，HA 才会把它列入可通过 UI 添加的集成。 |
| `single_config_entry` | 是否只允许创建一个配置条目。适合全局服务或系统级集成；多设备/多账号集成通常不要开启。 |
| `requirements` | Python 依赖包列表，HA 会按需安装/加载，例如 `aiohue==4.8.1`。运行时不要在代码里手动 `pip install`。 |
| `dependencies` | 强依赖集成。HA 会先加载这些集成；依赖失败通常会阻止当前集成正常 setup。 |
| `after_dependencies` | 弱依赖/加载顺序提示。存在时尽量在这些集成之后加载，但不会强制安装或强制启用。 |
| `loggers` | 集成相关第三方库 logger 名称；用于日志级别管理和诊断。 |
| `import_executor` | 是否在线程池中 import 集成模块。未显式设置时默认 `true`，避免同步 import 阻塞事件循环。 |
| `zeroconf` | mDNS/Zeroconf 发现 matcher。可以是服务类型字符串，也可以是带 `type`、`name`、`properties` 等条件的对象。匹配后进入 `async_step_zeroconf`。 |
| `ssdp` | SSDP/UPnP 发现 matcher。通常按响应头字段匹配；匹配后进入 `async_step_ssdp`。 |
| `dhcp` | DHCP 发现 matcher，可按 `macaddress`、`hostname`、`registered_devices` 等匹配；匹配后进入 `async_step_dhcp`。 |
| `usb` | USB 发现 matcher，可按 `vid`、`pid`、`serial_number`、`manufacturer`、`description` 等匹配；匹配后进入 `async_step_usb`。 |
| `bluetooth` | BLE 广播 matcher，可按 `service_uuid`、`service_data_uuid`、`local_name`、`manufacturer_id`、`manufacturer_data_start`、`connectable` 等匹配；匹配后进入 `async_step_bluetooth`。 |
| `homekit` | HomeKit 发现声明，例如 `models`；用于 HomeKit 控制器发现路径。 |
| `mqtt` | MQTT discovery 相关声明；用于让 MQTT 集成知道哪些 discovery payload 可归属到该集成。 |
| `quality_scale` | 内置集成的质量等级标记；自定义集成在 loader 属性中会归类为 `custom`。 |
| `preview_features` | 预览功能元数据，可为每个 feature 提供反馈、了解更多、报错链接；需要配合翻译资源展示名称和说明。 |
| `disabled` | 禁用原因。一般由 core/维护流程使用，自定义集成通常不主动写。 |
| `is_built_in` / `overwrites_built_in` | loader 运行时附加的内部元数据，不是自定义集成作者应该手写的 manifest 字段。 |

#### 翻译字符串：strings.json 与 translations/en.json

HA 有两套容易混淆的翻译文件入口：**内置 core 集成源码中通常维护 `strings.json`**；**自定义集成运行时应提供 `translations/<language>.json`，至少 `translations/en.json`**。源码中的翻译加载器通过 `Integration.has_translations` 判断集成目录是否有 `translations/` 目录，然后读取 `translations/en.json` 和当前语言 JSON；pylint 辅助工具也明确区分“core integrations use strings.json, custom integrations use translations/en.json”。因此写自定义集成时，最好把下面的结构放在 `translations/en.json`，而不是只放一个根级 `strings.json`。

翻译资源会按 category 拉取，并被扁平化成类似 `component.my_integration.config.step.user.data.host` 的 key。英文永远先作为 fallback 加载，其他语言只覆盖已有 key；占位符如 `{name}`、`{host}` 必须和英文字符串保持一致。

```json
{
  "title": "My Integration",
  "config": {
    "step": {
      "user": {
        "title": "Connect to device",
        "description": "Enter the connection details for {name}.",
        "data": {
          "host": "Host",
          "api_key": "API key"
        },
        "data_description": {
          "host": "Hostname or IP address of the device."
        }
      }
    },
    "error": {
      "cannot_connect": "Failed to connect",
      "invalid_auth": "Invalid authentication"
    },
    "abort": {
      "already_configured": "Device is already configured"
    }
  },
  "options": {
    "step": {
      "init": {
        "data": {
          "poll_interval": "Poll interval"
        }
      }
    }
  },
  "services": {
    "refresh": {
      "name": "Refresh data",
      "description": "Fetch fresh data from the device.",
      "fields": {
        "force": {
          "name": "Force refresh",
          "description": "Ignore cached data."
        }
      }
    }
  },
  "entity": {
    "sensor": {
      "temperature": {
        "name": "Temperature",
        "state_attributes": {
          "signal": {
            "name": "Signal strength"
          }
        }
      }
    }
  }
}
```

| 键 | 作用 |
|----|------|
| `title` | 集成显示标题。通常可以省略，让 HA 使用 manifest 的 `name`；如果要与品牌名不同才单独设置。 |
| `config` | ConfigFlow 的 UI 文案，对应 `config_flow.py` 中的 `async_step_*`、`async_show_form(errors=...)`、`async_abort(reason=...)`、`async_create_entry(...)`。 |
| `config.step.<step_id>.title` | 某个 step 表单标题。`step_id` 必须对应代码里的 `async_show_form(step_id="...")`，例如 `user`、`bluetooth_confirm`、`reauth_confirm`。 |
| `config.step.<step_id>.description` | 表单说明文字，可使用 `description_placeholders` 传入的 `{placeholder}`。 |
| `config.step.<step_id>.data.<field>` | 表单字段标签。`field` 应对应 `data_schema` 中的字段名。 |
| `config.step.<step_id>.data_description.<field>` | 字段帮助说明；必须对应同一 step 的 `data` 字段。 |
| `config.step.<step_id>.sections.<section>` | 表单分区。每个 section 可有 `name`、`description`、`data`、`data_description`，用于复杂表单分组。 |
| `config.step.<step_id>.menu_options` | 菜单式 flow 的选项标题，对应 `async_show_menu` 的 menu option。 |
| `config.step.<step_id>.submit` | 自定义提交按钮文案。 |
| `config.error.<error_key>` | 表单错误文案。代码中 `errors["base"] = "cannot_connect"` 或 `errors[field] = "invalid_auth"` 时，前端查这里。 |
| `config.abort.<reason>` | 终止 flow 的文案。代码中 `async_abort(reason="already_configured")` 的 reason 必须能在这里找到。`_abort_if_unique_id_configured()` 等助手也要求常见 reason 有翻译。 |
| `config.progress.<key>` | 长任务/进度型 flow 的进度文案。 |
| `config.create_entry.<key>` | 成功创建条目时的补充文案。 |
| `config.flow_title` | Flow 标题模板，常和 `title_placeholders` 一起用于发现流，例如显示设备名称。 |
| `config_subentries` | Config subentry flow 文案。结构类似 `config`，但还包含 `entry_type`、`initiate_flow`，用于一个 config entry 下再添加子设备/子配置。 |
| `options` | OptionsFlow 的 UI 文案，结构和 `config` 类似；`options.step.init.data.poll_interval` 对应选项表单字段。 |
| `services` | 服务面板文案。`services.<service>.name/description` 描述服务；`fields.<field>.name/description/example` 描述服务字段；`sections` 描述服务字段分区。 |
| `entity` | 具体实体翻译。结构通常是 `entity.<platform>.<translation_key>`；实体类通过 `_attr_translation_key` 或 description 的 `translation_key` 关联。 |
| `entity.<platform>.<translation_key>.name` | 实体名称翻译。配合 `_attr_has_entity_name = True` 可得到符合 HA 命名规范的实体名称。 |
| `entity.<platform>.<translation_key>.state` | 实体 state 值翻译，例如 sensor 枚举状态。 |
| `entity.<platform>.<translation_key>.state_attributes` | 实体属性名和属性枚举值翻译。常见结构是 `<attr>.name` 和 `<attr>.state.<value>`。 |
| `entity.<platform>.<translation_key>.unit_of_measurement` | 单位文本翻译，适合需要本地化展示的单位。 |
| `entity_component` | 平台级翻译，主要用于 `entity` / `helper` / `system` 型集成自身作为实体平台时的状态、属性等通用文案。 |
| `device` | 设备 translation key 的名称翻译。 |
| `device_automation` | 设备自动化文案，包括 `trigger_type`、`trigger_subtype`、`condition_type`、`action_type`、`extra_fields` 等。 |
| `issues` | Repairs/issue registry 文案。`issues.<issue_key>.title` 是问题标题；`description` 是说明；`fix_flow` 可定义修复流程表单文案。 |
| `selector` | selector 中 choices/options/fields/unit 的本地化文案。 |
| `conditions` / `triggers` | 自定义自动化 condition/trigger 的名称、说明和字段文案。 |
| `exceptions` | 可翻译异常文案。异常类或调用方使用 `translation_key` 和 `translation_placeholders` 时，会查这里。 |
| `preview_features` | manifest `preview_features` 中 feature 的显示名、说明和启停确认文案。 |
| `application_credentials` | OAuth/application credentials 配置说明。 |
| `system_health` / `config_panel` / `conversation` / `common` | 较少见的专用分类，分别用于系统健康信息、自定义面板、对话 agent、集成内复用通用文案。 |

翻译值还可以引用已有 key，避免重复维护：

```json
{
  "config": {
    "error": {
      "cannot_connect": "[%key:common::config_flow::error::cannot_connect%]"
    },
    "step": {
      "user": {
        "data": {
          "host": "[%key:common::config_flow::data::host%]"
        }
      }
    }
  }
}
```

`common::...` 引用 HA 全局通用字符串；`component::<domain>::...` 引用某个集成自己的翻译 key。注意不要拼接多个引用，也不要在翻译字符串中写 HTML；URL 应通过 description placeholders 传入。

#### icons.json

`icons.json` 是前端图标资源，不负责实体状态值或服务说明的文字翻译。源码中的 icon helper 会读取每个集成目录下的 `icons.json`，按 category 建缓存；frontend 通过 `frontend/get_icons` WebSocket 命令按 `conditions`、`entity`、`entity_component`、`services`、`triggers` 等分类获取资源。hassfest 还支持校验 `config`、`options`、`issues.fix_flow` 等表单 section 图标。

```json
{
  "entity": {
    "sensor": {
      "temperature": {
        "default": "mdi:thermometer",
        "state": {
          "unavailable": "mdi:thermometer-alert"
        },
        "state_attributes": {
          "signal": {
            "default": "mdi:wifi",
            "range": {
              "0": "mdi:wifi-strength-outline",
              "50": "mdi:wifi-strength-2",
              "80": "mdi:wifi-strength-4"
            }
          }
        }
      }
    }
  },
  "services": {
    "refresh": {
      "service": "mdi:refresh"
    }
  },
  "triggers": {
    "device_pressed": {
      "trigger": "mdi:gesture-tap-button"
    }
  }
}
```

| 键 | 作用 |
|----|------|
| `entity` | 具体实体图标。结构是 `entity.<platform>.<translation_key>`，通常与翻译文件里的实体 translation key 对齐。 |
| `entity.<platform>.<translation_key>.default` | 该实体 translation key 的默认图标。 |
| `entity.<platform>.<translation_key>.state.<state>` | 按实体 state 值切换图标。state 图标不要和 default 图标重复。 |
| `entity.<platform>.<translation_key>.range.<number>` | 按数值区间切换图标，key 必须是数字并按升序排列。适合电量、信号强度等。 |
| `entity.<platform>.<translation_key>.state_attributes.<attr>.default` | 某个属性的默认图标。 |
| `entity.<platform>.<translation_key>.state_attributes.<attr>.state.<value>` | 某个属性取特定枚举值时的图标。Hue 的 `effect=candle/fire/sunrise` 图标就是这种模式。 |
| `entity.<platform>.<translation_key>.state_attributes.<attr>.range.<number>` | 某个数值属性按区间切换图标。 |
| `entity_component` | 平台级实体图标，主要用于 `entity` / `helper` / `system` 类型集成。需要 `_` default 图标作为兜底。 |
| `services.<service>.service` | 服务图标，对应服务名。自定义集成还可以使用简写：`"refresh": "mdi:refresh"`，加载时会被转换成 `{ "service": "mdi:refresh" }`。 |
| `services.<service>.sections.<section>` | 服务字段分区图标。 |
| `conditions.<condition>.condition` | 自动化 condition 图标。 |
| `triggers.<trigger>.trigger` | 自动化 trigger 图标。 |
| `config.step.<step_id>.sections.<section>` | ConfigFlow 表单 section 图标。 |
| `options.step.<step_id>.sections.<section>` | OptionsFlow 表单 section 图标。 |
| `issues.<issue_key>.fix_flow.step.<step_id>.sections.<section>` | Repair issue 修复流程中表单 section 的图标。 |

所有图标值必须是 Material Design Icons 标识并以 `mdi:` 开头，例如 `mdi:refresh`。文字放在翻译资源里，图标放在 `icons.json`，两者通过相同的 service 名、trigger 名或 entity `translation_key` 对齐。

#### services.yaml

如果集成注册了自己的服务，建议同时提供 `services.yaml`、`translations/en.json` 的 `services` 文案，以及 `icons.json` 的服务图标。下面是一个自定义集成中常见的组合：

```python
# __init__.py
import voluptuous as vol

from homeassistant.core import HomeAssistant, ServiceCall
from homeassistant.helpers import config_validation as cv

DOMAIN = "my_integration"
SERVICE_REFRESH = "refresh"

REFRESH_SCHEMA = vol.Schema({
    vol.Optional("force", default=False): cv.boolean,
    vol.Optional("timeout", default=10): vol.All(vol.Coerce(int), vol.Range(min=1, max=60)),
})


async def async_setup_entry(hass: HomeAssistant, entry) -> bool:
    async def async_refresh(call: ServiceCall) -> None:
        force = call.data["force"]
        timeout = call.data["timeout"]
        await entry.runtime_data.async_refresh(force=force, timeout=timeout)

    hass.services.async_register(
        DOMAIN,
        SERVICE_REFRESH,
        async_refresh,
        schema=REFRESH_SCHEMA,
    )
    return True
```

```yaml
# services.yaml
refresh:
  fields:
    force:
      default: false
      example: true
      selector:
        boolean:
    advanced_options:
      collapsed: true
      fields:
        timeout:
          default: 10
          example: 30
          selector:
            number:
              min: 1
              max: 60
              unit_of_measurement: seconds
```

```json
// translations/en.json
{
  "services": {
    "refresh": {
      "name": "Refresh data",
      "description": "Fetch fresh data from the device.",
      "fields": {
        "force": {
          "name": "Force refresh",
          "description": "Ignore cached data and poll the device immediately."
        },
        "timeout": {
          "name": "Timeout",
          "description": "Maximum time to wait for the refresh request."
        }
      },
      "sections": {
        "advanced_options": {
          "name": "Advanced options"
        }
      }
    }
  }
}
```

```json
// icons.json
{
  "services": {
    "refresh": {
      "service": "mdi:refresh"
    }
  }
}
```

写自己的 `services.yaml` 时，按下面的顺序最不容易出错：

1. **先注册服务**：在 `async_setup` 或 `async_setup_entry` 中调用 `hass.services.async_register(...)`，或在实体平台里调用 `async_register_entity_service(...)`。服务名、字段名先在 Python schema 中确定。
2. **再写描述文件**：顶层 key 写服务名；无参数服务可以只写 `reload:`；有参数则写 `fields`、`selector`、`default`、`example`。
3. **实体服务写 target**：如果服务作用于实体，使用 `target.entity.domain` / `target.entity.integration` 限制可选实体；不要在 `target` 里写 device filter。
4. **复杂字段用 section**：一组高级参数可以写成带 `collapsed: true` 的 section，section 内再放 `fields`。
5. **文案放翻译文件**：自定义集成优先在 `translations/en.json` 的 `services.<service>` 下写 `name`、`description`、`fields`、`sections`，再按需添加其他语言。
6. **图标放 icons.json**：用 `services.<service>.service` 提供 `mdi:` 图标，让服务列表更容易识别。
7. **保持三处一致**：Python schema、`services.yaml`、`translations/en.json` 的字段名必须一致；默认值也应一致，否则 UI 展示和实际调用行为会产生偏差。

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

要启用 UI 配置流，`manifest.json` 中必须声明 `"config_flow": true`，并在集成目录下提供 `config_flow.py`。自定义集成的 `ConfigFlow` 只负责“收集输入、校验、去重、返回 flow result”；它不直接实例化 `ConfigEntry`。

#### 常用 step 设计

- `async_step_user`：用户手动添加入口。展示表单，校验 host/token，连接设备或云端账号，拿到稳定 ID 后创建条目。
- `async_step_import`：YAML 导入入口。通常复用 `async_step_user` 的校验逻辑，但 source 是 `import`，用于把 YAML 迁移成 config entry。
- `async_step_bluetooth` / `zeroconf` / `dhcp` / `ssdp`：自动发现入口。先从发现数据提取稳定 ID 并去重，再展示确认表单或补充认证表单。
- `async_step_reauth`：认证失效后的重新认证入口。成功后更新原条目并 reload，而不是创建新条目。
- `async_step_reconfigure`：用户修改已有条目的核心连接参数。成功后更新原条目。
- `async_get_options_flow`：返回 OptionsFlow，用于管理非身份类选项，例如轮询间隔、启用功能开关。

#### 手动配置 + Zeroconf/YAML 导入示例

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
            try:
                info = await validate_input(self.hass, user_input)
            except CannotConnect:
                errors["base"] = "cannot_connect"
            except InvalidAuth:
                errors["base"] = "invalid_auth"
            else:
                # 使用设备序列号/账号 ID/MAC 等稳定身份；host/IP 不稳定时不要作为 unique_id。
                await self.async_set_unique_id(info["serial_number"])
                self._abort_if_unique_id_configured(
                    updates={CONF_HOST: user_input[CONF_HOST]},
                )
                return self.async_create_entry(
                    title=info["title"],
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

    async def async_step_import(self, import_data):
        """YAML 导入步骤。"""
        return await self.async_step_user(import_data)

    async def async_step_zeroconf(self, discovery_info):
        """Zeroconf 自动发现步骤。"""
        host = discovery_info.host
        serial = discovery_info.properties.get("serial")

        if serial:
            await self.async_set_unique_id(serial)
            self._abort_if_unique_id_configured(updates={CONF_HOST: host})

        self.context["title_placeholders"] = {"name": discovery_info.name}
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
```

#### BLE 自动发现示例

BLE 集成不应假设每个广播包都会进入 `async_step_bluetooth`；HA 会先按 manifest matcher 匹配 domain，再用 Bluetooth matcher 和 discovery flow 机制去重。进入该方法后，仍然必须设置真实 `unique_id` 并检查重复配置。

```python
from homeassistant.components.bluetooth import BluetoothServiceInfoBleak
from homeassistant.const import CONF_ADDRESS, CONF_NAME
from homeassistant.helpers.device_registry import format_mac


class MyBleConfigFlow(config_entries.ConfigFlow, domain=DOMAIN):
    """BLE 设备配置流。"""

    VERSION = 1

    def __init__(self):
        self._discovered: dict[str, str] = {}

    async def async_step_bluetooth(
        self, discovery_info: BluetoothServiceInfoBleak,
    ):
        """处理匹配且未被去重的 BLE 发现事件。"""
        address = discovery_info.address
        await self.async_set_unique_id(format_mac(address))
        self._abort_if_unique_id_configured()

        if not device_advertisement_supported(discovery_info):
            return self.async_abort(reason="not_supported")

        self._discovered = {
            CONF_ADDRESS: address,
            CONF_NAME: discovery_info.name or address,
        }
        self.context["title_placeholders"] = {
            "name": self._discovered[CONF_NAME],
        }
        self._set_confirm_only()
        return self.async_show_form(step_id="bluetooth_confirm")

    async def async_step_bluetooth_confirm(self, user_input=None):
        """用户确认添加发现到的 BLE 设备。"""
        if user_input is not None:
            return self.async_create_entry(
                title=self._discovered[CONF_NAME],
                data={CONF_ADDRESS: self._discovered[CONF_ADDRESS]},
            )

        return self.async_show_form(
            step_id="bluetooth_confirm",
            description_placeholders=self.context["title_placeholders"],
        )
```

#### OptionsFlow 示例

```python
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

### 10.5 `sensor.py` — 向 sensor 域提供实体（CoordinatorEntity 模式）

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

### 10.6 从 YAML 配置加载集成

前面展示的集成仅支持 ConfigFlow（UI 配置），但自定义集成也可以支持从 `configuration.yaml` 加载。这种模式下，用户在 YAML 中声明集成配置，HA 解析后调用集成的 `async_setup` 函数。

#### 10.6.1 两种加载路径对比

| 特性 | ConfigFlow 路径 | YAML 路径 |
|------|-----------------|-----------|
| 入口函数 | `async_setup_entry(hass, entry)` | `async_setup(hass, config)` |
| 配置来源 | UI 交互创建的 ConfigEntry | `configuration.yaml` 中的 YAML 配置 |
| 平台设置方式 | `async_forward_entry_setups(entry, platforms)` | 通过 `EntityComponent.async_setup(config)` 自动遍历 |
| 卸载函数 | `async_unload_entry(hass, entry)` | 无（HA 停止时自动清理） |
| 运行时数据 | `entry.runtime_data` | `hass.data[DOMAIN]` |
| 配置校验 | ConfigFlow 中的 Schema | `CONFIG_SCHEMA` / `PLATFORM_SCHEMA` |

#### 10.6.2 仅 YAML 配置的集成

最简单的方式 — 集成只从 YAML 加载，不需要 ConfigFlow：

```python
# __init__.py
import voluptuous as vol
from homeassistant.core import HomeAssistant
from homeassistant.const import Platform
import homeassistant.helpers.config_validation as cv

DOMAIN = "my_integration"
PLATFORMS = [Platform.SENSOR]

# 定义 YAML 配置 Schema
CONFIG_SCHEMA = vol.Schema(
    {
        DOMAIN: vol.Schema(
            {
                vol.Required("host"): str,
                vol.Required("api_key"): cv.string,
                vol.Optional("poll_interval", default=30): int,
            }
        )
    },
    extra=vol.ALLOW_EXTRA,
)


async def async_setup(hass: HomeAssistant, config: dict) -> bool:
    """从 YAML 配置设置集成。"""
    conf = config[DOMAIN]

    # 存储配置到 hass.data，供平台文件使用
    hass.data[DOMAIN] = {
        "host": conf["host"],
        "api_key": conf["api_key"],
        "poll_interval": conf["poll_interval"],
    }

    # 创建 EntityComponent 并设置平台
    component = EntityComponent(_LOGGER, DOMAIN, hass)
    await component.async_setup(config)

    return True
```

对应的平台文件（`sensor.py`）需要实现 `async_setup_platform`（传统方式）：

```python
# sensor.py
from homeassistant.components.sensor import SensorEntity
from homeassistant.core import HomeAssistant


async def async_setup_platform(
    hass: HomeAssistant,
    config: dict,            # 平台级 YAML 配置（来自 configuration.yaml 中 my_integration: 下的 platform: sensor 块）
    async_add_entities,      # 添加实体的回调
    discovery_info=None,     # 发现信息（如果有）
) -> None:
    """从 YAML 配置设置传感器平台。"""
    data = hass.data[DOMAIN]  # 获取集成级配置

    async_add_entities([
        MySensor(data["host"], data["api_key"], "temperature"),
        MySensor(data["host"], data["api_key"], "humidity"),
    ])


class MySensor(SensorEntity):
    """YAML 模式的传感器。"""

    def __init__(self, host, api_key, sensor_type):
        self._host = host
        self._api_key = api_key
        self._sensor_type = sensor_type
        self._attr_unique_id = f"{host}_{sensor_type}"
        self._attr_name = f"My {sensor_type}"
        self._native_value = None

    @property
    def native_value(self):
        return self._native_value

    async def async_update(self):
        """EntityPlatform 轮询时调用此方法。"""
        self._native_value = await fetch_value(self._host, self._api_key, self._sensor_type)
```

对应的 `configuration.yaml`：

```yaml
my_integration:
  host: "192.168.1.100"
  api_key: "abc123"
  poll_interval: 30
```

#### 10.6.3 同时支持 YAML 和 ConfigFlow

许多内置集成同时支持两种加载方式。通常的做法是：

- YAML 配置通过 `async_setup` 处理，并在其中将配置导入为 ConfigEntry
- ConfigFlow 直接创建 ConfigEntry
- 核心逻辑统一由 `async_setup_entry` 处理

```python
# __init__.py
import voluptuous as vol
from homeassistant.config_entries import ConfigEntry
from homeassistant.core import HomeAssistant
from homeassistant.const import Platform
import homeassistant.helpers.config_validation as cv

DOMAIN = "my_integration"
PLATFORMS = [Platform.SENSOR]

# YAML 配置 Schema — 使用 config_entry_only_config_schema 可声明仅支持 Config Entry
# CONFIG_SCHEMA = cv.config_entry_only_config_schema(DOMAIN)  # 仅 ConfigFlow
#
# 同时支持 YAML 和 ConfigFlow 时：
CONFIG_SCHEMA = vol.Schema(
    {
        DOMAIN: vol.Schema(
            {
                vol.Required("host"): str,
                vol.Required("api_key"): cv.string,
            }
        )
    },
    extra=vol.ALLOW_EXTRA,
)


async def async_setup(hass: HomeAssistant, config: dict) -> bool:
    """从 YAML 配置设置集成，并将配置导入为 ConfigEntry。"""
    if DOMAIN not in config:
        return True  # 没有此集成的 YAML 配置，可能通过 ConfigFlow 配置

    conf = config[DOMAIN]

    # 检查是否已有同配置的 ConfigEntry（避免重复导入）
    for entry in hass.config_entries.async_entries(DOMAIN):
        if entry.source == SOURCE_IMPORT and entry.data.get("host") == conf["host"]:
            return True  # 已导入过

    # 通过 ConfigFlow 导入 YAML 配置
    # 这是 HA 生产代码中 YAML → ConfigEntry 导入的标准模式
    # 参见：sleepiq, yeelight, lutron_caseta, sun, thread 等集成
    hass.async_create_task(
        hass.config_entries.flow.async_init(
            DOMAIN,
            context={"source": SOURCE_IMPORT},
            data=conf,
        )
    )

    return True


async def async_setup_entry(hass: HomeAssistant, entry: ConfigEntry) -> bool:
    """从 ConfigEntry 设置集成（YAML 导入和 ConfigFlow 统一走此路径）。"""
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

    return True


async def async_unload_entry(hass: HomeAssistant, entry: ConfigEntry) -> bool:
    """卸载 ConfigEntry。"""
    unload_ok = await hass.config_entries.async_unload_platforms(entry, PLATFORMS)
    if unload_ok:
        await entry.runtime_data.async_disconnect()
    return unload_ok
```

对应的 ConfigFlow 中增加 import 步骤：

```python
# config_flow.py
class MyConfigFlow(config_entries.ConfigFlow, domain=DOMAIN):
    VERSION = 1

    async def async_step_import(self, import_data):
        """从 YAML 配置导入。"""
        host = import_data["host"]

        await self.async_set_unique_id(host)
        self._abort_if_unique_id_configured()

        return self.async_create_entry(
            title=f"My Device ({host})",
            data=import_data,
        )

    async def async_step_user(self, user_input=None):
        # ... 用户手动配置步骤（同 10.4 节）
```

同时，平台文件需要同时实现 `async_setup_platform`（YAML 路径）和 `async_setup_entry`（ConfigEntry 路径）：

```python
# sensor.py
from homeassistant.components.sensor import SensorEntity
from homeassistant.config_entries import ConfigEntry
from homeassistant.core import HomeAssistant
from homeassistant.helpers.entity_platform import AddEntitiesCallback


async def async_setup_entry(
    hass: HomeAssistant,
    config_entry: ConfigEntry,
    async_add_entities: AddEntitiesCallback,
) -> None:
    """ConfigEntry 路径设置传感器。"""
    runtime_data = config_entry.runtime_data
    async_add_entities([
        MySensor(runtime_data, "temperature"),
        MySensor(runtime_data, "humidity"),
    ])


async def async_setup_platform(
    hass: HomeAssistant,
    config: dict,
    async_add_entities: AddEntitiesCallback,
    discovery_info=None,
) -> None:
    """YAML 路径设置传感器（通常不再需要，因为 YAML 会导入为 ConfigEntry）。"""
    # 如果使用了 YAML → ConfigEntry 导入模式，此函数通常不会被执行
    # 但保留兼容性时，可以：
    data = hass.data[DOMAIN]
    async_add_entities([
        MySensorLegacy(data["host"], data["api_key"], "temperature"),
    ])
```

#### 10.6.4 平台级 YAML 配置（PLATFORM_SCHEMA）

某些集成（特别是 hub 型集成如 `mqtt`、`hue`）支持在 YAML 中为每个平台分别配置：

```yaml
# configuration.yaml
light:
  - platform: my_integration
    host: "192.168.1.100"
    name: "Living Room Light"

sensor:
  - platform: my_integration
    host: "192.168.1.100"
    sensor_type: "temperature"
```

此时平台文件需要定义 `PLATFORM_SCHEMA`：

```python
# sensor.py
import voluptuous as vol
from homeassistant.components.sensor import PLATFORM_SCHEMA, SensorEntity
from homeassistant.core import HomeAssistant

PLATFORM_SCHEMA = vol.Schema(
    {
        vol.Required("host"): str,
        vol.Required("sensor_type"): str,
        vol.Optional("name"): str,
    },
    extra=vol.ALLOW_EXTRA,
)


async def async_setup_platform(
    hass: HomeAssistant,
    config: dict,            # 已经过 PLATFORM_SCHEMA 校验的平台配置
    async_add_entities,
    discovery_info=None,
) -> None:
    host = config["host"]
    sensor_type = config["sensor_type"]
    name = config.get("name", f"My {sensor_type}")

    async_add_entities([MySensor(host, sensor_type, name)])
```

**注意**：`PLATFORM_SCHEMA` 在 `sensor.py` 等平台文件中定义，而 `CONFIG_SCHEMA` 在 `__init__.py` 中定义。两者的区别：

| Schema | 定义位置 | 校验范围 | 用途 |
|--------|----------|----------|------|
| `CONFIG_SCHEMA` | `__init__.py` | 整个集成的 YAML 配置块 | 集成级配置（host, api_key 等） |
| `PLATFORM_SCHEMA` | 平台文件（`sensor.py` 等） | 每个平台条目的 YAML 配置 | 平台级配置（sensor_type, name 等） |

#### 10.6.5 YAML 加载与 ConfigFlow 加载的内部流程差异

```
YAML 路径:
  bootstrap._async_set_up_integrations()
    → 解析 configuration.yaml
    → async_setup_component("my_integration", config)
      → _async_setup_component()
        → integration.async_get_component()  导入 __init__.py
        → conf_util.async_process_component_config()  用 CONFIG_SCHEMA 校验
        → component.async_setup(hass, processed_config)  ← 调用集成的 async_setup
          → EntityComponent.async_setup(config)
            → config_per_platform() 遍历各平台配置
            → async_setup_platform("sensor", platform_config)
              → 导入 sensor.py
              → EntityPlatform.async_setup(platform_config)
                → platform.async_setup_platform(hass, config, async_add_entities)

ConfigFlow 路径:
  用户在 UI 中添加集成
    → ConfigEntriesFlowManager.async_init()
      → ConfigFlow → async_create_entry()
        → ConfigEntry 创建
    → _async_setup_component() 阶段 10
      → entry.async_setup_locked(hass)
        → component.async_setup_entry(hass, entry)  ← 调用集成的 async_setup_entry
          → async_forward_entry_setups(entry, ["sensor"])
            → EntityComponent.async_setup_entry(entry)
              → 导入 sensor.py
              → EntityPlatform.async_setup_entry(config_entry)
                → platform.async_setup_entry(hass, config_entry, async_add_entities)
```

> **易混淆概念**：`discovery.async_load_platform` vs `hass.config_entries.flow.async_init`
>
> | | `hass.config_entries.flow.async_init` | `discovery.async_load_platform` |
> |---|---|---|
> | **用途** | 启动 ConfigFlow，创建 ConfigEntry | 平台发现：动态将平台加载到另一个域 |
> | **YAML 导入** | ✅ 标准做法（66+ 个内置集成使用） | ❌ 不用于 YAML 导入 |
> | **典型场景** | YAML → ConfigEntry 转换、UI 配置 | 集成动态加载自己的 notify 平台等 |
> | **是否创建 ConfigEntry** | 是 | 否，完全绕过 ConfigEntry |
> | **源码实例** | `sleepiq`、`yeelight`、`sun`、`thread` | `html5`（notify）、`template`（多平台） |
>
> `async_load_platform` 的工作原理：发送 `EVENT_LOAD_PLATFORM` dispatcher 信号 → `EntityComponent` 监听到信号 → 调用 `_async_component_platform_discovered` → 最终调用 `async_setup_platform`。它是一个**平台发现机制**，与 ConfigEntry 无关。

#### 10.6.6 当前推荐做法

Home Assistant 社区的趋势是**优先使用 ConfigFlow**，YAML 配置正逐步被淘汰。对于新集成：

1. **推荐**：仅支持 ConfigFlow，使用 `cv.config_entry_only_config_schema(DOMAIN)` 声明不支持 YAML
2. **兼容**：同时支持 YAML 和 ConfigFlow，YAML 配置通过 `async_step_import` 导入为 ConfigEntry
3. **不推荐**：仅支持 YAML 配置（新集成不应采用）

```python
# 仅支持 ConfigFlow（推荐）
CONFIG_SCHEMA = cv.config_entry_only_config_schema(DOMAIN)

# 不支持 YAML 配置（manifest.json 中设置 "config_flow": true）
```

### 10.7 不使用 Coordinator 的 SensorEntity 实现模式

前面 10.5 节展示了最常见的 `CoordinatorEntity + SensorEntity` 模式——通过 `DataUpdateCoordinator` 定时拉取数据，实体自动跟随刷新。然而，并非所有传感器都需要 Coordinator。许多内置集成只继承 `SensorEntity` 本身，根据数据来源的不同，采用两种截然不同的更新策略：

| 策略 | 适用场景 | 关键机制 | 典型集成 |
|------|----------|----------|----------|
| **轮询模式** | 本地可定时请求的数据源 | `async_update()` + `should_poll=True`（默认） | `moon` |
| **推送模式** | 数据由外部事件推送或本地计算 | `should_poll=False` + `async_write_ha_state()` | `sun` |

#### 10.7.1 轮询模式 — 纯 SensorEntity + async_update

当传感器需要定时从某个数据源拉取数据，且数据量较小、更新逻辑简单时，可以直接继承 `SensorEntity`，通过重写 `async_update()` 方法实现轮询更新。这种方式不需要 `DataUpdateCoordinator`，EntityPlatform 的轮询定时器会定期调用 `async_update()`，而 `DataUpdateCoordinator` 提供的错误重试、多实体共享刷新等能力在这种简单场景下并不必要。

**源码实例：Moon 集成** (`homeassistant/components/moon/`)

Moon 集成追踪月相，数据来源是纯计算（`astral.moon.phase()`），无需外部 API，一个传感器就足够。

**`manifest.json`** — 注意 `iot_class` 为 `calculated`（计算型，不是 `local_polling`）：

```json
{
  "domain": "moon",
  "name": "Moon",
  "config_flow": true,
  "integration_type": "service",
  "iot_class": "calculated",
  "single_config_entry": true
}
```

**`__init__.py`** — 极简入口，只转发平台设置：

```python
from homeassistant.config_entries import ConfigEntry
from homeassistant.core import HomeAssistant
from .const import PLATFORMS

async def async_setup_entry(hass: HomeAssistant, entry: ConfigEntry) -> bool:
    await hass.config_entries.async_forward_entry_setups(entry, PLATFORMS)
    return True

async def async_unload_entry(hass: HomeAssistant, entry: ConfigEntry) -> bool:
    return await hass.config_entries.async_unload_platforms(entry, PLATFORMS)
```

**`config_flow.py`** — 极简配置流，不需要任何配置参数：

```python
from homeassistant.config_entries import ConfigFlow, ConfigFlowResult
from .const import DEFAULT_NAME, DOMAIN

class MoonConfigFlow(ConfigFlow, domain=DOMAIN):
    VERSION = 1

    async def async_step_user(self, user_input=None) -> ConfigFlowResult:
        if user_input is not None:
            return self.async_create_entry(title=DEFAULT_NAME, data={})
        return self.async_show_form(step_id="user")
```

**`sensor.py`** — 核心实现，只继承 `SensorEntity`，不使用 `CoordinatorEntity`：

```python
from astral import moon
from homeassistant.components.sensor import SensorDeviceClass, SensorEntity
from homeassistant.config_entries import ConfigEntry
from homeassistant.core import HomeAssistant
from homeassistant.helpers.device_registry import DeviceEntryType, DeviceInfo
from homeassistant.helpers.entity_platform import AddConfigEntryEntitiesCallback
from homeassistant.util import dt as dt_util
from .const import DOMAIN

# 月相枚举值
STATE_FIRST_QUARTER = "first_quarter"
STATE_FULL_MOON = "full_moon"
STATE_LAST_QUARTER = "last_quarter"
STATE_NEW_MOON = "new_moon"
STATE_WANING_CRESCENT = "waning_crescent"
STATE_WANING_GIBBOUS = "waning_gibbous"
STATE_WAXING_CRESCENT = "waxing_crescent"
STATE_WAXING_GIBBOUS = "waxing_gibbous"


async def async_setup_entry(
    hass: HomeAssistant,
    entry: ConfigEntry,
    async_add_entities: AddConfigEntryEntitiesCallback,
) -> None:
    async_add_entities([MoonSensorEntity(entry)], True)


class MoonSensorEntity(SensorEntity):
    """Representation of a Moon sensor."""

    _attr_has_entity_name = True
    _attr_device_class = SensorDeviceClass.ENUM
    _attr_options = [
        STATE_NEW_MOON,
        STATE_WAXING_CRESCENT,
        STATE_FIRST_QUARTER,
        STATE_WAXING_GIBBOUS,
        STATE_FULL_MOON,
        STATE_WANING_GIBBOUS,
        STATE_LAST_QUARTER,
        STATE_WANING_CRESCENT,
    ]
    _attr_translation_key = "phase"

    def __init__(self, entry: ConfigEntry) -> None:
        self._attr_unique_id = entry.entry_id
        self._attr_device_info = DeviceInfo(
            name="Moon",
            identifiers={(DOMAIN, entry.entry_id)},
            entry_type=DeviceEntryType.SERVICE,
        )

    async def async_update(self) -> None:
        """Get the time and updates the states."""
        today = dt_util.now().date()
        state = moon.phase(today)

        if state < 0.5 or state > 27.5:
            self._attr_native_value = STATE_NEW_MOON
        elif state < 6.5:
            self._attr_native_value = STATE_WAXING_CRESCENT
        elif state < 7.5:
            self._attr_native_value = STATE_FIRST_QUARTER
        elif state < 13.5:
            self._attr_native_value = STATE_WAXING_GIBBOUS
        elif state < 14.5:
            self._attr_native_value = STATE_FULL_MOON
        elif state < 20.5:
            self._attr_native_value = STATE_WANING_GIBBOUS
        elif state < 21.5:
            self._attr_native_value = STATE_LAST_QUARTER
        else:
            self._attr_native_value = STATE_WANING_CRESCENT
```

**关键要点**：

1. **`should_poll` 默认为 `True`**：`SensorEntity` 默认就是轮询模式，不需要显式设置。EntityPlatform 会按 `scan_interval` 定时调用 `async_update()`。

2. **`async_update()` 方法**：这是轮询模式下 Entity 获取数据的入口。每次轮询时，EntityPlatform 先调用 `async_device_update()` → `async_update()`，然后自动调用 `async_write_ha_state()` 将更新写入状态机。因此 `async_update()` 中只需要更新 `_attr_*` 属性，不需要手动调用 `async_write_ha_state()`。

3. **`_attr_*` 类属性**：Moon 传感器大量使用 `_attr_*` 设置静态属性（`_attr_has_entity_name`、`_attr_device_class`、`_attr_options`、`_attr_translation_key`），在 `async_update()` 中通过 `self._attr_native_value = ...` 动态更新值。这是 HA 推荐的写法——避免定义过多的 property。

4. **`async_add_entities([MoonSensorEntity(entry)], True)`**：第二个参数 `True` 表示 `update_before_add`，即添加实体前先调用一次 `async_update()`，确保实体有初始值。

**何时选择此模式而非 CoordinatorEntity**：

- 数据源是纯计算或本地文件读取（无网络请求）
- 只有一个或少数几个实体，不需要共享刷新逻辑
- 不需要 Coordinator 提供的错误重试、刷新状态追踪等能力
- 更新逻辑简单，不需要在多个实体间共享同一数据快照

#### 10.7.2 推送模式 — SensorEntity + dispatcher + async_write_ha_state

当数据由外部事件推送（如 MQTT 消息、WebSocket 通知、定时事件变化）而非需要主动轮询时，应采用推送模式。核心思路是：`should_poll = False`（不轮询）+ 在事件回调中更新属性并调用 `async_write_ha_state()` 主动推送状态变更。

**源码实例：Sun 集成** (`homeassistant/components/sun/`)

Sun 集成的传感器追踪太阳位置（方位角、仰角）和下一个日出/日落时间。太阳位置数据由 Sun 实体自行计算，根据太阳相位以不同间隔更新，通过 dispatcher 信号通知传感器刷新。

**`__init__.py`** — 入口创建 Sun 实体并注册为 runtime_data：

```python
from homeassistant.config_entries import SOURCE_IMPORT
from homeassistant.const import Platform
from homeassistant.core import HomeAssistant
from homeassistant.helpers import config_validation as cv
from homeassistant.helpers.entity_component import EntityComponent
from .entity import Sun, SunConfigEntry

PLATFORMS = [Platform.BINARY_SENSOR, Platform.SENSOR]
CONFIG_SCHEMA = cv.empty_config_schema(DOMAIN)

async def async_setup(hass: HomeAssistant, config) -> bool:
    if not hass.config_entries.async_entries(DOMAIN):
        hass.async_create_task(
            hass.config_entries.flow.async_init(
                DOMAIN, context={"source": SOURCE_IMPORT}, data=config,
            )
        )
    return True

async def async_setup_entry(hass: HomeAssistant, entry: SunConfigEntry) -> bool:
    sun = Sun(hass)
    component = EntityComponent[Sun](_LOGGER, DOMAIN, hass)
    await component.async_add_entities([sun])
    entry.runtime_data = sun
    entry.async_on_unload(sun.remove_listeners)
    await hass.config_entries.async_forward_entry_setups(entry, PLATFORMS)
    return True

async def async_unload_entry(hass: HomeAssistant, entry: SunConfigEntry) -> bool:
    if unload_ok := await hass.config_entries.async_unload_platforms(entry, PLATFORMS):
        await entry.runtime_data.async_remove()
    return unload_ok
```

**`entity.py`** — Sun 实体自行计算太阳位置，并通过 dispatcher 信号推送更新：

```python
from homeassistant.helpers.dispatcher import async_dispatcher_send

SIGNAL_POSITION_CHANGED = "sun_position_changed"
SIGNAL_EVENTS_CHANGED = "sun_events_changed"

class Sun(Entity):
    """Representation of the Sun — 核心实体，自行计算并推送。"""

    _attr_name = "Sun"
    entity_id = "sun.sun"

    # ... 初始化、位置计算等方法 ...

    @callback
    def update_events(self, now=None):
        """计算下一个日出/日落等事件，完成后发送信号。"""
        # ... 计算逻辑 ...
        async_dispatcher_send(self.hass, SIGNAL_EVENTS_CHANGED)  # ← 通知传感器

        # 设置定时器，在下一个事件时间再次调用 update_events
        self._update_events_listener = event.async_track_point_in_utc_time(
            self.hass, self.update_events, self._next_change
        )

    @callback
    def update_sun_position(self, now=None):
        """计算当前太阳方位角和仰角，完成后发送信号。"""
        self.solar_azimuth = round(
            self.location.solar_azimuth(utc_point_in_time, self.elevation), 2
        )
        self.solar_elevation = round(
            self.location.solar_elevation(utc_point_in_time, self.elevation), 2
        )
        self.async_write_ha_state()  # ← Sun 实体自身也推送状态
        async_dispatcher_send(self.hass, SIGNAL_POSITION_CHANGED)  # ← 通知传感器

        # 根据太阳相位设置不同间隔的定时器
        delta = _PHASE_UPDATES[self.phase]
        self._update_sun_position_listener = event.async_track_point_in_utc_time(
            self.hass, self.update_sun_position, utc_point_in_time + delta
        )
```

**`sensor.py`** — 传感器只继承 SensorEntity，通过 dispatcher 监听信号，收到信号时调用 `async_write_ha_state()`：

```python
from dataclasses import dataclass
from homeassistant.components.sensor import (
    SensorDeviceClass, SensorEntity, SensorEntityDescription, SensorStateClass,
)
from homeassistant.const import DEGREE, EntityCategory
from homeassistant.core import HomeAssistant
from homeassistant.helpers.dispatcher import async_dispatcher_connect
from homeassistant.helpers.entity_platform import AddConfigEntryEntitiesCallback
from .const import DOMAIN, SIGNAL_EVENTS_CHANGED, SIGNAL_POSITION_CHANGED
from .entity import Sun, SunConfigEntry

# 自定义 EntityDescription，增加 value_fn 和 signal 字段
@dataclass(kw_only=True, frozen=True)
class SunSensorEntityDescription(SensorEntityDescription):
    """Describes a Sun sensor entity."""
    value_fn: Callable[[Sun], StateType | datetime]  # ← 从 Sun 实体获取值的函数
    signal: str                                      # ← 监听的 dispatcher 信号名

# 定义所有传感器的描述
SENSOR_TYPES: tuple[SunSensorEntityDescription, ...] = (
    SunSensorEntityDescription(
        key="next_rising",
        device_class=SensorDeviceClass.TIMESTAMP,
        translation_key="next_rising",
        value_fn=lambda data: data.next_rising,
        signal=SIGNAL_EVENTS_CHANGED,
    ),
    SunSensorEntityDescription(
        key="solar_elevation",
        state_class=SensorStateClass.MEASUREMENT,
        translation_key="solar_elevation",
        value_fn=lambda data: data.solar_elevation,
        native_unit_of_measurement=DEGREE,
        signal=SIGNAL_POSITION_CHANGED,
        entity_registry_enabled_default=False,
    ),
    # ... 更多传感器描述 ...
)


async def async_setup_entry(
    hass: HomeAssistant, entry: SunConfigEntry,
    async_add_entities: AddConfigEntryEntitiesCallback,
) -> None:
    sun = entry.runtime_data
    async_add_entities(
        [SunSensor(sun, description, entry.entry_id) for description in SENSOR_TYPES]
    )


class SunSensor(SensorEntity):
    """Representation of a Sun Sensor — 纯推送模式。"""

    _attr_has_entity_name = True
    _attr_should_poll = False          # ← 关键：不轮询！
    _attr_entity_category = EntityCategory.DIAGNOSTIC
    entity_description: SunSensorEntityDescription

    def __init__(self, sun, entity_description, entry_id) -> None:
        self.entity_description = entity_description
        self._attr_unique_id = f"{entry_id}-{entity_description.key}"
        self.sun = sun

    @property
    def native_value(self):
        """Return value of sensor — 从 Sun 实体直接读取计算好的值。"""
        return self.entity_description.value_fn(self.sun)

    async def async_added_to_hass(self) -> None:
        """Register signal listener when added to hass."""
        await super().async_added_to_hass()
        # ← 关键：监听 dispatcher 信号，收到信号时调用 async_write_ha_state()
        self.async_on_remove(
            async_dispatcher_connect(
                self.hass,
                self.entity_description.signal,
                self.async_write_ha_state,
            )
        )
```

**关键要点**：

1. **`_attr_should_poll = False`**：这是推送模式的核心标志。设置后，EntityPlatform 不会定时调用 `async_update()`，实体必须自己负责推送状态更新。

2. **`async_dispatcher_connect`**：在 `async_added_to_hass()` 中订阅 dispatcher 信号。当 Sun 实体计算完新数据后，发送 `SIGNAL_POSITION_CHANGED` 或 `SIGNAL_EVENTS_CHANGED` 信号，所有订阅该信号的 SunSensor 实体收到通知后调用 `async_write_ha_state()`，将最新的 `native_value`（通过 `value_fn(self.sun)` 从 Sun 实体获取）写入状态机。

3. **`value_fn` + EntityDescription 模式**：Sun 集成巧妙地在 `SensorEntityDescription` 中增加了 `value_fn` 字段，让每个传感器描述自带一个从 Sun 实体提取值的函数。这样一来，8 个传感器只需一个 `SunSensor` 类，通过不同的 `entity_description` 配置即可——避免了 8 个子类的冗余。

4. **`async_on_remove`**：在 `async_added_to_hass()` 中用 `async_on_remove()` 包裹信号订阅的取消逻辑，确保实体被移除时自动取消订阅，不会造成内存泄漏。

5. **`native_value` property**：推送模式中，`native_value` 不从 Coordinator 的 `self.coordinator.data` 中取值，而是直接从运行时对象（`self.sun`）读取。因为运行时对象已经通过定时器更新了数据，传感器只需在收到信号时反映最新值即可。

**推送模式的其他常见实现方式**：

dispatcher 信号是 HA 内部通信的一种方式，推送模式还有其他常见的事件监听手段：

| 监听方式 | 适用场景 | 注册方法 | 清理方法 |
|----------|----------|----------|----------|
| **dispatcher 信号** | 同一集成内部通信 | `async_dispatcher_connect(hass, signal, callback)` | `async_on_remove()` 包裹 |
| **EventBus 事件** | 监听 HA 全局事件（如状态变化） | `hass.bus.async_listen(event_type, callback)` | `async_on_remove()` 包裹 |
| **state_change 事件** | 监听其他实体状态变化 | `async_track_state_change_event(hass, [entity_ids], callback)` | `async_on_remove()` 包裹 |
| **MQTT 订阅** | 接收 MQTT 消息推送 | `mqtt_subscription.async_subscribe_topics(hass, ...)` | `async_on_remove()` 包裹 |
| **WebSocket 回调** | 接收设备推送数据 | 在 API 客户端中注册回调 | 连接关闭时自动清理 |

所有监听方式的共同模式：在 `async_added_to_hass()` 中注册，在 `async_on_remove()` 或 `async_will_remove_from_hass()` 中取消。回调中更新 `_attr_*` 属性，然后调用 `async_write_ha_state()`。

**何时选择推送模式而非 CoordinatorEntity**：

- 数据源是事件驱动（设备主动推送状态、MQTT 消息、定时事件变化）
- 实体可以从已有的运行时对象中直接读取值，不需要独立的拉取逻辑
- 需要更细粒度的更新控制（如 Sun 根据不同太阳相位以不同间隔更新）
- 一个数据源驱动多个传感器，但数据已经在别处计算好（如 Sun 实体）

### 10.8 关键实现要点

1. **runtime_data 模式**：使用 `ConfigEntry[MyData]` 泛型，在 `async_setup_entry` 中设置 `entry.runtime_data`，平台通过 `config_entry.runtime_data` 获取，类型安全。

2. **ConfigEntryNotReady**：连接失败时抛出此异常，HA 会自动重试设置。

3. **async_forward_entry_setups**：在集成入口的 `async_setup_entry` 中调用，将配置条目转发到各平台。

4. **async_unload_platforms**：在 `async_unload_entry` 中调用，卸载所有平台。

5. **async_on_unload**：注册卸载时的清理回调，如取消订阅、关闭连接。

6. **DataUpdateCoordinator**：轮询式更新的推荐方式，自动处理轮询间隔、错误重试、状态更新。适合多个实体共享同一数据源的场景。

7. **CoordinatorEntity**：与 Coordinator 配合的实体基类，自动在 Coordinator 刷新时更新状态。适合需要共享刷新逻辑的轮询式集成。

8. **纯 SensorEntity 模式**：简单传感器可以直接继承 `SensorEntity`，通过 `async_update()` 实现轮询（如 Moon），或通过 `should_poll=False` + `async_write_ha_state()` 实现推送（如 Sun）。不需要 Coordinator 时不应强加 Coordinator。

9. **EntityDescription**：将实体属性从子类移到描述对象，支持一个实体类多个实例。配合 `value_fn` 可以进一步简化多传感器场景。

10. **should_poll**：事件驱动集成的实体应设 `_attr_should_poll = False`，通过 `async_write_ha_state()` 主动推送更新；轮询式集成保持默认 `True`，由 EntityPlatform 定时调用 `async_update()`。

11. **YAML 与 ConfigFlow 共存**：通过 `SOURCE_IMPORT` 将 YAML 配置导入为 ConfigEntry，统一由 `async_setup_entry` 处理，避免维护两套逻辑。

---

## 11. 蓝图(Blueprint)机制

蓝图（Blueprint）是 Home Assistant 提供的一种配置复用机制——用户可以将自动化（automation）或脚本（script）的模板定义为一个蓝图，然后多次使用该蓝图，每次只需提供不同的输入参数（如不同的传感器、灯光实体）。蓝图让社区能够分享可复用的自动化模板，降低配置门槛。

### 11.1 蓝图概述与使用方法

#### 什么是蓝图？

蓝图本质上是一个包含 `blueprint:` 元数据头的 YAML 文件，其中定义了：

- **元数据**：蓝图名称、描述、所属域（automation 或 script）、作者、最低 HA 版本要求等
- **输入定义**（`input:`）：蓝图的可配置参数，每个输入可以指定名称、描述、默认值和选择器（selector）
- **配置模板**：触发器、条件、动作（automation）或序列（script）——其中引用输入的地方用 `!input <input_name>` 标记

用户使用蓝图创建自动化/脚本时，只需填写蓝图定义的输入参数，HA 会自动将 `!input` 替换为用户提供的实际值，生成完整的配置。

#### 使用蓝图的流程

1. **导入蓝图**：在 Settings → Automations & Scenes → Blueprints 页面，点击 "Import Blueprint"，输入蓝图 URL（支持社区论坛、GitHub 文件、GitHub Gist、HA 官网、任意 URL）
2. **创建自动化**：选择已导入的蓝图，点击 "Create Automation"，在编辑器中填写输入参数
3. **保存运行**：保存后，自动化使用 `use_blueprint:` 配置，HA 在每次加载时自动替换输入生成完整配置

#### 蓝图文件存储位置

蓝图 YAML 文件存储在配置目录下的 `blueprints/` 子目录中：

```
config/
├── blueprints/
│   ├── automation/                # 自动化蓝图
│   │   ├── homeassistant/         # HA 内置示例（首次启动自动复制）
│   │   │   └── motion_light.yaml
│   │   ├── community_user/        # 从社区论坛导入
│   │   │   └── some_blueprint.yaml
│   │   └── github_user/           # 从 GitHub 导入
│   │       └── another_blueprint.yaml
│   └── script/                    # 脚本蓝图
│       ├── homeassistant/
│       │   └── confirmable_notification.yaml
│       └── ...
```

### 11.2 蓝图 YAML 格式与自定义蓝图

#### 蓝图 YAML 结构

一个自动化蓝图的完整 YAML 格式如下：

```yaml
blueprint:
  name: Motion-activated Light          # 必需：蓝图名称
  description: Turn on a light when motion is detected.  # 可选：描述
  domain: automation                    # 必需：所属域（automation 或 script）
  source_url: https://github.com/...    # 可选：蓝图来源 URL
  author: Home Assistant                # 可选：作者
  homeassistant:                        # 可选：HA 版本要求
    min_version: 2024.1.0
  input:                                # 可选：输入参数定义
    motion_entity:                      # 输入名（与 !input 对应）
      name: Motion Sensor               # 输入的显示名称
      selector:                         # 选择器（控制前端输入控件）
        entity:
          filter:
            - device_class: motion
              domain: binary_sensor
    light_target:
      name: Light
      selector:
        target:
          entity:
            domain: light
    no_motion_wait:
      name: Wait time
      description: Time to leave the light on after last motion.
      default: 120                       # 默认值
      selector:
        number:
          min: 0
          max: 3600
          unit_of_measurement: seconds

# 以下是自动化配置模板，使用 !input 引用输入参数
mode: restart
max_exceeded: silent

triggers:
  - trigger: state
    entity_id: !input motion_entity      # ← 替换为用户提供的传感器实体
    from: "off"
    to: "on"

actions:
  - action: light.turn_on
    target: !input light_target          # ← 替换为用户提供的灯光目标
  - wait_for_trigger:
      trigger: state
      entity_id: !input motion_entity
      from: "on"
      to: "off"
  - delay: !input no_motion_wait        # ← 替换为用户提供的等待时间
  - action: light.turn_off
    target: !input light_target
```

#### `!input` 标记

`!input` 是 YAML 自定义标签，在解析时被转换为 `Input` 对象（`annotatedyaml.Input`）。`Input` 是一个仅包含 `name` 字段的 `dataclass`：

```python
@dataclass(slots=True, frozen=True)
class Input:
    name: str
```

当 YAML Loader 遇到 `!input motion_entity` 时，调用 `Input.from_node(loader, node)`，创建一个 `Input(name="motion_entity")` 对象，暂存于 YAML 数据结构中。这个对象不是最终值——它是一个"占位符"，将在蓝图替换阶段被替换为用户提供的实际值。

#### 输入定义的详细格式

每个输入可以有以下字段：

| 字段 | 必需 | 说明 |
|------|------|------|
| `name` | 可选 | 输入的显示名称（前端表单标签） |
| `description` | 可选 | 输入的详细说明 |
| `default` | 可选 | 默认值（用户未填时使用） |
| `selector` | 可选 | 选择器定义（控制前端输入控件类型和过滤条件） |

**选择器类型**（常用的）：

| 选择器 | 前端控件 | 适用输入类型 |
|--------|----------|-------------|
| `entity` | 实体选择器 | 选择特定实体 |
| `target` | 目标选择器 | 选择实体/设备/区域的组合 |
| `device` | 设备选择器 | 选择特定设备 |
| `number` | 数字滑块/输入 | 选择数值 |
| `text` | 文本输入框 | 输入文本 |
| `boolean` | 开关 | 选择 true/false |
| `select` | 下拉选择 | 选择预设选项 |
| `time` | 时间选择器 | 选择时间 |
| `date` | 日期选择器 | 选择日期 |
| `action` | 动作序列编辑器 | 定义一组动作 |
| `addon` | Add-on 选择器 | 选择 Hass.io Add-on |
| `area` | 区域选择器 | 选择区域 |

#### 输入分组（Input Section）

蓝图支持将输入参数分组显示，通过嵌套的 `input:` 字段实现：

```yaml
blueprint:
  input:
    notification_section:
      name: Notification Settings
      icon: mdi:bell
      description: Configure notification options
      collapsed: true               # 可折叠
      input:
        notify_device:
          name: Device to notify
          selector:
            device:
              filter:
                integration: mobile_app
        message:
          name: Message
          selector:
            text:
    action_section:
      name: Action Settings
      input:
        confirm_action:
          name: Confirmation Action
          default: []
          selector:
            action:
```

分组输入在引用时直接用内部 input 名称（如 `!input notify_device`），而不是用分组名。HA 在校验时会检查所有 `!input` 引用的名称是否都有对应的输入定义，分组内的 key 会被展平到同一层。

#### 使用蓝图创建自动化时的 YAML 格式

用户通过蓝图创建自动化时，生成的 YAML 配置使用 `use_blueprint:` 字段：

```yaml
# automations.yaml
- id: "1681234567"
  alias: "Motion Light - Living Room"
  use_blueprint:
    path: homeassistant/motion_light.yaml   # 蓝图文件路径
    input:
      motion_entity: binary_sensor.living_room_motion   # 用户提供的实际值
      light_target:
        entity_id: light.living_room
      no_motion_wait: 180
```

HA 加载此自动化时，会读取蓝图文件，将所有 `!input` 替换为 `input:` 中的实际值，生成完整的自动化配置。

### 11.3 蓝图源码解析：导入、校验与替换

#### 核心类结构

蓝图机制涉及以下核心类，分布在 `homeassistant/components/blueprint/` 和 `annotatedyaml` 包中：

| 类/文件 | 位置 | 说明 |
|---------|------|------|
| `Blueprint` | `models.py` | 蓝图数据模型，包含元数据和校验逻辑 |
| `BlueprintInputs` | `models.py` | 蓝图输入数据，负责输入校验和替换 |
| `DomainBlueprints` | `models.py` | 域级蓝图管理器，负责加载/存储蓝图文件 |
| `Input` | `annotatedyaml/objects.py` | YAML `!input` 标记的数据类（占位符） |
| `substitute()` | `annotatedyaml/input.py` | 递归替换 YAML 数据中的 `Input` 对象 |
| `extract_inputs()` | `annotatedyaml/input.py` | 递归提取 YAML 数据中所有 `Input` 引用名 |
| `BLUEPRINT_SCHEMA` | `schemas.py` | 蓝图元数据的 Voluptuous 校验 Schema |
| `BLUEPRINT_INSTANCE_FIELDS` | `schemas.py` | `use_blueprint:` 字段的校验 Schema |

#### 蓝图导入流程

**源码**: `homeassistant/components/blueprint/importer.py`

用户在前端点击 "Import Blueprint" 时，通过 WebSocket 调用 `ws_import_blueprint`，触发以下流程：

```
前端输入 URL → websocket_api: blueprint/import
  │
  ├── importer.fetch_blueprint_from_url(hass, url)
  │     │
  │     ├── 尝试 5 种导入函数（按顺序）：
  │     │     1. fetch_blueprint_from_community_post → 解析论坛帖子 JSON，提取 YAML 代码块
  │     │     2. fetch_blueprint_from_github_url → 转换 GitHub URL 为 raw.githubusercontent.com URL，下载 YAML
  │     │     3. fetch_blueprint_from_github_gist_url → 调用 GitHub Gist API，提取 .yaml 文件
  │     │     4. fetch_blueprint_from_website_url → 下载 HA 官网的 YAML 文件
  │     │     5. fetch_blueprint_from_generic_url → 下载任意 URL 的 YAML 文件
  │     │
  │     │     每种函数如果 URL 不匹配其模式，抛出 UnsupportedUrl，被 suppress
  │     │     继续尝试下一种。5 种都不匹配则抛出 "Unsupported URL" 错误
  │     │
  │     ├── yaml_util.parse_yaml(raw_yaml) → 解析 YAML 数据
  │     │     此阶段 !input 标签被解析为 Input 对象（占位符）
  │     │
  │     ├── Blueprint(data, schema=BLUEPRINT_SCHEMA) → 校验蓝图元数据
  │     │     校验: name, domain, input 定义, min_version 等
  │     │     校验: 所有 !input 引用的名称是否在 input 定义中存在
  │     │       → extract_inputs(data) 扫描整个数据，收集所有 Input.name
  │     │       → 与 blueprint.inputs 对比，缺失的抛出 InvalidBlueprint
  │     │
  │     └── 返回 ImportedBlueprint(suggested_filename, raw_data, blueprint)
  │
  ├── blueprint.update_metadata(source_url=url) → 记录来源 URL
  │
  └── 返回给前端: metadata, raw_data, suggested_filename, validation_errors
        前端显示预览 → 用户确认 → websocket_api: blueprint/save
          → DomainBlueprints.async_add_blueprint()
            → 写入 YAML 文件到 config/blueprints/<domain>/<path>
            → 存入内存缓存 _blueprints[path] = blueprint
```

**社区论坛导入的细节**：HA 从论坛帖子中提取 YAML 代码块时，会解析帖子的 HTML 内容，查找 `<code class="lang-yaml">` 或 `<code class="lang-auto">` 标签，对 YAML 代码块进行 HTML 反转义（`html.unescape`）后解析。只接受标记为 YAML 或 auto 语法类型的代码块。

**GitHub URL 转换**：`https://github.com/<repo>/blob/<path>` 被自动转换为 `https://raw.githubusercontent.com/<repo>/<path>`，以获取原始 YAML 内容而非 HTML 页面。

#### 蓝图校验流程

`Blueprint.__init__` 在构造时执行两轮校验：

**第一轮：Voluptuous Schema 校验**

```python
class Blueprint:
    def __init__(self, data, *, path, expected_domain, schema):
        data = self.data = schema(data)  # ← BLUEPRINT_SCHEMA 校验
```

`BLUEPRINT_SCHEMA` 校验以下内容：

```python
BLUEPRINT_SCHEMA = vol.Schema({
    vol.Required(CONF_BLUEPRINT): vol.Schema({
        vol.Required(CONF_NAME): str,                 # 蓝图名称
        vol.Optional(CONF_DESCRIPTION): str,          # 描述
        vol.Required(CONF_DOMAIN): str,               # 所属域（automation/script）
        vol.Optional(CONF_SOURCE_URL): cv.url,        # 来源 URL
        vol.Optional(CONF_AUTHOR): str,               # 作者
        vol.Optional(CONF_HOMEASSISTANT): {            # HA 版本约束
            vol.Optional(CONF_MIN_VERSION): version_validator  # 格式: X.Y.Z
        },
        vol.Optional(CONF_INPUT, default=dict): vol.All(  # 输入定义
            {str: vol.Any(None, BLUEPRINT_INPUT_SCHEMA, BLUEPRINT_INPUT_SECTION_SCHEMA)},
            unique_input_validator,                    # 禁止重复 input key
        ),
    }),
}, extra=vol.ALLOW_EXTRA)  # ← 允许自动化/脚本配置通过（不做域级校验）
```

注意 `extra=vol.ALLOW_EXTRA`：蓝图 Schema 只校验 `blueprint:` 元数据部分，自动化/脚本的配置内容（triggers、actions 等）不在蓝图导入时校验——它们在后续由域级 Schema（`PLATFORM_SCHEMA`）校验。

**第二轮：`!input` 引用完整性校验**

```python
class Blueprint:
    def __init__(self, data, ...):
        # ...
        missing = yaml_util.extract_inputs(data) - set(self.inputs)
        if missing:
            raise InvalidBlueprint(..., f"Missing input definition for {', '.join(missing)}")
```

`extract_inputs()` 递归遍历整个 YAML 数据树，收集所有 `Input` 对象的 `name`。然后与蓝图 `input:` 定义中展平后的所有 key 对比——如果 `!input` 引用了未定义的输入名，抛出 `InvalidBlueprint`。

#### 蓝图替换流程 — 核心机制

当自动化使用 `use_blueprint:` 引用蓝图时，HA 需要将蓝图模板中的所有 `!input` 占位符替换为用户提供的实际值。这是蓝图机制的核心。

**源码路径**：`automation/config.py::_async_validate_config_item` → `blueprint/models.py::BlueprintInputs.async_substitute` → `annotatedyaml/input.py::substitute`

```
自动化配置包含 use_blueprint:
  │
  ├── config.py 检测到 blueprint.is_blueprint_instance_config(config) == True
  │     即配置中有 "use_blueprint" 键
  │
  ├── blueprints.async_inputs_from_config(config)
  │     │
  │     ├── BLUEPRINT_INSTANCE_FIELDS(config) → 校验 use_blueprint 结构
  │     │     校验: path 必须是 .yaml 后缀的合法路径
  │     │     校验: input 必须是 dict
  │     │
  │     ├── blueprints.async_get_blueprint(bp_conf["path"])
  │     │     │ 加载蓝图文件 → 解析 YAML → Blueprint 对象
  │     │     │ 缓存：已加载的蓝图存入 _blueprints[path]
  │     │     │ 并发保护：asyncio.Lock 防止重复加载
  │     │
  │     └── BlueprintInputs(blueprint, config_with_inputs)
  │           │
  │           ├── inputs.validate() → 校验输入完整性
  │           │     缺失必需输入（无 default 且用户未提供） → MissingInput
  │           │
  │           └── inputs.inputs_with_default → 合并用户输入和默认值
  │                 用户未提供的输入 → 使用蓝图 input 定义中的 default
  │
  ├── blueprint_inputs.async_substitute() → 生成完整自动化配置
  │     │
  │     ├── yaml_util.substitute(blueprint.data, inputs_with_default)
  │     │     │ 递归遍历 blueprint.data（整个 YAML 数据树）
  │     │     │ 遇到 Input 对象 → 用 inputs_with_default[input.name] 替换
  │     │     │ 遇到 list → 递归替换每个元素
  │     │     │ 遇到 dict → 递归替换每个值（key 不替换）
  │     │     │ 遇到其他类型 → 保持不变
  │     │     │ 未定义的 Input → 抛出 UndefinedSubstitution
  │     │
  │     ├── combined = {**processed, **config_with_inputs}
  │     │     合并蓝图替换后的配置 + 用户配置中可能额外添加的字段
  │     │
  │     ├── combined.pop("use_blueprint")  # 移除蓝图引用
  │     ├── combined.pop("blueprint")      # 移除蓝图元数据
  │     │
  │     └── 返回: 纯自动化配置（无蓝图标记）
  │
  └── PLATFORM_SCHEMA(config) → 对替换后的配置做域级校验
        校验 triggers、conditions、actions 等
```

**`substitute()` 的核心实现**：

```python
# annotatedyaml/input.py
def substitute(obj: Any, substitutions: dict[str, Any]) -> Any:
    """递归替换 YAML 数据中的 Input 对象。"""
    if isinstance(obj, Input):
        # 找到占位符 → 用用户提供的值替换
        if obj.name not in substitutions:
            raise UndefinedSubstitution(obj.name)
        return substitutions[obj.name]

    if isinstance(obj, list):
        return [substitute(val, substitutions) for val in obj]

    if isinstance(obj, dict):
        return {key: substitute(val, substitutions) for key, val in obj.items()}

    return obj  # 原始值（字符串、数字等）保持不变
```

关键点：`substitute()` 是一个**深度递归**函数，会遍历整个 YAML 数据树的所有层级。这意味着 `!input` 可以出现在 YAML 的任何位置——不仅限于实体 ID 或参数值，还可以出现在字典值、列表元素、嵌套结构中。例如 `target: !input light_target` 会被替换为一个完整的 `target` 配置块（如 `{entity_id: light.living_room}`）。

#### 蓝图替换的可视化示例

```yaml
# 蓝图文件 motion_light.yaml（包含 !input 占位符）
triggers:
  - trigger: state
    entity_id: !input motion_entity      ← Input("motion_entity")
    from: "off"
    to: "on"
actions:
  - action: light.turn_on
    target: !input light_target           ← Input("light_target")
  - delay: !input no_motion_wait          ← Input("no_motion_wait")

# 用户自动化配置（use_blueprint + input）
use_blueprint:
  path: homeassistant/motion_light.yaml
  input:
    motion_entity: binary_sensor.hall_motion
    light_target: {entity_id: light.hall}
    no_motion_wait: 300

# substitute() 替换后生成的完整配置
triggers:
  - trigger: state
    entity_id: binary_sensor.hall_motion  ← 替换完成
    from: "off"
    to: "on"
actions:
  - action: light.turn_on
    target: {entity_id: light.hall}        ← 替换完成
  - delay: 300                             ← 替换完成
```

### 11.4 蓝图与自动化/脚本的集成

#### 自动化集成中的蓝图集成

**源码**: `homeassistant/components/automation/helpers.py`

自动化集成在 `async_setup` 中注册蓝图域：

```python
# automation/__init__.py
async def async_setup(hass, config):
    # ...
    async_get_blueprints(hass)  # 注册 automation 域的 DomainBlueprints
    # 首次启动时自动复制内置蓝图到 config/blueprints/automation/
    hass.async_create_task(async_get_blueprints(hass).async_populate())
```

`async_get_blueprints()` 使用 `@singleton` 装饰器，确保全局只创建一个 `DomainBlueprints` 实例：

```python
# automation/helpers.py
@singleton(DATA_BLUEPRINTS)
@callback
def async_get_blueprints(hass) -> blueprint.DomainBlueprints:
    return blueprint.DomainBlueprints(
        hass,
        DOMAIN,                                   # "automation"
        LOGGER,
        _blueprint_in_use,                        # 检查蓝图是否被自动化引用
        _reload_blueprint_automations,             # 重载引用蓝图的自动化
        AUTOMATION_BLUEPRINT_SCHEMA,               # 专用蓝图 Schema
    )
```

#### 配置校验中的蓝图处理

**源码**: `homeassistant/components/automation/config.py::_async_validate_config_item`

自动化配置校验时，如果检测到 `use_blueprint:` 键，执行蓝图替换后再校验：

```python
if blueprint.is_blueprint_instance_config(config):
    # 1. 从 DomainBlueprints 加载蓝图
    blueprint_inputs = await blueprints.async_inputs_from_config(config)

    # 2. 替换 !input 占位符
    config = blueprint_inputs.async_substitute()

# 3. 对替换后的完整配置做域级校验
validated_config = PLATFORM_SCHEMA(config)
```

蓝图替换后的配置与手动编写的自动化配置完全一致——HA 内部不再区分"蓝图自动化"和"手动自动化"，它们走同样的校验和执行流程。

#### 脚本集成中的蓝图集成

脚本（script）集成同样支持蓝图，使用类似的模式：

```
# script 集成的蓝图 Schema
AUTOMATION_BLUEPRINT_SCHEMA → BLUEPRINT_SCHEMA（通用）
SCRIPT_BLUEPRINT_SCHEMA → BLUEPRINT_SCHEMA（通用）

# script 使用蓝图时的 YAML
script:
  my_notify:
    use_blueprint:
      path: homeassistant/confirmable_notification.yaml
      input:
        notify_device: <device_id>
        title: "确认操作"
        message: "是否执行此操作？"
```

当前蓝图仅支持 `automation` 和 `script` 两个域，但 `DomainBlueprints` 的设计是通用的——任何域都可以注册蓝图支持。

#### 蓝图的更新与重载

当蓝图文件被更新（重新导入或手动编辑 YAML）时：

```
DomainBlueprints.async_add_blueprint(blueprint, path, allow_override=True)
  │
  ├── 覆盖现有 YAML 文件
  ├── 更新内存缓存
  │
  └── 如果覆盖了已有蓝图 → _reload_blueprint_consumers()
        │ 对于 automation → 调用 automation.reload 服务
        │ 所有引用该蓝图的自动化将被重新加载
        │ → 重新执行蓝图替换 → 生成更新后的配置
```

#### 蓝图的"Take Control"（接管）功能

前端提供的 "Take Control" 功能实际上调用了 `blueprint/substitute` WebSocket 命令：

```python
# websocket_api.py::ws_substitute_blueprint
blueprint_config = {"use_blueprint": {"path": msg["path"], "input": msg["input"]}}
blueprint_inputs = await domain_blueprints.async_inputs_from_config(blueprint_config)
config = blueprint_inputs.async_substitute()  # 生成完整配置
```

前端拿到替换后的完整配置后，将其保存为普通自动化（不含 `use_blueprint:` 字段）。从此该自动化与蓝图脱离关系，用户可以自由编辑——但也失去了随蓝图更新自动同步的能力。

#### 蓝图机制的整体架构图

```
┌───────────────────────────────────────────────────────────────────┐
│  1. 蓝图导入 (importer.py + websocket_api.py)                     │
│                                                                   │
│  URL → fetch_blueprint_from_url() → 解析 YAML → Blueprint 对象    │
│    → 校验 BLUEPRINT_SCHEMA + !input 完整性                        │
│    → DomainBlueprints.async_add_blueprint()                        │
│      → 写入 config/blueprints/<domain>/<path>.yaml                 │
│      → 存入内存缓存                                                │
└───────────────────────────┬───────────────────────────────────────┘
                            │ 蓝图文件存储在磁盘
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│  2. 自动化/脚本配置加载 (config.py)                                │
│                                                                   │
│  检测 use_blueprint: → DomainBlueprints.async_inputs_from_config()│
│    → 加载蓝图 Blueprint 对象 → BlueprintInputs                    │
│    → 合并用户输入 + 默认值 → inputs_with_default                   │
│    → substitute(blueprint.data, inputs_with_default)               │
│      → 递归替换所有 !input Input 对象为实际值                       │
│    → 移除 blueprint: 和 use_blueprint: → 纯自动化/脚本配置         │
│    → PLATFORM_SCHEMA 校验 → 正常自动化/脚本流程                    │
└───────────────────────────┬───────────────────────────────────────┘
                            │ 替换后的配置与手动配置无异
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│  3. 运行时 (automation/__init__.py)                                │
│                                                                   │
│  替换后的自动化配置 → AutomationEntity                             │
│    保留 raw_blueprint_inputs 用于追踪和 Trace                       │
│    referenced_blueprint 属性 → 路径字符串                           │
│    蓝图更新时 → reload 服务 → 重新执行蓝图替换                      │
└───────────────────────────────────────────────────────────────────┘
```

---

## 12. 关键设计模式总结

### 12.1 架构模式

| 模式 | 实现 |
|------|------|
| **事件驱动** | EventBus + StateMachine，所有组件间通信通过事件 |
| **分层架构** | 入口层 → 核心层 → 加载层 → 实体层 → 集成层 |
| **泛型约束** | `EntityComponent[LightEntity]`, `ConfigEntry[HueBridge]` |
| **注册表模式** | ServiceRegistry, EntityRegistry, DeviceRegistry, HANDLERS 等 |
| **观察者模式** | EventBus 监听器, Coordinator 订阅, ConfigEntry 更新监听 |

### 12.2 加载模式

| 模式 | 实现 |
|------|------|
| **分阶段启动** | Stage 0/1/2，基础设施先行 |
| **并行加载** | `asyncio.gather` 并行设置 ConfigEntry 和平台 |
| **依赖注入** | `hass.data` 全局字典 + `DATA_INSTANCES` |
| **惰性加载** | 自定义集成按需解析，平台模块按需导入 |
| **线性递增退避重试** | `PlatformNotReady` → `min(tries,6)*30` 秒递增等待（30, 60, 90, ... 180s） |

### 12.3 实体模式

| 模式 | 实现 | 典型集成 |
|------|------|----------|
| **轮询式（纯 Entity）** | `should_poll=True` + `async_update()` | `moon` |
| **协调器式** | `DataUpdateCoordinator` + `CoordinatorEntity` | `hue` (v1) |
| **推送式（纯 Entity）** | `should_poll=False` + dispatcher/事件监听 + `async_write_ha_state()` | `sun` |
| **描述模式** | `EntityDescription` + `value_fn` 将属性从子类移到描述对象 | `sun` |
| **_attr_* 模式** | 类属性默认值，减少 property 定义 | `moon` |

### 12.4 集成模式

| 模式 | 实现 |
|------|------|
| **runtime_data 模式** | `entry.runtime_data` 存储运行时对象，类型安全 |
| **平台转发模式** | `async_forward_entry_setups` 将设置传播到各平台 |
| **配置流模式** | `ConfigFlow` + 步骤方法，支持 UI 配置 |
| **蓝图模式** | `!input` 占位符 + `substitute()` 递归替换，配置模板化复用 |
| **统一错误处理** | `async_request_call` 包装 API 调用，统一异常转换 |
| **选项 = 重载** | 选项变更触发 `async_reload`，重新初始化整个集成 |

### 12.5 @final 保护

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
| `homeassistant/components/blueprint/models.py` | 385 | Blueprint、BlueprintInputs、DomainBlueprints |
| `homeassistant/components/blueprint/importer.py` | 288 | 蓝图 URL 导入逻辑（论坛/GitHub/Gist/官网/通用） |
| `homeassistant/components/blueprint/schemas.py` | 151 | BLUEPRINT_SCHEMA、BLUEPRINT_INSTANCE_FIELDS |
| `homeassistant/components/blueprint/websocket_api.py` | 274 | 蓝图 WebSocket API（导入/保存/删除/替换） |
| `homeassistant/components/automation/config.py` | 338 | 自动化配置校验（含蓝图替换流程） |
| `homeassistant/components/automation/helpers.py` | 40 | 自动化蓝图 DomainBlueprints 注册 |
| `annotatedyaml/input.py` | ~60 | `Input` 类、`substitute()`、`extract_inputs()` |
| `homeassistant/runner.py` | 330 | 运行器 |
