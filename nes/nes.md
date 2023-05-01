# NES hardware architecture
https://famicom.party/book/04-hardwareoverview/
If you were to open up an NES console and take a look at its insides, you would see something like this:
![](https://www.copetti.org/images/consoles/nes/_hu31e3638c2fc7a9de9f39a1bcb4bd9501_354137_7dbeaac9fb0451009e405b875830ef5e.webp)
Visually, the NES motherboard is dominated by the (unused) expansion slot at the top and two large chips. On the left side is this chip, labelled "RP2A03", it includes a 6502 CPU and a APU. And on the right side is this other chip, labelled "RP2C02" -- PPU.


![](https://www.copetti.org/images/consoles/nes/_hu05662e9582b538ada21b1313ba043bdc_267_a1cb762036ab1714936b0b8b329ac461.webp)

## Cartidges

NES games are distributed via plastic cartridges. Inside each cartridge is a small circuit board that looks something like this:

![|](https://famicom.party/processed_images/1803686258d292b900.jpg)

![](https://www.copetti.org/images/consoles/nes/_hu901c88f2e32815a78c9694e3a21f7cdf_72344_512bbe6425cebdc421279d68dfb42911.webp)

Much like the console motherboard, cartridge circuit boards are dominated by two large chips. The left-side chip is labelled "PRG", and the right-side chip is labelled "CHR". PRG-ROM is "program ROM", and it contains all of the game's code (machine code). CHR-ROM is "character ROM", which holds all of the game's graphics data.

These two ROM chips have their pins connected to a series of gold connectors at the edge of the cartridge board. When the cartridge is inserted into the console's cartridge slot, the gold connectors make electrical contact with an identical set of connectors in the console. The result is that the PRG-ROM is electrically wired directly to the 2A03 CPU, and the CHR-ROM is wired directly to the 2C02 PPU.

A diagram showing how the cartridge's ROM chips connect to the console's processors：

![](https://famicom.party/processed_images/a60b5e4c195a267c00.jpg)


# cpu address space

![|800](https://bugzmanov.github.io/nes_ebook/images/ch3/cpu_registers_memory.png)


![](https://www.zupimages.net/up/20/34/ffkd.png)


# iNES structure

![|800](https://bugzmanov.github.io/nes_ebook/images/ch5/image_2_ines_file_format.png)

##  file header
![](https://bugzmanov.github.io/nes_ebook/images/ch5/image_3_ines_header.png)


# cpu instruction

https://www.nesdev.org/obelisk-6502-guide/instructions.html

http://www.oxyron.de/html/opcodes02.html
http://visual6502.org/

https://www.qmtpro.com/~nes/chipimages/visual2a03/

https://bugzmanov.github.io/nes_ebook/chapter_3.html

https://c64os.com/post/6502instructions
https://skilldrick.github.io/easy6502/
https://www.copetti.org/writings/consoles/nes/
https://famicom.party/book/04-hardwareoverview/
https://taywee.github.io/NerdyNights/nerdynights/nesarchitecture.html



