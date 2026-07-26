---
date: '2026-07-21T10:00:00+08:00'
draft: false
title: 'Python BLE 库 Bleak 与 bleak-esphome 实战详解'
author: 'synodriver'
tags: ["python", "ble", "bluetooth", "esphome", "home-assistant", "iot"]
---

# Python BLE 库 Bleak 与 bleak-esphome 实战详解

---

## 一、为什么 Python 做 BLE 首选 Bleak

如果你在 Python 里做 BLE（Bluetooth Low Energy，低功耗蓝牙），很快会遇到一个现实问题：**不同操作系统的蓝牙协议栈完全不一样**。

- Windows 走 WinRT 蓝牙 API。
- Linux 通常走 BlueZ + D-Bus。
- macOS 走 CoreBluetooth，而且出于隐私原因不暴露真实 MAC 地址。
- Android / python-for-android 又是另一套后端。

BLE 的业务模型看似统一：扫描广播、连接 Peripheral、发现 GATT service / characteristic / descriptor、读写 characteristic、订阅 notify / indicate。但底层实现差异巨大。Bleak 的价值就在这里：它把这些平台差异收敛成一套 `asyncio` 风格的 Python API。

Bleak 的全名是 **Bluetooth Low Energy platform Agnostic Klient**。当前本地环境里的版本是 `bleak 3.0.2`，包元数据声明它支持：

| 平台 | 后端 | 关键依赖 |
|------|------|----------|
| Windows | WinRT | `winrt-runtime`、`winrt-windows-devices-bluetooth*`、`winrt-windows-foundation*` 等 |
| Linux | BlueZ D-Bus | `dbus-fast>=1.83.0` |
| macOS | CoreBluetooth | `pyobjc-core`、`pyobjc-framework-corebluetooth`、`pyobjc-framework-libdispatch` |
| Android | python-for-android | Bleak 自带 p4android 后端；包元数据里的 `bleak-pythonista` 是 `pythonista` extra，不是普通 Android 必需依赖 |

它要求 Python `>=3.10`。如果再叠加 `bleak-esphome`，本地版本是 `bleak-esphome 3.9.7`，要求 Python `>=3.11`，并依赖：

```text
aioesphomeapi >= 45.3.1
bleak >= 3.0.2
bleak-retry-connector >= 4.6.1
bluetooth-data-tools >= 1.18.0
habluetooth >= 6.26.5
lru-dict >= 1.2.0
```

安装最简单：

```bash
pip install bleak
pip install bleak-esphome
```

> 千万不要把自己的脚本命名为 `bleak.py`，否则会遮蔽第三方包，导致循环导入或 `ImportError`。

---

## 二、Bleak 的核心对象模型

Bleak 顶层最重要的两个类是：

```python
from bleak import BleakScanner, BleakClient
```

- `BleakScanner`：负责扫描 BLE 广播包，发现设备。
- `BleakClient`：负责连接一个 GATT Server，并执行 GATT 读写、订阅通知、配对等操作。

BLE 里的角色要先分清：

| BLE 概念 | 在 Python / Bleak 里怎么理解 |
|----------|------------------------------|
| Peripheral | 外设，例如温湿度计、手环、ESP32 传感器、蓝牙锁 |
| Central | 主机，例如电脑、手机、Home Assistant 主机、运行 Python 的网关 |
| Advertisement | 外设周期性广播的小包，常带名称、service UUID、manufacturer data、service data |
| GATT Server | 外设提供的一组服务和特征值数据库 |
| GATT Client | 主机连接外设后读取、写入、订阅这些特征值 |
| Service | 服务，例如 Battery Service `0x180F`、Device Information `0x180A` |
| Characteristic | 特征值，例如 Battery Level `0x2A19`、Model Number `0x2A24` |
| Descriptor | 描述符，最常见的是 CCCD `0x2902`，用于启用 notify / indicate |

Bleak 的 API 是 `asyncio` 风格，所以程序入口通常是：

```python
import asyncio

async def main() -> None:
    ...

asyncio.run(main())
```

---

## 三、扫描设备：BleakScanner

### 3.1 一次性扫描

最短的扫描代码：

```python
import asyncio
from bleak import BleakScanner

async def main() -> None:
    devices = await BleakScanner.discover(timeout=5.0)
    for dev in devices:
        print(dev.address, dev.name, dev.details)

asyncio.run(main())
```

`BleakScanner.discover()` 会扫描指定秒数，默认返回 `list[BLEDevice]`。如果你还想拿到广播数据，可以加 `return_adv=True`：

```python
import asyncio
from bleak import BleakScanner

async def main() -> None:
    result = await BleakScanner.discover(timeout=5.0, return_adv=True)
    for address, (device, adv) in result.items():
        print("address:", address)
        print("device:", device)
        print("name:", adv.local_name)
        print("rssi:", adv.rssi)
        print("service_uuids:", adv.service_uuids)
        print("manufacturer_data:", adv.manufacturer_data)
        print("service_data:", adv.service_data)
        print()

asyncio.run(main())
```

`AdvertisementData` 常用字段包括：

| 字段 | 含义 |
|------|------|
| `local_name` | 广播名或 scan response 里的完整名 / 短名 |
| `manufacturer_data` | 厂商数据，key 是 Company Identifier，value 是 bytes |
| `service_data` | Service Data，key 是 service UUID，value 是 bytes |
| `service_uuids` | 广播里声明的服务 UUID 列表 |
| `rssi` | 信号强度 |
| `tx_power` | 发射功率，设备支持时才有 |
| `platform_data` | 后端相关的原始数据 |

### 3.2 按名称或地址找设备

Bleak 提供了便捷方法：

```python
import asyncio
from bleak import BleakScanner

async def main() -> None:
    dev = await BleakScanner.find_device_by_name("LYWSD03MMC", timeout=10.0)
    if dev is None:
        print("not found")
        return
    print(dev.address, dev.name)

asyncio.run(main())
```

按地址找：

```python
dev = await BleakScanner.find_device_by_address("AA:BB:CC:DD:EE:FF", timeout=10.0)
```

但要注意：**复杂程序里不要过度依赖地址字符串**。Bleak 源码里对 `BleakClient(address)` 有明确警告：

1. macOS 不暴露真实蓝牙地址，而是给每个设备分配 UUID。
2. 直接传地址会导致 `BleakClient.connect()` 内部隐式调用一次扫描；多设备并发连接时，这种隐式扫描很容易和你自己的扫描、其他连接流程互相干扰。

更稳妥的模式是：**先用 `BleakScanner` 找到 `BLEDevice`，再把 `BLEDevice` 传给 `BleakClient`**。

### 3.3 流式扫描：实时处理广播包

一次性 `discover()` 适合调试。长期运行的网关、数据采集程序更适合用异步上下文管理器：

```python
import asyncio
from bleak import BleakScanner, BleakClient, BleakGATTCharacteristic

async def main() -> None:
    async with BleakScanner() as scanner:
        async for device, adv in scanner.advertisement_data():
            print(device.address, adv.rssi, adv.local_name, adv.service_data, adv.manufacturer_data)
            if 16965 in adv.manufacturer_data:
                print("found target device:", device.address)
asyncio.run(main())
```

也可以用 detection callback：

```python
import asyncio
from bleak import BleakScanner
from bleak.backends.device import BLEDevice
from bleak.backends.scanner import AdvertisementData


def on_detect(device: BLEDevice, adv: AdvertisementData) -> None:
    if adv.local_name:
        print(device.address, adv.local_name, adv.rssi)

async def main() -> None:
    async with BleakScanner(detection_callback=on_detect):
        await asyncio.sleep(10)

asyncio.run(main())
```

源码里有一个细节很重要：第一次回调拿到的 advertisement 不一定包含 scan response。很多设备把完整名称放在 scan response 里，所以如果你按名称匹配，要等 `adv.local_name is not None`，不要看到第一包没有名字就判定不是目标设备。

### 3.4 主动扫描与被动扫描

`BleakScanner` 构造函数里有 `scanning_mode`：

```python
scanner = BleakScanner(scanning_mode="active")   # 默认
scanner = BleakScanner(scanning_mode="passive")
```

- `active`：扫描器会发 `SCAN_REQ` 请求 scan response，拿到更多数据，例如完整名称。
- `passive`：只听广播，不主动问设备，功耗和空口占用更低。

限制：macOS 不支持 passive scanning，如果传 `scanning_mode="passive"` 会抛 `BleakError`。

Linux / BlueZ 还可以传后端参数。Bleak 3.0 开始，旧的 `adapter="hci0"` 参数已废弃，应该写成：

```python
scanner = BleakScanner(bluez={"adapter": "hci0"})
```

---

## 四、连接设备：BleakClient

### 4.1 读取一个标准特征值

以 Device Information Service 的 Model Number `0x2A24` 为例：

```python
import asyncio
from bleak import BleakClient, BleakScanner

MODEL_NUMBER_UUID = "2A24"

async def main() -> None:
    device = await BleakScanner.find_device_by_name("MyDevice", timeout=10.0)
    if device is None:
        raise RuntimeError("device not found")

    async with BleakClient(device) as client:
        data = await client.read_gatt_char(MODEL_NUMBER_UUID)
        print(data.decode(errors="replace"))

asyncio.run(main())
```

`BleakClient` 支持异步上下文管理器：进入时自动 `connect()`，退出时自动 `disconnect()`。这比手动连接再手动断开更安全，尤其是脚本异常退出时。

### 4.2 查看 services / characteristics / descriptors

连接后可以遍历 GATT 数据库：

```python
import asyncio
from bleak import BleakClient, BleakScanner

async def main() -> None:
    device = await BleakScanner.find_device_by_name("MyDevice", timeout=10.0)
    if device is None:
        return

    async with BleakClient(device) as client:
        for service in client.services:
            print(f"[Service] {service.uuid} handle={service.handle} {service.description}")
            for char in service.characteristics:
                print(
                    f"  [Char] {char.uuid} handle={char.handle} "
                    f"props={char.properties} max_wwr={char.max_write_without_response_size}"
                )
                for desc in char.descriptors:
                    print(f"    [Desc] {desc.uuid} handle={desc.handle}")

asyncio.run(main())
```

