---
title: 图片设置圆角性能问题
categories: iOS优化
---

通常设置圆角方式

```objc
imageView.clipsToBounds = YES;
imageView.layer.cornerRadius = 50;
```

这样设置会触发离屏渲染，比较消耗性能。比如当一个页面上有十几个头像，这样设置了圆角会明显感觉到卡顿。

注意：UIImageView 处理 png 图片的圆角是不会产生离屏渲染的。（iOS9.0 之后不会离屏渲染，iOS9.0 之前还是会离屏渲染）。


## 一、设置圆角的方法

①、直接使用 cornerRadius。这种是最常用的，也是最耗性能的。

②、设置 cornerRadius 圆角之后，shouldRasterize = YES 光栅化

```objc
imageView.clipsToBounds = YES;
imageView.layer.cornerRadius = 50;
imageView.layer.shouldRasterize = YES;  // 设置光栅化
imageView.layer.rasterizationScale = [UIScreen mainScreen].scale;  // UIImageView 不加这句会产生一点模糊
```

<font color=#cc0000>设置光栅化可以使离屏渲染的结果缓存到内存中存为位图</font>，使用的时候直接使用缓存，节省了一直离屏渲染损耗的性能。

但是如果 layer 及 sublayers 常常改变的话，它就会一直不停的渲染及删除缓存重新创建缓存，所以这种情况下建议不要使用光栅化，这样也是比较损耗性能的。

③、直接覆盖一张中间为圆形透明的图片（推荐使用）

这种方法就是多加了一张透明的图片，GPU 计算多层的混合渲染 blending 也是会消耗一点性能的，但比第一种方法还是好上很多的。

这种圆片覆盖的方法一般只用在<font color=#cc0000>底色为纯色</font>的时候，如果圆角图片的父 View 是张图片的时候就没办法了，而且底色如果是多种颜色的话那要做多张不同颜色的圆片覆盖。（可以用代码取底色的颜色值给圆片着色）

④、UIImage drawInRect 绘制圆角

这种方式 GPU 损耗低内存占用大。

```objc
@interface CornerImageView ()
{    
    NSBlockOperation * _operation;  // 任务 
    NSOperationQueue * _queue;  
    UIImage * _cornerImage;  // 圆角化的图片
}
@end

@implementation CornerImageView

- (instancetype)initWithFrame:(CGRect)frame
{
    if (self = [super initWithFrame:frame]) {
        _queue = [[NSOperationQueue alloc] init];
    }
    return self;
}

/**
 * 重写设置方法。如果是 UIButton 可以换成 setImage:forState:
 */
- (void)setImage:(UIImage *)image
{
    [super setImage:nil];

    [self roundedImage:image];
}

- (void)roundedImage:(UIImage *)image
{
    [_queue cancelAllOperations];

    [_operation cancel];
    _operation = nil;
    _operation = [NSBlockOperation blockOperationWithBlock:^{
        
        UIGraphicsBeginImageContextWithOptions(self.bounds.size, false, [UIScreen mainScreen].scale);
        // Add a clip before drawing anything, in the shape of an rounded rect
        [[UIBezierPath bezierPathWithRoundedRect:self.bounds
                                    cornerRadius:self.bounds.size.height / 2] addClip];
        [image drawInRect:self.bounds];
        
        _cornerImage = UIGraphicsGetImageFromCurrentImageContext();
        
        // Lets forget about that we were drawing
        UIGraphicsEndImageContext();

        if (!_operation) {
            return;
        }
        dispatch_async(dispatch_get_main_queue(), ^{
            [super setImage:_cornerImage];
        });
    }];
    [_queue addOperation:_operation];
}
```

这段方法可以写在 SDWebImage 的 completed 回调里，在主线程异步绘制。也可以封装到 UIImageView 里，后台线程异步绘制，不会阻塞主线程。

问题：这种方法图片很多的话 CPU 消耗会高，内存占用也会暴增，而且后台线程绘制会比在主线程绘制占用更多的内存，不知道怎么解决？

⑤、SDWebImage 处理图片时 CoreGraphics 绘制圆角

```objc
@interface UIImage (corner)
+ (id)createRoundedRectImage:(UIImage *)image;
@end

@implementation UIImage (corner)

static void addRoundedRectToPath(CGContextRef context, CGRect rect, float ovalWidth, float ovalHeight)
{
    float fw, fh;
    
    if (ovalWidth == 0 || ovalHeight == 0) {
        CGContextAddRect(context, rect);
        return;
    }
    
    CGContextSaveGState(context);
    CGContextTranslateCTM(context, CGRectGetMinX(rect), CGRectGetMinY(rect));
    CGContextScaleCTM(context, ovalWidth, ovalHeight);
    fw = CGRectGetWidth(rect) / ovalWidth;
    fh = CGRectGetHeight(rect) / ovalHeight;  // 使的圆角半径为 1
    
    CGContextMoveToPoint(context, fw, fh/2);  // Start at lower right corner
    CGContextAddArcToPoint(context, fw, fh, fw/2, fh, 1);  // Top right corner
    CGContextAddArcToPoint(context, 0, fh, 0, fh/2, 1); // Top left corner
    CGContextAddArcToPoint(context, 0, 0, fw/2, 0, 1); // Lower left corner
    CGContextAddArcToPoint(context, fw, 0, fw, fh/2, 1); // Back to lower right
    
    CGContextClosePath(context);
    CGContextRestoreGState(context);
}

+ (id)createRoundedRectImage:(UIImage *)image
{
    CGFloat wh = MIN(MAX(image.size.width, image.size.height), 160);
    CGSize imageSize = CGSizeMake(wh, wh);
    CGfloat radius = wh / 2;
        
    CGContextRef context = CGBitmapContextCreate( NULL, 
                                                  wh, 
                                                  wh, 
                                                  8, 
                                                  4 * wh, 
                                                  CGColorSpaceCreateDeviceRGB(), 
                                                  kCGImageAlphaPremultipliedFirst );
    // 绘制圆角    
    CGContextBeginPath(context);
    
    addRoundedRectToPath(context, CGRectMake(0, 0, wh, wh), radius, radius);

    CGContextClosePath(context);
    CGContextClip(context);
    CGContextDrawImage(context, CGRectMake(0, 0, w, h), img.CGImage);
    CGImageRef imageMasked = CGBitmapContextCreateImage(context);
    image = [UIImage imageWithCGImage:imageMasked];
        
    CGContextRelease(context);
    CGImageRelease(imageMasked);

    return image;
}
```

