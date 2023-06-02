Build and simulate all the chips listed in nand2tetris/projects/01.

# Not.hdl

> 使用Nand实现not，因为nand是只有同时为1输出才为0，其余都为0，直接将not的输入给到nand，也就是0 0 -> 1，1 1 -> 0，这样完成了not

```hdl
/**
 * Not gate:
 * out = not in
 */

CHIP Not {
    IN in;
    OUT out;

    PARTS:
    // Put your code here:
    Nand(a=in, b=in, out=out);
}
```
![](https://cdn.jsdelivr.net/gh/later-3/blog-img/20230505080337.png)

# And.hdl
> 因为nand就是not and，所以在nand后面再接一个not，就变成了and

```
/**
 * And gate: 
 * out = 1 if (a == 1 and b == 1)
 *       0 otherwise
 */

CHIP And {
    IN a, b;
    OUT out;

    PARTS:
    // Put your code here:
    Nand(a=a, b=b, out=out1);
    Not(in=out1, out=out);
}
```
![](https://cdn.jsdelivr.net/gh/later-3/blog-img/20230505080304.png)

16路AND就是写16行，每行都是每个位的And

# Or.hdl
or的真值表是:
```
0 0 1 1
0 1 0 1
0 1 1 1
```
去表示第一列，只有都为0的时候才为0. 都取反，然后走nand，因为nand是都为1就为0. 也可以根据高电平里面的那张图，将NOR的值取反
```
 /**
 * Or gate:
 * out = 1 if (a == 1 or b == 1)
 *       0 otherwise
 */

CHIP Or {
    IN a, b;
    OUT out;

    PARTS:
    // Put your code here:
    Not(in=a,out=n1);
    Not(in=b,out=n2);
    Nand(a=n1,b=n2,out=out);
}
```
![](https://cdn.jsdelivr.net/gh/later-3/blog-img/20230505083600.png)

# XOR
truth table
```
0 0 1 1
0 1 0 1
0 1 1 0
```

使用书上提到的方法，只看为1的列，那就是将第一个取反，与第二个and操作，然后将第二个取反，与第一个and操作，两个再或起来。
```
/**
 * Exclusive-or gate:
 * out = not (a == b)
 */

CHIP Xor {
    IN a, b;
    OUT out;

    PARTS:
    // Put your code here:
    Not(in=a,out=n1);
    And(a=n1,b=b,out=a1);

    Not(in=b,out=n2);
    And(a=a,b=n2,out=a2);

    Or(a=a1,b=a2,out=out);
}
```

# Multiplexer
多路输入选择器是一个三输入电路。两个输入分别为a、b，被称为数据位，第三个输入位是sel，被称为选择位。多选择器通过sel去选择和输出a或者b。所以，一个更容易理解的名称是选择器。sel为0时，选择输出a的值，sel为1时，选择输出b的值。
根据真值表：
![|300](https://cdn.jsdelivr.net/gh/later-3/blog-img/20230505210627.png)
(a and not(b) and not(sel)) or (a and b and not(sel)) or (not(a) and b and sel) or (a and b and sel)
```
CHIP Mux {
    IN a, b, sel;
    OUT out;

    PARTS:
    // Put your code here:
    And(a=a,b=b,out=and1);
    And(a=and1,b=sel,out=and2);

    Not(in=b,out=not1);
    Not(in=sel,out=not2);
    And(a=a,b=not1,out=and3);
    And(a=and3,b=not2,out=and4);

    And(a=a,b=b,out=and5);
    And(a=and5,b=not2,out=and6);

    Not(in=a,out=not3);
    And(a=not3,b=b,out=and7);
    And(a=and7,b=sel,out=and8);

    Or(a=and2,b=and4,out=or1);
    Or(a=or1,b=and6,out=or2);
    Or(a=or2,b=and8,out=out);
}
```
应该是可以化简的

## 4路选择器
```
sel a b c d out
0 0 1 0 0 0 1
0 1 0 1 0 0 1
1 0 0 0 1 0 1
1 1 0 0 0 1 1
```
not(sel\[0\]) and not(sel\[1\]) and a\[0\] 这是选择a，把其余的选择或起来。
```
CHIP Mux4Way16 {
    IN a[16], b[16], c[16], d[16], sel[2];
    OUT out[16];

    PARTS:
    // Put your code here:
    Not(in=sel[0],out=not1);
    Not(in=sel[1],out=not2);

    And(a=not1,b=not2,out=and1); // 0 0
    And(a=and1,b=a[0],out=and11); 
    And(a=and1,b=a[1],out=and12); 
    And(a=and1,b=a[2],out=and13); 
    And(a=and1,b=a[3],out=and14); 
    And(a=and1,b=a[4],out=and15); 
    And(a=and1,b=a[5],out=and16); 
    And(a=and1,b=a[6],out=and17); 
    And(a=and1,b=a[7],out=and18); 
    And(a=and1,b=a[8],out=and19); 
    And(a=and1,b=a[9],out=and110); 
    And(a=and1,b=a[10],out=and111); 
    And(a=and1,b=a[11],out=and112); 
    And(a=and1,b=a[12],out=and113); 
    And(a=and1,b=a[13],out=and114); 
    And(a=and1,b=a[14],out=and115); 
    And(a=and1,b=a[15],out=and116); 

    And(a=sel[0],b=not2,out=and3); // 0 1
    And(a=and3,b=b[0],out=and31);
    And(a=and3,b=b[1],out=and32);
    And(a=and3,b=b[2],out=and33);
    And(a=and3,b=b[3],out=and34);
    And(a=and3,b=b[4],out=and35);
    And(a=and3,b=b[5],out=and36);
    And(a=and3,b=b[6],out=and37);
    And(a=and3,b=b[7],out=and38);
    And(a=and3,b=b[8],out=and39);
    And(a=and3,b=b[9],out=and310);
    And(a=and3,b=b[10],out=and311);
    And(a=and3,b=b[11],out=and312);
    And(a=and3,b=b[12],out=and313);
    And(a=and3,b=b[13],out=and314);
    And(a=and3,b=b[14],out=and315);
    And(a=and3,b=b[15],out=and316);

    And(a=sel[1],b=not1,out=and5); // 1 0
    And(a=and5,b=c[0],out=and51);
    And(a=and5,b=c[1],out=and52);
    And(a=and5,b=c[2],out=and53);
    And(a=and5,b=c[3],out=and54);
    And(a=and5,b=c[4],out=and55);
    And(a=and5,b=c[5],out=and56);
    And(a=and5,b=c[6],out=and57);
    And(a=and5,b=c[7],out=and58);
    And(a=and5,b=c[8],out=and59);
    And(a=and5,b=c[9],out=and510);
    And(a=and5,b=c[10],out=and511);
    And(a=and5,b=c[11],out=and512);
    And(a=and5,b=c[12],out=and513);
    And(a=and5,b=c[13],out=and514);
    And(a=and5,b=c[14],out=and515);
    And(a=and5,b=c[15],out=and516);

    And(a=sel[0],b=sel[1],out=and7); // 1 1
    And(a=and7,b=d[0],out=and81);
    And(a=and7,b=d[1],out=and82);
    And(a=and7,b=d[2],out=and83);
    And(a=and7,b=d[3],out=and84);
    And(a=and7,b=d[4],out=and85);
    And(a=and7,b=d[5],out=and86);
    And(a=and7,b=d[6],out=and87);
    And(a=and7,b=d[7],out=and88);
    And(a=and7,b=d[8],out=and89);
    And(a=and7,b=d[9],out=and810);
    And(a=and7,b=d[10],out=and811);
    And(a=and7,b=d[11],out=and812);
    And(a=and7,b=d[12],out=and813);
    And(a=and7,b=d[13],out=and814);
    And(a=and7,b=d[14],out=and815);
    And(a=and7,b=d[15],out=and816);

    Or(a=and11,b=and31,out=or11);
    Or(a=and51,b=and81,out=or21);
    Or(a=or11,b=or21,out=out[0]);

    Or(a=and12,b=and32,out=or221);
    Or(a=and52,b=and82,out=or222);
    Or(a=or221,b=or222,out=out[1]);

    Or(a=and13,b=and33,out=or31);
    Or(a=and53,b=and83,out=or32);
    Or(a=or31,b=or32,out=out[2]);

    Or(a=and14,b=and34,out=or41);
    Or(a=and54,b=and84,out=or42);
    Or(a=or41,b=or42,out=out[3]);

    Or(a=and15,b=and35,out=or51);
    Or(a=and55,b=and85,out=or52);
    Or(a=or51,b=or52,out=out[4]);

    Or(a=and16,b=and36,out=or61);
    Or(a=and56,b=and86,out=or62);
    Or(a=or61,b=or62,out=out[5]);

    Or(a=and17,b=and37,out=or71);
    Or(a=and57,b=and87,out=or72);
    Or(a=or71,b=or72,out=out[6]);

    Or(a=and18,b=and38,out=or81);
    Or(a=and58,b=and88,out=or82);
    Or(a=or81,b=or82,out=out[7]);

    Or(a=and19,b=and39,out=or91);
    Or(a=and59,b=and89,out=or92);
    Or(a=or91,b=or92,out=out[8]);

    Or(a=and110,b=and310,out=or111);
    Or(a=and510,b=and810,out=or112);
    Or(a=or111,b=or112,out=out[9]);

    Or(a=and111,b=and311,out=or121);
    Or(a=and511,b=and811,out=or122);
    Or(a=or121,b=or122,out=out[10]);

    Or(a=and112,b=and312,out=or131);
    Or(a=and512,b=and812,out=or132);
    Or(a=or131,b=or132,out=out[11]);

    Or(a=and113,b=and313,out=or1331);
    Or(a=and513,b=and813,out=or1332);
    Or(a=or1331,b=or1332,out=out[12]);

    Or(a=and114,b=and314,out=or141);
    Or(a=and514,b=and814,out=or142);
    Or(a=or141,b=or142,out=out[13]);

    Or(a=and115,b=and315,out=or151);
    Or(a=and515,b=and815,out=or152);
    Or(a=or151,b=or152,out=out[14]);

    Or(a=and116,b=and316,out=or161);
    Or(a=and516,b=and816,out=or162);
    Or(a=or161,b=or162,out=out[15]);
}
```



# Demultiplexer
多路输出选择器，sel为0，把输入给到a，sel为1，把输入给到b。一个输入in，一个选择sel，两个输出。
![|300](https://cdn.jsdelivr.net/gh/later-3/blog-img/20230505210648.png)
```

sel in  a  b
0   0   0  0
0   1   1  0
1   0   0  0
1   1   0  1
a = (not(sel) and in)
b = (sel and in)

CHIP DMux {
    IN in, sel;
    OUT a, b;

    PARTS:
    // Put your code here:
    Not(in=sel,out=not1);
    And(a=not1,b=in,out=a);
    And(a=sel,b=in,out=b);
}
```

## DMu4Way

```
sel   in   a  b  c  d
0 0    0   0  0  0  0
0 0    1   1  0  0  0
0 1    0   0  0  0  0
0 1    1   0  1  0  0
1 0    0   0  0  0  0
1 0    1   0  0  1  0
1 1    0   0  0  0  0
1 1    1   0  0  0  1

CHIP DMux4Way {
    IN in, sel[2];
    OUT a, b, c, d;

    PARTS:
    // Put your code here:
    Not(in=sel[0],out=not1);
    Not(in=sel[1],out=not2);
    And(a=not1,b=not2,out=and1);
    And(a=and1,b=in,out=a);

    And(a=sel[0],b=not2,out=and2);
    And(a=and2,b=in,out=b);

    And(a=sel[1],b=not1,out=and3);
    And(a=and3,b=in,out=c);

    And(a=sel[0],b=sel[1],out=and4);
    And(a=and4, b=in, out=d);
}
```

## DMu8Way
```
sel    in  a b c d e f g h
0 0 0   1  1
0 0 1   1  0 1
0 1 0   1  0 0 1
0 1 1   1  0 0 0 1
1 0 0   1  0 0 0 0 1
1 0 1   1  0 0 0 0 0 1
1 1 0   1  0 0 0 0 0 0 1
1 1 1   1  0 0 0 0 0 0 0 1

CHIP DMux8Way {
    IN in, sel[3];
    OUT a, b, c, d, e, f, g, h;

    PARTS:
    // Put your code here:
    
    Not(in=sel[0], out=not1);
    Not(in=sel[1], out=not2);
    Not(in=sel[2], out=not3);

    And(a=not2,b=not3,out=and1);
    And(a=and1,b=not1,out=and2);
    And(a=and2,b=in,out=a);

    And(a=and1,b=sel[0],out=and3);
    And(a=and3,b=in,out=b);
    
    And(a=not1,b=sel[1],out=and4);
    And(a=and4,b=not3,out=and5);
    And(a=and5,b=in,out=c);

    And(a=sel[0],b=sel[1],out=and6);
    And(a=not3,b=and6,out=and7);
    And(a=and7,b=in,out=d);

    And(a=not1,b=not2,out=and8);
    And(a=sel[2],b=and8,out=and9);
    And(a=and9,b=in,out=e);

    And(a=sel[0],b=not2,out=and10);
    And(a=and10,b=sel[2],out=and11);
    And(a=and11,b=in,out=f);

    And(a=sel[1],b=sel[2],out=and12);
    And(a=and12,b=not1,out=and13);
    And(a=and13,b=in,out=g);

    And(a=sel[0],b=and12,out=and14);
    And(a=and14,b=in,out=h);
}
```



# 高电平

![](https://cdn.jsdelivr.net/gh/later-3/blog-img/20230505081744.png)
高电平要求不管输入是1还是0，输出都要是1，将输入取反和不取反的结果送入到or中，输出都为1.

