---
title: Autorelease
categories: iOS原理
---

## 一、什么是自动释放池

OC 中的一种内存自动回收机制，它可以延迟加入 AutoreleasePool 中的变量 release 的时机，即当我们创建了一个对象，并把它加入到了自动释放池中时，它不会立即被释放，会等到一次 runloop 结束或者作用域超出 {} 或者超出 [pool release] 之后再被释放。

## 二、自动释放池的创建与销毁时机

1. MRC：
	
	```
	NSAutoreleasePool * pool = [[NSAutoreleasePool alloc] init];
	Person * p = [[Person alloc] init];
	// 调 autorelease 方法将对象加入到自动释放池
	[person autorelease];
	
	// 手动释放自动释放池时，自动释放池会对加入池子中的对象做一次 release 操作
	[pool release];
	```

	自动释放池销毁时机：[pool release] 代码执行完后。

2. ARC

	```
	@autoreleasepool {
	    // 在 {} 之内的变量默认被添加到自动释放池
	    Person * p = [[Person alloc] init];
	}
	// 出了 {}，p 被释放
	```

## 三、Autorelease实现原理

```
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

然后在终端中使用 `clang -rewrite-objc main.m` 命令将上述 OC 代码重写成 C++ 的实现。

```
int main(int argc, char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        return UIApplicationMain(argc, argv, __null, NSStringFromClass(((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("AppDelegate"), sel_registerName("class"))));
    }
}
```

苹果通过声明一个 \_\_AtAutoreleasePool 类型的局部变量，@autoreleasepool 被转换成 \_\_AtAutoreleasePool 结构体类型

```
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};
```

可以看到 \_\_AtAutoreleasePool() 构造函数调用 objc\_autoreleasePoolPush()，~\_\_AtAutoreleasePool() 析构函数调用 objc\_autoreleasePoolPop()。

> objc\_autoreleasePoolPush 和 objc\_autoreleasePoolPop 是什么呢？

在 NSObject.mm 文件中：

```
void *objc_autoreleasePoolPush(void) {
        return AutoreleasePoolPage::push();
    }
    
void objc_autoreleasePoolPop(void *ctxt) {
        AutoreleasePoolPage::pop(ctxt);
    }
```

实际上是调用 AutoreleasePoolPage 的 push 和 pop 两个类方法。

```
class AutoreleasePoolPage {
        magic_t const magic;
        id *next;
        pthread_t const thread;
        AutoreleasePoolPage * const parent;
        AutoreleasePoolPage *child;
        uint32_t const depth;
        uint32_t hiwat;
    };
```    

* magic：用来校验 AutoreleasePoolPage 的结构是否完整；
* next：指向栈顶，也就是最新入栈的autorelease对象的下一个位置；
* thread：指向当前线程；
* parent：指向父节点
* child：指向子节点
* depth：表示链表的深度，也就是链表节点的个数
* hiwat：表示high water mark（最高水位标记）

每一个 AutoreleasePoolPage 都是以双链表的形式连接起来的。

<center>
![](http://dzliving.com/Autorelease_0.png)
</center>

parent 指向前一个 page , child 指向下一个 page

#### 3.1 push

<center>
![](http://dzliving.com/Autorelease_push.png)
</center>

一个 push 操作其实就是创建一个新的 autoreleasepool，对应 AutoreleasePoolPage 的具体实现就是往 AutoreleasePoolPage 中的 next 位置插入一个 POOL\_SENTINEL，并且返回插入的 POOL\_SENTINEL 的内存地址。

执行一个具体的插入操作时，分别对三种情况进行了不同的处理：

1. 当前 page 存在且没有满时，直接将对象添加到当前 page 中，即 next 指向的位置；
2. 当前 page 存在且已满时，创建一个新的 page ，并将对象添加到新创建的 page 中；
3. 当前 page 不存在时，即还没有 page 时，创建第一个 page ，并将对象添加到新创建的 page 中。

每调用一次 push 操作就会创建一个新的 AutoreleasePoolPage ，即往 AutoreleasePoolPage 中插入一个 POOL\_SENTINEL ，并且返回插入的 POOL\_SENTINEL 的内存地址。

#### 3.2 pop

<center>
![](http://dzliving.com/Autorelease_pop.png)
</center>

pop 函数的入参就是 push 函数的返回值，也就是 POOL\_SENTINEL 的内存地址即 pool token 。当执行 pop 操作时，内存地址在 pool token 之后的所有 autoreleased 对象都会被 release 。直到 pool token 所在 page 的 next 指向 pool token 为止。


## 文章

[河西七夕](https://www.jianshu.com/u/55c11e0806c4) - [ios 自动释放池](https://www.jianshu.com/p/9dad9c3247ed)