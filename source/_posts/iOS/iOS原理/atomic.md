---
title: atomic
categories: iOS原理
---

<font color=#cc0000>atomic 在 set 方法里加了锁</font>，防止了多线程一直去写这个 property，造成难以预计的数值。

当属性使用 atomic 修饰时，它的读和写是原子性的：当线程 A 进行写操作，这时其他线程的读或者写操作会因为该操作而等待。当 A 线程的写操作结束后，B 线程进行写操作，然后当 A 线程需要读操作时，获得了在 B 线程中修改的值。如果有 C 线程在 A 线程读操作之前 release 了该属性，可能导致程序崩溃。

导致崩溃并不是线程安全问题。<font color=#cc0000>所谓线程安全是保证同一时间只有一个线程对该内存进行访问</font>。只要我们使用 getter、setter 方法来访问，上面的表述中的每一个步骤都只有一条线程在访问该内存，哪个线程会获得锁完全取决于代码顺序，这个崩溃就是程序员自身的问题了。如果绕开 getter、setter 方法访问这个属性，才会造成线程不安全，比如使用 KVC。

## 一、atomic 是绝对安全的

在 64 位的操作系统下，所有类型的指针(包括 void *)都是占用 8 个字节的。超过 4 个字节的基本类型数据都会有线程并发的问题。

那所有的指针类型都会有这个问题。

以 Objective-C 的 NSArray * 为例子，如果一个多线程操作这个数据，会有两个层级的并发问题：

1. 指针本身
2. 指针所指向的内存

指针本身也是占用内存的，并且一定是 <font color=#cc0000>8</font> 个字节。第二部分，指针所指向的内存，有可能非常大，有可能也就 1 个字节。

所以考虑 NSArray * array 这个数据在进行多线程操作的时候，必须分成两部分来描述，一个是 &array 这个指针本身，另一个则是它所指向的内存 array。想象现在有两块内存，一块是 8 字节，一块 n 字节，8 字节里面放的值，就是 n 字节内存的首地址。

如果用 atomic 修饰之后，会有什么影响？

从内存的角度来解释这个过程。<font color=#cc0000>atomic 其实修饰的是这个指针 &array</font>，与指针指向的第二部分 n 字节数据没有任何关系，被 atomic 修饰之后，你不可能随意去多线程操作这个 8 字节，但是对 8 字节里面所指向的 n 字节没有任何限制！

atomic 已经完美的履行了它的指责，你不可能对这个 8 字节进行无序的多线程操作，这就够了呀！有问题的是程序员，程序员并未对 n 字节做任何的限制。

## 二、NSMutableArray 本身是线程不安全的

简单来说，线程安全就是多个线程访问同一段代码，程序不会异常、不 Crash。而编写线程安全的代码主要依靠线程同步。

1. 不使用 atomic 修饰属性。原因有二：

	- atomic 的内存管理语义是原子性的，<font color=#cc0000>仅保证</font>了属性的 setter 和 getter 方法是原子性的、线程安全的，但是属性的其他方法，如数组添加/移除元素等并不是原子操作，所以不能保证属性是线程安全的。

	- atomic 虽然保证了 getter、setter 方法线程安全，但是付出的<font color=#cc0000>代价很大</font>，执行效率要比 nonatomic 慢很多倍(有说法是慢 10-20 倍)。

	总之：使用 nonatomic 修饰 NSMutableArray 对象就可以了，而使用锁、dispatch_queue 来保证 NSMutableArray 对象的线程安全。

