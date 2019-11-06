---
title: iOS 页面流畅技巧
categories: iOS优化
---


## 一、屏幕显示图像原理

首先明确两个概念：水平同步信号、垂直同步信号。

![](https://upload-images.jianshu.io/upload_images/5294842-920b300dcddfb38b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

CRT 的电子枪按照上图中的方式，从上到下一行一行的扫描，扫描完成后显示器就呈现一帧画面，随后电子枪回到初始位置继续下一次的扫描。当电子枪切换到新的一行准备扫描时，显示器会发送一个水平同步信号（Horizonal Synchronization），简称HSync；完成一帧画面绘制后，电子枪会回到原位，显示器会发送一个垂直同步信号（Vertical Synchronization），简称VSync。

CUP 计算好显示内容提交到 GPU，GPU 渲染完成后将渲染结果放入帧缓冲区，之后视频控制器按照 VSync 信号逐行读取帧缓冲区中的数据，最后经过各种数模转换传递给显示器显示。

![](https://upload-images.jianshu.io/upload_images/5294842-1cc88c02ec5e957b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 二、卡顿产生的原因

<font color=#cc0000>如果在一个 VSync 时间内，CPU 或者 GPU 没有完成内容提交，则那一帧就会被丢弃</font>，等待下一次再显示，而这时显示屏会保留之前的内容不变，这就是卡顿的原因。


## 三、CPU 资源消耗的原因和解决方案

#### 3.1 对象的创建

对象的创建会分配内存、调整属性、甚至还有读取文件的操作，比较消耗 CPU 资源。因此可以：

①、尽量用轻量的对象代替重量的对象。如 CALayer 比 UIView 轻量的多，在不需要响应触摸事件时，用 CALayer 显示更合适；

②、如果对象不涉及 UI 操作，尽量放到后台线程去创建；

③、通过 storyboard 创建视图对象时，其资源消耗会比直接通过代码创建对象要大非常多，所以尽量避免使用；

④、尽量推迟对象创建的时间，并把对象的创建分散到多个任务中去；

⑤、如果对象可以复用，并且复用的代价比释放、创建新对象要小，那么这类对象应当尽量放到一个缓存池里复用。

#### 3.2 对象调整

对象的调整也是经常消耗 CPU 资源的地方。尤其是 CALayer：

①、CALayer 内部没有属性，当调用属性方法时，它内部是<font colro=#cc0000>通过运行时</font> resolveInstanceMethod 为对象临时添加一个方法，并把对应属性值保存到内部的一个 Dictionary 中，同时还会告知 delegate、创建动画等，非常消耗资源；

②、UIView 关于显示相关的属性（比如 frame/bouds/transform 等）实际上都是 CALayer 属性映射出来的，所以对UIView 的这些属性进行调整时，消耗的资源要远大于一般的属性，因此应该尽量减少类似的不必要的属性的修改；

③、当视图层次调整时，UIView、CALayer 之间会出现很多调用与通知，所以在优化性能时，应该尽量避免调整视图层次、添加和移除视图。

#### 3.3 对象销毁

当容器类持有大量对象时，其销毁时的资源消耗就非常明显。所以，尽量去后台线程释放对象。可以这么做：把对象捕获到 block 中，然后扔到后台队列去随便发送个消息以避免编译警告，就可以让对象在后台线程销毁了：

```objc
NSArray * tmp = self.arr_data;
self.arr_data = nil;
dispatch_async(queue, ^{
    [tmp class];
});
```

#### 3.4 对象布局

在后台线程提前计算好视图布局、并对视图的布局进行缓存。

不论通过何种技术对视图进行布局，最终都会落到对 UIView.frame/bounds/center 等属性的调整上。

#### 3.5 Autolayout

这是苹果本身提倡的技术，在大部分情况下能很好的提升开发效率，但<font color=#cc0000>对于复杂视图来说常会产生严重的性能问题</font>。随着视图数量的增长，Autolayout 带来的 CPU 消耗会呈指数级增长。

#### 3.6 文本计算

如果一个界面中包含大量的文本，文本的宽高计算会占用很大一部分资源，并且不可避免。

#### 3.7 文本渲染

屏幕上能看到的所有的文本内容控件包括 UIWebView，在底层都是通过 CoreText 排版、绘制为 Bitmap 显示的，并且该排版、绘制都是在主线程进行的。

显示大量文本时，CPU 的压力非常大，可以通过自定义文本控件，<font color=#cc0000>用 TextKit 或最底层的 CoreText 对文本异步绘制</font>，尽管麻烦但优势强大：

①、CoreText 对象能直接获取文本的宽高等信息，避免了多次计算（调整 UILabel 大小时算一遍、UILabel 绘制时内部再算一遍）；

②、CoreText 对象占用内存较小，可以缓存下来以备稍后多次渲染。

#### 3.8 图片解码

用 UIImage 或者 CGImageSource 的方法创建图片时，图片数据并不会立刻解码。图片设置到 UIImageView 或者 CALayer.contents 中，并且 CALayer 被提到 GPU 前，CGImage 中的数据才会得到解码。

解码过程是一个相当复杂的任务，需要消耗非常长的时间。解码后的图片将同样使用相当大的内存。

该步是发生在主线程，并且不可避免。如果想绕开这个机制，常见的方法是<font color=#cc0000>在后台线程先把图片绘制到 CGBitmapContext 中</font>，然后从 Bitmap 直接创建图片。目前常见的网络图片库都自带这个功能。

用于加载的 CPU 时间相对于解码来说根据图片格式而不同。对于 PNG 图片来说，加载会比 JPEG 更长，因为文件可能更大，但是解码会相对较快，而且 Xcode 会把 PNG 图片进行解码优化之后引入工程。JPEG 图片更小，加载更快，但是解压的步骤要消耗更长的时间，因为 JPEG 解压算法比基于 zip 的 PNG 算法更加复杂。


#### 3.9 图像的绘制

是指用那些以 CG 开头的方法把图像绘制到画布中，然后从画布创建图片并显示。常见的就是 [UIView drawRect: ]。CoreGraphic 方法通常是线程安全的，所以图像的绘制可以放到后台线程运行。如下：（实际情况比这个复杂，但原理基本一致）

```objc
- (void)display
{
    dispatch_async(backgroundQueue, ^{
        CGContextRef ctx = CGBitmapContextCreate(...);
        // draw in context...
        CGImageRef img = CGBitmapContextCreateImage(ctx);
        CFRelease(ctx);
        dispatch_async(mainQueue, ^{
            layer.contents = img;
        });
    });
}
```

## 四、GPU 资源消耗原因和解决方案

GPU 能干的事情比较单一：接受提交的纹理（Texture）和顶点描述（三角形）、应用变换（transform）、混合并渲染，然后输出到屏幕上。看到的内容通常主要是纹理（图片）和形状（三角模拟的矢量图形）两类。

#### 4.1 纹理的渲染

所有的 Bitmap，包括图片、文字、栅格化的内容，最终都要由内存提交到显存，绑定为 GPU Texture。不论是提交到显存的过程，还是 GPU 调整和渲染 Texture 的过程，都要消耗不少 GPU 资源。

当在短时间内显示大量图片时（如 TableView），CPU 占用率很低，GPU 占用非常高，界面会掉帧。

当图片过大，超过 GPU 的最大纹理尺寸时，图片需要先由 CPU 进行预处理，这对 CPU 跟 GPU 都会带来额外的消耗。

#### 4.2 视图的混合（Composing）

当多个视图（或者 CALayer）重叠在一起显示时，GPU 会首先把他们混合到一起。如果视图结构过于复杂，混合的过程也会消耗很多的 GPU 资源。

所以应当<font color=#cc0000>尽量减少视图数量和层次</font>，并在不透明的视图里标明 opaque 属性以避免无用的 Alpha 通道合成。

也可以把多个视图预先渲染为一张图片来显示。

#### 4.3 图形的生成

CALayer 的 border、圆角、阴影、遮罩（mask），CASharpLayer 的矢量图形显示，通常会触发离屏渲染，而离屏渲染通常发生在 GPU 中。

当列表中出现大量圆角的 CALayer 并且快速滑动时，GPU 资源可能几近占满，而 CPU 资源消耗很少，这时候界面仍能正常滑动但平均帧数降到很低。这时候可以尝试开启 CALayer.shouldRaster 属性，但这会离屏渲染操作转嫁到 CPU 上。

对于只需要圆角的某些场合，可以用一张已经绘制好的圆角图片覆盖到原视图上来模拟出相同的视觉效果。

最彻底的做法：<font color=#cc0000>把需要显示的图形在后台线程绘制为图片，避免使用圆角、阴影、遮罩等属性</font>。


## 五、文章
[幸运的芳1990](https://www.jianshu.com/u/93bd6c9bc835) & [浅谈iOS页面流畅技巧](https://www.jianshu.com/p/bade6ce45b8b)
