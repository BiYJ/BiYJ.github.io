---
title: Bitcode（1）
categories: iOS原理
---

<center>
![Bitcode](https://upload-images.jianshu.io/upload_images/5294842-2880de72202acbf3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

## 一、前言

在 [What is app thinning? (iOS, tvOS, watchOS)](https://help.apple.com/xcode/mac/current/#/devbbdc5ce4f) 一节中有以下定义：

> Bitcode is an intermediate representation of a compiled program. Apps you upload to iTunes Connect that contain bitcode will be compiled and linked on the App Store. Including bitcode will allow Apple to re-optimize your app binary in the future without the need to submit a new version of your app to the store.

bitcode 是被编译程序的一种<font color=#cc0000>中间形式的代码</font>。包含 bitcode 配置的程序将会在 iTunes Connect 上被编译和链接。bitcode 允许苹果在后期重新优化我们程序的二进制文件，而不需要我们重新提交一个新的版本到 store 上。

> Bitcode. When you archive for submission to the App Store, Xcode will compile your app into an intermediate representation. The App Store will then compile the bitcode down into the 64 or 32 bit executables as necessary.

当提交程序到 App Store 上时，Xcode 会将程序编译为一个中间表现形式(bitcode)。然后 App Store 会再将这个 bitcode 编译为可执行的 64 位或 32 位程序。

来个直观图就可以明白了：

<center>
![](https://upload-images.jianshu.io/upload_images/1163291-33b9b5383c2b8905.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/731)
</center>

这样可以<font color=#cc0000>减少包的大小</font>。

## 二、Bitcode 配置

> You must rebuild it with bitcode enabled (Xcode setting ENABLE_BITCODE), obtain an updated library from the vendor, or disable bitcode for this target. for architecture arm64

要么让所有引入的第三方库都支持 bitcode，要么关闭 target 的 bitcode 选项。

在最新的 Xcode 中，新建项目默认就打开了 bitcode 设置，这导致在不知情的情况下出现项目编译失败，而这些因为 bitcode 而编译失败的的项目都链接了第三方二进制的库或者框架(.a、.framework)，而编译失败的原因就是，这些框架或者库恰好没有设置 bitcode。所以每当遇到这个情况时候大部分人都是直接设置 Xcode 关闭 bitcode 功能，全部不生成 bitcode。

<center>

|平台|bitcode|
|:-------:|:------:|
|iOS|可选的|
|watchOS|必须的|
|Mac OS|不支持|

</center>

## 2.1 苹果的要求

> 为什么苹果默认要求 watchOS 和 tvOS 的 App 要上传 bitcode？

因为把 bitcode 上传到苹果的中心服务器后，它可以为目标安装 App 的设备进行优化二进制，减小安装包的下载大小。当然 iOS 开发者也可以上传多个版本而不是打包到单个包里，但这样会占用更多的存储空间。最重要的是允许苹果可以在后台服务器对应用程序进行签名，而不用导出任何密钥到终端开发者那。

上传到服务器的 bitcode 给苹果带来更好处是：以后新设计了新指令集的新 CPU，可以继续从这份 bitcode 开始编译出新 CPU 上执行的可执行文件，以供用户下载安装。

bitcode 给开发者带来的不便之处：没用 bitcode 之前，当应用程序崩溃后，开发者可以根据获取的的崩溃日志再配上上传到苹果服务器的二进制文件的调试符号表信息，还原程序运行过程到崩溃时后调用栈信息，对问题进行定位排查。但用了 bitcode 之后，用户安装的二进制不是开发者这边生成的，而是苹果服务器经过优化后生成的，其对应的调试符号信息丢失了，也就无法进行前面说的还原崩溃现场找原因了。

打开 BitCode 时，在iTunes Connect 里"我的App"->项目->活动->所有构建版本->具体版本的"综合信息"下"包含符号"那里可以下载到 dSYM 文件。（待验证）

相当于在编译的时候加一个标记：embed-bitcode-marker(调试构建) embed-bitcode(打包/真机构建)。这个在 clang 编译器的参数是-fembed-bitcode，swift 编译器的参数是-embed-bitcode。

#### 拓展知识

目前苹果采用的编译器工具链是 LLVM，Bitcode 是 LLVM 编译器的中间代码的一种编码。LLVM 的前端可以理解为C/C++/OC/Swift 等编程语言，后端可以理解为各个芯片平台上的汇编指令或者可执行机器指令数据，那么 BitCode 就是位于这两者之间的中间码。

LLVM 的编译工作原理是前端负责把项目程序源代码翻译成 Bitcode 中间码，然后再根据不同目标机器芯片平台转换为相应的汇编指令以及翻译为机器码。这样设计就可以让 LLVM 成为了一个编译器架构，可以轻而易举的在 LLVM 架构之上发明新的语言(前端)，以及在 LLVM 架构下面支持新的 CPU(后端)指令输出。虽然 Bitcode 仅仅只是一个中间码，不能在任何平台上运行，但是它可以转化为任何被支持的 CPU 架构，包括还未被发明的 CPU 架构，也就是说现在打开 Bitcode 功能提交一个 App 到应用商店，如果以后苹果使用全新设计的 CPU，在苹果后台服务器一样可以从这个 App 的 Bitcode 开始编译转化为新 CPU 上的可执行程序，可供新手机用户下载运行这个 App。

在 iPhone 出来之前，苹果主要的编译器技术是用经过稍微改进的 GCC 工具链来把 Objective-C 语言编写的代码编译出所指定的机器处理器上原生的可执行程序。编译器产生的可执行程序叫做"Fat Binaries"--类似于 Windows 中 PE 格式的 exe 和 Linux 下的 ELF 格式的二进制。

区别是，一个"Fat Binary"可以包含同一个程序的很多版本，所以同一个可执行文件可以在不同的处理器上运行。主要就是这个技术让苹果的硬件很容易的从 PowerPC 迁移到 PowerPC64 的处理器，以及后来再迁移到 Intel 和 Intel64 处理器。这个方案带来的负面影响就是同一个文件中存了多份可执行代码，除了当前机器可执行的那一份之外其他都是无用的，这个在市场上被称为"Universal Binary"。在苹果从 PowerPC 迁移到 Intel 处理器的事情开始存在的(一个二进制文件包含一份 PowerPC 版本和一份 Intel 版本)。慢慢的后来又支持同时包含 Intel 32bit 和 Intel 64bit。

在一个 Fat binary 中，有操作系统运行时根据处理器类型动态选择正确的二进制版本来运行，但是应用程序要支持不同平台的处理器的话，应用程序本身要多占用一些空间。当然也有一些瘦身的工具，比如 lipo，可以用来移除那些当前机器中不被支持的或者多余的可执行代码达到瘦身目的，lipo不会改变程序执行逻辑，仅仅只是文件的大小瘦身。

随着移动设备移动互联网的深入发展，程序大小变得越来越重要了。主要是因为移动设备中不会有电脑上那么大的硬盘驱动器，还有就是苹果早就从原始的 ARM 处理器迁移到自家设计的 A4、A5、A5X、A6、A7、A8、A8X、A9、A9X 以及后续的 A10 处理器，它们的指令集已经发生了改变和原始 ARM 设计的有所区别，iOS 操作系统底层以及 Xcode/LLVM 编译工具一定程度上透明了这些变化，编译出来的程序会包含很多执行代码版本。面对这个问题，苹果投入大量成本迁移到 LLVM 编译器架构并使用 bitcode 的必要性越来越大。从最开始的把 OPENGL 编译为特定的 GPU 指令到把 Clang 编译器(LLCM的C/OC编译前端)支持 Objective-C 的改进并作为 Xcode 的默认编译器。

LLVM 提供了一个虚拟指令集机制，它可以翻译出指定的所支持的处理器架构的执行代码(机器码)。这个就使得为 iOS 应用程序的编译开发一个完全基于 LLVM 架构的工具链成为可能。而 LLVM 的这个虚拟的通用的指令集可以用很多种表示格式：

*   叫做 IR 的文本表示的汇编格式（像汇编语言）
*   转换为二进制数据表示的格式（像目标代码），这个二进制格式就是我们所说的 bitcode。

Bitcode 和传统的可执行指令集不同，它维护的是函数功能的类型和签名。比如，传统可执行指令集中，一系列(<=8)的布尔值可以压缩存储到单个字节中，但是在 bitcode 中它们是各自独自表示的；此外，逻辑运算操作(比如寄存器清零操作)也有它们对应的逻辑表示方法($R=0)，当这些 BitCode 要转换为特定机器平台的指令集时，它可以用经过针对特定机器平台优化过的汇编指令来代替：xor eax, eax。（这个汇编指令同样是寄存器<eax>清零操作）。

然而 bitcode 也不是完全独立于处理器平台和调用约定的，寄存器的大小在指令集中是一个相当重要的特性。众所周知，64bit 寄存器可以比 32bit 寄存器存储更多的数据，生成 64bit 平台的 bitcode 和 32bit 平台的是明显不同的；还有，调用约定可以根据函数定义或者函数调用来定义，这些可以确定函数的参数传递是传寄存器值还是压栈。一些编程语言还有如 sizeof(long) 这样的预处理指令，这些将在 bitcode 生成之前前被翻译。一般情况下，对于支持 fastcc(fast calling convention)调用的 64bit 平台会生成与其一致的 bitcode 代码。

#### 代码验证

①、在 Objective-C 代码中实现 c 方法

@implementation Test

void Greeting(void)
{
    NSLog(@"hello world!");
}

@end

@implementation Demo

void Demo(void)
{
    NSLog(@"demo func!");
}

@end

②、用 Clang 编译成 ARM64 格式且带 bitcode 的目标文件 Test.o Demo.o。

xcrun -sdk iphoneos clang -arch arm64 -fembed-bitcode -c Test.m Demo.m

③、把两个目标文件打包为一个静态库文件。

xcrun -sdk iphoneos ar-r libTest.a Test.o Demo.o

④、用 Shell 命令 otool 查看静态库文件是否包含 bitcode 段。

otool -l Test.o | grep bitcode  或  otool -l libTest.a | grep bitcode

如果输出了 2 行 sectname __bitcode，就是说明这个静态库中的两个目标文件包含了 bitcode。

⑤、用下面的命令把 Demo.m 的 C 代码转换为 ARM64 汇编语言格式 demo.s。

xcrun -sdk iphoneos clang -arch arm64 -S Demo.m

⑥、删除 Demo.m，仅留下 Test.m 和 Demo.s。

rm Demo.m 或者手动在目录中删除

⑦、把 Test.m 和 Dmeo.s 这个汇编源代码来一起带着 -fembed-bitcode 参数来生成目标代码并打包为一个静态库。

xcrun -sdk iphoneos clang -arch arm64 -fembed-bitcode -c Test.m Demo.s

xcrun -sdk iphoneos ar -r libTest.a Test.o Demo.o

⑧、再运行 otool 工具来检查这个新的静态库中包含的 2 个目标文件是否都带有 bitcode 段。

otool -l libTest.a | grep bitcode

![image.png](https://upload-images.jianshu.io/upload_images/5294842-16b53d77a41ed25d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)![image.png](https://upload-images.jianshu.io/upload_images/5294842-2f0caab6f9eb2606.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上可以看到，包含 Test.o、Demo.o 的静态库有两行 sectname __bitcode 输出，包含 Test.o、Demo.s 的静态库只有一行输出。这就说明从 ARM64 汇编语言编译过来的目标文件 Demo.s 不带有 bitcode 段，哪怕在编译的时候指定了参数 -fembed-bitcode 也没有用。

结论：bitcode 的生成是由汇编语言以上的上层语言编译而来，它是上层语言与汇编语言(机器语言)之间的一个中间码。

目前日常的 iOS 应用开发中，一般不会需要用到汇编层面去优化的代码，所以我们主要关注第三方(开源)C代码，尤其是音视频编码解码这些计算密集型项目代码，关键计算的代码针对特定平台都有对应平台的汇编版本实现，当然也有 C 的实现，但是默认编译一般都是用的汇编版本，这样就会导致我们在编译这个开源代码的时候哪怕你带了 -fembed-bitcode 参数也仅仅只是让项目中的部分 C 代码的目标文件带了 bitcode 段，而那少数的汇编代码的目标文件一样不带 bitcode 段，这样编译出这个库交给上层开发者使用的时候，就会出现在打包上传或者真机调试的时候因为 Xcode 默认开了 bitcode 功能而链接失败，导致不能真机调试或者不能上传应用到 AppStore。（需要再研究）

.s 也是支持 -fembed-bitcode 的，只是并非真正带了 bitcode(通过 .s 无法编译出 bitcode)，只是在 .o 里做了标记以兼容 bitcode 模式。

#### 参考文章

[](http://www.cocoachina.com/ios/20150818/13078.html)[lansekuangtu](http://www.cocoachina.com/ios/20150818/13078.html)：[《理解Bitcode：一种中间代码》](https://link.jianshu.com?t=http%3A%2F%2Fwww.cocoachina.com%2Fios%2F20150818%2F13078.html)  
[戴维营教育](https://www.jianshu.com/u/1d3ee2e70e3d)：[《深入理解iOS开发中的BitCode功能》](https://www.jianshu.com/p/f42a33f5eb61)

LLVM 官方文档介绍的 bitcode 文件的格式：[LLVM Bitcode File Format](http://llvm.org/docs/BitCodeFormat.html#llvm-bitcode-file-format)

[Understanding and Analyzing Application Crash Reports](https://developer.apple.com/library/archive/technotes/tn2151/_index.html#//apple_ref/doc/uid/DTS40008184-CH1-SYMBOLICATION-BITCODE)

[Troubleshooting App Thinning and Bitcode Build Failures](https://developer.apple.com/library/archive/technotes/tn2432/_index.html)