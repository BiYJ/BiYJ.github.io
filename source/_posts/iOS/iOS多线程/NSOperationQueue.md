---
title: NSOperation、NSOperationQueue
categories: iOS多线程
---


## 一、NSOperation 和 NSOperationQueue 简介

NSOperation、NSOperationQueue 是苹果提供给开发者使用的一套多线程解决方案。<font color=#cc0000>实际上是基于 GCD 的更高一层的封装，完全面向对象</font>。但是比 GCD 更简单易用、代码可读性也更高。

为什么要使用 NSOperation、NSOperationQueue？

1.   添加在操作完成后执行的代码；
2.   添加操作之间的依赖关系，方便的控制执行顺序；
3.   设定操作执行的优先级；
4.   可以很方便的取消一个操作的执行。
5.   使用 KVO 观察对操作执行状态的更改：isExecuteing、isFinished、isCancelled。

## 二、NSOperation 和 NSOperationQueue 操作和操作队列

GCD 的一些概念同样适用于 NSOperation、NSOperationQueue，也有类似的任务（操作）和队列（操作队列）的概念。

* 操作（Operation）
    
	在 GCD 中，任务是放在 block 里的。在 NSOperation 中，使用 <font color=#c0000>NSOperation 子类</font> NSInvocationOperation、NSBlockOperation 或者自定义子类来封装操作。

* 操作队列（Operation Queues）
    
	即用来存放操作的队列。不同于 GCD 中的调度队列 FIFO（先进先出）的原则。NSOperationQueue 对于添加到队列中的操作，首先进入<font color=#cc0000>准备就绪的状态</font>（就绪状态取决于操作之间的依赖关系），然后进入就绪状态的操作的开始执行顺序（非结束执行顺序）由操作之间相对的优先级决定（优先级是操作对象自身的属性）。

	操作队列通过设置最大并发操作数（maxConcurrentOperationCount）来控制并发、串行。

	NSOperationQueue 为我们提供了两种不同类型的队列：主队列和自定义队列。主队列运行在主线程之上，而自定义队列在后台执行。

## 三、NSOperation 和 NSOperationQueue 使用步骤

NSOperation 需要配合 NSOperationQueue 来实现多线程。因为默认情况下，NSOperation 单独使用时是同步执行操作，配合 NSOperationQueue 能更好的实现异步执行。NSOperation 实现多线程的使用步骤分为三步：

1. 创建操作：先将需要执行的操作封装到一个 NSOperation 对象中。
2. 创建队列：创建 NSOperationQueue 对象。
3. 将操作加入到队列中：将 NSOperation 对象添加到 NSOperationQueue 对象中。

之后系统就会自动将 NSOperationQueue 中的 NSOperation 取出来，在新线程中执行操作。

## 四、NSOperation 和 NSOperationQueue 基本使用

#### 4.1 创建操作

NSOperation 是个抽象类，不能用来封装操作。只有使用它的子类来封装操作，有三种方式来封装操作。

1. 使用子类 NSInvocationOperation
2. 使用子类 NSBlockOperation
3. 自定义继承于 NSOperation 的子类，通过实现内部相应的方法来封装操作。

在不使用 NSOperationQueue，单独使用 NSOperation 的情况下是同步执行操作。

**NSInvocationOperation**

```
- (void)useInvocationOperation
{    
    NSInvocationOperation * operation = [[NSInvocationOperation alloc] initWithTarget:self
                                                                             selector:@selector(task)
                                                                               object:nil];
    [operation start];
}

- (void)task
{
    for (int i = 0; i < 2; i++) {
        NSLog(@"%@", [NSThread currentThread]);
    }
}

2019-03-19 18:03:18.840607+0800 Demo[46957:3117389] <NSThread: 0x600001ffd400>{number = 1, name = main}
2019-03-19 18:03:20.842158+0800 Demo[46957:3117389] <NSThread: 0x600001ffd400>{number = 1, name = main}
```

