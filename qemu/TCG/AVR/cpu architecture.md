# reference
[8-Bit AVR® Core - Developer Help (microchipdeveloper.com)](https://microchipdeveloper.com/8avr:avrcore)
[AVR 8-Bit CPU Core : Arduino / ATmega328p - Arnab Kumar Das](https://www.arnabkumardas.com/arduino-tutorial/avr-cpu-core/)
# 总结
根据Wikipedia，AVR系列芯片在1996年被开发，是第一个使用片上可编程flash存储的微控制器。

## 可编程存储内存
MCU是8位的，但是每条指令占一个或两个16bit字。程序内存的大小通常用设备本身的命名来表示(例如，ATmega64x 行有64 KB 的闪存，而 ATmega32x 行有32 KB)。

qemu里面的machine是 mega2560 ，在初始化函数中，能找到：
```c
amc->flash_size = 256 * KiB;
amc->eeprom_size = 4 * KiB;
amc->sram_size = 8 * KiB;
```
可以看到也是符合上面的命名的，有256K的flash


## 顺手跟踪下 -bios
BA.elf 是用-bios选项传入的，在参数解析时，有如下处理：

```c
qemu-options.hx
DEF("bios", HAS_ARG, QEMU_OPTION_bios, \
    "-bios file      set the filename for the BIOS\n", QEMU_ARCH_ALL)
SRST
``-bios file``
    Set the filename for the BIOS.
ERST

vl.c
case QEMU_OPTION_bios:
    qdict_put_str(machine_opts_dict, "firmware", optarg);
    break;
```

arduino.c里面是machine，atmega.c里面是mcu，有如下调用关系：
> arduino_machine_init -> avr_load_firmware 

## 内部数据内存
数据地址空间由寄存器、IO 寄存器和sram组成。

## 内部寄存器
AVRs 有32个8位寄存器