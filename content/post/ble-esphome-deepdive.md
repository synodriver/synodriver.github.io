---
date: '2026-07-16T22:10:00+08:00'
draft: false
title: 'BLE 协议、MicroPython aioble 与 ESPHome BLE 实现深度解析'
author: 'synodriver'
tags: ["ble", "bluetooth", "micropython", "esphome", "embedded", "iot", "cpp"]
---

# BLE 协议、MicroPython aioble 与 ESPHome BLE 实现深度解析

---

## 一、BLE（低功耗蓝牙）是什么

BLE（Bluetooth Low Energy，蓝牙 4.0 起引入，旧称 "Bluetooth Smart"）与经典蓝牙（BR/EDR）共存但相互独立。它面向低功耗、低数据量、间歇性通信场景：纽扣电池驱动一个温度计工作数年是典型用例。

### 1.1 核心架构分层

```
┌─────────────────────────────────────────┐
│ 应用层 (Application / GATT-based Profile)│
├─────────────────────────────────────────┤
│ Host：GAP / GATT / ATT / SMP / L2CAP     │  ← LE Host（控制器之上）
├─────────────────────────────────────────┤
│ HCI（Host-Controller Interface）          │
├─────────────────────────────────────────┤
│ Controller：LL / PHY                     │  ← Controller（蓝牙芯片内）
└─────────────────────────────────────────┘
```

- **PHY**：1Mbps GFSK（可选 2M、500k、125k 编码 PHY），与经典蓝牙物理层不兼容。
- **LL（链路层）**：定义 40 个信道（37 数据 + 3 广播），时分复用。状态机：Standby → Advertising → Scanning → Initiating → Connection。
- **HCI**：Host 与 Controller 间标准接口（UART/USB/SDIO），让两者物理解耦。
- **Host 层**
  - **L2CAP**：协议复用、分片、流控。
  - **ATT（Attribute Protocol）**：属性读写、发现、通知/指示的基础协议，运行在 LE-U 固定信道 CID `0x0004`。
  - **SMP/SM（Security Manager Protocol）**：配对（pairing）、绑定（bonding）、密钥分发（LE Legacy / Secure Connections）。
  - **GAP**：设备发现、连接管理、广播/扫描、安全模式。
  - **GATT**：基于 ATT 的服务/特征/描述符层级数据模型，是绝大多数 BLE 应用的数据面。

> 注意：SDP（Service Discovery Protocol）属于 BR/EDR Host；BLE-only 的服务发现通常通过 ATT/GATT 完成。

### 1.2 GAP：广播、扫描、连接

- **广播（Advertising）**：设备在 37/38/39 三个主广播信道上周期性发送广播包（adv），间隔从 20ms 到 10.24s。传统广播（Legacy Advertising）的 AdvData 最多 31 字节；主动扫描的 scan response 也最多 31 字节。BLE 5 起的 Extended Advertising 可通过辅助广播 PDU 承载更长数据，规范上 Host Advertising Data 总长度最高可到 1650 字节，但实际受控制器/协议栈支持限制。
- **扫描（Scanning）**：被动扫描只收 adv，主动扫描还会发 `SCAN_REQ` 拿 scan response。
- **连接（Initiating）**：Central/Initiator 收到可连接的 legacy advertising 后，可发送 `CONNECT_IND`（旧资料常称 `CONNECT_REQ`）建立连接；Extended Advertising 场景使用 `AUX_CONNECT_REQ` / `AUX_CONNECT_RSP`。Legacy 场景不是 Peripheral 回一个显式 ACK PDU 后才进入连接，而是双方根据连接请求中的参数切入后续连接事件。连接态常用 Central / Peripheral 描述两端角色；GAP 还定义 Broadcaster、Observer、Peripheral、Central 等角色。“Master/Slave”是旧术语，现代规范倾向使用 Central/Peripheral。

### 1.3 GATT 数据模型

```
Profile
└── Service (128-bit UUID)
    └── Characteristic (128-bit UUID)
        ├── Value (字节数组)
        └── Descriptor (128-bit UUID, 例如 CCCD 0x2902)
```

- **Service**：一组相关功能的集合。
- **Characteristic**：业务数据的载体，权限 `read` / `write` / `notify` / `indicate`。
- **CCCD（Client Characteristic Configuration Descriptor）**：UUID 固定为 `0x2902`。写入 `0x0001` 启用 Notify、`0x0002` 启用 Indicate，写 `0x0000` 关闭。**所有 notify/indicate 订阅都靠它**。

### 1.4 与经典蓝牙的关键差异

| 维度 | BLE | 经典蓝牙 |
|------|-----|---------|
| 建连速度 | 数毫秒 | 数秒 |
| 数据速率 | 1Mbps（PHY 可达 2M） | 1-3Mbps |
| 功耗 | μA 级休眠 | mA 级持续 |
| 拓扑 | 星形 | 微微网/散射网 |
| 协议栈 | GATT 之上即可用 | RFCOMM/L2CAP/SDP |
| 数据模型 | 自定义 GATT Service | SPP 串口等 Profile |

---

## 二、MicroPython 上的 BLE：原生 `bluetooth` 模块与 `aioble`

### 2.1 原生 `bluetooth` 模块

MicroPython 自 1.12 起在 ESP32、nRF 等端口提供 `bluetooth` 模块。它是事件驱动的低层 BLE API，封装底层 BLE stack，直接暴露 GAP、GATT、L2CAP、配对/绑定等操作和 IRQ 事件；但不是 HCI 命令的一一映射：

```python
import bluetooth
import ubinascii

_IRQ_CENTRAL_CONNECT = 1
_IRQ_CENTRAL_DISCONNECT = 2
_IRQ_GATTS_WRITE = 3
_IRQ_SCAN_RESULT = 5
_IRQ_SCAN_DONE = 6
_IRQ_PERIPHERAL_CONNECT = 7
_IRQ_GATTC_NOTIFY = 18

def bt_irq(event, data):
    if event == _IRQ_SCAN_RESULT:
        addr_type, addr, adv_type, rssi, adv_data = data
        print(ubinascii.hexlify(addr), rssi, ubinascii.hexlify(adv_data))

ble = bluetooth.BLE()
ble.active(True)
ble.irq(bt_irq)
ble.gap_scan(2000, 30000, 30000)  # 2s，interval=30ms，window=30ms
```

问题：**回调式 + 手工拼装广播包 + 手工解析 ADV、手写 UUID/句柄映射**，逻辑稍复杂就会变成回调地狱。于是有了 `aioble`。

### 2.2 aioble：基于 asyncio 的高层封装

仓库地址：<https://github.com/micropython/micropython-lib/blob/master/micropython/bluetooth/aioble/README.md>

aioble 是 MicroPython 官方维护的 OO + asyncio 封装，覆盖四大角色：

| 角色 | 功能 |
|------|------|
| Broadcaster | 自动生成常见 AD 字段，并在 legacy ADV 与 Scan Response 之间分配字段（不等同于 Extended Advertising 分片） |
| Peripheral | 等待连接、MTU 协商 |
| Observer (Scanner) | 主动/被动扫描、合并 adv+scan_resp、解析常用字段 |
| Central | 连接 Peripheral、MTU 协商 |
| GATT Client | 按 UUID 发现 Service/Char/Desc，read/write/subscribe、wait notify |
| GATT Server | 注册 Service/Char/Desc、等写、notify/indicate（可等待 ACK） |
| L2CAP CoC | L2CAP 面向连接信道 |
| Security | JSON 密钥存储、配对、查询加密状态 |

**关键设计**：所有远程操作（connect、disconnect、read、write、indicate、l2cap send/recv、pair）均返回 `awaitable`，支持 `timeout_ms`。

#### 2.2.1 安装

按需安装子包（在 MicroPython 设备上通常用 `mip` / `mpremote mip install aioble`，不是普通 CPython 的 `pip install`）：

```
aioble-central   # Central + Observer
aioble-client    # GATT Client
aioble-l2cap
aioble-peripheral
aioble-security
aioble-server
```

或直接安装 `aioble` 元包一次装齐。

#### 2.2.2 Observer：扫描

```python
import aioble
import bluetooth

async def scan():
    async with aioble.scan(duration_ms=5000, interval_us=30000, window_us=30000, active=True) as scanner:
        async for r in scanner:
            print(r, r.name(), r.rssi, list(r.services()), list(r.manufacturer()))
```

aioble 会自动合并同一设备的 adv 与 scan response，并提供常用 AD 字段解析方法（`name()`、`services()`、`manufacturer()`），不用再手工解 AD Type。

#### 2.2.3 Central：连接 + 读写特征

