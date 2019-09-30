---
title: ARC
categories: iOS原理
---

## 一、ARC

ARC 的想法来源于苹果在早期设计 Xcode 的 Analyzer 的时候，发现编译器在编译时可以帮助大家发现很多内存管理中的问题。后来苹果修改了一些内存管理代码的书写方式，干脆编译器在编译时把内存管理的代码都自动补上。

<font color=#cc0000>ARC 是编译器特性，而不是运行时特性，更不是垃圾回收器（GC）</font>。

> Automatic Reference Counting (ARC) is a compiler-level feature that simplifies the process of managing object lifetimes (memory management) in Cocoa applications.

程序在编译的时候，编译器会分析源码中每个对象的生命周期，然后基于这些对象的生命周期，编译器帮我们在合适的地方插入retain、release 等代码以管理对象的引用计数，从而达到自动管理对象生命周期的目的。

所以 ARC 是工作在编译期的一种技术方案，这样的好处：

1. 编译之后，ARC 与 MRC 代码是没有什么差别的，所以二者可以<font color=#cc0000>在源码中共存</font>。

2. 相对于垃圾回收这类内存管理方案，<font color=#cc0000>ARC 不会带来运行时的额外开销</font>，所以对于应用的运行效率不会有影响。相反的，由于ARC 能够深度分析每一个对象的生命周期，它能够做到比人工管理引用计数更加高效。例如在一个函数中，对一个对象刚开始有一个引用计数 +1 的操作，之后又紧接着有一个 -1 的操作，那么编译器就可以把这两个操作都优化掉。

只有编译器是无法单独完成这一工作的，还需要 OC 运行时库的配合协助，因此 ARC 的实现工具主要包括：

1.  LLVM 编译器（clang 3.0 以上）
2.  OC 运行时库 493.9 以上

weak 变量能够在引用计数为 0 时被自动设置成 nil，显然是有运行时逻辑在工作的。

ARC 能够解决 iOS 开发中 90% 的内存管理问题，但是另外 10% 的内存管理问题是需要开发者处理的，这主要是与底层 Core Foundation 对象交互的部分，底层 Core Foundation 对象由于不在 ARC 的管理下，所以需要自己维护这些对象的引用计数。

## 二、ARC 的开启和关闭

在 Targets -》Build Settings 中搜索 Automatic Reference Counting，可以修改它的布尔值，yes - 开启　no - 关闭。

