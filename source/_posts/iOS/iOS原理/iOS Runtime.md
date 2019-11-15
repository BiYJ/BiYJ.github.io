---
title: Runtime
categories: iOS原理
---


## 一、简介

C++ 是基于静态类型，而 Objective-C 是基于动态运行时类型。用 C++ 编写的程序通过编译器直接把函数地址硬编码进入可执行文件；Objective-C 则不能，而是在程序运行的时，利用 Runtime 根据条件判断作出决定。<font color=#cc0000>函数标识与函数实现之间的关联可以动态修改</font>。

<font color=#cc000>OC 把一些决定工作从编译链接推迟到运行时</font>，有很多类和成员变量在编译时是不知道的，而在运行时，编写的代码会转换成完整的确定的代码运行。因此，只有编译器是不够的，还需要一个运行时系统 （runtime system）来执行编译后的代码，它是整个 OC 运行框架的一块基石。

Runtime 又叫运行时，是一个用 C 和汇编编写的动态库，平时编写的 Objc 代码，底层都是基于它来实现的。它将 OC 和 C 紧密关联并提供动态特性，这个系统主要做两件事：

1. 封装 C 语言的结构体和函数，让开发者在运行时创建、检查或者修改类、对象和方法等。

2. 传递消息，找出方法的最终执行代码。

	①、静态类型编程语言在编译期就确定了函数的地址，OC 的方法调用（消息发送）是运行时动态确定（代价是性能下降，objc\_class 中的 objc\_cache 就是用来补偿这种性能下降的）；  

	②、类层次体系查找（isa + objc\_method\_list）+ 消息转发（动态解析 => 备用接收者 => 签名+打包+完整转发）

