---
title: iOS 单例
categories: iOS原理
---


## 一、单例介绍

单例：该类在程序运行期间有且仅有一个实例。

#### 1.1 单例模式的要点

1. 该类有且只有一个实例；
2. 该类必须能够自行创建这个实例；
3. 该类必须能够自行向整个系统提供这个实例。

#### 1.2 单例的主要优点

1. 单例可以保证系统中该类有且仅有一个实例，确保所有对象都访问这个唯一实例；
2. 因为类控制了实例化过程，所以类可以灵活更改实例化过程；
3. 基于第 1 条，对于项目中的个别场景的传值、存储状态等业务更加方便；
4. 可以节约系统资源，对于一些需要频繁创建和销毁的对象单例模式无疑可以提高系统的性能。


#### 1.3 单例的主要缺点

1. 由于单利模式中没有抽象层，因此单例类的扩展有很大的困难。单例不能被继承，不能有子类；
2. 不易被重写或扩展（可以使用分类）
3. 单例实例一旦创建，对象指针是保存在静态区，那么在堆区分配的空间只有在应用程序终止后才会被释放；
4. 单例类的职责过重，在一定程度上违背了“单一职责原则”。

#### 1.4 单例的生命周期

下面的表格展示了程序中中不同的变量在手机存储器中的存储位置；

|位置|存放的变量|
|:--------|:-------|
|栈|临时变量（由编译器管理自动创建/分配/释放的，栈中的内存被调用时处于存储空间中，调用完毕后由系统系统自动释放内存）|
|堆|通过 alloc、calloc、malloc 或 new 申请内存，由开发者手动在调用之后通过 free 或 delete 释放内存。动态内存的生存期可以由我们决定，如果我们不释放内存，程序将在最后才释放掉动态内存，在ARC模式下，由系统自动管理。|
|全局区域|静态变量（编译时分配，APP 结束时由系统释放）|
|常量|常量（编译时分配，APP结束时由系统释放）|
|代码区|存放代码|

在程序中，一个单例类在程序中只能初始化一次，为了保证在使用中始终都是存在的，所以<font color=#cc0000>单例是在存储器的全局区域</font>，在编译时分配内存，只要程序还在运行就会一直占用内存，在 APP 结束后由系统释放这部分内存内存。

> 单例的静态变量被置为 nil，是否内存会得到释放？

```
static Singletion * singleton;

- (void)dealloc
{
    NSLog(@"%s", __func__);
}

Singleton * s = [Singleton sharedSingleton];
s = nil;
singleton = nil;
```

将单例类实例对象赋值 nil 后，会触发单例的 dealloc 方法。

静态变量修饰的指针保存在了全局区域，不会被释放。但是指针保存的首地址关联的对象是保存在堆区的，是会被释放的。


## 二、单例的实现

单例的实现重点就是防止在外部调用的时候出现多个不同的实例，也就是说要从创建的方式入手禁止出现多个不同的实例。

主要是做到以下几点：

1. 防止调用 [[A alloc] init] 引起的错误  
2. 防止调用 new 引起的错误  
3. 防止调用 copy 引起的错误  
4. 防止调用 mutableCopy 引起的错误

#### 2.1 实现方式一

> 把所有可能出现的初始化方法做了相应的处理来其保证安全性

```objc
+ (instancetype)sharedSingleton
{
    static Singleton *_sharedSingleton = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        // 不能再使用 alloc 方法
        // 因为已经重写了 allocWithZone 方法，所以这里要调用父类的分配空间的方法
        _sharedSingleton = [[super allocWithZone:NULL] init];
    });
    return _sharedSingleton;
}

// ②、防止 [[A alloc] init] 和 new 引起的错误。因为 [[A alloc] init] 和 new 实际是一样的工作原理，都是执行了下面方法
+ (instancetype)allocWithZone:(struct _NSZone *)zone
{
    return [Singleton sharedSingleton];
}

// ③、NSCopying 防止 copy 引起的错误。当你的单例类不遵循 NSCopying 协议，外部调用本身就会出错.
- (id)copyWithZone:(nullable NSZone *)zone
{
    return [Singleton sharedSingleton];
}

// ④、防止 mutableCopy 引起的错误，当你的单例类不遵循 NSMutableCopying 协议，外部调用本身就会出错.
- (id)mutableCopyWithZone:(nullable NSZone *)zone 
{
    return [Singleton sharedSingleton];
}
```

dispatch_once 主要是根据 <font color=#cc0000>`onceToken`</font> 的值来决定怎么去执行代码。

1. 当 onceToken = 0 时，线程执行 dispatch_once 的 block 中代码；
2. 当 onceToken = -1 时，线程跳过 dispatch_once 的 block 中代码不执行；
3. 当 onceToken 为其他值时，线程被阻塞，等待 onceToken 值改变。

