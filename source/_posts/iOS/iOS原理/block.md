---
title: Block
categories: iOS原理
---

## 一、什么是闭包

在 wikipedia 上，闭包的定义是:

> In programming languages, a closure is a function or reference to a function together with a referencing environment—a table storing a reference to each of the non-local variables (also called free variables or upvalues) of that function.

翻译过来，闭包是一个函数（或指向函数的指针），再加上该函数执行的外部的上下文变量（有时候也称作自由变量）。

block 实际上就是 Objective-C 语言对于闭包的实现。 block 配合上 dispatch\_queue，可以方便地实现简单的多线程编程和异步编程，[《使用GCD》](http://blog.devtang.com/blog/2012/02/22/use-gcd/)。

本文主要介绍 Objective-C 语言的 block 在编译器中的实现方式。主要包括：

1.  block 的内部实现数据结构介绍
2.  block 的三种类型及其相关的内存管理方式
3.  block 如何通过 capture 变量来达到访问函数外的变量

## 二、实现方式

1.  block 本身也是一个 OC 对象，它里面也有 isa 指针
2.  block 是封装了函数调用（存储函数调用地址，函数访问变量）和函数调用环境的 OC 对象

在 main.m 中写入一个 block：  

```
int main(int argc, char * argv[]) {
    @autoreleasepool {

        int a = 15;
        void (^ block)(int, int) = ^ (int b, int c) {
            NSLog(@"%d", a);
        };
        block(10, 10);
        
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

终端进到项目 main.m 的目录下通过反编译成 c++ 文件：

```
xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m -o mian.cpp
```

 得到 main.cpp，找到这个 block 对象的底层结构：

```
int main(int argc, char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        int a = 15;
        void (* block)(int, int) = ((void (*)(int, int))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, a));
        ((void (*)(__block_impl *, int, int))((__block_impl *)block)->FuncPtr)((__block_impl *)block, 10, 10);

        return UIApplicationMain(argc, argv, __null, NSStringFromClass(((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("AppDelegate"), sel_registerName("class"))));
    }
}
```

实际上 block 在底层对应的就是 \_\_main\_block\_impl\_0：

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int a;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _a, int flags=0) : a(_a) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

里面存储着 \_\_block\_impl 的结构体 impl，以及 \_\_main\_block\_desc\_0 的结构体指针 Desc. 搜索对象的内容我们可以找到:

```
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
```

这里可以看到 \_\_block\_impl 包含着 isa 指针，以及 FuncPtr。FuncPtr 就是 block 的调用地址，是在声明 block 的时候初始化传递进来的。以及 \_\_main\_block\_desc\_0 包含着的 Block_size 为 block 的内存大小。还有 int a 也封装到了Block 内部，我们知道 OC 对象的特征就是 isa 指针，所以，block 就是封装了函数调用、以及函数调用环境的 OC 对象。

<center>
![](http://www.dzliving.com/wp-content/uploads/2018/12/BlockStructImage.jpg)
</center>

反编译成 C 文件：

```
xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m -o main.c
```

block 的数据结构定义如下：

<center>
![](http://www.dzliving.com/wp-content/uploads/2018/12/BlockStruct.jpg)
</center>

对应的结构体定义如下： 

```
struct Block_descriptor { 
    unsigned long int reserved; 
    unsigned long int size; 
    void (*copy)(void *dst, void *src);
    void (*dispose)(void *); 
}; 

struct Block_layout {
    void *isa;
    int flags;
    int reserved; 
    void (*invoke)(void *, …);
    struct Block_descriptor *descriptor; 
    /* Imported variables. */ 
}; 
```

通过该图，我们可以知道，一个 block 实例实际上由 6 部分构成：

1.  isa 指针，所有对象都有该指针，用于实现对象相关的功能。
2.  flags，用于按 bit 位表示一些 block 的附加信息，本文后面介绍 block copy 的实现代码可以看到对该变量的使用。
3.  reserved，保留变量。
4.  invoke，函数指针，指向具体的 block 实现的函数调用地址。
5.  descriptor， 表示该 block 的附加描述信息，主要是 size 大小，以及 copy 和 dispose 函数的指针。
6.  variables，capture 过来的变量，block 能够访问它外部的局部变量，就是因为将这些变量（或变量的地址）复制到了结构体中。

 

## 三、Capture (捕获)

对于局部变量：值传递，Block 只是把局部变量的值捕获存储在了 block 的结构体内存储。

```
int a = 10;
void (^block)(void) = ^{
    NSLog(@"%d", a);
};
a = 20;
block(); // 输出 10
```

对于 Static：指针传递，Block 把 static 的变量的指针存储在 block 的结构体内，所以取值的话就是取对应最后的赋值。

```
static int a = 10;
void (^block)(void) = ^{
    NSLog(@"%d", a);
};
a = 20;
block();  // 输出 20
```

全局变量：直接访问

```
static int a = 20;
int b = 15;

int main(int argc, const char * argv\[\]) {
    @autoreleasepool {
        
        void (^block)(void) = ^{
            NSLog(@"%d", a);
            NSLog(@"%d", b);
        };
        a = 25;
        b = 10;
        block();  // 输出 25 10
    }
    return 0;
}
```

## 四、block 的类型

block 分为 3 种类型，但是最终都是继承自 NSObject。

*    \_NSConcreteGlobalBlock   全局的静态 block，内部没有访问 auto 变量，不会访问任何外部变量。
*    \_NSConcreteStackBlock     保存在栈中的 block，内部访问了 auto 变量，当函数返回时会被销毁。
*    \_NSConcreteMallocBlock   保存在堆中的 block，当引用计数为 0 时会被销毁。
    

> stack block 存放在栈内存，如果 block 存放在函数内，一旦函数作用域结束，则 block 内容则会被清除，如果存放在堆内存（调用 copy），就会变成 malloc block，则不会自动清除，这也是为什么 block 需要用 copy 修饰的原因。

<center>
![](http://www.dzliving.com/wp-content/uploads/2018/12/BlockTypes.jpg)
</center>

### 4.1 NSConcreteGlobalBlock

```
#include <stdio.h>

int main() 
{ 
    ^{ printf("Hello, World!\\n"); } (); 
   
    return 0; 
}
```

在终端命令行中输入 

```
$ clang -rewrite-objc block.cpp(文件名) 
```

即可在目录中看到 clang 输出了一个名为 block.cpp 的文件。该文件就是 block 在 c 语言实现，将 block.cpp 中一些无关的代码去掉，将关键代码引用如下：

```
struct __block_impl { 
    void *isa; 
    int Flags; 
    int Reserved; 
    void *FuncPtr; 
}; 

struct __main_block_impl_0 { 
    struct __block_impl impl; 
    struct __main_block_desc_0* Desc; 
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) { 
        impl.isa = &_NSConcreteStackBlock; 
        impl.Flags = flags; 
        impl.FuncPtr = fp; 
        Desc = desc; 
    } 
}; 
    
static void __main_block_func_0(struct __main_block_impl_0 *__cself) { 
    printf("Hello, World!\n"); 
} 

static struct __main_block_desc_0 { 
    size_t reserved; 
    size_t Block_size; 
}__main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0) }; 

int main() 
{ 
    (void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA) (); 
    return 0; 
} 
```

具体看一下是如何实现的。\_\_main\_block\_impl\_0 就是该 block 的实现，从中可以看出：

1. 一个 block 实际是一个对象，它主要由一个 isa 和一个 impl 和一个 descriptor 组成。
2. 在本例中，isa 指向 \_NSConcreteGlobalBlock，主要是为了实现对象的所有特性，在此我们就不展开讨论了。
3. impl 是实际的函数指针，本例中，它指向 \_\_main\_block\_func\_0。这里的 impl 相当于之前提到的 invoke 变量，只是clang 编译器对变量的命名不一样而已。
4. descriptor 是用于描述当前这个 block 的附加信息的，包括结构体的大小，需要 capture 和 dispose 的变量列表等。结构体大小需要保存是因为，每个 block 因为会 capture 一些变量，这些变量会加到 \_\_main\_block\_impl\_0 这个结构体中，使其体积变大。在该例子中我们还看不到相关 capture 的代码，后面将会看到。

#### 4.2 NSConcreteStackBlock

```
#include <stdio.h> 

