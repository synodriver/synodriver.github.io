+++
date = '2026-04-28T22:10:28+08:00'
draft = false
title = 'ESPHOME ai 教程'
author = 'synodriver'
+++
# ESPHome AI 协作指南

本文档为与本项目交互的 AI 模型提供必要的上下文。遵循这些指南将确保一致性并维护代码质量。

## 1. 项目概述与目标

*   **主要目标：** ESPHome 是一个使用简单而强大的 YAML 配置文件来配置微控制器（如 ESP32、ESP8266、RP2040 以及基于 LibreTiny 的芯片）的系统。它生成可被编译并烧录到这些设备的 C++ 固件，允许用户通过家庭自动化系统远程控制它们。
*   **业务领域：** 物联网（IoT）、家庭自动化。

## 2. 核心技术栈

*   **语言：** Python（>=3.11）、C++（gnu++20）
*   **框架与运行时：** PlatformIO、Arduino、ESP-IDF。
*   **构建系统：** PlatformIO 是主要构建系统，CMake 作为替代方案。
*   **配置：** YAML。
*   **关键库/依赖：**
    *   **Python：** `voluptuous`（配置验证）、`PyYAML`（解析配置文件）、`paho-mqtt`（MQTT 通信）、`tornado`（Web 服务器）、`aioesphomeapi`（原生 API）。
    *   **C++：** `ArduinoJson`（JSON 序列化/反序列化）、`AsyncMqttClient-esphome`（MQTT）、`ESPAsyncWebServer`（Web 服务器）。
*   **包管理器：** `pip`（Python 依赖）、`platformio`（C++/PlatformIO 依赖）。
*   **通信协议：** Protobuf（原生 API）、MQTT、HTTP。

## 3. 架构模式

*   **整体架构：** 本项目采用代码生成架构。Python 代码解析用户定义的 YAML 配置文件，并生成 C++ 源代码。随后，该 C++ 代码通过 PlatformIO 被编译并烧录到目标微控制器。

*   **目录结构理念：**
    *   `/esphome`：包含 ESPHome 应用程序的核心 Python 源代码。
    *   `/esphome/components`：包含可在 ESPHome 配置中使用的各个组件。每个组件是一个自包含的单元，拥有自己的 C++ 和 Python 代码。
    *   `/tests`：包含 Python 代码的所有单元测试和集成测试。
    *   `/docker`：包含用于在容器中构建和运行 ESPHome 的 Docker 相关文件。
    *   `/script`：包含用于开发和维护的辅助脚本。

*   **核心架构组件：**
    1.  **配置系统**（`esphome/config*.py`）：处理 YAML 解析和使用 Voluptuous 的验证、Schema 定义以及多平台配置。
    2.  **代码生成**（`esphome/codegen.py`、`esphome/cpp_generator.py`）：管理 Python 到 C++ 的代码生成、模板处理和构建标志管理。
    3.  **组件系统**（`esphome/components/`）：包含具有平台特定实现和依赖管理的模块化硬件与软件组件。
    4.  **核心框架**（`esphome/core/`）：管理应用程序生命周期、硬件抽象和组件注册。
    5.  **仪表盘**（`esphome/dashboard/`）：用于设备配置、管理和 OTA 更新的 Web 界面。

*   **平台支持：**
    1.  **ESP32**（`components/esp32/`）：乐鑫 ESP32 系列。支持多种变体（原版、C2、C3、C5、C6、H2、P4、S2、S3），使用 ESP-IDF 框架。Arduino 框架仅支持部分变体（原版、C3、S2、S3）。
    2.  **ESP8266**（`components/esp8266/`）：乐鑫 ESP8266。仅支持 Arduino 框架，存在内存限制。
    3.  **RP2040**（`components/rp2040/`）：树莓派 Pico/RP2040。支持 Arduino 框架和 PIO（可编程 I/O）。
    4.  **LibreTiny**（`components/libretiny/`）：瑞昱和博通芯片。支持多个芯片系列和自动生成的组件。

## 4. 编码规范与风格指南

