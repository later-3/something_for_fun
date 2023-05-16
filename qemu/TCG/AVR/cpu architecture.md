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

avr_load_firmware

load_elf32(const char * name, int fd, uint64_t (*)(void *, void *, _Bool) elf_note_fn, uint64_t (*)(void *, uint64_t) translate_fn, void * translate_opaque, int must_swab, uint64_t * pentry, uint64_t * lowaddr, uint64_t * highaddr, uint32_t * pflags, int elf_machine, int clear_lsb, int data_swab, AddressSpace * as, _Bool load_rom, symbol_fn_t sym_cb) (\root\Code\qemu\include\hw\elf_ops.h:396)
load_elf_ram_sym(const char * filename, uint64_t (*)(void *, void *, _Bool) elf_note_fn, uint64_t (*)(void *, uint64_t) translate_fn, void * translate_opaque, uint64_t * pentry, uint64_t * lowaddr, uint64_t * highaddr, uint32_t * pflags, int big_endian, int elf_machine, int clear_lsb, int data_swab, AddressSpace * as, _Bool load_rom, symbol_fn_t sym_cb) (\root\Code\qemu\hw\core\loader.c:502)
avr_load_firmware(AVRCPU * cpu, MachineState * ms, MemoryRegion * program_mr, const char * firmware) (\root\Code\qemu\hw\avr\boot.c:74)
arduino_machine_init(MachineState * machine) (\root\Code\qemu\hw\avr\arduino.c:52)
machine_run_board_init(MachineState * machine, const char * mem_path, Error ** errp) (\root\Code\qemu\hw\core\machine.c:1414)
qemu_init_board() (\root\Code\qemu\softmmu\vl.c:2526)
qmp_x_exit_preconfig(Error ** errp) (\root\Code\qemu\softmmu\vl.c:2622)
qemu_init(int argc, char ** argv) (\root\Code\qemu\softmmu\vl.c:3630)
main(int argc, char ** argv) (\root\Code\qemu\softmmu\main.c:47)
```

在avr的的machine init中会调用qemu的load_elf_ram_sym函数，去加载elf中的text段，查看下text的内容：
```shell
avr-objdump -DSs demo.elf>demo.txt

Contents of section .text:
 0000 0c948800 0c94a900 0c94a900 0c94a900  ................
 0010 0c94a900 0c94a900 0c94a900 0c94a900  ................
 0020 0c94a900 0c94a900 0c94a900 0c94a900  ................
 0030 0c94a900 0c94a900 0c94a900 0c94a900  ................
 
Disassembly of section .text:

00000000 <__vectors>:
       0:	0c 94 88 00 	jmp	0x110	; 0x110 <__ctors_end>
       4:	0c 94 a9 00 	jmp	0x152	; 0x152 <__bad_interrupt>
       8:	0c 94 a9 00 	jmp	0x152	; 0x152 <__bad_interrupt>
       c:	0c 94 a9 00 	jmp	0x152	; 0x152 <__bad_interrupt>
      10:	0c 94 a9 00 	jmp	0x152	; 0x152 <__bad_interrupt>
      14:	0c 94 a9 00 	jmp	0x152	; 0x152 <__bad_interrupt>
      18:	0c 94 a9 00 	jmp	0x152	; 0x152 <__bad_interrupt>
      1c:	0c 94 a9 00 	jmp	0x152	; 0x152 <__bad_interrupt>
```

