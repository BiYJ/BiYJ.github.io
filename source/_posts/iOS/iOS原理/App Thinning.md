---
title: App Thinning
categories: iOS原理
---

当前 iOS App 的编译打包方式是<font color=#cc0000>把适配兼容多个设备的执行文件及资源文件合并一个文件</font>，上传和下载的文件则包含了所有的这些文件，导致占用较多的存储空间。

App Thinning 是一个关于`节省 iOS 设备存储空间`的功能，它可以让 App Store 和操作系统在安装、更新及运行 iOS 或者 watchOS 的 App 等场景时，通过一系列的优化，尽可能减少安装包的大小，仅下载所需的资源，减少 App 的占用空间，从而节省设备的存储空间。这个过程包括了三个过程：slicing、bitcode、on-demand resources。

## 一、slicing

App Slicing 在节省应用所需资源中发挥着最重要的作用。

很多应用需要在不同尺寸的设备上运行，针对这些不同的设备，它们内含不同的独立资源，而大部分是用户设备不需要的。所以，开发者把 App 安装包上传到 AppStore 后，Apple 服务会自动将安装包<font color=#cc0000>切割</font>为不同的应用变体(App variant)，或者称 App Store 会针对不同的设备制作不同的"简化版 App"，当你下载 app 时，系统会根据设备型号下载安装不同的"简化版 app"。（iOS9.0.2 以上支持）

iOS app 为了后向兼容，现在都同时包含了 32 bit 和 64 bit 两个 slice。另外在图片资源方面，更是 2x、3x 的图像一应俱全。而用户使用 app 时，因为设备是特定的，其实只需要其中的一套资源。但是现在在购买和下载的时候却是把整个 app 包都下载了。Apple 在 iOS 9 中终于可以仅选择需要的内容 (Slicing) 下载了。对于开发者来说，并没有太多要做的事情，只需要使用 asset catalog 来管理素材标记 2x 3x 就可以了。

比如用户使用的是 iPhone 5c，它运行的是 32 位 CPU 和 GPU，并不支持 Metal API。但如果用户下载的是一款最新的通用游戏应用，它的二进制中含有 64 位代码，iPad 和"@3x"iPhone 6 Plus 资源以及 Metal API 代码，这些都是你的设备用不上的。它只需要 32 位代码，"@2x"iPhone 尺寸资源以及 OpenGL 图形代码。

<center>
![](http://dzliving.com/AppThinning_1.png)
</center>

**Note : Sliced apps are supported on devices running 9.0 and later;**

Slicing 的主要的工作流程如下：

1.  在 Xcode 中选择好目标设备并且使用 asset catalog 提供多分辨率的图片资源；只有使用 asset catalog 才能正确使 Slicing 作用于资源文件。
2.  在模拟器或者设备上编译并运行 app；
3.  Xcode 会自动构建针对你运行设备的"简化版 app"，同时也是为了减少编译时间和进行本地的测试；
4.  打包 app（为了及时发现不同目标设备的配置错误，可以在本地为目标设备导出"简化版 app"，测试无误后再打包)
5.  上传打包好的 app 到 iTunes connect。App Store 将会为上传的 app 归档创建不同的"简化版 app"。
6.  在 iTunes Connect 中，发布一个预览版给合格的测试者进行测试；
7.  测试者通过 TestFlight 下载预览版。TestFlight 会自动根据测试者的设备下载合适的"简化版 app"。

最终苹果下载资源的效果如下：

<center>
![](http://dzliving.com/AppThinning_2.png)
</center>

## 二、Bitcode (iOS, watchOS)

开启 Bitcode 编译后，可以使得开发者上传 App 时只需上传 Intermediate Representation(中间件)，而非最终的可执行二进制文件。在用户下载 App 之前，AppStore 会自动编译中间件，产生设备所需的执行文件供用户下载安装。也就是当我们提交程序到 App Store 上时， Xcode 会将程序编译为一个中间表现形式( bitcode )。然后 App store 会再将这个 Bitcode 编译为可执行的 64 位或 32 位程序。苹果会根据下载应用的用户的手机指令集类型生成只有该指令集的二进制，进行下发。从而达到精简安装包体积的目的。

<center>
![](https://upload-images.jianshu.io/upload_images/1163291-33b9b5383c2b8905.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/731)
</center>

Bitcode 是 LLVM 编译器的中间代码的一种编码，LLVM 的前端可以理解为 C/C++/OC/Swift 等编程语言，LLVM 的后端可以理解为各个芯片平台上的汇编指令或者可执行机器指令数据，那么，BitCode 就是位于这两者之间的中间码。LLVM 的编译工作原理是前端负责把项目程序源代码翻译成 Bitcode 中间码，然后再根据不同目标机器芯片平台转换为相应的汇编指令以及翻译为机器码。这样设计就可以让 LLVM 成为了一个编译器架构，可以轻而易举的在 LLVM 架构之上发明新的语言(前端)，以及在 LLVM 架构下面支持新的 CPU(后端)指令输出。

虽然 Bitcode 仅仅只是一个中间码不能在任何平台上运行，但是它可以转化为任何被支持的 CPU 架构，包括现在还没被发明的 CPU 架构，也就是说现在打开 Bitcode 功能提交一个 App 到应用商店，对于 Apple 未来进行硬件升级的措施，此机制可以保证在开发者不重新发布版本的情况下而兼容新的设备。比如以后如果苹果新出了一款手机并 CPU 架构也是全新设计的，在苹果后台服务器一样可以从这个 App 的 Bitcode 开始编译转化为新 CPU 上的可执行程序，可供新手机用户下载运行这个 App。

在文档里可看到

> In fact, app slicing handles the majority of the app thinning process. ‘App Slicing’ feature finally switched on in iOS 9.0.2

说明 slicing 才是主要处理 app thinning 的，而且该功能需要在 iOS9.0.2 以上才支持（iOS9.0 中被关闭了，因为一个 iCloud 的 bug）。<font color=#cc0000>实际上 Bitcode，做的事情是指令集优化</font>。根据你设备的状态去做编译优化，进而提升性能。所以 Bitcode 对包的大小优化起不到什么本质上的作用。

**Bitcode 注意点**

1. Xcode 7 默认开启 Bitcode，如果应用开启 Bitcode，那么其集成的其他第三方库也需要是 Bitcode 编译的包才能真正进行 Bitcode 编译
2. 开启 Bitcode 编译后，编译产生的 .app 体积会变大(中间代码，不是用户下载的包)，且 .dSYM 文件不能用来崩溃日志的符号化(用户下载的包是 Apple 服务重新编译产生的，有产生新的符号文件)
3. 通过 Archive 方式上传 AppStore 的包，可以在 Xcode 的 Organizer 工具中下载对应安装包的新的符号文件

## 三、On-Demand Resources (iOS)

ODR（on-demand resources 随需应变资源)是 iOS 减少应用资源消耗的另外一种方法。比如多级游戏，用户需要的通常都是他们当前的级数以及下一级。ODR 意味着用户可以下载他们需要的几级游戏。随着你的级数不断增加，应用再下载其他级数，并将用户成功过关的级数删掉。

<center>
![](http://dzliving.com/AppThinning_3.png)
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
[App Thinning](https://www.jianshu.com/p/789df0adaac2)
[App Thinning](https://www.cnblogs.com/jvan/p/5473312.html)