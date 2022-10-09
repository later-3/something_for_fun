# 前言
参考：playing with ptrace
- https://www.linuxjournal.com/article/6100
- https://www.linuxjournal.com/article/6210

你有想过系统调用是如何被截获的吗？你有试过改变系统调用的参数去戏耍内核吗？你有想过调试器是如何暂定一个正在运行的进程并且让你控制进程的吗？

如果你思考用负责的内核编程去完成任务，再想想吧。Linux提供了一个优雅的机制去完成所有的事情：<mark style="background: #BBFABBA6;">ptrace(process trace)</mark> 系统调用。**ptrace** 提供了一个机制：允许父进程观察和控制另一个进程的执行。它可以执行和改变 core image 和 寄存器，常用于断点调试和系统调用跟踪。

在这篇文章里，我们学习如何去截获一个系统调用和改变它的参数。在第二部分中我们会学习进阶的技术：设置断点和注入代码到正在运行的程序中。我们会窥探子进程的寄存器和数据段，并且修改里面的内容。我们也会谈到注入代码的方法，可以使进程暂停和执行任意指令。

> 基础

操作系统通过叫做系统调用的标准机制 提供了服务。它们为访问硬件和底层服务提供了标准的API，比如文件系统。当一个进程想要调用系统调用时，它将参数放到寄存器中，然后调用软中断 0x80. 这个软中断就像是通往内核模式的门，内核将会在检查完参数后执行系统调用。

在i386的体系结构中，系统调用中断号是放到寄存器 %eax中的。系统调用的参数按顺序放到 %ebx，%ecx，%edx，%esi和%edi中。举个例子：
```c
write(2, "hello", 5)
```
通常被翻译成:
```asm
movl $4, %eax
movl $2, %ebx
movl $hello, %ecx
movl $5, %edx
int $0x80
```

注： `movl $hello` 这里的hello字符串应该是被放到内存中的，$hello 理解为首地址或者其地址的标号

那ptrace到底是如何发挥作用的呢？在执行系统调用前，内核会检查进程是否正在被跟踪。如果是，内核暂停进程，然后把控制权交给跟踪进程，这样就可以执行和修改被跟踪进程的寄存器了。

让我们用一个例子来更清楚的看看进程是如何工作的：
```c
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <linux/user.h>   /* For constants
                                   ORIG_EAX etc */
int main()
{   pid_t child;
    long orig_eax;
    child = fork();
    if(child == 0) {
        ptrace(PTRACE_TRACEME, 0, NULL, NULL);
        execl("/bin/ls", "ls", NULL);
    }
    else {
        wait(NULL);
        orig_eax = ptrace(PTRACE_PEEKUSER,
                          child, 4 * ORIG_EAX,
                          NULL);
        printf("The child made a "
               "system call %ld\n", orig_eax);
        ptrace(PTRACE_CONT, child, NULL, NULL);
    }
    return 0;
}
```

gcc编译后运行：
```shell
❯ ./a.out                   
The child made a system call 59
100ask               initrd64.ext4      qemu-net.txt                        
100ask_imx6ull-qemu  linux              rootfs.img
a.out                linux-5.18.10      something_for_fun
busybox-1.35.0       ptrace.c           start.sh
busybox-arm          qemu               ubuntu-18.04_imx6ul_qemu_system
initrd               qemu-7.1.0         up.sh
initrd128.ext4       qemu-7.1.0.tar.xz  versatile.sh
```



