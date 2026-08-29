---
title: 'Home Assistant 核心架构与实现深度解析'
date: '2026-06-05T00:41:47+08:00'
draft: false
tags: ['python', 'asyncio', 'home-assistant', 'iot']
author: 'synodriver'
---

# Home Assistant 核心架构与实现深度解析

> 基于源码版本 2026.9.0.dev0（提交 `478b722e25d`），Python 3.14.5+

## 目录

- [1. 项目总体结构](#1-项目总体结构)
- [2. 核心类与基类体系](#2-核心类与基类体系)
  - [2.9 ConfigSubentry 与 2026.9 设备归属模型](#29-configsubentry-与-20269-设备归属模型)
- [3. 启动流程详解](#3-启动流程详解)
- [4. 集成加载机制](#4-集成加载机制)
- [5. 配置流(ConfigFlow)机制](#5-配置流configflow机制)
  - [5.4 FlowManager 与专用 Manager](#54-flowmanager-与专用-manager)
    - [5.4.1 FlowHandler 与 FlowManager 的职责边界](#541-flowhandler-与-flowmanager-的职责边界)
    - [5.4.2 FlowManager 的公共调用流程](#542-flowmanager-的公共调用流程)
    - [5.4.3 Config Entry 体系的三个专用 Manager](#543-config-entry-体系的三个专用-manager)
    - [5.4.4 从前端 API 到 Manager 的完整路径](#544-从前端-api-到-manager-的完整路径)
    - [5.4.5 其他复用 FlowManager 的流程](#545-其他复用-flowmanager-的流程)
  - [5.7 ConfigFlow step 方法职责与调用时机](#57-configflow-step-方法职责与调用时机)
  - [5.8 ConfigEntry 创建、更新与 `unique_id`](#58-configentry-创建更新与-unique_id)
  - [5.9 ConfigFlow 实例生命周期与 BLE 发现去重](#59-configflow-实例生命周期与-ble-发现去重)
    - [5.9.1 多个蓝牙代理收到同一广播时如何去重](#591-多个蓝牙代理收到同一广播时如何去重)
    - [5.9.2 需要 GATT 连接时如何选择网关](#592-需要-gatt-连接时如何选择网关)
  - [5.10 OptionsFlow — 选项流](#510-optionsflow--选项流)
    - [5.10.1 OptionsFlow 的数据模型与源码入口](#5101-optionsflow-的数据模型与源码入口)
    - [5.10.2 OptionsFlow 的执行时序](#5102-optionsflow-的执行时序)
    - [5.10.3 OptionsFlow 的实现方式](#5103-optionsflow-的实现方式)
    - [5.10.4 OptionsFlow 与 ConfigFlow 的区别](#5104-optionsflow-与-configflow-的区别)
    - [5.10.5 选项流的常见误区](#5105-选项流的常见误区)
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
- [12. MQTT 集成：异步客户端与自动发现](#12-mqtt-集成异步客户端与自动发现)
  - [12.1 MQTT 集成的分层与启动顺序](#121-mqtt-集成的分层与启动顺序)
  - [12.2 把同步 Paho 客户端接入 asyncio](#122-把同步-paho-客户端接入-asyncio)
  - [12.3 将 publish/subscribe 的 MID 回调变成 await](#123-将-publishsubscribe-的-mid-回调变成-await)
  - [12.4 MQTT 实体自动发现](#124-mqtt-实体自动发现)
  - [12.5 manifest 的 MQTT 配置流发现](#125-manifest-的-mqtt-配置流发现)
  - [12.6 设备侧 discovery 示例与工程约束](#126-设备侧-discovery-示例与工程约束)
- [13. 关键设计模式总结](#13-关键设计模式总结)

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
  "requirements": ["aiohue==4.9.0"],
  "iot_class": "local_push",
  "zeroconf": ["_hue._tcp.local."],
  "codeowners": ["@marcelveldt"]
}
```

| 字段 | 说明 |
|------|------|
| `domain` | 集成域名，必须等于目录名 |
| `name` | 人类可读名称 |
| `integration_type` | `entity`(平台型)、`device`(设备型)、`hardware`(硬件基础设施)、`helper`(辅助型)、`hub`(集线器型)、`service`(服务型)、`system`(系统型)、`virtual`(虚拟型) |
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
        self.config.async_initialize()             # 初始化配置对象的事件循环侧状态
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
| `async_create_task()` / `async_create_background_task()` | Core 创建普通任务/长生命周期后台任务；集成优先使用 `ConfigEntry` 上的同名方法绑定卸载生命周期 |
| `async_add_executor_job()` | 把阻塞函数提交到默认线程池；不要在事件循环中直接执行文件、网络等阻塞 I/O |

`hass.async_add_job()` 和 `hass.async_add_hass_job()` 已经过弃用周期，不应再作为新集成的调度入口。Config Entry 驱动的集成应优先使用：

```python
# 普通任务会参与 setup/unload 生命周期管理
entry.async_create_task(hass, do_one_shot_work(), name="refresh device")

# 常驻任务在 entry unload 时自动取消，也不会阻塞 HA 完成启动
entry.async_create_background_task(
    hass, run_push_listener(), name="device push listener"
)

# 真正阻塞的同步调用才进入 executor
result = await hass.async_add_executor_job(sync_library.read, address)
```

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

    async def async_get_component(self) -> ComponentProtocol  # 获取 __init__.py 模块
    async def async_get_platform(self, platform) -> ModuleType  # 获取平台模块
    async def async_get_platforms(self, platforms) -> dict[str, ModuleType]
    async def resolve_dependencies(self) -> set[str] | None
```

关键数据键：
- `DATA_COMPONENTS` — 已加载的组件模块缓存
- `DATA_INTEGRATIONS` — 已解析的 Integration 对象缓存
- `DATA_CUSTOM_COMPONENTS` — 自定义集成缓存

### 2.9 ConfigSubentry 与 2026.9 设备归属模型

`ConfigSubentry` 用于在一个 `ConfigEntry` 下表示多个同类逻辑单元，例如一个账号下的多个站点，或 MQTT 配置条目下由 UI 管理的多个设备。它不是独立的 `ConfigEntry`，共享父条目的连接与生命周期：

```python
@dataclass(frozen=True, kw_only=True)
class ConfigSubentry:
    data: MappingProxyType[str, Any]
    subentry_id: str        # Core 分配的 ULID
    subentry_type: str
    title: str
    unique_id: str | None
```

平台添加实体时可传入 `config_subentry_id`，Core 会校验该 subentry 确实属于当前 entry，并把实体和设备注册到对应 subentry：

```python
async_add_entities(entities, config_subentry_id=subentry.subentry_id)
```

2026.8 之后 Device Registry 的关键约束发生了变化：一个设备只归属一个 `config_entry_id` 和一个可选的 `config_subentry_id`。旧的 `device.config_entries`、`config_entries_subentries` 和 `primary_config_entry` 只是兼容属性；新代码应读取单数属性。设备查找也应限定 entry，例如：

```python
device = device_registry.async_get_device_by_identifier(
    (DOMAIN, serial_number), config_entry.entry_id
)
```

2026.9 又新增了真正的轻量级 child device。它与 `DeviceInfo.via_device_id` 表达的“经由某个网关通信的完整设备”不同：child device 是父设备内部的逻辑组成部分，只拥有少量元数据，不能继续充当其他 child 的父级。实体返回 `ChildDeviceInfo` 后，`EntityPlatform` 会调用 `async_get_or_create_child()`：

```python
from homeassistant.helpers.device_registry import ChildDeviceInfo

self._attr_device_info = ChildDeviceInfo(
    identifiers={(DOMAIN, channel_id)},
    name="Channel 1",
    parent_device_id=parent_device_entry.id,
)
```

父设备必须先注册，并且 child 与父设备必须属于同一个 config entry 和同一个 config subentry。`parent_device_id` 是 Device Registry 的内部设备 ID，不是 `(domain, identifier)`；需要先通过限定 config entry 的查找 API 解析出来。

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
                    │     ├── fire EVENT_CORE_CONFIG_UPDATE
                    │     ├── fire EVENT_HOMEASSISTANT_START
                    │     ├── wait startup tasks and startup jobs (with timeout)
                    │     ├── CoreState → RUNNING
                    │     ├── fire EVENT_CORE_CONFIG_UPDATE
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

`async_start()` 不是简单地“发 START 事件后立即把状态改成 RUNNING”：它先等待 START 事件触发的 core tasks，再执行通过 `async_at_started()` 等 API 登记的 startup jobs；只有这些阶段在超时窗口内完成后，才把 `CoreState` 设为 `running` 并发出 `EVENT_HOMEASSISTANT_STARTED`。停止时还会依次发出 `EVENT_HOMEASSISTANT_STOP`、`EVENT_HOMEASSISTANT_FINAL_WRITE` 和 `EVENT_HOMEASSISTANT_CLOSE`，最后才进入 `CoreState.stopped`。

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

### 5.4 FlowManager 与专用 Manager

文章前面提到的 `FlowHandler` 和 `FlowManager` 是两条互补的继承体系：前者表示“一次流程中的业务步骤”，后者负责“同时编排和保存多次流程”。`ConfigEntries` 则是 ConfigEntry 的总管理器，它不是 `FlowManager` 的子类，而是在初始化时持有三个专用 flow manager：

```python
class ConfigEntries:
    def __init__(self, hass, hass_config):
        self.flow = ConfigEntriesFlowManager(hass, self, hass_config)
        self.options = OptionsFlowManager(hass)
        self.subentries = ConfigSubentryFlowManager(hass)
```

对应的两组继承关系如下：

```
流程处理器（一次会话一个实例）
FlowHandler
  ├── ConfigEntryBaseFlow
  │     ├── ConfigFlow
  │     └── OptionsFlow
  ├── ConfigSubentryFlow
  ├── RepairsFlow
  └── LoginFlow / MFA setup flow

流程管理器（通常由 HA 长期持有，一个实例管理多次会话）
FlowManager
  ├── ConfigEntriesFlowManager
  ├── OptionsFlowManager
  ├── ConfigSubentryFlowManager
  ├── RepairsFlowManager
  ├── AuthManagerFlowManager
  └── MfaFlowManager
```

#### 5.4.1 FlowHandler 与 FlowManager 的职责边界

| 对象 | 生命周期与数量 | 主要职责 | 不负责什么 |
|---|---|---|---|
| `FlowHandler` 及其子类 | 每启动一次 flow 创建一个实例 | 实现 `async_step_*`，执行业务校验，返回 `FORM`、`CREATE_ENTRY`、`ABORT` 等结果 | 不登记并发流程，不直接决定结果如何持久化 |
| `FlowManager` | 通常是长期对象，同时持有多个 flow | 创建 handler、分配 `flow_id`、注入上下文、校验 schema、推进 step、跟踪进度、终止和清理流程 | 不实现具体集成的 host/token/设备校验 |
| 专用 Manager | 每类流程一个长期对象 | 覆写 `async_create_flow()` 决定创建哪种 handler；覆写 `async_finish_flow()` 决定成功结果的副作用 | 不取代集成的 `async_step_*` |
| `ConfigEntries` | `hass.config_entries` 上的长期总管理器 | 保存和索引 `ConfigEntry`，负责 setup/unload/reload/update/remove，并持有三个 flow manager | 不是 flow 状态机，也不继承 `FlowManager` |

这个边界解释了为什么集成中的 `self.async_create_entry(...)` 只是构造一个 result：Handler 表达“流程成功且这些数据应被保存”，真正创建 ConfigEntry、更新 options 或增加 subentry 的动作由对应 Manager 在 `async_finish_flow()` 中解释。

#### 5.4.2 FlowManager 的公共调用流程

`homeassistant/data_entry_flow.py` 中的泛型 `FlowManager` 提供所有 flow 共用的状态机。它至少维护以下索引：

| 内部字段 | 作用 |
|---|---|
| `_progress[flow_id]` | 按 `flow_id` 找到正在进行的 `FlowHandler` |
| `_handler_progress_index[handler]` | 按 handler key 查找同类 flow，用于并发检查和去重 |
| `_init_data_process_index[type]` | 按初始 discovery data 类型查找 flow，Bluetooth、SSDP、USB 等用它抑制或中止重复发现 |
| `_preview` | 记录已完成 preview 初始化的 handler，避免重复 setup |

公共的启动、推进与结束过程为：

```
manager.async_init(handler, context=..., data=...)
  ├── manager.async_create_flow(handler, context, data)
  │     └── 专用 Manager 返回具体 FlowHandler
  ├── 注入 flow.hass / handler / flow_id / context / init_data
  ├── _async_add_flow_progress(flow)
  │     └── 写入 _progress、handler 和 init_data 类型索引
  └── _async_handle_step(flow, flow.init_step, data)
        ├── 检查 async_step_<step_id> 是否存在
        ├── await async_step_<step_id>(data)
        ├── 将 AbortFlow 异常规范化为 ABORT result
        ├── 管理 preview/progress task 和 result 类型
        ├── 未完成：保存 flow.cur_step，等待下一次输入
        └── 已完成：调用专用 manager.async_finish_flow(flow, result)
              ├── 专用副作用：创建条目/更新 options/增加 subentry/完成登录等
              ├── 若 Manager 返回 FORM：保存新的 cur_step，流程继续
              └── 否则 _async_remove_flow_progress(flow_id)
                    ├── 从全部索引删除
                    ├── 取消 progress task
                    └── 调用 flow.async_remove() 做清理
```

第一次返回 `FORM`、`MENU`、`EXTERNAL_STEP` 或 `SHOW_PROGRESS` 后，前端使用同一个 `flow_id` 继续：

```text
manager.async_configure(flow_id, user_input)
  ├── 从 _progress 取得 flow 和 flow.cur_step
  ├── 如果有 data_schema，先用 voluptuous 校验 user_input
  ├── MENU：转到 user_input["next_step_id"]
  └── 其他类型：再次调用当前 async_step_<step_id>(user_input)
```

`async_get()`、`async_progress()`、`async_progress_by_handler()` 和 `async_progress_by_init_data_type()` 只查询进行中的 flow；`async_abort(flow_id)` 则从索引移除流程并执行同样的取消/清理钩子。基类默认使用随机 UUID hex 作为 `flow_id`，`ConfigEntriesFlowManager` 为了自身的初始化与关闭控制覆写了启动逻辑，当前源码中使用 ULID。

#### 5.4.3 Config Entry 体系的三个专用 Manager

三个对象都挂在 `hass.config_entries` 下，但 handler key、创建的 Handler 和完成语义不同：

| Manager / 访问入口 | `async_init()` 的 handler key | 创建的 Handler | `async_finish_flow(CREATE_ENTRY)` 的动作 |
|---|---|---|---|
| `ConfigEntriesFlowManager` / `hass.config_entries.flow` | integration domain，例如 `"hue"` | 该 domain 注册的 `ConfigFlow` | 校验单实例和 `unique_id`，构造 `ConfigEntry`，调用 `ConfigEntries.async_add()` |
| `OptionsFlowManager` / `hass.config_entries.options` | 已有 `ConfigEntry.entry_id` | domain 的 `ConfigFlow.async_get_options_flow(entry)` 返回的 `OptionsFlow` | `async_update_entry(entry, options=result["data"])`，必要时安排 reload |
| `ConfigSubentryFlowManager` / `hass.config_entries.subentries` | `(entry_id, subentry_type)` | `ConfigFlow.async_get_supported_subentry_types(entry)` 注册的 `ConfigSubentryFlow` | 构造 `ConfigSubentry` 并调用 `async_add_subentry()` |

`OptionsFlowManager` 和 `ConfigSubentryFlowManager` 都混入 `_ConfigSubFlowManager`，该 mixin 只提供“通过 entry ID 取得已有 ConfigEntry”的公共能力。它不是第四种流程管理器。

`ConfigEntriesFlowManager` 的额外职责最多：

1. `async_init()` 强制 `context["source"]` 存在；reauth/reconfigure 还必须带 `entry_id`。
2. `async_create_flow()` 加载 domain 的 `config_flow.py`，从 `HANDLERS` 取得类并实例化，然后令 `init_step = context["source"]`。
3. 启动前检查 single-config-entry 限制，跟踪 YAML import 初始化任务，并允许 HA 关闭时取消尚在初始化的 flow。
4. 对 discovery flow 发送新增/移除通知，并使用进行中索引处理同 handler、同 `unique_id` 或默认 discovery ID 的冲突。
5. 完成时构造 `ConfigEntry`；如果 result 携带 `next_flow`，还会确认目标 Config、Options 或 Subentry flow 真实存在。

`OptionsFlowManager` 的完整保存路径见 [5.10.2](#5102-optionsflow-的执行时序)。`ConfigSubentryFlowManager` 的创建路径则是：

```
subentries.async_init(
    (entry_id, subentry_type),
    context={"source": "user"},
)
  ├── 找到父 ConfigEntry
  ├── 加载父 entry.domain 的 ConfigFlow handler
  ├── async_get_supported_subentry_types(entry)
  ├── 实例化指定 subentry_type 的 ConfigSubentryFlow
  ├── 公共 FlowManager 状态机推进 async_step_user()
  └── CREATE_ENTRY
        └── ConfigSubentryFlowManager.async_finish_flow()
              └── ConfigEntries.async_add_subentry(parent_entry, new_subentry)
```

若 context source 是 `reconfigure`，还会携带 `subentry_id` 并进入 `async_step_reconfigure()`；Handler 通过 `async_update_and_abort()` 或 `async_update_reload_and_abort()` 更新原 subentry。后者不能与父 entry 的 update listener 同时使用，否则 Core 会拒绝重复的更新/reload 路径。

#### 5.4.4 从前端 API 到 Manager 的完整路径

Core 没有让前端直接调用 Handler。`homeassistant/helpers/data_entry_flow.py` 提供通用 HTTP view，`homeassistant/components/config/config_entries.py` 把三个专用 Manager 分别绑定到路由：

| 用途 | 创建流程 POST | 继续流程 POST / 读取当前步骤 GET | handler 内容 |
|---|---|---|---|
| ConfigFlow | `/api/config/config_entries/flow` | `/api/config/config_entries/flow/{flow_id}` | domain；reconfigure 时另外传 `entry_id` |
| OptionsFlow | `/api/config/config_entries/options/flow` | `/api/config/config_entries/options/flow/{flow_id}` | `entry_id` |
| ConfigSubentryFlow | `/api/config/config_entries/subentries/flow` | `/api/config/config_entries/subentries/flow/{flow_id}` | `[entry_id, subentry_type]` |

通用 view 到 Manager 的映射如下：

```
POST index {handler, ...}
  → FlowManagerIndexView._post_impl()
  → manager.async_init(handler, context=路由构造的上下文)
  → 返回第一个 FlowResult

POST resource/{flow_id} {user_input...}
  → FlowManagerResourceView.post()
  → manager.async_configure(flow_id, user_input)
  → 返回下一个或最终 FlowResult

GET resource/{flow_id}
  → manager.async_configure(flow_id)
  → 推进/轮询 external 或 progress step

DELETE resource/{flow_id}
  → manager.async_abort(flow_id)
```

因此“前端提交表单后是谁调用 `async_step_user`”的完整答案是：HTTP resource view 调用专用 Manager 的 `async_configure()`，基类 `FlowManager` 读取 `cur_step`、校验 schema，再由 `_async_handle_step()` 反射调用 Handler 的 `async_step_user()`。自动发现不经过创建流程的 HTTP POST，而是 Bluetooth/Zeroconf/SSDP 等后端代码直接调用 `hass.config_entries.flow.async_init(domain, context, discovery_info)`；流程进入等待用户确认后，前端才通过同一个 `flow_id` 接管后续步骤。

#### 5.4.5 其他复用 FlowManager 的流程

`FlowManager` 是通用数据录入状态机，不只服务于 ConfigEntry：

| Manager | 持有位置/入口 | handler key | 完成语义 |
|---|---|---|---|
| `AuthManagerFlowManager` | `hass.auth.login_flow` | `(provider_type, provider_id)` | 把登录数据转换为 `Credentials`，必要时继续 MFA step |
| `MfaFlowManager` | Auth integration 的 `hass.data` | MFA module ID | 编排 MFA setup flow，最终结果由模块处理 |
| `RepairsFlowManager` | Repairs integration 的 `flow_manager` | issue domain，初始 data 带 `issue_id` | 创建集成专属 fix flow；非 ABORT 完成后删除已修复 issue |

它们共享 `async_init → async_step_* → async_configure → async_finish_flow → 清理` 的骨架，但各自有不同的 handler key、结果类型和最终副作用。至于 `AuthManager`、`TimeoutManager`、设备注册表等名称中也带 Manager 的类，只是一般意义上的服务管理器，并不继承 `data_entry_flow.FlowManager`，不能套用本节的 flow 调用约定。

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

### 5.9.1 多个蓝牙代理收到同一广播时如何去重

这里要把“同一份广播被多个代理上报”和“同一台设备被重复配置”分开。HA 不会把代理 MAC 当成被扫描设备的身份：`BluetoothServiceInfoBleak.address` 是外设地址，`source` 才是本地适配器或 ESPHome 蓝牙代理的来源 ID。因此两个代理看到同一 MAC 时，是“同一 address 的两条可达路径”，不是两台外设。

当前 Core 把这部分通用算法放在 `habluetooth==6.26.5` 依赖中；`homeassistant/components/bluetooth/manager.py` 的 `HomeAssistantBluetoothManager` 继承 `habluetooth.manager.BluetoothManager`，再接上 HA 的 matcher、callback 和 discovery flow。完整数据路径是：

```text
ESPHome proxy A ─┐
                  ├─ BluetoothServiceInfoBleak(address=外设 MAC, source=代理 MAC)
ESPHome proxy B ─┘
                             ↓
BluetoothManager.scanner_adv_received()
  ├─ 每个 scanner 仍保留自己看到的 BLEDevice/AdvertisementData
  ├─ _all_history[address]         # 全部路径中当前首选广播
  ├─ _connectable_history[address] # 可连接路径中当前首选广播
  └─ 只将接管成功且 payload 有变化的规范广播送入 HA discovery/callback
                             ↓
HomeAssistantBluetoothManager._discover_service_info()
  ├─ IntegrationMatcher 按 address 记录已见字段
  └─ DiscoveryKey(domain="bluetooth", key=address, version=1)
                             ↓
ConfigFlow.async_step_bluetooth()
  └─ async_set_unique_id(稳定设备 ID) + _abort_if_unique_id_configured()
                             ↓
Device Registry
  └─ connections={("bluetooth", address)}
```

`_all_history` 和 `_connectable_history` 都以外设 `address` 为 key，所以对上层暴露的是每个地址一份规范历史，而不是每个 `(address, source)` 各一份。但 manager 并没有丢掉其他路径：每个 scanner 的发现缓存仍然保留该 address，稍后 GATT 连接正是从这些候选中选网关。

当新广播来自另一个 `source` 时，`_should_keep_previous_adv()` / `_prefer_previous_adv_from_different_source()` 决定是否更换当前广播 owner：

- 对每个 `(address, source)` 的 RSSI 做 EWMA 平滑，当前系数为 `0.3`，避免一次瞬时尖峰导致换代理。
- 新来源的平滑 RSSI 需比当前 owner 强超过 `16 dB` 才主动接管；刚被降级的来源想立即抢回 owner，还要额外跨过 `6 dB` deadband。
- owner 超过已学习的广播间隔未出现时，再进入 stale handoff；被动广播设备和需要主动扫描的设备有不同的漫游保护，防止轮流丢包让 owner 在代理间往返抖动。
- 非可连接 scanner 可以成为 `_all_history` 的最佳广播来源，却不会覆盖 `_connectable_history` 中独立保留的可连接路径。因此 Shelly 这类只能扫描的代理不会把 ESPHome/本地适配器的 GATT 路径“挤掉”。

选出 owner 后，manager 先写入以 address 为 key 的 history。如果 manufacturer data、service data、service UUID 和 name 都与上次相同，会直接 `return`，不再调用 `_discover_service_info()`；所以两个代理先后上报相同 payload，通常只会向集成 callback/discovery 投递一次。RSSI、接收时间和 owner 仍可以在底层更新，但它们单独变化不会被当成新的设备数据。

即使极端时序让两次 discovery 都走到 ConfigFlow，仍有两层身份保护：

1. Bluetooth manager 传入的 `DiscoveryKey` 是蓝牙外设 address，用于关联发现与 ConfigEntry；集成的 `async_set_unique_id()` 会检查同 domain 下相同 ID 的进行中 flow，`_abort_if_unique_id_configured()` 再阻止创建第二个 ConfigEntry。
2. 实体进入 Device Registry 时，BLE 集成通常提供 `connections={(CONNECTION_BLUETOOTH, address)}`。`async_get_or_create()` 会在该 ConfigEntry 所属设备中先按 identifiers/connections 查找，匹配就更新原记录，不会因 `source` 不同再新建一台设备。

因而“只创建一个设备”不是一个 `set()` 就完成的，而是 **address 级广播仲裁 → payload 变化检测 → discovery/flow 去重 → ConfigEntry `unique_id` → Device Registry connection** 多层共同完成。同时要注意，通用蓝牙层只能把“相同 address”视为同一设备；若外设使用会轮换的 random private address，集成必须从 payload 中提取序列号等稳定 ID 作为 `unique_id`，HA 无法凭两个不同随机地址自动推断它们是同一台设备。

### 5.9.2 需要 GATT 连接时如何选择网关

GATT 网关不是在创建 ConfigEntry 时固化到配置里，也不一定沿用 `_all_history[address].source`。集成通常只保存外设 address，需要连接时调用 `async_ble_device_from_address(hass, address, connectable=True)` 取一个可连接 `BLEDevice`，然后使用 `bleak_retry_connector.establish_connection()`。例如 Xiaomi BLE 的 `_async_poll()` 在收到不可连接代理的广播时，会按 address 换成 `connectable=True` 的 `BLEDevice` 再调用库轮询。

HA 启动 Bluetooth manager 时，`habluetooth.usage.install_multiple_bleak_catcher()` 会把 Bleak/bleak-retry-connector 的 client 替换为 `HaBleakClientWrapper`。因此按标准方式发起的连接，最终会在 `HaBleakClientWrapper.connect()` 内动态选路：

```text
integration: establish_connection(..., BLEDevice(address), ...)
                         ↓
HaBleakClientWrapper.connect()
                         ↓
BluetoothManager.async_scanner_devices_by_address(address, connectable=True)
  ├─ local adapter:  BLEDevice + AdvertisementData + local scanner
  ├─ ESPHome proxy A: BLEDevice + AdvertisementData + ESPHome scanner
  └─ ESPHome proxy B: BLEDevice + AdvertisementData + ESPHome scanner
                         ↓
BluetoothScannerDevice.score_connection_path()
                         ↓
选第一个可用 backend：本地 Bleak backend 或某个代理的 connector.client
```

候选集首先排除 `connectable=False` 的 scanner，然后按 RSSI 排序，并以最强两条路径的 RSSI 差值作为处罚尺度计算 score。`BaseHaScanner._score_connection_paths()` 的当前评分因素为：

| 因素 | 对选路的影响 |
|---|---|
| 该代理看到的 RSSI | 基础分，越强越优先 |
| scanner 当前正在建立的其他连接数 | 每个 in-progress 连接扣分，避免将连接压在同一网关上 |
| 该 scanner 连接这个 address 的历史失败数 | 失败越多扣分越多，下次 retry 可改走其他路径 |
| GATT connection slots | `free == 0` 时该路径得到 `NO_RSSI_VALUE` 并在 backend 可用性检查中被跳过；只剩一个槽位也会小幅扣分 |
| connector 实时可用性 | ESPHome 路径还要通过 `scanner.connector.can_connect()`，它会检查代理 API 在线状态和 `ble_connections_free` |

评分后按优先级遍历候选。本地适配器必须能从 `BleakSlotManager` 分配到槽位；远程代理必须有 connector 且 `can_connect()` 为真。第一个通过的路径成为本次连接 backend；若是 ESPHome，便是 `bleak_esphome` 的 `ESPHomeClient` 通过该代理已建立的 HA API 连接要求 ESP32 发起 GATT 连接。

因此答案不是简单的“永远选 RSSI 最强的网关”，而是 **每次 connect 时在所有可连接路径中动态重选，以 RSSI 为基础，再考虑负载、历史失败、槽位和网关实时在线状态**。广播 owner 选择为了稳定上层数据，GATT 路径选择为了成功建连与负载分散；这是两个不同问题，不应把 `service_info.source` 直接持久化成“设备绑定网关”。

### 5.10 OptionsFlow — 选项流

`OptionsFlow` 用于修改**已经存在的 ConfigEntry 的可变选项**。例如轮询间隔、启用哪些实体、媒体播放参数、调试开关等，都属于选项；设备地址、用户名、令牌等决定“连接到哪个服务”的身份配置，通常属于 `ConfigEntry.data`，应由 ConfigFlow 或 reconfigure/reauth flow 管理。

OptionsFlow 的最终结果不会创建第二个 ConfigEntry，而是把流程返回的 `data` 写回原条目的 `options` 字段。可以把它理解为“绑定到某个 ConfigEntry 的表单编辑器”，而不是第二种设备发现流程。

### 5.10.1 OptionsFlow 的数据模型与源码入口

相关实现位于 `homeassistant/config_entries.py`：

```python
class ConfigEntries:
    def __init__(self, hass, hass_config):
        self.flow = ConfigEntriesFlowManager(hass, self, hass_config)
        self.options = OptionsFlowManager(hass)

class ConfigFlow(ConfigEntryBaseFlow):
    @staticmethod
    @callback
    def async_get_options_flow(config_entry: ConfigEntry) -> OptionsFlow:
        raise UnknownHandler

class OptionsFlow(ConfigEntryBaseFlow):
    @property
    def _config_entry_id(self) -> str: ...

    @property
    def config_entry(self) -> ConfigEntry: ...
```

`ConfigEntry.supports_options` 会检查对应 `ConfigFlow` 是否覆写了 `async_get_options_flow()`；前端据此决定是否显示“配置选项”入口。默认实现抛出 `UnknownHandler`，因此只实现 ConfigFlow 的集成不会意外获得一个空的选项页。

`OptionsFlow` 基类提供的 `config_entry` 和 `_config_entry_id` 属性在 flow 被 `OptionsFlowManager` 初始化之后才可用，不能在 `__init__` 中读取。当前 Core 中 `OptionsFlowWithConfigEntry` 只为旧自定义集成保留并处于淘汰阶段，新代码不要再继承它，也不要给只读的 `config_entry` 属性赋值。

### 5.10.2 OptionsFlow 的执行时序

用户从某个已配置集成的设置页进入选项时，调用链可以概括为：

```
前端
  │  hass.config_entries.options.async_init(entry.entry_id,
  │      context={"source": "user"})
  ↓
OptionsFlowManager.async_create_flow(entry_id)
  ├── 根据 entry_id 找到已有 ConfigEntry
  ├── 加载 entry.domain 的 config_flow.py
  └── handler.async_get_options_flow(entry) → OptionsFlow 实例
  ↓
FlowManager 设置 flow.hass / flow.handler / flow_id
  └── 默认进入 async_step_init(None)
  ↓
前端提交表单
  └── options.async_configure(flow_id, user_input)
        ├── 先用 data_schema 校验输入
        └── 反射调用 async_step_<当前 step>(user_input)
  ↓
async_create_entry(data=new_options)
  ↓
OptionsFlowManager.async_finish_flow()
  ├── ABORT：结束流程，不修改条目
  └── CREATE_ENTRY：async_update_entry(entry, options=new_options)
                    └── 持久化、触发 update listeners
                        └── OptionsFlowWithReload 时安排 reload
```

这条路径与 ConfigFlow 的关键差异在最后一步：ConfigFlow 的 `CREATE_ENTRY` 会由 Core 构造并加入一个新的 `ConfigEntry`；OptionsFlow 的 `CREATE_ENTRY` 只更新已有条目的 `options`。如果表单返回的数据与原 `options` 相同，`async_update_entry()` 返回 `False`，不会触发更新监听器，也不会因为 `OptionsFlowWithReload` 安排无意义的 reload。

源码中的 `OptionsFlowManager.async_finish_flow()` 还规定：只有 `result["type"] == CREATE_ENTRY` 且 `result["data"]` 不为 `None` 时才写入选项；因此取消或中止流程不会产生半成品配置。

### 5.10.3 OptionsFlow 的实现方式

最小实现通常包含两部分：在 ConfigFlow 中返回 OptionsFlow，在 OptionsFlow 的 `async_step_init` 中读取旧选项、显示表单并返回新选项。新代码可以直接使用 Core 注入的 `self.config_entry`：

```python
from typing import Any

import voluptuous as vol

from homeassistant.config_entries import (
    ConfigEntry,
    ConfigFlow,
    ConfigFlowResult,
    OptionsFlow,
    OptionsFlowWithReload,
)
from homeassistant.core import callback

from .const import DOMAIN


class MyConfigFlow(ConfigFlow, domain=DOMAIN):
    """创建 My Integration 的 ConfigEntry。"""

    @staticmethod
    @callback
    def async_get_options_flow(config_entry: ConfigEntry) -> OptionsFlow:
        """返回该条目的选项流。"""
        # 不要在这里给 flow.config_entry 赋值；Manager 会在初始化后注入关联关系。
        return MyOptionsFlow()


class MyOptionsFlow(OptionsFlowWithReload):
    """编辑不会改变设备身份的运行选项。"""

    async def async_step_init(
        self, user_input: dict[str, Any] | None = None
    ) -> ConfigFlowResult:
        """显示并保存选项。"""
        if user_input is not None:
            # data 会被 OptionsFlowManager 写入 ConfigEntry.options。
            return self.async_create_entry(title="", data=user_input)

        options = self.config_entry.options
        return self.async_show_form(
            step_id="init",
            data_schema=vol.Schema(
                {
                    vol.Required(
                        "poll_interval",
                        default=options.get("poll_interval", 30),
                    ): vol.All(vol.Coerce(int), vol.Range(min=5)),
                    vol.Optional(
                        "enable_extra_entities",
                        default=options.get("enable_extra_entities", True),
                    ): bool,
                }
            ),
        )
```

几点实现约定：

1. 表单默认值应从 `self.config_entry.options` 读取；`options` 是只读映射，不能原地修改。需要跨多个 step 暂存时，复制到 flow 自己的普通 `dict`，最后一次性传给 `async_create_entry(data=...)`。
2. `async_create_entry()` 中的 `data` 是“完整的新 options 映射”，不是自动合并的 patch。只提交表单中的一部分字段时，要显式合并旧值，例如 `self.config_entry.options | user_input`。
3. `async_create_entry()` 的 `title` 对 OptionsFlow 的持久化没有 ConfigFlow 那样的含义，通常传空字符串即可；真正被 `OptionsFlowManager` 使用的是 `data`。
4. 复杂的多页编辑可以像 ConfigFlow 一样定义 `async_step_xxx`、`async_show_menu`、外部步骤和进度步骤；起始步骤仍通常是 `async_step_init`。

对于纯 schema 驱动的集成，可以复用 `homeassistant.helpers.schema_config_entry_flow`：定义 `SchemaConfigFlowHandler.options_flow`，Core helper 会生成 `SchemaOptionsFlowHandler`；设置 `options_flow_reloads = True` 时生成带自动 reload 的变体。这和手写 `OptionsFlow` 使用同一个 `OptionsFlowManager`，只是省去了重复的 step 代码。

OptionsFlow 的文案放在集成的 `strings.json` / `translations/*.json` 的 `options` 节点下，而不是 `config` 节点；例如 `options.step.init.data.poll_interval` 对应上面表单的字段。step ID、字段键和错误键必须与 Python 返回的 flow result 保持一致。

选项变更后的运行时处理有两种互斥的常见模式：

- 继承普通 `OptionsFlow`，在 `async_setup_entry` 中注册 `entry.add_update_listener()`，由监听器调用 `async_reload()` 或更新运行对象。监听器应通过 `entry.async_on_unload(...)` 注册清理。
- 继承 `OptionsFlowWithReload`，由 Core 在选项实际变化后自动 `async_schedule_reload(entry.entry_id)`。此模式不能同时使用 ConfigEntry update listener；源码会直接抛出 `ValueError`，避免重复 reload。

### 5.10.4 OptionsFlow 与 ConfigFlow 的区别

| 对比项 | ConfigFlow | OptionsFlow |
|---|---|---|
| 目标 | 首次建立一个集成配置，或处理发现/导入/重新认证/重新配置 | 修改已有条目的运行选项 |
| 管理器 | `hass.config_entries.flow` | `hass.config_entries.options` |
| `async_init()` 的 handler | 集成 `domain`，例如 `"hue"` | 已有 `ConfigEntry.entry_id` |
| 关联对象 | 流程可能尚未有 ConfigEntry | 必须关联一个已存在的 ConfigEntry |
| 常见 step | `user`、`bluetooth`、`dhcp`、`zeroconf`、`import`、`reauth`、`reconfigure` | 通常从 `init` 开始，也可拆成多个自定义 step |
| 保存位置 | `CREATE_ENTRY` 的 `data` / `options` 参与创建新条目 | `CREATE_ENTRY` 的 `data` 覆盖原条目的 `options` |
| 是否创建条目 | 是，Core 最终调用 `async_add()` | 否，调用 `async_update_entry(entry, options=...)` |
| 身份去重 | 常用 `unique_id` 和 `_abort_if_unique_id_configured()` | 通常不改变 `unique_id`，也不创建新的设备身份 |
| 失败/取消 | 可返回 `ABORT`，未创建时不产生条目 | 可返回 `ABORT`，原 `options` 保持不变 |
| reload 行为 | 由创建/更新条目及集成 setup 生命周期决定 | 普通 OptionsFlow 依赖 update listener；`OptionsFlowWithReload` 自动安排 reload |

两者共享 `FlowHandler` 的结果类型、schema 校验、翻译和多步骤机制，所以前端看起来相似；但它们的 handler key、持久化目标和完成动作不同，不能把 OptionsFlow 当作“再次运行一次 ConfigFlow”。

### 5.10.5 选项流的常见误区

- **把连接身份放进 options**：host、账号、token 等若决定连接对象，应放在 `data`，需要修改时使用 reconfigure/reauth flow。options 适合行为偏好，不适合作为条目身份。
- **在 `__init__` 里访问 `self.config_entry`**：此时 Manager 还没有设置 `hass` 和 `handler`，源码会抛出 `ValueError`。把读取逻辑放到 `async_step_init` 或后续 step。
- **直接改 `entry.options`**：ConfigEntry 的数据和 options 都由 Core 管理，直接赋值会触发属性保护。通过 `async_create_entry(data=...)`（OptionsFlow）或 `hass.config_entries.async_update_entry(..., options=...)`（其他运行时代码）更新。
- **OptionsFlowWithReload 与 update listener 同时使用**：这会触发 Core 的保护性错误。二选一，并确保 reload 不会重复执行。
- **只提交局部字段却覆盖旧选项**：OptionsFlowManager 不会自动做 patch merge；需要保留的旧键必须显式合并。
- **误以为 manifest 里的 `config_flow: true` 自动启用 OptionsFlow**：manifest 只声明集成有 ConfigFlow。是否显示选项入口取决于 `ConfigFlow.async_get_options_flow()` 是否被覆写并返回有效的 OptionsFlow。

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

`async_update_ha_state()` 仍为兼容入口，但当前 Core 会在实体首次调用时记录弃用/误用警告；推送型实体更新 `_attr_*` 后应直接调用 `async_write_ha_state()`，只有确实需要在写状态前执行设备刷新时才使用 `async_update_ha_state(force_refresh=True)`。

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

`SensorEntityDescription` 的扩展字段和使用方式在 6.4 节统一说明。

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

`EntityDescription` 是 HA 推荐的“描述对象”模式：把一类实体的静态元数据放到 description 里，把运行时值留在实体实例里。这样一个实体类可以通过不同 description 创建多个实体，避免为每个传感器/开关写一个子类。

源码中的基类定义在 `homeassistant/helpers/entity.py`：

```python
class EntityDescription(metaclass=FrozenOrThawed, frozen_or_thawed=True):
    """A class that describes Home Assistant entities."""

    key: str
    device_class: str | None = None
    entity_category: EntityCategory | None = None
    entity_registry_enabled_default: bool = True
    entity_registry_visible_default: bool = True
    force_update: bool = False
    icon: str | None = None
    has_entity_name: bool = False
    name: str | UndefinedType | None = UNDEFINED
    translation_key: str | None = None
    translation_placeholders: Mapping[str, str] | None = None
    unit_of_measurement: str | None = None
```

`Entity` 的很多属性都会按“`_attr_*` 优先，否则读 `entity_description.*`”的顺序解析。例如 `translation_key` 会先读 `_attr_translation_key`，没有时读 `entity_description.translation_key`；`entity_category`、`device_class`、`icon` 等也遵循类似模式。

| 字段 | 作用 |
|----|----|
| `key` | description 的内部标识，通常用于生成 `unique_id`、从 coordinator 数据中取值、区分同一实体类创建出的多个实体。它不是 HA 的 `entity_id`，也不是 ConfigEntry 的 `unique_id`。 |
| `device_class` | 设备类别。基类中是 `str | None`，具体平台会收窄类型，例如 sensor 使用 `SensorDeviceClass`，binary_sensor 使用 `BinarySensorDeviceClass`。它影响前端图标、单位换算、状态/统计校验、自动化条件等。 |
| `entity_category` | 实体类别，常见为 `diagnostic` 或 `config`。诊断类实体会在 UI 中弱化展示；部分平台禁止某些 category，例如 sensor/binary_sensor 不允许 `EntityCategory.CONFIG`。 |
| `entity_registry_enabled_default` | 实体第一次进入 entity registry 时默认是否启用。适合把很少用或高频更新的实体默认禁用。用户之后在 UI 中的启用/禁用选择优先。 |
| `entity_registry_visible_default` | 实体第一次进入 registry 时默认是否可见。用于控制默认 UI 可见性，不等同于 enabled。 |
| `force_update` | 状态值即使没有变化也强制写入状态机。常规实体应保持 `False`；只有需要记录重复相同事件/采样的实体才考虑开启。 |
| `icon` | 默认图标，如 `mdi:temperature-celsius`。如果同时使用 `device_class` 或 `icons.json`，前端可能根据平台规则选择更具体的图标。 |
| `has_entity_name` | 是否启用 HA 新命名模型。设为 `True` 时，实体名只描述实体自身，设备名由 device registry 拼接；新集成通常应使用 `True`。 |
| `name` | 静态实体名称。`UNDEFINED` 表示未显式设置，HA 可根据 `translation_key`、device class 等生成名称；`None` 表示实体本身无名，常用于主实体。 |
| `translation_key` | 翻译 key，用来定位 `translations/en.json` / `strings.json` 中的实体名称、状态、属性、单位等文案。不是唯一 ID，也不参与状态计算。 |
| `translation_placeholders` | 给实体名称翻译字符串中的 `{placeholder}` 填值。只用于文字格式化，不参与实体注册、唯一性或状态计算。 |
| `unit_of_measurement` | 基类通用单位字段，适合部分非 sensor 平台；`SensorEntityDescription` 明确覆盖为 `None`，sensor 应使用 `native_unit_of_measurement` 和 `suggested_unit_of_measurement`。 |

最常见用法如下：

```python
from homeassistant.components.sensor import (
    SensorDeviceClass,
    SensorEntity,
    SensorEntityDescription,
    SensorStateClass,
)

SENSOR_DESCRIPTION = SensorEntityDescription(
    key="temperature",
    translation_key="temperature",
    translation_placeholders={"room": "Living room"},
    device_class=SensorDeviceClass.TEMPERATURE,
    native_unit_of_measurement="°C",
    state_class=SensorStateClass.MEASUREMENT,
)


class MySensor(SensorEntity):
    _attr_has_entity_name = True

    def __init__(self, description: SensorEntityDescription) -> None:
        self.entity_description = description
        self._attr_unique_id = f"{description.key}_sensor"
        self._native_value = None

    @property
    def native_value(self):
        return self._native_value
```

#### translation_key 与 translation_placeholders

`EntityDescription.translation_key` 是“去翻译文件里找哪一组文案”的 key。HA 构造实体名称翻译 key 时，会拼成：

```text
component.<integration_domain>.entity.<platform_domain>.<translation_key>.name
```

例如平台 domain 是 `sensor`、集成 domain 是 `my_integration`、`translation_key="temperature"` 时，对应翻译文件：

```json
{
  "entity": {
    "sensor": {
      "temperature": {
        "name": "{room} temperature",
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

`translation_placeholders` 是给 `{room}` 这类占位符填值的映射。源码中的 `_substitute_name_placeholders()` 会对实体名称执行 `name.format(**self.translation_placeholders)`；如果翻译字符串需要 `{room}`，但实体没有提供对应 key，开发版/非 stable 场景会更严格，stable 中也会记录 warning。

常见写法有两种：

```python
# 写在描述对象上，适合多个同类实体复用同一实体类
SensorEntityDescription(
    key="battery",
    translation_key="battery",
    translation_placeholders={"device_name": "Door lock"},
)

# 写在实体类属性上，适合运行时动态决定
class MyBatterySensor(SensorEntity):
    _attr_has_entity_name = True
    _attr_translation_key = "battery"
    _attr_translation_placeholders = {"device_name": "Door lock"}
```

#### SensorEntityDescription 扩展字段

`sensor` 平台在 `homeassistant/components/sensor/__init__.py` 中扩展了 `EntityDescription`：

```python
class SensorEntityDescription(EntityDescription, frozen_or_thawed=True):
    """A class that describes sensor entities."""

    device_class: SensorDeviceClass | None = None
    last_reset: datetime | None = None
    native_unit_of_measurement: str | None = None
    options: list[str] | None = None
    state_class: SensorStateClass | None = None
    suggested_display_precision: int | None = None
    suggested_unit_of_measurement: str | None = None
    unit_of_measurement: None = None  # Type override, use native_unit_of_measurement
```

| 字段 | 作用 |
|----|----|
| `device_class` | 传感器设备类别，如 `TEMPERATURE`、`HUMIDITY`、`POWER`、`ENERGY`、`TIMESTAMP`、`ENUM` 等。它决定单位换算、图标、状态合法性、长期统计规则等。 |
| `native_unit_of_measurement` | 设备/API 原始返回值的单位，例如 `°C`、`W`、`kWh`、`%`。`SensorEntity.state` 会基于 device class 和用户单位系统做转换。 |
| `state_class` | 状态类别，常见为 `MEASUREMENT`、`TOTAL`、`TOTAL_INCREASING`。它告诉 recorder/statistics 如何处理长期统计；错误组合会触发校验 warning。 |
| `options` | 枚举传感器的可选状态列表。通常配合 `device_class=SensorDeviceClass.ENUM` 使用，HA 会把它作为 capability attribute 暴露。 |
| `last_reset` | 总量型传感器的重置时间。源码只允许 `state_class == SensorStateClass.TOTAL` 时使用；其他 state class 设置 `last_reset` 会抛错。新集成通常更常使用 `TOTAL_INCREASING` 而不是手写 `last_reset`。 |
| `suggested_display_precision` | 建议前端显示的小数位数。它影响显示精度，不应该用来改变底层 `native_value` 的实际精度。 |
| `suggested_unit_of_measurement` | 建议显示单位，会在实体首次进入 registry 时写入 entity options。适合覆盖自动单位系统推荐值，例如希望温度默认显示为 °C。用户在 UI 中修改单位后，以用户选择为准。 |
| `unit_of_measurement` | 在 sensor description 中被强制覆盖为 `None`；不要在 `SensorEntityDescription` 里写这个字段，应写 `native_unit_of_measurement`。 |

典型传感器 description：

```python
SensorEntityDescription(
    key="energy_today",
    translation_key="energy_today",
    device_class=SensorDeviceClass.ENERGY,
    native_unit_of_measurement=UnitOfEnergy.KILO_WATT_HOUR,
    state_class=SensorStateClass.TOTAL_INCREASING,
    suggested_display_precision=2,
)
```

如果是枚举传感器：

```python
SensorEntityDescription(
    key="mode",
    translation_key="mode",
    device_class=SensorDeviceClass.ENUM,
    options=["auto", "heat", "cool"],
)
```

对应翻译可写：

```json
{
  "entity": {
    "sensor": {
      "mode": {
        "name": "Mode",
        "state": {
          "auto": "Auto",
          "heat": "Heat",
          "cool": "Cool"
        }
      }
    }
  }
}
```

#### BinarySensorEntityDescription 扩展字段

`binary_sensor` 平台的 description 很简单，只收窄/扩展了 `device_class`：

```python
class BinarySensorEntityDescription(EntityDescription, frozen_or_thawed=True):
    """A class that describes binary sensor entities."""

    device_class: BinarySensorDeviceClass | None = None
```

`BinarySensorEntityDescription.device_class` 决定二元传感器的语义和前端展示，例如：

- `MOTION`：运动传感器，`is_on=True` 表示检测到运动。
- `DOOR` / `WINDOW` / `OPENING`：门窗/开口状态，`True` 通常表示打开。
- `BATTERY`：电池低电量，`True` 通常表示低电量告警。
- `CONNECTIVITY`：连接状态，`True` 表示已连接。
- `PROBLEM`：问题/故障状态，`True` 表示存在问题。
- `SAFETY`：安全状态，`True` 表示不安全或告警，具体语义按设备类翻译展示。

典型写法：

```python
from homeassistant.components.binary_sensor import (
    BinarySensorDeviceClass,
    BinarySensorEntity,
    BinarySensorEntityDescription,
)

BINARY_SENSORS = (
    BinarySensorEntityDescription(
        key="motion",
        translation_key="motion",
        device_class=BinarySensorDeviceClass.MOTION,
    ),
    BinarySensorEntityDescription(
        key="low_battery",
        translation_key="low_battery",
        device_class=BinarySensorDeviceClass.BATTERY,
        entity_category=EntityCategory.DIAGNOSTIC,
    ),
)


class MyBinarySensor(BinarySensorEntity):
    _attr_has_entity_name = True

    def __init__(self, description: BinarySensorEntityDescription) -> None:
        self.entity_description = description
        self._attr_unique_id = description.key

    @property
    def is_on(self) -> bool | None:
        return self._device_state.get(self.entity_description.key)
```

#### 自定义 EntityDescription 子类

很多内置集成会继续继承平台 description，加入“如何从数据对象取值”的函数或字段。例如 AirPatrol 增加 `data_field`，Sun 传感器增加 `value_fn`。这种模式可以让实体类完全通用：

```python
from collections.abc import Callable
from dataclasses import dataclass


@dataclass(frozen=True, kw_only=True)
class MySensorEntityDescription(SensorEntityDescription):
    value_fn: Callable[[dict[str, Any]], StateType]


SENSORS = (
    MySensorEntityDescription(
        key="temperature",
        translation_key="temperature",
        device_class=SensorDeviceClass.TEMPERATURE,
        native_unit_of_measurement="°C",
        state_class=SensorStateClass.MEASUREMENT,
        value_fn=lambda data: data["temperature"],
    ),
)


class MySensor(CoordinatorEntity, SensorEntity):
    entity_description: MySensorEntityDescription

    @property
    def native_value(self):
        return self.entity_description.value_fn(self.coordinator.data)
```

选择是否自定义 description 子类的原则：如果只是设置 HA 已有静态元数据，用 `SensorEntityDescription` / `BinarySensorEntityDescription` 即可；如果每个实体还需要携带“从 API 数据取哪个字段”“调用哪个方法”“是否支持该实体”等集成私有逻辑，就定义自己的 description 子类。

### 6.5 CoordinatorEntity — 协调器实体

`CoordinatorEntity` 是“实体订阅 coordinator 数据”的实体基类；真正负责拉取、缓存、调度、通知的是不同类型的 coordinator。HA 源码里常见三类：

1. `DataUpdateCoordinator`：最常用，适合定时从一个 API / 设备 / 网关拉取一份共享数据。
2. `TimestampDataUpdateCoordinator`：`DataUpdateCoordinator` 的子类，额外记录最后一次成功更新时间。
3. `PassiveBluetoothDataUpdateCoordinator`：蓝牙广播 listener 模式，适合固定实体集合。
4. `PassiveBluetoothProcessorCoordinator`：蓝牙广播 processor 模式，支持动态 description、动态实体、按实体分发和恢复。
5. `ActiveBluetoothDataUpdateCoordinator`：被动接收广播，并在必要时主动连接设备轮询。

#### DataUpdateCoordinator — 共享轮询数据

`DataUpdateCoordinator` 的定位是“一个数据源，多实体共享”。它管理：

- `update_interval` 定时刷新。
- `update_method` 或子类 `_async_update_data()` 拉取数据。
- `data` 缓存最新结果。
- `last_update_success` / `last_exception` 表示刷新状态。
- `async_config_entry_first_refresh()` 在 setup 阶段做首次刷新；失败时会转成 `ConfigEntryNotReady` 等合适行为。
- `async_request_refresh()` 手动请求刷新，内部有 debouncer，避免短时间重复刷新。
- `async_set_updated_data(data)` 由外部推送手动更新缓存并通知实体。
- `always_update=False` 时，只有新旧数据不相等才通知 listener；这要求数据对象有可靠的 `__eq__`。

典型写法：

```python
from datetime import timedelta
import logging

from homeassistant.helpers.update_coordinator import (
    CoordinatorEntity,
    DataUpdateCoordinator,
    UpdateFailed,
)

_LOGGER = logging.getLogger(__name__)


async def async_setup_entry(hass, entry, async_add_entities):
    api = entry.runtime_data.api

    async def async_fetch_data():
        try:
            return await api.async_get_status()
        except ApiError as err:
            raise UpdateFailed(f"Error fetching data: {err}") from err

    coordinator = DataUpdateCoordinator(
        hass,
        _LOGGER,
        config_entry=entry,
        name="my_integration status",
        update_method=async_fetch_data,
        update_interval=timedelta(seconds=30),
        always_update=False,
    )
    await coordinator.async_config_entry_first_refresh()

    async_add_entities(MyEntity(coordinator, key) for key in ("temperature", "humidity"))


class MyEntity(CoordinatorEntity, SensorEntity):
    _attr_has_entity_name = True

    def __init__(self, coordinator, key):
        super().__init__(coordinator)
        self._attr_unique_id = key
        self._key = key

    @property
    def native_value(self):
        return self.coordinator.data[self._key]
```

`CoordinatorEntity` 自身做的事情很少但很关键：

- `should_poll=False`：实体不再让 EntityPlatform 单独轮询。
- `async_added_to_hass()`：注册 coordinator listener。
- coordinator 刷新完成后调用实体 `_handle_coordinator_update()`。
- 默认 `_handle_coordinator_update()` 只执行 `async_write_ha_state()`。
- 默认 `available` 取 `coordinator.last_update_success`。
- 用户手动调用 `homeassistant.update_entity` 时，`CoordinatorEntity.async_update()` 会调用 `coordinator.async_request_refresh()`。

因此，多数实体只需要从 `self.coordinator.data` 读值，不要在 `native_value` / `is_on` property 里直接做 I/O。

#### TimestampDataUpdateCoordinator — 记录最后成功刷新时间

`TimestampDataUpdateCoordinator` 继承自 `DataUpdateCoordinator`，只额外做一件事：每次 refresh 结束后，如果 `last_update_success=True`，就把 `last_update_success_time` 设为当前 UTC 时间。源码里的 `_async_refresh_finished()` 逻辑非常简单：成功才更新时间戳，失败则保留上一次成功时间。

它适合天气预报、系统监控、外部数据源等场景：实体除了显示数据本身，还可能需要知道“这份数据最后一次成功更新是什么时候”。例如 AccuWeather 的 forecast coordinator、NWS 的 forecast coordinator、System Monitor coordinator 都使用这种类型。

可以直接实例化：

```python
from homeassistant.helpers.update_coordinator import TimestampDataUpdateCoordinator

coordinator_forecast = TimestampDataUpdateCoordinator(
    hass,
    _LOGGER,
    config_entry=entry,
    name="my_integration forecast",
    update_method=api.async_get_forecast,
    update_interval=timedelta(minutes=30),
)
await coordinator_forecast.async_config_entry_first_refresh()
```

也可以继承后实现 `_async_update_data()`：

```python
class MyForecastCoordinator(TimestampDataUpdateCoordinator[list[dict[str, Any]]]):
    def __init__(self, hass, entry, api):
        self.api = api
        super().__init__(
            hass,
            _LOGGER,
            config_entry=entry,
            name="my forecast",
            update_interval=timedelta(minutes=30),
        )

    async def _async_update_data(self):
        return await self.api.async_get_forecast()
```

实体仍然继承普通 `CoordinatorEntity`，只是在需要时读取：

```python
self.coordinator.last_update_success_time
```

如果不需要“最后成功更新时间”，普通 `DataUpdateCoordinator` 就足够。

#### PassiveBluetoothDataUpdateCoordinator — 被动 BLE 广播更新

BLE 设备不一定适合定时轮询。许多设备会周期性广播状态，HA 的蓝牙栈已经在后台接收 advertisement。`PassiveBluetoothDataUpdateCoordinator` 用于这种“只听广播”的模式：

- 构造时绑定 `address`、扫描 `mode`、是否要求 `connectable`。
- `async_start()` 内部注册蓝牙 callback，并注册 unavailable tracking。
- 收到匹配 address 的 BLE 事件后，设置 `_available=True` 并通知 listener。
- 设备消失时 `_async_handle_unavailable()` 设置 `_available=False` 并通知 listener。
- `PassiveBluetoothCoordinatorEntity` 继承 `BaseCoordinatorEntity`，`async_update()` 是 no-op，因为所有更新都来自广播。
- 实体 `available` 取 `coordinator.available`，而不是 `last_update_success`。

一个被动 BLE coordinator 的骨架：

```python
from homeassistant.components.bluetooth import (
    BluetoothChange,
    BluetoothScanningMode,
    BluetoothServiceInfoBleak,
)
from homeassistant.components.bluetooth.passive_update_coordinator import (
    PassiveBluetoothCoordinatorEntity,
    PassiveBluetoothDataUpdateCoordinator,
)
from homeassistant.core import callback


class MyBleCoordinator(PassiveBluetoothDataUpdateCoordinator):
    def __init__(self, hass, address):
        super().__init__(
            hass,
            _LOGGER,
            address=address,
            mode=BluetoothScanningMode.PASSIVE,
            connectable=False,
        )
        self.data = None

    @callback
    def _async_handle_bluetooth_event(
        self,
        service_info: BluetoothServiceInfoBleak,
        change: BluetoothChange,
    ) -> None:
        self.data = parse_advertisement(service_info)
        super()._async_handle_bluetooth_event(service_info, change)


class MyBleSensor(PassiveBluetoothCoordinatorEntity[MyBleCoordinator], SensorEntity):
    @property
    def native_value(self):
        return self.coordinator.data.temperature
```

使用时要在 config entry 生命周期里启动并登记停止回调：

```python
coordinator = MyBleCoordinator(hass, address)
entry.async_on_unload(coordinator.async_start())
entry.runtime_data = coordinator
```

这种模式适合温湿度计、门磁、按键等“广播里已经包含状态”的设备。如果实体集合固定、每次广播只是更新同一份 coordinator data，使用它最直接。

#### PassiveBluetoothProcessorCoordinator — 广播解析、动态描述与实体分发

`PassiveBluetoothProcessorCoordinator` 同样用于纯 BLE advertisement 驱动，但它解决的问题比 `PassiveBluetoothDataUpdateCoordinator` 更高一层：不仅通知“数据更新了”，还把**广播解析、DeviceInfo、EntityDescription、实体值、动态实体创建、按实体差异通知和恢复数据**组织成一条标准管道。Aranet、BTHome、Govee BLE、Mopeka、Qingping、RuuviTag 等内置集成使用这种模式。

它由四个主要部分组成：

```text
BluetoothServiceInfoBleak
  │
  ▼ coordinator.update_method
解析后的第三方库数据 _DataT
  │
  ▼ PassiveBluetoothDataProcessor.update_method
PassiveBluetoothDataUpdate
  ├── devices             # device_id -> DeviceInfo
  ├── entity_descriptions # PassiveBluetoothEntityKey -> EntityDescription
  ├── entity_names        # PassiveBluetoothEntityKey -> 名称/None
  └── entity_data         # PassiveBluetoothEntityKey -> 实体值
  │
  ├── 新 description → 动态创建实体
  └── 值/描述变化 → 只通知对应实体 listener
```

| 类型 | 职责 |
|----|----|
| `PassiveBluetoothProcessorCoordinator[_DataT]` | 按 address 订阅 BLE 广播，调用第一层 `update_method(service_info)` 把原始广播转换成第三方库对象或其他中间数据，然后分发给所有 processor；同时跟踪设备可用性、解析是否成功和恢复数据。 |
| `PassiveBluetoothDataProcessor[_T, _DataT]` | 针对一个 HA 平台（如 sensor、binary_sensor）把中间数据转换成 `PassiveBluetoothDataUpdate[_T]`，累积设备、实体描述和实体值，并管理 listener。一个 coordinator 可以注册多个 processor。 |
| `PassiveBluetoothDataUpdate[_T]` | 一次广播产生的增量更新容器。广播长度有限，所以每次只需包含本次实际更新的 device/entity；processor 会把多次广播结果累积起来。 |
| `PassiveBluetoothEntityKey` | 实体键，由 `key` 和可选 `device_id` 组成。同一广播源可代表多个子设备时，`device_id` 用于区分；`key` 通常对应 description key。 |
| `PassiveBluetoothProcessorEntity` | 与 processor 配套的实体基类，绑定一个 `entity_key`，从 processor 读取 description、DeviceInfo 和数据，并只订阅该实体 key 的变化。 |

与简单的 `PassiveBluetoothDataUpdateCoordinator` 相比，关键差异是：

- `PassiveBluetoothDataUpdateCoordinator` 只有 coordinator listener；实体通常是集成代码预先创建的固定集合，并自行读取 `coordinator.data`。
- `PassiveBluetoothProcessorCoordinator` 允许 parser 在运行时返回 `EntityDescription`。processor 看到新的 description 后，可以通过 `async_add_entities_listener()` 动态创建新实体。
- processor 按 `PassiveBluetoothEntityKey` 追踪变化，可以只通知真正变化的实体，而不是每次广播刷新全部实体。
- processor 数据会周期性保存，并在 reload/restart 后恢复 description、name、data 和 DeviceInfo，避免等待下一次广播才恢复已有实体结构。

典型用法分为集成入口和平台文件两部分。

**1. 在 `__init__.py` 创建并启动 coordinator**

```python
from homeassistant.components.bluetooth import BluetoothScanningMode
from homeassistant.components.bluetooth.models import BluetoothServiceInfoBleak
from homeassistant.components.bluetooth.passive_update_processor import (
    PassiveBluetoothProcessorCoordinator,
)


def parse_service_info(service_info: BluetoothServiceInfoBleak) -> AdvertisementData:
    return AdvertisementData(service_info.device, service_info.advertisement)


async def async_setup_entry(hass, entry) -> bool:
    address = entry.unique_id
    assert address is not None

    coordinator = PassiveBluetoothProcessorCoordinator(
        hass,
        _LOGGER,
        address=address,
        mode=BluetoothScanningMode.PASSIVE,
        update_method=parse_service_info,
        connectable=False,
    )
    entry.runtime_data = coordinator

    # 先 setup 平台，让 processor 有机会订阅；再启动接收广播。
    await hass.config_entries.async_forward_entry_setups(entry, PLATFORMS)
    entry.async_on_unload(coordinator.async_start())
    return True
```

这里的 `update_method` 是同步 callback：每次匹配广播到来时，它接收 `BluetoothServiceInfoBleak` 并返回解析后的 `_DataT`。如果解析抛出异常，coordinator 会设置 `last_update_success=False`；下一次成功解析后恢复。

**2. 在 `sensor.py` 把解析结果转成 HA 实体更新**

```python
from homeassistant.components.bluetooth.passive_update_processor import (
    PassiveBluetoothDataProcessor,
    PassiveBluetoothDataUpdate,
    PassiveBluetoothEntityKey,
    PassiveBluetoothProcessorEntity,
)

SENSOR_DESCRIPTIONS = {
    "temperature": SensorEntityDescription(
        key="temperature",
        translation_key="temperature",
        device_class=SensorDeviceClass.TEMPERATURE,
        native_unit_of_measurement=UnitOfTemperature.CELSIUS,
        state_class=SensorStateClass.MEASUREMENT,
    ),
    "humidity": SensorEntityDescription(
        key="humidity",
        translation_key="humidity",
        device_class=SensorDeviceClass.HUMIDITY,
        native_unit_of_measurement=PERCENTAGE,
        state_class=SensorStateClass.MEASUREMENT,
    ),
}


def sensor_update(advertisement: AdvertisementData) -> PassiveBluetoothDataUpdate:
    address = advertisement.device.address
    temperature_key = PassiveBluetoothEntityKey("temperature", address)
    humidity_key = PassiveBluetoothEntityKey("humidity", address)

    return PassiveBluetoothDataUpdate(
        devices={
            address: DeviceInfo(
                connections={(CONNECTION_BLUETOOTH, address)},
                name=advertisement.name,
            )
        },
        entity_descriptions={
            temperature_key: SENSOR_DESCRIPTIONS["temperature"],
            humidity_key: SENSOR_DESCRIPTIONS["humidity"],
        },
        entity_names={temperature_key: None, humidity_key: None},
        entity_data={
            temperature_key: advertisement.temperature,
            humidity_key: advertisement.humidity,
        },
    )


async def async_setup_entry(hass, entry, async_add_entities) -> None:
    processor = PassiveBluetoothDataProcessor(sensor_update)

    entry.async_on_unload(
        processor.async_add_entities_listener(MyBluetoothSensor, async_add_entities)
    )
    entry.async_on_unload(
        entry.runtime_data.async_register_processor(
            processor,
            SensorEntityDescription,
        )
    )


class MyBluetoothSensor(
    PassiveBluetoothProcessorEntity[PassiveBluetoothDataProcessor],
    SensorEntity,
):
    @property
    def native_value(self):
        return self.processor.entity_data.get(self.entity_key)
```

注册顺序应保持和内置集成一致：先给 processor 注册 `async_add_entities_listener()`，再用 coordinator 的 `async_register_processor()` 注册 processor；两个取消函数都交给 `entry.async_on_unload()`。传给 `async_register_processor()` 的 description class 用于恢复持久化后的 description，应该传实际平台 description 类型；自定义 description 子类则传该自定义类。

`PassiveBluetoothDataUpdate` 是**增量**而不是必须包含全量快照。例如某个广播包只包含温度，就只返回温度对应的 description/name/data；此前从其他广播累积的湿度数据仍会保留。源码会比较新旧 `devices`、`entity_descriptions`、`entity_names`、`entity_data`：

- DeviceInfo 改变时更新所有相关实体。
- 只有某些 `PassiveBluetoothEntityKey` 改变时，只通知这些 key 的 listener。
- 新出现的 entity description 会触发实体创建；已经创建的 key 不会重复创建。
- 设备 unavailable 时，processor 通知实体更新可用性。

此外，`coordinator.async_set_updated_data(update)` 可把 BLE notification 等其他路径得到的 `_DataT` 手动送入同一 processor 管道。注意这里传入的是 coordinator 第一层 `_DataT`，不是 `PassiveBluetoothDataUpdate`；后者由各 processor 的 `update_method` 生成。

选择这一模式的典型条件是：一个 BLE 广播源可能产生多个平台/多个子设备/动态测量项，第三方 parser 已经能给出结构化更新，或者希望复用 HA 对动态实体和恢复数据的通用处理。若实体种类固定且只有少量字段，简单的 `PassiveBluetoothDataUpdateCoordinator` 更容易理解，不必为了抽象而使用 processor 管道。

#### ActiveBluetoothDataUpdateCoordinator — 广播触发 + 必要时连接轮询

`ActiveBluetoothDataUpdateCoordinator` 继承 `PassiveBluetoothDataUpdateCoordinator`，适合“广播能提供部分状态，但某些状态需要连接设备读取”的 BLE 设备。SwitchBot、Casper Glow 等集成就是这种模式。

它新增两个核心回调：

- `needs_poll_method(service_info, seconds_since_last_poll)`：每次收到广播时判断是否需要主动 poll。
- `poll_method(service_info)`：真正连接设备读取数据的 coroutine。

源码会保存最近一次 `BluetoothServiceInfoBleak`，当 `needs_poll()` 返回 `True` 时通过 debouncer 调度 `_async_poll()`；poll 成功后更新 `data` 并通知 listener，失败时记录 `last_poll_successful=False`。

典型写法：

```python
from homeassistant.components.bluetooth.active_update_coordinator import (
    ActiveBluetoothDataUpdateCoordinator,
)


class MyActiveBleCoordinator(ActiveBluetoothDataUpdateCoordinator[MyData]):
    def __init__(self, hass, device, address):
        super().__init__(
            hass=hass,
            logger=_LOGGER,
            address=address,
            mode=BluetoothScanningMode.ACTIVE,
            needs_poll_method=self._needs_poll,
            poll_method=self._async_poll_device,
            connectable=True,
        )
        self.device = device

    @callback
    def _needs_poll(self, service_info, seconds_since_last_poll):
        return seconds_since_last_poll is None or seconds_since_last_poll > 60

    async def _async_poll_device(self, service_info):
        # service_info.device 里有 BLEDevice，优先用它建立连接
        return await self.device.async_read_full_state(service_info.device)

    @callback
    def _async_handle_bluetooth_event(self, service_info, change):
        if data := parse_advertisement(service_info):
            self.data = data
        super()._async_handle_bluetooth_event(service_info, change)
```

实体通常仍然继承 `PassiveBluetoothCoordinatorEntity`，因为可用性和 listener 模型仍来自蓝牙 coordinator。

#### 选择哪种 Coordinator

| 场景 | 推荐模式 |
|----|----|
| 多个实体共享同一个 HTTP/API/网关请求结果 | `DataUpdateCoordinator + CoordinatorEntity` |
| 需要记录最后一次成功刷新时间 | `TimestampDataUpdateCoordinator + CoordinatorEntity` |
| 数据来自 BLE advertisement，实体集合固定，只需统一通知更新 | `PassiveBluetoothDataUpdateCoordinator + PassiveBluetoothCoordinatorEntity` |
| BLE 广播要生成多个平台/子设备/动态实体，需按实体分发和恢复 | `PassiveBluetoothProcessorCoordinator + PassiveBluetoothDataProcessor + PassiveBluetoothProcessorEntity` |
| BLE advertisement 触发更新，但偶尔需要连接设备补全状态 | `ActiveBluetoothDataUpdateCoordinator + PassiveBluetoothCoordinatorEntity` |
| 单个简单实体、数据源轻量、不需要共享刷新状态 | 直接继承实体类，实现 `async_update()` 或推送回调即可 |

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
| `requirements` | Python 依赖包列表，HA 会按需安装/加载，例如 `aiohue==4.9.0`。运行时不要在代码里手动 `pip install`。 |
| `dependencies` | 强依赖集成。HA 会先加载这些集成；依赖失败通常会阻止当前集成正常 setup。 |
| `after_dependencies` | 弱依赖/加载顺序提示。存在时尽量在这些集成之后加载，但不会强制安装或强制启用。 |
| `loggers` | 集成相关第三方库 logger 名称；用于日志级别管理和诊断。 |
| `import_executor` | 是否在线程池中 import 集成模块。未显式设置时默认 `true`，避免同步 import 阻塞事件循环。 |
| `zeroconf` | mDNS/Zeroconf 发现 matcher。可以是服务类型字符串，也可以是带 `type`、`name`、`properties` 等条件的对象。匹配后进入 `async_step_zeroconf`。 |
| `ssdp` | SSDP/UPnP 发现 matcher。通常按响应头字段匹配；匹配后进入 `async_step_ssdp`。 |
| `dhcp` | DHCP 发现 matcher，可按 `macaddress`、`hostname`、`registered_devices` 等匹配；匹配后进入 `async_step_dhcp`。 |
| `usb` | USB 发现 matcher，可按 `vid`、`pid`、`serial_number`、`manufacturer`、`description` 等匹配；匹配后进入 `async_step_usb`。 |
| `bluetooth` | BLE 广播 matcher 列表。**列表中每个对象之间是 OR（或）关系；同一个对象里填写的字段之间是 AND（与）关系。** 任意一个对象完整匹配即可让该集成成为候选，并触发 Bluetooth discovery flow；随后才可能进入 `async_step_bluetooth`。 |
| `homekit` | HomeKit 发现声明，例如 `models`；用于 HomeKit 控制器发现路径。 |
| `mqtt` | MQTT discovery 相关声明；用于让 MQTT 集成知道哪些 discovery payload 可归属到该集成。 |
| `quality_scale` | 内置集成的质量等级标记；自定义集成在 loader 属性中会归类为 `custom`。 |
| `preview_features` | 预览功能元数据，可为每个 feature 提供反馈、了解更多、报错链接；需要配合翻译资源展示名称和说明。 |
| `disabled` | 禁用原因。一般由 core/维护流程使用，自定义集成通常不主动写。 |
| `is_built_in` / `overwrites_built_in` | loader 运行时附加的内部元数据，不是自定义集成作者应该手写的 manifest 字段。 |

#### bluetooth matcher：列表是 OR，对象内字段是 AND

`manifest.json` 的 `bluetooth` 是 matcher 对象列表。逻辑可以理解为：

```text
matcher_1 完整匹配
OR matcher_2 完整匹配
OR matcher_3 完整匹配
...
```

而每一个 matcher 对象内部是：

```text
connectable 条件
AND service_uuid 条件
AND service_data_uuid 条件
AND manufacturer_id 条件
AND manufacturer_data_start 条件
AND local_name 条件
```

只计算对象中实际出现的字段；未填写的字段不参与限制。源码中的 `BluetoothMatcherIndex.match()` 会收集可能匹配的 matcher，然后逐个调用 `ble_device_matches()`。后者依次检查对象内的每个字段，只要任一已声明条件不满足就立即返回 `False`；最终通过的任意 matcher 都会把所属 integration domain 加入 `matched_domains` 集合。因此结论是：**外层列表 OR，单个对象内部 AND。**

例如：

```json
{
  "bluetooth": [
    {
      "connectable": false,
      "manufacturer_id": 89,
      "manufacturer_data_start": [3],
      "service_uuid": "0000fee5-0000-1000-8000-00805f9b34fb"
    },
    {
      "connectable": false,
      "local_name": "MySensor-*"
    }
  ]
}
```

它表示：

```text
(
  manufacturer_data 包含 company ID 89
  AND company ID 89 对应的数据以字节 0x03 开头
  AND service UUID 列表包含 0000fee5-...
  AND 不要求广播必须来自可连接设备
)
OR
(
  local name 匹配 MySensor-*
  AND 不要求广播必须来自可连接设备
)
```

各字段的匹配规则：

| matcher 字段 | 源码匹配方式 |
|----|----|
| `service_uuid` | 指定 UUID 必须存在于 `service_info.service_uuids`。UUID 应使用规范化的小写完整形式。 |
| `service_data_uuid` | 指定 UUID 必须是 `service_info.service_data` 的一个 key，即广播中包含该 Service Data。 |
| `manufacturer_id` | 指定 Company Identifier 必须是 `service_info.manufacturer_data` 的一个 key。 |
| `manufacturer_data_start` | 只有配合 `manufacturer_id` 才有意义；对应 manufacturer data 必须以这些十进制字节开头。源码使用 `bytes(list)` 和 `startswith()` 比较。 |
| `local_name` | 对 `service_info.name` 做 fnmatch 匹配，可使用 `*` 等模式，例如 `MySensor-*`。为避免过宽匹配，源码限制名称模式开头的若干字符不能出现通配符。 |
| `connectable` | 默认值相当于 `true`：要求 `service_info.connectable=True`。写成 `false` 的含义是**取消“必须可连接”的限制**，因此既可以匹配不可连接广播，也可以匹配可连接广播；它不是“只匹配不可连接设备”。 |

几个常见写法：

```json
{
  "bluetooth": [
    {"service_uuid": "0000abcd-0000-1000-8000-00805f9b34fb"},
    {"manufacturer_id": 1234}
  ]
}
```

这里是“有该 Service UUID **或**有 manufacturer ID 1234”。如果本意是两个条件必须同时满足，应该写在同一个对象：

```json
{
  "bluetooth": [
    {
      "service_uuid": "0000abcd-0000-1000-8000-00805f9b34fb",
      "manufacturer_id": 1234
    }
  ]
}
```

如果同一厂商 ID 下有多个协议版本/型号前缀，则写多个 matcher，表示多个完整条件分支，例如 HA 的 Mopeka 集成对 `[3]`、`[4]`、`[5]` 等不同 manufacturer data 起始字节分别写一个对象。不要为了“多写几个线索”把互斥型号条件塞进同一个对象，否则对象内 AND 会导致永远无法匹配。

另外，matcher 成功只表示该广播归属于该 integration domain。Bluetooth manager 随后调用 discovery flow，并且还会经过已见地址、进行中 flow、已有 config entry 等去重机制；因此不能把“一个 matcher 成功”理解为每个广播包都会新建 `ConfigFlow` 或反复调用 `async_step_bluetooth`。

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

#### translation_key 与各种 placeholders

HA 源码里经常同时出现 `translation_key` 和 `*_placeholders`，它们本质上是同一个翻译机制的两部分：

- `translation_key`：选择翻译资源中的哪一条模板字符串。
- `*_placeholders`：给模板字符串里的 `{name}`、`{host}`、`{url}` 等变量填值。

不同子系统会把 key 拼到不同 category 下：

| 使用位置 | key 如何映射到翻译文件 | placeholders 参数 |
|----|----|----|
| Entity / EntityDescription | `component.<integration>.entity.<platform>.<translation_key>.name`、`.state.<state>`、`.unit_of_measurement` | `_attr_translation_placeholders` 或 `EntityDescription.translation_placeholders`，主要替换实体名称中的变量。 |
| ConfigFlow 表单 | `config.step.<step_id>.description`、`config.abort.<reason>`、`config.flow_title` 等 | `description_placeholders`、`title_placeholders`。 |
| ConfigFlow 创建条目 | `config.create_entry.<key>` 或 create-entry description | `async_create_entry(description_placeholders=...)`。 |
| AbortFlow / `_abort_if_unique_id_configured()` | `config.abort.<reason>` | `description_placeholders`。 |
| Repairs / issue registry | `issues.<issue_key>.title`、`issues.<issue_key>.description` | `translation_placeholders`。 |
| 可翻译异常 | `exceptions.<translation_key>.message` | `translation_placeholders`。 |
| Selector | `selector.<translation_key>...` | 通常由 selector 配置的 `translation_key` 决定选项/字段文案。 |
| ServiceCall 描述 | `services.<service>.description` 可结合 runtime `description_placeholders` | 服务注册时传入的 `description_placeholders` 会随服务描述返回给前端。 |

以 ConfigFlow 为例，`description_placeholders` 不会改变 step id，也不会改变错误 key；它只给当前 step 的说明文案填变量：

```python
return self.async_show_form(
    step_id="bluetooth_confirm",
    description_placeholders={"name": discovery_info.name or discovery_info.address},
)
```

对应翻译：

```json
{
  "config": {
    "step": {
      "bluetooth_confirm": {
        "description": "Do you want to add {name}?"
      }
    }
  }
}
```

`title_placeholders` 则保存在 flow context 里，常用于自动发现流程的标题。源码中 `ConfigFlow.async_update_title_placeholders()` 会更新 `self.context["title_placeholders"]` 并通知前端刷新发现通知标题；如果马上要返回新 step，直接设置 `self.context["title_placeholders"]` 即可：

```python
self.context["title_placeholders"] = {"name": discovery_info.name}
return self.async_show_form(step_id="user")
```

对应翻译：

```json
{
  "config": {
    "flow_title": "{name}",
    "step": {
      "user": {
        "title": "Set up {name}"
      }
    }
  }
}
```

可翻译异常和 Repairs 使用的是更显式的 `translation_key` + `translation_placeholders`。例如自定义服务校验失败时，可以抛出可翻译异常：

```python
from homeassistant.exceptions import ServiceValidationError

raise ServiceValidationError(
    translation_domain=DOMAIN,
    translation_key="invalid_channel",
    translation_placeholders={"channel": channel},
)
```

对应翻译：

```json
{
  "exceptions": {
    "invalid_channel": {
      "message": "Channel {channel} is invalid"
    }
  }
}
```

源码中的 `async_get_exception_message()` 会查 `component.<domain>.exceptions.<translation_key>.message`，再执行 `message.format(**translation_placeholders)`；找不到翻译时会回退返回 `translation_key` 本身。

几个容易踩坑的点：

- `translation_key` 要稳定、语义化、可复用；不要把设备名、host、MAC、序列号这类动态值写进 key，动态值放到 placeholders。
- placeholders 的 key 必须和英文翻译字符串里的 `{placeholder}` 完全一致；其他语言也必须保留同一组 placeholder。
- placeholders 只用于文字格式化，不是安全边界，也不是校验逻辑；服务参数、配置输入仍要用 Python schema 校验。
- 不要把 HTML 拼进翻译字符串；URL、设备名、集成名等动态内容通过 placeholders 传入。
- `description_placeholders`、`title_placeholders`、`translation_placeholders` 名字不同，是因为它们挂在不同 flow/result/registry 对象上；作用都是给翻译模板填变量。

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

要启用 UI 配置流，`manifest.json` 中必须声明 `"config_flow": true`，并在集成目录下提供 `config_flow.py`。自定义集成的 `ConfigFlow` 只负责“收集输入、校验、去重、返回 flow result”；它不直接实例化 `ConfigEntry`。Flow 中的 `description_placeholders` / `title_placeholders` 只负责给翻译文案填变量，例如自动发现时把设备名 `{name}` 填进确认页标题或说明，不参与去重或条目创建。

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
from datetime import timedelta
import logging

from homeassistant.components.sensor import SensorEntity, SensorEntityDescription
from homeassistant.config_entries import ConfigEntry
from homeassistant.core import HomeAssistant
from homeassistant.helpers.entity_platform import AddEntitiesCallback
from homeassistant.helpers.update_coordinator import (
    CoordinatorEntity,
    DataUpdateCoordinator,
)

from .const import DOMAIN

_LOGGER = logging.getLogger(__name__)


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
        _LOGGER,
        config_entry=config_entry,
        name="my_integration sensors",
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

## 12. MQTT 集成：异步客户端与自动发现

**核心源码**：

- `homeassistant/components/mqtt/__init__.py`：Config Entry 初始化、协议迁移、平台预加载
- `homeassistant/components/mqtt/async_client.py`：Paho Client 子类与无锁适配
- `homeassistant/components/mqtt/client.py`：asyncio socket 驱动、订阅路由、ACK Future、重连
- `homeassistant/components/mqtt/discovery.py`：实体 discovery 与集成 config flow discovery
- `homeassistant/components/mqtt/entity.py`：discovery signal 到 MQTT Entity 的桥梁

当前 MQTT 集成使用 `paho-mqtt==2.1.0` 和 Callback API v2，新建连接默认 MQTT 5。旧条目启动时会尝试迁移到 MQTT 5；如果 broker 不支持，当前版本仍保留旧协议并创建 repair issue，这段兼容逻辑计划在 2027.1 移除。

MQTT 的 manifest 还声明了 `single_config_entry: true`：broker 连接全局只有一个 Config Entry，设备级配置可以作为该 entry 的 ConfigSubentry 管理。不要为每个 MQTT 设备再创建一个 broker Config Entry；实体 discovery 和 subentry 导入都会复用同一个客户端。

### 12.1 MQTT 集成的分层与启动顺序

源码里有三个名字相近、职责不同的对象：

| 层 | 对象 | 职责 |
|----|------|------|
| Paho 层 | `AsyncMQTTClient(Client)` | 仍是 Paho Client，只是在单事件循环模型下替换内部线程锁，并声明 Callback API v2 类型 |
| 构造层 | `MqttClientSetup` | 选择协议/transport，配置认证、TLS、WebSocket、will；因证书和文件访问而在线程池运行 |
| HA 适配层 | `MQTT` | 将 socket 注册给 asyncio，管理连接/重连、订阅、消息分发以及 publish/subscribe ACK |

`async_setup_entry()` 的关键顺序是：

```
读取 entry.data | entry.options
  ↓
MqttClientSetup.setup() 在线程池构造 Paho Client
  ↓
预加载配置中已经用到的 MQTT entity platforms
  ↓
MQTT.async_connect() 在线程池执行阻塞 connect
  ↓
转发已有平台的 config entry setup
  ↓
discovery.async_start() 注册 discovery wildcard subscriptions
  ↓
收到 retained discovery 消息后，按需加载尚未加载的平台
```

预加载发生在连接 broker 之前，是为了避免连接瞬间涌入大量 retained discovery 消息时，每条消息都等待平台 import，造成任务洪峰。

### 12.2 把同步 Paho 客户端接入 asyncio

这里最重要的事实是：`AsyncMQTTClient` 这个类名并不表示 Paho API 自动变成了 coroutine。真正的异步化由 Paho 的 external event loop hooks 和 `MQTT` 包装器共同完成。正式 MQTT Config Entry 运行期没有调用 `loop_start()` 创建 Paho 网络线程；只有配置流里的临时 `try_connection()` 使用 Paho 的线程循环做连通性测试。

Paho 把网络循环拆成三部分：

- `loop_read()`：socket 可读时解析入站报文并触发 `on_message`、`on_connect` 等回调。
- `loop_write()`：socket 可写时刷新 Paho 的发送缓冲区。
- `loop_misc()`：周期处理 keepalive、PING 和超时，不能因为没有网络流量就省略。

HA 通过 socket hooks 把它们挂到 asyncio：

```python
def _async_on_socket_open(self, client, userdata, sock):
    self.loop.add_reader(sock, self._async_reader_callback, client)
    self._start_misc_timer()
    self._async_reader_callback(client)  # 立即消费已有数据

def _async_reader_callback(self, client):
    status = client.loop_read(MAX_PACKETS_TO_READ)
    if status != mqtt.MQTT_ERR_SUCCESS:
        self._handle_error(status)

def _async_on_socket_register_write(self, client, userdata, sock):
    self.loop.add_writer(sock, self._async_writer_callback, client)

def _async_writer_callback(self, client):
    status = client.loop_write()
    if status != mqtt.MQTT_ERR_SUCCESS:
        self._handle_error(status)

def _async_on_socket_unregister_write(self, client, userdata, sock):
    self.loop.remove_writer(sock)

def _async_on_socket_close(self, client, userdata, sock):
    self.loop.remove_reader(sock)
    self._misc_timer.cancel()
```

`loop_misc()` 则由 `loop.call_at(loop.time() + 1, callback)` 每秒调度一次。使用 `call_at` 而不是一个永久 `while asyncio.sleep(1)` 任务，可以直接用 `TimerHandle.cancel()` 清理，并避免为简单定时工作常驻一个 Task。

连接是一个容易遗漏的线程边界。`Client.connect()` 会执行 DNS、TCP/TLS 握手等阻塞操作，所以 HA 使用：

```python
connect = partial(client.connect, host=host, port=port, keepalive=keepalive)
async with connection_lock:
    result = await hass.async_add_executor_job(connect)
```

但 `connect()` 在线程池里可能同步触发 `on_socket_open` 和 `on_socket_register_write`。线程池线程不能直接调用 `loop.add_reader()`，因此 HA 在连接期间临时安装线程安全桥接回调：

```python
def _on_socket_open(self, client, userdata, sock):
    self.loop.call_soon_threadsafe(
        self._async_on_socket_open, client, userdata, sock
    )
```

连接 executor job 结束后，再把 hook 切回直接运行的 `_async_on_socket_open` / `_async_on_socket_register_write`，减少每次写事件多一次跨线程投递。重连也受同一个 `asyncio.Lock` 保护，避免 connect、reconnect 和 disconnect 交叠。

`async_client.py` 还把 Paho 的 7 个内部 mutex 替换成 `NullLock`。这只是性能优化，成立的前提是运行期的网络处理和公开操作都由同一个事件循环串行化，只有受 `connection_lock` 保护的 connect/reconnect executor 边界例外，且没有 `loop_start()` 网络线程。把这段代码复制到一个仍会跨线程调用 `publish()` 的项目里会制造数据竞争；通用封装应先完成线程归一化，再考虑去锁。

### 12.3 将 publish/subscribe 的 MID 回调变成 await

Paho 的 `publish()`、`subscribe()` 和 `unsubscribe()` 本身立即返回，不代表 broker 已确认操作。HA 用 MQTT message id（MID）把回调式完成通知转换成 Future：

```python
def _async_get_mid_future(self, mid: int) -> asyncio.Future[None]:
    if future := self._pending_operations.get(mid):
        return future
    future = self.hass.loop.create_future()
    self._pending_operations[mid] = future
    return future

def _async_mqtt_on_callback(self, mid: int) -> None:
    future = self._async_get_mid_future(mid)
    if not future.done():
        future.set_result(None)

async def async_publish(self, topic, payload, qos, retain):
    info = self._mqttc.publish(topic, payload, qos, retain)
    await self._async_wait_for_mid_or_raise(info.mid, info.rc)
```

回调可能比 coroutine 创建 Future 更早发生，所以“等待方”和“回调方”都必须调用同一个幂等的 `_async_get_mid_future()`；谁先到都创建同一份 Future。这是包装 callback API 时很实用的竞态处理模式。当前实现对 Paho 的立即错误码抛出 `HomeAssistantError`，ACK 超时则记录警告并清理 MID。

订阅层还做了几件 Paho 原生 Client 不负责的事：

- 相同 topic 的多个 HA 订阅合并成一次 broker subscription，使用请求中的最大 QoS。
- 精确 topic 用字典 O(1) 查找，通配 topic 用 `MQTTMatcher` 匹配。
- MQTT 5 下给通配订阅分配 Subscription Identifier，避免重叠订阅导致同一报文被错误路由。
- 短时间内的 subscribe/unsubscribe 经 cooldown 和 debouncer 合并，降低启动期报文数量。
- `on_message` 把 Paho `MQTTMessage` 转成不可变的 `ReceiveMessage`，再按 `HassJobType` 直接执行 `@callback` 或调度 coroutine。

`await mqtt.async_subscribe(...)` 的语义是“把订阅加入 HA 的跟踪集合并返回取消函数”，不保证此刻已经收到 broker 的 SUBACK；实际 SUBSCRIBE 会由 debouncer 批量提交。确实需要确认订阅完成时，使用 `mqtt.async_on_subscribe_done()` 注册状态回调，而不要把 `await async_subscribe()` 当成网络 ACK。

自定义集成不应访问 `hass.data[DATA_MQTT]`、`async_subscribe_internal()` 或 `_mqttc`，应使用 MQTT 集成导出的公共 API：

```python
from homeassistant.components import mqtt
from homeassistant.core import callback

@callback
def message_received(msg: mqtt.ReceiveMessage) -> None:
    coordinator.async_set_updated_data(parse_payload(msg.payload))

unsubscribe = await mqtt.async_subscribe(
    hass, "my_device/+/state", message_received, qos=1
)
entry.async_on_unload(unsubscribe)

await mqtt.async_publish(
    hass,
    "my_device/node-1/command",
    "ON",
    qos=1,
    retain=False,
)
```

### 12.4 MQTT 实体自动发现

默认 discovery prefix 是 `homeassistant`，可以在 MQTT options 中修改。Core 同时支持两种实体发现格式。

**单组件 discovery topic**：

```
<prefix>/<component>/<object_id>/config
<prefix>/<component>/<node_id>/<object_id>/config
```

例如 `homeassistant/sensor/kitchen/temperature/config`。`component` 必须在 `SUPPORTED_COMPONENTS` 中，`node_id` 和 `object_id` 只允许字母、数字、下划线和连字符。`node_id` 只参与 discovery identity，不会自动成为设备标识。

**设备级 discovery topic**：

```
<prefix>/device/<object_id>/config
<prefix>/device/<node_id>/<object_id>/config
```

设备级 payload 必须包含：

- `device`：至少有 `identifiers` 或 `connections` 之一。
- `origin`：发布 discovery 的应用名称，可附带版本和支持 URL。
- `components`：一个或多个组件配置；每个组件必须有 `platform`，实体平台还必须有 `unique_id`。

设备级格式把公共 availability、state/command topic、QoS、encoding 等选项继承给各组件，组件自己的值优先。源码也接受 `dev`、`o`、`cmps` 等缩写，并在 schema 校验前展开。

收到 discovery 消息后的完整链路是：

```
Paho socket readable
  → MQTT._async_mqtt_on_message()
  → discovery.async_discovery_message_received()
  → 解析 topic / JSON / 缩写 / '~' base topic / schema
  → discovery_hash = (component, node_id + object_id)
  ├─ 新发现：MQTT_DISCOVERY_NEW
  │    → 平台 async_setup_entry 注册的 dispatcher listener
  │    → discovery schema 校验
  │    → async_add_entities([MqttEntity(...)])
  ├─ 已存在：MQTT_DISCOVERY_UPDATED
  │    → 已有实体重建订阅和配置并写入新状态
  └─ 空 payload：发送删除更新并清理实体/设备
```

如果消息对应的平台尚未加载，`discovery.py` 先通过 `async_forward_entry_setup_and_setup_discovery()` 加载平台，再发 `MQTT_DISCOVERY_NEW`。每个平台有独立的 `asyncio.Lock`，同一平台不会被并发加载多次。

同一 `discovery_hash` 的更新也不会并发处理。Core 在 `discovery_pending_discovered` 中为它维护 deque，只有实体发回 `MQTT_DISCOVERY_DONE` 后才取下一条。这避免 retained 初始配置、在线更新和删除消息交错，导致旧配置覆盖新配置。

设备应把 discovery 配置以 retained 消息发布，否则 HA 重启或 MQTT 重连后无法重新发现。删除实体或整个设备时，向原 discovery topic 发布**零长度的 retained payload**。单组件路径收到 `{}` 也会被当作空配置，但设备级 payload 的 `{}` 会因为缺少 `device`、`origin`、`components` 而校验失败；跨格式实现不要依赖 `{}`，统一发布零长度消息。状态是否 retained 则取决于设备语义，不能照搬 discovery 的规则。

### 12.5 manifest 的 MQTT 配置流发现

另一套容易混淆的机制是集成 manifest 中的 `"mqtt"` 字段：

```json
{
  "domain": "tasmota",
  "dependencies": ["mqtt"],
  "mqtt": ["tasmota/discovery/#"],
  "config_flow": true
}
```

它不是创建通用 MQTT Entity 的 discovery schema，而是“某个 MQTT topic 出现消息后，启动指定集成的 ConfigFlow”。内置 manifest 数据由 hassfest 生成到 `homeassistant/generated/mqtt.py`，自定义集成则由 loader 动态补入。

MQTT 集成为每个 matcher 订阅 topic。消息到达后构造：

```python
MqttServiceInfo(
    topic=msg.topic,
    payload=msg.payload,
    qos=msg.qos,
    retain=msg.retain,
    subscribed_topic=msg.subscribed_topic,
    timestamp=msg.timestamp,
)
```

随后用 `source=SOURCE_MQTT` 和以实际 topic 为 key 的 `DiscoveryKey` 调用 `discovery_flow.async_create_flow()`，最终进入目标集成的 `async_step_mqtt()`：

```python
async def async_step_mqtt(
    self, discovery_info: MqttServiceInfo
) -> ConfigFlowResult:
    await self.async_set_unique_id(stable_device_id)
    self._abort_if_unique_id_configured()
    # 校验 payload，保存发现信息，再进入 confirm 或直接创建 entry
```

Core 会缓存每个实际 topic 最近处理过的 payload，完全相同的消息不会重复启动 flow；删除由该 discovery key 创建的 Config Entry 后，会用缓存消息重新触发发现。空 payload 会清掉缓存。由于大量 retained 消息可能同时到达，这条路径还用一个锁串行启动 config flows，但这个锁是启动洪峰保护，不代替 `unique_id` 去重。

两套发现机制的区别可以概括为：

| 机制 | 输入 | 结果 |
|------|------|------|
| `homeassistant/.../config` | 标准 MQTT discovery JSON | MQTT 集成直接创建/更新实体、设备、tag 或 device automation |
| manifest `"mqtt": [...]` | 集成私有 topic 和私有 payload | 启动目标集成 `ConfigFlow.async_step_mqtt()`，通常创建该集成自己的 Config Entry |

### 12.6 设备侧 discovery 示例与工程约束

下面使用设备级 discovery 一次声明温度传感器和继电器。发布 topic 为 `homeassistant/device/env-node-01/config`，消息必须 retained：

```json
{
  "origin": {
    "name": "env-node-firmware",
    "sw_version": "1.2.0",
    "support_url": "https://example.com/env-node/support"
  },
  "device": {
    "identifiers": ["env-node-01"],
    "name": "Environment Node 01",
    "manufacturer": "Example Labs",
    "model": "EN-2"
  },
  "availability_topic": "env-node/01/status",
  "components": {
    "temperature": {
      "platform": "sensor",
      "unique_id": "env-node-01-temperature",
      "name": "Temperature",
      "device_class": "temperature",
      "state_class": "measurement",
      "unit_of_measurement": "°C",
      "state_topic": "env-node/01/state",
      "value_template": "{{ value_json.temperature }}"
    },
    "relay": {
      "platform": "switch",
      "unique_id": "env-node-01-relay",
      "name": "Relay",
      "state_topic": "env-node/01/relay/state",
      "command_topic": "env-node/01/relay/set",
      "payload_on": "ON",
      "payload_off": "OFF"
    }
  }
}
```

设备实现时还应满足以下约束：

1. `device.identifiers` 和每个实体的 `unique_id` 必须跨重启稳定，不能使用 IP、启动时间或随机数。
2. discovery topic 和 identity 一旦发布就应保持稳定；单组件与设备级格式之间迁移时使用 `migrate_discovery` 流程，普通改 topic 则先清理旧 retained topic，否则会留下重复实体。
3. 用 MQTT will 把 availability topic 设为 `offline`，连接成功后发布 `online`；这与 HA 自己的 `homeassistant/status` birth/will 不是同一个方向。
4. 配置更新继续向同一 discovery topic 发布完整 retained payload。要删除设备级组件，发布该组件的空配置或对整个设备发布零长度 retained payload，让 Core 生成清理更新。
5. 不要假定 QoS 1 等于业务动作“恰好一次”。MQTT QoS 1 是至少一次，命令如果不可重复执行，应在应用 payload 中加入 command id 并由设备去重。
6. 大 payload 和大量 retained 配置会在 HA 连接时同时到达。优先使用设备级 discovery 合并配置，并避免在高频 state topic 中携带无关大字段。

---

## 13. 关键设计模式总结

### 13.1 架构模式

| 模式 | 实现 |
|------|------|
| **事件驱动** | EventBus + StateMachine，所有组件间通信通过事件 |
| **分层架构** | 入口层 → 核心层 → 加载层 → 实体层 → 集成层 |
| **泛型约束** | `EntityComponent[LightEntity]`, `ConfigEntry[HueBridge]` |
| **注册表模式** | ServiceRegistry, EntityRegistry, DeviceRegistry, HANDLERS 等 |
| **观察者模式** | EventBus 监听器, Coordinator 订阅, ConfigEntry 更新监听 |

### 13.2 加载模式

| 模式 | 实现 |
|------|------|
| **分阶段启动** | Stage 0/1/2，基础设施先行 |
| **并行加载** | `asyncio.gather` 并行设置 ConfigEntry 和平台 |
| **依赖注入** | `hass.data` 全局字典 + `DATA_INSTANCES` |
| **惰性加载** | 自定义集成按需解析，平台模块按需导入 |
| **线性递增退避重试** | `PlatformNotReady` → `min(tries,6)*30` 秒递增等待（30, 60, 90, ... 180s） |

### 13.3 实体模式

| 模式 | 实现 | 典型集成 |
|------|------|----------|
| **轮询式（纯 Entity）** | `should_poll=True` + `async_update()` | `moon` |
| **协调器式** | `DataUpdateCoordinator` + `CoordinatorEntity` | `hue` (v1) |
| **推送式（纯 Entity）** | `should_poll=False` + dispatcher/事件监听 + `async_write_ha_state()` | `sun` |
| **描述模式** | `EntityDescription` + `value_fn` 将属性从子类移到描述对象 | `sun` |
| **_attr_* 模式** | 类属性默认值，减少 property 定义 | `moon` |

### 13.4 集成模式

| 模式 | 实现 |
|------|------|
| **runtime_data 模式** | `entry.runtime_data` 存储运行时对象，类型安全 |
| **平台转发模式** | `async_forward_entry_setups` 将设置传播到各平台 |
| **配置流模式** | `ConfigFlow` + 步骤方法，支持 UI 配置 |
| **MQTT 外部事件循环适配** | `add_reader/add_writer` 驱动 Paho `loop_read/loop_write`，MID callback 转 Future |
| **蓝图模式** | `!input` 占位符 + `substitute()` 递归替换，配置模板化复用 |
| **统一错误处理** | `async_request_call` 包装 API 调用，统一异常转换 |
| **选项 = 重载** | 选项变更触发 `async_reload`，重新初始化整个集成 |

### 13.5 @final 保护

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
| `homeassistant/core.py` | 3011 | HomeAssistant 主类、EventBus、StateMachine、ServiceRegistry |
| `homeassistant/config_entries.py` | 4221 | ConfigEntry、ConfigSubentry、ConfigFlow、ConfigEntries 管理器 |
| `homeassistant/bootstrap.py` | 1113 | 启动引导 |
| `homeassistant/loader.py` | 1804 | Integration 类、manifest 解析与 MQTT matcher 加载 |
| `homeassistant/setup.py` | 841 | async_setup_component |
| `homeassistant/data_entry_flow.py` | 939 | FlowHandler/FlowManager 基类 |
| `homeassistant/helpers/entity.py` | 1788 | Entity 基类 |
| `homeassistant/helpers/entity_platform.py` | 1353 | EntityPlatform、subentry 与 child device 注册 |
| `homeassistant/helpers/entity_component.py` | 399 | EntityComponent |
| `homeassistant/helpers/service.py` | 1416 | 服务辅助函数 |
| `homeassistant/const.py` | 1076 | 全局常量 |
| `homeassistant/components/mqtt/__init__.py` | 735 | MQTT Config Entry 设置、协议迁移、平台预加载 |
| `homeassistant/components/mqtt/client.py` | 1538 | Paho asyncio 适配、连接、订阅与消息分发 |
| `homeassistant/components/mqtt/async_client.py` | 71 | Paho Client 无锁子类 |
| `homeassistant/components/mqtt/discovery.py` | 733 | MQTT 实体发现和 manifest MQTT ConfigFlow 发现 |
| `homeassistant/components/mqtt/entity.py` | 1801 | MQTT Entity 基类与 discovery 更新 |
| `homeassistant/components/blueprint/models.py` | 385 | Blueprint、BlueprintInputs、DomainBlueprints |
| `homeassistant/components/blueprint/importer.py` | 288 | 蓝图 URL 导入逻辑（论坛/GitHub/Gist/官网/通用） |
| `homeassistant/components/blueprint/schemas.py` | 151 | BLUEPRINT_SCHEMA、BLUEPRINT_INSTANCE_FIELDS |
| `homeassistant/components/blueprint/websocket_api.py` | 274 | 蓝图 WebSocket API（导入/保存/删除/替换） |
| `homeassistant/components/automation/config.py` | 338 | 自动化配置校验（含蓝图替换流程） |
| `homeassistant/components/automation/helpers.py` | 40 | 自动化蓝图 DomainBlueprints 注册 |
| `annotatedyaml/input.py` | ~60 | `Input` 类、`substitute()`、`extract_inputs()` |
| `homeassistant/runner.py` | 330 | 运行器 |