int main()
{ 
    int a = 100; 
    void (^block)(void) = ^{ 
        printf("%d\n", a); 
    }; 
    block(); 

    return 0; 
}
```

反编译之后：

```
struct __main_block_impl_0 { 
    struct __block_impl impl; 
    struct __main_block_desc_0* Desc; 
    int a; 
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _a, int flags=0) : a(_a) { 
        impl.isa = &_NSConcreteStackBlock; 
        impl.Flags = flags; 
        impl.FuncPtr = fp; 
        Desc = desc; 
    } 
}; 
static void __main_block_func_0(struct __main_block_impl_0 *__cself) { 
    int a = __cself->a; // bound by copy 
    printf("%d\n", a); 
} 

static struct __main_block_desc_0 { 
    size_t reserved; 
    size_t Block_size; 
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)}; 

int main() 
{ 
    int a = 100; 
    void (*block)(void) = (void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, a); 
    ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block); 

    return 0; 
} 
```

在本例中，我们可以看到：

1. 本例中，isa 指向 \_NSConcreteStackBlock，说明这是一个分配在栈上的实例。
2. main\_block\_impl\_0 中增加了一个变量 a，在 block 中引用的变量 a 实际是在申明 block 时，被复制到 main\_block\_impl\_0 结构体中的那个变量 a。因为这样，我们就能理解，在 block 内部修改变量 a 的内容，不会影响外部的实际变量 a。
3. main\_block\_impl\_0 中由于增加了一个变量 a，所以结构体的大小变大了，该结构体大小被写在了 main\_block\_desc\_0 中。

#### 4.3 NSConcreteMallocBlock

NSConcreteMallocBlock 类型的 block 通常不会在源码中直接出现，因为默认它是当一个 block 被 copy 的时候，才会将这个 block 复制到堆中。以下是一个 block 被 copy 时的示例代码（来自[这里](http://www.galloway.me.uk/2013/05/a-look-inside-blocks-episode-3-block-copy/)），可以看到，在第 8 步，目标的 block 类型被修改为 \_NSConcreteMallocBlock。

```
static void *_Block_copy_internal(const void *arg, const int flags) {
    struct Block_layout *aBlock;
    const bool wantsOne = (WANTS_ONE & flags) == WANTS_ONE;

    // 1
    if (!arg) return NULL;

    // 2
    aBlock = (struct Block_layout *)arg;

    // 3
    if (aBlock->flags & BLOCK_NEEDS_FREE) {
        // latches on high
        latching_incr_int(&aBlock->flags);
        return aBlock;
    }

    // 4
    else if (aBlock->flags & BLOCK_IS_GLOBAL) {
        return aBlock;
    }

    // 5
    struct Block_layout *result = malloc(aBlock->descriptor->size);
    if (!result) return (void *)0;

    // 6
    memmove(result, aBlock, aBlock->descriptor->size); // bitcopy first

    // 7
    result->flags &= ~(BLOCK_REFCOUNT_MASK);    // XXX not needed
    result->flags |= BLOCK_NEEDS_FREE | 1;

    // 8
    result->isa = _NSConcreteMallocBlock;

    // 9
    if (result->flags & BLOCK_HAS_COPY_DISPOSE) {
        (*aBlock->descriptor->copy)(result, aBlock); // do fixup
    }

    return result;
}
```

## 五、变量的复制

block 内部默认是无法修改 auto 变量的，因为在 block 底部的话执行 block、声明局部变量 a（main 函数）的地方分别是两个不同的函数，并没有办法从一个函数去修改另一个函数的局部变量，而如果使用 static 或者使用全局变量是可以的，因为block 在底层存储 static 变量是存储它的指针地址，全局变量就全部都可以访问。

如果要修改 auto 变量的话，则需要使用 \_\_block。

对于 block 外的变量引用，block 默认是将其复制到其数据结构中来实现访问的：

<center>
![](http://www.dzliving.com/wp-content/uploads/2018/12/2Q.jpg)
</center>

对于用 \_\_block 修饰的外部变量引用，block 是复制其引用地址来实现访问的：

<center>
![](http://www.dzliving.com/wp-content/uploads/2018/12/9k.jpg)
</center>

修改上面的源码，在变量前面增加 \_\_block 关键字：

```
#include <stdio.h> 

int main() 
{ 
    __block int i = 1024; 
    void (^block)(void) = ^{ 
        printf("%d\n", i); 
        i = 1023; 
    }; 
    block(); 
    return 0; 
}
```

生成的关键代码如下，可以看到，差异相当大：

```
struct __Block_byref_a_0 {
  void *__isa;
__Block_byref_a_0 *__forwarding;
 int __flags;
 int __size;
 int a;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __Block_byref_a_0 *a; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_a_0 *_a, int flags=0) : a(_a->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __Block_byref_a_0 *a = __cself->a; // bound by ref

            printf("%dn", (a->__forwarding->a));
            (a->__forwarding->a) = 1023;
        }
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->a, (void*)src->a, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->a, 8/*BLOCK_FIELD_IS_BYREF*/);}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

