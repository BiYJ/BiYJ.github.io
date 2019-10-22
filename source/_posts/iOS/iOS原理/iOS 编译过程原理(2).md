---
title: iOS 编译过程原理(2)
categories: iOS原理
---

## 一、前言

《iOS编译过程的原理和应用》文章介绍了 iOS 编译相关基础知识和简单应用，但也很有多问题都没有解释清楚：

*   Clang 和 LLVM 究竟是什么
*   源文件到机器码的细节
*   Linker 做了哪些工作
*   编译顺序如何确定
*   头文件是什么？XCode 是如何找到头文件的？
*   Clang Module
*   签名是什么？为什么要签名

为了搞清楚这些问题，我们来挖掘下 XCode 编译 iOS 应用的细节。

## 二、编译器

> 把一种编程语言（原始语言）转换为另一种编程语言（目标语言）的程序叫做[编译器](https://en.wikipedia.org/wiki/Compiler)。

大多数编译器由两部分组成：<font color=#cc0000>前端和后端</font>。

*   前端负责词法分析、语法分析、生成中间代码；
*   后端以中间代码作为输入，进行与架构无关的代码优化，接着针对不同架构生成不同的机器码。

前后端依赖统一格式的<font color=#cc0000>中间代码（IR）</font>，使得前后端可以独立的变化。新增一门语言只需要修改前端，而新增一个 CPU 架构只需要修改后端即可。

Objective-C/C/C++ 使用的编译器前端是[clang](https://clang.llvm.org/docs/index.html)，swift 是 [swift](https://swift.org/compiler-stdlib/#compiler-architecture)，后端都是 [LLVM](https://llvm.org/)。

<center>
![19](https://upload-images.jianshu.io/upload_images/5294842-dd65ea5de43d8fc1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

## 三、LLVM

LLVM（Low Level Virtual Machine）是一个强大的编译器开发工具套件，听起来像是虚拟机，但实际上 LLVM 和传统意义的虚拟机关系不大，只不过项目最初的名字是 LLVM 罢了。

LLVM 的核心库提供了现代化的 source-target-independent [优化器](https://llvm.org/docs/Passes.html)和支持诸多流行 CPU 架构的代码生成器，这些核心代码是围绕着 LLVM IR（中间代码）建立的。

基于 LLVM 又衍生出了一些强大的子项目，其中 iOS 开发者耳熟能详的是：Clang 和 LLDB。

## 四、clang

clang 是 C 语言家族的编译器前端，诞生之初是为了替代 GCC，提供更快的编译速度。一张图了解 clang 编译的大致流程：

<center>
![20](https://upload-images.jianshu.io/upload_images/5294842-867e93a43ad184f9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

接下来，从代码层面看一下具体的转化过程，新建一个 main.c：

```
#include <stdio.h>
// 注释
#define DEBUG 1
int main() {
#ifdef DEBUG
  printf("hello debug\n");
#else
  printf("hello world\n");
#endif
  return 0;
}
```

## 五、预处理（preprocessor）

预处理会进行头文件引入、宏替换、注释处理、条件编译（#ifdef）等操作。

\#include "stdio.h" 就是告诉预处理器将这一行替换成头文件 stdio.h 中的内容，这个过程是递归的：因为 stdio.h 也有可能包含其他头文件。

用 clang 查看预处理的结果：

```
$ xcrun clang -E main.c
```

预处理后的文件有很多行，在文件的末尾，可以找到 main 函数。

```
$ xcrun clang -E main.c

...

...

extern int \_\_vsnprintf\_chk (char * restrict, size\_t, int, size\_t,
       const char * restrict, va_list);
\# 412 "/usr/include/stdio.h" 2 3 4
\# 10 "main.c" 2




int main() {

    printf("hello debug\\n");



    return 0;
}
```

可以看到，在预处理的时候，注释被删除，条件编译被处理。

## 六、词法分析（lexical anaysis）

词法分析器读入源文件的<font color=#cc0000>字符流</font>，将它们组织成<font color=#cc0000>有意义的词素（lexeme）序列</font>，对于每个词素，词法分析器产生词法单元（token）作为输出。

```
$ xcrun clang -fmodules -fsyntax-only -Xclang -dump-tokens main.c
```

输出：

```
$ xcrun clang -fmodules -fsyntax-only -Xclang -dump-tokens main.c
annot\_module\_include '#include <stdio.h>

// 一点注释

#define DEBUG 1
int main() {
#ifdef DEBUG
    printf("hello debug\\n");
#else
    printf'		Loc=<main.c:9:1>
int 'int'	 \[StartOfLine\]	Loc=<main.c:14:1>
identifier 'main'	 \[LeadingSpace\]	Loc=<main.c:14:5>
l_paren '('		Loc=<main.c:14:9>
r_paren ')'		Loc=<main.c:14:10>
l_brace '{'	 \[LeadingSpace\]	Loc=<main.c:14:12>
identifier 'printf'	 \[StartOfLine\] \[LeadingSpace\]	Loc=<main.c:16:5>
l_paren '('		Loc=<main.c:16:11>
string_literal '"hello debug\\n"'		Loc=<main.c:16:12>
r_paren ')'		Loc=<main.c:16:27>
semi ';'		Loc=<main.c:16:28>
return 'return'	 \[StartOfLine\] \[LeadingSpace\]	Loc=<main.c:20:5>
numeric_constant '0'	 \[LeadingSpace\]	Loc=<main.c:20:12>
semi ';'		Loc=<main.c:20:13>
r_brace '}'	 \[StartOfLine\]	Loc=<main.c:21:1>
eof ''		Loc=<main.c:21:2>
```

Loc=\<main.c:9:1\> 标示这个 token 位于源文件 main.c 的第 9 行，从第 1 个字符开始。保存 token 在源文件中的位置是方便后续 clang 分析的时候能够找到出错的原始位置。

## 七、语法分析（semantic analysis）

词法分析的 Token 流会被解析成一颗<font color=#cc0000>抽象语法树（abstract syntax tree - AST）</font>。

```
$ xcrun clang -fsyntax-only -Xclang -ast-dump main.c | open -f
```

main 函数 AST 的结构：

```
[0;1;32mTranslationUnitDecl[0m[0;33m 0x7fd9a18166e8[0m <[0;33m<invalid sloc>[0m> [0;33m<invalid sloc>[0m
[0;34m|-[0m[0;1;32mTypedefDecl[0m[0;33m 0x7fd9a1816c60[0m <[0;33m<invalid sloc>[0m> [0;33m<invalid sloc>[0m implicit[0;1;36m __int128_t[0m [0;32m'__int128'[0m
[0;34m| `-[0m[0;32mBuiltinType[0m[0;33m 0x7fd9a1816980[0m [0;32m'__int128'[0m
[0;34m|-[0m[0;1;32mTypedefDecl[0m[0;33m 0x7fd9a1816cd0[0m <[0;33m<invalid sloc>[0m> [0;33m<invalid sloc>[0m implicit[0;1;36m __uint128_t[0m [0;32m'unsigned __int128'[0m
[0;34m| `-[0m[0;32mBuiltinType[0m[0;33m 0x7fd9a18169a0[0m [0;32m'unsigned __int128'[0m
[0;34m|-[0m[0;1;32mTypedefDecl[0m[0;33m 0x7fd9a1816fa8[0m <[0;33m<invalid sloc>[0m> [0;33m<invalid sloc>[0m implicit[0;1;36m __NSConstantString[0m [0;32m'struct __NSConstantString_tag'[0m
[0;34m| `-[0m[0;32mRecordType[0m[0;33m 0x7fd9a1816db0[0m [0;32m'struct __NSConstantString_tag'[0m
[0;34m|   `-[0m[0;1;32mRecord[0m[0;33m 0x7fd9a1816d28[0m[0;1;36m '__NSConstantString_tag'[0m
[0;34m|-[0m[0;1;32mTypedefDecl[0m[0;33m 0x7fd9a1817040[0m <[0;33m<invalid sloc>[0m> [0;33m<invalid sloc>[0m implicit[0;1;36m __builtin_ms_va_list[0m [0;32m'char *'[0m
[0;34m| `-[0m[0;32mPointerType[0m[0;33m 0x7fd9a1817000[0m [0;32m'char *'[0m
[0;34m|   `-[0m[0;32mBuiltinType[0m[0;33m 0x7fd9a1816780[0m [0;32m'char'[0m
[0;34m|-[0m[0;1;32mTypedefDecl[0m[0;33m 0x7fd9a1817308[0m <[0;33m<invalid sloc>[0m> [0;33m<invalid sloc>[0m implicit referenced[0;1;36m __builtin_va_list[0m [0;32m'struct __va_list_tag [1]'[0m
[0;34m| `-[0m[0;32mConstantArrayType[0m[0;33m 0x7fd9a18172b0[0m [0;32m'struct 

...
```

有了抽象语法树，clang 就可以对这个树进行分析，找出代码中的错误。比如类型不匹配，亦或 Objective-C 中向 target 发送了一个未实现的消息。

<font color=#cc0000>AST 是开发者编写 clang 插件主要交互的数据结构</font>，clang 也提供很多 API 去读取 AST。更多细节：[Introduction to the Clang AST](https://clang.llvm.org/docs/IntroductionToTheClangAST.html)。

## 八、CodeGen

CodeGen 遍历语法树，<font color=#cc0000>生成 LLVM IR 代码</font>。LLVM IR 是前端的输出，后端的输入。

```
xcrun clang -S -emit-llvm main.c -o main.ll
```

main.ll 文件内容：

```
; ModuleID = 'main.c'
source_filename = "main.c"
target datalayout = "e-m:o-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-apple-macosx10.13.0"

@.str = private unnamed_addr constant \[13 x i8\] c"hello debug\\0A\\00", align 1

; Function Attrs: noinline nounwind optnone ssp uwtable
define i32 @main() #0 {
  %1 = alloca i32, align 4
  store i32 0, i32* %1, align 4
  %2 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds (\[13 x i8\], \[13 x i8\]* @.str, i32 0, i32 0))
  ret i32 0
}

...
```

Objective-C 代码在这一步会进行 runtime 的桥接：property 合成、ARC 处理等。

LLVM 会对生成的 IR 进行优化，优化会调用相应的 Pass 进行处理。Pass 由多个节点组成，都是 [Pass](http://llvm.org/doxygen/classllvm_1_1Pass.html) 类的子类，每个节点负责做特定的优化，更多细节：[Writing an LLVM Pass](https://llvm.org/docs/WritingAnLLVMPass.html)。

## 九、生成汇编代码

LLVM 对 IR 进行优化后，会针对不同架构生成不同的目标代码，最后以汇编代码的格式输出。

生成 arm 64 汇编：

```
$ xcrun clang -S main.c -o main.s
```

查看生成的 main.s 文件。对汇编感兴趣的同学可以看看这篇文章：[iOS汇编快速入门](https://github.com/LeoMobileDeveloper/Blogs/blob/master/Basic/iOS%20assembly%20toturial%20part%201.md)。

```
	.section	__TEXT,__text,regular,pure_instructions
	.macosx_version_min 10, 13
	.globl	_main                   ## -- Begin function main
	.p2align	4, 0x90
_main:                                  ## @main
	.cfi_startproc
## %bb.0:
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset %rbp, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register %rbp
	subq	$16, %rsp
	leaq	L_.str(%rip), %rdi
	movl	$0, -4(%rbp)
	movb	$0, %al
	callq	_printf
	xorl	%ecx, %ecx
	movl	%eax, -8(%rbp)          ## 4-byte Spill
	movl	%ecx, %eax
	addq	$16, %rsp
	popq	%rbp
	retq
	.cfi_endproc
                                        ## -- End function
	.section	__TEXT,__cstring,cstring_literals
L_.str:                                 ## @.str
	.asciz	"hello debugn"


.subsections_via_symbols
```

## 十、汇编器

汇编器以汇编代码作为输入，将汇编代码转换为机器代码，最后输出目标文件（object file）。

```
$ xcrun clang -fmodules -c main.c -o main.o
```

还记得代码中调用了一个函数 printf 么？通过 nm 命令，查看下 main.o 中的符号

```
$ xcrun nm -nm main.o
                 (undefined) external _printf
0000000000000000 (\_\_TEXT,\_\_text) external _main
```

\_printf 是一个 undefined external 的。undefined 表示在当前文件暂时找不到符号 \_printf，而 external 表示这个符号是外部可以访问的，对应表示文件私有的符号是 non-external。

> 什么是符号（Symbols）?

符号就是指向一段代码或者数据的名称。还有一种叫做 WeakSymols，也就是并不一定会存在的符号，需要在运行时决定。比如 iOS12 特有的 API，在 iOS11 上就没有。

## 十一、链接

连接器把编译产生的 .o 文件和（dylib、a、tbd）文件，生成一个 mach-o 文件。

```
$ xcrun clang main.o -o main
```

就得到了一个 mach o 格式的可执行文件

```
$ file main
main: Mach-O 64-bit executable x86_64
$ ./main
hello debug
```

再用 nm 命令，查看可执行文件的符号表：

```
$ nm -nm main
                 (undefined) external _printf (from libSystem)
                 (undefined) external dyld\_stub\_binder (from libSystem)
0000000100000000 (\_\_TEXT,\_\_text) \[referenced dynamically\] external \_\_mh\_execute_header
0000000100000f60 (\_\_TEXT,\_\_text) external _main
```

\_printf 仍然是 undefined，但是后面多了一些信息：from libSystem，表示这个符号来自于 libSystem，会在运行时动态绑定。

## 十二、XCode 编译

通过上文我们大概了解了 Clang 编译一个 C 语言文件的过程，但是 XCode 开发的项目不仅仅包含了代码文件，还包括了图片、plist 等。XCode 中编译一次都要经过哪些过程呢？

新建一个单页面的 Demo 工程：CocoaPods 依赖 AFNetworking 和 SDWebImage，同时依赖于一个内部 Framework。按下Command \+ B，在 XCode 的 Report Navigator 模块中，可以找到编译的详细日志：

<center>
![21](https://upload-images.jianshu.io/upload_images/5294842-850e886824831a69.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

详细的步骤：

*   创建 Product.app 的文件夹
*   把 Entitlements.plist 写入到 DerivedData 里，处理打包的时候需要的信息（比如 application-identifier）。
*   创建一些辅助文件，比如各种 .hmap，这是 headermap 文件，具体作用下文会讲解。
*   执行 CocoaPods 的编译前脚本：检查 Manifest.lock 文件。
*   编译 .m 文件，生成 .o 文件。
*   链接动态库。.o 文件，生成一个 mach o 格式的可执行文件。
*   编译 assets，编译 storyboard，链接 storyboard
*   拷贝动态库 Logger.framework，并且对其签名
*   执行 CocoaPods 编译后脚本：拷贝 CocoaPods Target 生成的 Framework
*   对 Demo.App 签名，并验证（validate）
*   生成 Product.app
*   生成 dYSM 文件

> Entitlements.plist 保存了 App 需要使用的特殊权限，比如 iCloud、远程通知、Siri 等。

## 十三、编译顺序

编译的时候有很多的 Task（任务）要去执行，XCode 如何决定 Task 的执行顺序呢？

答案是：<font color=#cc0000>依赖关系</font>。

还是以刚刚的 Demo 项目为例，整个依赖关系如下：

<center>
![22](https://upload-images.jianshu.io/upload_images/5294842-9b0c7342fdc5b545.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

可以从 XCode 的 Report Navigator 看到 Target 的编译顺序：

<center>
![23](https://upload-images.jianshu.io/upload_images/5294842-1c02d00b99428ab4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>
  
XCode 编译的时候会尽可能的利用多核性能，多 Target 并发编译。

那么，XCode 又从哪里得到了这些依赖关系呢？

*   Target Dependencies - 显式声明的依赖关系
*   Linked Frameworks and Libraries - 隐式声明的依赖关系
*   Build Phase - 定义了编译一个 Target 的每一步

## 十四、增量编译

日常开发中，一次完整的编译可能要几分钟，甚至几十分钟，而增量编译只需要不到 1 分钟，为什么增量编译会这么快呢？

因为 XCode 会对每一个 Task 生成一个哈希值，只有哈希值改变的时候才会重新编译。

比如，修改了 ViewControler.m，只有图中灰色的三个 Task 会重新执行（这里不考虑 build phase 脚本）。

<center>
![24](https://upload-images.jianshu.io/upload_images/5294842-0eaaea9ee243f79c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

## 十五、头文件

C 语言家族中，头文件（.h）文件用来引入函数/类/宏定义等声明，让开发者更灵活的组织代码，而不必把所有的代码写到一个文件里。

头文件对于编译器来说就是一个 promise。头文件里的声明，编译会认为有对应实现，在链接的时候再解决具体实现的位置。

<center>
![25](https://upload-images.jianshu.io/upload_images/5294842-b5be48b5d1a1c97d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

当只有声明，没有实现的时候，链接器就会报错。

```
Undefined symbols for architecture arm64:
“_umimplementMethod”, referenced from:
-\[ClassA method\] in ClassA.o
ld: symbol(s) not found for architecture arm64
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

Objective-C 的方法要到运行时才会报错，因为 Objective-C 是一门动态语言，编译器无法确定对应的方法名（SEL）在运行时到底有没有实现（IMP）。

日常开发中，两种常见的头文件引入方式：

```
#include "CustomClass.h" // 自定义
#include <Foundation/Foundation.h> // 系统或者内部 framework
```

引入的时候并没有指明文件的具体路径，编译器是如何找到这些头文件的呢？

回到 XCode 的 Report Navigator，找到上一个编译记录，可以看到编译 ViewController.m 的具体日志：

<center>
![27](https://upload-images.jianshu.io/upload_images/5294842-d620556105843b4f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![26](https://upload-images.jianshu.io/upload_images/5294842-d1cd2dd2e5bb8a0e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

把这个日志整体拷贝到命令行中，然后最后加上 -v，表示我们希望得到更多的日志信息，执行这段代码，在日志最后可以看到clang 是如何找到头文件的：

```
#include "..." search starts here:
 /Users/.../Build/Intermediates.noindex/Demo.build/Debug-iphoneos/Demo.build/Demo-generated-files.hmap (headermap)
 /Users/.../Build/Intermediates.noindex/Demo.build/Debug-iphoneos/Demo.build/Demo-project-headers.hmap (headermap)
 /Users/.../Build/Products/Debug-iphoneos/AFNetworking/AFNetworking.framework/Headers
 /Users/.../Build/Products/Debug-iphoneos/SDWebImage/SDWebImage.framework/Headers
 
#include <...> search starts here:
 /Users/.../Build/Intermediates.noindex/Demo.build/Debug-iphoneos/Demo.build/Demo-own-target-headers.hmap (headermap)
 /Users/.../Build/Intermediates.noindex/Demo.build/Debug-iphoneos/Demo.build/Demo-all-non-framework-target-headers.hmap (headermap)
 /Users/.../Build/Intermediates.noindex/Demo.build/Debug-iphoneos/Demo.build/DerivedSources
 /Users/.../Build/Products/Debug-iphoneos (framework directory)
 /Users/.../Build/Products/Debug-iphoneos/AFNetworking (framework directory)
 /Users/.../Build/Products/Debug-iphoneos/SDWebImage (framework directory)
 /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/clang/10.0.0/include
 /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include
 $SDKROOT/usr/include
 $SDKROOT/System/Library/Frameworks (framework directory)
 
End of search list.
```

这里有个文件类型叫做 heademap，headermap 是帮助编译器找到头文件的辅助文件：存储着头文件到其物理路径的映射关系。

可以通过一个辅助的小工具 [hmap](https://github.com/milend/hmap) 查看 hmap 中的内容：

```
$ ./hmap print Demo-project-headers.hmap 
AppDelegate.h -> /Users/huangwenchen/Desktop/Demo/Demo/AppDelegate.h
Demo-Bridging-Header.h -> /Users/huangwenchen/Desktop/Demo/Demo/Demo-Bridging-Header.h
Dummy.h -> /Users/huangwenchen/Desktop/Demo/Framework/Dummy.h
Framework.h -> Framework/Framework.h
TestView.h -> /Users/huangwenchen/Desktop/Demo/Demo/View/TestView.h
ViewController.h -> /Users/huangwenchen/Desktop/Demo/Demo/ViewController.h
```

> 这就是为什么备份/恢复 Mac 后，需要 clean build folder，因为两台 mac 对应文件的物理位置可能不一样。

clang 发现 #import "TestView.h" 的时候，先在 headermap(Demo-generated-files.hmap,Demo-project-headers.hmap) 里查找，如果 headermap 文件找不到，接着在 own target 的 framework 里找：

```
/Users/.../Build/Products/Debug-iphoneos/AFNetworking/AFNetworking.framework/Headers/TestView.h
/Users/.../Build/Products/Debug-iphoneos/SDWebImage/SDWebImage.framework/Headers/TestView.h
```

系统的头文件查找的时候也是优先 headermap，headermap 查找不到会查找 own target framework，最后查找 SDK 目录。

以 #import <Foundation/Foundation.h> 为例，在 SDK 目录查找时：

1. 首先查找 framework 是否存在

	```
	$SDKROOT/System/Library/Frameworks/Foundation.framework
	```

2. 如果 framework 存在，再在 headers 目录里查找头文件是否存在

	```
	$SDKROOT/System/Library/Frameworks/Foundation.framework/headers/Foundation.h
	```

## 十六、Clang Module

传统的 #include/#import 都是文本语义：预处理器在处理的时候会把这一行替换成对应头文件的文本，这种简单粗暴替换是有很多问题的：

1.  大量的预处理消耗。假如有 N 个头文件，每个头文件又 #include 了 M 个头文件，那么整个预处理的消耗是 N*M。
2.  文件导入后，宏定义容易出现问题。因为是文本导入，并且按照 include 依次替换，当一个头文件定义了 #define std hello\_world，而另一个头文件刚好又是 C++ 标准库，那么 include 顺序不同，可能会导致所有的 std 都会被替换。
3.  边界不明显。拿到一组 .a 和 .h 文件，很难确定 .h 是属于哪个 .a 的，需要以什么样的顺序导入才能正确编译。

[clang module](https://clang.llvm.org/docs/Modules.html) 不再使用文本模型，而是采用更高效的语义模型。clang module 提供了一种新的导入方式：@import，module 会被作为一个独立的模块编译，并且产生独立的缓存，从而大幅度提高预处理效率，这样时间消耗从 M*N 变成了 M+N。

XCode 创建的 Target 是 Framework 的时候，默认 define module 会设置为 YES，从而支持 module，当然像 Foundation 等系统的 framwork 同样支持 module。

\#import \<Foundation/NSString.h\> 的时候，编译器会检查 NSString.h 是否在一个 module 里，如果是的话，这一行会被替换成 @import Foundation。

<center>
![28](https://upload-images.jianshu.io/upload_images/5294842-f9261463ed11b9e9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

那么，如何定义一个 module 呢？答案是：modulemap 文件，这个文件描述了一组头文件如何转换为一个 module，举个例子：

```
framework module Foundation  [extern_c] [system] {
	umbrella header "Foundation.h" // 所有要暴露的头文件
 	export *
	module * {
 		export *
 	}
 	explicit module NSDebug { //submodule
 		header "NSDebug.h"
 		export *
 	}
 }
```

swift 是可以直接 import 一个 clang module 的，比如你有一些 C 库，需要在 Swift 中使用，就可以用 modulemap 的方式。

## 十七、Swift 编译

现代化的语言几乎都抛弃了头文件，swift 也不例外。问题来了，swift 没有头文件又是怎么找到声明的呢？

> 编译器干了这些脏活累活。编译一个 Swift 头文件，需要解析 module 中所有的 Swift 文件，找到对应的声明。

<center>
![29](https://upload-images.jianshu.io/upload_images/5294842-bd4a0844c9c85adb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

当开发中难免要有 Objective-C 和 Swift 相互调用的场景，两种语言在编译的时候查找符号的方式不同，如何一起工作的呢？

1. Swift 引用 Objective-C

	Swift 的编译器内部使用了 clang，所以 swift 可以直接使用 clang module，从而支持直接 import Objective-C 编写的framework。

	<center>
	![30](https://upload-images.jianshu.io/upload_images/5294842-cd4f5c7f8eafc71e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>
  
	swift 编译器会从 Objective-C 头文件里查找符号，头文件的来源分为两大类：

	*   Bridging-Header.h 中暴露给 swfit 的头文件
	*   framework 中公开的头文件，根据编写的语言不同，可能从 modulemap 或者 umbrella header 查找。

	XCode 提供了宏定义 NS\_SWIFT\_NAME 来让开发者定义 Objective-C => Swift的符号映射，可以通过 Related Items -> Generate Interface 来查看转换后的结果：
	
	<center>
	![31](https://upload-images.jianshu.io/upload_images/5294842-bd928c0ee2b8d45a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

2. Objective-C 引用 swift

	xcode 会以 module 为单位，为 swift 自动生成头文件，供 Objective-C 引用，通常这个文件命名为 ProductName-Swift.h。
	
	swift 提供了关键词 @objc 来把类型暴露给 Objective-C 和 Objective-C Runtime。
	
	```
	@objc public class MyClass
	```

## 十八、深入理解 Linker

> 链接器会把编译器编译生成的多个文件，链接成一个可执行文件。链接并不会产生新的代码，只是在现有代码的基础上做移动和补丁。

链接器的输入可能是以下几种文件：

*   object file(.o)，单个源文件的编辑结果，包含了由符号表示的代码和数据。
*   动态库（.dylib），mach o 类型的可执行文件，链接的时候只会绑定符号，动态库会被拷贝到 app 里，运行时加载
*   静态库（.a），由 ar 命令打包的一组 .o 文件，链接的时候会把具体的代码拷贝到最后的 mach-o。
*   tbd，只包含符号的库文件

这里提到了一个概念：符号（Symbols），那么符号是什么呢？

> 符号是一段代码或者数据的名称，一个符号内部也有可能引用另一个符号。

以一段代码为例，看看链接时究竟发生了什么？

源代码：

```
- (void)log
{
    printf("hello world\n");
}
```

.o 文件：

```
#代码
adrp    x0, l_.str@PAGE
add     x0, x0, l_.str@PAGEOFF
bl      _printf

#字符串符号
l_.str:                                 ; @.str
        .asciz  "hello world\\n"
```

在 .o 文件中，字符串 "hello world\\n" 作为一个符号（l\_.str）被引用，汇编代码读取的时候按照 l\_.str 所在的页加上偏移量的方式读取，然后调用 printf 符号。到这一步，CPU 还不知道怎么执行，因为还有两个问题没解决：

1.  l\_.str 在可执行文件的哪个位置？
2.  printf 函数来自哪里？

再来看看链接之后的 mach o 文件：

<center>
![32](https://upload-images.jianshu.io/upload_images/5294842-ac9f852d87c4ebb1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

链接器如何解决这两个问题呢？

1.  链接后，不再是以页+偏移量的方式读取字符串，而是直接读虚拟内存中的地址，解决了 l\_.str 的位置问题。
2.  链接后，不再是调用符号 \_printf，而是在 DATA 段上创建了一个函数指针 \_printf$ptr，初始值为 0x0(null)，代码直接调用这个函数指针。启动的时候，dyld 会把 DATA 段上的指针进行动态绑定，绑定到具体虚拟内存中的 \_printf 地址。更多细节，可以参考这篇文章：[深入理解iOS App的启动过程](https://blog.csdn.net/Hello_Hwc/article/details/78317863)。

Mach-O 有一个区域叫做 LINKEDIT，这个区域用来存储启动时 dyld 需要动态修复的一些数据：比如刚刚提到的 printf 在内存中的地址。

## 十九、理解签名

1. 非对称加密

	在密码学中，非对称加密需要两个密钥：公钥和私钥。私钥加密的只能用公钥解密，公钥加密的只能用私钥解密。

2. 数字签名

	数字签名表示我对数据做了个标记，表示这是我的数据，没有经过篡改。
	
	数据发送方 Leo 产生一对公私钥，私钥自己保存，公钥发给接收方 Lina。Leo 用摘要算法，对发送的数据生成一段摘要，摘要算法保证了只要数据修改，那么摘要一定改变。然后用私钥对这个摘要进行加密，和数据一起发送给 Lina。
	
	<center>
	![33](https://upload-images.jianshu.io/upload_images/5294842-b929196750ad26b6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>
	
	Lina 收到数据后，用公钥解密签名，得到 Leo 发过来的摘要；然后自己按照同样的摘要算法计算摘要，如果计算的结果和 Leo 的一样，说明数据没有被篡改过。
	
	<center>
	![34](https://upload-images.jianshu.io/upload_images/5294842-cacead84adbed4a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>
	
	但是，现在还有个问题：Lina 有一个公钥，假如攻击者把 Lina 的公钥替换成自己的公钥，那么攻击者就可以伪装成 Leo 进行通信，所以 Lina 需要确保这个公钥来自于 Leo，可以通过数字证书来解决这个问题。
	
	> 数字证书由 CA（Certificate Authority）颁发，以 Leo 的证书为例，里面包含了以下数据：签发者、Leo 的公钥、Leo 使用的 Hash 算法、证书的数字签名、到期时间等。
	
	有了数字证书后，Leo 再发送数据的时候，把自己从 CA 申请的证书一起发送给 Lina。Lina 收到数据后，先用 CA 的公钥验证证书的数字签名是否正确，如果正确说明证书没有被篡改过，然后以信任链的方式判断是否信任这个证书，如果信任证书，取出证书中的数据，可以判断出证书是属于 Leo 的，最后从证书中取出公钥来做数据签名验证。

## 二十、iOS App 签名

为什么要对 App 进行签名呢？签名能够让 iOS 识别出是谁签名了 App，并且签名后 App 没有被篡改过。

除此之外，Apple 要严格控制 App 的分发：

1.  App 来自 Apple 信任的开发者
2.  安装的设备是 Apple 允许的设备


#### 20.1 证书

通过上文的讲解，我们知道数字证书里包含着申请证书设备的公钥，所以在 Apple 开发者后台创建证书的时候，需要上传 CSR 文件（Certificate Signing Request），用 keychain 生成这个文件的时候，就生成了一对公/私钥：公钥在 CSR 里，私钥在本地的 Mac 上。Apple 本身也有一对公钥和私钥：私钥保存在 Apple 后台，公钥在每一台 iOS 设备上。

<center>
![35](https://upload-images.jianshu.io/upload_images/5294842-ec7d73889dc3f8e4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

#### 20.2 Provisioning Profile

iOS App 安装到设备的途径（非越狱）有以下几种：

*   开发包（插线，或者 archive 导出 develop 包）
*   Ad Hoc
*   App Store
*   企业证书

开发包和 Ad Hoc 都会严格限制安装设备，为了把设备 uuid 等信息一起打包进 App，开发者需要配置 Provisioning Profile。

<center>
![36](https://upload-images.jianshu.io/upload_images/5294842-696d4d9bbf81ec36.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

可以通过以下命令来查看 Provisioning Profile 中的内容：

```
security cms -D -i embedded.mobileprovision > result.plist
open result.plist
```

本质上就是一个编码过后的 plist。

<center>
![37](https://upload-images.jianshu.io/upload_images/5294842-46a1386a0e86d8aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

#### 20.3 iOS 签名

生成安装包的最后一步，XCode 会调用 codesign 对 Product.app 进行签名。

创建一个额外的目录 \_CodeSignature 以 plist 的方式存放安装包内每一个文件签名

```
<key>Base.lproj/LaunchScreen.storyboardc/01J-lp-oVM-view-Ze5-6b-2t3.nib</key>
<data>
T2g5jlq7EVFHNzL/ip3fSoXKoOI=
</data>
<key>Info.plist</key>
<data>
5aVg/3m4y30m+GSB8LkZNNU3mug=
</data>
<key>PkgInfo</key>
<data>
n57qDP4tZfLD1rCS43W0B4LQjzE=
</data>
<key>embedded.mobileprovision</key>
<data>
tm/I1g+0u2Cx9qrPJeC0zgyuVUE=
</data>
...
```

代码签名会直接写入到 mach-o 的可执行文件里，值得注意的是签名是以页（Page）为单位的，而不是整个文件签名：

<center>
![38](https://upload-images.jianshu.io/upload_images/5294842-8c96d1f4a8fcb7b1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

#### 20.4 验证

安装 App 的时候

*   从 embedded.mobileprovision 取出证书，验证证书是否来自 Apple 信任的开发者
*   证书验证通过后，从证书中取出 Leo 的公钥
*   读取 \_CodeSignature 中的签名结果，用 Leo 的公钥验证每个文件的签名是否正确
*   文件 embedded.mobileprovision 验证通过后，读取里面的设备 id 列表，判断当前设备是否可安装（App Store 和企业证书不做这步验证）
*   验证通过后，安装 App

启动 App 的时候

*   验证 bundle id、entitlements 和 embedded.mobileprovision中的 AppId，entitlements 是否一致
*   判断 device id 包含在 embedded.mobileprovision 里。App Store 和企业证书不做验证
*   如果是企业证书，验证用户是否信任企业证书
*   App 启动后，当缺页中断（page fault）发生的时候，系统会把对应的 mach-o 页读入物理内存，然后验证这个 page 的签名是否正确。
*   以上都验证通过，App 才能正常启动

## 二十一、文章
[黄文臣](https://me.csdn.net/Hello_Hwc) & [深入浅出iOS编译](https://blog.csdn.net/Hello_Hwc/article/details/85226147)