`BleakGATTServiceCollection` 支持用 UUID 或 handle 查找 service / characteristic / descriptor。但源码里也有一个非常实用的保护：如果多个 characteristic 拥有同一个 UUID，直接用 UUID 查找会抛 `BleakError`，提示你改用 handle 或 `BleakGATTCharacteristic` 对象。很多国产 BLE 模块或者私有协议设备确实会复用 UUID，所以生产代码最好在服务发现阶段把目标 characteristic 对象保存下来。

```python
target = None
for service in client.services:
    for char in service.characteristics:
        if char.uuid == "0000xxxx-0000-1000-8000-00805f9b34fb" and char.handle == 37:
            target = char

if target is None:
    raise RuntimeError("target characteristic not found")

data = await client.read_gatt_char(target)
```

### 4.3 写 characteristic：response 参数必须认真选

Bleak 的 `write_gatt_char()` 支持两种 BLE 写操作：

| 写法 | BLE 语义 | 特点 |
|------|----------|------|
| `response=True` | Write Request | 外设要回 ATT Write Response，可靠，但慢一点 |
| `response=False` | Write Command | 不等待响应，吞吐更高，但应用层不知道外设是否处理 |

示例：

```python
await client.write_gatt_char(WRITE_UUID, b"\x01\x02\x03", response=True)
await client.write_gatt_char(WRITE_UUID, b"\x04\x05", response=False)
```

Bleak 3.0 源码里保留了 `response=None` 的兼容逻辑：如果不传，它会根据 characteristic properties 优先选择 `write`。但注释也说了：有些设备的 properties 会乱报或漏报，所以推荐你显式传 `response=True/False`。

长度限制也不一样：

- write with response 通常单包最多 512 字节，这是 ATT 层限制。
- write without response 受 `char.max_write_without_response_size` 限制，通常是 `MTU - 3`。
- 默认 ATT MTU 是 23，所以默认 payload 常见是 20 字节。
- MTU 协商后可以更大，但不同 OS 后端暴露方式不同。Bleak 源码特别提示：BlueZ 后端的 `client.mtu_size` 总是返回 23；更推荐看 `BleakGATTCharacteristic.max_write_without_response_size`。

一个稳妥的分包写函数：

```python
async def write_chunks(client, char, payload: bytes, *, response: bool = False):
    if response:
        # ATT Write Request 理论最大 512；很多设备应用层协议会更小。
        chunk_size = 512
    else:
        chunk_size = char.max_write_without_response_size

    for offset in range(0, len(payload), chunk_size):
        await client.write_gatt_char(char, payload[offset:offset + chunk_size], response=response)
```

### 4.4 订阅 notify / indicate

BLE 里订阅通知的本质是写 CCCD `0x2902`：

- `0x0001`：启用 notify。
- `0x0002`：启用 indicate。
- `0x0000`：关闭。

但 Bleak 不允许用户直接写 CCCD。源码里 `write_gatt_descriptor()` 如果发现 descriptor UUID 是 `0x2902`，会抛：

```text
ValueError: Cannot write to CCCD (0x2902) directly. Use start_notify() or stop_notify() instead.
```

正确写法是：

```python
import asyncio
from bleak import BleakClient, BleakScanner, BleakGATTCharacteristic

NOTIFY_UUID = "0000xxxx-0000-1000-8000-00805f9b34fb"


def on_notify(sender: BleakGATTCharacteristic, data: bytearray) -> None:
    print("notify from", sender.uuid, "handle", sender.handle, "data", data.hex())

async def main() -> None:
    device = await BleakScanner.find_device_by_name("MyDevice", timeout=10.0)
    if device is None:
        return

    async with BleakClient(device) as client:
        await client.start_notify(NOTIFY_UUID, on_notify)
        await asyncio.sleep(30)
        await client.stop_notify(NOTIFY_UUID)

asyncio.run(main())
```

Bleak 0.18 以后，callback 的第一个参数是 `BleakGATTCharacteristic`，不是旧版本里的 handle int。callback 也可以是 async function：

```python
async def on_notify(sender: BleakGATTCharacteristic, data: bytearray) -> None:
    await save_to_db(sender.uuid, bytes(data))
```

Bleak 内部会 `asyncio.create_task()` 调用它，并维护一个 `_background_tasks` set 防止任务被垃圾回收。

### 4.5 把 notify 转成 async iterator

回调好用，但复杂业务里我更喜欢把 notify 转成队列：

```python
import asyncio
from bleak import BleakClient, BleakScanner

NOTIFY_UUID = "0000xxxx-0000-1000-8000-00805f9b34fb"

async def main() -> None:
    device = await BleakScanner.find_device_by_name("MyDevice", timeout=10.0)
    if device is None:
        return

    q: asyncio.Queue[bytes] = asyncio.Queue()

    def on_notify(sender: BleakGATTCharacteristic, data: bytearray) -> None:
        q.put_nowait(bytes(data))

    async with BleakClient(device) as client:
        await client.start_notify(NOTIFY_UUID, on_notify)
        try:
            while True:
                packet = await asyncio.wait_for(q.get(), timeout=10.0)
                print(packet.hex())
        finally:
            await client.stop_notify(NOTIFY_UUID)

asyncio.run(main())
```

这样做有几个好处：

- callback 保持极短，不在蓝牙回调路径里做慢操作。
- 上层可以用 `asyncio.wait_for()` 做超时。
- 可以自然接入解析器、状态机、数据库写入、MQTT 发布等异步流程。

---

## 五、典型 BLE 私有协议客户端写法

很多 IoT 设备的协议长这样：

1. 先扫描，按名称、service UUID、manufacturer data 或 service data 找设备。
2. 连接。
3. 找到 write characteristic 和 notify characteristic。
4. 订阅 notify。
5. 发送一条命令。
6. 等 notify 返回应答。
7. 解析二进制协议。

下面是一个完整骨架：

```python
import asyncio
from dataclasses import dataclass
from bleak import BleakClient, BleakScanner
from bleak.backends.characteristic import BleakGATTCharacteristic
from bleak.backends.device import BLEDevice
from bleak.backends.scanner import AdvertisementData

SERVICE_UUID = "0000fff0-0000-1000-8000-00805f9b34fb"
WRITE_UUID = "0000fff1-0000-1000-8000-00805f9b34fb"
NOTIFY_UUID = "0000fff2-0000-1000-8000-00805f9b34fb"

@dataclass
class DeviceChars:
    write: BleakGATTCharacteristic
    notify: BleakGATTCharacteristic


def build_command(cmd: int, payload: bytes = b"") -> bytes:
    # 示例帧：AA 55 LEN CMD PAYLOAD XOR
    body = bytes([cmd]) + payload
    length = len(body)
    checksum = 0
    for b in body:
        checksum ^= b
    return b"\xaa\x55" + bytes([length]) + body + bytes([checksum])


def parse_response(data: bytes) -> dict[str, int | bytes]:
    if len(data) < 5 or data[:2] != b"\xaa\x55":
        raise ValueError(f"bad frame: {data.hex()}")
    length = data[2]
    body = data[3:3 + length]
    checksum = data[3 + length]
    calc = 0
    for b in body:
        calc ^= b
    if checksum != calc:
        raise ValueError("bad checksum")
    return {"cmd": body[0], "payload": body[1:]}


async def find_target() -> BLEDevice | None:
    def match(device: BLEDevice, adv: AdvertisementData) -> bool:
        return SERVICE_UUID in adv.service_uuids or adv.local_name == "MySensor"

    return await BleakScanner.find_device_by_filter(match, timeout=10.0)


def resolve_chars(client: BleakClient) -> DeviceChars:
    write = client.services.get_characteristic(WRITE_UUID)
    notify = client.services.get_characteristic(NOTIFY_UUID)
    if write is None or notify is None:
        raise RuntimeError("required characteristics not found")
    return DeviceChars(write=write, notify=notify)


async def main() -> None:
    device = await find_target()
    if device is None:
        raise RuntimeError("device not found")

    q: asyncio.Queue[bytes] = asyncio.Queue()

    def on_notify(sender: BleakGATTCharacteristic, data: bytearray) -> None:
        q.put_nowait(bytes(data))

    async with BleakClient(device, timeout=30.0) as client:
        chars = resolve_chars(client)
        await client.start_notify(chars.notify, on_notify)

        command = build_command(0x01)
        await client.write_gatt_char(chars.write, command, response=True)

        raw = await asyncio.wait_for(q.get(), timeout=5.0)
        print(parse_response(raw))

asyncio.run(main())
```

这套骨架比“直接传 MAC 地址 + sleep + 全局变量接收 notify”的脚本更稳，尤其适合长期运行的网关程序。

---

## 六、实战：解析 EWD104-BT58 系列 BLE 信标扫描响应

下面用成都亿佰特 EWD104-BT58 系列信标做一个纯扫描解析例子。手册位于 `E:\EWD104-BT58系列用户手册.pdf`，这里参考的是 **5.3 BLE 扫描回复数据包**，忽略 AT 指令章节。

### 6.1 先澄清：手册里的 `Type=0xFF` 不是标准 Service Data

手册把扫描回复中的第二段写成：

```text
Len Type Data
... 0xFF ...
```

并在文字里称作 `service data（Type：0xFF）`。这里要分开看：**手册的叫法**和**BLE 标准里的 AD Type 定义**不是一回事。按 BLE 规范：

- `0x16` 才是 Service Data - 16-bit UUID。
- `0x20` 是 Service Data - 32-bit UUID。
- `0x21` 是 Service Data - 128-bit UUID。
- `0xFF` 是 Manufacturer Specific Data。

