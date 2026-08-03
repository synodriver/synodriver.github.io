---
date: '2026-08-01T18:50:15+08:00'
draft: false
title: 'ESP32工控机变形记'
author: 'synodriver'
tags: ["python", "c++", "esphome", "iot", "ble"]
---

# ESP32工控机变形记

## 引言

最近搞了个ESP32S3的工控机，卖家宣传是跑micropython的，看着评论说这个能自己更新到新版的micropython，
当场就猜测这东西能刷基于esp-idf的全部固件，再加上外置的天线和lora模组，实在是妙不可言，赶紧
整了一个来试试

## 软硬件

![正面](/images/ecm50-a07-1.PNG)

到手拆包，这小家伙还有8MB的psram，16MB的flash，外置wifi/蓝牙天线，lora天线
说是跑micropython的，不过我已经给他准备了更合适的固件：esphome蓝牙代理之工控机专用版，哈哈
```yaml
substitutions:
  name: "ecm50-a07"
  location: "车间"  #放置位置

esphome:
  name: "${name}"
  name_add_mac_suffix: true
  friendly_name: "${location}可编程rtu"
  comment: ECM50-A07
  project:
    name: synodriver.ECM50-A07
    version: "0.0.1"
  
preferences:
  flash_write_interval: 10min
esp32:
  variant: esp32s3
  # board: esp32-s3-devkitc-1
  flash_size: 16MB
  cpu_frequency: 240MHz
  framework:
    type: esp-idf
    advanced:
      execute_from_psram: true

psram:
  mode: octal
  speed: 80MHz

debug:
  update_interval: 5s
  
external_components:
  # - source:
  #     type: git
  #     url: https://github.com/n-serrette/esphome_sd_card
  #     ref: main
  #   components: [sd_mmc_card]
  - source: 
      type: git
      url: https://github.com/andrewbackway/esphome-sd_card_logger_web
      ref: main
    components: [sd_card]
  - source: 
      type: git
      # url: https://github.com/oxan/esphome-stream-server 还没适配新版esphome
      url: https://github.com/grob6000/esphome-stream-server
      ref: master
    components: [stream_server]

uart:
  - id: modbus_bus
    tx_pin: 17
    rx_pin: 18
    baud_rate: 4800
    stop_bits: 1
    rx_buffer_size: 512
  - id: lora_bus
    tx_pin: 39
    rx_pin: 40
    baud_rate: 9600
    stop_bits: 1
    rx_buffer_size: 512

stream_server:
  id: stream_server_
  uart_id: lora_bus
  port: 1234

modbus:
  # flow_control_pin: GPIOXX
  id: modbus1
  uart_id: modbus_bus

modbus_controller:
- id: modbus_device
  address: 0x1   ## address of the Modbus slave device on the bus
  modbus_id: modbus1
  setup_priority: -10
  
ethernet:
  type: W5500
  clk_pin: 12
  mosi_pin: 11
  miso_pin: 13
  cs_pin: 47
  interrupt_pin: 14
  reset_pin: 48

logger:
  level: DEBUG

# Enable Home Assistant API
api:
  on_client_connected:
    - logger.log:
        format: "Client %s connected to API with IP %s"
        args: ["client_info.c_str()", "client_address.c_str()"]

ota:
  - platform: esphome

binary_sensor:
  - platform: gpio
    name: DI1
    pin: 
      number: 4
      mode:
        input: true
        pulldown: True
    # on_click:
    #   then:
    #     - switch.toggle: relay1
  - platform: gpio
    name: DI2
    pin: 
      number: 5
      mode:
        input: true
        pulldown: True
    # on_click:
    #   then:
    #     - switch.toggle: relay1
  - platform: stream_server
    stream_server: stream_server_
    connected:
      name: Connected

text_sensor:
  - platform: ethernet_info
    ip_address:
      name: "${name} - IP Address"
      address_0:
        name: "${name} - IP Address 0"
      address_1:
        name: "${name} - IP Address 1"
      address_2:
        name: "${name} - IP Address 2"
      address_3:
        name: "${name} - IP Address 3"
      address_4:
        name: "${name} - IP Address 4"
    dns_address:
      name: "${name} - DNS Address"
    mac_address:
      name: "${name} - MAC Address"
      
  - platform: debug
    device: 
      name: "device info"
    reset_reason: 
      name: "reset reason"

switch:
  - platform: gpio
    name: "DO1"
    id: DO1
    pin: 15
  - platform: gpio
    name: "DO2"
    id: DO2
    pin: 16
  - platform: gpio
    name: "LORA_M1"
    id: M1
    pin: 41
    restore_mode: ALWAYS_OFF
  - platform: restart
    name: "${name} controller Restart"
  - platform: factory_reset
    name: Restart with Factory Default Settings
    disabled_by_default: true

sensor:
  - platform: adc
    pin: 6
    attenuation: 11db
    name: AI1
    update_interval: 10s
    unit_of_measurement: mA
    filters: 
      - lambda: "return x / 150.0f;"
  - platform: adc
    pin: 7
    attenuation: 11db
    name: AI2
    update_interval: 10s
    unit_of_measurement: mA
    filters: 
      - lambda: "return x / 150.0f;"
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
    name: "internal_temperature"
  - platform: stream_server
    stream_server: stream_server_
    connection_count:
      name: Number of connections
  - platform: sd_card
    sd_card_id: sd_card_1
    type: used_space
    name: "SD Used (GB)"
    filters:
      - lambda: return (float)sd_card::convertBytes((uint64_t)x, sd_card::MemoryUnits::GigaByte);
  - platform: sd_card
    sd_card_id: sd_card_1
    type: total_space
    name: "SD Card Capacity (GB)"
    filters:
      - lambda: return (float)sd_card::convertBytes((uint64_t)x, sd_card::MemoryUnits::GigaByte);
  - platform: sd_card
    sd_card_id: sd_card_1
    type: free_space
    name: "SD Card Remaining (GB)"
    filters:
      - lambda: return (float)sd_card::convertBytes((uint64_t)x, sd_card::MemoryUnits::GigaByte);

      
light:
  - platform: status_led
    name: "WiFi"
    id: WiFi
    pin: 45
  - platform: status_led
    name: "Run"
    id: Run
    pin: 21

    
sd_card:
  id: sd_card_1
  cmd_pin: 46   # MOSI / CMD
  clk_pin: 10   # SCK  (onboard TF slot)
  data0_pin: 9 # MISO / DATA0
  mode_1bit: true  # only 3 lines physically wired on LOLIN S3 Pro


esp32_ble:
  io_capability: none
  disable_bt_logs: true  # Default, saves flash
  connection_timeout: 60s  # Default, matches client timeout
  max_connections: 5  # Default, total BLE connections
  enable_on_boot: true
  max_notifications: 12  # Default, increase if needed
  use_psram: True

esp32_ble_tracker:
  id: ble_tracker
  scan_parameters:
    interval: 600ms
    window: 400ms
    active: True

bluetooth_proxy:
  active: True
  connection_slots: 5
```
以上只是基础，有兴趣还能自己加料，比如把modbus给用起来

## 效果

非同凡响，识别的设备远超别的蓝牙代理，大概2堵墙的BLE设备都能被识别到，当然了，前提是你得把天线给接好，外置天线的效果就是不一样
![效果](/images/ecm50-a07-2.PNG)