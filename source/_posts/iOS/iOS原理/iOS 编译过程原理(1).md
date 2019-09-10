---
title: iOS 编译过程原理(1)
categories: iOS原理
---


## 一、前言

一般可以将编程语言分为两种，[编译语言](https://zh.wikipedia.org/wiki/%E7%B7%A8%E8%AD%AF%E8%AA%9E%E8%A8%80)和[直译式语言](https://en.wikipedia.org/wiki/Interpreted_language)。

像 C++、Objective-C 都是编译语言。编译语言在执行的时候，必须<font color=#cc0000>先通过编译器生成机器码</font>，机器码可以直接在 CPU 上执行，所以执行效率较高。

像 JavaScript、Python 都是直译式语言。直译式语言不需要经过编译的过程，而是<font color=#cc0000>在执行的时候通过一个中间的解释器将代码解释为 CPU 可以执行的代码</font>。所以，较编译语言来说，直译式语言效率低一些，但是编写的更灵活。

iOS 开发目前的常用语言：Objective-C 和 Swift。二者都是编译语言，换句话说都是需要编译才能执行的。它们的编译都是依赖于 Clang(swift) + LLVM。本文只关注 Objective-C，原理上大同小异。

充分理解了编译的过程，会对你的开发大有帮助。本文的最后，会以以下几个例子，来讲解如何合理利用 XCode 和编译

*   \_\_attribute\_\_
*   Clang 警告处理
*   预处理
*   插入编译期脚本
*   提高项目编译速度


## 二、iOS 编译

Objective-C 采用 Clang 作为前端，而 Swift 则采用 swift() 作为前端，都是 LLVM（Low level vritual machine）作为编译器后端。所以简单的编译过程如图：

<center>
![1](https://upload-images.jianshu.io/upload_images/5294842-3a4fe4a482e93bda.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

其中，swift 的编译命令可以在这里找到

```
/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/swift
```

可以通过 Clang，来<font color=#cc0000>查看一个文件的编译具体过程</font>，新建 Demo.m

```
#import <Foundation/Foundation.h>
  
int main(){
    @autoreleasepool {
        NSLog(@"%@",@"Hello Leo");
    }
    return 0;
}
```

然后终端输入：

```
~ $ cd /Users/dubin/Desktop/Demo/Demo
Demo $ 
Demo $ clang -ccc-print-phases -framework Foundation Demo.m -o Demo 
0: input, "Foundation", object
1: input, "Demo.m", objective-c
2: preprocessor, {1}, objective-c-cpp-output  // 预处理
3: compiler, {2}, ir                          // 编译生成 IR（中间代码）
4: backend, {3}, assembler                    // 汇编器生成汇编代码
5: assembler, {4}, object                     // 生成机器码
6: linker, {0, 5}, image                      // 链接
7: bind-arch, "x86_64", {6}, image            // 生成 Image，也就是最后的可执行文件
```

在终端运行这个程序：

```
Demo $ gcc -framework Foundation Demo.m -o Demo
$ ./Demo
2019-03-27 13:55:30.426 Demo[14155:5478670] Hello Leo
```

<center>
![2](https://upload-images.jianshu.io/upload_images/5294842-9a33d4c189414790.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

另一种终端运行 OC 程序的顺序：

```
Demo $ cc -c Demo.m
Demo $ cc Demo.o -framework Foundation
ld: warning: text-based stub file /System/Library/Frameworks//Foundation.framework/Foundation.tbd and library file /System/Library/Frameworks//Foundation.framework/Foundation are out of sync. Falling back to library file for linking.
ld: warning: text-based stub file /System/Library/Frameworks//CoreFoundation.framework/Versions/A/CoreFoundation.tbd and library file /System/Library/Frameworks//CoreFoundation.framework/Versions/A/CoreFoundation are out of sync. Falling back to library file for linking.
Demo $ ./a.out
2019-03-27 14:01:52.933 a.out[14413:5483194] Hello Leo
```

cc -c tst.m    编译：生成 tst.o文件

cc -c man.m        编译： 生成 man.o 文件

cc tst.o man.o -framework Foundation  链接、合并：生成 a.out 可执行文件

./a.out 运行

当然 OC 程序还可以混编 C 程序，格式为：cc -c x.m x.c，或者直接将编译和链接合在一起：cc x.m x.c

1. 编译器前端

	编译器前端的任务是：语法分析、语义分析、生成中间代码（intermediate representation）。在这个过程中<font color=#cc0000>会进行类型检查，如果发现错误或者警告会标注出来在哪一行</font>。

	<center>
	![3](https://upload-images.jianshu.io/upload_images/5294842-eba0d4d3a3fbec8e?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

2. 编译器后端

	编译器后端会进行机器无关的代码优化，生成机器语言，并且进行机器相关的代码优化。iOS 的编译过程，后端的处理如下

	①、LVVM 优化器会进行 BitCode 的生成，链接期优化等等。
	
	![4](https://upload-images.jianshu.io/upload_images/5294842-7fc3096f9cf495e4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	
	②、LLVM 机器码生成器会针对不同的架构，比如 arm64 等生成不同的机器码。
	
	![5](https://upload-images.jianshu.io/upload_images/5294842-2037b2a6480b4d2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 三、执行一次 XCode build 的流程

当你在 XCode 中，选择 build 的时候（快捷键 command+B），会执行如下过程

*   编译信息写入辅助文件，创建编译后的文件架构（name.app）
*   处理文件打包信息，例如在 debug 环境下

```
Entitlements:
{
    "application-identifier" = "app的bundleid";
    "aps-environment" = development;
}
```

*   执行 CocoaPod 编译前脚本。例如对于使用 CocoaPod 的工程会执行 <font color=#cc0000>CheckPods Manifest.lock</font>
*   编译各个 .m 文件，使用 <font color=#cc0000>CompileC</font> 和 <font color=#cc0000>clang</font> 命令。

```
CompileC ClassName.o ClassName.m normal x86_64 objective-c com.apple.compilers.llvm.clang.1_0.compiler
export LANG=en_US.US-ASCII
export PATH="..."
clang -x objective-c -arch x86_64 -fmessage-length=0 -fobjc-arc... -Wno-missing-field-initializers ... -DDEBUG=1 ... -isysroot iPhoneSimulator10.1.sdk -fasm-blocks ... -I 上文提到的文件 -F 所需要的Framework  -iquote 所需要的Framework  ... -c ClassName.c -o ClassName.o
```

通过这个编译的命令，我们可以看到

```
clang是实际的编译命令
-x 		objective-c 指定了编译的语言
-arch 	x86_64制定了编译的架构，类似还有arm7等
-fobjc-arc 一些列-f开头的，指定了采用arc等信息。这个也就是为什么你可以对单独的一个.m文件采用非ARC编程。
-Wno-missing-field-initializers 一系列以-W开头的，指的是编译的警告选项，通过这些你可以定制化编译选项
-DDEBUG=1 一些列-D开头的，指的是预编译宏，通过这些宏可以实现条件编译
-iPhoneSimulator10.1.sdk 制定了编译采用的iOS SDK版本
-I 把编译信息写入指定的辅助文件
-F 链接所需要的Framework
-c ClassName.c 编译文件
-o ClassName.o 编译产物
```

*   链接需要的 Framework，例如 Foundation.framework、AFNetworking.framework、AliPay.framework
*   编译 xib 文件
*   拷贝 xib，图片等资源文件到结果目录
*   编译 ImageAssets
*   处理 info.plist
*   执行 CocoaPod 脚本
*   拷贝 Swift 标准库
*   创建 .app 文件和对其签名

## 四、ipa 包的内容

例如，通过 iTunes Store 下载微信，获得 ipa 安装包，然后实际看看其安装包的内容。

<center>
![6](https://upload-images.jianshu.io/upload_images/5294842-475c1b17e4bc6ae3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

*   右键 ipa，重命名为 .zip
*   双击 zip 文件，解压缩后会得到一个文件夹。所以，<font color=#cc0000>ipa 包就是一个普通的压缩包</font>。

<center>
![7](https://upload-images.jianshu.io/upload_images/5294842-a17d16dddf291011.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

*   右键图中的 [WeChat[，选择显示包内容，然后就能够看到实际的 ipa 包内容了。

## 五、二进制文件的内容

通过 XCode 的 Link Map File，我们可以窥探二进制文件中布局。在 XCode -> Build Settings -> 搜索 map -> 开启Write Link Map File。

<center>
![8](https://upload-images.jianshu.io/upload_images/5294842-ad796aa51811238f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

开启后，再编译，我们可以在对应的 Debug/Release 目录下看到对应的 link map 的 text 文件。

默认的目录：

```
~/Library/Developer/Xcode/DerivedData/<TARGET-NAME>-对应ID/Build/Intermediates/<TARGET-NAME>.build/Debug-iphoneos/<TARGET-NAME>.build/
```

例如 TargetName是 Demo 的目录：

```
/Users/cykj/Library/Developer/Xcode/DerivedData/Demo-bifdsullasutjrbgiiywunmnlmjf/Build/Intermediates.noindex/Demo.build/Debug-iphonesimulator/Demo.build
```

<center>
![9](https://upload-images.jianshu.io/upload_images/5294842-9879ec4e25c96df9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>
  
这个映射文件的主要包含以下部分：

1. Object files

	这个部分包括的内容：
	
	*   .o 文文件，也就是上文提到的 .m 文件编译后的结果。
	*   .a 文件
	*   需要 link 的 framework

	```
	# Arch: x86_64
	# Object files:
	[  0] linker synthesized
	[  1] dtrace
	[  2] /Users/cykj/Library/Developer/Xcode/DerivedData/Demo-bifdsullasutjrbgiiywunmnlmjf/Build/Intermediates.noindex/Demo.build/Debug-iphonesimulator/Demo.build/Objects-normal/x86_64/MyProxy.o
	[  3] /Users/cykj/Library/Developer/Xcode/DerivedData/Demo-bifdsullasutjrbgiiywunmnlmjf/Build/Intermediates.noindex/Demo.build/Debug-iphonesimulator/Demo.build/Objects-normal/x86_64/ViewController.o
	[  4] /Users/cykj/Library/Developer/Xcode/DerivedData/Demo-bifdsullasutjrbgiiywunmnlmjf/Build/Intermediates.noindex/Demo.build/Debug-iphonesimulator/Demo.build/Objects-normal/x86_64/Person.o
	[  5] /Users/cykj/Library/Developer/Xcode/DerivedData/Demo-bifdsullasutjrbgiiywunmnlmjf/Build/Intermediates.noindex/Demo.build/Debug-iphonesimulator/Demo.build/Objects-normal/x86_64/MyOperation.o
	[  6] /Users/cykj/Library/Developer/Xcode/DerivedData/Demo-bifdsullasutjrbgiiywunmnlmjf/Build/Intermediates.noindex/Demo.build/Debug-iphonesimulator/Demo.build/Objects-normal/x86_64/main.o
	
	
	[130] /Applications/Xcode10.1.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator12.1.sdk/System/Library/Frameworks//CoreGraphics.framework/CoreGraphics.tbd
	```

	这个区域的存储内容比较简单：前面是文件的编号，后面是文件的路径。文件的编号在后续会用到。

2. Sections

	这个区域<font color=#cc0000>提供了各个段（Segment）和节（Section）在可执行文件中的位置和大小</font>。这个区域完整的描述了可执行文件中的全部内容。其中，段分为两种：

	*   \_\_TEXT 代码段
	*   \_\_DATA 数据段

	从 Sections 区域可以看到，代码段的 \_\_text 节的地址是 0x100001000，大小是 0x000A5FF9，而二者相加的下一个位置正好是 \_\_stubs 的位置 0x1000A6FFA。
	
	```
	# Sections: 
	# 位置           大小            段       节 
	# Address       Size            Segment Section
	0x100001000	0x000A5FF9	__TEXT	__text             // 代码
	0x1000A6FFA	0x000003D8	__TEXT	__stubs
	0x1000A73D4	0x00000678	__TEXT	__stub_helper
	0x1000A7A4C	0x0000794A	__TEXT	__objc_methname    // OC 方法名
	0x1000AF396	0x000079F4	__TEXT	__cstring          // 字符串
	0x1000B6D8A	0x0000092C	__TEXT	__objc_classname   // OC 类名
	0x1000B76B6	0x00002293	__TEXT	__objc_methtype    // OC 方法类型
	0x1000B9950	0x000000E8	__TEXT	__const            // 常量
	0x1000B9A38	0x000043DC	__TEXT	__gcc_except_tab
	0x1000BDE14	0x0000004A	__TEXT	__ustring
	0x1000BDE5E	0x00000166	__TEXT	__entitlements
	0x1000BDFC4	0x0000037B	__TEXT	__dof_RACSignal
	0x1000BE33F	0x000002E8	__TEXT	__dof_RACCompou
	0x1000BE628	0x000009CC	__TEXT	__unwind_info
	0x1000BF000	0x00000010	__DATA	__nl_symbol_ptr
	0x1000BF010	0x000001B8	__DATA	__got
	0x1000BF1C8	0x00000520	__DATA	__la_symbol_ptr
	0x1000BF6E8	0x00005D28	__DATA	__const
	0x1000C5410	0x00002E80	__DATA	__cfstring
	0x1000C8290	0x00000268	__DATA	__objc_classlist   // OC 方法列表
	0x1000C84F8	0x000001C0	__DATA	__objc_catlist
	0x1000C86B8	0x00000098	__DATA	__objc_protolist   // OC 协议列表
	0x1000C8750	0x00000008	__DATA	__objc_imageinfo
	0x1000C8758	0x0000F0C0	__DATA	__objc_const       // OC 常量
	0x1000D7818	0x00001B28	__DATA	__objc_selrefs
	0x1000D9340	0x00000040	__DATA	__objc_protorefs
	0x1000D9380	0x00000360	__DATA	__objc_classrefs
	0x1000D96E0	0x00000170	__DATA	__objc_superrefs   // OC 父类引用
	0x1000D9850	0x00000610	__DATA	__objc_ivar        // OC ivar
	0x1000D9E60	0x00001810	__DATA	__objc_data
	0x1000DB670	0x00000768	__DATA	__data
	0x1000DBDD8	0x0000015F	__DATA	__bss
	```

3. Symbols

	<font color=#cc0000>Section 部分将二进制文件进行了一级划分。而 Symbols 对 Section 中的各个段进行了二级划分</font>，例如，对于 \_\_TEXT \_\_text 表示代码段中的代码内容。

	```
	0x100001000	0x000A5FF9	__TEXT	__text             // 代码
	```

	而对应的 Symbols，起始地址也是 0x1000021B0。其中，文件编号和上文的编号对应

	```
	0x100001000	0x000000A0	[  2] +[MyProxy proxyWithObj:]
	
	
	[ 2] /Users/cykj/Library/Developer/Xcode/DerivedData/Demo-bifdsullasutjrbgiiywunmnlmjf/Build/Intermediates.noindex/Demo.build/Debug-iphonesimulator/Demo.build/Objects-normal/x86_64/MyProxy.o 
	```

	具体内容：

	```
	# Symbols:
	# 地址           大小            文件编号 方法名
	# Address	Size    	File  Name
	0x100001000	0x000000A0	[  2] +[MyProxy proxyWithObj:]
	0x1000010A0	0x00000040	[  2] -[MyProxy methodSignatureForSelector:]
	0x1000010E0	0x000003F0	[  2] -[MyProxy forwardInvocation:]
	0x1000014D0	0x00000040	[  2] -[MyProxy .cxx_destruct]
	0x100001510	0x00000048	[  2] -[Dog barking:]
	0x100001560	0x00000060	[  3] -[ViewController dealloc]
	0x1000015C0	0x000001C0	[  3] -[ViewController drawImage:
	...
```

到这里，我们知道 OC 的方法是如何存储的，再来看看 ivar 是如何存储的。

首先找到数据栈中 \_\_DATA \_\_objc_ivar

```
0x1000D9850	0x00000610	__DATA	__objc_ivar
```

然后，搜索这个地址 0x1000D9850，就能找到 ivar 的存储区域。

```
0x1000D9850	0x00000008	[  2] _OBJC_IVAR_$_MyProxy.__innerObj
```

值得一提的是，对于 String，会显式的存储到数据段中，例如：

```
0x1000AF3F2	0x00000004	[  3] literal string: http://www.baidu.com
```

所以，若果你的加密 Key 以明文的形式写在文件里，是一件很危险的事情。

## 六、dSYM 文件

在每次编译过后，都会生成一个 dsym 文件。<font color=#cc0000>dsym 文件中，存储了 16 进制的函数地址映射</font>。

在 App 实际执行的二进制文件中，是通过地址来调用方法的。在 App crash 的时候，第三方工具（Fabric、友盟等）会帮我们抓到崩溃的调用栈，调用栈里会包含 crash 地址的调用信息。然后通过 dSYM 文件，我们就可以由地址映射到具体的函数位置。

XCode 中选择 Window -> Organizer 可以看到生成的 archier 文件。

<center>
![10](https://upload-images.jianshu.io/upload_images/5294842-6d216b1dd05b7413.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

然后

*   右键 -\> 在 finder 中显示。
*   右键 -\> 查看包内容。

关于如何用 dsym 文件来分析崩溃位置，查看另一篇博客：[iOS 如何调试第三方统计到的崩溃报告](http://blog.csdn.net/hello_hwc/article/details/50036323)

## 七、应用场景

#### 7.1 \_\_attribute\_\_

或多或少都会在第三方库或者 iOS 的头文件中，见到过 \_\_attribute\_\_。比如

```
__attribute__ ((warn_unused_result)) // 如果没有使用返回值，编译的时候给出警告
```

<font color=#cc0000>\_\_attribtue\_\_ 是一个高级的的编译器指令</font>，它允许开发者指定更多的编译检查和一些高级的<font color=#cc0000>编译期优化</font>。

分为三种：
	
*   函数属性（Function Attribute）
*   类型属性（Variable Attribute）
*   变量属性（Type Attribute）

语法结构

	__attribute__ 语法格式为：__attribute__ ((attribute-list))

放在声明分号 ";" 前面。

比如，在三方库中最常见的，声明一个属性或者方法在当前版本弃用了。

```
@property (nonatomic, strong) CLASSNAME * property __deprecated;
```
	
好处：

给开发者一个过渡的版本，让开发者知道这个属性被弃用了，应当使用最新的 API，但是被 \_\_deprecated 的属性仍然可以正常使用。如果直接弃用，会导致开发者在更新 Pod 的时候，代码无法运行了。

\_\_attribtue\_\_ 的使用场景很多，本文只列举 iOS 开发中常用的几个：

```
// 弃用 API，用作 API 更新
#define __deprecated	__attribute__((deprecated)) 

// 带描述信息的弃用
#define __deprecated_msg(_msg) __attribute__((deprecated(_msg)))

// 遇到 __unavailable 的变量/方法，编译器直接抛出 Error
#define __unavailable	__attribute__((unavailable))

// 告诉编译器，即使这个变量/方法没被使用，也不要抛出警告
#define __unused	__attribute__((unused))

// 和 __unused 相反
#define __used		__attribute__((used))

// 如果不使用方法的返回值，进行警告
#define __result_use_check __attribute__((__warn_unused_result__))

// OC 方法在 Swift 中不可用
#define __swift_unavailable(_msg)	__attribute__((__availability__(swift, unavailable, message=_msg)))
```

#### 7.2 Clang 警告处理

你一定还见过如下代码：

```
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wundeclared-selector"
/// 代码
#pragma clang diagnostic pop
```

这段代码的作用是

1.  <font color=#cc0000>对当前编译环境进行压栈</font>
2.  忽略 -Wundeclared-selector（未声明的）Selector 警告
3.  编译代码
4.  对编译环境进行出栈

通过 clang diagnostic push/pop 可以灵活的控制代码块的编译选项。

在另一篇文章：[iOS 合理利用 Clang 警告来提高代码质量](http://blog.csdn.net/Hello_Hwc/article/details/46425503)，详细的介绍了 XCode 的警告相关内容。


#### 7.3 预处理

所谓预处理，就是在编译之前的处理。<font color=#cc0000>预处理能够让你定义编译器变量，实现条件编译</font>。

比如，这样的代码很常见

```
#ifdef DEBUG
//...
#else
//...
#endif
```

我们同样也可以定义其他预处理变量，在 XCode -> 选中 Target -> build settings 中，搜索 preprocessor。可以分别为 Debug 和 Release 两种模式设置预处理宏。

<center>
![11](https://upload-images.jianshu.io/upload_images/5294842-f44b0523ec2d7d28.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

比如加上：TESTMODE = 1，表示在这个宏中的代码运行在测试服务器。

然后，配合多个 Target（右键 Target，选择 Duplicate），单独一个 Target 负责测试服务器。这样就不用每次切换测试服务器都要修改代码了。

<center>
![12](https://upload-images.jianshu.io/upload_images/5294842-9e591436b1f5e436.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

```
#ifdef TESTMODE
// 测试服务器相关的代码
#else
// 生产服务器相关代码
#endif
```

#### 7.4 插入脚本

通常，如果你使用 CocoaPod 来管理三方库，那么你的 Build Phase 是这样子的：

<center>
![13](https://upload-images.jianshu.io/upload_images/5294842-cbf92c80a034e8ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

其中：[CP] 开头的就是 CocoaPod 插入的脚本。

*   Check Pods Manifest.lock，用来检查 cocoapod 管理的三方库是否需要更新
*   Embed Pods Framework，运行脚本来链接三方库的静态/动态库
*   Copy Pods Resources，运行脚本来拷贝三方库的资源文件

而这些配置信息都存储在这个文件（.xcodeproj）里。

<center>
![14](https://upload-images.jianshu.io/upload_images/5294842-204e1892f65add4d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

到这里，CocoaPod 的原理也就大致搞清楚了，<font color=#cc0000>通过修改 xcodeproject，然后配置编译期脚本</font>，来保证三方库能够正确的编译连接。

同样，我们也可以插入自己的脚本来做一些额外的事情。比如，每次进行 archive 的时候，我们都必须手动调整 target 的 build 版本，如果一不小心，就会忘记。这个过程，我们可以通过插入脚本自动化。

```
buildNumber=$(/usr/libexec/PlistBuddy -c "Print CFBundleVersion" "${PROJECT_DIR}/${INFOPLIST_FILE}")
buildNumber=$(($buildNumber + 1))
/usr/libexec/PlistBuddy -c "Set :CFBundleVersion $buildNumber" "${PROJECT_DIR}/${INFOPLIST_FILE}"
```

这段脚本其实很简单，读取当前 plist 的 build 版本号，然后对其 +1，重新写入。

使用起来也很简单：

*   Xcode -> 选中 Target -> 选中 build phase
*   选择添加 Run Script Phase

<center>
![15](https://upload-images.jianshu.io/upload_images/5294842-7390774f5a869394.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

*   然后把这段脚本拷贝进去，并且勾选 Run Script Only When installing，保证只有我们在安装到设备上的时候，才会执行这段脚本。重命名脚本的名字为 Auto Increase build number

<center>
![16](https://upload-images.jianshu.io/upload_images/5294842-1bbe50512c4a785c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

*   然后，拖动这个脚本的到 Link Binary With Libraries 下面。

#### 7.5 脚本编译打包

脚本化编译打包对于 CI（持续集成）来说，十分有用。iOS 开发中，编译打包必备的两个命令是：

```
// 编译成.app
xcodebuild  -workspace $projectName.xcworkspace -scheme $projectName  -configuration $buildConfig clean build SYMROOT=$buildAppToDir
// 打包
xcrun -sdk iphoneos PackageApplication -v $appDir/$projectName.app -o $appDir/$ipaName.ipa

// 通过 info 命令，可以查看到详细的文档
info xcodebuild
```

在本文最后的附录中，提供一个自动打包的脚本。

#### 7.6 提高项目编译速度

通常，当项目很大，源代码和三方库引入很多的时候，我们会发现编译的速度很慢。在了解了 XCode 的编译过程后，我们可以从以下角度来优化编译速度。

1. 查看编译时间

	我们需要一个途径，能够看到编译的时间，这样才能有个对比，知道我们的优化究竟有没有效果。
	
	对于 XCode 8，关闭 XCode，终端输入以下指令

	```
	$ defaults write com.apple.dt.Xcode ShowBuildOperationDuration YES
	```
	
	然后，重启 XCode，再编译，你会在这里看到编译时间。

	<center>
	![17](https://upload-images.jianshu.io/upload_images/5294842-699c09578e290bc1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

2. 代码层面的优化

	*   forward declaration
	
	所谓 forward declaration，就是 @class CLASSNAME，而不是 #import CLASSNAME.h。这样，编译器能大大<font color=#cc0000>提高 #import 的替换速度</font>。
	
	*   对常用的工具类进行打包（Framework/.a）
	
	打包成 Framework 或者静态库，这样编译的时候这部分代码就<font color=#cc0000>不需要重新编译</font>了。
	
	*   常用头文件放到预编译文件里
	
	pch 文件是预编译文件，这里的内容在执行 XCode build 之前就已经被预编译，并且引入到了每一个 .m 文件里。

3. 编译器选项优化

	*   Debug 模式下，不生成 dsym 文件
	
	上文提到了，dysm 文件里存储了调试信息，在 Debug 模式下，我们可以借助 XCode 和 LLDB 进行调试。所以，不需要生成额外的 dsym 文件来降低编译速度。
	
	*   Debug 开启 Build Active Architecture Only
	
	在 XCode -> Build Settings -> Build Active Architecture Only 改为 YES。这样做，可以只编译当前的版本，比如 arm7/arm64 等等，记得只开启 Debug 模式。这个选项在高版本的 XCode 中自动开启了。
	
	*   Debug 模式下，关闭编译器优化

	<center>
	![18](https://upload-images.jianshu.io/upload_images/5294842-69b06dd791e83926.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>


## 八、附录

自动编译打包脚本

```
export LC_ALL=zh_CN.GB2312;
export LANG=zh_CN.GB2312
buildConfig="Release" //这里是build模式
projectName=[find . -name *.xcodeproj | awk -F "[/.]" '{print $(NF-1)}'[
projectDir=[pwd[
wwwIPADir=~/Desktop/$projectName-IPA
isWorkSpace=true
echo "~~~~~~~~~~~~~~~~~~~开始编译~~~~~~~~~~~~~~~~~~~"
if [ -d "$wwwIPADir" ]; then
echo $wwwIPADir
echo "文件目录存在"
else
echo "文件目录不存在"
mkdir -pv $wwwIPADir
echo "创建${wwwIPADir}目录成功"
fi
cd $projectDir
rm -rf ./build
buildAppToDir=$projectDir/build
infoPlist="$projectName/Info.plist"
bundleVersion=`/usr/libexec/PlistBuddy -c "Print CFBundleShortVersionString" $infoPlist`
bundleIdentifier=`/usr/libexec/PlistBuddy -c "Print CFBundleIdentifier" $infoPlist`
bundleBuildVersion=`/usr/libexec/PlistBuddy -c "Print CFBundleVersion" $infoPlist`

if $isWorkSpace ; then  #是否用CocoaPod
echo  "开始编译workspace...."
xcodebuild  -workspace $projectName.xcworkspace -scheme $projectName  -configuration $buildConfig clean build SYMROOT=$buildAppToDir
else
echo  "开始编译target...."
xcodebuild  -target  $projectName  -configuration $buildConfig clean build SYMROOT=$buildAppToDir
fi

if test $? -eq 0
then
echo "~~~~~~~~~~~~~~~~~~~编译成功~~~~~~~~~~~~~~~~~~~"
else
echo "~~~~~~~~~~~~~~~~~~~编译失败~~~~~~~~~~~~~~~~~~~"
exit 1
fi

ipaName=[echo $projectName | tr "[:upper:]" "[:lower:]"[ #将项目名转小写
findFolderName=[find . -name "$buildConfig-*" -type d |xargs basename[ #查找目录
appDir=$buildAppToDir/$findFolderName/  #app所在路径
echo "开始打包$projectName.app成$projectName.ipa....."
xcrun -sdk iphoneos PackageApplication -v $appDir/$projectName.app -o $appDir/$ipaName.ipa

if [ -f "$appDir/$ipaName.ipa" ]
then
echo "打包$ipaName.ipa成功."
else
echo "打包$ipaName.ipa失败."
exit 1
fi

path=$wwwIPADir/$projectName$(date +%Y%m%d%H%M%S).ipa
cp -f -p $appDir/$ipaName.ipa $path   #拷贝ipa文件
echo "复制$ipaName.ipa到${wwwIPADir}成功"
echo "~~~~~~~~~~~~~~~~~~~结束编译，处理成功~~~~~~~~~~~~~~~~~~~"
```

## 九、文章

[iOS编译过程的原理和应用](https://blog.csdn.net/Hello_Hwc/article/details/53557308)