*   **格式化：**
    *   **Python：** 使用 `ruff` 和 `flake8` 进行代码检查和格式化。配置位于 `pyproject.toml`。
    *   **C++：** 使用 `clang-format` 进行格式化。配置位于 `.clang-format`。

*   **命名规范：**
    *   **Python：** 遵循 PEP 8。使用清晰、描述性的蛇形命名法（snake_case）。
    *   **C++：** 遵循 Google C++ 风格指南，具体规定如下（遵循 clang-tidy 约定）：
        - 函数、方法和变量名：`lower_snake_case`（小写蛇形）
        - 类/结构体/枚举名：`UpperCamelCase`（大写驼峰）
        - 顶级常量（全局/命名空间作用域）：`UPPER_SNAKE_CASE`（全大写蛇形）
        - 函数内局部常量：`lower_snake_case`（小写蛇形）
        - 受保护/私有字段：`lower_snake_case_with_trailing_underscore_`（末尾带下划线的小写蛇形）
        - 优先使用描述性名称而非缩写

*   **C++ 字段可见性：**
    *   **优先使用 `protected`：** 对大多数类字段使用 `protected`，以实现可扩展性和可测试性。字段应采用 `lower_snake_case_with_trailing_underscore_` 命名。
    *   **在安全关键场景使用 `private`：** 当直接字段访问可能引入 bug 或违反不变量时，使用 `private` 可见性：
        1. **指针生命周期问题：** 当 setter 需要验证并存储来自已知列表的指针，以防止悬空引用时。
           ```cpp
           // 辅助函数：在 vector 中查找匹配字符串并返回其指针
           inline const char *vector_find(const std::vector<const char *> &vec, const char *value) {
             for (const char *item : vec) {
               if (strcmp(item, value) == 0)
                 return item;
             }
             return nullptr;
           }

           class ClimateDevice {
            public:
             void set_custom_fan_modes(std::initializer_list<const char *> modes) {
               this->custom_fan_modes_ = modes;
               this->active_custom_fan_mode_ = nullptr;  // 模式更改时重置
             }
             bool set_custom_fan_mode(const char *mode) {
               // 在支持列表中查找模式，并存储该指针（而非输入指针）
               const char *validated_mode = vector_find(this->custom_fan_modes_, mode);
               if (validated_mode != nullptr) {
                 this->active_custom_fan_mode_ = validated_mode;
                 return true;
               }
               return false;
             }
            private:
             std::vector<const char *> custom_fan_modes_;  // 指向 flash 中字符串字面量的指针
             const char *active_custom_fan_mode_{nullptr};  // 必须指向 custom_fan_modes_ 中的条目
           };
           ```
        2. **不变量耦合：** 当多个字段必须保持同步以防止缓冲区溢出或数据损坏时。
           ```cpp
           class Buffer {
            public:
             void resize(size_t new_size) {
               auto new_data = std::make_unique<uint8_t[]>(new_size);
               if (this->data_) {
                 std::memcpy(new_data.get(), this->data_.get(), std::min(this->size_, new_size));
               }
               this->data_ = std::move(new_data);
               this->size_ = new_size;  // 必须与 data_ 保持同步
             }
            private:
             std::unique_ptr<uint8_t[]> data_;
             size_t size_{0};  // 必须与 data_ 的分配大小一致
           };
           ```
        3. **资源管理：** 当 setter 执行派生类可能跳过的清理或注册操作时。
    *   **提供 `protected` 访问器方法：** 当派生类需要对 `private` 成员进行受控访问时。

*   **C++ 预处理器指令：**
    *   **避免使用 `#define` 定义常量：** 不鼓励使用 `#define` 定义常量，应替换为 `const` 变量或枚举。
    *   **仅在以下情况使用 `#define`：**
        - 条件编译（`#ifdef`、`#ifndef`）
        - Python 代码生成期间计算的编译时大小（例如，通过 `cg.add_define()` 配置 `std::array` 或 `StaticVector` 的维度）

