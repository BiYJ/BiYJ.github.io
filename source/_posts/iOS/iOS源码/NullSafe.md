---
title: NullSafe
categories: iOS源码
---

Github：[NullSafe](https://github.com/nicklockwood/NullSafe)、[QiSafeType](https://github.com/QiShare/QiSafeType)

## 一、NullSafe.m

这个文件是 NSNull 的分类，没有 .h 文件，用的是消息转发功能，逻辑较为简单。以下为代码：

```
// https://github.com/nicklockwood/NullSafe

#import <objc/runtime.h>
#import <Foundation/Foundation.h>


#ifndef NULLSAFE_ENABLED
#define NULLSAFE_ENABLED 1
#endif


#pragma clang diagnostic ignored "-Wgnu-conditional-omitted-operand"

@implementation NSNull (NullSafe)

#if NO

#if NULLSAFE_ENABLED

- (NSMethodSignature *)methodSignatureForSelector:(SEL)selector
{
    //look up method signature
    NSMethodSignature *signature = [super methodSignatureForSelector:selector];

    // 如果没有方法签名
    if (!signature)
    {
        for (Class someClass in @[
            [NSMutableArray class],
            [NSMutableDictionary class],
            [NSMutableString class],
            [NSNumber class],
            [NSDate class],
            [NSData class]
        ])
        {
            @try
            {
                // 实例是否能够响应 selector（即在 class 中是否能找到该方法）
                if ([someClass instancesRespondToSelector:selector])
                {
                    signature = [someClass instanceMethodSignatureForSelector:selector];
                    break;
                }
            }
            @catch (__unused NSException *unused) {
                
            }
        }
    }
    return signature;
}

- (void)forwardInvocation:(NSInvocation *)invocation
{
    invocation.target = nil;

    // 调用
    [invocation invoke];
}

- (void)setValue:(id)value forUndefinedKey:(NSString *)key
{
    
}

- (id)valueForUndefinedKey:(NSString *)key
{
    return nil;
}

#endif
#endif
@end
```

## 二、QiSafeType

1. NSNull+QiNullSafe.m

	与 NullSafe.m 的代码逻辑基本一样，新增了一方法实现 for 循环的逻辑，比较明显的区别是没有用 try-catch。

	```
	- (Class)qiResponedClassForSelector:(SEL)selector {
	    
	    respondClasses = @[
	                       [NSMutableArray class],
	                       [NSMutableDictionary class],
	                       [NSMutableString class],
	                       [NSNumber class],
	                       [NSDate class],
	                       [NSData class]
	                       ];
	    for (Class someClass in respondClasses) {
	        if ([someClass instancesRespondToSelector:selector]) {
	            return someClass;
	        }
	    }
	    return nil;
	}
	```

2. QiAvoidCommonCrash

	这部分功能可以查看 [AvoidCrash](https://github.com/chenfanfang/AvoidCrash)。通过 runtime 的 method-swizzle 实现方法实现的替换，然后对参数进行判断纠错。