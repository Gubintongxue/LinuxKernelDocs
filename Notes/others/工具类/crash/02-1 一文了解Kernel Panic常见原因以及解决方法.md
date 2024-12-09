# 一文了解Kernel Panic常见原因以及解决方法

### **出现原因**

1\. Linux在中断处理程序中，它不处于任何一个进程上下文，如果使用可能睡眠的函数，则系统调度会被破坏，导致kernel panic。因此，在中断处理程序中，是不能使用有可能导致睡眠的函数(例如信号量等)。

在中断发起的软中断中，其上下文环境有可能是中断上下文,同理，也不能调用可能导致睡眠的函数。软中断执行时，全局中断是打开的，而中断程序执行时，全局中断是禁止的。

软中断除了系统调度进入点，当软中断数量频繁时，内核中有一个专门的软中断的后台程序daemon来处理其事务。

2\. 内核堆栈溢出，或者指针异常访问时，会出现kernel panic。

堆栈溢出：程序循环或者多层嵌套的深度过多时，可能会导致栈溢出。

3\. 除0异常、内存访问越界、缓冲区溢出等错误时，当这些事件发生在应用程序时，Linux内核的异常处理机制可以对这些由应用程序引起的情况予以处理。当应用程序出现不可恢复性错误时，Linux内核可以仅仅终止产生错误的应用程序，而不影响其他程序。

如果上述操作发生在内核空间，就会引起kernel panic。

4\. 内核陷入死锁状态，自旋锁有嵌套使用的情况。

5\. 在内核线程中，存在死循环的操作。

### **解决方法**

1\. 全部排查内核中可能造成睡眠的函数调用地方。如果是自己写的模块，则在调用睡眠函数之前打印出特征日志，以备查验。

在内核代码中的特定位置加入printk调试调用，直接把需要关心的信息打印到屏幕上，从而得知程序执行的路径。

2\. 在可疑的地方，调用dump\_stack()函数或者\_\_backtrace()，打印当前CPU的堆栈调用函数。

3\. 打开Linux内核的崩溃转储机制(kdump机制，生产vmcore文件)，当系统crash时，将内存内容保存到磁盘，或者通过网络发送到故障服务器，或者直接使用内核调试器。crash工具用于调试内核崩溃转储文件。

5\. 使用内核自带的 notify\_chain机制。Linux内核提供“通知链”功能，并预定义了一个内核崩溃通知链。当kernel panic时，异常处理程序会沿着预定义的通知链顺序调用注册到通知链中的通知函数。

6\. 在RedHat、StackOverflow、查找出现bug的历史解决方案，

7\. 调试方法，采用kprobe来调试内核。

8\. 对于一些未定义指令的错误，在出现的错误log中 ，Oops - undefined instruction: 0 \[#1\] PREEMPT SMP ARM，结合原始镜像的system.map文件，来定位。

9\. systemtap调试工具

10\. gcore工具

在学习Linux中，从《LInux内核设计与实现》里面，看到一本《Linux 内核精髓：精通Linux内核必会的75个绝技》，这本书是日本人高桥浩和写的，在书籍的合住作者，大岩尚宏，他编写了《Debug Hack》一书，这本是有关Linux内核调试的书籍，大喜。真是按图索骥，逐渐发现新的宝贝书籍。

内核调试工具介绍以及使用

Kdb

> kdb是Linux内核的补丁，提供了一种在系统运行时，对内核内存和数据结构进行检查的方法，不是源码级别的调试工具。kdb主要目标在于开发和诊断一些内核的问题。  
> 打开KALLSYMS：General setup-->Configure standard kernel features-->Load all symblos for debugging/ksymoops  
> 开启kdb服务  

Kprobe

kprobe（内核探测，kernel probe）是一个动态地收集调试和性能信息的工具，如：收集寄存器和全局数据结构等调试信息，无需对Linux内核频繁编译和启动。用户可以在任何内核代码地址进行陷阱，指定调试断点触发时的处理例程。工作机制是：用户指定一个探测点，并把用户定义的处理函数关联到该探测点，当内核执行到该探测点时，相应的关联函数被执行，然后继续执行正常的代码路径。

Kprobes 提供了一个强行进入任何内核例程并从中断处理器无干扰地收集信息的接口

Kprobes 向运行的内核中给定地址写入断点指令，插入一个探测器。执行被探测的指令会导致断点错误。Kprobes 钩住（hook in）断点处理器并收集调试信息。Kprobes 甚至可以单步执行被探测的指令。

内核探测分为kprobe, jprobe和kretprobe（也称return probe，返回探测）三种。

kprobe可插入内核中任何指令处；

jprobe插入内核函数入口，方便于访问函数的参数；

return probe用于探测指定函数的返回值。

**内核配置**

> CONFIG\_KPROBES General Setup--->Kprobe  
> CONFIG\_MODULES √  
> CONFIG\_MODULE\_UNLOAD √  
> CONFIG\_KALLSYMS\_ALL General Setup--->Configure standard kernel configuration-->Include all symbols in kallsyms  
> CONFIG\_KALLSYMS General Setup--->Configure standard kernel configuration-->Load all symbols for debugging/ksymoops  
> CONFIG\_KALLSYMS\_EXTRA\_PASS General setup-->Configure standard kernel features-->Load all symbols for debugging/ksymoops  
> CONFIG\_DEBUG\_INFO Kernel hacking-->Kernel debugging-->Compile the kernel with debug info  
> CONFIG\_DEBUG\_FS Kernel hacking-->Debug Filesystem  
> 让内核支持DEBUGFS,使能宏CONFIG\_DEBUG\_FS  
> `CONFIG_RELAY: General Setup -> user spacerelay support`

编译通过，不过生成的镜像文件太大，要精简。

去掉I2C和MMC卡驱动的支持，

PPP网络支持， Device Drivers--->Netowork device supprot-->PPP protocol

去掉WiFI的支持 Device Drivers--->Netowork device supprot-->Wireless LAN protocol

去掉WiFi支持后，编译成的内核大小为1.28M可使用了。

经过查阅资料得知，kprobe的使用，还需要有debugfs调试文件系统的配合，因此，需要让系统启动时，生成debugfs目录