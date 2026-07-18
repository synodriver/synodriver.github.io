---
date: '2026-07-12T02:40:00+08:00'
draft: false
title: 'Modbus 协议详解与 ESPHome 驱动开发实战'
author: 'synodriver'
tags: ["嵌入式", "通信协议", "modbus", "esphome", "c++", "iot"]
---
# Modbus 协议详解与 ESPHome 驱动开发实战

Modbus 是工业自动化和物联网领域生命力最顽强的通信协议之一。它诞生于 1979 年，由 Modicon（今施耐德电气）为 PLC 通信设计，如今已是一种事实标准——几乎找不到一台不支持 Modbus 的工业仪表。本文从一个嵌入式工程师的视角，从协议本身到 ESPHome 的实现架构，再到如何亲手写一个自定义 Modbus 传感器驱动，做一次完整的梳理。

## 一、Modbus 协议总览

### 1.1 它是什么

Modbus 是一种**主从（client/server）、请求-应答式**的应用层协议。一条总线上有一个主站（client/master）和最多 247 个从站（server/slave），每个从站有一个 1~247 的地址。所有通信都由主站发起，从站被动应答；从站之间不能直接通信。

> 协议规范从 V1.1b3 起官方术语改为 client/server，但业内 master/slave 的说法仍根深蒂固。本文统一使用 client/server，必要时附注旧称。

Modbus 本身只定义了应用层协议（PDU），它可以承载在多种链路上：

| 变体       | 承载链路       | 帧头/地址 | 特点                                   |
| ---------- | -------------- | --------- | -------------------------------------- |
| Modbus RTU | 串口 RS-485/232 | RTU 地址 1 字节 | 二进制帧，CRC-16 校验，最常用          |
| Modbus ASCII | 串口 RS-485/232 | ASCII 地址 2 字符 | ASCII 字符，LRC 校验，可读但效率低     |
| Modbus TCP | 以太网 TCP/IP  | 7 字节 MBAP 头（含 Unit ID） | ADU 不带 CRC，依赖 TCP 校验，标准端口 502 |
| Modbus RTU over TCP | TCP/IP | RTU 地址 1 字节 | 非标准封装，通常把 RTU ADU 原样放进 TCP |

本文聚焦**最常用的 Modbus RTU**。

### 1.2 RTU 帧格式

一个 RTU 帧由「地址 + PDU + CRC」三段组成：

```
┌──────────┬──────────────────────────────┬─────────┐
│ Address  │           PDU                │   CRC   │
│ (1 byte) │ Function Code + Data         │ (2 byte)│
└──────────┴──────────────────────────────┴─────────┘
```

- **Address**：从站地址，1~247。`0x00` 为广播地址，所有从站执行但不应答。
- **PDU（Protocol Data Unit）**：功能码（1 字节）+ 数据。PDU 最大 253 字节，因此一个完整 RTU 帧最大 256 字节。
- **CRC-16**：对「地址 + PDU」做 CRC-16（Modbus 多项式 `0xA001`），**低字节在前、高字节在后**（小端序）。

**帧间定界**靠的是时间：RTU 规定帧间空闲至少 3.5 个字符时间。一帧发送完毕后，若总线空闲超过 3.5 个字符时间没有新字节，接收方就认为当前帧结束。按 11 bit/字符估算，9600bps 时约 4ms；串行链路规范在高于 19200bps 时通常使用固定的 1.75ms 帧间隔。

### 1.3 CRC-16 校验

Modbus RTU 使用的 CRC-16 算法：

- 初始值 `0xFFFF`
- 多项式 `0xA001`（即 `0x8005` 的位反转）
- 逐字节处理，先异或再循环右移 8 次

C 语言参考实现：

```c
uint16_t crc16(const uint8_t *data, size_t len) {
    uint16_t crc = 0xFFFF;
    for (size_t i = 0; i < len; i++) {
        crc ^= data[i];
        for (int j = 0; j < 8; j++) {
            if (crc & 0x0001)
                crc = (crc >> 1) ^ 0xA001;
            else
                crc >>= 1;
        }
    }
    return crc;  // 发送时先发低字节，再发高字节
}
```

### 1.4 数据模型

Modbus 把设备内部数据抽象为四张表，每张表有独立的地址空间（都是从 0 开始）：

| 数据类型           | 读写 | 单位    | 对应功能码           | 说明                 |
| ------------------ | ---- | ------- | -------------------- | -------------------- |
| 线圈 Coil           | R/W  | 1 bit   | 0x01, 0x05, 0x0F     | 开关量输出           |
| 离散输入 Discrete Input | R/O  | 1 bit   | 0x02                 | 开关量输入           |
| 保持寄存器 Holding Register | R/W  | 16 bit  | 0x03, 0x06, 0x10     | 模拟量参数/输出      |
| 输入寄存器 Input Register | R/O  | 16 bit  | 0x04                 | 模拟量测量值         |

注意：四种数据类型的地址空间互相独立，且 Modbus 协议地址从 0 开始，但很多厂商文档习惯用「1-based 寄存器号」（如「保持寄存器 40001」实际指地址 0）。文档里的 `40xxx` 前缀 4 表示保持寄存器，`3xxx` 表示输入寄存器，`0xxx` 表示线圈，`1xxx` 表示离散输入，减去基地址后才是协议地址。

## 二、常用功能码与报文示例

下面逐个列出常用功能码，以及 ESPHome 源码中枚举或解析器显式涉及的功能码；MEI（`0x2B`）等少见扩展只在速查表中标注。所有示例均以从站地址 `0x01` 为例，CRC 已用标准算法实际计算并附上，可直接用于抓包比对。