也就是说，这份手册里的这段 `Type=0xFF` 数据，从 BLE 标准角度更应当看作 **Manufacturer Specific Data**。在 Bleak 里，这类数据通常会出现在：

```python
adv.manufacturer_data
```

而不是：

```python
adv.service_data
```

这一点很关键。否则你一直盯着 `adv.service_data`，可能会误以为设备没有上报传感器数据。

### 6.2 手册中的扫描响应格式

手册 5.3 描述了两类扫描响应。

#### 无传感器版本

无传感器设备的响应包第二段为 `Type=0xFF`，手册示意字段为：

| 字段 | 长度 | 说明 |
|------|------|------|
| UUID / 厂商字段 | 2 字节 | 手册示例 `0x1803` |
| MAC | 6 字节 | 设备 MAC 地址 |
| Major | 2 字节 | iBeacon Major |
| Minor | 2 字节 | iBeacon Minor |
| 发射功率索引值 | 1 字节 | 例如 `0x06` |
| 广播间隔 | 2 字节 | 例如 `0x03E8` 表示 1000ms |
| 电池电量百分比 | 1 字节 | 例如 `0x51` 表示 81% |

#### 带传感器版本

带传感器设备的 `Type=0xFF` 数据字段示例为：

```text
45 42 13 00 FA C0 E4 80 39 C0 16 24 10 34 51
```

手册解析为：

| 字段 | 字节 | 示例 | 说明 |
|------|------|------|------|
| 厂商 ID | 2 | `45 42` | 手册写作 `0x4542` |
| 可连接标识 | 1 | `13` | `0x13` 可连接，`0x12` 不可连接 |
| 加密标识 | 1 | `00` | `0` 未加密，`1` 加密 |
| X 轴原始值 | 2 | `FA C0` | 高字节在前，按有符号 16 位解析更实用 |
| Y 轴原始值 | 2 | `E4 80` | 同上 |
| Z 轴原始值 | 2 | `39 C0` | 同上 |
| 温度 | 2 | `16 24` | 整数部分 `0x16=22`，小数部分 `0x24=36`，即 22.36℃ |
| 湿度 | 2 | `10 34` | 整数部分 `0x10=16`，小数部分 `0x34=52`，即 16.52%RH |
| 电池电量 | 1 | `51` | `0x51=81`，即 81% |

手册还特别说明：

- 温湿度第一个字节为整数部分，第二个字节为小数部分。
- 环境温度为负数时，温度数据最高位为 1，表示负温，其余位采用补码表示。
- 加速度传感器数据高字节在前、低字节在后。
- 若无对应传感器，对应位置数据为 0。

### 6.3 一个可直接跑的 Bleak 扫描解析脚本

下面脚本只依赖 Bleak，不需要连接设备，只解析扫描响应 / 广播里的 manufacturer data。

```python
import asyncio
from dataclasses import dataclass
from bleak import BleakScanner
from bleak.backends.device import BLEDevice
from bleak.backends.scanner import AdvertisementData


# 手册把厂商 ID 写成 0x4542，对应原始字节 45 42。
# BLE Manufacturer Specific Data 规范里的 Company Identifier 是 little-endian；
# 不同后端交给 Bleak 时通常已经剥离 company id，并把它作为 dict key。
# 为了兼容，这里同时接受 0x4542 和 0x4245。
EWD_COMPANY_IDS = {0x4542, 0x4245}


@dataclass(slots=True)
class EWD104BT58SensorFrame:
    address: str
    name: Optional[str]
    rssi: int
    company_id: int
    connectable: bool
    encrypted: bool
    accel_x_raw: int
    accel_y_raw: int
    accel_z_raw: int
    temperature_c: float
    humidity_percent: float
    battery_percent: int
    raw: bytes


def _parse_signed_be16(data: bytes) -> int:
    return int.from_bytes(data, byteorder="big", signed=True)


def _parse_temperature(integer_byte: int, fraction_byte: int) -> float:
    """解析手册中的温度格式。

    正温：0x16 0x24 -> 22.36℃。
    负温：整数部分最高位为 1；按 8 位补码解释整数部分，小数部分作为百分位。
    例如整数字节 0xEA 表示 -22，则 0xEA 0x24 -> -22.36℃。
    """
    fraction = fraction_byte / 100.0
    if integer_byte & 0x80:
        integer = integer_byte - 0x100
        return integer - fraction
    return integer_byte + fraction


def _parse_humidity(integer_byte: int, fraction_byte: int) -> float:
    return integer_byte + fraction_byte / 100.0


def _looks_like_sensor_payload(payload: bytes) -> bool:
    """判断是否像 EWD104-BT58 带传感器格式。

    Bleak 的 manufacturer_data value 通常不包含前 2 字节 company id，长度为 13：
        13 00 FA C0 E4 80 39 C0 16 24 10 34 51

    如果你从原始 AD 结构里自己剥数据，可能会拿到含 company id 的 15 字节：
        45 42 13 00 FA C0 E4 80 39 C0 16 24 10 34 51
    """
    if len(payload) == 13:
        flag = payload[0]
        encrypted = payload[1]
        battery = payload[12]
    elif len(payload) == 15 and payload[:2] == b"\x45\x42":
        flag = payload[2]
        encrypted = payload[3]
        battery = payload[14]
    else:
        return False

    return flag in (0x12, 0x13) and encrypted in (0x00, 0x01) and battery <= 100


def parse_ewd104_bt58_sensor_frame(device: BLEDevice, adv: AdvertisementData) -> EWD104BT58SensorFrame | None:
    """从 Bleak 的 AdvertisementData 中解析 EWD104-BT58 带传感器扫描响应。"""
    for company_id, payload in adv.manufacturer_data.items():
        # 常见 Bleak 表示：company_id 作为 key，payload 不再包含 company id。
        if company_id in EWD_COMPANY_IDS and _looks_like_sensor_payload(payload):
            return _parse_sensor_payload(
                address=device.address,
                name=adv.local_name or device.name,
                rssi=adv.rssi,
                company_id=company_id,
                payload=payload,
                raw=b"\x45\x42" + payload,
            )

        # 保险：有些自定义扫描栈可能把 45 42 留在 value 里。
        if _looks_like_sensor_payload(payload):
            return _parse_sensor_payload(
                address=device.address,
                name=adv.local_name or device.name,
                rssi=adv.rssi,
                company_id=company_id,
                payload=payload,
                raw=payload,
            )

    return None


def _parse_sensor_payload(
    *,
    address: str,
    name: Optional[str],
    rssi: int,
    company_id: int,
    payload: bytes,
    raw: bytes,
) -> EWD104BT58SensorFrame:
    if len(payload) == 15:
        # 原始数据包含 45 42 厂商 ID 时，先剥掉。
        company_id = int.from_bytes(payload[0:2], "big")
        payload = payload[2:]

    if len(payload) != 13:
        raise ValueError(f"unexpected EWD104-BT58 sensor payload length: {len(payload)}")

    connect_flag = payload[0]
    encrypted_flag = payload[1]
    accel_x = _parse_signed_be16(payload[2:4])
    accel_y = _parse_signed_be16(payload[4:6])
    accel_z = _parse_signed_be16(payload[6:8])
    temperature = _parse_temperature(payload[8], payload[9])
    humidity = _parse_humidity(payload[10], payload[11])
    battery = payload[12]

    return EWD104BT58SensorFrame(
        address=address,
        name=name,
        rssi=rssi,
        company_id=company_id,
        connectable=connect_flag == 0x13,
        encrypted=encrypted_flag == 0x01,
        accel_x_raw=accel_x,
        accel_y_raw=accel_y,
        accel_z_raw=accel_z,
        temperature_c=temperature,
        humidity_percent=humidity,
        battery_percent=battery,
        raw=raw,
    )


def on_detect(device: BLEDevice, adv: AdvertisementData) -> None:
    frame = parse_ewd104_bt58_sensor_frame(device, adv)
    if frame is None:
        return

    print(
        f"{frame.address} {frame.name or ''} RSSI={frame.rssi} "
        f"connectable={frame.connectable} encrypted={frame.encrypted} "
        f"accel=({frame.accel_x_raw},{frame.accel_y_raw},{frame.accel_z_raw}) "
        f"temp={frame.temperature_c:.2f}℃ "
        f"humidity={frame.humidity_percent:.2f}%RH "
        f"battery={frame.battery_percent}% "
        f"raw={frame.raw.hex()}"
    )


async def main() -> None:
    async with BleakScanner(detection_callback=on_detect, scanning_mode="active"):
        # 扫描响应通常需要 active scan 才更容易拿全。
        await asyncio.sleep(30)


if __name__ == "__main__":
    asyncio.run(main())
```

运行后，如果设备上报了手册示例中的数据：

```text
45 42 13 00 FA C0 E4 80 39 C0 16 24 10 34 51
```

解析结果应当是：

```text
company_id      = 0x4542
connectable     = True
encrypted       = False
accel_x_raw     = -1344
accel_y_raw     = -7040
accel_z_raw     = 14784
temperature_c   = 22.36
humidity_percent= 16.52
battery_percent = 81
```

这里的加速度值我保留为 `raw`，没有换算成 g。原因是手册只说“陀螺仪/加速度传感器原始数据，高字节在前，低字节在后”，没有给出量程、LSB/g 或姿态算法。贸然换算成 m/s² 反而容易误导。

### 6.4 如果你想同时兼容“无传感器版本”

无传感器版本字段更像传统 iBeacon 参数回显：MAC、Major、Minor、发射功率索引、广播间隔、电池百分比。可以单独写一个解析函数：

