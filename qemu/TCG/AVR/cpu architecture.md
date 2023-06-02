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
qemu里面的machine是 mega2560 ，在初始化函数中，

下面是qemu加载avr的elf文件路径梳理：
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
=======

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

# AVR Core
为了最大化性能和并行度，AVR采用了哈弗结构，具有独立的存储器和用于程序和数据的总线。程序内存中的指令使用单级流水线执行。当一条指令正在执行时，下一条指令将从程序内存中预取。这个概念使指令能够在每个时钟周期中执行。内置的程序内存是可编程的闪存。

# 寄存器
有32个8bit的通用寄存器，在一个时钟周期访问。32个寄存器中的6个可以用作3个16位间接地址寄存器指针，用于数据空间寻址，从而实现高效的地址计算。其中一个地址指针还可以用作 Flash 程序内存中查找表的地址指针。这些添加的函数寄存器是16位的 X-、 Y-和 Z-寄存器。还有pc、stack pointer、status register、instruction register

# 算术逻辑单元
ALU支持数学和逻辑操作，这种操作是在寄存器之间或者常量和寄存器之间。在算术运算后，状态寄存器会被更新，去反应操作的结果。程序流程由条件和无条件跳转和调用指令提供，能够直接寻址整个地址空间。大多数 AVR 指令只有一个16位字格式。每个程序内存地址都包含一个16位或32位指令。

# 内存
AVR 体系结构中的内存空间都是线性和规则的内存映射。程序 Flash 存储空间分为两个部分，引导程序部分和应用程序部分。这两个部分都有专门用于写和读/写保护的锁位。写入 ApplicationFlash 内存部分的存储程序内存(Store Program Memory，SPM)指令必须驻留在启动程序部分。

在中断和子例程调用期间，返回地址 Program Counter (PC)存储在堆栈上。堆栈被有效地分配到一般数据 SRAM 中，因此，堆栈大小只受到 SRAM 总体大小和 SRAM 使用量的限制。

所有用户程序都必须在 Reset 例程中初始化堆栈指针(SP)(在执行子例程或中断之前)。SP 是
在 I/O 空间中可读/写的。数据 SRAM 可以很容易地通过 AVR 架构支持的五种不同寻址模式访问。

输入/输出内存空间包含64个 CPU 外围功能的地址，如控制寄存器、序列周边接口(SPI)和其他输入/输出功能。I/O 内存可以直接访问，或者作为寄存器文件0x20-0x5F 后面的数据空间位置访问。此外，该设备扩展了 SRAM 中0x60-0xFF 的 I/O 空间。

# 中断
一个灵活的中断模块在 I/O 空间中有它的控制寄存器，在状态寄存器中有一个额外的全局中断启用位。所有的中断都有一个单独的中断向量中断向量。中断根据它们的中断向量位置有优先权。中断向量地址越低优先级越高。

# AVR 状态寄存器
状态寄存器包含有关最近执行的算术指令的结果的信息。此信息可用于更改程序流以执行条件操作。状态寄存器在所有算术逻辑单元(ALU)操作之后更新。在许多情况下，这将消除使用专用比较指令的需要，从而产生更快、更紧凑的代码。

8个bit位分别代表：
- bit 7    I : Global Interrupt Enable
- bit 6   T : Copy Storage
- bit 5   H : Half Carry Flag
- bit 4   S : Sign Flag, S = N xor V
- bit 3   V : Two's Compliment Overflow Flag
- bit 2   N : Negative Flag
- bit 1   Z : Zero Flag
- bit 0   C : Carry Flag

# qemu avr cpu实现
`cpu.h` 里面定义了上面所介绍到的寄存器，定义使用的是 `uint32_t`：
```c
typedef struct CPUArchState {
    uint32_t pc_w; /* 0x003fffff up to 22 bits */
    uint32_t sregC; /* 0x00000001 1 bit */
    uint32_t sregZ; /* 0x00000001 1 bit */
    uint32_t sregN; /* 0x00000001 1 bit */
```
就像使用c语言来实现一个cpu一样，在qemu中，还需要和tcg结合起来。

`cpu.c` 里面实现了设置pc、获取pc等，还定义了几个类型的avr cpu，在`class init`中还初始化了一些`CPUClass`的钩子函数，来看看与tcg相关的:
```c
static const struct TCGCPUOps avr_tcg_ops = {
    .initialize = avr_cpu_tcg_init,
    .synchronize_from_tb = avr_cpu_synchronize_from_tb,
    .restore_state_to_opc = avr_restore_state_to_opc,
    .cpu_exec_interrupt = avr_cpu_exec_interrupt,
    .tlb_fill = avr_cpu_tlb_fill,
    .do_interrupt = avr_cpu_do_interrupt,
};
```

`avr_cpu_tcg_init` 的实现在 `tranlate.c` 中
```c
static TCGv cpu_pc;
static TCGv cpu_Cf;

#define NUMBER_OF_CPU_REGISTERS 32
static TCGv cpu_r[NUMBER_OF_CPU_REGISTERS];

void avr_cpu_tcg_init(void)
{
    int i;
#define AVR_REG_OFFS(x) offsetof(CPUAVRState, x)
    cpu_pc = tcg_global_mem_new_i32(cpu_env, AVR_REG_OFFS(pc_w), "pc");
    cpu_Cf = tcg_global_mem_new_i32(cpu_env, AVR_REG_OFFS(sregC), "Cf");
```

在 [Documentation/TCG/frontend-ops - QEMU](https://wiki.qemu.org/Documentation/TCG/frontend-ops) 有介绍到，模拟的cpu的绝大多数核心寄存器应该有一个对应的TCG 寄存器。

> Declare a named TCG register
> TCGv reg = tcg_global_mem_new(TCG_AREG0, offsetof(CPUState, reg), "reg"); 

这样创建了对应的tcg寄存器。
