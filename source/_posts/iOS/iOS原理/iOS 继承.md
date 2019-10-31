---
title: iOS继承
categories: iOS原理
---

> 万不得已不要用继承，优先考虑组合等方式。

* 如果只是共享接口，我们可以使用<font color=#cc0000>协议</font>；

	```
	@protocol ptc <NSObject>
	- (void)do;
	@end
	
	
	@interface A : NSObject <ptc>
	@end
	
	@implementation A 
	
	- (void)do
	{
	
	}
	
	@end
	
	
	@interface B : NSObject <ptc>
	@end
	
	@implementation B
	
	- (void)do
	{
	
	}
	
	@end
	```

* 如果希望共用一个方法的部分实现，但希望根据需要执行不同的其他行为，我们可以使用<font color=#cc0000>代理</font>或者 <font color=#cc0000>AOP</font>；

	```
	@protocol ptc <NSObject>
	- (void)do;
	@end
	
	
	@interface A : NSObject 
	@property (nonatomic, weak) id<ptc> delegate;
	@end
	
	@implementation A
	
	- (void)func
	{
		...
		
		[self.delegate do];
		
		...
	}
	
	@end
	
	
	@interface B : NSObject <ptc>
	@end
	
	@implementation B
	
	- (void)do
	{
	
	}
	
	@end
	```

* 如果是添加方法，我们可以优先使用<font color=#cc0000>类别</font>；



* 如果是为了使用一个类的很多方法，我们可以使用<font color=#cc0000>组合</font>来实现。

	```
	@interface A : NSObject
	- (void)methodA;
	@end
	
	
	@interface B : NSObject	
	-(void)methodB;
	@end
	
	
	// 定义 C 以及其需要的 methodA，methodB
	@interface C : NSObject 
	{
		A * __a;
		B * __b;
	}
	- (id)initWithA:(A *)a b:(B *)b;
	- (void)methodA;
	- (void)methodB;
	@end
		
	@implementation  ClassC
	
	- (id)initWithA:(A *)a b:(B *)b
	{
		__a = [[A alloc] initWithA:a];  // [A copy];
		__b = [[B alloc] initWithB:b];  // [B copy];
	}
	
	- (void)methodA
	{
		[__a methodA];
	}
	
	- (void)methodB
	{
		[__b methodB];
	}
	
	@end
	```
	
如果只是出于代码复用的目的而不区分类别和场景，就采用继承是不恰当的。当你发现你的继承超过 2 层的时候，你就要好好考虑是否这个继承的方案了，第三层继承正是滥用的开端。

[跳出面向对象思想(一) 继承](https://casatwy.com/tiao-chu-mian-xiang-dui-xiang-si-xiang-yi-ji-cheng.html)