```python
@dataclass(slots=True)
class EWD104BT58PlainFrame:
    mac: str
    major: int
    minor: int
    tx_power_index: int
    adv_interval_ms: int
    battery_percent: int


def parse_plain_payload(payload: bytes) -> EWD104BT58PlainFrame | None:
    # Bleak manufacturer_data 的 value 通常已经去掉 2 字节 company id，
    # 无传感器格式剩余长度应为：MAC 6 + Major 2 + Minor 2 + Power 1 + Interval 2 + Battery 1 = 14。
    if len(payload) == 16:
        # 如果还带着 0x18 0x03 这类前缀，先剥掉。
        payload = payload[2:]
    if len(payload) != 14:
        return None

    mac = ":".join(f"{b:02X}" for b in payload[0:6])
    major = int.from_bytes(payload[6:8], "big")
    minor = int.from_bytes(payload[8:10], "big")
    tx_power_index = payload[10]
    adv_interval_ms = int.from_bytes(payload[11:13], "big")
    battery_percent = payload[13]

    return EWD104BT58PlainFrame(
        mac=mac,
        major=major,
        minor=minor,
        tx_power_index=tx_power_index,
        adv_interval_ms=adv_interval_ms,
        battery_percent=battery_percent,
    )
```

不过在实际项目里，我建议把“带传感器”和“无传感器”当成两个不同 payload schema 来处理，不要写一个过度聪明的万能解析器。信标类设备的广播格式往往会随配置模式变化，比如 iBeacon / Eddystone / 自定义包，强行合并只会让误判概率上升。

---

## 七、跨平台坑点与排障经验

### 7.1 macOS 的地址不是 MAC

CoreBluetooth 不给应用真实蓝牙 MAC，Bleak 里 `device.address` 会是一个 UUID。不要把 macOS 扫出来的 `address` 当物理地址写进数据库，然后拿去 Linux / Windows 上复用。

更好的设备标识方式：

- 广播 service UUID。
- manufacturer data 里的设备序列号。
- service data 里的设备 ID。
- GATT Device Information Service 里的序列号。

### 7.2 扫描和连接不能无限并发

Bleak 源码提醒：蓝牙适配器能否“边扫边连”、最多同时连几个设备，取决于硬件和 OS 后端。廉价 USB 蓝牙 dongle、树莓派板载蓝牙、笔记本蓝牙芯片差异很大。

实践建议：

- 多设备连接前，先扫描收集 `BLEDevice`，再停止扫描，再分批连接。
- 用 `asyncio.Semaphore` 限制并发连接数。
- 对连接、读写、notify 等操作加超时。

```python
sem = asyncio.Semaphore(3)

async def poll_device(device: BLEDevice) -> bytes:
    async with sem:
        async with BleakClient(device, timeout=20.0) as client:
            return await client.read_gatt_char("2A19")
```

### 7.3 Linux 权限与 BlueZ 版本

Linux 上 Bleak 依赖 BlueZ 和 D-Bus。常见问题：

- 用户没有访问蓝牙设备或 D-Bus 权限。
- BlueZ 版本太旧。Bleak 元数据里写的是 Linux 需要 BlueZ `>=5.55`。
- 适配器被 NetworkManager、桌面蓝牙管理器或其他进程占用。
- Docker 容器里没有正确挂载 D-Bus 和蓝牙设备。

检查命令：

```bash
bluetoothctl show
bluetoothctl scan on
hciconfig -a
```

### 7.4 Windows 配对与缓存

Windows 后端走 WinRT。某些设备如果 characteristic 需要加密，必须先配对。BleakClient 支持：

```python
async with BleakClient(device, pair=True, timeout=60.0) as client:
    ...
```

源码提示：开启 `pair=True` 时，要把 timeout 设长一点，因为用户可能需要输入 PIN 或确认弹窗。macOS 不支持手动 `pair()`；访问需要认证的 characteristic 时，系统会自动弹配对提示。

Windows 和 Linux 还支持 `unpair()`，其他平台会抛异常。

### 7.5 GATT 缓存问题

很多 BLE 调试中最诡异的问题来自 GATT 缓存：设备固件升级后 service / characteristic 变了，但系统还拿旧缓存。

排障套路：

- 改设备 MAC 或清系统蓝牙缓存。
- Windows 删除已配对设备再重新扫描。
- Linux 用 `bluetoothctl remove XX:XX:XX:XX:XX:XX`。
- 如果设备支持 Service Changed characteristic，正确实现它。
- 在应用层不要假设 handle 永远不变，优先按 UUID / 服务结构重新发现。

---

## 八、bleak-esphome 是什么

`bleak-esphome` 不是一个“解析 ESPHome 传感器数据”的库，而是一个 **Bleak backend of ESPHome**：它让运行 ESPHome Bluetooth Proxy 的 ESP32 变成远程 BLE 适配器，Python 程序通过 ESPHome Native API 操作这块远程适配器。

官方包描述里的核心句子可以翻译为：

> 它让任何 Python 应用——Home Assistant、加载项或独立脚本——使用远程 ESP32 作为适配器来发现和连接 BLE 外设。Bluetooth Proxy 固件在 ESPHome 项目里；这个库是主机侧客户端。

架构上可以这么理解：

```text
Python 应用
  │
  │ Bleak API: BleakScanner / BleakClient
  │
bleak-esphome backend
  │
  │ ESPHome Native API / aioesphomeapi，默认 6053 端口，可带 noise_psk
  │
ESP32 + ESPHome bluetooth_proxy
  │
  │ 本地 BLE radio
  │
BLE 外设：温湿度计、门锁、手环、传感器……
```

也就是说，应用层仍然写 Bleak 的思维模型：扫描、连接、读、写、notify。区别是底层蓝牙控制器不再是电脑本机的蓝牙芯片，而是网络另一端的 ESP32。

这在 IoT 场景非常有价值：

- 服务器或 Home Assistant 主机可能在机柜里，蓝牙覆盖差。
- ESP32 可以贴近传感器安装，Wi-Fi 回传。
- 多个 ESPHome proxy 可以组成分布式 BLE 覆盖。
- Python 程序不需要关心 ESP-IDF / NimBLE 细节，还是写 Bleak 风格代码。

---

## 九、bleak-esphome 源码结构解读

本地 `bleak_esphome` 包主要文件：

```text
bleak_esphome/
  __init__.py
  connect.py
  connection_manager.py
  _cancellation.py
  backend/
    cache.py
    client.py
    device.py
    scanner.py
```

### 9.1 对外导出

`__init__.py` 只导出几个关键入口：

```python
from .connect import connect_scanner
from .connection_manager import (
    APIConnectionManager,
    ESPHomeDeviceConfig,
    ESPHomeStartAborted,
)
```

这说明 `bleak-esphome` 更像是给 Home Assistant / habluetooth 生态接入用的库，而不是一个“直接替换 `BleakScanner` 的命令行工具”。你可以直接用 `APIConnectionManager` 管理到 ESPHome 设备的 API 连接，也可以在已有 aioesphomeapi 连接里调用 `connect_scanner()`。

### 9.2 connect_scanner：把 ESPHome APIClient 注册成远程 scanner

`connect_scanner(cli, device_info, available)` 做了几件关键事：

1. 从 `device_info.bluetooth_mac_address` 或 `device_info.mac_address` 得到 source。
2. 根据 ESPHome API version 和 feature flags 判断 proxy 是否支持 active connections。
3. 创建 `ESPHomeBluetoothDevice`，维护连接槽位、可用状态、GATT cache。
4. 创建 `ESPHomeClientData`，把 APIClient、设备信息、scanner、disconnect callbacks 串起来。
5. 创建 `HaBluetoothConnector`，其 `client` 是 `partial(ESPHomeClient, client_data=client_data)`。
6. 创建 `ESPHomeScanner(source, name, connector, connectable)`。
7. 订阅 ESPHome 的广播流：
   - 新固件支持 `RAW_ADVERTISEMENTS` 时，订阅 raw advertisements。
   - 否则订阅结构化 `BluetoothLEAdvertisement`。
8. 如果支持 scanner state / mode，则订阅扫描状态，并允许运行时切换 active / passive / auto 模式。

关键点是第 5 步：`HaBluetoothConnector` 里注入的是 `ESPHomeClient`，所以当上层通过 habluetooth 分配这个远程 scanner 去连接某个 BLE 设备时，实际创建的是 `ESPHomeClient` 后端。

### 9.3 APIConnectionManager：独立程序更友好的连接管理

`APIConnectionManager` 接收配置：

```python
class ESPHomeDeviceConfig(TypedDict):
    address: str
    noise_psk: str | None
```

`start()` 里会创建：

```python
APIClient(
    address=self._address,
    port=6053,
    password=None,
    noise_psk=self._noise_psk,
)
```

然后交给 `aioesphomeapi.ReconnectLogic` 自动重连。首次连接成功后：

1. 调用 `device_info()`。
2. 调用 `bleak_esphome.connect_scanner()`。
3. 调用 `scanner.async_setup()`。
4. 调用 `habluetooth.get_manager().async_register_scanner(scanner)` 注册到 Home Assistant Bluetooth manager。

`stop()` 则会停止重连逻辑、断开 APIClient、取消首次启动 future、注销 scanner。

这就是一个比较适合独立 Python 服务的入口：你只负责启动 manager，底层会把 ESPHome proxy 挂进 habluetooth 的远程扫描器体系。

### 9.4 ESPHomeScanner：远程广播包适配器

`ESPHomeScanner` 继承 `habluetooth.base_scanner.BaseHaRemoteScanner`。它有两个广播入口：

```python
async_on_advertisement(self, adv: BluetoothLEAdvertisement)
async_on_raw_advertisements(self, raw: BluetoothLERawAdvertisementsResponse)
```

结构化 advertisement 会被转换成：

```python
self._async_on_advertisement(
    int_to_bluetooth_address(adv.address),
    adv.rssi,
    adv.name,
    adv.service_uuids,
    adv.service_data,
    adv.manufacturer_data,
    None,
    {"address_type": adv.address_type},
    MONOTONIC_TIME(),
)
```

raw advertisement 则保留原始广播 payload：

