---
title: Xcode
categories: 工具
---

## 一、整行上下移动

Xcode 自带的配置文件路径：/Applications/Xcode.app/Contents/Frameworks/IDEKit.framework/Versions/A/Resources/IDETextKeyBindingSet.plist，用文本编辑 IDETextKeyBindingSet.plist，并添加以下代码：

```oc
	<key>GDI Commands</key>
	<dict>
        <key>GDI Duplicate Current Line</key>
        <string>selectLine:, copy:, moveToEndOfLine:,insertNewline:, paste:, deleteBackward:</string>
        <key>GDI Delete Current Line</key>
        <string>moveToEndOfLine:, deleteToBeginningOfLine:,deleteBackward:,moveDown:,moveToEndOfLine:</string>
        <key>GDI Move Current Line Up</key>
        <string>selectLine:, cut:, moveUp:, moveToBeginningOfLine:, insertNewLine:, paste:, moveBackward:</string>
        <key>GDI Move Current Line Down</key>
        <string>selectLine:, cut:, moveDown:, moveToBeginningOfLine:, insertNewLine:, paste:, moveBackward:</string>
        <key>GDI Insert Line Above</key>
        <string>moveUp:, moveToEndOfLine:, insertNewline:</string>
        <key>GDI Insert Line Below</key>
        <string>moveToEndOfLine:, insertNewline:</string>
    </dict>
```

<font color=#cc0000>注意</font>：Xcode.app 需要换成实际的应用名，如 Xcode10.1.app。

