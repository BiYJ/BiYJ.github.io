---
title: GCD实现
categories: iOS多线程
---

## 一、Dispatch Queue

不难想象 GCD 的实现需要使用以下这些工具。

①、用于管理追加的 Block 的 C 语言层实现的 <font color=#cc0000>FIFO 队列</font>；

②、Atomic 函数中实现的用于排他控制的<font color=#cc0000>轻量级信号</font>；

③、用于管理线程的 C 语言层实现的<font color=#cc0000>一些容器</font>。

但如果只用了这些内容便可实现，那么就不需要内核级的实现了。官方说明：

> 通常，应用程序中编写的线程管理用的代码要在系统级实现。

在系统（iOS 和 OS X 的核心 XNU 内核）级上的实现。因此，无论编程人员如何努力编写管理线程的代码，在性能方面都不可能超过 XNU 内核级所实现的 GCD。

使用 GCD 不必编写线程的创建、销毁等管理代码，而可以在线程中集中实现处理内容。我们应该尽量多使用 GCD 或者使用了 Cocoa 框架 GCD 的 NSOperationQueue 类等 API。

用于实现 Dispatch Queue 而使用的软件组件。

<center>
![1](https://upload-images.jianshu.io/upload_images/5294842-76ca86d8be6976a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

编程人员所使用的 GCD 的 API 全部为包含在 libdispatch 库的 C 语言函数。Dispatch Queue 通过<font color=#cc0000>结构体和链表</font>，被实现为 FIFO 队列。FIFO 队列管理通过 dispatch\_async 等函数所追加的 block。

Block 并不直接加入  FIFO 队列，而是先加入 Dispatch Continuation 这一个 <font color=#cc0000>dispatch\_continuation\_t</font> 类型结构体中，然后再加入 FIFO 队列。该 Dispatch Continuation 用于记录 Block 所属的 Dispatch Group 和一些其他信息，相当于一般常说的上下文。

<center>
![2](https://upload-images.jianshu.io/upload_images/5294842-7320d9613aed8c76.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

Dispatch Queue 可通过 <font color=#cc0000>dispatch\_set\_target\_queue</font> 函数设定执行该 Dispatch Queue 处理的 Dispatch Queue 为目标。该目标可像串珠子一样，设定多个连接在一起的 Dispatch Queue。但是在连接串的最后必须设定为 Main Dispatch Queue 或各种优先级的 Global Dispatch Queue，或者准备用于 Serial Dispatch Queue 的各种优先级的 Global Dispatch Queue。

Global Dispatch Queue 有 8 种：

*    Global Dispatch Queue（High Priority）
*    Global Dispatch Queue（Default Priority）
*    Global Dispatch Queue（Low Priority）
*    Global Dispatch Queue（Background Priority）
*    Global Dispatch Queue（High Overcommit Priority）
*    Global Dispatch Queue（Default Overcommit Priority）
*    Global Dispatch Queue（Low Overcommit Priority）
*    Global Dispatch Queue（Background Overcommit Priority）

优先级带有 <font color=#cc0000>Overcommit</font> 的 Global Dispatch Queue 使用在 Serial Dispatch Queue 中。如 Overcommit 这个名称所示，不管系统状态如何，都会<font color=#cc0000>强制生成</font>线程的 Dispatch Queue。

这 8 种 Global Dispatch Queue 各使用 1 个 pthread\_workqueue。GCD 初始化时，使用 <font color=#cc0000>pthread\_workqueue\_create\_np</font> 函数生成 pthread\_workqueue。

pthread\_workqueue 包含在 Libc 提供的 pthreads API 中。其使用 bsdthread\_register 和 workq\_open 系统调用，在初始化 XNU 内核的 workqueue 之后获取 workqueue 信息。

<center>
![image.png](https://upload-images.jianshu.io/upload_images/5294842-816faa7ca4c6ff96.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

XNU 内核持有 4 种 workqueue：

*    WORKQUEUE\_HIGH\_PRIOQUEUE
*    WORKQUEUE\_DEFAULT\_PRIOQUEUE
*    WORKQUEUE\_LOW\_PRIOQUEUE
*    WORKQUEUE\_BG\_PRIOQUEUE

该执行优先级和 Global Dispatch Queue 的 4 种优先级相同。

<center>
![9](https://upload-images.jianshu.io/upload_images/5294842-6e6083b134f2f075.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

当在 Global Dispatch Queue 中执行 Block 时，libdispatch 从 Global Dispatch Queue 自身的 FIFO 队列取出 Dispatch Continuation，调用 pthread\_workqueue\_additem\_np 函数，把该 Global Dispatch Queue 自身、符合其优先级的 workqueue 信息以及为执行 Dispatch Continuation 的回调函数等传递给参数。

pthread\_workqueue\_additem\_np 函数使用 workq\_kernreturn 系统调用，通知 workqueue 增加应当执行的项目。根据该通知，XNU 内核基于系统状态判断是否要生成线程。如果是 Overcommit 优先级的 Global Dispatch Queue，workqueue 则始终生成线程。

该线程虽然与 iOS 和 OS X 中通常使用的线程大致相同，但是有一部分 pthread API 不能使用。详见苹果官方文档《并行编码指南》的“与 POSIX 线程的互换性”。

另外，因为 workqueue 生成的线程在实现用于 workqueue 的线程计划表中运行，所以与一般线程的上下文切换不同。

workqueue 的线程执行 pthread\_workqueue 函数，该函数调用 libdispatch 的回调函数。在该回调函数中执行加入到 Dispatch Continuation 的 Block。

Block 执行结束后，进行通知 Dispatch Group 结束、释放 Dispatch Continuation 等处理，开始准备执行加入到 Global Dispatch Queue 中的下一个 Block。

总结：

①、block、block 所属的 group、其他信息加入的 dispatch\_continuation\_t 结构体中；

②、将 dispatch\_continuation\_t 加入 FIFO 队列；

③、Dispatch\_Queue 通过 dispatch\_set\_target\_queue 设置目标 Queue；

④、GCD 初始化时，pthread\_workqueue\_create\_np 生成 pthread\_workqueue;

⑤、pthread\_workqueue 使用 bsdthread\_register 和 workq\_open 系统调用，在初始化 XNU 内核的 workqueue 之后获取 workqueue 信息；

⑥、当在 Global Dispatch Queue 执行 Block 时 libdispatch 从 FIFO 队列中取出 Dispatch Continuation；

⑦、调用 pthread\_workqueue\_additem\_np 函数，将该 Global Dispatch Queue 自身、符合其优先级的 workqueue 信息以及为执行 Dispatch Continuation 的回调函数等传递给参数；

⑧、pthread\_workqueue\_additem\_np 内部调用 workq\_kernreturn 系统调用，通知 workqueue 增加应当执行的项目；

⑨、XNU 内核基于系统状态判断是否要生成线程；

⑩、workqueue 的线程执行 pthread\_workqueue 函数，该函数调用 libdispatch 的回调函数，在该回调函数中执行加入到 Dispatch Continuation 的 Block。Block 执行结束后，通知 Dispatch Group 结束、释放 Dispatch Continuation 等处理，开始准备下一个 Block 的执行。