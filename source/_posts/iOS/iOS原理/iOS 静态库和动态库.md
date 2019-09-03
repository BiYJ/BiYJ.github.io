---
title: iOS 静态库和动态库
categories: iOS原理
---

## 一、库

#### 1.1 什么是库？

库就是<font color=#cc0000>程序代码的集合</font>，将 N 个文件组织起来，是<font color=#cc0000>共享</font>程序代码的一种方式。从本质上来说是一种可执行代码的二进制格式，可以被载入内存中执行。

#### 1.2 库的分类

根据程序代码的开源情况，库可以分为两类

* 开源库

	源代码是公开的，你可以看到具体实现。比如知名的第三方框架：AFNetworking、SDWebImage。
    
*   闭源库

	不公开源代码，只公开调用的接口，看不到具体的实现，是一个<font color=#cc0000>编译后的二进制文件</font>。这种常见于一些公司的 SDK 包，比如高德地图 SDK、环信即时通讯 SDK 等。而闭源库又分为两类：静态库和动态库。
    

#### 1.3 从源代码到 app

当我们点击了 build 之后，做了什么事情呢？

* 预处理（Pre-process）：把宏替换、删除注释、展开头文件，产生 <font color=#cc0000>.i</font> 文件。  
* 编译（Compliling）：把之前的 .i 文件转换成汇编语言，产生 <font color=#cc0000>.s</font> 文件。  
* 汇编（Asembly）：把汇编语言文件转换为机器码文件，产生 <font color=#cc0000>.o</font> 文件。 
* 链接（Link）：对 .o 文件中的对于其他的库的引用的地方进行引用，生成最后的可执行文件（同时也包括多个 .o 文件进行 link）。
    

#### 1.4 iOS 设备的 CPU 架构

模拟器：

	4s-5: i386
	5s-iPhone X（包括 iPhone SE）: x86_64

真机（iOS设备）：
	
	armv6：iPhone、iPhone 2、iPhone 3G、iPod Touch（第一代）、iPod Touch（第二代）
	armv7：iPhone 3Gs、iPhone 4、iPhone 4s、iPad、iPad 2
	armv7s：iPhone 5、iPhone 5c（静态库只要支持了 armv7，就可以在 armv7s 的架构上运行，向下兼容）
	arm64：iPhone 5s、iPhone 6、iPhone 6 Plus、iPhone 6s、iPhone 6s Plus、iPad Air、iPad Air2、iPad mini2、iPad mini3、iPhone 7、iPhone 7 Plus、iPhone 8、iPhone 8 Plus、iPhone X


## 二、静态库和动态库

静态和动态是相对<font color=#cc0000>编译期和运行期</font>而言的：

> 静态库在程序编译时会被链接到目标代码中，程序运行时将不再需要该静态库；动态库在程序编译时并不会被链接到目标代码中，只是在程序运行时才被载入。

存在形式：

* 静态库
    
    以 “.a” 或者 “.framework” 为文件后缀名。
    

	<font color=#cc0000>.a 是一个纯二进制文件，.framework 中除了有二进制文件之外还有资源文件</font>。.a 要有 .h 文件以及资源文件配合，.framework 文件可以直接使用。总的来说，.a + .h + sourceFile = .framework。所以创建静态库最好还是用 .framework 的形式。

*   动态库
    
    以 “.dylib” 或者 “.framework” 为文件后缀名（Xcode7 之后 .tbd 代替了 .dylib）
    
使用区别：

