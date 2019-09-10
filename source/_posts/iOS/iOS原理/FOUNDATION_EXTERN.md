---
title: FOUNDATION_EXTERN
categories: C
---

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