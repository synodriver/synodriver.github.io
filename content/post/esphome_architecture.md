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

ESPHome 的自动化系统（`esphome/core/automation.h` + `esphome/core/base_automation.h`）是声明式的触发-动作系统，用于实现事件驱动的控制逻辑。

#### 3.5.1 核心架构

```
┌───────────────────────────────────────────────────────────────────┐
│  Trigger<Ts...>                                                  │
│    trigger(x...) → Automation::trigger(x...)                     │
│                         │                                         │
│                         ▼                                         │
│  Automation<Ts...>                                               │
│    trigger(x...) → ActionList::play(x...)                        │
│                         │                                         │
│                         ▼                                         │
│  ActionList<Ts...>     (actions_ 指向链头)                        │
│    play(x...) → action1.play_complex(x...)                       │
│                         │                                         │
│    ┌────────────────────▼─────────────────────────────────────┐   │
│    │ Action 链 (单向链表，next_ 连接)                           │   │
│    │                                                            │   │
│    │ Action1 ──next_──> Action2 ──next_──> Action3 ──> nullptr │   │
│    │                                                            │   │
│    │ 每个 play_complex() 做三件事:                              │   │
│    │   1. num_running_++                                        │   │
│    │   2. play(x...)        ← 子类实现具体动作                  │   │
│    │   3. play_next_(x...)  ← 传递到下一个 Action               │   │
│    └────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────┘
```

核心类：
- **`Trigger<Ts...>`**：触发器，持有 `Automation*` 指针，`trigger()` 激活绑定的动作链
- **`Automation<Ts...>`**：绑定 Trigger 和 ActionList，`trigger()` 转发到 `ActionList::play()`
- **`ActionList<Ts...>`**：管理 Action 单向链表，`play()` 调用链头的 `play_complex()`
- **`Action<Ts...>`**：动作基类，通过 `next_` 指针形成链表

#### 3.5.2 Action 的三个核心方法

```cpp
template<typename... Ts> class Action {
 public:
  // 入口方法：驱动整个动作链执行
  virtual void play_complex(const Ts &...x) {
    this->num_running_++;      // (1) 标记运行中
    this->play(x...);          // (2) 执行本动作的具体逻辑
    this->play_next_(x...);    // (3) 传递给链中的下一个动作
  }

 protected:
  // 纯虚函数：子类必须实现，定义动作的具体行为
  virtual void play(const Ts &...x) = 0;

  // 链传递方法：将参数传递给下一个 Action
  void play_next_(const Ts &...x) {
    if (this->num_running_ > 0) {       // 确认此动作仍在运行（未被 stop 中止）
      this->num_running_--;              // 递减运行计数
      if (this->next_ != nullptr) {
        this->next_->play_complex(x...); // 递归调用下一个动作
      }
    }
  }

  Action<Ts...> *next_{nullptr};   // 单向链表指针
  int num_running_{0};             // 并发运行实例计数
};
```

**三个方法的职责**：

| 方法 | 调用者 | 职责 |
|------|--------|------|
| `play_complex()` | 上一个 Action 的 `play_next_()` 或 `ActionList::play()` | 运行计数管理 + 调度 `play()` 和 `play_next_()` |
| `play()` | `play_complex()` | 执行动作的具体逻辑（纯虚函数，子类必须实现） |
| `play_next_()` | `play_complex()` | 检查运行状态，将参数透传给链中下一个 Action |

#### 3.5.3 调用链的形成过程

**1. 构建阶段（setup 时由代码生成器构建）**：

```cpp
// Python 侧 build_automation() 生成等价代码：
auto automation = new Automation<SomeType>(trigger);

auto action1 = new SomeAction<SomeType>(...);
auto action2 = new AnotherAction<SomeType>(...);
auto action3 = new DelayAction<SomeType>(...);

// add_actions() 将 Action 串成单向链表
automation->add_actions({action1, action2, action3});
// 内部：action1->next_ = action2;  action2->next_ = action3;  action3->next_ = nullptr;
```

**2. 运行时调用链**：

```
some_trigger.trigger(x...)
  → Automation::trigger(x...)           // ESPHOME_ALWAYS_INLINE 强制内联
    → ActionList::play(x...)            // ESPHOME_ALWAYS_INLINE 强制内联
      → action1.play_complex(x...)
          num_running_++
          play(x...)                    // action1 的具体逻辑
          play_next_(x...)
            num_running_--
            → action2.play_complex(x...)
                num_running_++
                play(x...)              // action2 的具体逻辑
                play_next_(x...)
                  num_running_--
                  → action3.play_complex(x...)
                      ...（继续递归）
```

> `Trigger::trigger()` → `Automation::trigger()` → `ActionList::play()` 这三层转发全部被 `ESPHOME_ALWAYS_INLINE`（`__attribute__((always_inline))`）强制内联，编译后等价于直接调用第一个 Action 的 `play_complex()`。

**3. 参数透传**：触发器参数 `Ts...` 完整无损地从链头传到链尾，每次都是 const 引用传递。对于需要延迟调用的场景（如 DelayAction），参数被**按值捕获**到 lambda 中，因为原始引用可能在延迟后失效。

#### 3.5.4 同步动作 vs 异步动作

**模式 A：同步动作**（最常见）— 只实现 `play()`，不重写 `play_complex()`

基类的 `play_complex()` 在 `play()` 返回后立即调用 `play_next_()`，动作链顺序执行：

```cpp
// Switch TurnOnAction — 最简单的同步动作
template<typename... Ts> class TurnOnAction : public Action<Ts...> {
  void play(const Ts &...x) override { this->switch_->turn_on(); }
  Switch *switch_;
};

// Switch ControlAction — 使用 TEMPLATABLE_VALUE 的同步动作
template<typename... Ts> class ControlAction : public Action<Ts...> {
  TEMPLATABLE_VALUE(bool, state)        // 仅 4 字节（TemplatableFn）
  void play(const Ts &...x) override {
    auto state = this->state_.optional_value(x...);
    if (state.has_value()) this->switch_->control(*state);
  }
};
```

**模式 B：异步/复合动作** — 重写 `play_complex()`，延迟调用 `play_next_()`

这些动作不能同步完成，需要在某个条件满足后才继续链：

```cpp
// DelayAction — 延迟后通过调度器回调继续
void play_complex(const Ts &...x) override {
  this->num_running_++;
  // 注册定时器，延迟后回调 play_next_()
  if constexpr (sizeof...(Ts) == 0) {
    App.scheduler.set_timeout(this, "", this->delay_.value(),
      [this]() { this->play_next_(); });
  } else {
    auto f = [this, x...]() mutable { this->play_next_(x...); };  // 按值捕获参数
    App.scheduler.set_timeout(this, "", this->delay_.value(x...), std::move(f));
  }
}
void play(const Ts &...x) override { /* 空 - 逻辑在 play_complex 中 */ }
void stop() override { App.scheduler.cancel_timeout(this); }  // 取消定时器
```

```cpp
// IfAction — 条件分支，子链通过 ContinuationAction 回到主链
void play_complex(const Ts &...x) override {
  this->num_running_++;
  if (this->condition_->check(x...)) {
    this->then_.play(x...);    // 执行 then 子链，末尾的 ContinuationAction 会调用 play_next_
    return;                     // 不直接 play_next_，等子链完成
  } else if constexpr (HasElse) {
    this->else_.play(x...);    // 执行 else 子链
    return;
  }
  this->play_next_(x...);      // 无匹配子链时直接继续
}
```

#### 3.5.5 ContinuationAction — 子链回到主链的桥梁

分支动作（If/While/Repeat）的子链末尾都有一个 `ContinuationAction`，其 `play()` 调用父动作的 `play_next_()`，使控制流从子链无缝回到主链：

```cpp
template<typename... Ts> class ContinuationAction : public Action<Ts...> {
 public:
  explicit ContinuationAction(Action<Ts...> *parent) : parent_(parent) {}
  void play(const Ts &...x) override { this->parent_->play_next_(x...); }
 protected:
  Action<Ts...> *parent_;   // 仅 4/8 字节
};
```

**IfAction 的链结构**：
```
IfAction → [then_: Action1 → Action2 → ContinuationAction(this)] → ActionAfterIf
         → [else_: Action3 → ContinuationAction(this)]            ↗
```

**WhileAction 的循环机制**：子链末尾的 `WhileLoopContinuation` 检查条件——条件为 true 则重新执行子链，为 false 则调用 `parent_->play_next_()` 退出循环：

```cpp
template<typename... Ts> void WhileLoopContinuation<Ts...>::play(const Ts &...x) {
  if (this->parent_->num_running_ > 0 && this->parent_->condition_->check(x...)) {
    this->parent_->then_.play(x...);   // 条件仍满足，重新执行子链（循环）
  } else {
    this->parent_->play_next_(x...);   // 条件不满足，继续主链
  }
}
```

**RepeatAction 的参数注入**：子链类型是 `ActionList<uint32_t, Ts...>`，每次迭代将当前迭代号作为第一个参数传入，子链中的动作可以使用这个迭代号。

#### 3.5.6 异步实现机制

ESPHome **不使用** coroutine/yield，异步通过两种机制实现：

| 机制 | 使用者 | 原理 |
|------|--------|------|
| **Scheduler 定时器** | DelayAction | `play_complex()` 注册定时器 → 等待 → 定时器回调 `play_next_()` |
| **Component::loop() 轮询** | WaitUntilAction, ScriptWaitAction | `play_complex()` 存入队列 + `enable_loop()` → 每 loop() 检查条件 → 条件满足时 `play_next_tuple_()` + `disable_loop()` |

WaitUntilAction 继承 `Component` 以使用 `loop()` 机制，使用队列存储等待中的参数（支持并发），按需启用/禁用 loop 以避免空转开销：

```cpp
// WaitUntilAction 简化逻辑
void play_complex(const Ts &...x) override {
  this->num_running_++;
  if (this->condition_->check(x...)) {
    this->play_next_(x...);   // 条件已满足，直接继续
  } else {
    this->var_queue_.emplace_back(millis(), timeout, std::make_tuple(x...));
    this->enable_loop();       // 启用 loop() 轮询
  }
}
void loop() override {
  // 每次主循环迭代检查队列中的等待项
  if (!this->process_queue_(App.get_loop_component_start_time()))
    this->disable_loop();      // 队列空了，停止轮询
}
```

#### 3.5.7 num_running_ 并发控制

`num_running_` 计数器支持同一动作链的多次并行触发。例如，一个按钮的 `on_press` 触发器被快速连按两次：

```
第1次触发：action1.num_running_ = 1 → action2.num_running_ = 1 → ...
第2次触发：action1.num_running_ = 2 → action2.num_running_ = 2 → ...
```

每次 `play_next_()` 递减计数，`stop_complex()` 直接将 `num_running_` 置 0 中止所有实例。DelayAction 使用 `skip_cancel` 参数控制并行实例的行为——当 `num_running_ > 1` 时，新延迟不会取消正在运行的旧延迟。

#### 3.5.8 TemplatableValue / TemplatableFn — 4 字节优化

`TEMPLATABLE_VALUE(type, name)` 宏为 Action 子类生成"可模板化"的值成员，支持常量值或 lambda 表达式：

```cpp
#define TEMPLATABLE_VALUE(type, name) \
 protected: TemplatableStorage<type, Ts...> name##_{}; \
 public: template<typename V> void set_##name(V name) { this->name##_ = name; }
```

`TemplatableStorage` 根据类型自动选择存储方式：
- **Trivially copyable 类型**（int, float, bool 等）→ `TemplatableFn`：仅 4 字节，只存函数指针
- **非 trivial 类型**（std::string 等）→ `TemplatableValue`：8 字节，存值或函数指针

Python 代码生成器将常量包装为无状态 lambda（可转为函数指针），避免 `std::function` 的 32 字节开销。LightControlAction 更进一步，将所有字段设置打包成单个 `ApplyFn` 函数指针，无论配置了多少字段，Action 对象始终只占 8 字节。

#### 3.5.9 TriggerForwarder — 回调式自动化构建

新式自动化构建不再创建 Trigger 对象，而是直接将 Automation 注册为回调：

```cpp
template<typename... Ts> struct TriggerForwarder {
  Automation<Ts...> *automation;
  void operator()(const Ts &...args) const { this->automation->trigger(args...); }
};
static_assert(sizeof(TriggerForwarder<>) <= sizeof(void *));  // 可内联存储在 Callback::ctx_ 中
```

特化前向器提供条件触发：
- `TriggerOnTrueForwarder`：仅当 bool 参数为 true 时触发
- `TriggerOnFalseForwarder`：仅当 bool 参数为 false 时触发

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
| `esphome/core/automation.h` | Trigger / Action / Automation / ActionList / TemplatableFn |
| `esphome/core/base_automation.h` | DelayAction / IfAction / WhileAction / RepeatAction / WaitUntilAction / ContinuationAction |
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

---

## 十二、StateClass 状态类别体系

### 12.1 概述

`StateClass` 是 ESPHome 传感器（`sensor::Sensor`）的核心属性之一，用于告诉 Home Assistant **该传感器数值的本质类型**——是"瞬时测量值"还是"累积总量"，是"只会增长"还是"可增可减"。这一属性直接影响 Home Assistant 的**长期统计（Long-term Statistics）**行为：哪些传感器会被纳入统计、统计的聚合方式（取均值还是求和）、以及如何处理数值重置。

`StateClass` 的定义位于 `esphome/components/sensor/sensor.h`：

```cpp
enum StateClass : uint8_t {
  STATE_CLASS_NONE = 0,
  STATE_CLASS_MEASUREMENT = 1,
  STATE_CLASS_TOTAL_INCREASING = 2,
  STATE_CLASS_TOTAL = 3,
  STATE_CLASS_MEASUREMENT_ANGLE = 4
};
constexpr uint8_t STATE_CLASS_LAST = static_cast<uint8_t>(STATE_CLASS_MEASUREMENT_ANGLE);
```

### 12.2 历史演变：从 last_reset 到 state_class

在 2021.9 版本之前，Home Assistant 使用 `last_reset` 属性来区分累积量的重置时刻。该机制要求传感器在累积值重置时主动发送 `last_reset` 时间戳，但实践中多数传感器无法提供准确的重置时刻，导致统计数据混乱。

ESPHome 的 `api.proto` 中保留了已废弃的 `SensorLastResetType`（API 版本 1.5 后弃用）：

```protobuf
// Deprecated in API version 1.5
enum SensorLastResetType {
  option deprecated = true;
  LAST_RESET_NONE = 0;
  LAST_RESET_NEVER = 1;
  LAST_RESET_AUTO = 2;
}
```

`state_class` 取代了 `last_reset`，将"重置检测"的责任从传感器端转移到 Home Assistant 端——HA 根据 `state_class` 的语义自动判断数值是否发生了重置，无需传感器显式报告。

### 12.3 五种 StateClass 详解

#### STATE_CLASS_NONE（值：0）

**含义**：不声明状态类别。传感器不参与 Home Assistant 的长期统计系统。

**适用场景**：
- 状态值不是数值，或数值含义模糊、不适合统计聚合
- 仅关注当前值、无需历史趋势的传感器
- 用户手动覆盖了组件默认的 state_class，设置为空

**Home Assistant 行为**：不生成长期统计数据。HA 的历史图表仅展示实时状态记录（`short-term statistics` 不生成）。

**典型组件**：大多数未明确设置 `state_class` 的传感器默认为 `NONE`。例如一些文本类型的传感器、调试用的诊断传感器等。

#### STATE_CLASS_MEASUREMENT（值：1）

**含义**：**瞬时测量值**。值代表当前时刻的观测读数，每个读数独立且有意义——今天 25°C 和明天 25°C 是两个独立的测量。

**核心特征**：
- 值**可增可减**，前后两次读数之间没有严格的单调关系
- 读数的统计聚合方式为**取均值**（mean）——5 分钟统计周期内所有读数的平均值
- 两次读数之间可能存在较大的波动（如温度从 30°C 降到 15°C，这不是"重置"）

**Home Assistant 统计行为**：
| 统计指标 | 计算方式 |
|----------|----------|
| mean | 5分钟内所有读数的平均值 |
| min | 5分钟内最小读数 |
| max | 5分钟内最大读数 |
| last | 5分钟内最后一个读数 |

**典型 ESPHome 组件**（使用最多）：

| 组件 | 传感器 | 说明 |
|------|---------|------|
| dht | 温度、湿度 | 环境温湿度瞬时读数 |
| bme280 | 温度、湿度、气压 | 环境测量 |
| adc | 电压 | ADC 采样瞬时值 |
| ade7953_base | 电流、电压、功率 | 电力测量 |
| gps | 速度、海拔 | GPS 瞬时速度/高度 |
| 各类距离传感器 | 超声波/激光测距 | 瞬时距离读数 |

**YAML 示例**：
```yaml
sensor:
  - platform: dht
    temperature:
      name: "Temperature"
      state_class: measurement    # 温度是瞬时测量值
```

> 在 ESPHome 源码中，`measurement` 是使用最广泛的 state_class——绝大多数传感器组件（温度、湿度、气压、电压、电流、功率、距离、速度等）都默认使用它。搜索源码中 `state_class=STATE_CLASS_MEASUREMENT` 的出现超过数百处。

#### STATE_CLASS_TOTAL_INCREASING（值：2）

**含义**：**单调递增的累积量**。值代表从某个起点开始的总量，只会增加（或重置后重新增加），永远不会减少。

**核心特征**：
- 值**只会增加或重置归零**，不会自然减少
- 两次读数之间的**差值**才是有意义的信息（如每小时消耗了多少电量）
- 如果读数突然变小，Home Assistant **自动判断为重置**（meter reset），并将该值视为新周期的起点，不当作"减少"
- 统计聚合方式为**求和**（sum）——5 分钟统计周期内所有增量之和

**Home Assistant 统计行为**：
| 统计指标 | 计算方式 |
|----------|----------|
| sum | 5分钟内增量之和 = Σ(next - prev)，遇到重置时仅计入重置后的值 |
| last | 5分钟内最后一个读数 |

**重置检测逻辑**：当 HA 发现新读数 < 上一读数时，认为发生了重置（如电表换表、固件重启后计数归零），此时：
- 不将差值计入 sum（避免负数）
- 将新读数视为新周期起点

**典型 ESPHome 组件**：

| 组件 | 传感器 | 说明 |
|------|---------|------|
| cse7766 | 累计电量 | 电量计单方向累计 |
| bl0942/bl0940/bl0939/bl0910/bl6552 | 累计电量(kWh) | 各型号电量计 |
| atm90e26/atm90e32 | 累计电量 | ATM90 系列电量计 |
| ade7880 | 累计电量 | ADE7880 电量计 |
| dsmr | 累计电量/气量 | 荷兰智能电表协议 |
| dlms_meter | 累计电量 | DLMS 协议电量计 |
| duty_time | 累计运行时间 | 设备累计活跃时长 |

**YAML 示例**：
```yaml
sensor:
  - platform: cse7766
    energy:
      name: "Total Energy"
      state_class: total_increasing    # 电量只会增加或重置归零
```

> `total_increasing` 主要用于**能源计量**场景——电量(kWh)、气量(m³)等。这些值的物理本质是"只会越用越多"的单向累积量。

#### STATE_CLASS_TOTAL（值：3）

**含义**：**可增可减的累积量**。值代表当前净总量，但总量本身可以减少（如电池充电后又放电）。

**核心特征**：
- 值**可增可减**——总量本身可能上升或下降
- 值的变化被视为"真实的总量变化"，而非"重置"
- 如果读数突然变小，Home Assistant **不会自动判断为重置**——它认为是总量真的减少了
- 统计聚合方式为**求和**（sum），但含义是"净变化量"而非"消耗量"

**Home Assistant 统计行为**：
| 统计指标 | 计算方式 |
|----------|----------|
| sum | 5分钟内净变化量之和 |
| last | 5分钟内最后一个读数 |

**与 `total_increasing` 的关键区别**：
```
total_increasing:  前值=1000 → 新值=50  → HA 判断为重置，不计负差值
total:             前值=1000 → 新值=50  → HA 认为总量真的减少了，差值 -950 计入统计
```

这意味着 `total` 不适合用于"只会增加"的消耗量——如果传感器重启后归零，HA 会将归零视为真实减少，统计数据会出现大幅负值。

**典型 ESPHome 组件**：

| 组件 | 传感器 | 说明 |
|------|---------|------|
| duty_time | 上次运行时长(last_time) | 上次激活的持续时间，可被重置 |

**YAML 示例**（duty_time 组件）：
```yaml
sensor:
  - platform: duty_time
    id: my_duty_time
    last_time:          # 上次运行时长 — 可被 reset 清零
      name: "Last Duty Time"
      state_class: total      # total: 值可增可减，重置不是"归零重计"
```

> 注意：同一个 duty_time 组件中，**累计运行时间**使用 `total_increasing`（只会增加），而**上次运行时长**使用 `total`（可以被 reset 动作清零后重新计时）。这完美展示了两种 state_class 的语义区别。

> `total` 在 ESPHome 组件中的使用极其罕见——整个源码中仅有 `duty_time` 的 `last_time` 子传感器使用了它。大多数"只会增加"的累积量应该使用 `total_increasing`，而非 `total`。

#### STATE_CLASS_MEASUREMENT_ANGLE（值：4）

**含义**：**角度瞬时测量值**。与 `measurement` 语义相同（瞬时读数），但专门标注为角度值。

**核心特征**：
- 值是**角度/方位**的瞬时读数（如航向角、经度）
- 统计聚合方式为**取均值**（mean），与 `measurement` 相同
- Home Assistant 在处理长期统计时，对角度值采用**环绕均值**（circular mean）算法——例如 350° 和 10° 的均值不是 180°，而是 0°（360°）
- 当前仅支持**度（degrees）**单位

**Home Assistant 统计行为**：
| 统计指标 | 计算方式 |
|----------|----------|
| mean | 环绕均值（考虑角度的周期性） |
| min | 5分钟内最小角度 |
| max | 5分钟内最大角度 |

