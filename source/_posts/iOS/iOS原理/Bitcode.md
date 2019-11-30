---
title: Bitcode
categories: iOS原理
---

<center>
![Bitcode](https://upload-images.jianshu.io/upload_images/5294842-2880de72202acbf3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

## 一、前言

苹果在 WWDC2015 大会上引入了 bitcode，随后在 Xcode7 中添加了在二进制中嵌入 bitcode(Enable Bitcode) 的功能，并且默认设置为开启状态。

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

## 二、什么是 Bitcode

Bitcode 是由 LLVM 引入的一种中间代码(Intermediate Representation，简称 IR)，它是源代码被编译为二进制机器码过程中的<font color=#cc0000>中间表示形态，它既不是源代码，也不是机器码</font>。从代码组织结构上看它比较接近机器码，但是在函数和指令层面使用了很多高级语言的特性。

LLVM 是一套优秀的编译器框架，目前 NDK/Xcode 均采用 LLVM 作为默认的编译器。LLVM 的编译过程可以简单分为 3 个部分:

<center>
![image](https://upload-images.jianshu.io/upload_images/5294842-2bf277be7a38c689.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

1.  前端(Frontend)负责把各种类型的源代码编译为中间表示，也就是 Bitcode。在 LLVM 体系内，不同的语言有不同的编译器前端，最常见的如 clang 负责 c/c++/oc 的编译，flang 负责 fortran 的编译，swiftc 负责 swift 的编译等等;
2.  优化(Optimizer)负责对 Bitcode 进行各种类型的优化，将 bitcode 代码进行一些逻辑等价的转换，使得代码的执行效率更高，体积更小，比如 DeadStrip/SimplifyCFG。
3.  后端(Backend)也叫 CodeGenerator，负责把优化后的 bitcode 编译为指定目标架构的机器码，比如 X86Backend 负责把 bitcode 编译为 x86 指令集的机器码。

在这个体系中，不同语言的源代码将会被转化为统一的 bitcode 格式，三个模块可以充分复用，防止重复造轮子。如果要开发一门新的语言 x，只需要造一个 x 语言的前端，将 x 语言的源代码编译为 bitcode，优化和后端的事情完全不用管。同理，如果新的芯片架构问世，则只需要基于 LLVM 重新写一套目标平台的后端，非常方便。

## 三、Bitcode 配置

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

## 四、苹果的要求

> 为什么苹果默认要求 watchOS 和 tvOS 的 App 要上传 bitcode？

因为把 bitcode 上传到苹果的中心服务器后，它可以为目标安装 App 的设备进行优化二进制，减小安装包的下载大小。当然 iOS 开发者也可以上传多个版本而不是打包到单个包里，但这样会占用更多的存储空间。<font color=#ccc0000>最重要的是允许苹果可以在后台服务器对应用程序进行签名，而不用导出任何密钥到终端开发者那</font>。

上传到服务器的 bitcode 给苹果带来更好处是：以后新设计了新指令集的新 CPU，可以继续从这份 bitcode 开始编译出新 CPU 上执行的可执行文件，以供用户下载安装。

bitcode 给开发者带来的不便之处：没用 bitcode 之前，当应用程序崩溃后，开发者可以根据获取的的崩溃日志再配上上传到苹果服务器的二进制文件的调试符号表信息，还原程序运行过程到崩溃时后调用栈信息，对问题进行定位排查。但用了 bitcode 之后，用户安装的二进制不是开发者这边生成的，而是苹果服务器经过优化后生成的，其对应的调试符号信息丢失了，也就<font color=#cc0000>无法进行前面说的还原崩溃现场找原因了</font>。

打开 BitCode 时，在iTunes Connect 里"我的App"->项目->活动->所有构建版本->具体版本的"综合信息"下"包含符号"那里可以下载到 dSYM 文件。<font color=#cc0000>（待验证）</font>

相当于在编译的时候加一个标记：embed-bitcode-marker(调试构建) embed-bitcode(打包/真机构建)。这个在 clang 编译器的参数是 \-fembed\-bitcode，swift 编译器的参数是 \-embed\-bitcode。

## 五、拓展知识

目前苹果采用的编译器工具链是 LLVM，Bitcode 是 LLVM 编译器的中间代码的一种编码。LLVM 的前端可以理解为 C/C++/OC/Swift 等编程语言，后端可以理解为各个芯片平台上的汇编指令或者可执行机器指令数据，那么 BitCode 就是位于这两者之间的中间码。

LLVM 的编译工作原理是前端负责把项目程序源代码翻译成 Bitcode 中间码，然后再根据不同目标机器芯片平台转换为相应的汇编指令以及翻译为机器码。<font color=#cc0000>这样设计就可以让 LLVM 成为了一个编译器架构</font>，可以轻而易举的在 LLVM 架构之上发明新的语言(前端)，以及在 LLVM 架构下面支持新的 CPU(后端)指令输出。虽然 Bitcode 仅仅只是一个中间码，不能在任何平台上运行，但是它可以转化为任何被支持的 CPU 架构，包括还未被发明的 CPU 架构，也就是说现在打开 Bitcode 功能提交一个 App 到应用商店，如果以后苹果使用全新设计的 CPU，在苹果后台服务器一样可以从这个 App 的 Bitcode 开始编译转化为新 CPU 上的可执行程序，可供新手机用户下载运行这个 App。

在 iPhone 出来之前，苹果主要的编译器技术是用经过稍微改进的 GCC 工具链来把 Objective-C 语言编写的代码编译出所指定的机器处理器上原生的可执行程序。编译器产生的可执行程序叫做"Fat Binaries"--类似于 Windows 中 PE 格式的 exe 和 Linux 下的 ELF 格式的二进制。

区别是，<font color=#cc0000>一个"Fat Binary"可以包含同一个程序的很多版本</font>，所以同一个可执行文件可以在不同的处理器上运行。主要就是这个技术让苹果的硬件很容易的从 PowerPC 迁移到 PowerPC64 的处理器，以及后来再迁移到 Intel 和 Intel64 处理器。这个方案带来的负面影响就是同一个文件中存了多份可执行代码，除了当前机器可执行的那一份之外其他都是无用的，这个在市场上被称为"Universal Binary"。在苹果从 PowerPC 迁移到 Intel 处理器的事情开始存在的(一个二进制文件包含一份 PowerPC 版本和一份 Intel 版本)。慢慢的后来又支持同时包含 Intel 32bit 和 Intel 64bit。

在一个 Fat binary 中，有操作系统运行时根据处理器类型动态选择正确的二进制版本来运行，但是应用程序要支持不同平台的处理器的话，应用程序本身要多占用一些空间。当然也有一些瘦身的工具，比如 <font color=#cc0000>lipo</font>，可以用来移除那些当前机器中不被支持的或者多余的可执行代码达到瘦身目的，lipo不会改变程序执行逻辑，仅仅只是文件的大小瘦身。

随着移动设备移动互联网的深入发展，程序大小变得越来越重要了。主要是因为移动设备中不会有电脑上那么大的硬盘驱动器，还有就是苹果早就从原始的 ARM 处理器迁移到自家设计的 A4、A5、A5X、A6、A7、A8、A8X、A9、A9X 以及后续的 A10 处理器，它们的指令集已经发生了改变和原始 ARM 设计的有所区别，iOS 操作系统底层以及 Xcode/LLVM 编译工具一定程度上透明了这些变化，编译出来的程序会包含很多执行代码版本。面对这个问题，苹果投入大量成本迁移到 LLVM 编译器架构并使用 bitcode 的必要性越来越大。从最开始的把 OPENGL 编译为特定的 GPU 指令到把 Clang 编译器(LLCM的C/OC编译前端)支持 Objective-C 的改进并作为 Xcode 的默认编译器。

LLVM 提供了一个虚拟指令集机制，它可以翻译出指定的所支持的处理器架构的执行代码(机器码)。这个就使得为 iOS 应用程序的编译开发一个完全基于 LLVM 架构的工具链成为可能。而 LLVM 的这个虚拟的通用的指令集可以用很多种表示格式：

*   叫做 IR 的文本表示的汇编格式（像汇编语言）
*   转换为二进制数据表示的格式（像目标代码），这个二进制格式就是我们所说的 bitcode。

<font color=#cc0000>Bitcode 和传统的可执行指令集不同，它维护的是函数功能的类型和签名</font>。比如，传统可执行指令集中，一系列(<=8)的布尔值可以压缩存储到单个字节中，但是在 bitcode 中它们是各自独自表示的；此外，逻辑运算操作(比如寄存器清零操作)也有它们对应的逻辑表示方法($R=0)，当这些 BitCode 要转换为特定机器平台的指令集时，它可以用经过针对特定机器平台优化过的汇编指令来代替：xor eax, eax。（这个汇编指令同样是寄存器<eax>清零操作）。

然而 bitcode 也不是完全独立于处理器平台和调用约定的，寄存器的大小在指令集中是一个相当重要的特性。众所周知，64bit 寄存器可以比 32bit 寄存器存储更多的数据，生成 64bit 平台的 bitcode 和 32bit 平台的是明显不同的；还有，调用约定可以根据函数定义或者函数调用来定义，这些可以确定函数的参数传递是传寄存器值还是压栈。一些编程语言还有如 sizeof(long) 这样的预处理指令，这些将在 bitcode 生成之前前被翻译。一般情况下，对于支持 fastcc(fast calling convention)调用的 64bit 平台会生成与其一致的 bitcode 代码。

## 六、初探

既然 bitcode 是代码的一种表示形式，因此它也会有自己的一套独立的语法，可以通过一个简单的例子来一探究竟，这里以 clang 为例，swift 的操作和结果可能稍有不同。

1. 先编写一段 c 语言代码(test.c)：

	```
	#include <stdio.h>
	int main(void)
	{
	    printf("hello, world.\\n");
	    return 0;
	}
	```

2. 通过以下命令可以将源代码编译为 object 文件。

	```
	$ clang -c test.c -o test.o
	$ file test.o
	test.o: Mach-O 64-bit object x86_64
	```

	这个命令同时完成了前端、优化、后端，可以通过 -emit-llvm -c 将前端这一步单独拆出来，这样就可以看到 bitcode 了。

	```
	$ clang -emit-llvm -c test.c -o test.bc   # 将源代码编译为 bitcode
	
	$ clang -c test.bc -o test.bc.o  # 将 bitcode 编译为 object
	
	$ clang -emit-llvm -c test.c -o test.bc
	$ file test.bc
	test.bc: LLVM bitcode, wrapper x86_64
	$ clang -c test.bc -o test.bc.o
	$ file test.bc.o
	test.bc.o: Mach-O 64-bit object x86_64
	$ md5 test.bc.o test.o
	MD5 (test.bc.o) = 9b90026b9c1d3fa0211e106ff921e9bd
	MD5 (test.o) = 9b90026b9c1d3fa0211e106ff921e9bd
	```

	bitcode 文件使用后缀名 .bc 表示。可以看到，将 bitcode 文件作为 clang 的输入，编出的 object 文件跟直接编源代码是相同的。

3. 查看 bitcode 文件。

	```
	$ hexdump -C test.bc | head
	00000000  de c0 17 0b 00 00 00 00  14 00 00 00 90 09 00 00  |................|
	00000010  07 00 00 01 42 43 c0 de  35 14 00 00 07 00 00 00  |....BC..5.......|
	00000020  62 0c 30 24 94 96 a6 a5  f7 d7 7f 4f d3 3e ed df  |b.0$.......O.>..|
	00000030  bd 6f ff b4 10 05 c8 14  00 00 00 00 21 0c 00 00  |.o..........!...|
	00000040  58 02 00 00 0b 82 20 00  02 00 00 00 13 00 00 00  |X..... .........|
	00000050  07 81 23 91 41 c8 04 49  06 10 32 39 92 01 84 0c  |..#.A..I..29....|
	00000060  25 05 08 19 1e 04 8b 62  80 10 45 02 42 92 0b 42  |%......b..E.B..B|
	00000070  84 10 32 14 38 08 18 4b  0a 32 42 88 48 90 14 20  |..2.8..K.2B.H.. |
	00000080  43 46 88 a5 00 19 32 42  e4 48 0e 90 11 22 c4 50  |CF....2B.H...".P|
	00000090  41 51 81 8c e1 83 e5 8a  04 21 46 06 51 18 00 00  |AQ.......!F.Q...|
	```
	
	通过 hexdump 可以看出它并非文本文件，全是乱码，这样的文件是很难分析的。其实 LLVM 提供了 llvm-dis/llvm-as 两个工具，用于将 bitcode 在二进制格式和可读的文本格式之间进行相互的转化，但遗憾的是 Xcode 的编译器工具链中并没有附带这个命令，因此需要另寻他法。

4. 我们知道通过编译器的 -S 参数可以将源代码编译为文本的 assembly 代码，不进行最后一步 assembly 到机器码的翻译工作，而 assembly 和机器码是等价的两种表示形式，bitcode 同样也是有文本和二进制(bitcode)两种等价表示形式，clang 也为 bitcode 保留了这一特性，可以通过 \-emit\-llvm \-S 将源代码编译为文本格式的 bitcode， 也叫做 LLVM Assembly Language，一般后缀名使用 .ll。

	```
	$ clang -emit-llvm -S test.c -o test.ll   \# 将源代码编译为 LLVM Assembly
	```

	test.ll 可用文本编辑器打开，全部内容：

	```
	; ModuleID = 'test.c'
	source_filename = "test.c"
	target datalayout = "e-m:o-i64:64-f80:128-n8:16:32:64-S128"
	target triple = "x86_64-apple-macosx10.13.0"
	
	@.str = private unnamed_addr constant \[15 x i8\] c"hello, world.\\0A\\00", align 1
	
	; Function Attrs: noinline nounwind ssp uwtable
	define i32 @main() #0 {
	  %1 = alloca i32, align 4
	  store i32 0, i32* %1, align 4
	  %2 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds (\[15 x i8\], \[15 x i8\]* @.str, i32 0, i32 0))
	  ret i32 0
	}
	
	declare i32 @printf(i8*, ...) #1
	
	attributes #0 = { noinline nounwind ssp uwtable "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+fxsr,+mmx,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
	attributes #1 = { "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+fxsr,+mmx,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
	
	!llvm.module.flags = !{!0}
	!llvm.ident = !{!1}
	
	!0 = !{i32 1, !"PIC Level", i32 2}
	!1 = !{!"Apple LLVM version 9.0.0 (clang-900.0.39.2)"}
	```

	这样看上去就很清晰明了了，我们重点关注下函数定义这部分，加了一些注释方便理解。

	```
	; 定义全局常量 @.str, 内容初始化为 'hello, world.\\n\\0'
	@.str = private unnamed_addr constant \[15 x i8\] c"hello, world.\\0A\\00", align 1
	
	; Function Attrs: noinline nounwind optnone ssp uwtable
	define i32 @main() #0 { ; 定义函数 @main，返回值为i32类型
	  %1 = alloca i32, align 4 ; 声明变量 %1 = 分配i32的内存空间
	  store i32 0, i32* %1, align 4 ; 将 0 存入 %1 的内存空间
	  %2 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds (\[15 x i8\], \[15 x i8\]* @.str, i32 0, i32 0)) ; 调用 @printf 函数，并将 @.str 的地址作为参数
	  ret i32 0 ; 返回 0
	}
	
	declare i32 @printf(i8*, ...) #1 ; 声明一个外部函数 @printf
	```

	这段代码不难阅读， 其含义和逻辑与我们所写的源代码基本一致，只是用了另外一种语法表示出来。因为没有经过优化，函数中的前两条语句其实是多余的，这在之后的优化阶段会被消除(dead_strip)。bitcode 的具体语法在此不做展开，虽然这个例子看起来非常简单易懂，但真实场景中，bitcode 的语法远比这个复杂，有兴趣的同学可以直接阅读 [LLVM Language Reference Manual](https://llvm.org/docs/LangRef.html)。

## 七、代码验证

1. 在 Objective-C 代码中实现 c 方法

	```
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
	```

2. 用 Clang 编译成 ARM64 格式且带 bitcode 的目标文件 Test.o Demo.o。

	```
	xcrun -sdk iphoneos clang -arch arm64 -fembed-bitcode -c Test.m Demo.m
	```

3. 把两个目标文件打包为一个静态库文件。

	```
	xcrun -sdk iphoneos ar-r libTest.a Test.o Demo.o
	```
	
4. 用 Shell 命令 otool 查看静态库文件是否包含 bitcode 段。

	```
	otool -l Test.o | grep bitcode  或  otool -l libTest.a | grep bitcode
	```
	
	如果输出了 2 行 sectname \_\_bitcode，就是说明这个静态库中的两个目标文件包含了 bitcode。

5. 用下面的命令把 Demo.m 的 C 代码转换为 ARM64 汇编语言格式 demo.s。

	```
	xcrun -sdk iphoneos clang -arch arm64 -S Demo.m
	```

6. 删除 Demo.m，仅留下 Test.m 和 Demo.s。

	```
	rm Demo.m 或者手动在目录中删除
	```

7. 把 Test.m 和 Dmeo.s 这个汇编源代码来一起带着 \-fembed\-bitcode 参数来生成目标代码并打包为一个静态库。

	```
	xcrun -sdk iphoneos clang -arch arm64 -fembed-bitcode -c Test.m Demo.s
	
	xcrun -sdk iphoneos ar -r libTest.a Test.o Demo.o
	```

8. 再运行 otool 工具来检查这个新的静态库中包含的 2 个目标文件是否都带有 bitcode 段。

	```
	otool -l libTest.a | grep bitcode
	```

	<center>
	![](https://upload-images.jianshu.io/upload_images/5294842-16b53d77a41ed25d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	![](https://upload-images.jianshu.io/upload_images/5294842-2f0caab6f9eb2606.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

从上可以看到，包含 Test.o、Demo.o 的静态库有两行 sectname \_\_bitcode 输出，包含 Test.o、Demo.s 的静态库只有一行输出。这就说明从 ARM64 汇编语言编译过来的目标文件 Demo.s 不带有 bitcode 段，哪怕在编译的时候指定了参数 \-fembed\-bitcode 也没有用。

<font color=#cc0000>结论：</font>

> bitcode 的生成是由汇编语言以上的上层语言编译而来，它是上层语言与汇编语言(机器语言)之间的一个中间码。

目前日常的 iOS 应用开发中，一般不会需要用到汇编层面去优化的代码，所以我们主要关注第三方(开源)C代码，尤其是音视频编码解码这些计算密集型项目代码，关键计算的代码针对特定平台都有对应平台的汇编版本实现，当然也有 C 的实现，但是默认编译一般都是用的汇编版本，这样就会导致我们在编译这个开源代码的时候哪怕你带了 -fembed-bitcode 参数也仅仅只是让项目中的部分 C 代码的目标文件带了 bitcode 段，而那少数的汇编代码的目标文件一样不带 bitcode 段，这样编译出这个库交给上层开发者使用的时候，就会出现在打包上传或者真机调试的时候因为 Xcode 默认开了 bitcode 功能而链接失败，导致不能真机调试或者不能上传应用到 AppStore。（需要再研究）

.s 也是支持 \-fembed\-bitcode 的，只是并非真正带了 bitcode(通过 .s 无法编译出 bitcode)，只是在 .o 里做了标记以兼容 bitcode 模式。

## 八、Enable Bitcode

在对 bitcode 有了一个直观的认识之后，再来看一下 Apple 围绕 bitcode 做了什么。Xcode 中对 Enable Bitcode 这个配置的解释是 [Xcode Help](https://help.apple.com/xcode/mac/10.1/index.html?localePath=en.lproj#/itcaec37c2a6)：

> **Enable Bitcode (ENABLE_BITCODE)**
> 
> Activating this setting indicates that the target or project should generate bitcode during compilation for platforms and architectures that support it. For Archive builds, bitcode will be generated in the linked binary for submission to the App Store. For other builds, the compiler and linker will check whether the code complies with the requirements for bitcode generation, but will not generate actual bitcode.

具体展开一下：

*   开启此设置将会在支持的平台和架构中开启 bitcode
    *   当前支持的平台主要是 iPhoneOS(armv7/arm64)，watchOS 等；
    *   注意不包括 iPhoneSimulator(i386/x86\_64) 和 macos，也就是说模拟器架构下不会编出 bitcode。这个限制只是 Xcode 自身的限制，并非编译器的限制，我们使用编译器提供的命令行工具自行操作仍然可以编译出这些架构下的bitcode，本文中的示例就是基于 macos 平台/x86\_64 架构。

*   进行 Archive 时，bitcode 会被嵌入到链接后的二进制文件中，用于提交给 App Store
    *   Enable Bitcode 设置为 YES 时，从编译日志中可以看出，Archive 时多了一个编译参数 \-fembed\-bitcode

*   进行其他类型的 Build(非 Archive)时，编译器只会检查是否满足开启 bitcode 的条件，但并不会真正生成 bitcode
    *   非 Archive 编译时，Enable Bitcode 将会增加编译参数 \-fembed\-bitcode\-marker， 只是在 object 文件中做了标记，表明可以有 bitcode，但是现在暂时没有带上它。因为本地编译调试时并不需要 bitcode，只有 AppStore 需要这玩意儿，去掉这个不必要的步骤，会加快编译速度。
    *   这就是为什么有的同学在开发 SDK 时，明明开启了 Enable Bitcode，交给客户后客户却说：你的 sdk 里没有bitcode，因为你没有使用 Archive 方式打包。
    *   当然，你可以将 Enable Bitcode 设置为 NO， 然后在 Other Compiler Flags 和 Other Linker Flags 中手动为真机架构添加 \-fembed\-bitcode 参数，这样任何类型的 Build 都会带上 bitcode。

接下来看一下 Enable Bitcode 之后，编译出的文件发生了什么变化， 直接在 clang 的参数中添加 \-fembed\-bitcode 即可。

```
$ clang -fembed-bitcode -c test.c -o test_bitcode.o
```

编译之后可以通过 otool 工具查看 object 文件的结构，此时你需要对 Mach-O 文件有一些基本的了解。

```
otool -l test_bitcode.o
test_bitcode.o:
Mach header
      magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
 0xfeedfacf 16777223          3  0x00           1     4        672 0x00002000
Load command 0
      cmd LC\_SEGMENT\_64
  cmdsize 552
  segname 
   vmaddr 0x0000000000000000
   vmsize 0x0000000000000a88
  fileoff 704
 filesize 2696
  maxprot 0x00000007
 initprot 0x00000007
   nsects 6
    flags 0x0
Section
  sectname __bitcode
   segname __LLVM
      addr 0x0000000000000040
      size 0x00000000000009a0
    offset 768
     align 2^4 (16)
    reloff 0
    nreloc 0
     flags 0x00000000
 reserved1 0
 reserved2 0
Section
  sectname __cmdline
   segname __LLVM
      addr 0x00000000000009e0
      size 0x0000000000000042
    offset 3232
     align 2^4 (16)
    reloff 0
    nreloc 0
     flags 0x00000000
 reserved1 0
 reserved2 0
Load command 1
      cmd LC\_VERSION\_MIN_MACOSX
  cmdsize 16
  version 10.13
      sdk n/a
Load command 2
     cmd LC_SYMTAB
 cmdsize 24
  symoff 3424
   nsyms 4
  stroff 3488
 strsize 56
Load command 3
            cmd LC_DYSYMTAB
        cmdsize 80
      ilocalsym 0
      nlocalsym 2
     iextdefsym 2
     nextdefsym 1
      iundefsym 3
      nundefsym 1
         tocoff 0
           ntoc 0
      modtaboff 0
        nmodtab 0
   extrefsymoff 0
    nextrefsyms 0
 indirectsymoff 0
  nindirectsyms 0
      extreloff 0
        nextrel 0
      locreloff 0
        nlocrel 0
```

或者使用 MachOView 软件。

<center>
![image.png](https://upload-images.jianshu.io/upload_images/5294842-9d803def2ae23856.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

可以发现生成的 object 文件中多了两个 Section，分别是 \_\_LLVM,\_\_bitcode 和 \_\_LLVM,\_\_cmdline，并且 otool 的输出中给出了这两个 section 在 object 文件中的偏移和大小，通过 dd 命令可以很方便地将这两个 Section 提取出来（待验证）

```
$ dd bs=1 skip=768 count=0x00000000000009a0 if=test_bitcode.o of=test_bitcode.o.bc

$ dd bs=1 skip=3608 count=0x0000000000000042 if=test_bitcode.o of=test_bitcode.o.cmdline
```

还有一种更便捷的方式，Xcode 提供的 segedit 命令可以直接将指定的 Section 导出，只需要给定 Section 的名字，和上面的命令效果是一样的，并且更为方便。

```
$ segedit -extract __LLVM __bitcode test_bitcode.o.bc -extract __LLVM __cmdline test_bitcode.o.cmdline test_bitcode.o

$ segedit -extract __LLVM __bitcode test_bitcode.o.bc 
>           -extract __LLVM __cmdline test_bitcode.o.cmdline 
>           test_bitcode.o
```

观察导出的文件：

```
$ file test_bitcode.o.bc
test_bitcode.o.bc: LLVM bitcode, wrapper x86_64
$ cat test_bitcode.o.cmdline | tr '0' ' '
-triple x86_64-apple-macosx10.13.0 -emit-obj -disable-llvm-passes
$ md5 test.bc test_bitcode.o.bc
MD5 (test.bc) = 718d88a109ba9e1a75119b04eac566f8
MD5 (test_bitcode.o.bc) = 1b3bd72eb7f380cfd6c6528674d90828
```

不难得出结论：

*   object 文件中嵌入的 \_\_LLVM,\_\_bitcode 正是完整的，未经任何加密或者压缩的 bitcode 文件，通过 -fembed-bitcode 参数，clang 把对应的 bitcode 文件整个嵌入到了 object 文件中。
*   \_\_LLVM,\_\_cmdline 是编译这个文件所用到的参数，如果要通过导出的 bitcode 重新编译这个 object 文件，必须带上这些参数
    *   导出的参数是 cc1 也就是 clang 中真正"前端"部分的参数(clang 命令其实是整合了各个环节，所以 clang 一个命令可以从源代码编出可执行文件)，所以编译时要带上 -cc1
*   导出的 bitcode 文件似乎和直接编译的 bitcode 不一样，先留个疑问，后面再研究。

首先， 来测试一下导出的 bitcode 文件结合 cmdline 能否编译出正常的 object：

```
$ clang -cc1 -triple x86_64-apple-macosx10.14.0 -emit-obj -disable-llvm-passes test_bitcode.o.bc -o test_rebuild.o
warning: overriding the module target triple with x86_64-apple-macosx10.14.0
1 warning generated.
$ file test_rebuild.o
test_rebuild.o: Mach-O 64-bit object x86_64
$ md5 test.o test_rebuild.o
MD5 (test.o) = 9b90026b9c1d3fa0211e106ff921e9bd
MD5 (test_rebuild.o) = d647be2f0a5cd4ff96b815aef8af5943
```

没有任何问题，并且通过内嵌的 bitcode 编译出的 object 文件与直接从源代码编译出来的 object 完全一样！

回到遗留的问题：为什么导出的 bitcode 文件和直接编译的 bitcode 会不一样？明明编出的 object 都是一模一样的！

这是因为二进制的 bitcode 文件中还保存了一些与实际代码无关的 meta 信息。如果能将 bitcode 转换为文本格式，将能更直观地进行对比。前面已经提到，xcode 中并没有附带转换工具，但是我们依然可以通过 clang 来完成这一操作，还记得前面用过的 -emit-llvm -S 吗？

```
$ clang -emit-llvm -S test_bitcode.o.bc -o test_bitcode.o.ll
```

神奇吧？输入虽然已经是 bitcode 了，并非源代码，但是 clang 也能"编译"出 LLVM Assembly。其实 clang 内部是先将输入的文件转换成 Module 对象，然后再执行对应的处理：

*   如果输入是源代码，会先进行前端编译，得到一个 Module；
*   如果输入是 bitcode 或者 LLVM Assembly，那么直接进行 parse 操作，即可得到 Module 对象；
*   如果输出类型是 LLVM Assembly，将 Module 对象序列化为文本格式；
*   如果输出类型是 bitcode，则将 Module 对象序列化为二进制格式

所以完全可以通过 clang 进行 bitcode 和 LLVM Assembly 的相互转换。

现在，可以对比一下前后两次生成的.ll文件：（待验证）

```
$ diff test_bitcode.o.ll test.ll

$ diff /Users/cykj/Desktop/Chart/Chart/test_bitcode.o.ll /Users/cykj/Desktop/Chart/Chart/test.ll  
1c1
< ; ModuleID = 'test_bitcode.o.bc'
---
> ; ModuleID = 'test.c'
```

除了 ModuleID，也就是来源的文件名以外，其余部分完全相同，这也就解决了前面的疑虑。

再来回顾一下，前文提到非 Archive 类型的 build，比如直接 ⌘ \+ B，即使开启了 bitcode，也不会编出 bitcode，那么会产生什么样的文件呢？通过观察编译日志可以看出 xcode 在此时使用了 -fembed-bitcode-marker 这样一个参数，试一下：

```
$ clang -fembed-bitcode-marker -c test.c -o test_bitcode_marker.o
$ otool -l test_bitcode_marker.o
Section
  sectname __bitcode
   segname __LLVM
      addr 0x0000000000000039
      size 0x0000000000000001
    offset 761
     align 2^0 (1)
    reloff 0
    nreloc 0
     flags 0x00000000
 reserved1 0
 reserved2 0
Section
  sectname __cmdline
   segname __LLVM
      addr 0x000000000000003a
      size 0x0000000000000001
    offset 762
     align 2^0 (1)
    reloff 0
    nreloc 0
     flags 0x00000000
 reserved1 0
 reserved2 0
$ objdump -s -section=__bitcode test_bitcode_marker.o

test_bitcode_marker.o:	file format Mach-O 64-bit x86-64

Contents of section __bitcode:
 0039 00
```

这样的方式编译出的文件结构与 -fembed-bitcode 的结果是一样的，唯一的区别就是 \_\_LLVM,\_\_bitcode 和 \_\_LLVM,\_\_cmdline 的内容并没有将实际的 bitcode 文件和编译参数嵌入进来，取而代之的一个字节的占位符 0x00。

## 九、Bitcode Bundle

已经搞清楚了 bitcode 是如何嵌入在 object 文件里的，但是 object 只是编译过程的中间产物，真正运行的代码是多个 object文件经过链接之后的可执行文件，接下来要分析下 object 中嵌入的 bitcode 是如何被链接的：

```
$ clang test.o -o test  # 链接原始 object

$ clang -fembed-bitcode test_bitcode.o -o test_bitcode # 链接带 bitcode 的object

$ clang test.o -o test
$ ./test
hello, world.
$ clang -fembed-bitcode test_bitcode.o -o test_bitcode
$ ./test_bitcode
hello, world.
$ otool -l test_bitcode
Section
  sectname __bundle
   segname __LLVM
      addr 0x0000000100002000
      size 0x0000000000001264
    offset 8192
     align 2^0 (1)
    reloff 0
    nreloc 0
     flags 0x00000000
 reserved1 0
 reserved2 0
```

object 中的 \_\_LLVM,\_\_bitcode 和 \_\_LLVM,\_\_cmdline不见了，取而代之的是一个 \_\_LLVM,\_\_bundle 的 Section， 通过名字可以基本推断出 object 中的 bitcode 被打包在了一起，把它从可执行文件中 dump 出来一探究竟：

```
$ segedit -extract __LLVM __bundle bundle test_bitcode
$ file bundle
bundle: xar archive version 1, SHA-1 checksum
```

这个 bundle 文件是一个 xar 格式的压缩包，xar 格式包含了一个 xml 格式的文件头(TOC)，里面用于存放各种文件的基本属性以及一些附加附加信息，可以通过 xar 命令查看并解压：

```
$ xar -d toc.xml -f bundle # 导出文件头

$ xar -x -C bundle.extract -f bundle # 解压文件

$ xar -d toc.xml -f bundle
$ mkdir bundle.extract
$ xar -x -C bundle.extract -f bundle
$ ls bundle.extract
1
$ file bundle.extract/1
bundle.extract/1: LLVM bitcode, wrapper x86_64
$ md5 bundle.extract/1 test_bitcode.o.bc
MD5 (bundle.extract/1) = 1b3bd72eb7f380cfd6c6528674d90828
MD5 (test_bitcode.o.bc) = 1b3bd72eb7f380cfd6c6528674d90828
```

查看导出的 toc.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<xar>
 <subdoc subdoc_name="Ld">
  <version>1.0</version>
  <architecture>x86_64</architecture>
  <platform>MacOSX</platform>
  <sdkversion>10.13.0</sdkversion>
  <dylibs>
   <lib>{SDKPATH}/usr/lib/libSystem.B.dylib</lib>
  </dylibs>
  <link-options>
   <option>-execute</option>
   <option>-macosx_version_min</option>
   <option>10.13.0</option>
   <option>-e</option>
   <option>_main</option>
   <option>-executable_path</option>
   <option>test_bitcode</option>
  </link-options>
 </subdoc>
 <toc>
  <checksum style="sha1">
   <size>20</size>
   <offset>0</offset>
  </checksum>
  <creation-time>2019-01-11T10:21:54</creation-time>
  <file id="1">
   <name>1</name>
   <type>file</type>
   <data>
    <archived-checksum style="sha1">d64be6fc7a9551555ccb4e8a78a87864cbef40b7</archived-checksum>
    <extracted-checksum style="sha1">d64be6fc7a9551555ccb4e8a78a87864cbef40b7</extracted-checksum>
    <size>2464</size>
    <offset>20</offset>
    <encoding style="application/octet-stream"/>
    <length>2464</length>
   </data>
   <file-type>Bitcode</file-type>
   <clang>
    <cmd>-triple</cmd>
    <cmd>x86_64-apple-macosx10.13.0</cmd>
    <cmd>-emit-obj</cmd>
    <cmd>-disable-llvm-passes</cmd>
   </clang>
  </file>
 </toc>
</xar>
```

header 的结构非常清晰，内容基本包含这些：

*   ld 的基本参数，我们链接时使用的是 clang，实际上 clang 内部调用了 ld，这里记录的是 ld的参数
    *   version: bitcode bundle 的版本号
    *   architecture: 目标架构
    *   platform: 目标平台
    *   sdkversion: sdk版本
    *   dylibs: 链接的动态库
    *   link-options: 其他链接参数
*   文件目录
    *   checksum类型
    *   创建时间
    *   每个文件的信息
        *   文件名，这里并非原始文件名，而是按照链接时输入的顺序被重命名为数字序号
        *   基本属性，包括 checksum、偏移、大小等
        *   文件类型，一般是 Bitcode，还有两种特殊类型，Object 以及 Bundle
        *   编译器类型(clang/swift)及编译参数，这部分就是 object 文件中 \_\_LLVM,\_\_cmdline 的内容
    *   下一个文件的信息(如有)
    *   重复

从 bundle 中解压出来的文件，就是 object 中嵌入的 bitcode，通过 MD5 对比可以看出链接时对 bitcode 文件自身没有做任何处理。可以注意到，用于编译各个 bitcode 文件的参数(cmdline)被放进了 TOC 中文件描述的区域，而 TOC 中多出了一个部分用于存放链接时所需要的信息和必要的参数，有了这些信息， 我们不难通过 bitcode 重新编译，并链接出一个新的可执行文件：

```
# 首先根据文件目录，将解压出的每一个bitcode文件编译为object
$ clang -cc1 -triple x86_64-apple-macosx10.14.0 -emit-obj -disable-llvm-passes bundle.extract/1 -o bundle.extract/1.o -x ir
# 由于解压出的文件没有后缀名，clang无法判断输入文件的格式，因此使用 -x ir 强制指定输入文件为ir格式
# 也可以将其重命名为1.bc，这样就不用指定-x ir

# 根据toc.xml中提供的链接参数，将所有object文件链接为可执行文件，本例中只有一个文件
$ ld 
    -arch x86_64 `# architecture` 
    -syslibroot `xcrun --show-sdk-path --sdk macosx` `# platform` 
    -sdk_version 10.14.0 `# sdkversion` 
    -lSystem `# dylibs` 
    -execute `# link-options` 
    -macosx_version_min 10.14.0 `# link-options` 
    -e _main `# link-options` 
    -executable_path test `# link-options` 
    -o test_rebuild `# 输出文件` 
    bundle.extract/1.o `# 输入文件`
$ ./test_rebuild
hello, world.
$ md5 test_rebuild test
MD5 (test_rebuild) = f4786288582decf2b8a1accb1aaa4a3c
MD5 (test) = f4786288582decf2b8a1accb1aaa4a3c
```

看！我们成功利用 bitcode 重新编了一份一模一样的可执行文件出来。

现在可以理解，为什么苹果要强推 bitcode 了吧？开发者把 bitcode 提交到 iTunes Connect 之后，如果苹果发布了使用新芯片的 iPhone，支持更高效的指令，开发者不需要做任何操作，iTunes Connect 自己就可以编译出针对新产品优化过的 app 并通过 App Store 分发给用户，不需要开发者自己重新打包上架，这样一来苹果的商店生态就不需要依赖开发者的积极性了。

## 十、使用 Bitcode 导出 ipa

前面已经提到，如果要以 bitcode 方式上传 app，必须在开启 bitcode 的状态下，进行 Archive 打包，才会得到带有 bitcode 的 app。大部分 app 都会依赖一堆第三方 sdk，如果此时项目里依赖的某一个或者几个 sdk 没有开启 bitcode，那么很遗憾，Xcode 会拒绝编译并给出类似这样的提示：

> ld: ‘name\_of\_the\_library\_or\_framework’ does not contain bitcode. You must rebuild it with bitcode enabled (Xcode setting ENABLE\_BITCODE), obtain an updated library from the vendor, or disable bitcode for this target.
> 
> ld: bitcode bundle could not be generated because ‘name\_of\_the\_library\_or_framework’ was built without full bitcode.

第一种提示表示这个第三方库完全没有开启 bitcode，而第二种提示表示它只有 bitcode-marker，也就是说它的开发者虽然在工程配置中设置了 Enable Bitcode 为 YES，但并没有以 Archive 方式编译，可能只是 ⌘ \+ B，然后顺手把 Products 拷贝出来交付了。

遇到这种问题，也需要分两种情况来看：

*   如果这个库是在本地编译的， 比如自己项目里或者子项目里的 target，或者通过 Pods 引入了源代码，那么这个 target 一定没有开启 bitcode，在工程中找到这个 target 的 Build Settings 把 Enable Bitcode 置为 YES 即可；
*   但如果是第三方提供的二进制库文件，则需要联系 sdk 的提供方确认是否能提供带 bitcode 的版本，否则只能关闭自己项目中的 bitcode。这也是 bitcode 时至今日都没有得到大面积应用的最大障阻碍。

当使用 Archive 方式打包出带有 bitcode 的包时，你会发现这个包里的二进制文件比没有开启 bitcode 时大出了许多，多出来的其实就是 bitcode 的体积，并且 bitcode 的体积，一般要比二进制文件本身还要大出许多。

```
$ ls -al test.o test_bitcode.o test.bc
-rw-r--r--  1 xelz  staff  2848 12 19 18:42 test.bc
-rw-r--r--@ 1 xelz  staff   784 12 19 18:24 test.o
-rw-r--r--@ 1 xelz  staff  3920 12 19 18:59 test_bitcode.o
$ ls -al test test_bitcode
-rwxr-xr-x@ 1 xelz  staff   8432 12 19 21:38 test
-rwxr-xr-x@ 1 xelz  staff  16624 12 19 20:50 test_bitcode
```

当然，这部分内容并不会导致用户下载到的 APP 变大，因为用户下载到的代码中只会有机器码，不会包含 bitcode。有的项目开启 bitcode 之后会发现二进制的体积增大到超出了苹果对[二进制体积的限制](https://help.apple.com/app-store-connect/#/dev611e0a21f)，但是完全不用担心，苹果的限制只是针对 \_\_TEXT 段，而嵌入的 bitcode 是存储在单独的 \_\_LLVM 段，不在苹果的限制范围内。

打包出带有 bitcode 的 xcarchive 之后，可以导出 Development IPA 进行上线前的最终测试，或者上传到 App Store Connect进行提审上架。进行此类操作时会发现 Xcode Organizer 中多出了 bitcode 相关的选项：

*   导出 Development 版本时，可以勾选 Rebuild from Bitcode，这时导出会变的很慢，因为 Xcode 在后台通过 bitcode 重新编译代码，这样导出的 ipa 最接近最终用户从 AppStore 下载的版本，为什么说是接近呢，因为苹果使用的编译器版本很可能和本地 Xcode 不一样，并且苹果可能在编译时增加额外的优化步骤，这些都会导致苹果编译后的二进制文件跟本地编译的版本产生差异。而如果不勾选此选项，则会直接使用 Archive 时编译出的二进制代码，并把 bitcode 从二进制中去除以减小体积。
    
    <center>
    ![](https://upload-images.jianshu.io/upload_images/5294842-dd76d4b61f40c4bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    </center>
    
*   导出 Store 版本或者直接进行上传时，默认会勾选 Include bitcode for iOS content，如果不勾选，则跟前面类似，将会去除内嵌的 bitcode，直接使用本地编译的二进制代码。
    
    <center>
    ![](https://upload-images.jianshu.io/upload_images/5294842-9769b90727879ca5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

    勾选后生成的 ipa 中将会只包含 bitcode，这个 ipa 是无法重签后安装到设备上进行测试的，因为里面没有任何可执行代码：
    
	<center>
    ![](https://upload-images.jianshu.io/upload_images/5294842-f5315841df4156cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

    \_\_TEXT 和 \_\_DATA 等跟已编译好的二进制相关的内容会被全部去除，但是会保留 \_\_LINKEDIT 中的部分信息，其中最重要的就是 LC\_UUID，用于在重编之后能跟原始的符号文件对应起来，如果用户下载经过 AppStore 重编之后的 app 发生了Crash，得到的 backtrace 地址是跟本地编译的版本对应不起来的，需要结合 UUID 和从 App Store Connect 下载的dSYM 文件才能得到符号化的 crash 信息。
    
    <center>
    ![](https://upload-images.jianshu.io/upload_images/5294842-7a7a5d2df3e7e1ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    </center>
    
```
Thread 0 name:  Dispatch queue: com.apple.main-thread
Thread 0 Crashed:
0   libsystem\_kernel.dylib            0x23269c84 \_\_pthread_kill + 8
1   libsystem\_pthread.dylib           0x2330bb46 pthread\_kill + 62
2   libsystem_c.dylib                 0x232000c4 abort + 108
3   libc++abi.dylib                   0x22d7a7dc \_\_cxa\_bad_cast + 0
4   libc++abi.dylib                   0x22d936a0 default\_unexpected\_handler() + 0
5   libobjc.A.dylib                   0x22d9f098 \_objc\_terminate() + 192
6   libc++abi.dylib                   0x22d90e16 std::__terminate(void (*)()) + 78
7   libc++abi.dylib                   0x22d905f4 \_\_cxxabiv1::exception\_cleanup\_func(\_Unwind\_Reason\_Code, \_Unwind\_Exception*) + 0
8   libobjc.A.dylib                   0x22d9eed2 objc\_exception\_throw + 250
9   CoreFoundation                    0x234e831e -\[__NSArrayI objectAtIndex:\] + 186
10  test                              0x000791f2 \_\_hidden#5\_ (\_\_hidden#43\_:35)
11  libdispatch.dylib                 0x2316fdd6 \_dispatch\_call\_block\_and_release + 10
12  libdispatch.dylib                 0x231794e6 \_dispatch\_after\_timer\_callback + 66
13  libdispatch.dylib                 0x2316fdc2 \_dispatch\_client_callout + 22
14  libdispatch.dylib                 0x231826d2 \_dispatch\_source\_latch\_and_call + 2042
15  libdispatch.dylib                 0x23171d16 \_dispatch\_source_invoke + 738
16  libdispatch.dylib                 0x231741fe \_dispatch\_main\_queue\_callback_4CF + 394
17  CoreFoundation                    0x23594fc4 \_\_CFRUNLOOP\_IS\_SERVICING\_THE\_MAIN\_DISPATCH\_QUEUE\_\_ + 8
18  CoreFoundation                    0x235934be __CFRunLoopRun + 1590
19  CoreFoundation                    0x234e5bb8 CFRunLoopRunSpecific + 516
20  CoreFoundation                    0x234e59ac CFRunLoopRunInMode + 108
21  GraphicsServices                  0x2475faf8 GSEventRunModal + 160
22  UIKit                             0x277d1fb4 UIApplicationMain + 144
23  test                              0x000797de main (\_\_hidden#317\_:14)
24  libdyld.dylib                     0x23198872 start + 2

\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-
 File: /Users/Breeze/Desktop/crash/test.app.dSYM/Contents/Resources/DWARF/test (armv7)
\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-
.debug_info contents:

0x00000000: Compile Unit: length = 0x00000073  version = 0x0002  abbr\_offset = 0x00000000  addr\_size = 0x04  (next CU at 0x00000077)

0x0000000b: TAG\_compile\_unit \[1\] *
             AT\_producer( "\_\_hidden#30_" )
             AT\_language( DW\_LANG_ObjC )
             AT\_name( "\_\_hidden#43_" )
             AT\_stmt\_list( 0x00000000 )
             AT\_comp\_dir( "\_\_hidden#41\_" )
             AT\_APPLE\_optimized( 0x01 )
             AT\_APPLE\_major\_runtime\_vers( 0x02 )
             AT\_low\_pc( 0x0000a0b0 )
             AT\_high\_pc( 0x0000a206 )

0x00000028:     TAG_subprogram \[2\]  
                 AT\_low\_pc( 0x0000a0b0 )
                 AT\_high\_pc( 0x0000a154 )
                 AT\_name( "\_\_hidden#45_" )

0x00000035:     TAG_subprogram \[2\]  
                 AT\_low\_pc( 0x0000a154 )
                 AT\_high\_pc( 0x0000a166 )
                 AT\_name( "\_\_hidden#1_" )

0x00000042:     TAG_subprogram \[2\]  
                 AT\_low\_pc( 0x0000a168 )
                 AT\_high\_pc( 0x0000a16e )
                 AT\_name( "\_\_hidden#2_" )

0x0000004f:     TAG_subprogram \[2\]  
                 AT\_low\_pc( 0x0000a170 )
                 AT\_high\_pc( 0x0000a176 )
                 AT\_name( "\_\_hidden#3_" )

0x0000005c:     TAG_subprogram \[2\]  
                 AT\_low\_pc( 0x0000a178 )
                 AT\_high\_pc( 0x0000a1a4 )
                 AT\_name( "\_\_hidden#44_" )

0x00000069:     TAG_subprogram \[2\]  
                 AT\_low\_pc( 0x0000a1a4 )
                 AT\_high\_pc( 0x0000a206 )
                 AT\_name( "\_\_hidden#42_" )

0x00000076:     NULL

0x00000077: Compile Unit: length = 0x000000db  version = 0x0002  abbr\_offset = 0x00000000  addr\_size = 0x04  (next CU at 0x00000156)

0x00000082: TAG\_compile\_unit \[1\] *
             AT\_producer( "\_\_hidden#30_" )
             AT\_language( DW\_LANG_ObjC )
             AT\_name( "\_\_hidden#301_" )
             AT\_stmt\_list( 0x000000bf )
             AT\_comp\_dir( "\_\_hidden#41\_" )
             AT\_APPLE\_optimized( 0x01 )
             AT\_APPLE\_major\_runtime\_vers( 0x02 )
             AT\_low\_pc( 0x0000a208 )
             AT\_high\_pc( 0x0000a796 )

0x0000009f:     TAG_subprogram \[2\]  
                 AT\_low\_pc( 0x0000a208 )
                 AT\_high\_pc( 0x0000a20c )
                 AT\_name( "\_\_hidden#315_" )

0x000000ac:     TAG_subprogram \[2\]  
                 AT\_low\_pc( 0x0000a20c )
                 AT\_high\_pc( 0x0000a20e )
                 AT\_name( "\_\_hidden#314_" )

0x000000b9:     TAG_subprogram \[2\]  
                 AT\_low\_pc( 0x0000a210 )
                 AT\_high\_pc( 0x0000a212 )
                 AT\_name( "\_\_hidden#313_" )

0x000000c6:     TAG_subprogram \[2\]  
                 AT\_low\_pc( 0x0000a214 )
                 AT\_high\_pc( 0x0000a216 )
                 AT\_name( "\_\_hidden#312_" )

0x000000d3:     TAG_subprogram \[2\]  
                 AT\_low\_pc( 0x0000a218 )
                 AT\_high\_pc( 0x0000a21a )
                 AT\_name( "\_\_hidden#311_" )

0x000000e0:     TAG_subprogram \[2\]  
                 AT\_low\_pc( 0x0000a21c )
                 AT\_high\_pc( 0x0000a22c )
                 AT\_name( "\_\_hidden#310_" )

0x000000ed:     TAG_subprogram \[2\]  
                 AT\_low\_pc( 0x0000a22c )
                 AT\_high\_pc( 0x0000a2a2 )
                 AT\_name( "\_\_hidden#309_" )

0x000000fa:     TAG_subprogram \[2\]  
                 AT\_low\_pc( 0x0000a2a4 )
                 AT\_high\_pc( 0x0000a372 )
                 AT\_name( "\_\_hidden#308_" )

0x00000107:     TAG_subprogram \[2\]  
                 AT\_low\_pc( 0x0000a374 )
                 AT\_high\_pc( 0x0000a5b6 )
                 AT\_name( "\_\_hidden#307_" )

0x00000114:     TAG_subprogram \[2\]  
                 AT\_low\_pc( 0x0000a5b8 )
                 AT\_high\_pc( 0x0000a65c )
                 AT\_name( "\_\_hidden#306_" )

0x00000121:     TAG_subprogram \[2\]  
                 AT\_low\_pc( 0x0000a65c )
                 AT\_high\_pc( 0x0000a702 )
                 AT\_name( "\_\_hidden#305_" )

0x0000012e:     TAG_subprogram \[2\]  
                 AT\_low\_pc( 0x0000a704 )
                 AT\_high\_pc( 0x0000a714 )
                 AT\_name( "\_\_hidden#304_" )

0x0000013b:     TAG_subprogram \[2\]  
                 AT\_low\_pc( 0x0000a714 )
                 AT\_high\_pc( 0x0000a73a )
                 AT\_name( "\_\_hidden#302_" )

0x00000148:     TAG_subprogram \[2\]  
                 AT\_low\_pc( 0x0000a73c )
                 AT\_high\_pc( 0x0000a796 )
                 AT\_name( "\_\_hidden#300_" )

0x00000155:     NULL

0x00000156: Compile Unit: length = 0x00000032  version = 0x0002  abbr\_offset = 0x00000000  addr\_size = 0x04  (next CU at 0x0000018c)

0x00000161: TAG\_compile\_unit \[1\] *
             AT\_producer( "\_\_hidden#30_" )
             AT\_language( DW\_LANG_ObjC )
             AT\_name( "\_\_hidden#317_" )
             AT\_stmt\_list( 0x00000320 )
             AT\_comp\_dir( "\_\_hidden#41\_" )
             AT\_APPLE\_optimized( 0x01 )
             AT\_APPLE\_major\_runtime\_vers( 0x02 )
             AT\_low\_pc( 0x0000a798 )
             AT\_high\_pc( 0x0000a7f4 )

0x0000017e:     TAG_subprogram \[2\]  
                 AT\_low\_pc( 0x0000a798 )
                 AT\_high\_pc( 0x0000a7f4 )
                 AT\_name( "\_\_hidden#316_" )

0x0000018b:     NULL
```

## 十一、bitcode 不是 bytecode

bitcode 不能翻译为字节码(bytecode)，显然从字面上看这两个词代表的含义并不等同：字节码是按照字节存取的，一般其控制代码的最小宽度是一个字节(也即 8 个bits)，而 bitcode 是按位(bit)存取，最大化利用空间。比如用 bitcode 中使用 6-bit characters来编码只包含字母/数字的字符串。

```
'a' .. 'z' ---  0 .. 25 ---> 00 0000 .. 01 1001
'A' .. 'Z' --- 26 .. 51 ---> 01 1010 .. 11 0011
'0' .. '9' --- 52 .. 61 ---> 11 0100 .. 11 1101
       '.' \-\-\- 62       ---> 11 1110
       '_' --- 63       ---> 11 1111
```

在这种编码模式下，4 字节的字符串 abcd只用 3 个字节就可以表示

```
  char:     a   |    b   |    c   |    d
binary: 00 00 00|00|00 01|00 00|10|00 00 11
   hex:     00     |     10    |    83
```

完整的编码格式可以参考官方文档[LLVM Bitcode File Format](http://llvm.org/docs/BitCodeFormat.html)

## 十二、bitcode 的兼容性

bitcode 的格式目前是一直在变化的，且无法向前兼容，举例来说 Xcode8 的编译器无法读取并解析 xcode9 产生的 bitcode。

另外苹果的 bitcode 格式与社区版 LLVM 的 bitcode 有一定差异，但苹果并不会及时开源 Xcode 最新版编译器的代码，所以如果你使用第三方基于社区版 LLVM 制作的编译器进行开发，不要尝试开启并提交 bitcode 到 App Store Connect，否则会因为App Store Connect 解析不了你的 bitcode 而被拒。

## 十三、bitcode 不是架构无关代码

如果一个 app 同时要支持 armv7 和 arm64 两种架构，那么同一个源代码文件将会被编译出两份 bitcode，也就是说，在一开始介绍 LLVM 的那张图中，并不是代表同一份 bitcode 代码可以直接被编译为不同目标机器的机器码。

LLVM 只是统一了中间语言的结构和语法格式，但不能像 Java 那样，Compile Once & Run Everywhere。

## 十四、如何判断是否开启 bitcode

可以通过 otool 检查二进制文件，网上有很多类似这样的方法：

```
otool -arch armv7 -l xxxx.a | grep __LLVM | wc -l
```

通过判断是否包含 \_\_LLVM 或者关键字来判断是否支持 bitcode，其实这种方式是完全错误的，通过前面的测试可以知道，这种方式区分不了 bitcode 和 bitcode-marker，确定是否包含 bitcode，还需要检查 otool 输出中 \_\_LLVM Segment 的长度，如果长度只有 1 个字节，则并不能代表真正开启了 bitcode：

```
$ otool -l test\_bitcode.o | grep -A 2  \_\_LLVM | grep size
      size 0x0000000000000b10
      size 0x0000000000000042
$ otool -l test\_bitcode\_marker.o | grep -A 2  __LLVM | grep size
      size 0x0000000000000001
      size 0x0000000000000001
```

## 十五、bitcode 是否能反编译出源代码

从科学严谨的角度来说，无法给出确定的答案，但是这个问题跟"二进制文件是否能反编译出源代码"是一样的道理。编译是一个将源代码一层一层不断低级化的过程，每一层都可能会丢失一些特性，产生不可逆的转换，把源代码编译为 bitcode 或是二进制机器码是五十步之于百步的关系。在通常情况下，反编译 bitcode 跟反编译二进制文件比要相对容易一些，但通过 bitcode 反编译出和源代码语义完全相同的代码，也是几乎不可能的。

另外，从安全的角度考虑，Xcode 引入了  Symbol Hiding 和 Debug info Striping 机制，在链接时，bitcode 中所有非导出符号均被隐藏，取而代之的是 \_\_hidden#0\_ 或者 \_\_ir\_hidden#1\_ 这样的形式，debug 信息也只保留了 line-table，所有跟文件路径、标识符、导出符号等相关的信息全部都从 bitcode 中移除，相当于做了一层混淆，防止源代码级别的信息泄露，可谓是煞费苦心。

## 十六、参考文章

[](http://www.cocoachina.com/ios/20150818/13078.html)
[lansekuangtu](http://www.cocoachina.com/ios/20150818/13078.html) & [《理解Bitcode：一种中间代码》](https://link.jianshu.com?t=http%3A%2F%2Fwww.cocoachina.com%2Fios%2F20150818%2F13078.html)  
[戴维营教育](https://www.jianshu.com/u/1d3ee2e70e3d) & [《深入理解iOS开发中的BitCode功能》](https://www.jianshu.com/p/f42a33f5eb61)
LLVM 官方文档介绍的 bitcode 文件的格式：[LLVM Bitcode File Format](http://llvm.org/docs/BitCodeFormat.html#llvm-bitcode-file-format)
[Understanding and Analyzing Application Crash Reports](https://developer.apple.com/library/archive/technotes/tn2151/_index.html#//apple_ref/doc/uid/DTS40008184-CH1-SYMBOLICATION-BITCODE)
[Troubleshooting App Thinning and Bitcode Build Failures](https://developer.apple.com/library/archive/technotes/tn2432/_index.html)
[http://xelz.info/blog/2018/11/24/all-you-need-to-know-about-bitcode/](http://xelz.info/blog/2018/11/24/all-you-need-to-know-about-bitcode/)
[iOS 打包上线 bitcode问题](https://blog.csdn.net/u012198553/article/details/54380856)
[判断一个库是否包含 bitcode](https://www.tuicool.com/articles/nMzABvM)