```python
self._async_on_raw_advertisement(
    int_to_bluetooth_address(adv.address),
    adv.rssi,
    adv.data,
    {"address_type": adv.address_type},
    now,
)
```

这也是 Home Assistant BLE 生态能解析 BTHome、MiBeacon、iBeacon、各种 manufacturer data 的基础：ESP32 只负责把广播包搬回来，主机侧用 Python 解析。

新版本还有 scanner mode 控制：

- `BluetoothScanningMode.ACTIVE` 映射到 ESPHome 固件 `ACTIVE`。
- `PASSIVE` 映射到 `PASSIVE`。
- `AUTO` 在固件上默认映射成 `PASSIVE`，需要主动窗口时短暂切到 `ACTIVE`。

源码里 `async_request_active_window(duration)` 会把 proxy 切到 active scan，一段时间后恢复。这个设计很适合低干扰运行：平时被动扫描省空口，需要补 scan response 时再主动扫描。

### 9.5 ESPHomeClient：把 BleakClient 操作翻译成 ESPHome API

`ESPHomeClient` 继承 `bleak.backends.client.BaseBleakClient`，是 bleak-esphome 的 GATT 客户端后端。它把 Bleak 的后端接口翻译成 aioesphomeapi 调用。

连接流程：

```python
await self._wait_for_free_connection_slot(CONNECT_FREE_SLOT_TIMEOUT)
await self._client.bluetooth_device_connect(...)
await self._get_services(...)
```

这里有两个很 IoT 的细节：

1. **连接槽位**：ESP32 同时能维持的 BLE 连接数有限。`ESPHomeBluetoothDevice` 会记录 `ble_connections_free`、`ble_connections_limit`、`ble_allocations`。如果没有空槽，`ESPHomeClient` 会等一小段时间，超时抛带设备名和槽位状态的 `TimeoutError`。
2. **GATT cache**：`ESPHomeBluetoothCache` 用 LRU 保存最多 128 个设备的 service collection 和 MTU。新固件支持 `REMOTE_CACHING` 时，ESP32 端可能为了省内存不保留完整服务列表，主机侧缓存就很重要。

服务发现 `_get_services()` 会把 ESPHome API 返回的 service / characteristic / descriptor 转成 Bleak 标准对象：

```python
BleakGATTService(...)
BleakGATTCharacteristic(...)
BleakGATTDescriptor(...)
BleakGATTServiceCollection()
```

也就是说，上层代码看到的仍然是 Bleak 的 `client.services`，不是 ESPHome 私有对象。

读写映射很直接：

| Bleak 后端方法 | ESPHome API 调用 |
|----------------|------------------|
| `read_gatt_char()` | `bluetooth_gatt_read(address, handle, timeout)` |
| `read_gatt_descriptor()` | `bluetooth_gatt_read_descriptor(address, handle, timeout)` |
| `write_gatt_char()` | `bluetooth_gatt_write(address, handle, bytes(data), response)` |
| `write_gatt_descriptor()` | `bluetooth_gatt_write_descriptor(address, handle, bytes(data))` |
| `start_notify()` | `bluetooth_gatt_start_notify(address, handle, callback)` |
| `disconnect()` | `bluetooth_device_disconnect(address)` |

错误处理上，`api_error_as_bleak_error` 会把 aioesphomeapi 的 `TimeoutAPIError`、`BluetoothGATTAPIError`、`BluetoothConnectionDroppedError`、`APIConnectionError` 转成 `TimeoutError` 或 `BleakError`，这样上层依旧按 Bleak 异常模型处理。

### 9.6 notify 与 CCCD

`ESPHomeClient.start_notify()` 先调用 ESPHome API 的 `bluetooth_gatt_start_notify()` 注册通知回调。对于支持 `REMOTE_CACHING` 的连接版本，还会自己找到 CCCD：

```python
cccd_descriptor = characteristic.get_descriptor(CCCD_UUID)
```

然后根据 characteristic properties 决定写入：

- notify：`b"\x01\x00"`
- indicate：`b"\x02\x00"`

这个逻辑和标准 Bleak 顶层 API 是一致的：应用层仍然只调用 `client.start_notify()`，不要手写 CCCD。

---

## 十、bleak 与 bleak-esphome 如何配合使用

### 10.1 最常见模式：Home Assistant 内部自动使用

在 Home Assistant 里，ESPHome Bluetooth Proxy、habluetooth、bleak-retry-connector、bleak-esphome 是一整套生态。典型流程是：

1. ESP32 刷 ESPHome，并启用 `bluetooth_proxy`。
2. Home Assistant 通过 ESPHome integration 连接 ESP32 Native API。
3. `bleak-esphome` 把这个 ESP32 注册为 remote scanner / connector。
4. Home Assistant Bluetooth manager 收到广播包。
5. 某个集成需要连接 BLE 设备时，habluetooth 根据信号、可连接性、槽位分配 proxy。
6. 上层仍然用 Bleak 风格客户端读写 GATT。

这种模式下你通常不直接 import `bleak_esphome`，但你写的 BLE 集成如果遵循 Bleak / habluetooth 的接口，就能自动受益于 ESPHome proxy。

### 10.2 独立 Python 程序：启动 ESPHome APIConnectionManager

如果你想写一个独立服务，连接某个 ESPHome proxy，可以从 `APIConnectionManager` 开始：

```python
import asyncio
from bleak_esphome import APIConnectionManager

async def main() -> None:
    manager = APIConnectionManager({
        "address": "192.168.1.50",
        "noise_psk": None,  # 如果 ESPHome API 配了 encryption key，这里填 noise_psk
    })
    await manager.start()
    try:
        print("ESPHome bluetooth proxy registered")
        await asyncio.Event().wait()
    finally:
        await manager.stop()

asyncio.run(main())
```

这段代码本身只是把 ESPHome proxy 注册进 habluetooth manager。真正的设备发现 / 连接通常由 habluetooth 生态分配 scanner / connector。对于已经使用 Home Assistant Bluetooth 抽象的项目，这种接入很自然。

### 10.3 应用层仍然写 Bleak 习惯

一旦远程 scanner / connector 被注册，上层 BLE 业务最好仍保持普通 Bleak 代码的形态：

```python
async def read_battery(device: BLEDevice) -> int:
    async with BleakClient(device, timeout=30.0) as client:
        data = await client.read_gatt_char("2A19")
        return data[0]
```

不要把业务逻辑写死成“必须本机蓝牙”或“必须某个 ESP32”。更好的分层是：

- 扫描 / 连接路由交给 Bluetooth manager / connector。
- GATT 协议解析写成纯 BleakClient 逻辑。
- BLE 设备身份识别依赖 service data / manufacturer data / GATT 信息，而不是某个平台特有的地址。

这样你的代码可以同时跑在：

- 笔记本本机 BLE。
- Linux 网关 USB dongle。
- Home Assistant + ESPHome Bluetooth Proxy。
- 多个 ESP32 proxy 组成的覆盖网络。

### 10.4 Home Assistant 源码中是如何使用 Bleak 的

Home Assistant 的 BLE 架构不是“每个集成自己 new 一个 `BleakScanner` 然后各扫各的”，而是把 Bleak 放在一个更高层的统一蓝牙管理器后面。源码路径基于 `E:\pyproject\core`。

#### 10.4.1 bluetooth 集成启动时创建统一 manager

入口在 `homeassistant/components/bluetooth/__init__.py`。`async_setup()` 里做了几件关键事：

```python
bluetooth_adapters = get_adapters()
bluetooth_storage = BluetoothStorage(hass)
slot_manager = BleakSlotManager()
integration_matcher = IntegrationMatcher(await async_get_bluetooth(hass))

manager = HomeAssistantBluetoothManager(
    hass, integration_matcher, bluetooth_adapters, bluetooth_storage, slot_manager
)
set_manager(manager)
await manager.async_setup()
```

这里有两个重点：

- `BleakSlotManager` 来自 `bleak_retry_connector`，负责管理 BLE 连接槽位，避免多个集成盲目并发连接把适配器或 ESPHome proxy 打满。
- `set_manager(manager)` 把 Home Assistant 自己的 `HomeAssistantBluetoothManager` 注册给 `habluetooth`，后续 scanner、connector、callback 都走这套全局 manager。

也就是说，Home Assistant 把本机蓝牙、远程 ESPHome proxy、历史广播缓存、连接槽位和集成匹配都收口到了 `HomeAssistantBluetoothManager`。

#### 10.4.2 async_get_scanner：给第三方库一个“看起来像 BleakScanner”的入口

`homeassistant/components/bluetooth/api.py` 里有：

```python
@hass_callback
def async_get_scanner(hass: HomeAssistant) -> BleakScanner:
    """Return a HaBleakScannerWrapper cast to BleakScanner."""
    return cast(BleakScanner, HaBleakScannerWrapper())
```

注释写得很直白：这是一个 `HaBleakScannerWrapper`，但为了兼容那些“参数类型写死成 `BleakScanner`”的第三方库，HA 把它 `cast(BleakScanner, ...)` 返回。

典型例子是 `homeassistant/components/acaia/coordinator.py`：

```python
self._scale = AcaiaScale(
    address_or_ble_device=entry.data[CONF_ADDRESS],
    name=entry.title,
    is_new_style_scale=entry.data[CONF_IS_NEW_STYLE_SCALE],
    notify_callback=debouncer.async_schedule_call,
    scanner=async_get_scanner(hass),
)
```

还有 `homeassistant/components/homekit_controller/utils.py`：

```python
bleak_scanner_instance = bluetooth.async_get_scanner(hass)

controller = Controller(
    async_zeroconf_instance=async_zeroconf_instance,
    bleak_scanner_instance=bleak_scanner_instance,
    char_cache=char_cache,
)
```

这种设计的意义是：第三方库以为自己拿到的是 Bleak scanner；但实际扫描数据来自 HA 的统一蓝牙管理器，能自动包含本机 adapter 和 ESPHome proxy。