> 动态加载：[NSBundle](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSBundle_Class/index.html#//apple_ref/occ/cl/NSBundle)类提供了许多面向对象的便捷接口用于动态加载；比如 Retina 设备自动加载 @2x 的图片。

```objc
[receiver message];  
// 底层运行时会被编译器转化为：objc_msgSend(receiver, selector)
[receiver message:(id)arg...]; 
// 底层运行时会被编译器转化为：objc_msgSend(receiver, selector, arg1, arg2, ...)
```

Runtime 其实有两个版本：modern 和 legacy。我们现在用的 Objective-C 2.0 采用的是现行（Modern）版的 Runtime 系统，只能运行在 iOS 和 OS X 10.5 之后的 64 位程序中。而 OS X 较老的 32 位程序仍采用 Objective-C 1 中的 Legacy 版本。

当更改一个类的实例变量的布局时，在早期版本中你需要重新编译它的子类，而现行版就不需要。

Runtime 基本是用 C 和汇编（437 版本开始较多使用 mm 文件，但是仍用 C 语法）实现的，可见苹果为了动态系统的高效而作出了很多努力。[runtime源码](http://opensource.apple.com/tarballs/objc4/)，苹果和 GNU 各自维护一个开源的 runtime 版本，两个版本在努力的保持一致。


## 二、作用

Objc 与 Runtime 相关：

①、通过 Objective-C 源代码  
②、通过 Foundation 框架的 NSObject 类定义的方法  
③、通过对 Runtime 库函数的直接调用


#### 2.1 Objective-C 源代码

多数情况开发者只需要编写 Objc 代码即可，Runtime 系统自动在幕后搞定一切，就像简介中调用方法一样，编译器会将 Objc 代码转换成运行时代码，在运行时确定数据结构和函数。


#### 2.2 通过 Foundation 框架的 NSObject 类定义的方法

Cocoa 程序中绝大部分类都是继承了 NSObject 的行为的子类。（NSProxy 是个抽象超类）

NSObject 类有时仅仅定义了完成某件事情的模板，并没有提供实现的代码。例如 -description 方法，该方法返回类内容的字符串表示，用来调试程序。NSObject 类并不知道子类的内容，所以它只是返回类的名字和对象的地址。

一些 NSObject 的方法可以从 Runtime 系统中获取信息，允许对象进行自我检查。例如：

*   -class 方法返回对象的类；
*   -isKindOfClass: 和 -isMemberOfClass: 方法检查对象是否存在于指定的类的继承体系中
*   -respondsToSelector: 检查对象能否响应指定的消息；
*   -conformsToProtocol: 检查对象是否实现了指定协议类的方法；
*   -methodForSelector: 返回指定方法实现的地址。


#### 2.3 通过对 Runtime 库函数的直接调用

Runtime 系统是具有公共接口的动态共享库。头文件存放于 /usr/include/objc 目录下，使用时 \#import <objc/Runtime.h> 头文件即可。

许多函数可以让你使用纯 C 代码来实现 Objc 中同样的功能。除非是写一些 Objc 与其他语言的桥接或是底层的 debug 工作，否则一般不会用到这些 C 语言函数。


## 三、Runtime 相关的头文件

ios 的 sdk 中 usr/include/objc 文件夹下面有这样几个文件

```objc
List.h
NSObjCRuntime.h
NSObject.h
Object.h
Protocol.h
a.txt
hashtable.h
hashtable2.h
message.h
module.map
objc-api.h
objc-auto.h
objc-class.h
objc-exception.h
objc-load.h
objc-runtime.h
objc-sync.h
objc.h
runtime.h
```

都是和运行时相关的头文件，其中主要使用的函数定义在 message.h 和 runtime.h 这两个文件中。 在 message.h 中主要包含了一些向对象发送消息的函数，这是 OC 对象方法调用的底层实现。 runtime.h 是运行时最重要的文件，其中包含了对运行时进行操作的方法。 主要包括：

#### 3.1 操作对象的类型的定义

```objc
/// An opaque type that represents a method in a class definition.  一个类型，代表着类定义中的一个方法
typedef struct objc_method *Method;

/// An opaque type that represents an instance variable.  代表实例(对象)的变量
typedef struct objc_ivar *Ivar;

/// An opaque type that represents a category.  代表一个分类
typedef struct objc_category *Category;

/// An opaque type that represents an Objective-C declared property.  代表OC声明的属性
typedef struct objc_property *objc_property_t;

// Class 代表一个类，它在 objc.h 中这样定义的 typedef struct objc_class *Class;
struct objc_class {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class _Nullable super_class                              OBJC2_UNAVAILABLE;
    const char * _Nonnull name                               OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list * _Nullable ivars                  OBJC2_UNAVAILABLE;
    struct objc_method_list * _Nullable * _Nullable methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache * _Nonnull cache                       OBJC2_UNAVAILABLE;
    struct objc_protocol_list * _Nullable protocols          OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
```

这些类型的定义，对一个类进行了完全的分解，将类定义或者对象的每一个部分都抽象为一个类型 type，对操作一个类属性和方法非常方便。OBJC2\_UNAVAILABLE 标记的属性是 Ojective-C 2.0 不支持的，但实际上可以用响应的函数获取这些属性，例如：如果想要获取 Class 的 name 属性，可以按如下方法获取：

```objc
Class cls = obj.class;
// NSLog(@"%s", cls->name); // 用这种方法已经不能获取 name 了因为OBJC2_UNAVAILABLE
const char * clsName = class_getName(cls);
NSLog(@"%s", clsName);
```

#### 3.2 函数的定义

> 操作对象的方法一般以 object\_ 开头  
> 操作类的方法一般以 class\_ 开头  
> 操作类或对象的方法的方法一般以 method\_ 开头  
> 操作成员变量的方法一般以 ivar\_ 开头  
> 操作属性的方法一般以 property\_ 开头  
> 操作协议的方法一般以 protocol\_ 开头
> 
> 以 objc\_ 开头的方法，则是 runtime 最终的管家，可以获取内存中类的加载信息、类的列表、关联对象和关联属性等操作。

根据以上的函数的前缀可以大致了解到层级关系。

```objc
// 使用 runtime 对当前的应用中加载的类进行打印
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    unsigned int count = 0;
    
    Class * clsList = objc_copyClassList(&count);
    
    for (int i = 0; i < count; i++) {
        const char * clsName = class_getName(clsList[i]);
        NSLog(@"%s", clsName);
    }
}
```


## 四、术语及其数据结构

#### 4.1 SEL

它是 selector 在 Objc 中的表示（Swift 中是 Selector 类）。selector 是方法选择器，<font color=#cc0000>本质上是一个根据方法名 hash 化了的 key 值，为了加快查询方法实现的速度</font>。它的数据结构是：

```objc
typedef struct objc_selector *SEL;  // An opaque type that represents a method selector.
```

可以看出它是个映射到方法的 C 字符串，你可以通过 Objc 编译器命令 @selector() 或者 Runtime 系统的 sel\_registerName 函数来获取一个 SEL 类型的方法选择器。

> 注意：不同类中相同名字的方法所对应的 selector 是相同的，由于变量的类型不同，所以不会导致它们调用方法实现混乱。


#### 4.2 id

id 是一个参数类型，它是指向某个类的实例的指针。定义如下：

```objc
typedef struct objc_object *id;
struct objc_object { Class isa; };
```

objc_object 结构体包含一个 isa 指针，根据 isa 指针就可以找到对象所属的类。

> 注意：isa 指针在代码运行时并不总指向实例对象所属的类型，所以不能用它来确定类型。可以用对象的 -class 方法和 Runtime 的 object\_getClass() 方法。
> 
> Direct access to Objective-C's isa is deprecated in favor of object\_getClass()

[KVO](http://lizhaoloveit.com/2014/05/11/KVO/) 的实现机理就是将被观察对象的 isa 指针指向一个中间类而不是真实类型。


#### 4.3 Class

```objc
typedef struct objc_class *Class;
```

Class 其实是指向 objc\_class 结构体的指针。objc\_class 的数据结构如下：

```objc
struct objc_class {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    /*  父类  */
    Class _Nullable super_class                              OBJC2_UNAVAILABLE;   
    /*  类名  */
    const char * _Nonnull name                               OBJC2_UNAVAILABLE; 

    /* 类的版本信息，默认为 0 */
    long version                                             OBJC2_UNAVAILABLE;

    /* 类信息，供运行时期使用的一些位标识。

       如 CLS_CLASS (0x1L) 表示该类为普通 class，其中包含实例方法和变量;
         CLS_META (0x2L) 表示该类为 metaclass，其中包含类方法;
     */ 
    long info                                                OBJC2_UNAVAILABLE;

    /* 实例变量大小（包括从父类继承下来的实例变量）*/
    long instance_size                                       OBJC2_UNAVAILABLE;  

    /* 成员变量地址列表 */
    struct objc_ivar_list * _Nullable ivars                  OBJC2_UNAVAILABLE; 

    /* 方法地址列表，与 info 的一些标志位有关。

       如 CLS_CLASS (0x1L)，则存储实例方法；CLS_META (0x2L)，则存储类方法;
     */
    struct objc_method_list * _Nullable * _Nullable methodLists   OBJC2_UNAVAILABLE;

    /* 缓存最近使用的方法地址，用于提升效率 */
    struct objc_cache * _Nonnull cache                       OBJC2_UNAVAILABLE; 

    /* 存储该类声明遵守的协议的列表 */
    struct objc_protocol_list * _Nullable protocols          OBJC2_UNAVAILABLE;  
#endif

} OBJC2_UNAVAILABLE;
```

从 objc\_class 可以看到：一个运行时类中关联了它的父类指针、类名、成员变量、方法、缓存以及附属的协议。

其中 objc\_ivar\_list 和 objc\_method\_list 分别是成员变量列表和方法列表：

```objc
// 成员变量列表
struct objc_ivar_list {
    int ivar_count                                           OBJC2_UNAVAILABLE;
#ifdef __LP64__
    int space                                                OBJC2_UNAVAILABLE;
#endif
    /* variable length structure */
    struct objc_ivar ivar_list[1]                            OBJC2_UNAVAILABLE;
}                                                            OBJC2_UNAVAILABLE;

// 方法列表
struct objc_method_list {
    struct objc_method_list *obsolete                        OBJC2_UNAVAILABLE;

    int method_count                                         OBJC2_UNAVAILABLE;
#ifdef __LP64__
    int space                                                OBJC2_UNAVAILABLE;
#endif
    /* variable length structure */
    struct objc_method method_list[1]                        OBJC2_UNAVAILABLE;
}
```

由此可见，我们可以动态修改 methodList 的值来添加成员方法，这也是 Category 实现的原理，同样解释了 Category 不能添加属性的原因。[深入理解 Objective-C: Category](http://tech.meituan.com/DiveIntoCategory.html)。

objc\_ivar\_list 结构体用来存储成员变量的列表，而 objc\_ivar 则是存储了单个成员变量的信息；同理，objc\_method\_list 结构体存储着方法数组的列表，而单个方法的信息则由 objc\_method 结构体存储。

值得注意的是，由 objc\_object 和 objc\_class 的代码可以看出，objc\_class 中也有一个 isa 指针，这说明 objc\_class 也是一个对象，分别称作类对象（class object）和实例对象（instance object）。

> 实例对象 objc\_object 的 isa 指针指向的类结构称为 class，也就是该对象所属的类，其中存放着普通成员变量与动态方法（" - " 开头的方法）；
> 
> 类对象 objc_class 的 isa 指针指向的类结构称为 meta class，其中存放着 static 类型的成员变量与 static 类型的方法（" + " 开头的方法）。

为了处理类和对象的关系，Runtime 库创建了 Meta Class (元类) ，类对象所属的类 Class 就叫做元类。Meta Class 表述了类对象本身所具备的元数据。

开发者所熟悉的类方法，就源自于 Meta Class。可以理解为类方法就是类对象的实例方法。每个类仅有一个类对象，而每个类对象仅有一个与之相关的元类。

当调用 +alloc 的消息时，这个消息实际上被发送给了一个类对象（Class Object），这个类对象必须是一个元类的实例，而这个元类同时也是一个根元类（Root Meta Class）的实例。所有元类的 isa 指针最终都指向根元类。

```objc
[NSObject.class alloc];    // [NSObject alloc]
```

所以当 [NSObject alloc] 这条消息发送给类对象的时候，运行时代码 objc\_msgSend() 会去它元类中查找能够响应消息的方法实现，如果找到了，就会对这个类对象执行方法调用。

<center>
![Meta Class](https://upload-images.jianshu.io/upload_images/5294842-f45adc9fd1faea0b?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

实线是 super\_class 指针，虚线是 isa 指针。而根元类的父类是 NSObject，isa 指向了自己，NSObject 没有父类。

最后 objc\_class 中还有一个 objc\_cache 缓存，它的作用很重要，后面会提到。


#### 4.4 Method

代表类中某个方法的类型。

```objc
typedef struct objc_method *Method;

struct objc_method {
    SEL _Nonnull method_name            OBJC2_UNAVAILABLE;   // 方法名
    char * _Nullable method_types       OBJC2_UNAVAILABLE;   // 方法类型
    IMP _Nonnull method_imp             OBJC2_UNAVAILABLE;   // 方法实现
}   
```   

*   method\_name 类型为 SEL
*   method\_types 是个 char 指针，存储方法的参数类型和返回值类型
*   method\_imp 指向了方法的实现，本质是一个函数指针


#### 4.5 Ivar

表示成员变量的类型。

```objc
typedef struct objc_ivar *Ivar;

struct objc_ivar {
    char * _Nullable ivar_name                OBJC2_UNAVAILABLE;
    char * _Nullable ivar_type                OBJC2_UNAVAILABLE;
    int ivar_offset                           OBJC2_UNAVAILABLE;   // 基地址偏移字节

#ifdef __LP64__
    int space                                 OBJC2_UNAVAILABLE;  // 占用空间
#endif
}    
```


#### 4.6 IMP

objc.h 中定义如下：

```objc
// 参数 1 : 如果是实例方法，则是 self 的内存地址；如果是类方法，则是指向元类的指针
typedef id (*IMP)(id, SEL, ...);
```

它就是一个<font color=#cc0000>由编译器生成的函数指针，指向方法实现的首地址</font>。当你发起一个 ObjC 消息之后，最终它会执行哪段代码，就是由这个函数指针指定的。而 IMP 这个函数指针就指向了这个方法的实现。

如果得到了执行某个实例某个方法的入口，就可以绕开消息传递阶段，直接执行方法，这在后面 Cache 中会提到。

你会发现 IMP 指向的方法与 objc\_msgSend() 函数类型相同，参数都包含 id 和 SEL 类型。每个方法名都对应一个 SEL 类型的方法选择器，而每个实例对象中的 SEL 对应的方法实现肯定是唯一的，通过一组 id 和 SEL 参数就能确定唯一的方法实现地址。一个确定的方法也只有唯一的一组 id 和 SEL 参数。



#### 4.7 Cache

runtime.h 中定义如下：

```objc
typedef struct objc_cache *Cache

struct objc_cache {
    /* 指定分配 cache buckets 的总数。在方法查找中，Runtime 使用这个字段确定数组的索引位置。*/
    unsigned int mask /* total = mask + 1 */                 OBJC2_UNAVAILABLE;

    /* 实际占用cache buckets的总数 */
    unsigned int occupied                                    OBJC2_UNAVAILABLE;

    /*  指定 Method 数据结构指针的数组。这个数组可能包含不超过 mask + 1 个元素。

       指针可能是 NULL，表示这个缓存 bucket 没有被占用，另外被占用的 bucket 可能是不连续的。这个数组可能会随着时间而增长
     */
    Method _Nullable buckets[1]                              OBJC2_UNAVAILABLE;
};
```

Cache 优化方法调用的性能。每当实例对象接收到一个消息时，优先在 Cache 中查找，它不会直接在 isa 指针指向的类的方法列表中遍历查找能够响应的方法，因为每次都要查找效率太低了。

Runtime 系统会把被调用的方法存到 Cache 中，如果一个方法被调用，那么它有可能今后还会被调用，下次查找的时候就会效率更高。就像计算机组成原理中 CPU 绕过主存先访问 Cache 一样。


#### 4.8 Property

```objc
typedef struct objc_property *Property;
typedef struct objc_property *objc_property_t;  // 这个更常用
```

可以通过 class\_copyPropertyList() 和 protocol\_copyPropertyList() 方法获取类和协议中的属性：

```objc
OBJC_EXPORT objc_property_t _Nonnull * _Nullable
class_copyPropertyList(Class _Nullable cls, unsigned int * _Nullable outCount)
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);

OBJC_EXPORT objc_property_t _Nonnull * _Nullable
protocol_copyPropertyList(Protocol * _Nonnull proto,
                          unsigned int * _Nullable outCount)
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);
```

> 返回的是属性列表，列表中每个元素都是一个 objc\_property\_t 指针。

property\_getName() 用来查找属性的名称，返回 c 字符串。  
property\_getAttributes() 函数挖掘属性的真实名称和 @encode 类型，返回 c 字符串。  
class\_getProperty() 和 protocol\_getProperty() 通过给出属性名在类和协议中获得属性的引用。

<center>
![类对象结构图](https://upload-images.jianshu.io/upload_images/5294842-e2cb59d1344df2d8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

## 五、消息

体会官方文档中的 messages aren’t bound to method implementations until Runtime。<font color=#cc0000>消息直到运行时才会与方法实现进行绑定</font>。

这里要清楚一点，objc\_msgSend() 方法看起来好像返回了数据，其实 objc\_msgSend() 从不返回数据，而是方法在运行时被调用实现后才会返回数据。下面详细叙述消息发送的步骤：

<center>
![消息发送](https://upload-images.jianshu.io/upload_images/5294842-2d3464848b8f4fc2?imageMogr2/auto-orient/strip)
</center>

①、首先检测这个 selector 是不是要忽略。比如 Mac OS X 开发，有了垃圾回收就不理会 retain、release 这些函数；

②、检测这个 selector 的 target 是不是 nil。Objc 允许对一个 nil 对象执行任何方法不会 Crash，因为运行时会被忽略掉。

③、如果上面两步都通过了，那么就开始查找这个类的实现 IMP，先从 cache 里查找，如果找到了就运行对应的函数去执行相应的代码。

④、如果 cache 找不到就找类的方法列表中是否有对应的方法。

⑤、如果类的方法列表中找不到就到父类的方法列表中查找，一直找到 NSObject 类为止。

⑥、如果还找不到，就要开始进入动态方法解析了，后面会提到。

在消息的传递中，编译器会根据情况在 objc\_msgSend()、objc\_msgSend\_stret()、objc\_msgSendSuper()、objc\_msgSendSuper\_stret() 这四个方法中选择一个调用。如果消息是传递给父类，那么会调用名字带有 Super 的函数；如果消息返回值是数据结构而不是简单值时，会调用名字带有 stret 的函数发送消息和接收返回值。

#### 5.1 方法中的隐藏参数

> 我们经常使用关键字 self，但是 self 是如何获取当前方法的对象呢？

其实，这也是 Runtime 系统的作用，self 是在方法运行时被动态传入的。

当 objc\_msgSend() 找到方法对应实现时，它将直接调用该方法实现，并将消息中所有参数都传递给方法实现，同时，它还将传递两个隐藏参数：

*   self 当前方法的对象指针，接受消息的对象
*   \_cmd 当前方法的 SEL 指针，方法选择器

因为在源代码方法的定义中，我们并没有发现这两个参数的声明。它们是在代码被编译时被插入方法实现中的。尽管这些参数没有被明确声明，在源代码中我们仍然可以引用它们。

这两个参数中，self 更实用。它是在方法实现中访问消息接收者对象的实例变量的途径。

这时我们可能会想到另一个关键字 super，实际上 super 关键字接收到消息时，编译器会创建一个 objc\_super 结构体：

```objc
struct objc_super {
    /// Specifies an instance of a class.
    __unsafe_unretained _Nonnull id receiver;

    /// Specifies the particular superclass of the instance to message. 
#if !defined(__cplusplus)  &&  !__OBJC2__
    /* For compatibility with old objc-runtime.h header */
    __unsafe_unretained _Nonnull Class class;
#else
    __unsafe_unretained _Nonnull Class super_class;
#endif
    /* super_class is the first class to search */
};
```

这个结构体指明了消息应该被传递给特定的父类。

receiver 仍然是 self 本身，当我们想通过 [super class] 获取父类时，编译器其实是将指向 self 的 id 指针和 class 的 SEL 传递给了 objc\_msgSendSuper() 函数。只有在 NSObject 类中才能找到 class 方法，然后 class 方法底层被转换为 object\_getClass()， 接着底层编译器将代码转换为 objc\_msgSend(objc\_super->receiver, @selector(class))，传入的第一个参数是指向 self 的 id 指针，与调用 [self class] 相同，所以我们得到的永远都是 self 的类型。因此你会发现：

```objc
NSLog(@"%@", NSStringFromClass([super class]));

2018-11-05 11:30:25.082134+0800 Demo[22838:128408] ViewController   // 当前类
```


#### 5.2 获取方法地址

NSObject 中有 - methodForSelector: 实例方法，你可以用它来获取某个方法选择器对应的 IMP：

```objc
{
     CFTimeInterval ti = CFAbsoluteTimeGetCurrent();
        
     for (int i = 0; i < 10000; i++) {
          [self fail:YES];
     }
     NSLog(@"%f", CFAbsoluteTimeGetCurrent() - ti);
}

- (void)fail:(BOOL)value
{

}

2018-11-05 13:06:22.683838+0800 Demo[36187:211037] 4.104993


{
     void (*setter)(id, SEL, BOOL) = (void (*)(id, SEL, BOOL))[self methodForSelector:@selector(fail:)];

     CFTimeInterval ti = CFAbsoluteTimeGetCurrent();
    
     for (int i = 0; i < 10000; i++) {
          setter(self, @selector(fail:), YES);
     }
     NSLog(@"%f", CFAbsoluteTimeGetCurrent() - ti);
}

2018-11-05 13:05:48.480498+0800 Demo[36095:209893] 3.751424
```

虽然是更高效的调用方法，但这种做法很少用，除非是<font color=#cc0000>需要持续大量重复调用某个方法</font>的情况，才会选择使用，<font color=#cc0000>以免消息发送泛滥</font>。

> 注意：methodForSelector: 方法是由 Runtime 系统提供的，而不是 Objc 自身的特性


## 六、动态方法解析

如果用关键字 @dynamic 在 .m 文件中修饰一个属性，表明我们会为这个属性动态提供存取方法，编译器不会再默认生成该属性的 setter 和 getter 方法。

```objc
@dynamic propertyName;
```

这时，可以通过分别重载 resolveInstanceMethod: 和 resolveClassMethod: 方法添加实例方法实现和类方法实现。

Runtime 系统会在 Cache 和类、父类的方法列表中找不到要执行的方法时，会调用 resolveInstanceMethod: 或 resolveClassMethod: 来给开发者一次动态添加方法实现的机会。

```objc
void dynamicIMP(id self, SEL _cmd) {
    // implementation ....
}

@implementation MyClass

+ (BOOL)resolveInstanceMethod:(SEL)aSEL
{
     if (aSEL == @selector(resolveThisMethodDynamically)) {
          class_addMethod([self class], aSEL, (IMP) dynamicIMP, "v@:");
          return YES;
     }
     return [super resolveInstanceMethod:aSEL];
}

@end
```

上面为 resolveThisMethodDynamically 方法添加了实现内容，就是 dynamicIMP 方法中的代码。其中 "v@:" 表示返回值和参数，这个符号表示的含义见：[Type Encoding](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)

> 动态方法解析会在消息转发机制侵入前执行，动态方法解析器将会首先给予提供该方法选择器对应的 IMP 的机会。如果你想让 aSEL 被传送到转发机制，就让 resolveInstanceMethod: 方法返回 NO。


## 七、消息转发

<center>
![消息转发](https://upload-images.jianshu.io/upload_images/5294842-3f7a92a32f8cc7d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

①、通过 resolveInstanceMethod: 方法决定是否动态添加方法。如果返回 YES 则通过 class\_addMethod() 动态添加方法，消息得到处理，结束；如果返回 NO，则进入下一步；

②、进入 forwardingTargetForSelector: 方法，用于指定备选对象响应这个selector，不能指定为 self。如果返回某个对象则会调用对象的方法，结束。如果返回 nil，则进入下一步；

③、通过 methodSignatureForSelector: 方法签名，如果返回 nil，则消息无法处理。如果返回 methodSignature，则进入下一步；

④、调用 forwardInvocation: 方法，可以通过 anInvocation 对象做很多处理，比如修改实现方法、修改响应对象等，如果方法调用成功，则结束。如果失败，则进入 doesNotRecognizeSelector 方法，若我们没有实现这个方法，那么就会 crash。

#### 7.1 重定向

消息转发机制执行前，Runtime 系统允许我们替换消息的接收者为其他对象。通过 - (id)forwardingTargetForSelector:(SEL)aSelector 方法。

```objc
- (id)forwardingTargetForSelector:(SEL)aSelector
{
     if(aSelector == @selector(xxx:)){
          return otherObj;
     }
     return [super forwardingTargetForSelector:aSelector];
}
```

如果此方法返回 nil 或者 self，则会计入消息转发机制（forwardInvocation:），否则将向返回的对象重新发送消息。


#### 7.2 转发

当动态方法解析不做处理返回 NO 时，则会触发消息转发机制。这时 forwardInvocation: 方法会被执行，我们可以重写这个方法来自定义我们的转发逻辑：

```objc
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
{
     return [otherObj methodSignatureForSelector:aSelector];
}

/**
 *  @param  anInvocation  封装了原始的消息和消息的参数
 */
- (void)forwardInvocation:(NSInvocation *)anInvocation
{
     // 判断 otherObj 对象是否实现了方法
     if ([otherObj respondsToSelector:[anInvocation selector]]) {
          [anInvocation invokeWithTarget:otherObj];
     }
     else {
          [super forwardInvocation:anInvocation];
     }
}
```

开发者可以实现 forwardInvocation: 方法来对不能处理的消息做一些处理。也可以将消息转发给其他对象处理，而不抛出错误。

> 参数 anInvocation 是从哪来的？
> 
> 在 forwardInvocation: 消息发送前，Runtime 系统会向对象发送 methodSignatureForSelector: 消息，并取到返回的方法签名用于生成 NSInvocation 对象。所以重写 forwardInvocation: 的同时也要重写 methodSignatureForSelector: 方法，否则会抛异常。

当一个对象由于没有相应的方法实现而无法相应某消息时，运行时系统将通过 forwardInvocation: 消息通知该对象。每个对象都继承了 forwardInvocation: 方法。但是 NSObject 中的方法实现只是简单的调用了 doesNotRecognizeSelector:。通过实现自己的 forwardInvocation: 方法，我们可以将消息转发给其他对象。

forwardInvocation: 方法就是一个不能识别消息的分发中心，将这些不能识别的消息转发给不同的接收对象，或者转发给同一个对象，再或者将消息翻译成另外的消息，亦或者简单的 “吃掉” 某些消息，因此没有响应也不会报错。这一切都取决于方法的具体实现。

> forwardInvocation: 方法只有在消息接收对象中无法正常响应消息时才会被调用。所以，如果我们想往一个对象将一个消息转发给其他对象时，要确保这个对象不能有该消息的所对应的方法。否则，forwardInvocation: 将不可能被调用。

#### 7.3 转发和多继承

转发和继承相似，可用于为 Objc 编程添加一些多继承的效果。就像下图那样，一个对象把消息转发出去，就好像它把另一个对象中的方法接过来或者 “继承” 过来一样。

<center>
![](https://upload-images.jianshu.io/upload_images/5294842-70cda98ab8c42661?imageMogr2/auto-orient/strip)
</center>

在上图中 Warrior 和 Diplomat 没有继承关系，但是 Warrior 将 negotiate 消息转发给了 Diplomat 后，就好似 Diplomat 是 Warrior 的超类一样。这使得在不同继承体系下的两个类可以实现继承对方的方法，消息转发弥补了 Objc 不支持多继承的性质，也避免了因为多继承导致单个类变得臃肿复杂。


#### 7.4 转发与继承

虽然转发可以实现继承的功能，但是 NSObject 还是必须表面上很严谨，像 respondsToSelector: 和 isKindOfClass: 这类方法只会考虑继承体系，不会考虑转发链。

如果判断上图中的 Warrior 对象是否能响应 negotiate 消息：

```objc
if ( [aWarrior respondsToSelector:@selector(negotiate)] )
    ...
```

尽管 Warrior 能转发消息给 Diplomat 类响应消息，但返回依然是 NO。

如果想要让外界以为 Warrior 继承到了 Diplomat 的 negotiate 方法，可以重新实现 respondsToSelector: 和 isKindOfClass: 来加入你的转发算法：

```objc
- (BOOL)respondsToSelector:(SEL)aSelector
{
     if ( [super respondsToSelector:aSelector] )
          return YES;
     else {
          /* Here, test whether the aSelector message can     *
           * be forwarded to another object and whether that  *
           * object can respond to it. Return YES if it can.  */
     }
     return NO;
}
```

除了 respondsToSelector: 和 isKindOfClass: 之外，instancesRespondToSelector: 中也应该写一份转发算法。如果使用了协议，conformsToProtocol: 同样需要。

如果一个对象想要转发它接受的任何远程消息，那得重新实现 - methodSignatureForSelector: 返回准确的方法描述 ，这个方法会最终响应被转发的消息，从而生成一个确定的 NSInvocation 对象描述消息和消息参数。这个方法最终响应被转发的消息。


## 八、应用场景

#### 8.1 获取属性/成员变量列表

```objc
// 简单的定义了一个成员变量和两个属性
@interface Person : NSObject
{
    @private
         CGFloat _height;
}
@property (nonatomic, copy) NSString * name;
@property (nonatomic, assign) NSInteger age;

@end
```

使用 class\_copyIvarList() 函数获取成员变量的列表，使用 class\_copyPropertyList() 函数获取属性列表：

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    Class cls = NSClassFromString(@"Person");   // Class cls = Person.class;
    
    unsigned int count = 0;
    // 获取成员变量数组
    Ivar * ivarList = class_copyIvarList(cls, &count);
    for (int i = 0; i < count; i++) {
        // 获取成员变量名
        const char * ivarName = ivar_getName(ivarList[i]); 
        NSLog(@"%s", ivarName);
    }
    
    // 获取属性数组
    objc_property_t * ptyList = class_copyPropertyList(cls, &count);
    for (int i = 0; i < count; i++) {
        const char * ptyName = property_getName(ptyList[i]);
        NSLog(@"%s", ptyName);
    }
}

2018-11-04 17:28:03.905326+0800 Demo[5894:1444503] _height
2018-11-04 17:28:03.905486+0800 Demo[5894:1444503] _name
2018-11-04 17:28:03.905616+0800 Demo[5894:1444503] _age
2018-11-04 17:28:03.905745+0800 Demo[5894:1444503] name
2018-11-04 17:28:03.905877+0800 Demo[5894:1444503] age
```

从这里就可以看出 @property 做了三件事：

①、生成一个带下划线的成员变量  
②、生成这个成员变量的 set 方法  
③、生成这个成员变量的 get 方法

因此会输出三个成员变量 \_height、\_age 和 \_name。并且从上面可知 ivarList 能够获取到 @property 关键字定义的属性 ，而 propertyList 不能获取到成员变量。即用 ivarList 可以获取到所有的成员变量和属性。

```objc
@property (nonatomic, copy, readonly) NSString * name;  // 只读属性

- (NSString *)name
{
    return @"job";
}

2018-11-04 17:52:52.690815+0800 Demo[6025:1474196] _height
2018-11-04 17:52:52.691025+0800 Demo[6025:1474196] _age
2018-11-04 17:52:52.691159+0800 Demo[6025:1474196] name
2018-11-04 17:52:52.691308+0800 Demo[6025:1474196] age
```

当只读属性 name 重写了 getter 方法时，无论使用 ivarList 还是使用 propertyList 都无法获取到 \_name 成员变量。

> 一个 readonly 的属性，到底是 didSet+set 好，还是重写 getter 好?

大部分的 readonly 的属性是计算型的，依赖于其他属性，因此可以使用 didSet+set，也就是在其他属性的 set 方法内，将只读属性 set。 但是 didSet+set 有时候完全没有必要，不符合懒加载的规则，浪费了计算能力，用重写 getter 的方法好一些。

> 在 KVC 时，想要获取全部的成员变量和属性， 怎么办呢？

首先要了解 setValue:forKeyPath: 方法的底层实现：

①、首先去类的方法列表去寻找有没有 setter 方法，如果有，就直接调用 [obj setXX:value]  
②、查找有没有成员变量 \_XX，如果有 \_XX = value；  
③、查找有没有成员变量 XX，如果有 XX = value；  
④、如果都没有找到，直接报错。

```objc
Terminating app due to uncaught exception 'NSUnknownKeyException', 
reason: '[<Person 0x102bb7388> setValue:forUndefinedKey:]: 
this class is not key value coding-compliant for the key name.'
```

首先，只读属性为什么要为它赋值呢，因此对它进行 kvc 也不合情理。

另外，对于重写了 getter 的只读属性而言：如果对 propertyList 的属性一次使用 kvc，就会报错，因此为保证代码正常，不能使用 propertyList 的属性进行 kvc；

使用 ivaList 时是无法获取到重写了 getter 的只读属性，因此是 kvc 的最佳方案。再者，使用 propertyList 无法获取成员变量 \_height，无法对成员变量进行赋值。而使用 ivaList 是可以将需要赋值的成员变量都获取的。

要想不对 \_height 成员变量赋值，在 kvc 时又可以这样改进一下，通过 ivarList 获取，去掉 propertyList 中没有的成员变量，这样就过滤掉了 \_height。

```objc
@property (nonatomic, weak) NSTimer * timer;
@property (nonatomic, strong) NSThread * thread;
@property (nonatomic, strong, readonly) AModel * a;  // 自定义对象

{
     unsigned int count = 0;
     objc_property_t * propertyList = class_copyPropertyList(self.class, &count);
    
     for (int i = 0; i < count; i++) {
          NSLog(@"%s", property_getAttributes(propertyList[i]));
     }
}

2018-11-05 15:09:37.839596+0800 Demo[39749:288880] T@"NSTimer",W,N,V_timer
2018-11-05 15:09:37.839692+0800 Demo[39749:288880] T@"NSThread",&,N,V_thread
2018-11-05 15:09:37.839771+0800 Demo[39749:288880] T@"AModel",R,N,V_a
```

通过 property\_getAttributes() 方法获取属性的参数。



#### 8.2 KVC字典转模型

获取属性/成员列表一个重要的应用就是：一次取出模型中的属性/成员变量，根据变量名获取字典中的 key 然后取出对应的 value，使用 setValue:forKeyPath: 方法设置值。

为什么要这样，而不再使用方法 setValuesForKeysWithDictionary:。因为在 setValuesForKeysWithDictionary: 方法内部会执行这样一个过程：

①、遍历字典里面的所有 key，取出 key；  
②、取出 key 的 value，即 dict[key]；  
③、使用方法 [setValue:value forKeyPath:key] 给模型的属性/成员变量进行赋值。

因此，开发中经常遇到的字典中的 key 比模型中多时，会出现的 this class is not key-value compliant for ‘xxx’ 这个 bug，是因为模型中没有这个属性/成员变量。当模型中的属性比字典中多时，使用 setValuesForKeysWithDictionary: ，多出来的属性是对象类型时为 null，基本数据类型时会有一个系统默认值（如 int 为 0）。

因此使用逐一为属性赋值的方法进行 KVC：

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
     Class cls = Person.class;
     unsigned int count = 0;
    
     Person * person = [[Person alloc] init];
     NSDictionary * dict = @{  @"name" : @"Tom", @"age" : @19, @"height": @175 };
    
     Ivar * ivars = class_copyIvarList(cls, &count);
    
     for (int i = 0; i < count; i++) {
          const char * clsName = ivar_getName(ivars[i]);
          NSString * name = [NSString stringWithUTF8String:clsName];
          NSString * key = [name substringFromIndex:1];  // 去掉'_'
          [person setValue:dict[key] forKey:key];
     }
}

2018-11-04 19:42:16.964474+0800 Demo[6425:1574210] height:175.0000，name:Tom，age:19，time:(null)
```

使用这种方式进行 kvc，即使字典中的 key 多的时候也不会有 bug。

但新的问题出现了，如果模型中的属性比字典中的 key 多便会出现 bug，而且如果多的是对象类型不会有 bug，该属性的值为 null，如果是基本数据类型就会出错 could not set nil as the value for the key ‘xxx’。

> setObject:forKey: 如果 value 传 nil 会直接报错；setValue:forKey: 则不会，会赋值 nil。具体可以看文档说明。

解决基础类型被赋值 nil 的 bug：可以在 [setValue:value forKeyPath:key] 方法调用之前取出属性对应的类型，如果类型是基本数据类型，value 替换为默认值（如 int 对应默认值为 0）。

runtime 提供的 ivar\_getTypeEncoding() 函数可以获取到属性的类型。[Type Encodings](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1)

```objc
for (int i = 0; i < count; i++) {
     const char * ivarName = ivar_getName(ivars[i]);
     NSString * name = [NSString stringWithUTF8String:ivarName];
     NSString * key  = [name substringFromIndex:1];
    
     const char * coding = ivar_getTypeEncoding(ivars[i]); // 获取类型
     NSString * strCode = [NSString stringWithUTF8String:coding];
     id value = dict[key];

     if ([strCode isEqualToString:@"f"]) {  // 判断类型是否是 float
          value = @(0.0);
     }
    
     [person setValue:value forKey:key];
}
```

method\_getTypeEncoding() 函数可以获取到方法类型编码

```objc
{
     Method m = class_getInstanceMethod(self.class, @selector(do:at:on:));
    
     NSLog(@"%s", method_getTypeEncoding(m));
}
- (BOOL)do:(NSString *)something at:(char)place on:(int)count;

2018-11-05 14:42:30.891829+0800 Demo[38588:270099] B32@0:8@16c24i28
```

property\_getAttributes() 函数可以获取到属性的参数。[Declared Properties](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtPropertyIntrospection.html#//apple_ref/doc/uid/TP40008048-CH101-SW6)


#### 8.3 NSCoding 归档和解档

获取属性/成员列表另外一个重要的应用就是进行归档和解档，其原理和上面的 kvc 基本上一样：

```objc
- (void)encodeWithCoder:(NSCoder *)aCoder
{
     unsigned int count = 0;
     Ivar * ivars = class_copyIvarList(self.class, &count);

     for (int i = 0; i < count; i++) {
          const char * ivarName = ivar_getName(ivars[i]);
          NSString * name = [NSString stringWithUTF8String:ivarName];
          NSString * key  = [name substringFromIndex:1];
        
          id value = [self valueForKey:key];  // 取出 key 对应的 value
          [aCoder encodeObject:value forKey:key];   // 编码
     }
}

- (id)initWithCoder:(NSCoder *)aDecoder
{
     if (self = [super init]) {

          unsigned int count = 0;
          Ivar * ivars = class_copyIvarList(self.class, &count);

          for (int i = 0; i < count; i++) {
               const char * ivarName = ivar_getName(ivars[i]);
               NSString * name = [NSString stringWithUTF8String:ivarName];
               NSString * key = [name substringFromIndex:1];
            
               id value = [aDecoder decodeObjectForKey:key];  // 解码
               [self setValue:value forKey:key];  // 设置 key 对应的 value
          }
     }
     return self;    
}
```

#### 8.4 交换方法实现

交换两个方法的实现一般写在类的 load 方法里面，因为 load 方法会在程序运行前加载一次，而 initialize 方法会在类或者子类第一次使用的时候调用，当有分类的时候会调用多次。

```objc
+ (void)load
{
     static dispatch_once_t onceToken;
     dispatch_once(&onceToken, ^{

          Method orginalMethod = class_getClassMethod([UIImage class], @selector(imageNamed:));
          Method swizzleMethod = class_getClassMethod([UIImage class], @selector(my_imageNamed:));
        
          //方法交换
          method_exchangeImplementations(orginalMethod, swizzleMethod);
     });
}

+ (UIImage *)my_imageNamed:(NSString *)name
{
     return [self my_imageNamed:name];
}
```

需要注意的是

①、可以交换的两个方法的参数必须是匹配的，参数的类型一致。  
②、如果想在 my\_imageNamed: 的内部调用 imageNamed: 方法，此时调用 [self my\_imageNamed:name] 实际上是在调用 imageName: 的代码实现。

任何一个方法都有两个重要的属性：SEL 方法的编号，IMP 方法的实现。方法的调用过程实际上是根据 SEL 去寻找 IMP。

#### 8.5 类/对象的关联对象

关联对象不是为类/对象添加属性或者成员变量（因为在设置关联后也无法通过 ivarList 或者 propertyList 取得) ，而是为类添加一个相关的对象，通常用于存储类信息，例如存储类的属性列表数组，为将来字典转模型的方便。 例如，将属性的名称存到数组中设置关联

```objc
/* 参数 1 : 关联到对象
   参数 2 : 关联的 key，可以是任意类型
   参数 3 : 被关联的对象
   参数 4 : 关联引用的规则
           enum {
                OBJC_ASSOCIATION_ASSIGN = 0,
                OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1,
                OBJC_ASSOCIATION_COPY_NONATOMIC = 3,
                OBJC_ASSOCIATION_RETAIN = 01401,
                OBJC_ASSOCIATION_COPY = 01403
           };
*/
objc_setAssociatedObject(self, key, value, OBJC_ASSOCIATION_COPY_NONATOMIC);

id value = objc_getAssociatedObject(self, key);
```


#### 8.6 动态添加方法，拦截未实现的方法

每个类都有继承自 NSObject 的两个类方法

```objc
+ (BOOL)resolveClassMethod:(SEL)sel;
+ (BOOL)resolveInstanceMethod:(SEL)sel;
```

一个适用于类方法，一个适用于对象方法。

在代码中调用没有实现的方法时，也就是 sel 标识的方法没有实现，都会先调用这两个方法中的一个拦截。 通常的做法是在 resolve 的内部指定 sel 对应的 IMP，从而完成方法的动态创建和调用两个过程，也可以不指定 IMP 打印错误信息后直接返回。

```objc
// 每个方法的内部都默认包含两个参数，被称为隐式参数：id self 和 SEL _cmd
void method(id self, SEL _cmd) {

}

+ (BOOL)resolveInstanceMethod:(SEL)sel
{
     if ([NSStringFromSelector(sel) isEqualToString:@"doSomething"]) {       
     
          /* 参数 4 : const char *types 方法的类型

             要注意函数至少有 self 和 _cmd 参数，第二个和第三个字符必须是 “@:”。

             如果想要再增加参数，就可以从实现的第三个参数算起：
                 class_addMethod(self, sel, method, "v@:@"); // 多一个对象类型参数增加了 @

                 void method(id self, SEL _cmd, NSString * name) {  }

             返回值：YES if the method was found and added to the receiver, otherwise NO.
          */ 
          class_addMethod(self, sel, method, "v@:");  // 为 sel 指定实现为 method
     }
     return YES;
}
```


#### 8.7 动态创建一个类

动态创建一个类，为这个类添加成员变量和方法，并创建这个类型的对象：

```objc
#import <objc/message.h>

void sayFunction(id self, SEL _cmd, id param) {
    NSLog(@"%ld岁的%@在%@说%@", [object_getIvar(self, class_getInstanceVariable([self class], "_age")) integerValue], object_getIvar(self, class_getInstanceVariable([self class], "_name")), object_getIvar(self, class_getInstanceVariable([self class], "schoolName")), param);
}

- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
    // 创建 Student 类。参数 1 : 父类   参数 2 : 类名   参数 3 : 通常为 0
    Class StudentClass = objc_allocateClassPair(NSObject.class, "Student", 0);
    
    // 添加一个 NSString 的变量，第四个参数是对齐方式，第五个参数是参数类型
    // 必须在 objc_allocateClassPair and 和 objc_registerClassPair 之间调用
    if (class_addIvar(StudentClass, "schoolName", sizeof(NSString *), 0, "@")) {
        NSLog(@"添加成员变量成功");
    }
    
    // 添加 NSString * _name 成员变量
    class_addIvar(StudentClass, "_name", sizeof(NSString *), 0, @encode(NSString *));
    // 添加 int _age 成员变量
    class_addIvar(StudentClass, "_age", sizeof(int), 0, @encode(int));
    
    // 为 Student 类添加方法 "v@:" 这种写法见参数类型连接
    SEL sel = sel_registerName("sayFunction:");
    if (class_addMethod(StudentClass, sel, (IMP)sayFunction, "v@:@")) {
        NSLog(@"添加方法成功");
    }
    
    // 注册这个类到 runtime 系统中就可以使用了
    objc_registerClassPair(StudentClass);
    
    // 使用创建的类
    id student = [[StudentClass alloc] init];
    
    // 给刚刚添加的变量赋值
    // object_setInstanceVariable(student, "schoolName", (void *)&str);在ARC下不允许使用
    [student setValue:@"清华大学" forKey:@"schoolName"];
    
    // KVC 动态改变实例变量
    [student setValue:@"Tom" forKey:@"name"];
    
    // 从类中获取成员变量Ivar
    Ivar ageIvar = class_getInstanceVariable(StudentClass, "_age");
    // 为peopleInstance的成员变量赋值
    object_setIvar(StudentClass, ageIvar, @18);
    
    // 调用 sayFunction 方法，也就是给 student 这个接受者发送 sayFunction: 这个消息
    objc_msgSend(student, "sayFunction:", @"你好~"); 
    // [student performSelector:sel withObject:@"你好~"]; // 动态调用未显式在类中声明的方法
    
    student = nil;
    StudentClass = nil;

//    objc_disposeClassPair(StudentClass);
}
```

直接使用 objc\_msgSend() 会报错 Too many arguments to function call, expected 0, have 3，此时需要在 Target -> Build Settings -> 搜索 msg -> 修改为 NO

![](https://upload-images.jianshu.io/upload_images/5294842-d8496fe34a524c20.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)


## 九、健壮的实例变量(Non Fragile ivars)

在 Runtime 的现行版本中，最大的特点就是健壮的实例变量。

当一个类被编译时，实例变量的布局也就形成了，它表明访问类的实例变量的位置。从对象头部开始，实例变量依次根据自己所占空间而产生位移：

<center>
![](https://upload-images.jianshu.io/upload_images/5294842-9b75736c89991ff2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

上图左边是 NSObject 类的实例变量布局，右边是我们写的类的布局，也就是在超类后面加上我们自己类的实例变量，看起来不错。但试想如果那天苹果更新了 NSObject 类，发布新版本的系统的话，那就悲剧了：

<center>
![](https://upload-images.jianshu.io/upload_images/5294842-0ddfa1eb170bae83.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

我们自定义的类被划了两道线，那是因为那块区域跟超类重叠了。唯有苹果将超类改为以前的布局才能拯救我们，但这样也导致它们不能再拓展它们的框架了，因为成员变量布局被死死地固定了。在脆弱的实例变量（Fragile ivars）环境下我们需要重新编译继承自 Apple 的类来恢复兼容性。那么在健壮的实例变量下会发生什么呢？

<center>
![健壮的实例变量自动偏移](https://upload-images.jianshu.io/upload_images/5294842-a4a0f676708f0a30.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

在健壮的实例变量下编译器生成的实例变量布局跟以前一样，但是当 runtime 系统检测到与超类有部分重叠时它会调整你新添加的实例变量的位移，那样你在子类中新添加的成员就被保护起来了。

需要注意的是在健壮的实例变量下，不要使用 sizeof(cls)，而是用 class\_getInstanceSize(cls) 代替；也不要使用 offsetof(cls, ivar)，而要用 ivar\_getOffset(class\_getInstanceVariable(cls, "ivar")) 来代替。


## 十、文章

[Mike_zh](http://home.cnblogs.com/u/Mike-zh/) & [iOS-Runtime知识点整理](https://www.cnblogs.com/Mike-zh/p/4557014.html)
[ian](https://www.ianisme.com/) & [Objective-C Runtime 1小时入门教程](https://www.ianisme.com/ios/2019.html)
[iOS开发-Runtime 详解](https://www.cnblogs.com/ioshe/p/5489086.html)
[iOS RunTime 之数据结构](https://www.jianshu.com/p/26c41f48267d)
[iOS 模块分解—「Runtime面试、工作」](https://www.jianshu.com/p/19f280afcb24)
[Runtime 源码](http://www.opensource.apple.com/source/objc4/)