### 2.1 功能码 0x01 — 读线圈（Read Coils）

请求：起始地址（2B）+ 数量（2B，1~2000）

```
01 01 00 13 00 25 0C 14
```
含义：从站 01，读线圈，起始地址 0x0013，数量 0x0025（37 个）

应答：字节数（1B）+ 数据（按 bit 打包，LSB 在前）

```
01 01 05 CD 6B B2 0E 1B 44 EA
```
含义：5 个字节承载 37 个线圈状态，`0xCD` = `1100 1101`，bit0 对应线圈 0x0013=ON。

### 2.2 功能码 0x02 — 读离散输入（Read Discrete Inputs）

报文结构与 0x01 完全一致，只是功能码不同且目标为只读输入。

```
请求: 01 02 00 C4 00 16 B8 39    # 读离散输入，起始 0x00C4，数量 0x0016（22 个）
应答: 01 02 03 A9 1B 00 A2 9E    # 3 字节数据
```

### 2.3 功能码 0x03 — 读保持寄存器（Read Holding Registers）

请求：起始地址（2B）+ 数量（2B，1~125）

```
01 03 00 6B 00 03 74 17
```
含义：读保持寄存器，起始 0x006B，数量 3 个

应答：字节数（1B）+ 数据（每个寄存器 2B，大端序）

```
01 03 06 02 2B 00 00 00 64 05 7A
```
含义：6 字节数据 = 3 个寄存器值：`0x022B`、`0x0000`、`0x0064`

### 2.4 功能码 0x04 — 读输入寄存器（Read Input Registers）

报文结构与 0x03 完全一致，目标为只读输入寄存器。

```
请求: 01 04 00 08 00 01 B0 08    # 读输入寄存器，起始 0x0008，数量 1
应答: 01 04 02 00 0A 39 37       # 2 字节数据，值为 0x000A = 10
```

### 2.5 功能码 0x05 — 写单个线圈（Write Single Coil）

请求：输出地址（2B）+ 值（2B，`0xFF00`=ON，`0x0000`=OFF）

```
01 05 00 AC FF 00 4C 1B
```
含义：写线圈 0x00AC 为 ON

应答：原样回显请求（用于确认）

```
01 05 00 AC FF 00 4C 1B
```

### 2.6 功能码 0x06 — 写单个寄存器（Write Single Register）

请求：寄存器地址（2B）+ 值（2B）

```
01 06 00 01 00 03 98 0B
```
含义：写寄存器 0x0001 = 0x0003

应答：原样回显请求

```
01 06 00 01 00 03 98 0B
```

### 2.7 功能码 0x07 — 读异常状态（Read Exception Status）

属于串行链路诊断功能，多数实现不支持。请求只有地址+功能码+CRC，应答返回 1 字节状态。ESPHome 未实现。

### 2.8 功能码 0x08 — 诊断（Diagnostics）

串行链路诊断子功能（如回环测试 0x0000）。ESPHome 未实现。

### 2.9 功能码 0x0B / 0x0C — 通信事件计数器/日志（Get Comm Event Counter / Log）

用于统计通信事件，ESPHome 未实现。

### 2.10 功能码 0x0F — 写多个线圈（Write Multiple Coils）

请求：起始地址（2B）+ 数量（2B）+ 字节数（1B）+ 数据（bit 打包）

```
01 0F 00 13 00 0A 02 CD 01 72 CB
```
含义：起始 0x0013，写 10 个线圈，2 字节数据 `0xCD 0x01`

应答：起始地址（2B）+ 数量（2B）

```
01 0F 00 13 00 0A 24 09
```
含义：回显起始地址 0x0013 和数量 0x000A，确认写入成功

### 2.11 功能码 0x10 — 写多个寄存器（Write Multiple Registers）

请求：起始地址（2B）+ 数量（2B）+ 字节数（1B）+ 数据（每个寄存器 2B）

```
01 10 00 01 00 02 04 00 0A 01 02 92 30
```
含义：起始 0x0001，写 2 个寄存器，4 字节数据：`0x000A`、`0x0102`

应答：起始地址（2B）+ 数量（2B）

```
01 10 00 01 00 02 10 08
```

### 2.12 功能码 0x11 — 报告从站 ID（Report Server ID）

返回设备类型和运行状态。ESPHome 未实现。

### 2.13 功能码 0x14 / 0x15 — 文件记录读写（Read/Write File Record）

面向文件系统的子功能，结构复杂（含子请求）。ESPHome 解析器能识别其长度但未实现业务逻辑。

### 2.14 功能码 0x16 — 屏蔽写寄存器（Mask Write Register）

请求：地址（2B）+ AND 掩码（2B）+ OR 掩码（2B），结果 = `(当前值 AND AND_mask) OR (OR_mask AND NOT AND_mask)`。ESPHome 未实现。

### 2.15 功能码 0x17 — 读写多个寄存器（Read/Write Multiple Registers）

在一个事务中先写后读，适合「写命令后立即读结果」的场景。请求含读起始地址、读数量、写起始地址、写数量、字节数、写数据。ESPHome 未实现。

### 2.16 功能码 0x18 — 读 FIFO 队列（Read FIFO Queue）

读 FIFO 寄存器队列。ESPHome 未实现。

### 2.17 用户自定义功能码

规范保留了两个区间供厂商扩展：

- `0x41 ~ 0x48`（65~72）
- `0x64 ~ 0x6E`（100~110）

