---
title: UITableView优化
categories: iOS优化
---

## 一、Cell 复用

在可见的页面会重复绘制页面，每次刷新显示都会去创建新的 Cell，非常耗费性能。 

<font color=#008200>解决方案</font>：创建一个<font color=#cc0000>静态变量</font> reuseID，防止重复创建（提高性能），使用系统的缓存池功能。

```objc
static NSString * CELL_RUID = @"CELL";  // 调用次数太多，static 保证只创建一次 reuseID，提高性能

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    // 缓存池中取已经创建的 cell
    UITableViewCell * cell = [tableView dequeueReusableCellWithIdentifier:CELL_RUID
                                                             forIndexPath:indexPath];
    return cell;
}
```

通过 identifier 标识不同类型的 cell，缓存池中只会保存已经被移出屏幕的不同类型的 cell。

```objc
- (nullable __kindof UITableViewCell *)dequeueReusableCellWithIdentifier:(NSString *)identifier;  // Used by the delegate to acquire an already allocated cell, in lieu of allocating a new one.
- (__kindof UITableViewCell *)dequeueReusableCellWithIdentifier:(NSString *)identifier forIndexPath:(NSIndexPath *)indexPath NS_AVAILABLE_IOS(6_0); // newer dequeue method guarantees a cell is returned and resized properly, assuming identifier is registered
```

<font color=#cc0000>复用 Cell 时 不会调用 awakeFromNib</font>。

*   获取方法的区别

dequeueReusableCellWithIdentifier:forIndexPath 如果没有注册复用 identifier，执行这句时会崩溃，提示：

```objc
reason: 'unable to dequeue a cell with identifier CELL - must register a nib or a class for the identifier or connect a prototype cell in a storyboard'
```

dequeueReusableCellWithIdentifier 如果没有注册复用 identifier，语句返回 nil，继续执行会崩溃。提示：

```objc
failed to obtain a cell from its dataSource
```

判断 nil 后可以自己创建 cell。

```
{
    MyCell * cell = [tableView dequeueReusableCellWithIdentifier:@"Cell"];
    if (cell == nil) {
        cell = [[MyCell alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:@"Cell"];
    }
}
```
*   为什么需要 forIndexPath:

因为在返回 cell 之前，会调用委托 tableView:heightForRowAtIndexPath:来确定 cell 尺寸（如果已经定义该函数）。

我们经常在 tableView:cellForRowAtIndexPath: 中为每一个 cell 绑定数据，实际上在调用 cellForRowAtIndexPath: 的时候 cell 还没有被显示出来，为了提高效率我们应该把数据绑定的操作放在 cell 显示出来后再执行，可以在 tableView:willDisplayCell:forRowAtIndexPath: 方法中绑定数据。

注意 willDisplayCell 中 cell 在 tableview 展示之前就会调用，此时 cell 实例已经生成，所以不能更改 cell 的结构，只能是改动 cell 上的 UI 的一些属性，如 label 的内容、控件的隐藏等。


## 二、定义一种（尽量少）类型的 Cell 及善用 hidden 隐藏（显示）subviews

分析 Cell 结构，尽可能的将相同内容的抽取到一种样式 Cell 中。UITableView 真正创建出的 Cell 可能只比屏幕显示的多一点。虽然 Cell 的"体积"可能会大点，但是因为 Cell 的<font color=#cc0000>数量不会很多</font>，完全可以接受的。

好处：

①、减少代码量，减少 Nib 文件的数量，在一个 Nib 文件定义 Cell，容易修改、维护；（<font color=#cc0000>多个 Cell 不是更容易维护？</font>）

②、基于复用机制，真正运行时铺满屏幕所需的 Cell 数量大致是固定的，设为 N 个。如果只有一种 cell，那就是只有 N + c 个 cell 的实例；但是如果有 M 种 cell，那么运行时最多可能会是 M * (N + c) 个 cell 的实例，虽然这可能并不会占用太多内存，但能少一些更好。

