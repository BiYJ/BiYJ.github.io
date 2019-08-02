---
title: 消息转发
categories: iOS原理
---

## 一、前言

在开发过程中，可能遇到服务端返回数据中有 null，当取到 null 值并对 null 发送消息的时候，就可能出现  unrecognized selector sent to instance，导致应用 crash 的情况。

针对这种情况，在每次取值的时候去做判断处理又不大合适，在 GitHub上发现了 [NullSafe](https://github.com/nicklockwood/NullSafe)。把这个文件拖到项目中，即使出现 null 的情况，也不会报出 unrecognized selector sent to instance 的问题。

消息转发的整个过程主要涉及的 3 个方法：

```objc
+(BOOL)resolveInstanceMethod:(SEL)sel;
-(id)forwardingTargetForSelector:(SEL)aSelector;
-(void)forwardInvocation:(NSInvocation*)anInvocation;
```

其中在 +(BOOL)resolveInstanceMethod:(SEL)sel 的时候，会有相应的方法缓存操作，这个操作是系统帮我们做的。


## 二、消息转发过程

首先贴一张消息转发的图，笔者聊到的内容会围绕着这张图展开。

![](https://upload-images.jianshu.io/upload_images/5294842-21f16eb1cfa08e77.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下边分析消息转发的过程，以 [MyObjet Length] 为例：

①、首先 MyObjet 在调用 length 方法后，会先进行动态方法解析，调用 +(BOOL)resolveInstanceMethod:(SEL)sel，我们可以在这里动态添加方法，而且如果在这里动态添加方法成功后，系统会把动态添加的 length 方法进行缓存，当 MyObjet 再次调用 length 方法的时候，将不会调用 +(BOOL)resolveInstanceMethod:(SEL)sel。会直接调用动态添加成功的 length 方法。

②、如果动态方法解析部分没有做操作，或者动态添加方法失败了的话，会进行寻找备援接收者的过程 -(id)forwardingTargetForSelector:(SEL)aSelector，这个过程用于寻找一个接收者，可以响应未知的方法。

③、如果寻找备援接收者的过程中返回值为 nil 的话，那么会进入到完整的消息转发流程中。完整的消息转发流程：首先创建 NSInvocation 对象，把与尚未处理的那条消息有关的全部细节都封于其中，此对象包含选择子、目标（target）及参数。在出发 NSInvocation 对象时，“消息派发系统”（message-dispatch system）将亲自出马，把消息指派给目标对象。


## 三、结合 MyObject 中的代码对消息转发流程进一步分析

①、先看第一部分 MyObject 在调用 length 方法后，会先进行动态方法解析，调用 +(BOOL)resolveInstanceMethod:(SEL)sel，如果我们在这里为 MyObject 动态添加方法。那么也能处理消息。相关代码如下：

```objc
+ (BOOL)resolveInstanceMethod:(SEL)sel 
{    
    printf("%s:%s \n", __func__ ,NSStringFromSelector(sel).UTF8String);

    if (sel == @selector(length)) {
         BOOL success = class_addMethod([self class], sel, (IMP)(length), "q@:"); 

         if (success) {
             return success;
         }
    }
    return [super resolveInstanceMethod:sel];
}
```

传入的 "q@:" 分别代表：

```objc
q : 返回值 long long
@ : 调用方法的的实例为对象类型
: : 表示方法
```

下图表示了编码类型。

![](https://upload-images.jianshu.io/upload_images/5294842-a5925ae21f498603.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

②、MyObject 在调用 length 方法后，动态方法解析部分如果返回值为 NO 的时候，会寻找备援接收者，调用 -(id)forwardingTargetForSelector:(SEL)aSelector，如果我们在这里为返回可以处理 length 的接收者。那么也能处理消息。相关代码如下：

```objc
static NSArray * respondClasses;

- (id)forwardingTargetForSelector:(SEL)aSelector {

    printf("%s:%s \n", __func__ , NSStringFromSelector(aSelector).UTF8String);

    id forwardTarget = [super forwardingTargetForSelector:aSelector];
    if (forwardTarget) {
        return forwardTarget;
    }

    Class someClass = [self myResponedClassForSelector:aSelector];
    if (someClass) {
        forwardTarget = [someClass new];
    }

    return forwardTarget;
}


- (Class)myResponedClassForSelector:(SEL)selector
{
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

\+(BOOL)instancesRespondToSelector:(SEL)aSelector; 用于返回 Class 对应的实例能否响应 aSelector。

③、MyObject 在调用 length 方法后，动态方法解析部分如果返回值为 NO 的时候，寻找备援接收者的返回值为 nil 的时候，会进行完整的消息转发流程。调用 -(void)forwardInvocation:(NSInvocation \*)anInvocation，这个过程会有一个插曲 -(NSMethodSignature \*)methodSignatureForSelector:(SEL)selector，只有我们返回了相应地 NSMethodSignature 实例的时候，完整地消息转发流程才能得以顺利完成。

>  -(NSMethodSignature*)methodSignatureForSelector:(SEL)selector。
> 
> 摘抄自文档：This method is used in the implementation of protocols. This method is also used in situations where an NSInvocation object must be created, such as during message forwarding.If your object maintains a delegate or is capable of handling messages that it does not directly implement, you should override this method to return an appropriate method signature.
> 
> 这个方法也会用于消息转发的时候，当 NSInvocation 对象必须创建的时候，如果我们的对象能够处理没有直接实现的方法，我们应该重写这个方法，返回一个合适的方法签名。

相关代码

```objc
- (void)forwardInvocation:(NSInvocation *)anInvocation
{

    printf("%s:%s \n\n\n\n", __func__ , NSStringFromSelector(anInvocation.selector).UTF8String);

    anInvocation.target = nil;
    [anInvocation invoke];
}


- (NSMethodSignature *)methodSignatureForSelector:(SEL)selector {

    NSMethodSignature *signature = [super methodSignatureForSelector:selector];
    if (!signature) {
        Class responededClass = [self myResponedClassForSelector:selector];
        if (responededClass) {
            @try {
                signature = [responededClass instanceMethodSignatureForSelector:selector];
            } @catch (NSException *exception) {

            }@finally {

            }
        }
    }
    return signature;
}

- (Class)myResponedClassForSelector:(SEL)selector {

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

这里有一个不常用的 API：+(NSMethodSignature \*)instanceMethodSignatureForSelector:(SEL)aSelector;，这个 API 通过 Class 及给定的 aSelector 返回一个包含实例方法标识描述的方法签名实例。

```objc
<NSMethodSignature: 0x6000030a17c0>
    number of arguments = 2
    frame size = 224
    is special struct return? NO
    return value: -------- -------- -------- --------
        type encoding (f) 'f'
        flags {isFloat}
        modifiers {}
        frame {offset = 16, offset adjust = 0, size = 16, size adjust = -12}
        memory {offset = 0, size = 4}
    argument 0: -------- -------- -------- --------
        type encoding (@) '@'
        flags {isObject}
        modifiers {}
        frame {offset = 0, offset adjust = 0, size = 8, size adjust = 0}
        memory {offset = 0, size = 8}
    argument 1: -------- -------- -------- --------
        type encoding (:) ':'
        flags {}
        modifiers {}
        frame {offset = 8, offset adjust = 0, size = 8, size adjust = 0}
        memory {offset = 0, size = 8}
```        

NSInvocation。

仍然以`myObject`调用`length`方法为例。 \- (void)forwardInvocation:(NSInvocation \*)anInvocation中的 anInvocation 的信息如下：

```objc
<NSInvocation: 0x6000025b8140>
return value: {Q} 0
target: {@} 0x60000322c360
selector: {:} length

> return value 指返回值，"Q" 表示返回值类型为 long long 类型；
> target 指的是消息的接收者，"@"标识对象类型；
> selector 指的是方法，":"表示是方法，后边的 length 为方法名。
```

更多内容可见下图 NSInvocation 的 types：

```objc
enum _NSObjCValueType {
    NSObjCNoType = 0,
    NSObjCVoidType = 'v',
    NSObjCCharType = 'c',
    NSObjCShortType = 's',
    NSObjCLongType = 'l',
    NSObjCLonglongType = 'q',
    NSObjCFloatType = 'f',
    NSObjCDoubleType = 'd',
    NSObjCBoolType = 'B',
    NSObjCSelectorType = ':',
    NSObjCObjectType = '@',
    NSObjCStructType = '{',
    NSObjCPointerType = '^',
    NSObjCStringType = '*',
    NSObjCArrayType = '[',
    NSObjCUnionType = '(',
    NSObjCBitfield = 'b'
} API_DEPRECATED("Not supported", macos(10.0,10.5), ios(2.0,2.0), watchos(2.0,2.0), tvos(9.0,9.0));
```

## 四、尚存疑点

细心的读者可能会发现在首次消息转发的时候流程并不是

```objc
+[MyObject resolveInstanceMethod:]:length 
-[MyObject forwardingTargetForSelector:]:length 
-[MyObject forwardInvocation:]:length 
```

而是

```objc
+[MyObject resolveInstanceMethod:]:length 
-[MyObject forwardingTargetForSelector:]:length 
+[MyObject resolveInstanceMethod:]:length 
+[MyObject resolveInstanceMethod:]:_forwardStackInvocation: 
-[MyObject forwardInvocation:]:length 
```

查看了开源源码 [NSObject.mm](https://opensource.apple.com/source/objc4/objc4-750.1/runtime/NSObject.mm.auto.html) 相关源码如下：

```
// Replaced by CF (returns an NSMethodSignature)
- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    _objc_fatal("-[NSObject methodSignatureForSelector:] "
                "not available without CoreFoundation");
}
- (void)forwardInvocation:(NSInvocation *)invocation {
    [self doesNotRecognizeSelector:(invocation ? [invocation selector] : 0)];
}
// Replaced by CF (throws an NSException)
- (void)doesNotRecognizeSelector:(SEL)sel {
    _objc_fatal("-[%s %s]: unrecognized selector sent to instance %p", 
                object_getClassName(self), sel_getName(sel), self);
}
```


## 五、NSNull+QiNullSafe.m

根据 [NullSafe](https://github.com/nicklockwood/NullSafe) 仿写的 [NSNull+QiNullSafe.m](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2FQiShare%2FQiSafeType%2Fblob%2Fmaster%2FQiSafeType%2FNullSafe%2FNSNull%252BQiNullSafe.m)。

NSNull+QiNullSafe.m 能够避免的问题有：

> NSNull *null = [NSNull null];
> 
> [null performSelector:@selector(addObject:) withObject:@"aaa"];
> [null performSelector:@selector(setValue:forKey:) withObject:@"aaa"];
> [null performSelector:@selector(valueForKey:) withObject:@"aaa"];
> [null performSelector:@selector(length) withObject:nil];
> [null performSelector:@selector(integerValue) withObject:nil];
> [null performSelector:@selector(timeIntervalSinceNow) withObject:nil];
> [null performSelector:@selector(bytes) withObject:nil];


## 六、NullSafe 是怎么处理 null 问题

其实 [NullSafe](https://github.com/nicklockwood/NullSafe)  处理 null 问题用的是消息转发的第三部分，走的是完整地消息转发流程。

不过我们开发过程中，如果可以的话，还是尽可能早地处理消息转发这部分，比如在动态方法解析的时候，动态添加方法（毕竟这一步系统可以为我们做方法的缓存处理）。 或者是在寻找备援接收对象的时候，返回能够响应未实现的方法的对象。

注意：相关的使用场景在测试的时候不要用，测试的时候尽可能还是要暴露出问题的。并且使用的时候，最好结合着异常日志上报。


## 七、单元测试

```objc
- (void)testStringValue
{
    id null = [NSNull null];
    
    NSString * string = [null stringValue];
    
    XCTAssertNil(string);
}

- (void)testFloatValue
{
    id null = [NSNull null];
    
    CGFloat f = [null floatValue];
    
    XCTAssertEqualWithAccuracy(f, 0.0f, 0.0f);
}

- (void)testPerformSelector
{
    NSNull * null = [NSNull null];
    [null performSelector:@selector(addObject:) withObject:@"aaa"];
}
```

## 八、文章

[iOS 消息转发](https://juejin.im/post/5c6e773be51d451b25716d0e)

[Protocol 协议分发器](http://www.olinone.com/?p=643)
