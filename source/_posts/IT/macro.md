---
title: macro
categories: IT
---

[Hello, 宏定义魔法世界](https://www.jianshu.com/p/4a1531bac39f)

## 宏定义和全局变量的区别

1. 宏会在预处理阶段被替换，而全局变量是在运行时；
2. 宏定义不分配内存，全局变量定义需要分配内存；
3. 宏不区分数据类型，它本质上是一段字符，在预处理的时候被替换到引用的位置，而全局变量区分数据类型；
4. 宏定义之后值是不能改变的，全局变量的值是可以改变的；
5. 宏定义只有在定义所在文件，或引用所在文件的其它文件中使用。 而全局变量可以在工程所有文件中使用，只需在使用前加一个声明。

## FOUNDATION_EXTERN

```
#if defined(__cplusplus)
#define FOUNDATION_EXTERN extern "C"
#else
#define FOUNDATION_EXTERN extern
#endif
```

由以上定义可以看出 FOUNDATION\_EXTERN 是可以兼容 C++ 的 extern 的宏。

```
FOUNDATION_EXTERN NSString * Extern_S = @"sss";

#define Define_S @"sss"

{
	NSString * s = @"tt";
	    
	CFTimeInterval begin = CACurrentMediaTime();
	    
	for (int i = 0; i < 10000; i++) {
		if ([s isEqualToString:Define_S]) {
		    
		}
	}
	NSLog(@"%f", CACurrentMediaTime() - begin);
	    
	begin = CACurrentMediaTime();
	    
	for (int i = 0; i < 10000; i++) {
		if ([s isEqualToString:Extern_S]) {
		    
		}
	}
	NSLog(@"%f", CACurrentMediaTime() - begin);
}

2019-09-09 16:06:04.849874+0800 Demo[86518:3255355] 0.000148
2019-09-09 16:06:04.850170+0800 Demo[86518:3255355] 0.000137
```

extern 比宏在字符串上的比较速度要快一些，因为 extern 直接比较指针地址，而宏是比较字符串是否相等。

## \_\_builtin\_expect

这个其实是个函数，针对编译器优化的一个函数。

写代码中我们经常会遇到条件判断语句

```
if (今天是工作日) {
    printf("好好上班");
}
else {
    printf("好好睡觉");
}
```

CPU 读取指令的时候并非一条一条的来读，而是多条一起加载进来，比如已经加载了 `if(今天是工作日) printf(“好好上班”);` 的指令，这时候条件式如果为非，也就是非工作日，那么 CPU 继续把 `printf(“好好睡觉”);` 这条指令加载进来，这样就造成了性能浪费的现象。

\_\_builtin\_expect 的第一个参数是实际值，第二个参数是预测值。使用这个目的是告诉编译器 if 条件式是不是有更大的可能被满足。


## likely 和 unlikely

解开这个宏后其实是对 \_\_builtin\_expect 封装，likely 表示更大可能成立，unlikely 表示更大可能不成立。

```
#define likely(x) __builtin_expect(!!(x), 1)
#define unlikely(x) __builtin_expect(!!(x), 0)
```

遇到这样的，if(likely(a == 0)) 理解成 if(a==0) 即可，unlikely 也是同样的。

## fastpath 和 slowpath

跟上面也是差不多的，fastpath 表示更大可能成立，slowpath 表示更大可能不成立

```
#define fastpath(x) ((typeof(x))__builtin_expect(_safe_cast_to_long(x), ~0l))
#define slowpath(x) ((typeof(x))__builtin_expect(_safe_cast_to_long(x), 0l))
```

这两个理解起来跟 likely 和 unlikely 一样，只需要关注里面的条件式是否满足即可。

## os\_atomic\_cmpxchg

其内部就是 atomic\_compare\_exchange\_strong\_explicit 函数，这个函数的作用是：第二个参数与第一个参数值比较，如果相等，第三个参数的值替换第一个参数的值。如果不相等，把第一个参数的值赋值到第二个参数上。

```
#define os_atomic_cmpxchg(p, e, v, m) \
        ({ _os_atomic_basetypeof(p) _r = (e); \
        atomic_compare_exchange_strong_explicit(_os_atomic_c11_atomic(p), \
        &_r, v, memory_order_##m, memory_order_relaxed); })
```
        
## os\_atomic\_store2o

将第二个参数，保存到第一个参数

```
#define os_atomic_store2o(p, f, v, m)  os_atomic_store(&(p)->f, (v), m)
#define os_atomic_store(p, v, m) \
      atomic_store_explicit(_os_atomic_c11_atomic(p), v, memory_order_##m)
```

## os\_atomic\_inc\_orig

将 1 保存到第一个参数中

```
#define os_atomic_inc_orig(p, m)  os_atomic_add_orig((p), 1, m)
#define os_atomic_add_orig(p, v, m) _os_atomic_c11_op_orig((p), (v), m, add, +)
#define _os_atomic_c11_op_orig(p, v, m, o, op) \
        atomic_fetch_##o##_explicit(_os_atomic_c11_atomic(p), v, \
        memory_order_##m)
```

[UNAVAILABLE\_ATTRIBUTE , \_\_has\_include](https://www.jianshu.com/p/3fb6033b06a4)