<center>
![arc](https://upload-images.jianshu.io/upload_images/5294842-186571a065bdb139.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 
</center>

如果需要对特定文件开启或关闭 ARC，可以在 Targets -》Build Phases -》Compile Sources，在里面找到对应文件，添加flag：

开启：-fobjc-arc  关闭：-fno-objc-arc

<center>
![arc](https://upload-images.jianshu.io/upload_images/5294842-de31d9803c80902b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 
</center>

## 三、ARC 的修饰符

主要提供了 4 种修饰符，他们分别是：\_\_strong、\_\_weak、\_\_autoreleasing、\_\_unsafe\_unretained。

#### 3.1 __strong

强引用。相当于 @property 的 "strong"。所有对象只有当没有任何一个强引用指向（引用计数为 0）时，才会被释放。

> 注意：如果在声明引用时不加修饰符，那么将默认是强引用。当需要释放强引用指向的对象时，需要将强引用置 nil。

使用 \_\_strong 修饰变量的程序运行过程。

```
{
    id __strong object = [[NSObject alloc] init];
}
```

转换后的模拟源代码为：

```
/*编译器的模拟代码*/
id object = objc_msgSend(NSObjct, @selector(alloc));
objc_msgSend(object, @selector(init));
objc_release(object);
```

对象变量生成时，分别调用 alloc 和 init 方法，对象变量作用域结束时调用 objc\_release 方法释放对象变量，虽然 ARC 情况下不能使用 release 方法，但是由此可见编译器编译时在合适的地方插入了 release。

在使用 alloc、new、copy、mutableCopy 以外的方法生成对象变量方法时会有什么不同

```
{
    id __strong object = [NSMutableArray array];
}
```

调用 array 的类方法转换后：

```
{
    /*编译器的模拟代码*/
    id object = objc_msgSend(NSMutableArray, @selector(array));
    objc_retainAutoreleasedReturnValue(object);
    objc_release(object);
}
```

objc\_retainAutoreleasedReturnValue(object) 函数的作用：最优化程序运行。

自己持有（retain）对象的函数，但它持有的应为返回注册在 autoreleasepool 中对象的方法或函数的返回值。

objc\_retainAutoreleasedReturnValue 函数与 objc\_autoreleasedReturnValue 是成对出现的，现在看看 NSMutableArray 类的 array 类方法的编译器实现。

```
+ (id)array {
    return [[NSMutableArray alloc] init];
}
```

转换后的源代码。

```
+ (id)array
{
    /*编译器的模拟代码*/
    id obj = objc_msgSend(NSMutableArray, @selector(alloc));
    objc_msgSend(obj, @selector(init));
    return objc_autoreleaseReturnValue(obj);
}
```

通过 objc\_autoreleaseReturnValue 函数将对象注册在自动释放池 autoreleasepool 中并返回，但是与 objc\_autorelease 函数不同的是，objc\_autoreleaseReturnValue 函数一般不仅限于注册对象到 autoreleasepool 中去。

objc\_autoreleaseReturnValue 与 objc\_retainAutoreleasedReturnValue 的配合使用，可以不将对象注册到autoreleasepool 中而直接传递，达到最优化。

objc\_autoreleaseReturnValue 函数会检查使用该函数的方法或者函数的调用方的执行命令列表，如果调用方在调用该函数或方法之后，紧接着调用了 objc\_retainAutoreleasedReturnValue 函数，那么不再将对象注册到 autoreleasepool 中去，而直接将对象传递给调用方。

相比于 objc\_retain 函数来说 objc\_retainAutoreleasedReturnValue 函数在返回一个即使没有注册到 autoreleasepool 中的对象，也能正确的获取对象。

#### 3.2 \_\_weak

弱引用。相当于 @property 的 "weak"。弱引用不会影响对象的引用计数，即只要对象没有任何强引用指向，即使有 n 个弱引用对象指向也没用，该对象依然会被释放。

对象在被释放的同时，指向它的弱引用（weak）会自动被置 nil，这个技术叫 <font color=#cc0000>zeroing weak pointer</font>。这样有效的防止无效指针、野指针的产生。\_\_weak 一般用在 delegate 关系中防止循环引用或者用来修饰指向由 Interface Builder 编辑与生成的 UI 控件。

```
{
    id _weak object = [[NSObject alloc] init];
}
```

转换后的模拟源代码。

```
{
    /* 编译器的模拟代码 */
    id object;
    id tmp = objc_msgSend(NSObject, @selector(alloc));
    objc_msgSend(tmp, @selector(init));
    objc_initWeak(&object, tmp);
    objc_release(tmp);
    objc_destoryWeak(&object);
}
```

自己生成并且持有的对象通过 objc\_initWeak 函数赋值给 \_\_weak 修饰符的变量，但是编译器判断并没有对其进行持有，因此该对象通过 objc\_release 函数被释放和废弃。

随后通过 objc\_destoryWeak 将引用废弃对象的附有 \_\_weak 修饰符的变量置为 nil。

如果不是直接赋值，而是通过使用 \_\_weak 修饰符来引用变量时。

```
{
    id __weak object = obj;
    NSLog(@"%@", object);
}
```

转换后的模拟源代码。

```
/*编译器的模拟代码*/
{
    id object;
    objc_initWeak(&object, obj);
    id temp = objc_loadWeakRetained(&object);
    objc_autorelease(temp);
    NSLog(@"%@", temp);
    objc_destoryWeak(&object);
}
```

明显增加了 objc\_loadWeakRetained 与 objc\_autorelease 函数调用，他们的主要作用是：

1.  objc\_loadWeakRetained 函数取出 \_\_weak 修饰符变量引用的对象并且 retain
2.  objc\_autorelease 函数将引用的对象注册到 autoreleasepool 中。

因此，使用 \_\_weak 修饰符引用的对象都被注册到 autoreleasepool 中，在 @autoreleasepool 块结束之前都可以放心使用，大量使用 \_\_weak 修饰符的变量，导致注册到 autoreleasepool 中的对象也大量地增加。所以在使用 \_\_weak 修饰符引用的变量时，最好先暂时用 \_\_strong 修饰符的变量进行引用后再使用。

2 种不能使用 \_\_weak 修饰符的情况：

*   重写了 retain/release 的类，例如 NSMachPort 类；
*   当 allowsWeakReference/retainWeakReference 实例方法返回 NO 时。

#### 3.3 \_\_autoreleasing

对象被加入到 autorelease pool，是会自动释放的引用，与 MRC 中 autorelease 的用法相同。定义 @property 时不能使用这个修饰符。

对于 alloc、new、copy、mutableCopy 的实现。

```
@autoreleasepool{
    id __autoreleasing object = [[NSObject alloc] init];
}
```

转换后的模拟源代码。

```
{
    /* 编译器的模拟代码 */
    id pool = objc_autoreleasePoolPush();
    id object = objc_msgSend(NSObjct, @selector(alloc));
    objc_msgSend(object, @selector(init));
    // 调用autorelease方法
    objc_autorelease(object);
    id pool = objc_autoreleasePoolPop();
}
```

NSMutableArray 类中的 array 方法如何实现 autorelease 功能。

```
@autoreleasepool{
    id __autoreleasing object = [NSMutableArray array];
}
```

转化后的模拟源代码。

```
{
    /* 编译器的模拟代码 */
    id pool = objc_autoreleasePoolPush();
    id object = objc_msgSend(NSMutableArray, @selector(array));
    objc_retainAutoreleasedReturnValue(object);
    // 调用 autorelease 方法
    objc_autorelease(object);
    id pool = objc_autoreleasePoolPop();
}
```

除了持有对象的方法从 alloc 变成了 objc\_retainAutoreleasedReturnValue 函数，但是注册到 autoreleasepool 的方法没有变化，都是调用了 objc\_autorelease 函数。

一个常见的误解是，在 ARC 中没有 autorelease，因为这样一个“自动释放”看起来好像有点多余。

这个误解可能源自于将 ARC 的“自动” 和 autorelease “自动” 的混淆。其实你只要看一下每个 iOS App 的 main.m 文件就能知道，autorelease 不仅好好的存在着，并且不需要再手工被创建，也不需要再显式得调用 [pool drain] 方法释放内存池。

```
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

以下两行代码的意义是相同的。

```
NSString * str = [[[NSString alloc] initWithFormat:@"China"] autorelease];   // MRC
NSString * __autoreleasing str = [[NSString alloc] initWithFormat:@"China"]; // ARC
```

\_\_autoreleasing 在 ARC 中主要用在参数传递返回值（out-parameters）和引用传递参数（pass-by-reference）的情况下。

> __autoreleasing is used to denote arguments that are passed by reference (id *) and are autoreleased on return.

比如常见的 NSError 的使用：

```
NSError * __autoreleasing error;
 
// writeToFile方法中 error 参数的类型为 (NSError *__autoreleasing *)）
￼if (![data writeToFile:filename options:NSDataWritingAtomic error:&error])  { 
　　NSLog(@"Error: %@", error.localizedDescription); 
}
```

注意：如果 error 的修饰符为 strong，那么，编译器会帮你隐式地做如下事情，保证最终传入函数的参数依然是个 \_\_autoreleasing 类型的引用。

```
NSError * error; 
NSError * __autoreleasing tempError = error;  // 编译器添加
 
if (![data writeToFile:filename options:NSDataWritingAtomic error:&tempError]) 
￼{ 
　　error = tempError; // 编译器添加 
　　NSLog(@"Error: %@", error.localizedDescription); 
}
```

为了避免这种情况，提高效率，一般在定义 error 的时候将其声明为\_\_autoreleasing 类型的：

```
NSError *__autoreleasing error;
```

加上 \_\_autoreleasing 之后，相当于在 MRC 中对返回值 error 做了如下事情：

```
*error = [[[NSError alloc] init] autorelease];
```

\*error 指向的对象在创建出来后，被放入到了 autoreleasing pool 中，等待使用结束后的自动释放，函数外 error 的使用者并不需要关心 \*error 指向对象的释放。

另外，在 ARC 中，所有这种指针的指针（NSError \*\*）的函数参数如果不加修饰符，编译器会默认将他们认定为 \_\_autoreleasing 类型。

比如下面的两段代码是等同的：

```
- (NSString *)doSomething:(NSNumber **)value
{
     // do something  
}
- (NSString *)doSomething:(NSNumber * __autoreleasing *)value
{ 
     // do something 
}
```

除非显式得给 value 声明了 \_\_strong，否则 value 默认就是 \_\_autoreleasing 的。

最后一点，某些类的方法会隐式地使用自己的 autorelease pool，在这种时候使用 \_\_autoreleasing 类型要特别小心。

比如 NSDictionary 的 - enumerateKeysAndObjectsUsingBlock: 方法会隐式地创建一个 autorelease pool.

```
- (void)loopThroughDictionary:(NSDictionary *)dict error:(NSError **)error
{
    [dict enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop){
          // do stuff  
          if (...)  {
                *error = [NSError errorWithDomain:@"Not Found" ￼code:404 userInfo:nil];
          }
    }];
￼}
```

上面代码实际类似于：

```
- (void)loopThroughDictionary:(NSDictionary *)dict error:(NSError **)error
{
    [dict enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop){
          @autoreleasepool  // 被隐式创建
　　　　　　{
              if (...) {
                    *error = [NSError errorWithDomain:@"Not Found" ￼code:404 userInfo:nil];
              }
￼          }
    }];
    // *error 在这里已经被dict的做枚举遍历时创建的 autorelease pool 释放掉了 
￼} 
```

为了能够正常的使用 \*error，我们需要一个 strong 型的临时引用，在 dict 的枚举 block 中使用这个临时引用，保证引用指向的对象不会在出了 dict 的枚举 block 后被释放，正确的方式如下：

```
- (void)loopThroughDictionary:(NSDictionary *)dict error:(NSError **)error
{
　　__block NSError * tempError;  // 加 __block 保证可以在 Block 内被修改  
　　
   [dict enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) { 
　　　　
      if (...)  { 
　　　　　　*tempError = [NSError errorWithDomain:@"Not Found" ￼code:404 userInfo:nil]; 
　　　　} ￼ 
　　}] 
　　if (error != nil) 
　　{ 
　　　　*error = tempError; 
　　} ￼
} 
```

#### 3.4 \_\_unsafe_unretained

ARC 是在 iOS 5 引入的，而这个修饰符主要是为了在 ARC 刚发布时兼容 iOS 4 以及版本更低的设备，因为这些低版本的设备没有 weak pointer system，这个系统简单的理解就是上面讲 weak 时提到的，能够在 weak 引用指向对象被释放后，把引用值自动设为 nil。

相当于 @property 的 "unsafe_unretained"，实际可以将它理解为 MRC 时代的 assign：纯粹只是将引用指向对象，没有任何额外的操作，在指向对象被释放时依然指向原来被释放的对象（所在的内存区域）。所以非常不安全。

现在可以完全忽略掉这个修饰符了，因为 iOS 4 早已退出历史舞台很多年。

```
{
    id __unsafe_unretained object = [[NSObject alloc] init];
}
```

转换后的模拟源代码。

```
{
     /*编译器的模拟代码*/
    id object = objc_msgSend(NSObject, @selector(alloc));
    objc_msgSend(object, @selector(init));
    objc_release(tmp);
}
```

可见通过 \_\_unsafe\_unretained 修饰的变量引用了对象但是并不持有对象，对象在释放和废弃后，并没有调用被 \_\_unsafe\_unretained 修饰的变量的 objc\_destoryWeak 函数，因此该对象的悬垂指针被赋值给变量 object，导致引用变量 object 时发生崩溃。

#### 3.5 正确使用修饰符

苹果的文档中明确地写道：

> You should decorate variables correctly. When using qualifiers in an object variable declaration,
> the correct format is:
>   ClassName * qualifier variableName;

按照这个说明，要定义一个 weak 修饰的 NSString 引用，它的写法应该是：

```
NSString * __weak str = @"Hello";   // 正确
```

而不应该是：

```
__weak NSString *str = @"Hello";   // 错误
```

那这里就有疑问了，既然文档说是错误的，为啥编译器不报错呢？文档又解释道：

> Other variants are technically incorrect but are "forgiven" by the compiler. To understand the issue, see [http://cdecl.org/](http://cdecl.org/).

看来是苹果考虑到很多人会用错，所以在编译器这边贴心地帮我们忽略并处理掉了这个错误。虽然不报错，但是我们还是应该按照正确的方式去使用这些修饰符。

#### 3.6 栈中指针默认值为 nil

无论是被 strong、weak 还是 autoreleasing 修饰，声明在栈中的指针默认值都会是 nil。所有这类型的指针不用再初始化的时候置 nil 了。这个特性更加降低了“野指针”出现的可能性。

在 ARC 中，以下代码会输出 null 而不是 crash。

```
- (void)myMethod 
{
    NSString * name;
    NSLog(@"%@", name);
}
```

## 四、ARC 与 Block

在手动管理内存时代，block 会隐式地对进入其作用域内的对象（或者说被 block 捕获的指针指向的对象）执行 retain 操作，来确保 block 使用到该对象时，能够正确的访问。

```
MyViewController * myController = [[MyViewController alloc] init…]; 

