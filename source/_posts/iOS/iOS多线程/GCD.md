---
title: GCD
categories: iOS多线程
---


## 一、GCD 简介

> [百度百科](http://baike.baidu.com/item/GCD)
> 
> Grand Central Dispatch(GCD) 是 Apple 开发的<font color=#cc0000>一个多核编程的较新的解决方法</font>。在 Mac OSX 10.6 首次推出，也可在 iOS4 及以上版本使用。它主要用于优化应用程序以支持多核处理器以及其他对称多处理系统。它是一个在线程池模式的基础上执行的并发任务。

使用 GCD 的好处：

*    GCD 可用于多核的并行运算
*    GCD 会<font color=#cc0000>自动利用</font>更多的 CPU 内核（比如双核、四核）
*    GCD 会<font color=#cc0000>自动管理</font>线程的生命周期（创建线程、调度任务、销毁线程），程序员只需要告诉 GCD 想要执行什么任务。
    

## 二、GCD 任务和队列

GCD 中两个核心概念：任务和队列。

#### 2.1 任务

要执行的操作，也就是在线程中执行的那段代码。在 GCD 是放在 block 里。

执行任务有两种方式：同步执行（sync）和异步执行（async）。

主要区别：<font color=#cc0000>是否等待队列的任务执行结束，以及是否具备开启新线程的能力</font>。

*   同步执行
 
    *   同步添加任务到指定的队列中，在添加的任务执行结束之前，会一直等待，直到队列里面的任务完成之后再继续执行。
    *   只能在当前线程中执行任务，不具备开启新线程的能力。
        
*   异步执行：
    
    *   异步添加任务到指定的队列中，它不会做任何等待，可以继续执行任务。
    *   可以在新的线程中执行任务，具备开启新线程的能力。
        

注意：异步执行（async）虽然具有开启新线程的能力，但是并不一定开启新线程，这跟任务所指定的队列类型有关。

#### 2.2 队列

指执行任务的等待队列，即用来存放任务的队列。

队列是一种特殊的线性表，采用 <font color=#cc0000>FIFO</font>（先进先出）的原则，即新任务总是被插入到队列的末尾，而读取任务的时候总是从队列的头部开始读取。每读取一个任务，则从队列中释放一个任务。队列的结构可参考下图：

<center>
![4](https://upload-images.jianshu.io/upload_images/1877784-01267bd211719167.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)
</center>

在 GCD 中有两种队列：串行队列（Serial Dispatch Queue）和并发队列（Concurrent Dispatch Queue）。两者都符合 FIFO（先进先出）的原则。

两者的主要区别是：执行顺序不同，以及开启线程数不同。

*   串行队列：
    
    *   每次只执行一个任务，任务一个接一个执行。（只开启一个线程）
        
*   并发队列：
    
    *   多个任务可以并发（同时）执行。（可以开启多个线程，并且同时执行任务）

注意：<font color=#cc0000>并发队列的并发功能只有在异步（dispatch\_async）函数下才有效</font>。

两者具体区别如下图：

<center>
![5](https://upload-images.jianshu.io/upload_images/1877784-4faca27116209f35.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)

![6](https://upload-images.jianshu.io/upload_images/1877784-97f3931d1b187b11.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)
</center>

## 三、GCD 的使用步骤

GCD 的使用只有两步：

1. 创建一个队列（串行队列或并发队列）

2. 将任务追加到任务的等待队列中，然后系统就会根据任务类型执行任务（同步执行或异步执行）。

#### 3.1 队列的创建方法/获取方法

*   使用 dispatch\_queue\_create 来创建队列。
    
	```
	/*
	   参数 1 : 表示队列的唯一标识符，用于 DEBUG，可以为空，推荐使用应用 BundleId 的逆序全程域名
	   参数 2 : 用于识别是串行队列、并发队列。DISPATCH_QUEUE_SERIAL:串行   DISPATCH_QUEUE_CONCURRENT:并发
	 */
	
	// 串行队列
	dispatch_queue_t queue = dispatch_queue_create("net.bujige.testQueue", DISPATCH_QUEUE_SERIAL);
	
	// 并发队列
	dispatch_queue_t queue = dispatch_queue_create("net.bujige.testQueue", DISPATCH_QUEUE_CONCURRENT); 
	```
	
*   GCD 提供了的一种特殊的串行队列：主队列（Main Dispatch Queue）
    
    *   所有放在主队列中的任务，都会放到主线程中执行。
        
    *   可使用 dispatch\_get\_main_queue() 获得主队列。
        
		```
		// 主队列
		dispatch\_queue\_t queue = dispatch\_get\_main_queue(); 
		```
	
*   GCD 默认提供了全局并发队列（Global Dispatch Queue）
    
	```
	/**
	 * 全局并发队列
	 * 
	 * 参数 1 : 队列优先级，一般用 DISPATCH_QUEUE_PRIORITY_DEFAULT
	 * 参数 2 : 暂时没用，传 0
	 */
	dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0); 
	```

#### 3.2 任务的创建方法

GCD 提供了同步执行任务的创建方法 dispatch\_sync 和异步执行任务创建方法 dispatch\_async。

```
// 同步
dispatch_sync(queue, ^{ 
    // do something
});

// 异步
dispatch_async(queue, ^{
    // do something
});
```

虽然使用 GCD 只需两步，但是既然有两种队列（串行队列/并发队列），两种任务执行方式（同步执行/异步执行），那么就有了四种不同的组合方式。这四种不同的组合方式是：

1.   同步执行 \+ 并发队列
2.   异步执行 \+ 并发队列
3.   同步执行 \+ 串行队列
4.   异步执行 \+ 串行队列

还有两种特殊队列：全局并发队列、主队列。因为主队列特殊，所以就又多了两种组合方式。这样就有 6 种不同的组合方式了。

5.   同步执行 \+ 主队列
6.   异步执行 \+ 主队列

<center>
![7](https://upload-images.jianshu.io/upload_images/5294842-c840e31e294a8e17.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

同步既然不会开辟新的线程，那同步阻塞的是当前线程还是当前队列？

<font color=#cc0000>阻塞当前队列，造成线程死锁</font>。

#### 3.3 GCD 线程间的通信

在 iOS 开发过程中，通常把一些耗时的操作放在其他线程，然后在主线程进行 UI 刷新。而当有时候在其他线程完成了耗时操作时，需要回到主线程，就用到了线程之间的通讯。

```
/**
 * 线程间通信
 */
- (void)communication
{
    // 获取全局并发队列
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    // 获取主队列
    dispatch_queue_t mainQueue = dispatch_get_main_queue(); 
    
    dispatch_async(queue, ^{
        // 异步追加任务
        for (int i = 0; i < 2; ++i) {
            [NSThread sleepForTimeInterval:2];          // 模拟耗时操作
            NSLog(@"1---%@", [NSThread currentThread]); // 打印当前线程
        }
        
        // 回到主线程
        dispatch_async(mainQueue, ^{
            // 追加在主线程中执行的任务
            [NSThread sleepForTimeInterval:2];          // 模拟耗时操作
            NSLog(@"2---%@", [NSThread currentThread]); // 打印当前线程
        });
    });
}

1---<NSThread: 0x600000271940>{number = 3, name = (null)}
1---<NSThread: 0x600000271940>{number = 3, name = (null)}
2---<NSThread: 0x60000007bf80>{number = 1, name = main}
```

可以看到在其他线程中先执行任务，执行完了之后回到主线程执行主线程的相应操作。

#### 3.4 GCD 的其他方法

##### 3.4.1 GCD 栅栏方法：dispatch\_barrier\_async

有时需要异步执行两组操作，而且第一组操作执行完之后，才能开始执行第二组操作。这样就需要有一个相当于栅栏一样的方法<font color=#cc0000>将两组异步执行的操作组（包含一个或多个任务）给分割开来</font>。这就需要用到 dispatch\_barrier\_async 方法。

dispatch\_barrier\_async 函数会等待前边追加到并发队列中的任务全部执行完毕之后，再将指定的任务追加到该异步队列。然后在 dispatch\_barrier\_async 函数追加的任务执行完毕之后，异步队列才恢复为一般动作，接着追加任务到该异步队列并开始执行。

具体如下图所示：

<center>
![8](https://upload-images.jianshu.io/upload_images/1877784-4d6d77fafd3ad007.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)
</center>

```
    // 并行队列
    dispatch_queue_t queue = dispatch_queue_create("com.dubinbin.Demo", DISPATCH_QUEUE_CONCURRENT);

    dispatch_async(queue, ^{
        NSLog(@"1111");
    });

    dispatch_async(queue, ^{
        NSLog(@"2222");
    });

    dispatch_barrier_async(queue, ^{
        NSLog(@"barrier");
    });
    
    dispatch_async(queue, ^{
        NSLog(@"3333");
    });
    
    dispatch_async(queue, ^{
        NSLog(@"4444");
    });
    
    NSLog(@"end");

2019-03-19 14:47:01.584536+0800 Demo[45246:3034531] end
2019-03-19 14:47:01.584562+0800 Demo[45246:3034564] 2222
2019-03-19 14:47:01.584566+0800 Demo[45246:3034562] 1111
2019-03-19 14:47:01.584676+0800 Demo[45246:3034562] barrier
2019-03-19 14:47:01.584755+0800 Demo[45246:3034562] 3333
2019-03-19 14:47:01.584765+0800 Demo[45246:3034564] 4444
```

dispatch\_barrier\_async 执行结果：在执行完栅栏前面的任务之后，才执行栅栏操作，最后再执行栅栏后边的任务。

##### 3.4.2 GCD 延时执行方法：dispatch\_after

经常会遇到这样的需求：在指定时间（例如 3 秒）之后执行某个任务。可以用 GCD 的 dispatch\_after 函数来实现。

注意：<font color=#cc0000>dispatch\_after 函数并不是在指定时间之后才开始执行处理，而是在指定时间之后将任务追加到主队列中</font>。严格来说，这个时间并不是绝对准确的，但想要大致延迟执行任务，dispatch\_after 函数是很有效的。

```
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(5 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        NSLog(@"Delay in Thread : %@", [NSThread currentThread]);
    });
    
    NSLog(@"end");

14:52:01.582676+0800  end
14:52:06.583078+0800  Delay in Thread : <NSThread: 0x600001026940>{number = 1, name = main}
```

dispatch\_after 执行结果：打印 end 之后大约 5.0 秒打印了 Delay in Thread。

##### 3.4.3 GCD 一次性代码：dispatch\_once

在创建单例或者整个程序运行过程中只执行一次的代码时，就用到了 GCD 的 dispatch\_once 函数。使用 dispatch\_once 函数能保证某段代码在程序运行过程中只被执行 1 次，并且即使在多线程的环境下，dispatch\_once 也可以保证线程安全。

```
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        // code to be executed once
    });
```

##### 3.4.4 GCD 快速迭代方法：dispatch\_apply

通常是用 for 循环遍历，但是 GCD 提供了快速迭代的函数 dispatch\_apply。<font color=#cc0000>dispatch\_apply 按照指定的次数将指定的任务追加到指定的队列中，并等待全部队列执行结束</font>。

如果是在串行队列（单线程）中使用 dispatch\_apply，那么就和 for 循环一样，按顺序同步执行，体现不出快速迭代的意义。

dispatch\_apply 可以在多个线程中同时（异步）遍历多个数字。

注意：无论是串行队列，还是异步队列，dispatch\_apply 都会等待全部任务执行完毕，这点就像是同步操作，也像是队列组中的 dispatch\_group\_wait 方法。

```
    dispatch_apply(5, queue, ^(size_t index) {
        NSLog(@"%zd---%@", index, [NSThread currentThread]);
    });
    
    NSLog(@"end");

1---<NSThread: 0x6000031499c0>{number = 3, name = (null)}
3---<NSThread: 0x600003149a00>{number = 5, name = (null)}
0---<NSThread: 0x600003126fc0>{number = 1, name = main}
2---<NSThread: 0x6000031414c0>{number = 4, name = (null)}
4---<NSThread: 0x6000031414c0>{number = 4, name = (null)}
end
```

因为是在并发队列中异步执行任务，所以各个任务的执行时间长短不定，最后结束顺序也不定，但是 end 一定在最后执行。

##### 3.4.5 GCD 队列组：dispatch\_group

有时候会有这样的需求：分别异步执行 2 个耗时任务，然后当 2 个耗时任务都执行完毕后再回到主线程执行任务。这时候就可以用到 GCD 的队列组。

*   调用队列组的 dispatch\_group\_async 先把任务放到队列中，然后将队列放入队列组中。或者使用队列组的 dispatch\_group\_enter、dispatch\_group\_leave 组合来实现。
    
*   调用队列组的 dispatch\_group\_notify 回到指定线程执行任务。或者使用 dispatch\_group\_wait 回到当前线程继续向下执行（会阻塞当前线程）。
    

**dispatch\_group\_notify**

监听 group 中任务的完成状态，当所有的任务都执行完成后，追加任务到 group 并执行。

```
    dispatch_queue_t queue = dispatch_queue_create("com.dubinbin.Demo", DISPATCH_QUEUE_CONCURRENT);
    dispatch_group_t group = dispatch_group_create();

    dispatch_group_async(group, queue, ^{
        NSLog(@"1111");
    });

    dispatch_group_async(group, queue, ^ {
        NSLog(@"2222");
    });

    dispatch_group_notify(group, queue, ^{
        NSLog(@"group");
    });

    dispatch_group_async(group, queue, ^{
        NSLog(@"3333");
    });

    dispatch_group_async(group, queue, ^{
        NSLog(@"4444");
    });
    
    NSLog(@"end");

2019-03-19 15:07:48.576680+0800 Demo[45472:3045696] end
2019-03-19 15:07:48.576680+0800 Demo[45472:3045738] 2222
2019-03-19 15:07:48.576687+0800 Demo[45472:3045736] 1111
2019-03-19 15:07:48.576702+0800 Demo[45472:3045735] 4444
2019-03-19 15:07:51.580427+0800 Demo[45472:3045737] 3333
2019-03-19 15:07:51.580874+0800 Demo[45472:3045737] group
```

dispatch\_group\_notify 运行结果：当所有任务都执行完成之后，才执行 dispatch\_group\_notify block 中的任务。

注意：barrier 和 group 的区别，<font color=#cc0000>barrier 只等待之前的任务执行完，group 需要等待加入到 group 中所有的任务执行完</font>。

**dispatch\_group\_wait**

暂停当前线程（阻塞当前线程），等待指定的 group 中的任务执行完成后，才会往下继续执行。

```
    dispatch_queue_t queue = dispatch_queue_create("com.dubinbin.Demo", DISPATCH_QUEUE_CONCURRENT);
    dispatch_group_t group = dispatch_group_create();

    dispatch_group_async(group, queue, ^{
        NSLog(@"1111");
    });

    dispatch_group_async(group, queue, ^ {
        NSLog(@"2222");
    });

    dispatch_group_notify(group, queue, ^{
        NSLog(@"group");
    });

    dispatch_group_async(group, queue, ^{
        NSLog(@"3333");
    });

    dispatch_group_async(group, queue, ^{
        [NSThread sleepForTimeInterval:5];
        NSLog(@"4444");
    });
    
    dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
    
    NSLog(@"end");

2019-03-19 15:14:18.051069+0800 Demo[45564:3049581] 2222
2019-03-19 15:14:18.051069+0800 Demo[45564:3049578] 1111
2019-03-19 15:14:18.051089+0800 Demo[45564:3049579] 3333
2019-03-19 15:14:23.055943+0800 Demo[45564:3049580] 4444
2019-03-19 15:14:23.056329+0800 Demo[45564:3049532] end
2019-03-19 15:14:23.056329+0800 Demo[45564:3049580] group
```

dispatch\_group\_wait 运行结果：dispatch\_group\_wait 会阻塞当前线程，当所有任务执行完成再继续向下执行。

**dispatch\_group\_enter、dispatch\_group\_leave**

dispatch\_group\_enter 标志着一个任务追加到 group，相当于 group 中未执行完毕任务数 +1

dispatch\_group\_leave 标志着一个任务离开了 group，相当于 group 中未执行完毕任务数 -1。

当 group 中未执行完毕任务数为 0 时，才会使 dispatch\_group\_wait 解除阻塞，并执行追加到 dispatch\_group\_notify 中的任务。

```
    dispatch_queue_t queue = dispatch_queue_create("com.dubinbin.Demo", DISPATCH_QUEUE_CONCURRENT);
    dispatch_group_t group = dispatch_group_create();

    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        NSLog(@"1111");
        dispatch_group_leave(group);
    });

    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        NSLog(@"2222");
        dispatch_group_leave(group);
    });

    dispatch_group_notify(group, queue, ^{
        NSLog(@"group");
    });

    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        [NSThread sleepForTimeInterval:5];
        NSLog(@"3333");
        dispatch_group_leave(group);
    });

    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        NSLog(@"4444");
        dispatch_group_leave(group);
    });
    
    dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
    
    NSLog(@"end");

2019-03-19 15:45:32.294390+0800 Demo[45813:3061727] 1111
2019-03-19 15:45:32.294399+0800 Demo[45813:3061725] 2222
2019-03-19 15:45:32.294411+0800 Demo[45813:3061728] 4444
2019-03-19 15:45:37.299935+0800 Demo[45813:3061726] 3333
2019-03-19 15:45:37.300306+0800 Demo[45813:3061728] group
2019-03-19 15:45:37.300309+0800 Demo[45813:3061668] end
```

dispatch\_group\_enter、dispatch\_group\_leave 运行结果：这里的 dispatch\_group\_enter、dispatch\_group\_leave 组合等同于 dispatch\_group\_async。当所有任务执行完成之后，才执行 dispatch\_group\_notify 中的任务。

##### 3.4.6 GCD 信号量：dispatch\_semaphore

GCD 中的信号量是指 Dispatch Semaphore，<font color=#cc0000>是持有计数的信号</font>。

在 Dispatch Semaphore 中，使用计数来完成这个功能，计数为 0 时等待；计数 ≥ 1 时不等待。

Dispatch Semaphore 提供了三个函数。

*   dispatch\_semaphore\_create：创建一个 Semaphore 并初始化信号的总量
    
*   dispatch\_semaphore\_signal：发送一个信号，让信号总量 +1
    
*   dispatch\_semaphore\_wait：使总信号量 -1，当信号总量为 0 时一直等待（阻塞所在线程），否则正常执行。
    

注意：想清楚你需要处理哪个线程等待（阻塞），又要哪个线程继续执行，然后才能使用信号量。

Dispatch Semaphore 在实际开发中主要用于：

*   保持线程同步，将异步执行任务转换为同步执行任务
    
*   保证线程安全，为线程加锁
    

**Dispatch Semaphore 线程同步**

在开发中会遇到这样的需求：异步执行耗时任务，并使用异步执行的结果进行一些额外的操作。换句话说，相当于将异步执行任务转换为同步执行任务。比如 AFNetworking 中 AFURLSessionManager.m 的 tasksForKeyPath: 方法，通过引入信号量的方式，等待异步执行任务结果，获取到 tasks，然后再返回该 tasks。

```
- (NSArray *)tasksForKeyPath:(NSString *)keyPath 
{
    __block NSArray * tasks = nil;

    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    
    [self.session getTasksWithCompletionHandler:^( NSArray * dataTasks, 
                                                   NSArray * uploadTasks,
                                                   NSArray * downloadTasks) {

        if ([keyPath isEqualToString:NSStringFromSelector(@selector(dataTasks))]) {
            tasks = dataTasks;
        }
        else if ([keyPath isEqualToString:NSStringFromSelector(@selector(uploadTasks))]) {
            tasks = uploadTasks;
        }
        else if ([keyPath isEqualToString:NSStringFromSelector(@selector(downloadTasks))]) {
            tasks = downloadTasks;
        }
        else if ([keyPath isEqualToString:NSStringFromSelector(@selector(tasks))]) {
            tasks = [@[dataTasks, uploadTasks, downloadTasks] valueForKeyPath:@"@unionOfArrays.self"];
        }

        dispatch_semaphore_signal(semaphore);
    }];

    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);

    return tasks;
}
```

下面，我们来利用 Dispatch Semaphore 实现线程同步，将异步执行任务转换为同步执行任务。

**Dispatch Semaphore 线程安全和线程同步（为线程加锁）**

<font color=#cc0000>线程安全</font>

如果你的代码所在的进程中有多个线程在同时运行，而这些线程可能会同时运行这段代码。如果每次运行结果和单线程运行的结果是一样的，而且其他的变量的值也和预期的是一样的，就是线程安全的。

若每个线程中对全局变量、静态变量只有读操作，而无写操作，一般来说，这个全局变量是线程安全的；若有多个线程同时执行写操作（更改变量），一般都需要考虑线程同步，否则的话就可能影响线程安全。

<font color=#cc0000>线程同步</font>

可以理解为线程 A 和 线程 B 一块配合，A 执行到一定程度时要依靠线程 B 的某个结果，于是停下来，示意 B 运行；B 依言执行，再将结果给 A；A 再继续操作。

##### 3.4.7 dispatch\_set\_target\_queue

```
/**
 * 参数 1 : 要执行变更的队列（不能指定主队列和全局队列）
 * 参数 2 : 目标队列（指定全局队列）
 */
dispatch_set_target_queue(dispatch_object_t object, dispatch_queue_t queue);
```

dispatch\_set\_target\_queue 函数有两个作用：①、变更队列的执行优先级；②、目标队列可以成为原队列的执行阶层。

```
    // 串行队列，默认优先级
    dispatch_queue_t serialQueue = dispatch_queue_create("com.gcd.setTargetQueue.serialQueue", NULL);
    // 串行队列（参照），默认优先级
    dispatch_queue_t serialDefaultQueue = dispatch_queue_create("com.gcd.setTargetQueue.serialDefaultQueue", NULL);
    
    // 1. 变更前
    dispatch_async(serialQueue, ^{
        NSLog(@"1");
    });
    dispatch_async(serialDefaultQueue, ^{
        NSLog(@"2");
    });
    
    // 全局队列，后台优先级
    dispatch_queue_t globalDefaultQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);
    // 2. 变更优先级
    dispatch_set_target_queue(serialQueue, globalDefaultQueue);
    
    // 3. 变更后
    dispatch_async(serialQueue, ^{
        NSLog(@"1");
    });
    dispatch_async(serialDefaultQueue, ^{
        NSLog(@"2");
    });

-------------------- 1 -------------------- 
2019-03-19 23:25:27.663809+0800 Demo[41898:2730565] 2
2019-03-19 23:25:27.663825+0800 Demo[41898:2730566] 1
2019-03-19 23:25:27.664036+0800 Demo[41898:2730565] 2
2019-03-19 23:25:27.664039+0800 Demo[41898:2730566] 1

-------------------- 2 -------------------- 
2019-03-19 23:25:27.663809+0800 Demo[41898:2730565] 1
2019-03-19 23:25:27.663825+0800 Demo[41898:2730566] 2
2019-03-19 23:25:27.664036+0800 Demo[41898:2730565] 2
2019-03-19 23:25:27.664039+0800 Demo[41898:2730566] 1
```

变更优先级前，serialQueue、serialDefaultQueue 打印随机；变更后，serialDefaultQueue 必定先于 serialQueue 打印。

```
    dispatch_queue_t serialQueue1 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue1", NULL);
    dispatch_queue_t serialQueue2 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue2", NULL);
    dispatch_queue_t serialQueue3 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue3", NULL);
    dispatch_queue_t serialQueue4 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue4", NULL);
    dispatch_queue_t serialQueue5 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue5", NULL);
    
    dispatch_async(serialQueue1, ^{
        NSLog(@"1");
    });
    dispatch_async(serialQueue2, ^{
        NSLog(@"2");
    });
    dispatch_async(serialQueue3, ^{
        NSLog(@"3");
    });
    dispatch_async(serialQueue4, ^{
        NSLog(@"4");
    });
    dispatch_async(serialQueue5, ^{
        NSLog(@"5");
    });

2019-03-19 23:31:27.856574+0800 Demo[41983:2736476] 1
2019-03-19 23:31:27.856583+0800 Demo[41983:2736474] 4
2019-03-19 23:31:27.856585+0800 Demo[41983:2736475] 2
2019-03-19 23:31:27.856586+0800 Demo[41983:2736477] 5
2019-03-19 23:31:27.856586+0800 Demo[41983:2736493] 3
```

未设置目标队列前，异步执行任务不阻塞，所以打印随机。

```
    dispatch_queue_t serialQueue1 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue1", NULL);
    dispatch_queue_t serialQueue2 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue2", NULL);
    dispatch_queue_t serialQueue3 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue3", NULL);
    dispatch_queue_t serialQueue4 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue4", NULL);
    dispatch_queue_t serialQueue5 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue5", NULL);

    dispatch_queue_t targetSerialQueue = dispatch_queue_create("com.gcd.setTargetQueue2.targetSerialQueue", NULL);

    //设置执行阶层
    dispatch_set_target_queue(serialQueue1, targetSerialQueue);
    dispatch_set_target_queue(serialQueue3, targetSerialQueue);
    dispatch_set_target_queue(serialQueue2, targetSerialQueue);
    dispatch_set_target_queue(serialQueue4, targetSerialQueue);
    dispatch_set_target_queue(serialQueue5, targetSerialQueue);

    dispatch_async(serialQueue1, ^{
        NSLog(@"1");
    });
    dispatch_async(serialQueue2, ^{
        NSLog(@"2");
    });
    dispatch_async(serialQueue3, ^{
        NSLog(@"3");
    });
    dispatch_async(serialQueue4, ^{
        NSLog(@"4");
    });
    dispatch_async(serialQueue5, ^{
        NSLog(@"5");
    });

2019-03-19 23:34:04.912115+0800 Demo[42048:2739515] 1
2019-03-19 23:34:04.912432+0800 Demo[42048:2739515] 2
2019-03-19 23:34:04.912573+0800 Demo[42048:2739515] 3
2019-03-19 23:34:04.912692+0800 Demo[42048:2739515] 4
2019-03-19 23:34:04.912874+0800 Demo[42048:2739515] 5
```

情况一：先设置目标队列，后添加任务，按顺序执行。

```
    dispatch_queue_t serialQueue1 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue1", NULL);
    dispatch_queue_t serialQueue2 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue2", NULL);
    dispatch_queue_t serialQueue3 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue3", NULL);
    dispatch_queue_t serialQueue4 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue4", NULL);
    dispatch_queue_t serialQueue5 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue5", NULL);
    
    dispatch_async(serialQueue1, ^{
        NSLog(@"1");
    });
    dispatch_async(serialQueue2, ^{
        NSLog(@"2");
    });
    dispatch_async(serialQueue3, ^{
        NSLog(@"3");
    });
    dispatch_async(serialQueue4, ^{
        NSLog(@"4");
    });
    dispatch_async(serialQueue5, ^{
        NSLog(@"5");
    });
    
    dispatch_queue_t targetSerialQueue = dispatch_queue_create("com.gcd.setTargetQueue2.targetSerialQueue", NULL);

    //设置执行阶层
    dispatch_set_target_queue(serialQueue1, targetSerialQueue);
    dispatch_set_target_queue(serialQueue2, targetSerialQueue);
    dispatch_set_target_queue(serialQueue3, targetSerialQueue);
    dispatch_set_target_queue(serialQueue4, targetSerialQueue);
    dispatch_set_target_queue(serialQueue5, targetSerialQueue);

2019-03-19 23:36:07.791578+0800 Demo[42077:2741664] 1
2019-03-19 23:36:07.791620+0800 Demo[42077:2741665] 4
2019-03-19 23:36:07.791578+0800 Demo[42077:2741663] 3
2019-03-19 23:36:07.791578+0800 Demo[42077:2741666] 2
2019-03-19 23:36:07.791635+0800 Demo[42077:2741693] 5
```

情况二：先添加任务，后设置目标队列，乱序输出。

在必须将不可并行执行的处理追加到多个 Serial Dispatch Queue 中时，如果使用 dispatch\_set\_target\_queue 函数将目标指定为某一个 Serial Dispatch Queue，即可防止处理并行执行。

## 四、文章

[iOS多线程：『GCD』详尽总结](https://www.jianshu.com/p/2d57c72016c6)
[iOS GCD之dispatch_semaphore（信号量）](http://blog.csdn.net/liuyang11908/article/details/70757534)
[iOS多线程：『pthread、NSThread』详尽总结](https://www.jianshu.com/p/cbaeea5368b1)
[iOS多线程：『NSOperation』详尽总结](https://www.jianshu.com/p/4b1d77054b35)
[iOS多线程：『RunLoop』详尽总结](https://www.jianshu.com/p/d260d18dd551)