*   **C++ 附加规范：**
    *   **成员访问：** 所有类成员访问均加 `this->` 前缀（例如，使用 `this->value_` 而非 `value_`）
    *   **缩进：** 使用空格（每级缩进两个空格），不使用制表符
    *   **类型别名：** 优先使用 `using type_t = int;` 而非 `typedef int type_t;`
    *   **行长：** 每行不超过 120 个字符
    *   **构造函数参数与 setter：** 既**必需**又**不变**（构造后永不改变）的组件属性应作为构造函数参数，而非通过 setter 方法设置。这使依赖关系明确，并防止在未完全初始化的状态下使用对象。在代码生成中，调用 `cg.new_Pvariable()` 或相关辅助函数创建组件时，应将这些属性作为参数传入。
        ```cpp
        // 正确 - 必需的不变依赖作为构造函数参数
        class SourceTextSensor : public text_sensor::TextSensor, public Component {
         public:
          explicit SourceTextSensor(text::Text *source) : source_(source) {}
         protected:
          text::Text *source_;
        };
        ```
        ```cpp
        // 错误 - 必需的不变依赖作为 setter
        class SourceTextSensor : public text_sensor::TextSensor, public Component {
         public:
          void set_source(text::Text *source) { this->source_ = source; }
         protected:
          text::Text *source_{nullptr};
        };
        ```

*   **组件结构：**
    *   **标准文件：**
        ```
        components/[component_name]/
        ├── __init__.py          # 组件配置 Schema 和代码生成
        ├── [component].h        # C++ 头文件（如需要）
        ├── [component].cpp      # C++ 实现（如需要）
        └── [platform]/          # 平台特定实现
            ├── __init__.py      # 平台特定配置
            ├── [platform].h     # 平台 C++ 头文件
            └── [platform].cpp   # 平台 C++ 实现
        ```

    *   **组件元数据：**
        - `DEPENDENCIES`：必需组件列表
        - `AUTO_LOAD`：自动加载的组件
        - `CONFLICTS_WITH`：不兼容的组件
        - `CODEOWNERS`：负责维护的 GitHub 用户名
        - `MULTI_CONF`：是否允许多个实例

*   **代码生成与常用模式：**
    *   **配置 Schema 模式：**
        ```python
        import esphome.codegen as cg
        import esphome.config_validation as cv
        from esphome.const import CONF_KEY, CONF_ID

        CONF_PARAM = "param"  # esphome/const.py 中尚不存在的常量

        my_component_ns = cg.esphome_ns.namespace("my_component")
        MyComponent = my_component_ns.class_("MyComponent", cg.Component)

        CONFIG_SCHEMA = cv.Schema({
            cv.GenerateID(): cv.declare_id(MyComponent),
            cv.Required(CONF_KEY): cv.string,
            cv.Optional(CONF_PARAM, default=42): cv.int_,
        }).extend(cv.COMPONENT_SCHEMA)

        async def to_code(config):
            var = cg.new_Pvariable(config[CONF_ID])
            await cg.register_component(var, config)
            cg.add(var.set_key(config[CONF_KEY]))
            cg.add(var.set_param(config[CONF_PARAM]))
        ```

    *   **C++ 类模式：**
        ```cpp
        namespace esphome::my_component {

        class MyComponent : public Component {
         public:
          void setup() override;
          void loop() override;
          void dump_config() override;

          void set_key(const std::string &key) { this->key_ = key; }
          void set_param(int param) { this->param_ = param; }

         protected:
          std::string key_;
          int param_{0};
        };

        }  // namespace esphome::my_component
        ```

    *   **常用组件示例：**
        - **传感器（Sensor）：**
          ```python
          from esphome.components import sensor
          CONFIG_SCHEMA = sensor.sensor_schema(MySensor).extend(cv.polling_component_schema("60s"))
          async def to_code(config):
              var = await sensor.new_sensor(config)
              await cg.register_component(var, config)
          ```

        - **二进制传感器（Binary Sensor）：**
          ```python
          from esphome.components import binary_sensor
          CONFIG_SCHEMA = binary_sensor.binary_sensor_schema().extend({ ... })
          async def to_code(config):
              var = await binary_sensor.new_binary_sensor(config)
          ```

        - **开关（Switch）：**
          ```python
          from esphome.components import switch
          CONFIG_SCHEMA = switch.switch_schema().extend({ ... })
          async def to_code(config):
              var = await switch.new_switch(config)
          ```

