---
title: 重构 AppDelegate
categories: iOS优化
---

## 一、Massive AppDelegate

> AppDelegate 是应用程序的根对象，<font color=#cc0000>它连接应用程序和系统，确保应用程序与系统以及其他应用程序正确的交互</font>，通常被认为是每个 iOS 项目的核心。

随着开发的迭代升级，不断增加新的功能和业务，它的代码量也不断增长，最终导致了 Massive AppDelegate。

在复杂 AppDelegate 里修改任何东西的成本都是很高的，因为它将会影响你的整个 APP，一不留神产生 bug。毫无疑问，保持 AppDelegate 的<font color=#cc0000>简洁和清晰</font>对于健康的 iOS 架构来说是至关重要的。本文将使用多种方法来重构，使之简洁、可重用和可测。

AppDelegate 常见的业务代码如下：

*   日志埋点统计数据分析
*   初始化数据存储系统
*   配置 UIAppearance
*   管理 App Badge 数字
*   管理通知：请求权限，存储令牌，处理自定义操作，将通知传播到应用程序的其余部分
*   管理 UI 堆栈配置：选择初始视图控制器，执行根视图控制器转换
*   管理 UserDefaults：设置首先启动标志，保存和加载数据
*   管理后台任务
*   管理设备方向
*   更新位置信息
*   初始化第三方库（如分享、日志、第三方登陆、支付）

这些臃肿的代码是反模式的，导致难于维护，显然支持扩展和测试这样的类非常复杂且容易出错。Massive AppDelegates 与我们经常谈的 Massive ViewController 的症状非常类似。

看看以下可能的解决方案，每个 Recipe（方案）<font color=#cc0000>遵循单一职责、易于扩展、易于测试原则</font>。



## 二、命令模式 Command Design Pattern

> 命令模式是一种数据驱动的设计模式，属于行为型模式。

请求以命令的形式包裹在对象中，并传给调用对象。调用对象寻找可以处理该命令的合适的对象，并把该命令传给相应的对象，该对象执行命令。因此命令的调用者无需关心命令做了什么以及响应者是谁。

可以为 AppDelegate 的每一个职责定义一个命令，这个命令的名字自行指定。

```objc
/// 命令协议
@protocol Command <NSObject>
- (void)execute;
@end

/// 初始化第三方库
@interface InitializeThirdPartiesCommand : NSObject <Command>

@end

/// 初始化主视图
@interface InitializeRootViewControllerCommand : NSObject <Command>
@property (nonatomic, strong) UIWindow * keyWindow;
@end

/// 初始化视图全局配置
@interface InitializeAppearanceCommand : NSObject <Command>

@end

/// ...
```

然后定义一个统一调用的类 StartupCommandsBuilder 来封装如何创建命令的详细信息。AppDelegate 调用这个 builder 去初始化命令并执行这些命令。

```objc
@implementation StartupCommandsBuilder

// 返回数组，元素为遵守 Command 协议的对象
- (NSArray<id<Command>> *)build
{
    return @[ [InitializeAppearanceCommand new], 
              [InitializeRootViewControllerCommand new], 
              [InitializeThirdPartiesCommand new] ];
}

@end
```

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{    
    [[[[StartupCommandsBuilder alloc] init] build] enumerateObjectsUsingBlock:^(id<Command> _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
        [obj execute];
    }];
    
    return YES;
}
```

如果 AppDelegate 需要添加新的职责，则可以创建新的命令，然后把命令添加到 Builder 里而无需去改变 AppDelegate。解决方案满足单一职责、易于扩展、易于测试原则。


## 三、组合设计模式 Composite Design Pattern

> 组合模式又叫部分整体模式，用于把一组相似的对象当作一个单一的对象。

组合模式依据树形结构来组合对象，用来表示部分以及整体层次。这种类型的设计模式属于<font color=#cc0000>结构型模式</font>，它创建了对象组的树形结构。一个很明显的例子就是 iOS 里的 UIView 以及它的 subviews。

这个想法主要是有一个组装类和叶子类，每个叶子类负责一个职责，而组装类负责调用所有叶子类的方法。

```objc
/// 组装类
@interface CompositeAppDelegate : UIResponder <UIApplicationDelegate>
+ (instancetype)makeDefault;
@end

@implementation CompositeAppDelegate

+ (instancetype)makeDefault
{
    // 这里要实现单例
    return [[CompositeAppDelegate alloc] init];
}

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    [[PushNotificationAppDelegate new] application:application didFinishLaunchingWithOptions:launchOptions];
    [[ThirdPartiesConfiguratorAppDelegate new] application:application didFinishLaunchingWithOptions:launchOptions];

    return YES;
}

@end
```

实现执行具体职责的叶子类。

```objc
/// 叶子类。推送消息处理
@interface PushNotificationAppDelegate : UIResponder <UIApplicationDelegate>

@end

/// 叶子类。初始化第三方库
@interface ThirdPartiesConfiguratorAppDelegate : UIResponder <UIApplicationDelegate>

@end


@implementation PushNotificationAppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    NSLog(@"PushNotificationAppDelegate");
    
    return YES;
}

@end



@implementation ThirdPartiesConfiguratorAppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    NSLog(@"ThirdPartiesConfiguratorAppDelegate");
    
    return YES;
}

@end
```

在 AppDelegate 通过工厂方法创建组装类，然后通过它去调用所有的方法

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    [[CompositeAppDelegate makeDefault] application:application didFinishLaunchingWithOptions:launchOptions];
    
    return YES;
}
```