当线程调用 shareInstance，此时 onceToken = 0，调用 block 中的代码，此时 onceToken 的值变为 140734537148864。当其他线程再调用 shareInstance 方法时，onceToken 的值已经是 140734537148864 了，线程阻塞。当 block 线程执行完 block 之后，onceToken 变为 -1，其他线程不再阻塞，跳过 block。下次再调用 shareInstance 时，block 已经为 -1，直接跳过 block。


#### 2.2 实现方式二

> 不做处理的情况下禁止外部调用

一些成熟的第三方代码的单例中也有使用该方法的。

```objc
.h 文件

- (instancetype)init NS_UNAVAILABLE;
+ (instancetype)new NS_UNAVAILABLE;
- (id)copy NS_UNAVAILABLE;
- (id)mutableCopy NS_UNAVAILABLE;

.m 文件

+ (instancetype)sharedSingleton
{
    static Singleton *_sharedSingleton = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
          _sharedSingleton = [[self alloc] init];  // 要使用 self 来调用
    });
    return _sharedSingleton;
}
```

当运行 [[A alloc] init] 或 [A new] 时，会直接报错 'init' is unavailable 或 'new' is unavailable。


## 三、单例的滥用

#### 3.1 全局状态

大多数的开发者都认同使用<font color=#cc0000>全局可变的状态</font>是不好的行为。有状态使得程序难以理解和难以调试。面向对象的程序员在最小化代码的有状态性方面，有很多还需要向函数式编程学习的地方。

```objc
@implementation SPMath
{
     NSInteger _a;
     NSInteger _b;
}  

- (NSInteger)add
{
     return _a + _b;
} 
```

在上面这个简单的数学库的实现中，程序员需要在调用 add 前正确的设置实例变量 \_a 和 \_b。这样有以下问题：

1. add 没有显式的通过使用参数的形式声明它依赖于 \_a 和 \_b 的状态。与仅仅通过查看函数声明就可以知道这个函数的输出依赖于哪些变量不同的是，另一个开发者必须查看这个函数的具体实现才能明白这个函数依赖那些变量。隐藏依赖是不好的。

2. 当修改 \_a 和 \_b 的数值为调用 add 做准备时，程序员需要保证修改不会影响任何其他依赖于这两个变量的代码的正确性。而这在多线程的环境中是尤其困难的。

把下面的代码和上面的例子做对比:

```objc
+ (NSUInteger)addOf:(NSUInteger)a plus:(NSUInteger)b
{
     return a + b;
}
```

这里，对变量 a 和 b 的依赖被显式的声明了，并且不需要为了调用这个方法而去改变实例变量的状态，也不需要担心调用这个函数会留下持久的副作用。甚至可以声明为<font color=#cc0000>类方法</font>，这样就显式的告诉了代码的阅读者：这个方法<font color=#cc0000>不会修改任何实例的状态</font>。