**环绕均值示意**：
```
普通均值:  avg(350°, 10°) = (350 + 10) / 2 = 180°  ← 错误！
环绕均值: avg(350°, 10°) = 0°                       ← 正确！角度跨越了 360° 周期
```

**典型 ESPHome 组件**：

| 组件 | 传感器 | 说明 |
|------|---------|------|
| gps | 经度(latitude)、航向(course) | GPS 方位/航向角度值 |

**YAML 示例**：
```yaml
sensor:
  - platform: gps
    latitude:
      name: "Latitude"
      state_class: measurement_angle    # 经度是角度测量值
    course:
      name: "Course"
      state_class: measurement_angle    # 航向角是角度测量值
```

> `measurement_angle` 是 ESPHome 中使用最少的 state_class——仅 `gps` 组件使用了它。这是因为角度值具有周期性（0° = 360°），普通均值会产生荒谬结果，HA 专门为此设计了环绕均值算法。

### 12.4 StateClass 的传输链路

StateClass 从 ESPHome YAML 配置到 Home Assistant 的完整传输路径：

```
YAML 配置                     Python 侧                      C++ 侧                    API 传输                    Home Assistant
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
state_class: measurement  →  STATE_CLASSES映射              →  Sensor::state_class_    →  SensorStateClass枚举       →  SensorStateClass.MEASUREMENT
                              "" → NONE                        (uint8_t)                  (Protobuf field 10)          → 长期统计: 取均值
                              "measurement" → 1                                           → ListEntitiesSensorResponse
                              "total_increasing" → 2                                       → APIConnection::list_entities
                              "total" → 3
                              "measurement_angle" → 4
```

**Python 侧注册链路**（`sensor/__init__.py`）：

```python
# YAML → Python 枚举映射
STATE_CLASSES = {
    "": StateClasses.STATE_CLASS_NONE,
    "measurement": StateClasses.STATE_CLASS_MEASUREMENT,
    "total_increasing": StateClasses.STATE_CLASS_TOTAL_INCREASING,
    "total": StateClasses.STATE_CLASS_TOTAL,
    "measurement_angle": StateClasses.STATE_CLASS_MEASUREMENT_ANGLE,
}

# sensor_schema() 中设置默认 state_class
def sensor_schema(class_, *, state_class=...):
    schema[cv.Optional(CONF_STATE_CLASS, default=default)] = validate_state_class

# setup_sensor_core_() 中生成代码
if (state_class := config.get(CONF_STATE_CLASS)) is not None:
    cg.add(var.set_state_class(state_class))  # → C++: sensor->set_state_class(STATE_CLASS_MEASUREMENT)
```

**C++ 侧存储**（`sensor.h`）：

```cpp
class Sensor : public EntityBase {
  StateClass state_class_{STATE_CLASS_NONE};  // 默认未设置
  struct SensorFlags {
    uint8_t has_state_class_override : 1;     // 位域标记：是否有手动覆盖
    ...
  } sensor_flags_{};
};

// get_state_class()：有覆盖则返回覆盖值，否则返回 NONE
StateClass Sensor::get_state_class() {
  if (this->sensor_flags_.has_state_class_override)
    return this->state_class_;
  return StateClass::STATE_CLASS_NONE;
}
```

**API 传输**（`api_connection.cpp`）：

```cpp
// 实体发现时将 state_class 编码为 Protobuf 枚举
msg.state_class = static_cast<enums::SensorStateClass>(sensor->get_state_class());
```

Protobuf 定义（`api.proto`）：

```protobuf
enum SensorStateClass {
  STATE_CLASS_NONE = 0;
  STATE_CLASS_MEASUREMENT = 1;
  STATE_CLASS_TOTAL_INCREASING = 2;
  STATE_CLASS_TOTAL = 3;
  STATE_CLASS_MEASUREMENT_ANGLE = 4;
}

message ListEntitiesSensorResponse {
  ...
  SensorStateClass state_class = 10;
  SensorLastResetType legacy_last_reset_type = 11 [deprecated=true];  // API 1.5 后弃用
  ...
}
```

**MQTT 传输**（`mqtt_sensor.cpp`）：

```cpp
// MQTT 发现消息中包含 state_class 属性
if (this->sensor_->get_state_class() != STATE_CLASS_NONE) {
  root[MQTT_STATE_CLASS] = state_class_to_string(this->sensor_->get_state_class());
}
```

### 12.5 选择指南：如何为传感器选择正确的 StateClass

```
                    该传感器的值是什么性质？
                           │
                ┌──────────┴──────────┐
                │                     │
          瞬时测量值？            累积总量？
                │                     │
          ┌─────┴─────┐        ┌─────┴─────┐
          │           │        │           │
       一般测量     角度测量   单调递增     可增可减
          │           │        │           │
    measurement   measurement_angle  total_increasing   total
          │           │        │           │
          │           │        │           │
      温度/湿度     经度/航向  电量(kWh)   净能量(充-放)
      气压/电压     方位角     气量(m³)    上次时长
      电流/功率               水量(L)      (可被reset)
      距离/速度               运行时间
```

**决策清单**：

| 问题 | 如果"是" → | 如果"否" → |
|------|------------|------------|
| 值代表某个时间点的瞬时读数？ | `measurement` 或 `measurement_angle` | 继续判断 |
| 值有周期性（0°=360°）？ | `measurement_angle` | `measurement` |
| 值代表从起点开始的累积量？ | `total_increasing` 或 `total` | `measurement` |
| 累积量只会增加（或重置归零）？ | `total_increasing` | `total` |
| 累积量可能自然减少（非重置）？ | `total` | `total_increasing` |
| 不关心长期统计？ | `none` | 根据以上判断 |

> **常见错误**：将"只会增加"的累积量设为 `total` 而非 `total_increasing`。这会导致传感器重启归零时，HA 将归零视为真实减少，统计数据出现大幅负值。**绝大多数电表、气表等消耗量应该使用 `total_increasing`**。

### 12.6 组件开发中的 StateClass 使用模式

ESPHome 组件开发中，`state_class` 通常通过 `sensor_schema()` 的默认值参数设置，而非要求用户在 YAML 中手动指定：

```python
# 组件定义传感器 schema 时预设 state_class
CONFIG_SCHEMA = cv.All(
    sensor.sensor_schema(
        DutyTimeSensor,
        state_class=STATE_CLASS_TOTAL_INCREASING,   # 默认值
        unit_of_measurement=UNIT_SECOND,
        device_class=DEVICE_CLASS_DURATION,
    )
    .extend(cv.polling_component_schema("60s"))
)
```

这遵循了 ESPHome 的设计哲学：**组件开发者根据传感器物理含义设置合理的默认 state_class**，用户通常不需要手动指定。如果用户需要覆盖，YAML 中 `state_class` 是 `Optional` 字段：

```yaml
sensor:
  - platform: dht
    temperature:
      name: "Room Temperature"
      state_class: measurement    # 通常无需指定，组件已设默认值
```

### 12.7 StateClass 在 Home Assistant 中的影响总结

| StateClass | 长期统计 | 聚合方式 | 重置检测 | 适用数值性质 |
|------------|---------|----------|----------|-------------|
| NONE | ❌ 不生成 | — | — | 无需统计 |
| MEASUREMENT | ✅ 生成 | mean/min/max | 无（值的变化都是真实的） | 瞬时测量值 |
| TOTAL_INCREASING | ✅ 生成 | sum/last | ✅ 自动（值变小=重置） | 单调递增累积量 |
| TOTAL | ✅ 生成 | sum/last | ❌ 不检测（值的变化都是真实的） | 可增可减累积量 |
| MEASUREMENT_ANGLE | ✅ 生成 | 环绕mean/min/max | 无 | 角度瞬时测量值 |

### 12.8 关键文件索引

| 文件 | 作用 |
|------|------|
| `esphome/components/sensor/sensor.h` | StateClass 枚举定义 + Sensor 类 state_class_ 成员 |
| `esphome/components/sensor/sensor.cpp` | state_class_to_string() + get/set_state_class() 实现 |
| `esphome/components/sensor/__init__.py` | STATE_CLASSES 映射表 + validate_state_class + sensor_schema() |
| `esphome/components/api/api.proto` | SensorStateClass Protobuf 枚举 + ListEntitiesSensorResponse 中的 state_class 字段 |
| `esphome/components/api/api_connection.cpp` | 实体发现时 state_class 编码传输 |
| `esphome/components/mqtt/mqtt_sensor.cpp` | MQTT 发现消息中 state_class 属性 |
| `esphome/components/duty_time/sensor.py` | 唯一使用 STATE_CLASS_TOTAL 的组件（与 TOTAL_INCREASING 对比） |
| `esphome/components/gps/__init__.py` | 唯一使用 STATE_CLASS_MEASUREMENT_ANGLE 的组件 |
| `esphome/const.py` | STATE_CLASS_* 常量字符串定义 |

---

## 十三、Dashboard Web UI 架构

### 13.1 概述

ESPHome Dashboard 是一个基于 Tornado 的 Web 服务器，提供图形化的设备管理界面。用户可以通过浏览器完成设备的创建、编辑、编译、上传、日志查看等全流程操作，无需手动在终端执行 CLI 命令。

Dashboard 的核心设计思想是：**Web UI 本身不执行编译/上传等核心逻辑，而是通过子进程调用 `esphome` CLI 命令**。这使得 Web 层和核心编译层完全解耦——Dashboard 负责 HTTP/WebSocket 交互和设备状态展示，CLI 负责实际的配置验证、代码生成和编译。

```
┌──────────────────────────────────────────────────────────────────┐
│                        浏览器                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ 设备列表     │  │ YAML编辑器   │  │ 编译/上传    │           │
│  │ (devices)    │  │ (ace/vscode) │  │ (WebSocket)  │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└───────────────────────────────┬──────────────────────────────────┘
                                │ HTTP / WebSocket
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Tornado Web Server                            │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  make_app() → tornado.web.Application                     │  │
│  │  ├── REST API 路由 (30+ 个)                               │  │
│  │  ├── WebSocket 路由                                       │  │
│  │  │   ├── EsphomeCommandWebSocket (编译/上传/日志等)       │  │
│  │  │   └── DashboardEventsWebSocket (实时状态推送)          │  │
│  │  └── 静态文件 (esphome-dashboard npm 包)                  │  │
│  └────────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  ESPHomeDashboard (全局单例 DASHBOARD)                     │  │
│  │  ├── DashboardEntries (设备列表管理 + 文件变更检测)       │  │
│  │  ├── EventBus (事件总线 → WebSocket推送)                  │  │
│  │  ├── MDNSStatus (mDNS 设备发现)                          │  │
│  │  ├── PingStatus (ICMP ping 设备在线检测)                 │  │
│  │  ├── MqttStatusThread (MQTT 设备发现，可选)              │  │
│  │  ├── DNSCache (DNS 解析缓存)                             │  │
│  │  └── DashboardSettings (认证/配置目录/模式)              │  │
│  └────────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬──────────────────────────────────┘
                                │ subprocess.Popen / tornado.process.Subprocess
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│                    esphome CLI 子进程                             │
│  python -m esphome --dashboard <command> <config> [--device X]  │
│                                                                  │
│  典型调用:                                                       │
│  • compile  → python -m esphome --dashboard compile my.yaml     │
│  • run      → python -m esphome --dashboard run my.yaml -d OTA  │
│  • upload   → python -m esphome --dashboard upload my.yaml -d /dev/ttyUSB0 │
│  • logs     → python -m esphome --dashboard logs my.yaml -d OTA │
│  • config   → python -m esphome config my.yaml --show-secrets   │
│  • clean    → python -m esphome --dashboard clean my.yaml       │
│  • rename   → python -m esphome --dashboard rename my.yaml newname │
│  • update-all → python -m esphome --dashboard update-all /config/ │
│  • vscode  → python -m esphome -q vscode dummy                  │
│  • vscode --ace → python -m esphome -q vscode --ace /config/    │
└──────────────────────────────────────────────────────────────────┘
```

### 13.2 启动流程

Dashboard 的启动入口在 `__main__.py` 的 `command_dashboard()` 函数，最终调用 `dashboard/dashboard.py` 的 `start_dashboard()`：

```
用户执行: esphome dashboard /path/to/configs/
    ↓
__main__.py → command_dashboard(args)
    ↓
dashboard/dashboard.py → start_dashboard(args)
    ↓  1. settings.parse_args(args) — 解析认证/端口/地址等参数
    ↓  2. 加载 cookie_secret（认证时从 EsphomeStorageJSON 读取）
    ↓  3. asyncio.set_event_loop_policy(DashboardEventLoopPolicy)
    ↓
async_start(args)
    ↓  1. DASHBOARD.async_setup() — 初始化 DashboardEntries、加载 ignored_devices
    ↓  2. start_web_server(make_app(), sock, addr, port, config_dir)
    ↓  3. 可选: webbrowser.open() — 自动打开浏览器
    ↓
DASHBOARD.async_run() — 持续运行直到 KeyboardInterrupt
    ↓  1. entries.async_update_entries() — 首次加载所有 YAML 设备
    ↓  2. MDNSStatus.async_setup() → mDNS 初始化
    ↓  3. asyncio.create_task(mdns_status.async_run()) — 启动 mDNS 发现循环
    ↓  4. 7.5秒后启动 PingStatus — 等 mDNS bootstrap 完成
    ↓  5. 可选: MqttStatusThread.start() — MQTT 发现线程
    ↓  6. await asyncio.Event().wait() — 阻塞直到关机信号
```

**DashboardEventLoopPolicy** 是一个自定义的 asyncio 事件循环策略，灵感来自 Home Assistant：
- 使用 `PidfdChildWatcher`（Linux）或 `ThreadedChildWatcher`（其他平台）监控子进程
- `ThreadPoolExecutor` 限制最大 48 个工作线程（`MAX_EXECUTOR_WORKERS`）
- `loop.time` 直接绑定 `time.monotonic`，减少方法调用开销（此方法是事件循环中调用最频繁的）
- 自定义异常处理器 `_async_loop_exception_handler`

### 13.3 REST API 路由总览

Dashboard 的所有路由定义在 `make_app()` 函数中（`web_server.py:1570`），以 Tornado 的路由表形式注册。所有路径前缀可通过 `ESPHOME_DASHBOARD_RELATIVE_URL` 环境变量自定义（默认 `/`）。

#### 13.3.1 页面路由

| 路径 | Handler | 方法 | 说明 |
|------|---------|------|------|
| `/` | MainRequestHandler | GET | 主页面，渲染 `index.template.html` |
| `/login` | LoginHandler | GET/POST | 登录页面（认证时） |
| `/logout` | LogoutHandler | GET | 登出，清除认证 cookie |

#### 13.3.2 设备管理 API

| 路径 | Handler | 方法 | 说明 |
|------|---------|------|------|
| `/devices` | ListDevicesHandler | GET | 返回所有已配置和可导入设备的 JSON 列表 |
| `/info` | InfoRequestHandler | GET | 返回指定设备的 StorageJSON 信息（配置、版本等） |
| `/edit` | EditRequestHandler | GET/POST | GET：读取 YAML 文件内容；POST：写入 YAML 文件内容 |
| `/delete` / `/archive` | ArchiveRequestHandler | POST | 将设备配置移至 archive 目录 |
| `/undo-delete` / `/unarchive` | UnArchiveRequestHandler | POST | 从 archive 目录恢复设备配置 |
| `/rename` | EsphomeRenameHandler | WS | 重命名设备（修改 YAML + 重新编译上传） |
| `/wizard` | WizardRequestHandler | POST | 创建新设备配置（basic/upload/empty 三种模式） |
| `/import` | ImportRequestHandler | POST | 导入发现的设备（从 mDNS 或项目链接） |
| `/ignore-device` | IgnoreDeviceRequestHandler | POST | 忽略/取消忽略可导入设备 |

#### 13.3.3 编译与上传 WebSocket

| 路径 | Handler | 类型 | 实际 CLI 命令 |
|------|---------|------|---------------|
| `/compile` | EsphomeCompileHandler | WS | `esphome --dashboard compile <yaml>` |
| `/upload` | EsphomeUploadHandler | WS | `esphome --dashboard upload <yaml> --device <port>` |
| `/run` | EsphomeRunHandler | WS | `esphome --dashboard run <yaml> --device <port>` |
| `/logs` | EsphomeLogsHandler | WS | `esphome --dashboard logs <yaml> --device <port>` |
| `/validate` | EsphomeValidateHandler | WS | `esphome config <yaml> [--show-secrets]` |
| `/clean` | EsphomeCleanHandler | WS | `esphome --dashboard clean <yaml>` |
| `/clean-mqtt` | EsphomeCleanMqttHandler | WS | `esphome --dashboard clean-mqtt <yaml>` |
| `/clean-all` | EsphomeCleanAllHandler | WS | `esphome --dashboard clean-all [config_dir]` |
| `/update-all` | EsphomeUpdateAllHandler | WS | `esphome --dashboard update-all <config_dir>` |
| `/vscode` | EsphomeVscodeHandler | WS | `esphome -q vscode dummy` |
| `/ace` | EsphomeAceEditorHandler | WS | `esphome -q vscode --ace <config_dir>` |

> `/vscode` 和 `/ace` 两个端点并非真正的 VSCode 启动，而是调用 `esphome vscode` 命令生成**验证/补全数据**——前端 Ace Editor 或 VSCode 使用这些数据提供 YAML 语法验证和自动补全。

#### 13.3.4 下载与查询 API

| 路径 | Handler | 方法 | 说明 |
|------|---------|------|------|
| `/downloads` | DownloadListRequestHandler | GET | 获取指定设备的可下载文件类型列表（固件、分区表等） |
| `/download.bin` | DownloadBinaryRequestHandler | GET | 下载指定设备的固件/分区表等二进制文件，支持 gzip 压缩 |
| `/serial-ports` | SerialPortRequestHandler | GET | 返回可用串口列表（含 OTA 虚拟端口） |
| `/ping` | PingRequestHandler | GET | 触发 ping 检查，返回各设备在线状态 |
| `/version` | EsphomeVersionHandler | GET | 返回 ESPHome 版本号 |
| `/secret_keys` | SecretKeysRequestHandler | GET | 返回 secrets.yaml 中定义的密钥名列表 |
| `/json-config` | JsonConfigRequestHandler | GET | 调用 CLI 解析 YAML 并返回 JSON 格式配置 |
| `/boards/<platform>` | BoardsRequestHandler | GET | 返回指定平台（esp32/esp8266/rp2040 等）的开发板列表 |
| `/prometheus-sd` | PrometheusServiceDiscoveryHandler | GET | 返回 Prometheus 服务发现格式的设备列表 |
| `/events` | DashboardEventsWebSocket | WS | 实时事件推送 WebSocket |

#### 13.3.5 静态资源

| 路径 | Handler | 说明 |
|------|---------|------|
| `/static/(.*)` | StaticFileHandler | 前端静态文件（JS/CSS/图标），来自 `esphome-dashboard` npm 包 |

> 前端静态文件由独立 npm 包 `esphome-dashboard`（版本 `20260425.0`）提供。Dashboard 通过 `import esphome_dashboard; esphome_dashboard.where()` 获取包路径。开发模式下通过 `ESPHOME_DASHBOARD_DEV` 环境变量指向本地前端项目。

### 13.4 WebSocket 通信机制

Dashboard 使用两种 WebSocket 机制：**命令型 WebSocket** 和 **事件型 WebSocket**。

#### 13.4.1 命令型 WebSocket — EsphomeCommandWebSocket

所有编译/上传/日志等操作都通过命令型 WebSocket 完成。核心设计是 **子进程 stdout 实时流式转发到浏览器**：

```
浏览器                         Dashboard                     CLI 子进程
─────────────────────────────────────────────────────────────────────
1. 连接 WebSocket
   ws://host/compile
                                      ↓
2. 发送 JSON: {"type":"spawn",       ↓
   "configuration":"my.yaml"}        ↓
                                      ↓  build_command() 构造命令
                                      ↓  ["python", "-m", "esphome",
                                      ↓   "--dashboard", "compile",
                                      ↓   "my.yaml"]
                                      ↓
                                      ↓  subprocess.Popen() 或
                                      ↓  tornado.process.Subprocess()
                                      ↓
3. 接收 stdout 行:                    ↓  子进程输出 stdout ──────→ _redirect_stdout()
   {"event":"line",                   ↓                          ↓ 解码
    "data":"Compiling...\n"}          ↓                          ↓ write_message()
                                      ↓                          ↓
4. 接收更多行...                      ↓                          ↓ (持续流式转发)
                                      ↓                          ↓
5. 可选: 发送 stdin                   ↓                          ↓
   {"type":"stdin",                   ↓                          ↓ → proc.stdin.write()
    "data":"yes\n"}                   ↓                          ↓   (交互式命令如 OTA 确认)
                                      ↓                          ↓
6. 接收退出:                          ↓                          ↓
   {"event":"exit",                   ↓  _proc_on_exit(returncode)
    "code":0}                         ↓  → write_message() → close()
                                      ↓
7. WebSocket 关闭                     ↓  terminate() 子进程
```

**关键设计细节**：

1. **双模式子进程**：Windows 使用 `subprocess.Popen` + 读取线程（因 Windows 不支持非阻塞管道），Linux/macOS 使用 `tornado.process.Subprocess`（异步 I/O）

```python
# Windows: Popen + 独立线程逐字节读取 stdout
if self._use_popen:  # os.name == "nt"
    self._proc = subprocess.Popen(command, stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
    stdout_thread = threading.Thread(target=self._stdout_thread)
    stdout_thread.daemon = True
    stdout_thread.start()

# Linux/macOS: Tornado 异步 Subprocess
else:
    self._proc = tornado.process.Subprocess(command, stdout=Subprocess.STREAM, stderr=subprocess.STDOUT)
    self._proc.set_exit_callback(self._proc_on_exit)
```

