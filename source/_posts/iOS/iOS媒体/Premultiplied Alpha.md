---
title: Premultiplied Alpha
categories: iOS媒体
---

Xcode 的工程选项里有一项 Compress PNG Files，会对 PNG 进行 Premultiplied Alpha。游戏开发中会更加关注这个格式，省一些运行时计算。

Premultiplied Alpha 是什么呢？

[Alpha Blending: To Pre or Not To Pre](https://developer.nvidia.com/content/alpha-blending-pre-or-not-pre) 这篇文章其实说的很清楚。还有《Real Time Rendering》

## 一、Alpha Blending

要搞清楚这个问题，先得理解 Alpha 通道的工作原理。

最常见的像素表示格式是 RGBA8888 即 （r, g, b, a），每个通道 8 位，0~255。例如红色 60% 透明度就是（255, 0, 0, 153），为了表示方便，alpha 通道一般记成正规化后的 0~1 的浮点数，也就是（255, 0, 0, 0.6）。而 Premultiplied Alpha 则是把 RGB 通道乘以透明度也就是（r * a, g * a, b * a, a），50% 透明红色就变成了（153, 0, 0, 0.6）。

[透明通道](https://www.baidu.com/s?wd=%E9%80%8F%E6%98%8E%E9%80%9A%E9%81%93&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)在渲染的时候通过 Alpha Blending 产生作用，如果一个透明度为 a<sub>s</sub> 的颜色 C<sub>s</sub> 渲染到颜色 C<sub>d</sub> 上，混合后的颜色通过以下公式计算：

C<sub>o</sub> = a<sub>s</sub>C<sub>s</sub> \+ (1 − a<sub>s</sub>)C<sub>d</sub>

以 60% 透明的红色渲染到白色背景为例：

C<sub>o</sub> = (255, 0, 0) \* 0.6 \+ (255, 255, 255) *(1 − 0.6) = (255, 102, 102)

也就是说，从视觉上（255, 0, 0, 0.6）渲染到白色背景上和（255, 102, 102）是同一个颜色。如果颜色以 Premultiplied Alpha 形式存储，也就是 C<sub>s</sub> 已经乘以透明度了，所以混合公式变成了：

C<sub>o</sub> = C<sub>s</sub>′ \+ (1 − a<sub>s</sub>)C<sub>d</sub>

## 二、为什么要 Premultiplied Alpha？

Premultiplied Alpha 后的像素格式变得不直观，因为在画图的时候都是先从调色板中选出一个 RGB 颜色，再单独设置透明度，如果 RGB 乘以透明度就搞不清楚原色是什么了。从前面的 Alpha Blending 公式可以看出，Premultiplied Alpha 之后，混合的时候可以<font color=#cc0000>少一次乘法，这可以提高一些效率</font>，但这并不是最主要的原因。最主要的原因是：

<font color=#cc0000>没有 Premultiplied Alpha 的纹理无法进行 Texture Filtering</font>（除非使用最近邻插值）。

以最常见的 filtering 方式线性插值为例，一个宽 2px 高 1px 的图片，左边的像素是红色，右边是绿色 10% 透明度，如果把这个图片缩放到 1x1 的大小，那么缩放后 1 像素的颜色就是左右两个像素线性插值的结果，也就是把两个像素各个通道加起来除以2。如果使用没有 Premultiplied Alpha 的颜色进行插值，那么结果就是：

((255, 0, 0, 1) \+ (0, 255, 0, 0.1)) \* 0.5 = (127, 127, 0, 0.55)

如果绿色 Premultiplied Alpha，也就是（0, 255 * 0.1, 0, 0.1），和红色混合后：

((255, 0, 0, 1) \+ (0, 25, 0, 0.1)) \* 0.5 = (127, 25, 0, 0.55)

Premultiplied Alpha 最重要的意义是<font color=#cc0000>使得带透明度图片纹理可以正常的进行线性插值</font>。这样旋转、缩放或者非整数的纹理坐标才能正常显示，否则就会像上面的例子一样，在透明像素边缘附近产生奇怪的颜色。

## 三、纹理处理

我们使用的 PNG 图片纹理，一般是不会 Premultiplied Alpha 的。游戏引擎在载入 PNG 纹理后会手动处理，然后再 glTexImage2D 传给 GPU，比如 Cocos2D-x 中的 CCImage::premultipliedAlpha：

```
void Image::premultipliedAlpha() {
    unsigned int* fourBytes = (unsigned int*)_data;
    for (int i = 0; i < _width * _height; i++) {
        unsigned char* p = _data + i * 4;
        fourBytes[i] = CC_RGB_PREMULTIPLY_ALPHA(p[0], p[1], p[2], p[3]);
    }  
    _hasPremultipliedAlpha = true;
}
```

而 GPU 专用的纹理格式，比如 PVR、ETC 一般在生成纹理都是默认 Premultiplied Alpha 的，这些格式一般是 GPU 硬解码，引擎用 CPU 处理会很慢。

总之 glTexImage2D 传给 GPU 的纹理数据最好都是 Multiplied Alpha 的，要么在生成纹理时由纹理工具 Pre-multiplied，要么载入纹理后由游戏引擎或 UI 框架 Post-multiplied。

## 四、iOS 中的 Premultiplied Alpha

Core Graphics 的 CGImage.h 对图像透明度信息有如下定义

```
typedef CF_ENUM(uint32_t, CGImageAlphaInfo){
    // ...
    /* For example, premultiplied RGBA */
    kCGImageAlphaPremultipliedLast, 
    /* For example, premultiplied ARGB */ 
    kCGImageAlphaPremultipliedFirst, 
    // ...
};
```

预乘透明度（Premultiplied Alpha**）**图像简单地说，即每个颜色分量都乘以 alpha 通道值作为结果值：

	color.rgb *= color.alpha

为什么关注预乘透明度图像？微信团队因 AR 抢红包场景的 OpenGL 混色结果出错引起注意。

> Premultiplied alpha is better than conventional blending for several reasons:
> 
> It works properly when filtering alpha cutouts \_(see below)\_
> 
> It works properly when doing image composition \_(stay tuned for my next post)\_
> 
> It is a superset of both conventional and additive blending. If you set alpha to zero while RGB is non zero, you get an additive blend. This can be handy for particle systems that want to smoothly transition from additive glowing sparks to dark pieces of soot as the particles age.
> 
> It plays nice with [DXT compression](http://blogs.msdn.com/shawnhar/archive/2008/10/28/texture-compression.aspx), which only supports transparent pixels with an RGB of zero.
> 
> 摘自：[Premultiplied alpha](https://blogs.msdn.microsoft.com/shawnhar/2009/11/06/premultiplied-alpha/)