---
title: iOS 内置图片瘦身
categories: iOS优化
---

## 一、iOS 内置资源的集中方式

#### 1.1 将图片存放在 bundle

这是一种很常见的方式，项目中各类文件分类放在各个 bundle 下，项目既整洁又能达到<font color=#cc0000>隔离资源</font>的目的。采用 bundle 的加载方式为 [UIImage imageNamed:"xx.bundle/xx.png"]。

这种方式有比较明显的缺点：

1. iOS 系统不会对其进行压缩存储，造成了应用体积的增大。

2. 使用 bundle 存储图片放弃了 APP thinning。明显的表现是 2 倍屏手机和 3 倍屏手机下载的应用包大小一样。如果能够实现 APP thinning，那么往往 2 倍屏幕的手机包大小会小于 3 倍屏手机的，起到差异性优化的目的。

在调研过程中发现，应用的体积与图片资源的数量密切相关。换句话说，iPhone 的 rom 存在 4K 对齐的情况，一张 498B 大小的图片在应用包中也要占据 4KB 大小。因此项目中每添加一张图片就至少增大了 4KB。

下面来证实。首先创建空应用，其大小在 iPhone7 上为 131KB ，引入一张 3KB 的图片前后对比如下：

![](https://upload-images.jianshu.io/upload_images/5294842-66c00dd63d65a971.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) ![](https://upload-images.jianshu.io/upload_images/5294842-b85758fdfe19f422.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/5294842-94a016df24024865.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) ![](https://upload-images.jianshu.io/upload_images/5294842-da19d42d7f21cb5b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

以上未经过 App Store 上线认证，仅仅通过本地真机运行测试，仅供参考。

#### 1.2 使用 .ttf 字体文件替代图标

使用字体文件替代图片也是一种比较常见的资源内置方式。很多应用都使用过这种方案，如淘宝、爱奇艺等知名应用。

使用字体文件的好处是显而易见的，如果 APP 中某个图片比较大，那么为了保证清晰度，UI 可能会提供比较大的图标。使用字体文件会避免这个问题，而且不必导入 @2x 和 @3x 图片，一套字体文件就能保证 UI 的清晰度。

字体文件使用起来比较简单，但是使用方法与 png 图片的使用方法有很大的不同，因为字体文件实际所展示的图标都是 UTF8 编码转来的字符串。因此当我们需要展示一个图标的时候不再是使用 UIImageView 了，而是 UILabel。

```objc
UILabel * iconLabel = [[UILabel alloc] initWithFrame:CGRectMake(0, 0, 50, 50)];
iconLabel.font = [UIFont fontWithName:@"icomoon" size:50];
iconLabel.text = [NSString stringWithUTF8String:"\ue902"];
```

由于使用了字体来替代图片，所以可以通过设置字体的颜色来改变图标的颜色。之前经常会遇到一个场景，如两个一模一样的图标但是由于颜色不同，UI 就需要提供两套图片，每套图片中包含 @2x 和 @3x 图片。如果采用了字体替代简单的图标，那么 UI 只需要提供一套字体即可，并且拉伸后也不会失真。

优点：

1. 可以降低应用图片内置资源的体积。
2. 可以随意缩放和修改颜色。

缺点：

1. 图标的查找和替换比较麻烦，不如直接使用图片那样简单。
2. 有些情况无法替换之前存在的图片，只能起到缩小增量的目的，无法减小全量。

任何一种需要大刀阔斧改革的优化都是一种不明智的行为。


#### 1.3 图片存在 Assets.xcassets

使用 Assets.xcassets 是苹果推荐的一种方式。Assets.xcassets 是 iOS7 推出的一种图片资源管理工具，将图片内置到Assets.xcassets 下系统会对图片资源进行压缩，并且支持 APP thinning。


## 二、优化

项目优化不能脱离场景，很多很好的方案由于场景的限制并不能起到优化的作用。

为了达到跨团队快速开发的目的，项目很早就利用 cocoapods 实现组件化。项目中存在多个业务 pod，每个 pod 都有各自的团队维护，各个团队的代码彼此不开放，各个 pod 最终会被编译为 .a 的形式。

与 .a 相对应的是 .framework，它们之间有一个重要的区别就是资源的问题。<font color=#cc0000>.framework 中可以存放资源，但 .a 不可以</font>，因此生成 .a 的 pod 下的资源会被转移到 main bundle 下，这为资源冲突造成了隐患。采用的 bundle 管理资源大大降低了资源冲突的可能性，因为 bundle 名很少会重复。

优化的前提之一也是不破坏这种组件化开发的模式，换句话说也就是各个业务线不产生资源耦合、业务线的 RD 不必担心彼此资源的冲突、业务 Pod 下的资源文件彼此隔离。

先要抛出两个问题：

1. cocoapods 是否支持使用 Assets.xcassets。
2. 各个 pod 维护自己的 Assets.xcassets 会不会造成资源冲突。

为了弄清楚上面两个问题，先要看下 podspec 的几个重要参数：

![](https://upload-images.jianshu.io/upload_images/5294842-d91c042de77985a8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
s.source_files ：源文件路径。

s.public_header_files ：表明了哪些路径下的文件可以在 framework 外被引用。

s.resources ：资源文件路径及文件类型。

s.resource_bundles ：资源文件路径及类型，同时资源文件会被打成 bundle。（推荐使用）。
```

实验发现各个 pod 下都可以创建自己的 xcassets，因此问题 ① 确定。

如果我们在各个业务 pod 下都创建 .xcassets 文件内置图片，那么 cocoapods 的脚本会在编译时将各个目录下的 xcassets 文件内容提取出来，合并到一个 xcassets 中并生成一个 .car 文件。这样的话如果资源文件重名，那么很可能其中某一个文件会被覆盖替换。因此我们主要是要解决问题 ②。

查看 podspec 的写法发现 s.resource_bundles 貌似是我们所需要的法宝。

![](https://upload-images.jianshu.io/upload_images/5294842-7ad0badccde39e2c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最终打包结果很理想，确实能够生成 Demo.bundle，并且 bundle 下存在 Assets.car。

![](https://upload-images.jianshu.io/upload_images/5294842-26d283bdb3e0ab53.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

运行发现通过 [UIImage imageNamed:@"Demo.bundle/1"];加载不出来图片。必须使用 [UIImage imageNamed:@"1" inBundle:bundle compatibleWithTraitCollection:nil]; 才能加载出来。也就是说如果 Assets.car 不在 main bundle 下，那么加载图片需要指定 bundle。

既然需要指定 bundle 加载图片，那么如何获取这个 bundle 呢？换句话说如何才能低成本的将项目中的图片放到特定 bundle 下的 Assets.car 文件中呢？对此我们提出了一个解决方案：

1. 在 pod 下新建一个空文件夹。找出该 pod 存放图片的所有 bundle，在新建文件夹下创建与 bundle 数量相等的 Asset。
2. 修改 podspec 文件，设置 resource_bundles 将 Asset 指定为资源，并指定 bundle 名称，如 A.bundle，其对应的 Asset 最终资源 bundle 为 A\_Asset.bundle。
3. 新增方法 imageWithName:，从符合 xx.bundle/yy.png 特征的参数中获取 bundle 名和图片名 xx\_Asset.bundle 和 yy.png，获取图片并返回。
4. 查找并全部替换 imageNamed: 和 imageWithContentOfFile: 为 imageWithName:。

只要能拿到原来代码中 imageNamed: 的参数就能知道现在图片存在哪个 bundle 下，这样就能通过 imageNamed:inBundle: 获取到图片，其思路如下图所示：

![](https://upload-images.jianshu.io/upload_images/5294842-1d1c0b0a5829ddce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/5294842-25e5c59026031224.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看到这里已经应该能遇见这种优化的成本了。加载图片都需要指定 bundle 也就意味着成千上万处的 API 需要修改。我们最初探讨到这里的时候首先想到的是脚本，但是这个方案很快就被否定了，因为项目中存在大量的 XIB，XIB 中设置图片我们无法通过脚本替换 API。

为了解决 XIB 设置图片的问题，我们首先想到了 AOP。通过 hook Xib 加载图片的方法将方法偷偷替换为 imageNamed:inBundle:，但是很遗憾 hook 了 UIImage 所有加载图片的方法，没有一个方法能拿到 XIB 上所设置的图片名，也就意味着我们无法得知优化后的图片在哪个 bundle 下，也就不知道图片该如何加载。虽然有坎坷，但是我们始终坚信 XIB 一定是通过某些方法将图片加载出来的，我们一定能拿到这个过程！为了验证这个问题，首先定义一个 UIImageView 的子类，并将XIB 上的 UIImageView 指定为这个子类。大家都知道通过 XIB 加载的视图都一定会执行 initWithCoder: 方法。

![](https://upload-images.jianshu.io/upload_images/5294842-e90e00f3d0c80b1f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

发现在执行 [super initWithCoder:aDecoder] 之前通过 lldb 查看 self.image 是 nil。当执行完这行代码后 self.image 就有值了。因此推断图片的信息（图片名称、路径等信息）都在 aDecoder 中！在网上搜索了一些资料后发现aDecoder 有一些固定的 key，可以通过这些固定的 key 得到一部分信息。如

![](https://upload-images.jianshu.io/upload_images/5294842-b33cce60e4a67ef3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

很显然通过 UIImage 这个 key 能拿到图片，但是很遗憾经过多次尝试没能找到图片的路径信息。因此这个问题的关键是怎么找到合适的 key，为了解决这个问题，最好是能拿到 aDecoder 的解码过程。因此 hook aDecoder 的解码方法 decodeObjectForKey:是个不错的选择。如果能拿到 xib 上设置的图片名称，那么我们就可以根据图片名称获取到正确的图片路径。经过断点查看 aDecoder 是 UINibDecoder（私有类）类型。

![](https://upload-images.jianshu.io/upload_images/5294842-a3bfbde2e9e119c1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```objc
- (id)swizzle_decodeObjectForKey:(NSString *)key
{
    Method originalMethod = class_getInstanceMethod([HookTool class], @selector(swizzle_decodeObjectForKey:));
    IMP function = method_getImplementation(originalMethod);
    id (*functionPoint)(id, SEL, id) = (id (*)(id, SEL, id)) function;
    id value = functionPoint(self, _cmd, key);
    
    return value;
}
```

打印系统 decode 的所有 key 后发现有个 key 为 UIResourceName，value 为图片的名称。也就是说我们能得到 XIB 上设置的图片名称了。但是这个图片的名称怎么传递给这个 XIB 对应的 UIImageView 对象呢？换句话说也就是说我们怎么把图片传给这个 XIB 对应的 view 呢？为了将图片名称传给 UIImageView，需要给 aDecoder 添加一个 block 的关联引用。

![](https://upload-images.jianshu.io/upload_images/5294842-b2355951a7c86b6d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在 hook 到的 decodeObjectForKey: 方法中将图片名称回传给 initWithDecoder: 方法。

![](https://upload-images.jianshu.io/upload_images/5294842-6d040ecd395f34fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里需要注意的是一点是：XIB 默认设置图片是在 rentun value 之后，也就是说如果我们回调过早有可能图片被替换为 nil。因此需要 dispatch_after 一下，等 return 之后再回调图片名称并设置图片。受此启发，我们也可以 hook UIImage 的imageNamed: 方法，根据参数的规则到 xxxCopy.bundle 下获取图片，并返回图片。这就意味着放弃通过脚本修改 API，减少了代码的改动。看到这里似乎是没有什么问题，但是我们忽略了一个很严重的问题 aDecoder 对象和 UIImageView 类型的对象是一一对应的吗？一个 imageView 它的 aDecoder 是它唯一拥有的吗？带着这个问题，我们先来看下打印信息：

![](https://upload-images.jianshu.io/upload_images/5294842-e65336fc35468c15.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

重复生成对象并打印后发现 aDecoder 的地址都相同，也就是说存在一个 aDecoder 对应多个 UIImageView 的现象。因此异步方案不适用，需要同步进行设置图片，因此全局变量最为合适。其实这一点很容易理解，aDecoder 是与 XIB 对应的，XIB 是不变的所以 aDecoder 是不变的。因此异步回调的方案不适用，需要同步进行设置图片，在这种情况（主线程串行执行）下跨类传值全局变量最为合适。

```objc
- (id)swizzle_decodeObjectForKey:(NSString *)key
{
    Method originalMethod = class_getInstanceMethod([HookTool class], @selector(swizzle_decodeObjectForKey:));
    IMP function = method_getImplementation(originalMethod);
    id (*functionPoint)(id, SEL, id) = (id (*)(id, SEL, id)) function;
    id value = functionPoint(self, _cmd, key);

    NSString* propKey = @"emaNecruoseRIU";
    // 反转字符串
    propKey = [XUtil stringByReversed:propKey];

    if ([key isEqualToString:propKey]) {
        if (normal_imageName) {
            select_imageName = value;
        }
        else {
            normal_imageName = value;
        }
    }
    
    return value;
}
```

hook UIImageView 的 initWithCoder:

```objc
- (id)swizzle_imageView_initWithCoder:(NSCoder *)aDecoder
{
    // 执行顺序：initWithCoder -》DecoderWithKey -》setImage：，所以每次给 imageView 设置图片时，需要将之前的置空。
    // tabbarItem 的图片设置不会执行 initWithCoder，如果不置空，会导致 imageView 设置成和 tabbarItem 一样的图片。
    normal_imageName = nil;
    select_imageName = nil;

    UIImageView * instance = (UIImageView *)[self swizzle_imageView_initWithCoder:aDecoder];

    if (normal_imageName && [normal_imageName isKindOfClass:[NSString class]] && normal_imageName.length > 0) {
        
        UIImage * normalImage = [HookTool imageAfterSearch:normal_imageName];
        // 赋值
        if (normalImage) {
            instance.image = normalImage;
        }
        normal_imageName = nil;
        select_imageName = nil;
    }
    
    return instance;
}
```

上面两段代码仅仅介绍思路。同理 hook 项目中 UIImage 所用到的加载图片的 API 即可加载图片。如果将所有的 hook 方法放到一个类中，那么只要将这个类拖入到项目中，并将项目中所有的 bundle 下的图片都放到对应的 Assets.xcassets 文件下那么无需修改一行代码即可将所有的图片迁移到 Assets.xcassets 下，达到应用瘦身的目的。

但是我们组内老练的架构师们指出：项目中 hook 如此重要的 API 对<font color=#cc0000>增加了项目维护的难度</font>。这也引发了对项目中 AOP 场景的思考，项目中到底 hook 了多少 API？为此特地赶制了一个基于 fishhook 的一个 hook 打印工具，检测和统计项目中的 AOP 情况。但是缺点是必须调整编译顺序保证工具类最先被 load。

hook method_exchangeImplementations 方法。

![](https://upload-images.jianshu.io/upload_images/5294842-53182782221fe7cd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

检测方法（字典写入时不要忘了加锁）。

![](https://upload-images.jianshu.io/upload_images/5294842-fe1b0b6cdef8e479.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<font color=#cc0000>这种方式不能区分 image 和 backgroundImage、normal 和 Selected</font>。目前根据观察顺序应该是：

```objc
UIResourceName ： normal - image(前景图)
UIResourceName ： normal - backgroundImage(背景图)
UIResourceName ： selected - image(前景图)
UIResourceName ： selected - backgroundImage(背景图)
```