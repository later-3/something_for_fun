# cpu.h
定义自己的`CPU6502State`


## translate.c
### 定义一个 `DisasContext`
- 第一个字段是固定的 `DisasContextBase base;`
- 第二个字段是自己的`CPU6AVRState`
- 定义pc
- 实现 `TranslatorOps` 结构体
- 实现 `gen_intermediate_code` 函数

