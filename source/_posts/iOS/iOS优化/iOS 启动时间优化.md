---
title: iOS 启动优化
categories: iOS优化
---

在 WWDC 2016 上首次提到了关于 App 应用启动速度优化的话题：[Session 406 Optimizing App Startup Time](https://developer.apple.com/videos/play/wwdc2016/406/)。

## 一、冷启动与热启动

热启动是，APP会恢复之前的状态继续运行，这种就是热启动，我们平时所说的APP在后台的存活时间，其实就是APP能执行热启动的最大时间间隔。而冷启动则是APP从被加载到内存到运行的状态，下面我们要讲的主要是冷启动。

* 热启动：由于某种原因，APP 的状态由 running 切换为 suspend，但此时 APP 并没有被系统 kill掉，当我们再次把 APP 切换到前台的时候，此时启动 app 所需要的数据仍然在缓存中，再次启动的时候称为热启动。通常情况下热启动能帮助提升启动速度，但有时也可能会出现 app 卡死手动退出进程后重新打开仍然是卡死状态。
* 冷启动：如果是比较长时间没有启动过 app 或者设备刚刚重启，这种情况下启动 app，就被称为冷启动。


## 二、查看启动时间

* 最佳速度：<font color=#cc0000>400ms</font>。因为不添加任何同步任务从图标被点击到显示 Launch Screen，然后 Launch Screen 消失这段时间就是 400ms。如果 app 启动时间接近这个数值，那证明 app 的启动任务已经优化到最佳。
* 最慢速度：不可以大于 <font color=#cc0000>20s</font>，否则会被系统杀掉。

配置 Xcode 环境变量在日志中打印启动时间：

* 打开工程 -> Edit Scheme -> Run -> Environment Variables

根据需要添加 `DYLD_PRINT_STATISTICS` 和 `DYLD_PRINT_STATISTICS_DETAILS` 环境变量。1 表示 YES，开启这个功能。

<center>
![](http://dzliving.com/DYLD_PRINT_STATISTICS.png)
</center>

```
Total pre-main time: 617.58 milliseconds (100.0%)
         dylib loading time: 472.75 milliseconds (76.5%)
        rebase/binding time:  27.01 milliseconds (4.3%)
            ObjC setup time:  28.90 milliseconds (4.6%)
           initializer time:  88.76 milliseconds (14.3%)
           slowest intializers :
             libSystem.B.dylib :   8.81 milliseconds (1.4%)
    libMainThreadChecker.dylib :  14.42 milliseconds (2.3%)
                  AFNetworking :  18.43 milliseconds (2.9%)
                         Realm :  20.98 milliseconds (3.3%)
               		 CYKJBasic :  12.96 milliseconds (2.0%)
```

包括执行以下步骤的所有时间

1. 解析镜像
2. 映射镜像
3. Rebase 镜像
4. Bind 镜像
5. 镜像初始化
6. 调用 main()方法
7. 调用 UIApplicationMain() 方法
8. 调用 applicationWillFinishLaunching 回调

DYLD\_PRINT\_STATISTICS 环境变量只能帮助测量 Dyld 的启动时间，无法帮助我们完整地测量 main() 方法执行之前的所有时间。

为了能够解析 dylib 的符号表，debugger 需要在加载每个 dylib 的时候暂停一下，同时通过 USB 线来传输这些数据，这是非常耗时的，但是 DYLD\_PRINT\_STATISTICS 在测量的时候已经减去了这些耗时，因此不用担心在 debug 模式下数据的准确性，而且测量出的耗时通常会比肉眼计算出来的耗时要小。


## 三、优化启动

启动时间其实包括了两部分：main 函数之前和 main 函数到第一个界面的 viewDidAppear:。

所以，优化也是从两个方面进行的，优化效果主要来自于后者，因为绝大多数 App 的瓶颈在自己的代码里。而对于 pre-main 的优化能做的无非是减少不必要的动态库引用、多个库合并成一个，从上面的打印数据也可以看出，主要耗时是在 `dylib loading`，消耗 `78.8%` 的时间。


## 四、Main 函数之后

从 main 函数开始执行，到第一个界面显示，期间一般做以下任务：

1. 执行 AppDelegate 的代理方法，主要是 didFinishLaunchingWithOptions
2. 初始化 Window，初始化基础的 ViewController 结构
3. 获取数据(Local DB／Network)，展示给用户。

优化：

1. 使用代码绘制 UI，减少或者不用 xib 和 storyboard

2. 延迟初始化和加载不必要的 UIViewController 和 View。

	比如 UITabBarController 有四个 Item，在启动的时候尽量只初始化首页的页面，其它 Item 页面先用空 VC 占位。而且首页的内容中不必要的内容也可以先不初始化，做成懒加载形式，在用户确实需要查看和使用时再初始化。

3. 对于确实需要启动时使用但又比较耗时的事物放倒后台处理，如果涉及到 UI 则在处理完成后把刷新任务放回主线程。

	* 日志功能，日志往往涉及到 DB 操作；
	* 文件读取，比如读取本地存储的省份城市区县文件和图片处理；
	* 大量的计算，比如图片处理，比较大的 json 数据转 Model；

4. 能延迟初始化的尽量延迟初始化

	三方 SDK 初始化，比如 Crash 统计，分享之类的，可以等到第一次调用再去初始化。


## 五、Main函数之前

Main 函数之前是 iOS 系统的工作，所以这部分的优化往往更具有通用性。

Pre-Main 包含以下工作：

```
- dylib loading time: 341.79 milliseconds (78.8%)
- rebase/binding time:  14.18 milliseconds (3.2%)
- ObjC setup time:  35.27 milliseconds (8.1%)
- initializer time:  41.79 milliseconds (9.6%)
- slowest intializers :
- libSystem.B.dylib :   3.40 milliseconds (0.7%)
- libMainThreadChecker.dylib :  19.68 milliseconds (4.5%)
- libViewDebuggerSupport.dylib :   8.75 milliseconds (2.0%)
```

Session 所要传达的内容:

* 使用 DYLD\_PRINT\_STATISTICS 测试启动加载时间
* 减少自定义的动态库集成
* 精简原有的 Objective-C 类和代码
* 移除静态的初始化操作
* 使用更多的 Swift 代码

优化：

1. loading dylib

	启动的第一步是加载动态库，加载系统的动态库使很快的，因为可以缓存，而加载内嵌的动态库速度较慢。所以，提高这一步的效率的关键是：<font color=#cc0000>减少动态库的数量</font>。

	加载 App 内嵌的动态库比较耗时，因为每加载一个动态库，系统都需要文件验证、注册签名、针对 segment 进行 mmap。embedded framework 一般用于 Extension 跟主 APP 共享代码逻辑。

	合并动态库，比如公司内部由私有 Pod 建立了如下动态库：XXTableView、XXHUD、XXLabel，强烈建议合并成一个 XXUIKit 来提高加载速度。尽量保证将 App 现有的非系统级的动态库个数保证在 <font color=#cc0000>6 个</font>以内.
		
	也可以使用静态库，虽然会增加些许 rebase/binding 的时间。

2. rebase/binding

	Rebase 和 Binding 都是为了解决指针引用的问题。所以为了加快 rebase/binding，则需要更少的做指针修复。当你的 app 当中有太多的 Objective-C 的类、方法选择器和类别，会增加这一部分的启动时间。有一个数据：当大于 20000 个时候，会增加 800ms 的时间。
	
	对于 Objective-C 开发来说，主要的时间消耗在 Class/Method 的符号加载上，所以常见的优化方案是：

	* 减少 __DATA 段中的指针数量。
	* 合并 Category 和功能类似的类，减少唯一 Selector 的个数。主要是为了加快程序的整个动态链接，在进行动态库的重定位和绑定(Rebase/binding)过程中减少指针修正的使用，加快程序机器码的生成。
	* 删除无用的方法和类。
	* 多用 Swift Structs，因为 Swfit Structs 是静态分发的。可以参考[Swift进阶之内存模型和方法调度](https%3A%2F%2Fblog.csdn.net%2Fhello_hwc%2Farticle%2Fdetails%2F53147910)
	* 减少 c++ 虚函数

3. ObjC setup time

	在 Objective-C 的运行时(runtime)，需要对类(class)，类别(category)进行注册，以及选择器的分配，所以参照 rebase/binding time，尽量减少类的数量，可以达到减少这一部分的时间。

4. Initializers

	* 用 initialize 替代 load。load 在程序启动的时候就会调用，而且必须阻塞等着所有类的 load 方法都执行完；initialize 在类首次使用的时候调用。
	* 减少使用 c/c++ 的 \_\_atribute\_\_((constructor))。推荐使用 dispatch\_once()、pthread\_once()、std:once()等方法；
	* 推荐使用 swift
	* 不要在初始化中调用 dlopen() 方法，因为加载过程是单线程，无锁，如果调用 dlopen 则会变成多线程，会开启锁的消耗，同时有可能死锁
	* 不要在初始化中创建线程


## 文章

[枫林风雨](https://www.jianshu.com/u/20b544b226ff) - [iOS性能(二) 启动时间优化](https://www.jianshu.com/p/8311b8d7e8bb)