那么，这个例子和单例相比又有什么关系呢？用 Miško Hevery 的话来说，“[单例就是披着羊皮的全局状态](http://misko.hevery.com/2008/08/25/root-cause-of-singletons/)” 。

一个单例可以在不需要显式声明对其依赖的情况下，被使用在任何地方。就像变量 \_a 和 \_b 在 add 内部被使用了，却没有被显式声明一样，程序的任意模块都可以调用 [A sharedInstance] 并且访问这个单例。这意味着任何和这个单例交互产生的副作用都会影响程序其他地方的任意代码。

```objc
@interface Singleton : NSObject

+ (instancetype)sharedInstance;
- (NSString *)name;
- (void)setName:(NSString *)name;

@end


@implementation A 

- (void)a 
{
     if ([[Singleton sharedInstance] name]) {
          // ...
     }
}

@end

@implementation B

- (void)b
{
     [[Singleton sharedInstance] setName:""];
}

@end 
```

在上面的代码中，A 和 B 是两个完全独立的模块。但是 B 可以通过使用单例提供的<font color=#cc0000>共享状态</font>来影响 A 的行为。这种情况应该只能发生在 B 显式引用了 A，显式建立了它们两者之间的关系时。由于这里使用了单例，单例的全局性和有状态性，导致<font color=#cc0000>隐式的在两个看起来完全不相关的模块之间建立了耦合</font>。

来看一个更具体的例子，并且暴露一个使用全局可变状态的额外问题。

想要在我们的应用中构建一个网页查看器(web viewer)。我们构建了一个简单的 URL cache 来支持这个网页查看器：

```objc
@interface URLCache

+ (NSCache *)sharedURLCache;
- (void)storeCachedResponse:(NSCachedURLResponse *)cachedResponse forRequest:(NSURLRequest *)request;

@end
```

这个开发者开始写了一些单元测试来保证代码在不同的情况下都能达到预期。首先，他写了一个测试用例来保证网页查看器在没有设备链接时能够展示出错误信息。然后他写了一个测试用例来保证网页查看器能够正确的处理服务器错误。最后，他为成功情况时写了一个测试用例，来保证返回的网络内容能够被正确的显示出来。这个开发者运行了所有的测试用例，并且它们都如预期一样正确。

几个月以后，这些测试用例开始出现失败，尽管网页查看器的代码从它写完后就从来没有再改动过！到底发生了什么？

原来，有人改变了测试的顺序。处理成功的那个测试用例首先被运行，然后再运行其他两个。处理错误的那两个测试用例现在竟然成功了，和预期不一样，因为 URL cache 这个单例把不同测试用例之间的 response 缓存起来了。

<font color=#cc0000>持久化状态是单元测试的敌人</font>，因为单元测试在各个测试用例相互独立的情况下才有效。如果状态从一个测试用例传递到了另外一个，这样就和测试用例的执行顺序就有关系了。有 bug 的测试用例是非常糟糕的事情，特别是那些有时候能通过测试，有时候又不能通过测试的。


#### 3.2 对象的生命周期

另外一个关键问题就是单例的生命周期。当你在程序中添加一个单例时，很容易会认为 “它们永远只能有一个实例”。但是在很多我看到过的 iOS 代码中，这种假定都可能被打破。

假设我们正在构建一个应用，在这个应用里用户可以看到他们的好友列表。他们的每个朋友都有一张个人信息的图片，并且我们想使我们的应用能够下载并且在设备上缓存这些图片。 使用 dispatch\_once 代码片段，写一个 ThumbnailCache 单例：

```objc
@interface ThumbnailCache : NSObject

+ (instancetype)sharedThumbnailCache;
- (void)cacheProfileImage:(NSData *)imageData forUserId:(NSString *)userId;
- (NSData *)cachedProfileImageForUserId:(NSString *)userId;

@end 
```

继续构建我们的应用，一切看起来都很正常，直到有一天，决定实现“注销”功能时，这样用户可以在应用中进行账号切换。突然发现我们将要面临一个讨厌的问题：用户相关的状态存储在全局单例中。

当用户注销后，我们希望能够清理掉所有的硬盘上的持久化状态。否则，我们将会把这些被遗弃的数据残留在用户的设备上，浪费宝贵的硬盘空间。对于用户登出又登录了一个新的账号这种情况，我们也想能够对这个新用户使用一个全新的 ThumbnailCache 实例。

问题在于按照定义单例被认为是“创建一次，永久有效”的实例。你可以想到一些对于上述问题的解决方案。或许我们可以在用户登出时移除这个单例：

```objc
static ThumbnailCache * sharedThumbnailCache;

+ (instancetype)sharedThumbnailCache
{
     if (!sharedThumbnailCache) {
           sharedThumbnailCache = [[self alloc] init];
     }
     return sharedThumbnailCache;
}

+ (void)cleanUp
{
     // The SPThumbnailCache will clean up persistent states when deallocated
     sharedThumbnailCache = nil;
} 
```

这是一个明显的对单例模式的滥用，但是它可以工作，对吧。

当然可以使用这种方式去解决，但代价实在是太大了。我们不能使用简单的、能够保证线程安全和所有的调用 [ThumbnailCache sharedThumbnailCache] 的地方都会访问同一个实例的 dispatch\_once 解决方案了。现在我们需要对使用 thumbnail cache 时的代码的执行顺序非常小心。假设当用户正在执行登出操作时，有一些后台任务正在执行把图片保存到缓存中的操作：

```objc
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
     [[ThumbnailCache sharedThumbnailCache] cacheProfileImage:newImage forUserId:userId];
});
```

需要保证在所有的后台任务完成前， cleanUp 一定不能被执行。这保证了 newImage 可以被正确的清理掉。或者，我们需要保证在 thumbnail cache 被移除时，后台缓存任务一定要被取消掉。否则，一个新的 thumbnail cache 的实例将会被延迟创建，并且之前用户的数据（newImage 对象）会被存储在它里面。

由于对于单例实例来说它没有明确的所有者，(比如，单例自己管理自己的生命周期)，永远“关闭”一个单例变得非常的困难。

分析到这里，希望能够意识到，这个 thumbnail cache 从来就不应该作为一个单例。问题在于一个对象的生命周期可能在项目的最初阶段没有被很好得考虑清楚。

举一个具体的例子，Dropbox 的 iOS 客户端曾经只支持一个账号登录。它以这样的状态存在了数年，直到有一天我们希望能够同时支持[多个用户账号](https://www.dropbox.com/business/two-dropboxes)登录（既包括个人账号也包括企业账号）。突然之间，我们以前的的假设“只能够同时有一个用户处于登录状态”就不成立了。 假定一个对象的生命周期和应用的生命周期一致，会限制你的代码的灵活扩展，早晚有一天当产品的需求产生变化时，你会为当初的这个假定付出代价的。

这里我们得到的教训是：<font color=#cc0000>单例应该只用来保存全局的状态，并且不能和任何作用域绑定</font>。如果这些状态的作用域比一个完整的应用程序的生命周期要短，那么这个状态就不应该使用单例来管理。用一个单例来管理用户绑定的状态，是代码的坏味道，你应该认真的重新评估你的对象图的设计。


## 四、避免使用单例

既然单例对局部作用域的状态有这么多的坏处，那么应该怎样避免使用它们呢？

重温上面的例子。既然我们的 thumbnail cache 的缓存状态是和具体的用户绑定的，那么定义一个 user 对象吧。

```objc
@interface User : NSObject
@property (nonatomic, readonly) ThumbnailCache * thumbnailCache;
@end 

@implementation User

- (instancetype)init
{
     if ((self = [super init])) {
          _thumbnailCache = [[ThumbnailCache alloc] init];
     }
     return self;
}

@end 
```

现在用一个对象来作为一个经过认证的用户会话的模型类，并且可以把所有和用户相关的状态存储在这个对象中。

现在假设我们有一个 VC 来展现好友列表：

```objc
@interface FriendListVC : UIViewController

- (instancetype)initWithUser:(User *)user; 

@end
```

我们可以显式的把经过认证的 user 对象作为参数传递给这个 vc。这种把依赖性传递给依赖对象的技术正式的叫法是[依赖注入](http://en.wikipedia.org/wiki/Dependency_injection)，并且它有很多优点：

①、对于阅读这个 FriendListVC 头文件的人来说，可以很清楚的知道它只有在有登录用户的情况下才会被展示。

②、这个 FriendListVC 只要还在使用中，就可以强引用 user 对象。

```objc
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
     [_user.thumbnailCache cacheProfileImage:newImage forUserId:userId];
}); 
```

这种后台任务仍然意义重大，当第一个实例失效时，应用其他地方的代码可以创建和使用一个全新的 User 对象，而不会阻塞用户交互。

为了更详细的说明一下第二点，让我们画一下在使用依赖注入之前和之后的对象图。

1. 假设 FriendListVC 是当前 window 的 root view controller。使用单例时，对象图看起来如下所示：

![](https://upload-images.jianshu.io/upload_images/5294842-5a3d085872ee313f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

vc 以及自定义的 imageView，都会和 sharedThumbnailCache 产生交互。

当用户登出后，清理 rootViewController 并且退出到登录页面：

![](https://upload-images.jianshu.io/upload_images/5294842-d7ce97acfc9e7890.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里的问题在于这个 FriendListVC 可能仍然在执行代码（由于后台操作的原因），并且可能因此仍然有一些调用被挂起到 sharedThumbnailCache 上。

2. 使用依赖注入的对象图：

![](https://upload-images.jianshu.io/upload_images/5294842-42f2c8e7764a543b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

简单起见，假设 UIApplicationDelegate 管理 User 的实例（在实际中，为了[简化](http://www.objc.io/issue-1/lighter-view-controllers.html) applicationDelegate 可能会把这些用户状态的管理工作交给另外一个对象来做）。当展现 FriendListVC 时，会传递进去一个 user 的引用。这个引用也会向下传递给 profileImageView。现在，当用户登出时，我们的对象图如下所示：

![](https://upload-images.jianshu.io/upload_images/5294842-a00b35bf8a96f6c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个对象图看起来和使用单例时很像。这有什么区别？

关键问题是作用域。在单例情况下，sharedThumbnailCache 仍然可以被程序的任意模块访问。假如用户快速的登录了一个新的账号。该用户也想看看他的好友列表，这也就意味着需要再一次的和 thumbnailCache 产生交互：

![](https://upload-images.jianshu.io/upload_images/5294842-b011ee68eba70df4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当用户登录一个新账号，我们应该能够构建并且与全新的 ThumbnailCache 交互，而不需要再在销毁老的 thumbnailCache 上花费精力。基于对象管理的典型规则，旧的 vc 和老的 thumbnailCache 应该能够自己在后台延迟被清理掉。简而言之，我们应该隔离用户 A 相关联的状态和用户 B 相关联的状态：

![](https://upload-images.jianshu.io/upload_images/5294842-f0268e3703574ccb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 五、结论

在 iOS 开发的世界中，单例的使用是如此的普遍以至于我们有时候忘记了多年来在其他面向对象编程中学到的教训。

这一切的关键点在于，在面向对象编程中我们想要最小化可变状态的作用域。但是单例却站在了对立面，因为它们使可变的状态可以被程序中的任何地方访问。下一次使用单例时，希望能够好好考虑一下使用依赖注入作为替代方案。

## 六、文章

[避免滥用单例](https://blog.csdn.net/zhengang007/article/details/70336612)