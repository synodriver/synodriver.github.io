+++
date = '2026-04-20T23:10:28+08:00'
draft = false
title = 'Esphome'
author = 'synodriver'
+++
# ESPHOME自定义组件崩溃调试方法

[LD2460组件](https://github.com/ha-china/esphome_external_componnets/tree/main/components/ld2460)过程中遇到运行的崩溃问题，记录调试方法

- 看日志
```
[21:29:59]ESP-ROM:esp32c6-20220919
[21:29:59]Build:Sep 19 2022
[21:29:59]rst:0x15 (USB_UART_HPSYS),boot:0xc (SPI_FAST_FLASH_BOOT)
[21:29:59]Saved PC:0x408052da
[21:29:59]SPIWP:0xee
[21:29:59]mode:DIO, clock div:2
[21:29:59]load:0x40875730,len:0x16bc
[21:29:59]load:0x4086c110,len:0xdf4
[21:29:59]load:0x4086e610,len:0x3078
[21:29:59]entry 0x4086c110
[21:29:59]I (23) boot: ESP-IDF 5.4.2 2nd stage bootloader
[21:29:59]I (23) boot: compile time Oct 14 2025 13:19:17
[21:29:59]I (23) boot: chip revision: v0.1
[21:29:59]I (23) boot: efuse block revision: v0.3
[21:29:59]I (24) boot.esp32c6: SPI Speed      : 80MHz
[21:29:59]I (24) boot.esp32c6: SPI Mode       : DIO
[21:29:59]I (24) boot.esp32c6: SPI Flash Size : 4MB
[21:29:59]I (25) boot: Enabling RNG early entropy source...
[21:29:59]I (25) boot: Partition Table:
[21:29:59]I (25) boot: ## Label            Usage          Type ST Offset   Length
[21:29:59]I (26) boot:  0 otadata          OTA data         01 00 00009000 00002000
[21:29:59]I (26) boot:  1 phy_init         RF data          01 01 0000b000 00001000
[21:29:59]I (27) boot:  2 app0             OTA app          00 10 00010000 001c0000
[21:29:59]I (27) boot:  3 app1             OTA app          00 11 001d0000 001c0000
[21:29:59]I (28) boot:  4 nvs              WiFi data        01 02 00390000 0006d000
[21:29:59]I (28) boot: End of partition table
[21:29:59]I (29) esp_image: segment 0: paddr=00010020 vaddr=420c0020 size=2009ch (131228) map
[21:29:59]I (54) esp_image: segment 1: paddr=000300c4 vaddr=40800000 size=0ff54h ( 65364) load
[21:29:59]I (69) esp_image: segment 2: paddr=00040020 vaddr=42000020 size=bb090h (766096) map
[21:29:59]I (213) esp_image: segment 3: paddr=000fb0b8 vaddr=4080ff54 size=0c18ch ( 49548) load
[21:29:59]I (229) boot: Loaded app from partition at offset 0x10000
[21:29:59]I (230) boot: Disabling RNG early entropy source...
[21:29:59][I][logger:165]: Log initialized
[21:29:59][C][safe_mode:084]: Unsuccessful boot attempts: 0
[21:29:59][D][esp32.preferences:143]: Writing 1 items: 0 cached, 1 written, 0 failed
[21:29:59][D][switch:020]: '' Turning ON.
[21:29:59][D][switch:063]: '': Sending state ON
[21:29:59]Guru Meditation Error: Core  0 panic'ed (Load access fault). Exception was unhandled.
[21:29:59]
[21:29:59]Core  0 register dump:
[21:29:59]MEPC    : 0x4200a47a  RA      : 0x4200a4f6  SP      : 0x4082a520  GP      : 0x408192d4  
[21:29:59]TP      : 0x4082a6f0  T0      : 0x40028192  T1      : 0x40829ffc  T2      : 0x0000000a  
[21:29:59]S0/FP   : 0x00000000  S1      : 0x4081c000  A0      : 0x00000000  A1      : 0x00000006  
[21:29:59]A2      : 0x4082a54f  A3      : 0x00000001  A4      : 0x00000030  A5      : 0x00000001  
[21:29:59]A6      : 0x00000004  A7      : 0x0000000a  S2      : 0x40822000  S3      : 0x4081c110  
[21:29:59]S4      : 0x40825080  S5      : 0x40822000  S6      : 0x420cf000  S7      : 0x420c4000  
[21:29:59]S8      : 0x40822000  S9      : 0x40822000  S10     : 0x40822000  S11     : 0x40822000  
[21:29:59]T3      : 0x00000000  T4      : 0x420c3cd0  T5      : 0x0000003f  T6      : 0x00000020  
[21:29:59]MSTATUS : 0x00001881  MTVEC   : 0x40800001  MCAUSE  : 0x00000005  MTVAL   : 0x0000000c  
[21:29:59]MHARTID : 0x00000000  
[21:29:59]
[21:29:59]Stack memory:
[21:29:59]4082a520: 0x00000004 0x0000000a 0x40822000 0x4081c110 0x40825080 0x40822000 0x4082a610 0x420c4000
[21:29:59]4082a540: 0x40822000 0x40822000 0x40822000 0x01822000 0x00000000 0x420c3cd0 0x0000003f 0x4201edd6
[21:29:59]4082a560: 0xa5a5a5a5 0xa5a5a5a5 0xa5a5a5a5 0x40824cac 0xa5a5a5a5 0xa5a5a5a5 0x4082a610 0x00000001
[21:29:59]4082a580: 0x4082a610 0x00000001 0x4082a610 0x00000001 0x4082a610 0x00000001 0x4082a610 0x00000001
[21:29:59]4082a5a0: 0x4082a610 0x00000001 0x4082a610 0x00000001 0x4082a610 0x00000001 0x4082a610 0x00000001
[21:29:59]4082a5c0: 0x4082a610 0x00000001 0x4082a610 0x00000001 0x682d7067 0x6c2d6b6c 0x36343264 0xa5a50030
[21:29:59]4082a5e0: 0x4082a610 0x00000001 0x00000021 0xa5a5a5a5 0xa5a5a5a5 0xa5a5a5a5 0x40824604 0x00000013
[21:29:59]4082a600: 0x00000013 0xa5006361 0xa5a5a5a5 0xa5a5a5a5 0x40824dd0 0x00000006 0x62616e65 0x3600656c
[21:29:59]4082a620: 0x35333636 0xa5002d00 0x4082a630 0x00000004 0x30435455 0x00000000 0x63616131 0xa5a5a500
[21:29:59]4082a640: 0x4082a648 0x00000000 0x6a7a6a00 0x36383336 0x35333636 0x00000000 0x00000000 0x008192d4
[21:29:59]4082a660: 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000
[21:29:59]4082a680: 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000
[21:29:59]4082a6a0: 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x42007c0a
[21:29:59]4082a6c0: 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000
[21:29:59]4082a6e0: 0x00000000 0xa5a5a5a5 0xa5a5a5a5 0xa5a5a5a5 0xa5a5a5a5 0x00000150 0x4082a4e0 0x00000068
[21:29:59]4082a700: 0x4081cdd8 0x4081cdd8 0x4082a6f8 0x4081cdd0 0x00000018 0x00000000 0x00000000 0x4082a6f8
[21:29:59]4082a720: 0x00000000 0x00000001 0x408286f4 0x706f6f6c 0x6b736154 0x00000000 0x00000000 0x4082a6f0
[21:29:59]4082a740: 0x00000001 0x00000000 0x00000000 0x00000000 0x00000000 0x40822b78 0x40822be0 0x40822c48
[21:29:59]4082a760: 0x00000000 0x00000000 0x00000001 0x00000000 0x00000000 0x00000000 0x4002849c 0x00000000
[21:29:59]4082a780: 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000
[21:29:59]4082a7a0: 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000
[21:29:59]4082a7c0: 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000
[21:29:59]4082a7e0: 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000
[21:29:59]4082a800: 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000
[21:29:59]4082a820: 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000
[21:29:59]4082a840: 0x00000000 0x00000000 0x0000000c 0x4082a85c 0x00000000 0x4082a844 0x00000014 0x0c160108
[21:29:59]4082a860: 0x00000000 0x00000000 0x99e59f84 0x4082a854 0x00000010 0x00000000 0x4082a6f8 0x00000000
[21:29:59]4082a880: 0x00000000 0x00000020 0x682d7067 0x6c2d6b6c 0x36343264 0x63322d30 0x63616131 0x420d5800
[21:29:59]4082a8a0: 0x420c159c 0x4082a880 0x00000024 0x420d4bd4 0x420d4c04 0x420c07f8 0x00000032 0x4082b644
[21:29:59]4082a8c0: 0xa89c0000 0xe6a0bce4 0x99e59f84 0x4082a8a4 0x00000048 0xe68db8e4 0xaee5af98 0x858ee5a2
[21:29:59]4082a8e0: 0x3432444c 0xbae43036 0xa89ce5ba 0xe6a0bce4 0x99e59f84 0x633220a8 0x63616131 0x3b305b00
[21:29:59]4082a900: 0x5b6d3633 0x735b5d44 0x63746977 0x36303a68 0x203a5d33 0x203a2727 0x4082a8cc 0x0000005c
```
setup方法中的LOG根本没有，说明在初始化之前就出现了崩溃。
看到消息日志的存储信息，根据PC/MEPC和RA指针定位崩溃位置，[参考这里](https://docs.espressif.com/projects/esp-techpedia/zh_CN/latest/esp-friends/advanced-development/debugging/guru-meditation.html#esp)
和[这里](https://docs.espressif.com/projects/esp-techpedia/zh_CN/latest/esp-friends/advanced-development/debugging/backtrace-coredump.html)

esphome下载ELF文件，这个有用。
使用```riscv32-esp-elf-objdump -S xxx.elf > a.txt```进行反汇编，得到反汇编文件，之后有用。
使用```riscv32-esp-elf-addr2line -pfiaC -e path.elf 0x4200a47a 0x4200a4f6```，查询PC指针和RA指针是什么，得到崩溃位置的函数，
```
riscv32-esp-elf-addr2line -pfiaC -e "gp-hlk-ld2460 (2).elf"  0x4200a47a  0x4200a4f6
0x4200a47a: esphome::ld2460::LD2460Component::send_command(unsigned char, unsigned char const*, unsigned short) at /config/.esphome/build/gp-hlk-ld2460/src/esphome/components/ld2460/ld2460.cpp:395
0x4200a4f6: esphome::ld2460::LD2460Component::enable_upload(bool) at /config/.esphome/build/gp-hlk-ld2460/src/esphome/components/ld2460/ld2460.cpp:285
```
汇编文件中
```aiignore
4200a474 <_ZN7esphome6ld246015LD2460Component12send_commandEhPKht>:
4200a474:	1101                	addi	sp,sp,-32
4200a476:	cc22                	sw	s0,24(sp)
4200a478:	842a                	mv	s0,a0
4200a47a:	4548                	lw	a0,12(a0)  # 崩溃位置
4200a47c:	c452                	sw	s4,8(sp)
4200a47e:	8a2e                	mv	s4,a1
4200a480:	ca26                	sw	s1,20(sp)
4200a482:	81c18593          	addi	a1,gp,-2020 # 40818af0 <_ZN7esphome6ld2460L15LD2460_CMD_HEADE>
4200a486:	84b2                	mv	s1,a2
4200a488:	4611                	li	a2,4
4200a48a:	ce06                	sw	ra,28(sp)
4200a48c:	c84a                	sw	s2,16(sp)
4200a48e:	c64e                	sw	s3,12(sp)
4200a490:	8936                	mv	s2,a3
4200a492:	00b68993          	addi	s3,a3,11
4200a496:	d1bff0ef          	jal	4200a1b0 <_ZN7esphome4uart10UARTDevice11write_arrayEPKhj.isra.0>
4200a49a:	4448                	lw	a0,12(s0)
4200a49c:	85d2                	mv	a1,s4
4200a49e:	09c2                	slli	s3,s3,0x10
4200a4a0:	d17ff0ef          	jal	4200a1b6 <_ZN7esphome4uart10UARTDevice10write_byteEh.isra.0>
4200a4a4:	4448                	lw	a0,12(s0)
4200a4a6:	0109d993          	srli	s3,s3,0x10
4200a4aa:	0ff9f593          	zext.b	a1,s3
4200a4ae:	d09ff0ef          	jal	4200a1b6 <_ZN7esphome4uart10UARTDevice10write_byteEh.isra.0>
4200a4b2:	4448                	lw	a0,12(s0)
4200a4b4:	0089d593          	srli	a1,s3,0x8
4200a4b8:	cffff0ef          	jal	4200a1b6 <_ZN7esphome4uart10UARTDevice10write_byteEh.isra.0>
4200a4bc:	c491                	beqz	s1,4200a4c8 <_ZN7esphome6ld246015LD2460Component12send_commandEhPKht+0x54>
4200a4be:	4448                	lw	a0,12(s0)
4200a4c0:	864a                	mv	a2,s2
4200a4c2:	85a6                	mv	a1,s1
4200a4c4:	cedff0ef          	jal	4200a1b0 <_ZN7esphome4uart10UARTDevice11write_arrayEPKhj.isra.0>
4200a4c8:	4448                	lw	a0,12(s0)
4200a4ca:	4462                	lw	s0,24(sp)
4200a4cc:	40f2                	lw	ra,28(sp)
4200a4ce:	44d2                	lw	s1,20(sp)
4200a4d0:	4942                	lw	s2,16(sp)
4200a4d2:	49b2                	lw	s3,12(sp)
4200a4d4:	4a22                	lw	s4,8(sp)
4200a4d6:	4611                	li	a2,4
4200a4d8:	81818593          	addi	a1,gp,-2024 # 40818aec <_ZN7esphome6ld2460L15LD2460_CMD_TAILE>
4200a4dc:	6105                	addi	sp,sp,32
4200a4de:	cd3ff06f          	j	4200a1b0 <_ZN7esphome4uart10UARTDevice11write_arrayEPKhj.isra.0>

4200a4e2 <_ZN7esphome6ld246015LD2460Component13enable_uploadEb>:
4200a4e2:	1101                	addi	sp,sp,-32
4200a4e4:	00b107a3          	sb	a1,15(sp)
4200a4e8:	00f10613          	addi	a2,sp,15
4200a4ec:	4685                	li	a3,1
4200a4ee:	4599                	li	a1,6
4200a4f0:	ce06                	sw	ra,28(sp)
4200a4f2:	f83ff0ef          	jal	4200a474 <_ZN7esphome6ld246015LD2460Component12send_commandEhPKht>
4200a4f6:	40f2                	lw	ra,28(sp)
4200a4f8:	6105                	addi	sp,sp,32
4200a4fa:	8082                	ret"""
```
是send_command方法中崩溃的，而它是enable_upload调用的。查询哪调用了enable_upload，在yaml文件和全部代码中分析，只能是EnableUploadSwitch调用的。

查看A0,A1等寄存器，MTVAL接近0，可能是读取了一个```NULL```附近的地址。
比如把```nullptr```当对象访问了他的成员，此时A0是0，即 *this指针是nullptr* ，A1是6，
符合enable_upload调用send_command的command参数的值，A3是1，符合data_size参数的值，所以定位崩溃函数enable_upload。
```cpp
void LD2460Component::enable_upload(bool enable) {
  uint8_t data = enable ? 0x01 : 0x00;
  this->send_command(0x06, &data, 1);
}
```

最终原因是对nullptr调用了他的enable_upload方法，导致了崩溃。为什么this指针是nullptr？
```cpp
#pragma once

#include "esphome/components/switch/switch.h"
#include "../ld2460.h"

namespace esphome {
namespace ld2460 {

class EnableUploadSwitch : public switch_::Switch, public Parented<LD2460Component> {
 public:
  EnableUploadSwitch() {
    this->turn_on();
  }
  
 protected:
  void write_state(bool state) override;
};

}  // namespace ld2460
}  // namespace esphome
```
```python
# switch/__init__.py
async def to_code(config):
    ld2460_component = await cg.get_variable(config[CONF_LD2460_ID])

    if enable_upload_config := config.get(CONF_ENABLE_UPLOAD):
        s = await switch.new_switch(enable_upload_config)
        await cg.register_parented(s, config[CONF_LD2460_ID])
        cg.add(ld2460_component.set_enable_upload_switch(s))
```
看py的codegen文件，原来这个Switch实例是先构造（```switch.new_switch```），再赋值```this->parent_ = parent;```，
构造的时候```this->parent_```为nullptr，但是这时候就开始调用它的方法了(```turn_on```会调用```write_state```，后面涉及到对parent_的调用)，
这显然是造成崩溃的罪魁祸首。
因此，不要在Switch、Button、Select类的构造方法中调用parent的方法，弄个成员列表初始化即可。

当然你硬要在构造方法调用parent的方法也行，new_xxx的时候parent就应该作为构造方法的参数传入。
这样后面就不需要```register_parented```了，构造方法中就可以拿到parent对象。
这两种方法都可以，看esphome官方这两种方法都有。

另外idf5.4似乎还整了个活，如果在完全初始化之前调用uart、i2c、spi的write之类的方法。
多半会得到一个freertos底层的新信错误，这个没法修，只能进一步推迟执行时间到loop方法开始执行的时候,这可以用一个队列实现。

### 2025.12.7更新
官方的[好东西](https://esphome.github.io/esp-stacktrace-decoder/)，直接查看调用栈