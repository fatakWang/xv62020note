#第五章 中断和设备驱动

驱动程序是连接交互IO设备与内核的中转，告诉IO怎么做，也向内核提供统一的接口，驱动程序的代码有两套环境，上半部分是内核运行，当系统调用发生时被调用，下半部分是中断处理程序

内核负责识别何种设备产生中断，并调用驱动程序处理中断，具体是在devintr实现

## 经典的驱动程序，控制台输入/出console.c

当你敲下键盘，read(0)发生了什么？

### 前置知识

1. 驱动程序分为两部分，顶部**top half**（运行在内核中，提供相关的系统调用入口），底部**bottom half**（中断时运行）
2. 键盘通过UART的控制寄存器与内核通信，内核通过内存映射的方式读uart的控制寄存器

###初始化

首先kernel还在机器模式下就在main中调用了**consoleinit**，初始化控制台。

**consoleinit**主要调用了**uartinit**，**uartinit**主要初始化了uart的几个寄存器，相关的寄存器有：

1. RHR，输入内核的寄存器，键盘上面敲下去就到RHR存着了
2. THR,输入控制台（这里就是屏幕）
3. IER,指示发送或读入信息是否产生中断
4. LsR，指示输入字符是否正在等待软件读取的位。thr是否可以输出新的位

**uartinit**配置好传输的波特率，重置FIFO缓冲区，最后开启IER的**接收中断**和**发送完成中断**，之后每接收一个字符或者每发送一个，都会产生中断。

从**uartinit**返回后，**consoleinit**还把0，1，2的read，write导向了**consoleread**和**consolewrite**，这之后只要是printf都会调用到**consolewrite**，当然这里两个函数对应着驱动程序的顶部top half，**consoleintr**对应着底部bottom half。

当kernel创建第一个进程init的时候，会创建并打开名叫console的设备，并复制0的文件描述符到1、2此时才算真正全部设置完了

###键盘上的字母怎么到了内核里

要分成两部分来讨论驱动程序的顶部和底部

#### 底部（中断处理）

当敲下键盘会触发设备中断，最终到trap.c中的**usertrap**或者**kerneltrap**

但是最终都会调用**devintr**来区分是什么设备引起的中断，可以从scause、plic寄存器中获知。如果是uart发出的，就调用**uartintr**来处理，**uartintr**会循环的从RHR中读输入的字符，在调用**consoleintr**处理这个字符，里面会根据这个字符（如退格）进行相应的处理。普通的字符会放入缓冲区cons中，并调用**consputc**回显。

**cons**除了有一个数组，还要处理程序已经读了的字符位置，什么时候把数据送上去（用户敲回车时），还要新到的字符位置，用三个指针来处理，r、w、e，e就是新到的字符位置，r是已经读了的字符位置，w是把把与r对应的，要送给内核的数据右区间。

当human敲下回车时，就应该唤醒等待数据的进程（调用顶部的那个进程）了。

### 顶部

当进程想要read（0，etc）时，就实际上就会调用到**consoleread**，首先检测缓冲区的r、w位。如果相等，至少说明还没有巧回车或者human还没有敲键盘，然后就会sleep，只有前面的底部唤醒时，才能从sleep，离开，然后从cons取出一个字符，再**either_copyout**从内核传回用户空间或者内核空间，以上步骤重复read希望读入的字节数，或者遇到回车键就提前终止，返回实际读入的字节数。

### write/printf怎么显示到了控制台

####顶部

当write时实际是调用了**consolewrite**，每次从目标数组（copyin一下）读出一个字节，重复希望输出的字节数才停止，这一个字节拿去**uartputc**，也有一个缓冲区uart_tx_buf，有r和w位表示已经输出和新到字符的位置。如果缓冲区满了，就sleep。如果还有空，就对w位加一，然后调用**uartstart**，再return。**uartstart**主要是判断当前缓冲区r、w位置是否相同，THR有空吗？，如果都没问题，然后从缓冲区中取一个字符，再放到THR中，然后唤醒可能因此睡眠的进程。

#### 底部

每当发送一个字符也会触发中断，然后最后跑到**uartintr**中，调用一个**uartstart**，直到把缓冲区清空。

注意uartputc是接口异步（非阻塞）的，也就是说输入完后马上去做些别的，除非缓冲区满，程序会很快的完成write。

所以xv6还提供了同步的，阻塞的**uartputc_sync**，满足马上响应的需求，只有打印完了才做别的事情，printf就是用的这个

## 并发问题

## 定时器中断

在start.c中，就在**timeinit**做了初始化，设置了mscratch，mscratch，保持了类似trapframe的东西，主要是设置了mtvec指向**timervec**，每当发送定时器中断时，就会切换到机器模式，保存寄存器a1、a2、a3。然后a1、a2、a3就可以随便用了，做了一些CLINT，interval相关的工作，通过设置sip寄存器，来发送一个软件中断，最后恢复寄存器，并且mret返回。返回后马上中断，进入traphandler，通过**yield**，来进行调度。