#### 10.4.3 async_register_callback：广播数据先变成 BluetoothServiceInfoBleak

HA 集成通常不直接处理 Bleak 的 `(BLEDevice, AdvertisementData)`，而是处理 `BluetoothServiceInfoBleak`。

`homeassistant/components/bluetooth/models.py` 定义：

```python
type BluetoothCallback = Callable[[BluetoothServiceInfoBleak, BluetoothChange], None]
type ProcessAdvertisementCallback = Callable[[BluetoothServiceInfoBleak], bool]
```

`HomeAssistantBluetoothManager` 在 `manager.py` 里维护 callback matcher。当收到广告包后，核心路径在 `_discover_service_info()`：

```python
def _discover_service_info(self, service_info: BluetoothServiceInfoBleak) -> None:
    matched_domains = self._integration_matcher.match_domains(service_info)

    for match in self._callback_index.match_callbacks(service_info):
        callback = match[CALLBACK]
        callback(service_info, BluetoothChange.ADVERTISEMENT)

    for domain in matched_domains:
        discovery_flow.async_create_flow(
            self.hass,
            domain,
            {"source": config_entries.SOURCE_BLUETOOTH},
            service_info,
            discovery_key=discovery_key,
        )
```

所以 HA 的广播分发有两条线：

1. 调用已注册的 callback，让已加载集成更新数据。
2. 根据 manifest / matcher 匹配域，触发 config flow 自动发现新设备。

上层集成注册 callback 用的是 `api.py` 的：

```python
def async_register_callback(
    hass: HomeAssistant,
    callback: BluetoothCallback,
    match_dict: BluetoothCallbackMatcher | None,
    mode: BluetoothScanningMode,
    *,
    scan_interval: float | None = None,
    scan_duration: float | None = None,
) -> Callable[[], None]:
    return _get_manager(hass).async_register_callback(
        callback, match_dict, mode, scan_interval, scan_duration
    )
```

这比直接拿 BleakScanner 自己扫更适合 HA：matcher 可以指定 `address`、service UUID、manufacturer id、是否 connectable 等条件，并且还能配合 active scan 调度。

#### 10.4.4 async_process_advertisements：等待特定广播出现

对于 config flow 或迁移逻辑，HA 还提供了一个“等到匹配广播为止”的协程：

```python
async def async_process_advertisements(
    hass: HomeAssistant,
    callback: ProcessAdvertisementCallback,
    match_dict: BluetoothCallbackMatcher,
    mode: BluetoothScanningMode,
    timeout: int,
) -> BluetoothServiceInfoBleak:
    done: Future[BluetoothServiceInfoBleak] = hass.loop.create_future()

    def _async_discovered_device(
        service_info: BluetoothServiceInfoBleak, change: BluetoothChange
    ) -> None:
        if not done.done() and callback(service_info):
            done.set_result(service_info)

    with ExitStack() as stack:
        unload = manager.async_register_callback(
            _async_discovered_device, match_dict, mode
        )
        stack.callback(unload)

        if mode is BluetoothScanningMode.ACTIVE:
            task = hass.async_create_task(manager.async_request_active_scan(timeout))
            stack.callback(task.cancel)

        async with asyncio.timeout(timeout):
            return await done
```

`homeassistant/components/gardena_bluetooth/__init__.py` 就是这样用的：

```python
def _data_callback(info: BluetoothServiceInfoBleak) -> bool:
    if info.device.address != address:
        return False

    data.update(info.manufacturer_data.get(ManufacturerData.company, b""))
    return data.product_type is not ProductType.UNKNOWN

await bluetooth.async_process_advertisements(
    hass,
    _data_callback,
    bluetooth.BluetoothCallbackMatcher(
        address=address, manufacturer_id=ManufacturerData.company
    ),
    mode=bluetooth.BluetoothScanningMode.ACTIVE,
    timeout=PRODUCT_TYPE_TIMEOUT,
)
```

这和我们前面解析 EWD104-BT58 的思路一致：BLE 广播里的 manufacturer data 到 HA 里变成 `BluetoothServiceInfoBleak.manufacturer_data`，集成只关心解析业务字段，不直接管理 scanner 生命周期。

#### 10.4.5 async_ble_device_from_address：需要连接时再拿 BLEDevice

真正要 GATT 连接时，HA 才会从 manager 取 `BLEDevice`：

```python
@hass_callback
def async_ble_device_from_address(
    hass: HomeAssistant, address: str, connectable: bool = True
) -> BLEDevice | None:
    """Return BLEDevice for an address if its present."""
    return _get_manager(hass).async_ble_device_from_address(address, connectable)
```

例如 `homeassistant/components/bluetooth_le_tracker/device_tracker.py` 里读取电池电量：

```python
if service_info.connectable:
    device = service_info.device
elif connectable_device := bluetooth.async_ble_device_from_address(
    hass, service_info.device.address, True
):
    device = connectable_device
else:
    return

async with BleakClient(device) as client:
    bat_char = await client.read_gatt_char(BATTERY_CHARACTERISTIC_UUID)
    battery = ord(bat_char)
```

这体现了 HA 的推荐模式：

- 广播能解决的，只处理 `BluetoothServiceInfoBleak`。
- 必须连接时，优先使用 `service_info.device`，或者通过 `async_ble_device_from_address()` 找一个可连接路径。
- 最后才把 `BLEDevice` 交给 `BleakClient`。

`gardena_bluetooth` 也类似：它定义 `_device_lookup()`，内部通过 `bluetooth.async_ble_device_from_address(hass, address, connectable=True)` 找设备，然后交给第三方库的 `CachedConnection`。

#### 10.4.6 Passive / Active processor：HA 对“广播解析”和“连接补读”的封装

HA 还提供了两类更高层的 coordinator：

- `PassiveBluetoothProcessorCoordinator`：只靠广播更新实体。
- `ActiveBluetoothProcessorCoordinator`：先解析广播，如果需要再触发 GATT 轮询。

`passive_update_processor.py` 里的说明很清楚：`update_method` 每收到一次 `BluetoothServiceInfoBleak` 就返回解析后的数据，然后分发给实体。

```python
update_method: Callable[[BluetoothServiceInfoBleak], _DataT]
```

`active_update_processor.py` 则进一步支持 `poll_method`：

```python
async def poll_method(svc_info: BluetoothServiceInfoBleak) -> YourDataType:
    return YourDataType(...)
```

源码注释特别提醒：

```text
BluetoothServiceInfoBleak.device contains a BLEDevice. You should use this in
your poll function, as it is the most efficient way to get a BleakClient.
```

这正是 HA 使用 Bleak 的核心范式：**平时靠广播，必要时用 `BluetoothServiceInfoBleak.device` 建立 BleakClient 短连接补读**。

#### 10.4.7 ESPHome Bluetooth Proxy 如何进入这条链路

ESPHome 集成的接入点在 `homeassistant/components/esphome/bluetooth.py`：

```python
from bleak_esphome import connect_scanner

client_data = connect_scanner(cli, device_info, entry_data.available)
scanner = client_data.scanner

callbacks = [
    async_register_scanner(
        hass,
        scanner,
        source_domain=DOMAIN,
        source_model=device_info.model,
        source_config_entry_id=entry_data.entry_id,
        source_device_id=device_id,
    ),
    scanner.async_setup(),
]
```

也就是说，ESPHome proxy 不是绕开 HA 的蓝牙系统单独工作，而是通过 `bleak_esphome.connect_scanner()` 生成一个 remote scanner，再调用 HA bluetooth API 的 `async_register_scanner()` 注册进去。注册后：

```text
ESP32 广播 / GATT 能力
  -> bleak-esphome ESPHomeScanner / ESPHomeClient
  -> habluetooth manager
  -> HomeAssistantBluetoothManager
  -> BluetoothServiceInfoBleak / BLEDevice
  -> 各个 HA 集成
```

所以从 HA 集成作者视角看，本机蓝牙和 ESPHome proxy 的差异被隐藏了：拿到的仍然是 `BluetoothServiceInfoBleak`、`BLEDevice`，连接时仍然可以走 `BleakClient(device)`。

#### 10.4.8 总结：HA 并不是简单地“使用 Bleak”，而是把 Bleak 变成平台能力

Home Assistant 对 Bleak 的使用可以概括为四层：

| 层级 | HA 源码 / 类型 | 作用 |
|------|----------------|------|
| 扫描兼容层 | `async_get_scanner()` / `HaBleakScannerWrapper` | 给第三方库一个 BleakScanner 兼容入口 |
| 广播服务层 | `BluetoothServiceInfoBleak` | 把 BLEDevice + AdvertisementData 包装成 HA 统一服务信息 |
| 分发与匹配层 | `HomeAssistantBluetoothManager` / `BluetoothCallbackMatcher` | 根据地址、service UUID、manufacturer id、connectable 等条件分发 |
| 连接层 | `async_ble_device_from_address()` + `BleakClient(device)` | 需要 GATT 时才短连接读写 |
| 远程代理层 | `bleak_esphome.connect_scanner()` + `async_register_scanner()` | 把 ESPHome proxy 注册成 HA scanner / connector |

这也是为什么在 HA 生态里写 BLE 集成时，不建议自己创建独立 `BleakScanner`。正确姿势通常是：

```text
广播解析：注册 bluetooth callback，处理 BluetoothServiceInfoBleak；
需要等待特定广播：用 async_process_advertisements；
需要连接：从 service_info.device 或 async_ble_device_from_address() 取得 BLEDevice，再交给 BleakClient；
第三方库要求 BleakScanner：传 bluetooth.async_get_scanner(hass)。
```

这样写的集成天然支持本机蓝牙、多个 USB 适配器、ESPHome Bluetooth Proxy，以及 HA 的连接槽位管理和 active scan 调度。

### 10.5 结合 habluetooth 看类关系与数据流