```python
import aioble
import bluetooth
import asyncio

_ENV_SENSE_UUID = bluetooth.UUID(0x181A)
_TEMP_UUID = bluetooth.UUID(0x2A6E)

async def read_temp(addr="aa:bb:cc:dd:ee:ff"):
    device = aioble.Device(aioble.ADDR_PUBLIC, addr)
    try:
        conn = await device.connect(timeout_ms=2000)
    except asyncio.TimeoutError:
        print("connect timeout")
        return

    service = await conn.service(_ENV_SENSE_UUID)
    char = await service.characteristic(_TEMP_UUID)
    data = await char.read(timeout_ms=1000)
    print("temperature raw:", data)
    await conn.disconnect()
```

#### 2.2.4 Client：订阅 Notify

```python
async def subscribe_temp(addr="aa:bb:cc:dd:ee:ff"):
    device = aioble.Device(aioble.ADDR_PUBLIC, addr)
    conn = await device.connect(timeout_ms=2000)

    service = await conn.service(_ENV_SENSE_UUID)
    char = await service.characteristic(_TEMP_UUID)
    await char.subscribe(notify=True)   # aioble 自动写 CCCD=0x0001
    while True:
        data = await char.notified(timeout_ms=1000)
        print("notify:", data)
```

`subscribe(notify=True)` 会查找并写入 CCCD（通常写入 `0x0001`）以启用 notification；随后 `notified()` 协程等待后续 `_IRQ_GATTC_NOTIFY` 数据。Indication 对应 `subscribe(indicate=True)` 和 `indicated()`。业务代码不用再维护底层状态机。

#### 2.2.5 Peripheral + Server：作为 GATT Server

```python
import aioble
import bluetooth
from micropython import const

_ENV_SENSE_UUID = bluetooth.UUID(0x181A)
_TEMP_UUID = bluetooth.UUID(0x2A6E)
_GENERIC_THERMOMETER = const(768)
_ADV_INTERVAL_US = const(250000)

temp_service = aioble.Service(_ENV_SENSE_UUID)
temp_char = aioble.Characteristic(temp_service, _TEMP_UUID, read=True, notify=True, indicate=True)
aioble.register_services(temp_service)

async def peripheral_loop():
    while True:
        conn = await aioble.advertise(
            _ADV_INTERVAL_US,
            name="temp-sense",
            services=[_ENV_SENSE_UUID],
            appearance=_GENERIC_THERMOMETER,
            manufacturer=(0xabcd, b"1234"),
        )
        print("connected from", conn)
        # 业务循环…
        await conn.disconnected()
```

更新本地值并通知：

```python
temp_char.write(b'\x00\x01')                       # 仅本地写
temp_char.write(b'\x00\x01', send_update=True)    # 本地写 + 通知订阅者
temp_char.notify(conn)                              # 用当前值通知指定连接
await temp_char.indicate(conn, timeout_ms=2000)     # 等待 indication 确认；未确认状态可抛 GattError，超时抛 asyncio.TimeoutError
```

aioble 让原本"手工拼装 ADV / Scan Response、GAP 事件回调、CCCD 写、indicate 确认等待"的一团乱麻变成几行协程代码。下面我们转向 ESP32 上 C++ 生态的事实标准——ESPHome。

---

## 三、ESPHome 的 BLE 实现剖析

ESPHome 在 `esphome/components/` 下围绕 ESP-IDF 的 Bluedroid/NimBLE 提供了一套组件。源码引用路径基于 `E:\pyproject\esphome`。

### 3.1 组件目录总览

| 组件 | 职责 |
|------|------|
| `esp32_ble` | BLE 协议栈初始化、UUID、Advertising、ScanResult 数据结构 |
| `esp32_ble_tracker` | **扫描器**与所有监听设备的"广播包路由器" |
| `esp32_ble_client` | GATT 客户端基础设施（`BLEClientBase`） |
| `ble_client` | 用户配置入口，把 BLEClientBase 暴露为 yaml 组件，挂载 sensor/switch/text_sensor |
| `ble_client/sensor` | 在 BLEClient 上轮询/订阅特征的 sensor |
| `esp32_ble_server` | GATT Server，提供 service/char 注册 |
| `esp32_ble_beacon` | iBeacon 广播 |
| `bluetooth_proxy` | 把 ESPHome 设备变成 Home Assistant 的 BLE 代理（**扫描广播 + GATT 透传**） |
| `bthome_mithermometer` 等 | 各种"广播包解析器"，把厂商/BTHome 格式 adv 转成 sensor |
| `ble_presence` / `ble_rssi` / `ble_scanner` | 基于 mac/rssi/adv 文本的简单监听 |

整个 BLE 子系统的核心抽象是 `ESPBTDeviceListener`，所有想拿到广播包的组件都继承它。

### 3.2 `esp32_ble_tracker`：扫描器与设备路由

源码：`esphome/components/esp32_ble_tracker/esp32_ble_tracker.{h,cpp}`

#### 3.2.1 状态机

```cpp
// esp32_ble_tracker.h:175
enum class ClientState : uint8_t {
  INIT, DISCONNECTING, IDLE, DISCOVERED,
  CONNECTING, CONNECTED, ESTABLISHED,
};

// esp32_ble_tracker.h:193
enum class ScannerState {
  IDLE, STARTING, RUNNING, FAILED, STOPPING,
};
```

客户端状态用版本号优化（`esp32_ble_tracker.h:243`）：

```cpp
class ESPBTClient : public ESPBTDeviceListener {
  void set_tracker_state_version(uint8_t *version) { this->tracker_state_version_ = version; }
  void set_state_internal_(ClientState st) {
    this->state_ = st;
    if (this->tracker_state_version_ != nullptr) {
      (*this->tracker_state_version_)++;   // 通知 tracker::loop() 下次重新统计 client 状态
    }
  }
};
```

这是一个典型嵌入式优化：`state_version_` 是 tracker 的 fast-path 失效计数器。client 通过 `set_state()` / `set_state_internal_()` 设置状态时会递增它，让 `ESP32BLETracker::loop()` 下次跳出 fast path 重新统计 client 状态。注意当前实现不比较新旧状态是否相同；它是“状态设置通知”，不是严格的“状态实际变化通知”。

#### 3.2.2 启动扫描

`ESP32BLETracker::start_scan_()`（`esp32_ble_tracker.cpp`）的核心步骤：

1. 检查 `scanner_state_ == IDLE`，否则 `log_unexpected_state_()`。
2. `set_scanner_state_(ScannerState::STARTING)`。
3. 调用所有 listener 的 `on_scan_end()`（前一轮扫描结束通知）。
4. 清空 `already_discovered_`（每轮重新发现）。
5. 填充 `scan_params_`：`scan_type`（ACTIVE/PASSIVE）、`own_addr_type`、`scan_filter_policy`、`scan_interval`、`scan_window`。
6. 启动超时监控：`scan_timeout_state_ = ScanTimeoutState::MONITORING`，`scan_timeout_ms_ = scan_duration_ * 2000`。
7. 调 ESP-IDF API：`esp_ble_gap_set_scan_params()` → `esp_ble_gap_start_scanning(scan_duration_)`。

#### 3.2.3 广播包分发

`ESPBTDeviceListener`（`esp32_ble_tracker.h:143`）是所有广播监听者的基类：

```cpp
class ESPBTDeviceListener {
 public:
  virtual void on_scan_end() {}
  virtual bool parse_device(const ESPBTDevice &device) = 0;        // 高层解析
  virtual bool parse_devices(const BLEScanResult *scan_results, size_t count) { return false; }  // 原始包
  virtual AdvertisementParserType get_advertisement_parser_type() {
    return AdvertisementParserType::PARSED_ADVERTISEMENTS;
  }
};
```

ESPHome 支持两种 listener 需求：

- `PARSED_ADVERTISEMENTS`：tracker 把 raw adv 解析成 `ESPBTDevice`（含 name/rssi/service_uuids/service_data/manufacturer_data/ibeacon），再回调 `parse_device(device)`。`bthome`、`xiaomi_ble`、`atc` 等本地广播解析组件走这条。
- `RAW_ADVERTISEMENTS`：把原始 `BLEScanResult` 数组直接回调 `parse_devices(...)`，用于 `bluetooth_proxy`（避免重复解析，原样转发给 HA）。

这两条路径对**单个 listener** 是二选一，但对整个 tracker **不是全局互斥**：`ESP32BLETracker::recalculate_advertisement_parser_types()` 会汇总所有 listener/client 的需求，分别设置 `raw_advertisements_` 与 `parse_advertisements_`。如果系统里同时有 raw 和 parsed 需求，`process_scan_result_()` 会先分发 raw `BLEScanResult`，再构造 `ESPBTDevice` 分发 parsed advertisement。scanner 状态变化则由另一个接口 `BLEScannerStateListener::on_scanner_state(ScannerState state)` 通知。