自定义功能码的帧长度无法按规范推断，接收端只能靠 CRC 试探。ESPHome 对此有专门处理（见后文）。

### 2.18 异常响应

当从站无法处理请求时，返回异常响应：把功能码最高位置 1（即 `功能码 | 0x80`），后跟 1 字节异常码。

```
01 83 02 C0 F1
```
含义：从站 01，对 0x03 的异常响应（`0x83`），异常码 `0x02`（非法数据地址）

全部异常码：

| 异常码 | 名称（Hex / 名称）                         | 含义                         |
| ------ | ------------------------------------------ | ---------------------------- |
| 0x01   | ILLEGAL_FUNCTION                           | 不支持该功能码               |
| 0x02   | ILLEGAL_DATA_ADDRESS                       | 地址越界                     |
| 0x03   | ILLEGAL_DATA_VALUE                         | 数据值非法                   |
| 0x04   | SERVICE_DEVICE_FAILURE                     | 设备内部故障                 |
| 0x05   | ACKNOWLEDGE                               | 已接收，需要长时间处理       |
| 0x06   | SERVER_DEVICE_BUSY                         | 设备忙                       |
| 0x08   | MEMORY_PARITY_ERROR                        | 存储奇偶校验错               |
| 0x0A   | GATEWAY_PATH_UNAVAILABLE                   | 网关路径不可用               |
| 0x0B   | GATEWAY_TARGET_DEVICE_FAILED_TO_RESPOND    | 网关目标设备无响应           |

### 2.19 功能码速查表

| 功能码 | 名称                      | 操作对象       | 读写 | 最大数量 |
| ------ | ------------------------- | -------------- | ---- | -------- |
| 0x01   | Read Coils                | 线圈           | R    | 2000     |
| 0x02   | Read Discrete Inputs      | 离散输入       | R    | 2000     |
| 0x03   | Read Holding Registers    | 保持寄存器     | R    | 125      |
| 0x04   | Read Input Registers      | 输入寄存器     | R    | 125      |
| 0x05   | Write Single Coil         | 线圈           | W    | 1        |
| 0x06   | Write Single Register     | 保持寄存器     | W    | 1        |
| 0x0F   | Write Multiple Coils      | 线圈           | W    | 1968     |
| 0x10   | Write Multiple Registers  | 保持寄存器     | W    | 123      |
| 0x07~0x0C, 0x11, 0x14~0x18, 0x2B | 其它诊断/文件/MEI 类 | 各种     | -    | -        |

## 三、ESPHome 中的 Modbus 实现架构

ESPHome 将 Modbus 拆成了**两层**：底层 `modbus` 组件负责收发原始帧，上层 `modbus_controller` 组件负责寄存器映射、命令队列和轮询调度。这种分层让「写一个新设备驱动」既可以直接基于底层（轻量、自由），也可以挂到上层（省心、自动调度）。

### 3.1 整体分层

```
┌─────────────────────────────────────────────────────┐
│  用户 YAML 配置                                      │
│  (modbus + modbus_controller + 各传感器组件)         │
└───────────────┬─────────────────────────────────────┘
                │  Python codegen 生成 C++
┌───────────────▼─────────────────────────────────────┐
│  上层: modbus_controller                            │
│  - SensorItem / RegisterRange / ModbusCommandItem   │
│  - 自动合并相邻寄存器、命令队列、重试、离线检测      │
│  - 派生: sensor / binary_sensor / switch / number...│
└───────────────┬─────────────────────────────────────┘
                │  on_modbus_data / send()
┌───────────────▼─────────────────────────────────────┐
│  底层: modbus                                       │
│  - ModbusClientHub / ModbusServerHub                │
│  - UART 收发、CRC 校验、帧定界、TX 队列、流控       │
│  - ModbusClientDevice / ModbusServerDevice (设备基类)│
└───────────────┬─────────────────────────────────────┘
                │  read_array / write_array
┌───────────────▼─────────────────────────────────────┐
│  uart 组件 (硬件串口)                               │
└─────────────────────────────────────────────────────┘
```

### 3.2 底层 modbus 组件

源码位于 `esphome/components/modbus/`，核心是 `Modbus` 基类及其两个子类 `ModbusClientHub`、`ModbusServerHub`。

#### 3.2.1 类继承关系

```cpp
// modbus.h
class Modbus : public uart::UARTDevice, public Component { ... };

class ModbusClientHub : public Modbus { ... };   // 主站
class ModbusServerHub : public Modbus { ... };   // 从站

class ModbusClientDevice { ... };  // 挂在主站下的从设备
class ModbusServerDevice { ... };  // 挂在从站下的本地设备
```

`Modbus` 继承自 `UARTDevice`，直接持有串口；`ModbusClientDevice` 是「逻辑从设备」的基类，持有 `ModbusClientHub*` 指针和自己的从站地址。

#### 3.2.2 帧的表示与发送

`ModbusFrame` 结构体负责拼装一帧：拷贝地址+PDU，末尾追加 CRC（低字节在前）。

```cpp
// modbus.h
struct ModbusFrame {
  std::unique_ptr<uint8_t[]> data;
  uint16_t size;
  ModbusFrame(uint8_t address, const uint8_t *pdu, uint16_t pdu_len)
      : data(std::make_unique<uint8_t[]>(pdu_len + 3)), size(pdu_len + 3) {
    data[0] = address;
    memcpy(data.get() + 1, pdu, pdu_len);
    auto crc = crc16(data.get(), pdu_len + 1);
    data[pdu_len + 1] = crc >> 0;   // CRC 低字节
    data[pdu_len + 2] = crc >> 8;   // CRC 高字节
  }
};
```

