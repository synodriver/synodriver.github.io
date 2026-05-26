---
date: '2026-05-23T22:43:15+08:00'
draft: false
title: 'ESPHome 架构与设计思路深度解析'
author: 'synodriver'
tags: ["python", "c++", "esphome", "iot"]
---

# ESPHome 架构与设计思路深度解析

## 概述

**ESPHome** 是一个通过 YAML 配置自动生成 C++ 固件的 IoT 框架，专为 ESP32/ESP8266 等 MCU 设计，与 Home Assistant 深度集成。其核心思想是：**用户编写 YAML 配置 → Python 代码生成器产出 C++ 源码 → 交叉编译后烧录到 MCU**。这种"配置即代码"的模式让不熟悉嵌入式开发的用户也能快速构建智能家居设备。

ESPHome 的架构可以分为两大子系统：

1. **Python 侧代码生成器** — 运行在开发机/服务器上，解析 YAML、验证配置、调度代码生成、输出 `main.cpp`
2. **C++ 侧运行时框架** — 运行在 MCU 上，提供组件生命周期管理、调度器、自动化框架、Native API 通信等基础设施

本文基于 [ESPHome 源码](https://github.com/esphome/esphome)，深入剖析其设计思路和实现细节。

---

## 一、整体架构总览

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        用户层                                            │
│   YAML 配置文件 (device.yaml)                                            │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    Python 侧代码生成器                                    │
│  ┌──────────┐  ┌───────────┐  ┌───────────┐  ┌───────────────────┐      │
│  │ YAML解析  │→│ 配置验证   │→│ 协程调度   │→│ C++代码生成/写入   │      │
│  │yaml_util │  │config.py  │  │coroutine  │  │cpp_generator.py   │      │
│  │          │  │cv模块     │  │.py        │  │writer.py          │      │
│  └──────────┘  └───────────┘  └───────────┘  └───────────────────┘      │
│       ↑              ↑              ↑                   │                │
│  ┌────┴──────────────┴──────────────┴───────────────────┘                │
│  │  各组件 __init__.py:  CONFIG_SCHEMA + to_code()                       │
│  └───────────────────────────────────────────────────────────────────────│
└───────────────────────────────┬──────────────────────────────────────────┘
                                │ 生成 main.cpp + 构建文件
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    C++ 侧运行时框架 (MCU)                                 │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  Application (全局单例)                                            │  │
│  │  ├── setup(): 按优先级初始化所有 Component                         │  │
│  │  ├── loop(): 调度器 + 组件循环                                     │  │
│  │  └── Scheduler: set_timeout / set_interval / defer                │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                    │
│  │ Component    │  │ EntityBase   │  │ Controller   │                    │
│  │ (生命周期)   │  │ (实体属性)   │  │ (状态观察)   │                    │
│  └──────────────┘  └──────────────┘  └──────────────┘                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                    │
│  │ Automation   │  │ APIServer    │  │ HAL          │                    │
│  │ (触发/动作)  │  │ (Native API) │  │ (硬件抽象)   │                    │
│  └──────────────┘  └──────────────┘  └──────────────┘                    │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 二、Python 侧代码生成器

### 2.1 核心流程

```
YAML 配置
    ↓
__main__.py → 解析 CLI 命令 (run, compile, etc.)
    ↓
config.py + config_validation.py → 验证 YAML 合法性
    ↓
loader.py → 动态加载组件模块 (ComponentManifest)
    ↓
各组件的 __init__.py:
  - CONFIG_SCHEMA: 定义 YAML schema (基于 voluptuous，通过 `cv` 模块封装)
  - to_code(): async def, 使用 cg (codegen) 生成 C++ 表达式
    ↓
coroutine.py → FakeEventLoop 按优先级调度所有 to_code() 协程
    ↓
cpp_generator.py → 构建 C++ 表达式树 (MockObj, Expression, Statement)
    ↓
writer.py → 将表达式树序列化为 main.cpp
    ↓
build_gen/ → 生成 PlatformIO 或 ESP-IDF 构建文件
    ↓
编译 & 上传 (通过 PlatformIO 或 ESP-IDF)
```

### 2.2 Python 侧的 async 从何而来

ESPHome 的 Python 侧使用了一套**自建的伪协程系统**，而非 Python 标准库的 `asyncio` 事件循环。这套系统定义在 `esphome/coroutine.py` 中。

核心组件包括：

- **`FakeEventLoop`**：模拟事件循环，使用优先队列（`heapq`）调度任务，而非真正的异步 I/O
- **`@coroutine` 装饰器**：将普通函数/生成器函数转为 ESPHome 协程
- **`@coroutine_with_priority(priority)`**：带优先级的协程装饰器
- **`CoroPriority` 枚举**：定义代码生成阶段的执行优先级

`coroutine.py` 开头的文档注释清晰地解释了设计动机：

> The Problem: When running the code generation, components can depend on variables being registered. For example, an i2c-based sensor would need the i2c bus component to first be declared before the codegen can emit code using that variable (or otherwise the C++ won't compile).
>
> ESPHome's codegen system solves this by using coroutine-like methods. When a component depends on a variable, it waits for it using `await cg.get_variable()`. If the variable hasn't been registered yet, control will be yielded back to another component until the variable is registered. This leads to a topological sort, solving the dependency problem.
>
> **Importantly, ESPHome only uses the coroutine *syntax*, no actual asyncio event loop is running in the background.** This is so that we can ensure the order of execution is constant for the same YAML configuration, thus main.cpp only has to be recompiled if the configuration actually changes.

### 2.3 为什么代码生成器需要 async

核心原因有三个：

#### 1) 依赖解析与拓扑排序

代码生成的核心问题是**组件间存在依赖关系**。例如：
- 一个 I2C 传感器需要先声明 I2C 总线变量
- 一个 GPIO 扩展器上的引脚需要先声明扩展器本身

`await cg.get_variable(id)` 的工作方式：
- 如果目标变量已注册，立即返回
- 如果未注册，`yield` 回 `FakeEventLoop`，让其他协程继续执行
- 其他协程注册了该变量后，被阻塞的协程恢复执行

这本质上是一个**协作式调度**，实现了组件间的拓扑排序。

#### 2) 确定性输出

ESPHome **故意不使用真正的 asyncio 事件循环**。`FakeEventLoop.flush_tasks()` 的执行是确定性的——相同的 YAML 配置永远产生相同的 `main.cpp`。这使得 ESPHome 可以判断配置是否真的发生了变化，避免不必要的重新编译。

#### 3) 优先级控制

`CoroPriority` 定义了从 `EARLY_INIT`(1100) 到 `FINAL`(-1000) 的优先级层次：

| 优先级 | 值 | 示例组件 |
|--------|------|----------|
| EARLY_INIT | 1100 | logger |
| PLATFORM | 1000 | esp32, esp8266, rp2040 |
| NETWORK | 201 | network |
| NETWORK_TRANSPORT | 200 | async_tcp |
| DIAGNOSTICS | 90 | esp32_ble_tracker |
| STATUS | 80 | status_led |
| WEB_SERVER_BASE | 65 | web_server_base |
| CAPTIVE_PORTAL | 64 | captive_portal |
| COMMUNICATION | 60 | wifi, ethernet |
| NETWORK_SERVICES | 55 | api, ota |
| OTA_UPDATES | 54 | ota |
| WEB_SERVER_OTA | 52 | web_server (OTA) |
| PREFERENCES | 51 | preferences |
| APPLICATION | 50 | 各实体基类 (sensor, switch, light...) |
| WEB | 40 | web_server |
| AUTOMATION | 30 | automation |
| BUS | 1 | i2c |
| COMPONENT | 0 | 默认优先级 |
| LATE | -100 | globals |
| WORKAROUNDS | -999 | 组件兼容性补丁 |
| FINAL | -1000 | add_includes, 平台定义 |

高优先级的协程先执行，确保基础设施（如平台初始化、网络、总线）在依赖它们的组件之前完成代码生成。

### 2.4 代码生成引擎 (cpp_generator.py)

`cpp_generator.py` 是一个精巧的**C++ 代码模板引擎**，核心设计是 `MockObj`——一个用 Python 对象模拟 C++ 表达式的系统：

```python
# Python 侧代码生成
var = cg.new_Pvariable(id_, ...)    # 创建 new Type(...) 表达式
cg.add(var.set_name("foo"))         # 生成 var->set_name("foo");
cg.add(App.register_sensor(var))    # 生成 App.register_sensor(var);
```

`MockObj` 通过 Python 的魔术方法实现了运算符重载：

```python
class MockObj(Expression):
    def __getattr__(self, attr):
        # obj.set_name → MockObj("obj.set_name", ".")
        # obj.Pset_name → MockObj("obj.set_name", "->")  (当 op 非 "::" 或 "" 时)
    
    def __call__(self, *args):
        # obj(args) → CallExpression
    
    def __lt__(self, other):
        # BinOpExpression(self, "<", other)
    
    @property
    def new(self):
        # Type.new → MockObj("new Type", "->")
```

`Pvariable()` 函数尤其重要——对于 `new` 表达式，它使用**placement new**将对象分配到静态存储中，避免在嵌入式设备上产生堆碎片：

```cpp
// Python: cg.Pvariable(id_, id_.type.new(...))
// 生成的 C++:
alignas(ActualType) static unsigned char {component_ns}__{id}__pstorage[sizeof(ActualType)];
static BaseType *const id = reinterpret_cast<BaseType *>({component_ns}__{id}__pstorage);
new(id) ActualType(constructor_args...);
```

其中 `{component_ns}` 是从类型中提取的组件命名空间（如 `sensor`、`logger`），`{id}` 是变量 ID。例如 `my_sensor` 的存储名为 `sensor__my_sensor__pstorage`。

### 2.5 CORE 全局状态对象

`esphome/core/__init__.py` 中的 `CORE` 对象是代码生成期间的**全局上下文**，保存了所有中间状态：

- `CORE.config` — 解析后的 YAML 配置
- `CORE.data` — 组件间共享的临时数据
- `CORE.cpp_global_section` — 全局 C++ 代码段
- `CORE.defines` — 条件编译宏定义
- `CORE.variables` — 已注册的变量映射
- `CORE.component_ids` — 组件 ID 追踪

`CORE.add()` 将表达式添加到 `setup()` 函数体中，`CORE.add_global()` 添加到全局作用域。

---

## 三、C++ 侧运行时框架

### 3.1 核心基类体系

```
Component (esphome/core/component.h)
├── setup() — 初始化（类似 Arduino setup()）
├── loop() — 主循环（类似 Arduino loop()）
├── dump_config() — 打印配置信息
├── get_setup_priority() — 初始化优先级
├── mark_failed() — 标记为失败
├── disable_loop() / enable_loop() — 控制循环参与
├── status_set_warning/error() — 状态指示
├── set_timeout() / set_interval() / defer() — 定时/延迟
└── on_shutdown() / teardown() — 关机钩子

PollingComponent : Component
├── update() = 0 — 周期性更新接口
├── set_update_interval() — 设置更新间隔
└── call_setup() override — 自动注册 interval

EntityBase (esphome/core/entity_base.h)
├── name_, object_id_hash_
├── is_internal(), is_disabled_by_default()
├── get_device_class_to(), get_unit_of_measurement_ref()
├── get_icon_to(), get_entity_category()
└── configure_entity_() — 代码生成调用的统一配置方法

StatefulEntityBase<T> : EntityBase
├── get_state() — 获取当前状态
├── add_on_state_callback() — 状态变化回调
├── add_full_state_callback() — 含旧值的状态变化回调
└── set_new_state() — 状态更新 + 回调触发

Controller (esphome/core/controller.h)
└── on_<entity>_update() — 各实体类型的虚方法（X-macro 生成）

Application (esphome/core/application.h)
├── 全局单例 App
├── setup() / loop() — 主循环入口
├── register_component_<T>() — 模板化组件注册（编译期检测 loop 重写）
├── register_<entity>() — 实体注册（X-macro 生成）
├── scheduler — 调度器
└── looping_components_ — 分区向量 [active | inactive]
```

### 3.2 各基类的作用

#### Component — 组件生命周期

`Component` 是所有 MCU 端组件的基类，定义了统一的生命周期：

1. **CONSTRUCTION** → **SETUP** → **LOOP** → **LOOP_DONE** （或 **FAILED**）
2. `setup()` 只执行一次，按 `setup_priority` 排序
3. `loop()` 在主循环中反复调用
4. 组件可以通过 `disable_loop()` 退出循环以节省 CPU

`setup_priority` 命名空间定义了 C++ 端的初始化优先级，与 Python 端的 `CoroPriority` 对应：

| 优先级 | 值 | 含义 |
|--------|------|------|
| BUS | 1000 | 通信总线 (I2C/SPI) |
| IO | 900 | GPIO 扩展器 |
| HARDWARE | 800 | 硬件组件 |
| DATA | 600 | 直连传感器 (默认) |
| PROCESSOR | 400 | 数据处理器 (display) |
| BLUETOOTH | 350 | 蓝牙组件 |
| AFTER_BLUETOOTH | 300 | 蓝牙后初始化 |
| WIFI | 250 | WiFi |
| ETHERNET | 250 | 以太网 |
| BEFORE_CONNECTION | 220 | 连接前初始化 |
| AFTER_WIFI | 200 | WiFi 后初始化 |
| AFTER_CONNECTION | 100 | 连接建立后 |
| LATE | -100 | 延迟初始化 |

#### PollingComponent — 轮询组件

对于周期性检查状态的传感器，`PollingComponent` 封装了定时逻辑：
- 开发者只需实现 `update()` 方法
- 框架自动通过 `set_interval()` 按配置的 `update_interval` 调用

#### EntityBase — 实体属性

`EntityBase` 是所有"实体"（sensor, switch, light 等）的属性基类，管理：
- 名称 (`name_`)、对象 ID 哈希 (`object_id_hash_`)
- 设备类别 (`device_class_idx_`)、单位 (`uom_idx_`)、图标 (`icon_idx_`) — 使用索引而非字符串，节省 RAM
- `internal`、`disabled_by_default`、`entity_category` 等标志 — 位域打包为 1 字节
- `configure_entity_()` 方法：被代码生成器调用，一次性设置所有属性

#### StatefulEntityBase\<T\> — 有状态实体

模板化的有状态实体基类，管理状态变化和回调分发。注意：并非所有实体类型都继承 `StatefulEntityBase`，例如 `Sensor` 直接继承 `EntityBase` 并自行管理 `float state`，而 `BinarySensor` 则继承 `StatefulEntityBase<bool>`：
- `BinarySensor : StatefulEntityBase<bool>`
- `TextSensor : StatefulEntityBase<std::string>`
- `Sensor : EntityBase`（自行管理 `float state`，未使用 `StatefulEntityBase<float>`）

回调分为两级：
- `state_callbacks_`：仅在新值有效且非首次设置（或配置了触发首次）时触发
- `full_state_callbacks_`：每次变化都触发，携带 `optional<T>` 旧值和新值

#### Controller — 状态观察者

`Controller` 是观察者模式的基类，通过 X-macro 从 `entity_types.h` 生成虚方法：

```cpp
// 生成的虚方法（简化）
class Controller {
public:
    virtual void on_binary_sensor_update(binary_sensor::BinarySensor *obj) {}
    virtual void on_sensor_update(sensor::Sensor *obj) {}
    virtual void on_switch_update(switch_::Switch *obj) {}
    virtual void on_light_update(light::LightState *obj) {}
    // ... 所有实体类型
};
```

`APIServer` 和 `WebServer` 都继承 `Controller`，重写这些虚方法来推送状态更新。

#### Application — 全局管理器

`Application` 是全局单例（`App`），管理所有组件和实体的注册与调度。其核心设计：

1. **分区向量**：`looping_components_` 分为 `[active | inactive]` 两段，避免循环时检查标志
2. **编译期检测**：`HasLoopOverride<T>` 通过 SFINAE 检测 `T` 是否重写了 `loop()`
3. **模板化注册**：`register_component_<T>()` 在编译期决定组件是否参与循环

```cpp
template<typename T> void register_component_(T *comp, uint8_t source_index = 0) {
    if (source_index != 0)
        comp->set_component_source_(source_index);
    this->register_component_impl_(comp, HasLoopOverride<T>::value);
}
```

### 3.3 X-macro 实体类型系统

ESPHome 使用 X-macro 技术（`entity_types.h`）来消除大量重复代码。该文件被多次 include，每次定义不同的宏来生成不同的代码：

```cpp
// entity_types.h (X-macro 定义)
#ifdef USE_SENSOR
ENTITY_CONTROLLER_TYPE_(sensor::Sensor, sensor, sensors, ESPHOME_ENTITY_SENSOR_COUNT, SENSOR, sensor_update)
#endif
```

在 `application.h` 中用于生成注册方法：

```cpp
#define ENTITY_TYPE_(type, singular, plural, count, upper) \
  void register_##singular(type *obj) { this->plural##_.push_back(obj); }
#define ENTITY_CONTROLLER_TYPE_(type, singular, plural, count, upper, callback) \
  ENTITY_TYPE_(type, singular, plural, count, upper)
#include "esphome/core/entity_types.h"
```

在 `controller.h` 中生成虚方法，在 `entity_base.h` 中生成查找表索引等。

### 3.4 Scheduler — 调度器

`Scheduler`（`esphome/core/scheduler.h`）是 MCU 端的定时任务管理器，提供：
- `set_timeout()` — 一次性定时
- `set_interval()` — 周期性定时
- `defer()` — 延迟到下一个 loop() 执行
- `cancel_timeout()` / `cancel_interval()` — 取消定时

调度器在每次 `Application::loop()` 中被调用，按时间戳排序执行到期的任务。这提供了类似 JavaScript 的 `setTimeout/setInterval` 语义，但**不保证精确时序**——因为嵌入式主循环可能被阻塞。

### 3.5 Automation 框架

ESPHome 的自动化系统（`esphome/core/automation.h`）是声明式的触发-动作系统：

```
Trigger<T...> ──触发──→ Action<T...> ──执行──→ 下一个 Action
```

核心类：
- **`Trigger<T...>`**：触发器，调用 `trigger()` 激活所有绑定的动作
- **`Action<T...>`**：动作基类，`play()` 执行，支持链式组合
- **`TemplatableFn<T, X...>`**：4 字节函数指针存储（替代 8 字节的 `TemplatableValue`）
- **`TemplatableStorage<T, X...>`**：自动选择 `TemplatableFn`（trivially copyable）或 `TemplatableValue`（非 trivial）

动作链通过 `play_complex()` 实现异步执行，支持 DelayAction、IfAction、WhileAction 等控制流。

---

## 四、Native API 通信

### 4.1 架构概览

ESPHome 与 Home Assistant 通过 **Native API** 通信（TCP 端口 6053），使用自定义的 Protobuf 协议。API 组件位于 `esphome/components/api/`。

```
Home Assistant ←──── TCP ────→ APIServer (MCU)
                                    │
                                    ├── APIConnection (每个客户端)
                                    │     ├── 消息读写
                                    │     ├── 实体列表推送
                                    │     └── 状态订阅/命令下发
                                    ├── Proto (自定义 Protobuf 编解码)
                                    ├── APIFrameHelper (帧分帧/批处理)
                                    │     ├── PlaintextFrameHelper
                                    │     └── NoiseFrameHelper (加密)
                                    └── api.proto (服务定义)
```

### 4.2 APIServer — 服务端入口

`APIServer` 继承 `Component` 和 `Controller`（条件编译时还继承 `camera::CameraListener`）：

```cpp
class APIServer final : public Component, public Controller
#ifdef USE_CAMERA
    , public camera::CameraListener
#endif
{
    void setup() override;     // 监听 TCP 端口
    void loop() override;      // 接受新连接、处理 I/O
    float get_setup_priority() const override; // AFTER_CONNECTION
    
    // Controller 回调 — 实体状态变化时推送到所有客户端
    void on_sensor_update(sensor::Sensor *obj) override;
    void on_switch_update(switch_::Switch *obj) override;
    // ... 所有实体类型
};
```

当实体状态变化时，`APIServer` 的 `on_*_update()` 回调被触发，将新状态推送到所有订阅了该实体的客户端。

### 4.3 APIConnection — 客户端连接

`APIConnection` 代表一个与 Home Assistant 的连接，负责：
- **消息读写**：基于帧协议的异步消息收发
- **实体发现**：响应 `list_entities_*` 请求，发送实体的名称、类型、属性
- **状态订阅**：响应 `subscribe_states`，在状态变化时推送
- **命令下发**：处理 `switch_command`、`light_command` 等控制命令
- **日志流**：支持 `subscribe_logs` 实时推送日志
- **蓝牙代理**：支持 HA 通过 ESPHome 代理蓝牙设备
- **Voice Assistant**：语音助手支持

连接数受 `MAX_API_CONNECTIONS` 限制，该值由 Python 代码生成器根据目标平台动态定义（ESP8266 默认 4，ESP32 默认 5，Host 默认 8），使用 `std::array<unique_ptr<APIConnection>, N>` 管理。

### 4.4 自定义 Protobuf 编解码

ESPHome 没有使用 Google protobuf 库（太重），而是实现了自己的轻量级编解码器（`proto.h`）：

- 支持 varint、zigzag 编码
- 自动生成的 `api_pb2.h/.cpp`：消息序列化/反序列化
- 自动生成的 `api_pb2_service.h/.cpp`：服务端/客户端 stub

`script/api_protobuf/api_protobuf.py` 从 `api.proto` 生成这些文件。

### 4.5 帧层协议与加密

帧层协议（`api_frame_helper.h`）负责消息分帧和批处理：

- **PlaintextFrameHelper**：明文传输
- **NoiseFrameHelper**：使用 [Noise Protocol Framework](https://noiseprotocol.org/) 加密

加密配置通过 YAML 中的 `api.encryption.key` 设置，PSK 存储在非易失性存储中。

### 4.6 api.proto 中的核心 RPC 方法

```
┌─────────────┬──────────────────────────────────────────────┐
│ 类别         │ RPC 方法                                      │
├─────────────┼──────────────────────────────────────────────┤
│ 连接管理     │ hello, ping, disconnect, device_info          │
│ 实体发现     │ list_entities_binary_sensor, list_entities_   │
│             │ sensor, list_entities_switch, ... (20+ 类型)   │
│ 状态订阅     │ subscribe_states                              │
│ 命令下发     │ switch_command, light_command, cover_command, │
│             │ climate_command, number_command, ...           │
│ HA 集成      │ subscribe_homeassistant_services,             │
│             │ subscribe_home_assistant_states, execute_service│
│ 日志         │ subscribe_logs                                │
│ 蓝牙代理     │ bluetooth_device_request, bluetooth_gatt_*    │
│ 语音助手     │ voice_assistant_*                             │
│ 相机         │ camera_image                                  │
│ 加密         │ noise_encryption_set_key                      │
└─────────────┴──────────────────────────────────────────────┘
```

---

## 五、多 MCU 平台支持

### 5.1 HAL 硬件抽象层

ESPHome 通过 HAL（Hardware Abstraction Layer）支持不同的 MCU 平台。核心思路是**编译期条件分发**：

```cpp
// esphome/core/hal.h
#if defined(USE_ESP32)
#include "esphome/components/esp32/hal.h"
#elif defined(USE_ESP8266)
#include "esphome/components/esp8266/hal.h"
#elif defined(USE_LIBRETINY)
#include "esphome/components/libretiny/hal.h"
#elif defined(USE_RP2040)
#include "esphome/components/rp2040/hal.h"
#elif defined(USE_HOST)
#include "esphome/components/host/hal.h"
#elif defined(USE_ZEPHYR)
#include "esphome/components/zephyr/hal.h"
#endif
```

每个平台组件（如 `esphome/components/esp32/`）提供：
- `hal.h`：平台特定的内联函数（`delayMicroseconds()`, `arch_feed_wdt()`, `millis()`, `micros()` 等）
- `__init__.py`：Python 侧的平台注册、构建配置、GPIO schema
- 平台特定的代码（如 ESP32 的 BLE、RMT 外设支持）

### 5.2 条件编译 (defines.h)

ESPHome 大量使用条件编译宏来裁剪固件大小。`defines.h` 由 Python 代码生成器根据 YAML 配置动态生成：

```cpp
// 自动生成的 defines.h
#define USE_SENSOR
#define USE_BINARY_SENSOR
#define USE_API
#define USE_WIFI
// ... 仅包含 YAML 中用到的功能
```

这使得未使用的功能完全不占 Flash/RAM，对于资源受限的 ESP8266 尤为重要。

### 5.3 GPIO 抽象

`GPIOPin` / `InternalGPIOPin` 是引脚抽象基类：

```cpp
class GPIOPin {
public:
    virtual void setup() = 0;
    virtual bool digital_read() = 0;
    virtual void digital_write(bool value) = 0;
    virtual Flags get_flags() = 0;
};
```

每个平台实现自己的 `InternalGPIOPin`，GPIO 扩展器（如 PCF8574、MCP23017）实现 `GPIOPin`。

Python 侧的引脚系统通过 `pins.py` 中的 `PIN_SCHEMA_REGISTRY` 注册各平台的引脚 schema，代码生成时调用 `gpio_pin_expression()` 自动选择正确的实现。

### 5.4 组件平台模式

ESPHome 的组件采用**平台模式**实现多硬件支持。以传感器为例：

```
YAML:
  sensor:
    - platform: dht        ← 具体平台实现
      pin: GPIO4
      temperature:
        name: "Temperature"
```

Python 侧：
- `sensor/__init__.py`：定义传感器基类的 `CONFIG_SCHEMA`
- `sensor/dht/__init__.py`：定义 DHT 平台的 schema 和 `to_code()`

C++ 侧：
- `sensor/sensor.h`：`Sensor` 基类（继承 `EntityBase`，自行管理 `float state`）
- `dht/dht.h`：`DHTSensor` 继承 `Sensor` + `PollingComponent`，实现 `update()` 读取硬件

### 5.5 唤醒系统

不同平台的睡眠/唤醒机制不同，ESPHome 通过平台特定的唤醒实现处理：

```
esphome/core/wake/
├── wake_esp8266.cpp/.h     — ESP8266: esp_delay/yield
├── wake_freertos.cpp/.h    — ESP32/LibreTiny: FreeRTOS task notification
├── wake_host.cpp/.h        — Host: select() on socket
├── wake_rp2040.cpp/.h      — RP2040: WFE
└── wake_zephyr.cpp/.h      — Zephyr: k_sem
```

`Application::loop()` 末尾通过 `esphome::internal::wakeable_delay(delay_time)` 进入低功耗等待，可以被：
- 定时器到期
- 外部中断
- 其他线程/ISR 的 `wake_loop_threadsafe()` 唤醒

### 5.6 持久化存储 (Preferences)

ESPHome 通过 Preferences 系统实现断电后状态恢复，采用**分层架构**：

```
组件层 (fan/switch/light/cover...)
    │  调用 rtc_.save(&state) / rtc_.load(&recovered)
    ▼
ESPPreferenceObject (类型安全模板包装器)
    │  调用 backend_->save(data, len) / backend_->load(data, len)
    ▼
PreferenceBackend (平台相关后端)
    ├── ESP32: NVS (Non-Volatile Storage)
    ├── ESP8266: RTC 用户内存 + Flash 扇区
    ├── RP2040 / LibreTiny / Host / Zephyr: 各自实现
    └── Stub: 空实现
```

#### ESPPreferenceObject — 类型安全包装器

```cpp
class ESPPreferenceObject {
  PreferenceBackend *backend_{nullptr};
public:
  template<typename T> bool save(const T *src) {
    return this->backend_->save(reinterpret_cast<const uint8_t *>(src), sizeof(T));
  }
  template<typename T> bool load(T *dest) {
    return this->backend_->load(reinterpret_cast<uint8_t *>(dest), sizeof(T));
  }
};
```

通过模板将任意 `trivially_copyable` 类型序列化为字节流，内部只持有 `PreferenceBackend*` 指针。

#### Key 生成机制

`EntityBase::make_entity_preference<T>(version)` 生成持久化对象，key 计算方式：

```
key = fnv1_hash(object_id) ^ device_id ^ version
```

- `object_id`：YAML 中的实体名称
- `device_id`：设备标识（多设备时区分）
- `version`：硬编码随机常量（如 Fan 的 `0x71700ABB`），修改存储结构时更改，旧数据自动失效

#### ESP32 实现：基于 NVS

ESP32 使用 ESP-IDF 的 NVS（Non-Volatile Storage），调用以下 ESP-IDF 函数：

| 函数 | 用途 |
|------|------|
| `nvs_flash_init()` | 初始化 NVS 分区 |
| `nvs_open("esphome", NVS_READWRITE, &handle)` | 打开 "esphome" 命名空间 |
| `nvs_get_blob(handle, key, data, &len)` | 读取二进制数据 |
| `nvs_set_blob(handle, key, data, len)` | 写入二进制数据 |
| `nvs_commit(handle)` | 提交写入 |
| `nvs_flash_erase()` | 擦除整个 NVS 分区（损坏/重置时） |

**核心设计 — 延迟写入**：`save()` 不直接写 flash，而是追加到 `s_pending_save` 内存向量。同一 key 多次 save 只保留最新值。`sync()` 时通过 `is_changed_()` 先读取旧值 memcmp 比较，仅写入真正变化的数据，最大限度减少 flash 写入。

```
组件 save(&state) → 存入 s_pending_save 内存
                          ↓ (每 60s 或关机时)
sync() → is_changed_(旧值 vs 新值) → nvs_set_blob() → nvs_commit()
```

NVS 自带磨损均衡和校验，ESP32 上 `in_flash` 参数被忽略（全部走 NVS）。

#### ESP8266 实现：RTC 内存 + Flash 扇区

ESP8266 采用**双存储架构**，远比 ESP32 复杂：

1. **RTC 用户内存**（`0x60001200`）：128 个 32-bit word（512 字节）
   - 深度睡眠后保留，**断电后丢失**
   - Normal 区域仅 78 words（312 字节），空间极其有限
2. **Flash 扇区**：SPIFFS 之后的整扇区
   - 断电后保留
   - 通过 `spi_flash_erase_sector()` / `spi_flash_write()` 操作
   - **无磨损均衡**，整扇区擦除+重写

每个偏好数据附带 CRC word（`type` 参数参与计算），load 时校验。ESP32 无需此机制（NVS 内部有校验）。

#### 两平台差异对比

| 特性 | ESP32 (NVS) | ESP8266 (RTC+Flash) |
|------|-------------|---------------------|
| 存储后端 | NVS（自带磨损均衡+校验） | 手动管理 RTC + Flash |
| save() 语义 | 延迟（内存缓冲） | RTC: 立即; Flash: 写 RAM 缓冲 |
| 变更检测 | memcmp 旧值，跳过未变化数据 | Flash: dirty flag; RTC: 无条件写 |
| Flash 写入 | 单键值对（nvs_set_blob） | 整扇区擦除+重写 |
| 空间限制 | NVS 分区（16-24KB） | RTC: 312B; Flash: 256-512B |

#### Sync 机制

`IntervalSyncer` 组件（优先级 `BUS`）负责定时同步：

- 默认每 **60 秒** `sync()` 一次（可通过 `preferences.flash_write_interval` 配置）
- 设为 `0s` 时，每次 `loop()` 都 sync（编译宏 `USE_PREFERENCES_SYNC_EVERY_LOOP`）
- **关机时额外 sync**（`on_shutdown`），确保数据不丢失

#### 哪些对象需要持久化

需要断电恢复状态的实体才使用持久化，典型包括：

| 组件 | 存储结构 | 大小 | 说明 |
|------|---------|------|------|
| Switch | `bool` | 1B | 开关状态 |
| Fan | `FanRestoreState` | ~8B | 风速、方向、振荡 |
| Light | `LightStateRTCState` | ~44B | 亮度、色温、颜色、效果 |
| Cover | `CoverRestoreState` | ~8B | 位置、倾斜角 |
| Number | 平台相关 | 4-8B | 数值设定（如 LD2450 的存在超时） |

不需要持久化的实体：
- **Sensor / BinarySensor**：实时读取硬件值，无需恢复
- **TextSensor**：动态生成文本
- **Button**：瞬时动作，无状态
- **Select**：部分实现使用，部分不使用

### 5.7 构建系统

ESPHome 支持两种构建后端：

1. **PlatformIO**（ESP8266、部分 ESP32 Arduino）— `esphome/build_gen/platformio.py`
2. **ESP-IDF CMake**（ESP32 Arduino/ESP-IDF）— `esphome/build_gen/espidf.py`

Python 代码生成器根据目标平台生成相应的构建文件。

---

## 六、典型组件结构

以 `binary_sensor` 为例，展示一个组件的完整结构：

```
esphome/components/binary_sensor/
├── __init__.py (22KB)        # Python: CONFIG_SCHEMA + to_code()
├── binary_sensor.h           # C++ BinarySensor 类
├── binary_sensor.cpp         # C++ 实现
├── automation.h              # C++ 触发器/动作
├── automation.cpp            # C++ 自动化实现
├── filter.h                  # C++ 过滤器基类
└── filter.cpp                # C++ 过滤器实现
```

Python 侧 (`__init__.py`)：
```python
# 声明 C++ 类型
binary_sensor_ns = cg.esphome_ns.namespace("binary_sensor")
BinarySensor = binary_sensor_ns.class_("BinarySensor", cg.EntityBase)

# 定义 YAML schema
CONFIG_SCHEMA = cv.Schema({...})

# 代码生成
async def to_code(config):
    var = cg.new_Pvariable(config[CONF_ID])
    await cg.register_component(var, config)
    # ... 设置属性
```

C++ 侧 (`binary_sensor.h`)：
```cpp
class BinarySensor : public StatefulEntityBase<bool> {
public:
    void publish_state(bool state);  // 发布新状态
    void add_filter(Filter *filter);
    // ...
};
```

注意 `BinarySensor` 只继承 `StatefulEntityBase<bool>`，不继承 `Component`。轮询逻辑由具体的平台实现类（如 `GPIOBinarySensor`）通过 `PollingComponent` 单独提供。

---

### 6.1 实战分析：LD2450 毫米波雷达组件

LD2450 是一个典型的复杂组件，展示了 ESPHome 组件架构的多种模式。它是一个 24GHz 毫米波人体存在传感器，通过 UART 通信，支持多目标追踪和区域检测。

### 组件结构

```
esphome/components/ld2450/
├── __init__.py                    # CONFIG_SCHEMA + to_code()
├── ld2450.h / ld2450.cpp          # 核心 C++ 类 (UART + Component)
├── binary_sensor.py               # 目标检测二值传感器
├── sensor.py                      # 坐标/速度/距离等数值传感器
├── text_sensor.py                 # 版本/MAC/方向文本传感器
├── button/                        # 恢复出厂 + 重启按钮
├── number/                        # 存在超时 + 区域坐标 Number
├── select/                        # 波特率 + 区域类型选择器
└── switch/                        # 蓝牙 + 多目标模式开关
```

依赖共享基类 `ld24xx/ld24xx.h`，提供 `SensorWithDedup<T>` 去重传感器模板和辅助宏。

### UART 通信协议

LD2450 使用自定义二进制协议，两种帧格式：

**命令帧**：`FD FC FB FA` + 长度 + 命令 + 参数 + `04 03 02 01`

**数据帧**（周期性）：`AA FF 03 00` + 3×8字节目标数据 + `55 CC`

```
每个目标 8 字节：
┌─────────┬──────────┬──────────┬──────────┬──────────┐
│ X(2B)   │ Y(2B)    │ Speed(2B)│ Res(1B)  │ Flags(1B)│
└─────────┴──────────┴──────────┴──────────┴──────────┘
```

通信流程：`send_command_()` → `readline_()` 逐字节解析 → `handle_periodic_data_()` / `handle_ack_data_()`。大部分命令需要先进入配置模式（`CMD_ENABLE_CONF`）。

### 提供的实体类型

| 类型 | 数量 | 说明 |
|------|------|------|
| Binary Sensor | 3 | 有目标/移动目标/静止目标（带存在超时） |
| Sensor | 6+9×3+9×3=60 | 全局计数 + 每目标坐标/速度/角度/距离 + 每区域计数 |
| Text Sensor | 2+3 | 版本/MAC + 每目标方向 |
| Number | 1+3×4=13 | 存在超时 + 3区域×4坐标 |
| Select | 2 | 波特率 / 区域类型 |
| Switch | 2 | 蓝牙 / 多目标模式 |
| Button | 2 | 恢复出厂 / 重启 |

### 区域检测系统

最多 3 个矩形区域，每个区域由对角点 (x1,y1)-(x2,y2) 定义：

- **Detection 模式**：仅检测区域内的目标
- **Filter 模式**：过滤掉区域内的目标
- **Disabled**：区域禁用

`count_targets_in_zone_()` 在 `handle_periodic_data_()` 中对每个目标判断是否在区域内，使用严格不等号。

### 持久化存储

LD2450 **仅持久化一个值**：`presence_timeout`（存在超时，默认 5 秒）。

```cpp
// setup() 中初始化偏好对象
this->pref_ = this->presence_timeout_number_->make_entity_preference<float>();

// 保存到 flash
void LD2450Component::save_to_flash_(float value) { this->pref_.save(&value); }

// 从 flash 恢复
float LD2450Component::restore_from_flash_() {
    float value;
    if (!this->pref_.load(&value))
        value = DEFAULT_PRESENCE_TIMEOUT;  // 5 秒
    return value;
}
```

区域坐标、蓝牙状态、多目标模式等**不持久化**——雷达硬件自身有内部存储，重启后从硬件重新读取。

### 设计亮点

1. **条件编译**：所有实体类型通过 `#ifdef USE_xxx` 裁剪，未使用的实体零开销
2. **去重机制**：`SensorWithDedup<T>` + `Deduplicator<T>`，避免重复发布相同值
3. **存在超时**：Binary Sensor 不立即清除状态，等待可配置超时后才切换 OFF
4. **自动帧同步**：UART 解析器通过帧尾检测实现自动重新同步
5. **懒加载回调**：`LazyCallbackManager<void()>` 仅在注册 `on_data` 自动化时才分配内存

---

## 七、代码生成的 main.cpp 结构

`writer.py` 生成的 `main.cpp` 结构如下：

```cpp
// ========== AUTO GENERATED INCLUDE BLOCK BEGIN ===========
#include "esphome.h"
using namespace esphome;
// 全局变量声明（placement new 存储区、指针声明）
// ========== AUTO GENERATED INCLUDE BLOCK END ===========

// ========== AUTO GENERATED CODE BEGIN ===========
void setup() {
  // 1. App.pre_setup("device_name", ...)
  // 2. App.register_component_<Type>(var, source_idx)  // 按优先级排序
  // 3. App.register_sensor(sensor_var, name, hash, fields)
  // 4. 各种 set_* 调用
  // 5. App.setup()
}

void loop() {
  App.loop();
}
// ========== AUTO GENERATED CODE END ===========
```

`esphome.h` 是自动生成的超级头文件，include 了所有用到的组件头文件。

---

## 八、命令行接口 (CLI)

ESPHome 的所有功能通过命令行驱动，CLI 定义在 `esphome/__main__.py` 中，使用 `argparse` 的 subparsers 机制注册子命令。

### 8.1 全局选项

所有子命令共享以下选项：

| 选项 | 短选项 | 说明 |
|------|--------|------|
| `--verbose` | `-v` | 启用详细日志（等价于 `--log-level DEBUG`） |
| `--quiet` | `-q` | 禁用所有日志（等价于 `--log-level CRITICAL`） |
| `--log-level` | `-l` | 设置日志级别：`DEBUG`、`INFO`、`WARNING`、`ERROR`、`CRITICAL` |
| `--substitution` | `-s` | 添加 YAML 替换，格式：`-s key value`，可多次使用 |
| `--toolchain` | | 选择编译工具链：`platformio` 或 `esp-idf`，覆盖 YAML 中的设置 |
| `--version` | | 打印版本号并退出 |

相关环境变量：

| 环境变量 | 用途 |
|----------|------|
| `ESPHOME_VERBOSE` | 默认启用 verbose |
| `ESPHOME_LOG_LEVEL` | 默认日志级别 |
| `ESPHOME_UPLOAD_SPEED` | esptool 默认上传速度（回退值 `460800`） |
| `ESPHOME_SERIAL_LOGGING_RESET` | 默认启用串口日志前重置设备 |

### 8.2 常用命令

#### `run` — 编译 + 上传 + 日志（最常用）

```bash
esphome run device.yaml
```

这是最常用的命令，执行完整流程：生成 C++ → 编译 → 上传 → 显示日志。

| 选项 | 说明 |
|------|------|
| `--device` | 手动指定上传目标（串口/IP/`OTA`），可多次指定用于回退 |
| `--upload_speed` | 覆盖上传速度 |
| `--no-logs` | 上传成功后不启动日志查看 |
| `--no-states` | 不显示实体状态变化 |
| `-r` / `--reset` | 串口日志前重置设备 |
| `--ota-platform` | OTA 平台：`esphome`（默认）或 `web_server` |

**`--device` 特殊值**：
- 串口路径：`/dev/ttyUSB0`、`COM3` — 通过 esptool 串口刷写
- `OTA` — 从配置自动解析（mDNS/DNS/MQTT）
- IP 地址 / mDNS 主机名 — 网络 OTA 上传
- `MQTT` — 通过 MQTT 发现 IP
- `BOOTSEL` — RP2040 BOOTSEL 模式（通过 picotool）

```bash
# 编译 + 通过串口上传 + 查看日志
esphome run device.yaml --device /dev/ttyUSB0

# 编译 + OTA 上传
esphome run device.yaml --device 192.168.1.100

# 仅编译上传，不看日志
esphome run device.yaml --no-logs --device OTA
```

#### `compile` — 编译固件

```bash
esphome compile device.yaml
```

| 选项 | 说明 |
|------|------|
| `--only-generate` | 仅生成 C++ 源代码，不编译 |

```bash
# 仅生成 main.cpp 等源文件，不触发编译
esphome compile device.yaml --only-generate

# 使用 ESP-IDF 工具链编译
esphome compile device.yaml --toolchain esp-idf
```

#### `upload` — 上传固件

```bash
esphome upload device.yaml --device /dev/ttyUSB0
```

| 选项 | 说明 |
|------|------|
| `--device` | 指定上传目标（同 `run`） |
| `--upload_speed` | 覆盖上传速度 |
| `--file` | 手动指定二进制文件路径 |
| `--ota-platform` | `esphome` 或 `web_server` |
| `--partition-table` | 通过 OTA 上传分区表（需 `allow_partition_access: true`） |
| `--bootloader` | 通过 OTA 上传引导加载程序（需 `allow_partition_access: true`） |

**上传路径选择**：
1. **串口**：ESP32/ESP8266 用 `esptool`，RP2040/LibreTiny 用 PlatformIO
2. **BOOTSEL**：用 `picotool load -v -x <elf>`
3. **网络 OTA**：
   - `esphome` 平台 → 原生 API 挑战-响应认证（更安全，默认）
   - `web_server` 平台 → HTTP Basic 认证

esptool 默认上传速度为 `460800`，失败后自动回退 `115200`。

#### `logs` — 查看日志

```bash
esphome logs device.yaml
```

| 选项 | 短选项 | 说明 |
|------|--------|------|
| `--device` | | 指定日志来源（同 `run`） |
| `--reset` | `-r` | 串口日志前重置设备 |
| `--no-states` | | 不显示实体状态变化 |

**日志来源优先级**：
1. **串口**：直接读取串口数据，波特率从配置的 `logger.baud_rate` 读取
2. **API**：通过原生 API 连接获取日志（默认订阅状态变化）
3. **MQTT**：最后的回退方案

```bash
# 串口日志（波特率自动从配置读取）
esphome logs device.yaml --device /dev/ttyUSB0

# 通过 WiFi 查看远程日志
esphome logs device.yaml --device 192.168.1.100

# 串口日志前重置设备
esphome logs device.yaml -r --device COM3
```

#### `config` — 验证并输出配置

```bash
esphome config device.yaml
```

| 选项 | 说明 |
|------|------|
| `--show-secrets` | 在输出中显示密码/密钥（默认隐藏） |

### 8.3 其他命令

| 命令 | 说明 |
|------|------|
| `wizard <config>` | 交互式 4 步设置向导（设备名 → 平台 → 开发板 → WiFi） |
| `version` | 打印版本号 |
| `clean <config>` | 清理构建文件 |
| `clean-all` | 清理所有构建和平台文件 |
| `clean-mqtt <config>` | 清理 MQTT 保留消息 |
| `dashboard <dir>` | 启动 Web 仪表盘（默认端口 6052） |
| `rename <config> <name>` | 重命名设备（修改 YAML + 重新编译上传） |
| `idedata <config>` | 输出 PlatformIO IDE 数据（仅 PlatformIO 工具链） |
| `analyze-memory <config>` | 组件级内存分析（使用 objdump + readelf） |
| `bundle <config>` | 创建自包含配置打包文件 (.esphomebundle) |
| `update-all <dir>` | 编译+上传目录下所有 YAML 配置 |
| `vscode <config>` | VSCode 集成模式（stdin/stdout JSON 验证协议） |
| `discover <config>` | 通过 MQTT 发现设备 |
| `config-hash <config>` | 计算配置哈希值 |

### 8.4 Dashboard 命令

```bash
esphome dashboard /path/to/configs/
```

| 选项 | 默认值 | 说明 |
|------|--------|------|
| `--port` | `6052` | HTTP 端口 |
| `--address` | `0.0.0.0` | 绑定地址 |
| `--username` | | 认证用户名 |
| `--password` | | 认证密码（也可从 `$PASSWORD` 环境变量读取） |
| `--open-ui` | | 自动在浏览器中打开 |

```bash
# 本地启动（自动打开浏览器）
esphome dashboard ./configs/ --open-ui

# 带认证的远程访问
esphome dashboard ./configs/ --address 0.0.0.0 --port 6052 --username admin --password secret
```

### 8.5 典型工作流

```bash
# 1. 首次使用：创建配置
esphome wizard my-device.yaml

# 2. 验证配置正确
esphome config my-device.yaml

# 3. 首次烧录（串口）
esphome run my-device.yaml --device /dev/ttyUSB0

# 4. 后续更新（OTA 无线）
esphome run my-device.yaml --device OTA

# 5. 仅查看日志
esphome logs my-device.yaml

# 6. 仅编译不烧录
esphome compile my-device.yaml

# 7. 调试：查看生成的 C++ 源码
esphome compile my-device.yaml --only-generate -v

# 8. 分析内存使用
esphome analyze-memory my-device.yaml

# 9. 使用 YAML 替换变量
esphome run my-device.yaml -s wifi_ssid MyHome -s wifi_password secret123

# 10. 批量更新所有设备
esphome update-all ./configs/
```

---

## 九、工具链安装与路径管理

ESPHome 支持两种编译工具链：**PlatformIO** 和 **ESP-IDF**，通过 `--toolchain` 参数或 YAML 中的 `esphome.toolchain:` 配置选择，默认为 PlatformIO。首次编译时，ESPHome 会自动下载安装所需工具链。

### 9.1 PlatformIO 工具链

#### 安装方式

PlatformIO 作为 Python 依赖通过 pip 安装（版本固定在 `requirements.txt` 中：`platformio==6.1.19`），安装 ESPHome 时自动安装。首次编译时，PlatformIO 会自动下载目标平台的编译器、框架等。

#### 默认存储路径

```
~/.platformio/                          # PlatformIO 核心目录（不可自定义 core_dir）
├── platforms/                          # 平台（如 espressif32）
├── packages/                           # 工具包（如 toolchain-xtensa32、framework-arduinoespressif32）
└── appstate.json                       # 设置文件（core_dir 不可自定义的原因）

<config_dir>/.esphome/<device_name>/    # ESPHome 构建目录
├── .pioenvs/                           # 构建输出（firmware.bin 等）
└── .piolibdeps/                        # 库依赖
```

- **Linux/macOS**: `~/.platformio`
- **Windows**: `%USERPROFILE%\.platformio`

#### 自定义选项

| 环境变量 | 说明 | 默认值 |
|----------|------|--------|
| `PLATFORMIO_BUILD_DIR` | 构建输出目录 | `<build_path>/.pioenvs` |
| `PLATFORMIO_LIBDEPS_DIR` | 库依赖目录 | `<build_path>/.piolibdeps` |

> **注意**：PlatformIO 的 `core_dir`（`~/.platformio`）**不可自定义**，因为其设置文件 `appstate.json` 存储在 core_dir 中。但可以自定义其下的 `platforms/`、`packages/`、`cache/` 子目录（Docker 环境中通过 `PLATFORMIO_PLATFORMS_DIR`、`PLATFORMIO_PACKAGES_DIR`、`PLATFORMIO_CACHE_DIR` 控制）。

### 9.2 ESP-IDF 工具链

#### 安装方式

ESP-IDF 工具链由 ESPHome **自行管理**，不依赖 PlatformIO。首次使用 ESP-IDF 工具链编译时，`check_esp_idf_install()` 自动执行：

1. **框架下载**：从 GitHub 镜像下载 ESP-IDF tar.xz 包并解压
2. **工具安装**：通过 `idf_tools.py install` 安装交叉编译器（xtensa-esp-elf）、cmake、ninja 等
3. **Python 环境**：创建独立 venv 并安装 IDF Python 依赖

#### 默认存储路径

```
<config_dir>/.esphome/idf/             # ESP-IDF 工具根目录 (IDF_TOOLS_PATH)
├── frameworks/
│   └── <version>/                     # IDF 框架源码（如 5.5.2/）
│       ├── tools/idf_tools.py
│       ├── components/
│       └── version.txt
├── penvs/
│   └── <version>/                     # Python 虚拟环境
│       ├── bin/python
│       └── lib/
├── tools/                             # 下载的编译器等工具
│   └── tools/
│       ├── xtensa-esp-elf/
│       ├── cmake/
│       └── ninja/
└── espidf.constraints.v<ver>.txt      # pip 约束文件
```

#### 自定义选项

| 环境变量 | 说明 | 默认值 |
|----------|------|--------|
| `ESPHOME_ESP_IDF_PREFIX` | ESP-IDF 工具根目录 | `<data_dir>/idf` |
| `ESPHOME_IDF_FRAMEWORK_MIRRORS` | 框架下载镜像 URL 列表 | GitHub esphome-libs 镜像 |
| `ESP_IDF_CONSTRAINTS_MIRRORS` | pip 约束文件 URL | dl.espressif.com |
| `IDF_PATH` | 已安装的 IDF 路径（若已设置则跳过安装） | 由 ESPHome 自动设置 |
| `IDF_TOOLS_PATH` | IDF 工具安装根目录 | 同 `ESPHOME_ESP_IDF_PREFIX` |

镜像 URL 支持模板替换：`{VERSION}`、`{MAJOR}`、`{MINOR}`、`{PATCH}`、`{EXTRA}`。

```bash
# 自定义 ESP-IDF 安装到 /opt/esphome-idf
export ESPHOME_ESP_IDF_PREFIX=/opt/esphome-idf
esphome compile device.yaml --toolchain esp-idf

# 使用国内镜像加速下载
export ESPHOME_IDF_FRAMEWORK_MIRRORS='https://mirrors.example.com/esp-idf/v{VERSION}/esp-idf-v{VERSION}.tar.xz'
esphome compile device.yaml --toolchain esp-idf

# 使用已安装的 ESP-IDF（跳过自动安装）
export IDF_PATH=/opt/esp/v5.5.2
esphome compile device.yaml --toolchain esp-idf
```

### 9.3 ESPHome 数据目录

`CORE.data_dir` 是 ESPHome 的核心数据目录，控制所有中间文件的存储位置：

| 环境 | 路径 |
|------|------|
| 本地开发（默认） | `<config_dir>/.esphome/` |
| 自定义 | `$ESPHOME_DATA_DIR` |
| Home Assistant 插件 | `/data/` |
| Docker | `/config/.esphome/` 或 `$ESPHOME_DATA_DIR` |

```bash
# 自定义 ESPHome 数据目录
export ESPHOME_DATA_DIR=/home/user/.local/share/esphome
```

### 9.4 完整路径结构示例

以本地开发环境为例，YAML 配置文件为 `/home/user/mydevice.yaml`：

```
/home/user/
├── mydevice.yaml                       # 用户配置文件
└── .esphome/                           # ESPHome 数据目录 (CORE.data_dir)
    ├── esphome.json                    # ESPHome 元数据
    ├── storage/mydevice.yaml.json      # 配置存储
    ├── idf/                            # ESP-IDF 工具根目录
    │   ├── frameworks/5.5.2/           # IDF 框架
    │   ├── penvs/5.5.2/               # Python venv
    │   └── tools/                      # 交叉编译器
    └── mydevice/                       # 构建目录 (CORE.build_path)
        ├── platformio.ini              # PlatformIO 项目文件
        ├── CMakeLists.txt              # ESP-IDF 项目文件
        ├── src/                        # 生成的 C++ 源码
        │   ├── main.cpp
        │   └── esphome.h
        ├── .pioenvs/mydevice/          # PIO 构建输出
        │   └── firmware.bin
        └── build/                      # IDF 构建输出
            └── firmware.factory.bin

~/.platformio/                          # PlatformIO 核心目录
├── platforms/espressif32/              # ESP32 平台
├── packages/
│   ├── toolchain-xtensa32/            # 交叉编译器
│   └── framework-arduinoespressif32/  # Arduino 框架
└── appstate.json
```

### 9.5 编译流程

```
compile_program()
├── PlatformIO 路径:
│   run_platformio_cli_run()
│   → python -m esphome.platformio.runner  [子进程，带补丁]
│   → platformio run
│
└── ESP-IDF 路径:
    check_esp_idf_install()              [首次运行：下载框架 + 工具 + Python 环境]
    → run_idf_py("reconfigure")          [组件发现]
    → write_project()                    [重新生成 CMakeLists.txt]
    → run_idf_py("build", "size")        [编译]
    → create_factory_bin()               [合并 bootloader + 分区表 + 固件]
    → create_ota_bin()                   [OTA 固件副本]
```

PlatformIO runner 对 PlatformIO 做了两项重要补丁：
- **patch_structhash()**：避免结构哈希变化导致完整重建，改用 mtime 检查
- **patch_file_downloader()**：为包下载添加指数退避重试（5 次）

ESP-IDF runner 使 `isatty()` 返回 True 以获取 TTY 格式的进度输出，并过滤嘈杂的 IDF/CMake/Ninja 输出。

---

## 十、设计哲学总结

### 10.1 配置即代码

YAML 配置 → C++ 固件 的全自动转换，用户无需编写任何 C++ 代码。代价是灵活性受限于组件开发者提供的 schema。

### 10.2 编译期裁剪

通过条件编译宏（`defines.h`）和模板元编程（`HasLoopOverride<T>`、`StaticVector`），ESPHome 在编译期就确定了组件数量和类型，避免了运行时开销，使固件在资源受限的 MCU 上也能高效运行。

### 10.3 确定性代码生成

Python 侧的伪协程系统保证相同 YAML 总是生成相同的 C++ 代码，使得增量编译成为可能。

### 10.4 观察者模式

`Controller` 基类 + `EntityBase` 的回调系统构成了经典的观察者模式。实体状态变化时自动通知 `APIServer`、`WebServer` 等控制器，无需组件代码显式推送。

### 10.5 X-macro 代码生成

C++ 侧使用 X-macro 技术消除实体类型相关的重复代码（注册方法、控制器回调、计数宏等），而 Python 侧也有对应的 `entity_helpers.py` 生成字符串查找表。

### 10.6 嵌入式友好的内存管理

- **Placement new**：避免堆碎片
- **StaticVector**：编译期固定大小的向量
- **StringRef**：零拷贝字符串引用
- **位域打包**：`EntityFlags` 仅占 1 字节
- **PROGMEM**：ESP8266 上字符串存储在 Flash 中

---

## 十一、关键文件索引

| 文件 | 作用 |
|------|------|
| `esphome/coroutine.py` | 伪协程调度系统 |
| `esphome/cpp_generator.py` | C++ 表达式/语句生成引擎 |
| `esphome/cpp_helpers.py` | 组件注册、GPIO 表达式等辅助 |
| `esphome/cpp_types.py` | C++ 类型声明（MockObj） |
| `esphome/core/__init__.py` | CORE 全局状态对象、ID 类 |
| `esphome/writer.py` | main.cpp 写入、源文件复制 |
| `esphome/config.py` | YAML 配置加载与验证 |
| `esphome/automation.py` | 自动化框架的 Python 侧注册 |
| `esphome/loader.py` | 组件动态加载 |
| `esphome/__main__.py` | CLI 命令行接口（所有子命令定义） |
| `esphome/core/component.h` | Component / PollingComponent 基类 |
| `esphome/core/entity_base.h` | EntityBase / StatefulEntityBase 基类 |
| `esphome/core/controller.h` | Controller 观察者基类 |
| `esphome/core/application.h` | Application 全局管理器 |
| `esphome/core/automation.h` | Trigger / Action / TemplatableFn |
| `esphome/core/scheduler.h` | 定时任务调度器 |
| `esphome/core/helpers.h` | CallbackManager、StaticVector 等工具 |
| `esphome/core/hal.h` | 硬件抽象层分发 |
| `esphome/core/entity_types.h` | X-macro 实体类型定义 |
| `esphome/core/defines.h` | 条件编译宏（自动生成） |
| `esphome/components/api/` | Native API 完整实现 |
| `esphome/components/api/api.proto` | Protobuf 服务定义 |
| `esphome/platformio/toolchain.py` | PlatformIO 编译调用 |
| `esphome/platformio/runner.py` | PlatformIO 补丁（structhash、重试） |
| `esphome/espidf/framework.py` | ESP-IDF 框架下载与安装 |
| `esphome/espidf/toolchain.py` | ESP-IDF 编译调用 |
| `esphome/espidf/runner.py` | ESP-IDF 输出过滤 |
| `esphome/core/preference_backend.h` | PreferenceBackend 接口 + ESPPreferenceObject + PreferencesMixin |
| `esphome/core/preferences.h` | 平台分发头文件 |
| `esphome/components/esp32/preferences.cpp` | ESP32 NVS 后端实现 |
| `esphome/components/esp8266/preferences.cpp` | ESP8266 RTC+Flash 后端实现 |
| `esphome/components/preferences/syncer.h` | IntervalSyncer 定时同步 |
