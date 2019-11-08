---
title: iOS 启动优化
categories: iOS优化
---

在 WWDC 2016 上首次提到了关于 App 应用启动速度优化的话题：[Session 406 Optimizing App Startup Time](https://developer.apple.com/videos/play/wwdc2016/406/)。

## 一、冷启动与热启动

* 热启动：如果刚刚启动过 app，这时启动 app 所需要的数据仍然在缓存中，再次启动的时候称为热启动。通常情况下热启动能帮助提升启动速度，但有时也可能会出现 app 卡死手动退出进程后重新打开仍然是卡死状态。
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
Total pre-main time: 433.19 milliseconds (100.0%)
         dylib loading time: 341.79 milliseconds (78.8%)
        rebase/binding time:  14.18 milliseconds (3.2%)
            ObjC setup time:  35.27 milliseconds (8.1%)
           initializer time:  41.79 milliseconds (9.6%)
           slowest intializers :
             libSystem.B.dylib :   3.40 milliseconds (0.7%)
    libMainThreadChecker.dylib :  19.68 milliseconds (4.5%)
  libViewDebuggerSupport.dylib :   8.75 milliseconds (2.0%)
```


## 三、优化启动

启动时间其实包括了两部分：main 函数之前和 main 函数到第一个界面的 viewDidAppear:。

所以，优化也是从两个方面进行的，优化效果主要来自于后者，因为绝大多数 App 的瓶颈在自己的代码里。而对于 pre-main 的优化能做的无非是减少不必要的动态库引用、多个库合并成一个，从上面的打印数据也可以看出，主要耗时是在 `dylib loading`，消耗 `78.8%` 的时间。


## 四、Main 函数之后

从 main 函数开始执行，到第一个界面显示，期间一般做以下任务：

1. 执行 AppDelegate 的代理方法，主要是 didFinishLaunchingWithOptions
2. 初始化 Window，初始化基础的 ViewController 结构
3. 获取数据(Local DB／Network)，展示给用户。

优化：

1. 延迟初始化和加载不必要的 UIViewController 和 View。

	比如 UITabBarController 有四个 Item，在启动的时候尽量只初始化首页的页面，其它 Item 页面先用空 VC 占位。而且首页的内容中不必要的内容也可以先不初始化，做成懒加载形式，在用户确实需要查看和使用时再初始化。

2. 对于确实需要启动时使用但又比较耗时的事物放倒后台处理，如果涉及到 UI 则在处理完成后把刷新任务放回主线程。

	* 日志功能，日志往往涉及到 DB 操作；
	* 文件读取，比如读取本地存储的省份城市区县文件和图片处理；
	* 大量的计算，比如图片处理，比较大的 json 数据转 Model；

3. 能延迟初始化的尽量延迟初始化

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

	合并动态库，比如公司内部由私有 Pod 建立了如下动态库：XXTableView、XXHUD、XXLabel，强烈建议合并成一个 XXUIKit 来提高加载速度。尽量保证将 App 现有的非系统级的动态库个数保证在 6 个以内.

2. rebase/binding & ObjC Runtime setup

	Rebase 和 Binding 都是为了解决指针引用的问题。
	
	对于 Objective-C 开发来说，主要的时间消耗在 Class/Method 的符号加载上，所以常见的优化方案是：

	* 减少 __DATA 段中的指针数量。
	* 合并 Category 和功能类似的类，减少唯一 Selector 的个数。主要是为了加快程序的整个动态链接，在进行动态库的重定位和绑定(Rebase/binding)过程中减少指针修正的使用，加快程序机器码的生成。
	* 删除无用的方法和类。
	* 多用 Swift Structs，因为 Swfit Structs 是静态分发的。可以参考[Swift进阶之内存模型和方法调度](https%3A%2F%2Fblog.csdn.net%2Fhello_hwc%2Farticle%2Fdetails%2F53147910)

3. Initializers

	* 用 initialize 替代 load。load 在程序启动的时候就会调用，而且必须阻塞等着所有类的 load 方法都执行完；initialize 在类首次使用的时候调用。
	* 减少 \_\_atribute\_\_((constructor)) 的使用。


## 文章

[枫林风雨](https://www.jianshu.com/u/20b544b226ff) - [iOS性能(二) 启动时间优化](https://www.jianshu.com/p/8311b8d7e8bb)