---
title: iOS CALayer
categories: iOS原理
---


## 一、CoreGraphics

这是一个 C 语言写就的库，来看看他的头文件:

```
#ifndef COREGRAPHICS_H_
#define COREGRAPHICS_H_

#include <CoreGraphics/CGBase.h>

#include <CoreGraphics/CGAffineTransform.h>
#include <CoreGraphics/CGBitmapContext.h>
#include <CoreGraphics/CGColor.h>
#include <CoreGraphics/CGColorConversionInfo.h>
#include <CoreGraphics/CGColorSpace.h>
#include <CoreGraphics/CGContext.h>
#include <CoreGraphics/CGDataConsumer.h>
#include <CoreGraphics/CGDataProvider.h>
#include <CoreGraphics/CGError.h>
#include <CoreGraphics/CGFont.h>
#include <CoreGraphics/CGFunction.h>
#include <CoreGraphics/CGGeometry.h>
#include <CoreGraphics/CGGradient.h>
#include <CoreGraphics/CGImage.h>
#include <CoreGraphics/CGLayer.h>
#include <CoreGraphics/CGPDFArray.h>
#include <CoreGraphics/CGPDFContentStream.h>
#include <CoreGraphics/CGPDFContext.h>
#include <CoreGraphics/CGPDFDictionary.h>
#include <CoreGraphics/CGPDFDocument.h>
#include <CoreGraphics/CGPDFObject.h>
#include <CoreGraphics/CGPDFOperatorTable.h>
#include <CoreGraphics/CGPDFPage.h>
#include <CoreGraphics/CGPDFScanner.h>
#include <CoreGraphics/CGPDFStream.h>
#include <CoreGraphics/CGPDFString.h>
#include <CoreGraphics/CGPath.h>
#include <CoreGraphics/CGPattern.h>
#include <CoreGraphics/CGShading.h>


#endif  /* COREGRAPHICS_H_ */
```

通过头文件可以看到，CoreGraphics 的类名都是以 CG 开头的，平时所用的 CGRect、CGPoint 就在CGGeometry 这个几何相关的类中定义，CGFont 类则被封装成了 UIFont，CGImage 构成了 UIImage，CGContext 是绘图的上下文等等。所以 CoreGraphics 是系统绘制界面、文字、图像等UI的基础。

## 二、Quartz2D

这是一个基于CoreGraphics API的绘图框架，系统中并没有Quartz2D.framework这么一个库，他只是包含了CoreGraphics中的部分API，是一个抽象的引擎，并不是一个实体，他在iOS和MAC系统中负责：

1. 绘制图形 : 线条/三角形/矩形/圆/弧等
2. 绘制文字
3. 绘制/生成图片(图像)
4. 读取/生成 PDF
5. 截图/裁剪图片
6. 自定义 UI 控件

因为是 API 是 C 语言写成的，所以 ARC 并不起作用，仍然需要手动管理内存。

## 三、QuartzCore 和 CoreAnimation

打开QuartzCore的头文件可以看到

```
#ifndef QUARTZCORE_H
#define QUARTZCORE_H

#include <QuartzCore/CoreAnimation.h>

#endif /* QUARTZCORE_H */
```

QuartzCore就是引用了CoreAnimation的头文件。

CoreAnimation中文名核心动画，看名字是负责动画的，其实不然，作用相当大，来看看他的头文件

```
#ifndef COREANIMATION_H
#define COREANIMATION_H

#include <QuartzCore/CABase.h>
#include <QuartzCore/CATransform3D.h>

#ifdef __OBJC__
#import <Foundation/Foundation.h>
#import <QuartzCore/CAAnimation.h>
#import <QuartzCore/CADisplayLink.h>
#import <QuartzCore/CAEAGLLayer.h>
#import <QuartzCore/CAEmitterBehavior.h>
#import <QuartzCore/CAEmitterCell.h>
#import <QuartzCore/CAEmitterLayer.h>
#import <QuartzCore/CAGradientLayer.h>
#import <QuartzCore/CALayer.h>
#import <QuartzCore/CAMediaTiming.h>
#import <QuartzCore/CAMediaTimingFunction.h>
#import <QuartzCore/CAReplicatorLayer.h>
#import <QuartzCore/CAScrollLayer.h>
#import <QuartzCore/CAShapeLayer.h>
#import <QuartzCore/CATextLayer.h>
#import <QuartzCore/CATiledLayer.h>
#import <QuartzCore/CATransaction.h>
#import <QuartzCore/CATransform3D.h>
#import <QuartzCore/CATransformLayer.h>
#import <QuartzCore/CAValueFunction.h>
#endif

#endif /* COREANIMATION_H */
```

以CA开头的都是他的类，其中带layer的类是构成UIView的基石，用来呈现内容。其中：

1. CAShapeLayer

	用来根据CGPath来渲染的图层，这里CGPath就是上文中提到的CoreGraphics的一部分，比如常做的圆角，都是用cornerRadius来做，但是这是四个角都有的，要想做一个角的就得用他了。

