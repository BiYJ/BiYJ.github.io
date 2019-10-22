---
title: mmap
categories: iOS原理
---

## 一、常规文件操作

常规文件操作（read/write）有那几个重要步骤：

1. 进程发起读文件请求
2. 内核通过查找进程文件符表，定位到内核已打开文件集上的文件信息，从而找到此文件的 inode
3. inode 在 address_space 上查找要请求的文件页是否已经缓存在内核页高速缓冲中。如果存在，则直接返回这片文件页的内容
4. 如果不存在，则通过 inode 定位到文件磁盘地址，将数据从磁盘复制到内核页高速缓冲。之后再次发起读页面过程，进而将内核页高速缓冲中的数据发给用户进程

需要注意的几点：

1. 常规文件操作为了提高读写效率和保护磁盘，使用了页缓存机制。由于页缓存处在内核空间，不能被用户进程直接寻址，所以需要将页缓存中数据页再次拷贝到内存对应的用户空间中
2. read/write 是系统调用很耗时，如下图，它首先将文件内容从硬盘拷贝到内核空间的一个缓冲区，然后再将这些数据拷贝到用户空间，实际上完成了两次数据拷贝
3. 如果两个进程都对磁盘中的一个文件内容进行访问，那么这个内容在物理内存中有三份：进程 A 的地址空间 + 进程 B 的地址空间 + 内核页高速缓冲空间
4. 写操作也是一样，待写入的 buffer 在内核空间不能直接访问，必须要先拷贝至内核空间对应的主存，再写回磁盘中（延迟写回），也是需要两次数据拷贝