myController.dismissBlock =  ^(NSString * result) {
    // 隐式地调用 [myController retain]; 造成循环引用
    [myController dismissViewControllerAnimated:YES completion:nil];
};

[self presentViewController:myController animated:YES completion:^{
    // 调用[myController release];是在 MRC 中的一个常规写法，并不能解决上面循环引用的问题
    [myController release]; 
}];

@interface SecondVC : UIViewController
@property (nonatomic, copy) void (^ block)(void);
@end
{
    SecondVC * vc = [[SecondVC alloc] init];
    
    NSLog(@"%lu", (unsigned long)vc.retainCount);
    
    vc.block = ^ {
        NSLog(@"%lu", (unsigned long)vc.retainCount);
    };
    vc.block();
}
2018-11-16 10:26:05.872092+0800 Demo[49289:1083433] 1
2018-11-16 10:26:05.872214+0800 Demo[49289:1083433] 2
```

dismissBlock 调用了 [myController dismiss..] 方法，这时 dismissBlock 会对 myController 执行 retain 操作。

而作为 myController 的属性，myController 对 dismissBlock 也至少有一个 retain（一般准确讲是 copy），这时就出现了在内存管理中最糟糕的情况：循环引用。也就是说：相互持有对方。循环引用导致了 myController 和 dismissBlock 最终都不能被释放。

对 delegate 指针用 weak 就是为了避免这种问题。

不过好在，编译器会及时地给我们一个警告，提醒我们可能会发生这类型的问题：

![arc](https://upload-images.jianshu.io/upload_images/5294842-56f8ed9fe539a49f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 

我们一般用如下方法解决：给进入 block 的指针加一个 \_\_block 修饰符。

这个 \_\_block 在 MRC 时代有两个作用：

*   说明变量可改
*   说明指针指向的对象不做隐式的 retain 操作

除了静态变量和全局变量不需要加 \_\_block 就可以在 block 中修改外，其他变量不加则不能在 block 中修改。

对代码做出修改，解决了循环引用的问题：

```
MyViewController * __block myController = [[MyViewController alloc] init…]; 
myController.dismissBlock =  ^(NSString * result) {
    [myController dismissViewControllerAnimated:YES completion:nil];
}; 
// 之后正常的 release 或者 retain
```

在 ARC 环境下，没有了 retain 和 release 等操作，情况也发生了改变：

> 在任何情况下，\_\_block 修饰符的作用只有上面的第一条：说明变量可改。即使加上了 \_\_block 修饰符，一个被 block 捕获的强引用也依然是一个强引用。

所以在 ARC 下，如果还按照 MRC 下的写法，添加 \_\_block 是没有解决循环引用的问题。

代码修改如下：

```
__block MyViewController * myController = [[MyViewController alloc] init…]; 
myController.dismissBlock =  ^(NSString * result) {
    [myController dismissViewControllerAnimated:YES completion:nil];
    myController = nil;  // 注意这里，保证了 block 结束对 myController 强引用的解除
};
```

在 block 中将捕获的指针置为 nil，保证了 dismissBlock 对 myController 强引用的解除，不过也同时解除了myController 指针对 myController 对象的强引用。

更好的方法就是使用 weak。（或者为了考虑 iOS4 的兼容性用 unsafe\_unretained，具体用法和 weak 相同）

为了保证 dismissBlock 对 myController 没有强引用，我们可以定义一个临时的弱引用 weakMyViewController 来指向原myController 的对象，并把这个弱引用传入到 dismissBlock 内，这样就保证了 dismissBlock 对 myController 持有的是一个弱引用，而不是一个强引用。如此，继续修改代码如下：

```
MyViewController * __weak weakMyViewController = myController;
myController.dismissBlock =  ^(NSString * result) {
    [weakMyViewController dismissViewControllerAnimated:YES completion:nil];
};
```

这样循环引用的问题就解决了，但是却引入了一个新的问题：由于传入 dismissBlock 的是一个弱引用，那么当 myController指向的对象在 dismissBlock 被调用前释放，那么 dismissBlock 就不能正常的运作了。在一般的单线程环境中，这种问题出现的可能性不大，但是到了多线程环境，就很不好说了，所以我们需要继续完善这个方法。

为了保证在 dismissBlock 内能够访问到正确的 myController，我们在 dismissBlock 内新定义一个强引用strongMyController 来指向 weakMyController 指向的对象，这样多了一个强引用，就能保证这个 myController 对象不会在 dismissBlock 被调用前释放掉了。于是，对代码再次做出修改：

```
MyViewController * __weak weakMyController = myController;
// __weak typeof(myController) weakMyController = myController;
myController.dismissBlock =  ^(NSString * result) {
    MyViewController * strongMyController = weakMyController;
    // __strong typeof(weakMyController) strongMyController = weakMyController;
    if (strongMyController) {
         [strongMyController dismissViewControllerAnimated:YES completion:nil];
    }
    else {
    }
};
```

很多读者会有疑问，不是不希望 block 对原 myController 对象增加强引用么，这里为什么堂而皇之地在 block 内新定义了一个强引用，这个强引用不会造成循环引用么？

理解这个问题的关键在于被 block 捕获的引用和在 block 内定义的引用的区别。为了搞得明白这个问题，这里需要了解一些Block 的实现原理，详细的内容可以参考其他的文章：[谈Objective-C block的实现](http://blog.devtang.com/2013/07/28/a-look-inside-blocks/ "谈Objective-C block的实现")、[block 实现](http://blog.csdn.net/hherima/article/details/38586101)、[正确使用Block避免Cycle Retain和Crash](http://tanqisen.github.io/blog/2013/04/19/gcd-block-cycle-retain/)。

为了更清楚地说明问题，这里用一个简单的程序举例。如下程序：

```
#include <stdio.h>
int main()
{
    int b = 10;
    int *a = &b;
    
    void (^ block)() = ^() {
         int *c = a;
    };
    block(); 
    return 1;
}
```

程序中，同为 int 型的指针，a 变量被 block 捕获，而 c 变量是在 block 内定义的。用 clang -rewrite-objc 命令处理后，可以看到如下代码。

原 main 函数：

```
int main()
{
    int b = 10;
    int *a = &b;
   
    void (*block)() = (void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, a);
    ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
    
    return 1;
}
```

block 的结构：

```
struct __main_block_impl_0 { 
    struct __block_impl impl;
    struct __main_block_desc_0* Desc; 
    