发送路径（client 侧）：
1. `ModbusClientDevice::send()` → 构造 PDU
2. `ModbusClientHub::queue_raw_()` → 封装成 `ModbusDeviceCommand` 压入 `tx_buffer_`（`std::deque`，FIFO）
3. `loop()` 中 `send_next_frame_()` 取队首 → `send_frame_()` 实际写串口，并把命令移入 `waiting_for_response_`

#### 3.2.3 接收与帧定界

`Modbus::loop()` 每轮做两件事：`receive_bytes_()` 把串口数据读入 `rx_buffer_`，`parse_modbus_frames()` 解析帧。

帧定界依靠**功能码推断长度 + 超时**两种手段：

- `helpers::server_frame_length()` / `client_frame_length()` 根据功能码计算期望帧长。例如 0x03 响应 = `5 + frame[2]`（地址+功能码+字节数+数据+CRC）。
- `timeout_()` 以 3.5 字符时间为基础；在 UART 接收 FIFO 阈值可能导致长响应分块到达时，还会使用 `long_rx_buffer_delay_ms_`（默认 50ms，或按 FIFO 阈值/波特率估算）避免过早清包。

对**自定义功能码**（0x41~0x48、0x64~0x6E），长度无法预知，于是 `find_custom_frame_end_()` 从最小长度起逐个尝试 CRC，CRC 为 0 即找到帧尾：

```cpp
// modbus.cpp
uint16_t Modbus::find_custom_frame_end_(uint16_t min_length) const {
  const uint8_t *raw = &this->rx_buffer_[0];
  const size_t size = this->rx_buffer_.size();
  for (uint16_t len = min_length; len <= std::min(size, size_t(MAX_FRAME_SIZE)); len++) {
    if (crc16(raw, len) == 0)   // CRC 校验通过(余数为0)
      return len;
  }
  return 0;
}
```

#### 3.2.4 响应分发

client 收到一帧后，`process_modbus_server_frame()` 校验地址和功能码是否与 `waiting_for_response_` 匹配：

- 匹配且正常 → 调 `device->on_modbus_data(data)`
- 匹配但是异常帧（功能码最高位为 1）→ 调 `device->on_modbus_error()`
- 不匹配 → 告警，调用 `on_modbus_no_response()`，并把当前等待标记为 `interrupted`

```cpp
// modbus.cpp
if (helpers::is_function_code_exception(function_code)) {
  uint8_t exception = len > 0 ? data[0] : 0;
  if (device) device->on_modbus_error(function_code & FUNCTION_CODE_MASK, exception);
} else if (device) {
  device->on_modbus_data(std::vector<uint8_t>(data, data + len));
}
```

#### 3.2.5 发送阻塞与流控

`Modbus::tx_blocked()` 在以下情况阻止发送，避免半双工冲突；client 侧的 `ModbusClientHub::tx_blocked()` 还会在 `waiting_for_response_` 存在时继续阻塞下一帧：

1. UART 接收缓冲区还有字节
2. `rx_buffer_` 非空
3. 距上次发送不足帧间隔时间
4. 距最后接收字节不足帧间隔时间（防止还有数据在路上）
5. client 已发出请求但还在等待响应

若有硬件流控引脚（RS-485 的 DE/RE），发送时拉高、发完 flush 后拉低。

#### 3.2.6 数据类型与字节序

`modbus_helpers.h` 定义了 `SensorValueType` 枚举，覆盖 1/2/4/8 字节的有无符号整数及 FP32，并区分大小端字序（`U_DWORD` 为大端先高字，`U_DWORD_R` 为字反转先低字）：

```cpp
enum class SensorValueType : uint8_t {
  RAW = 0x00, U_WORD, U_DWORD, S_WORD, S_DWORD, BIT,
  U_DWORD_R, S_DWORD_R, U_QWORD, S_QWORD, U_QWORD_R, S_QWORD_R,
  FP32, FP32_R
};
```

`payload_to_number()` 负责把响应字节按指定类型解码成 `int64_t`；对 `FP32/FP32_R`，它返回 IEEE-754 原始 bit pattern，上层 `payload_to_float()` 再 `bit_cast<float>`。`number_to_payload()` 是其逆操作，`mask_and_shift_by_rightbit()` 支持「一个寄存器里打包多个值」的位掩码提取。

#### 3.2.7 YAML 配置入口

`modbus/__init__.py` 定义了 `modbus` 块的 schema，用 `cv.typed_schema` 按 `role` 分 client/server：

```yaml
# 配置示例
uart:
  - id: uart_bus
    baud_rate: 9600
    tx_pin: GPIO1
    rx_pin: GPIO3

modbus:
  role: client
  send_wait_time: 2000ms
  turnaround_time: 600ms
  flow_control_pin: GPIO2   # RS-485 DE 引脚，可选
  id: modbus1
```

### 3.3 上层 modbus_controller 组件

源码位于 `esphome/components/modbus_controller/`。它在底层之上提供了「寄存器自动映射 + 周期轮询」的能力，是大多数用户的选择。

#### 3.3.1 核心数据结构