2. **stdout 线性转发**：使用 `read_until_regex(b"[\n\r]")` 逐行读取，每读到一行就立即 `write_message({"event":"line","data":text})` 发送给浏览器

3. **set_nodelay(True)**：WebSocket 连接建立时立即启用 TCP_NODELAY，避免 200-500ms 的发送延迟

4. **stdin 交互**：浏览器可通过 `{"type":"stdin","data":"text"}` 发送数据到子进程 stdin，用于交互式确认（如 OTA 时的 Yes/No 提示）

5. **`--dashboard` 标志**：所有通过 Dashboard 调用的 CLI 命令都附加 `--dashboard` 参数。这影响 `__main__.py` 中的行为：设置 `CORE.dashboard = True`，在设备地址无法解析时提供不同提示信息

#### 13.4.2 设备端口命令 — EsphomePortCommandWebSocket

`logs`、`upload`、`run` 三个端点继承 `EsphomePortCommandWebSocket`，需要额外的 `--device <port>` 参数。这类命令还实现了**地址缓存传递**机制：

```python
class EsphomePortCommandWebSocket(EsphomeCommandWebSocket):
    async def build_device_command(self, args, json_message):
        configuration = json_message["configuration"]
        port = json_message["port"]

        # 仅 OTA 模式 + 设备有 api 集成时，传递地址缓存
        cache_args = []
        if port == "OTA" and entry and "api" in entry.loaded_integrations:
            cache_args = build_cache_arguments(entry, dashboard, time.monotonic())

        # 缓存参数必须放在子命令之前
        cmd = [*DASHBOARD_COMMAND, *cache_args, *args, config_file, "--device", port]
        return cmd
```

`build_cache_arguments()` 将 Dashboard 已知的 mDNS/DNS 解析结果作为 CLI 参数传递：

```bash
# 示例：传递 mDNS 缓存地址
python -m esphome --dashboard --mdns-address-cache mydevice.local=192.168.1.100 upload my.yaml --device OTA
```

这解决了 Dashboard 环境中 mDNS 解析可能延迟的问题——Dashboard 已经通过 mDNS/DNS 发现了设备 IP，直接传递给 CLI 子进程，避免 CLI 重新做 mDNS 发现。

#### 13.4.3 事件型 WebSocket — DashboardEventsWebSocket

`/events` WebSocket 提供**实时设备状态推送**，采用订阅-发布模式：

```
浏览器                         Dashboard                     后台检测
─────────────────────────────────────────────────────────────────────
1. 连接 ws://host/events
                                      ↓
                                      ↓  await entries.async_request_update_entries()
                                      ↓  _send_initial_state()
                                      ↓
2. 接收初始状态:                      ↓
   {"event":"initial_state",          ↓
    "data":{"devices":[...],          ↓  ← build_device_list_response()
           "ping":{"my.yaml":true}}}  ↓  ← entry_state_to_bool()
                                      ↓
                                      ↓  _subscribe_to_events()
                                      ↓  注册6个事件监听器
                                      ↓  DASHBOARD_SUBSCRIBER.subscribe(self)
                                      ↓
                                      ↓  DashboardSubscriber._event_loop()
                                      ↓  每2秒:
                                      ↓    dashboard.ping_request.set() → PingStatus
                                      ↓    每10秒: entries.async_request_update_entries()
                                      ↓
3. 接收状态变更:                      ↓  PingStatus → entry.state = ONLINE
   {"event":"entry_state_changed",    ↓  → bus.async_fire(ENTRY_STATE_CHANGED)
    "data":{"filename":"my.yaml",     ↓  → WebSocket write_message()
    "name":"mydevice",                ↓
    "state":true}}                    ↓
                                      ↓
4. 接收设备添加:                      ↓  文件系统出现新 YAML →
   {"event":"entry_added",            ↓  entries.async_update_entries()
    "data":{"device":{...}}}          ↓  → bus.async_fire(ENTRY_ADDED)
                                      ↓
5. 客户端发送心跳:                    ↓
   {"event":"ping"}                   ↓  → 响应 {"event":"pong"}
                                      ↓
6. 客户端请求刷新:                    ↓
   {"event":"refresh"}                ↓  → DASHBOARD_SUBSCRIBER.request_refresh()
                                      ↓  → 立即触发下一次轮询
```

**DashboardSubscriber 生命周期管理**：

```python
class DashboardSubscriber:
    _subscribers: set[DashboardEventsWebSocket] = set()
    _event_loop_task: asyncio.Task | None = None

    def subscribe(self, subscriber):
        self._subscribers.add(subscriber)
        if not self._event_loop_task or self._event_loop_task.done():
            # 第一个订阅者加入时启动轮询循环
            self._event_loop_task = asyncio.create_task(self._event_loop())
        return partial(self._unsubscribe, subscriber)

    def _unsubscribe(self, subscriber):
        self._subscribers.discard(subscriber)
        if not self._subscribers and self._event_loop_task:
            # 所有订阅者离开时停止轮询循环
            self._event_loop_task.cancel()
```

这种设计实现了**按需轮询**——只有浏览器打开时才执行 ping/mDNS 检查，浏览器关闭后自动停止，节省系统资源。

**事件类型**（定义在 `const.py:DashboardEvent`）：

| 事件方向 | 事件名 | 说明 |
|----------|--------|------|
| Server → Client | `initial_state` | 连接时发送完整设备列表和在线状态 |
| Server → Client | `entry_state_changed` | 设备在线状态变更 |
| Server → Client | `entry_added` | 新设备配置文件被发现 |
| Server → Client | `entry_removed` | 设备配置文件被删除 |
| Server → Client | `entry_updated` | 设备配置文件被修改 |
| Server → Client | `importable_device_added` | 发现可导入的新设备（mDNS 发现） |
| Server → Client | `importable_device_removed` | 可导入设备消失 |
| Server → Client | `pong` | 心跳响应 |
| Client → Server | `ping` | 心跳请求 |
| Client → Server | `refresh` | 请求立即刷新设备状态 |

### 13.5 设备发现与在线检测

Dashboard 通过**三级优先级**检测设备是否在线：

```
优先级（从高到低）:
  ┌─────────────────────────────────────────────────────────┐
  │  1. mDNS (MDNSStatus)                                   │
  │     最快且最准确 — 监听 _esphomelib._tcp.local 服务     │
  │     如果 mDNS 发现设备在线 → 直接设为 ONLINE            │
  │     如果 mDNS 发现设备离线 → 仅在来源为 mDNS 时更新     │
  ├─────────────────────────────────────────────────────────┤
  │  2. MQTT (MqttStatusThread, 可选)                        │
  │     通过 esphome/discover/# topic 发现设备               │
  │     优先级低于 mDNS — 不覆盖 mDNS 的在线判定            │
  │     可覆盖 mDNS 的离线判定（如果 MQTT 发现设备在线）    │
  ├─────────────────────────────────────────────────────────┤
  │  3. ICMP Ping (PingStatus)                              │
  │     最底层 — 用 icmplib 发送 ICMP ping                   │
  │     如果 ping 成功 → 设为 ONLINE（覆盖任何来源）        │
  │     如果 ping 失败 → 仅在来源为 ping/unknown 时设离线   │
  │     DNS 解析失败 → 标记为 DNS_FAILURE                    │
  └─────────────────────────────────────────────────────────┘

状态来源标记 (EntryStateSource):
  MDNS  → mDNS 发现
  PING  → ICMP ping
  MQTT  → MQTT 发现
  UNKNOWN → 初始状态

可达状态 (ReachableState):
  ONLINE        → 设备在线
  OFFLINE       → 设备离线
  DNS_FAILURE   → DNS 解析失败（地址无法解析为 IP）
  UNKNOWN       → 初始/未知状态
```

**mDNS 发现服务**（`MDNSStatus`）：
- 使用 `zeroconf` 库监听 `_esphomelib._tcp.local` 服务
- `DashboardBrowser` 同时注册两个回调：`DashboardStatus`（设备在线/离线）和 `DashboardImportDiscovery`（发现可导入设备）
- 可导入设备信息包含 `project_name`、`package_import_url` 等，前端据此生成"Adopt"按钮

**ICMP Ping 检测**（`PingStatus`）：
- 使用 `icmplib.async_ping()` 异步 ping
- 优先尝试 `privileged=True`（原始 socket），失败后回退 `privileged=False`
- 按批次 ping（`GROUP_SIZE = MAX_EXECUTOR_WORKERS / 2 = 24`），避免并发过大
- 最低 ping 间隔 5 秒（`MIN_PING_INTERVAL`）
- ping 失败 + DNS 解析失败 → `DNS_FAILURE` 状态（区分"真的离线"和"域名解析不了"）

**DNS 缓存**（`DNSCache`）：
- TTL 120 秒
- 对 `.local` 地址解析失败时，尝试去掉 `.local` 后缀作为回退（某些系统 mDNS 不工作但单播 DNS 可工作）

**设备导入发现**：

mDNS 服务发现的可导入设备包含项目信息（如 ESPHome 项目模板链接），前端可以一键"Adopt"导入：

```python
# DiscoveredImport 数据结构
@dataclass
class DiscoveredImport:
    device_name: str
    friendly_name: str
    package_import_url: str    # 项目模板 URL
    project_name: str          # 如 "esphome.bluetooth-proxy"
    project_version: str
    network: str               # wifi / ethernet
```

### 13.6 认证机制

Dashboard 支持三种认证模式：

```
┌─────────────────────────────────────────────────────────────────┐
│  1. 无认证 (默认)                                               │
│     --username 和 --password 均空                              │
│     is_authenticated() 直接返回 True                           │
│     清除 _xsrf cookie（不需要 CSRF 保护）                      │
├─────────────────────────────────────────────────────────────────┤
│  2. 本地认证 (--username/--password)                           │
│     密码使用 PBKDF2-HMAC-SHA512 哈希存储                       │
│     cookie_secret 从 EsphomeStorageJSON 自动加载              │
│     支持 Basic Auth (Authorization header) 和 Cookie 认证     │
│     hmac.compare_digest() 防止时序攻击                        │
│     XSRF cookie 保护 POST 请求                                │
├─────────────────────────────────────────────────────────────────┤
│  3. Home Assistant Addon 认证 (--ha-addon)                     │
│     X-HA-Ingress: YES 头 → 自动认证（HA ingress 端口）       │
│     非 ingress 端口 → 调用 Supervisor /auth API               │
│     可通过 DISABLE_HA_AUTHENTICATION 环境变量禁用 HA 认证     │
└─────────────────────────────────────────────────────────────────┘
```

`authenticated` 装饰器和 `is_authenticated()` 函数的工作流程：

```python
def is_authenticated(handler):
    # HA Addon ingress: X-HA-Ingress == "YES" → 直接通过
    if settings.on_ha_addon:
        header = handler.request.headers.get("X-HA-Ingress", "NO")
        if str(header) == "YES":
            return True

    # 认证模式: 检查 Basic Auth 或 Cookie
    if settings.using_auth:
        if auth_header := handler.request.headers.get("Authorization"):
            # Basic Auth: base64(username:password)
            return settings.check_password(username, password)
        # Cookie: AUTH_COOKIE_NAME == "yes"
        return handler.get_secure_cookie(AUTH_COOKIE_NAME) == COOKIE_AUTHENTICATED_YES

    # 无认证模式: 直接通过
    return True
```

### 13.7 设备列表管理 — DashboardEntries

`DashboardEntries` 是设备列表的核心管理类，负责从配置目录扫描 YAML 文件并维护设备状态：

```python
class DashboardEntries:
    _entries: dict[Path, DashboardEntry]  # 文件路径 → 设备条目
    _name_to_entry: dict[str, set[DashboardEntry]]  # 设备名 → 条目（支持重名）
    _update_lock: asyncio.Lock  # 防止并发更新

    async def async_update_entries(self):
        async with self._update_lock:
            # 1. 扫描配置目录中所有 .yaml/.yml 文件
            path_to_cache_key = self._get_path_to_cache_key()

            # 2. 计算 cache_key = (inode, device, mtime, size)
            #    仅 stat() 调用，不读取文件内容 → 大多数情况下缓存命中

            # 3. 比较新旧 cache_key，判断增/删/改
            #    cache_key 变化 → 调用 entry.load_from_disk() 从 StorageJSON 加载

            # 4. 触发事件: ENTRY_ADDED / ENTRY_REMOVED / ENTRY_UPDATED
            bus.async_fire(DashboardEvent.ENTRY_ADDED, {"entry": entry})
```

**缓存键设计**：

```python
def _get_path_to_cache_key(self):
    for file in util.list_yaml_files([self._config_dir]):
        # 优先从 StorageJSON 文件获取 stat（比 YAML 更准确）
        stat = ext_storage_path(file.name).stat()
        # cache_key = (inode, device, mtime, size)
        # inode+device 保证文件身份，mtime+size 保证内容变化
        path_to_cache_key[file] = (stat.st_ino, stat.st_dev, stat.st_mtime, stat.st_size)
```

这种设计避免每次轮询都读取 YAML 文件内容——仅通过 `stat()` 检查文件是否变化，在绝大多数情况下（文件未修改）是缓存命中，零 I/O 开销。

**DashboardEntry** 数据结构：

```python
@dataclass
class DashboardEntry:
    path: Path                    # YAML 文件完整路径
    filename: str                 # 如 "mydevice.yaml"
    storage: StorageJSON | None   # 从 StorageJSON 加载的元数据
    state: EntryState             # 设备在线状态 (ONLINE/OFFLINE/DNS_FAILURE/UNKNOWN)

    # 从 storage 派生的属性:
    name: str                     # 设备名 (如 "mydevice")
    friendly_name: str            # 用户友好名
    address: str | None           # 设备地址 (IP/mDNS)
    target_platform: str | None   # 目标平台 (ESP32/ESP8266/RP2040 等)
    loaded_integrations: set[str] # 已加载的集成列表
    update_available: bool        # 是否有可用更新 (部署版本 ≠ 当前版本)
```

### 13.8 YAML 编辑机制

Dashboard 支持两种 YAML 编辑模式：

#### Ace Editor（内嵌编辑器）

```
浏览器 → /ace WebSocket
         → EsphomeAceEditorHandler
         → esphome -q vscode --ace <config_dir>
         → CLI 生成验证/补全数据 (stdin/stdout JSON 协议)
         → 实时流式返回验证结果和自动补全建议
```

Ace Editor 的验证和补全数据通过 `esphome vscode --ace` 命令生成。该命令在 `__main__.py` 中以 stdin/stdout JSON 协议运行，为前端提供：
- YAML schema 验证（标记错误位置）
- 自动补全建议（可用组件、选项名等）

#### VSCode 编辑器（外部）

```
浏览器 → /vscode WebSocket
         → EsphomeVscodeHandler
         → esphome -q vscode dummy
         → CLI 生成 IDE 数据
```

此端点用于生成 VSCode 集成所需的配置数据（`idedata`），并非真正启动 VSCode。

#### 文件读写 API

```
读取: GET /edit?configuration=my.yaml → 返回 YAML 文件原始文本
写入: POST /edit?configuration=my.yaml → 写入浏览器提交的 YAML 内容
      → write_file(filename, request.body)
      → DASHBOARD.entries.async_schedule_storage_json_update(filename)
        → 后台任务: esphome compile --only-generate <filename>
```

写入 YAML 后，Dashboard 自动调度一个后台任务执行 `esphome compile --only-generate`，更新 StorageJSON（设备元数据缓存），确保设备列表显示的信息（平台、集成列表等）是最新的。

### 13.9 设备创建 — Wizard 与 Import

#### Wizard（配置向导）

`/wizard` POST 端点支持三种创建模式：

| 模式 | 说明 | 生成的配置 |
|------|------|------------|
| `basic` | 传统 4 步向导 | 基础 WiFi + OTA + API 配置，自动生成 OTA 密码和 Noise PSK |
| `upload` | 上传已有 YAML | 用户上传的 YAML 文件内容（Base64 编码） |
| `empty` | 空配置 | 仅包含 `esphome:` 和 `name:` 的空壳 |

```python
# basic 模式自动生成的安全凭据
kwargs["ota_password"] = secrets.token_hex(16)           # 32字符随机 OTA 密码
noise_psk = secrets.token_bytes(32)                       # 32字节 Noise PSK
kwargs["api_encryption_key"] = base64.b64encode(noise_psk).decode()  # Base64 编码
```

#### Import（设备导入）

`/import` POST 端点用于导入 mDNS 发现的可导入设备（Adopt 功能）：

```python
# 导入流程
1. 从 dashboard.import_result 查找发现的设备信息
2. 获取 project_name 和 package_import_url（如 GitHub 上的模板 YAML）
3. import_config() 生成配置文件，包含:
   - 基础设备配置
   - external_components 引用（指向项目模板 URL）
   - WiFi 配置（从 mDNS 发现的网络类型推导）
4. dashboard.ping_request.set() → 立即检查设备在线状态
```

### 13.10 固件下载机制

#### 下载类型列表 — `/downloads`

返回设备可下载的文件类型（不同平台提供不同类型）：

```python
# ESP32 平台的下载类型 (示例)
[
  {"type": "firmware", "name": "firmware.bin"},
  {"type": "factory", "name": "firmware.factory.bin"},
  {"type": "ota", "name": "firmware.ota.bin"},
  {"type": "partition_table", "name": "partition-table.bin"},
  {"type": "bootloader", "name": "bootloader.bin"},
]
```

每个平台通过 `get_download_types(storage_json)` 函数提供自己的下载类型列表。

#### 二进制文件下载 — `/download.bin`

```python
# 请求参数
?type=firmware       # 下载类型（旧参数名，兼容）
&file=firmware.bin   # 文件名（新参数名，优先）
&download=mydevice-firmware  # 下载时文件名
&compressed=1        # gzip 压缩（减少传输大小）

# 工作流程
1. 加载 StorageJSON → 获取 firmware_bin_path（构建目录）
2. 在构建目录中查找请求的文件
3. 如果文件不在构建目录 → 调用 esphome idedata 获取分区表路径
4. 路径安全检查: path.relative_to(base_dir) → 防止目录穿越攻击
5. 读取文件内容，可选 gzip 压缩
6. 设置 Content-Disposition: attachment; filename="..."
```

### 13.11 Prometheus 服务发现

`/prometheus-sd` 端点提供 Prometheus HTTP Service Discovery 格式的设备列表：

```json
[
  {
    "targets": ["192.168.1.100:80"],
    "labels": {
      "__meta_name": "mydevice",
      "__meta_esp_platform": "ESP32",
      "__meta_esphome_version": "2026.6.0",
      "__meta_integration_api": "true",
      "__meta_integration_wifi": "true"
    }
  }
]
```

仅包含配置了 Web Server（`entry.web_port` 不为 None）的设备。

### 13.12 前端架构

前端由独立 npm 包 `esphome-dashboard` 提供（版本 `20260425.0`），打包为静态资源：

```
esphome_dashboard/
├── static/
│   ├── js/esphome/
│   │   ├── index.js          # 入口文件（生产环境使用带 hash 的文件名）
│   │   └── ...               # 其他 JS 模块
│   ├── css/                  # 样式文件
│   └── fonts/                # 字体/图标
├── index.template.html       # 主页面模板
└── login.template.html       # 登录页面模板
```

**静态文件 URL 生成**：

```python
# 生产环境: 文件名 + MD5 hash → 缓存控制
def get_static_file_url(name):
    base = f"./static/{name}"
    path = get_static_path(name)
    hash_ = hashlib.md5(path.read_bytes()).hexdigest()[:8]
    return f"{base}?hash={hash_}"    # 如 ./static/js/esphome/index-abc12345.js?hash=deadbeef

# 开发环境: 直接引用原始文件名，无缓存
if ENV_DEV in os.environ:
    return base                      # 如 ./static/js/esphome/index.js
```

Tornado 的 `StaticFileHandler` 自定义了缓存策略：
- 开发模式：缓存时间 0（每次重新加载）
- 生产模式：含 `hash` 参数或 JavaScript 文件 → `CACHE_MAX_AGE`（长期缓存）；其他 → 默认缓存时间

### 13.13 设计亮点总结

1. **Web 层零核心逻辑**：Dashboard 不包含任何编译/代码生成逻辑，全部委托给 CLI 子进程。Web 层仅处理 HTTP 交互和状态展示，核心功能保持 CLI 可用

2. **子进程 stdout 流式转发**：编译/上传/日志等操作的输出通过 WebSocket 实时逐行转发到浏览器，用户体验接近终端

3. **三级设备发现优先级**：mDNS（最快最准） > MQTT（可选） > Ping（兜底），各级之间不互相覆盖在线判定

4. **按需轮询**：DashboardSubscriber 仅在有浏览器连接时才执行 ping/mDNS 检查，无连接时自动停止

5. **缓存键 stat() 而非 read()**：设备列表更新仅检查文件 stat 信息（inode/mtime/size），避免每次轮询读取 YAML 内容

6. **地址缓存传递**：Dashboard 已知的 mDNS/DNS 地址直接作为 CLI 参数传递，避免 CLI 子进程重复做 mDNS 发现

7. **stdin 交互**：浏览器可通过 WebSocket stdin 消息与 CLI 子进程交互（如 OTA 确认），实现终端级交互体验

8. **Windows 兼容**：Windows 使用 `subprocess.Popen` + 独立读取线程代替 Tornado 异步 Subprocess（因 Windows 不支持非阻塞管道）

9. **路径安全**：`settings.rel_path()` 严格检查路径必须在配置目录内，`DownloadBinaryRequestHandler` 检查下载路径必须在构建目录内

10. **两种 YAML 编辑模式**：内嵌 Ace Editor（通过 `vscode --ace` 获取验证/补全数据）和文件读写 API（直接 GET/POST YAML 内容）

### 13.14 关键文件索引