`ESPBTDevice`（`esp32_ble_tracker.h`）解析后的字段：

```cpp
class ESPBTDevice {
  esp_bd_addr_t address_;
  esp_ble_addr_type_t address_type_;
  int rssi_;
  std::string name_;
  std::vector<int8_t> tx_powers_;
  optional<uint16_t> appearance_;
  optional<uint8_t> ad_flag_;
  std::vector<ESPBTUUID> service_uuids_;
  std::vector<ServiceData> manufacturer_datas_;
  std::vector<ServiceData> service_datas_;
  const BLEScanResult *scan_result_;          // 原始包也保留
  void parse_adv_(const uint8_t *payload, uint8_t len);   // 解析 AD Type
  optional<ESPBLEiBeacon> get_ibeacon() const;
};
```

> 这是 ESPHome 解析 BLE 广播的核心抽象：**每个组件只需关心"我感兴趣的设备在 adv 里塞了什么"，不用碰 HCI 回调**。

### 3.3 `bthome_mithermometer`：广播包解析器范例

源码：`esphome/components/bthome_mithermometer/bthome_ble.{h,cpp}`

```cpp
// bthome_ble.h:15
class BTHomeMiThermometer final : public esp32_ble_tracker::ESPBTDeviceListener, public Component {
  void set_address(uint64_t address);
  void set_bindkey(std::initializer_list<uint8_t> bindkey);
  bool parse_device(const esp32_ble_tracker::ESPBTDevice &device) override;
  bool handle_service_data_(const ServiceData &service_data, const ESPBTDevice &device);
  bool decrypt_bthome_payload_(...) const;
  sensor::Sensor *temperature_, *humidity_, *battery_level_, *battery_voltage_, *signal_strength_;
};
```

它的工作流程：

1. 注册成 tracker 的 listener：`parse_device` 被每条广播调用。
2. 比对 mac：`if (device.address_uint64() != this->address_) return false;`
3. 取 service data（UUID `0x181C` BTHome）：
   ```cpp
   for (auto &sd : device.get_service_datas()) {
     if (sd.uuid == espbt::ESPBTUUID::from_uint16(0x181C))
       return this->handle_service_data_(sd, device);
   }
   ```
4. 如果有 bindkey，先用 AES-CCM 解密（`decrypt_bthome_payload_`）。
5. 解析 BTHome 帧头：版本、packet id、是否加密。
6. 按 BTHome object ID 解码传感器值，例如 `bthome_ble.cpp:39`：

```cpp
switch (obj_type) {
  case 0x00: /* packet id */         // 1 byte
  case 0x01: /* battery */            // uint8
  case 0x02: /* temperature 0.01C */ // sint16 LE
  case 0x03: /* humidity */           // uint16 LE
  case 0x0C: /* voltage mV */         // uint16
  case 0x12: /* CO2 */                // uint16
  case 0x13: /* TVOC */               // uint16
  ...
}
```

后续 `case 0x02: { /* temperature */ }` 在 `bthome_ble.cpp:409` 把两个字节组装成 `int16_t`，除以 100 后 publish 到 `temperature_->publish_state()`。

> 这是 ESPHome "广播型传感器"模式的标准范式：**实现 `parse_device`，匹配 mac/服务 UUID，解析负载，调用 `sensor->publish_state()`**。全程不建连。

### 3.4 `bluetooth_proxy`：把 ESP32 变成 HA 的 BLE 远程代理

源码：`esphome/components/bluetooth_proxy/bluetooth_proxy.{h,cpp}` + `bluetooth_connection.{h,cpp}`

#### 3.4.1 角色定位

BluetoothProxy 同时继承 `ESPBTDeviceListener` 和 `BLEScannerStateListener`：

```cpp
// bluetooth_proxy.h:56
class BluetoothProxy final : public esp32_ble_tracker::ESPBTDeviceListener,
                             public esp32_ble_tracker::BLEScannerStateListener,
                             public Component {
  bool parse_device(const esp32_ble_tracker::ESPBTDevice &device) override;
  bool parse_devices(const esp32_ble::BLEScanResult *scan_results, size_t count) override;
  AdvertisementParserType get_advertisement_parser_type() override;

  void bluetooth_device_request(const api::BluetoothDeviceRequest &msg);
  void bluetooth_gatt_read(const api::BluetoothGATTReadRequest &msg);
  void bluetooth_gatt_write(const api::BluetoothGATTWriteRequest &msg);
  void bluetooth_gatt_read_descriptor(const api::BluetoothGATTReadDescriptorRequest &msg);
  void bluetooth_gatt_write_descriptor(const api::BluetoothGATTWriteDescriptorRequest &msg);
  void bluetooth_gatt_send_services(const api::BluetoothGATTGetServicesRequest &msg);
  void bluetooth_gatt_notify(const api::BluetoothGATTNotifyRequest &msg);
  void bluetooth_set_connection_params(const api::BluetoothSetConnectionParamsRequest &msg);
  ...
};
```

它支持两类功能位：

```cpp
// bluetooth_proxy.h:41
enum BluetoothProxyFeature : uint32_t {
  FEATURE_PASSIVE_SCAN = 1 << 0,
  FEATURE_ACTIVE_CONNECTIONS = 1 << 1,
  FEATURE_REMOTE_CACHING = 1 << 2,
  FEATURE_PAIRING = 1 << 3,
  FEATURE_CACHE_CLEARING = 1 << 4,
  FEATURE_RAW_ADVERTISEMENTS = 1 << 5,
  FEATURE_STATE_AND_MODE = 1 << 6,
  FEATURE_CONNECTION_PARAMS_SETTING = 1 << 7,
};
```

#### 3.4.2 被动广播代理

`bluetooth_proxy` 自身始终声明 `RAW_ADVERTISEMENTS`，因此不会通过 `parse_device(device)` 消费已经解析好的 `ESPBTDevice`。它在 `parse_devices(scan_results, count)` 中把原始 `adv + scan response` 字节、MAC、RSSI、地址类型打包成 `api::BluetoothLERawAdvertisementsResponse`，通过 ESPHome 原生 API 批量推给 HA。HA 端再把这些字节转换为 bleak 风格的 `AdvertisementData` / `BluetoothServiceInfoBleak`，由本地解析器（如 `homeassistant.components.bluetooth` 下的各集成）继续处理。

#### 3.4.3 主动 GATT 代理

这是真正的难点。HA 想读某个 BLE 设备的特征时，发 `BluetoothGATTReadRequest` 给 ESPHome，proxy 调用：

```cpp
// bluetooth_proxy.cpp:274
void BluetoothProxy::bluetooth_gatt_read(const api::BluetoothGATTReadRequest &msg) {
  // 1. 找到 address 对应的 BluetoothConnection 槽位
  // 2. 通过它发 esp_ble_gattc_read_char(...)
  // 3. 在 gattc_event_handler 收到 ESP_GATTC_READ_CHAR_EVT 时
  //    把 value 回封 api::BluetoothGATTReadResult 推给 HA
}
```

`BluetoothConnection` 继承 `BLEClientBase`：

```cpp
// bluetooth_connection.h:11
class BluetoothConnection final : public esp32_ble_client::BLEClientBase { ... };
```

它复用 `BLEClientBase` 的全部 GATT 状态机（见下节），并叠加"把 GATT 事件翻译成 api 消息"的逻辑。ESP32 同时支持 `BLUETOOTH_PROXY_MAX_CONNECTIONS` 个并发主动连接（典型 3 个）。

#### 3.4.4 服务缓存：V3 协议

```cpp
// esp32_ble_tracker.h:218
enum class ConnectionType : uint8_t {
  V1,                  // 全程保留 services 在内存
  V3_WITH_CACHE,       // 客户端已有缓存，不重复拉取
  V3_WITHOUT_CACHE,    // 拉一次发完即丢，节省内存
};
```

HA 维护一个 GATT 服务/特征缓存。连接请求层面区分 `V3_WITH_CACHE` 与 `V3_WITHOUT_CACHE`：前者表示 HA 端已有服务缓存，proxy 可以跳过重复服务发现；后者表示需要拉取服务并发回 HA，发送完成后可释放内存。ESPHome 还可通过 `cache_services` 配置启用 IDF 的 GATTC NVS cache（`CONFIG_BT_GATTC_CACHE_NVS_FLASH`）。旧的 V1 connect request 在当前 proxy 中已经移除，运行时会返回失败。

