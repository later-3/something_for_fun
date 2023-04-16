> ASIC

应用专用集成电路。

> Attribute

视频存储器的一部分，它选择调色板的哪一部分用于给定的精灵或背景区域。

> Audio Processing Unit (APU)

输出音频的硬件

> Bank switching

映射器的主要功能。由于 CPU 和 PPU 没有足够的地址空间一次看到整个 ROM，ROM 在概念上被划分为“页面”或“银行”，其中 ROM 地址的最重要位被视为一个页码。CPU 将一个页码写入到映射器的端口，映射器将这个页码放在 ROM 地址的最重要的位上，以使所选的页面对系统的其余部分可见。更先进的映射器具有“细粒度银行交换”，这使得独立的银行可以在地址空间的独立窗口中使用。

> Battery RAM

通过添加电池备份电路使 RAM 不易失效。这是用来让玩家保存多个章节的游戏进度。NES 游戏中几乎所有的电池内存都是 PRG 内存; 一些非常罕见的映射器上的游戏有电池支持的CHR内存。

> CGRAM (color generator RAM)

任天堂用来指调色板内存。

> CHR (character)

模式表的另一种说法，在传统的名字字符生成器之后，用于平铺的背景平面。

> CHR RAM

卡带上的 SRAM，通常为8192字节，通常映射为 $0000-$1FFF 并保存模式表。

> CHR ROM

ROM 在卡带中，它连接到 PPU，通常映射在 $0000-$1FFF 和持有模式表。

> IRQ interrupt request

> mapper

游戏包中的电路，可以地址解码和计数。

> mapper hack

rom的一种补丁，它使用一个映射器使其与另一种映射器一起工作。

>  memory management controller (mmc)

Nintendo 的ASIC mappers 名字

> Mirroring

在内存映射的多个位置存在一个内存区域。在 PPU 内存中，在命名表中使用镜像来水平或垂直地重复屏幕，因此使用区分术语“水平镜像”和“垂直镜像”。

> NMI

非可屏蔽中断。在 NES/Famicom 中用于垂直空白。

> Object Attribute Memory (OAM)

一个在PPU里面的，256字节的 DRAM，  持有精灵显示列表。

> palette

一个颜色查询表。

> PRG

program

>  PRG RAM

卡带中的ram，链接在cpu总线上。与之对比的是，在0x0000~0x07FF的内部ram，一般来说，PRG RAM一般在0x6000~0x7FFF.

> PRG ROM

CPU总线上的ROM，包含了一个要被执行的程序，和该程序要使用的数据。对于使用CHR RAM的游戏，ROM里面可能也包含了需要拷贝到 CHR RAM的数据。

> Picture Processing Unit (PPU)

生成视频信号

> trainer

在视频游戏中，trainer是添加作弊菜单的补丁。在 iNES 中，“训练器”是一个额外的512字节的 PRG ROM 片段，可用于添加了训练器的 ROM 映像，但更常用于支持例程，这些例程被用于过时的基于软盘的复印机的映射器黑客使用。