| 文件 | 作用 |
|------|------|
| `esphome/dashboard/dashboard.py` | Dashboard 启动入口、事件循环策略 |
| `esphome/dashboard/web_server.py` | Tornado 路由定义、所有 Handler 实现（1646行，核心文件） |
| `esphome/dashboard/core.py` | ESPHomeDashboard 全局单例、EventBus |
| `esphome/dashboard/entries.py` | DashboardEntries 设备列表管理、DashboardEntry 数据结构 |
| `esphome/dashboard/const.py` | DashboardEvent 枚举、DASHBOARD_COMMAND 常量 |
| `esphome/dashboard/settings.py` | DashboardSettings（认证、配置目录、模式） |
| `esphome/dashboard/models.py` | 数据模型（DeviceListResponse、ImportableDeviceDict） |
| `esphome/dashboard/dns.py` | DNSCache（TTL 120秒、.local 回退解析） |
| `esphome/dashboard/status/mdns.py` | MDNSStatus（zeroconf mDNS 发现） |
| `esphome/dashboard/status/ping.py` | PingStatus（icmplib ICMP ping 检测） |
| `esphome/dashboard/status/mqtt.py` | MqttStatusThread（MQTT 设备发现） |
| `esphome/dashboard/util/subprocess.py` | async_run_system_command（异步 CLI 调用） |
| `esphome/dashboard/util/password.py` | password_hash（PBKDF2-HMAC-SHA512） |
| `esphome/__main__.py` | CLI dashboard 命令定义、--dashboard 参数处理 |
| `esphome/storage_json.py` | StorageJSON 设备元数据存储格式 |

---

## 十四、External Components 外部组件开发

### 14.1 概述

ESPHome 的核心组件库覆盖了数百种硬件，但仍有许多专用硬件未被收录。**External Components** 机制允许用户编写自己的组件，存放于独立的 Git 仓库或本地目录，通过 YAML 配置中的 `external_components` 声明引用，无需修改 ESPHome 主仓库源码即可使用。

外部组件与内置组件共享完全相同的开发接口——`CONFIG_SCHEMA`、`to_code()`、`DEPENDENCIES` 等所有模块级属性和函数都可以正常使用。ESPHome 通过 `ComponentMetaFinder`（自定义的 Python `sys.meta_path` finder）将外部组件的路径动态注入到模块搜索路径中，使 `importlib.import_module("esphome.components.my_component")` 能找到外部仓库中的组件代码。

### 14.2 引用外部组件

`external_components` 的配置 schema（`esphome/components/external_components/__init__.py`）：

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/user/my-esphome-components.git
      ref: main              # 可选：分支/标签/commit
      username: user         # 可选：私有仓库认证
      password: pass         # 可选：私有仓库认证
      path: components       # 可选：组件目录子路径
    components: [my_sensor, my_switch]  # 指定要加载的组件，或 "all"
    refresh: 1d              # Git 仓库刷新间隔（默认1天）

  - source:
      type: local
      path: /home/user/my-components    # 本地目录
    components: [my_sensor]

  # 简写形式（最常用）
  - source: github://user/my-esphome-components@main
    components: [my_sensor]
```

#### Source 类型

| 类型 | Schema | 说明 |
|------|--------|------|
| `git` | `{type: git, url: ..., ref?: ..., username?: ..., password?: ..., path?: ...}` | 从 Git 仓库克隆 |
| `local` | `{type: local, path: ...}` | 从本地目录加载 |

#### 简写格式

ESPHome 支持 GitHub/Codeberg/GitLab 的简写 URL，在 `config_validation.py:validate_source_shorthand()` 中解析：

```
github://owner/repo[@ref]         → https://github.com/owner/repo.git
github://pr#1234                  → https://github.com/esphome/esphome.git, ref=pull/1234/head
codeberg://owner/repo[@ref]       → https://codeberg.org/owner/repo.git
gitlab://owner/repo[@ref]         → https://gitlab.org/owner/repo.git
```

#### 组件目录查找顺序

当未指定 `path` 时，ESPHome 按以下顺序查找组件目录：

```
1. <repo_dir>/esphome/components/   ← 推荐：与内置组件同级路径
2. <repo_dir>/components/           ← 兼容旧格式
3. 报错：Could not find components folder
```

指定 `path` 时直接在 `<repo_dir>/<path>/` 下查找，必须是一个目录。

#### `components` 过滤

```yaml
components: all          # 加载仓库中所有组件（默认）
components: [my_sensor]  # 仅加载指定组件
```

安全机制：当选择 `all` 时，如果仓库中的组件数量超过 100 个，ESPHome 会拒绝加载——这防止了用户不小心引用了整个 ESPHome fork 仓库（包含所有内置组件）的情况：

```python
if config[CONF_COMPONENTS] == "all":
    num_components = len(list(components_dir.glob("*/__init__.py")))
    if num_components > 100:
        raise cv.Invalid(
            "This source is an ESPHome fork or branch. "
            "Please manually specify the components you want to import using the 'components' key"
        )
```

指定 `components` 列表时，ESPHome 会验证每个组件是否存在 `__init__.py` 文件：

```python
for name in config[CONF_COMPONENTS]:
    expected = components_dir / name / "__init__.py"
    if not expected.is_file():
        raise cv.Invalid(f"Could not find __init__.py file for component {name}")
```

#### Git 仓库缓存与刷新

ESPHome 使用 SHA256 哈希的前 8 位作为缓存目录名（`git.py:_compute_destination_path()`）：

```python
def _compute_destination_path(key, domain):
    base_dir = Path(CORE.data_dir) / domain    # 如 .esphome/external_components/
    h = hashlib.new("sha256")
    h.update(key.encode())                      # key = "url@ref"
    return base_dir / h.hexdigest()[:8]          # 如 .esphome/external_components/a1b2c3d4
```

克隆和更新策略：

```
首次克隆:
  git clone --depth=1 <url> <dest>
  如果指定了 ref:
    git fetch --depth=1 origin <ref>
    git reset --hard FETCH_HEAD

后续更新（超过 refresh 间隔时）:
  git stash push --include-untracked   # 保存可能的本地修改
  git fetch --depth=1 origin [ref]     # 浅克隆更新
  git reset --hard FETCH_HEAD          # 重置到最新
  返回 revert 回调（允许回退到旧版本）

损坏恢复:
  如果仓库状态损坏 → 删除整个目录 → 重新克隆（仅尝试一次，防止无限递归）
```

### 14.3 加载机制 — ComponentMetaFinder

外部组件的加载核心是 `loader.py:ComponentMetaFinder`，一个自定义的 Python `sys.meta_path` finder：

```python
class ComponentMetaFinder(importlib.abc.MetaPathFinder):
    def __init__(self, components_path, allowed_components=None):
        self._allowed_components = allowed_components
        self._finders = []
        for hook in sys.path_hooks:
            try:
                finder = hook(str(components_path))  # 将外部组件目录注册为 Python 路径
            except ImportError:
                continue
            self._finders.append(finder)

    def find_spec(self, fullname, path=None, target=None):
        if not fullname.startswith("esphome.components."):
            return None
        parts = fullname.split(".")
        if len(parts) != 3:  # 仅处理直接组件（esphome.components.xxx），不处理平台
            return None
        component = parts[2]
        if self._allowed_components is not None and component not in self._allowed_components:
            return None  # 白名单过滤
        for finder in self._finders:
            spec = finder.find_spec(fullname, target=target)
            if spec is not None:
                return spec
        return None
```

**工作原理**：

```
1. _process_single_config() 解析 source 和 components 过滤
2. 确定组件目录路径（Git 克隆目录或本地路径）
3. loader.install_meta_finder(components_dir, allowed_components)
   → sys.meta_path.insert(0, ComponentMetaFinder(components_dir, allowed_components))

4. 当 ESPHome 配置验证需要加载组件时：
   importlib.import_module("esphome.components.my_sensor")
   → Python 遍历 sys.meta_path
   → ComponentMetaFinder.find_spec("esphome.components.my_sensor")
   → 在外部组件目录中查找 my_sensor/__init__.py
   → 返回 ModuleSpec → 正常导入
```

**关键设计细节**：

- `ComponentMetaFinder` 插入到 `sys.meta_path[0]`（最前面），优先于内置组件查找。这意味着**同名外部组件会覆盖内置组件**
- 仅处理 `esphome.components.xxx`（3段路径），平台（如 `esphome.components.dht.sensor`）通过父组件自动加载
- `allowed_components` 白名单确保仅加载用户指定的组件，避免意外引入无关组件

### 14.4 配置处理流程

`external_components` 在配置验证流程中的位置非常关键——它是**第一步**（step 1.3），在所有其他组件验证之前执行：

```
config.py:full_config()
  Step 1.1: 加载 substitutions
  Step 1.2: 解析 !extend / !remove
  Step 1.3: 加载 external_components ← 此时注册 ComponentMetaFinder
  Step 1.4+: 组件验证（此时外部组件已经可被 importlib 发现）
```

`do_external_components_pass()` 的完整流程：

```python
def do_external_components_pass(config):
    conf = config.get(CONF_EXTERNAL_COMPONENTS)
    if conf is None:
        return
    conf = CONFIG_SCHEMA(conf)            # 验证 external_components schema
    for c in conf:
        _process_single_config(c)         # 处理每个 source 条目
        # → Git: clone_or_update() → install_meta_finder()
        # → Local: install_meta_finder(local_path)
```

### 14.5 编写外部组件

#### 14.5.1 组件仓库结构

一个外部组件仓库的推荐结构：

```
my-esphome-components/
├── esphome/
│   └── components/
│       ├── my_sensor/
│       │   ├── __init__.py        # Python: CONFIG_SCHEMA + to_code()
│       │   ├── my_sensor.h        # C++: 头文件
│       │   └── my_sensor.cpp      # C++: 实现文件
│       ├── my_switch/
│       │   ├── __init__.py
│       │   ├── my_switch.h
│       │   └── my_switch.cpp
│       └── my_sensor/             # 平台子组件
│           └── some_platform/
│               ├── __init__.py     # 平台的 CONFIG_SCHEMA + to_code()
│               ├── some_platform.h
│               └── some_platform.cpp
└── README.md
```

> `esphome/components/` 是推荐路径——ESPHome 的 `_process_git_config()` 首先查找此路径。如果使用 `components/` 路径，需要在 YAML 中显式指定 `path: components`。

#### 14.5.2 组件模块级属性

每个组件的 `__init__.py` 可以声明以下模块级属性，ESPHome 通过 `ComponentManifest`（`loader.py`）读取：

| 属性 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `CONFIG_SCHEMA` | `cv.Schema` | YAML 配置验证 schema（必须） | 见下文 |
| `to_code` | `async def` | 代码生成函数（必须） | 见下文 |
| `DEPENDENCIES` | `list[str]` | 依赖的其他组件 | `["uart", "spi"]` |
| `AUTO_LOAD` | `list[str]` 或 `Callable` | 自动加载的组件 | `["sensor"]` |
| `CONFLICTS_WITH` | `list[str]` | 冲突的组件 | `["wifi"]` |
| `MULTI_CONF` | `bool` | 允许同一组件多次配置 | `True` |
| `MULTI_CONF_NO_DEFAULT` | `bool` | 多次配置但不自动生成 ID | `True` |
| `IS_PLATFORM_COMPONENT` | `bool` | 标记为平台组件（如 sensor/output） | `True` |
| `IS_TARGET_PLATFORM` | `bool` | 标记为目标平台（如 esp32） | `True` |
| `CODEOWNERS` | `list[str]` | 代码维护者 GitHub 用户名 | `["@myuser"]` |
| `FINAL_VALIDATE_SCHEMA` | `Callable` | 跨组件最终验证函数 | 见下文 |

#### 14.5.3 最简传感器组件示例

以下是一个完整的最简外部传感器组件，读取一个自定义 UART 传感器：

**`my_sensor/__init__.py`**（Python 侧）：

```python
import esphome.codegen as cg
from esphome.components import sensor, uart
import esphome.config_validation as cv
from esphome.const import (
    CONF_ID,
    CONF_UART_ID,
    DEVICE_CLASS_TEMPERATURE,
    STATE_CLASS_MEASUREMENT,
    UNIT_CELSIUS,
)

DEPENDENCIES = ["uart"]                    # 依赖 UART 组件
AUTO_LOAD = ["sensor"]                    # 自动加载 sensor 基类
CODEOWNERS = ["@myuser"]

my_sensor_ns = cg.esphome_ns.namespace("my_sensor")
MySensorComponent = my_sensor_ns.class_(
    "MySensorComponent", cg.PollingComponent, uart.UARTDevice, sensor.Sensor
)

CONFIG_SCHEMA = (
    sensor.sensor_schema(
        MySensorComponent,
        unit_of_measurement=UNIT_CELSIUS,
        accuracy_decimals=1,
        device_class=DEVICE_CLASS_TEMPERATURE,
        state_class=STATE_CLASS_MEASUREMENT,
    )
    .extend(uart.UART_DEVICE_SCHEMA)       # 继承 UART 设备 schema
    .extend(cv.polling_component_schema("60s"))  # 继承轮询组件 schema
)

FINAL_VALIDATE_SCHEMA = uart.final_validate_device_schema(
    "my_sensor", baud_rate=9600, require_rx=True, require_tx=False
)


async def to_code(config):
    var = await sensor.new_sensor(config)    # 创建 sensor Pvariable
    await cg.register_component(var, config) # 注册为 Component
    await uart.register_uart_device(var, config)  # 注册为 UARTDevice
```

**`my_sensor/my_sensor.h`**（C++ 侧）：

```cpp
#pragma once

#include "esphome/core/component.h"
#include "esphome/core/defines.h"
#ifdef USE_SENSOR
#include "esphome/components/sensor/sensor.h"
#endif
#include "esphome/components/uart/uart.h"

namespace esphome::my_sensor {

class MySensorComponent : public PollingComponent, public uart::UARTDevice, public sensor::Sensor {
 public:
  void update() override;    // PollingComponent: 定期调用
  void dump_config() override;
};

}  // namespace esphome::my_sensor
```

**`my_sensor/my_sensor.cpp`**（C++ 侧）：

```cpp
#include "my_sensor.h"
#include "esphome/core/log.h"

namespace esphome::my_sensor {

static const char *const TAG = "my_sensor";

void MySensorComponent::update() {
  uint8_t data[4];
  if (!this->read_array(data, 4)) {
    ESP_LOGW(TAG, "Read failed!");
    this->status_set_warning();
    return;
  }
  this->status_clear_warning();

  float temperature = (data[0] + data[1] * 256.0f) / 10.0f;
  this->publish_state(temperature);
}

void MySensorComponent::dump_config() {
  LOG_SENSOR("", "MySensor", this);
  ESP_LOGCONFIG(TAG, "  UART baud_rate: %d", this->get_baud_rate());
}

}  // namespace esphome::my_sensor
```

#### 14.5.4 三种组件架构模式

ESPHome 的组件有三种架构模式，从简单到复杂递进。选择哪种取决于组件的功能需求——单个实体还是多个实体、单一类型还是跨多种实体类型。

```
┌──────────────────────────────────────────────────────────────────────────┐
│  模式 A: 单实体组件                                                     │
│  组件类 = 实体类 (如 Sensor)                                             │
│  CONFIG_SCHEMA 使用 sensor.sensor_schema(MyClass)                       │
│  一个 YAML 块 → 一个实体                                                 │
│  示例: CSE7766, BME680, DHT                                             │
├──────────────────────────────────────────────────────────────────────────┤
│  模式 B: 单平台多子实体组件                                              │
│  __init__.py 定义主组件类 (cv.declare_id)                                │
│  sensor.py 中 cv.GenerateID(): cv.declare_id + cv.Optional 子传感器     │
│  一个 YAML 块 → 一个主组件 + 多个子传感器                                 │
│  示例: DHT, BME680 (在 sensor.py 中定义 schema)                         │
├──────────────────────────────────────────────────────────────────────────┤
│  模式 C: 多平台组件                                                      │
│  __init__.py 定义主组件类 (cv.declare_id + CONF_XXX_ID)                  │
│  sensor.py/binary_sensor.py 等使用 cv.use_id(XXXComponent)              │
│  to_code() 以 cg.get_variable(config[CONF_XXX_ID]) 开头                │
│  多个 YAML 块 → 各自独立，通过 use_id 关联到同一主组件                    │
│  示例: LD2450, ADS1115, ADE7953                                         │
└──────────────────────────────────────────────────────────────────────────┘
```

##### 模式 A：单实体组件

组件类本身就是一个实体（如 `Sensor`），一个 YAML 配置块对应一个实体。这是最简单的写法，也是上面 14.5.3 中展示的模式。

**典型特征**：
- `CONFIG_SCHEMA` 使用 `sensor.sensor_schema(MyClass)` —— 组件类同时是实体类
- `to_code()` 中使用 `sensor.new_sensor(config)` 创建变量
- C++ 类同时继承 `PollingComponent` 和 `sensor::Sensor`
- YAML 中直接定义在 `sensor:` 下

**YAML 使用方式**：

```yaml
sensor:
  - platform: my_sensor           # ← 直接在 sensor: 下，platform 指定组件
    name: "My Temperature"
```

**适用场景**：组件只提供单一传感器读数（如一个温度传感器、一个电流传感器）。

> 上面的 14.5.3 示例（MySensorComponent 继承 PollingComponent + UARTDevice + Sensor）就是典型的模式 A，此处不再重复代码。

##### 模式 B：单平台多子实体组件

一个硬件芯片通常提供多个测量值（如 DHT 同时输出温度和湿度，BME680 同时输出温度、湿度、气压、气体电阻）。此时需要一个主组件管理硬件通信，多个子传感器分别发布各自的数据。

**典型特征**：
- `__init__.py` 通常留空（或仅声明 `CODEOWNERS`/`DEPENDENCIES`）
- `sensor.py` 中定义主 `CONFIG_SCHEMA`，使用 `cv.GenerateID(): cv.declare_id(MainComponent)` 声明主组件
- `sensor.py` 中用 `cv.Optional(CONF_TEMPERATURE): sensor.sensor_schema(...)` 定义子传感器
- `to_code()` 先创建主组件 `cg.new_Pvariable(config[CONF_ID])`，再创建各子传感器 `sensor.new_sensor(config[CONF_TEMPERATURE])`
- C++ 主组件持有子传感器指针 (`sensor::Sensor *temperature_sensor_`)
- 子传感器**不继承 `Component`**，只是纯粹的 `Sensor` 实体

**完整示例**：自定义环境传感器（温度+湿度）

**`my_env_sensor/__init__.py`**（几乎为空）：

```python
CODEOWNERS = ["@myuser"]
```

**`my_env_sensor/sensor.py`**（所有逻辑在此）：

```python
from esphome import pins
import esphome.codegen as cg
from esphome.components import sensor
import esphome.config_validation as cv
from esphome.const import (
    CONF_HUMIDITY,
    CONF_ID,
    CONF_PIN,
    CONF_TEMPERATURE,
    DEVICE_CLASS_HUMIDITY,
    DEVICE_CLASS_TEMPERATURE,
    STATE_CLASS_MEASUREMENT,
    UNIT_CELSIUS,
    UNIT_PERCENT,
)
from esphome.cpp_helpers import gpio_pin_expression

DEPENDENCIES = ["uart"]                     # 或其他依赖
AUTO_LOAD = ["sensor"]

my_env_sensor_ns = cg.esphome_ns.namespace("my_env_sensor")
MyEnvSensorComponent = my_env_sensor_ns.class_(
    "MyEnvSensorComponent", cg.PollingComponent
)

CONFIG_SCHEMA = cv.Schema(
    {
        cv.GenerateID(): cv.declare_id(MyEnvSensorComponent),  # ← 主组件 ID
        cv.Required(CONF_PIN): pins.internal_gpio_input_pullup_pin_schema,
        cv.Optional(CONF_TEMPERATURE): sensor.sensor_schema(   # ← 子传感器（可选）
            unit_of_measurement=UNIT_CELSIUS,
            accuracy_decimals=1,
            device_class=DEVICE_CLASS_TEMPERATURE,
            state_class=STATE_CLASS_MEASUREMENT,
        ),
        cv.Optional(CONF_HUMIDITY): sensor.sensor_schema(     # ← 子传感器（可选）
            unit_of_measurement=UNIT_PERCENT,
            accuracy_decimals=0,
            device_class=DEVICE_CLASS_HUMIDITY,
            state_class=STATE_CLASS_MEASUREMENT,
        ),
    }
).extend(cv.polling_component_schema("60s"))


async def to_code(config):
    # 1. 创建主组件
    var = cg.new_Pvariable(config[CONF_ID])
    await cg.register_component(var, config)

    pin = await gpio_pin_expression(config[CONF_PIN])
    cg.add(var.set_pin(pin))

    # 2. 创建子传感器并注入到主组件
    if CONF_TEMPERATURE in config:
        sens = await sensor.new_sensor(config[CONF_TEMPERATURE])
        cg.add(var.set_temperature_sensor(sens))
    if CONF_HUMIDITY in config:
        sens = await sensor.new_sensor(config[CONF_HUMIDITY])
        cg.add(var.set_humidity_sensor(sens))
```

**`my_env_sensor/my_env_sensor.h`**：

```cpp
#pragma once
#include "esphome/core/component.h"
#include "esphome/core/defines.h"
#include "esphome/core/hal.h"
#ifdef USE_SENSOR
#include "esphome/components/sensor/sensor.h"
#endif

namespace esphome::my_env_sensor {

class MyEnvSensorComponent : public PollingComponent {
#ifdef USE_SENSOR
  SUB_SENSOR(temperature)    // ← 替代手动编写的成员指针 + setter
  SUB_SENSOR(humidity)       // ← 替代手动编写的成员指针 + setter
#endif

 public:
  void set_pin(InternalGPIOPin *pin) { pin_ = pin; }

  void setup() override;
  void update() override;
  void dump_config() override;

 protected:
  InternalGPIOPin *pin_;
  bool read_sensor_(float *temperature, float *humidity);
};

}  // namespace esphome::my_env_sensor
```

**`my_env_sensor/my_env_sensor.cpp`**：

```cpp
#include "my_env_sensor.h"
#include "esphome/core/log.h"