它满足我们在开始时提出的所有要求，如果要添加一个新的功能，很容易添加一个叶子类，无需改变 AppDelegate，解决方案满足单一职责、易于扩展、易于测试原则。


## 四、中介者模式 Mediator Design Pattern

> 中介者模式是用来降低多个对象和类之间的通信复杂性。

这种模式提供了一个中介类，该类通常处理不同类之间的通信，并支持松耦合，使代码易于维护。中介者模式属于<font color=#cc0000>行为型模式</font>。

如果想了解有关此模式的更多信息，建议查看 Mediator Pattern Case Study。或者阅读文末给出关于设计模式比较经典的书籍。

让我们定义 AppLifecycleMediator 将 UIApplication 的生命周期通知底下的监听者，这些监听者必须遵循AppLifecycleListener 协议，如果需要监听者要能扩展新的方法。

```objc
@interface APPLifeCycleMediator : NSObject

+ (instancetype)makeDefaultMediator;

@end


@implementation APPLifeCycleMediator
{
    @private
        NSArray<id<AppLifeCycleListener>> * _listeners;
}
- (void)dealloc
{
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

- (instancetype)initWithListeners:(NSArray<id<AppLifeCycleListener>> *)listeners
{
    if (self = [super init]) {
        
        _listeners = listeners;
        
        // 通知
        [[NSNotificationCenter defaultCenter] addObserver:self
                                                 selector:@selector(onAppWillEnterForeground)
                                                     name:UIApplicationWillEnterForegroundNotification
                                                   object:nil];
        [[NSNotificationCenter defaultCenter] addObserver:self
                                                 selector:@selector(onAppDidEnterBackgroud)
                                                     name:UIApplicationDidEnterBackgroundNotification
                                                   object:nil];
        [[NSNotificationCenter defaultCenter] addObserver:self
                                                 selector:@selector(onAppDidFinishLaunching)
                                                     name:UIApplicationDidFinishLaunchingNotification
                                                   object:nil];
    }
    
    return self;
}

/// 定义好静态类方法，初始化所有监听者
+ (instancetype)makeDefaultMediator
{
    static APPLifeCycleMediator * mediator;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        mediator = [[APPLifeCycleMediator alloc] initWithListeners:@[[VideoListener new], [SocketListener new]]];
    });
    return mediator;
}

- (void)onAppWillEnterForeground
{
    [_listeners[1] onAppWillEnterForeground];
}

- (void)onAppDidEnterBackgroud
{
    [_listeners[0] onAppDidEnterBackgroud];
}

- (void)onAppDidFinishLaunching
{

}

@end
```

定义 AppLifecycleListener 协议，以及协议的的实现者。

```objc
/// 监听协议
@protocol AppLifeCycleListener <NSObject>
@optional
- (void)onAppWillEnterForeground;
- (void)onAppDidEnterBackgroud;
- (void)onAppDidFinishLaunching;

@end

@interface VideoListener : NSObject <AppLifeCycleListener>

@end


@interface SocketListener : NSObject <AppLifeCycleListener>

@end


@implementation VideoListener

- (void)onAppDidEnterBackgroud
{
    NSLog(@"停止视频播放");
}

@end

@implementation SocketListener

- (void)onAppWillEnterForeground
{
    NSLog(@"开启长链接");
}

@end
```

加入到 AppDelegate 中

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    [APPLifeCycleMediator makeDefaultMediator];
    
    return YES;
}
```

这个中介者自动订阅了所有的事件。AppDelegate 仅仅需要初始化它一次，就能让它正常工作。每个监听者都有一个单一职责，很容易添加一个监听者，而无需改变 Appdelgate 的内容，每个监听者以及中介者能够容易的被单独测试。

## 五、总结

大多数 AppDelegates 的设计都不太合理，过于复杂并且职责过多。我们称这样的类为 Massive App Delegates。

通过应用软件设计模式，Massive App Delegate 可以分成几个单独的类，每个类都有单一的责任，可以单独测试。

这样的代码很容易更改维护，因为它不会在您的应用程序中产生一连串的更改。它非常灵活，可以在将来提取和重用。

## 六、学习文章

[最佳实践：重构AppDelegate](https://mp.weixin.qq.com/s?__biz=MjM5NTQ2NzE0NQ==&mid=2247484070&idx=1&sn=8f784d2931c90bbc10c1d07bb634f01d&chksm=a6f95840918ed156b8333751242eab54caacc5502af2e177018f0fc9df8f8955cee36e92eaf8&mpshare=1&scene=23&srcid=1201w1cWjo0tBpIGir7EcNQD#rd)

[Refactoring Massive App Delegate](https://www.vadimbulavin.com/refactoring-massive-app-delegate/)

[iOSTips](https://github.com/GesanTung/iOSTips)

OC设计模式：《Objective-C 编程之道：iOS 设计模式解析》

 Swift 设计模式：《[Design\_Patterns\_by\_Tutorials\_v0.9.1](http://mp.weixin.qq.com/s?__biz=MjM5NTQ2NzE0NQ==&mid=2247483977&idx=1&sn=5994f8456884df158e7263be8179b79f&chksm=a6f958af918ed1b92625986d22a19e7ddaf85367386b9edf19b93e8ce836f83e0c5556997d13&scene=21#wechat_redirect)》

重构：《重构改善既有代码的设计》