#### 3.4.5 ESPHome 解析还是 HA 解析？答案是"原样透传"

这是理解 ESPHome BluetoothProxy 的**关键事实**：对于广播（advertising）阶段，ESPHome **完全不解析业务数据**，直接把原始字节流透传给 Home Assistant，由 HA 端做所有解析。

**ESPHome 端的代码铁证**（`bluetooth_proxy.cpp`）：

```cpp
// bluetooth_proxy.cpp:143
esp32_ble_tracker::AdvertisementParserType BluetoothProxy::get_advertisement_parser_type() {
  return esp32_ble_tracker::AdvertisementParserType::RAW_ADVERTISEMENTS;
}

// bluetooth_proxy.cpp:73
bool BluetoothProxy::parse_device(const esp32_ble_tracker::ESPBTDevice &device) {
  // This method should never be called since bluetooth_proxy always uses raw advertisements
  // but we need to provide an implementation to satisfy the virtual method requirement
  return false;
}

// bluetooth_proxy.cpp:80
bool BluetoothProxy::parse_devices(const esp32_ble::BLEScanResult *scan_results, size_t count) {
  if (!api::global_api_server->is_connected() || this->api_connection_ == nullptr)
    return false;

  auto &advertisements = this->response_.advertisements;

  for (size_t i = 0; i < count; i++) {
    auto &result = scan_results[i];
    uint8_t length = result.adv_data_len + result.scan_rsp_len;

    // Fill in the data directly at current position
    auto &adv = advertisements[this->response_.advertisements_len];
    adv.address = esp32_ble::ble_addr_to_uint64(result.bda);
    adv.rssi = result.rssi;
    adv.address_type = result.ble_addr_type;
    adv.data_len = length;
    std::memcpy(adv.data, result.ble_adv, length);   // ← 原始 adv+scan_resp 字节流，零解析
    this->response_.advertisements_len++;
    ...
  }
  return true;
}
```

几个关键设计点：

1. **`RAW_ADVERTISEMENTS` 路径**：proxy 显式声明走“原始广播”路径，因此它自身不会收到 `ESPBTDevice::parse_adv_()` 拆好的业务字段，而是直接接收 `BLEScanResult` 原始扫描结果。注意这只描述 proxy 这个 listener；tracker 全局仍可同时服务其他 `PARSED_ADVERTISEMENTS` listener。
2. **零拷贝式的 `memcpy`**：proxy 把 31 字节 adv + 31 字节 scan response 直接 `memcpy` 到 `api::BluetoothLERawAdvertisement::data[62]`（头部有 `static_assert` 校验数组大小恰好 62 字节，符合 BLE 规范），加上 mac、rssi、addr_type，作为一条 `BluetoothLERawAdvertisement` 进批次缓存。
3. **批量发送**：攒够 `BLUETOOTH_PROXY_ADVERTISEMENT_BATCH_SIZE`（codegen 期生成）才 flush，最大化 WiFi 吞吐效率。
4. **能力声明**：`FEATURE_RAW_ADVERTISEMENTS = 1 << 5` 在 capability bitmap 中告知 HA "我能给你原始包"。

**Home Assistant 端的接包链路**（`E:\pyproject\core\homeassistant`）：

```
ESPHome bluetooth_proxy (raw adv bytes)
   │  via aioesphomeapi / bleak_esphome
   ▼
bleak_esphome.connect_scanner(...) 创建的 ESPHomeScanner
   │  在 HA core 中通过 esphome/bluetooth.py 注册
   ▼
homeassistant.components.bluetooth.api.async_register_scanner(...)
   ▼
homeassistant.components.bluetooth.manager.HomeAssistantBluetoothManager
   │  async_register_hass_scanner(...) 接入 scanner pipeline
   ▼
HomeAssistantBluetoothManager._discover_service_info(service_info)
   │  _integration_matcher.match_domains(service_info)  ← 用各集成 manifest.json 的 "bluetooth" 规则做 discovery 匹配
   │  _callback_index.match_callbacks(service_info)      ← 已注册设备/协调器的运行时 callback
   ▼
两条分发路径：
   ├─→ callback(service_info, BluetoothChange.ADVERTISEMENT)   ← 给已注册设备（bthome/xiaomi/atc 等 coordinator）
   └─→ discovery_flow.async_create_flow(domain, ..., service_info)  ← 触发新设备发现流程
```

关键代码（`homeassistant/components/bluetooth/manager.py:137-170`，`HomeAssistantBluetoothManager._discover_service_info`）：

```python
matched_domains = self._integration_matcher.match_domains(service_info)
...
for match in self._callback_index.match_callbacks(service_info):
    callback = match[CALLBACK]
    try:
        callback(service_info, BluetoothChange.ADVERTISEMENT)   # 推给具体集成
    except Exception:
        _LOGGER.exception("Error in bluetooth callback")

if not matched_domains:
    return  # avoid creating DiscoveryKey if there are no matches

discovery_key = discovery_flow.DiscoveryKey(domain=DOMAIN, key=service_info.address, version=1)
for domain in matched_domains:
    discovery_flow.async_create_flow(
        self.hass, domain,
        {"source": config_entries.SOURCE_BLUETOOTH},
        service_info,
        discovery_key=discovery_key,
    )
```

`BluetoothServiceInfoBleak`（来自 `home_assistant_bluetooth` 包）封装了解析后的 BLE 字段：

```python
# homeassistant/components/bluetooth/models.py:6-10
from home_assistant_bluetooth import BluetoothServiceInfoBleak

BluetoothChange = Enum("BluetoothChange", "ADVERTISEMENT")           # 唯一事件：收到广播
type BluetoothCallback = Callable[[BluetoothServiceInfoBleak, BluetoothChange], None]
type ProcessAdvertisementCallback = Callable[[BluetoothServiceInfoBleak], bool]
```

`BluetoothServiceInfoBleak` 来自 `home_assistant_bluetooth` 包。HA core 侧把它作为统一的 BLE 广播服务信息对象使用；scanner / habluetooth / bleak_esphome 已经把底层广播转换为 bleak 风格的 `BLEDevice + AdvertisementData` 后，上层集成通过 `BluetoothServiceInfoBleak` 读取 `.name`、`.address`、`.rssi`、`.service_data`、`.service_uuids`、`.manufacturer_data` 等字段。

**具体集成的消费方式**（以 bthome 为例）：

BTHome 有两层逻辑：

1. **新设备 discovery**：`manifest.json` 的 `"bluetooth"` 规则匹配 service data UUID（如 `181C` / `181E` / `FCD2`），`HomeAssistantBluetoothManager._discover_service_info()` 触发 `bthome` config flow，`BTHomeConfigFlow.async_step_bluetooth()` 再判断设备是否支持。
2. **已配置设备运行时更新**：`async_setup_entry()` 创建 `BTHomePassiveBluetoothProcessorCoordinator`；coordinator 按设备 address 注册 bluetooth callback。收到匹配地址的 `BluetoothServiceInfoBleak` 后，调用同步函数 `process_service_info(...)`，内部再调用 `BTHomeBluetoothDeviceData.update(service_info)` 得到 `SensorUpdate`，最后由 `PassiveBluetoothDataProcessor` 分发给实体。

```python
# homeassistant/components/bthome/__init__.py:76
def process_service_info(
    hass: HomeAssistant,
    entry: BTHomeConfigEntry,
    device_registry: DeviceRegistry,
    service_info: BluetoothServiceInfoBleak,
) -> SensorUpdate:
    """Process a BluetoothServiceInfoBleak."""
    coordinator = entry.runtime_data
    data = coordinator.device_data
    update = data.update(service_info)
    ...
    return update
```

ESPHome 集成注册 scanner 的入口（`homeassistant/components/esphome/bluetooth.py:39-75`，核心调用为 `connect_scanner(...)` 和 `async_register_scanner(...)`）：

```python
# homeassistant/components/esphome/bluetooth.py
from bleak_esphome import connect_scanner

@hass_callback
def async_connect_scanner(hass, entry, entry_data, cli, device_info, device_id):
    """Connect scanner."""
    client_data = connect_scanner(cli, device_info, entry_data.available)   # ← 桥接器
    scanner = client_data.scanner                                             # ESPHomeScanner 实例
    feature_flags = device_info.bluetooth_proxy_feature_flags_compat(api_version)
    state_and_mode = bool(feature_flags & BluetoothProxyFeature.FEATURE_STATE_AND_MODE)
    ...
    callbacks = [
        async_register_scanner(                                              # ← 注册进 HA BluetoothManager
            hass, scanner,
            source_domain=DOMAIN,
            source_model=device_info.model,
            source_config_entry_id=entry_data.entry_id,
            source_device_id=device_id,
        ),
        scanner.async_setup(),
    ]
```

