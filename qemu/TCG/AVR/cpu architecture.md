# reference
[8-Bit AVR® Core - Developer Help (microchipdeveloper.com)](https://microchipdeveloper.com/8avr:avrcore)
[AVR 8-Bit CPU Core : Arduino / ATmega328p - Arnab Kumar Das](https://www.arnabkumardas.com/arduino-tutorial/avr-cpu-core/)
# 总结
根据Wikipedia，AVR系列芯片在1996年被开发，是第一个使用片上可编程flash存储的微控制器。

## 可编程存储内存
MCU是8位的，但是每条指令占一个或两个16bit字。程序内存的大小通常用设备本身的命名来表示(例如，ATmega64x 行有64 KB 的闪存，而 ATmega32x 行有32 KB)。

qemu里面的machine是 mega2560 ，在初始化函数中，能找到：

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