2. CATextLayer

	这个layer可以呈现文字，包含了 UILabel 的功能，如果你闲的蛋疼，也可以用他来实现一个 label。

3. CAGradientLayer

	是用来生成两种或更多颜色平滑渐变的。

4. CAReplicatorLayer

	是为了高效生成许多相似的图层。它会绘制一个或多个图层的子图层，并在每个复制体上应用不同的变换。

5. CAScrollLayer

	有点像UIScrollView，可以用来呈现比他大的内容。

6. CATiledLayer

	Tiled意思的瓷砖，就像铺地板一样，可以把一个很大的图片，切成一格一格的来呈现。

7. CAEAGLLayer

	用来进行 OpenGL 绘图的工作，需要有 OpenGL 的基础。

这其中可能存在疑问，比如既然有了 CATextLayer，为啥还要 UILabel，有了 CAScrollLayer 还要有 UIScrollView。答案是 layer 层只能呈现视图，不能响应用户事件，比如输入，点击，拖拽等，所以才会有UIKit对layer的封装，用来处理事件，然后用layer来呈现内容。

CAAnimation类继承与NSObject，他是一个抽象类，并不直接负责动画，他有一个子类CAPropertyAnimation，也是抽象类。CAPropertyAnimation的两个子类才直接对layer层进行动画操作，其中CABasicAnimation负责基础动画，CAKeyframeAnimation负责关键帧动画。

带有Emitter的类是负责粒子动画的类，可以用来做炫酷的粒子动画。

带有 Trans 的类负责仿射动画，所谓仿射，就是模仿阳光照射，就有了立体的感觉，可以做三维空间的变换，不像 CAAnimation，只能在平面上动手脚。

CAMediaTiming 是一个协议，里边规定了几个属性，用来精确控制时间，animation和layer实现了这个协议。

## 四、CoreImage

照例，来看看头文件

```
#ifdef __OBJC__

#import <CoreImage/CoreImageDefines.h>
#import <Foundation/Foundation.h>

#define UNIFIED_CORE_IMAGE 1

#import <CoreImage/CIVector.h>
#import <CoreImage/CIColor.h>
#import <CoreImage/CIImage.h>
#import <CoreImage/CIContext.h>
#import <CoreImage/CIFilter.h>
#import <CoreImage/CIKernel.h>
#import <CoreImage/CIDetector.h>
#import <CoreImage/CIFeature.h>
#import <CoreImage/CIImageProvider.h>
#import <CoreImage/CIImageProcessor.h>
#import <CoreImage/CIImageAccumulator.h>
#import <CoreImage/CIFilterConstructor.h>
#import <CoreImage/CIFilterShape.h>
#import <CoreImage/CISampler.h>
#import <CoreImage/CIRAWFilter.h>
#if !TARGET_OS_IPHONE
#import <CoreImage/CIFilterGenerator.h>
#import <CoreImage/CIPlugIn.h>
#endif

#endif /* __OBJC__ */
```

头文件里可以看出来，这个库主要针对图片，跟上文里讲到的不太相关。

早年间，想要对图片做滤镜操作，要用到一个叫 GPUImage 的第三方库，如今，系统有了自己的库，可以对图片进行操作。其中 CIFilter 定义了各种滤镜，比如把图片变成黑白的，变成仿古的。还有以前流行的模糊图片等等，在美颜相机上做的，就能用这个库来做。

## 五、总结

写了这么多，可能又蒙了，这里总结一下

1. Quartz2D 是 CoreGraphics 的一部分 API 的抽象，不是实际存在的 .framework
2. CoreGraphics定义了颜色、位置、字体、路径、图片等UIKit的常见属性。是构成UIKit的基石。
3. QuartzCore和CoreAnimation是雌雄同体的同义词。
4. CoreAnimation定义了动画类来对layer做动画，定义了layer来呈现内容。定义了仿射变换来做3D动画。
5. CoreImage定义了滤镜，来对图片进行颜色过滤混合等操作。

UIKit 里的 UIView，封装了 layer 来呈现内容，内容通过 CoreGraphics 来绘制到layer上，其中位置、大小、颜色，也都在 CoreGraphics 里定义了。并且加上了用户事件，用来响应用户的输入、点击、拖拽等操作。



