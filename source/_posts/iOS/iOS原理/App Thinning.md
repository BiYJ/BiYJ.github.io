---
title: App Thinning
categories: iOS原理
---

App Thinning 可以译成"应用瘦身"。指的是 App Store 和操作系统在安装 iOS 或者 watchOS 的 app 的时候通过一系列的优化，尽可能减少安装包的大小，使得 app 以最小的合适的大小被安装到你的设备上。而这个过程包括了三个过程：slicing、bitcode、on-demand resources。

## 一、slicing

App Slicing 在节省应用所需资源中发挥着最重要的作用。

很多应用需要在不同尺寸的设备上运行，针对这些不同的设备，它们内含不同的独立资源，而大部分是你的设备不需要的。所以App Store 会针对不同的设备制作不同的"简化版 App"，当你下载 app 时候只需要下载不同的"简化版 app"就可以了。

比如用户使用的是 iPhone 5c，它运行的是 32 位 CPU 和 GPU，并不支持 Metal API。但如果用户下载的是一款最新的通用游戏应用，它的二进制中含有 64 位代码，iPad 和"@3x"iPhone 6 Plus 资源以及 Metal API 代码，这些都是你的设备用不上的。它只需要 32 位代码，"@2x"iPhone 尺寸资源以及 OpenGL 图形代码。

<center>
![](https://upload-images.jianshu.io/upload_images/5294842-2c4819d0348f492f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)
</center>

**Note : Sliced apps are supported on devices running 9.0 and later;**

Slicing 的主要的工作流程如下：

1.  在 Xcode 中选择好目标设备并且使用 asset catalog 提供多分辨率的图片资源；只有使用 asset catalog 才能正确使Slicing 作用于资源文件。
2.  在模拟器或者设备上编译并运行 app；
3.  Xcode 会自动构建针对你运行设备的"简化版 app"，同时也是为了减少编译时间和进行本地的测试；
4.  打包 app（为了及时发现不同目标设备的配置错误，可以在本地为目标设备导出"简化版 app"，测试无误后再打包)
5.  上传打包好的 app 到 iTunes connect。App Store 将会为上传的 app 归档创建不同的"简化版 app"。
6.  在 iTunes Connect 中，发布一个预览版给合格的测试者进行测试；
7.  测试者通过 TestFlight 下载预览版。TestFlight 会自动根据测试者的设备下载合适的"简化版 app"。

## 二、Bitcode (iOS, watchOS)

Bitcode 是一个编译好的程序的中间表示形式。上传到 iTunes Connect 中的包含 Bitcode 的 app 将会在 App Store 中进行链接和编译。苹果会对包含 Bitcode 的二进制 app 进行二次优化，而不需要提交一个新的 app 版本到 App Store 中。

## 三、On-Demand Resources (iOS)

ODR（on-demand resources 随需应变资源)是 iOS 减少应用资源消耗的另外一种方法。比如多级游戏，用户需要的通常都是他们当前的级数以及下一级。ODR 意味着用户可以下载他们需要的几级游戏。随着你的级数不断增加，应用再下载其他级数，并将用户成功过关的级数删掉。