*   **配置验证：**
    *   **常用验证器：** `cv.int_`、`cv.float_`、`cv.string`、`cv.boolean`、`cv.int_range(min=0, max=100)`、`cv.positive_int`、`cv.percentage`。
    *   **复杂验证：** `cv.All(cv.string, cv.Length(min=1, max=50))`、`cv.Any(cv.int_, cv.string)`。
    *   **平台特定：** `cv.only_on(["esp32", "esp8266"])`、`esp32.only_on_variant(...)`、`cv.only_on_esp32`、`cv.only_on_esp8266`、`cv.only_on_rp2040`。
    *   **框架特定：** `cv.only_with_framework(...)`、`cv.only_with_arduino`、`cv.only_with_esp_idf`。
    *   **Schema 扩展：**
        ```python
        CONFIG_SCHEMA = cv.Schema({ ... })
         .extend(cv.COMPONENT_SCHEMA)
         .extend(uart.UART_DEVICE_SCHEMA)
         .extend(i2c.i2c_device_schema(0x48))
         .extend(spi.spi_device_schema(cs_pin_required=True))
        ```

## 5. 关键文件与入口点

*   **主入口点：** `esphome/__main__.py` 是 ESPHome 命令行界面的主入口点。
*   **配置：**
    *   `pyproject.toml`：定义 Python 项目元数据和依赖。
    *   `platformio.ini`：为不同微控制器配置 PlatformIO 构建环境。
    *   `.pre-commit-config.yaml`：配置用于代码检查和格式化的 pre-commit 钩子。
*   **CI/CD 流水线：** 定义于 `.github/workflows`。
*   **静态分析与开发：**
    *   `esphome/core/defines.h`：一个包含所有 `#define` 指令的综合头文件，这些指令可由组件通过 Python 中的 `cg.add_define()` 添加。该文件仅用于开发、静态分析工具和 CI 测试——不在运行时编译中使用。在开发添加新 define 的组件时，必须将它们加入此文件，以确保正确的 IDE 支持和静态分析覆盖。该文件包含功能标志、构建配置以及平台特定的 define，帮助静态分析器在无需针对特定平台编译的情况下理解完整代码库。

## 6. 开发与测试工作流

*   **本地开发环境：** 使用提供的 Docker 容器，或创建 Python 虚拟环境并从 `requirements_dev.txt` 安装依赖。
*   **运行命令：** 使用 `script/run-in-env.py` 脚本在项目虚拟环境中执行命令。例如，运行代码检查：`python3 script/run-in-env.py pre-commit run`。
*   **测试：**
    *   **Python：** 使用 `pytest` 运行单元测试。
    *   **C++：** 使用 `clang-tidy` 进行静态分析。
    *   **组件测试：** 基于 YAML 的编译测试位于 `tests/`，结构如下：
        ```
        tests/
        ├── test_build_components/ # 基础测试配置
        └── components/[component]/ # 组件特定测试
        ```
        使用 `script/test_build_components` 运行。使用 `-c <component>` 测试特定组件，使用 `-t <target>` 指定特定平台。
    *   **整体测试所有组件：** 要验证所有组件能够一起测试而不存在 ID 冲突或配置问题，使用：
        ```bash
        ./script/test_component_grouping.py -e config --all
        ```
        这会在单次构建中测试所有组件，以捕获单独测试时可能不会出现的冲突。使用 `-e config` 进行快速配置验证，或使用 `-e compile` 进行完整编译测试。
*   **调试与故障排除：**
    *   **调试工具：**
        - `esphome config <file>.yaml` 验证配置。
        - `esphome compile <file>.yaml` 编译而不上传。
        - 通过仪表盘查看实时日志。
        - 使用组件特定的调试日志。
    *   **常见问题：**
        - **导入错误**：检查组件依赖和 `PYTHONPATH`。
        - **验证错误**：审查配置 Schema 定义。
        - **构建错误**：检查平台兼容性和库版本。
        - **运行时错误**：审查生成的 C++ 代码和组件逻辑。

## 7. AI 协作的具体说明

