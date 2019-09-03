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