---
title: isa 和 Class
categories: iOS原理
---


## 一、Runtime 简介

Runtime 又叫运行时，是一套底层的 C 语言 API，是 iOS 系统的核心之一。开发者在编码过程中，可以给任意一个对象发送消息，<font color=#cc0000>在编译阶段只是确定了要向接收者发送这条消息</font>，而接受者将要如何响应和处理这条消息，那就要由运行时来决定了。

C 语言中，函数的调用在编译期就会决定调用哪个函数。而 OC 的函数属于动态调用过程，在编译期并不能决定真正调用哪个函数，只有在真正运行时才会根据函数的名称找到对应的函数来调用。

Objective-C 是一个动态语言，不仅需要一个编译器，也需要一个运行时系统来动态得创建类和对象、进行消息传递和转发。

Objc 在三种层面上与 Runtime 系统进行交互：

![1](https://upload-images.jianshu.io/upload_images/5294842-b4ed02142f8f97fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1、通过 Objective-C 源代码

一般情况开发者只需要编写 OC 代码即可，Runtime 系统自动在幕后把我们写的源代码在编译阶段转换成运行时代码，在运行时确定对应的数据结构和调用具体哪个方法。

2、通过 Foundation 框架的 NSObject 类定义的方法

在 OC 中，除了 NSProxy 类以外，所有的类都是 NSObject 的子类。在 Foundation 框架下，NSObject 和 NSProxy 两个基类，定义了类层次结构中该类下方所有类的公共接口和行为。NSProxy 是专门用于实现代理对象的类，暂且不提。这两个类都遵循了 NSObject 协议。在 NSObject 协议中，声明了所有 OC 对象的公共方法。

在 NSObject 协议中，有以下 5 个方法是可以从 Runtime 中获取信息，让对象进行自我检查。

```objc
/**
 * 返回对象的类
 */
- (Class)class OBJC_SWIFT_UNAVAILABLE("use 'anObject.dynamicType' instead");

/**
 * 检查对象是否存在于指定类的继承体系中，是否是为某个类或它的子类
 */
- (BOOL)isKindOfClass:(Class)aClass;

/**
 * 检查对象是否是某个类的实例
 */
- (BOOL)isMemberOfClass:(Class)aClass;

/**
 * 检查对象能否响应指定的消息
 */
- (BOOL)conformsToProtocol:(Protocol *)aProtocol;

/**
 * 检查对象是否实现了指定协议类的方法
 */
- (BOOL)respondsToSelector:(SEL)aSelector;
```

在 NSObject 的类中还定义了一个方法

```objc
/**
 * 返回指定方法实现的地址 IMP
 */
- (IMP)methodForSelector:(SEL)aSelector;
```

3、通过对 Runtime 库函数的直接调用

关于库函数可以在 [Objective-C Runtime Reference](https://developer.apple.com/documentation/objectivec/objective-c_runtime?language=objc) 中查看 Runtime 函数的详细文档。

关于这一点，其实还有一个小插曲。当我们导入了 objc/Runtime.h 和 objc/message.h 两个头文件之后，我们查找到了Runtime 的函数之后，写代码时发现没有代码提示，那些函数里面的参数和描述都没有了。对于熟悉 Runtime 的开发者来说，这并没有什么难的，因为参数早已铭记于胸。但是对于新手来说，这是相当不友好的。而且，如果是从 iOS6 开始开发的同学，依稀可能能感受到，关于 Runtime 的具体实现的官方文档越来越少了？可能还怀疑是不是错觉。其实从 Xcode5 开始，苹果就不建议开发者手动调用 Runtime 的 API，也同样希望我们不要知道具体底层实现。所以 IDE 上面默认带了一个参数，禁止了 Runtime 的代码提示，源码和文档方面也删除了一些解释。

具体设置如下：

![](https://upload-images.jianshu.io/upload_images/1194012-4a2ea408888ae8cd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/811)

如果发现导入了两个库文件之后，仍然没有代码提示，就需要把这里的设置改成 NO，即可。


## 二、NSObject 起源

与 Runtime 交互有 3 种方式，前两种方式都与 NSObject 有关，那我们就从 NSObject 基类开始说起。以下源码分析均来自[objc4-680](https://link.jianshu.com?t=http://opensource.apple.com//source/objc4/)

NSObject 的定义如下：

```objc
typedef struct objc_class *Class;

@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}
```

在 Objc2.0 之前，objc\_class 源码如下：

```objc
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;
    
#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;  // 指向成员变量列表的指针
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;  // 指向方法列表指针的指针
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif
    
} OBJC2_UNAVAILABLE;
```

在这里可以看到，在一个类中，有超类的指针、类名、版本的信息。

动态修改 *methodLists 的值来添加成员方法，这也是 Category 实现的原理，同样解释了 Category 不能添加属性的原因。

关于 Category，推荐 2 篇文章可以仔细研读：[深入理解Objective-C：Category](https://link.jianshu.com?t=http://tech.meituan.com/DiveIntoCategory.html)、[结合 Category 工作原理分析 OC2.0 中的 runtime](https://link.jianshu.com?t=https://bestswifter.com/jie-he-category-gong-zuo-yuan-li-fen-xi-oc2-0-zhong-de-runtime/)

然后在 2006 年苹果发布 Objc 2.0 之后，objc\_class 的定义就变成下面这个样子了，源码 [objc_private](https://opensource.apple.com//source/objc4/objc4-680/runtime/objc-private.h.auto.html)。

```objc
typedef struct objc_class *Class;
typedef struct objc_object *id;

@interface Object { 
    Class isa; 
}

@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}

struct objc_object {
private:
    isa_t isa;
}

struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
}

union isa_t 
{

}
```

![](https://upload-images.jianshu.io/upload_images/1194012-06a854913380136c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/956)

把源码的定义转化成类图，就是上图的样子。

从上述源码中，我们可以看到，<font color=#cc0000>Objective-C 对象都是 C 语言结构体实现的</font>，在 objc2.0 中，所有的对象都会包含一个 isa\_t 类型的结构体。

objc\_object 被源码 typedef 成了 id 类型，这也就是我们平时遇到的 id 类型。这个结构体中就只包含了一个 isa\_t 类型的结构体。这个结构体在下面会详细分析。

objc\_class 继承于 objc\_object。所以在 objc\_class 中也会包含 isa\_t 类型的结构体 isa。至此，可以得出结论：

> <font color=#cc0000>Objective-C 中类也是一个对象</font>。在 objc\_class 中，除了 isa 之外，还有 3 个成员变量，一个是父类的指针，一个是方法缓存，最后一个是这个类的实例方法链表。

object 类和 NSObject 类里面分别都包含一个 objc\_class 类型的 isa。

#### 2.1 isa

```objc
union isa_t 
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }
    Class cls;
    uintptr_t bits;

# if __arm64__
    struct {
        uintptr_t indexed           : 1;  // 是否开启 isa 指针优化。index = 1 表示开启 isa 指针优化  
        uintptr_t has_assoc         : 1;  // 是否有设置过关联对象，如果没有，释放时会更快
        uintptr_t has_cxx_dtor      : 1;  // 是否有 C++ 的析构函数（.cxx_destruct），如果没有，释放时会更快
        uintptr_t shiftcls          : 33; // MACH_VM_MAX_ADDRESS 0x1000000000 存储着Class、Meta-Class对象的内存地址信息
        uintptr_t magic             : 6;  // 用于在调试时分辨对象是否未完成初始化
        uintptr_t weakly_referenced : 1;  // 是否有被弱引用指向过，如果没有，释放时会更快
        uintptr_t deallocating      : 1;  // 对象是否正在释放
        uintptr_t has_sidetable_rc  : 1;  // 引用计数器是否过大无法存储在 isa 中。如果为 1，那么引用计数会存储在一个叫 SideTable 的类的属性中
        uintptr_t extra_rc          : 19; // 里面存储的值是引用计数 - 1
    };

# elif __x86_64__
    struct {
        uintptr_t indexed           : 1;
        uintptr_t has_assoc         : 1;  
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 44; // MACH_VM_MAX_ADDRESS 0x7fffffe00000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 8;
    };

# else

# endif
}
```

[isa 详解](https://blog.csdn.net/u012581760/article/details/81230721)

在 arm64 之前 isa 就是普通的指针，只存储类对象、元类对象的指针。但是 arm64 之后 isa 做了优化，采取了共用体结构，将一个 <font color=#cc0000>64</font> 位的内存数据分开存储了很多东西，其中 33 位用来存储地址值。

当一个对象的实例方法被调用的时候，会通过 isa 找到相应的类，然后在该类的 class\_data\_bits\_t 中去查找方法。class\_data\_bits\_t 是指向了类对象的数据区域，在该数据区域内查找相应方法的对应实现。

但是在我们调用类方法的时候，类对象的 isa 里面是什么呢？这里为了和对象查找方法的机制一致，遂引入了元类（meta-class）的概念。关于元类，更多具体可以研究这篇文章 [What is a meta-class in Objective-C?](https://link.jianshu.com?t=http://www.cocoawithlove.com/2010/01/what-is-meta-class-in-objective-c.html)

在引入元类之后，类对象和对象查找方法的机制就完全统一了。

> 对象的实例方法调用时，通过对象的 isa 在类中获取方法的实现。
> 
> 类对象的类方法调用时，通过类的 isa 在元类中获取方法的实现。

meta-class 之所以重要，是因为它存储着一个类的所有类方法。每个类都会有一个单独的 meta-class，因为每个类的类方法基本不可能完全相同。

对应关系的图如下图，下图很好的描述了对象，类，元类之间的关系：

![](https://upload-images.jianshu.io/upload_images/1194012-d7b097e86f9e488d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/625)

实线是 super\_class 指针，虚线是 isa 指针。

1.  Root class（class） 其实就是 NSObject，NSObject 是没有超类的，所以 Root class（class）的 superclass 指向 nil。
    
2.  每个 Class 都有一个 isa 指针指向唯一的 Meta class
    
3.  Root class（meta）的 superclass 指向 Root class（class），也就是 NSObject，形成一个回路。
    
4.  每个 Meta class 的 isa 指针都指向 Root class（meta）。
    

我们其实应该明白，类对象和元类对象是唯一的，对象是可以在运行时创建无数个的。而在 main 方法执行之前，从 dyld 到 runtime 这期间，类对象和元类对象在这期间被创建。具体可看 sunnyxx 这篇 [iOS 程序 main 函数之前发生了什么](https://link.jianshu.com?t=http://blog.sunnyxx.com/2014/08/30/objc-pre-main/)


###### 2.1.1 isa\_t 结构体的具体实现

接下来我们就该研究研究 isa 的具体实现了。objc\_object 里面的 isa 是 isa\_t 类型。通过查看源码，我们可以知道 isa\_t 是一个 union 联合体。

```objc
struct objc_object {
private:
    isa_t isa;
public:
    // initIsa() should be used to init the isa of new objects only.
    // If this object already has an isa, use changeIsa() for correctness.
    // initInstanceIsa(): objects with no custom RR/AWZ
    void initIsa(Class cls /*indexed=false*/);
    void initInstanceIsa(Class cls, bool hasCxxDtor);
private:
    void initIsa(Class newCls, bool indexed, bool hasCxxDtor);
    ...
｝
```

那就从 initIsa 方法开始研究。下面以 arm64 为例，源码 [objc_object](https://opensource.apple.com//source/objc4/objc4-680/runtime/objc-object.h.auto.html)。

```objc
inline void
objc_object::initInstanceIsa(Class cls, bool hasCxxDtor)
{
    initIsa(cls, true, hasCxxDtor);
}

inline void
objc_object::initIsa(Class cls, bool indexed, bool hasCxxDtor)
{
    if (!indexed) {
        isa.cls = cls;
    } else {
        isa.bits = ISA_MAGIC_VALUE;
        isa.has_cxx_dtor = hasCxxDtor;
        isa.shiftcls = (uintptr_t)cls >> 3;
    }
}
```

initIsa 第二个参数传入了一个 true，所以 initIsa 就会执行 else 里面的语句。

```objc
# if __arm64__
#   define ISA_MASK        0x0000000ffffffff8ULL
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL
    struct {
        uintptr_t indexed           : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 33; // MACH_VM_MAX_ADDRESS 0x1000000000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 19;
#       define RC_ONE   (1ULL<<45)
#       define RC_HALF  (1ULL<<18)
    };

# elif __x86_64__
#   define ISA_MASK        0x00007ffffffffff8ULL
#   define ISA_MAGIC_MASK  0x001f800000000001ULL
#   define ISA_MAGIC_VALUE 0x001d800000000001ULL
    struct {
        uintptr_t indexed           : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 44; // MACH_VM_MAX_ADDRESS 0x7fffffe00000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 8;
#       define RC_ONE   (1ULL<<56)
#       define RC_HALF  (1ULL<<7)
    };
```

![](https://upload-images.jianshu.io/upload_images/1194012-2f2760cc2bc4034e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)

ISA\_MAGIC\_VALUE = 0x000001a000000001ULL 转换成二进制是 11010000000000000000000000000000000000001，结构如下图：

![6](https://upload-images.jianshu.io/upload_images/1194012-78ff71b4e40f616f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/829)

参数的说明：

* index

> 代表是否开启 isa 指针优化。index = 1 代表开启 isa 指针优化。

在 2013 年 9 月，苹果推出了 iPhone5s，与此同时，iPhone5s 配备了首个采用 64 位架构的 A7 双核处理器，为了节省内存和提高执行效率，苹果提出了 Tagged Pointer 的概念。对于 64 位程序，引入 Tagged Pointer 后，相关逻辑能减少一半的内存占用，以及 3 倍的访问速度提升，100 倍的创建、销毁速度提升。

在 WWDC2013 的《Session 404 Advanced in Objective-C》视频中，苹果介绍了 Tagged Pointer。

<font color=#cc0000>Tagged Pointer 的存在主要是为了节省内存</font>。我们知道，<font color=#cc0000>对象的指针大小一般是与机器字长有关</font>，在 32 位系统中，一个指针的大小是 32 位（4 字节），而在 64 位系统中，一个指针的大小将是 64 位（8 字节）。

假设我们要存储一个 NSNumber 对象，其值是一个整数。正常情况下，如果这个整数只是一个 NSInteger 的普通变量，那么它所占用的内存是与 CPU 的位数有关，在 32 位 CPU 下占 4 个字节，在 64 位 CPU 下是占 8 个字节的。而指针类型的大小通常也是与 CPU 位数相关，一个指针所占用的内存在 32 位 CPU 下为 4 个字节，在 64 位 CPU 下也是 8 个字节。如果没有 Tagged Pointer 对象，从 32 位机器迁移到 64 位机器中后，虽然逻辑没有任何变化，但这种 NSNumber、NSDate 一类的对象所占用的内存会翻倍。如下图所示：

![](https://upload-images.jianshu.io/upload_images/5294842-1f205ac1ee6d1db9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

苹果提出了 Tagged Pointer 对象。由于 NSNumber、NSDate 一类的变量本身的值需要占用的内存大小常常不需要 8 个字节，拿整数来说，4 个字节所能表示的有符号整数就可以达到 20 多亿（注：231 = 2147483648，另外 1 位作为符号位)，对于绝大多数情况都是可以处理的。所以，引入了 Tagged Pointer 对象之后，64 位 CPU 下 NSNumber 的内存图变成了以下这样：

![](https://upload-images.jianshu.io/upload_images/5294842-c1a948684d801b06.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

关于 [Tagged Pointer 技术](http://www.infoq.com/cn/articles/deep-understanding-of-tagged-pointer/)详细的，可以看上面链接那个文章。

* has\_assoc

对象含有或者曾经含有关联引用，没有关联引用的可以更快地释放内存

* has\_cxx\_dtor

表示该对象是否有 C++ 或者 Objc 的析构器

* shiftcls

类的指针。arm64 架构中有 33 位可以存储类指针。

源码中 isa.shiftcls = (uintptr\_t)cls >> 3;

将当前地址右移三位的主要原因是用于<font color=#cc0000>将 Class 指针中无用的后三位清除减小内存的消耗</font>，因为类的指针要按照字节（8 bits）对齐内存，其指针后三位都是没有意义的 0。具体可以看[从 NSObject 的初始化了解 isa](https://link.jianshu.com?t=https://github.com/Draveness/iOS-Source-Code-Analyze/blob/master/contents/objc/%E4%BB%8E%20NSObject%20%E7%9A%84%E5%88%9D%E5%A7%8B%E5%8C%96%E4%BA%86%E8%A7%A3%20isa.md#shiftcls)这篇文章里面的 shiftcls 分析。

* magic

判断对象是否初始化完成，在 arm64 中 0x16 是调试器判断当前对象是真的对象还是没有初始化的空间。

* weakly\_referenced

对象被指向或者曾经指向一个 ARC 的弱变量，没有弱引用的对象可以更快释放

* deallocating

对象是否正在释放内存

* has\_sidetable\_rc

判断该对象的引用计数是否过大，如果过大则需要其他散列表来进行存储。

* extra\_rc

存放该对象的引用计数值减一后的结果。对象的引用计数超过 1，会存在这个里面，如果引用计数为 10，extra\_rc 的值就为 9。

ISA\_MAGIC\_MASK 和 ISA\_MASK 分别是通过掩码的方式获取 MAGIC 值和 isa 类指针。

```objc
inline Class 
objc_object::ISA() 
{
    assert(!isTaggedPointer()); 
    return (Class)(isa.bits & ISA_MASK);
}
```

关于 x86\_64 的架构，具体可以看[从 NSObject 的初始化了解 isa](https://link.jianshu.com?t=https://github.com/Draveness/iOS-Source-Code-Analyze/blob/master/contents/objc/%E4%BB%8E%20NSObject%20%E7%9A%84%E5%88%9D%E5%A7%8B%E5%8C%96%E4%BA%86%E8%A7%A3%20isa.md)文章里面的详细分析。


#### 2.2 cache\_t 的具体实现

继续看源码

```objc
struct cache_t {
    struct bucket_t *_buckets;
    mask_t _mask;      // 分配用来缓存 bucket 的总数
    mask_t _occupied;  // 表明实际占用的缓存 bucket 的个数
}

typedef unsigned int uint32_t;
typedef uint32_t mask_t;  // x86_64 & arm64 asm are less efficient with 16-bits

typedef unsigned long  uintptr_t;
typedef uintptr_t cache_key_t;

struct bucket_t {
private:
    cache_key_t _key;
    IMP _imp;
}
```

![](https://upload-images.jianshu.io/upload_images/1194012-3ab871ca22e8e5a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/492)

根据源码，我们可以知道 cache\_t 中存储了一个 bucket\_t 的结构体，和两个 unsigned int 的变量。

bucket\_t 的结构体中存储了一个 unsigned long 和一个 IMP。IMP 是一个函数指针，指向了一个方法的具体实现。

cache\_t 中的 bucket\_t *\_buckets 其实就是一个散列表，用来存储 Method 的链表。

Cache 的作用主要是为了优化方法调用的性能。当对象 receiver 调用方法 message 时，首先根据对象 receiver 的 isa 指针查找到它对应的类，然后在类的 methodLists 中搜索方法，如果没有找到，就使用 super\_class 指针到父类中的 methodLists 查找，一旦找到就调用方法。如果没有找到，有可能消息转发，也可能忽略它。但这样查找方式效率太低，<font color=#cc0000>因为往往一个类大概只有 20% 的方法经常被调用，占总调用次数的 80%</font>。所以使用 Cache 来缓存经常调用的方法，当调用方法时，优先在 Cache 查找，如果没有找到，再到 methodLists 查找。

#### 2.3 class\_data\_bits\_t 的具体实现

源码实现：

```objc
struct class_data_bits_t {

    // Values are the FAST_ flags above.
    uintptr_t bits;
}

struct class_rw_t {
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;

    Class firstSubclass;
    Class nextSiblingClass;

    char *demangledName;
}

struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;

    method_list_t *baseMethods() const {
        return baseMethodList;
    }
};
```

![](https://upload-images.jianshu.io/upload_images/5294842-dfecb7c37d335fc6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在 objc\_class 结构体中的注释写到 class\_data\_bits\_t 相当于 class\_rw\_t 指针加上 rr/alloc 的标志。

```objc
class_data_bits_t bits; // class_rw_t * plus custom rr/alloc flags
```

它为我们提供了便捷方法用于返回其中的 class\_rw\_t * 指针：

```objc
class_rw_t *data() {
    return bits.data();
}
```

Objc 的类的属性、方法、以及遵循的协议在 obj 2.0 的版本之后都放在 class\_rw\_t 中。class\_ro\_t 是一个指向常量的指针，存储来编译器决定了的属性、方法和遵守协议。rw-readwrite、ro-readonly

在编译期，类的结构中的 class\_data\_bits\_t *data 指向的是一个 class\_ro\_t * 指针：

![](https://upload-images.jianshu.io/upload_images/5294842-2dad1ac70ec7dac6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在运行时调用 realizeClass方法，会做以下 3 件事情：

1.  从 class\_data\_bits\_t 调用 data 方法，将结果从 class\_rw\_t 强制转换为 class\_ro\_t 指针
    
2.  初始化一个 class\_rw\_t 结构体
    
3.  设置结构体 ro 的值以及 flag
    

最后调用 methodizeClass 方法，把类里面的属性、协议、方法都加载进来。

```objc
struct method_t {
    SEL name;   // 方法名字
    const char *types;  // Type Encoding 类型编码
    IMP imp;  

    struct SortBySELAddress :
        public std::binary_function<const method_t&,
                                    const method_t&, bool>
    {
        bool operator() (const method_t& lhs,
                         const method_t& rhs)
        { return lhs.name < rhs.name; }
    };
};
```

方法 method 的定义如上。里面包含 3 个成员变量。Type Encoding 类型编码可参考 [Type Encoding](https://link.jianshu.com?t=https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)。

IMP 是一个函数指针，指向的是函数的具体实现。在 runtime 中消息传递和转发的目的就是为了找到 IMP，并执行函数。

整个运行时过程描述如下：

![](https://upload-images.jianshu.io/upload_images/5294842-06da58b9bbe05c6f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

更加详细的分析，请看[@Draveness](https://link.jianshu.com?t=https://github.com/Draveness) 的这篇文章[深入解析 ObjC 中方法的结构](https://link.jianshu.com?t=https://github.com/Draveness/iOS-Source-Code-Analyze/blob/master/contents/objc/%E6%B7%B1%E5%85%A5%E8%A7%A3%E6%9E%90%20ObjC%20%E4%B8%AD%E6%96%B9%E6%B3%95%E7%9A%84%E7%BB%93%E6%9E%84.md#%E6%B7%B1%E5%85%A5%E8%A7%A3%E6%9E%90-objc-%E4%B8%AD%E6%96%B9%E6%B3%95%E7%9A%84%E7%BB%93%E6%9E%84)

到此，总结 objc\_class 1.0 和 2.0 的差别。

![](https://upload-images.jianshu.io/upload_images/1194012-8b2987b38e6e5d2b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)

![](https://upload-images.jianshu.io/upload_images/1194012-cd2c3afd17d40e9f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)


## 三、测试题

1. [self class] 与 [super class]

```objc
@implementation Son : Father

- (id)init
{
    if (self = [super init])
    {
        NSLog(@"%@", NSStringFromClass([self class]));
        NSLog(@"%@", NSStringFromClass([super class]));
    }
    return self;
}
@end
```

self 和 super 的区别：

    self 是类的一个<font color=#cc0000>隐藏参数</font>，每个方法的实现的第一个参数即为 self。

　 super 并不是隐藏参数，它实际上只是一个“<font color=#cc0000>编译器标示符</font>”，它负责告诉编译器：当调用方法时，去调用父类的方法，而不是本类中的方法。

在调用 [super class] 的时候，runtime 会去调用 <font color=#cc0000>objc\_msgSendSuper</font> 方法，而不是 objc\_msgSend。

```objc
OBJC_EXPORT void objc_msgSendSuper(void /* struct objc_super *super, SEL op, ... */ )


/// Specifies the superclass of an instance. 
struct objc_super {
    /// Specifies an instance of a class.
    __unsafe_unretained id receiver;

    /// Specifies the particular superclass of the instance to message. 
#if !defined(__cplusplus)  &&  !__OBJC2__
    /* For compatibility with old objc-runtime.h header */
    __unsafe_unretained Class class;
#else
    __unsafe_unretained Class super_class;
#endif
    /* super_class is the first class to search */
};
```

在 objc\_msgSendSuper 方法中，第一个参数是一个 objc\_super 的结构体，这个结构体里面有两个变量，一个是接收消息的receiver，一个是当前类的父类 super\_class。

objc\_msgSendSuper 的工作原理应该是这样的：

> 从 objc\_super 结构体指向的 superClass 父类的方法列表开始查找 selector，找到后以 objc-receiver 去调用父类的这个 selector。注意，最后的调用者是 objc->receiver，而不是 super\_class。

那么 objc\_msgSendSuper 最后就转变成

```objc
// 注意这里是从父类开始 msgSend，而不是从本类开始。
objc_msgSend(objc_super->receiver, @selector(class))

/// Specifies an instance of a class.  这是类的一个实例
    __unsafe_unretained id receiver;   


// 由于是实例调用，所以是减号方法
- (Class)class {
    return object_getClass(self);
}
```

由于找到了父类 NSObject 里面的 class 方法的 IMP，又因为传入的入参 objc\_super->receiver = self。self 就是 son，调用 class，所以父类的方法 class 执行 IMP 之后，输出还是 son，最后输出两个都一样，都是输出 son。


2. isKindOfClass 与 isMemberOfClass

```objc
@interface Sark : NSObject
@end

@implementation Sark@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
         BOOL res1 = [(id)[NSObject class] isKindOfClass:[NSObject class]];
         BOOL res2 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];
         BOOL res3 = [(id)[Sark class] isKindOfClass:[Sark class]];
         BOOL res4 = [(id)[Sark class] isMemberOfClass:[Sark class]];

         NSLog(@"%d %d %d %d", res1, res2, res3, res4);
    }
    return 0;
}
```

先来分析一下源码这两个函数的对象实现

```objc
+ (Class)class {
    return self;
}

- (Class)class {
    return object_getClass(self);
}

Class object_getClass(id obj)
{
    if (obj) return obj->getIsa();
    else return Nil;
}

inline Class 
objc_object::getIsa() 
{
    if (isTaggedPointer()) {
        uintptr_t slot = ((uintptr_t)this >> TAG_SLOT_SHIFT) & TAG_SLOT_MASK;
        return objc_tag_classes[slot];
    }
    return ISA();
}

inline Class 
objc_object::ISA() 
{
    assert(!isTaggedPointer()); 
    return (Class)(isa.bits & ISA_MASK);
}

+ (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = object_getClass((id)self); tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}

- (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}

+ (BOOL)isMemberOfClass:(Class)cls {
    return object_getClass((id)self) == cls;
}

- (BOOL)isMemberOfClass:(Class)cls {
    return [self class] == cls;
}
```

首先题目中 NSObject 和 Sark 分别调用了 class 方法。

\+ (BOOL)isKindOfClass:(Class)cls 方法内部，会先去获得 object\_getClass 的类，而 object\_getClass 的源码实现是去<font color=#cc0000>调用当前类的 obj->getIsa()</font>，最后在 <font color=#cc0000>ISA()</font> 方法中获得 meta class 的指针。

接着在 isKindOfClass 中有一个<font color=#cc0000>循环</font>，先判断 class 是否等于 meta class，不等就继续循环判断是否等于 super class，不等再继续取 super class，如此循环下去。

[NSObject class] 执行完之后调用 isKindOfClass，第一次判断先判断 NSObject 和 NSObject 的 meta class 是否相等，之前讲到 meta class 的时候放了一张很详细的图，从图上我们也可以看出，NSObject 的 meta class 与本身不等。接着第二次循环判断 NSObject 与 meta class 的 superclass 是否相等。还是从那张图上面我们可以看到：Root class(meta) 的 superclass 就是 Root class（class），也就是 NSObject 本身。所以第二次循环相等，于是第一行 res1 输出应该为YES。

同理，[Sark class] 执行完之后调用 isKindOfClass，第一次 for 循环，Sark 的 Meta Class 与 [Sark class] 不等，第二次 for 循环，Sark Meta Class 的 super class 指向的是 NSObject Meta Class，和 Sark Class 不相等。第三次 for 循环，NSObject Meta Class 的 super class 指向的是 NSObject Class，和 Sark Class 不相等。第四次循环，NSObject Class 的 super class 指向 nil， 和 Sark Class 不相等。第四次循环之后，退出循环，所以第三行的 res3 输出为 NO。

如果把这里的 Sark 改成它的实例对象，[sark isKindOfClass:[Sark class]]，那么此时就应该输出 YES 了。因为在 isKindOfClass 函数中，判断 sark 的 isa 指向是否是自己的类 Sark，第一次 for 循环就能输出 YES 了。

> isMemberOfClass 的源码实现是拿到自己的 isa 指针和自己比较，是否相等。

第二行 isa 指向 NSObject 的 Meta Class，所以和 NSObject Class 不相等。第四行，isa 指向 Sark 的 Meta Class，和 Sark Class 也不等，所以第二行 res2 和第四行 res4 都输出 NO。


3. Class 与内存地址

下面的代码会？Compile Error / Runtime Crash / NSLog...?

```objc
@interface Sark : NSObject
@property (nonatomic, copy) NSString *name;
- (void)speak;
@end

@implementation Sark
- (void)speak {
    NSLog(@"my name's %@", self.name);
}
@end

@implementation ViewController

- (void)viewDidLoad
{
    [super viewDidLoad];

    id cls = [Sark class];
    void *obj = &cls;
    [(__bridge id)obj speak];
}
@end
```

这道题有两个难点。难点一，obj 调用 speak 方法到底会不会崩溃。难点二，如果 speak 方法不崩溃，应该输出什么？

首先需要谈谈隐藏参数 self 和 \_cmd 的问题。

当 [receiver message] 调用方法时，系统会在运行时偷偷地动态传入两个隐藏参数 self 和 \_cmd，之所以称它们为隐藏参数，是因为在源代码中没有声明和定义这两个参数。self 在上面已经讲解明白了，接下来就来说说 \_cmd。\_cmd 表示当前调用方法，其实它就是一个方法选择器 SEL。

难点一：能不能调用 speak 方法？

```objc
id cls = [Sark class]; 
void *obj = &cls;
```

答案是可以的。obj 被转换成了一个指向 Sark Class 的指针，然后使用 id 转换成了 objc\_object 类型。obj 现在已经是一个 Sark 类型的实例对象了。当然接下来可以调用 speak 的方法。

难点二：如果能调用 speak，会输出什么呢？

很多人可能会认为会输出 sark 相关的信息。这样答案就错误了。

正确的答案会输出

```objc
my name is <ViewController: 0x7ff6d9f31c50>
```

内存地址每次运行都不同，但是前面一定是 ViewController。why？

我们把代码改变一下，打印更多的信息出来。

```objc
- (void)viewDidLoad 
{
    [super viewDidLoad];
    
    NSLog(@"ViewController = %@ , 地址 = %p", self, &self);
    
    id cls = [Sark class];
    NSLog(@"Sark class = %@ 地址 = %p", cls, &cls);
    
    void *obj = &cls;
    NSLog(@"Void *obj = %@ 地址 = %p", obj, &obj);
    
    [(__bridge id)obj speak];
    
    Sark *sark = [[Sark alloc]init];
    NSLog(@"Sark instance = %@ 地址 = %p",sark, &sark);
    
    [sark speak];
}
```

我们把对象的指针地址都打印出来。输出结果：

```objc
ViewController = <ViewController: 0x7fb570e2ad00> , 地址 = 0x7fff543f5aa8
Sark class = Sark 地址 = 0x7fff543f5a88
Void *obj = <Sark: 0x7fff543f5a88> 地址 = 0x7fff543f5a80

my name is <ViewController: 0x7fb570e2ad00>

Sark instance = <Sark: 0x7fb570d20b10> 地址 = 0x7fff543f5a78
my name is (null)
```

![](//upload-images.jianshu.io/upload_images/1194012-c794987c90515f8e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)

```objc
// objc_msgSendSuper2() takes the current search class, not its superclass.
OBJC_EXPORT id objc_msgSendSuper2(struct objc_super *super, SEL op, ...)
    __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_2_0);
```

objc\_msgSendSuper2 方法入参是一个 objc\_super *super。

```objc
/// Specifies the superclass of an instance. 
struct objc_super {
    /// Specifies an instance of a class.
    __unsafe_unretained id receiver;

    /// Specifies the particular superclass of the instance to message. 
#if !defined(__cplusplus)  &&  !__OBJC2__
    /* For compatibility with old objc-runtime.h header */
    __unsafe_unretained Class class;
#else
    __unsafe_unretained Class super_class;
#endif
    /* super_class is the first class to search */
};
#endif
```

所以按 viewDidLoad 执行时各个变量入栈顺序从高到底为 self、\_cmd、super\_class（等同于 self.class）、receiver（等同于 self）、obj。

![](https://upload-images.jianshu.io/upload_images/5294842-75370b4b3f3e6c04.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第一个 self 和第二个 \_cmd 是隐藏参数。第三个 self.class 和第四个 self 是 [super viewDidLoad] 方法执行时候的参数。

在调用 self.name 的时候，本质上是 self 指针在内存向高位地址偏移一个指针。

![](https://upload-images.jianshu.io/upload_images/5294842-1de5156e2caa715e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从打印结果我们可以看到，obj 就是 cls 的地址。在 obj 向上偏移一个指针就到了 0x7fff543f5a90，这正好是 ViewController 的地址。所以输出为 my name is <ViewController: 0x7fb570e2ad00>。

至此，Objc 中的对象到底是什么呢？

实质：<font color=#cc0000>Objc 中的对象是一个指向 ClassObject 地址的变量，即 id obj = &ClassObject**，**而对象的实例变量 void *ivar = &obj + offset(N)</font>

加深一下对上面这句话的理解，下面这段代码会输出什么？

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    NSLog(@"ViewController = %@ , 地址 = %p", self, &self);
    
    NSString *myName = @"halfrost";
    
    id cls = [Sark class];
    NSLog(@"Sark class = %@ 地址 = %p", cls, &cls);
    
    void *obj = &cls;
    NSLog(@"Void *obj = %@ 地址 = %p", obj,&obj);
    
    [(__bridge id)obj speak];
    
    Sark *sark = [[Sark alloc]init];
    NSLog(@"Sark instance = %@ 地址 = %p",sark,&sark);
    
    [sark speak];
    
}


ViewController = <ViewController: 0x7fff44404ab0> ,  地址  = 0x7fff56a48a78
Sark class = Sark  地址  = 0x7fff56a48a50
Void *obj = <Sark: 0x7fff56a48a50>  地址 = 0x7fff56a48a48

my name is halfrost

Sark instance = <Sark: 0x6080000233e0>  地址 = 0x7fff56a48a40
my name is (null)
```

由于加了一个字符串，结果输出就完全变了，[(\_\_bridge id)obj speak]; 这句话会输出“my name is halfrost”。

原因还是和上面的类似。按 viewDidLoad 执行时各个变量入栈顺序从高到底为 self、\_cmd、self.class（super\_class）、self（receiver）、myName、obj。obj 往上偏移一个指针，就是 myName 字符串，所以输出变成了输出 myName 了。

![](https://upload-images.jianshu.io/upload_images/5294842-86a075b8fd3adf92.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里有一点需要额外说明的是，栈里面有两个 self，可能有些人认为是指针偏移到了第一个 self 了，于是打印出了 ViewController：

```objc
my name is <ViewController: 0x7fb570e2ad00>
```

![](https://upload-images.jianshu.io/upload_images/5294842-428635ce01f15146.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其实这种想法是不对的，从 obj 往上找 name 属性，完全是指针偏移了一个 offset 导致的，也就是说指针只往下偏移了一个。那么怎么证明指针只偏移了一个，而不是偏移了 4 个到最下面的 self 呢？

![](https://upload-images.jianshu.io/upload_images/1194012-cccbecc99506dbe5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)

obj 的地址是 0x7fff5c7b9a08，self 的地址是 0x7fff5c7b9a28。每个指针占 8 个字节，所以从 obj 到 self 中间确实有 4 个指针大小的间隔。如果从 obj 偏移一个指针，就到了 0x7fff5c7b9a10。我们需要把这个内存地址里面的内容打印出来。

LLDB 调试中，可以使用 examine 命令（简写是 x）来查看内存地址中的值。x 命令的语法如下所示：

> x/
> 
> n、f、u 是可选的参数。
> 
> n 是一个正整数，表示显示内存的长度，也就是说从当前地址向后显示几个地址的内容。
> 
> f 表示显示的格式，参见上面。如果地址所指的是字符串，那么格式可以是 s，如果是指令地址，那么格式可以是 i。
> 
> u 表示从当前地址往后请求的字节数，如果不指定的话，GDB 默认是 4 个 bytes。
> 
> u 参数可以用下面的字符来代替，b 表示单字节，h 表示双字节，w 表示四字节，g 表示八字节。当我们指定了字节长度后，GDB 会从指内存定的内存地址开始，读写指定字节，并把其当作一个值取出来。

![](https://upload-images.jianshu.io/upload_images/1194012-3111309aaef61c73.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)

我们用 x 命令分别打印出 0x7fff5c7b9a10 和 0x7fff5c7b9a28 内存地址里面的内容，我们会发现两个打印出来的值是一样的，都是 0x7fbf0d606aa0。

这两个 self 的地址不同，里面存储的内容是相同的。所以 obj 是偏移了一个指针，而不是偏移到最下面的 self。


## 四、文章

[一缕殇流化隐半边冰霜](https://www.jianshu.com/u/12201cdd5d7a) & [神经病院Objective-C Runtime入院第一天--isa和Class](https://www.jianshu.com/p/9d649ce6d0b8)