*   **贡献工作流（Pull Request 流程）：**
    1.  **Fork 与分支：** 基于 `dev` 分支创建新分支（始终使用 `git checkout -b <branch-name> dev`，以确保从 `dev` 而非当前检出的分支创建）。
    2.  **进行修改：** 遵守所有编码规范和模式。
    3.  **测试：** 为所有支持的平台创建组件测试，并在本地运行完整测试套件。
    4.  **代码检查：** 运行 `pre-commit` 确保代码合规。
    5.  **提交：** 提交更改。提交消息格式没有严格要求。
    6.  **Pull Request：** 向 `dev` 分支提交 PR。PR 标题应包含所处理组件的前缀（例如，`[display] Fix bug`、`[abc123] Add new component`）。按需更新文档、示例并添加 `CODEOWNERS` 条目。Pull Request 应始终使用 `.github/PULL_REQUEST_TEMPLATE.md` 模板——完整填写所有部分，不删除模板的任何内容。

*   **文档贡献：**
    *   文档托管在独立的 `esphome/esphome-docs` 仓库中。
    *   贡献工作流与代码库相同。
    *   编辑组件文档页面时，也需更新对应的组件索引页面，以确保两个页面保持同步。

*   **最佳实践：**
    *   **组件开发：** 保持最小依赖，提供清晰的错误信息，编写全面的文档字符串和测试。
    *   **代码生成：** 生成最小且高效的 C++ 代码。彻底验证所有用户输入。支持多种平台变体。
    *   **配置设计：** 以简洁为目标，设置合理的默认值，同时允许高级自定义。
    *   **嵌入式系统优化：** ESPHome 面向资源受限的微控制器。请注意 flash 大小和 RAM 用量。

        **为什么堆内存分配很重要：**

        ESP 设备持续运行数月，堆内存由 Wi-Fi、BLE、LWIP 和应用代码共享，空间有限。随着时间推移，不同大小的反复分配会导致堆碎片化。即使总空闲堆仍然较大，当最大连续块缩小时，分配失败就会发生。我们曾见过由此引起的设备崩溃。

        **除非绝对不可避免，否则应避免在 `setup()` 之后进行堆分配。** 每次分配/释放循环都会加剧碎片化。ESPHome 将运行时堆分配视为长期可靠性问题，而非性能问题。隐藏分配的辅助工具（`std::string`、`std::to_string`、返回字符串的辅助函数）正在被弃用，并被基于缓冲区和视图的 API 所替代。

        **STL 容器使用指南：**

        ESPHome 运行于资源有限的嵌入式系统上。请谨慎选择容器：

        1. **编译时已知大小：** 当大小在编译时已知时，使用 `std::array` 替代 `std::vector`。
           ```cpp
           // 错误 - 生成 STL 重新分配代码
           std::vector<int> values;

           // 正确 - 无动态分配
           std::array<int, MAX_VALUES> values;
           ```
           使用 `cg.add_define("MAX_VALUES", count)` 从 Python 配置中设置大小。

           **对于字节缓冲区：** 除非缓冲区需要增长，否则避免使用 `std::vector<uint8_t>`，改用 `std::unique_ptr<uint8_t[]>`。

           > **注意：** `std::unique_ptr<uint8_t[]>` **不**提供像 `std::vector<uint8_t>` 那样的边界检查或迭代器支持。仅在不需要这些特性且希望最小化开销时使用。

           ```cpp
           // 错误 - 简单字节缓冲区的 STL 开销
           std::vector<uint8_t> buffer;
           buffer.resize(256);

           // 正确 - 最小开销，单次分配
           std::unique_ptr<uint8_t[]> buffer = std::make_unique<uint8_t[]>(256);
           // 或者如果大小是常量：
           std::array<uint8_t, 256> buffer;
           ```

        2. **编译时已知固定大小且需要类 vector API：** 使用 `esphome/core/helpers.h` 中的 `StaticVector`，获得编译时固定大小和 `push_back()` 接口（无动态分配）。
           ```cpp
           // 错误 - 生成 STL 重新分配代码（_M_realloc_insert）
           std::vector<ServiceRecord> services;
           services.reserve(5);  // 仍包含重新分配机制

           // 正确 - 编译时固定大小，无动态分配
           StaticVector<ServiceRecord, MAX_SERVICES> services;
           services.push_back(record1);
           ```
           使用 `cg.add_define("MAX_SERVICES", count)` 从 Python 配置中设置大小。
           类似 `std::array` 但提供类 vector API（`push_back()`、`size()`），且无 STL 重新分配代码。

        3. **运行时已知大小：** 当大小仅在运行时初始化时已知，使用 `esphome/core/helpers.h` 中的 `FixedVector`。
           ```cpp
           // 错误 - 生成 STL 重新分配代码（_M_realloc_insert）
           std::vector<TxtRecord> txt_records;
           txt_records.reserve(5);  // 仍包含重新分配机制

           // 正确 - 运行时大小，单次分配，无重新分配机制
           FixedVector<TxtRecord> txt_records;
           txt_records.init(record_count);  // 在运行时以精确大小初始化
           ```
           **优势：**
           - 消除 `_M_realloc_insert`、`_M_default_append` 模板实例化（每个实例节省 200-500 字节）
           - 单次分配，无需上限
           - 无重新分配开销
           - 使用 `[(fixed_vector) = true]` 选项时与 protobuf 代码生成兼容

        4. **小型数据集（1-16 个元素）：** 使用带简单结构体的 `std::vector` 或 `std::array`，而非 `std::map`/`std::set`/`std::unordered_map`。
           ```cpp
           // 错误 - 红黑树/哈希表超过 2KB 的开销
           std::map<std::string, int> small_lookup;
           std::unordered_map<int, std::string> tiny_map;

           // 正确 - 带线性搜索的简单结构体（std::vector 即可）
           struct LookupEntry {
             const char *key;
             int value;
           };
           std::vector<LookupEntry> small_lookup = {
             {"key1", 10},
             {"key2", 20},
             {"key3", 30},
           };
           // 或者如果大小是编译时常量，使用 std::array：
           // std::array<LookupEntry, 3> small_lookup = {{ ... }};
           ```
           对于小型数据集（1-16 个元素），线性搜索通常比哈希/树的开销更快，但这取决于查找频率和访问模式。对于热点代码路径中的频繁查找，即使是小型数据集，O(1) 与 O(n) 的复杂度差异仍可能存在影响。`std::vector` 配合简单结构体通常没问题——应避免的是重量级容器（`map`、`set`、`unordered_map`），除非性能分析表明有必要使用它们处理小型数据集。

        5. **避免 `std::deque`：** 无论元素大小如何，它都以 512 字节块为单位分配，立即保证至少 512 字节的 RAM 使用。这是内存受限设备崩溃的主要来源之一。

        6. **检测方法：** 在编译器输出中查找以下模式：
           - 带有 STL 符号（vector、map、set）的大型代码段
           - 符号名称中的 `alloc`、`realloc`、`dealloc`
           - `_M_realloc_insert`、`_M_default_append`（vector 重新分配）
           - 红黑树代码（`rb_tree`、`_Rb_tree`）
           - 哈希表基础设施（`unordered_map`、`hash`）

        **优先优化以下内容：**
        - 核心组件（API、网络、日志）
        - 广泛使用的组件（mdns、wifi、ble）
        - 引发 flash 大小投诉的组件

        注意：无论组件类型如何，避免在 `setup()` 之后进行堆分配始终是必须的。上述优先级是关于容器优化工作的投入（例如，从 `std::vector` 迁移到 `StaticVector`）。

    *   **状态管理：** 使用 `CORE.data` 存储配置生成期间需要持久化的组件状态。避免使用模块级可变全局变量。

        **反面模式（模块级全局变量）：**
        ```python
        # 不要这样做 - 状态会在多次编译运行之间持久化
        _component_state = []
        _use_feature = None

        def enable_feature():
            global _use_feature
            _use_feature = True
        ```

        **反面模式（扁平键名）：**
        ```python
        # 不要这样做 - 键名应在组件域下命名空间化
        MY_FEATURE_KEY = "my_component_feature"
        CORE.data[MY_FEATURE_KEY] = True
        ```

        **正确模式（dataclass）：**
        ```python
        from dataclasses import dataclass, field
        from esphome.core import CORE

        DOMAIN = "my_component"

        @dataclass
        class MyComponentData:
            feature_enabled: bool = False
            item_count: int = 0
            items: list[str] = field(default_factory=list)

        def _get_data() -> MyComponentData:
            if DOMAIN not in CORE.data:
                CORE.data[DOMAIN] = MyComponentData()
            return CORE.data[DOMAIN]

        def request_feature() -> None:
            _get_data().feature_enabled = True

        def add_item(item: str) -> None:
            _get_data().items.append(item)
        ```

        如需真实示例，可在代码库中搜索使用 `@dataclass` 与 `CORE.data` 的组件。注意：某些组件可能使用 `TypedDict` 进行基于字典的存储；根据需求，两种模式都是可接受的。

        **为什么这很重要：**
        - 如果仪表盘不进行 fork/exec，模块级全局变量会在多次编译运行之间持久化
        - `CORE.data` 在运行之间自动清除
        - 在 `DOMAIN` 下命名空间化可防止组件间的键名冲突
        - `@dataclass` 提供类型安全和更清晰的属性访问

