# Nand
| a   | b   | out |
| --- | --- | --- |
| 0   | 0   | 1   |
| 1   | 0   | 1   |
| 0   | 1   | 1   |
| 1   | 1   | 0   |

# And

| a   | b   | out |
| --- | --- | --- |
| 0   | 0   | 0   |
| 1   | 0   | 0   |
| 0   | 1   | 0   |
| 1   | 1   | 1    |

Nand(a=a, b=b, out=out1)
Nand(a=out1, b=out1, out=out)

# And16
IN a\[16], b\[16];
OUT out\[16];

每一位都做And操作


# DMux
| in  | sel | a   | b   |
| --- | --- | --- | --- |
| 0   | 0   | 0   | 0   |
| 1   | 0   | 1   | 0   |
| 0   | 1   | 0   | 0   |
| 1   | 1   | 0   | 1   |

a只有在in为1，sel为0的情况下为1
b只有在in为1，sel为1的情况下为1

# DMux4Way
```c
/**
 * 4-way demultiplexor:
 * {a, b, c, d} = {in, 0, 0, 0} if sel == 00
 *                {0, in, 0, 0} if sel == 01
 *                {0, 0, in, 0} if sel == 10
 *                {0, 0, 0, in} if sel == 11
 */
```

a：sel为00，in为1才为1