`bleak_esphome.connect_scanner` 把 ESPHome 的原生 API 包装成一个 `BaseHaScanner` 子类（`ESPHomeScanner`）；HA core 侧只负责注册 scanner，ESPHome 原生 API 到 scanner 的具体转换由外部 `bleak_esphome` 包提供。该 scanner 收到 ESPHome 转发过来的 `BluetoothLERawAdvertisements` 后，调用 `async_step_advertisement()` 喂给 HA 的 BluetoothManager。此后 HA 端的扫描匹配逻辑完全把这台 ESP32 当成“一个普通的本地蓝牙适配器”对待。

**两条路径的对比总结**

| 数据类型 | ESPHome 解析？ | HA 解析？ | 说明 |
|----------|---------------|----------|------|
| 被动广播（adv/service_data/manufacturer_data） | ❌ 不解析，原样透传 | ✅ 在 `BluetoothManager` 统一解析分发 | ESPHome 只关心把它打包成 `BluetoothLERawAdvertisement` 发走 |
| 主动 GATT（read/write/notify service） | ✅ 由 `BluetoothConnection` 状态机完成底层 GATT 操作 | ⚠️ HA 通过 `bleak_esphome` 的"虚拟 bleak client"发起请求，ESPHome 只是个"远程 GATT 代理" | ESPHome 不知道 sensor 含义，只搬运字节 |
| `bthome_mithermometer` 等 ESPHome 内置 sensor 组件 | ✅ 在 ESP32 上完整解析 | — | 这是另一条独立的路径：ESPHome 自己当 sensor 平台，结果通过 ESPHome 自己的 `sensor` 平台上报 HA（实体由 ESPHome 拥有）；它不依赖 HA 的 bluetooth proxy 解析链路 |

**为什么这样设计？**

1. **节省 ESP32 算力**：adv 解析、协议匹配、数据解码都搬到 HA 上跑，ESP32 只做"BLE 收发器"。
2. **协议演进只改 HA**：BTHome 加一个 object id，只升级 HA 的 `bthome-ble` 库即可，无需重刷每台 ESP32 固件。
3. **解耦集成发现**：HA 的 BluetoothManager 通过 `manifest.json` 里的 `bluetooth` 匹配规则做集成匹配，新设备类型在 HA 端注册即可被发现。
4. **单一数据源**：HA 端的 bthome 集成既能接收 ESPHome 代理的 adv，也能接收本地 USB 蓝牙 dongle 的 adv，处理代码完全相同——因为它们都变成 `BluetoothServiceInfoBleak`。

> **结论一句话**：proxy 模式下，ESPHome 是个“原样转发 BLE 流量的远程无线适配器”；只有当你在 ESP32 端直接用 `bthome_mithermometer`、`xiaomi_ble` 这类“内置 sensor 组件”时，ESP32 才会自己解析广播并作为 sensor 拥有实体。两者是两条数据归属不同的链路：proxy 链路让 HA 解析并拥有实体，本地组件链路让 ESPHome 解析并拥有实体。它们不是 tracker 层面的全局互斥；同一固件里可以同时存在 raw proxy listener 和 parsed 本地 listener，但同一传感器设备若被两边都配置，可能在 HA 中出现重复实体或语义冲突。

### 3.4.6 如果要在 HA 中写一个“广播 + 主动 GATT”的自定义集成

这类集成最像 `switchbot`、`ld2410_ble`、`eufylife_ble` 与 `bthome` 的组合，而不是纯 `bthome`。核心思路是：**把 ESPHome proxy 送进来的 `BluetoothServiceInfoBleak` 当成统一入口，先吃广播，再按需发起 GATT 连接**。

这里“基于 `bleak_esphome`”的准确含义是：你的集成通常**不直接导入 `bleak_esphome`**，也不直接连接某台 ESPHome 的 native API。ESPHome 集成本身已经在 `homeassistant/components/esphome/bluetooth.py` 里调用 `bleak_esphome.connect_scanner(...)`，把 ESPHome proxy 注册成 HA 的 Bluetooth scanner。之后你的集成只面向 HA 的 Bluetooth API：

- 广播入口是 `BluetoothServiceInfoBleak`；
- `service_info.device` 是 bleak 的 `BLEDevice`；
- 如果这个 `BLEDevice` 来自 ESPHome proxy，那么后续 `BleakClient(service_info.device)` 的 GATT read/write/notify 会经由 `bleak_esphome` 隧道转成 ESPHome native API 请求。

前提是 ESPHome 端 `bluetooth_proxy` 开启了 active connection 能力（例如 `active: true`，并且设备能力包含 `FEATURE_ACTIVE_CONNECTIONS`）。如果 proxy 只有 passive scan，HA 只能收到广播，不能通过它发起 GATT 连接。

#### 集成文件结构

一个最小的 HA custom integration 可以长这样：

```text
custom_components/foo_ble/
├── manifest.json
├── const.py
├── config_flow.py
├── __init__.py
├── coordinator.py
├── sensor.py
└── button.py          # 可选：触发 GATT 配置写入
```

`manifest.json` 里用 `bluetooth` matcher 让 HA 在收到匹配广播时自动触发 discovery flow。匹配字段可用 `service_uuid`、`service_data_uuid`、`manufacturer_id`、`manufacturer_data_start`、`local_name` 等；如果设备需要 GATT 连接，`connectable` 应设为 `true`：

```json
{
  "domain": "foo_ble",
  "name": "Foo BLE",
  "codeowners": ["@yourhandle"],
  "config_flow": true,
  "dependencies": ["bluetooth"],
  "bluetooth": [
    {
      "connectable": true,
      "service_data_uuid": "0000fff0-0000-1000-8000-00805f9b34fb"
    }
  ],
  "requirements": []
}
```

`config_flow.py` 负责把 discovery 变成 config entry。它收到的 `discovery_info` 就是 `BluetoothServiceInfoBleak`：

```python
# custom_components/foo_ble/config_flow.py
from homeassistant import config_entries
from homeassistant.components.bluetooth import BluetoothServiceInfoBleak

from .const import DOMAIN

class FooBleConfigFlow(config_entries.ConfigFlow, domain=DOMAIN):
    VERSION = 1

    async def async_step_bluetooth(
        self, discovery_info: BluetoothServiceInfoBleak
    ):
        address = discovery_info.address
        await self.async_set_unique_id(address)
        self._abort_if_unique_id_configured()

        # 这里可以解析 service_info.service_data / manufacturer_data 判断型号
        self.context["title_placeholders"] = {"name": discovery_info.name or address}
        return self.async_create_entry(
            title=discovery_info.name or address,
            data={"address": address},
        )
```

运行时入口 `__init__.py` 注册蓝牙 callback。这里和 `eufylife_ble` / `ld2410_ble` 的写法类似：按 address 注册 `BluetoothCallbackMatcher`，扫描模式用 `ACTIVE`，这样 HA 会尽量使用可连接扫描器；收到广播时先解析广播，再按需调度 GATT 连接：

```python
# custom_components/foo_ble/__init__.py
from bleak import BleakClient, BleakError

from homeassistant.components import bluetooth
from homeassistant.components.bluetooth.match import ADDRESS, BluetoothCallbackMatcher
from homeassistant.const import Platform
from homeassistant.core import HomeAssistant, callback
from homeassistant.exceptions import ConfigEntryNotReady

from .const import DOMAIN
from .coordinator import FooBleCoordinator

PLATFORMS = [Platform.SENSOR, Platform.BUTTON]

async def async_setup_entry(hass: HomeAssistant, entry):
    address = entry.data["address"]

    # 若启动时还没见过广播，可以等下一次 callback；若必须立即连接，
    # 可像 ld2410_ble 一样先 async_ble_device_from_address(..., connectable=True)。
    ble_device = bluetooth.async_ble_device_from_address(hass, address, connectable=True)
    if ble_device is None:
        raise ConfigEntryNotReady(f"{address} not reachable by a connectable scanner")

    coordinator = FooBleCoordinator(hass, entry, ble_device)
    entry.runtime_data = coordinator

    @callback
    def _async_update_ble(service_info, change):
        coordinator.async_handle_advertisement(service_info)

    entry.async_on_unload(
        bluetooth.async_register_callback(
            hass,
            _async_update_ble,
            BluetoothCallbackMatcher({ADDRESS: address}),
            bluetooth.BluetoothScanningMode.ACTIVE,
        )
    )

    await hass.config_entries.async_forward_entry_setups(entry, PLATFORMS)
    return True
```

