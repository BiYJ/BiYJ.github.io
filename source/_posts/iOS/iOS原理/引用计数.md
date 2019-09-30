---
title: 引用计数
categories: iOS
---

## 一、简介

> OC 在创建对象时，不会直接返回该对象，而是返回一个指向对象的指针。

OC 在内存管理上采用了引用计数，它是一个<font color=#cc0000>简单而有效管理对象生命周期的方式</font>。在对象内部保存一个用来表示被引用次数的数字，init、new 和 copy 都会让计数 +1，调用 release 让计数 -1。当计数等于 0 的时候，系统调用 dealloc 方法来销毁对象。

```
A  * a = [[A alloc] init];  // retain count = 1
A  * b = a;   // 指针赋值时，retain count 不会自动增加
[b retain];   // retain count = 2
```

```
{
    OBJC_EXTERN int _objc_rootRetainCount(id);
    
    NSObject * obj = [[NSObject alloc] init];
    // 创建对象并引用，引用计数为 1
    NSLog(@"obj retainCount:%lu", (unsigned long)_objc_rootRetainCount(obj));
    
    NSObject * obj1 = [[NSObject alloc] init];
    // 创建对象并引用，引用计数为 1
    NSLog(@"obj1 retainCount:%lu", (unsigned long)_objc_rootRetainCount(obj1));
    
    // obj 指向了 obj1 所指的对象 B，失去了对原来对象A的引用,所以对象A的引用计数-1，为 0。A 被销毁
    // 对于 B，obj 引用了它，所以引用计数 +1，为 2
    obj = obj1;
    // self.obj 又引用了 A,所以引用计数 +1，为 3
    self.obj = obj;
    NSLog(@"strong obj1 retainCount:%lu",(unsigned long)_objc_rootRetainCount(obj1));
    NSLog(@"strong obj retainCount:%lu",(unsigned long)_objc_rootRetainCount(obj));
}
```

引用计数分为自动引用计数「ARC :  Automatic Reference Counting」和手动引用计数「MRC : Manual Reference Counting」。

## 二、原理