    int *a;  // 被捕获的引用 a 出现在了 block 的结构体里面
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_a, int flags=0) : a(_a) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
```

实际执行的函数：

```
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
     int *a = __cself->a; // bound by copy
     int *c = a; // 在 block 中声明的引用 c 在函数中声明，存在于函数栈上
}
```

可以清楚的看到，a 和 c 存在的位置完全不同，如果 block 存在于堆上（在 ARC 下 block 默认在堆上），那么 a 作为 block 结构体的一个成员，也自然会存在于堆上，而 c 无论如何，永远位于 block 内实际执行代码的函数栈内。这也导致了两个变量生命周期的完全不同：c 在 block 的函数运行完毕，即会被释放，而 a 只有在 block 被从堆上释放的时候才会释放。

回到之前的示例，如果直接让 dismissBlock 捕获 myController 引用，那么这个引用会被复制后作为 dismissBlock 的成员变量存在于其所在的堆空间中，也就是为 dismissBlock 增加了一个指向 myController 对象的强引用，这就是造成循环引用的本质原因。

对于 MyViewController 的例子，dismissBlock 的结构体大概是这个样子：

```
struct __main_block_impl_0 {
    
    struct __block_impl impl; 
    struct __main_block_desc_0* Desc;
    MyViewController * __strong myController;  // 被捕获的强引用 myController
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_a, int flags=0) : a(_a) {
         impl.isa = &_NSConcreteStackBlock;
         impl.Flags = flags;
         impl.FuncPtr = fp;
         Desc = desc;
    }
};
```

而给 dismissBlock 传入一个弱引用 weakMyController，这时 dismissBlock 的结构：

```
struct __main_block_impl_0 {
    struct __block_impl impl; 
    struct __main_block_desc_0* Desc;
    MyViewController * __weak weakMyController;  // 被捕获的弱引用 weakMyController
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_a, int flags=0) : a(_a) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
```

在 dismissBlock 内声明的强引用 strongMyController，它虽然是强引用，但存在于函数栈中，在函数执行期间，它一直存在，一直持有 myController 对象，但当函数执行完毕，strongMyController 即被销毁，于是它对 myController 对象的强引用被解除，这时 dismissBlock 对 myController 对象就不存在强引用关系了！

加入了 strongMyController 的函数大体会是这个样子：

```
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
     MyViewController * __strong strongMyController = __cself->weakMyController;
}
```

在 ARC 中，block 捕获的引用和 block 内声明的引用，存储空间与生命周期都是不同的。

实际上，在自动引用计数环境下，对 block 捕获对象的内存管理已经简化了很多，由于没有了 retain 和 release 等操作，实际只需要考虑循环引用的问题就行了。

## 五、ARC 与 Toll-Free Bridging

> There are a number of data types in the Core Foundation framework and the Foundation framework that can be used interchangeably. This capability, called *toll-free bridging*, means that you can use the same data type as the parameter to a Core Foundation function call or as the receiver of an Objective-C message.

Toll-Free Briding 保证了在程序中，可以方便和谐的使用 Core Foundation 类型的对象和 Objective-C 类型的对象。详细的内容可参考[官方文档](https://developer.apple.com/library/ios/documentation/General/Conceptual/CocoaEncyclopedia/Toll-FreeBridgin/Toll-FreeBridgin.html)。以下是官方文档中给出的示例：

```
NSLocale * gbNSLocale = [[NSLocale alloc] initWithLocaleIdentifier:@"en_GB"];
CFLocaleRef gbCFLocale = (CFLocaleRef) gbNSLocale;
CFStringRef cfIdentifier = CFLocaleGetIdentifier (gbCFLocale);
NSLog(@"cfIdentifier: %@", (NSString *)cfIdentifier); // logs: "cfIdentifier: en_GB"
CFRelease((CFLocaleRef) gbNSLocale);
CFLocaleRef myCFLocale = CFLocaleCopyCurrent();
NSLocale * myNSLocale = (NSLocale *) myCFLocale;
[myNSLocale autorelease];
NSString * nsIdentifier = [myNSLocale localeIdentifier];
CFShow((CFStringRef) [@"nsIdentifier: " stringByAppendingString:nsIdentifier]); // logs identifier for current locale
```

在 MRC 时代，由于 Objective-C 类型的对象和 Core Foundation 类型的对象都是相同的 retain 和 release 操作规则，所以Toll-Free Bridging 的使用比较简单，但是自从 ARC 加入后，Objective-C 类型的对象内存管理规则改变了，而 Core Foundation 依然是之前的机制，换句话说，Core Foundation 不支持 ARC。

这个时候就必须要考虑一个问题，在做 Core Foundation 与 Objective-C 类型转换的时候，用哪一种规则来管理对象的内存。显然，对于同一个对象，我们不能够同时用两种规则来管理，所以这里就必须要确定一件事情：哪些对象用 Objective-C（也就是ARC）的规则，哪些对象用 Core Foundation 的规则（也就是 MRC）的规则。或者说要确定对象类型转换了之后，内存管理的ownership 的改变。

> If you cast between Objective-C and Core Foundation-style objects, you need to tell the compiler about the ownership semantics of the object using either a cast (defined in objc/runtime.h) or a Core Foundation-style macro (defined in NSObject.h)

于是苹果在引入 ARC 之后对 Toll-Free Bridging 的操作也加入了对应的方法与修饰符，用来指明用哪种规则管理内存，或者说是内存管理权的归属。

#### 5.1 \_\_bridge

> 只是声明类型转变，但是不做内存管理规则的转变。

示例：

```
CFStringRef s = (__bridge CFStringRef)[[NSString alloc] initWithFormat:@"Hi, %@!", name];
```

只是 NSString 到 CFStringRef 的类型转化，但管理规则未变，依然要用 Objective-C 类型的 ARC 来管理 s，不能用CFRelease() 去释放 s。

#### 5.2 \_\_bridge\_retained、CFBridgingRetain()

> 将指针类型转变的同时，将内存管理的责任由原来的 Objective-C 交给 Core Foundation 来处理，也就是，将 ARC 转变为 MRC。

示例：

```
NSString * s1 = [[NSString alloc] initWithFormat:@"Hi, %@!", name];
￼CFStringRef s2 = (__bridge_retained CFStringRef)s1;
...
￼CFRelease(s2);  // 注意在使用结束后释放
```

在第二行做了转化，这时内存管理规则由 ARC 变成了 MRC，需要手动的来管理 s2 的内存，而对于 s1，即使将其置为 nil，也不能释放内存。

也可以写成：

```
NSString * s1 = [[NSString alloc] initWithFormat:@"Hi, %@!", name];
￼CFStringRef s2 = (CFStringRef)CFBridgingRetain(s1);
...
￼CFRelease(s2);  // 注意在使用结束后释放
```

#### 5.3 \_\_bridge\_transfer、CFBridgingRelease()

> 功能与 \_\_bridge\_retained 相反，表示将管理的责任由 Core Foundation 转交给 Objective-C，即将管理方式由MRC 转变为 ARC。

比如：

```
CFStringRef result = CFURLCreateStringByAddingPercentEscapes(. . .);
￼NSString * s = (__bridge_transfer NSString *)result; 
// 或 NSString * s = (NSString *)CFBridgingRelease(result);
￼return s;
```

这里将 result 的管理责任交给了 ARC 来处理，就不需要再显式地调用 CFRelease() 了。

这里和 ARC 中 4 个主要的修饰符 \_\_strong、\_\_weak、\_\_autoreleasing... 不同，这里修饰符的位置是放在类型前面的，虽然官方文档中没有说明，但最好与官方的相同。

![arc](https://upload-images.jianshu.io/upload_images/5294842-77a3391ea512ac3d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 

## 六、ARC下获取引用计数

#### 6.1 使用 KVC

```
[obj valueForKey:@"retainCount"];
```

#### 6.2 使用私有 API

```
OBJC_EXTERN int _objc_rootRetainCount(id);

_objc_rootRetainCount(obj);
```

这个不一定完全可信。Xcode 10.1 用的示例一直返回 1。

#### 6.3 使用 CFGetRetainCount

```
CFGetRetainCount((__bridge CFTypeRef)(obj))
```

使用 Toll-Free-Bridging 将 OC 对象的内容管理转为 Core Foundation 对象。

## 七、学习文章
[iOS 开发ARC内存管理技术要点](https://www.cnblogs.com/flyFreeZn/p/4264220.html)
[谈 Objective-C block的实现](http://blog.devtang.com/2013/07/28/a-look-inside-blocks/ "谈Objective-C block的实现")
[block 的实现](http://blog.csdn.net/hherima/article/details/38586101)
[正确使用 Block 避免Cycle Retain和Crash](http://tanqisen.github.io/blog/2013/04/19/gcd-block-cycle-retain/)
[ARC 的实现原理](https://blog.csdn.net/geeklee609/article/details/82142337)
