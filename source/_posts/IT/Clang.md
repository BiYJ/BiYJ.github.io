---
title: clang
categories: IT
---

## 一、编译器

> 为什么需要编译？

计算机 CPU 只能读懂机器码（machine code，也就是由一堆 0 和 1 组成的编码），但程序员现在编写的代码并不是机器码，而是高级编程语言（Objective-C、Swift、Java、...），最终也可以被计算机所执行，这就需要编译了。在编译的过程中，编译器的作用便是把我们的高级编程语言通过一系列的操作转化成可被计算机执行的机器语言。

> 编译器是如何设计的？

经典的三段式设计（three phase design）:前端(Frontend)->优化器(Optimizer)->后端(Backend)

<center>
![](http://dzliving.com/Clang_0.png)
</center>

1. 前端负责分析源代码，可以检查语法级错误，并构建针对该语言的抽象语法树（AST）
2. 抽象语法树可以进一步转换为优化，最终转为新的表示方式，然后再交给让优化器和后端处理
3. 最终由后端生成可执行的机器码

> 为什么要使用三段式设计？优势在哪？

首先解决了一个很大的问题：假如有 N 种语言（C、OC、C++、Swift...）的前端，同时也有 M 个架构（模拟器、arm64、x86...）的 Target，是否就需要 N × M 个编译器？

三段式架构的价值就体现出来了，通过共享优化器的中转，很好的解决了这个问题。

假如你需要增加一种语言，只需要增加一种前端；假如你需要增加一种处理器架构，也只需要增加一种后端，而其他的地方都不需要改动。

<center>
![](http://dzliving.com/Clang_1.png)
</center>

> 编译源文件有哪些主要步骤？

先列举一些整个编译过程的主要步骤，后面再详细介绍每个步骤都做了哪些事情。

主要编译步骤如下：

<center>
源代码（source code）

↓ 
	
预处理器（preprocessor）

↓ 

编译器（compiler）

↓ 

汇编程序（assembler）

↓ 

目标代码（object code）

↓ 

链接器（Linker）

↓ 

可执行文件（executables）

</center>

## 二、Xcode 编译器发展简史

> Xcode 3 以前：GCC；<br>
> Xcode 3：增加 LLVM，GCC(前端) + LLVM(后端)；<br>
> Xcode 4.2：出现 Clang - LLVM 3.0 成为默认编译器；<br>
> Xcode 4.6： LLVM 升级到 4.2 版本；<br>
> Xcode 5： GCC 被废弃，新的编译器是 LLVM 5.0

为什么苹果的 Xcode 会使用 Clang+LLVM 取代 GCC？

GCC 是第三方开源的，不属于苹果维护也不能完全掌控其开发进程，Apple 为 Objective-C 增加许多新特性，但 GCC 开发者对这些支持却不友好；Apple 需要做模块化，GCC 开发者却拖着迟迟不实现。

随着 Apple 对其 IDE(也就是 Xcode)性能的要求越来越高，最终还是从零开发了一个 Clang 前端加 LLVM 后端的编译器，这个编译器的作者是大名鼎鼎的 Swift 之父 Chris Lattner。

> Clang 比 GCC 优秀在哪些方面？

1. 传说新的 Clang 编译器编译 Objective-C 代码速度比 GCC 快 3 倍
2. 提供了友好的代码提示


## 三、Clang 的简介


> Clang: a C language family frontend for LLVM。
> LLVM 的 C 语言家族（C、C++、OC）前端。

上面是官网对于 Clang的一句话介绍，其实 Clang 就是上文所提到的编译器前端。

用途：输出代码对应的抽象语法树（Abstract Syntax Tree, AST），并将代码编译成 LLVM Bitcode。接着在后端（back-end）使用 LLVM 编译成平台相关的机器语言。

## 四、Clang 的编译过程

#### 4.1 预处理

预处理顾名思义是预先处理。预处理的内容如下：

1. import 头文件替换

	在面向对象编程的思维下，写代码会经常用到其他类的属性或方法等，只需要导入头文件就可以用了，如：

	```
	#import <Foundation/Foundation.h> 
	// 这里将会在预处理时把 Foundation.h 文件的内容拷贝过来并替换
	```
	
	如果 A.h 文件引用 B.h，并且 B.h 也引用了 A.h，就会导致了循环引入。
	
	解决办法是在头文件中使用 `@class A;` 代替 `#import "A.h"`。
	
	这个意思是声明 A 是一个类，这样就可以使用 A 做类名，如果需要使用 A 的方法属性等，可以在 .m 实现文件中通过 `#import A.h` 的方式使用，这种方法不但可以解决互相引入的问题还可以优化编译速度。

2. macro 宏展开

	* 无参宏。如：

		```
		#define COUNT  3
		```
	

	* 带参宏。如：

		```
		#define SUM(a, b)  a + b
		```

	在宏定义的作用域内，输入了 COUNT、SUM()，在预处理过程中都会被替换。

3. 处理其他的预编译指令

	条件编译语句也是在预处理阶段完成，并且条件编译只允许编译源程序中满足条件的程序段，使生成的目标程序较短，从而减少了内存的开销并提高了程序的效率。如以下代码就只会保留一个 return 语句：

	```
	#if DEBUG        
	     return YES;
	#else
	     return NO;
	#endif
	```

简单来说，<font color=#cc0000>`“#”`</font> 这个符号是编译器预处理的标志。

<center>

|预处理指令|说明|
|:------|:-----|
|#undef|取消已定义的宏|
|#if|如果给定条件为真，则编译以下代码|
|#ifdef|如果宏已经定义，则编译以下代码|
|#ifndef|如果宏没有定义，则编译以下代码|
|#elif|如果前面的 #if 的条件不为真，当前条件为真，则编译以下代码|
|#endif|结束一个 #if……#else 条件编译块|

</center>


#### 4.2 Lexical Analysis - 词法分析（输出 token 流）

词法分析其实是编译器开始工作真正意义上的第一个步骤，其所做的工作主要为<font color=#cc0000>将输入的代码转换为一系列符合特定语言的词法单元</font>，这些词法单元类型包括了关键字、操作符、变量等等。

举个例子:

> Objective-C 语言包含了关键字 if、else、new 等，那么在词法分析步骤时，遇到 i与f 或 n与e与w 组合在一起的时候，需要将这几个字母组合为关键字 if 或 new 等词法单元。

词法分析，只需要将源代码以字符文本的形式转化成 Token 流的形式，不涉及校验语义，不需要递归，是线性的。

> 什么是 token 流？

就是有“类型”，有“值”的一些小单元。

举个例子：

> 一个运算表达式：(28 + 78) * 2 
> 
> 只需要解析出 <font color=#cc0000>(</font> 是一个开括号，<font color=#cc0000>28</font> 是数字整形，<font color=#cc0000>+</font> 是一个运算符号即可。

编译指令： `$clang -fmodules -fsyntax-only -Xclang -dump-tokens main.m`

<center>
![](http://dzliving.com/Clang_2.png?imageView2/0/w/400)
![](http://dzliving.com/Clang_3.png)
</center>

里面的每一行都可以说是一个 token 流。一个表达式也会被逐个的解析。

#### 4.3 Semantic Analysis - 语法分析（输出(AST)抽象语法树）

编译指令：`$clang -fmodules -fsyntax-only -Xclang -ast-dump main.m`

![](http://dzliving.com/Clang_4.png)

* 语法分析的最终产物是输出抽象语法树
* 语法分析在 Clang 中由 Parser 和 Sema 两个模块配合完成
* 校验语法是否正确
* 根据当前语言的语法，生成语意节点，并将所有节点组合成抽象语法树（AST）
* 这一步跟源码等价，可以反写出源码
* Static Analysis 静态分析
	* 通过语法树进行代码静态分析，找出非语法性错误
	* 模拟代码执行路径，分析出 control-flow graph(CFG) 【MRC时代会分析出引用计数的错误】
	* 预置了常用 Checker（检查器）

#### 4.4 CodeGen - IR(Intermediate Representation)中间代码生成

CodeGen 负责将语法树从顶至下遍历，翻译成 LLVM IR。

LLVM IR 是 Frontend 的输出，LLVM Backend 的输入，前后端的桥接语言 （Swift也是转成这个）

与 Objective-C Runtime 桥接

* Class/Meta Class/Protocol/Category 内存结构生成，并存放在指定 section 中（如 Class：\_DATA, \_objc\_classrefs）
* Method/lvar/Property 内存结构生成
* 组成 method\_list/ivar\_list/property\_list 并填入 Class
* Non-Fragile ABI：为每个 Ivar 合成 OBJC\_IVAR\_$\_ 偏移值常量
* 存取 Ivar 的语句（ivar = 123; int a = ivar;）转写成 base + OBJC\_IVAR$\_ 的形式
* 将语法树中的 ObjcMessageExpr 翻译成相应版本的 objc\_msgSend，对 super 关键字的调用翻译成 objc\_msgSendSuper
* 根据修饰符 strong/weak/copy/atomic 合成 @property 自动实现的 setter/getter
* 处理 @synthesize
* 生成 block_layout 的数据结构
* 变量的 capture(\_\_block/\_\_weak)
* 生成 \_block\_invoke 函数
* ARC：分析对象引用关系，将 objc\_storeStrong/objc\_storeWeak 等 ARC 代码插入
* 将 ObjCAutoreleasePoolStmt 转译成 objc_autoreleasePoolPush/Pop
* 实现自动调用 [super dealloc]
* 为每个拥有 ivar 的 Class 合成 .cxx_destructor 方法来自动释放类的成员变量，代替 MRC 时代的“self.xxx = nil”

#### 4.5 Optimize - 优化 IR

递归优化成尾递归

#### 4.6 LLVM Bitcode - 生成字节码

#### 4.7 Assemble - 生成 Target 相关汇编

Assemble - 生成Target相关Object(Mach-O)

#### 4.8 Link 生成 Executable


## 五、问题

> clang 编译错误: fatal error: 'UIKit/UIKit.h' file not found

```
fatal error: 'UIKit/UIKit.h' file not found
#import <UIKit/UIKit.h>
        ^~~~~~~~~~~~~~~
1 error generated.
```

**解决 1：**

```
$ clang -rewrite-objc xx.m
```

替换成:

```
$ clang -x objective-c -rewrite-objc -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk xx.m
```

这个命令很繁琐，可以采用 alias 来起一个别名来代替这个命令。

1. 终端键入 $ vim ~/.bash_profile
2. 编辑状态键入 $ alias rewriteoc='clang -x objective-c -rewrite-objc -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk'
3. 键入完毕后, esc 退出编辑状态, 再键入 :wq 退出 vim
4. 键入命令 source ~/.bash_profile

**解决 2：**

```
# 模拟器
$ xcrun -sdk iphonesimulator clang -rewrite-objc main.m

# 真机
$ xcrun -sdk iphoneos clang -rewrite-objc main.m

# 带有版本的真机、模拟器
$  xcrun -sdk iphonesimulator9.3 clang -rewrite-objc main.m
```

查看设备上都装哪些 SDK。

```
$ xcodebuild -showsdks

...

iOS SDKs:
	iOS 12.1                      	-sdk iphoneos12.1

iOS Simulator SDKs:
	Simulator - iOS 12.1          	-sdk iphonesimulator12.1

macOS SDKs:
	macOS 10.14                   	-sdk macosx10.14

tvOS SDKs:
	tvOS 12.1                     	-sdk appletvos12.1

tvOS Simulator SDKs:
	Simulator - tvOS 12.1         	-sdk appletvsimulator12.1

watchOS SDKs:
	watchOS 5.1                   	-sdk watchos5.1

watchOS Simulator SDKs:
	Simulator - watchOS 5.1       	-sdk watchsimulator5.1
```

## 六、文章

[Developer_Yancy](https://www.jianshu.com/u/3f995dac0230) - [iOS底层探索(一) - 从零开始认识Clang与LLVM](https://www.jianshu.com/p/513a9bd35a7d)
[Developer_Yancy](https://www.jianshu.com/u/3f995dac0230) - [iOS底层探索(二) - 从零开始认识Clang与LLVM](https://www.jianshu.com/p/c9fccc93ed15)
[http://clang.llvm.org/](http://clang.llvm.org/)
[http://www.aosabook.org/en/llvm.html](http://www.aosabook.org/en/llvm.html)
[Clang](https://zh.wikipedia.org/wiki/Clang)
[LLVM](https://zh.wikipedia.org/wiki/LLVM)
[C预处理器](https://zh.wikipedia.org/wiki/C%E9%A2%84%E5%A4%84%E7%90%86%E5%99%A8)
[The Compiler](https://www.objc.io/issues/6-build-tools/compiler/)