`habluetooth` 是 Home Assistant 蓝牙体系的“中间层”。它不是替代 Bleak 的底层 BLE 协议栈，而是在 Bleak 之上增加 Home Assistant 需要的高可用能力：多 scanner 聚合、远程 scanner、连接路径选择、广播历史、可用性判断、active/passive scan 调度、连接槽位评分、Bleak 兼容 wrapper 等。

源码位置：

```text
D:\conda\envs\hass\Lib\site-packages\habluetooth
E:\pyproject\core\homeassistant\components\bluetooth
```

#### 10.5.1 核心类关系

可以把 HA + habluetooth + Bleak 的类关系简化成下面这张图：

```text
Bleak 层
  BLEDevice
  AdvertisementData
  BleakScanner / BleakClient
        ▲
        │ 兼容 wrapper
        │
habluetooth
  HaBleakScannerWrapper
  HaBleakClientWrapper
  BaseHaScanner
    ├── HaScanner                  # 本机适配器 scanner
    └── BaseHaRemoteScanner         # 远程 scanner 基类，例如 ESPHomeScanner
  BluetoothScannerDevice            # 某个 scanner 看到的某个 BLEDevice + AdvertisementData
  BluetoothServiceInfoBleak          # 对外分发的统一广播信息
  BluetoothManager                   # scanner 注册、历史、连接路径、callback、active scan 调度
        ▲
        │ subclass
        │
Home Assistant
  HomeAssistantBluetoothManager      # 增加集成匹配、config flow、存储、HA 生命周期
  bluetooth.api                      # async_get_scanner / async_register_callback / async_ble_device_from_address
        ▲
        │
各个集成
  bthome / govee_ble / gardena_bluetooth / homekit_controller / esphome / ...
```

各类职责可以这样理解：

| 类 / 模块 | 来源 | 职责 |
|----------|------|------|
| `BLEDevice` | Bleak | 表示扫描到的 BLE 外设，含 address/name/details |
| `AdvertisementData` | Bleak | 表示一次广播/扫描响应解析结果，含 manufacturer_data、service_data、RSSI 等 |
| `BaseHaScanner` | habluetooth | HA scanner 抽象，保存 source、adapter、connectable、connector、历史状态 |
| `HaScanner` | habluetooth | 本机蓝牙 scanner，包装本机 adapter 的 Bleak 扫描能力 |
| `BaseHaRemoteScanner` | habluetooth | 远程 scanner 基类，ESPHome proxy 这类远程数据源继承它 |
| `BluetoothScannerDevice` | habluetooth | 一个 scanner 对一个设备的观测：scanner + BLEDevice + AdvertisementData |
| `BluetoothServiceInfoBleak` | habluetooth / home_assistant_bluetooth | HA 对外分发的 BLE 服务信息，包装 BLEDevice + AdvertisementData + source/connectable/time |
| `BluetoothManager` | habluetooth | 多 scanner 聚合、地址历史、连接路径评分、active scan 调度、callback 分发 |
| `HomeAssistantBluetoothManager` | HA | 继承 BluetoothManager，加入 HA discovery flow、storage、matcher、生命周期 |
| `HaBleakScannerWrapper` | habluetooth | 给第三方库提供 BleakScanner 兼容 facade |
| `HaBluetoothConnector` | habluetooth | 连接器抽象，用于把某个 scanner/source 映射到对应 BleakClient backend |

#### 10.5.2 BaseHaScanner：从原始广播到 BluetoothServiceInfoBleak

`habluetooth/base_scanner.py` 里的 `BaseHaScanner` 是 scanner 的共同基类。初始化时它保存：

```python
self.connectable = connectable
self.source = source
self.connector = connector
self.adapter = adapter
self.name = adapter_human_name(adapter, source) if adapter != source else source
self.details = HaScannerDetails(...)
self._previous_service_info: dict[str, BluetoothServiceInfoBleak] = {}
```

几个字段很关键：

- `source`：这条广播来自哪里。可能是本机 HCI adapter，也可能是 ESPHome proxy 的 MAC/source。
- `connectable`：这个 scanner 是否能发起 GATT 连接。
- `connector`：如果能连接，用哪个 `HaBluetoothConnector` 创建 client。
- `_previous_service_info`：每个地址上一次广播状态，用于去重、可用性、历史恢复。

`BaseHaScanner` 会把底层 scanner 收到的 `BLEDevice + AdvertisementData` 转成 `BluetoothServiceInfoBleak` 再交给 manager。实际本机 scanner 和远程 scanner 的输入来源不同：

```text
本机 HaScanner:
  OS / Bleak backend -> BLEDevice + AdvertisementData -> BaseHaScanner

ESPHomeScanner:
  ESPHome Native API raw advertisement -> _async_on_raw_advertisement / _async_on_advertisement -> BaseHaScanner
```

但进入 manager 后，二者都变成同一种 `BluetoothServiceInfoBleak`。

#### 10.5.3 BluetoothScannerDevice：为什么同一个设备会有多条“路径”

`habluetooth/scanner_device.py` 定义：

```python
@dataclass(slots=True)
class BluetoothScannerDevice:
    scanner: BaseHaScanner
    ble_device: BLEDevice
    advertisement: AdvertisementData

    def score_connection_path(self, rssi_diff: int) -> float:
        return self.scanner._score_connection_paths(rssi_diff, self)
```

这说明 HA 不是只关心“有没有看到某个 MAC”，而是关心：

```text
某个 scanner/source 以什么 RSSI 看到了这个设备，
这个 scanner 是否可连接，
这个 scanner 当前是否还有连接槽位，
它最近连接这个地址是否失败过，
它当前是否已经有连接进行中。
```

`BaseHaScanner._score_connection_paths()` 里会把 RSSI、连接中数量、失败次数、连接槽位都纳入评分。比如：

- 没有空闲连接槽位时，直接认为该路径不可用。
- 有连接正在进行时降分。
- 最近连接失败过降分。
- 只有最后一个空闲 slot 时也会轻微降分。

这就是为什么 Home Assistant 能在多个 ESPHome proxy 或多个本机 adapter 之间选择“更适合连接”的路径，而不是简单选第一个看到设备的 scanner。

#### 10.5.4 BluetoothManager：habluetooth 的核心调度器

`habluetooth/manager.py` 的 `BluetoothManager` 是真正的数据调度核心。它维护了很多状态：

```python
self._all_history: dict[str, BluetoothServiceInfoBleak] = {}
self._connectable_history: dict[str, BluetoothServiceInfoBleak] = {}
self._connectable_scanners: set[BaseHaScanner] = set()
self._non_connectable_scanners: set[BaseHaScanner] = set()
self._sources: dict[str, BaseHaScanner] = {}
self._bleak_callbacks: set[BleakCallback] = set()
self._advertisement_tracker = AdvertisementTracker()
self._auto_scheduler = AutoScanScheduler(self)
self.slot_manager = slot_manager or BleakSlotManager()
```

这些状态对应几个能力：

| 状态 | 作用 |
|------|------|
| `_all_history` | 所有广播历史，包括不可连接来源 |
| `_connectable_history` | 可连接路径的广播历史，用于 GATT 连接 |
| `_connectable_scanners` | 能建立连接的 scanner 集合 |
| `_non_connectable_scanners` | 只能观察广播的 scanner 集合 |
| `_sources` | source -> scanner 映射 |
| `_bleak_callbacks` | 给 `HaBleakScannerWrapper` 兼容 Bleak callback 的回调集合 |
| `_advertisement_tracker` | 学习广告周期、判断 stale/unavailable、fallback 可用性 |
| `_auto_scheduler` | AUTO 扫描模式下按需触发 active scan window |
| `slot_manager` | 本机/远程连接槽位协调 |

从这里可以看出，`habluetooth` 做的是“蓝牙资源调度”，不是简单的数据结构封装。

#### 10.5.5 HaBleakScannerWrapper：为什么第三方库还能继续用 BleakScanner 风格

`habluetooth/wrappers.py` 里的 `HaBleakScannerWrapper` 是兼容层。它暴露了类似 BleakScanner 的方法：

```python
async def start(self) -> None
async def stop(self) -> None
async def advertisement_data(self) -> AsyncGenerator[tuple[BLEDevice, AdvertisementData], None]
async def discover(...) -> list[BLEDevice] | dict[str, tuple[BLEDevice, AdvertisementData]]
async def find_device_by_address(...) -> BLEDevice | None
async def find_device_by_name(...) -> BLEDevice | None
async def find_device_by_filter(...) -> BLEDevice | None
```

但它内部不是自己开一个 OS scan，而是从 `get_manager()` 取 HA 的全局 manager：

```python
infos = get_manager().async_discovered_service_info(True)
return {info.address: (info.device, info.advertisement) for info in infos}
```

`advertisement_data()` 也不是直接调用 Bleak 后端，而是注册到 manager：

```python
queue: asyncio.Queue[tuple[BLEDevice, AdvertisementData]] = asyncio.Queue()
cancel = get_manager().async_register_bleak_callback(
    lambda bd, ad: queue.put_nowait((bd, ad)),
    self._mapped_filters,
)
```

所以第三方库继续以 `BleakScanner` 的思维使用：

```python
async with scanner:
    async for device, adv in scanner.advertisement_data():
        ...
```

但数据源实际上已经是：

```text
所有 HA scanner 聚合后的数据，而不是某一个进程私有扫描器。
```

这就是 HA 需要 `habluetooth` 再封装 Bleak 的一个核心原因：**兼容第三方库，同时避免每个库各自打开扫描器造成资源冲突**。

#### 10.5.6 数据流：本机蓝牙广播如何到达集成

本机蓝牙的大致数据流是：

```text
OS 蓝牙栈 / BlueZ / CoreBluetooth / WinRT
  -> Bleak backend
  -> BLEDevice + AdvertisementData
  -> habluetooth.HaScanner / BaseHaScanner
  -> BluetoothServiceInfoBleak
  -> habluetooth.BluetoothManager 历史、去重、路径评分
  -> HA HomeAssistantBluetoothManager._discover_service_info()
  -> async_register_callback 的订阅者
  -> config flow discovery
  -> 具体集成实体 / coordinator
```

