---
title: GCD源码分析
categories: iOS多线程
---

## 一、Mach

Mach 是 XNU 的核心，被 BSD 层包装。XNU 由以下几个组件组成：

1. MACH 内核

	* 进程和线程抽象
	* 虚拟内存管理
	* 任务调度
	* 进程间通信和消息传递机制

2. BSD

	* UNIX进程模型
	* POSIX线程模型
	* UNIX用户与组
	* 网络协议栈
	* 文件系统访问
	* 设备访问

3. libKern
4. I/O Kit

Mach 的独特之处在于选择了<font color=#cc0000>通过消息传递的方式实现对象与对象之间的通信</font>。而其他架构一个对象要访问另一个对象需要通过一个大家都知道的接口，而 Mach 对象不能直接调用另一个对象，而是必须传递消息。

一条消息就像网络包一样，定义为透明的 <font color=#cc0000>blob(binary larger object，二进制大对象)</font>，通过固定的包头进行分装

```
typedef struct
{
        mach_msg_header_t       header;
        mach_msg_body_t         body;
} mach_msg_base_t;

typedef struct 
{
  mach_msg_bits_t   msgh_bits; // 消息头标志位
  mach_msg_size_t   msgh_size; // 大小
  mach_port_t       msgh_remote_port; // 目标(发消息)或源(接消息)
  mach_port_t       msgh_local_port; // 源(发消息)或目标(接消息)
  mach_port_name_t  msgh_voucher_port;
  mach_msg_id_t     msgh_id; // 唯一id
} mach_msg_header_t;
```

Mach 消息的发送和接收都是通过同一个 API 函数 `mach_msg()` 进行的。这个函数在用户态和内核态都有实现。为了实现消息的发送和接收，mach\_msg() 函数调用了一个 Mach 陷阱(trap)。Mach 陷阱就是 Mach 中和系统调用等同的概念。在用户态调用 `mach_msg_trap()` 会引发陷阱机制，切换到内核态，在内核态中，内核实现的 mach\_msg() 会完成实际的工作。这个函数也将会在下面的源码分析中遇到。

每一个 BSD 进程都在底层关联一个 Mach 任务对象，因为 Mach 提供的都是非常底层的抽象，提供的 API 从设计上讲很基础且不完整，所以需要在这之上提供一个更高的层次以实现完整的功能。<font color=#cc0000>开发层遇到的进程和线程就是 BSD 层对 Mach 的任务和线程的复杂包装</font>。

进程填充的是线程，而线程是二进制代码的实际执行单元。用户态的线程始于对 pthread\_create 的调用。这个函数的又由 bsdthread\_create 系统调用完成，而 bsdthread\_create 又其实是 Mach 中的 thread\_create 的复杂包装，说到底真正的线程创建还是由 Mach 层完成。

在 UNIX 中，进程不能被创建出来，都是通过 `fork()` 系统调用复制出来的。复制出来的进程都会被要加载的执行程序覆盖整个内存空间。

## 二、GCD 知识点摘录

![](http://dzliving.com/iOSGCD_0.png)

1. GCD 用到的 queue，无论是自己创建的，或是获取系统的 main queue还是 global queue，其最终都是落脚于 GCD root queue 中。 我们可以在代码中的任意位置创建 queue，但最终 GCD 管理的，也不过这 12 个 root queue，这种思路有点类似于命令模式，即<font color=#cc0000>分散创建，集中管理</font>。

	![](http://dzliving.com/QueueThread.png)

2. 有两种方式提交任务：

	* dispatch_async  异步不等待任务完成就返回
	* dispatch_sync  同步任务，需要等待任务完成

	这两种提交任务的方式有所不同：

	* dispatch_async：底层运用了线程池，会在和当前线程不同的线程上处理任务。
	* dispatch_sync ：一般不会新开启线程，而是在当前线程执行任务（比较特殊的是main queue，它会<font color=#cc0000>利用 main runloop 将任务提交到主线程来执行</font>），同时，它会阻塞当前线程，等待提交的任务执行完毕。当 target queue 是并发线程时，会直接执行任务。而 target queue 是串行队列时，会检测当前线程是否已经拥有了该串行队列，如果答案是肯定的，则会触发crash，这与老版本 GCD 中会触发死锁不同，因为在新版 GCD 中，已经加入了这种死锁检测机制，从而触发 crash，避免了调试困难的死锁的发生。

3. GCD 会发生死锁的情况必须同时满足 3 个条件，才会形成 task1 和 task2 相互等待的情况：

	* 串行队列正在执行 task1
	* 在 task1 中又向串行队列投递了 task2
	* task2 是以 dispatch_sync 方式投递的，会阻塞当前线程

4. dispatch async串行队列

	将 work 打包成 dispatch\_continuation\_t，然后将 dq 入队到响应的 root queue 中，root queue 中的线程池中的线程会被唤醒，执行线程函数 \_dispatch\_worker\_thread3，root queue 会被倾倒，执行 queue 中的任务。
	
5. dispatch_async 并行队列

	
6. dispatch_once

	其避免多线程的方法主要是用了 C++11 中的原子操作。这说一下 `dispatch_once_f_slow` 实现的大致思路：

	如果是线程第一次进入 `dispatch_once_f_slow` 方法，此时 \*vval == NULL, 进入 if 分支，执行单例方法。同时，通过原子操作 os\_atomic\_cmpxchg ， \*vval 被设置为 tail，也就是 `_dispatch_once_waiter_t`，指向一个空的 `_dispatch_once_waiter_s` 结构体。

	如果其他线程再次进入 `dispatch_once_f_slow` 方法，此时 *vval != NULL, 走 else 分支。在 else 分支里面主要是将与当前线程相关的 `_dispatch_once_waiter_t` 头插入 vval 列表中。然后，调用 `_dispatch_thread_event_wait(&dow.dow_event)` 阻塞当前线程。也就是说，在单例方法调用完毕前，其他要访问单例方法的线程都会被阻塞。

	回到第一个线程的 if 分支中，当调用完单例方法后（_dispatch\_client\_callout(ctxt, func)），会将 onceToken val 设置为 DLOCK\_ONCE\_DONE，表明已经执行过一次单例方法。同时，会将 val 之前的值赋值给 next。如果之前有其他线程进来过，根据步骤 2 中 val 的赋值，则 val 肯定不等于初始的 tail 值，而在 val 的链表中，保存了所有代表其他线程的`_dispatch_once_waiter_t`，这时候就会遍历这个链表，一次唤醒其他线程，直到 next 等于初始的 tail 为止。


7. Dispatch Source 是 BSD 系统内核惯有功能 kqueue 的包装，kqueue 是在 XNU 内核中发生各种事件时，在应用程序编程方执行处理的技术。它的 CPU 负荷非常小，尽量不占用资源。当事件发生时，Dispatch Source 会在指定的 Dispatch Queue 中执行事件的处理。


[GCD源码分析（一）](https://www.jianshu.com/p/bd629d25dc2e)


[GCD源码吐血分析(1)——GCD Queue](https://blog.csdn.net/u013378438/article/details/81031938)
[GCD源码吐血分析(2)——dispatch_async/dispatch_sync/dispatch_once/dispatch group](https://blog.csdn.net/u013378438/article/details/81076116)