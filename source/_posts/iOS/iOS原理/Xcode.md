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

5. Rebuild from Bitcode

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


## 五、Build Settings

#### 5.1 Generate Debug Symbols 

官方文档说明：

> Enables or disables generation of debug symbols. When debug symbols are enabled, the level of detail can be controlled by the build 'Level of Debug Symbols' setting. 

调试符号是在编译时生成的。在 Xcode 中查看构建过程，可以发现，当 Generate Debug Symbols 选项设置为 YES 时，每个源文件在编译成 `.o` 文件时，编译参数多了 `-g` 和 `-gmodules` 两项。但链接等其他的过程没有变化。

Clang 文档对 `-g` 的描述是：

```
Generate complete debug info.
```

当 Generate Debug Symbols 设置为 `YES` 时，编译产生的 .o 文件会大一些，当然最终生成的可执行文件也大一些。

当 Generate Debug Symbols 设置为 `NO` 的时候，在 Xcode 中设置的断点不会中断。但是在程序中打印 [NSThread callStackSymbols]，依然可以看到类名和方法名，比如：

```
** 0   Demo           0x00000001000667f4 -[ViewController viewDidLoad] + 100**
```

在程序崩溃时，也可以得到带有类名和方法名的函数调用栈。

#### 5.2 Debug Information Level 

有两个选项：Compiler default、Line tables only。

官方文档的描述是：

> Toggles the amount of debug information emitted when debug symbols are enabled. This can impact the size of the generated debug information, which can matter in some cases for large projects (such as when using LTO). 

当把 Debug Information Level 设置为 Line tables only 的时候，然后构建 app，每个源文件在编译时，都多了一个编译参数：`-gline-tables-only`。

Clang的文档中这样解释 -gline-tables-only：

> Generate line number tables only.
This kind of debug info allows to obtain stack traces with function names, file names and line numbers (by such tools as gdb or addr2line). It doesn’t contain any other data (e.g. description of local variables or function parameters).
>
>这种类型的调试信息允许获得带有函数名、文件名和行号的函数调用栈，但是不包含其他数据（比如局部变量和函数参数）。

所以当 Debug Information Level 设置为 Line tables only 的时候，断点依然会中断，但是无法在调试器中查看局部变量的值。


#### 5.3 Strip Linked Product

当把这一设置选项改为 NO 的时候，最终构建生成的 app 大小没有任何变化。

原来，Strip Linked Product 也受到 Deployment Postprocessing 设置选项的影响。在 Build Settings 中可以看到，Strip Linked Product 是在 Deployment 这栏中的，而 Deployment Postprocessing 相当于是 Deployment 的总开关。

把 Deployment Postprocessing 设置为 YES，对比 Strip Linked Product 设为 YES 和 NO 的这两种情况，发现当 Strip Linked Product 设为 YES 的时候，app 的构建过程多了这样两步：

1. 在 app 构建的开始，会生成一些 .hmap 辅助文件
2. 在 app 构建的末尾，会执行 Strip 操作。

当 Strip Linked Product 设为 YES 的时候，运行 app，断点不会中断，在程序中打印 [NSThread callStackSymbols] 也无法看到类名和方法名：

```
** 0   Demo                      0x000000010001a7f4 XSQSymbolsDemo + 26612**
```

而在程序崩溃时，函数调用栈中也无法看到类名和方法名，注意右上角变成了 unnamed_function。

DEBUG 下设为 NO，RELEASE 下设为 YES，用于 RELEASE 模式下缩减 app 的大小。


#### 5.4 Strip Style

官方文档描述：

>Defines the level of symbol stripping to be performed on the linked product of the build. The default value is defined by the target's product type. [STRIP_STYLE]
>
>All Symbols - Completely strips the binary, removing the symbol table and relocation information. [all, -s]
Non-Global Symbols - Strips non-global symbols, but saves external symbols. [non-global, -x]
Debugging Symbols - Strips debugging symbols, but saves local and global symbols. [debugging, -S]

选择不同的 Strip Style 时，app 构建末尾的 Strip 操作会被带上对应的参数。

如果选择 debugging symbols 的话，函数调用栈中，类名和方法名还是可以看到的。

如果我们构建的不是一个 app，而是一个静态库，需要注意，静态库是不可以 strip all 的。这时构建会失败。想想符号在重定位时的作用，如果构建的静态库真的能剥离所有符号，那么它也就没法被链接了。

#### 5.5 Strip Debug Symbols During Copy

网上有很多文章，以为 Strip Debug Symbols During Copy 开启的时候，app 中的调试符号会被剥离掉。感觉他们混淆了 Strip Linked Product 和 Strip Debug Symbols During Copy 的用法。

文档上的描述是：

> Activating this setting causes binary files which are copied during the build (e.g., in a Copy Bundle Resources or Copy Files build phase) to be stripped of debugging symbols. It does not cause the linked product of a target to be stripped (use Strip Linked Product for that).

Strip Debug Symbols During Copy 中的 During Copy 是什么意思呢？我觉得可能是 app 中引入的某些类型的库，在 app 的构建过程中需要被复制一次。虽然我暂时没找全究竟什么样的“库”需要在 app 构建时被复制，但是我发现，当 app 中包含 extension 或者 watch app 的时候，构建过程中会有 Copy 的步骤。

当将 app（而非 extension）的 Strip Debug Symbols During Copy 设置为 YES 之后，在这句 copy 的命令中会多出 `-strip-debug-symbols` 参数。

但是这里，strip 并不能成功，并且出现了 warning：