1. 静态库链接时会被完整的复制到可执行文件中，被多次使用就有多份拷贝。
    
	<center>
	![image.png](https://upload-images.jianshu.io/upload_images/5294842-4d4e19837481d9e1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

	利用静态函数库编译成的<font color=#cc0000>文件比较大</font>，因为整个函数库的所有数据都会被整合进目标代码中。

	它的优点就显而易见了，即<font color=#cc0000>编译后的执行程序不需要外部的函数库支持</font>，因为所有使用的函数都已经被编译进去了。当然这也会成为他的缺点，因为如果静态函数库改变了，那么你的程序必须重新编译。

2. 动态库链接时不复制，程序运行时由系统动态加载到内存，供程序调用。而且系统只加载一次，多个程序共用，节省内存。
    
    <center>
	![image.png](https://upload-images.jianshu.io/upload_images/5294842-4209e8ff6f8cb0e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

	相对于静态函数库，动态函数库在编译的时候 并没有被编译进目标代码中，你的程序执行到相关函数时才调用该函数库里的相应函数，因此动态函数库所产生的<font color=#cc0000>可执行文件比较小</font>。由于函数库没有被整合进你的程序，而是程序运行时动态的申请并调用，所以程序的运行环境中必须提供相应的库。动态函数库的改变并不影响你的程序，所以动态函数库的升级比较方便。

各自优点：

* 静态库：

	①、模块化，分工合作，提高了代码的复用及核心技术的保密程度
	②、避免少量改动经常导致大量的重复编译连接
	③、也可以重用，注意不是共享使用

*   动态库：

	①、可以将最终可执行文件体积缩小，将整个应用程序分模块，团队合作，进行分工，影响比较小
	②、多个应用程序共享内存中得同一份库文件，节省资源
	③、可以不重新编译连接可执行程序的前提下，更新动态库文件达到更新应用程序的目的
	④、应用插件化
	⑤、软件版本实时模块升级
	⑥、在其它大部分平台上，动态库都可以用于不同应用间共享， 共享可执行文件，这就大大节省了内存。

在 <font color=#cc0000>iOS8 之前</font>，苹果不允许第三方框架使用动态方式加载，从 iOS8 开始允许开发者<font color=#cc0000>有条件地</font>创建和使用动态框架，这种框架叫做 Cocoa Touch Framework。虽然同样是动态框架，但是和系统 framework 不同，苹果系统专属的 framework 是共享的（如 UIKit），使用 Cocoa Touch Framework 制作的动态库在打包和提交 app 时会被放到 app main bundle 的根目录中，<font color=#cc0000>运行在沙盒里</font>，而不是系统中。也就是说，不同的 app 就算使用了同样的 framework，但还是会有多份的框架被<font color=#cc0000>分别</font>签名、打包和加载。不过 iOS8 上开放了 App Extension 功能，可以为一个应用创建插件，这样主 app 和插件之间共享动态库还是可行的。


## 三、静态库的处理方式

* 对于一个静态库而言，其实已经是编译好的了（类似一个 .o 的集合），这里并没有连接。在 build 的过程中只会参与链接的过程，而这个链接的过程简单的讲就是<font color=#cc0000>合并</font>，并且链接器只会将静态库中被使用的部分合并到可执行文件中去。相比较于动态库，静态库的处理起来要简单的多，具体如下图：
    
	<center>
	![23](https://upload-images.jianshu.io/upload_images/5294842-41e5b14bdf5ce946.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

* 链接器会将所有 .o 用到的 global symbol 和 unresolved symbol 放入一个临时表，而且是 global symbol 是不能重复的。
* 对于静态库的 .o，链接器会将没有任何 symbol 在 unresolved symbol table 的给忽略。
* unresolved symbol 类似 extern int test(); ---  .h 的声明?  
* global symbol 类似 void test() { print("test") } \-\- .m 的实现?
* 最后，链接器会用函数的实际地址来代替函数引用。
    

## 四、动态库的处理方式

* 首先，对于动态库而言其实分<font color=#cc0000>动态链接库和动态加载库</font>两种的，这两个最本质的区别还是<font color=#cc0000>加载时间</font>。
    
	动态链接库：在没有被加载到内存的前提下，当可执行文件被加载，动态库也随着被加载到内存中。在 Linked Framework and Libraries 设置的一些 share libraries。【随着程序启动而启动】

	动态加载库：当需要的时候再使用 <font color=#cc0000>dlopen</font> 等通过代码或者命令的方式来加载。【在程序启动之后】

* 但是不论是哪种动态库，相比较与静态库，动态库处理起来要棘手的多。由于动态库是动态的，所以你事先不知道某个函数的具体地址，因此动态链接器在链接函数的时候需要做大量的工作。
    
> 因为动态库在链接函数需要做大量的工作，而静态库已经实现处理好了。所以单纯的在所有都没有加载的情况下，静态库的加载速度会更快一点。而在 [iOS 开发中的『库』(一)](https://github.com/Damonvvong/iOSDevNotes/blob/master/Notes/framework.md) 提到的有所不妥，正确应该是，虽然动态库更加耗时，但是对于加载过的 share libraries 不需要再加载的这个前提下，使用动态库可以节省一些启动时间。

* 而实现这个动态链接是使用了 Procedure Linkage Table (PLT)。首先这个 PLT 列出了程序中每一个函数的调用，当程序开始运行，如果动态库被加载到内存中，PLT 会去寻找动态的地址并记录下来，如果每个函数都被调用过的话，下一次调用就可以通过 PLT 直接跳转了，但是和静态库还是有点区别的是，每一个函数的调用还是需要通过一张 PLT。这也正是 sunny 所说的所有静态链接做的事情都搬到运行时来做了，会导致更慢的原因。
    

## 五、动态库的作用

1. 应用插件化

	<font color=#cc0000>每一个功能点都是一个动态库</font>，在用户想使用某个功能的时候让其从<font color=#cc0000>网络下载</font>，然后手动加载动态库，实现功能的的插件化。

	虽然技术上来说这种动态更新是可行的，但是对于 AppStore 上上架的 app 是不可以的。<font color=#cc0000>iOS8</font> 之后虽然可以上传含有动态库的 app，但是苹果不仅需要你动态库和 app 的<font color=#cc0000>签名一致</font>，而且苹果会在你上架的时候<font color=#cc0000>再经过一次 AppStore 的签名</font>。所以你想在线更新动态库，首先你得有苹果 AppStore 私钥，而这个基本不可能。

	除非你的应用不需要通过 AppStore 上架，比如企业内部的应用，<font color=#cc0000>通过企业证书发布</font>，那么就可以实现应用插件化在线更新动态库了。

2. 共享可执行文件

	在其它大部分平台上，动态库都可以用于不同应用间共享，这就大大节省了内存。从目前来看，iOS 仍然不允许进程间共享动态库，即 iOS 上的动态库只能是<font color=#cc0000>私有的</font>，因为我们仍然不能将动态库文件放置在除了自身<font color=#cc0000>沙盒</font>以外的其它任何地方。

	iOS8 上开放了 App Extension 功能，可以为一个应用创建插件，这样主 app 和插件之间共享动态库还是可行的。

## 六、签名

系统在加载动态库时，会检查 framework 的签名，签名中必须包含 <font color=#cc0000>TeamIdentifier</font>，并且 framework 和 host app 的 TeamIdentifier 必须一致。

我们在 Debug 测试的时候是不会报错的，在打包时如果有动态库，那么就会检查 TeamIdentifier。

如果不一致，否则会报下面的错误：

```
Error loading /path/to/framework: dlopen(/path/to/framework, 265): no suitable image found. Did find:/path/to/framework: mmap() error 1
```

如果用来打包的证书是 iOS 8 发布之前生成的，则打出的包验证的时候会没有 TeamIdentifier 这一项。这时在加载 framework 的时候会报下面的错误：

```
[deny-mmap] mapped file has no team identifier and is not a platform binary:/private/var/mobile/Containers/Bundle/Application/5D8FB2F7-1083-4564-94B2-0CB7DC75C9D1/YourAppNameHere.app/Frameworks/YourFramework.framework/YourFramework
```

可以通过 <font color=#cc0000>codesign</font> 命令来验证。

```
codesign -dv /path/to/YourApp.app
或
codesign -dv /path/to/YourFramework.framework
```

如果证书太旧，输出的结果如下：

```
Executable=/path/to/YourApp.app/YourApp
Identifier=com.company.yourapp
Format=bundle with Mach-O thin (armv7)
CodeDirectory v=20100 size=221748 flags=0x0(none) hashes=11079+5 location=embedded
Signature size=4321
Signed Time=2015年10月21日 上午10:18:37
Info.plist entries=42
TeamIdentifier=not set
Sealed Resources version=2 rules=12 files=2451
Internal requirements count=1 size=188
```

注意其中的 <font color=#cc0000>TeamIdentifier=not set</font>。

我们在用 cocoapods 的 use_framework! 的时候生成的动态库也可以用 ``codesign -dv /path/to/youFramework.framework`` 查看到 TeamIdentifier=not set。


## 七、Framework

#### 7.1 什么是 Framework

Framework 是 Cocoa/Cocoa Touch 程序中使用的一种<font color=#cc0000>资源打包方式</font>，可以将代码文件、头文件、资源文件、说明文档等集中在一起，方便开发者使用。一般如果是静态 Framework 的话，资源打包进 Framework 是读取不了的。静态 Framework 和 .a 文件都是编译进可执行文件里面的。只有动态 Framework 能在 .app 下面的 Framework 文件夹下看到，并读取 .framework 里的资源文件。

<center>
![13](https://upload-images.jianshu.io/upload_images/5294842-27dfb1bcb970edb1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

Cocoa/Cocoa Touch 开发框架本身提供了大量的 Framework，比如 Foundation.framework/UIKit.framework 等。需要注意的是，这些 Framework 无一例外<font color=#cc0000>都是动态库</font>。

平时用的第三方 SDK 的 Framework 都是静态库，真正的动态库是上不了 AppStore（iOS8 之后能上 AppStore，因为 App Extension，需要动态库支持)。

> Framework 为什么既是静态库又是动态库？
    
系统的 .framework 是动态库，我们自己建立的 .framework 一般都是静态库。但是现在用 xcode 创建Framework 的时候默认是动态库，一般打包成 SDK 给别人用的话都使用的是静态库，可以修改 Build Settings的 ``Mach-O Type`` 为 Static Library。

## 八、Framework 目录

<center>
![14](https://upload-images.jianshu.io/upload_images/5294842-79fef5245c586c10.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

* Headers
    
	表示暴露的头文件，一般都会有一个和 Framework 同名的 .h 文件，在创建 Framework 的时候文件夹里也会默认生成这样一个文件。有这个和 Framework 同名的 .h 文件 <font color=#cc0000>@import</font> 导入库的时候编译器才能找到这个库（@import 导入头文件可参考 [iOS里的导入头文件](https://link.jianshu.com?t=https://www.zybuluo.com/qidiandasheng/note/602118)）。

* info.plist
    
	主要就是这个 Framework 的一些配置信息。

* Modules

	这个文件夹里有个 <font color=#cc0000>module.modulemap</font> 文件

	```
	framework module DynamicFramework {
	  umbrella header "DynamicFramework.h"
	
	  export *
	  module * { export * }
	}
	```

	这里面有这样一句 umbrella header "DynamicFramework.h"，umbrella 有保护伞、庇护的意思。

	也就是说 Headers 中暴露的 DynamicFramework.h 文件被放在 umbrella 雨伞下保护起来了，所以我们需要将其他的所有需要暴露的 .h 文件放到 DynamicFramework.h 文件中保护起来，不然会出现警告。<font color=#cc0000>@import 的时候也只能找到 umbrella 雨伞下保护起来的 .h 文件</font>。

* 二进制文件（Unix 可执行文件）

	这个就是你源码编译而成的二进制文件，主要的执行代码就在这个里面。

* .bundle 文件

	如果我们在 Build Phases -> Copy Bundle Resources 里加入 .bundle 文件，那么创建出来的 .Framework 里就会有这个 .bundle 的资源文件夹。


## 九、Framework 的资源文件

CocoaPods 如何生成 Framework 的资源文件？

我们能看到用 cocoapods 创建 Framework 的时候，Framework 里面有一个 .bundle 文件，跟 Framework 同级目录里也有一个 .bundle文件。这两个文件其实是一样的。

那这两个 .bundle 是怎么来的呢？我们能看到用 <font color=#cc0000>use_frameworks!</font> 生成的 pod 里面，pods 这个 PROJECT 下面会为每一个 pod 生成一个 target。

<center>
![15](https://upload-images.jianshu.io/upload_images/5294842-fd111dc35645dcbc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

那么如果这个 pod 有资源文件的话，就会有一个叫 xxx-bundleName 的 target，最后这个 target 生成的就是 bundleName.bundle。

<center>
![16](https://upload-images.jianshu.io/upload_images/5294842-a7a001c18e499390.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

在 xxx 的 target 的 Build Phases -> Copy Bundle Resources 里加入这个 .bundle，在 Framework 里面就会生成这样一个 bundle。在 xxx 的 target 的 Build Phases -> Target Dependencies 里加入这个 target：xxx-bundleName，就会在 Framework 的同级目录里生成这样一个 bundle。

<font color=#cc0000>静态 Framework 里不需要加入资源文件</font>。一般资源打包进静态 Framework 是读取不了的。

静态 Framework 和 .a 文件都是编译进可执行文件里面的。只有动态 Framework 能在 .app 的 Framework 文件夹下看到，并读取 .framework 里的资源文件。

你可以用 NSBundle \* bundle = \[\[NSBundle mainBundle\] bundlePath\]; 得到 .app 目录，如果是动态库你能在 Framework 目录下看到这个动态库以及动态库里面资源文件。然后你只要用 NSBundle \* bundle = \[NSBundle <font color=#cc0000>bundleForClass:<#ClassFromFramework#></font>\]; 得到这个动态库的路径就能读取到里面的资源了。但是如果是静态库的话，因为编译进了可执行文件里面，你也就没办法读到这个静态库了，你能看到 .app 下的 Framework 目录为空。

[在 Framework 或子工程中使用 xib](https://link.jianshu.com?t=https://blog.cnbluebox.com/blog/2014/12/08/xib-in-frameworks/)


## 十、问题

* 如果静态库中有 category 类，则在使用静态库的项目配置中【Other Linker Flags】需要添加参数[-ObjC\] 或者 [-all\_load]。
    
* 出现 Umbrella header for module 'XXXX' does not include header 'XXXXX.h'

	因为把 xxxxx.h 错误的拖到了 public 中。

* 出现 dyld: Library not loaded:XXXXXX

	是因为打包的 Framework 版本太高。比如打包 Framework 时，选择的是 iOS 9.0，而实际的工程环境是 iOS 8 开始的。需要到 iOS Deployment Target 设置对应版本。

* 报错 "Include of non-modular header inside framework module"

	如果创建的 Framework 类中使用了 .dylib 或者 .tbd，首先需要在实际项目中导入 .dylib 或者 .tbd 动态库，然后需要设置 Allow Non-modular Includes In Framework Modules = YES

* 有时候我们会发现在使用的时候加载不了动态 Framework 里的资源文件，其实是加载方式不对，比如用 pod 的时候使用的是 use_frameworks!，那么资源是在 Framework 里面的，需要使用以下代码加载（具体可参考[给pod添加资源文件](https://link.jianshu.com?t=https://www.zybuluo.com/qidiandasheng/note/392639#%E7%BB%99pod%E6%B7%BB%E5%8A%A0%E8%B5%84%E6%BA%90%E6%96%87%E4%BB%B6)）：
    
	```
	NSBundle * bundle = [NSBundle bundleForClass:<#ClassFromFramework#>];
	
	NSString * path = [bundle pathForResource:@"imageName@2x"(@"bundleName.bundle/imageName@2x") ofType:@"png"];
	UIImage * image = [UIImage imageWithContentsOfFile:path];
	```

*   报错 Reason: image not found    

	如果直接在工程里使用创建的动态库时候会出现此错误，需要在工程的 General 里的 Embedded Binaries 添加这个动态库才能使用。

	因为创建的这个动态库其实也不能给其他程序使用的，而你的 App Extension 和 APP 之间是需要使用这个动态库的。这个动态库可以 App Extension 和 APP 之间共用一份（App 和 Extension 的 Bundle 是共享的），因此苹果又把这种 Framework 称为 Embedded Framework。


## 十一、Swift 支持

跟着 iOS8/Xcode6 同时发布的还有 Swift。如果要在项目中使用外部的代码，可选的方式只有两种：1、把代码拷贝到工程中；2、用动态 Framework。使用静态库是不支持的。

造成这个问题的原因主要是 Swift 的运行库没有被包含在 iOS 系统中，而是会打包进 App 中（这也是造成 Swift App 体积大的原因），静态库会导致最终的目标程序中包含重复的运行库（这是苹果[自家的解释](https://github.com/ksm/SwiftInFlux#static-libraries)）。同时拷贝 Runtime 这种做法也会导致在纯 ObjC 的项目中使用 Swift 库出现问题。苹果声称等到 Swift 的 Runtime 稳定之后会被加入到系统当中，到时候这个限制就会被去除了（参考[这个问题](https://stackoverflow.com/questions/25020783/how-to-distribute-swift-library-without-exposing-the-source-code)的问题描述，也是来自苹果自家文档）。


## 十二、CocoaPods 的做法

在纯 ObjC 的项目中，CocoaPods 使用编译静态库 .a 方法将代码集成到项目中。在 Pods 项目中的每个 target 都对应着一个 Pod 的静态库。

<center>
![18](https://upload-images.jianshu.io/upload_images/5294842-3a3243986133d7ee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

当不想发布代码的时候，也可以使用 Framework 发布 Pod，CocoaPods 提供了 vendored_framework 选项来使用第三方 Framework。

对于 Swift 项目，CocoaPods 提供了动态 Framework 的支持。通过 use_frameworks! 选项控制。<font color=#cc0000>对于 Swift 写的库来说，想通过 CocoaPods 引入工程，必须加入 use_frameworks! 选项</font>。

<center>
![17](https://upload-images.jianshu.io/upload_images/5294842-c6d620cad68fb083.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>


## 十三、关于 use_frameworks!

在使用 CocoaPods 的时候在 Podfile 里加入 use\_frameworks!，那么你在编译的时候就会默认帮你生成动态库，我们能看到每个源码 Pod 都会在 Pods 工程下面生成一个对应的动态库 Framework 的 target，我们能在这个 target 的 Build Settings -> Mach-O Type 看到默认设置是 Dynamic Library，也就是会生成一个动态 Framework，我们能在 Products 下面看到每一个 Pod 对应生成的动态库。

这些生成的动态库将链接到主项目给主工程使用，但是我们上面说过动态库需要在主工程 target 的 General -> Embedded Binaries 中添加才能使用，而我们并没有在 Embedded Binaries 中看到这些动态库。那这是怎么回事呢，<font color=#cc0000>其实是 cocoapods 已经执行了脚本把这些动态库嵌入到了 .app 的 Framework 目录下</font>，相当于在 Embedded Binaries 加入了这些动态库。我们能在主工程 target 的 Build Phase -> Embed Pods Frameworks 里看到执行的脚本。

<center>
![19](https://upload-images.jianshu.io/upload_images/5294842-8acb981c27c9b453.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

所以 <font color=#cc0000>Pod 默认是生成动态库</font>，然后嵌入到 .app 下面的 Framework 文件夹里。我们去 Pods 工程的 target 里把 Build Settings -> Mach-O Type 设置为 Static Library。那么生成的就是静态库，但是 cocoapods 也会把它嵌入到 .app 的Framework目录下，而因为它是静态库，所以会报错：unrecognized selector sent to instanceunrecognized selector sent to instance。

## 十四、参考

[创建一个 iOS Framework 项目](https://link.jianshu.com?t=http://www.samirchen.com/create-a-framework/)
[Xcode7创建静态库和Framework](https://link.jianshu.com?t=https://www.gitbook.com/book/xianjun/xcode7_framework_and_static_lib/details)
[iOS 静态库开发](https://www.jianshu.com/p/8f5b9855efb8)
[静态库与动态库的使用](https://link.jianshu.com?t=https://www.gitbook.com/book/leon_lizi/-framework-/details)
[iOS 静态库，动态库与 Framework](https://link.jianshu.com?t=https://skyline75489.github.io/post/2015-8-14_ios_static_dynamic_framework_learning.html)
[签名](https://link.jianshu.com?t=http://nixwang.com/2015/11/09/ios-dynamic-update/)
[iOS打包静态库（完整篇）](https://www.cnblogs.com/weiming4219/p/7827197.html)
[iOS armv7、armv7s、 arm64](https://blog.csdn.net/u011545443/article/details/42294883)
[iOS 创建 .a 和 .framework 静态库，以及 Bundle 资源文件的使用](https://blog.csdn.net/pangshishan1/article/details/72179898)
[iOS 静态库和动态库（库详解）](https://www.cnblogs.com/junhuawang/p/7598236.html)
[齐滇大圣](https://www.jianshu.com/u/9b722b9acb00) & [iOS里的动态库和静态库](https://www.jianshu.com/p/42891fb90304)