---
title: dyld
categories: iOS原理
---

## 一、介绍

在 MacOS 和 iOS 上，可执行程序的启动依赖于 xnu 内核进程运作和动态链接加载器 dyld。

> dyld 全称 the dynamic link editor，即动态链接器，其本质是 Mach-O 文件，是专门用来加载动态库的库。

源码下载地址：[https://opensource.apple.com/tarballs/dyld/](https://opensource.apple.com/tarballs/dyld/)

dyld 会将 App 依赖的动态库和 App 文件加载到内存以后执行，动态库不是可执行文件，无法独自执行。当点击 App 的时候，系统在内核态完成一些必要配置，从 App 的 MachO 文件解析出 dyld 的地址，这里会记录在 MachO 的 LC_LOAD_DYLINKER 命令中，内容参考如下：

```
          cmd LC_LOAD_DYLINKER
      cmdsize 28
         name /usr/lib/dyld (offset 12)
Load command 8
     cmd LC_UUID
 cmdsize 24
    uuid DF0F9B2D-A4D7-37D0-BC6B-DB0297766CE8
Load command 9
      cmd LC_VERSION_MIN_IPHONEOS
```

dyld 位于 ``/usr/lib/dyld``，可以从越狱机或者 mac 电脑中找到。以 mac 为例，终端执行命令：

```
$ cd /usr/lib
$ file dyld
```

<center>
![](http://dzliving.com/dyld_path.png)
</center>

dyld 是 <font color=#cc0000>Mach-O 类型的通用二进制文件</font>，支持 x86_64 和 i386 两种架构。iPhone 真机对应的 dyld 支持的为 arm 系列架构。


## 二、otool

> otool 是专门用来查看 Mach-O 类型文件的工具

Mac OS X 下二进制可执行文件的动态链接库是 ``dylib`` 文件。

> dylib 也就是 bsd 风格的动态库。基本可以认为等价于 windows 的 dll 和 linux 的so。mac 基于 bsd，所以也使用的是 dylib。

Linux 下用 ldd 查看，苹果系统用 otool。

#### 2.1 查看 otool 地址

电脑已安装 Xcode。终端输入：

```
$ otool
Usage: /Applications/Xcode10.1.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/otool [-arch arch_type] [-fahlLDtdorSTMRIHGvVcXmqQjCP] [-mcpu=arg] [--version] <object file> ...
	-f print the fat headers
	-a print the archive header
	-h print the mach header
	-l print the load commands
	-L print shared libraries used
	-D print shared library id name
	-t print the text section (disassemble with -v)
	-p <routine name>  start dissassemble from routine name
	-s <segname> <sectname> print contents of section
	-d print the data section
	-o print the Objective-C segment
	-r print the relocation entries
	-S print the table of contents of a library (obsolete)
	-T print the table of contents of a dynamic shared library (obsolete)
	-M print the module table of a dynamic shared library (obsolete)
	-R print the reference table of a dynamic shared library (obsolete)
	-I print the indirect symbol table
	-H print the two-level hints table (obsolete)
	-G print the data in code table
	-v print verbosely (symbolically) when possible
	-V print disassembled operands symbolically
	-c print argument strings of a core file
	-X print no leading addresses or headers
	-m don't use archive(member) syntax
	-B force Thumb disassembly (ARM objects only)
	-q use llvm's disassembler (the default)
	-Q use otool(1)'s disassembler
	-mcpu=arg use `arg' as the cpu for disassembly
	-j print opcode bytes
	-P print the info plist section as strings
	-C print linker optimization hints
	--version print the version of /Applications/Xcode10.1.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/otool
```

由上可知 otool 的地址：<font color=#cc0000>``/Applications/Xcode10.1.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/otool``</font>

进入地址发现 otool 文件是一个替身（软连接）。

查看 otool 指向的软连接地址：

```
$ cd /Applications/Xcode10.1.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/
$
$ ls -l
total 223352
-r-xr-xr-x  1 cykj  staff     33920 10 20  2018 ar
-r-xr-xr-x  1 cykj  staff     28000 10 20  2018 as
-rwxr-xr-x  1 cykj  staff     18176 10 20  2018 asa
-rwxr-xr-x  1 cykj  staff    212208 10 20  2018 bison
-r-xr-xr-x  1 cykj  staff    150048 10 20  2018 bitcode_strip
lrwxr-xr-x  1 cykj  staff         5 11 22  2018 c++ -> clang
-rwxr-xr-x  1 cykj  staff     23152 10 20  2018 c89
-rwxr-xr-x  1 cykj  staff     23248 10 20  2018 c99
lrwxr-xr-x  1 cykj  staff         5 11 22  2018 cc -> clang
-rwxr-xr-x  1 cykj  staff  78705232 10 20  2018 clang
lrwxr-xr-x  1 cykj  staff         5 11 22  2018 clang++ -> clang
-r-xr-xr-x  1 cykj  staff    120064 10 20  2018 cmpdylib
-r-xr-xr-x  1 cykj  staff    145872 10 20  2018 codesign_allocate
lrwxr-xr-x  1 cykj  staff        17 11 22  2018 codesign_allocate-p -> codesign_allocate
-rwxr-xr-x  1 cykj  staff   4937600 10 20  2018 coremlcompiler
-rwxr-xr-x  1 cykj  staff      3344  9 26  2018 cpp
-rwxr-xr-x  1 cykj  staff     27712 10 20  2018 ctags
-r-xr-xr-x  1 cykj  staff    145824 10 20  2018 ctf_insert
lrwxr-xr-x  1 cykj  staff        13 11 22  2018 dsymutil -> llvm-dsymutil
-rwxr-xr-x  1 cykj  staff   1006032 10 20  2018 dwarfdump
-rwxr-xr-x  1 cykj  staff    219088 10 20  2018 dyldinfo
-rwxr-xr-x  2 cykj  staff    569056 10 20  2018 flex
-rwxr-xr-x  2 cykj  staff    569056 10 20  2018 flex++
lrwxr-xr-x  1 cykj  staff         8 11 22  2018 gcov -> llvm-cov
-rwxr-xr-x  2 cykj  staff    142336 10 20  2018 gm4
-rwxr-xr-x  1 cykj  staff     90960 10 20  2018 gperf
-rwxr-xr-x  1 cykj  staff     65520 10 20  2018 indent
-r-xr-xr-x  1 cykj  staff    136784 10 20  2018 install_name_tool
-rwxr-xr-x  1 cykj  staff   2480704 10 20  2018 ld
-rwxr-xr-x  1 cykj  staff       230  9 26  2018 lex
-r-xr-xr-x  1 cykj  staff    154592 10 20  2018 libtool
-r-xr-xr-x  1 cykj  staff     66000 10 20  2018 lipo
-rwxr-xr-x  1 cykj  staff   3320816 10 20  2018 llvm-cov
-rwxr-xr-x  1 cykj  staff  29723968 10 20  2018 llvm-dsymutil
-rwxr-xr-x  1 cykj  staff  10591472 10 20  2018 llvm-nm
-rwxr-xr-x  1 cykj  staff  11899296 10 20  2018 llvm-objdump
-r-xr-xr-x  1 cykj  staff     32672 10 20  2018 llvm-otool
-rwxr-xr-x  1 cykj  staff   1272096 10 20  2018 llvm-profdata
-rwxr-xr-x  1 cykj  staff   2873440 10 20  2018 llvm-size
-rwxr-xr-x  1 cykj  staff      3567  9 26  2018 lorder
-rwxr-xr-x  2 cykj  staff    142336 10 20  2018 m4
-rwxr-xr-x  1 cykj  staff     24800 10 20  2018 metal
-rwxr-xr-x  1 cykj  staff     24768 10 20  2018 metal-ar
-rwxr-xr-x  1 cykj  staff     24768 10 20  2018 metal-as
-rwxr-xr-x  1 cykj  staff     24768 10 20  2018 metal-link
-rwxr-xr-x  1 cykj  staff     24768 10 20  2018 metal-opt
-rwxr-xr-x  1 cykj  staff     24768 10 20  2018 metallib
-rwxr-xr-x  1 cykj  staff      7604  9 26  2018 mig
lrwxr-xr-x  1 cykj  staff         7 11 22  2018 nm -> llvm-nm
-r-xr-xr-x  1 cykj  staff    132896 10 20  2018 nm-classic
-r-xr-xr-x  1 cykj  staff    162720 10 20  2018 nmedit
lrwxr-xr-x  1 cykj  staff        12 11 22  2018 objdump -> llvm-objdump
lrwxr-xr-x  1 cykj  staff        10 11 22  2018 otool -> llvm-otool
-r-xr-xr-x  1 cykj  staff    648720 10 20  2018 otool-classic
-r-xr-xr-x  1 cykj  staff    132784 10 20  2018 pagestuff
lrwxr-xr-x  1 cykj  staff         7 11 22  2018 ranlib -> libtool
-rwxr-xr-x  1 cykj  staff     59344 10 20  2018 rebase
-r-xr-xr-x  1 cykj  staff    204960 10 20  2018 redo_prebinding
-rwxr-xr-x  1 cykj  staff     73664 10 20  2018 rpcgen
-r-xr-xr-x  1 cykj  staff     48864 10 20  2018 segedit
lrwxr-xr-x  1 cykj  staff         9 11 22  2018 size -> llvm-size
-r-xr-xr-x  1 cykj  staff    120080 10 20  2018 size-classic
-r-xr-xr-x  1 cykj  staff    120400 10 20  2018 strings
-r-xr-xr-x  1 cykj  staff    189568 10 20  2018 strip
-rwxr-xr-x  1 cykj  staff  87671328 10 20  2018 swift
lrwxr-xr-x  1 cykj  staff         5 11 22  2018 swift-autolink-extract -> swift
-rwxr-xr-x  1 cykj  staff   5031520 10 20  2018 swift-build
-rwxr-xr-x  1 cykj  staff    384480 10 20  2018 swift-build-tool
-rwxr-xr-x  1 cykj  staff    461136 10 20  2018 swift-demangle
-rwxr-xr-x  1 cykj  staff   5031552 10 20  2018 swift-package
-rwxr-xr-x  1 cykj  staff   5031472 10 20  2018 swift-run
-rwxr-xr-x  1 cykj  staff     53024 10 20  2018 swift-stdlib-tool
-rwxr-xr-x  1 cykj  staff   5031504 10 20  2018 swift-test
lrwxr-xr-x  1 cykj  staff         5 11 22  2018 swiftc -> swift
-rwxr-xr-x  1 cykj  staff  12042320 10 20  2018 tapi
-rwxr-xr-x  1 cykj  staff     32592 10 20  2018 unifdef
-rwxr-xr-x  1 cykj  staff      2946  9 26  2018 unifdefall
-rwxr-xr-x  1 cykj  staff     59776 10 20  2018 unwinddump
-rwxr-xr-x  1 cykj  staff       135  9 26  2018 yacc
```

可以看到 otool 指向 <font color=#cc0000>llvm-otool</font>，而 llvm-otool 和 otool 在同一个目录中。

另外，还可以发现，这个文件夹下面还有很多有用的文件，如 ``lipo``。

#### 2.2 otool -L

> 查看动态链接库

终端执行命令：

```
$ cd /Users/cykj/Library/Developer/Xcode/DerivedData/Demo-fpfdxjbemnwnqcfjimbqpbzpnpem/Build/Products/Debug-iphonesimulator/Demo.app/
$
$ otool -L Demo
Demo:
	/System/Library/Frameworks/Foundation.framework/Foundation (compatibility version 300.0.0, current version 1560.10.0)
	/usr/lib/libobjc.A.dylib (compatibility version 1.0.0, current version 228.0.0)
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1252.200.5)
	/System/Library/Frameworks/CoreFoundation.framework/CoreFoundation (compatibility version 150.0.0, current version 1560.10.0)
	/System/Library/Frameworks/UIKit.framework/UIKit (compatibility version 1.0.0, current version 61000.0.0)
```

查看动态库的依赖库：

```
$ otool -L /usr/lib/system/libdispatch.dylib
/usr/lib/system/libdispatch.dylib:
	/usr/lib/system/libdispatch.dylib (compatibility version 1.0.0, current version 913.60.3)
	/usr/lib/system/libdyld.dylib (compatibility version 1.0.0, current version 551.4.0)
	/usr/lib/system/libcompiler_rt.dylib (compatibility version 1.0.0, current version 62.0.0)
	/usr/lib/system/libsystem_kernel.dylib (compatibility version 1.0.0, current version 4570.71.8)
	/usr/lib/system/libsystem_platform.dylib (compatibility version 1.0.0, current version 161.50.1)
	/usr/lib/system/libsystem_pthread.dylib (compatibility version 1.0.0, current version 301.50.1)
	/usr/lib/system/libsystem_malloc.dylib (compatibility version 1.0.0, current version 140.50.6)
	/usr/lib/system/libsystem_c.dylib (compatibility version 1.0.0, current version 1244.50.9)
	/usr/lib/system/libsystem_blocks.dylib (compatibility version 1.0.0, current version 67.0.0)
	/usr/lib/system/libunwind.dylib (compatibility version 1.0.0, current version 35.3.0)
	/usr/lib/libobjc.A.dylib (compatibility version 1.0.0, current version 228.0.0)
```

#### 2.3 otool -ov

> 显示 Objective-C 类结构及其定义的方法。

终端执行命令：

```
$ otool -ov Demo
Demo:
Contents of (__DATA,__objc_classlist) section
00000001000041f0 0x100005080 _OBJC_CLASS_$_HookTool
           isa 0x1000050a8 _OBJC_METACLASS_$_HookTool
    superclass 0x0 _OBJC_CLASS_$_NSObject
         cache 0x0 __objc_empty_cache
        vtable 0x0
          data 0x100004328 (struct class_ro_t *)
                    flags 0x80
            instanceStart 8
             instanceSize 8
                 reserved 0x0
               ivarLayout 0x0
                     name 0x100003555 HookTool
              baseMethods 0x1000042f0 (struct method_list_t *)
		   entsize 24
		     count 2
		      name 0x1000028b3 swizzle_decodeObjectForKey:
		     types 0x1000035c4 @24@0:8@16
		       imp 0x1000015f0 -[HookTool swizzle_decodeObjectForKey:]
		      name 0x100002914 swizzle_button_initWithCoder:
		     types 0x1000035c4 @24@0:8@16
		       imp 0x1000017c0 -[HookTool swizzle_button_initWithCoder:]
            baseProtocols 0x0
                    ivars 0x0
           weakIvarLayout 0x0
           baseProperties 0x0
Meta Class
	...
	
```

#### 2.4 otool -tV [Mach-O]

> 查看 ARM 汇编码

```
$ otool -tV Demo
Demo:
(__TEXT,__text) section
+[HookTool load]:
0000000100001400	pushq	%rbp
0000000100001401	movq	%rsp, %rbp
0000000100001404	subq	$0x40, %rsp
0000000100001408	movl	$0x2, %eax
000000010000140d	movl	%eax, %edx
000000010000140f	movq	%rdi, -0x8(%rbp)
0000000100001413	movq	%rsi, -0x10(%rbp)
0000000100001417	movq	0x3c1a(%rip), %rsi ## Objc class ref: _OBJC_CLASS_$_NSMutableArray
000000010000141e	movq	0x3b33(%rip), %rdi ## Objc selector ref: arrayWithCapacity:
0000000100001425	movq	%rdi, -0x20(%rbp)
0000000100001429	movq	%rsi, %rdi
000000010000142c	movq	-0x20(%rbp), %rsi
0000000100001430	callq	*0x2bf2(%rip) ## Objc message: +[NSMutableArray arrayWithCapacity:]
0000000100001436	movq	%rax, %rdi
0000000100001439	callq	0x10000265a ## symbol stub for: _objc_retainAutoreleasedReturnValue
000000010000143e	movq	__imageViewImageArray(%rip), %rdx
0000000100001445	movq	%rax, __imageViewImageArray(%rip)
000000010000144c	movq	%rdx, %rdi
000000010000144f	callq	*0x2bdb(%rip) ## literal pool symbol address: _objc_release
0000000100001455	leaq	0x2cb4(%rip), %rax ## Objc cfstring ref: @"emaNecruoseRIU"
000000010000145c	movq	0x3bdd(%rip), %rdx ## Objc class ref: HookTool
0000000100001463	movq	0x3af6(%rip), %rsi ## Objc selector ref: stringByReversed:
000000010000146a	movq	%rdx, %rdi

	...
	
```

#### 2.5 otool -h [Mach-O]

> 查看 Mach-O 头结构等

```
$ otool -h Demo
Mach header
      magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
 0xfeedfacf 16777223          3  0x00           2    21       3272 0x00200085
```

一个 Mach-O 的文件头结构为：

<center>
![](http://dzliving.com/MachOHeader.png)
</center>

各字段的含义，可参看 ``/usr/include/mach-o/loader.h``。

#### 2.6 otool -l [Mach-O] | grep crypt1

> 查看 ipa 包是否加壳

```
$ otool -l Demo | grep crypt1
$
```

没有进行过加壳处理。

```
     cryptoff 16384
    cryptsize 6651904
      cryptid 0
     cryptoff 16384
    cryptsize 6553600
      cryptid 0123456
```

cryptid 代表是否加壳，1 - 加壳，0 - 已脱壳。

上面打印了两遍，其实代表着该可执行文件支持两种架构 armv7 和 arm64。

Mach-O 文件可以用 GUI 图形软件 [MachOView](https://github.com/gdbinit/MachOView) 更加直观的查看相关信息。

<center>
![](http://dzliving.com/MachOView.png)
</center>


## 三、dyld加载

> 动态库链接、load 方法执行都是在 main 函数执行之前的。

如图所示进行操作：

<center>
![](http://dzliving.com/SymbolicBreakPoint.png)
![](http://dzliving.com/NSObjectLoad.png)
![](http://dzliving.com/ThreadStatck.png)
</center>

由上可知，load 的加载是从 ``__dyld_start`` 这个函数开始的。

#### 3.1 \_\_dyld\_start

系统内核在加载动态库前，会加载 dyld，然后调用去执行 \_\_dyld\_start（汇编语言实现）。该函数会执行 dyldbootstrap::start()，后者会执行 \_main()函数，dyld 的加载动态库的代码就是从\_main()开始执行的。这里可以查看 dyldStartup.s的部分内容（以x86\_x64架构做参考)，其中标出了 \_dyld\_start() 与 dyldbootstrap 的 start 方法。

```c
#if __x86_64__
#if !TARGET_IPHONE_SIMULATOR
	.data
	.align 3
__dyld_start_static:
	.quad   __dyld_start
#endif


#if !TARGET_IPHONE_SIMULATOR
	.text
	.align 2,0x90
	.globl __dyld_start
__dyld_start:
	popq	%rdi		# param1 = mh of app
	pushq	$0		# push a zero for debugger end of frames marker
	movq	%rsp,%rbp	# pointer to base of kernel frame
	andq    $-16,%rsp       # force SSE alignment
	subq	$16,%rsp	# room for local variables

	# call dyldbootstrap::start(app_mh, argc, argv, slide, dyld_mh, &startGlue)
	movl	8(%rbp),%esi	# param2 = argc into %esi
	leaq	16(%rbp),%rdx	# param3 = &argv[0] into %rdx
	movq	__dyld_start_static(%rip), %r8
	leaq	__dyld_start(%rip), %rcx
	subq	 %r8, %rcx	# param4 = slide into %rcx
	leaq	___dso_handle(%rip),%r8 # param5 = dyldsMachHeader
	leaq	-8(%rbp),%r9
	call	__ZN13dyldbootstrap5startEPK12macho_headeriPPKclS2_Pm
	movq	-8(%rbp),%rdi
	cmpq	$0,%rdi
	jne	Lnew

    	# clean up stack and jump to "start" in main executable
	movq	%rbp,%rsp	# restore the unaligned stack pointer
	addq	$8,%rsp 	# remove the mh argument, and debugger end frame marker
	movq	$0,%rbp		# restore ebp back to zero
	jmp	*%rax		# jump to the entry point

	# LC_MAIN case, set up stack for call to main()
```


#### 3.2 dyldInitialization.cpp

\_\_dyld\_start 内部调用 dyldbootstrap::start，位于 dyldInitialization.cpp。

```c++
//
//  This is code to bootstrap dyld.  This work in normally done for a program by dyld and crt.
//  In dyld we have to do this manually.
//
uintptr_t start(const struct macho_header* appsMachHeader, int argc, const char* argv[], 
				intptr_t slide, const struct macho_header* dyldsMachHeader,
				uintptr_t* startGlue)
{
	// if kernel had to slide dyld, we need to fix up load sensitive locations
	// we have to do this before using any global variables
    // ①、获取 dyld 对应的 slide
    slide = slideOfMainExecutable(dyldsMachHeader);
    bool shouldRebase = slide != 0;
#if __has_feature(ptrauth_calls)
    shouldRebase = true;
#endif
    if ( shouldRebase ) {
        // ②、通过 slide 对 dyld 进行 rebase
        rebaseDyld(dyldsMachHeader, slide);
    }

	// allow dyld to use mach messaging
    // ③、mach 初始化
	mach_init();

	// kernel sets up env pointer to be just past end of agv array
	const char** envp = &argv[argc+1];
	
	// kernel sets up apple pointer to be just past end of envp array
	const char** apple = envp;
    // ④、栈溢出保护
	while(*apple != NULL) { ++apple; }
	++apple;

	// set up random value for stack canary
	__guard_setup(apple);

#if DYLD_INITIALIZER_SUPPORT
	// run all C++ initializers inside dyld
	runDyldInitializers(dyldsMachHeader, slide, argc, argv, envp, apple);
#endif

	// now that we are done bootstrapping dyld, call dyld's main
    // ⑤、获取应用的 slide（appsSlide）
	uintptr_t appsSlide = slideOfMainExecutable(appsMachHeader);
    
    // ⑥、调用 dyld 的 main 函数
	return dyld::_main(appsMachHeader, appsSlide, argc, argv, envp, apple, startGlue);
}
```

#### 3.3 slide、rebase

由于 apple 采用了 [ASLR（Address space layout randomization）](https://baike.baidu.com/item/aslr/5779647?fr=aladdin)技术，所以 Mach-O 每次加载到内存中的<font color=#cc0000>首地址是变化的</font>，此时想找到代码在内存中对应的地址需要重定位 rebase。rebase 要用到 slide 值：

```
//
//  The kernel may have slid a Position Independent Executable
//
static uintptr_t slideOfMainExecutable(const struct macho_header* mh)
{
    // Mach-O 文件中 load commands 数量
	const uint32_t cmd_count = mh->ncmds;
    
    // 偏移地址到 load commands 的首地址
	const struct load_command* const cmds = (struct load_command*)(((char*)mh)+sizeof(macho_header));
	const struct load_command* cmd = cmds;
    
    
	for (uint32_t i = 0; i < cmd_count; ++i) {
        // 选中 cmd = LC_SEGMENT_COMMAND
		if ( cmd->cmd == LC_SEGMENT_COMMAND ) {
			const struct macho_segment_command* segCmd = (struct macho_segment_command*)cmd;
            // 实际对应 LC_SEGMENT_COMMAND(_TEXT)
			if ( (segCmd->fileoff == 0) && (segCmd->filesize != 0)) {
                
                // Mach-O 文件首地址 - LC_SEGMENT_COMMAND(_TEXT).vmaddr
				return (uintptr_t)mh - segCmd->vmaddr;
			}
		}
        // 偏移 command 指针
		cmd = (const struct load_command*)(((char*)cmd)+cmd->cmdsize);
	}
	return 0;
}
```

<center>
![](http://dzliving.com/MachOTEXT.png)
</center>

应用本身的 Mach-O 及 dyld 采用的是 ``slideOfMainExecutable`` 的方式获取 slide。从上代码得知：side = Mach-O header 首地址 - Load Commands 中 \_\_TEXT 段的 VM Address 的值。

```c++
intptr_t _dyld_get_image_slide(const mach_header* mh)
{
    log_apis("_dyld_get_image_slide(%p)\n", mh);

    // 获取 Mach-O 文件加载对象
    const MachOLoaded* mf = (MachOLoaded*)mh;
    
    // 如果 mach 文件头没有 magic 值
    if ( !mf->hasMachOMagic() )
        return 0;

    // 调用 MachOLoaded::getSlide() 方法
    return mf->getSlide();
}

intptr_t _dyld_get_image_vmaddr_slide(uint32_t imageIndex)
{
    log_apis("_dyld_get_image_vmaddr_slide(%d)\n", imageIndex);

    // 获取到 Mach-O 文件
    const mach_header* mh = gAllImages.imageLoadAddressByIndex(imageIndex);
    if ( mh != nullptr )
        // 调用上面的方法
        return dyld3::_dyld_get_image_slide(mh);
    return 0;
}
```

```c++
intptr_t MachOLoaded::getSlide() const
{
    // 诊断对象。
    Diagnostics diag;
    __block intptr_t slide = 0;
    
    // 循环 load command
    forEachLoadCommand(diag, ^(const load_command* cmd, bool& stop) {
        
        // 64 位
        if ( cmd->cmd == LC_SEGMENT_64 ) {
            const segment_command_64* seg = (segment_command_64*)cmd;
            
            // LC_SEGMENT_64(__TEXT)
            if ( strcmp(seg->segname, "__TEXT") == 0 ) {
                // mach-O 首地址 - LC_SEGMENT_64(__TEXT).vmaddr
                slide = (uintptr_t)(((uint64_t)this) - seg->vmaddr);
                stop = true;
            }
        }
        // 32 位
        else if ( cmd->cmd == LC_SEGMENT ) {
            const segment_command* seg = (segment_command*)cmd;
            
            // LC_SEGMENT(__TEXT)
            if ( strcmp(seg->segname, "__TEXT") == 0 ) {
                // mach-O 首地址 - LC_SEGMENT(__TEXT).vmaddr
                slide = (uintptr_t)(((uint64_t)this) - seg->vmaddr);
                stop = true;
            }
        }
    });
    diag.assertNoError();   // any malformations in the file should have been caught by earlier validate() call
    return slide;
}
```

动态库加载采用的是 ``\_dyld\_get\_image\_vmaddr\_slide`` 的方式获取 slide。

简单验证一下，以应用 Mach-O 为例：

1. Load Commands \_\_TEXT 段 VM Address 值。

	<center>
	![](http://dzliving.com/MachOVMAddress.png)
	</cetner>

	VM Address 的地址为 4294967296（10进制）。

2. 在 Demo 项目中 ViewController.m ``viewDidLoad`` 方法设置断点，触发后，在 lldb 执行 ``image list``

	<center>
	![](http://dzliving.com/ImageList.png)
	</center>

	应用 Mach-O 的地址为 0x00000001004f8000（16进制）。

3. 计算 viewDidLoad 在应用 Mach-O 文件中的地址，``symbol address = stack address - slide``。

	<center>
	![](http://dzliving.com/LLVMAddress.png)
	</center>

	①、用 Mach-O 的 VM Address 减去对应虚拟地址，得到的 5210112（10进制）为 slide 值；
	②、获取 viewDidLoad 函数在当前<font color=#cc0000>内存</font>中的地址；
	③、用 viewDidLoad 内存地址减去 slide 得到它在 Mach-O 中对应的虚拟地址；
	④、将 10 进制转化为 16 进制。

	计算得到地址：0x00000001000022c0

4. 在 Mach-O 文件中查看。

	<center>
	![](http://dzliving.com/ViewDidLoadAddress.png)
	</center>

	可以看到，通过计算得出的值 0x100001750 与 Mach-O 中看到的值一致。

当然，也可以通过命令行直接获取 slide 的值。


#### 3.4 dyld::\_main

对 ASLR 有了基本认知后，接着看看位于 ``dyld.cpp`` 中的 _main 干了什么。

##### 3.4.1 设置运行环境

```c++
//
// Entry point for dyld.  The kernel loads dyld and jumps to __dyld_start which
// sets up some registers and call this function.
//
// Returns address of main() in target program which __dyld_start jumps to
//
uintptr_t
_main(const macho_header* mainExecutableMH, uintptr_t mainExecutableSlide, 
		int argc, const char* argv[], const char* envp[], const char* apple[], 
		uintptr_t* startGlue)
{
	if (dyld3::kdebug_trace_dyld_enabled(DBG_DYLD_TIMING_LAUNCH_EXECUTABLE)) {
		launchTraceID = dyld3::kdebug_trace_dyld_duration_start(DBG_DYLD_TIMING_LAUNCH_EXECUTABLE, (uint64_t)mainExecutableMH, 0, 0);
	}
	
    // Grab the cdHash of the main executable from the environment
	uint8_t mainExecutableCDHashBuffer[20];
	const uint8_t* mainExecutableCDHash = nullptr;
	if ( hexToBytes(_simple_getenv(apple, "executable_cdhash"), 40, mainExecutableCDHashBuffer) )
		// 获取主程序 hash
		mainExecutableCDHash = mainExecutableCDHashBuffer;
	
	// Trace dyld's load
	// 告知 kernel，dyld 已加载
	notifyKernelAboutImage((macho_header*)&__dso_handle, _simple_getenv(apple, "dyld_file"));
#if !TARGET_IPHONE_SIMULATOR
	// Trace the main executable's load
	// 告知 kernel，主程序 Mach-O 已加载
	notifyKernelAboutImage(mainExecutableMH, _simple_getenv(apple, "executable_file"));
#endif
	
	uintptr_t result = 0;
	// 赋值参数。
	// mach_header 类型结构体，表示当前 App 的 Mach-O头部信息。有了头部信息，加载器就可以从头开始，遍历整个 Mach-O 文件的信息。
	sMainExecutableMachHeader = mainExecutableMH;
	// long 类型数据，表示 ASLR 位移长度
	sMainExecutableSlide = mainExecutableSlide;
#if __MAC_OS_X_VERSION_MIN_REQUIRED
	// if this is host dyld, check to see if iOS simulator is being run
	// 获取 dyld 路径
	const char* rootPath = _simple_getenv(envp, "DYLD_ROOT_PATH");
	if ( (rootPath != NULL) ) {
		// look to see if simulator has its own dyld
		char simDyldPath[PATH_MAX]; 
		strlcpy(simDyldPath, rootPath, PATH_MAX);
		strlcat(simDyldPath, "/usr/lib/dyld_sim", PATH_MAX);
		// 打开 dyld_sim 路径
		int fd = my_open(simDyldPath, O_RDONLY, 0);
		
		// 成功
		if ( fd != -1 ) {
			// 如果是模拟器，并且正确加载`dyld_sim`，则直接返回主程序地址
			const char* errMessage = useSimulatorDyld(fd, mainExecutableMH, simDyldPath, argc, argv, envp, apple, startGlue, &result);
			if ( errMessage != NULL )
				halt(errMessage);
			return result;
		}
	}
#endif
	
	CRSetCrashLogMessage("dyld: launch started");
	
	// 设置一个全局链接上下文，包括一些回调函数、参数与标志设置信息，其中的 context 结构体实例、回调函数都是 dyld 自己的实现
	setContext(mainExecutableMH, argc, argv, envp, apple);
	
	// Pickup the pointer to the exec path.
	// 获取主程序路径
	sExecPath = _simple_getenv(apple, "executable_path");
	
	// <rdar://problem/13868260> Remove interim apple[0] transition code from dyld
	if (!sExecPath) sExecPath = apple[0];
	
	// 获取应用 Mach-O 文件的绝对路径
	if ( sExecPath[0] != '/' ) {
		// have relative path, use cwd to make absolute
		char cwdbuff[MAXPATHLEN];
	    if ( getcwd(cwdbuff, MAXPATHLEN) != NULL ) {
			// maybe use static buffer to avoid calling malloc so early...
			char* s = new char[strlen(cwdbuff) + strlen(sExecPath) + 2];
			strcpy(s, cwdbuff);
			strcat(s, "/");
			strcat(s, sExecPath);
			sExecPath = s;
		}
	}
	
	// Remember short name of process for later logging
	// 设置进程名称
	sExecShortName = ::strrchr(sExecPath, '/');
	if ( sExecShortName != NULL )
		++sExecShortName;
	else
		sExecShortName = sExecPath;
	
	// 配置进程受限模式。根据当前进程是否受限，再次配置链接上下文以及其他环境参数
	configureProcessRestrictions(mainExecutableMH);
	
	// 再次检测/设置上下文环境
#if __MAC_OS_X_VERSION_MIN_REQUIRED
    if ( !gLinkContext.allowEnvVarsPrint && !gLinkContext.allowEnvVarsPath && !gLinkContext.allowEnvVarsSharedCache ) {
		pruneEnvironmentVariables(envp, &apple);
		// set again because envp and apple may have changed or moved
		setContext(mainExecutableMH, argc, argv, envp, apple);
	}
	else
#endif
	{
		checkEnvironmentVariables(envp);
		defaultUninitializedFallbackPaths(envp);
	}
#if __MAC_OS_X_VERSION_MIN_REQUIRED
	if (  ((dyld3::MachOFile*)mainExecutableMH)->supportsPlatform(dyld3::Platform::iOSMac)
	  && !((dyld3::MachOFile*)mainExecutableMH)->supportsPlatform(dyld3::Platform::macOS)) {
		gLinkContext.rootPaths = parseColonList("/System/iOSSupport", NULL);
		gLinkContext.marzipan = true;
		if ( sEnv.DYLD_FALLBACK_LIBRARY_PATH == sLibraryFallbackPaths )
			sEnv.DYLD_FALLBACK_LIBRARY_PATH = sRestrictedLibraryFallbackPaths;
		if ( sEnv.DYLD_FALLBACK_FRAMEWORK_PATH == sFrameworkFallbackPaths )
			sEnv.DYLD_FALLBACK_FRAMEWORK_PATH = sRestrictedFrameworkFallbackPaths;
	}
#endif
	
	// 如果设置了DYLD_PRINT_OPTS，则打印参数
	if ( sEnv.DYLD_PRINT_OPTS )
		printOptions(argv);
	
	// 如果设置了DYLD_PRINT_ENV，则打印环境变量
	if ( sEnv.DYLD_PRINT_ENV ) 
		printEnvironmentVariables(envp);
	
	// 获取主程序架构信息
	getHostInfo(mainExecutableMH, mainExecutableSlide);
	
	...
```

从源码可以看到，在模拟器运行程序时，通过 ``dyld_sim`` 来进行后续加载工作的，与正常真机加载流程略有不同。

模拟器：
	
<center>
![](http://dzliving.com/DyldMainSim.png)
</center>
	
真机：
	
<center>
![](http://dzliving.com/DyldMainPhone.png)
</center>

具体实现在 ``useSimulatorDyld`` 这个函数中，本文不做进一步解析。

这里还有一个知识点，环境变量 <font color=#cc0000>``DYLD_PRINT_OPTS``</font> 与 <font color=#cc0000>``DYLD_PRINT_ENV``</font>。在 processDyldEnvironmentVariable 方法中：

```c++
	else if ( strcmp(key, "DYLD_IMAGE_SUFFIX") == 0 ) {
		gLinkContext.imageSuffix = parseColonList(value, NULL);
	}
	else if ( strcmp(key, "DYLD_INSERT_LIBRARIES") == 0 ) {
		sEnv.DYLD_INSERT_LIBRARIES = parseColonList(value, NULL);
#if SUPPORT_ACCELERATE_TABLES
		sDisableAcceleratorTables = true;
#endif
	}
```
	
在 secheme 添加这两个环境变量，对应的字段会被设置为 true，并不需要设置 value。
	
<center>
![](http://dzliving.com/DyldPrintSetting.png)
![](http://dzliving.com/DyldPrintLog.png)
</center>
	
但是并非每个环境变量都不需要配置 value，如：
	
```c++
void processDyldEnvironmentVariable(const char* key, const char* value, const char* mainExecutableDir)
{
	if ( strcmp(key, "DYLD_FRAMEWORK_PATH") == 0 ) {
		appendParsedColonList(value, mainExecutableDir, &sEnv.DYLD_FRAMEWORK_PATH);
	}
	else if ( strcmp(key, "DYLD_FALLBACK_FRAMEWORK_PATH") == 0 ) {
		appendParsedColonList(value, mainExecutableDir, &sEnv.DYLD_FALLBACK_FRAMEWORK_PATH);
	}
	else if ( strcmp(key, "DYLD_LIBRARY_PATH") == 0 ) {
		appendParsedColonList(value, mainExecutableDir, &sEnv.DYLD_LIBRARY_PATH);
	}
	else if ( strcmp(key, "DYLD_FALLBACK_LIBRARY_PATH") == 0 ) {
		appendParsedColonList(value, mainExecutableDir, &sEnv.DYLD_FALLBACK_LIBRARY_PATH);
	}
	
	...
```
	
##### 3.4.2 加载共享缓存

dyld3 与 dyld 不同点在 \_main 方法中可以看出。在 dyld 的 \_main 方法中，完成第一步以后会初始化主 App，然后加载共享缓存。到了 dyld3，调整了顺序：加载缓存的步骤可以划分为 mapSharedCache 和 checkVersionedPaths，先执行 mapSharedCache，然后加载主 App，最后checkVersionedPaths。（苹果在 2017 年发布的 dyld3，[视频链接](https://developer.apple.com/videos/play/wwdc2017/413/)）
	
对于共享缓存的理解：dyld 加载时，为了优化程序启动，启用了共享缓存（shared cache）技术。共享缓存会在进程启动时被 dyld 映射到内存中，之后，当任何 Mach-O 映像加载时，dyld 首先会检查该 Mach-O 映像及所需的动态库是否在共享缓存中，如果存在，则直接将它在共享内存中的内存地址映射到进程的内存地址空间。

```c++
uintptr_t
_main(const macho_header* mainExecutableMH, uintptr_t mainExecutableSlide, 
    int argc, const char* argv[], const char* envp[], const char* apple[], 
    uintptr_t* startGlue)
{
	...
	
	// load shared cache
	// 检查共享缓存是否可用
	checkSharedRegionDisable((dyld3::MachOLoaded*)mainExecutableMH, mainExecutableSlide);
#if TARGET_IPHONE_SIMULATOR
	// <HACK> until <rdar://30773711> is fixed
	gLinkContext.sharedRegionMode = ImageLoader::kUsePrivateSharedRegion;
	// </HACK>
#endif
	// 非 Dont Use
	if ( gLinkContext.sharedRegionMode != ImageLoader::kDontUseSharedRegion ) {
		// 映射共享缓存到共享区
		mapSharedCache();
	}
	
	// 缓存是否兼容（DyldSharedCache * loadAddress 为空 || 版本相同 -》YES）
	bool cacheCompatible = (sSharedCacheLoadInfo.loadAddress == nullptr) || (sSharedCacheLoadInfo.loadAddress->header.formatVersion == dyld3::closure::kFormatVersion);
	
	//  设置了 DYLD_USE_CLOSURES || 在白名单
	if ( cacheCompatible && (sEnableClosures || inWhiteList(sExecPath)) ) {
		
	}
	else {
		if ( gLinkContext.verboseWarnings )
			// 不使用closure，因为共享缓存格式版本与 dyld 不匹配
			dyld::log("dyld: not using closure because shared cache format version does not match dyld's\n");
	}
	// could not use closure info, launch old way
	
	
	
	// install gdb notifier
	stateToHandlers(dyld_image_state_dependents_mapped, sBatchHandlers)->push_back(notifyGDB);
	stateToHandlers(dyld_image_state_mapped, sSingleHandlers)->push_back(updateAllImages);
	// make initial allocations large enough that it is unlikely to need to be re-alloced
	sImageRoots.reserve(16);
	sAddImageCallbacks.reserve(4);
	sRemoveImageCallbacks.reserve(4);
	sAddLoadImageCallbacks.reserve(4);
	sImageFilesNeedingTermination.reserve(16);
	sImageFilesNeedingDOFUnregistration.reserve(8);
	
#if !TARGET_IPHONE_SIMULATOR
#ifdef WAIT_FOR_SYSTEM_ORDER_HANDSHAKE
	// <rdar://problem/6849505> Add gating mechanism to dyld support system order file generation process
	WAIT_FOR_SYSTEM_ORDER_HANDSHAKE(dyld::gProcessInfo->systemOrderFlag);
#endif
#endif
	
	
	try {
		// add dyld itself to UUID list
		// 添加 dyld 的 UUID 到共享缓存 UUID 列表中
		addDyldImageToUUIDList();
		
	...
}
```

* 检测共享缓存是否可用；
* 如果可用，映射共享缓存到共享区；
* 添加 dyld 的 UUID 到缓存列表。

其中，检测共享缓存是否可用的函数 <font color=#cc0000>``checkSharedRegionDisable``</font> 中有两句注释：

```c++
static void checkSharedRegionDisable(const dyld3::MachOLoaded* mainExecutableMH, uintptr_t mainExecutableSlide)
{
#if __MAC_OS_X_VERSION_MIN_REQUIRED
	// if main executable has segments that overlap the shared region, then disable using the shared region
	// 如果主程序 Mach-O 有 segments 与共享区重叠，那么共享区不可用。并且，iOS 不开启共享区无法运行。
	
	// 检测两者是否重叠
	if ( mainExecutableMH->intersectsRange(SHARED_REGION_BASE, SHARED_REGION_SIZE) ) {
		gLinkContext.sharedRegionMode = ImageLoader::kDontUseSharedRegion;
		if ( gLinkContext.verboseMapping )
			dyld::warn("disabling shared region because main executable overlaps\n");
	}
#if __i386__
	if ( !gLinkContext.allowEnvVarsPath ) {
		// <rdar://problem/15280847> use private or no shared region for suid processes
		gLinkContext.sharedRegionMode = ImageLoader::kUsePrivateSharedRegion;
	}
#endif
#endif
	// iOS cannot run without shared region
}
```

具体检测代码：

```c++
bool MachOLoaded::intersectsRange(uintptr_t start, uintptr_t length) const
{
    __block bool result = false;
    uintptr_t slide = getSlide();
    forEachSegment(^(const SegmentInfo& info, bool& stop) {
        /*
            ①、主程序 segment 中的虚拟地址 + 虚拟地址大小 + 偏移量 >= 共享区起始地址
            ②、主程序 segment 中的虚拟地址 + 偏移量 < 共享区终止地址
            
            ① 和 ② 同时 YES，那么认为主程序 Mach-O 有 segments 与共享区重叠，此时共享区不可用，从而动态库缓存不可用
        
            疑问：地址是从高到低分配？
        */
        if ( (info.vmAddr+info.vmSize+slide >= start) && (info.vmAddr+slide < start+length) )
            result = true;
    });
    return result;
}
```

可以看到这段检测代码在满足重叠条件后，并没有设置 stop = true 停止 <font color=#cc0000>``forEachLoadCommand``</font> 中的循环，这里值得深究和讨论。

加载共享缓存最核心的步骤在 <font color=#cc0000>``mapSharedCache``</font> 中：
	
```c++
static void mapSharedCache()
{
	dyld3::SharedCacheOptions opts;
	opts.cacheDirOverride	= sSharedCacheOverrideDir;
	opts.forcePrivate		= (gLinkContext.sharedRegionMode == ImageLoader::kUsePrivateSharedRegion);
	
	
#if __x86_64__ && !TARGET_IPHONE_SIMULATOR
	opts.useHaswell			= sHaswell;
#else
	opts.useHaswell			= false;
#endif
	opts.verbose			= gLinkContext.verboseMapping;
	
	// 加载 dyld 缓存
	loadDyldCache(opts, &sSharedCacheLoadInfo);
	
	// update global state
	// 更新进程的全局状态信息
	if ( sSharedCacheLoadInfo.loadAddress != nullptr ) {
		gLinkContext.dyldCache 								= sSharedCacheLoadInfo.loadAddress;
		dyld::gProcessInfo->processDetachedFromSharedRegion = opts.forcePrivate;
		dyld::gProcessInfo->sharedCacheSlide                = sSharedCacheLoadInfo.slide;
		dyld::gProcessInfo->sharedCacheBaseAddress          = (unsigned long)sSharedCacheLoadInfo.loadAddress;
		sSharedCacheLoadInfo.loadAddress->getUUID(dyld::gProcessInfo->sharedCacheUUID);
		dyld3::kdebug_trace_dyld_image(DBG_DYLD_UUID_SHARED_CACHE_A, (const uuid_t *)&dyld::gProcessInfo->sharedCacheUUID[0], {0,0}, {{ 0, 0 }}, (const mach_header *)sSharedCacheLoadInfo.loadAddress);
	}
}
```
	
SharedCacheRuntime.cpp 文件：
	
```c++
bool loadDyldCache(const SharedCacheOptions& options, SharedCacheLoadInfo* results)
{
    results->loadAddress        = 0;
    results->slide              = 0;
    results->errorMessage       = nullptr;
	
#if TARGET_IPHONE_SIMULATOR
    // simulator only supports mmap()ing cache privately into process
    // 模拟器只支持 mmap（内存映射） 缓存到当前进程
    return mapCachePrivate(options, results);
#else
    if ( options.forcePrivate ) {
        // mmap cache into this process only
        // 只加载 mmap（内存映射） 缓存到当前进程
        return mapCachePrivate(options, results);
    }
    else {
        // fast path: when cache is already mapped into shared region
        bool hasError = false;
        // 已加载过的
        if ( reuseExistingCache(options, results) ) {
            hasError = (results->errorMessage != nullptr);
        }
        // 未加载过的
        else {
            // slow path: this is first process to load cache
            hasError = mapCacheSystemWide(options, results);
        }
        return hasError;
    }
#endif
}
```

加载缓存分三种情况：
	
①、仅加载到当前进程。通过 ``mapCachePrivate()`` 加载并返回错误信息；
②、已经加载过的。通过 ``reuseExistingCache()`` 加载并返回错误信息，同时返回是否加载过 BOOL 值；
③、未加载过的。通过 ``mapCacheSystemWide()`` 加载缓存并映射，返回错误信息。
	
options.forcePrivate 的定义：
	
```c++
// dyld.cpp
opts.forcePrivate = (gLinkContext.sharedRegionMode == ImageLoader::kUsePrivateSharedRegion)
	
gLinkContext.sharedRegionMode		= ImageLoader::kUseSharedRegion;
	
// ImageLoader.h
class ImageLoader {
public:
		...
		enum SharedRegionMode { kUseSharedRegion, kUsePrivateSharedRegion, kDontUseSharedRegion, kSharedRegionIsSharedCache };
	
		...		
}
```
	
gLinkContext.sharedRegionMode 在 setContext() 方法中设置默认值，默认值为 kUseSharedRegion，也就是之前检测共享区是否可用的标识值。


##### 3.4.3 实例化主程序
	
系统会对已经映射到进程空间的主程序（在 XNU 解析 MachO 阶段就完成了映射操作）创建一个ImageLoaderMachO，再将其加入到 master list 中（sAllImages）。如果加载的 MachO 的硬件架构与本设备相符，就执行 imageLoader 的创建和添加操作。其中主要实现是ImageLoaderMachO::instantiateMainExecutable方法，该方法将主 App 的 MachHeader、ASLR，文件路径和前面提到的链接上下文作为参数，做 imageLoader 的实例化操作。
	
```c++
uintptr_t
_main(const macho_header* mainExecutableMH, uintptr_t mainExecutableSlide, 
    int argc, const char* argv[], const char* envp[], const char* apple[], 
    uintptr_t* startGlue)
{
	...
	
  		CRSetCrashLogMessage(sLoadingCrashMessage);
		// instantiate ImageLoader for main executable
		// 实例化主程序
		sMainExecutable = instantiateFromLoadedImage(mainExecutableMH, mainExecutableSlide, sExecPath);
		gLinkContext.mainExecutable = sMainExecutable;
		// 代码签名
		gLinkContext.mainExecutableCodeSigned = hasCodeSignatureLoadCommand(mainExecutableMH);
	
#if TARGET_IPHONE_SIMULATOR
		// check main executable is not too new for this OS
		// 检测主程序是否支持当前设备版本
		{
			// 检查是否是模拟器二进制文件
			if ( ! isSimulatorBinary((uint8_t*)mainExecutableMH, sExecPath) ) {
				throwf("program was built for a platform that is not supported by this runtime");
			}
			uint32_t mainMinOS = sMainExecutable->minOSVersion();
	
			// dyld is always built for the current OS, so we can get the current OS version
			// from the load command in dyld itself.
			// 获取 dyld 中存储的当前 OS 版本
			uint32_t dyldMinOS = ImageLoaderMachO::minOSVersion((const mach_header*)&__dso_handle);
			// 应用 mach-O 文件的版本超过了当前模拟器设备的版本，抛出异常
			if ( mainMinOS > dyldMinOS ) {
	#if TARGET_OS_WATCH
				throwf("app was built for watchOS %d.%d which is newer than this simulator %d.%d",
						mainMinOS >> 16, ((mainMinOS >> 8) & 0xFF),
						dyldMinOS >> 16, ((dyldMinOS >> 8) & 0xFF));
	#elif TARGET_OS_TV
				throwf("app was built for tvOS %d.%d which is newer than this simulator %d.%d",
						mainMinOS >> 16, ((mainMinOS >> 8) & 0xFF),
						dyldMinOS >> 16, ((dyldMinOS >> 8) & 0xFF));
	#else
				throwf("app was built for iOS %d.%d which is newer than this simulator %d.%d",
						mainMinOS >> 16, ((mainMinOS >> 8) & 0xFF),
						dyldMinOS >> 16, ((dyldMinOS >> 8) & 0xFF));
	#endif
			}
		}
#endif
	
	
	#if __MAC_OS_X_VERSION_MIN_REQUIRED
		// <rdar://problem/22805519> be less strict about old mach-o binaries
		uint32_t mainSDK = sMainExecutable->sdkVersion();
		gLinkContext.strictMachORequired = (mainSDK >= DYLD_MACOSX_VERSION_10_12) || gLinkContext.allowInsertFailures;
	#else
		// simulators, iOS, tvOS, and watchOS are always strict
		gLinkContext.strictMachORequired = true;
	#endif
	
	#if SUPPORT_ACCELERATE_TABLES
		sAllImages.reserve((sAllCacheImagesProxy != NULL) ? 16 : INITIAL_IMAGE_COUNT);
	#else
		sAllImages.reserve(INITIAL_IMAGE_COUNT);
	#endif
	
		// Now that shared cache is loaded, setup an versioned dylib overrides
	#if SUPPORT_VERSIONED_PATHS
		checkVersionedPaths(); // 设置加载的动态库版本。这里的动态库还没有包括经 DYLD_INSERT_LIBRARIES 插入的库。
	#endif
	
	
		// dyld_all_image_infos image list does not contain dyld
		// add it as dyldPath field in dyld_all_image_infos
		// for simulator, dyld_sim is in image list, need host dyld added
		// dyld 加载的 image_infos 并不包含 dyld 本身，它被放到 dyld_all_image_infos 的 dyldPath 字段中去了。而对于模拟器，dyld 加载的 image_infos 是包含 dyld_sim 的。
#if TARGET_IPHONE_SIMULATOR
		// get path of host dyld from table of syscall vectors in host dyld
		void* addressInDyld = gSyscallHelpers;
#else
		// get path of dyld itself
		void*  addressInDyld = (void*)&__dso_handle;
#endif
		
		// 获取 dyld 路径并与 gProcessInfo->dyldPath 对比
		char dyldPathBuffer[MAXPATHLEN+1];
		int len = proc_regionfilename(getpid(), (uint64_t)(long)addressInDyld, dyldPathBuffer, MAXPATHLEN);
		if ( len > 0 ) {
			dyldPathBuffer[len] = '\0'; // proc_regionfilename() does not zero terminate returned string
			// 如果不同将获取到的路径复制给 gProcessInfo->dyldPath
			if ( strcmp(dyldPathBuffer, gProcessInfo->dyldPath) != 0 )
				gProcessInfo->dyldPath = strdup(dyldPathBuffer);
		}
	
	...
}
```

<font color=#cc0000>``dyld_all_image_infos``</font> 是个结构体，同样分为 32 位和 64 位两个版本，分别对应 dyld\_all\_image\_infos\_32 与 dyld\_all\_image\_infos\_64，由于获取 dyld\_all\_image\_infos 需要用到一些未开源信息，这里为了方便，从侧面验证一下这条注释信息：
	
```c++
#import <mach-o/dyld.h>

- (void)viewDidLoad 
{
	[super viewDidLoad];
    
    for (uint32_t i = 0; i < _dyld_image_count(); ++i) {
        NSLog(@"%s", _dyld_get_image_name(i));
    }
}
```

模拟器：
	
<center>
![](http://dzliving.com/DyldSimAllImageInfo.png)
</center>

真机：
	
<center>
![](http://dzliving.com/DyldPhoneAllImageInfo.png)
</center>

可以看到：模拟器打印的 image 没有 dyld，第 0 个 image 是 dyld\_sim，第一个 image 才是主程序；真机打印出的加载 image 中也没有 dyld，第 0 个 image 是主程序。

回到最核心的 <font color=#cc0000>``instantiateFromLoadedImage``</font> 实例化主程序函数：
	
```c++
// The kernel maps in main executable before dyld gets control.  We need to 
// make an ImageLoader* for the already mapped in main executable.
// kernel 在 dyld 之前已经映射了主程序 Mach-O，dyld 判断 Mach-O 的兼容性后，实例化成 ImageLoader 加载到内存中交给 dyld 管理
static ImageLoaderMachO* instantiateFromLoadedImage(const macho_header* mh, uintptr_t slide, const char* path)
{
	// try mach-o loader
	// CPU 架构是否匹配
	if ( isCompatibleMachO((const uint8_t*)mh, path) ) {
		// 实例化 ImageLoader 对象。参数：macho header、ASLR、执行路径、链接上下文
		ImageLoader* image = ImageLoaderMachO::instantiateMainExecutable(mh, slide, path, gLinkContext);
		// 分配主程序image的内存，更新。
		addImage(image);
		
		return (ImageLoaderMachO*)image;
	}
	
	throw "main executable not a known format";
}
```

kernel 在 dyld 之前已经映射了主程序 Mach-O，dyld 判断 Mach-O 的兼容性后，实例化ImageLoader 对象，加载到内存，返回交给 dyld 管理。

```c++
// create image for main executable
ImageLoader* ImageLoaderMachO::instantiateMainExecutable(const macho_header* mh, uintptr_t slide, const char* path, const LinkContext& context)
{
	//dyld::log("ImageLoader=%ld, ImageLoaderMachO=%ld, ImageLoaderMachOClassic=%ld, ImageLoaderMachOCompressed=%ld\n",
	//	sizeof(ImageLoader), sizeof(ImageLoaderMachO), sizeof(ImageLoaderMachOClassic), sizeof(ImageLoaderMachOCompressed));
	bool compressed;
	unsigned int segCount;
	unsigned int libCount;
	const linkedit_data_command* codeSigCmd;
	const encryption_info_command* encryptCmd;
	
	// sniffLoadCommands 函数会对主程序 Mach-O进 行一系列的校验：对代码签名，MachO加密，动态库数量，段的数量相关信息的 loadCommand 做解析，提取出 command 数据。
	/*      case LC_DYLD_INFO:
	 	case LC_DYLD_INFO_ONLY:
	 		*compressed = true;
	 */
	sniffLoadCommands(mh, path, false, &compressed, &segCount, &libCount, context, &codeSigCmd, &encryptCmd);
	// instantiate concrete class based on content of load commands
	// 已解密
	if ( compressed ) 
		// Compressed
		return ImageLoaderMachOCompressed::instantiateMainExecutable(mh, slide, path, segCount, libCount, context);
	else
#if SUPPORT_CLASSIC_MACHO
		// Classic
		return ImageLoaderMachOClassic::instantiateMainExecutable(mh, slide, path, segCount, libCount, context);
#else
		throw "missing LC_DYLD_INFO load command";
#endif
}
```

sniffLoadCommands 的校验并不包括对主程序 Mach-O 的解密操作，解密操作是由 xnu 完成的。
	
ImageLoaderMachOCompressed::instantiateMainExecutable、ImageLoaderMachOClassic::instantiateMainExecutable 两者内部的逻辑相同，只是返回类型一个是 ImageLoaderMachOCompressed 一个是 ImageLoaderMachOClassic。
	
以 ImageLoaderMachOCompressed 为例：
	
```c++
// create image for main executable
ImageLoaderMachOCompressed* ImageLoaderMachOCompressed::instantiateMainExecutable(const macho_header* mh, uintptr_t slide, const char* path, 
																		unsigned int segCount, unsigned int libCount, const LinkContext& context)
{
	// 初始化 image
	ImageLoaderMachOCompressed* image = ImageLoaderMachOCompressed::instantiateStart(mh, path, segCount, libCount);
	
	// set slide for PIE programs
	// 设置 image 偏移量
	image->setSlide(slide);
	
	// for PIE record end of program, to know where to start loading dylibs
	if ( slide != 0 )
		// 设置动态库起始地址
		fgNextPIEDylibAddress = (uintptr_t)image->getEnd();
	
	// 禁用段覆盖检测
	image->disableCoverageCheck();
	// 结束 image 上下文
	image->instantiateFinish(context);
	// 设置 image 加载状态为 dyld_image_state_mapped
	image->setMapped(context);
	
	if ( context.verboseMapping ) {
		dyld::log("dyld: Main executable mapped %s\n", path);
		for(unsigned int i=0, e=image->segmentCount(); i < e; ++i) {
			const char* name = image->segName(i);
			if ( (strcmp(name, "__PAGEZERO") == 0) || (strcmp(name, "__UNIXSTACK") == 0)  )
				dyld::log("%18s at 0x%08lX->0x%08lX\n", name, image->segPreferredLoadAddress(i), image->segPreferredLoadAddress(i)+image->segSize(i));
			else
				dyld::log("%18s at 0x%08lX->0x%08lX\n", name, image->segActualLoadAddress(i), image->segActualEndAddress(i));
		}
	}
	
	return image;
}
```
	
```c++
void ImageLoader::setMapped(const LinkContext& context)
{
	fState = dyld_image_state_mapped;
	context.notifySingle(dyld_image_state_mapped, this, NULL);  // note: can throw exception
}
```

instantiateFinish() 在内部解析 loadCmds、设置动态库连接信息、设置符号表相关信息等。setMapped() 内部调用 notifySingle 进行处理。
	
	
##### 3.4.4 加载插入的动态库

```c++
uintptr_t
_main(const macho_header* mainExecutableMH, uintptr_t mainExecutableSlide, 
        int argc, const char* argv[], const char* envp[], const char* apple[], 
        uintptr_t* startGlue)
{
	...
	
	// load any inserted libraries
	// 插入动态库
	if	( sEnv.DYLD_INSERT_LIBRARIES != NULL ) {
		for (const char* const* lib = sEnv.DYLD_INSERT_LIBRARIES; *lib != NULL; ++lib) 
			loadInsertedDylib(*lib);
	}
	// record count of inserted libraries so that a flat search will look at 
	// inserted libraries, then main, then others.
	// 记录插入的动态库个数
	sInsertedDylibCount = sAllImages.size()-1;
	
	...
}
```

如果配置了 <font color=#cc0000>``DYLD_INSERT_LIBRARIES``</font> 环境变量，通过loadInsertedDylib() 方法插入配置的动态库。对于越狱插件而言，其实就是通过添加 DYLD\_INSERT\_LIBRARIES 这个环境变量达到加载插件的目的。
	
```c++
static void loadInsertedDylib(const char* path)
{
	ImageLoader* image = NULL;
	unsigned cacheIndex;
	try {
		LoadContext context;
		context.useSearchPaths		= false;
		context.useFallbackPaths	= false;
		context.useLdLibraryPath	= false;
		context.implicitRPath		= false;
		context.matchByInstallName	= false;
		context.dontLoad			= false;
		context.mustBeBundle		= false;
		context.mustBeDylib			= true;
		context.canBePIE			= false;
		context.enforceIOSMac		= true;
		context.origin				= NULL;	// can't use @loader_path with DYLD_INSERT_LIBRARIES
		context.rpath				= NULL;
		image = load(path, context, cacheIndex);
	}
	catch (const char* msg) {
		if ( gLinkContext.allowInsertFailures )
			dyld::log("dyld: warning: could not load inserted library '%s' into hardened process because %s\n", path, msg);
		else
			halt(dyld::mkstringf("could not load inserted library '%s' because %s\n", path, msg));
	}
	catch (...) {
		halt(dyld::mkstringf("could not load inserted library '%s'\n", path));
	}
}
```

内部构建 context 后调用 load() 函数生成 image。
	
```c++
/**
*  @brief  做路径展开，搜索查找，排重，以及缓存查找工作。其中路径的展开和搜索分几个阶段（phase）
*/
ImageLoader* load(const char* path, const LoadContext& context, unsigned& cacheIndex)
{
	...
 
	// 查找 image
	ImageLoader* image = loadPhase0(path, orgPath, context, cacheIndex, NULL);
	// 没有找到
	if ( image != NULL ) {
		// 继续查找
		CRSetCrashLogMessage2(NULL);
		return image;
	}
	
	// try all path permutations and try open() until first success
	std::vector<const char*> exceptions;
	image = loadPhase0(path, orgPath, context, cacheIndex, &exceptions);
#if !TARGET_IPHONE_SIMULATOR
	// <rdar://problem/16704628> support symlinks on disk to a path in dyld shared cache
	// 在共享缓存中查找
	if ( image == NULL)
		image = loadPhase2cache(path, orgPath, context, cacheIndex, &exceptions);
#endif
    CRSetCrashLogMessage2(NULL);
	if ( image != NULL ) {
		// <rdar://problem/6916014> leak in dyld during dlopen when using DYLD_ variables
		for (std::vector<const char*>::iterator it = exceptions.begin(); it != exceptions.end(); ++it) {
			free((void*)(*it));
		}
		// if loaded image is not from cache, but original path is in cache
		// set gSharedCacheOverridden flag to disable some ObjC optimizations
		if ( !gSharedCacheOverridden && !image->inSharedCache() && image->isDylib() && cacheablePath(path) && inSharedCache(path) ) {
			gSharedCacheOverridden = true;
		}
		return image;
	}
	
	...
}
```

load 方法不仅被 loadInsertedDylib 调用，也会被 dlopen 等运行时加载动态库的方法使用。
	
内部有一整套 loadPhase0~loadPhase6 的流程来查找及加载 image。如果在共享缓存中找到则直接调用 instantiateFromCache 实例化 image，否则通过 loadPhase5open 打开文件并调用loadPhase6，内部通过 instantiateFromFile 实例化 image，最后再调用 checkandAddImage 将image 加载进内存。
	
 这些 phase 的搜索路径对应各个环境变量：DYLD\_ROOT\_PATH->LD\_LIBRARY\_PATH->DYLD\_FRAMEWORK\_PATH->原始路径->DYLD\_FALLBACK\_LIBRARY\_PATH。
 
 ImageLoaderMachO 的 ``instantiateFromFile``、``instantiateFromCache`` 是 loader 将 MachO 文件解析映射到内存的核心方法，两个都会进入 Compressed 和 Classic 的分叉步骤。以 Compressed 下的 instantiateFromFile 来分析。
 
```c++
// create image by mapping in a mach-o file
ImageLoaderMachOCompressed* ImageLoaderMachOCompressed::instantiateFromFile(const char* path, int fd, const uint8_t* fileData, size_t lenFileData,
														uint64_t offsetInFat, uint64_t lenInFat, const struct stat& info, 
														unsigned int segCount, unsigned int libCount, 
														const struct linkedit_data_command* codeSigCmd, 
														const struct encryption_info_command* encryptCmd, 
														const LinkContext& context)
{
		ImageLoaderMachOCompressed* image = ImageLoaderMachOCompressed::instantiateStart((macho_header*)fileData, path, segCount, libCount);
	
		try {
			// record info about file  
			image->setFileInfo(info.st_dev, info.st_ino, info.st_mtime);
	
			// if this image is code signed, let kernel validate signature before mapping any pages from image
			// ①、交给内核去验证动态库的代码签名
			image->loadCodeSignature(codeSigCmd, fd, offsetInFat, context);
			
			// Validate that first data we read with pread actually matches with code signature
			// ②、映射到内存的 first page, （4k大小）与代码签名是否match。在这里会执行沙盒，签名认证
			image->validateFirstPages(codeSigCmd, fd, fileData, lenFileData, offsetInFat, context);
	
			// mmap segments
			image->mapSegments(fd, offsetInFat, lenInFat, info.st_size, context);
	
			// if framework is FairPlay encrypted, register with kernel
			// 根据 DYLD_ENCRYPTION_INFO，让内核去注册加密信息。在该方法中，会调用内核方法 mremap_encrypted，传入加密数据的地址和长度等数据，查看了内核代码，应该是根据cryptid是否为1做了解密操作。
			image->registerEncryption(encryptCmd, context);
			
			// probe to see if code signed correctly
			image->crashIfInvalidCodeSignature();
	
			// finish construction
			image->instantiateFinish(context);
		
		...
		}
}
```

其中几个需要留意的步骤：

* 交给内核去验证动态库的代码签名 loadCodeSignature。
* 映射到内存的 first page（4k 大小）与代码签名是否 match。在这里会执行沙盒，签名认证，对于在线上运行时加载动态库的需求，可以重点研究[这里](https://mp.weixin.qq.com/s/fdDPyjRkVf9AdWiikBagHg)。
* 根据 DYLD\_ENCRYPTION\_INFO，让内核去注册加密信息 registerEncryption。在该方法中，会调用内核方法 mremap\_encrypted，传入加密数据的地址和长度等数据，查看了[内核代码](http://newosxbook.com/src.jl?tree=xnu-3789.70.16&file=bsd/kern/kern_mman.c#mremap_encrypted)，应该是根据 cryptid 是否为 1 做了解密操作。
* 如果走到 Phase6, 会调 xmap 函数将动态库从本地 mmap 到用户态内存空间。

根据上面的分析，<font color=#cc0000>主程序 imageLoader 在全局 image 表的首位</font>，后面的是插入的动态库的 imageLoader，每个动态库对应一个 loader。

##### 3.4.5 链接主程序

链接所有动态库，进行符号修正绑定工作。

```oc
uintptr_t
_main(const macho_header* mainExecutableMH, uintptr_t mainExecutableSlide, 
    int argc, const char* argv[], const char* envp[], const char* apple[], 
    uintptr_t* startGlue)
{
	...
        // link main executable
        gLinkContext.linkingMainExecutable = true;
#if SUPPORT_ACCELERATE_TABLES
        if ( mainExcutableAlreadyRebased ) {
            // previous link() on main executable has already adjusted its internal pointers for ASLR
            // work around that by rebasing by inverse amount
            sMainExecutable->rebase(gLinkContext, -mainExecutableSlide);
        }
#endif
                // 链接主程序
        link(sMainExecutable, sEnv.DYLD_BIND_AT_LAUNCH, true, ImageLoader::RPathChain(NULL, NULL), -1);
        sMainExecutable->setNeverUnloadRecursive();
        if ( sMainExecutable->forceFlat() ) {
            gLinkContext.bindFlat = true;
            gLinkContext.prebindUsage = ImageLoader::kUseNoPrebinding;
        }
	...
}
```

可以看到，主程序的链接是通过 ``link`` 这个函数完成的：

```oc
void link(ImageLoader* image, bool forceLazysBound, bool neverUnload, const ImageLoader::RPathChain& loaderRPaths, unsigned cacheIndex)
{
	// add to list of known images.  This did not happen at creation time for bundles
	// 添加到已知镜像列表中。这在创建 bundles 时没有处理。
	if ( image->isBundle() && !image->isLinked() )
		addImage(image);
	
	// we detect root images as those not linked in yet
	// 在根镜像中检测是否尚未链接
	if ( !image->isLinked() )
		addRootImage(image);
	
	// process images
	try {
		const char* path = image->getPath();
#if SUPPORT_ACCELERATE_TABLES
		if ( image == sAllCacheImagesProxy )
			path = sAllCacheImagesProxy->getIndexedPath(cacheIndex);
#endif
		// 调用 ImageLoader::link() 链接
		image->link(gLinkContext, forceLazysBound, false, neverUnload, loaderRPaths, path);
	}
	catch (const char* msg) {
		// 标记 image 为未使用，处理
		garbageCollectImages();
		throw;
	}
}
	
void ImageLoader::link(const LinkContext& context, bool forceLazysBound, bool preflightOnly, bool neverUnload, const RPathChain& loaderRPaths, const char* imagePath)
{
	//dyld::log("ImageLoader::link(%s) refCount=%d, neverUnload=%d\n", imagePath, fDlopenReferenceCount, fNeverUnload);
	
	// clear error strings
	(*context.setErrorStrings)(0, NULL, NULL, NULL);
	
	// 起始时间。用于记录时间间隔
	uint64_t t0 = mach_absolute_time();
	// ①、递归加载主程序依赖的库，完成之后发送一个状态为 dyld_image_state_dependents_mapped的通知。
	this->recursiveLoadLibraries(context, preflightOnly, loaderRPaths, imagePath);
	context.notifyBatch(dyld_image_state_dependents_mapped, preflightOnly);
	
	// we only do the loading step for preflights  只做预检的装载步骤
	if ( preflightOnly )
		return;
	
	uint64_t t1 = mach_absolute_time();
	// 清空 image 层级关系
	context.clearAllDepths();
	// 递归更新 image 层级关系
	this->recursiveUpdateDepth(context.imageCount());
	
	__block uint64_t t2, t3, t4, t5;
	{
		dyld3::ScopedTimer(DBG_DYLD_TIMING_APPLY_FIXUPS, 0, 0, 0);
		t2 = mach_absolute_time();
		// ②、递归修正自己和依赖库的基地址，因为 ASLR 的原因，需要根据随机 slide 修正基地址。
		this->recursiveRebase(context);
		context.notifyBatch(dyld_image_state_rebased, false);
	
		t3 = mach_absolute_time();
		if ( !context.linkingMainExecutable )
			// ③、递归绑定 noLazy 的符号表，lazy的符号会在运行时动态绑定（首次被调用才去绑定）
			this->recursiveBindWithAccounting(context, forceLazysBound, neverUnload);
	
		t4 = mach_absolute_time();
		if ( !context.linkingMainExecutable )
			// ④、绑定弱符号表，比如未初始化的全局变量就是弱符号。
			this->weakBind(context);
		t5 = mach_absolute_time();
	}
	
    if ( !context.linkingMainExecutable )
        context.notifyBatch(dyld_image_state_bound, false);
	uint64_t t6 = mach_absolute_time();	
	
	std::vector<DOFInfo> dofs;
	// ⑤、递归获取/注册程序的 DOF 节区，dtrace 会用其动态跟踪。
	this->recursiveGetDOFSections(context, dofs);
	// 注册
	context.registerDOFs(dofs);
	uint64_t t7 = mach_absolute_time();	
	
	// interpose any dynamically loaded images
	if ( !context.linkingMainExecutable && (fgInterposingTuples.size() != 0) ) {
		dyld3::ScopedTimer timer(DBG_DYLD_TIMING_APPLY_INTERPOSING, 0, 0, 0);
		// 递归应用插入的动态库
		this->recursiveApplyInterposing(context);
	}
	
	// clear error strings
	(*context.setErrorStrings)(0, NULL, NULL, NULL);
	
	// 计算出各种时间间隔
	fgTotalLoadLibrariesTime += t1 - t0;
	fgTotalRebaseTime += t3 - t2;
	fgTotalBindTime += t4 - t3;
	fgTotalWeakBindTime += t5 - t4;
	fgTotalDOF += t7 - t6;
	
	// done with initial dylib loads
	fgNextPIEDylibAddress = 0;
}
```

内部加载动态库、rebase、绑定符号表、注册dofs信息等，同时还计算各步骤的耗时。如果想获取这些耗时，只需要在环境变量中添加 <font color=#cc0000>``DYLD_PRINT_STATISTICS``</font> 就可以了，这个环境变量不需要 value。
	
在步骤 ① 里，递归加载主 App 在打包阶段就确定好的动态库的操作，会使用前面提到的 setContext 里的链接上下文，调用它的 loadLibrary 方法；然后优先去加载依赖的动态库。loadLibary 的实现在设置链接上下文的时候就已经赋值确定，即 libraryLocator，在这个方法里会用到上面提到的 load 方法。

在步骤 ③ 里，会有符号绑定的操作。
	
```c++
/**
*  @brief   recursiveBind 完成递归绑定符号表的操作。此处的符号表针对的是非延迟加载的符号表，对于 DYLD_BIND_AT_LAUNCH 等特殊情况下的 non-lazy 符号才执行立即绑定。
*/
void ImageLoader::recursiveBind(const LinkContext& context, bool forceLazysBound, bool neverUnload)
{
	// Normally just non-lazy pointers are bound immediately.
	// The exceptions are:
	//   1) DYLD_BIND_AT_LAUNCH will cause lazy pointers to be bound immediately
	//   2) some API's (e.g. RTLD_NOW) can cause lazy pointers to be bound immediately
	if ( fState < dyld_image_state_bound ) {
		// break cycles
		fState = dyld_image_state_bound;
	
		try {
			// bind lower level libraries first
			for(unsigned int i=0; i < libraryCount(); ++i) {
				ImageLoader* dependentImage = libImage(i);
				if ( dependentImage != NULL )
					dependentImage->recursiveBind(context, forceLazysBound, neverUnload);
			}
			// bind this image
			// 绑定。this 表示递归调用时，recursiveBind 方法的调用者
			this->doBind(context, forceLazysBound);	
			// mark if lazys are also bound
			if ( forceLazysBound || this->usablePrebinding(context) )
				fAllLazyPointersBound = true;
			// mark as never-unload if requested
			if ( neverUnload )
				this->setNeverUnload();
			
			// 通知
			context.notifySingle(dyld_image_state_bound, this, NULL);
		}
		catch (const char* msg) {
			// restore state
			fState = dyld_image_state_rebased;
            CRSetCrashLogMessage2(NULL);
			throw;
		}
	}
}
```
	
方法的核心是 ImageLoaderMach 的 doBind，读取 image 的动态链接信息的 bind\_off 与 bind\_size 来确定需要绑定的数据偏移与大小，然后挨个对它们进行绑定，绑定操作具体使用 <font color=#cc0000>bindAt</font> 函数；调用 resolve 解析完符号表后，调用 bindLocation 完成最终的绑定操作，需要绑定的符号信息有三种：

- BIND\_TYPE\_POINTER：需要绑定的是一个指针。直接将计算好的新值屿值即可。
- BIND\_TYPE\_TEXT\_ABSOLUTE32：一个32位的值。取计算的值的低32位赋值过去。
- BIND\_TYPE\_TEXT\_PCREL32：重定位符号。需要使用新值减掉需要修正的地址值来计算出重定位值。

对延迟绑定的实现感兴趣的可以在Xcode中调试查看，或者参考[这个](https://leylfl.github.io/2018/05/28/dyld%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B/)。

##### 3.4.6 链接插入的动态库

```c++
uintptr_t
_main(const macho_header* mainExecutableMH, uintptr_t mainExecutableSlide, 
    int argc, const char* argv[], const char* envp[], const char* apple[], 
    uintptr_t* startGlue)
{
	...
		// link any inserted libraries
		// do this after linking main executable so that any dylibs pulled in by inserted 
		// dylibs (e.g. libSystem) will not be in front of dylibs the program uses
		// 链接其他被插入的动态库
		if ( sInsertedDylibCount > 0 ) {
			// 循环处理
			for(unsigned int i=0; i < sInsertedDylibCount; ++i) {
				ImageLoader* image = sAllImages[i+1];
				// 链接
				link(image, sEnv.DYLD_BIND_AT_LAUNCH, true, ImageLoader::RPathChain(NULL, NULL), -1);
				// 递归修改 image 的 fNeverUnload 属性
				image->setNeverUnloadRecursive();
			}
			// only INSERTED libraries can interpose
			// register interposing info after all inserted libraries are bound so chaining works
			// 只有插入可插入的库。在绑定所有插入的库后注册插入信息，以便链接工作
			for(unsigned int i=0; i < sInsertedDylibCount; ++i) {
				ImageLoader* image = sAllImages[i+1];
				image->registerInterposing(gLinkContext);
			}
		}
	
		// <rdar://problem/19315404> dyld should support interposition even without DYLD_INSERT_LIBRARIES
		// 即使没有 DYLD_INSERT_LIBRARIES，dyld 也应该支持插入
		for (long i=sInsertedDylibCount+1; i < sAllImages.size(); ++i) {
			ImageLoader* image = sAllImages[i];
			if ( image->inSharedCache() )
				continue;
			image->registerInterposing(gLinkContext);
		}
	#if SUPPORT_ACCELERATE_TABLES  // !TARGET_IPHONE_SIMULATOR，非模拟器
		if ( (sAllCacheImagesProxy != NULL) && ImageLoader::haveInterposingTuples() ) {
			// Accelerator tables cannot be used with implicit interposing, so relaunch with accelerator tables disabled
			// 加速键表不能与隐式插入一起使用，因此在禁用加速键表的情况下重新启动
			ImageLoader::clearInterposingTuples();
			// unmap all loaded dylibs (but not main executable)
			// 取消映射所有加载的 dylib 文件，除了主程序
			for (long i=1; i < sAllImages.size(); ++i) {
				ImageLoader* image = sAllImages[i];
				if ( image == sMainExecutable )
					continue;
				if ( image == sAllCacheImagesProxy )
					continue;
				image->setCanUnload();
				ImageLoader::deleteImage(image);
			}
			// note: we don't need to worry about inserted images because if DYLD_INSERT_LIBRARIES was set we would not be using the accelerator table
			sAllImages.clear();
			sImageRoots.clear();
			sImageFilesNeedingTermination.clear();
			sImageFilesNeedingDOFUnregistration.clear();
			sAddImageCallbacks.clear();
			sRemoveImageCallbacks.clear();
			sAddLoadImageCallbacks.clear();
			sDisableAcceleratorTables = true;
			sAllCacheImagesProxy = NULL;  // 下次不再进入
			sMappedRangesStart = NULL;
			mainExcutableAlreadyRebased = true;
			gLinkContext.linkingMainExecutable = false;
			resetAllImages();
			
			// 跳转回上面的步骤，重新执行，加载所有的镜像
			goto reloadAllImages;
		}
	#endif
	
		// apply interposing to initial set of images
		for(int i=0; i < sImageRoots.size(); ++i) {
			// 是调用 ImageLoader::applyInterposing()，不是 ClosureWriter.cpp。内部递归，最终是执行 doInterpose() 方法
			sImageRoots[i]->applyInterposing(gLinkContext);
		}
		// 插入信息存入 dyld 缓存
		ImageLoader::applyInterposingToDyldCache(gLinkContext);
		// 修改主程序插入标识
		gLinkContext.linkingMainExecutable = false;
	
		// Bind and notify for the main executable now that interposing has been registered
		// 从主程序开始调用，递归执行绑定、通知（插入信息已经注册）
		uint64_t bindMainExecutableStartTime = mach_absolute_time();
		// 内部执行 doBind()、notifySingle()
		sMainExecutable->recursiveBindWithAccounting(gLinkContext, sEnv.DYLD_BIND_AT_LAUNCH, true);
		uint64_t bindMainExecutableEndTime = mach_absolute_time();
		// 绑定和通知处理时间
		ImageLoaderMachO::fgTotalBindTime += bindMainExecutableEndTime - bindMainExecutableStartTime;
		gLinkContext.notifyBatch(dyld_image_state_bound, false);
	
		// Bind and notify for the inserted images now interposing has been registered
		if ( sInsertedDylibCount > 0 ) {
			for(unsigned int i=0; i < sInsertedDylibCount; ++i) {
				ImageLoader* image = sAllImages[i+1];
				// 绑定插入的动态库
				image->recursiveBind(gLinkContext, sEnv.DYLD_BIND_AT_LAUNCH, true);
			}
		}
	...
}
```

这里参与链接的动态库根据第 4 步中加载的插入的动态库，从 sAllImages 的第二个 imageLoader 开始，取出 image，重复 ``link`` 操作进行连接。registerInterposing 内部会加载 loadCmds 并查找 \_\_interpose 及 \_\_DATA 段，读取段信息保存到 fgInterposingTuples 中，然后调用 applyInterposing，内部调用 recursiveApplyInterposing，通过这个函数调用到 doInterpose。
	
```c++
void ImageLoaderMachOCompressed::doInterpose(const LinkContext& context)
{
	if ( context.verboseInterposing )
		dyld::log("dyld: interposing %lu tuples onto image: %s\n", fgInterposingTuples.size(), this->getPath());
	
	// update prebound symbols。更新预绑定的符号
	eachBind(context, ^(const LinkContext& ctx, ImageLoaderMachOCompressed* image,
						uintptr_t addr, uint8_t type, const char* symbolName,
						uint8_t symbolFlags, intptr_t addend, long libraryOrdinal,
						ExtraBindData *extraBindData,
						const char* msg, LastLookup* last, bool runResolver) {
		// 直接调用 interposeAt()
		return ImageLoaderMachOCompressed::interposeAt(ctx, image, addr, type, symbolName, symbolFlags,
													   addend, libraryOrdinal, extraBindData,
													   msg, last, runResolver);
	});
	eachLazyBind(context, ^(const LinkContext& ctx, ImageLoaderMachOCompressed* image,
							uintptr_t addr, uint8_t type, const char* symbolName,
							uint8_t symbolFlags, intptr_t addend, long libraryOrdinal,
							ExtraBindData *extraBindData,
							const char* msg, LastLookup* last, bool runResolver) {
		// 直接调用 interposeAt()
		return ImageLoaderMachOCompressed::interposeAt(ctx, image, addr, type, symbolName, symbolFlags,
													   addend, libraryOrdinal, extraBindData,
													   msg, last, runResolver);
	});
}
```

interposeAt 通过 interposedAddress 在上文提到的 fgInterposingTuples 中找到需要替换的符号地址进行替换。

##### 3.4.7 弱符号绑定

```c++
uintptr_t
_main(const macho_header* mainExecutableMH, uintptr_t mainExecutableSlide, 
    int argc, const char* argv[], const char* envp[], const char* apple[], 
    uintptr_t* startGlue)
{
	...
        // <rdar://problem/12186933> do weak binding only after all inserted images linked
        // 弱符号绑定
        sMainExecutable->weakBind(gLinkContext);
	...
}

void ImageLoader::weakBind(const LinkContext& context)
{
	if ( context.verboseWeakBind )
		dyld::log("dyld: weak bind start:\n");
	uint64_t t1 = mach_absolute_time();
	// get set of ImageLoaders that participate in coalecsing
	ImageLoader* imagesNeedingCoalescing[fgImagesRequiringCoalescing];
	unsigned imageIndexes[fgImagesRequiringCoalescing];
	// 合并所有动态库的弱符号到列表中
	int count = context.getCoalescedImages(imagesNeedingCoalescing, imageIndexes);
	
	// count how many have not already had weakbinding done
	int countNotYetWeakBound = 0;
	int countOfImagesWithWeakDefinitionsNotInSharedCache = 0;
	for(int i=0; i < count; ++i) {
		if ( ! imagesNeedingCoalescing[i]->weakSymbolsBound(imageIndexes[i]) )
			// 获取未进行绑定的弱符号的个数
			++countNotYetWeakBound;
		if ( ! imagesNeedingCoalescing[i]->inSharedCache() )
			// 获取在共享缓存中已绑定的弱符号个数
			++countOfImagesWithWeakDefinitionsNotInSharedCache;
	}
	
	// don't need to do any coalescing if only one image has overrides, or all have already been done
	if ( (countOfImagesWithWeakDefinitionsNotInSharedCache > 0) && (countNotYetWeakBound > 0) ) {
		// make symbol iterators for each
		ImageLoader::CoalIterator iterators[count];
		ImageLoader::CoalIterator* sortedIts[count];
		for(int i=0; i < count; ++i) {
			// 对需要绑定的弱符号排序
			imagesNeedingCoalescing[i]->initializeCoalIterator(iterators[i], i, imageIndexes[i]);
			sortedIts[i] = &iterators[i];
			if ( context.verboseWeakBind )
				dyld::log("dyld: weak bind load order %d/%d for %s\n", i, count, imagesNeedingCoalescing[i]->getIndexedPath(imageIndexes[i]));
		}
		
		// walk all symbols keeping iterators in sync by 
		// only ever incrementing the iterator with the lowest symbol 
		int doneCount = 0;
		while ( doneCount != count ) {
			//for(int i=0; i < count; ++i)
			//	dyld::log("sym[%d]=%s ", sortedIts[i]->loadOrder, sortedIts[i]->symbolName);
			//dyld::log("\n");
			// increment iterator with lowest symbol
			// 计算弱符号偏移量及大小，绑定弱符号
			if ( sortedIts[0]->image->incrementCoalIterator(*sortedIts[0]) )
				++doneCount; 
			...
        }
}
```

主要流程：合并所有动态库的弱符号到列表中 -> 对需要绑定的弱符号排序 -> 计算弱符号偏移量及大小，绑定弱符号

##### 3.4.8 初始化主程序

```c++
uintptr_t
_main(const macho_header* mainExecutableMH, uintptr_t mainExecutableSlide, 
        int argc, const char* argv[], const char* envp[], const char* apple[], 
        uintptr_t* startGlue)
{
	...
		CRSetCrashLogMessage("dyld: launch, running initializers");
	#if SUPPORT_OLD_CRT_INITIALIZATION
		// Old way is to run initializers via a callback from crt1.o
		if ( ! gRunInitializersOldWay )
			// 初始化主程序
			initializeMainExecutable(); 
	#else
		// run all initializers
		// 初始化主程序
		initializeMainExecutable(); 
	#endif
	
		// notify any montoring proccesses that this process is about to enter main()
		// 通知任何监视进程，此进程将要进入main（）。
		if (dyld3::kdebug_trace_dyld_enabled(DBG_DYLD_TIMING_LAUNCH_EXECUTABLE)) {
			dyld3::kdebug_trace_dyld_duration_end(launchTraceID, DBG_DYLD_TIMING_LAUNCH_EXECUTABLE, 0, 0, 2);
		}
		notifyMonitoringDyldMain();
	...
}
	
void initializeMainExecutable()
{
	// record that we've reached this step。开始初始化标识
	gLinkContext.startedInitializingMainExecutable = true;
	
	// run initialzers for any inserted dylibs
	ImageLoader::InitializerTimingList initializerTimes[allImagesCount()];
	initializerTimes[0].count = 0;
	const size_t rootCount = sImageRoots.size();
	if ( rootCount > 1 ) {
		for(size_t i=1; i < rootCount; ++i) {
			// 初始化动态库
			sImageRoots[i]->runInitializers(gLinkContext, initializerTimes[0]);
		}
	}
	
	// run initializers for main executable and everything it brings up
	// 初始化主程序
	sMainExecutable->runInitializers(gLinkContext, initializerTimes[0]);
	
	// register cxa_atexit() handler to run static terminators in all loaded images when this process exits
	if ( gLibSystemHelpers != NULL ) 
		(*gLibSystemHelpers->cxa_atexit)(&runAllStaticTerminators, NULL, NULL);
	
	// dump info if requested
	if ( sEnv.DYLD_PRINT_STATISTICS )
		ImageLoader::printStatistics((unsigned int)allImagesCount(), initializerTimes[0]);
	if ( sEnv.DYLD_PRINT_STATISTICS_DETAILS )
		ImageLoaderMachO::printStatisticsDetails((unsigned int)allImagesCount(), initializerTimes[0]);
}
```

先初始化动态库，然后初始化主程序。上文提到的 DYLD\_PRINT\_STATISTICS 环境变量在这里也出现了，除此之外还有个 detail 版的环境变量 <font color=#cc0000>DYLD\_PRINT\_STATISTICS_DETAILS</font>。

```c++
void ImageLoader::runInitializers(const LinkContext& context, InitializerTimingList& timingInfo)
{
	uint64_t t1 = mach_absolute_time();
	
	// 获取线程
	mach_port_t thisThread = mach_thread_self();
	ImageLoader::UninitedUpwards up;
	up.count = 1;
	up.images[0] = this;
	// 在进程中初始化
	processInitializers(context, thisThread, timingInfo, up);
	context.notifyBatch(dyld_image_state_initialized, false);
	
	mach_port_deallocate(mach_task_self(), thisThread);
	
	uint64_t t2 = mach_absolute_time();
	// 初始化耗时
	fgTotalInitTime += (t2 - t1);
}

// <rdar://problem/14412057> upward dylib initializers can be run too soon
// To handle dangling dylibs which are upward linked but not downward, all upward linked dylibs
// have their initialization postponed until after the recursion through downward dylibs
// has completed.
void ImageLoader::processInitializers(const LinkContext& context, mach_port_t thisThread,
									 InitializerTimingList& timingInfo, ImageLoader::UninitedUpwards& images)
{
	uint32_t maxImageCount = context.imageCount()+2;
	ImageLoader::UninitedUpwards upsBuffer[maxImageCount];
	ImageLoader::UninitedUpwards& ups = upsBuffer[0];
	ups.count = 0;
	// Calling recursive init on all images in images list, building a new list of
	// uninitialized upward dependencies.
	for (uintptr_t i=0; i < images.count; ++i) {
		images.images[i]->recursiveInitialization(context, thisThread, images.images[i]->getPath(), timingInfo, ups);
	}
	// If any upward dependencies remain, init them.
	if ( ups.count > 0 )
		// 递归调用
		processInitializers(context, thisThread, timingInfo, ups);
}
```

动态库和主程序的初始化是调用 runInitializers，内部通过 processInitializers 调用 recursiveInitialization 递归初始化当前 image 所依赖的库。
	
```c++
void ImageLoader::recursiveInitialization(const LinkContext& context, mach_port_t this_thread, const char* pathToInitialize,
									  InitializerTimingList& timingInfo, UninitedUpwards& uninitUps)
{
	// 递归锁
	recursive_lock lock_info(this_thread);
	recursiveSpinLock(lock_info);
	
	if ( fState < dyld_image_state_dependents_initialized-1 ) {
		uint8_t oldState = fState;
		// break cycles
		// 退出递归循环
		fState = dyld_image_state_dependents_initialized-1;
		try {
			// initialize lower level libraries first
			// 先初始化低级别的库
			for(unsigned int i=0; i < libraryCount(); ++i) {
				ImageLoader* dependentImage = libImage(i);
				if ( dependentImage != NULL ) {
					// don't try to initialize stuff "above" me yet
					// 不要试图初始化级别高于我的
					if ( libIsUpward(i) ) {
						uninitUps.images[uninitUps.count] = dependentImage;
						uninitUps.count++;
					}
					else if ( dependentImage->fDepth >= fDepth ) {
						dependentImage->recursiveInitialization(context, this_thread, libPath(i), timingInfo, uninitUps);
					}
                }
			}
			
			// record termination order. 记录终止命令
			if ( this->needsTermination() )
				context.terminationRecorder(this);
			
			// let objc know we are about to initialize this image
			uint64_t t1 = mach_absolute_time();
			fState = dyld_image_state_dependents_initialized;
			oldState = fState;
			// 
			context.notifySingle(dyld_image_state_dependents_initialized, this, &timingInfo);
			
			// initialize this image
			// 真正初始化镜像
			bool hasInitializers = this->doInitialization(context);
	
			// let anyone know we finished initializing this image
			fState = dyld_image_state_initialized;
			oldState = fState;
			context.notifySingle(dyld_image_state_initialized, this, NULL);
			
			if ( hasInitializers ) {
				uint64_t t2 = mach_absolute_time();
				timingInfo.addTime(this->getShortName(), t2-t1);
			}
		}
		catch (const char* msg) {
			// this image is not initialized
			fState = oldState;
			recursiveSpinUnLock();
			throw;
		}
	}
	
	recursiveSpinUnLock();
}
```

注意内部有个调用 context.notifySingle(dyld\_image\_state\_initialized, this, NULL)，其实每次 image 状态改变都会调用 notifySingle 这个方法：
	
```c++
static void notifySingle(dyld_image_states state, const ImageLoader* image, ImageLoader::InitializerTimingList* timingInfo)
{
	//dyld::log("notifySingle(state=%d, image=%s)\n", state, image->getPath());
	std::vector<dyld_image_state_change_handler>* handlers = stateToHandlers(state, sSingleHandlers);
	if ( handlers != NULL ) {
		dyld_image_info info;
		info.imageLoadAddress	= image->machHeader();
		info.imageFilePath		= image->getRealPath();
		info.imageFileModDate	= image->lastModified();
		for (std::vector<dyld_image_state_change_handler>::iterator it = handlers->begin(); it != handlers->end(); ++it) {
			const char* result = (*it)(state, 1, &info);
			if ( (result != NULL) && (state == dyld_image_state_mapped) ) {
				//fprintf(stderr, "  image rejected by handler=%p\n", *it);
				// make copy of thrown string so that later catch clauses can free it
				const char* str = strdup(result);
				throw str;
			}
		}
	}
	if ( state == dyld_image_state_mapped ) {
		// <rdar://problem/7008875> Save load addr + UUID for images from outside the shared cache
		if ( !image->inSharedCache() ) {
			dyld_uuid_info info;
			if ( image->getUUID(info.imageUUID) ) {
				info.imageLoadAddress = image->machHeader();
				addNonSharedCacheImageUUID(info);
			}
		}
	}
	if ( (state == dyld_image_state_dependents_initialized) && (sNotifyObjCInit != NULL) && image->notifyObjC() ) {
		uint64_t t0 = mach_absolute_time();
		dyld3::ScopedTimer timer(DBG_DYLD_TIMING_OBJC_INIT, (uint64_t)image->machHeader(), 0, 0);
		(*sNotifyObjCInit)(image->getRealPath(), image->machHeader());
		uint64_t t1 = mach_absolute_time();
		uint64_t t2 = mach_absolute_time();
		uint64_t timeInObjC = t1-t0;
		uint64_t emptyTime = (t2-t1)*100;
		if ( (timeInObjC > emptyTime) && (timingInfo != NULL) ) {
			timingInfo->addTime(image->getShortName(), timeInObjC);
		}
	}
    // mach message csdlc about dynamically unloaded images
	if ( image->addFuncNotified() && (state == dyld_image_state_terminated) ) {
		notifyKernel(*image, false);
		const struct mach_header* loadAddress[] = { image->machHeader() };
		const char* loadPath[] = { image->getPath() };
		notifyMonitoringDyld(true, 1, loadAddress, loadPath);
	}
}
```

当 state == dyld\_image\_state\_mapped 时，将 image 对应的 UUID 存起来，当state == dyld\_image\_state\_dependents\_initialized 并且有 sNotifyObjCInit 回调时调用sNotifyObjCInit函数。
	
搜索回调函数赋值入口：
	
```c++
void registerObjCNotifiers(_dyld_objc_notify_mapped mapped, _dyld_objc_notify_init init, _dyld_objc_notify_unmapped unmapped)
{
    // record functions to call
    sNotifyObjCMapped   = mapped;
    sNotifyObjCInit     = init;
    sNotifyObjCUnmapped = unmapped;
	
	...
}
	
void _dyld_objc_notify_register(_dyld_objc_notify_mapped    mapped,
                                _dyld_objc_notify_init      init,
                                _dyld_objc_notify_unmapped  unmapped)
{
    dyld::registerObjCNotifiers(mapped, init, unmapped);
}
```

发现是通过 \_dyld\_objc\_notify\_register 这个函数注册回调的。
	
[NSObject load] 的堆栈：
	
```oc
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.2
  * frame #0: 0x000000010944f3b1 libobjc.A.dylib`+[NSObject load]
    frame #1: 0x000000010943d317 libobjc.A.dylib`call_load_methods + 691
    frame #2: 0x000000010943e814 libobjc.A.dylib`load_images + 77
    frame #3: 0x0000000108b73b97 dyld_sim`dyld::registerObjCNotifiers(void (*)(unsigned int, char const* const*, mach_header const* const*), void (*)(char const*, mach_header const*), void (*)(char const*, mach_header const*)) + 260
    frame #4: 0x000000010b779bf3 libdyld.dylib`_dyld_objc_notify_register + 113
    frame #5: 0x000000010944ca12 libobjc.A.dylib`_objc_init + 115
    frame #6: 0x000000010b7015c0 libdispatch.dylib`_os_object_init + 13
    frame #7: 0x000000010b70f4e5 libdispatch.dylib`libdispatch_init + 300
    frame #8: 0x0000000109e05a78 libSystem.B.dylib`libSystem_initializer + 164
    frame #9: 0x0000000108b82b96 dyld_sim`ImageLoaderMachO::doModInitFunctions(ImageLoader::LinkContext const&) + 506
    frame #10: 0x0000000108b82d9c dyld_sim`ImageLoaderMachO::doInitialization(ImageLoader::LinkContext const&) + 40
    frame #11: 0x0000000108b7e3fc dyld_sim`ImageLoader::recursiveInitialization(ImageLoader::LinkContext const&, unsigned int, char const*, ImageLoader::InitializerTimingList&, ImageLoader::UninitedUpwards&) + 324
    frame #12: 0x0000000108b7e392 dyld_sim`ImageLoader::recursiveInitialization(ImageLoader::LinkContext const&, unsigned int, char const*, ImageLoader::InitializerTimingList&, ImageLoader::UninitedUpwards&) + 218
    frame #13: 0x0000000108b7d5d3 dyld_sim`ImageLoader::processInitializers(ImageLoader::LinkContext const&, unsigned int, ImageLoader::InitializerTimingList&, ImageLoader::UninitedUpwards&) + 133
    frame #14: 0x0000000108b7d665 dyld_sim`ImageLoader::runInitializers(ImageLoader::LinkContext const&, ImageLoader::InitializerTimingList&) + 73
    frame #15: 0x0000000108b71333 dyld_sim`dyld::initializeMainExecutable() + 129
    frame #16: 0x0000000108b75434 dyld_sim`dyld::_main(macho_header const*, unsigned long, int, char const**, char const**, char const**, unsigned long*) + 4384
    frame #17: 0x0000000108b70630 dyld_sim`start_sim + 136
    frame #18: 0x00000001155c1234 dyld`dyld::useSimulatorDyld(int, macho_header const*, char const*, int, char const**, char const**, char const**, unsigned long*, unsigned long*) + 2238
    frame #19: 0x00000001155bf0ce dyld`dyld::_main(macho_header const*, unsigned long, int, char const**, char const**, char const**, unsigned long*) + 522
    frame #20: 0x00000001155ba503 dyld`dyldbootstrap::start(macho_header const*, int, char const**, long, macho_header const*, unsigned long*) + 1167
    frame #21: 0x00000001155ba036 dyld`_dyld_start + 54
```

可以看到，\_dyld\_objc\_notify\_register 是在初始化 libobjc.A.dylib 这个动态库时调用的，然后 \_objc\_init 内部调用了 load\_images，进而调用 call\_load\_methods，从而调用各个类中的load方法，[Objc源码](https://links.jianshu.com/go?to=https%3A%2F%2Fopensource.apple.com%2Ftarballs%2Fobjc4%2F)。
	
notifySingle 调用完毕后，开始真正初始化工作 <font color=#cc0000>``doInitialization``</font>：
	
```c++
bool ImageLoaderMachO::doInitialization(const LinkContext& context)
{
    CRSetCrashLogMessage2(this->getPath());
	
    // mach-o has -init and static initializers
    doImageInit(context);
    doModInitFunctions(context);
    
    CRSetCrashLogMessage2(NULL);
    
    return (fHasDashInit || fHasInitializers);
}
```

doImageInit 执行 ``LC_ROUTINES_COMMAND`` segment 中保存的函数，doModInitFunctions执行 \_\_DATA,\_\_mod\_init\_func section 中保存的函数。这个 section 中保存的是 C++ 的构造函数及带有 attribute((constructor)) 的 C 函数，简单验证一下：
	
```oc
// ViewController.mm
	
class Test {
public:
    Test();
};
	
Test::Test(){
    NSLog(@"%s", __func__);
}
	
Test test;
	
__attribute__((constructor)) void testConstructor() {
    NSLog(@"%s", __func__);
}
	
- (void)viewDidLoad 
{
    [super viewDidLoad];
	
    testConstructor();
	
    Test * t = new Test();
}
	
2019-08-19 13:26:33.587051+0800 Demo[7105:314102] testConstructor
2019-08-19 13:26:33.587109+0800 Demo[7105:314102] Test
```

通过 MachOView 可以看到：

显然，\_\_mod\_init\_func 中的函数在类对应的 load 方法之后调用。

1. 对于 dumpdcrypted 这一类注入方法实现功能的插件，他们添加的静态方法会在 doModInitFunctions方法中被解析出来，位置在 MachO 文件的 \_\_DATA 段的 \_\_mod\_init\_func section。C++ 的全局对象也会出现在这个section中。
2. 在递归初始化 (recursiveInitialization）中，如果当前执行的是主程序 image，doInitialization 完毕后会执行 notifySingle 方法去通知观察者。在 doInitialization 方法前会发送 state 为 dyld\_image\_state\_dependents\_initialized 的通知，由这个通知，会调用 libobjc 的 load\_images，最后去依次调用各个 OC 类的 load 方法以及分类的 load 方法。
3. Objective-C 的入口方法是 \_objc\_init，dyld 唤起它的执行路径是从 runInitializers -> recursiveInitialization -> doInitialization -> doModInitFunctions ->.. \_objc\_init。

	```c++
	void _objc_init(void)
	{       
		// Register for unmap first, in case some +load unmaps something
		_dyld_register_func_for_remove_image(&unmap_image);
		dyld_register_image_state_change_handler(dyld_image_state_bound,
		                                         1/*batch*/, &map_2_images);
		dyld_register_image_state_change_handler(dyld_image_state_dependents_initialized, 0/*not batch*/, &load_images);
	}
	```

4. \_objc\_init 会在 dyld 中注册两个通知，对应的回调会分别执行将 OC 类加载到内存和调用 load 方法的操作。后面的就是 OC 类加载的经典方法 map\_2\_images 了。

5. 从 recursiveInitialization 的以下代码片段可以看出 load 是在全局实例或者方法调用前被触发的。

	```c++
        context.notifySingle(dyld_image_state_dependents_initialized, this, &timingInfo);
        // initialize this image
        bool hasInitializers = this->doInitialization(context);
        // let anyone know we finished initializing this image
        fState = dyld_image_state_initialized;
        oldState = fState;
        context.notifySingle(dyld_image_state_initialized, this, NULL);
	```

##### 3.4.9 查找主程序入口函数指针并返回

调用getEntryFromLC_MAIN 获取主程序 main 函数的地址，如果未找到则调用getEntryFromLC_UNIXTHREAD 获取。

```c++
void* ImageLoaderMachO::getEntryFromLC_MAIN() const
{
	const uint32_t cmd_count = ((macho_header*)fMachOData)->ncmds;
	const struct load_command* const cmds = (struct load_command*)&fMachOData[sizeof(macho_header)];
	const struct load_command* cmd = cmds;
	for (uint32_t i = 0; i < cmd_count; ++i) {
		if ( cmd->cmd == LC_MAIN ) {
			entry_point_command* mainCmd = (entry_point_command*)cmd;
			void* entry = (void*)(mainCmd->entryoff + (char*)fMachOData);
			// <rdar://problem/8543820&9228031> verify entry point is in image
			if ( this->containsAddress(entry) )
				return entry;
			else
				throw "LC_MAIN entryoff is out of range";
		}
		cmd = (const struct load_command*)(((char*)cmd)+cmd->cmdsize);
	}
	return NULL;
}
	
	
void* ImageLoaderMachO::getEntryFromLC_UNIXTHREAD() const
{
	const uint32_t cmd_count = ((macho_header*)fMachOData)->ncmds;
	const struct load_command* const cmds = (struct load_command*)&fMachOData[sizeof(macho_header)];
	const struct load_command* cmd = cmds;
	for (uint32_t i = 0; i < cmd_count; ++i) {
		if ( cmd->cmd == LC_UNIXTHREAD ) {
	#if __i386__
			const i386_thread_state_t* registers = (i386_thread_state_t*)(((char*)cmd) + 16);
			void* entry = (void*)(registers->eip + fSlide);
			// <rdar://problem/8543820&9228031> verify entry point is in image
			if ( this->containsAddress(entry) )
				return entry;
	#elif __x86_64__
			const x86_thread_state64_t* registers = (x86_thread_state64_t*)(((char*)cmd) + 16);
			void* entry = (void*)(registers->rip + fSlide);
			// <rdar://problem/8543820&9228031> verify entry point is in image
			if ( this->containsAddress(entry) )
				return entry;
	#elif __arm64__ && !__arm64e__
			// temp support until <rdar://39514191> is fixed
			const uint64_t* regs64 = (uint64_t*)(((char*)cmd) + 16);
			void* entry = (void*)(regs64[32] + fSlide); // arm_thread_state64_t.__pc
			// <rdar://problem/8543820&9228031> verify entry point is in image
			if ( this->containsAddress(entry) )
				return entry;
	#endif
		}
		cmd = (const struct load_command*)(((char*)cmd)+cmd->cmdsize);
	}
	throw "no valid entry point";
}
```

可以看到，入口是在 load_command 的 LC_MAIN 或者 LC_UNIXTHREAD 中 LC_MAIN。

## 四、dyld 闭包

在第 2 步和第 3 步之间有一个查找闭包并以其结果作为程序入口返回的代码，这里是 [WWDC 2017](https://developer.apple.com/videos/play/wwdc2017/413/) 推出的 dyld3 中提出的一种优化 App 启动速度的技术。大致步骤如下：

1. 如果满足条件：开启闭包（DYLD\_USE\_CLOSURES 环境变量为 1），App 的路径在白名单中（目前只有系统 Ap p享有使用闭包的特权），共享缓存加载地址不为空，则往下执行。
2. 去内存中查找闭包数据，这里的方法是 findClosure。如果内存中不存在，再去 ``/private/var/staged_system_apps`` 路径下去查找硬盘数据，找到就返回结果。
3. 如果没有闭包数据，就会调用 socket 通信走 RPC 去获取闭包数据，执行方法为 callClosureDaemon，感兴趣可以研究下。
4. 如果闭包数据不为空，就会走核心方法：launchWithClosure，基于闭包去启动 App，并且返回该方法中获取的程序入口地址给外界。这个方法重复了上面的各个步骤。具体实现和内部的数据结构有待分析。

## 五、共享缓存机制

在 iOS 系统中，每个程序依赖的动态库都需要通过 dyld 一个一个加载到内存，然而，很多系统库几乎是每个程序都会用到的，如果在每个程序运行的时候都重复的去加载一次，势必造成运行缓慢，为了优化启动速度和提高程序性能，共享缓存机制就应运而生。所有默认的动态链接库被合并成一个大的缓存文件，放到 <font color=#cc0000>``/System/Library/Caches/com.apple.dyld/``</font> 目录下，按不同的架构保存分别保存着，原作者的 iPhone6 里面就有 dyld\_shared\_cache\_armv7s 和 dyld\_shared\_cache\_armv64 两个文件，如下图所示。

想要分析某个系统库，就需要从 dyld\_shared\_cache 里先将的原始二进制文件提取出来，这里从易到难提供 3 种方法：

#### 5.1 dyld\_cache\_extract 提取

[dyld\_cache\_extract](https://github.com/macmade/dyld_cache_extract) 是一个可视化的工具，使用极其简单，把 dyld\_shared\_cache 载入即可解析出来，如下图所示。

#### 5.2 jtool 提取

以提取 CFNetwork 为例，使用如下命令即可：

```
$ jtool -extract CFNetwork ./dyld_shared_cache_arm64
```

#### 5.3 dsc\_extractor 提取

在 dyld 源代码的 launch-cache 文件夹里面找到 dsc\_extractor.cpp，将 653 行的“#if 0”修改为“#if 1”，然后用如下命令编译生成 dsc\_extractor，并使用它提取所有缓存文件：

```
$ clang++ dsc_extractor.cpp dsc_iterator.cpp  -o dsc_extractor
$ ./dsc_extractor ./dyld_shared_cache_arm64 ./
```

## 六、总结

每个 MachO 都会由一个 imageLoader 来处理加载和依赖管理的操作，这里是由 dyld 来安排。主程序 app 的 image 的加载是由内核来完成的。其他的动态库的加载细节可以参考上面提到的 link 方法实现，当一个 image 加载完毕，dyld 会发送 dyld\_image\_state\_bound 通知；著名的 hook 工具 fishhook 的实现原理也是借助监听这个通知，在回调里完成 hook 操作的。

## 七、文章

[01_Jack](https://www.jianshu.com/u/02a488e1e71e) & [dyld源码解读](https://www.jianshu.com/p/82e6fdaa0e41)
[伊织__](https://me.csdn.net/lovechris00) & [Mac - otool](https://blog.csdn.net/lovechris00/article/details/81561627#otool_0)
[RemisKrlet](https://www.jianshu.com/u/3b5a95e93778) & [App启动过程 - dyld加载动态库](https://www.jianshu.com/p/72e34948dac0)
[dyld详解](https://www.dllhook.com/post/238.html)