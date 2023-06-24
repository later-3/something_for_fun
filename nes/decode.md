# insn decode
> SEI

Set Interrupt Disable Status 1 -> I
```c
tcg_gen_movi_tl(cpu_interrupt_flag, 1);
```


> CLD

Clear Decimal Mode  0 -> D
```c
tcg_gen_movi_tl(cpu_decimal_flag, 0);
```

>LDA

Load Accumulator with Memory M -> A

|N|Z|C|I|D|V|
|---|---|---|---|---|---|
|+|+|-|-|-|-|

|addressing|assembler|opc|bytes|cycles|
|---|---|---|---|---|
|immediate|LDA #oper|A9|2|2|
|zeropage|LDA oper|A5|2|3|
|zeropage,X|LDA oper,X|B5|2|4|
|absolute|LDA oper|AD|3|4|
|absolute,X|LDA oper,X|BD|3|4|
| absolute,Y | LDA oper,Y | B9 | 3 | 4 |
|(indirect,X)|LDA (oper,X)|A1|2|6|
|(indirect),Y|LDA (oper),Y|B1|2|5|

> STA

store accumulator
A -> M

|N|Z|C|I|D|V|
|---|---|---|---|---|---|
|-|-|-|-|-|-|

| addressing   | assembler    | opc | bytes | cycles |
| ------------ | ------------ | --- | ----- | ------ |
| zeropage     | STA oper     | 85  | 2     | 3      |
| zeropage,X   | STA oper,X   | 95  | 2     | 4      |
| absolute     | STA oper     | 8D  | 3     | 4      |
| absolute,X   | STA oper,X   | 9D  | 3     | 5      |
| absolute,Y   | STA oper,Y   | 99  | 3     | 5      |
| (indirect,X) | STA (oper,X) | 81  | 2     | 6      |
| (indirect),Y | STA (oper),Y | 91  | 2     | 6      |

> LDX

Load Index X with Memory
M -> X

|N|Z|C|I|D|V|
|---|---|---|---|---|---|
|+|+|-|-|-|-|

|addressing|assembler|opc|bytes|cycles|
|---|---|---|---|---|
|immediate|LDX #oper|A2|2|2|
|zeropage|LDX oper|A6|2|3|
|zeropage,Y|LDX oper,Y|B6|2|4|
|absolute|LDX oper|AE|3|4|
|absolute,Y|LDX oper,Y|BE|3|4|


> TXS

Transfer Index X to Stack Register

X -> SP

> BPL

Branch on Result Plus
branch on N = 0

> LDY


Load Index Y with Memory

M -> Y

|N|Z|C|I|D|V|
|---|---|---|---|---|---|
|+|+|-|-|-|-|

| addressing | assembler  | opc | bytes | cycles |
| ---------- | ---------- | --- | ----- | ------ |
| immediate  | LDY #oper  | A0  | 2     | 2      |
| zeropage   | LDY oper   | A4  | 2     | 3      |
| zeropage,X | LDY oper,X | B4  | 2     | 4      |
| absolute   | LDY oper   | AC  | 3     | 4      |
| absolute,X | LDY oper,X | BC  | 3     | 4      |

> CMP

Compare Memory with Accumulator

A - M

|N|Z|C|I|D|V|
|---|---|---|---|---|---|
|+|+|+|-|-|-|

|addressing|assembler|opc|bytes|cycles|
|---|---|---|---|---|
|immediate|CMP #oper|C9|2|2|
|zeropage|CMP oper|C5|2|3|
|zeropage,X|CMP oper,X|D5|2|4|
|absolute|CMP oper|CD|3|4|
|absolute,X|CMP oper,X|DD|3|4|
|absolute,Y|CMP oper,Y|D9|3|4|
|(indirect,X)|CMP (oper,X)|C1|2|6|
|(indirect),Y|CMP (oper),Y|D1|2|5[*](https://www.masswerk.at/6502/6502_instruction_set.html#opcodes-footnote1 "add 1 to cycles if page boundary is crossed")|

> BCS

Branch on Carry Set

branch on C = 1

|N|Z|C|I|D|V|
|---|---|---|---|---|---|
|-|-|-|-|-|-|

|addressing|assembler|opc|bytes|cycles|
|---|---|---|---|---|
|relative|BCS oper|B0|2|2|

> DEX

Decrement Index X by One

X - 1 -> X

|N|Z|C|I|D|V|
|---|---|---|---|---|---|
|+|+|-|-|-|-|

|addressing|assembler|opc|bytes|cycles|
|---|---|---|---|---|
|implied|DEX|CA|1|2|

> BNE

Branch on Result not Zero

branch on Z = 0

|N|Z|C|I|D|V|
|---|---|---|---|---|---|
|-|-|-|-|-|-|

| addressing | assembler | opc | bytes | cycles |
| ---------- | --------- | --- | ----- | ------ |
| relative   | BNE oper  | D0  | 2     | 2      |

> JSR

Jump to New Location Saving Return Address

push (PC+2),  
(PC+1) -> PCL  
(PC+2) -> PCH

|N|Z|C|I|D|V|
|---|---|---|---|---|---|
|-|-|-|-|-|-|

| addressing | assembler | opc | bytes | cycles |
| ---------- | --------- | --- | ----- | ------ |
| absolute   | JSR oper  | 20  | 3     | 6      |




