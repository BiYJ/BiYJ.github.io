
## 前言

[iOS性能优化系列篇之“优化总体原则”](http://www.cocoachina.com/articles/22940)

* 不要提前过度优化
* 要找到性能瓶颈
* 要在不同性能指标间权衡
* 要理解优化任务的底层运行机制
* 要有技术保障体系


## 一、启动速度优化

### 1.1 学习文章

* WWDC 启动速度优化视频 [Session 406 Optimizing App Startup Time](https://developer.apple.com/videos/play/wwdc2016/406/)
* [iOS性能(二) 启动时间优化](https://www.jianshu.com/p/8311b8d7e8bb)

### 1.2 操作步骤

1. 查看启动时间

	配置 Xcode 环境变量在日志中打印启动时间：

	* 打开工程 -> Edit Scheme -> Run -> Environment Variables

	根据需要添加 `DYLD_PRINT_STATISTICS` 和 `DYLD_PRINT_STATISTICS_DETAILS` 环境变量。value 设置为 1（表示 YES），开启这个功能。

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

	包括执行以下步骤的所有时间：
	
	1. 解析镜像
	2. 映射镜像
	3. Rebase 镜像
	4. Bind 镜像
	5. 镜像初始化
	6. 调用 main()方法
	7. 调用 UIApplicationMain() 方法
	8. 调用 applicationWillFinishLaunching 回调


2. 优化 `main()` 函数之后的执行时间

	* 使用代码绘制 UI，减少或者不用 xib 和 storyboard
	* 延迟初始化和加载不必要的 UIViewController 和 View。
	* 使用后台线程处理耗时的任务
	* 能延迟初始化的尽量延迟初始化

3. 优化 `main()` 函数之前的执行时间

	Session 所要传达的内容:
	
	* 使用 DYLD\_PRINT\_STATISTICS 测试启动加载时间
	* 减少自定义的动态库集成
	* 精简原有的 Objective-C 类和代码
	* 移除静态的初始化操作
	* 使用更多的 Swift 代码


	优化：

	* loading dylib

		<font color=#cc0000>减少动态库的数量</font>。尽量保证将 App 现有的非系统级的动态库个数保证在 <font color=#cc0000>6 个</font>以内。采取的方式：1、合并动态库；2、使用静态库。

	* rebase/binding

		Rebase 和 Binding 都是为了解决指针引用的问题。对于 Objective-C 开发来说，主要的时间消耗在 Class/Method 的符号加载上，所以常见的优化方案是：

		* 减少 \_\_DATA 段中的指针数量。
		* 合并 Category 和功能类似的类，减少唯一 Selector 的个数。
		* 删除无用的方法和类。
		* 多用 Swift Structs，因为 Swfit Structs 是静态分发的。
		* 减少 c++ 虚函数

	* ObjC setup time

		<font color=#cc0000>尽量减少类的数量</font>，可以达到减少这一部分的时间。

	* Initializers

		* 用 initialize 替代 load。（注意：要根据实际情况替换）
		* 减少使用 c/c++ 的 \_\_atribute\_\_((constructor))。推荐使用 dispatch\_once()、pthread\_once()、std:once()等方法；
		* 推荐使用 swift
		* 不要在初始化中调用 dlopen() 方法，因为加载过程是单线程，无锁，如果调用 dlopen 则会变成多线程，会开启锁的消耗，同时有可能死锁
		* 不要在初始化中创建线程

### 1.3 实际处理

实际处理时因为要考虑团队的原因，所以只采用了一下三个步骤：

1. 删除不需要的三方库，因为工程中用 use_frameworks! pod 进来的库是动态库。
2. 将大部分 +load 方法改为 +initialize 方法，保留部分必要的 +load 方法；
3. 动态库 -> 静态库，技术上实现了，讨论后最终弃用。
4. 删除无用的方法和类，减少 rebase/binding 时间。。

### 1.4 处理效果

慈云 app 启动时间有 700 ~ 850ms，降为 600 ~ 700ms。

如果采用静态方式 pod，可以降为 450 ~ 550ms。


## 二、接口请求优化

主要集中处理一级 tab 的切换体验，频繁的调用接口，频繁的刷新界面显然是影响用户体验的。优化的思路有以下几点：

1. 使用 loading 框 + 默认灰色矩形视图；

	<center>
	![](https://upload-images.jianshu.io/upload_images/5294842-4c639c39d38a8c44.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)
	</center>
	
2. 使用本地缓存

3. 每隔 15s（或者 10s） 以上才请求一次，防止频繁触发请求

	```
	@property (nonatomic, assign) CFTimeInterval lastTi;

	- (void)viewWillAppear:(BOOL)animated
	{
		[super viewWillAppear:animated];

		CFTimeInterval nowTi = CACurrentMediaTime();
        // 10 秒内不请求
        if (nowTi - self.lastTi > 10) {
            self.reqData.pageNo = 1;
            [self.service requestPatientConsultList:self.reqData];
            self.lastTi = nowTi;
        }
	}
	
	- (void)makeRequestAction
	{
		// 接口请求
	}
	```
	
	CACurrentMediaTime() 在退到后台、手动修改设备时间后没有影响。
	
4. 对数据进行判断，数据没有更新不需要刷新界面。

	```
	@property (nonatomic, copy) NSString * primaryCompareMD5;
	
	// 初始值 @""
	self.primaryCompareMD5 = @"";

	SELF_WEEK;
	// 异步执行
   dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
		@synchronized (self) {
			// 这里的 data 是要判断的接口数据
			NSString * string = [data componentsJoinedByString:@""];
			if (!string) {
				string = @"";
			}
			NSString * md5 = [CYDXUtil doMd5:string];
		
			SELF_STRONG;
			// 数据未发生改变，直接返回
			if ([strongSelf.primaryCompareMD5 isEqualToString:md5]) {
				return;
			}
			strongSelf.primaryCompareMD5 = md5;
                
          // 主线程刷新界面
			dispatch_sync(dispatch_get_main_queue(), ^{
				[strongSelf.tableView reloadData];
			});
		}
	});
	```

这里的策略还是存在点问题：

1. 不能处理好未读标识。看实际情况，处理从子界面返回时的逻辑。


## 三、界面滑动优化

### 3.1 监测界面卡顿工具

1. 使用系统的 Instrument - CoreAnimation
2. 使用系统的 Instrument - [Time Profiler](https://www.jianshu.com/p/21d29be26479)
3. [YYFPSLabel](https://github.com/yehot/YYFPSLabel)。

在未滑动时，Instrument - CoreAnimation 显示的是 0 FPS。

![](https://upload-images.jianshu.io/upload_images/5294842-d673400fbd2b9f9e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/340)

YYFPSLabel 显示的是 60FPS。

![](https://upload-images.jianshu.io/upload_images/5294842-0bdeb37332ef2f55.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/340)

卡顿问题修改后，Instrument - CoreAnimation 需要在设备上运行一遍，然后重新用工具检测；YYFPSLabel 可以直接加入工程，可视化，更加的友好，方便。

Time Profiler 可以查看哪些方法占用时间多，优化那些可以优化的。

![](https://upload-images.jianshu.io/upload_images/5294842-30806a343a46eeb6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 3.2 优化处理

![顶部菜单栏](https://upload-images.jianshu.io/upload_images/5294842-177b6b3ea753ae1f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


[Instrument之Core Animation工具](https://blog.csdn.net/xiaoxiaobukuang/article/details/51076944)

1. 图层混合
	
	> Xcode 顶部菜单栏 -> Debug -> View Debugging -> Rendering -> Color Blended Layers
	
	正常的为绿色；出现图层混合时为红色。
	
	* 确保控件的 opaque 属性设置为 true，确保 backgroundColor 和父视图颜色一致且不透明； 
	* 如无特殊需要，不要设置低于 1 的 alpha 值；
	* 确保 UIImage 没有 alpha 通道；
	
	
2. 图片缩放

	> Xcode 顶部菜单栏 -> Debug -> View Debugging -> Rendering -> Color Misaligned Images
	
	如果图片需要缩放则标记为黄色，如果没有像素对齐则标记为紫色。
	
	这个看情况是否优化，是否需要 UI 配合。
	
	* 确保图片大小和frame一致，不要在滑动时缩放图片；
	* 确保图片颜色格式被 GPU 支持，避免劳烦 CPU 转换；
	
3. 离屏渲染

	> Xcode 顶部菜单栏 -> Debug -> View Debugging -> Rendering -> Color Offscreen-Rendered Yellow
	
	触发离屏渲染的地方标记为黄色。
	
	下面的情况或操作会引发离屏渲染：

	* 为图层设置遮罩（layer.mask）
	* 同时设置 layer.masksToBounds 和 corneRadius 属性设置为 true
	* 将图层的 layer.allowsGroupOpacity 属性设置为 YES 和 layer.opacity < 1.0
	* 为图层设置阴影（layer.shadow*）
	* 为图层设置 layer.shouldRasterize = true
	* 文本（任何种类，包括 UILabel、CATextLayer、CoreText 等）
	* 使用 CGContext 在 drawRect: 方法中绘制大部分情况下会导致离屏渲染，甚至仅仅是一个空的实现

	解决：
	
	* 使用 CoreGraphics 绘制圆角，不使用 mask 或 masksToBounds + corneRadius；
	* 设置阴影是使用 shadowPath；
	* UILabel 如果不是透明的，设置 opaque = YES，Clip To bounds = YES，backgroundColor；
	* 设置图片圆角可以让 UI 切图，也可以使用 CoreGraphics 绘制图片的方式。在 iOS 9.0 以上，图片设置圆角不会触发离屏渲染。
	
	
	[iOS 阴影，圆角，避免离屏渲染](https://www.jianshu.com/p/15b8b1844e9c)
	[iOS - 圆角的设计策略 - 及绘画出没有离屏渲染的圆角](https://www.jianshu.com/p/af70feefbc89)
	
4. UITableViewCell 优化

	* Cell 复用；
	* 提前计算并缓存 Cell 的高度；
	* 减少 subviews 的个数和层级；
	* 少用 subviews 的透明图层；
	* 如果不是透明视图，背景色不要使用 clearColor；
	* 注意离屏渲染问题；
	* 图片提前在子线程异步解码，SDWebImage 已经实现；
	* 异步绘制（自定义 Cell 绘制）[VVeboTableViewDemo](https://github.com/johnil/VVeboTableViewDemo)、[YYAsyncLayer](https://github.com/ibireme/YYAsyncLayer)、[AsyncDisplayKit](https://github.com/facebookarchive/AsyncDisplayKit)
	* 滑动时，按需加载。注意：这个会导致滑动时出现大量空白，不友好。
	* 尽量显示“大小刚好合适的”图片资源
	* 避免同步的从网络、文件获取数据
	

## 四、内存占用

主要检查工程中的内存泄露、循环引用的问题。

1. instrument - Allocations 动态分析
2. Analyze 静态分析
3. 腾讯 MLeaksFinder
4. 脸书 FBMemoryProfiler

说明:

1. MLeaksFinder 效果比较好，能查找工程中出现循环引用的的场景，在慈云找到十几处循环引用。
2. 官方的 instrument 工具效果一般，在慈云 app 只检测出了 3 处循环引用，2 处内存泄露；
3. Analyze 静态分析可检测出“创建了但未被使用的”变量，需要结合代码逻辑，小心处理。
4. 脸书的 FBMemoryProfiler 工具不大好用，主要是记录内存的开辟与销毁。

如果要更多的处理内存占用问题，需要分析工程中的代码逻辑，通过使用不同的存储方式、容器类、加载数据方式等，达到减少内存使用的目的。


## 五、缩小 ipa 包大小

[爱奇艺移动应用优化之路：如何让崩溃率小于千分之二](http://www.zijin.net/news/tech/1317802.html)

安装包大小的优化，主要包含两大块：资源大小的优化和二进制大小的优化。

### 5.1 资源大小的优化

资源大小的优化主要包括以下几个方面：

1. 资源压缩
2. 未使用、重复资源的删除
3. 资源上云

具体步骤：

1. 资源压缩

	Xcode 的编译选项中，提供了 `Compress PNG Files` 及 `Remove Text MetaData From PNG Files`
	
	![](http://dzliving.com/iOSiPA_0.jpg)
	
	但是由于 PNG 是无损压缩，经过 Xcode 压缩后的图片资源，依然很大。
	
	使用 [pngquant](https://pngquant.org/) 对大多数的 32 位图进行了处理，将其转为 8 位图，并且使用 Zopfli 进行了压缩，这样整体的 PNG 图片资源大概被压缩了 70% 左右。这里要注意，由于一些渐变背景的颜色覆盖范围较大，转为 8 位图颜色丢失较大，表现效果会差很多，所以这些图片要谨慎处理。

2. 未使用、重复资源的删除

3. 资源上云

	资源上云可以有效减少包内资源，唯一要注意的是这些资源由于是 lazy load，所以比较适合层级较深的页面使用。

### 5.2 其他处理

1. 配置编译选项 Generate Debug Symbols 设置为 NO；

	![](http://dzliving.com/iOSOptimize_13.png?imageView2/0/w/400)

2. 舍弃架构，如：armv7，根据实际选择。

	![](http://dzliving.com/iOSOptimize_14.png)

3. 编译的版本必须是 release 版本，

4. 查找内部使用到的第三方库，一方面可以进行删减代码，用不到的类，直接删除，还有第三方库中的图片资源统统删除掉，如果能够自己手写实现的，那费功夫自己写吧


## 六、引用库升级及替换

持续更新的第三方库也会对性能做出优化，在替换前，需先确定不会给项目引入问题。


## 七、更多优化

[iOS 性能调试](https://www.cnblogs.com/zhangying-domy/p/5948613.html)
[iOS视图成像理论及性能优化](https://www.jianshu.com/p/c7869be4ab56)
[iOS性能优化系列篇之“列表流畅度优化”](http://www.cocoachina.com/articles/24599)