namespace esphome::my_env_sensor {

static const char *const TAG = "my_env_sensor";

void MyEnvSensorComponent::update() {
  float temperature, humidity;
  if (!read_sensor_(&temperature, &humidity)) {
    ESP_LOGW(TAG, "Read failed!");
    this->status_set_warning();
    return;
  }
  this->status_clear_warning();

  // ← 通过 SUB_SENSOR 生成的成员名访问子实体
  if (temperature_sensor_ != nullptr)
    temperature_sensor_->publish_state(temperature);
  if (humidity_sensor_ != nullptr)
    humidity_sensor_->publish_state(humidity);
}

void MyEnvSensorComponent::dump_config() {
  ESP_LOGCONFIG(TAG, "MyEnvSensor:");
  LOG_SENSOR("  ", "Temperature", this->temperature_sensor_);
  LOG_SENSOR("  ", "Humidity", this->humidity_sensor_);
}

}  // namespace esphome::my_env_sensor
```

**YAML 使用方式**：

```yaml
sensor:
  - platform: my_env_sensor
    pin: GPIO4
    temperature:
      name: "Room Temperature"
    humidity:
      name: "Room Humidity"
```

> **模式 B 的关键区别**：主组件类**不继承** `sensor::Sensor`。它是一个纯粹的 `Component`（或 `PollingComponent`），通过 `sensor::Sensor *` 指针持有子传感器。子传感器由 `sensor.new_sensor()` 创建并注册，主组件只负责硬件通信和数据分发。

> **为什么 `__init__.py` 留空**：ESPHome 的 `loader.py` 加载组件时首先导入 `esphome.components.xxx`（即 `__init__.py`）。如果 `__init__.py` 不定义 `CONFIG_SCHEMA`，ESPHome 会查找子模块（如 `sensor.py`）。这种设计允许将不同实体类型的 schema 分散到各自的子模块中，保持文件职责清晰。

##### 模式 C：多平台组件

复杂组件提供多种实体类型（sensor + binary_sensor + switch + number 等），每种类型在独立的子模块中定义。各子模块通过 `cv.use_id()` 引用主组件，通过 `cg.get_variable()` 获取主组件变量。

**典型特征**：
- `__init__.py` 定义主组件：`cv.GenerateID(): cv.declare_id(MainComponent)` + `CONF_XXX_ID = "xxx_id"` + `XXXBaseSchema`
- 各平台子模块（`sensor.py`、`binary_sensor.py` 等）使用 `cv.GenerateID(CONF_XXX_ID): cv.use_id(MainComponent)` 引用主组件
- `to_code()` 以 `parent = await cg.get_variable(config[CONF_XXX_ID])` 开头获取主组件变量
- `DEPENDENCIES = ["xxx"]` — 各子模块声明依赖主组件模块
- 主组件 `MULTI_CONF = True` — 允许多实例（如多个 ADS1115 芯片）

**完整示例**：自定义毫米波雷达组件（主组件 + sensor + binary_sensor）

**`my_radar/__init__.py`**（定义主组件）：

```python
import esphome.codegen as cg
from esphome.components import uart
import esphome.config_validation as cv
from esphome.const import CONF_ID

AUTO_LOAD = ["sensor", "binary_sensor"]    # ← 自动加载所有子平台
DEPENDENCIES = ["uart"]
CODEOWNERS = ["@myuser"]
MULTI_CONF = True                          # ← 允许多个雷达实例

my_radar_ns = cg.esphome_ns.namespace("my_radar")
MyRadarComponent = my_radar_ns.class_(
    "MyRadarComponent", cg.Component, uart.UARTDevice
)

CONF_MY_RADAR_ID = "my_radar_id"           # ← 子模块引用主组件的 ID key

CONFIG_SCHEMA = cv.Schema(
    {
        cv.GenerateID(): cv.declare_id(MyRadarComponent),  # ← 主组件 ID
    }
).extend(uart.UART_DEVICE_SCHEMA).extend(cv.COMPONENT_SCHEMA)

# ← 子模块共享的基础 schema，包含 use_id 声明
MyRadarBaseSchema = cv.Schema(
    {
        cv.GenerateID(CONF_MY_RADAR_ID): cv.use_id(MyRadarComponent),
    },
)

FINAL_VALIDATE_SCHEMA = uart.final_validate_device_schema(
    "my_radar", baud_rate=256000, require_rx=True, require_tx=True
)


async def to_code(config):
    var = cg.new_Pvariable(config[CONF_ID])     # ← 创建主组件
    await cg.register_component(var, config)
    await uart.register_uart_device(var, config)
```

**`my_radar/sensor.py`**（子传感器模块）：

```python
import esphome.codegen as cg
from esphome.components import sensor
import esphome.config_validation as cv
from esphome.const import CONF_ID, DEVICE_CLASS_DISTANCE, UNIT_MILLIMETER

from . import CONF_MY_RADAR_ID, MyRadarComponent     # ← 从父模块导入

DEPENDENCIES = ["my_radar"]                           # ← 依赖主组件

CONF_TARGET_COUNT = "target_count"

CONFIG_SCHEMA = cv.Schema(
    {
        cv.GenerateID(CONF_MY_RADAR_ID): cv.use_id(MyRadarComponent),  # ← 引用主组件
        cv.Optional(CONF_TARGET_COUNT): sensor.sensor_schema(
            accuracy_decimals=0,
            device_class=DEVICE_CLASS_DISTANCE,
            unit_of_measurement=UNIT_MILLIMETER,
        ),
    }
)


async def to_code(config):
    # ← 关键: 先获取主组件变量
    radar = await cg.get_variable(config[CONF_MY_RADAR_ID])

    if target_count_config := config.get(CONF_TARGET_COUNT):
        sens = await sensor.new_sensor(target_count_config)
        cg.add(radar.set_target_count_sensor(sens))   # ← 注入到主组件
```

**`my_radar/binary_sensor.py`**（子二值传感器模块）：

```python
import esphome.codegen as cg
from esphome.components import binary_sensor
import esphome.config_validation as cv
from esphome.const import CONF_ID, DEVICE_CLASS_OCCUPANCY

from . import CONF_MY_RADAR_ID, MyRadarComponent

DEPENDENCIES = ["my_radar"]

CONF_HAS_TARGET = "has_target"

CONFIG_SCHEMA = cv.Schema(
    {
        cv.GenerateID(CONF_MY_RADAR_ID): cv.use_id(MyRadarComponent),  # ← 引用主组件
        cv.Optional(CONF_HAS_TARGET): binary_sensor.binary_sensor_schema(
            device_class=DEVICE_CLASS_OCCUPANCY,
        ),
    }
)


async def to_code(config):
    # ← 同样先获取主组件变量
    radar = await cg.get_variable(config[CONF_MY_RADAR_ID])

    if has_target_config := config.get(CONF_HAS_TARGET):
        sens = await binary_sensor.new_binary_sensor(has_target_config)
        cg.add(radar.set_target_binary_sensor(sens))  # ← 注入到主组件
```

**`my_radar/my_radar.h`**（C++ 主组件）：

```cpp
#pragma once
#include "esphome/core/component.h"
#include "esphome/core/defines.h"
#ifdef USE_SENSOR
#include "esphome/components/sensor/sensor.h"
#endif
#ifdef USE_BINARY_SENSOR
#include "esphome/components/binary_sensor/binary_sensor.h"
#endif
#include "esphome/components/uart/uart.h"

namespace esphome::my_radar {

class MyRadarComponent : public Component, public uart::UARTDevice {
#ifdef USE_SENSOR
  SUB_SENSOR(target_count)         // ← 替代手动编写的成员指针 + setter
#endif
#ifdef USE_BINARY_SENSOR
  SUB_BINARY_SENSOR(target)        // ← 替代手动编写的成员指针 + setter
#endif

 public:
  void setup() override;
  void loop() override;
  void dump_config() override;
};

}  // namespace esphome::my_radar
```

**YAML 使用方式**：

```yaml
# 主组件定义
my_radar:                                 # ← 独立的 YAML 顶级块
  id: my_radar_1                          # ← 显式 ID，子模块引用此 ID
  uart_id: uart_bus

# 各子实体定义（独立的 YAML 块，各自引用主组件 ID）
sensor:
  - platform: my_radar                    # ← sensor 的 platform 指向 my_radar 子模块
    my_radar_id: my_radar_1               # ← 引用主组件 ID
    target_count:
      name: "Target Count"

binary_sensor:
  - platform: my_radar                    # ← binary_sensor 的 platform
    my_radar_id: my_radar_1               # ← 同一引用
    has_target:
      name: "Has Target"
```

> **模式 C 的关键区别**：主组件和子实体各自是独立的 YAML 配置块。子实体通过 `my_radar_id` 显式引用主组件 ID。在 `to_code()` 中通过 `cg.get_variable(config[CONF_MY_RADAR_ID])` 获取主组件变量——这是**协程依赖解析**机制的核心应用：如果主组件尚未注册，`get_variable()` 会 yield 回调度器等待。

**`cv.declare_id` vs `cv.use_id` 的区别**：

| 函数 | 作用 | 创建变量？ | 使用时机 |
|------|------|-----------|---------|
| `cv.declare_id(XXXClass)` | 声明一个新的组件变量，将 ID 绑定到该类 | ✅ 是（`cg.new_Pvariable`） | 主组件定义处 |
| `cv.use_id(XXXClass)` | 引用一个已存在的组件变量，不创建新变量 | ❌ 否（`cg.get_variable`） | 子模块引用主组件处 |

对应的代码生成调用：

```
cv.declare_id → cg.new_Pvariable(config[CONF_ID])     # 创建新对象
cv.use_id    → cg.get_variable(config[CONF_XXX_ID])    # 获取已注册对象（等待依赖）
```

**`register_parented` 机制**：

某些模式 C 的子组件（如 ADS1115 的传感器）使用 `cg.register_parented()` 将子实体自动注册到主组件：

```python
# ADS1115 sensor.py 的 to_code()
var = cg.new_Pvariable(config[CONF_ID])
await sensor.register_sensor(var, config)
await cg.register_component(var, config)
await cg.register_parented(var, config[CONF_ADS1115_ID])   # ← 自动调用 parent->register_child(var)
```

`register_parented()` 生成的 C++ 代码：

```cpp
parent->register_child(var);   // 主组件持有子组件引用，便于生命周期管理
```

**源码中的真实示例**：

| 组件 | `__init__.py` | `sensor.py` | `binary_sensor.py` | 其他子模块 |
|------|--------------|-------------|--------------------|-----------|
| LD2450 | 主组件 + CONF_LD2450_ID + BaseSchema | use_id + 多子传感器 | use_id + 3 子传感器 | number/select/switch/button/text_sensor |
| ADS1115 | 主组件 + CONF_ADS1115_ID + MULTI_CONF | use_id + register_parented | — | — |
| ADE7953_base | 主组件类定义 | — | — | 由 ade7953_i2c/spi 子组件继承 |
| BL0942 | 仅 CODEOWNERS | sensor.py 中 cv.GenerateID + use_id | — | — |

##### 三种模式的对比总结

| 特征 | 模式 A | 模式 B | 模式 C |
|------|--------|--------|--------|
| 主组件类继承 | Component + Sensor | 仅 Component | 仅 Component |
| 实体类 | 主组件即是实体 | 子 Sensor 指针 | 子 Sensor/BinarySensor 指针 |
| CONFIG_SCHEMA 位置 | `__init__.py` | `sensor.py` | `__init__.py` + 各子模块 |
| cv.GenerateID | sensor_schema 内自动 | `cv.declare_id(MainComp)` | `__init__.py`: declare_id; 子模块: use_id |
| YAML 结构 | `sensor:` 下的单一块 | `sensor:` 下的单一块（含子键） | 多个顶级块，子块用 xxx_id 关联 |
| `to_code()` 开头 | `sensor.new_sensor()` | `cg.new_Pvariable(CONF_ID)` | `cg.get_variable(CONF_XXX_ID)` |
| 多实体类型 | ❌ 仅 1 种 | ❌ 仅 sensor | ✅ sensor + binary_sensor + ... |
| 文件结构 | `__init__.py` + `.h/.cpp` | `__init__.py`(空) + `sensor.py` + `.h/.cpp` | `__init__.py` + `sensor.py` + `binary_sensor.py` + `.h/.cpp` |
| 适用场景 | 单一读数的简单传感器 | 一芯片多测量值 | 一芯片多种实体类型 |

##### 模式选择决策树

```
你的组件需要提供什么？
│
├── 仅一个传感器读数？
│   → 模式 A（单实体组件）
│   例: 一个温度传感器、一个 ADC 电压读数
│
├── 一个芯片多个同类测量值？
│   → 模式 B（单平台多子实体）
│   例: DHT(温度+湿度)、BME680(温度+气压+湿度+气体)、电量计(电压+电流+功率+电量)
│
├── 一个芯片多种实体类型？
│   → 模式 C（多平台组件）
│   例: LD2450(sensor+binary_sensor+number+select+switch)、
│       ADS1115(主组件+sensor平台)
│
└── 不确定？
│   → 从模式 A 开始，需要更多子实体时升级到 B 或 C
```

#### 14.5.5 C++ 侧辅助宏与条件编译

上面三种模式的 C++ 示例中使用了 `SUB_*`、`LOG_*` 等宏以及 `#ifdef USE_*` 条件编译保护。这些是 ESPHome 源码中的标准实践，本节详细说明。

##### SUB_* 宏 — 子实体声明语法糖

模式 B 和 C 中，主组件需要持有子实体的指针成员和对应的 setter 方法。ESPHome 为每种实体类型提供了 `SUB_*` 宏，一行声明即可自动生成成员变量和 setter 方法：

```cpp
// 手动编写（模式 B / C 的原始写法）
 public:
  void set_temperature_sensor(sensor::Sensor *sens) { temperature_sensor_ = sens; }
  void set_humidity_sensor(sensor::Sensor *sens) { humidity_sensor_ = sens; }
 protected:
  sensor::Sensor *temperature_sensor_{nullptr};
  sensor::Sensor *humidity_sensor_{nullptr};

// 使用 SUB_* 宏（等价写法，更简洁）
#ifdef USE_SENSOR
  SUB_SENSOR(temperature)     // 生成: protected sensor::Sensor *temperature_sensor_{nullptr};
  SUB_SENSOR(humidity)        //      + public void set_temperature_sensor(sensor::Sensor *)
#endif
```

每种 `SUB_*` 宏的展开规则遵循固定模式：`SUB_ETYPE(name)` → `protected: etype::EType *name##_etype_{nullptr}` + `public: void set_##name##_etype(etype::EType *)`。

完整的 `SUB_*` 宏列表：

| 宏 | 生成的成员变量类型 | 生成的 setter 方法名 | 定义位置 |
|-----|-------------------|---------------------|---------|
| `SUB_SENSOR(name)` | `sensor::Sensor *name##_sensor_` | `set_##name##_sensor(sensor::Sensor *)` | `sensor/sensor.h` |
| `SUB_BINARY_SENSOR(name)` | `binary_sensor::BinarySensor *name##_binary_sensor_` | `set_##name##_binary_sensor(binary_sensor::BinarySensor *)` | `binary_sensor/binary_sensor.h` |
| `SUB_SWITCH(name)` | `switch_::Switch *name##_switch_` | `set_##name##_switch(switch_::Switch *)` | `switch/switch.h` |
| `SUB_TEXT_SENSOR(name)` | `text_sensor::TextSensor *name##_text_sensor_` | `set_##name##_text_sensor(text_sensor::TextSensor *)` | `text_sensor/text_sensor.h` |
| `SUB_NUMBER(name)` | `number::Number *name##_number_` | `set_##name##_number(number::Number *)` | `number/number.h` |
| `SUB_SELECT(name)` | `select::Select *name##_select_` | `set_##name##_select(select::Select *)` | `select/select.h` |
| `SUB_BUTTON(name)` | `button::Button *name##_button_` | `set_##name##_button(button::Button *)` | `button/button.h` |

> 注意 `SUB_SWITCH` 的 setter 参数类型是 `switch_::Switch *s`（简写 `s` 而非 `switch`），因为 `switch` 是 C++ 关键字。

**宏的完整展开**（以 `SUB_SENSOR` 为例）：

```cpp
#define SUB_SENSOR(name) \
 protected: \
  sensor::Sensor *name##_sensor_{nullptr}; \
\
 public: \
  void set_##name##_sensor(sensor::Sensor *sensor) { this->name##_sensor_ = sensor; }
```

其中 `##` 是 C 预处理器的 token 粘贴运算符，将 `name` 与固定后缀拼接。例如 `SUB_SENSOR(temperature)` 展开：

```cpp
protected:
  sensor::Sensor *temperature_sensor_{nullptr};    // name##_sensor_ → temperature_sensor_

public:
  void set_temperature_sensor(sensor::Sensor *sensor) { this->temperature_sensor_ = sensor; }
                                           // set_##name##_sensor → set_temperature_sensor
```

**Python 侧的对应调用**不变——`SUB_*` 宏生成的 setter 方法名与手动编写的一致，因此 Python 的 `cg.add(var.set_temperature_sensor(sens))` 调用方式完全相同：

```python
# Python 侧的 to_code() — SUB_* 宏不影响 Python 端代码
sens = await sensor.new_sensor(config[CONF_TEMPERATURE])
cg.add(var.set_temperature_sensor(sens))    # ← 无论 C++ 端使用 SUB_SENSOR 还是手动编写，调用方式相同
```

**源码中的真实使用示例**：

| 组件 | 实体类型 | `SUB_*` 宏使用 |
|------|---------|---------------|
| APDS9960 | sensor + binary_sensor | `SUB_SENSOR(red)`, `SUB_SENSOR(green)`, `SUB_SENSOR(blue)`, `SUB_SENSOR(clear)`, `SUB_SENSOR(proximity)` + `SUB_BINARY_SENSOR(up_direction)` 等 4 个方向 |
| BL0906 | sensor | `SUB_SENSOR(voltage)` + 6×`SUB_SENSOR(current)` + 6×`SUB_SENSOR(power)` + `SUB_SENSOR(energy)` |
| LD2450 | binary_sensor + sensor | `SUB_BINARY_SENSOR(has_target)` + `SUB_SENSOR(x)` / `SUB_SENSOR(y)` 等 |
| AS3935 | sensor + binary_sensor | `SUB_SENSOR(distance)` + `SUB_SENSOR(energy)` + `SUB_BINARY_SENSOR(thunder_alert)` |
| Daly BMS | binary_sensor + sensor | `SUB_BINARY_SENSOR(charging_mos_enabled)` 等 + `SUB_SENSOR(voltage)` 等 |
| GPIO Switch | switch | `SUB_SWITCH(output)` |

**`SUB_*` 宏的使用注意事项**：

1. **宏本身包含 `protected:` 和 `public:` 标签**——每个 `SUB_*` 宏展开后自带 `protected:` 成员段和 `public:` setter段。因此宏应放在类体的**最前面**（在第一个显式访问修饰符之前），而非在已有的 `protected:` 区域内。在已有的 `protected:` 内调用 `SUB_*` 会产生 `protected:` → `protected:` → `public:` 的交替切换，虽能编译但不够清晰

2. **宏应受 `#ifdef USE_*` 保护**——与 `#include` 一样，`SUB_*` 宏应放在对应的 `#ifdef USE_SENSOR` / `#ifdef USE_BINARY_SENSOR` 等条件编译块内。这样当 YAML 中没有该实体类型时，宏完全不展开，类中不存在对应的成员和方法

3. **成员变量名遵循固定模式 `name##_sensor_`**——访问子实体时使用 `this->temperature_sensor_`，而非 `this->temperature`

4. **setter 方法名遵循 `set_##name##_sensor`**——Python 侧调用 `cg.add(var.set_temperature_sensor(sens))`

5. **不能用于自身即是实体的模式 A**——模式 A 中组件类继承 `Sensor`，自身就是实体，不需要子实体指针。`SUB_*` 仅用于模式 B/C 中主组件持有子实体指针的场景

6. **`SUB_SENSOR_WITH_DEDUP` 是 LD24XX 系列的专用变体**——它不生成普通 `Sensor *` 指针，而是生成 `ld24xx::SensorWithDedup<T>` 对象，提供去重发布功能。这是组件特定的扩展宏，不属于通用 `SUB_*` 系列

##### LOG_* 宏 — dump_config 中的实体日志

与 `SUB_*` 宏配套使用的还有 `LOG_*` 实体日志宏，用于在 `dump_config()` 中统一输出实体信息：

```cpp
void MyEnvSensorComponent::dump_config() {
  ESP_LOGCONFIG(TAG, "MyEnvSensor:");
  LOG_SENSOR("  ", "Temperature", this->temperature_sensor_);
  LOG_SENSOR("  ", "Humidity", this->humidity_sensor_);
}
```

`LOG_SENSOR(prefix, type, obj)` 的展开：

```cpp
// 展开为:
log_sensor(TAG, "  ", LOG_STR_LITERAL("Temperature"), this->temperature_sensor_);

// log_sensor() 实际输出（当 obj != nullptr 时）:
//   [I][my_env_sensor:042]:   Temperature: 'Room Temperature'
//     Unit of measurement: '°C'
//     Accuracy decimals: 1
//     State class: 'measurement'
//     Device class: 'temperature'
```

当 `obj == nullptr` 时，`log_sensor()` 输出 `Not connected!` 而非崩溃。

完整的 `LOG_*` 实体日志宏列表：