在主线程中单独使用使用子类 NSInvocationOperation 执行一个操作的情况下，操作是在当前线程执行的，没有开启新线程。

```
    [NSThread detachNewThreadSelector:@selector(useInvocationOperation) toTarget:self withObject:nil];

2019-03-19 18:06:22.667818+0800 Demo[46995:3119165] <NSThread: 0x6000025e4b40>{number = 3, name = (null)}
2019-03-19 18:06:22.668057+0800 Demo[46995:3119165] <NSThread: 0x6000025e4b40>{number = 3, name = (null)}
```

在其他线程中单独使用子类 NSInvocationOperation，操作是在当前调用的其他线程执行的，并没有开启新线程。

**NSBlockOperation**

```
- (void)useBlockOperation
{
    NSBlockOperation * operation = [NSBlockOperation blockOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"%@", [NSThread currentThread]);
        }
    }];
    
    [operation start];
}

2019-03-19 18:09:56.666612+0800 Demo[47047:3121571] <NSThread: 0x600003222940>{number = 1, name = main}
2019-03-19 18:09:56.666754+0800 Demo[47047:3121571] <NSThread: 0x600003222940>{number = 1, name = main}
```

在主线程中单独使用 NSBlockOperation 执行一个操作的情况下，操作是在当前线程执行的，并没有开启新线程。

NSBlockOperation 还提供了一个方法 addExecutionBlock:，通过 addExecutionBlock: 就可以为 NSBlockOperation 添加额外的操作。这些操作（包括 blockOperationWithBlock 中的操作）<font color=#cc0000>可以在不同的线程中并发执行</font>。只有当所有相关的操作已经完成执行时，才视为完成。如果添加的操作多的话，blockOperationWithBlock: 中的操作也可能会在非当前线程执行，这是由系统决定的，并不是说添加到 blockOperationWithBlock: 中的操作一定会在当前线程中执行。

```
- (void)useBlockOperationAddExecutionBlock
{    
    NSBlockOperation * op = [NSBlockOperation blockOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"1---%@", [NSThread currentThread]);
        }
    }];
    
    [op addExecutionBlock:^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"2---%@", [NSThread currentThread]);
        }
    }];
    [op addExecutionBlock:^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"3---%@", [NSThread currentThread]);
        }
    }];
    [op addExecutionBlock:^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"4---%@", [NSThread currentThread]);
        }
    }];
    [op addExecutionBlock:^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"5---%@", [NSThread currentThread]);
        }
    }];
    [op addExecutionBlock:^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"6---%@", [NSThread currentThread]);
        }
    }];
    [op addExecutionBlock:^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"7---%@", [NSThread currentThread]);
        }
    }];
    [op addExecutionBlock:^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"8---%@", [NSThread currentThread]);
        }
    }];
    
    [op start];
}

2019-03-19 18:14:55.099519+0800 Demo[47115:3124684] 2---<NSThread: 0x600002d51f80>{number = 4, name = (null)}
2019-03-19 18:14:55.099548+0800 Demo[47115:3124683] 3---<NSThread: 0x600002d5cb00>{number = 5, name = (null)}
2019-03-19 18:14:55.099549+0800 Demo[47115:3124621] 4---<NSThread: 0x600002d3e880>{number = 1, name = main}
2019-03-19 18:14:55.099555+0800 Demo[47115:3124682] 1---<NSThread: 0x600002d51f40>{number = 3, name = (null)}
2019-03-19 18:14:55.099717+0800 Demo[47115:3124621] 4---<NSThread: 0x600002d3e880>{number = 1, name = main}
2019-03-19 18:14:55.099720+0800 Demo[47115:3124684] 2---<NSThread: 0x600002d51f80>{number = 4, name = (null)}
2019-03-19 18:14:55.099730+0800 Demo[47115:3124682] 1---<NSThread: 0x600002d51f40>{number = 3, name = (null)}
2019-03-19 18:14:55.099735+0800 Demo[47115:3124683] 3---<NSThread: 0x600002d5cb00>{number = 5, name = (null)}
2019-03-19 18:14:55.099873+0800 Demo[47115:3124684] 5---<NSThread: 0x600002d51f80>{number = 4, name = (null)}
2019-03-19 18:14:55.099873+0800 Demo[47115:3124682] 6---<NSThread: 0x600002d51f40>{number = 3, name = (null)}
2019-03-19 18:14:55.099878+0800 Demo[47115:3124683] 7---<NSThread: 0x600002d5cb00>{number = 5, name = (null)}
2019-03-19 18:14:55.100183+0800 Demo[47115:3124621] 8---<NSThread: 0x600002d3e880>{number = 1, name = main}
2019-03-19 18:14:55.100414+0800 Demo[47115:3124684] 5---<NSThread: 0x600002d51f80>{number = 4, name = (null)}
2019-03-19 18:14:55.100590+0800 Demo[47115:3124682] 6---<NSThread: 0x600002d51f40>{number = 3, name = (null)}
2019-03-19 18:14:55.100763+0800 Demo[47115:3124683] 7---<NSThread: 0x600002d5cb00>{number = 5, name = (null)}
2019-03-19 18:14:55.100968+0800 Demo[47115:3124621] 8---<NSThread: 0x600002d3e880>{number = 1, name = main}
```

