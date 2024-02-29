# 第二章 操作系统架构

xv6 code: [kernel/proc.h](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.h), [kernel/defs.h](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/defs.h), [kernel/entry.S](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/entry.S), [kernel/main.c](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/main.c), [user/initcode.S](https://github.com/mit-pdos/xv6-riscv/blob/riscv/user/initcode.S), [user/init.c](https://github.com/mit-pdos/xv6-riscv/blob/riscv/user/init.c), and skim [kernel/proc.c](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c) and [kernel/exec.c](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/exec.c) kernel/start.c

## code note

#####kernel/entry.S

la rd rs #将rs的地址传给了rs

li  rd im #将im传给rd

CSR是Control and Status Register（控制和状态[寄存器](https://so.csdn.net/so/search?q=寄存器&spm=1001.2101.3001.7020)）的简写。CSR指令就是操作CSR寄存器的一组指令（读、改、写）。

CSRR rd, csr 把控制状态寄存器 *csr* 的值写入rd，mhartid 寄存器获取 hart 的 ID、防止走串、

设置好每个核的sp寄存器（指明每个初始的有4096字节）后，然后call start

kernel/start.c

\_\_attribute\_\_允许你在定义类型时指定特殊属性， \_\_attribute\_\_((aligned(n)))：此属性指定了指定类型的变量的最小对齐(以字节为单位)。

#####start.c()围绕着mret在做事

1. 通过设置mstatus寄存器切换为监管模式
2. 将mepc设置为main函数入口
3. 通过satp寄存器来禁用分页
4. 所有中断和异常都将交由在监管者模式下处理
5. 让监管者模式可以直接访问物理内存
6. 开启时钟中断（就是可以让多进程共享cpu的东西）
7. hartid->tp(thread pointer)
8. 最后调用mret转到kernel/main.c

#####kernel/main.c

cpuid()将tp中的核id取出来

procinit()初始化进程表

userinit()创建init进程（即为第一个进程）

#####kernel/proc.h

有**用户栈**和**内核栈**，用于切换保存寄存器的上下文、还有用于系统调用陷入的trapframe

#####kernel/proc.c

###### procinit()

###### allocproc()

在进程表中找到一个UNUSED的进程，若无则返回

然后调用allocpid()分配一个独一无二的pid，更改state，分配trapframe、用户栈、上下文


######userinit()

调用allocproc()分配一个进程

为进程初始化页表

然后进入initcode，再来系统调用exec到init里面去

##### user/init.c

打开console，复制0到1与2

再exec到shell

由此便是一个完整的启动过程

### 系统调用过程

将系统调用需要的参数放在寄存器`a0`和`a1`中，将系统调用号放在`a7`中。系统调用号与`syscalls`数组中的条目相匹配，然后再ecall陷入内核、内核再来uservec`、`usertrap`和`syscall，syscall中从陷阱帧（trapframe）中保存的`a7`中检索系统调用号，并调用对应的系统调用函数。







操作系统对于多进程必须满足三个要求：多路复用、隔离和交互。

xv6是用基于“LP64”的C语言编写的，这意味着C语言中的`long`（L）和指针（P）变量都是64位的，但`int`是32位的。

## 2.1 抽象系统资源

有缺点的方式：系统调用实现为一个库、所有用户都通过这个系统调用来访问硬件资源

虽然性能好，但是隔离、复用没做好

应该禁止用户直接访问底层硬件资源。我们设计了操作系统，现在有一层内核来为我们提供服务，实现了用户进程与硬件资源间的隔离。

还将硬件资源从设备中**抽象**出来、抛开硬盘资源而代之以文件系统的概念】同时这种抽象又便于交互

## 2.2 用户态，核心态，以及系统调用

要实现强隔离，操作系统必须保证应用程序不能修改（甚至读取）操作系统的数据结构和指令，以及应用程序不能访问其他进程的内存。于是提出了三种riscv的模式：机器模式(Machine Mode)、用户模式(User Mode)和管理模式(Supervisor Mode)。

在机器模式下执行的指令具有完全特权；CPU在机器模式下启动。机器模式主要用于配置计算机。Xv6在机器模式下执行很少的几行代码，然后更改为管理模式。

在管理模式下，CPU被允许执行特权指令，应用程序只能执行用户模式的指令（例如，数字相加等），并被称为在**用户空间**中运行，而此时处于管理模式下的软件可以执行特权指令，并被称为在**内核空间**中运行。在内核空间（或管理模式）中运行的软件被称为内核。

因此要想调用系统调用，必须有一个过渡到内核的过程：

CPU提供一个特殊的指令，将CPU从用户模式切换到管理模式，并在内核指定的入口点进入内核（RISC-V为此提供`ecall`指令）。

系统调用过程、填充a0、a1、a7、调用ecall、转为

## 2.3 内核组织

作系统的哪些部分应该以管理模式运行？

宏内核与微内核

整个操作系统都驻留在内核中，这样所有系统调用的实现都以管理模式运行。这种组织被称为**宏内核**，xv6就属于宏内核

为了降低内核出错的风险，操作系统设计者可以最大限度地减少在管理模式下运行的操作系统代码量，并在用户模式下执行大部分操作系统。这种内核组织被称为**微内核**

##2.4 代码（XV6架构篇）

XV6的源代码位于***kernel/\***子目录中，源代码按照模块化的概念划分为多个文件，模块间的接口都被定义在了**def.h（kernel/defs.h）。**

| **文件**             | **描述**                                    |
| -------------------- | ------------------------------------------- |
| **bio.c**            | 文件系统的磁盘块缓存                        |
| **console.c**        | 连接到用户的键盘和屏幕                      |
| **entry.S**       | 首次启动指令                                |
| ***exec.c***        | `exec()`系统调用                            |
| ***file.c***        | 文件描述符支持                              |
| ***fs.c***          | 文件系统                                    |
| ***kalloc.c***      | 物理页面分配器                              |
| ***kernelvec.S***   | 处理来自内核的陷入指令以及计时器中断        |
| ***log.c***         | 文件系统日志记录以及崩溃修复                |
| ***main.c***        | 在启动过程中控制其他模块初始化              |
| ***pipe.c***        | 管道                                        |
| ***plic.c***        | RISC-V中断控制器                            |
| ***printf.c***      | 格式化输出到控制台                          |
| ***proc.c***        | 进程和调度                                  |
| **sleeplock.c**   | Locks that yield the CPU                    |
| ***spinlock.c***    | Locks that don’t yield the CPU.             |
| ***start.c***       | 早期机器模式启动代码                        |
| ***string.c***      | 字符串和字节数组库                          |
| ***swtch.c***       | 线程切换                                    |
| ***syscall.c***     | Dispatch system calls to handling function. |
| ***sysfile.c***     | 文件相关的系统调用                          |
| ***sysproc.c***     | 进程相关的系统调用                          |
| ***trampoline.S***  | 用于在用户和内核之间切换的汇编代码          |
| ***trap.c***        | 对陷入指令和中断进行处理并返回的C代码       |
| ***uart.c***        | 串口控制台设备驱动程序                      |
| ***virtio_disk.c*** | 磁盘设备驱动程序                            |
| ***vm.c***          | 管理页表和地址空间                          |

## 2.5 进程概述

Xv6（和其他Unix操作系统一样）中的隔离单位是一个进程

进程为程序提供了一个看起来像是私有内存系统、也好像在时间上独占机器

Xv6使用页表（在内存中）为每个进程提供自己的地址空间，用户使用虚拟地址，实际cpu使用物理地址

Xv6为每个进程维护一个单独的页表

![img](http://xv6.dgs.zone/tranlate_books/book-riscv-rev1/images/c2/p2.png)

最大地址是2^38-1=0x3fffffffff，即`MAXVA`（定义在***kernel/riscv.h\***:348）

在地址空间的顶部，xv6为`trampoline`（用于在用户和内核之间切换）和映射进程切换到内核的`trapframe`分别保留了一个页面、、存疑第四章讲解



xv6内核为每个进程维护许多状态片段，并将它们聚集到一个`proc`(***kernel/proc.h\***:86)结构体中。一个进程最重要的内核状态片段是它的页表、内核栈区和运行状态。

每个进程都有一个执行线程（或简称线程）来执行进程的指令。

线程的大部分状态（本地变量、函数调用返回地址）存储在线程的栈区上。

每个进程有两个栈区：一个用户栈区和一个内核栈区（`p->kstack`），当进程执行用户指令时，只有它的用户栈在使用，它的内核栈是空的。当进程进入内核（由于系统调用或中断）时，内核代码在进程的内核堆栈上执行；

一个进程可以通过执行RISC-V的`ecall`指令进行系统调用，调用`sret`指令返回用户空间

`p->state`表明进程是已分配、就绪态、运行态、等待I/O中（阻塞态）还是退出。

带下划线的才会加载到操作系统

