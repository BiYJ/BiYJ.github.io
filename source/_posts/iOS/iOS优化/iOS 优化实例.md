---
title: iOS 优化实例
categories: iOS优化
---


## 一、接口请求优化

在工程项目中，多个一级界面包含状态，如：服务入口的动态配置，未读消息数量，图片文字等，因此产品设计要每次切换 tab 时都请求数据，及时的更新页面状态。在实际开发中，频繁的调用接口，频繁的刷新界面显然是影响用户体验的，所以需要进行优化，优化的思路有以下几点：

1. 使用 loading + 默认灰色矩形视图；
2. 每隔 15s 以上才请求一次，防止频繁触发请求

	```
	@property (nonatomic, assign) CFTimeInterval lastTi;

	- (void)viewWillAppear:(BOOL)animated
	{
		[super viewWillAppear:animated];

		CFTimeInterval nowTi = CACurrentMediaTime();
		
		if (self.lastTi <= 0 || nowTi - self.lastTi > 15) {
			[self makeRequestAction];
			self.lastTi = nowTi;
		}
	}
	
	- (void)makeRequestAction
	{
		// 接口请求
	}
	```
	
	CACurrentMediaTime() 在退到后台、手动修改设备时间后没有影响。
	
3. 对数据进行判断，数据没有更新不需要刷新界面。这里要注意，存储的是不是字符串，而是字符串的 hash 值，因为长度比较小。

	```
	@property (nonatomic, assign) NSUInteger compareHash;  // 判断列表数据是否更新
	
	{
		NSUInteger hash = [data.mainTools componentsJoinedByString:@""].hash;
		
		if (self.compareHash == hash)	 {
			return;
		}
		self.compareHash = hash;
		
		// 更新界面
	}
	```

这里的策略还是存在点问题：

1. 不能处理好未读标识。比如当进入子界面详情时，这里需要更新未读状态，但因为 15s 的限制，而不能及时刷新；


## 二、界面优化

1. 使用 Debug -> View Debugging -> Rendering 中的界面渲染选项，进行界面优化

	<center>
	![](http://dzliving.com/iOSDebugRender.png)
	</center>
	
2. cell 滚动停止时加载
	
	
## 三、资源加载优化

1. 图片加载

	> imageNamed:
	
	利用它可以方便加载资源图片。用 imageNamed 的方式加载时，会把图像数据根据它的名字缓存在系统内存中，以提高 imageNamed 方法获得相同图片的 image 对象的性能。即使生成的对象被  autoReleasePool 释放了，这份缓存也不释放。而且没有明确的释放方法。如果图像比较大，或者图像比较多，用这种方式会消耗很大的内存。
	
	优点：加载图片速度快；
	缺点：耗内存大，适用于小图片并且会被频繁读取的图片。

	> imageWithContentsOfFile:

	加载的图片是不会缓存的。得到的对象时 autoRelease 的，当 autoReleasePool 释放时才释放。
	
	优点：图片数据不缓存，节省内存；
	缺点：加载图片速度相对慢，每次读取图片都要从路径下寻找并解析图片数据。适用于会用到的图片数据不大，且不经常用到的。
	
	imageNamed: 加载会缓存在内存中，对于常用的图片可以放在 asset 里，不常用的图片放在 bundle 的路径下通过 imageWithContentsOfFile: 获取图片资源。


	> initWithContentsOfFile:
	
	一般用在封面等图比较大的地方。适用于大图片，且不经常要用的图片。

	```
	dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(4 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
		for (int i = 0; i < 100000; i++) {
			self.loadImageView.image = [UIImage imageNamed:@"icon"];
			//self.loadImageView.image = [UIImage imageWithContentsOfFile:[[NSBundle mainBundle] pathForResource:@"icon" ofType:@"png"]];
		}
	});
	```
	
	* imageNamed:

		<center>
		![](http://dzliving.com/iOSimageNamed_0.png)
		![](http://dzliving.com/iOSimageNamed_1.png)
		</center>
		
	* imageWithContentsOfFile:

		<center>
		![](http://dzliving.com/iOSimageFile_0.png)
		![](http://dzliving.com/iOSimageFile_1.png)
		</center>
		
	在有缓存的情况下，即使运行 100000 次，也不会等待很久，而 imageWithContentsOfFile: 会等待很久，且内存持续增长。
	
2. UITableView 中使用 imageNamed: 加载图片，频繁上下滑动时 CPU 耗时不断增长。

	原本期望 imageNamed: 创建了缓存的，不应该持续耗费 cpu 时间，结果却耗费了 6-700ms，多处合起来有几秒。
	
	优化思路是自己使用 NSDictionary 或 NSCache 存储 images，使用时根据 key 来获取。
	
	```
	// .h 文件
	FOUNDATION_EXTERN NSString * kAPlaceholderImageKey;
	FOUNDATION_EXTERN NSString * kBPlaceholderImageKey;
	
	- (UIImage *)imageForKey:(NSString *)placeholderImageKey;
	
	
	// .m 文件
	NSString * kAPlaceholderImageKey = @"APlaceholderImageKey";
	NSString * kBPlaceholderImageKey = @"BPlaceholderImageKey";

	- (UIImage *)imageForKey:(NSString *)placeholderImageKey
	{
	    return [self.imageCache objectForKey:placeholderImageKey];
	}
	
	/**
	 *  @brief 缓存 cell 中需要重复使用的图片
	 */
	- (NSCache *)imageCache
	{
	    if (_imageCache == nil) {
	        _imageCache = [[NSCache alloc] init];
	        [_imageCache setObject:[UIImage imageNamed:@"1"] forKey:kAPlaceholderImageKey];
	        [_imageCache setObject:[UIImage imageNamed:@"2"] forKey:kBPlaceholderImageKey];
	    }
	    return _imageCache;
	}
	```

	
## 四、启动速度优化

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
               CYKJBasicModule :  12.96 milliseconds (2.0%)
```

1. 删除不需要的三方库

2. +load -> +initialize

	注意有的功能需要写在 +load 方法。可以改为 +initialize 后，用断点看看是否被调用。	
3. 动态库 -> 静态库

	如果 Podfile 中使用 use_frameworks!，那么第三方库都是以动态库的方式添加到工程中，动态库在启动阶段由 dyld 加载，静态库是运行时加载，所以为了减少 dylib loading time，可以减少工程中的动态库。