[青山不改](https://www.jianshu.com/u/b9fffc0fd80f) - [傻傻分不清：Quartz2D、QuartzCore、CoreAnimation、CoreImage、CoreGraphics](https://www.jianshu.com/p/397690fd4555)


## 六、CALayer

<center>
![](http://dzliving.com/iOSCALayer_1.png)
</center>

* 最底层是图形硬件(GPU)
* 上层是 OpenGL 和 CoreGraphics，提供一些接口来访问 GPU；
* 再上层的 CoreAnimation 在此基础上封装了一套动画的 API；
* 最上面的 UIKit 属于应用层，处理与用户的交互。

CoreAnimation 属于 <font color=#cc0000>QuartzCore</font> 框架。

Quartz 原本是 macOS 的 Darwin 核心之上的绘图技术，现在也在 iOS 平台中使用。iOS 开发使用 UIKit 库中 UIView，在 macOS 中对应 AppKit 里的 NSView，因为 macOS 是基于鼠标指针操作的系统，与 iOS 的多点触控有本质的区别，然而，虽然 iOS 在交互上与 macOS 有所不同，但在显示层面却可以使用同一套技术，都是通过 QuartzCore 中的 CALayer 显示出来的，动画效果也是加在这个 CALayer 上的。

CALayer 图层类是 CoreAnimation 的基础，它提供了一套抽象概念。CALayer 是整个图层类的基础，它是所有核心动画图层的父类。

> 图层不但给自己提供可视化的内容和管理动画，而且充当了其他图层的容器类，构建图层层次结构


## 七、图层树

layer tree 分为：

1. model layer tree：模型图层树
2. presentation layer tree：表示图层树
3. render layer tree：渲染图层树

![](http://dzliving.com/iOSCALayer_0.jpeg)

官方文档说明：

>模型图层树中的对象是应用程序与之交互的对象。此树中的对象是存储任何动画的目标值的模型对象。每当更改图层的属性时，都使用其中一个对象。
>
>表示图层树中的对象包含任何正在运行的动画的飞行中值。模型图层树对象包含动画的目标值，而表示树中的对象反映屏幕上显示的当前值。您永远不应该修改此树中的对象。相反，您可以使用这些对象来读取当前动画值，也许是为了从这些值开始创建新动画。
>
>渲染图层树中的对象执行实际动画，并且是 Core Animation 的私有动画。

这三个图层在 CALayer 中可以使用的属性有两个，分别是：modelLayer、presentationLayer。渲染图层在 CALayer 没有提供直接的属性给开发者使用，是 core Animation 私有的。


#### 7.1 modelLayer

模型图层就是创建一个 layer，然后给这个 layer 赋上你需要的数据。大家是不是在创建 layer 后，都会给生成的layer 赋值各种属性，通过这种赋值，有没感觉到这个layer本身就是modelLayer呢？那我告诉大家，必须的、必须的、必须的。重要的事情说三边。。。 layer = layer.modelLayer.（后面的栗子会证明这点）

#### 7.2 presentationLayer

“表现图层”就是当前显示在屏幕上的图层。屏幕刷新时，就会调用 presentationLayer。

在 core animation 动画中，可以通过这个属性，获取动画过程中每个时刻动画图层的数据，这样如果在动画过程中需要做什么处理，就可以动态的获取 layer 上相关的数据了。动画的过程中presentationLayer 是时刻变化的，而 modelLayer 是不会变的。

```
CABasicAnimation * animation = [CABasicAnimation animationWithKeyPath:@"position.x"];
animation.fromValue = @(__layer.position.x);
animation.toValue = @(__layer.position.x + 150);
animation.duration = 2.0;
animation.delegate = self;
animation.removedOnCompletion = NO;
animation.fillMode = kCAFillModeForwards;
[__layer addAnimation:animation forKey:@"LAYER_LAYERTREE"];
    
NSLog(@"layer = %@\nmodelLayer = %@\npresentationLayer = %@", __layer, __layer.modelLayer, __layer.presentationLayer);


layer = <CALayer: 0x6000007db240>
modelLayer = <CALayer: 0x6000007db240>
presentationLayer = <CALayer: 0x6000007de9a0>

modelLayerFrame = {{25, 125}, {50, 50}}
presentationLayerFrame = {{174.99992847442627, 125}, {50, 50}}
```

为什么 presentationLayer 的 frame 会发生变化呢？

因为做动画的时候改变了 layer 的 position.x 值，position 值的改变，会影响 frame 的值。

从这里我们就可以看出 modelLayer 和 presentationLayer 本质区别了，modelLayer 负责存储动画的目标值的模型对象。每当更改图层的属性时，它都会把数据存储下来。当动画开始执行时，presentationLayer 就上场了。屏幕上显示的就是 presentationLayer，动画的过程中，你可以时刻访问动态变化的数据。例如：视频中的滚动弹幕如果是使用 layer 做动画的，当弹幕正在滚动时，你需要点击它以处理需要做的事情，这时候你就会需要 presentationLayer。

#### 7.3 总结

>layer = laye.modelLayer，
>layer != layer.presentationLayer。 

CALayer 内部控制两个属性 presentationLayer 和 modelLayer，modelLayer 为当前 layer 真实的状态，presentationLayer 为当前 layer 在屏幕上展示的状态。presentationLayer 会在每次屏幕刷新时更新状态，如果有动画则根据动画获取当前状态进行绘制，动画移除后则取 modelLayer 的状态。



## 文章

[CALayer与iOS动画 讲解及使用](https://blog.csdn.net/zmmzxxx/article/details/74276077)