---
title: iOS App的启动过程
categories: iOS原理
---


## 一、mach-O

* Executable 可执行文件
* Dylib 动态库
* Bundle 无法被连接的动态库，只能通过 dlopen() 加载
* Image 指的是 Executable，Dylib 或者 Bundle 的一种。
* Framework 动态库和对应的头文件和资源文件的集合

Apple 出品的操作系统的可执行文件格式几乎都是 mach-O。

mach-O 可以大致分为三部分：

<center>
![](http://dzliving.com/iOSStartUp_0.png)
</center>

* Header 头部，包含可以执行的 CPU 架构，比如 x86，arm64
* Load commands 加载命令，包含文件的组织架构和在虚拟内存中的布局方式
* Data 数据，包含 load commands 中需要的各个段(segment)的数据，每个 Segment 的大小都是 Page 的整数倍。

用 MachOView 打开 Demo 工程的可以执行文件，来验证下 mach-O 的文件布局：

<center>
![](http://dzliving.com/iOSStartUp_1.png)
</center>

图中分析的 mach-O 文件来源于 [PullToRefreshKit](https://github.com/LeoMobileDeveloper/PullToRefreshKit)。这是一个纯 Swift 的编写的工程。

那么 Data 部分又包含那些 segment 呢？绝大多数 mach-O 包括以下三个阶段（支持用户自定义Segment，但是很少使用）

* \_\_TEXT 代码段，只读。包含函数和只读的字符串，上图中类似 \_\_TEXT,\_\_text 的都是代码段
* \_\_Data 数据段，读写。包括可读写的全局变量等，\_\_DATA,\_\_data 都是数据段
* \_\_LINKEDIT 包含了方法和变量的元数据(位置，偏移量)，以及代码签名等信息。

关于 mach-O 更多细节，可以看看文档：[《Mac OS X ABI Mach-O File Format Reference》](https://github.com/LeoMobileDeveloper/React-Native-Files/blob/master/Mac%20OS%20X%20ABI%20Mach-O%20File%20Format%20Reference.pdf)


## 二、dyld

dyld 的全称是 [dynamic loader](https://developer.apple.com/library/content/releasenotes/DeveloperTools/RN-dyld/)，它的作用是加载一个进程所需要的 image，dyld 是[开源的](https://opensource.apple.com/source/dyld/)。


## 三、Virtual Memory

>虚拟内存是在物理内存上建立的一个逻辑地址空间，它向上（应用）提供了一个连续的逻辑地址空间，向下隐藏了物理内存的细节。
> 虚拟内存使得逻辑地址可以没有实际的物理地址，也可以让多个逻辑地址对应到一个物理地址。
> 虚拟内存被划分为一个个大小相同的 Page（64 位系统上是 16KB），提高管理和读写的效率。 Page 又分为只读和读写的 Page。

虚拟内存是建立在物理内存和进程之间的中间层。在 iOS 上，当内存不足的时候，会尝试释放那些只读的 Page，因为只读的 Page 在下次被访问的时候，可以再从磁盘读取。如果没有可用内存，会通知在后台的App（也就是在这个时候收到了memory warning），如果在这之后仍然没有可用内存，则会杀死在后台的App。


## 四、Page fault

在应用执行的时候，它被分配的逻辑地址空间都是可以访问的，当应用访问一个逻辑 Page，而在对应的物理内存中并不存在的时候，这时候就发生了一次 Page fault。当 Page fault 发生的时候，会中断当前的程序，在物理内存中寻找一个可用的 Page，然后从磁盘中读取数据到物理内存，接着继续执行当前程序。

## 五、Dirty Page & Clean Page

* 如果一个 Page 可以从磁盘上重新生成，那么这 Page 称为 Clear Page
* 如果一个 Page 包含了进程相关信息，那么这个 Page 称为 Dirty Page

像代码段这种只读的 Page 就是 Clean Page。而数据段(__DATA)这种读写的 Page，当写数据发生的时候，会触发 CO(Copy on write)，也就是写时复制，Page 会被标记成 Dirty，同时会被复制。

想要了解更多细节，可以阅读文档：[Memory Usage Performance Guidelines](https://developer.apple.com/library/content/documentation/Performance/Conceptual/ManagingMemory/ManagingMemory.html#//apple_ref/doc/uid/10000160-SW1)


## 六、启动过程

使用 dyld2 启动应用的过程如图：

<center>
![](http://dzliving.com/iOSStartUp_3.png)
</center>

大致的过程如下：

* 加载 dyld 到 App 进程
* 加载动态库（包括所依赖的所有动态库）
* Rebase
* Bind
* 初始化 Objective-C Runtime
* 其它的初始化代码

#### 6.1 加载动态库

dyld 会首先读取 mach-O 文件的 Header 和 load commands。

接着就知道了这个可执行文件依赖的动态库。例如加载动态库 A 到内存，接着检查 A 所依赖的动态库，就这样的递归加载，直到所有的动态库加载完毕。通常一个 App 所依赖的动态库在 100~400 个左右，其中大多数都是系统的动态库，它们会被缓存到 dyld shared cache，这样读取的效率会很高。

查看 mach-O 文件所依赖的动态库，可以通过 MachOView 的图形化界面（展开 Load Command 就能看到），也可以通过命令行 otool。

```
$ otool -L demo 
demo:
    @rpath/PullToRefreshKit.framework/PullToRefreshKit (compatibility version 1.0.0, current version 1.0.0)
    /System/Library/Frameworks/Foundation.framework/Foundation (compatibility version 300.0.0, current version 1444.12.0)
    /usr/lib/libobjc.A.dylib (compatibility version 1.0.0, current version 228.0.0)
    @rpath/libswiftCore.dylib (compatibility version 1.0.0, current version 900.0.65)
    @rpath/libswiftCoreAudio.dylib (compatibility version 1.0.0, current version 900.0.65)
    //...
```
    

#### 6.2 Rebase && Bind

先讲讲为什么要 Rebase。

有两种主要的技术来保证应用的安全：<font color=#cc0000>ASLR</font> 和 <font color=#cc0000>Code Sign</font>。

ASLR 的全称是 Address space layout randomization，翻译过来就是“地址空间布局随机化”。App被启动的时候，程序会被影射到逻辑的地址空间，这个逻辑的地址空间有一个起始地址，而 ASLR 技术使得这个起始地址是随机的。如果是固定的，那么黑客很容易就可以由`起始地址 + 偏移量`找到函数的地址。

Code Sign 相信大多数开发者都知晓，这里要提一点的是，在进行 Code sign 的时候，<font color=#cc0000>加密哈希不是针对于整个文件，而是针对于每一个 Page 的</font>。这就保证了在 dyld 进行加载的时候，可以对每一个 page 进行独立的验证。

mach-O 中有很多符号，有指向当前 mach-O 的，也有指向其他 dylib 的，比如 printf。那么，在运行时，代码如何准确的找到 printf 的地址呢？

mach-O 中采用了 <font color=#cc0000>PIC</font> 技术，全称是 Position Independ code。当你的程序要调用 printf 的时候，会先在 __DATA 段中建立一个指针指向 printf，在通过这个指针实现间接调用。dyld 这时候需要做一些 fix-up 工作，即帮助应用程序找到这些符号的实际地址。主要包括两部分

* Rebase 修正内部（指向当前 mach-O 文件）的指针指向
* Bind 修正外部指针指向

<center>
![](http://dzliving.com/iOSStartUp_2.png)
</center>

之所以需要 Rebase，是因为刚刚提到的 ASLR 使得地址随机化，导致起始地址不固定，另外由于 Code Sign，导致不能直接修改 Image。Rebase 的时候只需要增加对应的偏移量即可。待 Rebase 的数据都存放在 __LINKEDIT 中。

可以通过 MachOView 查看：Dynamic Loader Info -> Rebase Info

```
$ xcrun dyldinfo -bind demo 
bind information:
segment section          address        type    addend dylib            symbol
__DATA  __got            0x10003C038    pointer      0 PullToRefreshKit __T016PullToRefreshKit07DefaultC4LeftC9textLabelSo7UILabelCvWvd
__DATA  __got            0x10003C040    pointer      0 PullToRefreshKit __T016PullToRefreshKit07DefaultC5RightC9textLabelSo7UILabelCvWvd
__DATA  __got            0x10003C048    pointer      0 PullToRefreshKit __T016PullToRefreshKit07DefaultC6FooterC9textLabelSo7UILabelCvWvd
__DATA  __got            0x10003C050    pointer      0 PullToRefreshKit __T016PullToRefreshKit07DefaultC6HeaderC7spinnerSo23UIActivityIndicatorViewCvWvd
//...
```

Rebase 解决了内部的符号引用问题，而外部的符号引用则是由 Bind 解决。在解决 Bind 的时候，是根据字符串匹配的方式查找符号表，所以这个过程相对于 Rebase 来说是略慢的。

同样，也可以通过 xcrun dyldinfo 来查看 Bind 的信息，比如我们查看 bind 信息中，包含 UITableView 的部分：

```
$ xcrun dyldinfo -bind demo | grep UITableView
__DATA  __objc_classrefs 0x100041940    pointer      0 UIKit            _OBJC_CLASS_$_UITableView
__DATA  __objc_classrefs 0x1000418B0    pointer      0 UIKit            _OBJC_CLASS_$_UITableViewCell
__DATA  __objc_data      0x100041AC0    pointer      0 UIKit            _OBJC_CLASS_$_UITableViewController
__DATA  __objc_data      0x100041BE8    pointer      0 UIKit            _OBJC_CLASS_$_UITableViewController
__DATA  __objc_data      0x100042348    pointer      0 UIKit            _OBJC_CLASS_$_UITableViewController
__DATA  __objc_data      0x100042718    pointer      0 UIKit            _OBJC_CLASS_$_UITableViewController
__DATA  __data           0x100042998    pointer      0 UIKit            _OBJC_METACLASS_$_UITableViewController
__DATA  __data           0x100042A28    pointer      0 UIKit            _OBJC_METACLASS_$_UITableViewController
__DATA  __data           0x100042F10    pointer      0 UIKit            _OBJC_METACLASS_$_UITableViewController
__DATA  __data           0x1000431A8    pointer      0 UIKit            _OBJC_METACLASS_$_UITableViewController
```


#### 6.3 Objective-C

Objective-C 是动态语言，所以在执行 main 函数之前，需要把类的信息注册到一个全局的 Table 中。同时，Objective-C 支持 Category，在初始化的时候，也会把 Category 中的方法注册到对应的类中，同时会唯一 Selector，这也是为什么当你的 Cagegory 实现了类中同名的方法后，类中的方法会被覆盖。

另外，由于 iOS 开发是基于 Cocoa Touch 的，所以绝大多数的类起始都是系统类，所以大多数的 Runtime 初始化起始在 Rebase 和 Bind 中已经完成。


#### 6.4 Initializers

接下来就是必要的初始化部分了，主要包括几部分：

* +load方法。
* C／C++ 静态初始化对象和标记为 attribute(constructor) 的方法

+load 方法已经被弃用了，如果你用 Swift 开发，你会发现根本无法去写这样一个方法，官方的建议是使用 initialize。区别就是，load 是在类装载的时候执行，initialize 是在类第一次收到 message 前调用。

#### 6.5 dyld3

上面讲解是 dyld2 的加载方式。而最新的是 dyld3 加载方式略有不同：

<center>
![](http://dzliving.com/iOSStartUp_4.png)
</center>

dyld2 是纯粹的 in-process，也就是在程序进程内执行的，也就意味着只有当应用程序被启动的时 dyld2 才能开始执行任务。

dyld3 则是部分 out-of-process，部分 in-process。图中，虚线之上的部分是 out-of-process的，在 App 下载安装和版本更新的时候会去执行，out-of-process 会做如下事情：

* 分析 Mach-o Headers
* 分析依赖的动态库
* 查找需要 Rebase & Bind 之类的符号
* 把上述结果写入缓存

这样，在应用启动的时候，就可以直接从缓存中读取数据，加快加载速度。


## 文章

[为自己丶拼个未来](https://www.jianshu.com/u/d6bd975a14aa) - [深入理解iOS App的启动过程](https://www.jianshu.com/p/a51fcabc9c71)