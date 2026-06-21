---
date: '2026-06-21T19:30:09+08:00'
draft: false
title: '臭鱼烂虾养成记'
author: 'synodriver'
tags: ["iot", "esphome"]
---

# 臭鱼烂虾养成记之手搓tds传感器

## 壹

发现一个[TDS传感器模块](https://www.atombit.cn/)，esphome上没有现成的组件，看他数据手册，串口的，写的挺明确，正好就给鱼缸的臭鱼烂虾加个传感器，免得出事

## 贰

对着洞洞板一顿焊接，东西出来了，
![反面](/images/tds-sensor-pic1.png)
![正面](/images/tds-sensor-pic2.png)

yaml一顿写:
```yaml
substitutions:
  name: "tds-sensor"
  location: "客厅"  #放置位置

esphome:
  name: "${name}"
  name_add_mac_suffix: true
  friendly_name: "${location}tds传感器"
  comment: "tds-sensor"
  project:
    name: synodriver.tds-sensor
    version: "0.0.1"
preferences:
  flash_write_interval: 10min
esp32:
  board: esp32-s3-devkitc-1
  flash_size: 8MB
  cpu_frequency: 240MHz
  framework:
    type: esp-idf
    advanced:
      execute_from_psram: true


psram:
  mode: quad
  speed: 80MHz
# Enable logging
logger:
  level: DEBUG

# Enable Home Assistant API
api:
  actions:
    - action: twice_calibrate
      variables:
        salinity: float
      then:
        - bax.twice_calibrate:
            id: bax_
            salinity: !lambda "return salinity;"


  on_client_connected:
    - logger.log:
        format: "Client %s connected to API with IP %s"
        args: ["client_info.c_str()", "client_address.c_str()"]

ota:
  - platform: esphome
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    password: ""
  #use_address: 192.168.5.240
captive_portal:

web_server:
  port: 80    

external_components:
# 这是我的本地组件，你们用下面的
#  - source:
#      type: local
#      path: components
  - source:
      type: git
      url: https://github.com/ha-china/esphome_external_componnets
      ref: main
      components: ["bax"]  # 指定要加载的组件，或 "all"
      refresh: 1h              # Git 仓库刷新间隔（默认1天） 


uart:
  id: uart_bus
  tx_pin: 17
  rx_pin: 18
  baud_rate: 9600
  stop_bits: 1

bax:
  id: bax_
  uart_id: uart_bus
  type: BA234
  update_interval: 10s

debug:
  update_interval: 5s

sensor:
  - platform: bax
    bax_id: bax_
    tds:
      name: "TDS"
    ec:
      name: "EC"
    salinity:
      name: "salinity"
    specific_gravity:
      name: "specific_gravity"
    temperature:
      name: "temperature"
    hardness:
      name: "hardness"
    
    
  - platform: wifi_signal # Reports the WiFi signal strength/RSSI in dB
    name: "${name} WiFi Signal dB"
    id: wifi_signal_db
    update_interval: 60s
    entity_category: "diagnostic"

  - platform: copy # Reports the WiFi signal strength in %
    source_id: wifi_signal_db
    name: "${name} WiFi Signal Percent"
    filters:
      - lambda: return min(max(2 * (x + 100.0), 0.0), 100.0);
    unit_of_measurement: "%"
    entity_category: "diagnostic"

  - platform: debug
    free:
      name: "Heap Free"
    block:
      name: "Heap Max Block"
    loop_time:
      name: "Loop Time"
    cpu_frequency:
      name: "CPU Frequency"
    psram: 
      name: "Free PSRAM"

  - platform: internal_temperature
    name: "Internal Temperature"

time:
  - platform: sntp
    id: my_time

switch:
  - platform: restart
    name: "${name} controller Restart"
  - platform: factory_reset
    name: Restart with Factory Default Settings
    disabled_by_default: true

text_sensor:
  - platform: wifi_info
    ip_address:
      name: ${name} IP Address
    ssid:
      name: ${name} Connected SSID
    bssid:
      name: ${name} Connected BSSID
    mac_address:
      name: ${name} Mac Wifi Address
    # scan_results:
    #   name: ${name} Latest Scan Results
    dns_address:
      name: ${name} DNS Address
  - platform: version
    name: "ESPHome Version"
  
  - platform: debug
    device: 
      name: "device info"
    reset_reason: 
      name: "reset reason"

button:
  - platform: bax
    bax_id: bax_
    zero_point_calibrate:
      name: "zeropoint-baseline calibrate"
  # 下面这些也可以写到api里面，做成service，可惜的是这是其他型号的，BA234不支持这些功能
  # - platform: template
  #   name: "calibrate"
  #   on_press: 
  #     then:
  #       - bax.zero_point_calibrate:
  #           id: bax_    
  
  # - platform: template
  #   name: "twice_calibrate"
  #   on_press: 
  #     then:
  #       - bax.twice_calibrate:
  #           id: bax_
  #           salinity: !lambda "return 5;"
        
  # - platform: template
  #   name: "set_ntc_resistance"
  #   on_press: 
  #     then:
  #       - bax.set_ntc_resistance:
  #           id: bax_
  #           resistance: !lambda "return 10000;"

  # - platform: template
  #   name: "set_ntc_b_value"
  #   on_press: 
  #     then:
  #       - bax.set_ntc_b_value:
  #           id: bax_
  #           b: !lambda "return 3850;"

# 秘籍：炫酷浴缸灯，没有的就不加
light:
  - platform: esp32_rmt_led_strip
    rgb_order: GRB
    pin: 48
    num_leds: 1
    chipset: ws2812
    name: "status led"
```
刷机开机，有数据了
![ha](/images/tds-sensor-pic3.png)
![效果](/images/tds-sensor-pic4.png)



## 配置说明

bax 组件是用于atombit系列水质检测芯片（BA012 / BA022 / BA111 / BA121 / BA234 / BA311 / BAT3U）的ESPHome external component

### 安装

通过 `external_components` 拉取即可，无需手动 clone：

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/ha-china/esphome_external_componnets
      ref: main
      components: ["bax"]
      refresh: 1h
```

- 也可以用本地路径开发：把 `source.type` 改成 `local`，`path` 指向放 `bax` 目录的文件夹。组件本身 `MULTI_CONF = true`，所以你可以在一台设备上挂多个 bax 实例（接多路传感器）。


## 基础组件 `bax`

bax 是一个 `PollingComponent`，每个 `update_interval` 周期主动向传感器发送一次测量指令（BA311 除外，它自动上传），随后在 `loop()` 里解析串口返回的帧。

```yaml
# 必须先定义一路 uart
uart:
  id: uart_bus
  tx_pin: 17
  rx_pin: 18
  baud_rate: 9600
  stop_bits: 1

bax:
  id: bax_
  uart_id: uart_bus
  type: BA234
  update_interval: 10s
```

### 配置变量

- **id**（必填，ID）：组件实例 id，供 sensor / button / automation 引用。
- **type**（必填，枚举）：芯片型号。可选 `BA012`、`BA022`、`BA111`、`BA121`、`BA234`、`BA311`、`BAT3U`。
- **uart_id**（选填，ID）：挂到哪一路 uart，多路 uart 时必填。
- **update_interval**（选填，时间，默认 `20s`）：主动测量间隔。BA311 会忽略此值（自动上传）。
- note: UART 要求 : **组件在编译期会校验 uart 配置：必须同时配置 `tx_pin` 与 `rx_pin`，波特率 `9600`，`stop_bits: 1`，无校验位。**


### 各型号支持的测量量

不同芯片返回的帧格式不同，能解析出的传感器也不一样，✅ 表示该型号会发布对应数据：

| 传感器            | BA012 | BA022 | BA111 | BA121 | BA234 | BA311 | BAT3U |
| ----------------- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| `tds`             |  ✅   |       |  ✅   |       |  ✅   |  ✅   |  ✅   |
| `tds2`            |  ✅   |       |       |       |       |       |  ✅   |
| `tds3`            |       |       |       |       |       |       |  ✅   |
| `ec`              |       |  ✅   |       |  ✅   |  ✅   |       |       |
| `ec2`             |       |  ✅   |       |       |       |       |       |
| `salinity`        |       |       |       |       |  ✅   |       |       |
| `specific_gravity`|       |       |       |       |  ✅   |       |       |
| `temperature`     |  ✅   |  ✅   |  ✅   |  ✅   |  ✅   |       |  ✅   |
| `temperature2`    |  ✅   |  ✅   |       |       |       |       |  ✅   |
| `temperature3`    |       |       |       |       |       |       |  ✅   |
| `hardness`        |       |       |       |       |  ✅   |       |       |


## 传感器平台 `sensor.bax`

`sensor.bax` 负责把 bax 解析出来的数值发布成 Home Assistant 传感器。所有子项都是选填，按你的芯片支持情况挑着配；配了但芯片不返回的那个不会出值（也不会报错）。

```yaml
sensor:
  - platform: bax
    bax_id: bax_
    tds:
      name: "TDS"
    ec:
      name: "EC"
    salinity:
      name: "salinity"
    specific_gravity:
      name: "specific_gravity"
    temperature:
      name: "temperature"
    hardness:
      name: "hardness"
```

### 配置变量

- **bax_id**（必填，ID）：指向对应的 `bax` 组件实例。
- **tds** / **tds2** / **tds3**（选填）：TDS 值，单位 `ppm`，图标水滴。
- **ec** / **ec2**（选填）：电导率，单位 `µS/cm`，图标交流电。
- **salinity**（选填）：盐度，单位 `%`，两位小数。
- **specific_gravity**（选填）：比重，单位 `#`，四位小数。
- **temperature** / **temperature2** / **temperature3**（选填）：温度，单位 `°C`，设备类 `temperature`。
- **hardness**（选填）：硬度，单位 `ppm`，图标烧瓶。

每项下面都可以再写标准传感器字段（`name`、`id`、`unit_of_measurement`、`accuracy_decimals`、`icon`、`device_class`、`state_class`、`filters` 等），默认值已按上表设好，通常只写 `name` 就够。

## 按钮 `button.bax`

提供一个零点（基线）校准按钮，按下后通过 uart 给传感器发校准指令。

```yaml
button:
  - platform: bax
    bax_id: bax_
    zero_point_calibrate:
      name: "zeropoint-baseline calibrate"
```

### 配置变量

- **bax_id**（必填，ID）：指向对应的 `bax` 组件实例。
- **zero_point_calibrate**（选填）：零点校准按钮。设备类 `restart`，归类到 `config`。

warning 型号差异

**零点校准指令按型号分两种：BA234 用 `0xA1`，其余（BA012/BA022/BA111/BA121/BAT3U）用 `0xA6` 基线校准，BA311 不支持校准。所以校准按钮只对除 BA311 外的型号有意义。**


## 自动化动作

除了按钮，组件还注册了几个 `automation.Action`，可以在 `on_press`、`api.actions`、`lambda` 等任意 automation 上下文里调用。

### `bax.zero_point_calibrate`

触发一次零点校准，等价于上面的按钮。

```yaml
on_press:
  then:
    - bax.zero_point_calibrate:
        id: bax_
```

也可以用简写形式 `- bax.zero_point_calibrate: bax_`。

### `bax.twice_calibrate`

两点校准。`salinity` 是校准用的标准液盐度，**以百分数输入**（4% 浓度就填 `4.0`），支持 lambda。

```yaml
- bax.twice_calibrate:
    id: bax_
    salinity: !lambda "return salinity;"
```

**warning 仅 BA234 支持，其它型号调用会被忽略。**

### `bax.set_ntc_resistance`

设置 NTC 热敏电阻阻值（`uint32_t`，单位 Ω），用于温度补偿标定。支持 BA012/BA022/BA111/BA121/BAT3U，BA234 与 BA311 不支持。

```yaml
- bax.set_ntc_resistance:
    id: bax_
    resistance: !lambda "return 10000;"
```

### `bax.set_ntc_b_value`

设置 NTC 的 B 值（`uint16_t`），同样用于温度补偿标定，支持的型号同上。

```yaml
- bax.set_ntc_b_value:
    id: bax_
    b: !lambda "return 3850;"
```

## 排错

- **日志里反复 `CRC mismatch`**：多半是接线不稳或波特率不对，组件对每帧都做累加和 / CRC-8 校验，校验失败会整帧丢弃。确认 `baud_rate: 9600`、`stop_bits: 1`、TX/RX 没接反。
- **传感器一直没值**：先对照上面的型号支持表，确认你配的传感器项该型号确实会返回；再把 `logger.level` 调到 `DEBUG` 看 `bax:` 组件的 `dump_config` 输出和原始帧。
- **校准点了没反应**：检查型号是否支持对应校准动作（两点校准仅 BA234，NTC 设置排除 BA234/BA311，零点校准排除 BA311）。

## See Also

- [ESPHome external components 文档](https://esphome.io/components/external_components.html)
- [uart 组件](https://esphome.io/components/uart.html)
- [sensor 组件](https://esphome.io/components/sensor/index.html)
- [button 组件](https://esphome.io/components/button/index.html)
- [Automation 指南](https://esphome.io/guides/automations.html)
- 组件源码与各芯片 PDF 数据手册见仓库 `components/bax` 目录和官方网站