使用 NSBlockOperation 并调用方法 AddExecutionBlock: 的情况下，blockOperationWithBlock: 方法中的操作和 addExecutionBlock: 中的操作是在不同的线程中异步执行的，包括主线程一共有 4 个线程。而且，这次执行结果中 blockOperationWithBlock: 方法中的操作也不是在当前线程（主线程）中执行的。

一般情况下，如果一个 NSBlockOperation 对象封装了多个操作。<font color=#cc0000>NSBlockOperation 是否开启新线程，取决于操作的个数</font>。如果添加的操作的个数多，就会自动开启新线程。当然开启的线程数是由系统来决定的。

**自定义继承自 NSOperation 的子类**

如果 NSInvocationOperation、NSBlockOperation 不能满足日常需求，可以自定义继承于 NSOperation 的子类。

可以通过重写 main 或者 start 方法来定义自己的 NSOperation 对象。重写 main 方法比较简单，我们不需要管理操作的状态属性 isExecuting 和 isFinished。当 main 执行完返回的时候，这个操作就结束了。

先定义一个继承自 NSOperation 的子类，重写 main 方法。

```
#import <Foundation/Foundation.h>

@interface MyOperation : NSOperation
@end

@implementation MyOperation

- (void)main
{
    if (!self.isCancelled) {
        for (int i = 0; i < 2; i++) {
            NSLog(@"%@", [NSThread currentThread]);
        }
    }
}

@end

{
    [[[MyOperation alloc] init] start];
}
2019-03-19 18:25:22.022105+0800 Demo[47221:3129776] <NSThread: 0x600003486940>{number = 1, name = main}
2019-03-19 18:25:22.022289+0800 Demo[47221:3129776] <NSThread: 0x600003486940>{number = 1, name = main}
```

在主线程单独使用自定义继承于 NSOperation 的子类的情况下，是在主线程执行操作，并没有开启新线程。

#### 4.2 创建队列

NSOperationQueue 一共有两种队列：主队列、自定义队列。其中自定义队列同时包含了串行、并发功能。下边是主队列、自定义队列的基本创建方法和特点。

* 主队列

	除使用 addExecutionBlock: 添加的额外操作外，凡是添加到主队列中的操作都会放到主线程中执行。
	
	```
	NSOperationQueue * queue = [NSOperationQueue mainQueue];
	```

* 自定义队列（非主队列）

	添加到这种队列中的操作，就会自动放到子线程中执行。同时包含了：串行、并发功能。

	```
	NSOperationQueue * queue = [[NSOperationQueue alloc] init];
	```

#### 4.3 将操作加入到队列

NSOperation 需要配合 NSOperationQueue 来实现多线程。那么我们需要将创建好的操作加入到队列。总共有两种方法：

