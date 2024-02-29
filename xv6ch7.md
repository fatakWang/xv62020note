#第七章 调度

xv6通过cpu的多路复用实现对进程占有虚拟cpu的抽象

### 7.1多路复用

Xv6通过在两种情况下将每个CPU从一个进程切换到另一个进程来实现多路复用.

第一：当进程等待设备或管道I/O完成，或等待子进程退出，或在`sleep`系统调用中等待时，xv6使用睡眠（sleep）和唤醒（wakeup）机制切换。（主动放弃，或中断的放弃）

第二：xv6周期性地强制切换以处理长时间计算而不睡眠的进程。（时间片轮转被动）

挑战：如何切换，透明，cpu的轮换的影响，退出时资源的释放，sleepwakeup唤醒丢失，每个核心知道自己在执行哪个进程

### 7.2上下文切换

![img](http://xv6.dgs.zone/tranlate_books/book-riscv-rev1/images/c7/p1.png)

从一个用户进程（旧进程）切换到另一个用户进程（新进程）所涉及的步骤：

一个到旧进程内核线程的用户-内核转换，一个到当前CPU调度程序线程的上下文切换，一个到新进程内核线程的上下文切换，以及一个返回到用户级进程的陷阱。

## 一个进程的一生

UNUSED, USED, SLEEPING, RUNNABLE, RUNNING, ZOMBIE，一个进程共有这六种状态

调度线程即为启动时的线程栈用的是stack0，保护寄存器用的是

使用TRAMPOLINE + (uservec - trampoline)，并且用户和内核都要映射的原因，是在调用userret、uservec 会切换页表

###第一个进程initproc

从main的userinit到scheduler到swtch到forkret到usertrapret到userret到init





在main的userinit()中调用，在userinit里，uvminit在init的虚拟地址为0的位置放置了一段代码，主要是进行了系统调用ecall来进行exec（“init”）。并且在陷阱帧的epc放了0来配合。完成userinit后掉入scheduler（后续创建的核心也要进入scheduler，）

