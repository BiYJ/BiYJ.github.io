---
title: CoreText
---


<center>
![](http://dzliving.com/CoreText_0.png)
![](http://dzliving.com/CoreText_1.png)
</center>

    
你不需要自己创建 CTRun。CoreText 将根据 NSAttributedString 的属性来自动创建 CTRun。每个 CTRun 对象对应不同的属性，正因此，你可以自由的控制字体、颜色、字间距等等信息。


通常步聚：

1. 使用 core text 就是先有一个要显示的 string
2. 然后定义这个 string 每个部分的样式 attributedString
3. 生成 CTFramesetter
4. 得到 CTFrame
5. 绘制 CTFrameDraw，其中可以更详细的设置换行方式、对齐方式、绘制区域的大小等。
6. 点击事件：CTFrame 包含了多个 CTLine，并且可以得到各个 line 的真实位置与大小。判断点击处在不在某个 line 上。CTLine 又可以判断这个点（相对于 CTLine 的坐标）处的文字范围。然后遍历这个 string 的所有 NSTextCheckingResult，根据 result 的 Range 判断点击处在不在这个 Range上，从而得到点击的链接与位置。


<center>
![](http://dzliving.com/CoreText_2.png)
![](http://dzliving.com/CoreText_3.png)
![](http://dzliving.com/CoreText_4.png)

![](http://dzliving.com/CoreText_5.png)

![](http://dzliving.com/CoreText_6.png)
</center>