既然只定义一种 Cell，那么需要把所有不同类型的 view 都定义好，放在 Cell 里面，通过 hidden 属性控制，来显示不同类型的内容。毕竟，在用户快速滑动中，只是单纯的显示/隐藏 subview 比实时创建要快得多。

尽量少用 [cell addSubview:] 动态添加 View，可以初始化时就添加，然后通过 hidden 属性来控制。


## 三、提前计算并缓存 Cell 的高度

#### 3.1 固定高度的 cell

```objc
self.tableView.rowHeight = 88;
```

直接采用上面方式给定高度，不需要实现 tableView:heightForRowAtIndexPath: 以节省不必要的计算和开销。

#### 3.2 动态高度的 cell

实现代理方法后，上面的 rowHeight 属性的设置将会变成无效。

tableView:estimatedHeightForRowAtIndexPath: -> tableView:heightForRowAtIndexPath: 获取每个 Cell 即将显示的高度，从而确定表格视图的布局，实际是要获取滚动视图的 contentSize，然后调用 tableView:cellForRowAtIndexPath:，获取每个 Cell，进行赋值。如果有很多个 Cell 要显示，那么方法会执行很多次。

<font color=#008200>解决方案</font>：在 Model（Entity）中计算并保存 Cell 的高度。其实 Model 中保存 UI 的参数是很奇怪的，最好放在 MVVM 模式的 ViewModel（视图模型）中，让 Model（数据模型）只负责处理数据。

```objc
@interface Model : NSObject

@property (nonatomic, assign) CGFloat cellHeight;  // Cell 高度

/**
 * @brief  计算高度
 */ 
- (void)calculateCellHeight;

@end
```

在 tableView:heightForRowAtIndexPath: 中尽量不使用 cellForRowAtIndexPath: 方法来获取 cell，如果你需要用到它，只用一次然后缓存结果。

还可以继续进行优化，提前创建真正显示的、需要加工的数据并缓存。如：接口返回 NSString 而展示 NSAttributeString。

## 四、异步绘制（自定义 Cell 绘制）

