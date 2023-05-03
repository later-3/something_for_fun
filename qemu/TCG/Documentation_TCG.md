#TCG #QEMU

# qemu的官方文档
## 主要内容
- 极小的代码生成器
- 源码树文档
- wiki上的其他资料
- presentations和其他外部资源

### 极小的代码生成器(TCG)
TCG的存在是为了将target的指令转换到host上的指令，首先是通过TCG的前端转换到TCG ops，然后通过TCG的后端转到host insns.
如果有人想要移植qemu去支持一个新的处理器，是需要关心后端的。还有TCI (TCG解释器)，它为TCGops提供了一个后端无关的解释器。同样也需要关心前端。
### 源码树文档
大量在源码树中的文档应该能帮助理解TCG的工作原理：
- Details about [docs/devel/tcg.rst](https://gitlab.com/qemu-project/qemu/-/blob/master/docs/devel/tcg.rst) [[Translator Interals]]
- How we approach [multithreaded TCG](https://gitlab.com/qemu-project/qemu/-/blob/master/docs/devel/multi-thread-tcg.txt)

### 其他wiki上的资料
-   [Backend Ops](https://wiki.qemu.org/Documentation/TCG/backend-ops "Documentation/TCG/backend-ops")
-   [Frontend Ops](https://wiki.qemu.org/Documentation/TCG/frontend-ops "Documentation/TCG/frontend-ops")

### presentations和其他外部资源
-   [Slides](https://dl.dropboxusercontent.com/u/8976842/TCG.pdf) and [recording](http://chemnitzer.linux-tage.de/2012/vortraege/1062) of a talk on TCG mechanics
-   [StackOverflow answer](http://stackoverflow.com/questions/20675226/qemu-code-flow-instruction-cache-and-tcg) showing code flow of a TCG translation

https://people.linaro.org/~alex.bennee/org/presentations/vectoring-in-on-tcg.html