`coordinator.py` 中维护状态，并把 GATT 操作集中在一个地方。关键点是：**连接时优先使用最近一次广播里的 `service_info.device`**，不要只传 address 字符串；这个 `BLEDevice` 带着 scanner/source 信息，若它来自 ESPHome proxy，`BleakClient` 才能走 `bleak_esphome` 后端。

```python
# custom_components/foo_ble/coordinator.py
import asyncio
from bleak import BleakClient, BleakError

from homeassistant.components import bluetooth
from homeassistant.core import HomeAssistant, callback
from homeassistant.helpers.update_coordinator import DataUpdateCoordinator

FOO_SERVICE_UUID = "0000fff0-0000-1000-8000-00805f9b34fb"
FOO_CONFIG_CHAR_UUID = "0000fff2-0000-1000-8000-00805f9b34fb"

class FooBleCoordinator(DataUpdateCoordinator):
    def __init__(self, hass: HomeAssistant, entry, ble_device):
        super().__init__(hass, None, name=entry.title)
        self.entry = entry
        self.ble_device = ble_device
        self.state = {}
        self._configure_task: asyncio.Task | None = None

    @callback
    def async_handle_advertisement(
        self, service_info: bluetooth.BluetoothServiceInfoBleak
    ) -> None:
        self.ble_device = service_info.device

        # 解析 ESPHome proxy 转来的广播：service_data / manufacturer_data / rssi / name 等
        self.state.update(parse_foo_advertisement(service_info))
        self.async_set_updated_data(self.state)

        if self.state.get("needs_configure") and self._configure_task is None:
            self._configure_task = self.hass.async_create_task(self.async_configure())

    async def async_configure(self) -> None:
        try:
            async with BleakClient(self.ble_device) as client:
                await client.write_gatt_char(FOO_CONFIG_CHAR_UUID, b"\x01\x02", response=True)
                raw = await client.read_gatt_char(FOO_CONFIG_CHAR_UUID)
        except BleakError as exc:
            self.logger.warning("GATT configure failed for %s: %s", self.ble_device.address, exc)
            return
        finally:
            self._configure_task = None

        self.state["config_value"] = bytes(raw)
        self.state["needs_configure"] = False
        self.async_set_updated_data(self.state)
```

上面是最小骨架。生产代码通常还会加：

- **连接可达性判断**：参考 `switchbot`，`needs_poll()` 里先检查 `bluetooth.async_ble_device_from_address(hass, address, connectable=True)`，没有可连接 scanner 就不要发起 GATT。
- **连接清理和重试**：参考 `ld2410_ble`，可用 `bleak_retry_connector.close_stale_connections_by_address()` / `get_device()` 处理 stale connection 和启动时找不到设备的问题。
- **广播驱动的主动轮询**：如果 GATT 读是周期性补充数据，直接继承 `ActiveBluetoothProcessorCoordinator` 或 `ActiveBluetoothDataUpdateCoordinator`；它们会在每次广告到达时调用 `needs_poll_method(service_info, last_poll)`，需要时再执行 `poll_method(service_info)`。
- **按钮/服务触发配置**：像 `button.py` 或 HA service 那样调用 `await coordinator.async_configure()`，不要在 entity 里散落 GATT 连接逻辑。
- **卸载清理**：保存 `async_register_callback()` 返回的取消函数到 `entry.async_on_unload()`；如果设备库持有长连接或 notify 订阅，卸载/HA stop 时要 `stop()` / `disconnect()`。

与 BTHome 最大的差异是：BTHome 是 `connectable=False` 的纯被动解析；这里应该使用 `BluetoothScanningMode.ACTIVE`，并确保 matcher / config entry 针对的是可连接设备。只要你使用 `BluetoothServiceInfoBleak.device` 创建 `BleakClient`，同一套集成既可走本机蓝牙适配器，也可走 ESPHome `bluetooth_proxy` + `bleak_esphome` 的远程 GATT 代理。

### 3.5 `esp32_ble_client`：GATT 客户端基础设施

源码：`esphome/components/esp32_ble_client/ble_client_base.{h,cpp}`

#### 3.5.1 核心类

```cpp
// ble_client_base.h:26
class BLEClientBase : public espbt::ESPBTClient, public Component {
  void setup() override; void loop() override;
  bool parse_device(const espbt::ESPBTDevice &device) override;   // 等到匹配 mac 才 connect
  bool gattc_event_handler(esp_gattc_cb_event_t event, esp_gatt_if_t gattc_if,
                           esp_ble_gattc_cb_param_t *param) override;
  void gap_event_handler(...) override;
  void connect() override; void disconnect() override;
  void unconditional_disconnect();
  void release_services();

  bool connected() { return this->state() == espbt::ClientState::ESTABLISHED; }

  BLEService *get_service(espbt::ESPBTUUID uuid);
  BLECharacteristic *get_characteristic(espbt::ESPBTUUID service, espbt::ESPBTUUID chr);
  BLEDescriptor *get_descriptor(...);
  BLEDescriptor *get_config_descriptor(uint16_t handle);   // 找 CCCD

  void set_address(uint64_t address);   // 把 uint64 拆成 esp_bd_addr_t
  void set_remote_addr_type(esp_ble_addr_type_t);
  uint16_t get_conn_id() const { return this->conn_id_; }
  int get_gattc_if() const { return this->gattc_if_; }
};
```

#### 3.5.2 GATT 状态机

`BLEClientBase` 的 GATT 客户端生命周期由 `gattc_event_handler` 里的大 switch 驱动，但当前 ESPHome 的职责分配和早期教程常见写法略有不同：

| ESP-IDF 事件 / 调用点 | 当前 ESPHome 处理 |
|--------------|------|
| `loop()` + `INIT` | 注册 GATTC app：`esp_ble_gattc_app_register(app_id)` |
| `ESP_GATTC_REG_EVT` | 保存 `gattc_if_`；不在这里发起连接 |
| `parse_device()` | 扫描到匹配 MAC 后把 client 状态置为 `DISCOVERED` |
| `connect()` | tracker 在合适时机调用它；内部执行 `esp_ble_gattc_open(...)` |
| `ESP_GATTC_CONNECT_EVT` | 保存 `conn_id_`，调用 `esp_ble_gattc_send_mtu_req()` 请求 MTU |
| `ESP_GATTC_OPEN_EVT` | 连接打开成功；非 `V3_WITH_CACHE` 模式启动 `esp_ble_gattc_search_service()` |
| `ESP_GATTC_CFG_MTU_EVT` | 记录协商到的 MTU，不负责启动服务发现 |
| `ESP_GATTC_SEARCH_RES_EVT` | 收到一个 service，push 到 `services_` |
| `ESP_GATTC_SEARCH_CMPL_EVT` | 服务发现完成，base client 进入 `ESTABLISHED`；上层 node/proxy 在此后按需找 characteristic/descriptor |
| `get_characteristic()` / `get_descriptor()` | characteristic / descriptor 是 lazy parse：上层查询时才通过 `esp_ble_gattc_get_all_char()` 等 API 拉取 |
| `ESP_GATTC_REG_FOR_NOTIFY_EVT` | notify 注册完成；非 V3 模式下 base class 查找 CCCD 并写入 notify/indicate enable |
| `ESP_GATTC_NOTIFY_EVT` | 真正的通知数据到达，交给上层 `BLEClientNode` 或 `BluetoothConnection` 处理 |
| `ESP_GATTC_DISCONNECT_EVT` | 释放 services，`set_idle_()` |

关键状态转移（`ble_client_base.h:150`）：

```cpp
void set_idle_() {
  this->set_state(espbt::ClientState::IDLE);
  this->conn_id_ = UNSET_CONN_ID;
}

void set_disconnecting_() {
  this->disconnecting_started_ = millis();
  this->set_state(espbt::ClientState::DISCONNECTING);
  this->enable_loop();   // 让 loop() 能跑 10s 安全超时
}
```

> 教科书级别的 BLE GATT 客户端实现。手工维护状态机是 ESP-IDF Bluedroid API 的固有形态——所有 GATT 操作都是异步事件回调，没有 await/future 概念。

#### 3.5.3 `ble_client` 用户层

`ble_client` 在 `BLEClientBase` 之上提供 yaml 配置入口：

```yaml
ble_client:
  - mac_address: AA:BB:CC:DD:EE:FF
    id: mybeacon

sensor:
  - platform: ble_client
    ble_client_id: mybeacon
    name: "Beacon Temperature"
    type: characteristic
    service_uuid: "181A"
    characteristic_uuid: "2A6E"
    notify: true        # 订阅 notify
    update_interval: 60s
```

`BLESensor`（`ble_client/sensor/ble_sensor.h`）：

