---
title: iOS Hook
categories: iOS原理
---

HOOK 译为“钩子”或挂钩。在 iOS 逆向中指改变程序运行流程的一种技术。

iOS 中 hook 技术的几种方式

1. Method Swizzle

	利用 OC 的 Runtime 特性，动态改变 SEL（方法编号）和 IMP（方法实现）的对应关系，达到 OC 方法调用流程改变的目的。主要用于 OC 方法。
	
2. fishhook

	它是 Facebook 提供的一个动态修改链接 mach-O 文件的工具。利用 MachO 文件加载原理，通过修改懒加载和非懒加载两个表的指针达到 C 函数 Hook 的目的。
	
3. Cydia Substrate

	Cydia Substrate 原名为 Mobile Substrate，它的主要作用是针对 OC 方法、C 函数以及函数地址进行 Hook 操作。当然它并不是仅仅针对 iOS 而设计的，安卓一样可以用。[官方地址](http://www.cydiasubstrate.com/)
	

## 一、Method Swizzle

在 OC 中，SEL 和 IMP 之间的关系，就好像一本书的“目录”。

SEL 是方法编号，就像“标题”一样，IMP 是方法实现的真实地址，就像“页码”一样。他们是一一对应的关系。

Runtime 提供了交换两个 SEL 和 IMP 对应关系的函数。

```
OBJC_EXPORT void
method_exchangeImplementations(Method _Nonnull m1, Method _Nonnull m2)
	OBJC_AVAILABLE(10.5, 2.0, 9.0 1.0, 2.0);
```

通过这个函数交换两个 SEL 和 IMP 对应关系的技术，称之为 Method Swizzle（方法欺骗）。

<center>
![](http://dzliving.com/iOSHook_0.png?imageView2/0/w/200)
</center>

使用场景：

1. 埋点
2. 防止程序崩溃
	
方法交换后要注意`死递归`现象。死递归最终导致栈溢出，因为每次递归方法，会将方法压栈。

	
## 二、Cydia Substrate

1. Mobile Hooker 

	顾名思义用于 Hook。它定义一系列的宏和函数，底层调用 objc 的 runtime 和 fishhook 来替换系统或者目标应用的函数。

	* MSHookMessageEx
		
		主要作用于 Objective-C 方法。
		
		```
		void MSHookMessageEx(Class class, SEL selector, IMP replacement, IMP result)
		```
		
	* MSHookFunction
	
		主要作用于 C 和 C++ 函数。
		
		```
		void MSHookFunction(void function, void * replacement, void** p_original)
		```
		
		Logos 语法 %hook 就是对此函数做了一层封装

2. MobileLoader

	MobileLoader 用于加载第三方 dylib 在运行的应用程序中。启动时 MobileLoader 会根据规则把指定目录的第三方的动态库加载进去，第三方的动态库也就是我们写的破解程序。
	
3. safe mode

	破解程序本质是 dylib，寄生在别的进程里。系统进程一旦出错，可能导致整个进程崩溃，崩溃后就会造成 iOS 瘫痪。所以 CydiaSubstrate 引入了安全模式，在安全模式下所有基于 CydiaSubstrate 的三方 dylib 都会被禁用，便于查错和修复。		

## 三、fishHook
	
#### 3.1 使用过程

```
- (void)viewDidLoad
{
    [super viewDidLoad];
    
    // 定义 rebinding 结构体
    struct rebinding nslog;
    nslog.name = "NSLog";
    nslog.replacement = myNSLog;
    nslog.replaced = (void **)&sys_nslog;
    
    // 定义数组
    struct rebinding rebs[1] = { nslog };
    
    
    /**  用来重新绑定符号
     
        参数 1：存放 rebinding 结构体的数组
        参数 2：数组的长度
       */
    rebind_symbols(rebs, 1);
}

// 函数指针，用来保存原始的函数地址
static void(* sys_nslog)(NSString * format, ...);


/// 定义一个新函数。
void myNSLog(NSString *format, ...)
{
    format = [format stringByAppendingString:@"被 hook 了！"];
    
    // 保留原始的调用
    sys_nslog(format);
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    NSLog(@"NSLog ");
}

NSLog 被 hook 了！
```


#### 3.2 fishhook 原理

[fishhook原理](https://www.jianshu.com/p/e5586ecddca4)

* PIC

	Position Independent Code 位置代码独立。
	
	为了使多个进程能够共享内存中的同一份代码拷贝，已达到节约内存资源的目的。

* ASLR

	Address Space Layout Randomization 地址空间布局随机化技术。
	
	它将进程的某些内存空间地址进行随机化来增大入侵者预测目的地址的难度，从而降低进程被成功入侵的风险。因而每次启动程序时，程序在内存中的地址都不一样。

* DYLD

	Dynamic Loader 动态加载器。
	
	在 Darwin/OS X 被叫做 dyld，它用来加载所有的 frameworks, dynamic libraries, and bundles (plug-ins)。

	由于苹果的动态库遵循 PIC，存在于共享缓存库，同时采用了 ASLR 技术。每次重启手机后，动态库的地址都会随机偏移，<font color=#cc0000>需要通过基地址加偏移地址来获取代码的真实地址</font>。

* MachO 文件

	MachO 文件是苹果系统的可执行文件。通过 DYLD 动态加载。
	
	我们的应用程序为了能够获取动态库的真实地址，由 DYLD 动态加载 MachO 中的符号表，将符号表中的指针指向动态库地址，从而达到调用系统库的目的。

fishhook 正是通过 dyld 来修改符号表中指向动态库的指针来达到 hook 的目的。

MachO 文件中，\_\_DATA 段中与动态符号绑定相关的有两个 section：`__nl_symbol_ptr` 和`__la_symbol_ptr`。

<center>
![](http://dzliving.com/iOSHook_2.png)
</center>

* \_\_nl\_symbol\_ptr 是一个非懒加载数据的指针数组。
* \_\_la\_symbol\_ptr 在第一次调用前，dyld 会去通过 dyld\_stub\_binder 填充指针数组。在第一次调用之后才会真正链接到动态库的位置。

<center>
![](http://dzliving.com/iOSHook_3.png)
</center>

为了在 MachO 中找到想要 hook 的函数的名字，需要在几个间接的层间进行跳转。在这两个 section 的 section header 提供了一个 `offset` 用来指向对应的 section。

<center>
![](http://dzliving.com/iOSHook_4.png)
</center>

而 \_\_la\_symbol\_ptr 指针数组的下标和 indirect\_symbol\_table 中的下标是一一对应的。

<center>
![](http://dzliving.com/iOSHook_5.png)
</center>

indirect\_symbol\_table 中的每个元素的 data 中保存着一个十六进制的数，这个数就是真正的符号表中的数组下标。比如 NSLog 这个函数的 Data 为 0x185，对应十进制 389，在symbol\_table 中的第 389 位即可找到。

<center>
![](http://dzliving.com/iOSHook_6.png)
</center>

而 symbol\_table 对应的元素中的 data 同样保存着一个十六进制的数，这个数是字符串表中的偏移地址。比如 NSLog 在 string\_table 中的偏移为 0x000000C1，找到 string\_table 的首地址，为 0x00009934，则该函数的字符串常量位于 0x9934 + 0xC1 = 0x99F5 的位置。

<center>
![](http://dzliving.com/iOSHook_7.png)
</center>

<center>
![](http://dzliving.com/iOSHook_1.png)
</center>
	
#### 3.3 验证方法交换过程

注意：因为 NSLog 是懒加载的，所以在第一次使用之前，是没有确定位置的。

所以上面代码改成:

```
- (void)viewDidLoad
{
    [super viewDidLoad];
    
    NSLog(@"First Use");
    
    // 使用结构体的简便创建方式
    struct rebinding rebs[1] = { { "NSLog", myNSLog, (void **)&sys_nslog } };
  
    rebind_symbols(rebs, 1);
}
```

添加断点在 

```
rebind_symbols(rebs, 1);
```

lldb 命令输出加载的模块的列表。第一个加载可执行文件，第二个加载 dyld。

```
(lldb) image list
[  0] 069E0DBD-6F91-39FC-BEDD-5463147B622A 0x000000010461d000 /Users/dubinbin/Library/Developer/Xcode/DerivedData/fishhookDmeo-bxkdwcgogbunzidzmmsmdpjedffq/Build/Products/Debug-iphonesimulator/fishhookDmeo.app/fishhookDmeo 
[  1] CE635DB2-D47E-3C05-A0A3-6BD982E7E750 0x000000010bc55000 /usr/lib/dyld 
[  2] C3514384-926E-3813-BF0C-69FFC704E283 0x000000010462e000 /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot/usr/lib/dyld_sim 
[  3] E5391C7B-0161-33AF-A5A7-1E18DBF9041F 0x0000000104917000 /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot/System/Library/Frameworks/Foundation.framework/Foundation 
[  4] 177A61B3-9E02-3A09-9A98-C1C3C9AB7958 0x0000000104f3d000 /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot/usr/lib/libobjc.A.dylib 
...
```

由上可知 MachO 的地址是 0x000000010461d000，MachO 文件中 NSLog 的偏移为 0x50A0。读取指针：

```
(lldb) x 0x000000010461d000+0x50A0
0x1046220a0: 76 d2 9b 04 01 00 00 00 70 8c 9a 04 01 00 00 00  v.......p.......
0x1046220b0: 91 b7 80 08 01 00 00 00 72 05 62 04 01 00 00 00  ........r.b.....
```

查看汇编（注意：iOS 是小端序，地址从右往左读取，即为 0x00000001049bd276）：

```
(lldb) dis -s 0x01049bd276
Foundation`NSLog:
    0x1049bd276 <+0>:  pushq  %rbp
    0x1049bd277 <+1>:  movq   %rsp, %rbp
    0x1049bd27a <+4>:  subq   $0xd0, %rsp
    0x1049bd281 <+11>: testb  %al, %al
    0x1049bd283 <+13>: je     0x1049bd2ab               ; <+53>
    0x1049bd285 <+15>: movaps %xmm0, -0xa0(%rbp)
    0x1049bd28c <+22>: movaps %xmm1, -0x90(%rbp)
(lldb) p sys_nslog
(void (*)(NSString *, ...)) $1 = 0x0000000000000000
```

如果没有先执行一条 NSLog，这里输出是:

```
(lldb) dis -s 0x01049bd276
    0x10d5c4460: pushq  $0x1e
    0x10d5c4465: jmp    0x10d5c4450
    0x10d5c446a: pushq  $0x2c
    0x10d5c446f: jmp    0x10d5c4450
    0x10d5c4474: pushq  $0x152                    ; imm = 0x152 
    0x10d5c4479: jmp    0x10d5c4450
```

这个位置还不是 NSlog。

替换方法实现后，重新读取 MachO 相同偏移位置的指针内容：

```
(lldb) x 0x000000010461d000+0x50A0
0x1046220a0: 50 de 61 04 01 00 00 00 70 8c 9a 04 01 00 00 00  P.a.....p.......
0x1046220b0: 91 b7 80 08 01 00 00 00 2b 51 42 07 01 00 00 00  ........+QB.....
(lldb) dis -s 0x010461de50
fishhookDmeo`myNSLog:
    0x10461de50 <+0>:  pushq  %rbp
    0x10461de51 <+1>:  movq   %rsp, %rbp
    0x10461de54 <+4>:  subq   $0x30, %rsp
    0x10461de58 <+8>:  movq   $0x0, -0x8(%rbp)
    0x10461de60 <+16>: leaq   -0x8(%rbp), %rax
    0x10461de64 <+20>: movq   %rdi, -0x10(%rbp)
    0x10461de68 <+24>: movq   %rax, %rdi
    0x10461de6b <+27>: movq   -0x10(%rbp), %rsi
(lldb) p sys_nslog
(void (*)(NSString *, ...)) $2 = 0x000000011c72780a
```

可以看到 sys_nslog 保存了交换前 NSLog 的地址

#### 3.4  总结

fishhook 正是通过查找到需要 hook 的函数指针，然后替换成自己写的函数指针，从而实现 hook 系统函数的目的。

fishhook 不能 hook 自定义的函数，是因为自定义的函数不在 \_\_la\_symbol\_ptr 中。


## 文章

[【逻辑教育】iOS逆向开发之HOOK原理](https://www.bilibili.com/video/av34657491)