```
warning: skipping copy phase strip, binary is code signed: /Users/xsq/Library/Developer/Xcode/DerivedData/XSQSymbolsDemo-cysszdsykroyyddkvvyffgboglvo/Build/Products/Debug-iphoneos/Today.appex/Today
```

这似乎是由于 app 中的 today extention 已经经过了 code sign，导致无法被篡改引起的警告。

那么如果略过 code sign 的过程，是否就能成功 strip 呢？我想使用模拟器调试可以略过 code sign过程，于是便在模拟器上试了试。果然这个 warning 消失了。

Strip Debug Symbols During Copy 设置为 YES 时，打开对应 .app 文件的“显式包内容”，可以看到，/PlugIns/Today.appex 文件的大小变小了。（不过这些只能在使用模拟器时奏效）

Strip Debug Symbols During Copy 置为 YES 的时候，today extension 中的断点将不会中断，但是打印 [NSThread callStackSymbols] 时的类名和方法名还是可以看见的。

DEBUG 下设为 NO，RELEASE 下设为 YES。
 
 
#### 5.6 Debug Information Format

官方文档解释：

> This setting controls the format of debug information used by the developer tools. [DEBUG\_INFORMATION\_FORMAT]
>
>DWARF - Object files and linked products will use DWARF as the debug information format. [dwarf]
DWARF with dSYM File - Object files and linked products will use DWARF as the debug information format, and Xcode will also produce a dSYM file containing the debug information from the individual object files (except that a dSYM file is not needed and will not be created for static library or object file products). [dwarf-with-dsym]

当 Debug Information Format 为 `DWARF with dSYM File` 的时候，构建过程中多了一步`Generate dSYM File`.

最终产出的文件也多了一个 dSYM 文件。

不过，既然这个设置叫做 Debug Information Format，所以首先得有调试信息。如果此时 Generate Debug Symbols 选择的是 NO 的话，是没法产出 dSYM 文件的。

<font color=#cc0000>dSYM 文件的生成，是在 Strip 等命令执行之前</font>。所以无论 Strip Linked Product 是否开启，生成的 dSYM 文件都不会受影响。

不过正如文档中所说，无法为静态库生成 dSYM 文件。即便为给一个静态库的 Debug Information Format 设置为 DWARF with dSYM File，构建过程中依然不会有生成 dSYM 文件的步骤。


#### 5.7 Dead Code Stripping 

yes - 确定 dead code（代码被定义但从未被调用）被剥离，去掉冗余的代码。
	
#### 5.8 Optimization Level

编译器优化级别。

比较早期的时候，硬件资源是比较缺乏的。为了提高性能，开发编译器的大师们，都会对编译器(从c到汇编的编译过程)加上一定的优化策略。优化后的代码效率比较高，但是可读性比较差，且编译时间更长。

优化是指编译器一级的措施，与机器指令比较接近，所以很可能会导致硬件不兼容，进而产生了你目前遇到的软件装不上的问题。 

* None：不优化。[-O0] 与此设置，编译器的目标是降低成本的编译和调试产生预期的结果。语句是独立的：如果你停止程序语句之间有一个断点,然后您可以指定一个新值的任何变量或任何其他声明改变程序计数器的功能和得到你期望的结果从源代码。
* Fast：优化编译需要更多的时间，和更多的内存大的功能。[-O,O1] 与此设置，编译器试图减少代码大小和执行时间，没有执行任何优化,需要大量的编译时间。在苹果的编译器,严格的混叠,阻挡重新排序,并内嵌调度优化时默认是禁用的。
* Faster：编译器执行几乎所有支持优化，不涉及 space-speed 权衡。[-O2]与此设置,编译器不执行循环展开或内联函数,或寄存器重命名。相比“快速”的设置,该设置增加编译时间和生成的代码的性能。
* Fastest:打开“更快”指定的优化设置,也取决于函数内联和寄存器重命名选项。此设置可能会导致一个更大的二进制。
* Fastest [-O3],最小的优化尺寸。这个设置允许所有“更快”的优化通常不增加代码大小。它还旨在减少代码的大小进行进一步的优化。

	
#### 5.9 Symbols Hidden by Default 

会把所有符号都定义成”private extern”，设了后会减小体积。


#### 5.10 Other C Flags

1. -fembed-bitcode

	让 framework 支持 bitcode。如果不进行以上操作。别人在集成你的 framework 时可以编译，亦可以真机测试。唯独在打包时会发出警告并打包失败：警告为 framework 不支持 bitcode！

#### 5.11 Other Warning Flags

忽略整个工程中的指定类型的警告。例如填入 "-Wno-deprecated-declarations"
	
![](https://img-blog.csdn.net/20160214130836151?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


## 六、忽略警告

```
#pragma clang diagnostic push
#pragma clang diagnostic ignored"-Wdeprecated-declarations"

// 写在这个中间的代码，都不会被编译器提示 -Wdeprecated-declarations 类型的警告
dispatch_queue_t currentQueue = dispatch_get_current_queue();

#pragma clang diagnostic pop
```


[Xcode缩小ipa包大小及symbols设置等](https://blog.csdn.net/u013030990/article/details/72621380)


## 七、@available、#available

Swift 2.0 中,引入了可用性的概念。对于函数、类、协议等，可以使用 @available 声明这些类型的生命周期依赖于特定的平台和操作系统版本。而 #available 用在判断语句中(if, guard, while等)，在不同的平台上做不同的逻辑。

```
@available(iOS 9, *) func myMethod() { 
    
  // do something 
}      
```

[iOS @available 和 #available 的用法](https://www.jianshu.com/p/fe59934a0152)
[Xcode 9.0中新增的API版本检查@available](https://www.jianshu.com/p/b877be6d6570)