```
/**
 * 使用 addOperation: 将操作加入到操作队列中
 */
- (void)addOperationToQueue
{
    // 1.创建队列
    NSOperationQueue * queue = [[NSOperationQueue alloc] init];

    // 2.创建操作
    NSInvocationOperation * op1 = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(task1) object:nil];
    NSInvocationOperation * op2 = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(task2) object:nil];

    NSBlockOperation * op3 = [NSBlockOperation blockOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"3---%@", [NSThread currentThread]);
        }
    }];

    [op3 addExecutionBlock:^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"4---%@", [NSThread currentThread]);
        }
    }];

    // 3.添加
    [queue addOperation:op1]; // [op1 start]
    [queue addOperation:op2]; // [op2 start]
    [queue addOperation:op3]; // [op3 start]
}

- (void)task1
{
    for (int i = 0; i < 2; i++) {
        NSLog(@"1---%@", [NSThread currentThread]);
    }
}

- (void)task2
{
    for (int i = 0; i < 2; i++) {
        NSLog(@"2---%@", [NSThread currentThread]);
    }
}

2019-03-19 18:34:24.016396+0800 Demo[47313:3134164] 3---<NSThread: 0x6000000d2840>{number = 5, name = (null)}
2019-03-19 18:34:24.016394+0800 Demo[47313:3134163] 2---<NSThread: 0x6000000d2800>{number = 3, name = (null)}
2019-03-19 18:34:24.016400+0800 Demo[47313:3134165] 4---<NSThread: 0x6000000da480>{number = 6, name = (null)}
2019-03-19 18:34:24.016423+0800 Demo[47313:3134162] 1---<NSThread: 0x6000000da3c0>{number = 4, name = (null)}
2019-03-19 18:34:24.016590+0800 Demo[47313:3134164] 3---<NSThread: 0x6000000d2840>{number = 5, name = (null)}
2019-03-19 18:34:24.016609+0800 Demo[47313:3134163] 2---<NSThread: 0x6000000d2800>{number = 3, name = (null)}
2019-03-19 18:34:24.016620+0800 Demo[47313:3134165] 4---<NSThread: 0x6000000da480>{number = 6, name = (null)}
2019-03-19 18:34:24.016624+0800 Demo[47313:3134162] 1---<NSThread: 0x6000000da3c0>{number = 4, name = (null)}
```

使用 addOperation: 将操作加入到操作队列后能够开启新线程，进行并发执行。

```
/**
 * 使用 addOperationWithBlock: 将操作加入到操作队列中
 */
- (void)addOperationWithBlockToQueue
{
    // 1. 创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];
    
    // 2. 添加
    [queue addOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"1---%@", [NSThread currentThread]);
        }
    }];
    [queue addOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"2---%@", [NSThread currentThread]);
        }
    }];
    [queue addOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"3---%@", [NSThread currentThread]);
        }
    }];
}

2019-03-19 18:36:28.275696+0800 Demo[47351:3135582] 1---<NSThread: 0x600002d0df40>{number = 5, name = (null)}
2019-03-19 18:36:28.275818+0800 Demo[47351:3135583] 2---<NSThread: 0x600002d0df00>{number = 3, name = (null)}
2019-03-19 18:36:28.275846+0800 Demo[47351:3135581] 3---<NSThread: 0x600002d06440>{number = 4, name = (null)}
2019-03-19 18:36:28.275899+0800 Demo[47351:3135582] 1---<NSThread: 0x600002d0df40>{number = 5, name = (null)}
2019-03-19 18:36:28.275973+0800 Demo[47351:3135583] 2---<NSThread: 0x600002d0df00>{number = 3, name = (null)}
2019-03-19 18:36:28.275977+0800 Demo[47351:3135581] 3---<NSThread: 0x600002d06440>{number = 4, name = (null)}
```

使用 addOperationWithBlock: 将操作加入到操作队列后能够开启新线程，进行并发执行。

## 五、NSOperationQueue 控制串行执行、并发执行