| 宏 | 输出内容 | 定义位置 |
|-----|---------|---------|
| `LOG_SENSOR(prefix, type, obj)` | 名称、单位、精度、状态类、设备类 | `sensor/sensor.h` |
| `LOG_BINARY_SENSOR(prefix, type, obj)` | 名称、设备类 | `binary_sensor/binary_sensor.h` |
| `LOG_SWITCH(prefix, type, obj)` | 名称、设备类 | `switch/switch.h` |
| `LOG_NUMBER(prefix, type, obj)` | 名称、步长、范围、单位 | `number/number.h` |
| `LOG_SELECT(prefix, type, obj)` | 名称、选项列表 | `select/select.h` |
| `LOG_BUTTON(prefix, type, obj)` | 名称 | `button/button.h` |
| `LOG_TEXT_SENSOR(prefix, type, obj)` | 名称、设备类 | `text_sensor/text_sensor.h` |
| `LOG_COVER(prefix, type, obj)` | 名称、设备类 | `cover/cover.h` |
| `LOG_CLIMATE(prefix, type, obj)` | 名称、设备类、视觉配置 | `climate/climate.h` |
| `LOG_FAN(prefix, type, obj)` | 名称、速度、方向 | `fan/fan.h` |
| `LOG_LIGHT(prefix, type, obj)` | 名称、效果、亮度 | `light/light.h` |
| `LOG_LOCK(prefix, type, obj)` | 名称 | `lock/lock.h` |

此外还有硬件相关的日志宏：

| 宏 | 输出内容 | 定义位置 |
|-----|---------|---------|
| `LOG_PIN(prefix, pin)` | 引脚编号和模式 | `core/gpio.h` |
| `LOG_I2C_DEVICE(this)` | I2C 地址 | `i2c/i2c.h` |
| `LOG_SPI_DEVICE(this)` | SPI CS 引脚 | `spi/spi.h` |
| `LOG_ONE_WIRE_DEVICE(this)` | 1-Wire 地址 | `one_wire/one_wire.h` |
| `LOG_UPDATE_INTERVAL(this)` | 更新间隔 | `core/component.h` |

`LOG_ENTITY_ICON`、`LOG_ENTITY_DEVICE_CLASS`、`LOG_ENTITY_UNIT_OF_MEASUREMENT` 这三个宏受条件编译控制——仅在设置了对应属性时才输出：

```cpp
// entity_base.h 中条件编译控制的 LOG 宏
#ifdef USE_ENTITY_ICON
#define LOG_ENTITY_ICON(tag, prefix, obj) log_entity_icon(tag, prefix, obj)
#else
#define LOG_ENTITY_ICON(tag, prefix, obj) ((void) 0)       // ← 编译为空操作
#endif

#ifdef USE_ENTITY_DEVICE_CLASS
#define LOG_ENTITY_DEVICE_CLASS(tag, prefix, obj) log_entity_device_class(tag, prefix, obj)
#endif

#ifdef USE_ENTITY_UNIT_OF_MEASUREMENT
#define LOG_ENTITY_UNIT_OF_MEASUREMENT(tag, prefix, obj) log_entity_unit_of_measurement(tag, prefix, obj)
#endif
```

这些 `USE_ENTITY_*` 宏不是实体类型宏，而是由 `entity_helpers.py` 在 `setup_device_class()` / `setup_unit_of_measurement()` / `setup_entity()` 中动态添加——只有 YAML 中实际设置了 `device_class`、`unit_of_measurement` 或 `icon` 的实体才会触发对应的 `USE_ENTITY_*` define。这使得未设置这些属性的实体不会输出冗余日志行，同时节省 Flash（`log_entity_icon()` 等函数不被编译）。

##### USE_* 条件编译宏体系

ESPHome 的条件编译宏体系分为三个层次：**平台实体类型宏**、**功能子特性宏**、**实体属性宏**。

**层次 1：平台实体类型宏 `USE_<PLATFORM>`**

这是最基础的条件编译层——当 YAML 配置中存在某实体类型的实体时，Python 代码生成器自动添加对应的 `USE_*` define：

```
YAML 配置中有 sensor → register_sensor() → CORE.platform_counts["sensor"] += 1
                                                    ↓
                    _add_platform_defines() (FINAL 优先级协程)
                    ↓
                    cg.add_define("USE_SENSOR")
                    cg.add_define("ESPHOME_ENTITY_SENSOR_COUNT", count)
```

`_add_platform_defines()` 在 `core/config.py` 中以 `CoroPriority.FINAL` 优先级运行，确保在所有组件 `to_code()` 完成后才统计实体数量。完整的实体类型宏映射：

| YAML 实体平台 | 自动添加的 define | 实体数量 define |
|---------------|------------------|----------------|
| sensor | `USE_SENSOR` | `ESPHOME_ENTITY_SENSOR_COUNT` |
| binary_sensor | `USE_BINARY_SENSOR` | `ESPHOME_ENTITY_BINARY_SENSOR_COUNT` |
| switch | `USE_SWITCH` | `ESPHOME_ENTITY_SWITCH_COUNT` |
| button | `USE_BUTTON` | `ESPHOME_ENTITY_BUTTON_COUNT` |
| number | `USE_NUMBER` | `ESPHOME_ENTITY_NUMBER_COUNT` |
| select | `USE_SELECT` | `ESPHOME_ENTITY_SELECT_COUNT` |
| text_sensor | `USE_TEXT_SENSOR` | `ESPHOME_ENTITY_TEXT_SENSOR_COUNT` |
| cover | `USE_COVER` | `ESPHOME_ENTITY_COVER_COUNT` |
| fan | `USE_FAN` | `ESPHOME_ENTITY_FAN_COUNT` |
| light | `USE_LIGHT` | `ESPHOME_ENTITY_LIGHT_COUNT` |
| climate | `USE_CLIMATE` | `ESPHOME_ENTITY_CLIMATE_COUNT` |
| lock | `USE_LOCK` | `ESPHOME_ENTITY_LOCK_COUNT` |
| valve | `USE_VALVE` | `ESPHOME_ENTITY_VALVE_COUNT` |
| alarm_control_panel | `USE_ALARM_CONTROL_PANEL` | `ESPHOME_ENTITY_ALARM_CONTROL_PANEL_COUNT` |
| water_heater | `USE_WATER_HEATER` | `ESPHOME_ENTITY_WATER_HEATER_COUNT` |
| media_player | `USE_MEDIA_PLAYER` | `ESPHOME_ENTITY_MEDIA_PLAYER_COUNT` |
| event | `USE_EVENT` | `ESPHOME_ENTITY_EVENT_COUNT` |
| update | `USE_UPDATE` | `ESPHOME_ENTITY_UPDATE_COUNT` |
| infrared | `USE_INFRARED` | `ESPHOME_ENTITY_INFRARED_COUNT` |
| radio_frequency | `USE_RADIO_FREQUENCY` | `ESPHOME_ENTITY_RADIO_FREQUENCY_COUNT` |
| datetime(date) | `USE_DATETIME_DATE` | `ESPHOME_ENTITY_DATE_COUNT` |
| datetime(time) | `USE_DATETIME_TIME` | `ESPHOME_ENTITY_TIME_COUNT` |
| datetime(datetime) | `USE_DATETIME_DATETIME` | `ESPHOME_ENTITY_DATETIME_COUNT` |
| text | `USE_TEXT` | `ESPHOME_ENTITY_TEXT_COUNT` |

这些 define 控制了 `entity_types.h` 中 X-macro 的条件展开——未使用的实体类型完全不编译，对应的类、方法、注册函数、Controller 回调等零 Flash/RAM 开销：

```cpp
// entity_types.h — #ifdef USE_SENSOR 决定了 sensor 相关的所有代码是否编译
#ifdef USE_SENSOR
ENTITY_CONTROLLER_TYPE_(sensor::Sensor, sensor, sensors, ESPHOME_ENTITY_SENSOR_COUNT, SENSOR, sensor_update)
#endif
// ↑ 如果 YAML 中没有 sensor 实体，USE_SENSOR 不会被定义
//   → sensor::Sensor 类不编译 → App.register_sensor() 不生成 → Controller::on_sensor_update() 不生成
```

**层次 2：功能子特性宏**

某些实体类型有可选的子功能，仅在用户配置了特定选项时才添加对应的 define：

| 宏 | 触发条件 | 定义位置（Python） | C++ 作用 |
|-----|---------|-------------------|---------|
| `USE_SENSOR_FILTER` | YAML 配置了 `filters` | `sensor/__init__.py:962` | 编译 `Filter` 类链 + `Sensor::add_filter()` 等方法 |
| `USE_BINARY_SENSOR_FILTER` | YAML 配置了 `filters` | `binary_sensor/__init__.py:602` | 编译 `BinarySensorFilter` 类链 |
| `USE_TEXT_SENSOR_FILTER` | YAML 配置了 `filters` | `text_sensor/__init__.py:208` | 编译 `TextSensorFilter` 类链 |
| `USE_LIGHT_GAMMA_LUT` | YAML 配置了非默认 `gamma_correct` | `light/__init__.py:408` | 编译 gamma 校正查找表 |
| `USE_OUTPUT` | YAML 中有 `output` 组件（如 rtttl） | `output/__init__.py:159` / `rtttl/__init__.py:88` | 编译 `FloatOutput` / `BinaryOutput` 基类 |
| `USE_OUTPUT_FLOAT_POWER_SCALING` | YAML 配置了 `max_power`/`min_power`/`zero_means_zero` | `output/__init__.py:57,60,66,130,150` | 编译功率缩放逻辑 |
| `USE_CLIMATE_VISUAL_OVERRIDES` | YAML 配置了自定义视觉参数（min_temp 等） | `climate/__init__.py:279,282,285,293,296` | 编译视觉配置覆盖 |
| `USE_API_USER_DEFINED_ACTIONS` | YAML 配置了 `api.actions` | `api/__init__.py:362` | 编译用户自定义 API 服务 |
| `USE_API_CUSTOM_SERVICES` | YAML 有 `api.custom_services` | `api/__init__.py:366` | 编译外部组件动态服务注册 |
| `USE_API_HOMEASSISTANT_SERVICES` | YAML 有 `api.homeassistant_services` | `api/__init__.py:369` | 编译 HA 服务调用 |
| `USE_API_HOMEASSISTANT_STATES` | YAML 有 `api.homeassistant_states` | `api/__init__.py:372` | 编译 HA 状态同步 |
| `USE_API_NOISE_PSK_FROM_YAML` | YAML 配置了 `api.encryption.key` | `api/__init__.py:468` | 编译 Noise PSK 预加载 |
| `USE_API_CLIENT_CONNECTED_TRIGGER` | YAML 配置了 `api.on_client_connected` | `api/__init__.py:449` | 编译客户端连接触发器 |
| `USE_API_CLIENT_DISCONNECTED_TRIGGER` | YAML 配置了 `api.on_client_disconnected` | `api/__init__.py:457` | 编译客户端断开触发器 |

**层次 3：实体属性宏**

这些宏由 `entity_helpers.py` 在设置实体属性时动态添加：

| 宏 | 触发条件 | C++ 作用 |
|-----|---------|---------|
| `USE_ENTITY_ICON` | YAML 配置了 `icon` | 编译 `log_entity_icon()`、icon 查找表 |
| `USE_ENTITY_DEVICE_CLASS` | YAML 配置了 `device_class` | 编译 `log_entity_device_class()`、device_class 查找表 |
| `USE_ENTITY_UNIT_OF_MEASUREMENT` | YAML 配置了 `unit_of_measurement` | 编译 `log_entity_unit_of_measurement()`、uom 查找表 |

**条件编译宏对外部组件开发的影响**：

1. **头文件 `#include` 受 `USE_*` 保护**——实体类型的 `#include` 和 `SUB_*` 宏都应放在对应的 `#ifdef USE_SENSOR` 等条件编译块内，参考 ESPHome 源码中的标准实践（如 `apds9960.h`、`as3935.h`）

2. **外部组件不需要手动添加 `USE_*` define**——实体类型宏由 `CORE.platform_counts` → `_add_platform_defines()` 自动处理。当外部组件的 `to_code()` 调用 `sensor.register_sensor()` 时，`register_sensor()` 内部会调用 `CORE.register_platform_component("sensor")`，自动增加计数，最终触发 `USE_SENSOR` define

3. **外部组件可以添加自定义 define**——如果外部组件需要 C++ 端的条件编译控制，可以在 Python 端手动 `cg.add_define("USE_MY_FEATURE")`，然后在 C++ 端用 `#ifdef USE_MY_FEATURE` 裁剪

```python
# 外部组件 Python 侧
if config.get(CONF_ADVANCED_MODE):
    cg.add_define("USE_MY_SENSOR_ADVANCED")

# 外部组件 C++ 侧
#ifdef USE_MY_SENSOR_ADVANCED
  void advanced_feature();
#endif
```

**条件编译宏的生成时序**：

```
1. YAML 配置验证阶段:
   各组件 __init__.py 的 CONFIG_SCHEMA 验证 → 加载组件模块

2. 代码生成阶段:
   各组件 to_code() → register_sensor() / register_binary_sensor() 等
                    → CORE.register_platform_component("sensor")
                    → CORE.platform_counts["sensor"] += 1
   特定功能 → cg.add_define("USE_SENSOR_FILTER") 等子特性宏
   实体属性 → setup_device_class() → cg.add_define("USE_ENTITY_DEVICE_CLASS") 等属性宏

3. FINAL 优先级阶段（所有 to_code() 完成后）:
   _add_platform_defines() → 遍历 CORE.platform_counts
                            → cg.add_define("USE_SENSOR", count)
                            → cg.add_define("ESPHOME_ENTITY_SENSOR_COUNT", 3)
   _add_controller_registry_define() → USE_CONTROLLER_REGISTRY
   _generate_tables_job() → entity string pool → PROGMEM 查找表

4. 写入阶段:
   writer.py → CORE.defines → 写入 defines.h
```

这个时序保证了一个关键特性：**同一 YAML 配置永远产生相同的 defines.h**——因为 `platform_counts` 和子特性宏的数量和值完全取决于配置内容，而 `_add_platform_defines()` 在所有代码生成完成后才执行统计，不存在时序依赖问题。

#### 14.5.6 代码生成核心 API

外部组件的 `to_code()` 函数使用以下核心 API：

| API | 说明 | 示例 |
|-----|------|------|
| `cg.new_Pvariable(id, args)` | 创建 `new Type(args)` 表达式 | `var = cg.new_Pvariable(config[CONF_ID])` |
| `cg.Pvariable(id, var)` | 创建 placement new 变量（避免堆碎片） | `var = cg.Pvariable(config[CONF_ID], var)` |
| `cg.add(expression)` | 生成一条 C++ 语句 | `cg.add(var.set_name("foo"))` |
| `cg.add_define("USE_xxx")` | 添加条件编译宏 | `cg.add_define("USE_MY_SENSOR")` |
| `cg.register_component(var, config)` | 注册为 Component（自动 setup/loop） | `await cg.register_component(var, config)` |
| `sensor.new_sensor(config)` | 创建并注册 sensor Pvariable | `var = await sensor.new_sensor(config)` |
| `sensor.register_sensor(var, config)` | 注册 sensor 实体 | `await register_sensor(var, config)` |
| `cg.get_variable(id)` | 获取已注册的变量（等待依赖） | `uart_var = await cg.get_variable(config[CONF_UART_ID])` |
| `cg.esphome_ns.namespace("xxx")` | 创建 C++ 命名空间 | `ns = cg.esphome_ns.namespace("my_sensor")` |
| `ns.class_("Xxx", bases)` | 声明 C++ 类（继承基类） | `MySensor = ns.class_("MySensor", cg.PollingComponent)` |

**实体注册函数**（各实体类型提供）：

| 实体类型 | 注册函数 | 说明 |
|----------|----------|------|
| sensor | `sensor.new_sensor(config)` / `sensor.register_sensor(var, config)` | 数值传感器 |
| binary_sensor | `binary_sensor.new_binary_sensor(config)` | 二值传感器 |
| switch | `switch.new_switch(config)` | 开关 |
| light | `light.new_light(config)` | 灯光 |
| output | `output.register_output(var, config)` | 输出平台 |
| fan | `fan.new_fan(config)` | 风扇 |
| cover | `cover.new_cover(config)` | 窗帘/门 |
| text_sensor | `text_sensor.new_text_sensor(config)` | 文本传感器 |
| number | `number.new_number(config)` | 数值输入 |
| select | `select.new_select(config)` | 选择器 |
| button | `button.new_button(config)` | 按钮 |

#### 14.5.7 FINAL_VALIDATE_SCHEMA — 跨组件验证

`FINAL_VALIDATE_SCHEMA` 在所有组件验证完成后执行，可以检查跨组件约束。典型用例是 UART 设备验证波特率：

```python
# 验证 UART 波特率是否符合设备要求
FINAL_VALIDATE_SCHEMA = uart.final_validate_device_schema(
    "my_sensor",
    baud_rate=9600,           # 要求的波特率
    require_rx=True,          # 需要 RX 引脚
    require_tx=False,         # 不需要 TX 引脚
    data_bits=8,              # 数据位
    parity="NONE",            # 校验位
)
```

自定义跨组件验证：

```python
def _final_validate(config):
    full_config = fv.full_config.get()        # 获取完整配置
    # 例如：检查该组件是否与 wifi 共存
    if "wifi" in full_config and "my_sensor" in full_config:
        raise cv.Invalid("my_sensor conflicts with wifi")

FINAL_VALIDATE_SCHEMA = _final_validate
```

### 14.6 实战分析：dashboard_import 组件

`dashboard_import` 是一个特殊的组件，展示了一些高级模式：

```python
# esphome/components/dashboard_import/__init__.py

DEPENDENCIES = ["api"]                # 依赖 API（因为 mDNS 发现只在 API 启用时工作）
CODEOWNERS = ["@esphome/core"]

CONFIG_SCHEMA = cv.All(
    cv.Schema({
        cv.Required("package_import_url"): validate_import_url,
        cv.Optional("import_full_config", default=False): cv.boolean,
    }),
    validate_full_url,               # 跨字段验证
)

FINAL_VALIDATE_SCHEMA = _final_validate  # 要求 esphome.project 信息

async def to_code(config):
    cg.add_define("USE_DASHBOARD_IMPORT")
    url = config["package_import_url"]
    cg.add(dashboard_import_ns.set_package_import_url(url))   # 将 URL 传递给 C++ 代码
```

此组件在设备被"Adopt"导入时使用，`import_config()` 函数生成设备的 YAML 配置文件：

```python
def import_config(path, name, friendly_name, project_name, import_url, network, encryption):
    # 生成配置文件
    config = {
        "substitutions": {"name": name},
        "packages": {project_name: import_url},    # 使用 packages 引用项目模板
        "esphome": {"name": "${name}", "name_add_mac_suffix": False},
    }
    if encryption:
        key = base64.b64encode(secrets.token_bytes(32)).decode()
        config["api"] = {"encryption": {"key": key}}   # 自动生成加密密钥
```

### 14.7 packages 配置引用

外部组件仓库中通常包含项目模板 YAML，用户通过 `packages` 配置引用：

```yaml
# 用户的设备配置
substitutions:
  name: my-bluetooth-proxy

packages:
  bluetooth-proxy: github://esphome/bluetooth-proxy@main   # 引用项目模板

esphome:
  name: ${name}
  name_add_mac_suffix: false
```

`packages` 组件（`esphome/components/packages/__init__.py`）负责：
1. 解析简写 URL（与 `external_components` 相同的 `validate_source_shorthand()`）
2. 从 Git 仓库下载 YAML 文件
3. 将下载的配置与用户本地配置合并（`merge_config()`）
4. 支持 substitutions 变量传递（`CONF_VARS` / `CONF_SUBSTITUTIONS`）
5. 递归 include 深度限制：`MAX_INCLUDE_DEPTH = 20`

### 14.8 C++ 头文件包含规则

外部组件的 C++ 文件必须正确 include ESPHome 的头文件。关键规则：

```cpp
// 1. 总是先 include 自己的头文件（防止隐式依赖）
#include "my_sensor.h"

// 2. include ESPHome 核心头文件
#include "esphome/core/component.h"
#include "esphome/core/log.h"
#include "esphome/core/hal.h"

// 3. include 依赖组件的头文件（与 DEPENDENCIES 对应）
#include "esphome/components/uart/uart.h"
#include "esphome/components/sensor/sensor.h"

// 4. 不要 include 不需要的头文件（条件编译会裁剪未使用的组件）
```

ESPHome 的 `writer.py` 会自动生成 `esphome.h` 超级头文件，include 所有用到的组件头文件。生成的 `main.cpp` 中 `#include "esphome.h"` 即可访问所有组件。

### 14.9 组件开发注意事项

#### 避免的常见错误

| 问题 | 说明 | 正确做法 |
|------|------|---------|
| 组件目录名与内置组件同名 | 外部组件优先于内置组件，同名会覆盖内置组件 | 使用独特的组件名（如 `my_` 前缀） |
| 忘记声明 `DEPENDENCIES` | 组件间依赖无法解析，`cg.get_variable()` 会失败 | 列出所有依赖组件 |
| 直接使用 `CONFIG_SCHEMA = cv.Schema({...})` | sensor 等实体需要使用 `sensor.sensor_schema()` | 使用对应实体类型的 schema 构造器 |
| 忘记 `await cg.register_component()` | 组件不会被注册到 `App`，`setup()`/`loop()` 不执行 | 所有 Component 子类必须注册 |
| `to_code()` 不是 async def | 代码生成器期望协程函数 | 使用 `async def to_code(config)` |
| 在 C++ 中使用 `std::string` 而非 `std::string` | ESPHome 使用 `StringRef` 和 `LogString` 节省 RAM | 使用 `const char *` 或 `LogString` |

#### 组件注册流程

```
to_code(config)
  │
  ├── 1. cg.new_Pvariable(config[CONF_ID], ...) → 创建 C++ 对象
  │     等价于: auto *var = new MySensorComponent(...)
  │
  ├── 2. await cg.register_component(var, config) → 注册到 App
  │     等价于: App.register_component(var)
  │     → 自动注册 setup/loop
  │
  ├── 3. await entity_type.new_entity(config) → 注册实体
  │     等价于: App.register_sensor(var)
  │     → 自动设置 name/hash/属性
  │
  ├── 4. cg.add(var.set_xxx(value)) → 设置属性
  │     等价于: var->set_xxx(value)
  │
  └── 5. await cg.get_variable(dependency_id) → 获取依赖变量
        如果未注册 → yield 回调度器等待
        已注册 → 立即返回变量引用
```