[Linux 内核剖析](https://www.jianshu.com/p/4f79680b54a4)

<center>
![](http://dzliving.com/mmap_1.jpeg)
</center>

## 二、mmap内存映射

#### 2.1 mmap 介绍

在日常开发中偶尔会遇到 mmap，它最常用到的场景是 [MMKV](https://github.com/Tencent/MMKV)，其次用到的是日志打印。虽然都已经被封装好，但也需要了解下 mmap 的基本原理和过程。

进程是 App 运行的基本单位，进程之间相对独立。iOS 系统中 <font color=#cc0000>App 运行的内存空间地址是虚拟空间地址，存储数据是在各自的沙盒</font>。

当我们在 App 中去读写沙盒中的文件时，我们会使用 NSFileManager 去查找文件，然后可以使用 NSData 去加载二进制数据。文件操作的更底层实现过程，是使用 linux 的 ``read()``、``write()`` 函数直接操作文件句柄（也叫文件描述符、fd）。

在操作系统层面，当 App 读取一个文件时，实际是有两步：

1. 将文件从磁盘读取到物理内存；
2. 从系统空间拷贝到用户空间（可以认为是复制到系统给 App 统一分配的内存）。

iOS 系统使用[页缓存机制](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fgdj0001%2Farticle%2Fdetails%2F80136364)，通过 [MMU](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2FMMU%2F4542218)（Memory Management Unit）<font color=#cc0000>将虚拟内存地址和物理地址进行映射</font>，并且由于进程的地址空间和系统的地址空间不一样，所以还需要多一次拷贝。

而 mmap 将磁盘上文件的地址信息与进程用的虚拟逻辑地址进行映射，建立映射的过程与普通的内存读取不同：正常的是将文件拷贝到内存，<font color=#cc0000>mmap 只是建立映射而不会将文件加载到内存中</font>。

<center>
![](http://dzliving.com/mmap_2.jpeg)
</center>

在内存映射的过程中，并没有实际的数据拷贝，文件没有被载入内存，只是逻辑上被放入了内存，具体到代码，就是建立并初始化了相关的数据结构（struct address_space），这个过程由系统调用 mmap() 实现，所以建立内存映射的效率很高。

既然建立内存映射没有进行实际的数据拷贝，那么进程又怎么能最终直接通过内存操作访问到硬盘上的文件呢？那就要看内存映射之后的几个相关的过程了。

mmap() 会返回一个指针 ptr，它指向进程逻辑地址空间中的一个地址，这样以后，进程无需再调用 read 或 write 对文件进行读写，而只需要通过 ptr 就能够操作文件。但是 ptr 所指向的是一个逻辑地址，要操作其中的数据，必须通过 MMU 将逻辑地址转换成物理地址，如上图中过程 2 所示。这个过程与内存映射无关。

前面讲过，建立内存映射并没有实际拷贝数据，这时，MMU 在地址映射表中是无法找到与 ptr 相对应的物理地址的，也就是 MMU 失败，将产生一个缺页中断，缺页中断的中断响应函数会在 swap 中寻找相对应的页面，如果找不到（也就是该文件从来没有被读入内存的情况），则会通过 mmap() 建立的映射关系，从硬盘上将文件读取到物理内存中，如上图中过程 3 所示。这个过程与内存映射无关。

如果在拷贝数据时，发现物理内存不够用，则会通过虚拟内存机制（swap）将暂时不用的物理页面交换到硬盘上，如图1中过程4所示。这个过程也与内存映射无关。

mmap 内存映射的实现过程，总的来说可以分为三个阶段：

1. 进程启动映射过程，并在虚拟地址空间中为映射创建虚拟映射区域
2. 调用内核空间的系统调用函数 mmap（不同于用户空间函数），实现文件物理地址和进程虚拟地址的一一映射关系
3. 进程发起对这片映射空间的访问，引发缺页异常，实现文件内容到物理内存（主存）的拷贝

[认真分析mmap：是什么 为什么 怎么用](https://www.cnblogs.com/huxiao-tee/p/4660352.html)

#### 2.2 适合的场景

- 有一个很大的文件，因为映射有额外的性能消耗，所以适用于频繁读操作的场景；（单次使用的场景不建议使用）
- 有一个小文件，它的内容您想要立即读入内存并经常访问。这种技术最适合那些大小不超过几个虚拟内存页的文件。（页是地址空间的最小单位，虚拟页和物理页的大小是一样的，通常为 <font color=#cc0000>4KB</font>。）
- 需要在内存中缓存文件的特定部分。文件映射消除了缓存数据的需要，这使得系统磁盘缓存中的其他数据空间更大

当随机访问一个非常大的文件时，通常最好只映射文件的一小部分。映射大文件的问题是文件会消耗活动内存。如果文件足够大，系统可能会被迫将其他部分的内存分页以加载文件。将多个文件映射到内存中会使这个问题更加复杂。

#### 2.3 不适合的场景

- 希望从开始到结束的顺序从头到尾读取一个文件
- 文件有几百兆字节或者更大。将大文件映射到内存中会快速地填充内存，并可能导致分页，这将抵消首先映射文件的好处。对于大型顺序读取操作，禁用磁盘缓存并将文件读入一个小内存缓冲区
- 该文件大于可用的连续虚拟内存地址空间。对于 64 位应用程序来说，这不是什么问题，但是对于 32 位应用程序来说，这是一个问题。32 位虚拟内存最大是 4GB，可以只映射部分。
- 因为每次操作内存会同步到磁盘，所以不适用于移动磁盘或者网络磁盘上的文件；
- 变长文件不适用；

## 三、iOS 中的 mmap

以[官网的 demo](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.apple.com%2Flibrary%2Farchive%2Fdocumentation%2FPerformance%2FConceptual%2FFileSystem%2FArticles%2FMappingFiles.html%23%2F%2Fapple_ref%2Fdoc%2Fuid%2F20001990-CJBJFIDD) 为例，其他的代码很简明直接，核心就在于mmap函数。

```
/**
 *  @param  start  映射开始地址，设置 NULL 则让系统决定映射开始地址
 *  @param  length  映射区域的长度，单位是 Byte
 *  @param  prot  映射内存的保护标志，主要是读写相关，是位运算标志；（记得与下面fd对应句柄打开的设置一致）
 *  @param  flags  映射类型，通常是文件和共享类型
 *  @param  fd  文件句柄
 *  @param  off_toffset  被映射对象的起点偏移
 */
void* mmap(void* start,size_t length,int prot,int flags,int fd,off_t offset);

*outDataPtr = mmap(NULL,
                   size,
                   PROT_READ|PROT_WRITE,
                   MAP_FILE|MAP_SHARED,
                   fileDescriptor,
                   0);
```

用官网的代码做参考，写了一个读写的例子：

```
#import "ViewController.h"
#import <sys/mman.h>
#import <sys/stat.h>

int MapFile(const char * inPathName, void ** outDataPtr, size_t * outDataLength, size_t appendSize)
{
    int outError;
    int fileDescriptor;
    struct stat statInfo;
    
    // Return safe values on error.
    outError = 0;
    *outDataPtr = NULL;
    *outDataLength = 0;
    
    // Open the file.
    fileDescriptor = open( inPathName, O_RDWR, 0 );
    if( fileDescriptor < 0 )
    {
        outError = errno;
    }
    else
    {
        // We now know the file exists. Retrieve the file size.
        if( fstat( fileDescriptor, &statInfo ) != 0 )
        {
            outError = errno;
        }
        else
        {
            ftruncate(fileDescriptor, statInfo.st_size + appendSize);
            fsync(fileDescriptor);
            *outDataPtr = mmap(NULL,
                               statInfo.st_size + appendSize,
                               PROT_READ|PROT_WRITE,
                               MAP_FILE|MAP_SHARED,
                               fileDescriptor,
                               0);
            if( *outDataPtr == MAP_FAILED )
            {
                outError = errno;
            }
            else
            {
                // On success, return the size of the mapped file.
                *outDataLength = statInfo.st_size;
            }
        }
        
        // Now close the file. The kernel doesn’t use our file descriptor.
        close( fileDescriptor );
    }
    
    return outError;
}


void ProcessFile(const char * inPathName)
{
    size_t dataLength;
    void * dataPtr;
    char *appendStr = " append_key";
    int appendSize = (int)strlen(appendStr);
    if( MapFile(inPathName, &dataPtr, &dataLength, appendSize) == 0) {
        dataPtr = dataPtr + dataLength;
        memcpy(dataPtr, appendStr, appendSize);
        // Unmap files
        munmap(dataPtr, appendSize + dataLength);
    }
}

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad 
{
    [super viewDidLoad];
    
    NSString * path = [NSHomeDirectory() stringByAppendingPathComponent:@"test.data"];
    NSLog(@"path: %@", path);
    NSString *str = @"test str";
    [str writeToFile:path atomically:YES encoding:NSUTF8StringEncoding error:nil];
    
    ProcessFile(path.UTF8String);
    NSString *result = [NSString stringWithContentsOfFile:path encoding:NSUTF8StringEncoding error:nil];
    NSLog(@"result:%@", result);
}
```

在 iOS 的应用实例：

- [MMKV--基于 mmap 的 iOS 高性能通用 key-value 组件](https://cloud.tencent.com/developer/article/1066229)
- [iOS图片加载速度极限优化—FastImageCache解析](http://blog.cnbang.net/tech/2578/)
- [FastImageCache](https://github.com/path/FastImageCache)

## 四、MMKV 和 mmap

NSUserDefault 是常见的缓存工具，但是数据有时会同步不及时，比如说在 crash 前保存的值很容易出现保存失败的情况，在 App 重新启动之后读取不到保存的值。

MMKV 很好的解决了 NSUserDefault 的局限，具体的好处可以见[官网](https://links.jianshu.com/go?to=http%3A%2F%2F)。

但是同样由于其独特设计，在数据量较大、操作频繁的场景下，会产生性能问题。

这里的使用给出两个建议：

1. 不要全部用 defaultMMKV，根据业务大的类型做聚合，避免某一个 MMKV 数据过大，特别是对于某些只会出现一次的新手引导、红点之类的逻辑，尽可能按业务聚合，使用多个 MMKV 的对象；
2. 对于需要频繁读写的数据，可以在内存持有一份数据缓存，<font color=#cc0000>必要时再更新</font>到 MMKV；

## 五、NSData 与 mmap

NSData 有一个静态方法和 mmap 有关系。

```
+ (id)dataWithContentsOfFile:(NSString *)path options:(NSDataReadingOptions)readOptionsMask error:(NSError **)errorPtr;

typedef NS_OPTIONS(NSUInteger, NSDataReadingOptions) {

	// Hint to map the file in if possible and safe. 在保证安全的前提下使用 mmap
	NSDataReadingMappedIfSafe =   1UL << 0,
	// Hint to get the file not to be cached in the kernel. 不要缓存。如果该文件只会读取一次，这个设置可以提高性能
    NSDataReadingUncached = 1UL << 1,
    // Hint to map the file in if possible. This takes precedence over NSDataReadingMappedIfSafe if both are given.  总使用 mmap
    NSDataReadingMappedAlways API_AVAILABLE(macos(10.7), ios(5.0), watchos(2.0), tvos(9.0)) = 1UL << 3,
   	...
};
```

1. Mapped 的意思是使用 mmap，那么 ifSafe 是什么意思呢？
2. NSDataReadingMappedIfSafe 和 NSDataReadingMappedAlways 有什么区别？

如果使用 mmap，则在 NSData 的生命周期内，都不能删除对应的文件。

如果文件是在固定磁盘，非可移动磁盘、网络磁盘，则满足 NSDataReadingMappedIfSafe。对 iOS 而言，这个 NSDataReadingMappedIfSafe = NSDataReadingMappedAlways。

> 那什么情况下应该用对应的参数？

如果文件很大，直接使用 ``dataWithContentsOfFile`` 方法，会导致 load 整个文件，出现内存占用过多的情况；此时用 NSDataReadingMappedIfSafe，则会使用 mmap 建立文件映射，减少内存的占用。

使用场景：视频加载。视频文件通常比较大，但是使用的过程中不会同时读取整个视频文件的内容，可以使用 mmap 优化。

## 六、总结

> mmap 就是文件的内存映射。

通常读取文件是将文件读取到内存，会占用真正的物理内存；而 mmap 是用进程的内存虚拟地址空间去映射实际的文件中，这个过程由操作系统处理。mmap 不会为文件分配物理内存，而是相当于将内存地址指向文件的磁盘地址，后续对这些内存进行的读写操作，会由操作系统同步到磁盘上的文件。

iOS 中使用 mmap 可以用 c 方法的 mmap()，也可以使用 NSData 的接口带上NSDataReadingMappedIfSafe 参数。前者自由度更大，后者用于读取数据。

## 七、文章

[落影loyinglin](https://www.jianshu.com/u/815d10a4bdce) & [iOS的文件内存映射——mmap](https://www.jianshu.com/p/516e7ff6f251)
[mmap 苹果官方文档](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.apple.com%2Flibrary%2Farchive%2Fdocumentation%2FPerformance%2FConceptual%2FFileSystem%2FArticles%2FMappingFiles.html%23%2F%2Fapple_ref%2Fdoc%2Fuid%2F20001990-CJBJFIDD)
[NSDataReadingMappedIfSafe](https://links.jianshu.com/go?to=http%3A%2F%2Fwww.openradar.me%2F12254262)
[小凉介](https://www.jianshu.com/u/20fbcc23dce5) & [iOS内存映射mmap详解](https://www.jianshu.com/p/13f254cf58a7)
[linux中的页缓存和文件IO](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fgdj0001%2Farticle%2Fdetails%2F80136364)
[从内核文件系统看文件读写过程](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.cnblogs.com%2Fhuxiao-tee%2Fp%2F4657851.html)
[linux内存映射mmap原理分析](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fjoejames%2Farticle%2Fdetails%2F37958017)
[mmap实例及原理分析](http://ju.outofmemory.cn/entry/224106)