<center>
![](https://upload-images.jianshu.io/upload_images/5294842-8a6895639c25af4c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)
</center>

## 三、示例

```
NSObject * obj1 = [NSObject new];
NSLog(@"引用计数: %lu", (unsigned long)[obj1 retainCount]);
NSObject * obj2 = [obj1 retain];
NSObject * obj3 = [obj1 retain];
NSLog(@"引用计数: %lu", (unsigned long)[obj1 retainCount]);
[obj1 release];
NSLog(@"引用计数: %lu %@", (unsigned long)[obj1 retainCount], obj1);
[obj1 release];
NSLog(@"引用计数: %lu %@", (unsigned long)[obj1 retainCount], obj1);
[obj1 release];
NSLog(@"引用计数: %lu %@", (unsigned long)[obj1 retainCount], obj1);

引用计数：1
引用计数：3
引用计数：2 <NSObject：0x60400001ecd0>
引用计数：1 <NSObject：0x60400001ecd0>
*** -[NSObject retainCount]: message sent to deallocated instance 0x60400001ecd0
```

根据 Debug 输出可以看到：obj1 可以调用多次 release 方法。

从两次打印 obj1 的地址相同可以猜测，在 [obj1 release] 执行之后对象的引用计数 -1，不再强引用对象，但 obj1 仍然指向对象所在的那片内存空间。在第三次执行 release 后，对象的引用计数为 0，对象所在的内存空间被销毁，但是 obj1 指针仍然存在，此时调用 retainCount 会报野指针错误。可以通过置 obj1 = nil 解决这个问题。

对 Linux 文件系统比较了解的可能发现，引用计数的这种管理方式类似于文件系统里面的**硬链接**。在 Linux 文件系统中，我们用 ln 命令可以创建一个硬链接（相当于 retain），当删除一个文件时（相当于 release），系统调用会检查文件的 link count 值，如果大于 1，则不会回收文件所占用的磁盘区域。直到最后一次删除前，系统发现 link count 值为 1，则系统才会执行直正的删除操作，把文件所占用的磁盘区域标记成未用。

## 四、僵尸对象、野指针、空指针

僵尸对象：所占用内存已经被回收的对象，僵尸对象不能再使用。

> 野指针：指向僵尸对象（不可用内存）的指针，给野指针发送消息会报错（EXC_BAD_ACCESS）。
>
> 空指针：没有指向任何对象的指针（存储的是 nil、NULL），给空指针发送消息不会报错；空指针的一个经典使用场景就是在开发中获取服务器 API 数据时，转换野指针为空指针，避免发送消息报错。

## 五、为什么需要引用计数？

引用计数真正派上用场的场景是在面向对象的程序设计架构中，用于<font color=#cc0000>对象之间传递和共享数据</font>。

举个例子：

对象 A 生成了一个对象 O，需要调用对象 B 的某个方法，并将对象 O 作为参数传递过去。

```
[objB doSomething:O];
```

在没有引用计数的情况下，一般内存管理的原则是「谁申请谁释放」。

那么对象 A 就需要在对象 B 不再需要 O 的时候，将 O 销毁。但对象 B 可能临时用一下 O，也可能将它设置为自己的一个成员变量，在这种情况下，什么时候销毁就成了一个难题了。

<center>
![](https://upload-images.jianshu.io/upload_images/5294842-ed988afe64d00f52.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 
</center>

对于以上情况有两种做法：

1. 对象 A 在调用完对象 B 的某个方法之后，马上销毁参数 O；然后对象 B 需要将对象 O 复制一份，生成另一个对象 O2，同时自己来管理对象 O2 的生命周期。

	这种做法带来更多的内存申请、复制、释放的工作。本来可以复用的对象，因为不方便管理它的生命周期，就简单地把它销毁，又重新构造一份一样的，实在太影响性能。

2. 对象 A 只负责生成 O，之后就由对象 B 负责完成 O 的销毁工作。如果对象 B 只是临时用一下 O，就可以用完后马上销毁；如果对象 B 需要长时间使用 O，就不销毁它。

	这种做法看似解决了对象复制的问题，但是它强烈依赖于 A 和 B 两个对象的配合，代码维护者需要明确地记住这种编程约定。而且，由于 O 的生成和释放在不同对象中，使得它的内存管理代码分散在不同对象中，管理起来也很费劲。如果这个时候情况更加复杂一些，例如对象 B 需要再向对象 C 传递参数 O，那么这个对象在对象 C 中又不能让对象 C 管理。所以这种方法带来的复杂度更高，更加不可取。

<center>
![](https://upload-images.jianshu.io/upload_images/5294842-15dc0b70f8c7558f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 
</center>

引用计数的出现很好地解决这个问题，在参数 O 的传递过程中，哪些对象需要长时间使用它，就把它的引用计数 +1，使用完就-1。所有对象遵守这个规则，对象的生命周期管理就可以完全交给引用计数了。我们也可以很方便地享受到共享对象带来的好处。

## 六、ARC 下的内存管理问题

问题主要体现在：

1.  过度使用 block 之后，无法解决循环引用问题。
2.  遇到底层 Core Foundation 对象，需要手工管理它们的引用计数时，显得一筹莫展。

#### 6.1 循环引用

引用计数这种管理内存的方式虽然很简单，但是有一个比较大的瑕疵，即它不能很好的解决循环引用问题。如下图所示：对象 A和对象 B，相互引用了对方作为自己的成员变量，只有当自己销毁时，才会将成员变量的引用计数减 1。因为对象 A 的销毁依赖于对象 B 销毁，而对象 B 的销毁又依赖于对象 A 的销毁，这样就造成了循环引用 Reference Cycle 的问题，这两个对象即使在外界已经没有任何指针能够访问到它们了，它们也无法被释放。

<center>
![](https://upload-images.jianshu.io/upload_images/5294842-9bf50b90b4ebb81c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

不止两对象存在循环引用问题，多个对象依次持有对方，形式一个环状，也可以造成循环引用问题，而且在真实编程环境中，<font color=#cc0000>环越大就越难被发现</font>。下图是 4 个对象形成的循环引用问题。

<center>
![](https://upload-images.jianshu.io/upload_images/5294842-c679f014ddb0842b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

#### 6.2 主动断开循环引用

解决循环引用问题主要有两个办法。第一个办法：明确知道这里会存在循环引用，在合理的位置主动断开环中的一个引用，使得对象得以回收。如下图所示：

<center>
![](https://upload-images.jianshu.io/upload_images/5294842-c6f3631df5d44320.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

主动断开循环引用这种方式常见于各种与 block 相关的代码逻辑中。

不过，主动断开循环引用这种操作依赖于程序员自己手工显式地控制，相当于回到了以前 “谁申请谁释放” 的内存管理年代，它依赖于程序员自己有能力发现循环引用并且知道在什么时机断开循环引用回收内存，所以这种解决方法并不常用，更常见的办法是使用弱引用的办法。

#### 6.3 使用弱引用

弱引用虽然持有对象，但是并不增加引用计数，这样就避免了循环引用的产生。在 iOS 开发中，弱引用通常在 delegate 模式中使用。如下所示：

<center>
![](https://upload-images.jianshu.io/upload_images/5294842-d7832da14f2870e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

#### 6.4 弱引用的实现原理

弱引用的实现原理是这样，系统对于每一个有弱引用的对象，都维护一个表来记录它所有的弱引用的指针地址。这样，当一个对象的引用计数为 0 时，系统就通过这张表，找到所有的弱引用指针，继而把它们都置成 nil。

从这个原理中，我们可以看出，弱引用的使用是有额外的开销的。虽然这个开销很小，但是如果一个地方我们肯定它不需要弱引用的特性，就不应该盲目使用弱引用。举个例子，有人喜欢在手写界面的时候，将所有界面元素都设置成 weak 的，这某种程度上与Xcode 通过 Storyboard 拖拽生成的新变量是一致的。但是我个人认为这样做并不太合适。因为：

1. 在创建这个对象时，需要注意临时使用一个强引用持有它，否则因为 weak 变量并不持有对象，就会造成一个对象刚被创建就销毁掉。

2. 大部分 ViewController 的视图对象的生命周期与 ViewController 本身是一致的，没有必要额外做这个事情。

3. 早先苹果这么设计，是有历史原因的。在早年，当时系统收到 Memory Warning 的时候，ViewController 的 View 会被 unLoad 掉。这个时候，使用 weak 的视图变量是有用的，可以保持这些内存被回收。但是这个设计已经被废弃了，替代方案是将相关视图的 CALayer 对应的 CABackingStore 类型的内存区会被标记成 volatile 类型，详见[《再见，viewDidUnload方法》](http://blog.devtang.com/2013/05/18/goodbye-viewdidunload/)。

#### 6.5 检测循环引用

<center>
![](https://upload-images.jianshu.io/upload_images/5294842-6d266d7ad597be42.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/100)
</center>

## 七、学习文章

[iOS 内存管理](https://www.cnblogs.com/huangjianwu/p/4962772.html)
[iOS 的内存管理](https://cloud.tencent.com/developer/article/1127761)
