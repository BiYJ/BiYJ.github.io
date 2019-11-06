---
title: iOS umbrella header
categories: iOS原理
---

> <font color=#FF9900>Lexical or Preprocessor Issue - Umbrella header for module 'xxx' does not include header 'xxx.h'</font>

framework 的文件明明被主工程引用了，但是在编译的时候依旧抛出上面的警告。


## 一、什么是 umbrella header? 

参考官方文档[《Introduction to Framework Programming Guide》](https://developer.apple.com/library/ios/documentation/MacOSX/Conceptual/BPFrameworks/Frameworks.html#//apple_ref/doc/uid/10000183-SW1)，可以了解到 Framework 区分Standard Framework 和 Umbrella Framework。但是并没有找到官方文档有对 Umberlla framework 给出明确的定义。在官方文档[《Anatomy of Framework Bundles》](https://developer.apple.com/library/ios/documentation/MacOSX/Conceptual/BPFrameworks/Concepts/FrameworkAnatomy.html#//apple_ref/doc/uid/20002253-97623-BAJJHAJC)章节中， 找到三段比较合理说明 Umbrella Framework 的话：

> Umbrella frameworks add minor refinements to the standard framework structure， such as the ability to encompass other frameworks

> The structure of an umbrella framework is similar to that of a standard framework， and applications do not distinguish between umbrella frameworks and standard frameworks when linking to them. However， two factors distinguish umbrella frameworks from other frameworks. The first is the manner in which they include header files. The second is the fact that they encapsulate subframeworks.

> Physically， umbrella frameworks have a similar structure to standard frameworks. One significant difference is the addition of a Frameworks directory to contain the subframeworks that make up the umbrella framework.

字面上的意思应该是在标准的 Framework 做了一些改良的工作，使其可以嵌套包含 Framework。在物理结构上，Umbrella Framework 只在包含头文件的方式以及是否包含子 Framework 和普通的 Framework 存在区别。

那么引用头文件的地方又有什么区别呢? 还是参考官方文档引用:

>For most frameworks， you can include header files other than the master header file. You can include any specific header file you want as long as it is available in the framework’s Headers directory. However， if you are including an umbrella framework， you must include the master header file. Umbrella frameworks do not allow you to include the headers of their constituent subframeworks directly. See Restrictions on Subframework Linking for more information.

简单翻译一下: 普通的 Framework 可以通过引用对应的 heaedr 文件而不是 <font color=#cc0000>Master Header File</font> 去引用需要使用的类，只需要对应的 header 头文件在 Headers 文件夹下暴露，并没有强制要求引用 Master Header File。Umbrella Framework 要求必须要引用 Master Header File，并且头文件中不能直接引用子 Framework 的东西。

上述描述已经说了 Umbrella Framework 一定要引用 Master Header File，而 <font color=#cc0000>Umbrella Framework 的 Master Header File 就是 Umbrella header 文件</font>。

官方说明中只有强制规定一定要引用 Umbrella Header 文件，但是却没有说能不能单独引用 Umbrella Framework 的其他头文件呢? 我们可以自己试验一下:

1. 在 Umbrella Framework 新建一个 testObject 类，分别产生了 testObject.h 和 testObject.m 文件。
2. 打开 Framework 配置文件，在 Build Phases的Headers 里的 Public 目录下，将 testObject.h 文件添加进去。
3. Build Framework 看是否报错。
4. 在主工程中调用初始化 testObject 对象，看编译是否报错。

执行结果:

* 步骤3: 没有编译报错， 但是报出了 <font color=#ff9900>Lexical or Preprocessor Issue - Umbrella header for module 'STDemoUI' does not include header 'testObject.h'</font>的警告。
* 步骤4: 执行正常。

总结一下:

1. Standard Framework 不能包含 Sub Framework；Umbrella Framework 可以包含子 Framework;
2. Standard Framework 可以直接引用需要使用的头，也可以通过引用 Master Header file 来引用需要使用的类；Umbrella Framework 需要通过引用 Master Header File(Umbrella Header) 来引用需要使用的类;


## 二、规范的写法

Umbrella Framework 默认会创建一个同名 .h 文件做为 Umbrella Header 文件。规范的写法当然是遵从默认的模式，将所有需要暴露的头文件都写在 Umbrella Header 文件中。

例如: STDemoUI.framework 工程包含了 STClassOne、STClassTwo 和 STClassThree 三个类。

STDemoUI 会生成一个默认的伞头文件（直译 Umbrella Header）STDemoUI.h。假设该 framework 的三个类均需要在外部调用使用，则 STDemoUI.h 需要将三个类的引用均写入伞头文件中。

```
// STDemoUI.h
// ...

#import <STDemoUI/STClassOne.h>
#import <STDemoUI/STClassTwo.h>
#import <STDemoUI/STClassThree.h>
```

在需要调用的主工程中， 仅仅只要将 Umbrella Header 引用即可调用所有在 Umbrella Header 中包含的类了。

```
// 在主工程需要应用的类中包含Umbrella Header
#import <STDemoUI/STDemoUI.h>
```

## 三、重命名 umbrella header

如果大家都遵从默认的 Umbrella Framework 的写法，在同名头文件中写需要暴露的引用头文件，那么就不需要考虑怎么重命名 Umbrella header 了。

很多时候，理想和现实是有差距的，程序员写代码多数是在二次接手进行开发的。假设公司的前辈已经将 Framework 的同名文件用作了一个逻辑类， 给同名文件创建了 .m 文件， 并已经书写了逻辑并应用了各个工程里面去了。那么显然迁移头文件功能代码是不可能的，因为很多依赖该 Framework 的业务部门都需要针对库进行代码优化。

在这种不能将同名文件作为 Umbrella header 的情况下，我们又不想通过 Public 强制暴露头文件的情况下（不写在 Umbrella Header 中会有警告）。我们就需要对 Umbrella Header 进行指定了。

1. 指定 Umbrella Header 入口在哪里呢?

	* 在工程全局搜索 umbrella 关键字 - <font color=#cc0000>Failed</font>
	* 在Build Settings里搜索umbrella关键字 - <font color=#cc0000>Failed</font>
	* 在打包好的 STDemoUI.framework 中搜索 umbrella 关键字 - <font color=#65AC47>Success</font>
	
	双击点开 STDemo.framework，内容如下：
	
	![](http://dzliving.com/iOSUmbrellaHeader_1.png)

	初略看名称可以推测出每个文件以及文件夹所承担的作用:
	
	* _CodeSignature：保存签名相关文件
	* Headers：framework 暴露的所有头文件
	* Info.plist：描述了该 framework 所包含的项目配置信息
	* UmbrellaFramework：编译后的核心库文件
	* Modules：模块相关文件夹， 目测只包含了 module.modulemap 文件
	* Frameworks：包含的子 framework

	我们在 module.modulemap 文件中找到了 umbrella 关键字。文件内容如下:

	```
	framework module STDemoUI {
	  umbrella header "STDemoUI.h"
	
	  export *
	  module * { export * }
	}
	```

	原来 Framework 的 umbrella header 是在这个位置被指定的，但是这个已经是编译好的工程， 我们总不能每次编译好了再进到包里面修改下。既然我们已经找到 umbrella header 是在 module 中去指定，那么我们就用 module 作为关键字再去 Build Settings 里重新搜索下。
	
	这回在 Kernel Module 和 Packaging 中均找到了 Module 关键字，在 Packaging 标签中，有一项 Module Map File 属性，看名字应该是用来指定 modulemap 文件的。
	
	![](http://dzliving.com/iOSUmbrellaHeader_0.png)

2. 指定 Modulemap 文件

	1. 创建一个新的 .h 文件，如：STHeader.h。将所有需要暴露的头文件均写入 STHeader.h
	2. 创建一个新的 modulemap 文件，如：stdemoalt.modulemap
	3. 在新的 modulemap 中指定 umbrella header

		```
		framework module STDemoUI {
		    umbrella header "STHeader.h"
		    
		    export *
		    module * { export * }
		}
		```
	4. 在 framework 的 Build Settings 中的 Module Map File 指定新建的 modulemap 文件。

	5. CMD+B 编译，打开 framework 包中的 Module 文件夹，看是否包含了新指定的 modulemap。

抛出两个疑问:

1. Module 是什么?
2. 如果 Defines Module 指定为 NO， 那会发生什么事情呢?


## 四、总结

本文简单的梳理了官方文章关于 Umbrella Framework 和 Umbrella Header 的介绍说明，产生警告的原因是没有引用 umbrella header 或者暴露头没有写在 umbrella header 中。在 umbrella header 被已使用的前提下，本文提供了一种通过重命名 Umbrella Header 文件的方式来消除警告的解决方案。

虽然引用警告可以被消除，但是建议大家还是采用规范的做法：尽量不要在同名头文件中写业务逻辑代码， 用同名文件作为 Umbrella 库的 Master Header File。

## 文章

[iOS - Umbrella Header在framework中的应用](https://www.jianshu.com/p/894a36643758)