- **`SensorItem`**：描述一个被监视的寄存器。含 `register_type`、`start_address`、`register_count`、`offset`、`bitmask`、`sensor_value_type`、`skip_updates` 等。是各平台（sensor/switch/...）条目的基类，子类实现 `parse_and_publish()`。
- **`RegisterRange`**：一段连续的寄存器读取范围，内含一组 `SensorItem`。轮询时按 range 发一个读命令，响应回来后分发给 range 内所有 sensor。
- **`ModbusCommandItem`**：一条待发送命令，有若干工厂方法（`create_read_command`、`create_write_single_command`、`create_custom_command` 等），携带 `on_data_func` 回调。

#### 3.3.2 寄存器范围合并

`create_register_ranges_()` 是 controller 最精巧的部分。它把所有 `SensorItem` 按 `(register_type, force_new_range, start_address, offset)` 排序，然后贪心合并相邻寄存器：

- **复用**：两个 sensor 落在同一寄存器 → 共享数据，仅改 offset
- **扩展**：地址恰好衔接 → 合成一个大 range 一次读出

这样 N 个分散的传感器可能只需 1~2 次读操作，大幅减少总线流量。`force_new_range` 可强制切断合并。

#### 3.3.3 命令队列与调度

`ModbusController` 继承 `PollingComponent`，周期性触发 `update()`：

```
update()  →  对每个 RegisterRange 调 update_range_()
                ├─ skip_updates 计数 >0 → 跳过本次
                └─ 否则 queue_command(create_read_command(...))
loop()    →  incoming_queue_ 非空 → process_modbus_data_() 分发
             否则 → send_next_command_() 发队首命令
```

命令队列（`command_queue_`）是 `std::list<unique_ptr<ModbusCommandItem>>`。发送策略：

- `command_throttle_` 控制两次发送的最小间隔
- `ready_for_immediate_send()` 确认底层 TX 缓冲空且未被阻塞
- `should_retry(max_cmd_retries)` 控制重试次数；超过则标记 `module_offline_` 并触发 offline 回调

#### 3.3.4 响应处理

底层 `on_modbus_data()` 被回调时，controller 把响应数据挂到当前队首命令上，移入 `incoming_queue_`；`loop()` 再从 `incoming_queue_` 取出，调 `on_data_func` → 最终 `on_register_data()` 找到对应 range，遍历其中所有 `SensorItem::parse_and_publish()`。

这种「先入队、loop 中处理」的设计避免了在底层回调里做耗时操作（解析、发布状态），保证收发时序不被拖累。

#### 3.3.5 派生平台

`modbus_controller/` 下有 `sensor/`、`binary_sensor/`、`switch/`、`number/`、`select/`、`output/`、`text_sensor/` 七个平台目录，各自实现对应 `SensorItem` 子类。用户在 YAML 里直接用这些平台，无需写 C++：

```yaml
modbus_controller:
  - id: mc1
    modbus_id: modbus1
    address: 0x01
    update_interval: 10s

sensor:
  - platform: modbus_controller
    modbus_controller_id: mc1
    address: 0x0000
    register_type: holding
    value_type: U_WORD
    name: "电压"
    unit_of_measurement: "V"
    accuracy_decimals: 1
    filters:
      - multiply: 0.1
```

## 四、实现一个自定义 Modbus 传感器驱动

`modbus_controller` 虽然通用，但有些设备（如 PZEM 系列电能表）协议固定、字段密集，用通用配置反而啰嗦。此时可以像 `pzemac` 那样写一个**专用组件**，直接继承 `ModbusClientDevice`，享受底层收发能力，同时把寄存器解析封装成开箱即用的传感器。

### 4.1 参考对象：pzemac

源码位于 `esphome/components/pzemac/`，只有 4 个文件：

| 文件        | 作用                                       |
| ----------- | ------------------------------------------ |
| `pzemac.h`  | 声明 `PZEMAC` 类、传感器 setter、复位动作  |
| `pzemac.cpp`| 实现 `update()` 发读命令、`on_modbus_data()` 解析 |
| `sensor.py` | Python 配置 schema + codegen              |
| `__init__.py`| 空，仅标记包                               |

### 4.2 PZEMAC 的实现剖析

#### 4.2.1 类定义

```cpp
// pzemac.h
class PZEMAC final : public PollingComponent, public modbus::ModbusClientDevice {
 public:
  void set_voltage_sensor(sensor::Sensor *s) { voltage_sensor_ = s; }
  // ... 其余 setter 省略
  void update() override;
  void on_modbus_data(const std::vector<uint8_t> &data) override;
  void dump_config() override;
 protected:
  sensor::Sensor *voltage_sensor_{nullptr};
  // ... 其余传感器指针
  void reset_energy_();
};
```

关键点：
- 多继承 `PollingComponent`（提供周期 `update()`）+ `ModbusClientDevice`（提供 `send()`/`on_modbus_data()`）
- `final` 防止被进一步继承
- 每个物理量对应一个 `sensor::Sensor*`，由 Python codegen 注入

#### 4.2.2 发起读取

`update()` 极其简洁——PZEM-014/016 协议固定读 10 个输入寄存器（地址 0 起）：

```cpp
// pzemac.cpp
static const uint8_t PZEM_CMD_READ_IN_REGISTERS = 0x04;
static const uint8_t PZEM_REGISTER_COUNT = 10;

void PZEMAC::update() {
  this->send(PZEM_CMD_READ_IN_REGISTERS, 0, PZEM_REGISTER_COUNT);
}
```

`send()` 来自 `ModbusClientDevice`，签名是 `(功能码, 起始地址, 数量)`，内部构造 PDU 并交给 hub 入队。底层会自动补地址、CRC、控制时序。

#### 4.2.3 解析响应