NSOperationQueue 有个关键属性 maxConcurrentOperationCount 叫做最大并发操作数。用来控制一个特定队列中可以有多少个操作同时参与并发执行。

注意：<font color=#cc0000>maxConcurrentOperationCount 控制的不是并发线程的数量，而是一个队列中同时能并发执行的最大操作数</font>。而且一个操作也并非只能在一个线程中运行。

maxConcurrentOperationCount 默认情况下为 -1 表示不进行限制，可进行并发执行。= 1 为串行队列；\> 1 为并发队列。

当然这个值不应超过系统限制，即使自己设置一个很大的值，系统也会自动调整为 MIN{ 自定的值，系统默认最大值 }。

## 六、NSOperation 操作依赖

NSOperation、NSOperationQueue 最吸引人的地方是它能添加操作之间的依赖关系。

通过操作依赖，我们可以很方便的控制操作之间的执行先后顺序。NSOperation 提供了 3 个接口供开发者管理和查看依赖。

```
- (void)addDependency:(NSOperation *)op;    // 添加依赖，使当前操作依赖于操作 op 的完成
- (void)removeDependency:(NSOperation *)op; // 移除依赖，取消当前操作对操作 op 的依赖

@property (readonly, copy) NSArray<NSOperation *> *dependencies;  // 在当前操作开始执行之前完成执行的所有操作对象数组
```

有 A、B 两个操作，A 执行完操作，B 才能执行操作。如果使用依赖来处理的话，那么就需要让操作 B 依赖于操作 A。

```
// op2 依赖于 op1。先执行 op1，再执行 op2

[op2 addDependency:op1]; 
```

## 七、NSOperation 优先级

<font color=#cc0000>queuePriority 属性适用于同一操作队列中的操作</font>，不适用于不同操作队列中的操作。默认情况下，所有新创建的操作对象优先级都是 NSOperationQueuePriorityNormal。但是我们可以通过 setQueuePriority: 方法来改变当前操作在同一队列中的执行优先级。

```
// 优先级的取值
typedef NS_ENUM(NSInteger, NSOperationQueuePriority) {
    NSOperationQueuePriorityVeryLow = -8L,
    NSOperationQueuePriorityLow = -4L,
    NSOperationQueuePriorityNormal = 0,
    NSOperationQueuePriorityHigh = 4,
    NSOperationQueuePriorityVeryHigh = 8
};
```

对于添加到队列中的操作，首先进入准备就绪的状态（就绪状态取决于操作之间的依赖关系），然后进入就绪状态的操作的开始执行顺序由操作之间相对的优先级决定。那么，什么样的操作才是进入就绪状态的操作呢？

* 当一个操作的所有依赖都已经完成时，操作对象通常会进入准备就绪状态，等待执行。

	举个例子，现在有 4 个优先级都是 NSOperationQueuePriorityNormal（默认级别）的操作：op1、op2、op3、op4。其中 op3 依赖于 op2，op2 依赖于 op1，即 op3->op2->op1。现在将这 4 个操作添加到队列中并发执行。

	因为 op1 和 op4 都没有需要依赖的操作，所以在 op1、op4 执行之前，就是处于准备就绪状态的操作。

	而 op3 和 op2 都有依赖的操作，所以 op3 和 op2 都不是准备就绪状态下的操作。

	理解了进入就绪状态的操作，那么就理解了queuePriority 属性的作用对象。

* <font color=#cc0000>queuePriority 属性决定了进入准备就绪状态下的操作之间的开始执行顺序</font>。并且，<font color=#cc0000>优先级不能取代依赖关系</font>。
    
	如果一个队列中既包含高优先级操作，又包含低优先级操作，并且两个操作都已经准备就绪，那么队列先执行高优先级操作。如果 op1 和 op4 是不同优先级的操作，那么就会先执行两者中优先级高的操作。

	如果一个队列中既包含了准备就绪状态的操作，又包含了未准备就绪的操作，未准备就绪的操作优先级比准备就绪的操作优先级高。那么，虽然准备就绪的操作优先级低，也会优先执行。优先级不能取代依赖关系。如果要控制操作间的启动顺序，则必须使用依赖关系。

