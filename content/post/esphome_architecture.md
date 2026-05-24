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
  - CONFIG_SCHEMA: 定义 YAML schema (使用 voluptuous 库)
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
| CORE | 100 | 各实体基类 (sensor, switch, light...) |
| COMMUNICATION | 60 | wifi, ethernet |
| BUS | 1 | i2c |
| COMPONENT | 0 | 默认优先级 |
| LATE | -100 | globals |
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
        # obj.Pset_name → MockObj("obj->set_name", "->")
    
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
alignas(ActualType) static unsigned char ns__id__pstorage[sizeof(ActualType)];
static BaseType *const id = reinterpret_cast<BaseType *>(ns__id__pstorage);
new(id) ActualType(constructor_args...);
```

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
├── get_state() = 0 — 获取当前状态
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

模板化的有状态实体基类，管理状态变化和回调分发：
- `Sensor : StatefulEntityBase<float>`
- `BinarySensor : StatefulEntityBase<bool>`
- `TextSensor : StatefulEntityBase<std::string>`

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

`APIServer` 继承 `Component` 和 `Controller`：

```cpp
class APIServer final : public Component, public Controller {
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
- `sensor/sensor.h`：`Sensor` 基类（继承 `StatefulEntityBase<float>` + `Component`）
- `dht/dht.h`：`DHTSensor` 继承 `Sensor`，实现 `update()` 读取硬件

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

### 5.6 构建系统

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
class BinarySensor : public StatefulEntityBase<bool>, public Component {
public:
    void publish_state(bool state);  // 发布新状态
    void add_filter(Filter *filter);
    // ...
};
```

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

## 八、设计哲学总结

### 8.1 配置即代码

YAML 配置 → C++ 固件 的全自动转换，用户无需编写任何 C++ 代码。代价是灵活性受限于组件开发者提供的 schema。

### 8.2 编译期裁剪

通过条件编译宏（`defines.h`）和模板元编程（`HasLoopOverride<T>`、`StaticVector`），ESPHome 在编译期就确定了组件数量和类型，避免了运行时开销，使固件在资源受限的 MCU 上也能高效运行。

### 8.3 确定性代码生成

Python 侧的伪协程系统保证相同 YAML 总是生成相同的 C++ 代码，使得增量编译成为可能。

### 8.4 观察者模式

`Controller` 基类 + `EntityBase` 的回调系统构成了经典的观察者模式。实体状态变化时自动通知 `APIServer`、`WebServer` 等控制器，无需组件代码显式推送。

### 8.5 X-macro 代码生成

C++ 侧使用 X-macro 技术消除实体类型相关的重复代码（注册方法、控制器回调、计数宏等），而 Python 侧也有对应的 `entity_helpers.py` 生成字符串查找表。

### 8.6 嵌入式友好的内存管理

- **Placement new**：避免堆碎片
- **StaticVector**：编译期固定大小的向量
- **StringRef**：零拷贝字符串引用
- **位域打包**：`EntityFlags` 仅占 1 字节
- **PROGMEM**：ESP8266 上字符串存储在 Flash 中

---

## 九、关键文件索引

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