2. 打造线程安全的 NSMutableArray

	在**《Effective Objective-C 2.0》书中第 41 条：**多用派发队列，少用同步锁中指出：使用“串行同步队列”(serial synchronization queue)，将读取操作及写入操作都安排在同一个队列里，即可保证数据同步。而通过并发队列，结合GCD 的栅栏块(barrier)来不仅实现数据同步线程安全，还比串行同步队列方式更高效。

	<center>
	![GCD 的栅栏块作用示意图](https://upload-images.jianshu.io/upload_images/5294842-5fb527a419acf5a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)
	</center>

	**说明**：栅栏块单独执行，不能与其他块并行。直到当前所有并发块都执行完毕，才会单独执行这个栅栏块
	
	线程安全实现如下：
	
	```
	@interface QSThreadSafeMutableArray()
	@property (nonatomic, strong) NSMutableArray * MDataArray;
	@property (nonatomic, strong) dispatch_queue_t MSyncQueue;
	@end
	
	@implementation QSThreadSafeMutableArray
	- (instancetype)initCommon
	{
	    if (self = [super init]) {
	        // %p 以 16 进制的形式输出内存地址，附加前缀 0x
	        NSString * uuid = [NSString stringWithFormat:@"com.jzp.array_%p", self];
	        // 注意：_MSyncQueue 是并行队列
	        _MSyncQueue = dispatch_queue_create([uuid UTF8String], DISPATCH_QUEUE_CONCURRENT);
	    }
	    return self;
	}
	
	- (instancetype)init
	{
	    if (self = [self initCommon]) {
	        _MDataArray = [NSMutableArray array];
	    }
	    return self;
	}
	
	- (id)objectAtIndex:(NSUInteger)index
	{
	    __block id obj;
	    
	    dispatch_sync(_MSyncQueue, ^{
	        if (index < [_MDataArray count]) {
	            obj = _MDataArray[index];
	        }
	    });
	    return obj;
	}
	
	- (NSEnumerator *)objectEnumerator
	{
	    __block NSEnumerator * enu;
	    
	    dispatch_sync( _MSyncQueue, ^{
	        enu = [_MDataArray objectEnumerator];
	    });
	    return enu;
	}
	
	- (void)insertObject:(id)anObject atIndex:(NSUInteger)index
	{
	    dispatch_barrier_async( _MSyncQueue, ^{
	        if (anObject && index < [_MDataArray count]) {
	            [_MDataArray insertObject:anObject atIndex:index];
	        }
	    });
	}
	
	- (void)addObject:(id)anObject
	{
	    dispatch_barrier_async( _MSyncQueue, ^{
	        if(anObject){
	            [_MDataArray addObject:anObject];
	        }
	    });
	}
	
	- (void)removeObjectAtIndex:(NSUInteger)index
	{
	    dispatch_barrier_async( _MSyncQueue, ^{
	        if (index < [_MDataArray count]) {
	            [_MDataArray removeObjectAtIndex:index];
	        }
	    });
	}
	
	- (void)removeLastObject
	{
	    dispatch_barrier_async( _MSyncQueue, ^{
	        [_MDataArray removeLastObject];
	    });
	}
	
	- (void)replaceObjectAtIndex:(NSUInteger)index withObject:(id)anObject
	{
	    dispatch_barrier_async( _MSyncQueue, ^{
	        if (anObject && index < [_MDataArray count]) {
	            [_MDataArray replaceObjectAtIndex:index withObject:anObject];
	        }
	    });
	}
	
	- (NSUInteger)indexOfObject:(id)anObject
	{
	    __block NSUInteger index = NSNotFound;
	    
	    dispatch_sync( _MSyncQueue, ^{
	        for (int i = 0; i < [_MDataArray count]; i ++) {
	            if ([_MDataArray objectAtIndex:i] == anObject) {
	                index = i;
	                break;
	            }
	        }
	    });
	    return index;
	}
	
	- (void)dealloc
	{
	    if (_MSyncQueue) {
	        _MSyncQueue = NULL;
	    }
	}
	
	@end
	```
	
	说明 ①：使用 dispatch queue 实现线程同步；将同步与异步派发结合起来，可以实现与普通加锁机制一样的同步行为，又不会阻塞执行异步派发的线程；使用同步队列及栅栏块，可以令同步行为更加高效。
	
	说明 ②：NSMutableDictionary 本身也是线程不安全的，实现线程安全的 NSMutableDictionary 原理同线程安全的NSMutableArray。(代码见 [QSUseCollectionDemo](https://github.com/buaa0300/QSKitDemo/tree/master/QSUseCollectionDemo))

3. 线程安全的 NSMutableArray 使用

	```
	- (void)testQsMutableArray
	{
	    _MSafeArray = [[QSThreadSafeMutableArray alloc] init];
	    
	    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
	    
	    for (NSInteger i = 0; i < 10; i++) {
	        dispatch_async(queue, ^{
	            NSString * str = [NSString stringWithFormat:@"数组%d", (int)i+1];
	            [_MSafeArray addObject:str];
	        });
	    }
	    
	    sleep(1);
	    
	    NSEnumerator * enu = [_MSafeArray objectEnumerator];
	    
	    for (NSObject * object in enu) {
	        NSLog(@"value: %@", object);
	    }
	}
	```

## 三、atomic 与 nonatomic 的区别

在默认情况下，由编译器生成的属性的 set、get 方法会通过锁定机制确保其原子性(atomicity)。如果属性具备 nonatomic 特质，则不需要同步锁。

尽管没有指明 atomic 的特质（如果某属性不具备 nonatomic 特质，那它就是"原子的"(atomic)），仍然可以在属性特质中写明这一点，编译器是不会报错的。

一般 iOS 程序中，所有属性都声明为 nonatomic。这样做的原因是：

1. 在 iOS 中使用同步锁的开销比较大， 会带来性能问题。

2. 一般情况下并不要求属性必须是"原子的"，因为这并不能保证线程安全。若要实现线程安全的操作，还需采用更为深层的锁的机制。

一个线程在连续多次读取某个属性值的过程中有别的线程在同时改写该值，那么即便将属性声明为 atomic 也还是会读取到不同的属性值。

因此，iOS 程序一般都会使用 nonatomic 属性。但在 Mac OS X 程序时，使用 atomic 属性通常都不会有性能瓶颈。

nonatomic 的实现：

```
- (void)setImage:(UIImage *)image
{
    if (_image != image) {
        [_image release];
        _image = [image retain];
        ...
    }
}

- (UIImage *)image
{
    return _image;
}
```

atomic 的实现：

```
- (void)setImage:(UIImage *)image
{
    @synchronized(self) {
        // 锁
        if (_image != image) {
            [_image release];
            _image = [image retain];
            ...
        }
    }
}

- (UIImage *)image
{
    @synchronized(self) {
        return _image;
    }
}
```

@synchronized 的介绍：

>The @synchronized directive is a convenient way to create mutex locks on the fly in Objective-C code. The @synchronized directive does what any other mutex lock would do—it prevents different threads from acquiring the same lock at the same time. In this case, however, you do not have to create the mutex or lock object directly. Instead, you simply use any Objective-C object as a lock token, as shown in the following example:
>
> \- (void)myMethod:(id)anObj
> {
>
> @synchronized(anObj) {
>
>// Everything between the braces is protected by the @synchronized directive.
>
> }
>
> }
>
>The object passed to the @synchronized directive is a unique identifier used to distinguish the protected block. If you execute the preceding method in two different threads, passing a different object for the anObj parameter on each thread, each would take its lock and continue processing without being blocked by the other. If you pass the same object in both cases, however, one of the threads would acquire the lock first and the other would block until the first thread completed the critical section.
>
> As a precautionary measure, the @synchronized block implicitly adds an exception handler to the protected code. This handler automatically releases the mutex in the event that an exception is thrown. This means that in order to use the @synchronized directive, you must also enable Objective-C exception handling in your code. If you do not want the additional overhead caused by the implicit exception handler, you should consider using the lock classes.
>
>For more information about the @synchronized directive, see The Objective-C Programming Language.

更准确的说应该是读写安全，但并不是线程安全的，因为别的线程还能进行读写之外的其他操作。线程安全需要开发者自己来保证。

## 四、文章

[清雨未尽时](https://blog.csdn.net/kangguang) & [NSMutableArray使用中忽视的问题](http://blog.csdn.net/kangguang/article/details/79194563)
