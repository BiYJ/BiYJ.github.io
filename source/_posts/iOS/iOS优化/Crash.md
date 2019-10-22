---
title: Crash
categories: iOS优化
---

## 一、Crash类型

crash 一般产生自 iOS 的微内核 Mach，然后在 BSD 层转换成 UNIX SIGABRT 信号，以标准 POSIX 信号的形式提供给用户。NSException 是使用者在处理 App 逻辑时，用编程的方法抛出。

iOS 端的 crash 分为三类：

- Mach 异常：EXC\_CRASH
- UNIX 信号：SIGABRT
- 系统崩溃而引起的程序 NSException 异常退出

[常见的 iOS 崩溃类型](https://cnbin.github.io/blog/2016/03/15/ioszhong-de-beng-kui-lei-xing/)有：

1. EXC\_BAD\_ACCESS

	> 在访问一个已经释放的对象或向它发送消息时，就会出现 EXC\_BAD\_ACCESS。
	
	造成 EXC\_BAD\_ACCESS 最常见的原因，是在初始化变量时用错了所有权修饰符，这会导致对象过早地被释放。例如，一个 NSArray 属性的所有权修饰符设成了 assign 而不是 strong。
	
	这个崩溃发生时，查看崩溃日志，往往得不到有用的栈信息。可以通过 NSZombieEnabled 来解决。

	NSZombieEnabled 是一个环境变量，用来调试与内存相关的问题，跟踪对象的释放过程。启用之后，它会在对象调用 dealloc 时，也就是在引用计数降到 0，用一个僵尸实现将该对象转换成僵尸对象。僵尸对象的作用是在你向它发送消息时，它会显示一段日志并自动跳入调试器。

	所以，当在应用中启用 NSZombie 而不是让应用直接崩溃时，一个错误的内存访问就会变成一条无法识别的消息发送给僵尸对象。僵尸对象会显示接收到的消息，然后跳入调试器，这样就可以查看到底哪里出了问题。 启用如图所示：
	
	<center>
	![](http://dzliving.com/ZombieEnabled.png)
	</center>
	
2. SIGSEGV

	> 段错误信息（SIGSEGV）是操作系统产生的一个更严重的问题。
	
	当硬件出现错误、访问不可读的内存地址或向受保护的内存地址写入数据时，就会发生这个错误。
	
	当要读取保存在 RAM 中的数据，而该位置的 RAM 硬件有问题时，会收到 SIGSEGV，这种情况并不常见。下面两种情况更多出现：
	
	- 当应用中的某个指针指向代码页并试图修改指向位置的值；
	- 当要读取一个指针的值，而它被初始化成指向无效内存地址的垃圾值

	SIGSEGV 错误调试起来更困难，而导致 SIGSEGV 的最常见原因是<font color=#cc0000>不正确的类型转换</font>。要避免过度使用指针或尝试手动修改指针来读取私有数据结构。如果你那样做了，而在修改指针时没有注意内存对齐和填充问题，就会收到 SIGSEGV。

3. SIGBUS

	> 总线错误信号（SIGBUG）代表无效内存访问，即访问的内存是一个无效的内存地址。
	
	也就是说，那个地址指向的位置根本不是物理内存地址（它可能是某个硬件芯片的地址）。SIGSEGV 和 SIGBUS 都 EXC\_BAD\_ACCESS 的子类型。

4. SIGTRAP

	> SIGTRAP 代表陷阱信号。
	
	它并不是一个真正的崩溃信号。它会在处理器执行 trap 指令发送。LLDB 调试器通常会处理此信号，并在指定的断点处停止运行。如果你收到了原因不明的 SIGTRAP，先清除上次的输出，然后重新进行构建通常能解决这个问题。

5. EXC_ARITHETIC

	> 当要作除零运算时，应用会收到 EXC_ARITHMETIC 信号。
	
	这个错误应该很容易解决。

6. SIGILL

	> SIGILL 代表 signal illegal instruction(非法指令信号)。
	
	当在处理器上执行非法指令时，它就会发生。执行非法指令是指，将函数指针会给另外一个函数时，该函数指针由于某种原因是坏的，指向了一段已经释放的内存或是一个数据段。有时你收到的是 EXC\_BAD\_INSTRUCTION 而不是SIGILL，虽然它们是一回事，不过 EXC\_* 等同于此信号不依赖体系结构。

7. SIGABRT

	> SIGABRT 代表 SIGNAL ABORT（中止信号）。
	
	当操作系统发现不安全的情况时，它能够对这种情况进行更多的控制；必要的话，它能要求进程进行清理工作。在调试造成此信号的底层错误时，并没有什么妙招。Cocos2d 或 UIKit 等框架通常会在特定的前提条件没有满足或一些糟糕的情况出现时调用 C 函数 abort（由它来发送此信号）。
	
	当 SIGABRT 出现时，控制台通常会输出大量的信息，说明具体哪里出错了。由于它是可控制的崩溃，所以可以在 LLDB 控制台上键入 ``bt`` 命令打印出回溯信息。

8. 看门狗超时

	这种崩溃通常比较容易分辨，因为错误码是固定的 0x8badf00d。（程序员幽默的把它读作 Ate Bad Food。）在 iOS 上，它经常出现在执行一个同步网络调用而阻塞主线程的情况。因此，永远不要进行同步网络调用。

## 二、Crash捕获

日常开发中，可以使用的 crash 收集方式有：

- 第三方平台：Fabric、友盟、腾讯 Bugly、[Flurry](https://www.flurry.com/)、[Crashlytics](https://www.jianshu.com/p/ee5ccb0d39af) 等，数据会上传到这些平台
- 第三方工具：KSCrash、plcrashreporter 等，可自行处理收集的 crash
- 自定义捕获 + 堆栈符号化

本文主要讨论自定义捕获。

#### 1.1 Mach 异常捕获

如果想要做 mach 异常捕获，需要注册一个异常端口，这个异常端口会对当前任务的所有线程有效，如果想要针对单个线程，可以通过 thread\_set\_exception\_ports 注册自己的异常端口，发生异常时，首先会将异常抛给线程的异常端口，然后尝试抛给任务的异常端口，当我们捕获异常时，就可以做一些自己的工作，比如，当前堆栈收集等。

#### 1.2 NSException 异常捕获

> NSException 异常是 OC 代码导致的 crash。

NSException 异常和 Signal 信号异常，这两类都可以通过<font color=#cc0000>注册相关函数</font>来捕获。

```
// 保存注册的 exception 捕获方法
NSUncaughtExceptionHandler * oldExceptionHandler;
// 自定义的 exception 异常处理
void ExceptionHandler(NSException * exception);

void RegisterExceptionHandler() 
{
    if(NSGetUncaughtExceptionHandler() != ExceptionHandler) {
        oldExceptionHandler = NSGetUncaughtExceptionHandler();
    }
    NSSetUncaughtExceptionHandler(ExceptionHandler);
}
```

NSSetUncaughtExceptionHandler 用来做异常处理，功能非常有限。引起崩溃的大多数原因如：内存访问错误、重复释放等错误，它就无能为力了，因为这种错误它抛出的是 Signal。

同时值得注意

> 如果一个应用中注册了多个 crash 收集组件，必然会存在冲突问题。

这个时候，我们需要在注册之前判断是否已经注册过 handler，如果有注册过，需要把之前注册的 handler函数指针保存，待处理完 crash 后，再把对应的 handler 抛出去。

```
/**
 *  @brief  exception 崩溃处理
 */
void ExceptionHandler(NSException * exception)
{
    // 使 UncaughtExceptionCount 递增
    int32_t exceptionCount = OSAtomicIncrement32(&UncaughtExceptionCount);
    
    // 超出允许捕获错误的次数
    if (exceptionCount > UncaughtExceptionMaximum) {
        return;
    }
    
    // 获取调用堆栈
    NSMutableDictionary * userInfo = [NSMutableDictionary dictionaryWithDictionary:[exception userInfo]];
    userInfo[kUncaughtCallStackKey] = [exception callStackSymbols];
    
    NSException * exp = [NSException exceptionWithName:exception.name
                                                reason:exception.reason
                                              userInfo:userInfo];
    // 在主线程中执行方法
    [[[UncaughtExceptionHandler alloc] init] performSelectorOnMainThread:@selector(dealException:)
                                                              withObject:exp
                                                           waitUntilDone:YES];
    
    // 调用保存的 handler
    if (oldExceptionHandler) {
        oldExceptionHandler(exception);
    }
}
```

#### 1.3 Signal 信号捕获

> Signal 信号是由 iOS 底层 mach 信号异常转换后以 signal 信号抛出的异常。

既然是<font color=#cc0000>兼容 posix 标准</font>的异常，我们同样可以通过 sigaction 函数注册对应的信号。

因为 signal 信号有很多，有些信号在 iOS 应用中也不会产生，我们只需要注册常见的几类信号：

|信号| 值|介绍|场景|
|:----:|:---:|:----:|:----:|
| SIGILL |4|非法指令|1. 执行了非法指令. <br>2. 通常是因为可执行文件本身出现错误或者试图执行数据段. <br>3. 堆栈溢出时也有可能产生这个信号.|
|SIGABRT|6|调用abort| 程序自己发现错误并调用 abort 时产生，一些 C 库函数（如：strlen）|
|SIGSFPE|8|浮点运算错误|如：除 0 操作|
|SIGSEGV|11|段非法错误|1. 试图访问未分配给自己的内存<br>2. 或试图往没有写权限的内存地址写数据<br>3. 空指针<br>4. 数组越界<br>5. 栈溢出等|

下面注册一个 SIGABRT 信号，在注册 handler 之前，需要保存之前注册的 hander:

```
typedef void (* SignalHandlerClass)(int, struct __siginfo *, void *);

// 已注册的 singal 捕获方法
SignalHandlerClass oldSignalHandler;

static void MySignalHandler(int signal, siginfo_t* info, void* context) {
    
    // do something。。。
    
    if (signal == SIGABRT) {
	    if (oldSignalHandler) {
	        oldSignalHandler(signal, info, context);
	    }
    }
}

void registerSignalHandler()
{
    // 获取已注册的 handler
    struct sigaction old_action;
    sigaction(SIGABRT, NULL, &old_action);
    if (old_action.sa_flags & SA_SIGINFO) {
        SignalHandlerClass handler = old_action.sa_sigaction;
        if (handler != MySignalHandler) {
            oldSignalHandler = handler;
        }
    }
    
    struct sigaction action;
    action.sa_sigaction = MySignalHandler;
    action.sa_flags = SA_NODEFER | SA_SIGINFO;
    sigemptyset(&action.sa_mask);
    sigaction(signal, &action, 0);
}
```

## 三、收集调用堆栈

调用堆栈的收集我们可以利用系统 api，也可以参考 PLCrashRepoter 等第三方实现获取所有线程堆栈。使用系统 api 关键代码如下:


```
+ (NSArray *)backtrace
{
    /*  指针列表。

        ①、backtrace 用来获取当前线程的调用堆栈，获取的信息存放在这里的 callstack 中
        ②、128 用来指定当前的 buffer 中可以保存多少个 void* 元素
     */
    void * callstack[128];
    
    // 返回值是实际获取的指针个数
    int frames = backtrace(callstack, 128);
    
    // backtrace_symbols 将从 backtrace 函数获取的信息转化为一个字符串数组，每个字符串包含了一个相对于 callstack 中对应元素的可打印信息，包括函数名、偏移地址、实际返回地址。
    // 返回一个指向字符串数组的指针
    char **strs = backtrace_symbols(callstack, frames);
    
    NSMutableArray * backtrace = [NSMutableArray arrayWithCapacity:frames];
    for (int i = 0; i < frames; i++) {
        [backtrace addObject:[NSString stringWithUTF8String:strs[i]]];
    }
    free(strs);
    
    return backtrace;
}
```
        

## 四、堆栈符号化

通过系统 api 获取的堆栈信息可能只是一串内存地址，很难从中获取有用的信息协助排查问题，因此，需要对堆栈信息符号化。

```
// 未符号化前
Thread 0 name:  Dispatch queue: com.apple.main-thread
Thread 0 Crashed:
0   libobjc.A.dylib                 0x000000018b816f30 0x18b7fc000 + 110384 (objc_msgSend + 16)
1   UIKit                           0x0000000192e0a79c 0x192c05000 + 2119580 (<redacted> + 72)
2   UIKit                           0x0000000192c4db48 0x192c05000 + 297800 (<redacted> + 312)
3   UIKit                           0x0000000192c4d988 0x192c05000 + 297352 (<redacted> + 160)
4   QuartzCore                      0x00000001900d6404 0x18ffc5000 + 1119236 (<redacted> + 260)

// 符号化后
Thread 0 name:  Dispatch queue: com.apple.main-thread
Thread 0 Crashed:
0   libobjc.A.dylib                 0x000000018b816f30 objc_msgSend + 16
1   UIKit                           0x0000000192e0a79c -[UISearchDisplayController _sendDelegateDidBeginDidEndSearch] + 72
2   UIKit                           0x0000000192c4db48 -[UIViewAnimationState sendDelegateAnimationDidStop:finished:] + 312
3   UIKit                           0x0000000192c4d988 -[UIViewAnimationState animationDidStop:finished:] + 160
4   QuartzCore                      0x00000001900d6404 CA::Layer::run_animation_callbacks(void*) + 260
```

符号化的思路是找到当前应用对于的 <font color=#cc0000>dsym 符号表文件</font>，利用 symbolicatecrash（Xcode 的 Organizer 内置了）、dwarfdump，atos 等工具还原 crash 堆栈内存地址对应的符号名。需要注意，如果应用中使用了自己或第三方的动态库，应用崩溃在动态库 Image 而不是主程序 Image 中，我们需要有对应动态库的 dsym 符号表才能符号化。

思路明确之后，接下来面临的是两个问题。一个问题是如何把当前 crash 的应用和 dsym 符号表对应上。另一个问题是如何通过内存地址符号化。在解决这两个问题之前，我们需要先了解可执行文件的二进制格式和加载过程。

#### 4.1 Mach-O文件格式

不同操作系统都会定义不同的可执行文件格式。如 Linux平台的 ELF 格式，Windows 平台的 PE 格式，iOS 的可执行文件格式被称作 Mach-O。可执行文件、动态库、dsym 文件都是这种文件格式。

下图是官方的 Mach-O 格式结构：

<center>
![Mach-O文件格式](http://dzliving.com/MachO_1.png)
</center>

可以看到，Mach-O 文件分为三部分。

1. **header**

	hander 定义了文件的基本信息，包括文件大小、文件类型、使用的平台等信息。我们可以从 loader.h 头文件中找到相关定义：
	
	```
	/*
	 * The 64-bit mach header appears at the very beginning of object files for
	 * 64-bit architectures.
	 */
	struct mach_header_64 {
	    uint32_t    magic;      /* mach magic number identifier */
	    cpu_type_t  cputype;    /* cpu specifier */
	    cpu_subtype_t   cpusubtype; /* machine specifier */
	    uint32_t    filetype;   /* type of file */
	    uint32_t    ncmds;      /* number of load commands */
	    uint32_t    sizeofcmds; /* the size of all the load commands */
	    uint32_t    flags;      /* flags */
	    uint32_t    reserved;   /* reserved */
	};
	```

2. **load commands**

	这一部分定义了详细的加载指令，指明如何加载到内存。
	
	从头文件定义可以看到，基础的 load_command 结构体只包含了 cmd 以及 cmdsize。通过 cmd 类型，可以转义成不同类型的 load command 结构体：

	```
	struct load_command {
	    uint32_t cmd;       /* type of load command */
	    uint32_t cmdsize;   /* total size of command in bytes */
	};
	```

3. **数据部分**

	包括了代码段、数据段、符号表等具体的二进制数据。

	我们可以用 otool 查看二进制文件的具体内容，更直观的，可以用 Mach-O View 来浏览可执行文件的具体内容。

	下图是一个可执行文件与其所对应的符号表文件。可执行文件的 load command 比较多，里面包含了有代码段、数据段、函数入口、加载动态库等指令。其中的 LC\_UUID 字段和符号表中的 LC\_UUID 是完全对应的，也就是说，可以<font color=#cc0000>通过 UUID 字段匹配可执行文件和 dsym 符号表</font>。

<center>
![可执行文件](http://dzliving.com/MachO_2.png)
![符号表文件](http://dzliving.com/MachO_3.png)
</center>

#### 4.2 可执行文件加载过程

一个 iOS 应用的加载过程是这样的：

1. 首先，由内核加载可执行文件（Mach-O），并从中获得 dyld 的路径。
2. 然后加载 dyld，由 dyld 接管动态库加载，符号绑定等工作，runtime 的初始化工作也在这一阶段进行。
3. 最后 dyld 调用 main 函数，这样便来到了 main 函数入口。

在这个过程中，操作系统为了安全考虑，使用了 <font color=#cc0000>ASLR</font> 技术。地址空间布局随机化(Address space layout randomization)，就是每次应用加载时，使用随机的一个地址空间，这样能有效防止被攻击。

VM Address 是编译后 Image 的起始位置，Load Address 是在运行时加载到虚拟内存的起始位置，Slide 是加载到内存的偏移，这个偏移值是一个随机值，每次运行都不相同，有下面公式：

> Load Address = VM Address + Slide

由于 dsym 符号表是编译时生成的地址，crash 堆栈的地址是运行时地址，这个时候需要经过转换才能正确的符号化。crash 日志里的符号地址被称为 Stack Address，而编译后的符号地址被称为 Symbol Address，他们之间的关系如下：

> Stack Address = Symbol Address + Slide

<font color=#cc0000>符号化就是通过 Symbol Address 到 dsym 文件中寻找对应符号信息的过程</font>。

#### 4.3 获取 Binary Images 信息

当前采集到的 crash 日志，报错地址 Stack Address 位于 0x1046eea14，相对 Load Address 0x1046e8000 偏移了 27156。这里的 27156 并不是 ASLR 的随机偏移Slide，而是符号相对位置offset（Symbol Address - VM Address）：

<center>
![报错堆栈](http://dzliving.com/Crash_1.png)
</center>

再观察 crash 日志最后有一栏 Binary Images，记录了所有加载 image 的 UUID 和加载的 Load Address：

<center>
![Binary Images](http://dzliving.com/Crash_2.png)
</center>

根据前文提到的 UUID 对应关系以及 Load Address 和 Symbol Address 的转换关系，只要能获取 Binary Images 信息，就可以实现符号化。

UUID 存放在 Mach-O 的 load command 中，对应 uuid\_command 结构体的 uuid 字段，可以通过遍历所有 load command 获取。

Slide 偏移可以通过 image\_dyld\_get\_image\_vmaddr\_slide 方法遍历所有 Image 获取。

VM Address 也存放在 load command 中，对应 segment\_command 结构体的 vmaddr 字段，需要注意 segment\_command 存在多种类型以及需要区分32位和64位应用的细微差别。

解析代码如下：

```
for (uint32_t i = 0; i < _dyld_image_count(); i++) {
        uint64_t vmbase = 0;
        uint64_t vmslide = 0;
        uint64_t vmsize = 0;
        
        uint64_t loadAddress = 0;
        uint64_t loadEndAddress = 0;
        NSString *imageName = @"";
        NSString *uuid;
        
        const struct mach_header *header = _dyld_get_image_header(i);
        const char *name = _dyld_get_image_name(i);
        vmslide = (i);
        imageName = [NSString stringWithCString:name encoding:NSUTF8StringEncoding];
        BOOL is64bit = header->magic == MH_MAGIC_64 || header->magic == MH_CIGAM_64;
        uintptr_t cursor = (uintptr_t)header + (is64bit ? sizeof(struct mach_header_64) : sizeof(struct mach_header));
        struct load_command *loadCommand = NULL;
        for (uint32_t i = 0; i < header->ncmds; i++, cursor += loadCommand->cmdsize) {
            loadCommand = (struct load_command *)cursor;
            if(loadCommand->cmd == LC_SEGMENT) {
                const struct segment_command* segmentCommand = (struct segment_command*)loadCommand;
                if (strcmp(segmentCommand->segname, SEG_TEXT) == 0) {
                    vmsize = segmentCommand->vmsize;
                    vmbase = segmentCommand->vmaddr;
                }
            } else if(loadCommand->cmd == LC_SEGMENT_64) {
                const struct segment_command_64* segmentCommand = (struct segment_command_64*)loadCommand;
                 if (strcmp(segmentCommand->segname, SEG_TEXT) == 0) {
                    vmsize = segmentCommand->vmsize;
                    vmbase = (uintptr_t)(segmentCommand->vmaddr);
                }
            }
            else if (loadCommand->cmd == LC_UUID) {
                const struct uuid_command *uuidCommand = (const struct uuid_command *)loadCommand;
                NSString *uuidString = [[[NSUUID alloc] initWithUUIDBytes:uuidCommand->uuid] UUIDString];
                uuid = [[uuidString stringByReplacingOccurrencesOfString:@"-" withString:@""] lowercaseString];
            }
        }
        
        loadAddress = vmbase + vmslide;
        loadEndAddress = loadAddress + vmsize - 1;
    }
  // do something...
```  
  
#### 4.4 符号化

通过上述代码，我们可以采集到和系统一样的 crash 日志。接下来，可以使用 dwarfdump 和 atos 进行符号化。

1. dwarfdump

	拿到 crash 日志后，我们要先确定 dsym 文件是否匹配。可以使用 dwarfdump --uuid 命令查看 dsym 文件所有架构的 UUID：
	
	```
	$ dwarfdump --uuid mytest.app.dSYM 
	UUID: B4217D5B-0349-3D9F-9D70-BC7DD60DA121 (armv7) mytest.app.dSYM/Contents/Resources/DWARF/mytest
	UUID: A52E3452-C2EF-3291-AE37-9392EDCCE572 (arm64) mytest.app.dSYM/Contents/Resources/DWARF/mytest
	```
	
	可以看到 dsym 文件的 arm64 架构中包含的 A52E3452-C2EF-3291-AE37-9392EDCCE572 和 Binary Images 中的 UUID 是相匹配的。
	
	<center>
	![UUID](http://dzliving.com/Crash_3.png)
	</center>

	下面就可以用 dwarfdump --lookup 命令对报错堆栈符号化，格式如下：

	```
	dwarfdump --arch [arch type] --lookup [Symbol Address] [dsym file path]
	```

	对于报错堆栈的 Stack Address 0x1046eea14，需要进行一个转换。已知 VM Address 为0x100000000，Load Address 为 0x1046e8000，可以得到 Slide 为 0x46e8000。通过公式Symbol Address = Stack Address - Slider 求得 Symbol Address 为 0x100006a14，输入命令：
	
	```
	$ dwarfdump --arch arm64 --lookup 0x100006a14 mytest.app.dSYM 
	----------------------------------------------------------------------
	 File: mytest.app.dSYM/Contents/Resources/DWARF/mytest (arm64)
	----------------------------------------------------------------------
	Looking up address: 0x0000000100006a14 in .debug_info... found!
	
	0x0003ebb7: Compile Unit: length = 0x000000d4  version = 0x0004  abbr_offset = 0x00000000  addr_size = 0x08  (next CU at 0x0003ec8f)
	
	0x0003ebc2: TAG_compile_unit [120] *
	             AT_producer( "Apple LLVM version 9.1.0 (clang-902.0.39.2)" )
	             AT_language( DW_LANG_ObjC )
	             AT_name( "/Users/worthyzhang/Desktop/mytest/mytest/ViewController.m" )
	             AT_stmt_list( 0x00009151 )
	             AT_comp_dir( "/Users/worthyzhang/Desktop/mytest" )
	             AT_APPLE_optimized( true )
	             AT_APPLE_major_runtime_vers( 0x02 )
	             AT_low_pc( 0x00000001000069bc )
	             AT_high_pc( 0x000000a4 )
	
	0x0003ebf9:     TAG_subprogram [122] *
	                 AT_low_pc( 0x00000001000069bc )
	                 AT_high_pc( 0x00000070 )
	                 AT_frame_base( reg29 )
	                 AT_object_pointer( {0x0003ec12} )
	                 AT_name( "-[ViewController viewDidLoad]" )
	                 AT_decl_file( "/Users/worthyzhang/Desktop/mytest/mytest/ViewController.m" )
	                 AT_decl_line( 17 )
	                 AT_prototyped( true )
	                 AT_APPLE_optimized( true )
	Line table dir : '/Users/worthyzhang/Desktop/mytest/mytest'
	Line table file: 'ViewController.m' line 25, column 1 with start address 0x0000000100006a14
	
	Looking up address: 0x0000000100006a14 in .debug_frame... not found.
	```

	可以定位到报错所在的函数名 [ViewController viewDidLoad] 以及文件名、行号等信息。

2. atos

	如果只是简单的获取符号名，可以用 atos 来符号化，命令格式如下：

	```
	atos -o [dsym file path] -l [Load Address] -arch [arch type] [Stack Address]
	```
	
	需要注意这里的 dsym file path 是 dsym 文件而不是 .dSYM 结尾的文件夹，输入命令：

	```
	$ atos -o mytest.app.dSYM/Contents/Resources/DWARF/mytest -l 0x1046e8000 --arch arm64 0x1046eea14
	-[ViewController viewDidLoad] (in mytest) (ViewController.m:25)
	```
	
	得到结果和dwarfdump是一致的。

## 五、问题

1. Debug 环境正常，Release 环境崩溃

	属性内存语义错误，如 NSArray 使用 assign 修饰，导致访问了释放掉的内存地址。

2. 闪退

	- 数据库损坏：在日常使用异常退出、断电或者错误的操作
	- 文件损坏：处理文件时如果没有 @try...catch，损坏文件会抛出 NSException 导致 crash
	- 网络返回数据异常：数据类型不对，或返回破损的 Tar 包，在解压失败导致 crash。
	- 代码 bug：当必 crash 的代码出现在启动关键路径中，就会导致连续闪退。

	解决：
	
	- 通过工具修复数据库，或者删除 DB。
	- 删除文件来进行修复
	- 具体地分析 crash 案例，通过 JSPatch 来进行修复。

3. 数组越界，nil 值初始化导致的崩溃。
4. 对字典插入 nil 值，或者读取 NSNULL 导致的崩溃。
5. 字符串的截取越界导致的崩溃。
6. doesNotRecognizeSelector 导致的崩溃。
7. 子线程初始化 UIView 导致的崩溃。
8. KVO的重复添加、删除，或者忘了删除导致的崩溃。

## 六、文章

[悟行Worthy](https://www.jianshu.com/u/796e9139e445) & [iOS实现Crash捕获与堆栈符号化](https://www.jianshu.com/p/ea8926762121)
[Mach-O Executables](https://www.objc.io/issues/6-build-tools/mach-o-executables/)
[Mach-O Programming Topics](https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/MachOTopics/0-Introduction/introduction.html#//apple_ref/doc/uid/TP40001827-SW1)
[iOS崩溃堆栈信息的符号化解析](https://www.jianshu.com/p/953f0961157a)
[oncezou](https://www.jianshu.com/u/2b54d468523d) & [iOS Crash的捕获知识](https://www.jianshu.com/p/5fcf7bb7955f)
[iOS 崩溃处理（拦截和捕获）](https://www.jianshu.com/p/ca95fdee78d8)
[iOS开发：Crash异常总结与捕获](https://blog.csdn.net/weixin_38633659/article/details/82496635)