`on_modbus_data()` 收到的是「PDU 数据部分」（已剥除地址、功能码、字节数、CRC）。PZEM 响应含 20 字节 = 10 个寄存器，按大端序排列，部分字段跨 2 个寄存器（32 位）：

```cpp
// pzemac.cpp
void PZEMAC::on_modbus_data(const std::vector<uint8_t> &data) {
  if (data.size() < 20) { ESP_LOGW(TAG, "Invalid size for PZEM AC!"); return; }

  auto pzem_get_16bit = [&](size_t i) -> uint16_t {
    return (uint16_t(data[i + 0]) << 8) | (uint16_t(data[i + 1]) << 0);
  };
  auto pzem_get_32bit = [&](size_t i) -> uint32_t {
    return (uint32_t(pzem_get_16bit(i + 2)) << 16) | (uint32_t(pzem_get_16bit(i + 0)) << 0);
  };

  float voltage = pzem_get_16bit(0) / 10.0f;          // 0.1V
  float current = pzem_get_32bit(2) / 1000.0f;        // 0.001A，字反转
  float active_power = pzem_get_32bit(6) / 10.0f;     // 0.1W，字反转
  float active_energy = pzem_get_32bit(10);            // 1Wh
  float frequency = pzem_get_16bit(14) / 10.0f;        // 0.1Hz
  float power_factor = pzem_get_16bit(16) / 100.0f;    // 0.01

  if (this->voltage_sensor_) this->voltage_sensor_->publish_state(voltage);
  // ... 其余 publish
}
```

对应的真实报文（与源码注释一致）：

```
请求: 01 04 00 00 00 0A 70 0D
应答: 01 04 14 08 D1 00 6C 00 00 00 F4 00 00 00 26 00 00 01 F4 00 64 00 00 51 34
       ^^ ^^ ^^ \-------------- 20 字节数据 --------------------------/ \--CRC-/
       地址 功能码 字节数=0x14(20)
```

注意 PZEM 的 32 位值采用**字反转**（低字在前），这正是 `pzem_get_32bit` 里先取 `i+2` 再取 `i+0` 的原因——对应 ESPHome 的 `U_DWORD_R` 类型。

#### 4.2.4 自定义命令

复位能量功能用非标准功能码 `0x42`，走 `send_raw()` 发「地址 + PDU」（不含 CRC）的原始帧：

```cpp
void PZEMAC::reset_energy_() {
  std::vector<uint8_t> cmd;
  cmd.push_back(this->address_);
  cmd.push_back(PZEM_CMD_RESET_ENERGY);  // 0x42
  this->send_raw(cmd);
}
```

`send_raw()` 的 payload 第一字节当作地址，剩余作为 PDU，CRC 由底层补全。`0x42` 落在自定义区间 `0x41~0x48`，底层会用 CRC 试探法定界其响应。

#### 4.2.5 Python 配置层

`sensor.py` 做三件事：

1. 声明类继承关系和 schema
2. 注册 `reset_energy` automation action
3. `to_code()` 生成 C++ 实例并注入传感器

```python
# sensor.py
pzemac_ns = cg.esphome_ns.namespace("pzemac")
PZEMAC = pzemac_ns.class_("PZEMAC", cg.PollingComponent, modbus.ModbusClientDevice)
ResetEnergyAction = pzemac_ns.class_("ResetEnergyAction", automation.Action)

CONFIG_SCHEMA = (
    cv.Schema({ ... })
    .extend(cv.polling_component_schema("60s"))
    .extend(modbus.modbus_device_schema(0x01))   # 默认地址 0x01
)

async def to_code(config):
    var = cg.new_Pvariable(config[CONF_ID])
    await cg.register_component(var, config)
    await modbus.register_modbus_client_device(var, config)
    if CONF_VOLTAGE in config:
        sens = await sensor.new_sensor(config[CONF_VOLTAGE])
        cg.add(var.set_voltage_sensor(sens))
    # ...
```

`modbus.modbus_device_schema(0x01)` 自动加上 `modbus_id` 和 `address` 两个字段；`register_modbus_client_device` 把组件挂到对应 hub 并设置地址。

### 4.3 亲手写一个：以 SDM120 电表为例

假设我们要为一个支持 Modbus RTU 的 SDM120 单相电表写专用驱动。ESPHome 已经内置了更完整的 `sdm_meter` 组件；这里仍手写一个精简版，主要用于演示如何正确继承 `ModbusClientDevice`。它的输入寄存器布局（节选）如下：

| 地址 | 含义       | 类型       | 单位/缩放 |
| ---- | ---------- | ---------- | --------- |
| 0x0000 | 电压       | FP32 (2 reg) | V，×1   |
| 0x0006 | 电流       | FP32       | A，×1     |
| 0x000C | 有功功率   | FP32       | W，×1     |
| 0x0048 | 导入有功电能 | FP32     | kWh，×1   |

> 有些 SDM 文档还会列出 `0x0156` 的 total active energy。它离前面的瞬时量太远，若要读取它通常应单独发一条读命令，不能假装它和 `0x0000~0x000D` 在同一个小范围响应里。

#### 第 1 步：建目录结构

```
esphome/components/sdm120/
├── __init__.py      # 空
├── sdm120.h
├── sdm120.cpp
└── sensor.py
```

#### 第 2 步：写头文件 `sdm120.h`

