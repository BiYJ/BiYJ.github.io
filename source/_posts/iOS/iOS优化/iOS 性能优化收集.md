---
title: iOS 性能优化收集
categories: iOS优化
---


[iOS 性能调试](https://www.cnblogs.com/zhangying-domy/p/5948613.html)

## instrument

[Instrument](https://www.jianshu.com/p/6a3b1d29f657)
[Instrument之Core Animation工具](https://blog.csdn.net/xiaoxiaobukuang/article/details/51076944)

1. 避免图层混合

	①、确保控件的opaque属性设置为true，确保backgroundColor和父视图颜色一致且不透明；
	②、如无特殊需要，不要设置低于1的alpha值；
	③、确保UIImage没有alpha通道；

2. 避免临时转换

	①、确保图片大小和frame一致，不要在滑动时缩放图片；
	②、确保图片颜色格式被GPU支持，避免劳烦CPU转换；

3. 慎用离屏渲染

	①、绝大多数时候离屏渲染会影响性能；
	②、重写drawRect方法，设置圆角、阴影、模糊效果，光栅化都会导致离屏渲染；
	③、设置阴影效果是加上阴影路径；
	④、滑动时若需要圆角效果，开启光栅化；


[ios性能优化--label上汉字图层混合问题](https://www.jianshu.com/p/b8ee7a40e219)
[iOS 正确设置圆角](https://www.jianshu.com/p/3c47bf0dea80)