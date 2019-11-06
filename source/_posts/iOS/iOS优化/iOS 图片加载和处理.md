---
title: iOS 图片加载和处理
categories: iOS优化
---


## 一、图片显示

图片的显示分为三步：加载、解码、渲染。解码和渲染是由 UIKit 进行，通常我们操作的只有加载。

![](https://upload-images.jianshu.io/upload_images/5294842-08b5bf1385f387ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

以 UIImageView 为例。当其显示在屏幕上时，需要 UIImage 作为数据源。<font color=#cc0000>UIImage 持有的数据是未解码的压缩数据</font>，能节省较多的内存和加快存储。

当 UIImage 被赋值给 UIImage 时（例如 imageView.image = image;），图像数据会被解码，变成 RGB 的颜色数据。

解码是一个<font color=#cc0000>计算量较大且需要 CPU 来执行的任务</font>；并且解码出来的图片体积与图片的宽高有关系，而与图片原来的体积无关。其体积大小可简单描述为：宽 * 高 * 每个像素点的大小 = width * height * 4bytes。

![](https://upload-images.jianshu.io/upload_images/5294842-819f47a678632a85.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

图像解码操作会造成什么问题？

以常见的 UITableView 和 UICollectionView 为例，假如在使用一个多图片显示的功能：在上下滑动显示图片的过程中，我们会在 cellForRow 的方法加载 UIImage 图片、赋值给 UIImageView，相当于<font color=#cc0000>在主线程同时进行 IO 操作、解码操作</font>等，会造成<font color=#cc0000>内存迅速增长和 CPU 负载瞬间提升</font>。

并且内存的迅速增加会触发系统的内存回收机制，尝试回收其他后台进程的内存，增加 CPU 的工作量。如果系统无法提供足够的内存，则会先结束其他后台进程，最终无法满足的话会结束当前进程。

![](https://upload-images.jianshu.io/upload_images/5294842-2746a469e302ca58.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 1.1 优化一：降采样

在滑动显示的过程中，图片显示的宽高远比真实图片要小，我们可以采用<font color=#cc0000>加载缩略图的方式减少图片的占用内存</font>。如下图所示：

![](https://upload-images.jianshu.io/upload_images/5294842-362199218d9322bc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们加载 JPEG的图片，然后进行相关设置，解码后根据设置生成 CGImage 缩略图，最后包装成 UIImage，最终传递给UIImageView 渲染。

思考：这里的解码步骤为何不是上文提到的 imageView.image = image 时机？

```objc
- (UIImage *)downsampleImageAt:(NSURL *)imageURL to:(CGSize)pointSize scale:(CGFloat)scale
{
    CFDictionaryRef imageSourceOptions = CFDictionaryCreate ( CFAllocatorGetDefault(),
                                                              (void *)@[ (NSString *)kCGImageSourceShouldCache ],
                                                              (void *)@[ @(YES) ],
                                                              1,
                                                              &kCFTypeDictionaryKeyCallBacks,
                                                              &kCFTypeDictionaryValueCallBacks);
    
    CGImageSourceRef imageSource = CGImageSourceCreateWithURL((__bridge CFURLRef)imageURL, imageSourceOptions);
    
    NSInteger maxDimensionInPixels = MAX(pointSize.width, pointSize.height) * scale;
    
    CFDictionaryRef downsampleOptions = (__bridge CFDictionaryRef)@{ (NSString *)kCGImageSourceCreateThumbnailFromImageAlways : @(YES),
                                           (NSString *)kCGImageSourceShouldCacheImmediately : @(YES),
                                           (NSString *)kCGImageSourceCreateThumbnailWithTransform : @(YES),
                                           (NSString *)kCGImageSourceThumbnailMaxPixelSize : @(maxDimensionInPixels) };
    
    CGImageRef downsampledImage = CGImageSourceCreateThumbnailAtIndex(imageSource, 0, downsampleOptions);
    
    return [UIImage imageWithCGImage:downsampledImage];
}
```

正常的 UIImage 加载是从 App 本地读取，或者从网络下载图片，此时不涉及图片内容相关的操作，并不需要解码；当图片被赋值给 UIImageView 时，CALayer 读取图片内容进行渲染，所以需要对图片进行解码；而上文的缩略图生成过程中，已经对图片进行解码操作，此时的 UIImage 只是一个 CGImage 的封装，所以当 UIImage 赋值给 UIImageView 时，CALayer 可以直接使用 CGImage 所持有的图像数据。

#### 1.2 优化二：异步处理

![](https://upload-images.jianshu.io/upload_images/5294842-fb940b0c9a90d4b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从用户的体验来分析，滑动的操作往往是间断性触发，在滑动的瞬间有较大的工作量，而且由于都是在主线程进行操作无法进行任务分配，CPU 2 处于闲置。由此引申出两种优化手段：Prefetching（预处理）和 Background decoding / downsampling（子线程解码和降采样）。综合起来，可以<font color=#cc0000>在 Prefetching 时把降采样放到子线程进行处理</font>，因为降采样过程就包括解码操作。

![](https://upload-images.jianshu.io/upload_images/1049769-adc5d8f6870f9d47.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/890)

Prefetching 回调中，把降采样的操作放到同步队列 serialQueue 中，处理完毕之后抛给主线程进行 update 操作。

需要特别注意，此处不能是并发队列，否则会造成<font color=#cc0000>线程爆炸</font>，原因见总结部分。

```objc
{
    // 创建串行队列
    _serialQueue = dispatch_queue_create("DecodeQueue", DISPATCH_QUEUE_SERIAL);
}

/**
 *  @brief  获取单元格的图片
 */
- (void)collectionView:(UICollectionView *)collectionView prefetchItemsAt:(NSArray<NSIndexPath *> *)indexPaths
{
    for (NSIndexPath * indexPath in indexPaths) {
        dispatch_async(_serialQueue, ^{
            UIImage * downsampledImage = [self downsample];
            
            dispatch_async(dispatch_get_main_queue(), ^{
                [self updateAt:indexPath with:downsampledImage];
            });
        });
    }
}

/** 
 *  @brief  降采样。自行实现。 
 */
- (UIImage *)downsample
{
    return nil;
}

/**
 *  @brief  更新单元格的图片
 */
- (void)updateAt:(NSIndexPath *)indexPath with:(UIImage *)image
{
    
}
```

#### 1.3 优化三：使用 Image Asset Catalogs

苹果推荐的图片资源管理工具，压缩效率更高，在 iOS 12 的机器上有 10~20% 的空间节约，并且苹果会持续对其进行优化。

[WWDC Session](https://developer.apple.com/videos/play/wwdc2018/227/)。


## 二、总结

应用上述的优化策略，已经能对图片加载有比较好的优化。

WWDC 后续还有对 CustomDrawing 和 CALayer 的 BackingStore 的介绍，与图片关系不大，不在此赘述。

## 三、WWDC学习

原作者的经验：[落影loyinglin](https://www.jianshu.com/u/815d10a4bdce)

先主观假设两个前提：

1. 大部分苹果工程师对 iOS 系统内部实现都比我们要清楚；
2. 能到 WWDC 分享的工程师在苹果内部也是优秀的工程师；那么 WWDC 所讲的内容我们可以认为是正确的事实。

所以可以基于自己已掌握的基础知识，还有对 iOS 系统的了解来分析 WWDC 上面所提到的现象，<font color=#cc0000>看我们的 iOS 知识体系是否存在缺陷</font>；另外，WWDC 介绍的很多知识点同样<font color=#cc0000>免验证</font>的加入自己的知识体系。

以上文提到的线程爆炸为例，看看这种方式的好处。

原文如下：

> Thread Explosion（线程爆炸）
>
> More images to decode than available CPUs（解码图像数量大于 CPU 数量）  
GCD continues creating threads as new work is enqueued（GCD 创建新线程处理新的任务）  
Each thread gets less time to actually decode images（每个线程获得很少的时间解码图像）

从这个案例我们学习到<font color=#cc0000>如何避免图像解码的线程爆炸</font>，我们分析苹果工程师的逻辑，然后扩散思维：

> 原因：解码任务过多 => 过程：GCD 开启更多线程=> 结果：每个线程获得更少的时间

延伸出来的问题：

> GCD 是如何处理并发队列？为何会启动多个线程处理？多少的线程数量合适？线程的 cpu 时间分配和切换代价？...

举一反三。但是这样的思考稍显混乱，仍有优化的空间。把脑海关于 GCD 的认知提炼出来：

1. GCD 是用来处理一系列任务的同步和异步执行，队列有串行和并发两种，与线程的关系只有主线程和非主线程的区别；
2. 串行队列是执行完当前的任务，才会执行下一个 block 任务；并行队列是多个 block 任务并行执行，GCD 会根据任务的执行情况分配线程，原则是尽快完成所有任务。

接下来的表现是操作系统相关的知识：

1. iOS 系统中进程和线程的关联，每个启动的 App 都是一个进程，其中有多个线程；
2. cpu 的时间是分为多个时间片，每个线程轮询执行；
3. 线程切换执行有代价，但比进程切换小得多；
4. 每个 cpu 核心在同一时刻只能执行一个线程。

至此我们可以结合操作系统和 GCD 的知识，猜测底层 GCD 的实现思路和线程爆炸情况下的表现：

主线程把多个任务 block 放到并发队列，GCD 先启动一个线程处理解码任务，线程执行过程中遇到耗时操作时（IO 等待、大量 CPU 计算），短时间内无法完成，为了不阻塞后续任务的执行，GCD 启动新的线程处理新的任务。

结合此案例，我们能回答相关问题：

1. 现在有一个很复杂的计算任务，例如统计一个 5000\*5000 图片中像素点的 RGB 颜色通道，如果用分为 25 个任务放到 GCD 并发队列，把大图切分成 25 个 1000\*1000 小图分别统计，是否会速度的提升？
2. GCD 的串行队列和并发队列的应用场景有何不同？


## 四、文章

[iOS性能优化--图片加载和处理](https://www.jianshu.com/p/7d8a82115060)

[WWDC2018-Image and Graphics Best Practices](https://developer.apple.com/videos/play/wwdc2018/219/)
