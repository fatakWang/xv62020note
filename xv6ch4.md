# 第四章：陷阱与系统调用

有三种事件会导致控制权转移到处理该事件的特殊代码上。

一种是ecall，也就是系统调用

一种是异常（一些非法的事情）

一种是设备中断

用陷阱（trap）作为这些情况的通用术语，内核处理这些陷阱，分为四个步骤：

RISC-V CPU采取的硬件操作、

为内核C代码执行而准备的汇编程序即“向量”、

决定如何处理陷阱的C陷阱处理程序以及系统调用或设备驱动程序服务例程。

## 4.1RISC-V陷入机制

`stvec`：陷阱处理程序的地址，发生陷阱后，pc会被这个值覆盖

`sepc`：发生陷阱时，RISC-V会在这里保存pc，`sret`（从陷阱返回）指令会将`sepc`复制到pc

`scause`： RISC-V在这里放置一个描述陷阱原因的数字。

`sscratch`：一个有用的值

`sstatus`：**SIE**位控制设备中断是否启用，**SPP**位指示陷阱是来自用户模式还是管理模式，并控制`sret`返回的模式。

机器模式下也有一组等效的寄存器，计时器中断时用的到

当要执行陷阱时，RISC-V硬件（内核也参与不了，我们以ecall为例）执行以下操作

1. 如果陷阱是设备中断，并且状态**SIE**位被清空，则不执行以下任何操作。
2. 清除**SIE**以禁用中断。
3. 将`pc`复制到`sepc`。
4. 将当前模式（用户或管理）保存在状态的**SPP**位中。
5. 设置`scause`以反映产生陷阱的原因。
6. 将模式设置为管理模式。
7. 将`stvec`复制到`pc`。（在哪里这个寄存器被替换成了陷阱处理程序？）这是一个保证了隔离的有效措施
8. 在新的`pc`上开始执行。

注意，CPU不会切换到内核页表，不会切换到内核栈，也不会保存除`pc`之外的任何寄存器。所以剩下的事情都是由内核软件来做的。



## 从用户空间陷落

来自用户空间的陷阱路径是uservec-》usertrap-》usertrapret-》userret

当执行用户代码时，`stvec`始终设置为`uservec`，且用户页表必须包括`uservec`，且内核页表与用户页表的`uservec`虚拟地址必须相同（这也是在页表都映射trampoline 的原因）。

###uservec

uservec主要是保护用户程序的现场，主要就是32个寄存器。而有点尴尬这32个寄存器每个都要保存，可是在uservec又需要新的寄存器来做一些事情，所以可以用sscratch来提供帮助，让sscratch来暂时保存。

`sscratch`中原始的值是指向进程的陷阱帧TRAPFRAME（保存所有用户寄存器的空间）。当使用内核页表，没有映射TRAPFRAME时，用进程的`p->trapframe`

uservec首先交换a0`和`sscratch。`a0`此时持有指向当前进程陷阱帧的指针，然后保存所有寄存器，陷阱帧额外恢复指向当前进程内核栈的指针kernel_sp、当前CPU的`kernel_hartid`、`usertrap`的地址和内核页表的地址kernel_satp。

将这些值取出更新后恢复完完全全的内核状态时，装载内核页表到satp，更新tlb，然后眺转到usertrap。

注意此时还有sepc没有操作过。

### usertrap

usertrap确定陷阱的原因，处理并返回

首先确保是从user mode来的，然后更新stvec为kenellvec（内核下的traphaddle），再保存sepc到进程的陷阱帧，之所以切换是因为，后面尽管处理尚未结束但是要开中断。

然后根据scause的值判断是系统调用ecall还是设备中断（**devintr**来处理），否则就是一个异常，p->killed置为1来结束掉这个进程。另外如果which_dev是2的话，则yield来处理计时器中断。

如果是系统调用会在保存的sepc下加4指向ecall的下一条，然后开中断，来进行syscall。

最后的最后进入usertrapret

### usertrapret（返回用户空间）

首先关中断，`stvec`改为指向`uservec`（不是早就设置好了吗），设置uservec需要的陷阱帧的字段，将`sepc`设置为之前保存的用户程序计数器。设置sstatus寄存器，最后调用userret以TRAPFRAME为a0，用户页表为a1.

### userret

将satp更新为a1，将之前保存的a0放在sscratch中，然后把之前保存的寄存器全部恢复，最后再跟a0交换一下sscratch，这样所有寄存器都是陷阱之前的了（pc除外），最后执行sret（设置 *pc* 为sepc，权限模式为用户态）

返回用户态。

## 系统调用

系统调用的实现都是

```assembly
fork:
 li a7, SYS_fork
 ecall
 ret
```

同时通过形式参数将a0，a1设置好，系统SYS_fork在kernel/syscall.h预定义好了，所以a7作为调用号随后通过ecall进入陷阱，由硬件设置后执行`uservec`、`usertrap`和`syscall`。syscall根据p->trapframe->a7的值来索引具体的函数，并将返回值保存到p->trapframe->a0，经过usertrapret，userret来变成之前调用系统调用的样子。

###系统调用参数

系统调用函数的参数通过陷阱帧保存的寄存器实现，

##从内核空间陷入

内核态下`stvec`指向`kernelvec`（一种是一开始启动机器时设置，一种是进入内核态后设置），由于内核态下已经使用内核栈，所以寄存器可以方便地使用栈来保存，

### kernelvec

保存寄存器，然后call kerneltrap ，call完后再恢复寄存器sret（将pc设置为sepc，根据sstatus寄存器的设置，切换到监管者模式，并重新开放中断、、存疑，），

### **kerneltrap**

处理设备中断和异常，如果是计时器中断就响应（我猜之后会把中断开了），因为要响应中断可能导致sepc、sstatus改变，因此前面先保存，之后再恢复。然后返回kernelvec

##页面错误异常

Risc-v有三种不同的页面错误:

1. 加载页面错误 (当加载指令无法转换其虚拟地址时)
2. 存储页面错误 (当存储指令无法转换其虚拟地址时)
3. 指令页面错误 (当指令的地址无法转换时)

`scause`寄存器中的值指示页面错误的类型，`stval`寄存器包含无法翻译的地址。