*   **安全性：** 在修改 API、Web 服务器或任何其他网络相关代码时，请注意安全问题。不要硬编码密钥或密码。

*   **依赖与构建系统集成：**
    *   **Python：** 添加新的 Python 依赖时，将其加入相应的 `requirements*.txt` 文件和 `pyproject.toml`。
    *   **C++ / PlatformIO：** 添加新的 C++ 依赖时，将其加入 `platformio.ini` 并使用 `cg.add_library`。
    *   **构建标志：** 使用 `cg.add_build_flag(...)` 添加编译器标志。

## 8. 公共 API 与破坏性变更

*   **公共 C++ API：**
    *   **组件**：仅 [esphome.io](https://esphome.io) 上记录的功能属于公共 API。未记录的 `public` 成员为内部成员。
    *   **核心/基类**（`esphome/core/`、`Component`、`Sensor` 等）：所有 `public` 成员均为公共 API。
    *   **带全局访问器的组件**（`global_api_server` 等）：所有 `public` 成员均为公共 API（配置 setter 除外）。

*   **公共 Python API：**
    *   [esphome.io](https://esphome.io) 上所有已记录的配置选项均为公共 API。
    *   现有核心组件主动使用的 `esphome/core/` 中的 Python 代码被视为稳定 API。
    *   其他 Python 代码为内部代码，除非明确记录为供外部组件使用。

*   **破坏性变更政策：**
    *   尽可能提供 **6 个月的弃用窗口期**
    *   以下情况允许直接破坏：签名变更、深度重构、资源限制
    *   必须在 PR 描述中记录迁移路径（生成发布说明）
    *   核心/基类变更或重大架构变更需要撰写博客文章
    *   完整细节：https://developers.esphome.io/contributing/code/#public-api-and-breaking-changes

*   **破坏性变更检查清单：**
    - [ ] 有明确的理由（RAM/flash 节省、架构改进）
    - [ ] 已探索非破坏性替代方案
    - [ ] 尽可能添加弃用警告（C++ 使用 `ESPDEPRECATED` 宏）
    - [ ] 在 PR 描述中以前后对比示例记录迁移路径
    - [ ] 更新所有内部用法和 esphome-docs
    - [ ] 在弃用期间测试向后兼容性

*   **弃用模式（C++）：**
    ```cpp
    // 在 2026.6.0 之前移除
    ESPDEPRECATED("Use new_method() instead. Removed in 2026.6.0", "2025.12.0")
    void old_method() { this->new_method(); }
    ```

*   **弃用模式（Python）：**
    ```python
    # 在 2026.6.0 之前移除
    if CONF_OLD_KEY in config:
        _LOGGER.warning(f"'{CONF_OLD_KEY}' deprecated, use '{CONF_NEW_KEY}'. Removed in 2026.6.0")
        config[CONF_NEW_KEY] = config.pop(CONF_OLD_KEY)  # 自动迁移
    ```

