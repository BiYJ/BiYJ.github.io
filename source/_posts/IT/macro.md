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