```cpp
class BLESensor : public sensor::Sensor, public PollingComponent, public BLEClientNode {
  void gattc_event_handler(esp_gattc_cb_event_t, esp_gatt_if_t, esp_ble_gattc_cb_param_t *) override;
  void update() override;             // 周期 esp_ble_gattc_read_char
  void set_enable_notify(bool n) { this->notify_ = n; }
};
```

`BLESensor` 作为 `BLEClientNode` 接收父 `BLEClient` 转发的 GATTC 事件。它在 `ESP_GATTC_SEARCH_CMPL_EVT` 后调用 `parent()->get_characteristic(service_uuid_, char_uuid_)` 查找 characteristic；如果启用 notify，则先调用 `esp_ble_gattc_register_for_notify()`，待 `ESP_GATTC_REG_FOR_NOTIFY_EVT` 后把 node state 设为 `ESTABLISHED`。`update()` 只有在 `node_state == ESTABLISHED` 时才调用 `esp_ble_gattc_read_char(...)`；`ESP_GATTC_READ_CHAR_EVT` / `ESP_GATTC_NOTIFY_EVT` 中再把字节流交给 `parse_data_` / 用户 lambda 并 `publish_state`。

---

## 四、自己实现一个 ESPHome 尚未支持的 BLE 信标监听组件

### 4.1 场景假设

假设有一种自定义传感器信标 `FooBeacon`：

- **不广播传感器数据**（不像 BTHome 那样塞 service_data），而是必须**建立 GATT 连接后通过某个 characteristic 的 notify 推送**数据。
- Service UUID：`0000fff0-0000-1000-8000-00805f9b34fb`（即 16-bit `0xFFF0` 展开）
- Notify Char UUID：`0000fff1-...`
- 数据帧：4 字节 `[seq, temp_lo, temp_hi, bat]`，温度为 `int16` 小端，单位 0.01°C；电量为 uint8 百分比。

ESPHome 官方没有这个组件，我们需要自己写一个 **external component**。

### 4.2 总体方案

两条可选路径：

| 路径 | 实现 | 优劣 |
|------|------|------|
| A. 基于广播解析（继承 `ESPBTDeviceListener`） | 在 `parse_device` 里解 adv | 简单，但不适用——本信标无 adv 数据 |
| B. 基于 GATT 客户端（继承 `BLEClientNode`） | 复用 `BLEClientBase` 状态机 + notify 订阅 | **本场景必须用 B** |

下面给出 B 路径的完整骨架，放在 `custom_components/foo_beacon/`。

### 4.3 文件结构

```
custom_components/foo_beacon/
├── __init__.py         # Python 配置入口（注册 COMPONENT_SCHEMA）
├── foo_beacon.h        # C++ 头文件
├── foo_beacon.cpp      # C++ 实现
└── sensor.py           # 注册 sensor 平台
```

### 4.4 `__init__.py`：配置注册

ESPHome 的 external component 通过 Python 文件声明命名空间和 codegen 类型；这个例子只有 `sensor` 平台，不需要单独的根级 `foo_beacon:` 配置：

```python
# custom_components/foo_beacon/__init__.py
import esphome.codegen as cg

CODEOWNERS = ["@yourhandle"]
foo_beacon_ns = cg.esphome_ns.namespace("foo_beacon")
```

### 4.5 `sensor.py`：sensor 平台

```python
# custom_components/foo_beacon/sensor.py
import esphome.codegen as cg
import esphome.config_validation as cv
from esphome.components import ble_client, sensor
from esphome.const import (
    CONF_ID, UNIT_CELSIUS,
    STATE_CLASS_MEASUREMENT, DEVICE_CLASS_TEMPERATURE,
)

from . import foo_beacon_ns

CONF_TEMPERATURE = "temperature"
CONF_BATTERY = "battery"

FooBeaconSensor = foo_beacon_ns.class_(
    "FooBeaconSensor", sensor.Sensor, cg.PollingComponent, ble_client.BLEClientNode
)

CONFIG_SCHEMA = (
    sensor.sensor_schema(
        unit_of_measurement=UNIT_CELSIUS,
        accuracy_decimals=2,
        device_class=DEVICE_CLASS_TEMPERATURE,
        state_class=STATE_CLASS_MEASUREMENT,
    )
    .extend(
        {
            cv.GenerateID(): cv.declare_id(FooBeaconSensor),
        }
    )
    .extend(cv.polling_component_schema("60s"))
    .extend(ble_client.BLE_CLIENT_SCHEMA)
)

async def to_code(config):
    var = cg.new_Pvariable(config[CONF_ID])
    await cg.register_component(var, config)
    await sensor.register_sensor(var, config)
    await ble_client.register_ble_node(var, config)
```

### 4.6 `foo_beacon.h`：C++ 头文件

```cpp
// custom_components/foo_beacon/foo_beacon.h
#pragma once

#ifdef USE_ESP32

#include "esphome/core/component.h"
#include "esphome/components/ble_client/ble_client.h"
#include "esphome/components/esp32_ble_tracker/esp32_ble_tracker.h"
#include "esphome/components/sensor/sensor.h"

#include <esp_gattc_api.h>

namespace esphome::foo_beacon {

namespace espbt = esphome::esp32_ble_tracker;

// 信标约定的 Service / Characteristic UUID（16-bit 0xFFF0/0xFFF1 展开为 128-bit）
static constexpr uint16_t FOO_SERVICE_UUID16 = 0xFFF0;
static constexpr uint16_t FOO_CHAR_NOTIFY_UUID16 = 0xFFF1;

class FooBeaconSensor : public sensor::Sensor,
                        public PollingComponent,
                        public ble_client::BLEClientNode {
 public:
  void setup() override;
  void loop() override;
  void update() override;            // PollingComponent 周期触发
  void gattc_event_handler(esp_gattc_cb_event_t event, esp_gatt_if_t gattc_if,
                           esp_ble_gattc_cb_param_t *param) override;
  void dump_config() override;

  // 当前 BLEClientNode 没有 node_connected()；连接后初始化在 gattc_event_handler()
  // 的 ESP_GATTC_SEARCH_CMPL_EVT 分支中完成。

 protected:
  void handle_frame_(const uint8_t *data, size_t len);   // 解析 [seq,t_lo,t_hi,bat]

  uint16_t char_handle_{0};   // notify 特征句柄，发现时填
  bool subscribed_{false};
};

}  // namespace esphome::foo_beacon

#endif  // USE_ESP32
```

### 4.7 `foo_beacon.cpp`：实现