遇到比较复杂的界面时（复杂点的图文混排），上面缓存行高的方式可能就不能满足要求了。[详细整理：UITableView 优化技巧](http://www.cocoachina.com/ios/20150602/11968.html)

```objc
/**
 *  @brief  cell 添加 draw 方法
 */
- (void)draw
{
    // 异步绘制
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        
    });
}

/** 
 *  @brief  重写 drawRect: 方法 
 */
- (void)drawRect:(CGRect)rect
{
    // 不需要用 GCD 异步线程，因为 drawRect: 本来就是异步绘制的。
}
```

绘制的各个信息都是根据之前算好的布局进行绘制的。这里是需要异步绘制。


## 五、滑动时，按需加载

自定义 Cell 的种类千奇百怪，但它本来就是用来显示数据的，差不多 100% 带有图片，这个时候就要考虑，下滑的过程中可能会有点卡顿，尤其网络不好的时候，异步加载图片是个程序员都会想到，但是如果给每个循环对象都加上异步加载，开启的线程太多，一样会卡顿。这个时候利用 UIScrollViewDelegate 两个代理方法就能很好地解决这个问题。

```objc
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath 
{ 
    if (needLoadArr.count > 0 && [needLoadArr indexOfObject:indexPath] == NSNotFound) {
         [cell clear];  // 清掉内容
    } 
    return cell;
}

// 按需加载 - 如果目标行与当前行相差超过指定行数，只在目标滚动范围的前后指定 3 行加载。
- (void)scrollViewWillEndDragging:(UIScrollView *)scrollView withVelocity:(CGPoint)velocity targetContentOffset:(inout CGPoint *)targetContentOffset
{
    NSIndexPath * ip  = [self.tableView indexPathForRowAtPoint:CGPointMake(0, targetContentOffset->y)];
    NSIndexPath * cip = [[self.tableView indexPathsForVisibleRows] firstObject];
    
    NSInteger skipCount = 8;
    
    // -8 < 当前位置 - 目标位置 < 8
    if (labs(cip.row - ip.row) > skipCount) {
        
        // 目标区域的 cell 的 indexPaths
        NSArray * temp = [self.tableView indexPathsForRowsInRect:CGRectMake(0, targetContentOffset->y, self.tableView.frame.size.width, self.tableView.frame.size.height)];
        
        NSMutableArray * arr = [NSMutableArray arrayWithArray:temp];
        
        if (velocity.y < 0) {
            NSIndexPath * indexPath = [temp lastObject];
            
            if (indexPath.row + 33) {
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row - 3 inSection:0]];
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row - 2 inSection:0]];
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row - 1 inSection:0]];
            }
        }
        [needLoadArr addObjectsFromArray:arr];
    }
}
```

思想：识别 UITableView 拖拽即将结束的时候，进行异步加载图片，快滑动过程中，只加载目标范围内的 Cell，这样按需加载，极大的提高流畅度。而 SDWebImage 可以实现异步加载，与这条性能配合就完美了，尤其是大量图片展示的时候。而且也不用担心图片缓存会造成内存警告的问题。


## 六、缓存 View

当 Cell 中的部分 View 是非常独立且不便于重用的，"体积"非常小，在内存可控的前提下，完全可以将这些 view 缓存起来。


## 七、尽量显示“大小刚好合适的”图片资源

避免大量的图片缩放、颜色渐变等。


## 八、避免同步的从网络、文件获取数据

Cell 内实现的内容来自 web，使用异步加载，缓存请求结果。


## 九、渲染

1、减少 subviews 的个数和层级

      子控件的层级越深，渲染到屏幕上所需要的计算量就越大；如多用 drawRect 绘制元素，替代用 view 显示。

2、少用 subviews 的透明图层

      渲染最耗时的操作之一就是混合(blending)了。对于不透明的 View，设置 opaque = YES，这样在绘制该 View 时，避免 GPU 对 View 覆盖的其他内容也进行绘制。

3、背景色不要使用 clearColor

4、避免 CALayer 特效（shadowPath）

      给 Cell 中 View 加阴影会引起性能问题，如下面代码会导致滚动时有明显的卡顿：

```objc
view.layer.shadowColor   = color.CGColor;
view.layer.shadowOffset  = offset;
view.layer.shadowOpacity = 1;
view.layer.shadowRadius  = radius;
```

5、当有图像时，预渲染图像，在 bitmap context 先将其画一遍，导出成 UIImage 对象，然后再绘制到屏幕，这会大大提高渲染速度。具体内容可以自行查找“利用预渲染加速显示 iOS 图像”相关资料。


## 十、总结

UITableView 的优化主要从四个方面入手：

1、提前计算并缓存好高度（布局），因为 tableView:heightForRowAtIndexPath: 是调用最频繁的方法；

2、滑动时按需加载，防止卡顿。这个在大量图片展示，网络加载的时候很管用，配合 SDWebImage；

3、异步绘制，遇到复杂界面，遇到性能瓶颈时，可能就是突破口；

4、缓存一切可以缓存的，这个在开发的时候，往往是性能优化最多的方向。

大概需要关注的：

1、cell 复用

2、cell 高度的计算

3、渲染（混合问题）

4、减少视图的数目（重写 drawRect:）

5、减少多余的绘制操作

6、不要给 cell 动态添加 subView

7、异步化 UI，不要阻塞主线程

8、滑动时按需加载对应的内容


## 十一、资料

图片加载优化官方 Demo：[LazyTableImages](https://developer.apple.com/library/archive/samplecode/LazyTableImages/Introduction/Intro.html#//apple_ref/doc/uid/DTS40009394-Intro-DontLinkElementID_2)

文章：[提升 UITableView 性能-复杂页面的优化](http://tutuge.me/2015/02/19/%E6%8F%90%E5%8D%87UITableView%E6%80%A7%E8%83%BD-%E5%A4%8D%E6%9D%82%E9%A1%B5%E9%9D%A2%E7%9A%84%E4%BC%98%E5%8C%96/)

代码：[VVeboTableViewDemo](https://github.com/johnil/VVeboTableViewDemo)
