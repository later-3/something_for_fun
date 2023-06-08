# registers
https://www.nesdev.org/wiki/CPU_registers
### Accumulator

**A** is byte-wide and along with the [arithmetic logic unit](https://en.wikipedia.org/wiki/arithmetic_logic_unit "wikipedia:arithmetic logic unit") (ALU), supports using the status register for carrying, overflow detection, and so on.
The **Accumulator** (A) is the main register that instructions operate on. Many instructions use it in their inputs or write their result to it. It is also useful as a temporary storage area when moving data from one memory location to another.

### Indexes
**X** and **Y** are byte-wide and used for several addressing modes.

### Program Counter
The 2-byte program counter **PC**

### Stack Pointer
**S**

### Status Register
**P** 

[Programming the NES: The 6502 in detail (middle-engine.com)](https://www.middle-engine.com/blog/posts/2020/06/23/programming-the-nes-the-6502-in-detail)

如上述，一共6个寄存器。


# 6502 addressing modes
13 addressing modes
https://www.nesdev.org/obelisk-6502-guide/addressing.html

### Implicit
### Accumulator
### Immediate
### Zero Page
### Zero Page,X
### Zero Page,Y
### Relative
### Absolute
### Absolute,X
### Absolute,Y
### Indirect
### **Indirect X**
### **Indirect Y**


# instruction set
https://www.masswerk.at/6502/6502_instruction_set.html
里面的status register的bit位的位置与osdev wiki里面的不一样，litenes的代码与osdev wiki里面一致。

## litesnes cpu instruction implementation
6205的opcode matrix是一个二维的，以4个bit为单位，一个字节的opcode分成了行和列。在litenes中，用了一个长度为256的数组来完成对opcode的解析，比如0x78就不分为行和列了，直接就是数组的索引0x78里面保存了，该opcode需要完成的事情。

```c
void (*cpu_op_address_mode[256])(); // Array of address modes
void (*cpu_op_handler[256])(); // Array of instruction function pointers
bool cpu_op_in_base_instruction_set[256]; // true if instruction is in base 6502 instruction set
char *cpu_op_name[256]; // Instruction names
int cpu_op_cycles[256]; // CPU cycles used by instructions
```

根据不同的地址模式，获取操作数。同时，改变pc，在执行opcode时，也需要完成设置状态寄存器。


加载rom后，会将cpu reset到 65,532 -> 0xFFFC
rom的前16字节是ines头，后是prg rom的空间，以0x4000为粒度，在头文件中有描述prg block的数量，马里奥是2，所以是0x8000,在litenes中，有一个大小为0x10000的memery数组，在load rom的时候，将prg rom的内容读取到了0x8000的位置，其实这个memery是cpu的寻址空间，按照寻址空间的划分，0x8000的位置就是prg rom的位置。 读取0xfffc位置，也就是0xfffc-0x8000的位置，再对应到nes rom中的偏移就是，0x800c

这是rom中的第一条指令，在马里奥rom中是 00 80，将80往左移8bit，与00做与运算，得到rom中第一指令的地址，是小端模式。

第一条指令就是偏移为0x10的指令，就是prg rom的第一条指令： 0x78 (注意opcode只有一个字节)

然后查表，行为7，列为8找到单元格: SEI impl

https://www.nesdev.org/wiki/Status_flags
7  bit  0
---- ----
NVss DIZC
|||| ||||
|||| |||+- [Carry](https://www.nesdev.org/wiki/Status_flags#C:_Carry)
|||| ||+-- [Zero](https://www.nesdev.org/wiki/Status_flags#Z:_Zero)
|||| |+--- [Interrupt Disable](https://www.nesdev.org/wiki/Status_flags#I:_Interrupt_Disable)
|||| +---- [Decimal](https://www.nesdev.org/wiki/Status_flags#D:_Decimal)
||++------ No CPU effect, see: [the B flag](https://www.nesdev.org/wiki/Status_flags#The_B_flag)
|+-------- [Overflow](https://www.nesdev.org/wiki/Status_flags#V:_Overflow)
+--------- [Negative](https://www.nesdev.org/wiki/Status_flags#N:_Negative)

上面是status register P 的每个bit含义。

在litenes中 sei 指令就只做了 设置P为2一件事。
接下来的指令是0xD8, 和sei一样，只是做了设置状态寄存器。

再下来是0xA9 LDA \#
这个是立即数寻址，立即数就跟在A9后面，立即数的A9(LDA)，有一个操作数，也就是这条指令是两字节的，所以需要进一步读出下一个字节，并将pc++，这也是立即数寻址模式需要做的事情。