int main(int argc, char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        __attribute__((__blocks__(byref))) __Block_byref_a_0 a = {(void*)0,(__Block_byref_a_0 *)&a, 0, sizeof(__Block_byref_a_0), 1024};
        void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_a_0 *)&a, 570425344));
        ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);

        return UIApplicationMain(argc, argv, __null, NSStringFromClass(((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("AppDelegate"), sel_registerName("class"))));
    }
}
```

从代码中可以看到：

1. 源码中增加一个名为 \_\_Block\_byref\_i\_0 的结构体，用来保存我们要 capture 并且修改的变量 a。
2. main\_block\_impl\_0 中引用的是 Block\_byref\_i\_0 的结构体指针，这样就可以达到修改外部变量的作用。
3. \_\_Block\_byref\_i\_0 结构体中带有 isa、a以及 \_\_forwarding（指向自己的指针）等其他信息，它也是一个对象。
4. 我们需要负责 Block\_byref\_i\_0 结构体相关的内存管理，所以 main\_block\_desc\_0 中增加了 copy 和 dispose 函数指针，对于在调用前后修改相应变量的引用计数。

```
__attribute__((__blocks__(byref))) __Block_byref_a_0 a = {(void*)0,(__Block_byref_a_0 *)&a, 0, sizeof(__Block_byref_a_0), 1024};
void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_a_0 *)&a, 570425344));
```

这里声明了一个 \_\_Block\_byref\_a\_0 的对象，并把 &a 传递给了 \_\_forwarding，10 传递给了 a。

```
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __Block_byref_a_0 *a = __cself->a; // bound by ref

            printf("%dn", (a->__forwarding->a));
            (a->__forwarding->a) = 1023;
        }
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->a, (void*)src->a, 8/*BLOCK_FIELD_IS_BYREF*/);}
```

这里执行 block 时，取出了 \_\_Block\_byref\_a\_0 所存储的 &a（\_\_forwarding：\_\_Block\_byref\_a\_0 的指针地址），再取出 a，最后进行修改/使用。

## 六、ARC 对 block 类型的影响

在 ARC 开启的情况下，将只会有 NSConcreteGlobalBlock 和 NSConcreteMallocBlock 类型的 block。

原本的 NSConcreteStackBlock 的 block 会被 NSConcreteMallocBlock 类型的 block 替代。在苹果的[官方文档](http://developer.apple.com/library/ios/#releasenotes/ObjectiveC/RN-TransitioningToARC/Introduction/Introduction.html)中也提到，当把栈中的 block 返回时，不需要调用 copy 方法了。

```
int main(int argc, char * argv[]) {
    @autoreleasepool {
        int i = 1024;
        void (^block)(void) = ^{
            printf("%d\n", i);
        };
        block();
        NSLog(@"%@", block);
        
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

个人认为这么做的原因是，由于 ARC 已经能很好地处理对象的生命周期的管理，这样所有对象都放到堆上管理，对于编译器实现来说，会比较方便。

## 七、Block 循环引用问题

#### 7.1 RetainCircle 的由来

当 A 对象里面强引用了 B 对象，B 对象又强引用了 A 对象，这样两者的 retainCount 值一直都无法为 0，于是内存始终无法释放，导致内存泄露。所谓的内存泄露就是本应该释放的对象，在其生命周期结束之后依旧存在。

<center>
![](http://www.dzliving.com/wp-content/uploads/2018/02/BlockObject.jpeg)
</center>

这是 2 个对象之间的，相应的，这种循环还能存在于 3、4 ... n 个对象之间，只要<font color=#cc0000>相互形成环</font>，就会导致 Retain Cicle 的问题。

当然也存在自身引用自身的。当一个对象内部的一个 obj，强引用的自身，也会导致循环引用的问题出现。常见的就是 block 里面引用的问题。

<center>
![](http://www.dzliving.com/wp-content/uploads/2018/02/BlockRelate.jpeg)
</center>

#### 7.2 \_\_weak、\_\_strong 的实现原理

在 ARC 环境下，id 类型和对象类型、C 语言其他类型不同，类型前必须加上所有权的修饰符。

所有权修饰符总共有 4 种：

1. \_\_strong 修饰符  
2. \_\_weak 修饰符  
3. \_\_unsafe\_unretained 修饰符  
4. \_\_autoreleasing 修饰符

一般我们如果不写，默认的修饰符是 \_\_strong。

要想弄清楚 \_\_strong、\_\_weak 的实现原理，我们就需要研究研究 clang(LLVM编译器)和 objc4 Objective-C runtime 库了。

关于 clang 有一份[关于ARC详细的文档](http://clang.llvm.org/docs/AutomaticReferenceCounting.html)，有兴趣的可以仔细研究一下文档里面的说明和例子，很有帮助。

以下的讲解，也会来自于上述文档中的函数说明。

<center>
![](http://www.dzliving.com/wp-content/uploads/2018/02/Block1.jpg)
</center>

1. \_\_strong 的实现原理

	> ①、对象持有自己
	
	首先我们先来看看生成的对象持有自己的情况，利用 alloc/new/copy/mutableCopy 生成对象。
	
	当我们声明了一个 \_\_strong 对象
	
	```
	{  
		id __strong obj = [[NSObject alloc] init];
	}
	```
	
	LLVM 编译器会把上述代码转换成下面的样子
	
	```
	id __attribute__((objc_ownership(strong))) obj = ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("alloc")), sel_registerName("init"));
	```
	
	相应的会调用
	
	```
	id obj = objc_msgSend(NSObject, @selector(alloc));
	
	objc_msgSend(obj,selector(init));
	
	objc_release(obj);
	```
	
	上述这些方法都好理解。在 ARC 有效的时候就会自动插入 release 代码，在作用域结束的时候自动释放。
	
	> ②、对象不持有自己
	
	生成对象的时候不用 alloc/new/copy/mutableCopy 等方法。
	
	```
	{  
		id __strong obj = [NSMutableArray array];  
	}
	```
	
	LLVM 编译器会把上述代码转换成下面的样子
	
	```
	id __attribute__((objc_ownership(strong))) array = ((NSMutableArray *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSMutableArray"), sel_registerName("array"));
	```
	
	查看 LLVM 文档，其实是下述的过程，相应的会调用
	
	```
	id obj = objc_msgSend(NSMutableArray, @selector(array));
	
	objc_retainAutoreleasedReturnValue(obj);
	
	objc_release(obj);
	```
	
	与之前对象会持有自己的情况不同，这里多了一个 objc\_retainAutoreleasedReturnValue 函数。
	
	这里有 3 个函数需要说明：
	
	1、[id objc_retainAutoreleaseReturnValue(id value);](http://clang.llvm.org/docs/AutomaticReferenceCounting.html#id69)
	
	
	>\_Precondition:\_ value is null or a pointer to a valid object.
	>
	>If value is null, this call has no effect. Otherwise, it performs a retain operation followed by the operation described in [objc_autoreleaseReturnValue](http://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-autoreleasereturnvalue).
	>
	>Equivalent to the following code:
	>
	>id objc\_retainAutoreleaseReturnValue(id value) {
	>
	>return objc\_autoreleaseReturnValue(objc\_retain(value));
	>
	>}
	>
	>Always returns value
	```
	
	2、[id objc_retainAutoreleasedReturnValue(id value);](http://clang.llvm.org/docs/AutomaticReferenceCounting.html#id70)
	
	>\_Precondition:\_ value is null or a pointer to a valid object.
	>
	>If value is null, this call has no effect. Otherwise, it attempts to accept a hand off of a retain count from a call to [objc_autoreleaseReturnValue](http://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-autoreleasereturnvalue) on value in a recently-called function or something it calls. If that fails, it performs a retain operation exactly like [objc_retain](http://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-retain).
	>
	>Always returns value
	
	3、[id objc_autoreleaseReturnValue(id value);](http://clang.llvm.org/docs/AutomaticReferenceCounting.html#id59)
	
	>\_Precondition:\_ value is null or a pointer to a valid object.
	>
	>If value is null, this call has no effect. Otherwise, it makes a best effort to hand off ownership of a retain count on the object to a call to[objc_retainAutoreleasedReturnValue](http://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-retainautoreleasedreturnvalue) for the same object in an enclosing call frame. If this is not possible, the object is autoreleased as above.
	>
	>Always returns value
	
	这 3 个函数其实都是在描述一件事情：it makes a best effort to hand off ownership of a retain count on the object to a call to objc_retainAutoreleasedReturnValue for the same object in an enclosing call frame。
	
	这属于 LLVM 编译器的一个优化。objc\_retainAutoreleasedReturnValue 函数是用于自己持有(retain)对象的函数，它持有的对象应为返回注册在 autoreleasepool 中对象的方法或者是函数的返回值。
	
	在 ARC 中原本对象生成之后是要注册到 autoreleasepool 中，但是调用了objc\_autoreleasedReturnValue 之后，紧接着调用了 objc\_retainAutoreleasedReturnValue，objc\_autoreleasedReturnValue 函数会去检查该函数方法或者函数调用方的执行命令列表，如果里面有objc\_retainAutoreleasedReturnValue() 方法，那么该对象就直接返回给方法或者函数的调用方。达到了即使对象不注册到 autoreleasepool 中，也可以返回拿到相应的对象。

2. \_\_weak 的实现原理

	声明一个 \_\_weak 对象
	
	```
	{  
		id __weak obj = strongObj; // 假设这里的 strongObj 是一个已经声明好了的对象。
	}
	```
	
	LLVM 转换成对应的代码
	
	```
	id __attribute__((objc_ownership(none))) obj1 = strongObj;
	```
	
	相应的会调用
	
	```
	id obj ;
	
	objc_initWeak(&obj,strongObj);
	
	objc_destoryWeak(&obj);
	```
	
	看看文档描述
	
	>[id objc\_initWeak(id *object, id value);](http://clang.llvm.org/docs/AutomaticReferenceCounting.html#id62)
	>
	>\_Precondition:\_ object is a valid pointer which has not been registered as a \_\_weak object. value is null or a pointer to a valid object. If value is a null pointer or the object to which it points has begun deallocation, object is zero-initialized. Otherwise, object is registered as a \_\_weak object pointing to value
	>
	>Equivalent to the following code:
	>
	>id objc\_initWeak(id \_object, id value) { \_
	>
	>object = nil;
	>
	>return objc_storeWeak(object, value);
	>
	>}
	>
	>Returns the value of object after the call.Does not need to be atomic with respect to calls to objc\_storeWeak on object
	
	objc_initWeak 的实现其实是这样的：
	
	```
	id objc_initWeak(id * object, id value) {
	
		*object = nil;
		
		return objc_storeWeak(object, value);
	}
	```
	
	会把传入的 object 变成 0 或者 nil，然后执行 objc\_storeWeak 函数。
	
	那么 objc\_destoryWeak 函数是干什么的呢？
	
	>[void objc_destroyWeak(id *object);](http://clang.llvm.org/docs/AutomaticReferenceCounting.html#id61)
	>
	>\_Precondition:\_ object is a valid pointer which either contains a null pointer or has been registered as a \_\_weak object. object is unregistered as a weak object, if it ever was. The current value of object is left unspecified; otherwise, equivalent to the following code:
	>
	>void objc\_destroyWeak(id * object) {
	>
	>objc\_storeWeak(object, nil);
	>
	>}
	>
	>Does not need to be atomic with respect to calls to objc_storeWeak on object
	
	objc\_destoryWeak 函数的实现：
	
	```
	void objc_destroyWeak(id * object) {
	
		objc_storeWeak(object, nil);
	
	}
	```
	
	也是会去调用 objc\_storeWeak 函数。objc\_initWeak 和 objc\_destroyWeak 函数都会去调用 objc\_storeWeak 函数，唯一不同的是调用的入参不同，一个是 value，一个是 nil。
	
	那么重点就都落在 objc\_storeWeak 函数上了。
	
	>[id objc_storeWeak(id *object, id value);](http://clang.llvm.org/docs/AutomaticReferenceCounting.html#id73)
	>
	>\_Precondition:\_ object is a valid pointer which either contains a null pointer or has been registered as a \_\_weak object. value is null or a pointer to a valid object. If value is a null pointer or the object to which it points has begun deallocation, object is assigned null and unregistered as a \_\_weak object. Otherwise, object is registered as a \_\_weak object or has its registration updated to point to value
	>
	>Returns the value of object after the call.
	
	objc\_storeWeak 函数的用途就很明显了。由于 weak 表也是用 Hash table 实现的，所以objc\_storeWeak 函数就把第一个入参的变量地址注册到 weak 表中，然后根据第二个入参来决定是否移除。如果第二个参数为 0，那么就把 \_\_weak 变量从 weak 表中删除记录，并从引用计数表中删除对应的键值记录。
	
	所以如果 \_\_weak 引用的原对象如果被释放了，那么对应的 \_\_weak 对象就会被指为 nil。原来就是通过 objc\_storeWeak 函数这些函数来实现的。
	
	以上就是 ARC 中 \_\_strong 和 \_\_weak 的简单的实现原理，更加详细的还请大家去看看这一章开头提到的那个 LLVM 文档，里面说明的很详细。
	
	<center>
	![](http://www.dzliving.com/wp-content/uploads/2018/02/Block2.png)
	</center>

#### 7,3 weakSelf、strongSelf 的用途

在提 weakSelf、strongSelf 之前，我们先引入一个 Retain Cicle 的例子。

假设自定义的一个 student 类

例子 1：

**Student.h 文件**

```
#import <Foundation/Foundation.h>

typedef void(^ Study)();

@interface Student : NSObject

@property (nonatomic, copy) NSString * name;
@property (nonatomic, copy) Study study;

@end
```

**ViewController.m 文件**

```
#import "ViewController.h"
#import "Student.h"

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad
{

	[super viewDidLoad];
	
	Student * student = [[Student alloc] init];
	
	student.name = @"Hello World";
	
	student.study = ^{
		NSLog(@"my name is = %@", student.name);
	};

}
```

到这里，大家应该看出来了，这里肯定出现了循环引用了。student 的 study 的 Block 里面强引用了 student 自身。根据[上篇文章](http://www.jianshu.com/p/ee9756f3d5f6)的分析，可以知道，\_NSConcreteMallocBlock 捕获了外部的对象，会在内部持有它。retainCount 值会加一。

我们用 Instruments 来观察一下。添加 Leak 观察器。

当程序运行起来之后，在 Leak Checks观察器里面应该可以看到红色的❌，点击它就会看到内存 leak 了。有 2 个泄露的对象。Block 和 Student 相互循环引用了。

<center>
![](http://www.dzliving.com/wp-content/uploads/2018/02/Block3.jpeg)
</center>

打开 Cycles & Roots 观察一下循环的环。

<center>
![](http://www.dzliving.com/wp-content/uploads/2018/02/Block4.jpg)
</center>

这里形成环的原因 block 里面持有 student 本身，student 本身又持有 block。

那再看一个例子 2：

```
#import "ViewController.h"
#import "Student.h"

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad
{
	[super viewDidLoad];
	
	Student * student = [[Student alloc] init];
	
	student.name  = @"Hello World";
	
	student.study = ^(NSString * name){
		NSLog(@"my name is = %@",name);
	};
	
	student.study(student.name);
	
}
```

我把 block 新传入一个参数，传入的是 student.name。这个时候会引起循环引用么？

答案肯定是不会。

<center>
![](http://www.dzliving.com/wp-content/uploads/2018/02/Block5.jpeg)
</center>

如上图，并不会出现内存泄露。原因是因为，student 是作为形参传递进 block 的，block 并不会捕获形参到 block 内部进行持有。所以肯定不会造成循环引用。

再改一下。看例子 3：

```
#import "ViewController.h"
#import "Student.h"

@interface ViewController ()

@property (nonatomic, copy) NSString * name;
@property (nonatomic, strong) Student * stu;

@end

@implementation ViewController

- (void)viewDidLoad
{
	[super viewDidLoad];
	
	Student * student = [[Student alloc] init];
	
	self.name = @"halfrost";
	self.stu  = student;
	
	student.study = ^{
		NSLog(@"my name is = %@", self.name);
	};

	student.study();
}
```

这样会形成循环引用么？

<center>
![](http://www.dzliving.com/wp-content/uploads/2018/02/Block6.jpg)
</center>

答案也是会的(ARC 环境)。

vc → student → block → vc 已经成环。这里即使是 self.name 也是循环引用了，因为 block 不可能说去单独的强持有某个实例的变量，这不符合内存管理规则(交叉管理了)，但是 instruments 检测不出来。

(原文写着没有循环引用，我在 ARC 环境测试时，dealloc 不会被调用，说明还被引用着。可以自行验证)。

那遇到循环引用我们改如何处理呢？？类比平时我们经常写的 delegate，可以知道，只要有一边是 \_\_weak 就可以打破循环。

先说一种做法，利用 __block 解决循环的做法。例子 4：

```
#import "ViewController.h"
#import "Student.h"

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad
{
	[super viewDidLoad];
	
	Student * student = [[Student alloc] init];
	
	__block Student * stu = student;
	
	student.name = @"Hello World";
	
	student.study = ^{
		NSLog(@"my name is = %@", stu.name);
		stu = nil;
	};
}
```

这样写会循环么？看上去应该不会。但是实际上却是会的。

<center>
![](http://www.dzliving.com/wp-content/uploads/2018/02/Block7.jpg)
</center>

由于没有执行 study 这个 block，现在 student 持有该 block，block 持有 \_\_block 变量，\_\_block 变量又持有 student 对象。3 者形成了环，导致了循环引用了。

想打破环就需要破坏掉其中一个引用。\_\_block 不持有 student 即可。

只需要执行一下 block 即可。例子 5：

```
student.study();
```

这样就不会循环引用了。

<center>
![](http://www.dzliving.com/wp-content/uploads/2018/02/Block8.png)
</center>

使用 \_\_block 解决循环引用虽然可以控制对象持有时间，在 block 中还能动态的控制 \_\_block 变量的值，可以赋值 nil，也可以赋值其他的值，但是有一个唯一的缺点就是需要执行一次 block 才行。否则还是会造成循环引用。

**值得注意的是，在 ARC 下 \_\_block** **会导****致对象被 retain，有可能导致循环引用。而在 MRC 下，则不会 retain 这个对象，也不会导致循环引用。**

接下来可以正式开始讲讲 weakSelf 和 strongSelf 的用法了。

1. weakSelf

	说道 weakSelf，需要先来区分几种写法。

	①、\_\_weak \_\_typeof(self)weakSelf = self;   // 这是 AFN 里面的写法。。

	②、#define WEAKSELF typeof(self) __weak weakSelf = self;

	> 先区分 __typeof() 和 typeof()

	AFNetWorking 的库里面的代码都很整洁，里面各方面的代码都可以当做代码范本来阅读。遇到不懂疑惑的，都要深究，肯定会有收获。这里就是一处，平时我们的写法是不带 \_\_ 的，AFN 里面用这种写法有什么特殊的用途么？

	在 SOF 上能找到相关的[答案](http://stackoverflow.com/questions/14877415/difference-between-typeof-typeof-and-typeof-objective-c)：

	>\_\_typeof\_\_() and \_\_typeof() are compiler-specific extensions to the C language, because standard C does not include such an operator. Standard C requires compilers to prefix language extensions with a double-underscore (which is also why you should never do so for your own functions, variables, etc.)  
	>typeof() is exactly the same, but throws the underscores out the window with the understanding that every modern compiler supports it. (Actually, now that I think about it, Visual C++ might not. It does support decltype() though, which generally provides the same behaviour as typeof().)  
	>All three mean the same thing, but none are standard C so a conforming compiler may choose to make any mean something different.

	其实两者都是一样的东西，只不过是 C 里面不同的标准，兼容性不同罢了。更加详细的[官方说明](http://gcc.gnu.org/onlinedocs/gcc/Alternate-Keywords.html#Alternate-Keywords)

	那么抽象出来就是这 2 种写法。
	
	```
	#define WEAKSELF __weak typeof(self)weakSelf  = self;  
	#define WEAKSELF typeof(self) __weak weakSelf = self;
	```
	
	这样子看就清楚了，两种写法就是完全一样的。

	我们可以用 WEAKSELF 来解决循环引用的问题。例子 6：
	
	```
	#import "ViewController.h"
	#import "Student.h"
	
	@interface ViewController ()
	@end
	
	@implementation ViewController
	
	- (void)viewDidLoad
	{
		
		[super viewDidLoad];
		
		Student * student = [[Student alloc]init];
		student.name = @"Hello World";
		
		__weak typeof(student) weakSelf = student;
		student.study = ^{
			NSLog(@"my name is = %@",weakSelf.name);
		};
		
		student.study();
	}
	```
	
	这样就解决了循环引用的问题了。
	
	解决循环应用的问题一定要分析清楚哪里出现了循环引用，只需要把其中一环加上 weakSelf 这类似的宏，就可以解决循环引用。如果分析不清楚，就只能无脑添加 weakSelf、strongSelf，这样的做法不可取。
	
	在上面的例子 3 中，就完全不存在循环引用，要是无脑加 weakSelf、strongSelf 是不对的。在例子 6 中，也只需要加一个 weakSelf 就可以了，也不需要加 strongSelf。
	
	曾经在 segmentfault 也看到过这样一个问题，问：[为什么 iOS 的 Masonry 中的 self 不会循环引用?](https://segmentfault.com/q/1010000004343510)
	
	```
	UIButton * testButton = [[UIButton alloc] init];
	[self.view addSubview:testButton];
	testButton.backgroundColor = [UIColor redColor];
	
	[testButton mas_makeConstraints:^(MASConstraintMaker * make) {
	
		make.width.equalTo(@100);
		make.height.equalTo(@100);
		make.left.equalTo(self.view.mas_left);
		make.top.equalTo(self.view.mas_top);
	}];
	
	[testButton bk_addEventHandler:^(id sender) {
		[self dismissViewControllerAnimated:YES completion:nil];
	} forControlEvents:UIControlEventTouchUpInside];
	```
	
	如果我用 blocksKit 的 bk\_addEventHandler方法，其中使用 strong self，该 viewController 就无法 dealloc，我理解是因为 self → self.view → testButton → self。 但是如果只用 Mansonry 的 mas\_makeConstraints方法，同样使用 strong self，该 viewController 却能正常 dealloc，请问为什么 Masonry 没有导致循环引用？
	
	看到这里，读者应该就应该能回答这个问题了。
	
	```
	- (NSArray *)mas_makeConstraints:(void(^)(MASConstraintMaker *))block
	{
		self.translatesAutoresizingMaskIntoConstraints = NO;
		
		MASConstraintMaker * maker = [[MASConstraintMaker alloc] initWithView:self];
		
		block(maker);
		
		return [maker install];
	}
	```

	关于 Masonry，它捕获了变量 self，然后对其执行了 setTranslatesAutoresizingMaskIntoConstraints: 方法。但是，因为执行完毕后，block 会被销毁，没有形成环。所以，没有引起循环依赖。

2. strongSelf

	上面介绍完了 weakSelf，既然 weakSelf 能完美解决 Retain Circle 的问题了，那为何还需要strongSelf 呢？
	
	还是先从 AFN 经典说起，以下是 AFN 其中的一段代码：
	
	```
	#pragma mark - NSOperation
	
	- (void)setCompletionBlock:(void (^)(void))block
	{
		[self.lock lock];
		
		if (!block) {	
			[super setCompletionBlock:nil];
		}
		else {
			__weak __typeof(self)weakSelf = self;
	
			[super setCompletionBlock:^ {
				__strong __typeof(weakSelf)strongSelf = weakSelf;
	
	#pragma clang diagnostic push
	#pragma clang diagnostic ignored "-Wgnu"
	
				dispatch_group_t group = strongSelf.completionGroup ?: url_request_operation_completion_group();
				dispatch_queue_t queue = strongSelf.completionQueue ?: dispatch_get_main_queue();
				
	#pragma clang diagnostic pop
	
				dispatch_group_async(group, queue, ^{
					block();
				});
	
				dispatch_group_notify(group, url_request_operation_completion_queue(), ^{
					[strongSelf setCompletionBlock:nil];
				});
			}];
		}
		[self.lock unlock];
	}
	```
	
	如果 block 里面不加 \_\_strong \_\_typeof(weakSelf)strongSelf = weakSelf 会如何呢？
	
	```
	#import "ViewController.h"
	#import "Student.h"
	
	@interface ViewController ()
	@end
	
	@implementation ViewController
	
	- (void)viewDidLoad
	{
		[super viewDidLoad];
		
		Student * student = [[Student alloc]init];
		student.name = @"Hello World";
		
		__weak typeof(student) weakSelf = student;
		student.study = ^{
			dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
				NSLog(@"my name is = %@",weakSelf.name); 
			});
		};
		
		student.study();
	}
	```
	
	输出：
	
	```
	my name is = (null)
	```
	
	为什么输出是这样的呢？
	
	重点就在 dispatch\_after 这个函数里面。在 study() 的 block 结束之后，student 被自动释放了。又由于 dispatch_after 里面捕获的 \_\_weak 的 student，根据第二章讲过的 \_\_weak 的实现原理，在原对象释放之后 \_\_weak 对象就会变成 null，防止野指针。所以就输出了 null了。
	
	那么我们怎么才能在 weakSelf 之后，block里面还能继续使用 weakSelf 之后的对象呢？
	
	究其根本原因就是 weakSelf 之后，无法控制什么时候会被释放，为了保证在 block 内不会被释放，需要添加 \_\_strong。
	
	在 block 里面使用的 \_\_strong 修饰的 weakSelf 是为了在函数生命周期中防止 self 提前释放。strongSelf 是一个自动变量当 block 执行完毕就会释放自动变量 strongSelf 不会对 self 进行一直进行强引用。
	
	```
	#import "ViewController.h"
	#import "Student.h"
	
	@interface ViewController ()
	@end
	
	@implementation ViewController
	
	- (void)viewDidLoad
	{
		[super viewDidLoad];
		
		Student * student = [[Student alloc] init];
		student.name = @"Hello World";
		
		__weak typeof(student) weakSelf = student;
		student.study = ^{
			__strong typeof(student) strongSelf = weakSelf;
		
			dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
				NSLog(@"my name is = %@",strongSelf.name);
			});
		};
		
		student.study();
	}
	```
	
	输出
	
	```
	my name is = Hello World
	```
	
至此，我们就明白了 weakSelf、strongSelf 的用途了。
	
weakSelf 是为了 block 不持有 self，避免 Retain Circle 循环引用。在 Block 内如果需要访问 self 的方法、变量，建议使用 weakSelf。
	
strongSelf 的目的是因为一旦进入 block 执行，假设不允许 self 在这个执行过程中释放，就需要加入 strongSelf。block 执行完后这个 strongSelf 会自动释放，没有不会存在循环引用问题。如果在 Block 内需要多次 访问 self，则需要使用 strongSelf。
	
关于 Retain Circle 最后总结一下，有 3 种方式可以解决循环引用。
	
结合《Effective Objective-C 2.0》(编写高质量 iOS 与 OS X 代码的 52 个有效方法)这本书的例子，来总结一下。

**EOCNetworkFetcher.h 文件**

```
typedef void (^ EOCNetworkFetcherCompletionHandler)(NSData * data);

@interface EOCNetworkFetcher : NSObject

@property (nonatomic, strong, readonly) NSURL * url;

- (id)initWithURL:(NSURL *)url;
- (void)startWithCompletionHandler:(EOCNetworkFetcherCompletionHandler)completion;

@end
```

**EOCNetworkFetcher.m 文件**

```
@interface EOCNetworkFetcher ()

@property (nonatomic, strong, readwrite) NSURL * url;
@property (nonatomic, copy) EOCNetworkFetcherCompletionHandler completionHandler;
@property (nonatomic, strong) NSData * downloadData;

@end

@implementation EOCNetworkFetcher

- (id)initWithURL:(NSURL *)url
{
	if (self = [super init]) {
		_url = url;
	}
	return self;
}

- (void)startWithCompletionHandler:(EOCNetworkFetcherCompletionHandler)completion
{
	self.completionHandler = completion;   // 开始网络请求
	
	dispatch_async(dispatch_get_global_queue(0, 0), ^{
		_downloadData = [[NSData alloc] initWithContentsOfURL:_url]; 
		
		dispatch_async(dispatch_get_main_queue(), ^{  // 网络请求完成
			[self p_requestCompleted];
		});
	});
}
	
- (void)p_requestCompleted
{
	if(_completionHandler) {
		_completionHandler(_downloadData);
	}
}

@end
```

**EOCClass.m 文件**

```
@implementation EOCClass
{
	NSData * _fetchedData;
	EOCNetworkFetcher * _networkFetcher;
}

- (void)downloadData
{
	NSURL * url = [NSURL URLWithString:@"http://www.baidu.com"];
	
	_networkFetcher = [[EOCNetworkFetcher alloc] initWithURL:url];
	[_networkFetcher startWithCompletionHandler:^(NSData * data) {
		_fetchedData = data;
	}];
}

@end
```

在这个例子中，存在 3 者之间形成环

①、completion handler 的 block 因为要设置 \_fetchedData 实例变量的值，所以它必须捕获 self 变量，也就是说 handler 块保留了 EOCClass 实例；

②、EOCClass 实例通过 strong 实例变量保留了 EOCNetworkFetcher，最后EOCNetworkFetcher 实例对象也会保留了 handler 的 block。

书上说的 3 种方法来打破循环。

1. 手动释放 EOCNetworkFetcher 使用之后持有的 \_networkFetcher，这样可以打破循环引用

	```
	- (void)downloadData
	{
		NSURL * url = [NSURL URLWithString:@"http://www.baidu.com"];
		
		_networkFetcher = [[EOCNetworkFetcher alloc] initWithURL:url];
		[_networkFetcher startWithCompletionHandler:^(NSData * data) {
			_fetchedData = data;
			_networkFetcher = nil;   // 加上此行，打破循环引用
		}];
	}
	```

2. 直接释放 block。因为在使用完对象之后需要人为手动释放，如果忘记释放就会造成循环引用了。如果使用完 completion handler 之后直接释放 block 即可。打破循环引用

	```
	- (void)p_requestCompleted
	{
		if(_completionHandler) {
			_completionHandler(_downloadData);
		}
		self.completionHandler = nil;  // 加上此行，打破循环引用
	}
	```

3. 使用 weakSelf、strongSelf

	```
	- (void)downloadData
	{
		__weak __typeof(self) weakSelf = self;
		
		NSURL * url = [NSURL URLWithString:@"http://www.baidu.com"];
		
		_networkFetcher = [[EOCNetworkFetcher alloc] initWithURL:url];
		[_networkFetcher startWithCompletionHandler:^(NSData * data) {
			__typeof(&*weakSelf) strongSelf = weakSelf;
			
			if (strongSelf) {
				strongSelf.fetchedData = data;
			}
		}];
	}
	```

<center>
![](http://www.dzliving.com/wp-content/uploads/2018/11/1194012-59b08429238b088d.png)
</center>

#### 7.4 @weakify、@strongify 实现原理

上面讲完了 weakSelf、strongSelf 之后，接下来再讲讲 @weakify、@strongify，这两个关键字是 RAC 中避免 Block 循环引用而开发的 2 个宏，这 2 个宏的实现过程很牛，值得我们学习。

@weakify、@strongify 的作用和 weakSelf、strongSelf 对应的一样。这里我们具体看看大神是怎么实现这 2 个宏的。

直接从源码看起来。

```
#define weakify(...) \
rac_keywordify \
metamacro_foreach_cxt(rac_weakify_,, __weak, __VA_ARGS__)

#define strongify(...) \
rac_keywordify \
_Pragma("clang diagnostic push") \
_Pragma("clang diagnostic ignored "-Wshadow"") \
metamacro_foreach(rac_strongify_,, __VA_ARGS__) \
_Pragma("clang diagnostic pop")
```

看到这种宏定义，咋一看什么都不知道。那就只能一层层的往下看。

1. weakify

	先从 weakify(...) 开始。
	
	```
	#if DEBUG
	#define rac_keywordify autoreleasepool {}
	#else
	#define rac_keywordify try {} @catch (...) {}
	#endif
	```
	
	这里在 debug 模式下使用 @autoreleasepool 是为了维持编译器的分析能力，而使用 @try/@catch 是为了防止插入一些不必要的 autoreleasepool。rac\_keywordify 实际上就是autoreleasepool {}的宏替换。因为有了 autoreleasepool {}的宏替换，所以 weakify 要加上 @，形成 @autoreleasepool {}。
	
	```
	#define metamacro_foreach_cxt(MACRO, SEP, CONTEXT, ...) \
	
	metamacro_concat(metamacro_foreach_cxt, metamacro_argcount(__VA_ARGS__))(MACRO, SEP, CONTEXT, __VA_ARGS__)
	```
	
	\_\_VA\_ARGS\_\_：总体来说就是将左边宏中 ... 的内容原样抄写在右边 \_\_VA\_ARGS\_\_ 所在的位置。它是一个可变参数的宏，是新的 C99 规范中新增的，目前似乎只有 gcc支持(VC 从 VC2005 开始支持)。
	
	那么我们使用 @weakify(self) 传入进去。\_\_VA\_ARGS\_\_ 相当于 self。此时我们可以把最新开始的 weakify 套下来。于是就变成了这样：
	
	```
	rac_weakify_,, __weak, __VA_ARGS__ 整体替换 MACRO, SEP, CONTEXT, ...
	```
	
	这里需要注意的是，源码中就是给的两个","逗号是连着的，所以我们也要等效替换参数，相当于 SEP 是空值。
	
	替换完成之后就是下面这个样子：
	
	```
	autoreleasepool {} metamacro_concat(metamacro_foreach_cxt, metamacro_argcount(self))(rac_weakify_, , __weak, self)
	```
	
	现在我们需要弄懂的就是 metamacro\_concat 和 metamacro\_argcount 是干什么用的。
	
	继续看看 metamacro_concat 的实现
	
	```
	#define metamacro_concat(A, B) \
	
	metamacro_concat_(A, B) #define metamacro_concat_(A, B) A ## B
	```
	\#\# 是宏连接符。举个例子：
	
	假设宏定义为 #define XNAME(n) x##n，代码为：XNAME(4)，则在预编译时，宏发现XNAME(4) 与 XNAME(n) 匹配，则令 n 为 4，然后将右边的 n 的内容也变为 4，然后将整个XNAME(4) 替换为 x##n，亦即 x4，故最终结果为 XNAME(4) 变为 x4。所以 A##B 就是 AB。
	
	**metamacro_argcount 的实现**
	
	```
	#define metamacro_argcount(...) \
	
	metamacro_at(20, __VA_ARGS__, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1) #define metamacro_at(N, ...) \
	
	metamacro_concat(metamacro_at, N)(__VA_ARGS__)
	```
	
	metamacro\_concat 是上面讲过的连接符，那么 metamacro\_at，N = metamacro\_atN，由于 N = 20，于是 metamacro\_atN = metamacro\_at20。
	
	```
	#define metamacro_at0(...) metamacro_head(__VA_ARGS__)
	
	#define metamacro_at1(_0, ...) metamacro_head(__VA_ARGS__)
	
	#define metamacro_at2(_0, _1, ...) metamacro_head(__VA_ARGS__)
	
	#define metamacro_at3(_0, _1, _2, ...) metamacro_head(__VA_ARGS__)
	
	#define metamacro_at4(_0, _1, _2, _3, ...) metamacro_head(__VA_ARGS__)
	
	#define metamacro_at5(_0, _1, _2, _3, _4, ...) metamacro_head(__VA_ARGS__)
	
	#define metamacro_at6(_0, _1, _2, _3, _4, _5, ...) metamacro_head(__VA_ARGS__)
	
	#define metamacro_at7(_0, _1, _2, _3, _4, _5, _6, ...) metamacro_head(__VA_ARGS__)
	
	#define metamacro_at8(_0, _1, _2, _3, _4, _5, _6, _7, ...) metamacro_head(__VA_ARGS__)
	
	#define metamacro_at9(_0, _1, _2, _3, _4, _5, _6, _7, _8, ...) metamacro_head(__VA_ARGS__)
	
	#define metamacro_at10(_0, _1, _2, _3, _4, _5, _6, _7, _8, _9, ...) metamacro_head(__VA_ARGS__)
	
	#define metamacro_at11(_0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, ...) metamacro_head(__VA_ARGS__)
	
	#define metamacro_at12(_0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, ...) metamacro_head(__VA_ARGS__)
	
	#define metamacro_at13(_0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, ...) metamacro_head(__VA_ARGS__)
	
	#define metamacro_at14(_0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, ...) metamacro_head(__VA_ARGS__)
	
	#define metamacro_at15(_0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, ...) metamacro_head(__VA_ARGS__)
	
	#define metamacro_at16(_0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, ...) metamacro_head(__VA_ARGS__)
	
	#define metamacro_at17(_0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, _16, ...) metamacro_head(__VA_ARGS__)
	
	#define metamacro_at18(_0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, _16, _17, ...) metamacro_head(__VA_ARGS__)
	
	#define metamacro_at19(_0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, _16, _17, _18, ...) metamacro_head(__VA_ARGS__)
	
	#define metamacro_at20(_0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, _16, _17, _18, _19, ...) metamacro_head(__VA_ARGS__)
	```
	
	metamacro\_at20 的作用就是截取前 20 个参数，剩下的参数传入 metamacro\_head。
	
	```
	#define metamacro_head(...) \
	
	metamacro_head_(__VA_ARGS__, 0)
	
	#define metamacro_head_(FIRST, ...) FIRST
	```
	
	metamacro\_head 的作用返回第一个参数。返回到上一级 metamacro\_at20，如果我们从最源头的 @weakify(self)，传递进来，那么 metamacro\_at20(self, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1)，截取前 20 个参数，最后一个留给metamacro\_head\_(1)，那么就应该返回 1。
	
	```
	metamacro_concat(metamacro_foreach_cxt, metamacro_argcount(self)) = metamacro_concat(metamacro_foreach_cxt, 1)
	```
	
	最终可以替换成 metamacro\_foreach\_cxt1。
	
	在源码中继续搜寻。
	
	```
	// metamacro_foreach_cxt expansions
	
	#define metamacro_foreach_cxt0(MACRO, SEP, CONTEXT)
	
	#define metamacro_foreach_cxt1(MACRO, SEP, CONTEXT, _0) MACRO(0, CONTEXT, _0)
	
	#define metamacro_foreach_cxt2(MACRO, SEP, CONTEXT, _0, _1) \
	
	metamacro_foreach_cxt1(MACRO, SEP, CONTEXT, _0) \
	
	SEP \
	
	MACRO(1, CONTEXT, _1)
	
	#define metamacro_foreach_cxt3(MACRO, SEP, CONTEXT, _0, _1, _2) \
	
	metamacro_foreach_cxt2(MACRO, SEP, CONTEXT, _0, _1) \
	
	SEP \
	
	MACRO(2, CONTEXT, _2)
	
	#define metamacro_foreach_cxt4(MACRO, SEP, CONTEXT, _0, _1, _2, _3) \
	
	metamacro_foreach_cxt3(MACRO, SEP, CONTEXT, _0, _1, _2) \
	
	SEP \
	
	MACRO(3, CONTEXT, _3)
	
	#define metamacro_foreach_cxt5(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4) \
	
	metamacro_foreach_cxt4(MACRO, SEP, CONTEXT, _0, _1, _2, _3) \
	
	SEP \
	
	MACRO(4, CONTEXT, _4)
	
	#define metamacro_foreach_cxt6(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5) \
	
	metamacro_foreach_cxt5(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4) \
	
	SEP \
	
	MACRO(5, CONTEXT, _5)
	
	#define metamacro_foreach_cxt7(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6) \
	
	metamacro_foreach_cxt6(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5) \
	
	SEP \
	
	MACRO(6, CONTEXT, _6)
	
	#define metamacro_foreach_cxt8(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7) \
	
	metamacro_foreach_cxt7(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6) \
	
	SEP \
	
	MACRO(7, CONTEXT, _7)
	
	#define metamacro_foreach_cxt9(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8) \
	
	metamacro_foreach_cxt8(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7) \
	
	SEP \
	
	MACRO(8, CONTEXT, _8)
	
	#define metamacro_foreach_cxt10(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9) \
	
	metamacro_foreach_cxt9(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8) \
	
	SEP \
	
	MACRO(9, CONTEXT, _9)
	
	#define metamacro_foreach_cxt11(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10) \
	
	metamacro_foreach_cxt10(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9) \
	
	SEP \
	
	MACRO(10, CONTEXT, _10)
	
	#define metamacro_foreach_cxt12(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11) \
	
	metamacro_foreach_cxt11(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10) \
	
	SEP \
	
	MACRO(11, CONTEXT, _11)
	
	#define metamacro_foreach_cxt13(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12) \
	
	metamacro_foreach_cxt12(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11) \
	
	SEP \
	
	MACRO(12, CONTEXT, _12)
	
	#define metamacro_foreach_cxt14(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13) \
	
	metamacro_foreach_cxt13(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12) \
	
	SEP \
	
	MACRO(13, CONTEXT, _13)
	
	#define metamacro_foreach_cxt15(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14) \
	
	metamacro_foreach_cxt14(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13) \
	
	SEP \
	
	MACRO(14, CONTEXT, _14)
	
	#define metamacro_foreach_cxt16(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15) \
	
	metamacro_foreach_cxt15(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14) \
	
	SEP \
	
	MACRO(15, CONTEXT, _15)
	
	#define metamacro_foreach_cxt17(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, _16) \
	
	metamacro_foreach_cxt16(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15) \
	
	SEP \
	
	MACRO(16, CONTEXT, _16)
	
	#define metamacro_foreach_cxt18(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, _16, _17) \
	
	metamacro_foreach_cxt17(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, _16) \
	
	SEP \
	
	MACRO(17, CONTEXT, _17)
	
	#define metamacro_foreach_cxt19(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, _16, _17, _18) \
	
	metamacro_foreach_cxt18(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, _16, _17) \
	
	SEP \
	
	MACRO(18, CONTEXT, _18)
	
	#define metamacro_foreach_cxt20(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, _16, _17, _18, _19) \
	
	metamacro_foreach_cxt19(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, _16, _17, _18) \
	
	SEP \
	
	MACRO(19, CONTEXT, _19)
	```
	
	metamacro\_foreach\_cxt 这个宏定义有点像递归，这里可以看到 N 最大就是 20，于是metamacro\_foreach\_cxt19 就是最大，metamacro\_foreach\_cxt19 会生成rac\_weakify\_(0,\_\_weak,\_18)，然后再把前 18 个数传入 metamacro\_foreach\_cxt18，并生成rac\_weakify\_(0,\_\_weak,\_17)，依次类推，一直递推到 metamacro\_foreach\_cxt0。
	
	```
	#define metamacro_foreach_cxt0(MACRO, SEP, CONTEXT)
	```
	
	metamacro\_foreach\_cxt0 就是终止条件，不做任何操作了。
	
	于是最初的 @weakify 就被替换成
	
	```
	autoreleasepool {}
	
	metamacro_foreach_cxt1(rac_weakify_, , __weak, self)
	
	#define metamacro_foreach_cxt1(MACRO, SEP, CONTEXT, _0) MACRO(0, CONTEXT, _0)
	```
	
	代入参数
	
	```
	autoreleasepool {} rac_weakify_（0,__weak,self）
	```
	
	最终需要解析的就是 rac\_weakify\_
	
	```
	#define rac_weakify_(INDEX, CONTEXT, VAR) \
	
	CONTEXT __typeof__(VAR) metamacro_concat(VAR, _weak_) = (VAR);
	```
	
	把 (0,__weak,self) 的参数替换进来 (INDEX, CONTEXT, VAR)。
	
	```
	INDEX = 0， CONTEXT = __weak，VAR = self，
	```
	
	于是
	
	```
	CONTEXT __typeof__(VAR) metamacro_concat(VAR, _weak_) = (VAR);
	```
	
	等效替换为
	
	```
	__weak __typeof__(self) self_weak_ = self;
	```
	
	最终 @weakify(self) = \_\_weak \_\_typeof\_\_(self) self\_weak\_ = self; 这里的 self\_weak\_ 就完全等价于我们之前写的 weakSelf。

2. strongify

	再继续分析 strongify(...)
	
	rac_keywordify 还是和 weakify 一样，是 autoreleasepool {}，只为了前面能加上 @
	
	```
	_Pragma("clang diagnostic push") \
	
	_Pragma("clang diagnostic ignored "-Wshadow"") \
	
	_Pragma("clang diagnostic pop")
	```
	
	strongify 比 weakify 多了这些 _Pragma 语句。
	
	关键字 _Pragma 是 C99 里面引入的。_Pragma 比 #pragma(在设计上)更加合理，因而功能也有所增强。
	
	上面的等效替换
	
	```
	#pragma clang diagnostic push
	
	#pragma clang diagnostic ignored "-Wshadow"
	
	#pragma clang diagnostic pop
	```
	
	这里的 clang 语句的作用：忽略当一个局部变量或类型声明遮盖另一个变量的警告。
	
	最初的
	
	```
	#define strongify(...) \
	
	rac_keywordify \
	
	_Pragma("clang diagnostic push") \
	
	_Pragma("clang diagnostic ignored "-Wshadow"") \
	
	metamacro_foreach(rac_strongify_,, __VA_ARGS__) \
	
	_Pragma("clang diagnostic pop")
	```
	
	strongify 里面需要弄清楚的就是 metamacro\_foreach 和 rac\_strongify\_。
	
	```
	#define metamacro_foreach(MACRO, SEP, ...) \
	
	metamacro_foreach_cxt(metamacro_foreach_iter, SEP, MACRO, __VA_ARGS__)
	
	#define rac_strongify_(INDEX, VAR) \
	
	__strong __typeof__(VAR) VAR = metamacro_concat(VAR, _weak_);
	```
	
	我们先替换一次，SEP = 空 ， MACRO = rac\_strongify\_ ， \_\_VA\_ARGS\_\_ ,  于是替换成这样。
	
	```
	metamacro_foreach_cxt(metamacro_foreach_iter,,rac_strongify_,self)
	```
	
	根据之前分析，metamacro\_foreach\_cxt 再次等效替换，metamacro\_foreach\_cxt##1(metamacro\_foreach\_iter,,rac\_strongify\_,self)
	
	根据
	
	```
	#define metamacro_foreach_cxt1(MACRO, SEP, CONTEXT, _0) MACRO(0, CONTEXT, _0)
	```
	
	再次替换成 metamacro\_foreach\_iter(0, rac\_strongify\_, self)
	
	继续看看 metamacro\_foreach\_iter 的实现
	
	```
	#define metamacro_foreach_iter(INDEX, MACRO, ARG) MACRO(INDEX, ARG)
	```
	
	最终替换成 rac\_strongify\_(0,self)
	
	```
	#define rac_strongify_(INDEX, VAR) \
	
	__strong __typeof__(VAR) VAR = metamacro_concat(VAR, _weak_);
	```
	
	INDEX = 0, VAR = self, 于是 @strongify(self) 就等价于
	
	```
	__strong __typeof__(VAR) VAR = metamacro_concat(VAR, _weak_);
	```
	
	等价于
	
	```
	__strong __typeof__(self) self = self_weak_;
	```
	
	注意 @strongify(self) 只能使用在 block 中，如果用在 block 外面，会报错，因为这里会提示你 ``Redefinition of 'self'``。
	
	**实例**
	
	```
	#ifndef weakify
	
	#if DEBUG   // 判断当前代码运行模式
	
	#if __has_feature(objc_arc)  // 判断 ARC 环境
	
	#define weakify(object) autoreleasepool{} __weak __typeof__(object) weak##_##object = object;     // ## 为连接符
	
	#else
	
	#define weakify(object) autoreleasepool{} __block __typeof__(object) block##_##object = object;
	
	#endif      // 结束 if _has_feature()
	
	#else
	
	#if __has_feature(objc_arc)
	
	#define weakify(object) try{} @finally{} {} __weak __typeof__(object) weak##_##object = object;
	
	#else
	
	#define weakify(object) try{} @finally{} {} __block __typeof__(object) block##_##object = object;
	
	#endif      // 结束 if _has_feature()
	
	#endif      // 结束 if DEBUG
	
	#endif      // 结束 ifndef weakify
	
	#ifndef strongify
	
	#if DEBUG
	
	#if __has_feature(objc_arc)
	
	#define strongify(object) autoreleasepool{} __typeof__(object) object = weak##_##object;
	
	#else
	
	#define strongify(object) autoreleasepool{} __typeof__(object) object = block##_##object;
	
	#endif
	
	#else
	
	#if __has_feature(objc_arc)
	
	#define strongify(object) try{} @finally{} __typeof__(object) object = weak##_##object;
	
	#else
	
	#define strongify(object) try{} @finally{} __typeof__(object) object = block##_##object;、#endif
	
	#endif
	
	#endif
	```

3. 总结

	```
	@weakify(self) = @autoreleasepool{} __weak __typeof__ (self) self_weak_ = self;
	
	@strongify(self) = @autoreleasepool{} __strong __typeof__(self) self = self_weak_;
	```
	
	经过分析以后，其实 @weakify(self) 和 @strongify(self) 就是比我们日常写的 weakSelf、strongSelf 多了一个 @autoreleasepool{} 而已，至于为何要用这些复杂的宏定义来做，目前我还没有理解。如果有大神指导其中的原因，还请多多指点。

## 八、问题

1. block 原理是什么？本质是什么？

	**封装了函数调用以及调用环境的 OC 对象。**

2. Block 属性的修饰词为什么是 copy？\_\_block 的作用是什么？有什么注意的点？

	一旦没有进行 copy 操作，block 就不会在堆上。\_\_block 能够修改自动变量的值。注意循环引用。

3. block 修改 NSMutableArray 时，是否需要添加 \_\_block？

	```
	{
	    NSMutableArray * mArr1 = [NSMutableArray arrayWithObjects:@"a", @"b", @"abc", nil];
	    NSMutableArray * mArr2 = [NSMutableArray arrayWithCapacity:mArr1.count];
	    
	    [mArr1 enumerateObjectsUsingBlock: ^(NSString * obj, NSUInteger idx, BOOL *stop){
	        [mArr2 addObject:@(obj.length)];
	    }];
	    
	    NSLog(@"%@", mArr2);
	}
	
	2018-12-03 00:24:41.754700+0800 Demo[13081:1899280] (
	    1,
	    1,
	    3
	)
	```

	这里确实没有修改 mArr2 这个局部变量。mArr2 是一个指针，指向一个可变长度的数组。在 block 里面，并没有修改这个指针，而是修改了这个指针指向的数组。换句话说，mArr2 保存的是一块内存区域的地址，在 block 里，并没有改变这个地址，而是读取出这个地址，然后去操作这块地址空间的内容。 因为声明 block 的时候实际上是把当时的临时变量又复制了一份，在 block 里即使修改了这些复制的变量，也不影响外面的原始变量。即所谓的闭包。 但是当变量是一个指针的时候，block 里只是复制了一份这个指针，两个指针指向同一个地址。所以，在 block 里面对指针指向内容做的修改，在 block 外面也一样生效。  

## 九、学习文章

[宁夏灼雪__](https://www.jianshu.com/u/edda0ce4a193) & [iOS底层day6 - 探索block](https://www.jianshu.com/p/460c9f43d20f)
[http://blog.csdn.net/qq_30513483/article/details/52587551](http://blog.csdn.net/qq_30513483/article/details/52587551)