以上代码写成了 UIImage 的类别。并在 SDWebImage 库里处理 image 的时候使用类别方法绘制圆角并缓存。

```objc
/**
 * @brief 在上下文的路径中添加一条圆弧，可能前面有一条直线段。弧与当前点到 '(x1，y1)' 的直线相切，与 '(x1，y1)' 到 '(x2, y2)' 的直线相切。
 */
CG_EXTERN void CGContextAddArcToPoint(CGContextRef cg_nullable c,
    CGFloat x1, CGFloat y1, CGFloat x2, CGFloat y2, CGFloat radius)
    CG_AVAILABLE_STARTING(10.0, 2.0);
```


## 二、使用 Instruments 的 Core Animation 查看性能

*   Color Offscreen-Rendered Yellow
    

    开启后会把那些需要离屏渲染的图层高亮成黄色，这就意味着黄色图层可能存在性能问题。

*   Color Hits Green and Misses Red
    
    如果 shouldRasterize 被设置成 YES，对应的渲染结果会被缓存，如果图层是绿色，就表示这些缓存被复用；如果是红色就表示缓存会被重复创建，这就表示该处存在性能问题了。
    

用 Instruments 测试得：

①、直接设置 cornerRadius，UIImageView 和 UIButton 都高亮为黄色。

②、增加光栅化，UIImageView 和 UIButton 都高亮为绿色。

③、添加圆形透明图片，无任何高亮，说明没离屏渲染。

④、drawInRect 方法无任何高亮，说明没离屏渲染（但是 CPU 消耗和内存占用会很大）

⑤、CoreGraphics 绘制方法无任何高亮，说明没离屏渲染，而且内存占用也不大。(暂时感觉是最优方法)


## 三、问题

①、有提到还有一种 mask 方法。 

这种方法比第一种方法其实更卡顿。一次 mask 发生了两次离屏渲染和一次主屏渲染。 具体可以参考[小心别让圆角成了你列表的帧数杀手](http://www.cocoachina.com/ios/20150803/12873.html)。

②、第四种比第一种更卡。

第一种能明显的感觉到卡顿，第四种还是挺顺畅的，有兴趣的可以自己试试看。第四种是解决了离屏渲染 GPU 的问题。

 可以用 Instruments的 GPU Driver 进行测试：

*    Renderer Utilization 如果这个值 > 50%，就意味着你的动画可能对帧率有所限制，很可能因为离屏渲染或者是重绘导致的过度混合。
    
*   Tiler Utilization     如果这个值 > 50%，就意味着你的动画可能限制于几何结构方面，也就是在屏幕上有太多的图层占用了。
    

![](https://upload-images.jianshu.io/upload_images/5294842-9f4398579f83477e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第 1 种方法的 Renderer Utilization 和 Tiler Utilization 基本在 90% 左右，帧率 20 左右。

第 2 种方法的 Renderer Utilization 和 Tiler Utilization 基本在 20% 左右，帧率接近 60。

<font color=#cc0000>帧率越接近 60 滑动越顺畅</font>。

发现第 4 种 Core Graphics 绘制圆角会有大量的内存占用，而且每次绘制的时候 CUP 消耗会很大。

如果使用了 UITableView 进行测试，因为 UITableView 滚动的时候是一直在复用的，UIImageView 会重复绘制，所以会一直消耗 CUP，然后你就能看的明显的卡顿。将图片的绘制在后台线程进行绘制，解决了卡顿问题，但是由于是在后台线程的异步绘制所以在滚动的时候会看到图片先是正方形然后再变成圆形。

而使用 UIScrollView 进行测试，只有第一次绘制的时候会占用 CUP 资源，所以滑动的时候还是挺流畅的，但是内存消耗还是很大。如果是主线程绘制的话会阻塞一点时间的主线程，而后台线程绘制的话内存消耗会更大，特别容易崩溃。

所以第四种方法当图片特别多的时候很容易 Received memory warning 导致崩溃。

## 四、参考文章

*   [内存恶鬼drawRect - 谈画图功能的内存优化](http://mp.weixin.qq.com/s?__biz=MjM5NTIyNTUyMQ==&mid=447105405&idx=1&sn=054dc54289a98e8a39f2b9386f4f620e&scene=0#wechat_redirect)
    
*   github 绘制圆角源码参考 [NZCircularImageView](https://github.com/NZN/NZCircularImageView)、[HJCornerRadius](https://github.com/panghaijiao/HJCornerRadius)