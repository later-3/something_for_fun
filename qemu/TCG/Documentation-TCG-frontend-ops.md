# 约定
- 当术语寄存器使用没有指定时，它很可能指的是 TCG 寄存器。
- 生成 TCG 操作码的前端助手通常采用以下形式: tcg_gen_\<op>[i]___<reg_size>_. 
 - < op > 是将为其参数生成的 TCG 操作。
 - 后缀[i]用于表示 TCG 操作采用立即寄存器而不是普通寄存器。
 - <reg_size> 表示正在使用的 TCG 寄存器的大小。在绝大多数情况下，这将匹配模拟目标的本机大小，因此不会一直强制人们键入 i32或 i64，而是为所有助手提供简写 tl。例如，要对32位目标执行32位寄存器移动，只需使用 tcg_gen_mov_tl而不是 tcg_gen_mov_i32。
- 我们不会介绍函数的直接变体，因为一旦掌握了寄存器版本，它们的用法应该相当明显。
- 为了简单起见，我们将在这里介绍 tl 变量。如果您需要深入研究显式类型，那么无论如何您都应该知道自己在做什么，因此您已经超出了这个 wiki 页面的当前范围。
- 类似地，与其到处使用 TCGv _ i32和 TCGv _ i64，人们应该坚持使用 TCGv 和 TCGv _ ptr。
- Tcg_gen_xxx参数通常先放置返回值，然后放置操作数。因此，要添加两个寄存器(a = b + c) ，代码应该是 tcg_gen_xxx(a，b，c) ; 。

### Registers
仿真处理器的大多数核心寄存器应该具有等效的 TCG 寄存器。
```c
Declare a named TCG register：
TCGv reg = tcg_global_mem_new(TCG_AREG0, offsetof(CPUState, reg), "reg");
```

### Temporaries
通常，目标 insns 不能分解成一个或两个简单的 RISC insns，这意味着可能需要临时寄存器来存储中间结果。每个前端不需要维护自己的静态临时抓取寄存器集，而是提供帮助器来动态管理这些寄存器。
```c
Create a new temporary register
TCGv tmp = tcg_temp_new();

Create a local temporary register. Simple temporary register cannot carry its value across jump/brcond, only local temporary can.
TCGv tmpl = tcg_temp_local_new();

Free a temporary register
tcg_temp_free(tmp);
```

您不应该尝试“保留”当前生成的目标 inn 之外的临时寄存器。如果您需要长期存活的寄存器，请考虑分配一个合适的寄存器。

### Labels
标签用于生成条件代码。在这里，我们只是介绍它们的管理; 稍后，我们将介绍如何将它们与操作码结合使用。

## Ops
这些前端ops扩展到后端ops。查阅该页面可能有助于更深入地了解前端/后端关系。

### Math
单寄存器上的数学运算:
```c
tcg_gen_mov_tl(ret, arg1); // Assign one register to another, ret = arg1
tcg_gen_neg_tl(ret, arg1); // Negate the sign of a register, ret = - arg1

```
两个寄存器上的数学运算:
```c
tcg_gen_add_tl(ret, arg1, arg2);// Add two registers, ret = arg1 + arg2
tcg_gen_sub_tl(ret, arg1, arg2);//Subtract two registers,ret = arg1 - arg2
tcg_gen_mul_tl(ret, arg1, arg2);//Multiply two signed registers and return the result, ret = arg1 * arg2
...
```