详细文章：[xcode 设置快捷键 整行上下移动](https://www.cnblogs.com/goodboy-heyang/p/4732365.html)

## 二、Other linker flags

Other linker flags 用来填写 XCode 的<font color=#cc0000>链接器参数</font>。

从 C 代码到可执行文件经历的步骤是：

> 源代码 -> 预处理器 -> 编译器 -> 汇编器 -> 机器码 -> 链接器 -\> 可执行文件

在最后一步需要<font color=#cc0000>把 .o 文件和 C 语言运行库链接起来</font>，这时候需要用到 <font color=#cc0000>ld</font> 命令。

源文件经过一系列处理以后，会生成对应的 <font color=#cc0000>.obj</font> 文件，然后一个项目必然会有许多 .obj 文件，并且这些文件之间会有各种各样的联系，例如函数调用。<font color=#cc0000>链接器做的事就是把这些目标文件和所用的一些库链接在一起形成一个完整的可执行文件</font>。

Other linker flags 设置的值<font color=#cc0000>实际上就是 ld 命令执行时后面所加的参数</font>。

> The “selector not recognized” runtime exception occurs due to an issue between the implementation of standard UNIX static libraries, the linker and the dynamic nature of Objective-C. Objective-C does not define linker symbols for each function (or method, in Objective-C) - instead, linker symbols are only generated for each class. If you extend a pre-existing class with categories, the linker does not know to associate the object code of the core class implementation and the category implementation. This prevents objects created in the resulting application from responding to a selector that is defined in the category. 
> 
> 运行时的异常是由于标准 XNIX 静态库、链接器与 OC 语言的动态的特性之间的问题，OC 语言并不是对每一个函数或者方法建立链接器符号表，而只是对每一个类创建了符号表。如果一个类有了分类，那么链接器就不知道将核心类与分类之间的代码实现联系起来，这就导致最终的应用程序中的可执行文件缺失了分类中的代码，这样函数调用就失败了。

常用参数：

- －ObjC

	链接器就会把静态库中所有的 Objective-C 类和分类都加载到最后的可执行文件中。
	
	这样编译之后的 app 会变大，因为加载了很多不必要的文件而导致可执行文件变大。但是如果静态库中有类和 category 的话只有加入这个 flag 才行。但是 Objc 也不是万能的，当静态库中只有分类而没有类的时候，Objc 就失效了，这就需要使用 -all\_load 或者 -force\_load 了。
	
	```
	@implementation MyStaticLib
	+ (void)test
	{
	    NSLog(@"sssss");
	}
	
	@end
	
	@implementation MyStaticLib (Cate)
	/**
	 * 重写方法
	 */
	+ (void)test
	{
	    NSLog(@"哈哈哈");
	}
	
	@end
	```
	
	静态库中分类重写了方法，导入工程中，设置 -Objc 参数将打印：哈哈哈；不设置将打印：sssss。

- －all_load

	会让链接器把所有找到的目标文件都加载到可执行文件中，即使没有 objc 代码。但是这个参数也有一个弊端，假如你使用了不止一个静态库文件，然后又使用了这个参数，那么你很有可能会遇到 <font color=#cc0000>ld: duplicate symbol</font> 错误，因为不同的库文件里面可能会有相同的目标文件，有两种方法解决：1、用命令行进行拆包；2、使用 -force_load 参数。

- -force_load

	适用于 Xcode3.2+ 版本，它允许 finer 得到文档加载的控制，所做的事情跟 -all\_load 其实是一样的，但是每一个 -force\_load 操作必须跟着一个文档路径，文档中的每一个对象文件将会被加载，这样的话，你就只是完全加载了一个库文件，不影响其余库文件的按需加载。

- -lstdc++

	OC 和 C++ 混编时，在 Compile 阶段一切顺利，Clang 会根据后缀（.m .cpp）选择编译器进行编译，产物都是 Object File（.o 文件）。如果一个文件调用另一个文件的方法，编译出的 Object File 中会出现 undefined symbol 去代表这个方法。在链接阶段，Linker 通过把依赖的文件也加到最终的 executable 中 resolve undefined symbol。

	Linker 没有主动的去 link stdc++ 库，解决方案 1：在 Other Linker Flags 中新增标志 -lstdc++；解决方案2：在 Linked Framework and Libraries 中添加 libstdc++.tbd。

- 总结：

	建议 -ObjC 与 -force_load 搭配使用比较好。
	
	包含静态库时需要在 Target 的 Other linker flags 里面加上值：-objC、-all\_load、-force\_load
	
	对于 64 位机器和 iPhone O S应用，解决方法是使用 -all\_load 或者 -force\_load。

- 文章：

	[Xcode 中 other linker flags 的作用](https://blog.csdn.net/bobo553443/article/details/78633340)
	[当我们在设置 Other Linker Flags -lstdc++时，我们到底在设置什么？](https://blog.csdn.net/fly1183989782/article/details/80558831)
	

## 三、Archive

![](https://upload-images.jianshu.io/upload_images/5294842-1b5c4ce68e064e85.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. iOS App Store

	保存到本地，准备上传 App Store 或者在越狱的 iOS 设备上使用，利用的是 Distribution 描述文件，关联 production 证书；

2. Ad Hoc

	保存到本地，准备在开发者账户下添加了 UDID 的设备上使用，利用的是 Distribution 描述文件，关联 production 证书；

	> 官方解释：Ad Hoc 模式的包和将来发布到 App Store 的包在各种功能测试上是一样的，只要 Ad Hoc 模式下测试（推送、内购等）没有问题，发布到 App Store 也是没有问题的。

3. Enterprise

	主要针对企业级账户下准备本地服务器分发的 app。利用的是 Distribution 描述文件，关联 production 证书；

4. Development

	保存到本地，给添加了 UDID 的设备使用，开发者模式打包 ipa，通过 development 描述文件，关联 development 证书。

	<center>
	![](https://upload-images.jianshu.io/upload_images/5294842-1c407b5f2ac4059c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	![](https://upload-images.jianshu.io/upload_images/5294842-5981e6b0bcac4065.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

5. Rebuild from Bitcode**

	如果工程 Bitcode 为 NO，则不会有此选项。

6. Strip Swift symbols

	去除 swift 符号，勾选后会让 ipa 包内存小一些，对包进行了一个压缩。如果项目中未包含 swift 代码，则没有此选项。

7. Include manifest for over-the-air installation

	勾选后用户可以在 safari 中下载应用，而不必移步 App Store。

8. Upload your app's symbols to receive symbolicated reports from Apple

	上传应用程序的符号以接收来自苹果的符号化报告。

	<center>
	![](https://upload-images.jianshu.io/upload_images/5294842-988e01e7cc35ade7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>
	
9. 学习文章

	[Understanding and Analyzing Application Crash Reports](https://developer.apple.com/library/archive/technotes/tn2151/_index.html)
	

## 四、armv7、armv7s、arm64

#### 4.1 前言

ARM 处理器，特点是体积小、低功耗、低成本、高性能，所以几乎所有手机处理器都基于 ARM，在嵌入式系统中应用广泛。

armv6｜armv7｜armv7s｜arm64 都是 ARM 处理器的指令集，这些指令集都是<font color=#cc0000>向下兼容</font>的，例如 armv7 指令集兼容 armv6，只是使用 armv6 的时候无法发挥出其性能，无法使用 armv7 的新特性，从而会导致程序执行效率没那么高。

#### 4.2 介绍

*   armv7｜armv7s｜arm64 都是 ARM 处理器的指令集
*   i386｜x86_64 是 Mac 处理器的指令集

|:-------------:|:-------------:|:-----:|
|arm64|iPhone6s \| iphone6s plus \| iPhone6 \| iPhone6 plus \| iPhone5S \| iPad Air \| iPad mini2 | 真机 64 位 |
|armv7s|iPhone5 \| iPhone5C \| iPad4|真机 32 位|
|armv7|iPhone4\|iPhone4S\|iPad\|iPad2\|iPad3\|iPad mini\|iPod Touch 3G\|iPod Touch4|真机 32 位|
|i386|针对 intel 通用微处理器 32 位处理器|模拟器 32 位|
|x86_64|是针对 x86 架构的 64 位处理器|模拟器 64 位|

模拟器并不运行 arm 代码，软件会被编译成 x86 可以运行的指令。所以生成静态库时都是会先生成两个 .a，一个是 i386 的用于在模拟器运行，另一个是在真实设备上运行的，然后再用命令将两个 .a 合并成一个。

#### 4.3 Xcode 的指令集选项

1. Architectures

	> Space-separated list of identifiers. Specifies the architectures (ABIs, processor models) to which the binary is targeted. When this build setting specifies more than one architecture, the generated binary may contain object code for each of the specified architectures. 

	指定工程被编译成可支持哪些指令集类型。支持的指令集越多，就会编译出包含多个指令集代码的数据包，对应生成二进制包就越大，也就是 ipa 包会变大。

2. Valid Architectures

	> Space-separated list of identifiers. Specifies the architectures for which the binary may be built. During the build, this list is intersected with the value of ARCHS build setting; the resulting list specifies the architectures the binary can run on. If the resulting architecture list is empty, the target generates no binary. 

	限制可能被支持的指令集的范围，也就是 Xcode 编译出来的二进制包类型最终从这些类型产生。而编译出哪种指令集的包，将由Architectures 与 Valid Architectures 的交集来确定。

	①、Valid Architectures 支持 arm 指令集版本设置为：armv7/armv7s/arm64，对应的 Architectures 支持 arm 指令集版本为：armv7s，这时 Xcode 只会生成一个 armv7s 指令集的二进制包。

	②、将 Architectures 支持 arm 指令集设置为：armv7/armv7s，对应的 Valid Architectures 的支持的指令集设置为：armv7s/arm64，那么此时，XCode 生成二进制包所支持的指令集只有 armv7s。

3. Build Active Architecture Only

	指定是否只对当前连接设备所支持的指令集编译。

	debug 时设置成 YES 是为了编译速度更快，它只编译当前的 architecture 版本；而 release 时设置为 NO 会编译所有的版本。 编译出的版本是向下兼容的，连接的设备的指令集匹配是由高到低（arm64 > armv7s > armv7）依次匹配的。比如设置为 YES，用 iphone4 编译出来的是 armv7 版本的，iphone5 也可以运行，但是 armv6 的设备就不能运行。  所以，一般 debug 的时候可以选择设置为 YES，release 的时候要改为 NO，以适应不同设备。 

	<center>
	![](http://dzliving.com/iOSArchitecture.png)
	</center>

如果你对 ipa 安装包大小有要求，可以减少安装包的指令集的数量，这样就可以尽可能的减少包的大小。当然这样做会使部分设备出现性能损失，当然在普通应用中这点体现几乎感觉不到，至少不会威胁到用户体检。

*   $(ARCHS_STANDARD)   
    默认值，以各版本实际的值为准。XCode5 中值为 armv7 armv7s，在 XCode5.1 时，强制加入了对 arm64 的编译，于是该值为 armv7,armv7s,arm64。当前 Xcode10.1 默认为 Standard architectures(armv7,arm64)。
*   $(ARCHS\_STANDARD\_32_BIT)   
    Xcode5 和 5.1 都为 armv7,armv7s，旧一点的版本中应该对应的就只有 armv7。<font color=#cc0000>（待验证）</font>
*   $(ARCHS\_STANDARD\_INCLUDING\_64\_BIT)   
    XCode5 和 5.1 都为 armv7,armv7s,arm64

使用 standard architectures (including 64-bit)(armv7,arm64) 参数，则打的包里面有 32 位、64 位两份代码，在iPhone5s（64 位）下，会首选运行 64 位代码包。包含两种架构的代码包，只有运行在 iOS6、iOS7 系统上。 

而使用 standard architectures (armv7,armv7s) 参数， 则打的包里只有 32 位代码， iPhone5s 可以兼容 32 位代码，但是这会降低 iPhone5s 的性能。 

开启 arm64 支持后，不能开发 iOS 5.1.1 之前的版本，要强制将 deployment target 设置为 5.1.1 或之后。Xcode4.5 中移除了对 arm6 的支持。

#### 4.4 查看 .a/framework 库支持的指令集

通过 lipo 命令查看 .a 库所支持的指令集。

```
$ lipo -info AFNetworking
$ lipo -info AFNetworking.framework/AFNetworking
Non-fat file: AFNetworking.framework/AFNetworking is architecture: x86_64
$ lipo -info *.a
Architectures in the fat file: libPods-AFNetworking.a are: armv7 armv7s
Architectures in the fat file: libPods.a are: armv7 armv7s
$ lipo -info libBloodTester.a
Architectures in the fat file: libBloodTester.a are: armv7 i386 x86_64 arm64
```

#### 4.5 CocoaPods与Architecture

出现问题 <font color=#cc0000>ld: library not found for -lAFNetworking</font>，需要将 pods 的 Architectures 设置成与工程 targets 里的相同。

![](https://upload-images.jianshu.io/upload_images/5294842-ac1144fb7f36f684.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 4.6 如何选择支持的指令集？

如果你的软件对安装包大小非常敏感，你可以减少安装包中的指令集数据包，而且这能达到立竿见影的效果。

很久前 xcode 支持的指令集是 armv7/armv7s，后来改成只支持 armv7 后，比原来小了 10MB 左右。目前 AppStore 上的一些知名应用，比如百度地图、腾讯地图通过反汇编工具查看后，也都只支持 armv7 指令集。<font color=#cc0000>（待验证）</font>

根据向下兼容原则，armv7 指令集的应用是可以正常在支持 armv7s/arm64 指令集的机器上运行的。
