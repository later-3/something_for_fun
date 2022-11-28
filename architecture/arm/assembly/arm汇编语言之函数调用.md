# 前言
在之前看书的时候就已经接触过这方面的内容，所以大体上知道一些，但之前接触的时候只是被动接收，且没有动手实际操作过，只觉得好像还挺复杂的。这次因为一些原因，我必须得好好解析这部分了😅.

```ad-tip
c语言的函数被翻译成汇编是什么样的 ==> The ABI for ARM 64-bit Architecture
```

# 目标
首先我先定义一个本文需要去解决的问题或者要达到的目标:
> 能够解释 "c代码编译后的汇编(主要是函数之间的调用)" 

# 方法
仍然采取我最喜欢的方法 --> <mark style="background: #BBFABBA6;">自底向上</mark>
首先是对一篇材料的翻译，然后我会时不时的跳出来，去思考这到底是怎么回事，最后总结。

当不清楚一些背景的时候，会觉得很难，这时候就越要去思考为什么会这样，想通了之后就简单了。

当程序被编译成二进制的时候，放到内存中，然后开始从程序的入口出开始一条一条的执行，这就是当前脑海中的一个初始状态，接下来就开始解析 函数对应的指令是怎么回事。

# arm汇编之函数调用
原文: https://www.cs.princeton.edu/courses/archive/spr11/cos217/lectures/15AssemblyFunctions.pdf

## 课程目标
- 函数调用问题
	- 调用和返回
	- 参数传递
	- 保存局部变量
	- 没有影响的使用寄存器
	- 返回值
- IA-32的解决办法
	- 想关的指令和约定

```ad-tip
没错，这个是一个课件。这符合自下而上的思想，看看制作课件的老师是想要如何一步一步的介绍这些内容的
```

<mark style="background: #FFF3A3A6;">注：这篇内容是需要有一点点汇编基础的，就是大概知道mov和add是干啥的就好了，其余的不清楚的遇到再去了解就好。</mark>

### 阶段思考

> 这里需要总结一下，我没有想到上面课程目标中的 `函数调用问题` ，只是我的惯性了，以此告诫自己，这就是一个再去了解新东西之前应该思考的内容，举一反三，以后应该先做这些思考然后再去了解。

## 问题一：调用和返回
1. 调用函数（caller funciotn）是如何跳转到被调用函数（callee）的？
	亦即，跳转到被调用函数的第一条指令执行的地方。

2. 如何从被调用函数返回到调用函数的正确的地址上？
	亦即，跳转到调用函数 <mark style="background: #FFB86CA6;">最近刚执行的指令</mark> 下面 <mark style="background: #FFB86CA6;">立即要执行</mark> 的 指令。

```ad-note
1. 注意前面提到的初始状态，可执行程序在内存中就是一条一条指令，当然这里说的是代码段区域。
2. 像操作系统最开始的代码就是用汇编语句写的，之前在 李忠先生 的汇编语言书籍中看到过一些用汇编语言去实现一些功能。
3. 实际一个函数就是 一坨连续的指令，从一坨指令跳转到另一坨指令，然后再跳转回来，这是这个问题想要去讲清楚的地方。
4. 一坨指令可以使用 CPU 的寄存器资源，这里面需要特别注意的是 sp， 也就是栈，函数是有栈的。CPU 的这些寄存器资源只有一套，在使用的时候要考虑是否要保存当前的值，考虑当前的值是否会被其他地方用到。
```

### 调用和返回的尝试解决方案
- 方案一：使用跳转指令

![](https://raw.githubusercontent.com/later-3/img_picgo/main/img/20221127145032.png)
使用jmp指令跳转到另一个函数去。但是这样会有一个问题，因为一个函数可能被多个函数调用，所以return的时候就出问题了，不知道要返回到哪个地址：
![](https://raw.githubusercontent.com/later-3/img_picgo/main/img/20221127150028.png)

- 方案二：使用寄存器保存返回地址
![](https://raw.githubusercontent.com/later-3/img_picgo/main/img/20221127151207.png)
先将下一条指令的地址放到寄存器中，然后在被调用的函数中使用jmp返回调用函数中。
但这样会有一个问题是：不能处理嵌套调用的场景，也就是A调用B，然后B再调用C。
![](https://raw.githubusercontent.com/later-3/img_picgo/main/img/20221127151933.png)

### IA-32解决方案：使用栈
> 可能需要保存很多返回地址
- 嵌套调用的深度是不能提前确定的
- 只要函数调用的关系还在，返回地址就应该被保存下来，调用关系结束后再丢弃

> 保存地址的顺序应和调用时的顺序相反
- 例如：P调用Q，Q再调用R，应该从R返回到Q，最后从Q返回P
![](https://raw.githubusercontent.com/later-3/img_picgo/main/img/20221127153445.png)

> 后进先出的数据结构: 栈
- 调用者将返回地址放到栈中
- 被调用者将返回地址从栈中弹出

> IA 32解决方案：在调用和返回时使用栈

### 调用的实现
- ESP：栈指针寄存器，指向栈的顶部
- 栈指针的操作指令：push，pop，但也可以用sub、add指令操作
![|400](https://raw.githubusercontent.com/later-3/img_picgo/main/img/20221127155956.png)
- EIP：类似pc，指向下一条要执行的指令的地址
![|500](https://raw.githubusercontent.com/later-3/img_picgo/main/img/20221128214217.png)

注：并不能直接访问ip，这里只是举例

call指令调用前后，栈内容对比：
![|400](https://raw.githubusercontent.com/later-3/img_picgo/main/img/20221128214620.png)
- ret指令，会将old eip弹出栈，并赋值给eip，使eip指向刚刚调用的函数的下一条指令


## 问题二：参数传递
> 调用者如何传递参数给被调用函数？

![|400](https://raw.githubusercontent.com/later-3/img_picgo/main/img/20221128220712.png)

f函数调用add3，三个参数是如何传递过去的

>  尝试解决的方案：使用寄存器

- 问题：不能处理嵌套调用，函数再调用函数时，寄存器里面的值会被覆盖
- 还有如果传递超过4个字节的参数怎么办，也就是超过寄存器的宽度

### IA-32的解决方案：使用栈

- 调用者在执行call指令之前，将参数压入栈中
- 注意是反序压入，就是先压入最后一个参数，然后是倒数第二个.....
![|300](https://raw.githubusercontent.com/later-3/img_picgo/main/img/20221128223343.png)
- 被调用者使用使用esp去取参数

![|300](https://raw.githubusercontent.com/later-3/img_picgo/main/img/20221128223856.png)

### IA-32 参数传递的例子
![](https://raw.githubusercontent.com/later-3/img_picgo/main/img/20221128224055.png)


### EBP: 栈基址寄存器