```cpp
#pragma once

#include "esphome/core/component.h"
#include "esphome/components/sensor/sensor.h"
#include "esphome/components/modbus/modbus.h"

#include <vector>

namespace esphome::sdm120 {

class SDM120 final : public PollingComponent, public modbus::ModbusClientDevice {
 public:
  void set_voltage_sensor(sensor::Sensor *s) { voltage_sensor_ = s; }
  void set_current_sensor(sensor::Sensor *s) { current_sensor_ = s; }
  void set_power_sensor(sensor::Sensor *s) { power_sensor_ = s; }
  void set_energy_sensor(sensor::Sensor *s) { energy_sensor_ = s; }

  void update() override;
  void on_modbus_data(const std::vector<uint8_t> &data) override;
  void dump_config() override;

 protected:
  sensor::Sensor *voltage_sensor_{nullptr};
  sensor::Sensor *current_sensor_{nullptr};
  sensor::Sensor *power_sensor_{nullptr};
  sensor::Sensor *energy_sensor_{nullptr};
};

}  // namespace esphome::sdm120
```

#### 第 3 步：写实现 `sdm120.cpp`

```cpp
#include "sdm120.h"
#include "esphome/core/helpers.h"
#include "esphome/core/log.h"

namespace esphome::sdm120 {

static const char *const TAG = "sdm120";

// SDM120 用功能码 0x04 读输入寄存器
static const uint8_t CMD_READ_INPUT = 0x04;
// 从 0x0000 连续读到 0x0049，覆盖电压/电流/功率/导入有功电能
static const uint16_t REG_MAIN_START = 0x0000;
static const uint16_t REG_COUNT_MAIN = 0x004A;  // 74 个 16-bit registers
static const uint16_t REG_VOLTAGE = 0x0000;
static const uint16_t REG_CURRENT = 0x0006;
static const uint16_t REG_ACTIVE_POWER = 0x000C;
static const uint16_t REG_IMPORT_ACTIVE_ENERGY = 0x0048;

void SDM120::update() {
  this->send(CMD_READ_INPUT, REG_MAIN_START, REG_COUNT_MAIN);
}

void SDM120::on_modbus_data(const std::vector<uint8_t> &data) {
  if (data.size() < REG_COUNT_MAIN * 2) {
    ESP_LOGW(TAG, "Invalid frame size: %zu", data.size());
    return;
  }

  // SDM120 的 FP32 是标准 Modbus 大端（高字在前），对应 ESPHome 的 FP32
  auto get_fp32 = [&](uint16_t reg) -> float {
    size_t i = (reg - REG_MAIN_START) * 2;
    uint32_t raw = encode_uint32(data[i + 0], data[i + 1], data[i + 2], data[i + 3]);
    return bit_cast<float>(raw);
  };

  float voltage = get_fp32(REG_VOLTAGE);
  float current = get_fp32(REG_CURRENT);
  float power = get_fp32(REG_ACTIVE_POWER);
  float energy = get_fp32(REG_IMPORT_ACTIVE_ENERGY);

  ESP_LOGD(TAG, "V=%.1f I=%.3f P=%.1f E=%.2f", voltage, current, power, energy);

  if (this->voltage_sensor_) this->voltage_sensor_->publish_state(voltage);
  if (this->current_sensor_) this->current_sensor_->publish_state(current);
  if (this->power_sensor_) this->power_sensor_->publish_state(power);
  if (this->energy_sensor_) this->energy_sensor_->publish_state(energy);
}

void SDM120::dump_config() {
  ESP_LOGCONFIG(TAG, "SDM120:");
  ESP_LOGCONFIG(TAG, "  Address: 0x%02X", this->address_);
  LOG_SENSOR("", "Voltage", this->voltage_sensor_);
  LOG_SENSOR("", "Current", this->current_sensor_);
  LOG_SENSOR("", "Power", this->power_sensor_);
  LOG_SENSOR("", "Energy", this->energy_sensor_);
}

}  // namespace esphome::sdm120
```

> 注意 `this->address_` 是 `ModbusClientDevice` 的 protected 成员，可直接访问。

#### 第 4 步：写配置 `sensor.py`