## 八、NSOperation 和 NSOperationQueue 线程间的通信

```
/**
 * 线程间通信
 */
- (void)communication
{

    // 1. 创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc]init];

    // 2. 添加操作
    [queue addOperationWithBlock:^{

        for (int i = 0; i < 2; i++) {
            NSLog(@"1---%@", [NSThread currentThread]);
        }

        // 回到主线程
        [[NSOperationQueue mainQueue] addOperationWithBlock:^{

            for (int i = 0; i < 2; i++) {
                NSLog(@"2---%@", [NSThread currentThread]);
            }
        }];
    }];
}

2019-03-19 18:58:11.699590+0800 Demo[47501:3143587] 1---<NSThread: 0x600003c44c40>{number = 3, name = (null)}
2019-03-19 18:58:11.699836+0800 Demo[47501:3143587] 1---<NSThread: 0x600003c44c40>{number = 3, name = (null)}
2019-03-19 18:58:11.705709+0800 Demo[47501:3143542] 2---<NSThread: 0x600003c1d400>{number = 1, name = main}
2019-03-19 18:58:11.706378+0800 Demo[47501:3143542] 2---<NSThread: 0x600003c1d400>{number = 1, name = main}
```

通过线程间的通信，先在其他线程中执行操作，等操作执行完了之后再回到主线程执行主线程的相应操作。

## 九、NSOperation 和 NSOperationQueue 线程同步和线程安全

1. 线程安全

	线程可能会同时运行一段代码，如果每次运行结果和单线程运行的结果是一样的，而且其他的变量的值也和预期的是一样的，就是线程安全的。

	若每个线程中对全局变量、静态变量只有读操作，而无写操作，一般来说，这个全局变量是线程安全的；若有多个线程同时执行写操作（更改变量），一般都需要考虑线程同步，否则的话就可能影响线程安全。

2. 线程同步

	线程 A 和 线程 B 一块配合，A 执行到一定程度时要依靠线程 B 的某个结果，于是停下来，示意 B 运行；B 依言执行，再将结果给 A；A 再继续操作。

## 十、NSOperation 和 NSOperationQueue 常用属性和方法归纳

```
/**
 * NSOperation
 */
- (void)cancel;   // 取消操作，实质是标记 isCancelled 状态。
- (void)waitUntilFinished; // 阻塞当前线程，直到该操作结束。可用于线程执行顺序的同步。
- (void)setCompletionBlock:(void (^)(void))block;  // 会在当前操作执行完毕时执行 block。取消/暂停/恢复操作- (void)cancelAllOperations; 可以取消队列的所有操作。

/**
 * NSOperationQueue
 */
- (void)waitUntilAllOperationsAreFinished;  // 阻塞当前线程，直到队列中的操作全部执行完毕。
- (void)addOperations:(NSArray *)ops waitUntilFinished:(BOOL)wait;  // 向队列中添加操作数组，wait 标志是否阻塞当前线程直到所有操作结束
```

这里的暂停和取消（操作和队列）并不代表可以将当前的操作立即取消，而是在当前的操作执行完毕之后不再执行新的操作。

暂停和取消的区别：暂停操作之后还可以恢复操作，继续向下执行；而取消操作之后，所有的操作就清空了，无法继续执行。

## 十一、文章

[行走少年郎](https://www.jianshu.com/u/6d09d915f1bf) & [iOS 多线程：『NSOperation、NSOperationQueue』详尽总结](https://www.jianshu.com/p/4b1d77054b35)
[苹果官方--并发编程指南：Operation Queues](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationObjects/OperationObjects.html)
[苹果官方文档：NSOperation](https://developer.apple.com/documentation/foundation/nsoperation?language=occ)
[Objc 中国：并发编程：API 及挑战](https://objccn.io/issue-2-1/)