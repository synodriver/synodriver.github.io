---
date: '2026-07-18T00:00:00+08:00'
draft: false
title: 'BTHome v2 协议详解：广播格式、加密机制与两个开源实现'
author: 'synodriver'
tags: ["ble", "bluetooth", "bthome", "esphome", "micropython", "iot", "embedded"]
---

# BTHome v2 协议详解：广播格式、加密机制与两个开源实现

BTHome 是一套面向 BLE 广播的轻量传感器协议，核心目标很直接：**让电池设备不需要连接，也能把状态稳定地广播出去**。它适合温湿度、门磁、按钮、人体存在、开关状态这类低频、小数据量场景。

官方站点把 **BTHome v2** 作为当前主版本，并把 **v1** 单独归到 Legacy；v2 于 2022 年 11 月发布。本文按官方文档先讲协议格式，再结合两个开源实现看看它们是怎么落地的：

- [dz0ny/esphome-bthome](https://github.com/dz0ny/esphome-bthome)
- [DavesCodeMusings/BTHome-MicroPython](https://github.com/DavesCodeMusings/BTHome-MicroPython)

> 说明：下面的协议细节主要参考官方文档 [Format](https://bthome.io/format/) 和 [Encryption](https://bthome.io/encryption/)，实现分析参考上述两个仓库的源码。

## 一、BTHome v2 的广播长什么样

BTHome v2 本质上是 **BLE Advertising Payload** 的一种组织方式。它通常由几个 AD element 组成：

1. **Flags**，建议带上；
2. **Service Data (16-bit UUID)**，这是 BTHome 的主体；
3. 可选 **Local Name**，方便调试和识别。

官方文档给出的示例 payload 是：

```text
020106 0B094449592D73656E736F72 0A16D2FC4002C40903BF13
```

拆开看：

- `020106`：Flags AD element
- `0B09...`：Complete Local Name，名字是 `DIY-sensor`
- `0A16...`：Service Data，里面才是真正的 BTHome 数据

### 1.1 Service Data 的结构

BTHome 的服务数据部分是这样的：

```text
UUID(2 bytes, little-endian) + Device Information(1 byte) + one or more Measurements
```

以官方示例为例：

```text
D2FC 40 02C409 03BF13
```

含义是：

- `D2FC`：BTHome 的 16-bit UUID，字节序是 **小端**，所以逻辑上的 UUID 是 `0xFCD2`
- `40`：Device Information byte
- `02C409`：温度对象
- `03BF13`：湿度对象

### 1.2 Device Information byte

这个字节很关键，它告诉接收端当前广播是怎么发的。官方文档定义了这些位：

- **bit 0**：Encryption flag
  - `0` = 未加密
  - `1` = 已加密
- **bit 1**：保留
- **bit 2**：Trigger based device flag
  - `0` = 规则周期上报
  - `1` = 事件触发/不规则上报
- **bit 3-4**：保留
- **bit 5-7**：BTHome version
  - `010` = v2

所以：

- `0x40` = `0100 0000`，表示 **v2、未加密、周期上报**
- `0x41` = `0100 0001`，表示 **v2、已加密、周期上报**

这个设计的好处是：接收端不用先解密就能判断包的大致类型，也能知道它是不是触发型设备。

### 1.3 Measurements 的编码规则

BTHome 的测量项都遵循统一格式：

```text
Object ID + Value
```

其中：

- **Object ID**：1 字节，表示这个值是什么
- **Value**：按对象类型定义，通常是整数或字节串
- 数值型数据都使用 **little-endian**
- 读取时再乘以官方定义的 factor 恢复真实值

例如官方示例里：

- `02 C4 09`：
  - `0x02` = temperature
  - `0xC409`（小端）= 2500
  - factor = `0.01`
  - 结果 = `25.00°C`
- `03 BF 13`：
  - `0x03` = humidity
  - `0x13BF`（小端）= 5055
  - factor = `0.01`
  - 结果 = `50.55%`

### 1.4 Object ID 必须按数值顺序排列

官方明确要求：**对象 ID 要从小到大排列**。这不是摆设，而是兼容性策略。

原因很实用：如果设备发了多个对象，而接收端只认识其中一部分，那么接收端可以在遇到未知 object id 后停止解析，仍然拿到前面已经认识的数据。

这对长期维护很重要：

- 新设备可以先升级协议
- 老接收端仍能解析旧对象
- 不会因为新增一个对象就把整包都废掉

## 二、BTHome 支持哪些数据类型

官方文档把对象类型分成几类：

### 2.1 常规传感器数据

这是最常见的一类，例如：

- 温度
- 湿度
- 压力
- 光照
- 电压
- 电流
- 功率
- 距离
- 速度
- 能量

这些对象大多是整数，带 factor，常见宽度有：

- `uint8`
- `uint16`
- `uint24`
- `uint32`
- `uint48`
- `sint8`
- `sint16`
- `sint32`

### 2.2 Binary sensor data

二值传感器只有一个字节：

- `0` = false / off
- `1` = true / on

比如：

- motion
- door
- occupancy
- smoke
- tamper
- window

### 2.3 Events

BTHome 也支持事件类对象，例如：

- button press
- long press
- dimmer rotate

事件和普通传感器不同，它通常表示“发生了什么”，而不是“当前是什么状态”。

官方文档里也提到，同类型事件可以连续发送多个；`None` 事件可用于占位。

### 2.4 Commands

命令对象用于执行器场景，例如控制类设备。官方把 command 作为可变长度对象处理：

- object id = `0x3B`
- 后面一个字节里包含参数长度
- 再后面是 opcode 和参数

官方建议这类命令**尽量使用加密广告**，否则容易被旁路观察或伪造。

### 2.5 文本和原始字节

`text` 和 `raw` 是变长类型：

- object id 后面要紧跟一个长度字节
- 再后面才是内容
- `text` 使用 UTF-8
- `raw` 则按字节串处理

这类对象适合：

- 固件版本
- 设备标识
- 任意自定义负载

## 三、Packet ID：去重和重发的关键

BTHome 还有一个很重要但容易忽略的对象：**packet id**，object id 是 `0x00`。

它的作用是：

- 让接收端识别重复包
- 允许设备为了提高投递成功率，重复发同一条广播
- 但接收端应该只接受不同 packet id 的新包

这和“广播不是可靠传输”这个事实完全匹配：

- BLE 广播可能丢包
- 设备可以重复发几次提高命中率
- 接收端通过 packet id 去重

从源码实现看，这个对象在两个仓库里都被认真处理了：

- ESPHome 实现里，`packet_id_` 在**构建新广告**时递增；重发时复用同一份广告数据，不重新构建
- MicroPython 实现里，`packet_id` 属性每次访问自增并按 `0xFF` 回绕

## 四、加密：BTHome 的安全边界在哪里

官方文档明确支持 **AES-CCM** 加密的 BLE 广播，密钥是 **16 字节**。

### 4.1 加密了什么

加密后，真正的数据载荷会被保护起来，但这里要注意：

- 广播本身仍然能被抓到
- 设备标识、广播行为等元信息并不会“消失”
- 安全性不是“只要加密就万无一失”

官方特别警告：如果接收端没有做严格检查，BTHome 加密**并不能自动变成安全保障**。

原因有两个：

1. 攻击者可以发一个未加密的伪造包，接收端如果不区分，就可能照样处理；
2. 接收端必须检查 **counter**，否则可以被重放攻击。

### 4.2 计数器 counter 的作用

加密包里带有递增计数器，用于防重放。接收端应该记住上一次的 counter，只有更大的值才接受。

如果不检查 counter，攻击者录下一个合法包，之后再重复发，就可能触发错误动作。

### 4.3 官方与实现中的 nonce 设计

从 `esphome-bthome` 的实现可以看到，nonce 由这些部分组成：

- MAC 地址 6 字节
- BTHome UUID 2 字节
- device info 1 字节
- counter 4 字节

也就是一个 13 字节 nonce。

实现里使用了 AES-CCM：

- ESP32/NimBLE 分支用 tinycrypt
- ESP32/Bluedroid 分支用 mbedtls
- nRF52 分支也用 tinycrypt

这和官方“BTHome supports AES encryption (CCM mode)”是对应的。

### 4.4 设备信息字节在加密中的意义

加密时，device info 不只是“声明字段”，它还参与到 nonce 构造里。也就是说：

- 未加密/已加密
- trigger-based / regular

这些状态都会影响广播和解密上下文。

这也是为什么实现里会把 device info 作为一个独立概念处理，而不是简单塞进 payload 里就完事。

## 五、两个开源实现是怎么做的

### 5.1 dz0ny/esphome-bthome：ESPHome 风格的完整广播组件

这个仓库的思路很“ESPHome”：把 BTHome 做成一个组件，让用户在 YAML 里声明 sensor，组件负责组包、加密、重发和广播。

### 5.1.1 广播构造流程

源码里可以看到它会先构建 AD flags，然后构建 service data：

- `Flags`
- `Service Data 16-bit UUID`
- `UUID + device info + measurements`

再把这些写入广告缓冲区。

### 5.1.2 Device info 的拼接方式

实现会根据状态决定 device info byte：

- 是否加密
- 是否 trigger-based

也就是把协议位直接映射到内部状态，不靠外部再手工拼字节。

### 5.1.3 packet id 和重发

这是这个仓库里很值得看的地方。

它不是每次 loop 都重新生成一包，而是区分：

- **新数据**：重新 build 广告，packet id 增加
- **重发**：复用已有广告，避免把 packet id 也一起改掉

这样接收端就能把重复广播识别出来，同时提高广播可靠性。

### 5.1.4 加密实现

源码显示它支持两套 ESP32 BLE 栈：

- NimBLE
- Bluedroid

并分别用 tinycrypt / mbedtls 做 CCM 加密。加密时会把 counter 写进包里，并在每次成功构建加密广告后递增。

这说明它不是“只会把 payload 变成乱码”，而是完整遵循了 BTHome 的加密广播逻辑。

### 5.1.5 触发式设备与循环调度

组件还支持 trigger-based 场景：

- 普通周期上报：靠 loop 或调度广播
- 触发式上报：传感器变更时立即触发广告

这和官方 device info bit 2 的语义是对齐的。

## 5.2 DavesCodeMusings/BTHome-MicroPython：轻量、清晰的打包器

这个仓库更像一个“协议参考实现”，重点是**在 MicroPython 里把 BTHome 广播拼出来**。

### 5.2.1 直接拼 BLE Advertising Payload

它的实现非常直观：

- 固定 `020106` 作为 flags
- 再拼 local name
- 再拼 service data
- 最后检查总长度是否超过 31 字节

这和官方广播格式是一一对应的。

### 5.2.2 Device info 被硬编码为 0x40

源码里 `_DEVICE_INFO_FLAGS = 0x40`，注释写得很明确：

- no encryption
- regular updates
- version 2

也就是说，这个库默认只做**未加密的 v2 广播**。

### 5.2.3 数据类型映射非常完整

它把 object id 和 pack 函数做了一个大映射：

- `pack("<BH")`
- `pack("<BL")[:-1]`
- `pack("<BQ")[:-2]`
- binary 用 `pack("BB", object_id, 0/1)`

这点很关键，因为它把 BTHome 的“字节序”和“字段宽度”落到了代码里。

### 5.2.4 对象 ID 按数值排序

源码在 `_pack_service_data()` 里对 `args` 做了 `sorted()`，这和官方要求完全一致：

- object id 从低到高
- 保证兼容性

### 5.2.5 packet id 也有实现

它通过 `packet_id` 属性递增并按 8 位回绕，符合官方的重复包去重思路。

### 5.2.6 它没有做加密

这是这个仓库和 ESPHome 实现最大的不同：

- 它专注于广播封包构造
- 没有实现 AES-CCM
- 没有 counter / nonce 这一整套安全逻辑

所以它更适合作为**协议入门参考**，而不是完整的生产级加密发送端。

## 六、把官方文档和两个实现放在一起看

这三个来源放在一起，BTHome v2 的设计就很清楚了：

| 层次 | 作用 | 关键点 |
|------|------|--------|
| 官方协议 | 规定格式 | UUID、device info、object id、加密、packet id、事件、命令 |
| ESPHome 实现 | 面向产品 | 完整广播、加密、counter、重发、trigger-based |
| MicroPython 实现 | 面向教学/原型 | 轻量拼包、类型映射清晰、未加密广播 |

## 七、实战上最容易踩的坑

### 7.1 小端序别写反

BTHome 的数值字段大多是 little-endian。温度、湿度、压力、计数器等都要先按协议宽度编码，再按小端放进 payload。

### 7.2 object id 一定要排序

如果乱序，老接收端可能在遇到未知对象时直接停掉，导致前面的数据也拿不全。

### 7.3 packet id 不是装饰字段

如果要重复发同样的广告，packet id 要保持一致；如果是新数据，packet id 才应该变化。

### 7.4 加密不等于安全

官方已经说得很直接：接收端必须检查 counter，否则仍然会被重放。

### 7.5 31 字节限制是真限制

特别是使用 Local Name、多个对象、长文本时，很容易超出传统广播长度上限。

## 八、一个最小 BTHome v2 广播模板

如果你要手工拼一个最小的 BTHome v2 未加密广播，结构可以记成这样：

```text
Flags
+ Service Data (0x16)
+ UUID 0xFCD2 (little-endian: D2 FC)
+ Device Info (0x40)
+ Object ID + Value
+ Object ID + Value
+ ...
```

例如：

```text
020106
0A16D2FC4002C40903BF13
```

这就是一个带温度和湿度的典型 BTHome 广播。

## 九、结论

BTHome v2 的设计很“嵌入式”：

- 用 BLE 广播，省电、无连接、部署简单；
- 用固定的 UUID 和 device info，快速识别版本和加密状态；
- 用 object id + little-endian 数据值，做到紧凑、统一、易扩展；
- 用 packet id 和 counter，分别解决重复投递和重放攻击；
- 用排序规则保证兼容性；
- 用 AES-CCM 把加密广播标准化。

如果你正在写一个 ESPHome 组件、MicroPython 传感器广播器，或者想自己实现一个 BTHome 接收器，最重要的不是先写代码，而是先把这几件事搞清楚：

1. 广播包到底怎么分层；
2. device info 每一位是什么意思；
3. object id / 数据类型 / 字节序怎么对应；
4. packet id 和 counter 分别解决什么问题；
5. 接收端到底要不要接受未加密包。

只要这几件事对了，BTHome 的实现其实并不复杂。

## 参考资料

- [BTHome Format](https://bthome.io/format/)
- [BTHome Encryption](https://bthome.io/encryption/)
- [BTHome Legacy v1](https://bthome.io/v1/)
- [dz0ny/esphome-bthome](https://github.com/dz0ny/esphome-bthome)
- [DavesCodeMusings/BTHome-MicroPython](https://github.com/DavesCodeMusings/BTHome-MicroPython)
- [Home Assistant BTHome integration](https://www.home-assistant.io/integrations/bthome)