```cpp
// custom_components/foo_beacon/foo_beacon.cpp
#include "foo_beacon.h"

#ifdef USE_ESP32

#include "esphome/components/esp32_ble_client/ble_client_base.h"
#include "esphome/core/log.h"

#include <esp_gattc_api.h>
#include <cstring>

namespace esphome::foo_beacon {

static const char *const TAG = "foo_beacon";

void FooBeaconSensor::dump_config() {
  ESP_LOGCONFIG(TAG, "FooBeacon Sensor:");
  ESP_LOGCONFIG(TAG, "  Service: 0x%04X", FOO_SERVICE_UUID16);
  ESP_LOGCONFIG(TAG, "  Char:    0x%04X", FOO_CHAR_NOTIFY_UUID16);
  LOG_UPDATE_INTERVAL(this);
}

void FooBeaconSensor::setup() {
  this->node_state = espbt::ClientState::IDLE;
}

void FooBeaconSensor::loop() {
  // Parent BLEClientNode 有 loop()，本组件依赖 polling update() 和 GATT 回调即可。
  this->disable_loop();
}

void FooBeaconSensor::update() {
  // 仅当 node 自己完成 service/notify 初始化后才主动读一次（轮询模式）
  auto *client = this->parent();   // ble_client::BLEClient *
  if (client == nullptr || this->node_state != espbt::ClientState::ESTABLISHED)
    return;

  if (this->char_handle_ != 0) {
    esp_ble_gattc_read_char(client->get_gattc_if(), client->get_conn_id(),
                            this->char_handle_, ESP_GATT_AUTH_REQ_NONE);
  }
}

void FooBeaconSensor::gattc_event_handler(esp_gattc_cb_event_t event,
                                           esp_gatt_if_t gattc_if,
                                           esp_ble_gattc_cb_param_t *param) {
  auto *client = this->parent();
  if (client == nullptr) return;
  if (gattc_if != client->get_gattc_if()) return;   // 不是我的接口

  switch (event) {
    case ESP_GATTC_SEARCH_CMPL_EVT: {
      ESP_LOGI(TAG, "Connected, discovering FOO service 0x%04X", FOO_SERVICE_UUID16);

      espbt::ESPBTUUID svc = espbt::ESPBTUUID::from_uint16(FOO_SERVICE_UUID16);
      espbt::ESPBTUUID chr = espbt::ESPBTUUID::from_uint16(FOO_CHAR_NOTIFY_UUID16);

      auto *characteristic = client->get_characteristic(svc, chr);
      if (characteristic == nullptr) {
        ESP_LOGW(TAG, "FOO characteristic not found, disconnecting");
        client->disconnect();
        return;
      }
      this->char_handle_ = characteristic->handle;

      auto status = esp_ble_gattc_register_for_notify(client->get_gattc_if(), client->get_remote_bda(),
                                                      characteristic->handle);
      if (status) {
        ESP_LOGW(TAG, "esp_ble_gattc_register_for_notify failed, status=%d", status);
      }
      break;
    }
    case ESP_GATTC_REG_FOR_NOTIFY_EVT: {
      if (param->reg_for_notify.handle != this->char_handle_) return;
      if (param->reg_for_notify.status != ESP_GATT_OK) {
        ESP_LOGW(TAG, "Error registering for notifications at handle %d, status=%d",
                 param->reg_for_notify.handle, param->reg_for_notify.status);
        return;
      }
      // 非 V3 模式下，BLEClientBase 会在 REG_FOR_NOTIFY_EVT 里查找 CCCD 并写入 notify enable。
      this->subscribed_ = true;
      this->node_state = espbt::ClientState::ESTABLISHED;
      ESP_LOGI(TAG, "Registered FOO notify (handle=%u)", this->char_handle_);
      break;
    }
    case ESP_GATTC_NOTIFY_EVT: {
      if (param->notify.handle != this->char_handle_) return;
      this->handle_frame_(param->notify.value, param->notify.value_len);
      break;
    }
    case ESP_GATTC_READ_CHAR_EVT: {
      if (param->read.handle != this->char_handle_) return;
      if (param->read.status == ESP_GATT_OK)
        this->handle_frame_(param->read.value, param->read.value_len);
      break;
    }
    case ESP_GATTC_DISCONNECT_EVT:
      this->subscribed_ = false;
      this->char_handle_ = 0;
      this->node_state = espbt::ClientState::IDLE;
      break;
    default:
      break;
  }
}

void FooBeaconSensor::handle_frame_(const uint8_t *data, size_t len) {
  if (len < 4) {
    ESP_LOGW(TAG, "frame too short: %u", len);
    return;
  }
  // [seq, temp_lo, temp_hi, bat]
  int16_t temp_raw = (int16_t)((data[2] << 8) | data[1]);
  float temp_c = temp_raw / 100.0f;
  uint8_t bat = data[3];

  this->publish_state(temp_c);
  ESP_LOGD(TAG, "seq=%u temp=%.2f°C bat=%u%%", data[0], temp_c, bat);
}

}  // namespace esphome::foo_beacon

#endif  // USE_ESP32
```

### 4.8 yaml 配置示例

```yaml
# 把 external_components 引入本地路径
external_components:
  - source:
      type: local
      path: custom_components

esp32:
  board: esp32dev
  framework:
    type: esp-idf     # 推荐用 ESP-IDF 而非 Arduino，BLE 更稳定

esp32_ble_tracker:
  scan_parameters:
    interval: 30ms
    window: 30ms
    active: true

ble_client:
  - mac_address: AA:BB:CC:DD:EE:FF
    id: my_foo_beacon

sensor:
  - platform: foo_beacon
    ble_client_id: my_foo_beacon
    name: "FooBeacon Temperature"
    update_interval: 60s
    # 本最小示例只把温度作为 sensor 发布；bat 字段在 C++ 中仅记录日志
```

### 4.9 关键工程要点

1. **`USE_ESP32` 守卫**：所有 BLE 代码必须用 `#ifdef USE_ESP32` 包裹，否则非 ESP32 平台编译失败。
2. **`BLEClientNode` 注册流程**：Python 端调用 `ble_client.register_ble_node(var, config)`，codegen 把该 node 注册到父 `BLEClient` 的节点列表；C++ 端 node 通过 `gattc_event_handler()`、`gap_event_handler()`、`loop()` 接收生命周期事件。当前 `BLEClientNode` 没有 `node_connected()` 虚函数，连接后初始化应放在 `ESP_GATTC_SEARCH_CMPL_EVT` 分支。
3. **notify 订阅流程**：node 调用 `esp_ble_gattc_register_for_notify()` 注册 notify；随后在 `ESP_GATTC_REG_FOR_NOTIFY_EVT` 中确认注册结果。非 V3 模式下，`BLEClientBase` 会在该事件里查找 CCCD 并写入 enable 值。
4. **并发与重入**：所有 GATT 操作都是异步回调，**禁止在 handler 中同步等待**。状态用成员变量保存，事件驱动推进。
5. **断线重连**：断线时把 `subscribed_`、`char_handle_`、`node_state` 复位；等待下一轮 `ESP_GATTC_SEARCH_CMPL_EVT` 重新发现 characteristic 并注册 notify。
6. **资源约束**：ESP32 BLE 同时连接数有限（典型 3 个），多个 beacon 监听会受 active BLE client 连接槽位限制。低数据率场景优先考虑“广播+广播解析”（像 BTHome 那样），不要轻易走 GATT 路径。
7. **日志裁剪**：嵌入式 flash 紧张，用 `ESP_LOGV/D/W/E` 分级，发布版本关掉 `V`。
8. **yaml 校验**：用 `ble_client.BLE_CLIENT_SCHEMA` / `cv.use_id` 拿到 `BLEClient` 引用，避免硬编码字符串。

---

## 五、总结

| 维度 | 协议层 | MicroPython aioble | ESPHome C++ |
|------|--------|-------------------|-------------|
| 抽象层级 | LL/HCI/ATT/GATT | asyncio 协程 + 对象模型 | Component + 事件回调状态机 |
| 典型代码量 | 数百行 | 十几行 | 几十至上百行 |
| 适用场景 | 调试/底层 | 原型/教学/小型项目 | 产品级固件、量产 |

- **协议层**：BLE = Controller(LL/PHY) + HCI + LE Host(L2CAP/ATT/GATT/SMP/GAP)，40 信道、连接角色、GATT 数据模型是核心三件套。
- **aioble**：把“ADV / Scan Response 拼装、扫描合并、连接、UUID 发现、CCCD 订阅、indicate 确认等待”包装成 `awaitable`，是 MicroPython 上 BLE 的事实标准。
- **ESPHome**：
  - `esp32_ble_tracker` 是“广播路由器”，所有监听者继承 `ESPBTDeviceListener`；listener 可声明 `PARSED` / `RAW` 需求，tracker 全局可同时启用两条路径；
  - `bthome_mithermometer` 等是“广播型传感器”的标准范例：`parse_device` → 匹配 mac → 解 service_data → `publish_state`；
  - `bluetooth_proxy` 是“远程 BLE 网关”：raw advertisements 原样转发 + 主动 GATT 代理，用 V3 缓存协议省内存；
  - `esp32_ble_client/ble_client_base` 是“通用 GATT 客户端状态机”，处理 app 注册、连接、MTU、服务发现、lazy characteristic/descriptor 解析、notify 注册、读写和断线；
- **自定义组件**：对于“必须建连才能拿数据”的信标，继承 `BLEClientNode`，在 `gattc_event_handler()` 的 `ESP_GATTC_SEARCH_CMPL_EVT` 里发现特征并注册 notify，在 `ESP_GATTC_REG_FOR_NOTIFY_EVT` 里进入 `ESTABLISHED`，在 `ESP_GATTC_NOTIFY_EVT` 里解析帧并 `publish_state`。

ESPHome 之美在于"组件即配置"——把一套复杂的 BLE GATT 状态机封装成 yaml 几行配置；ESPHome 之难在于"事件驱动 + 内存约束"——任何想自研 BLE 组件的人，都必须先理解 `BLEClientBase` 那个巨大的 `switch(event)`。掌握这套范式后，无论信标是广播型还是 GATT 型，都能在 1~2 个文件内把它接入 ESPHome。

---

## 参考

- Bluetooth Core Specification 5.x
- MicroPython aioble：<https://github.com/micropython/micropython-lib/blob/master/micropython/bluetooth/aioble/README.md>
- ESPHome 源码 `esphome/components/esp32_ble_tracker/`
- ESPHome 源码 `esphome/components/esp32_ble_client/`
- ESPHome 源码 `esphome/components/bluetooth_proxy/`
- ESPHome 源码 `esphome/components/bthome_mithermometer/`
- BTHome 规范：<https://bthome.io/>