如果是第三方库通过 `async_get_scanner()` 使用 BleakScanner 风格 API，流向是：

```text
BluetoothManager
  -> HaBleakScannerWrapper.advertisement_data()
  -> 第三方库 callback / async iterator
```

所以 HA 不是把 Bleak 暴露为“全局唯一真实 BleakScanner”，而是暴露了一个“兼容 BleakScanner 的 HA wrapper”。

#### 10.5.7 数据流：ESPHome proxy 广播如何到达集成

ESPHome proxy 的路径稍微长一点：

```text
ESP32 bluetooth_proxy
  -> ESPHome Native API / aioesphomeapi
  -> bleak_esphome.connect_scanner()
  -> ESPHomeScanner(BaseHaRemoteScanner)
  -> raw advertisement / structured advertisement 转成 BLEDevice + AdvertisementData
  -> BluetoothServiceInfoBleak
  -> habluetooth.BluetoothManager
  -> HomeAssistantBluetoothManager
  -> 各集成 callback / discovery flow
```

连接时则是：

```text
集成需要 GATT
  -> service_info.device 或 async_ble_device_from_address()
  -> manager 选择合适 scanner/source
  -> scanner.connector / HaBluetoothConnector
  -> bleak_esphome ESPHomeClient 或本机 Bleak backend client
  -> BleakClient(device) 读写 GATT
```

这也是 `BluetoothScannerDevice` 和连接路径评分存在的原因：同一个 BLE 外设可能同时被客厅 ESPHome proxy、卧室 ESPHome proxy、本机 USB 蓝牙看到；HA 需要选择更可靠、更近、更有空闲 slot 的路径去连接。

#### 10.5.8 为什么 HA 不直接裸用 Bleak

如果 Home Assistant 直接让每个集成裸用 Bleak，会有几个问题：

1. **扫描资源冲突**：多个集成各自 `BleakScanner.discover()`，会导致重复扫描、耗电、丢包，甚至某些平台不允许并行扫描。
2. **连接资源有限**：BLE adapter / ESP32 proxy 同时连接数有限，需要统一 slot 管理和排队。
3. **多路径选择复杂**：同一个设备可能被多个 scanner 看到，裸 Bleak 不知道哪个 source 更适合连接。
4. **远程 scanner 不属于 Bleak 原生模型**：ESPHome proxy、未来其他网关都不是本机 OS 蓝牙适配器，需要抽象成 HA scanner。
5. **active/passive 扫描需要调度**：很多设备名称或 scan response 只有 active scan 才有，但长期 active scan 会增加空口和功耗，需要 AUTO 策略。
6. **HA 需要统一 discovery flow**：BLE 广播要能触发集成自动发现、回调已加载实体、保存历史，而不是散落在每个库里。
7. **第三方库兼容**：很多库已经要求 `BleakScanner` / `BLEDevice` / `BleakClient`，`HaBleakScannerWrapper` 可以兼容它们，同时仍走 HA 的统一管理。

因此 `habluetooth` 的价值是：

```text
Bleak 提供跨平台 BLE 原语；
habluetooth 把这些原语扩展成 HA 可调度、可聚合、可远程代理、可高可用的蓝牙资源层；
Home Assistant bluetooth 集成再在其上增加 discovery、storage、config entry、entity 生命周期。
```

#### 10.5.9 一句话总结类关系

```text
Bleak 负责“怎么和 BLE 设备说话”；
habluetooth 负责“HA 里这么多 scanner / proxy / integration 应该怎样共享蓝牙资源”；
HomeAssistantBluetoothManager 负责“把共享后的蓝牙数据接入 HA 的发现、回调、实体和配置系统”。
```

---

## 十一、一个面向传感器的完整设计范式

假设我们要支持一个 BLE 温湿度计，它广播里带 service data，也支持连接后读历史数据。推荐拆成三层。

### 11.1 广播解析层

```python
from dataclasses import dataclass

@dataclass
class SensorAdvertisement:
    address: str
    rssi: int
    temperature: float | None
    humidity: float | None
    battery: int | None


def parse_adv(device: BLEDevice, adv: AdvertisementData) -> SensorAdvertisement | None:
    # 假设 service data UUID 是 0000181a-0000-1000-8000-00805f9b34fb
    data = adv.service_data.get("0000181a-0000-1000-8000-00805f9b34fb")
    if not data or len(data) < 5:
        return None

    temp_raw = int.from_bytes(data[0:2], "little", signed=True)
    hum_raw = int.from_bytes(data[2:4], "little")
    battery = data[4]

    return SensorAdvertisement(
        address=device.address,
        rssi=adv.rssi,
        temperature=temp_raw / 100,
        humidity=hum_raw / 100,
        battery=battery,
    )
```

### 11.2 GATT 客户端层

```python
HISTORY_CHAR = "0000fff3-0000-1000-8000-00805f9b34fb"

async def read_history(client: BleakClient) -> bytes:
    raw = await client.read_gatt_char(HISTORY_CHAR)
    # 解析历史记录……
    return raw
```

### 11.3 调度层

```python
import asyncio
from bleak import BleakClient, BleakScanner

async def scan_loop() -> None:
    async with BleakScanner() as scanner:
        async for device, adv in scanner.advertisement_data():
            parsed = parse_adv(device, adv)
            if parsed:
                print(parsed)

async def read_history_once(device: BLEDevice) -> bytes:
    async with BleakClient(device, timeout=30.0) as client:
        return await read_history(client)
```

这样的代码在本机 Bleak 和 ESPHome proxy 场景下都比较容易迁移。

---

## 十二、生产环境建议

### 12.1 所有 BLE 操作都要有超时

蓝牙不是 TCP。外设可能睡眠、离线、RSSI 飘、连接参数慢、GATT 操作卡住。建议：

```python
async def read_once(device: BLEDevice, char_uuid: str) -> bytes:
    async with BleakClient(device, timeout=30.0) as client:
        return await asyncio.wait_for(client.read_gatt_char(char_uuid), timeout=10.0)

data = await asyncio.wait_for(read_once(device, UUID), timeout=60.0)
```

### 12.2 不要在 notify callback 里做慢操作

错误示范：

```python
def on_notify(sender: BleakGATTCharacteristic, data: bytearray) -> None:
    requests.post("https://example.com", data=data)  # 阻塞，危险
```

正确姿势：

```python
def on_notify(sender: BleakGATTCharacteristic, data: bytearray) -> None:
    queue.put_nowait(bytes(data))
```

然后在独立 async task 里处理队列。

### 12.3 对短连接设备做重试

很多 BLE 设备连接成功率不是 100%。可以使用 `bleak-retry-connector`，或者自己写指数退避：

```python
async def read_with_retry(device: BLEDevice, char_uuid: str, attempts: int = 3) -> bytes:
    last_error = None
    for i in range(attempts):
        try:
            async with BleakClient(device, timeout=30.0) as client:
                return await client.read_gatt_char(char_uuid)
        except Exception as err:
            last_error = err
            await asyncio.sleep(0.5 * (i + 1))
    raise last_error
```

实际项目里要把“连接 + GATT 操作”作为一个整体放进重试块，并确保每次失败都会释放连接。不要返回一个已经离开 context manager 的 client，也不要长期持有未明确断开的连接。

### 12.4 明确连接生命周期

长期保持 BLE 连接会带来：

- 外设耗电增加。
- ESP32 proxy 连接槽位被占用。
- 其他集成无法连接同一设备。
- 某些设备会因为 supervision timeout 或固件 bug 掉线。

传感器类设备优先考虑广播解析；只有配置、读历史、OTA、控制类操作才短暂连接。

### 12.5 记录原始数据

做私有 BLE 协议逆向或排障时，一定记录：

- `device.address` / macOS UUID。
- `adv.local_name`。
- `adv.rssi`。
- `adv.manufacturer_data`。
- `adv.service_data`。
- characteristic UUID / handle / properties。
- 原始 read / notify / write payload 的 hex。

否则现场问题复现会非常痛苦。

---

## 十三、总结

Bleak 解决的是 **Python 跨平台访问 BLE GATT 设备** 的问题。它用 `BleakScanner` 和 `BleakClient` 把 Windows WinRT、Linux BlueZ、macOS CoreBluetooth、Android 后端统一到 `asyncio` API：

- 扫描：`BleakScanner.discover()`、`advertisement_data()`、`find_device_by_filter()`。
- 连接：优先把扫描得到的 `BLEDevice` 传给 `BleakClient`。
- 服务发现：通过 `client.services` 遍历 service / characteristic / descriptor。
- 读写：`read_gatt_char()`、`write_gatt_char(response=True/False)`。
- 通知：`start_notify()` / `stop_notify()`，不要直接写 CCCD。
- 配对：`pair=True`、`pair()`、`unpair()`，但平台行为不同。

`bleak-esphome` 解决的是另一个更 IoT 的问题：**让 ESPHome Bluetooth Proxy 变成 Bleak / Home Assistant 生态里的远程 BLE 适配器**。它通过 aioesphomeapi 连接 ESP32，把 ESPHome 广播流适配成 habluetooth scanner，把 GATT 读写适配成 Bleak backend，并处理连接槽位、远程缓存、scanner mode、notify、错误转换等细节。

对于真正的 IoT 系统，推荐架构是：

```text
广播能解决的，尽量只解析广播；
必须连接时，短连接 + 超时 + 重试 + 明确释放；
业务协议只依赖 Bleak 抽象，不绑定本机蓝牙或某个 proxy；
覆盖不够时，用 ESPHome Bluetooth Proxy 扩展空间范围。
```

这样写出来的 Python BLE 程序，既能在开发机上直接调试，也能平滑迁移到 Home Assistant / ESPHome 的分布式蓝牙网络里。