<center>
![](https://upload-images.jianshu.io/upload_images/5294842-3691bd525660ea81.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)
</center>

当用户点击应用内容的时候，就会动态从 App Store 上进行下载，也就是说用户只会在需要的时候占用存储空间。这项功能有趣之处还在于当将这些内容在后台进行下载之后，当存储空间紧张的时候会自动进行删除。

<center>
![](https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/On_Demand_Resources_Guide/Art/ODR_flow_2x.png)
</center>

On-Demand Resources 可以是除了可执行代码外的任意类型。

<center>
![](https://upload-images.jianshu.io/upload_images/5294842-66c28b72674b28d8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)
</center>

在开发过程中，你可以通过分配一个或多个 tag 来识别 On-Demand Resources。你可以使用 tag 的别名来确定什么时候将它加载到你的 App 中。

好处：

*   Smaller app size. app 体积更小。
*   Lazy loading of app resources. 懒加载应用资源。
*   Remote storage of rarely used resources. 远端存储较少使用的资源。
*   Remote storage of in-app purchase resources. 远端存储内购资源。

下图展示了一个在 App 中保持最小资源占用的例子。

<center>
![](https://upload-images.jianshu.io/upload_images/5294842-d1ee45f7ae6ebb73.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)

![](https://upload-images.jianshu.io/upload_images/5294842-d5bfaeb1150e88bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)
</center>

可以给资源设置优先级，比如当 App 从 AppStore 安装后就立即加载。

>  A \_tag\_ is a string identifier you create. Apps request tags, not individual resources.

#### 3.1 On-Demand Resources 的生命周期

1. App 向操作系统请求资源。操作系统将请求发送给包含所有所需资源的 asset packs。

	<center>
	![](https://upload-images.jianshu.io/upload_images/5294842-8053f6eb07507b8a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)
	</center>

2. asset packs 检查请求的资源本地是否存在。如果存在，则直接提供 App 使用。

3. 如果请求的资源本地不存在，则它们被保存在 App Store。

	<center>
	![](https://upload-images.jianshu.io/upload_images/5294842-316aab844be76e4a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

4. 操作系统开始下载本地不存在的资源

5. 远程资源下载完毕
	
	<center>
	![](https://upload-images.jianshu.io/upload_images/5294842-c5191d663f1b8e89.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

6. 当资源下载成功或监测到资源包已经被下载，资源包内存计数将会被 ＋1，并通知 App 此资源可用。

	<center>
	![](https://upload-images.jianshu.io/upload_images/5294842-bbbb952f642fd22b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

7. 当请求的资源可用，App 使用资源标签对应的资源。

	<center>
	![](https://upload-images.jianshu.io/upload_images/5294842-9e2fc891d113fdac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

8. 操作系统在本地释放资源标签

9. 操作系统在本地清除资源缓存。当一个缓存资源不与任何请求相关联时，操作系统会在一定时间后将它释放掉。

完整的生命周期如下图所示：

<center>
![](https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/On_Demand_Resources_Guide/Art/ODR_flow_2x.png)
</center>

## 四、实际处理方法

1. iOS9 以后 Xcode 默认开启 On-Demand Resources 功能，可以在下图所示位置进行设置。

	<center>
	![](https://upload-images.jianshu.io/upload_images/5294842-20fd1426c699a6c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

2. 在 App 中创建 Tags

	<center>
	![](https://upload-images.jianshu.io/upload_images/5294842-1b4cfc77690df19f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	
	![](https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/On_Demand_Resources_Guide/Art/ODR_Add_New_Tag_2x.png)
	</center>

3. 给文件设置 tag

	<center>
	![](https://upload-images.jianshu.io/upload_images/5294842-62516bc6ffbdee87.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>
	
4. 给图片设置 tag

	<center>
	![](https://upload-images.jianshu.io/upload_images/5294842-b56358b46094e85e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

5. 给 tag 设置加载的优先级

	<center>
	![](https://upload-images.jianshu.io/upload_images/5294842-e2b226aa8a0fdfd3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

#### 4.1 加载优先级

*   Initial install tags.资源和 App 同时下载。在 App Store 中，App 的大小计算已经包含了这部分资源。当没有NSBundleResourceRequest 对象访问它们时，它们将会从设备上清除。
*   Prefetch tag order. 在 App 安装后开始下载，按照预加载列表中的顺序依次下载。
*   Dowloaded only on demand. 只有在 App 中发出请求时才会下载。

#### 4.2 资源大小限制

<center>
![](http://upload-images.jianshu.io/upload_images/3402279-ea2bc5a27c368c93.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/715)
</center>

## 五、学习文章

[What is app thinning? (iOS, tvOS, watchOS)](https://help.apple.com/xcode/mac/current/#/devbbdc5ce4f)
[On-Demand Resources Essentials](https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/On_Demand_Resources_Guide/index.html#//apple_ref/doc/uid/TP40015083)
[https://www.jianshu.com/p/789df0adaac2](https://www.jianshu.com/p/789df0adaac2)