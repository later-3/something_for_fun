# PPU
PPU：picture processing unit，生成一个合成的视频信号，由像素组成的240条线，输出到电视。Famicon(family computer)的芯片组是在20世纪80年代初被设计出来时，它被认为是相当先进的2D视频游戏图像生成器。

它有自己的地址空间，通常包含10K的内存:8k在卡带中的rom或ram，可能有一个或多个通用的mapper，用于存储背景形状和精灵切片，还有2K在卡带机中的 ram，用于存储一个或两个map。两个独立的较小的地址空间保存着一个调色板，用于控制与各种索引相关联的颜色，以及 OAM (对象属性存储器) ，用于存储精灵或独立移动对象的位置、方向、形状和颜色。这些是 PPU 本身的内部，而调色板由静态内存组成，OAM 使用动态内存(如果 PPU 不渲染，动态内存将缓慢衰减)。

# PPU Registers
![](https://bugzmanov.github.io/nes_ebook/images/ch6.1/image_1_ppu_registers_memory.png)

https://www.nesdev.org/wiki/PPU_registers
PPU有8个寄存器，memory-mapped to the CPU， 这里就和arm的寻址空间，其中一部分映射到了I/O设备上一样的道理。在CPU的0x2000到0x20007寻找空间对应PPU的8个寄存器，然后这8个字节又从0x2008到0x3FFF范围内镜像了，所以写0x3456和写0x2006是一样的。这部分可以参考 https://www.nesdev.org/wiki/CPU_memory_map 

## 端口
PPU有一个内部的数据总线，用于和cpu交互。这种总线在 Visual 2C02中被称为 _ io _ db，在 FCEUX 被称为 PPugenlatch [1] ，它表现为一个8位动态锁存器，这是由于运行到 PPU 各个部分的非常长的跟踪电容所致。

# PPU power up state