#### 组件命名规范

```
C++ 命名空间:  esphome::my_sensor         (与目录名一致)
C++ 类名:      MySensorComponent          (PascalCase + Component)
C++ TAG:       "my_sensor"                (与命名空间一致)
Python 命名空间: my_sensor_ns = cg.esphome_ns.namespace("my_sensor")
Python 类:     MySensorComponent = my_sensor_ns.class_("MySensorComponent", ...)
```

### 14.10 发布与分发

#### Git 仓库发布

1. 在 GitHub/Codeberg/GitLab 创建仓库
2. 按 `esphome/components/` 结构放置组件
3. 用户通过 `github://owner/repo@branch` 引用

#### 项目模板发布（Adopt 模式）

ESPHome 支持通过 mDNS 发现的"可导入设备"一键 Adopt：

1. 设备固件中声明 `dashboard_import` 和 `project` 信息：

```yaml
esphome:
  project:
    name: "myorg.my-device"
    version: "1.0.0"

dashboard_import:
  package_import_url: github://myorg/my-device-firmware@main
  import_full_config: false    # false: 使用 packages 引用; true: 下载完整 YAML
```

2. 设备在 mDNS 发现中广播项目信息
3. Dashboard 的 `DashboardImportDiscovery` 捕获广播
4. 用户在 Dashboard 中点击 "Adopt" → 自动生成配置文件

**两种导入模式**：

| 模式 | `import_full_config` | 生成的配置 |
|------|---------------------|------------|
| packages 引用（推荐） | `false` | 仅包含 `substitutions` + `packages` + `esphome` + `wifi`，引用远程项目模板 |
| 完整配置下载 | `true` | 下载整个 YAML 文件到本地，完全独立 |

#### 分发最佳实践

1. **使用 `packages` 引用而非完整 YAML**：让设备配置保持最小化，更新通过 Git 仓库推送
2. **在 `packages` 中使用 `ref` 指定稳定版本**：避免 `@main` 导致的不稳定更新
3. **声明 `DEPENDENCIES`**：让 ESPHome 自动加载依赖组件
4. **使用 `FINAL_VALIDATE_SCHEMA`**：验证 UART 波特率等跨组件约束
5. **提供 `CODEOWNERS`**：方便社区协作和维护

### 14.11 关键文件索引

| 文件 | 作用 |
|------|------|
| `esphome/components/external_components/__init__.py` | CONFIG_SCHEMA、Git/本地 source 处理、install_meta_finder |
| `esphome/loader.py` | ComponentManifest、ComponentMetaFinder、组件加载与缓存 |
| `esphome/git.py` | clone_or_update（Git 仓库克隆/刷新/损坏恢复）、GitFile 简写解析 |
| `esphome/config_validation.py` | SOURCE_SCHEMA（Git/Local schema）、validate_source_shorthand |
| `esphome/config.py` | full_config() 中的 external_components 加载步骤（step 1.3） |
| `esphome/core/entity_helpers.py` | setup_entity() 实体属性设置（device_class/unit/icon/name） |
| `esphome/components/dashboard_import/__init__.py` | dashboard_import 组件 + import_config() 生成配置 |
| `esphome/components/packages/__init__.py` | packages 配置引用与合并 |

---

## 十五、Sensor Filter 过滤器链架构

### 15.1 概述

ESPHome 的传感器过滤器（Filter）是一套**链式处理系统**，位于传感器原始读数与最终发布值之间。当传感器通过 `publish_state()` 发布新值时，该值不是直接传递给回调函数和 Controller，而是先经过一个**单向链表**形式的过滤器链——每个 Filter 可以修改值、丢弃值、延迟发送，或基于窗口统计计算新值。

过滤器的存在使得传感器数据在 MCU 端就能完成去噪、平滑、采样控制等预处理，减轻 Home Assistant 侧的处理负担。整个过滤器系统通过条件编译宏 `USE_SENSOR_FILTER` 裁剪——只有 YAML 中配置了 `filters:` 的传感器才编译过滤器相关代码。

```
传感器硬件读数
    ↓ publish_state(raw_value)
    ↓ raw_callback_.call(raw_value)  ← on_raw_value 触发器收到原始值
    ↓
┌───┴─────────────────────────────────────────────────────────┐
│  Filter 链 (单向链表)                                        │
│                                                              │
│  filter_list_ ──→ Filter1 ──next_──→ Filter2 ──next_──→ ... │
│                                                              │
│  每个 Filter:                                                │
│    input(value) → new_value(value) → optional<float>        │
│      • 有值 → output(value) → next_->input(value)           │
│      • 空值 → 链终止，值被丢弃                              │
│                                                              │
│  链尾 (next_ == nullptr):                                    │
│    output(value) → parent_->internal_send_state_to_frontend │
└──────────────────────────────────────────────────────────────┘
    ↓
callback_.call(filtered_value)  ← on_value 触发器收到过滤后值
    ↓
ControllerRegistry::notify_sensor_update(this)  ← APIServer/WebServer 收到
```

### 15.2 Filter 链的核心机制

#### 链的构建

Filter 链在代码生成阶段通过 `Sensor::add_filters()` 构建，形成单向链表：

```cpp
void Sensor::add_filter(Filter *filter) {
  if (this->filter_list_ == nullptr) {
    this->filter_list_ = filter;   // 第一个 filter 成为链头
  } else {
    Filter *last = this->filter_list_;
    while (last->next_ != nullptr)
      last = last->next_;
    last->initialize(this, filter); // 将新 filter 链到末尾
  }
  filter->initialize(this, nullptr); // 新 filter 的 next_ 暂为 nullptr
}
```

每个 Filter 持有两个指针：

```cpp
class Filter {
 protected:
  Filter *next_{nullptr};     // 链中下一个 Filter
  Sensor *parent_{nullptr};   // 所属 Sensor（链尾时用于回调）
};
```

#### 值的流转

核心流转通过两个方法实现：

```cpp
void Filter::input(float value) {
  optional<float> out = this->new_value(value);  // 子类实现过滤逻辑
  if (out.has_value())
    this->output(*out);    // 有值 → 继续传递
  // 空值 → 链终止，值被丢弃
}

void Filter::output(float value) {
  if (this->next_ == nullptr) {
    this->parent_->internal_send_state_to_frontend(value); // 链尾 → 发布
  } else {
    this->next_->input(value);  // 链中 → 传递给下一个
  }
}
```

**关键语义**：`new_value()` 返回 `optional<float>`：
- **有值**：继续在链中传递（值可能被修改）
- **空值**（`nullopt`）：值被丢弃，链中断，**后续 Filter 和回调都不会收到该值**

#### publish_state() 的完整路径

```cpp
void Sensor::publish_state(float state) {
  this->raw_state = state;                     // 保存原始值（deprecated 属性）
  this->raw_callback_.call(state);             // on_raw_value 触发器（始终收到原始值）

  if (this->filter_list_ == nullptr) {
    this->internal_send_state_to_frontend(state); // 无过滤器 → 直接发布
  } else {
    this->filter_list_->input(state);             // 有过滤器 → 进入链
  }
}

void Sensor::internal_send_state_to_frontend(float state) {
  this->set_has_state(true);
  this->state = state;                         // 更新最终状态
  this->callback_.call(state);                 // on_value 触发器
  ControllerRegistry::notify_sensor_update(this); // 推送到 APIServer 等
}
```

> **原始值 vs 过滤后值**：`on_raw_value` 回调始终收到未经任何过滤的原始值，`on_value` 回调仅收到过滤链最终输出的值。这使得自动化系统可以在两个层面分别响应。

### 15.3 过滤器分类体系

ESPHome 的过滤器按功能分为六大类：

```
Filter (基类)
│
├── 数值变换类 — 修改值的数值
│   ├── OffsetFilter              偏移: x → x + offset
│   ├── MultiplyFilter            缩放: x → x * multiplier
│   ├── CalibrateLinearFilter     线性校准: x → k*x + b (分段)
│   ├── CalibratePolynomialFilter 多项式校准: x → Σ a_i * x^i
│   ├── ToNTCResistanceFilter     NTC温度→电阻 (Steinhart-Hart逆)
│   ├── ToNTCTemperatureFilter    NTC电阻→温度 (Steinhart-Hart正)
│   ├── ClampFilter               限幅: x → clamp(x, min, max)
│   ├── RoundFilter               精度取整: x → round(x, N位小数)
│   ├── RoundMultipleFilter       倍数取整: x → round(x, multiple)
│   ├── RoundSignificantDigitsFilter 有效数字取整
│   └── LambdaFilter / StatelessLambdaFilter 自定义Lambda: x → f(x)
│
├── 窗口统计类 — 基于滑动窗口计算统计量
│   ├── SlidingWindowFilter (基类, ring buffer)
│   │   ├── SlidingWindowMovingAverageFilter  滑动窗口均值
│   │   ├── MinFilter                          窗口最小值
│   │   ├── MaxFilter                          窗口最大值
│   │   └── SortedWindowFilter (排序窗口基类)
│   │       ├── MedianFilter                   中位数
│   │       └── QuantileFilter                 分位数
│   ├── ExponentialMovingAverageFilter  指数移动均值 (无需窗口)
│   └── StreamingFilter (批量窗口基类, O(1)内存)
│       ├── StreamingMinFilter           批量最小值
│       ├── StreamingMaxFilter           批量最大值
│       └── StreamingMovingAverageFilter 批量均值
│
├── 采样控制类 — 控制值的发送频率
│   ├── ThrottleFilter              限流: 两次发送最小间隔
│   ├── ThrottleWithPriorityFilter  优先限流: 特定值立即发送
│   ├── ThrottleWithPriorityNanFilter NaN优先限流 (特化版)
│   ├── ThrottleAverageFilter       限流均值: 时间段内取均值后发送
│   ├── DebounceFilter              去抖: 延迟发送, 新值覆盖旧定时器
│   ├── HeartbeatFilter             心跳: 周期性重发最后值
│   ├── TimeoutFilterBase (Component+Filter)
│   │   ├── TimeoutFilterLast       超时发最后值
│   │   └── TimeoutFilterConfigured 超时发配置值
│   ├── SkipInitialFilter           跳过前N个值
│   └── DeltaFilter                 差值过滤: 仅在变化超阈值时发送
│
├── 值筛选类 — 根据值的内容决定是否转发
│   ├── FilterOutValueFilter<N>     过滤掉特定值
│   └── OrFilter<N>                 多个过滤器的逻辑OR
│
└── 未分类
    └── RoundSignificantDigitsFilter<Digits>  有效数字取整 (模板类)
```

### 15.4 数值变换类过滤器

#### OffsetFilter — 偏移

最简单的变换过滤器，将输入值加上固定偏移量：

```cpp
optional<float> new_value(float value) override {
  return value + this->offset_.value();   // TemplatableFn<float>, 可为常量或lambda
}
```

**YAML 示例**：
```yaml
filters:
  - offset: 2.0          # 温度 +2°C 校准
  - offset: !lambda "return x * 0.5;"  # 动态偏移
```

#### MultiplyFilter — 缩放

将输入值乘以固定系数：

```cpp
optional<float> new_value(float value) override {
  return value * this->multiplier_.value();
}
```

**YAML 示例**：
```yaml
filters:
  - multiply: 1.8        # °C → °F 的缩放因子
```

> Offset + Multiply 组合可以实现简单的线性校准，如 `multiply: 1.8; offset: 32` 即 °C → °F 转换。但对于多点校准，应使用 `calibrate_linear`。

#### CalibrateLinearFilter — 分段线性校准

最常用的校准过滤器，支持两种方法：

| 方法 | 说明 | 计算方式 |
|------|------|----------|
| `least_squares` | 最小二乘拟合（默认） | 所有点拟合一条直线 y = kx + b |
| `exact` | 精确分段映射 | 每对相邻点之间独立线性插值 |

**最小二乘法**：所有校准点拟合为单一线性函数 `y = kx + b`，Python 侧使用 `fit_linear()` 计算系数，生成的 C++ 代码仅存储一对 `[k, b, NaN]`：

```cpp
// least_squares: 2个校准点 → 1个线性函数
// 如 0.0 -> 0.0, 100.0 -> 105.0 → k=1.05, b=0
optional<float> new_value(float value) override {
  return calibrate_linear_compute(this->linear_functions_.data(), N, value);
}
// calibrate_linear_compute: 找到 value < functions[i][2] 的段(或最后段)
// → (value * functions[i][0]) + functions[i][1]
```

**精确分段法**：每对校准点 `(x_i, y_i)` 和 `(x_i+1, y_i+1)` 之间计算独立的斜率和截距，Python 侧使用 `map_linear()` 生成多段 `[k, b, x_max]`，C++ 侧按 `x_max` 分段查找：

```cpp
// exact: 3个校准点 → 2个线性函数段
// 如 0->0, 50->52, 100->105 → [[1.04, 0, 50], [1.06, -3, NaN]]
// 0≤x<50: y = 1.04*x + 0
// x≥50:   y = 1.06*x - 3
```

**YAML 示例**：
```yaml
filters:
  - calibrate_linear:
      datapoints:
        - 0.0 -> 0.0
        - 50.0 -> 52.0
        - 100.0 -> 105.0
      method: exact           # 或 least_squares (默认)
```

> `CalibrateLinearFilter` 是模板类 `CalibrateLinearFilter<N>`，`N` 在代码生成时确定，等于校准段数。所有系数存储在 `std::array<std::array<float, 3>, N>` 中，零堆分配。

#### CalibratePolynomialFilter — 多项式校准

高阶校准过滤器，使用多项式 `y = Σ a_i * x^i` 映射输入值：

```cpp
optional<float> new_value(float value) override {
  return calibrate_polynomial_compute(this->coefficients_.data(), N, value);
}
// 实现: res = 0, x = 1; for i in 0..N-1: res += x * coefficients[i]; x *= value;
```

Python 侧通过 `_lstsq()` 求解最小二乘拟合系数。用户指定 `datapoints`（校准点）和 `degree`（多项式阶数），要求 `degree < len(datapoints)`。

**YAML 示例**：
```yaml
filters:
  - calibrate_polynomial:
      datapoints:
        - 0.0 -> 0.0
        - 10.0 -> 10.5
        - 20.0 -> 21.2
      degree: 2              # 二阶多项式，需要至少3个校准点
```

#### ToNTCResistanceFilter / ToNTCTemperatureFilter — NTC 热敏电阻校准

这两个过滤器实现了 **Steinhart-Hart 方程**的正向和逆向计算：

- **`to_ntc_resistance`**：温度 → 电阻（逆向 Steinhart-Hart）
- **`to_ntc_temperature`**：电阻 → 温度（正向 Steinhart-Hart，最常用）

Steinhart-Hart 方程：`1/T = a + b·ln(R) + c·ln(R)³`

校准参数支持三种输入方式：

| 输入方式 | 说明 | 适用场景 |
|----------|------|----------|
| 三组 `(温度, 电阻)` | 自动计算 a、b、c | 最精确，需要3个校准点 |
| `{b_constant, reference_temperature, reference_resistance}` | B常数法，c=0 | 简化模型，仅2个参数 |
| `{a, b, c}` | 直接提供系数 | 已知 Steinhart-Hart 系数 |

**YAML 示例**（最常见的电阻→温度转换）：
```yaml
filters:
  - to_ntc_temperature:
      calibration:
        - 25°C -> 10000Ω     # 三点 Steinhart-Hart 校准
        - 0°C -> 32550Ω
        - 100°C -> 58Ω
```

#### ClampFilter — 限幅

将值限制在 `[min, max]` 范围内，超范围值有两种处理方式：

```cpp
optional<float> new_value(float value) override {
  if (value < min_) {
    return ignore_out_of_range_ ? {} : min_;  // 丢弃 或 截断到最小值
  }
  if (value > max_) {
    return ignore_out_of_range_ ? {} : max_;  // 丢弃 或 截断到最大值
  }
  return value;  // 在范围内，正常传递
}
```

**YAML 示例**：
```yaml
filters:
  - clamp:
      min_value: 0
      max_value: 100
      ignore_out_of_range: false  # true=丢弃, false=截断(默认)
```

#### RoundFilter / RoundMultipleFilter / RoundSignificantDigitsFilter — 取整

三种不同的取整策略：

| 过滤器 | 算法 | YAML 关键字 |
|--------|------|-------------|
| `RoundFilter` | `round(10^N * x) / 10^N`，保留 N 位小数 | `round: 2` |
| `RoundMultipleFilter` | `x - remainder(x, multiple)`，取整到倍数 | `round_to_multiple_of: 0.5` |
| `RoundSignificantDigitsFilter<Digits>` | `round(x * 10^(Digits-1-log10(x))) / 10^(...)`，保留 D 位有效数字 | `round_to_significant_digits: 3` |

`RoundSignificantDigitsFilter` 是模板类 `<uint8_t Digits>`，在编译期确定精度位数，范围 1-6。

#### LambdaFilter / StatelessLambdaFilter — 自定义 Lambda

最灵活的过滤器，用户通过 Lambda 表达式定义任意变换逻辑：

```cpp
// LambdaFilter — 使用 std::function，32字节
optional<float> new_value(float value) override {
  return this->lambda_filter_(value);  // float → optional<float>
}

// StatelessLambdaFilter — 仅存储函数指针，4字节
optional<float> new_value(float value) override {
  return this->lambda_filter_(value);  // 无捕获的函数指针
}
```

**代码生成的优化**：Python 代码生成器在 `new_lambda_pvariable()` 中自动判断 Lambda 是否有捕获变量——无捕获的 Lambda 使用 `StatelessLambdaFilter`（4字节），有捕获的 Lambda 使用 `LambdaFilter`（32字节）。

**YAML 示例**：
```yaml
filters:
  - lambda: "return x * 1.8 + 32;"           # °C → °F（无捕获 → StatelessLambdaFilter）
  - lambda: "if (x > 100) return {}; return x;"  # 过滤掉 >100 的值
```

> 返回 `{}`（空 optional）意味着丢弃该值——这是 Lambda 过滤器实现值筛选的唯一方式。

### 15.5 窗口统计类过滤器

窗口统计类过滤器基于最近 N 个读数计算统计量，是传感器去噪的核心工具。

#### SlidingWindowFilter — 滑动窗口基类

所有窗口统计过滤器的基础，使用**环形缓冲区**（`FixedRingBuffer<float>`）维护固定大小的滑动窗口：

```cpp
class SlidingWindowFilter : public Filter {
  FixedRingBuffer<float> window_;  // 环形缓冲区，自动覆盖最旧值
  uint16_t send_every_;            // 每N个输入值发送一次结果
  uint16_t send_at_;               // 当前计数器
};

optional<float> new_value(float value) final {
  this->window_.push_overwrite(value);    // 新值入窗口，最旧值被覆盖

  if (++this->send_at_ >= this->send_every_) {
    this->send_at_ = 0;
    return this->compute_result();        // 计算统计量并发送
  }
  return {};                              // 未到发送时机，丢弃
}
```

**`send_every` 和 `send_first_at` 机制**：

| 参数 | 含义 | 默认值 |
|------|------|--------|
| `window_size` | 窗口大小（参与计算的值数量） | 因过滤器而异 |
| `send_every` | 每多少个输入值发送一次结果 | 默认等于 window_size |
| `send_first_at` | 首次发送在第几个输入值时 | 默认 1 |

`send_first_at` 的实现巧妙：初始计数器 `send_at_ = send_every_ - send_first_at`，这样第 `send_first_at` 个值时计数器就达到 `send_every_` 触发发送。

**FixedRingBuffer 设计**：

```cpp
template<typename T, size_t MAX_CAPACITY = 65535> class FixedRingBuffer {
  // 一次 init(capacity) 分配，后续 push_overwrite 自动覆盖最旧值
  // 无 pop_front() 开销，无 deque 碎片问题
  // 支持 range-based for 循环遍历窗口内所有值
};
```

对 `trivially_copyable` 类型（如 `float`），使用 `::operator new` 而非 `new T[]` 分配内存，避免不必要的初始化开销。

#### SlidingWindowMovingAverageFilter — 滑动窗口均值

最常用的去噪过滤器，对窗口内所有有效值（跳过 NaN）取平均：

```cpp
float compute_result() override {
  float sum = 0;
  size_t valid_count = 0;
  for (float v : this->window_) {
    if (!std::isnan(v)) { sum += v; valid_count++; }
  }
  return valid_count ? sum / valid_count : NAN;
}
```

**YAML 示例**：
```yaml
filters:
  - sliding_window_moving_average:
      window_size: 15      # 15个值的滑动窗口
      send_every: 15       # 每15个值发送一次均值
      send_first_at: 1     # 第1个值时就开始计算
```

#### MinFilter / MaxFilter — 窗口最小值/最大值

继承 `MinMaxFilter`（基于 `SlidingWindowFilter`），使用 `find_extremum_<Compare>()` 查找窗口内的极值：

```cpp
// MinFilter
float compute_result() override { return this->find_extremum_<std::less<float>>(); }
// MaxFilter
float compute_result() override { return this->find_extremum_<std::greater<float>>(); }

// 通用实现：遍历窗口，跳过 NaN，用比较器找极值
template<typename Compare> float find_extremum_() {
  float result = NAN;
  Compare comp;
  for (float v : this->window_) {
    if (!std::isnan(v))
      result = std::isnan(result) ? v : (comp(v, result) ? v : result);
  }
  return result;
}
```

#### MedianFilter — 中位数

继承 `SortedWindowFilter`，使用 `std::nth_element` 做 O(n) 部分排序而非完整排序：

