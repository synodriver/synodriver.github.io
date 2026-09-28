---
date: '2026-09-26T13:53:32+08:00'
draft: false
title: '温湿度计争霸战'
author: 'synodriver'
tags: ["homeassistant", "ble", "bthome", "iot"]
---

> *青萍商用温湿度气压传感器* VS *SONOFF SNZB-02M* VS *USTONE 环境传感器* VS *Ebyte EWD104-BT58* VS *ESP32H2+SHT45 diy bthome*

全家福

![参赛选手](/images/th_sensors-pic1.png)

## 接入hass难度

- *青萍商用温湿度气压传感器*

本来是不能直接接入hass的，但我给上游[pr](https://github.com/Bluetooth-Devices/qingping-ble/pull/111)了一下，下个版本应该有了，就当他支持吧
（之前是手写集成向他的cloud轮询的，说实话很不稳定）

- *SONOFF SNZB-02M*

原生zigbee3.0，zigbee2mqtt直接支持，但需要额外的zigbee网关

- *USTONE 环境传感器*

他自己捣鼓了一套基于ble广播的协议，数据放在manufacturer data里，hass原生不支持，但我写了个[集成](https://github.com/synodriver/homeassistant_custom_components/tree/main/custom_components/ha_ustone)，于是就支持了

- *Ebyte EWD104-BT58*

还是自己捣鼓的ble广播，数据还是塞在manufacturer data里，hass原生不支持，但我写了个[集成](https://github.com/synodriver/homeassistant_custom_components/tree/main/custom_components/ha_ebyte_beacon)，于是就又支持了

- *ESP32H2+SHT45 diy bthome*

自己写的固件肯定是用最简单的bthome v2广播，hass原生支持，加个蓝牙代理就能用

> 综上所述，这轮*ESP32H2+SHT45 diy bthome*获胜

## 数据更新频率

- *青萍商用温湿度气压传感器*

实测5s-20s一次，似乎取决于环境变化的剧烈程度

- *SONOFF SNZB-02M*

根据它的说明书，第一次使用要稳定30min，之后没有固定的更新间隔，依靠环境变化的剧烈程度来动态调节上报频率

- *USTONE 环境传感器*

10s一次广播，30s更新传感器

- *Ebyte EWD104-BT58*

实测5min更新一次

- *ESP32H2+SHT45 diy bthome*

我自己写的固件想怎么更新就怎么更新，还能动态调节（这并不是bthome规定的内容，我自己塞了个gatt server进去用于配置），
1s一次到10min一次都可以调

> 综上所述，这轮*ESP32H2+SHT45 diy bthome*获胜


## 精度

*青萍商用温湿度气压传感器* 
 
它的local_name是Temp RH Baro Pro S

根据第三方拆解，里面是SHT40，和博世的气压传感器，但根据记录，它似乎会定期上报全0数据，
导致hass中瞬间跌落一下，这个如果要设置自动化的话需要注意

*SONOFF SNZB-02M*

根据第三方拆解，里面还是SHT40，和博世(BMP280/BME280)的气压传感器

*USTONE 环境传感器* 

他的local_name是T1，发射功率似乎要低一些，测试中好几次掉线，查看网关没收到广播

客服自己承诺精度是典型±0.2℃（最大±1.4℃），典型±2%RH（最大±6%RH）

*Ebyte EWD104-BT58*

客服自己承诺不是sht系列传感器，温度精度理论±0.2℃
（不过似乎它时有出现不广播的情况，需要AT指令下发reset才正常）

*ESP32H2+SHT45 diy bthome*

自己设置了一个local_name是H2-BTHOME

顾名思义是SHT45，是各个传感器的老大哥（可惜买不到HDC302X系列的，各个店问了都说没有货，相当之坑）

上图：
![数据对比](/images/th_sensors-pic2.png)

因为SNZB-02M和青萍还有个气压传感器，把他们的数据和第三方BMP390L模块的数据也对比下

![气压数据对比](/images/th_sensors-pic3.png)

> 综上所述，这轮还是*ESP32H2+SHT45 diy bthome*获胜

## 续航

*青萍商用温湿度气压传感器* 

看记录约30-40天，不过可以充电

*SONOFF SNZB-02M*

官方说可以用5年（CR2477果真nb），目前没发现电池下降

*USTONE 环境传感器*

客服说可以用一年（CR2450），目前没发现电池下降

*Ebyte EWD104-BT58*

2个月电池没了2%，官方说是3-4年

*ESP32H2+SHT45 diy bthome*

虽然固件里面用了nimble框架，加入了自动lightsleep和自动广播间隔调节，
但根据计算，使用CR2450，5s广播一次，大约3-6个月

> 综上所述，这轮是*SONOFF SNZB-02M*获胜

## 未完待续