```python
import esphome.codegen as cg
from esphome.components import modbus, sensor
import esphome.config_validation as cv
from esphome.const import (
    CONF_CURRENT, CONF_ENERGY, CONF_ID, CONF_POWER, CONF_VOLTAGE,
    DEVICE_CLASS_CURRENT, DEVICE_CLASS_ENERGY, DEVICE_CLASS_POWER,
    DEVICE_CLASS_VOLTAGE, STATE_CLASS_MEASUREMENT,
    STATE_CLASS_TOTAL_INCREASING, UNIT_AMPERE, UNIT_KILOWATT_HOURS,
    UNIT_VOLT, UNIT_WATT,
)

AUTO_LOAD = ["modbus"]

sdm120_ns = cg.esphome_ns.namespace("sdm120")
SDM120 = sdm120_ns.class_("SDM120", cg.PollingComponent, modbus.ModbusClientDevice)

CONFIG_SCHEMA = (
    cv.Schema({
        cv.GenerateID(): cv.declare_id(SDM120),
        cv.Optional(CONF_VOLTAGE): sensor.sensor_schema(
            unit_of_measurement=UNIT_VOLT, accuracy_decimals=1,
            device_class=DEVICE_CLASS_VOLTAGE, state_class=STATE_CLASS_MEASUREMENT),
        cv.Optional(CONF_CURRENT): sensor.sensor_schema(
            unit_of_measurement=UNIT_AMPERE, accuracy_decimals=3,
            device_class=DEVICE_CLASS_CURRENT, state_class=STATE_CLASS_MEASUREMENT),
        cv.Optional(CONF_POWER): sensor.sensor_schema(
            unit_of_measurement=UNIT_WATT, accuracy_decimals=1,
            device_class=DEVICE_CLASS_POWER, state_class=STATE_CLASS_MEASUREMENT),
        cv.Optional(CONF_ENERGY): sensor.sensor_schema(
            unit_of_measurement=UNIT_KILOWATT_HOURS, accuracy_decimals=2,
            device_class=DEVICE_CLASS_ENERGY, state_class=STATE_CLASS_TOTAL_INCREASING),
    })
    .extend(cv.polling_component_schema("60s"))
    .extend(modbus.modbus_device_schema(0x01))
)

def _final_validate(config):
    return modbus.final_validate_modbus_device("sdm120", role="client")(config)

FINAL_VALIDATE_SCHEMA = _final_validate

async def to_code(config):
    var = cg.new_Pvariable(config[CONF_ID])
    await cg.register_component(var, config)
    await modbus.register_modbus_client_device(var, config)
    if CONF_VOLTAGE in config:
        sens = await sensor.new_sensor(config[CONF_VOLTAGE])
        cg.add(var.set_voltage_sensor(sens))
    if CONF_CURRENT in config:
        sens = await sensor.new_sensor(config[CONF_CURRENT])
        cg.add(var.set_current_sensor(sens))
    if CONF_POWER in config:
        sens = await sensor.new_sensor(config[CONF_POWER])
        cg.add(var.set_power_sensor(sens))
    if CONF_ENERGY in config:
        sens = await sensor.new_sensor(config[CONF_ENERGY])
        cg.add(var.set_energy_sensor(sens))
```

#### 第 5 步：用户 YAML

```yaml
uart:
  - id: uart_bus
    baud_rate: 9600
    tx_pin: GPIO1
    rx_pin: GPIO3

modbus:
  role: client
  id: modbus1
  flow_control_pin: GPIO2

sensor:
  - platform: sdm120
    modbus_id: modbus1
    address: 0x01
    update_interval: 10s
    voltage:
      name: "SDM120 电压"
    current:
      name: "SDM120 电流"
    power:
      name: "SDM120 功率"
    energy:
      name: "SDM120 电能"
```

### 4.4 两种方案的选择

| 维度       | 专用组件（仿 pzemac）        | modbus_controller 通用配置   |
| ---------- | ---------------------------- | ---------------------------- |
| 开发成本   | 需写 C++ + Python            | 纯 YAML                      |
| 灵活性     | 高，可塞任意解析逻辑、动作   | 受限于预定义 value_type/filter |
| 字节序支持 | 自己处理，任意               | 内置 U_DWORD/FP32/_R 等      |
| 寄存器合并 | 需手动优化                   | 自动合并相邻寄存器           |
| 适用场景   | 协议固定、字段多、有特殊命令 | 寄存器零散、设备种类多       |

经验法则：**一两台固定型号设备 → 写专用组件；接一堆不同设备、寄存器零散 → 用 modbus_controller**。

### 4.5 调试技巧

1. **先抓包**：用 USB-RS485 + 串口调试助手手动发请求，确认设备能回、回的什么。Modbus Poll 这类工具也很好用。
2. **开 VERBOSE 日志**：ESPHome 的 modbus 组件在 `ESP_LOGV` 级别会打印每个收发的帧（`format_hex_pretty`），在 YAML 里加 `logger: level: VERY_VERBOSE` 即可看到 `Write: 01 04 00 00 00 0A 70 0D` 这类输出。
3. **CRC 对不上**：99% 是字节序问题。Modbus RTU 的多字节字段（地址、数量、寄存器值）都是**大端**，但 CRC 是**小端**（低字节在前）。
4. **响应地址/功能码不匹配**：检查 `modbus_controller` 的 `address` 是否和设备实际拨码一致；总线上是否有多个主站。
5. **自定义功能码无响应**：确认设备是否真支持该码；底层对自定义码靠 CRC 试探定界。若私有响应很长，优先检查波特率、UART FIFO 阈值和 VERY_VERBOSE 日志；`long_rx_buffer_delay_ms_` 是 ESPHome 内部按 UART 阈值估算或默认 50ms 的保护延时，不是常规 YAML 配置项。

## 五、总结

Modbus 之所以能活四十多年，靠的就是「简单到极致」——一个地址、一个功能码、一段数据、一个 CRC。它的全部功能码加起来不过二十来个，真正常用的只有 `0x03/0x04/0x06/0x10` 四个。理解了帧格式和数据模型，就掌握了 90%。

ESPHome 的 Modbus 实现把这层简单性封装得恰到好处：底层 `modbus` 组件处理了 RTU 帧定界、CRC、半双工流控、自定义功能码试探等所有脏活；上层 `modbus_controller` 提供了寄存器自动合并、命令队列、重试离线检测等调度能力；而 `pzemac` 这类专用组件则示范了如何用最小代码量（一个 `update()` + 一个 `on_modbus_data()`）把一台设备变成开箱即用的传感器。

写一个自己的 Modbus 驱动，本质上就是回答三个问题：**读哪些寄存器？怎么发命令？响应怎么解析？** 把这三个问题在 C++ 里对应到 `update()`、`send()`、`on_modbus_data()` 三个函数，剩下的交给 ESPHome 的组件框架即可。

---

*本文中所有报文示例的 CRC 均按 Modbus 标准 CRC-16（多项式 0xA001，初值 0xFFFF，低字节在前）实际计算，可直接用于抓包比对。*