```cpp
float compute_result() override {
  FixedVector<float> values = this->get_window_values_(); // 复制窗口，去除NaN
  size_t size = values.size();
  size_t mid = size / 2;

  if (size % 2) {
    // 奇数个元素：nth_element 找中间元素
    std::nth_element(values.begin(), values.begin() + mid, values.end());
    return values[mid];
  }
  // 偶数个元素：nth_element 找上半中位数 + max_element 找下半中位数
  std::nth_element(values.begin(), values.begin() + mid, values.end());
  float upper = values[mid];
  float lower = *std::max_element(values.begin(), values.begin() + mid);
  return (lower + upper) / 2.0f;
}
```

> `FixedVector` 是 ESPHome 的固定容量向量（预分配，无增长），在此用于一次性复制窗口数据做排序计算，避免修改环形缓冲区本身。

#### QuantileFilter — 分位数

继承 `SortedWindowFilter`，使用 `std::nth_element` 计算 0-1 范围的分位数：

```cpp
float compute_result() override {
  FixedVector<float> values = this->get_window_values_();
  size_t position = ceilf(values.size() * this->quantile_) - 1;
  std::nth_element(values.begin(), values.begin() + position, values.end());
  return values[position];
}
```

**YAML 示例**：
```yaml
filters:
  - quantile:
      window_size: 10
      send_every: 10
      quantile: 0.9          # 90%分位数（默认0.9）
```

#### ExponentialMovingAverageFilter — 指数移动均值

不需要滑动窗口的均值过滤器，使用指数衰减权重：

```cpp
optional<float> new_value(float value) override {
  if (!std::isnan(value)) {
    if (this->first_value_) {
      this->accumulator_ = value;           // 首值直接作为初始值
      this->first_value_ = false;
    } else {
      // EMA公式: new_avg = α * new_value + (1 - α) * old_avg
      this->accumulator_ = (this->alpha_ * value) + (1.0f - this->alpha_) * this->accumulator_;
    }
  }

  const float average = std::isnan(value) ? value : this->accumulator_;
  // send_every 计数逻辑（与 SlidingWindowFilter 相同）
}
```

**α（alpha）参数的意义**：
- α 越大（接近 1）→ 越重视最新值，响应快但平滑少
- α 越小（接近 0）→ 越重视历史均值，响应慢但平滑强
- 默认 α = 0.1

**与 SlidingWindowMovingAverage 的对比**：

| 特性 | 滑动窗口均值 | 指数移动均值 |
|------|-------------|-------------|
| 内存 | O(window_size) | O(1)（仅 accumulator_） |
| 响应速度 | 窗口大小的延迟 | 由 α 控制 |
| 计算复杂度 | 每次遍历窗口 | 每次仅1次乘加 |
| 历史权重 | 窗口内等权 | 指数衰减 |
| 适用场景 | 短期突发去噪 | 长期趋势平滑 |

**YAML 示例**：
```yaml
filters:
  - exponential_moving_average:
      alpha: 0.1             # 衰减因子（默认0.1）
      send_every: 15
      send_first_at: 1
```

#### StreamingFilter — 批量窗口优化（O(1) 内存）

当 `window_size == send_every` 时（最常见的配置），滑动窗口不需要保留所有中间值——只需一个统计量（min/max/sum）即可。代码生成器自动选择 `StreamingFilter` 替代 `SlidingWindowFilter`：

```python
# Python 侧自动优化（__init__.py）
@FILTER_REGISTRY.register("min", Filter, MIN_SCHEMA)
async def min_filter_to_code(config, filter_id):
  window_size = config[CONF_WINDOW_SIZE]
  send_every = config[CONF_SEND_EVERY]

  if window_size == send_every:
    # 批量窗口 → 使用 O(1) StreamingMinFilter
    rhs = StreamingMinFilter.new(window_size, send_first_at)
    return cg.Pvariable(filter_id, rhs, StreamingMinFilter)
  # 普通滑动窗口 → 使用 O(n) MinFilter + ring buffer
  rhs = MinFilter.new(window_size, send_every, send_first_at)
  return cg.Pvariable(filter_id, rhs, MinFilter)
```

**内存对比**（`window_size = 5000` 的场景）：

| 过滤器 | 内存占用 | 说明 |
|--------|---------|------|
| SlidingWindowFilter (ring buffer) | ~20KB | 5000 × 4字节 float |
| StreamingFilter | ~4字节 | 仅跟踪 min/max/sum |

节省 99.98% 内存，对 RAM 受限的 MCU 至关重要。

StreamingFilter 的三个子类实现极简：

```cpp
class StreamingMinFilter : public StreamingFilter {
  float current_min_{NAN};
  void process_value(float v) override {
    if (!std::isnan(v)) current_min_ = std::isnan(current_min_) ? v : std::min(current_min_, v);
  }
  float compute_batch_result() override { return current_min_; }
  void reset_batch() override { current_min_ = NAN; }
};

class StreamingMaxFilter : public StreamingFilter {  // 类似，用 std::max
};

class StreamingMovingAverageFilter : public StreamingFilter {
  float sum_{0.0f}; size_t valid_count_{0};
  void process_value(float v) override { if (!std::isnan(v)) { sum_ += v; valid_count_++; } }
  float compute_batch_result() override { return valid_count_ > 0 ? sum_ / valid_count_ : NAN; }
  void reset_batch() override { sum_ = 0; valid_count_ = 0; }
};
```

### 15.6 采样控制类过滤器

采样控制类过滤器不改变值的数值，而是控制值的**发送时机和频率**——丢弃值意味着不发送，而非修改值。

#### ThrottleFilter — 限流

最简单的采样控制：两次发送之间必须间隔至少 `min_time_between_inputs` 毫秒：

```cpp
optional<float> new_value(float value) override {
  const uint32_t now = App.get_loop_component_start_time();
  if (this->last_input_ == 0 || now - this->last_input_ >= min_time_between_inputs_) {
    this->last_input_ = now;
    return value;    // 间隔足够 → 发送
  }
  return {};          // 间隔不足 → 丢弃
}
```

**YAML 示例**：
```yaml
filters:
  - throttle: 10s       # 最少10秒发送一次
```

#### ThrottleWithPriorityFilter — 优先限流

与 `ThrottleFilter` 相同的限流逻辑，但**特定值立即发送**（不限流）：

```cpp
// 通用版: value_list_matches_any() 检查值是否在优先列表中
optional<float> throttle_with_priority_new_value(...) {
  const uint32_t now = App.get_loop_component_start_time();
  if (last_input == 0 || now - last_input >= min_time ||
      value_list_matches_any(parent, value, values, count)) {
    last_input = now;
    return value;    // 间隔足够 或 值在优先列表 → 发送
  }
  return {};          // 否则 → 丢弃
}
```

**ThrottleWithPriorityNanFilter — NaN 优先特化版**：

最常见的优先值是 `NaN`（传感器断连时发布 NaN）。代码生成器自动检测——当 YAML 中的 `value` 全为 NaN 时，使用更轻量的 `ThrottleWithPriorityNanFilter`：

```cpp
// 特化版: 直接 std::isnan() 检查，无 TemplatableFn 数组开销
optional<float> new_value(float value) override {
  const uint32_t now = App.get_loop_component_start_time();
  if (last_input_ == 0 || now - last_input_ >= min_time_ || std::isnan(value)) {
    last_input_ = now;
    return value;
  }
  return {};
}
```

**YAML 示例**：
```yaml
filters:
  - throttle_with_priority:
      timeout: 10s
      value: nan            # NaN值立即发送（默认），其他值限流
```

#### ThrottleAverageFilter — 限流均值

在限流的同时对时间段内的所有值取平均——既控制发送频率，又保证数据不失真：

```cpp
optional<float> new_value(float value) override {
  if (std::isnan(value)) {
    this->have_nan_ = true;    // 标记有 NaN
  } else {
    this->sum_ += value;       // 累加
    this->n_++;                // 计数
  }
  return {};                    // 永不立即发送
}

// initialize() 中注册定时器：
void initialize(Sensor *parent, Filter *next) override {
  Filter::initialize(parent, next);
  App.scheduler.set_interval(this, this->time_period_, [this]() {
    if (this->n_ == 0) {
      if (this->have_nan_) this->output(NAN);  // 仅 NaN → 输出 NaN
    } else {
      this->output(this->sum_ / this->n_);      // 有值 → 输出均值
      this->sum_ = 0.0f; this->n_ = 0;          // 重置
    }
    this->have_nan_ = false;
  });
}
```

**内存优化**：`n_` 和 `have_nan_` 打包在一个 32 位字中——`n_` 占 31 位（最大 2^31 ≈ 2.1B，远大于实际采样率 × 24小时），`have_nan_` 占 1 位。

**YAML 示例**：
```yaml
filters:
  - throttle_average: 60s     # 每60秒发送一次时间段内的均值
```

#### DebounceFilter — 去抖

收到新值后不立即发送，而是延迟 `time_period` 毫秒——如果延迟期间收到新值，旧定时器被取消（同 key），新值重新开始延迟：

```cpp
optional<float> new_value(float value) override {
  // self key: 同一个 Filter 的定时器自动互斥（cancel + replace）
  App.scheduler.set_timeout(this, this->time_period_, [this, value]() {
    this->output(value);
  });
  return {};    // 永不立即发送，等定时器到期
}
```

> 与 `ThrottleFilter` 的区别：Throttle 是"发送后静默 N 秒"，Debounce 是"收到后等 N 秒才发送（新值覆盖等待）"。Debounce 适合按钮/开关场景（最终值才重要），Throttle 适合传感器场景（定期采样）。

#### HeartbeatFilter — 心跳

周期性重发最后收到的值，确保 Home Assistant 始终看到"活"的传感器状态：

```cpp
optional<float> new_value(float value) override {
  this->last_input_ = value;
  this->has_value_ = true;
  if (this->optimistic_) return value;  // optimistic: 每次都发送 + 心跳重发
  return {};                             // 否则: 仅通过心跳重发
}

// initialize() 中注册定时器：
void initialize(Sensor *parent, Filter *next) override {
  App.scheduler.set_interval(this, this->time_period_, [this]() {
    if (this->has_value_) this->output(this->last_input_);
  });
}
```

`optimistic` 模式：收到新值时立即发送（`return value`），同时仍然心跳重发。非 optimistic 模式：仅在心跳定时器到期时发送。

**YAML 示例**：
```yaml
filters:
  - heartbeat: 60s                    # 每60秒重发最后值
  - heartbeat:
      period: 60s
      optimistic: true                # 收到新值也立即发送
```

#### TimeoutFilter — 超时

当传感器长时间不发值时，自动发布一个替代值。**有意继承 `Component`**（而非使用 Scheduler）——这是经过深思熟虑的 RAM 优化设计：

```
为什么 TimeoutFilter 继承 Component 而不使用 Scheduler？

每台设备可能有多个传感器使用 timeout filter（如多LD2450板）。
如果每个 armed filter 都持有一个 live SchedulerItem，
RAM 开销远大于 Component 的 BSS 字节。

SchedulerItem ≈ 80+ bytes (live, while armed)
Component overhead ≈ 几字节 (one-time, BSS)

loop() 方案的优势:
- enable_loop()/disable_loop() 在无超时时零开销
- armed 时仅一个 timestamp 比较，无 scheduler cancel/insert 路径
```

两个子类：

| 过滤器 | 超时后发布的值 | 适用场景 |
|--------|---------------|---------|
| `TimeoutFilterLast` | 最后收到的值 | "超时后重发最后一次读数" |
| `TimeoutFilterConfigured` | 用户配置的固定值 | "超时后发布 NaN/0 等" |

```cpp
// TimeoutFilterLast: 超时发布最后值
optional<float> new_value(float value) override {
  this->pending_value_ = value;        // 记住当前值
  this->timeout_start_time_ = millis(); // 记录超时起点
  this->enable_loop();                  // 启用 loop 检查
  return value;                         // 正常发送当前值
}

// loop() 中检查超时
void loop() override {
  const uint32_t now = App.get_loop_component_start_time();
  if (now - this->timeout_start_time_ >= this->time_period_) {
    this->output(this->get_output_value()); // 超时 → 发布替代值
    this->disable_loop();                    // 停止 loop 检查
  }
}
```

**YAML 示例**：
```yaml
filters:
  - timeout:                    # 超时5秒后发布NaN
      timeout: 5s
      value: nan
  - timeout: 5s                 # 简写: 默认 value=nan
  - timeout:                    # 超时5秒后重发最后值
      timeout: 5s
      value: last
```

> `TimeoutFilter` 在 LD2450/LD2412 等雷达组件中广泛使用——当目标消失后雷达不再发值，timeout 确保距离传感器及时归零。

#### SkipInitialFilter — 跳过初始值

跳过前 N 个值，之后变为透明（所有值直接通过）：

```cpp
optional<float> new_value(float value) override {
  if (this->num_to_ignore_ > 0) {
    this->num_to_ignore_--;
    return {};    // 跳过
  }
  return value;   // N个值后，直接通过
}
```

适用场景：传感器刚启动时的读数不稳定，跳过前几个值避免发送错误数据。

#### DeltaFilter — 差值过滤

仅在新值与旧值的差值在指定范围内时发送：

```cpp
optional<float> new_value(float value) override {
  if (std::isnan(this->last_value_)) {
    this->last_value_ = value;
    return value;    // 首值总是发送
  }
  float ref = this->baseline_(this->last_value_); // 默认: baseline = last_value
  float min = fabsf(this->min_a0_ + ref * this->min_a1_); // 线性阈值
  float max = fabsf(this->max_a0_ + ref * this->max_a1_);
  float delta = fabsf(value - ref);
  if (delta > min && delta <= max) {  // delta 在 [min, max) 范围内
    this->last_value_ = value;
    return value;
  }
  return {};   // 变化太小或太大 → 丢弃
}
```

**阈值是线性方程**：`min_threshold = |min_a0 + value * min_a1|`，支持固定阈值（仅 a0）和百分比阈值（仅 a1）的混合：

| YAML 配置 | 计算方式 | 含义 |
|-----------|---------|------|
| `delta: 0.1` | min=0.1, max=∞ | 变化至少 > 0.1 才发送 |
| `delta: 5%` | min=last*0.05, max=∞ | 变化至少 > 5% 才发送 |
| `delta: {min_value: 0.1, max_value: 10}` | min=0.1, max=10 | 变化在 0.1~10 之间才发送 |

`baseline` 可自定义——默认是上一个值，但可以设为 lambda 函数（如设为0，则 delta 变为绝对值过滤）。

**YAML 示例**：
```yaml
filters:
  - delta: 0.1                    # 变化 > 0.1 才发送
  - delta: 5%                     # 变化 > 5% 才发送
  - delta:
      min_value: 0.1
      max_value: 10               # 变化在 0.1~10 之间才发送
      baseline: !lambda "return 0;" # 基线为0（变为绝对值过滤）
```

### 15.7 值筛选类过滤器

#### FilterOutValueFilter<N> — 过滤特定值

丢弃匹配指定值列表的读数，其他值正常通过：

```cpp
optional<float> new_value(float value) override {
  if (this->value_matches_any_(value))
    return {};    // 匹配 → 丢弃
  return value;   // 不匹配 → 通过
}
```

**值的匹配考虑精度**：`value_list_matches_any()` 使用传感器的 `accuracy_decimals` 做 `round()` 比较——精度为 2 位小数的传感器，值 1.005 和配置值 1.01 会被视为匹配（先 round 再比较）。NaN 值的匹配是特殊处理的。

**YAML 示例**：
```yaml
filters:
  - filter_out: nan              # 过滤掉 NaN 值
  - filter_out: [0.0, nan]       # 过滤掉 0 和 NaN
```

> `FilterOutValueFilter` 是模板类 `<size_t N>`，N 在代码生成时确定。值列表存储在 `std::array<TemplatableFn<float>, N>` 中，零堆分配。

#### OrFilter<N> — 逻辑 OR 组合

将多个过滤器逻辑 OR 组合——**任一子过滤器输出值，则最终输出该值**：

```cpp
template<size_t N> class OrFilter : public Filter {
  std::array<Filter *, N> filters_;    // 子过滤器数组
  PhiNode phi_{this};                  // 特殊的链尾节点
  bool has_value_{false};              // 本轮是否已有子过滤器输出

  class PhiNode : public Filter {
    OrFilter *or_parent_;
    optional<float> new_value(float value) override {
      if (!this->or_parent_->has_value_) {
        this->or_parent_->output(value); // 首次输出 → 发送到链尾
        this->or_parent_->has_value_ = true;
      }
      return {};   // 后续输出 → 丢弃（已发过一次了）
    }
  };
};
```

**PhiNode 的工作机制**：

所有子过滤器共享同一个 `PhiNode` 作为链尾。当任一子过滤器输出值时，`PhiNode` 将该值通过 `OrFilter::output()` 发送到主链的下一个过滤器。同一轮输入中，只有第一个子过滤器的输出被传递，后续子过滤器的输出被丢弃——保证每个输入值最多产生一个输出。

```
新值 → OrFilter::new_value(value)
         │  has_value_ = false
         │  对每个子过滤器调用 input(value)
         │
         ├→ 子Filter1 → ... → PhiNode.output(v1) → OrFilter.output(v1) → 下一个主链Filter
         │                    has_value_ = true
         ├→ 子Filter2 → ... → PhiNode → has_value_ 已为 true → 丢弃
         ├→ 子Filter3 → ... → PhiNode → has_value_ 已为 true → 丢弃
```

**YAML 示例**：
```yaml
filters:
  - or:
      - throttle: 10s            # 每10秒发一次
      - delta: 5%                # 变化超5%也发一次
```

> 这是 "定期采样 + 大变化立即发" 的经典组合——日常10秒一次采样，突发大变化立即感知。

### 15.8 过滤器的条件编译与内存优化

整个过滤器系统通过 `USE_SENSOR_FILTER` 宏裁剪：

```cpp
// sensor.h
#ifdef USE_SENSOR_FILTER
#include "esphome/components/sensor/filter.h"
#endif

// sensor.cpp
#ifdef USE_SENSOR_FILTER
  if (this->filter_list_ == nullptr) {
    this->internal_send_state_to_frontend(state);
  } else {
    this->filter_list_->input(state);
  }
#else
  this->internal_send_state_to_frontend(state);  // 无过滤器 → 直接发布
#endif
```

Python 侧仅在 YAML 中配置了 `filters:` 时才添加 `USE_SENSOR_FILTER` 定义：

```python
if config.get(CONF_FILTERS):  # must exist and not be empty
  cg.add_define("USE_SENSOR_FILTER")
  filters = await build_filters(config[CONF_FILTERS])
  cg.add(var.set_filters(filters))
```

**模板类参数化**：多个过滤器使用 `<size_t N>` 模板参数，N 在代码生成时确定（等于配置的列表长度），保证所有数据存储在 `std::array` 中，零堆分配：

| 过滤器 | 模板参数 N | 存储结构 |
|--------|-----------|---------|
| `FilterOutValueFilter<N>` | 过滤值数量 | `std::array<TemplatableFn<float>, N>` |
| `ThrottleWithPriorityFilter<N>` | 优先值数量 | 同上 |
| `OrFilter<N>` | 子过滤器数量 | `std::array<Filter *, N>` |
| `CalibrateLinearFilter<N>` | 校准段数 | `std::array<std::array<float, 3>, N>` |
| `CalibratePolynomialFilter<N>` | 系数数量 | `std::array<float, N>` |
| `RoundSignificantDigitsFilter<Digits>` | 有效数字位数 | 无额外存储 |

**StatelessLambdaFilter 优化**：无捕获的 Lambda 使用函数指针（4字节）而非 `std::function`（32字节），节省 28 字节/过滤器。

**TimeoutFilter 继承 Component 而非使用 Scheduler**：已在上文详述——多传感器场景下，Component 的 BSS 开销远小于 SchedulerItem 的 live 开销。

### 15.9 过滤器组合实战示例

#### 温度传感器去噪 + 校准

```yaml
sensor:
  - platform: dht
    temperature:
      name: "Living Room Temperature"
      accuracy_decimals: 1
      filters:
        - offset: -0.5              # 校准偏移
        - sliding_window_moving_average:  # 去噪
            window_size: 10
            send_every: 5
```

#### 功率传感器限流 + 去抖

```yaml
sensor:
  - platform: bl0942
    power:
      name: "Power"
      filters:
        - throttle_average: 60s     # 每60秒发均值，避免频繁更新
        - filter_out: 0.0           # 过滤掉0W（断线时误报）
```

#### 雷达距离传感器超时 + Delta

```yaml
sensor:
  - platform: ld2450
    distance:
      name: "Target Distance"
      filters:
        - delta: 0.1               # 变化 >0.1m 才发送
        - timeout:                  # 5秒无新值 → 发布NaN
            timeout: 5s
            value: nan
```

#### 复合条件："定期采样或大变化立即发"

```yaml
sensor:
  - platform: adc
    voltage:
      name: "Battery Voltage"
      filters:
        - or:
            - throttle: 30s         # 每30秒发一次
            - delta: 5%             # 变化超5%立即发
        - clamp:
            min_value: 0
            max_value: 15
            ignore_out_of_range: true  # 超范围直接丢弃
```

### 15.10 关键文件索引

| 文件 | 作用 |
|------|------|
| `esphome/components/sensor/filter.h` | 所有 Filter 类定义（743行），含继承体系、模板类 |
| `esphome/components/sensor/filter.cpp` | Filter 链流转机制、各过滤器实现 |
| `esphome/components/sensor/sensor.h` | Sensor 类 filter_list_ 成员、add/set/clear_filters 方法 |
| `esphome/components/sensor/sensor.cpp` | publish_state() 过滤器链入口、internal_send_state_to_frontend() |
| `esphome/components/sensor/__init__.py` | FILTER_REGISTRY 注册、各过滤器 schema + to_code()、StreamingFilter 自动优化 |
| `esphome/core/helpers.h` | FixedRingBuffer（环形缓冲区）、FixedVector、TemplatableFn |
| `esphome/core/defines.h` | USE_SENSOR_FILTER 条件编译宏（自动生成） |
| `esphome/const.py` | TYPE_GIT/TYPE_LOCAL/CONF_EXTERNAL_COMPONENTS 等常量 |
