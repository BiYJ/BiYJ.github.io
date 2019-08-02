---
title: 利用预渲染加速iOS设备的图像显示
categories: iOS优化
---

使用 UITableView 时，发现滚动时的性能还不错，但来回滚动时，第一次显示的图像不如再次显示的图像流畅，出现前会有稍许的停顿感。

于是猜想显示过的图像肯定是被缓存起来了，查了下文档后发现果然如此。在《Improving Image Drawing Performance on iOS》一文中找到了一些提示：原来在显示图像时，解压和重采样会消耗很多 CPU 时间；而如果预先在一个 bitmap context 里画出图像，再缓存这个图像，就能省去这些繁重的工作了。

接着下面举个例子程序来验证：

```objc
#import <UIKit/UIKit.h>

@interface ImageView : UIView
@property (nonatomic, strong) UIImage * image;
@end

#import "ImageView.h"
#include <mach/mach_time.h>

static const CGRect imageRect = { {0, 0}, {100, 100}};

@implementation ImageView

- (void)awakeFromNib
{
     if (!self.image) {
          self.image = [UIImage imageNamed:@"xxx"];
     }
     [superawakeFromNib];
} 

- (void)drawRect:(CGRect)rect
{
     if (CGRectContainsRect(rect, imageRect)) {
 
          uint64_t start = getTickCount();

          [self.image drawInRect:imageRect];

          uint64_t drawTime = getTickCount() - start;
 
          NSLog(@"%llu", drawTime);  // 打印时间间隔    
     }
}

// mach_absolute_time() 的单位是 Mach absolute time unit，而不是纳秒。它们之间的换算关系和 CPU 相关，不是一个常量。最简单的办法是用 CoreServices 框架的 AbsoluteToNanoseconds 和 AbsoluteToDuration 函数来转换。此外也可以用 mach_timebase_info 函数来获取这个比值。
uint64_t getTickCount(void)
{
     static mach_timebase_info_data_t sTimebaseInfo;

     uint64_t machTime = mach_absolute_time();

     // Convert to nanoseconds - if this is the first time we've run, get the timebase.
    if (sTimebaseInfo.denom == 0) {
         (void)mach_timebase_info(&sTimebaseInfo);
    }

    uint64_t millis = (machTime * sTimebaseInfo.numer) / sTimebaseInfo.denom;  // 纳秒

    return millis;
}

@end
```

测试用一张 1838 * 890 的图

2018-07-05 11:05:25.950978+0800 Demo[5831:113872] <font color=#cc0000>31802012</font>

接下来就是见证奇迹的时刻了，把这段代码加入程序：

```objc
static const CGSize imageSize = {100, 100};

- (void)awakeFromNib 
{
     if (!self.image) {
          self.image = [UIImage imageNamed:@"xxx"];
 
          // 由于JPEG图像是不透明的，所以第二个参数就设为YES
          // 第三个参数是缩放比例。虽然这里可以用 [UIScreen mainScreen].scale 来获取，但实际上设为 0 后，系统就会自动设置正确的比例了
          UIGraphicsBeginImageContextWithOptions(imageSize, YES, 0);
          // 将图像画到当前的 image context 里，此时就完成了解压缩和重采样的工作
          [image drawInRect:imageRect];
          self.image = UIGraphicsGetImageFromCurrentImageContext();
          UIGraphicsEndImageContext();
     }
}
```

值得一提的是，图像本身也有缩放比例，普通的图像是 1.0（除了 imageNamed: 外，大部分 API 都只能获得这种图像，而且缩放比例是不可更改的），高清图像是 2.0。图像的点和屏幕的像素就是依靠两者的缩放比例来计算的，例如普通图像在视网膜显示屏上是 1:4，而高清图像在视网膜显示屏上则是 1:1。

时间间隔：2018-07-05 11:30:46.284490+0800 Demo[6401:133240] <font color=#cc0000>127939</font>，缩短了很多。

还能更快吗？让我们来试试 Core Graphics。

先定义一个全局的 CGImageRef 变量：

```objc
static CGImageRef imageRef;

- (void)awakeFromNib
{
    imageRef = self.image.CGImage;
}
```

然后

```
- (void)drawRect:(CGRect)rect
{
     CGContextRef context = UIGraphicsGetCurrentContext();
     CGContextDrawImage(context, imageRect, imageRef);
}
```

运行一下，发现时间间隔为 2018-07-05 11:36:19.837131+0800 Demo[6677:139386] <font color=#cc0000>27425265</font>，而且图像还上下颠倒了⋯

这个原因是 UIKit 和 Core Graphics 的坐标系 y 轴是相反的，于是加上下面代码来修正：

```objc
CGContextRef context = UIGraphicsGetCurrentContext();
CGContextTranslateCTM(context, 0, 100);
CGContextScaleCTM(context, 1, -1);
CGContextDrawImage(context, imageRect, imageRef);
```

这下图像终于正常显示了，时间增加到了 2018-07-05 11:39:27.557629+0800 Demo[6817:142712] <font color=#cc0000>34242146</font>，成效不大，看来直接用 -drawAtPoint: 和 -drawInRect: 就足够好了。