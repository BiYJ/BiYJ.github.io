---
title : 静态、动态 pod
categories : iOS架构
---

## 一、静态和动态

在项目中使用 pod 实现模块化，对于子模块和第三类库的导入方式存在两种：静态库、动态库。

当在 podfile 中指定 use\_frameworks! 时，子模块和第三方类库将被打包成 ``.framework`` 动态库，模块之间的代码不能直接引用，需要添加依赖；

<center>
![](https://coding-net-production-file-ci.codehub.cn/75905080-cb13-11e9-9abe-7341ed163cae.png?sign=RRAM4F+VgvaHo5cWxdQc/B0+mPlhPTEyNTcyNDI1OTkmaz1BS0lEYXk4M2xGbWFTNlk0TFRkek1WTzFTZFpPeUpTTk9ZcHImZT0xNTY3Mzc4MDE4JnQ9MTU2NzE2MjAxOCZyPTY0OTk0NDg3JmY9Lzc1OTA1MDgwLWNiMTMtMTFlOS05YWJlLTczNDFlZDE2M2NhZS5wbmcmYj1jb2RpbmctbmV0LXByb2R1Y3Rpb24tZmlsZQ==)
</center>

反之（默认情况）将打包成 ``.a`` 静态库。

<center>
![](https://coding-net-production-file-ci.codehub.cn/72253910-cb13-11e9-9abe-7341ed163cae.png?sign=M90GRY0IuaFLhxUmCdGCIwZWBU9hPTEyNTcyNDI1OTkmaz1BS0lEYXk4M2xGbWFTNlk0TFRkek1WTzFTZFpPeUpTTk9ZcHImZT0xNTY3Mzc4MDU5JnQ9MTU2NzE2MjA1OSZyPTg4OTQ2NDMyJmY9LzcyMjUzOTEwLWNiMTMtMTFlOS05YWJlLTczNDFlZDE2M2NhZS5wbmcmYj1jb2RpbmctbmV0LXByb2R1Y3Rpb24tZmlsZQ==)
</center>

动态库和静态库的区别：

- 资源加载方式
- 包的大小 
- 编译速度

#### 1.1 资源加载方式

1. s.dependency 'xx’

	静态方式中各模块的 podspec 文件不用设置依赖，就可以直接 #import 其他模块的类头文件。
	
	<center>
￼	![](https://coding-net-production-file-ci.codehub.cn/f62209b0-cb0d-11e9-87ff-492b3da9ab9b.png?sign=zeTBf+qWdiDpqv3D9g3V/XEc9qhhPTEyNTcyNDI1OTkmaz1BS0lEYXk4M2xGbWFTNlk0TFRkek1WTzFTZFpPeUpTTk9ZcHImZT0xNTY3Mzc1ODUxJnQ9MTU2NzE1OTg1MSZyPTYwMTg1NTk1JmY9L2Y2MjIwOWIwLWNiMGQtMTFlOS04N2ZmLTQ5MmIzZGE5YWI5Yi5wbmcmYj1jb2RpbmctbmV0LXByb2R1Y3Rpb24tZmlsZQ==)
	</center>
	
	而动态方式则会报错。

	<center>
	![](https://coding-net-production-file-ci.codehub.cn/9fc06e80-cb0e-11e9-87ff-492b3da9ab9b.png?sign=mV3UXDHRZj69PKMsyZJEpYYhcblhPTEyNTcyNDI1OTkmaz1BS0lEYXk4M2xGbWFTNlk0TFRkek1WTzFTZFpPeUpTTk9ZcHImZT0xNTY3Mzc1OTQzJnQ9MTU2NzE1OTk0MyZyPTg1MDc4MzY5JmY9LzlmYzA2ZTgwLWNiMGUtMTFlOS04N2ZmLTQ5MmIzZGE5YWI5Yi5wbmcmYj1jb2RpbmctbmV0LXByb2R1Y3Rpb24tZmlsZQ==)
	</center>


2. s.resources
	
	```
	s.resources = ['Classes/**/*.{xib,storyboard,Bundle,png,gif,jpg,jpeg,txt}', 'Resource/**/*']
	``` 
	
	图片等资源是都放入 mainbundle，直接用 imageNamed: 访问，不用增加很多获取 bundle 的代码。
	
	```
	s.resource = 'xx/xxx.bundle'

	s.resource_bundles = { 'xxx' => ['/Classes/**/*.{xib,storyboard,Bundle,png,gif,jpg,jpeg,txt}', 'Resource/**/*'] }
	```

	这两种写法，资源都在模块自己的 bundle 里面，文件名为 xxx.bundle，工程中需要通过 ``bundleForClass`` 等获取资源路径。
	
#### 1.2 包的大小

<center>
![](https://coding-net-production-file-ci.codehub.cn/c3a89a40-cb11-11e9-9abe-7341ed163cae.png?sign=H6IebxDROcDaD0PGAQYPUWb2RophPTEyNTcyNDI1OTkmaz1BS0lEYXk4M2xGbWFTNlk0TFRkek1WTzFTZFpPeUpTTk9ZcHImZT0xNTY3Mzc3Mjg5JnQ9MTU2NzE2MTI4OSZyPTYxNjMxOTMzJmY9L2MzYTg5YTQwLWNiMTEtMTFlOS05YWJlLTczNDFlZDE2M2NhZS5wbmcmYj1jb2RpbmctbmV0LXByb2R1Y3Rpb24tZmlsZQ==)
</center>

在图中，上面的是使用 use\_frameworks! 的动态包， 下面的是默认（或使用 use\_modular\_headers!）的静态包，几次验证，都是<font color=#cc0000>动态的更小</font>。

#### 1.3 编译速度

这个未验证。

#### 1.4 工程实例

在项目开发中的场景是一个第三方类库 bongSDK.framework 引入了 Realm.framework 和 RealmSwift.framework，RealmSwift.framework 是通过 swift 语言写的，它的内部调用 Realm。

最初静态方式的 pod 遇到了难以理解的报错，因为知识的欠缺和时间的紧迫，放弃了静态这条路，使用 use\_frameworks! 动态 pod 的方式。

动态方式在 pod install 阶段没有报错，但子模块需要添加依赖，更困难的是图片、xib、storyboard 等资源需要获取到模块的 bundle 才能加载，导致工程大面积的图片加载错误，页面跳转崩溃。因此不得不增加很多获取 bundle 路径的代码，修改的位置几百上千处。

```oc
+ (NSBundle *)bundleWithClassName:(Class)cls moduleName:(NSString*)module
{
    if (module == nil) {
        return [NSBundle mainBundle];
    }
    
    NSBundle * bundle = [NSBundle bundleForClass:cls];
    NSURL * bundleURL = [bundle URLForResource:module withExtension:@"bundle"];
    
    if (bundleURL == nil) {

        __block UINavigationController* nav;
        [[UIApplication sharedApplication].windows enumerateObjectsUsingBlock:^(__kindof UIWindow * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
            UIViewController * windowVC = obj.rootViewController;
            if ([windowVC isKindOfClass:[UINavigationController class]]) {
                nav = (UINavigationController *)windowVC;
                *stop = YES;
            }
        }];

        if (nav != nil) {
            Class callerCls = [nav.viewControllers.firstObject class];
            bundle = [NSBundle bundleForClass:callerCls];
            bundleURL = [bundle URLForResource:module withExtension:@"bundle"];
        }

        if (bundleURL == nil) {
            return [NSBundle mainBundle];
        }
    }
    return [NSBundle bundleWithURL:bundleURL];
}
```

图片加载则更加困难，因为很多图片是在 xib 中写的，通过断点发现，系统并没有调用 ``imageNamed:`` 方法，导致使用 runtime 替换方法实现图片位置修改的方式失败，通过查找资料，发现 xib 中的 UIButton、UIImageView 会调用 ``- initWithCoder:`` 方法，底层会调用 UINibDecoder 类的 ``decodeObjectForKey``。

runtime 替换 decodeObjectForKey 方法后，打印输出发现，UIButton、UIImageView 控件加载的图片名称在 <font color=#cc0000>``UIResourceName``</font> 字段。由此就有了如下的处理方式：

```oc
+ (void)load
{
    _imageViewImageArray = [NSMutableArray arrayWithCapacity:2];

    propKey = [CYKJXUtil stringByReversed:@"emaNecruoseRIU"];
    btnKey = [CYKJXUtil stringByReversed:@"tnetnoClufetatSnottuBIU"];

    // hook UINibDecoder      - decodeObjectForKey:
    NSString* clsName = [NSString stringWithFormat:@"redoce%@biNIU", @"D"];
    clsName = [CYKJXUtil stringByReversed:clsName];

    [HookTool exchangeInstanceMethod:NSClassFromString(clsName)
                         originalSEL:@selector(decodeObjectForKey:)
                         swizzledSEL:@selector(swizzle_decodeObjectForKey:)];

    // hook UIImageView        - initWithCoder:
    [HookTool exchangeInstanceMethod:UIImageView.class
                         originalSEL:@selector(initWithCoder:)
                         swizzledSEL:@selector(swizzle_imageView_initWithCoder:)];

    // hook UIButton        - initWithCoder:
    [HookTool exchangeInstanceMethod:UIButton.class
                         originalSEL:@selector(initWithCoder:)
                         swizzledSEL:@selector(swizzle_button_initWithCoder:)];
}

- (id)swizzle_decodeObjectForKey:(NSString *)key
{
    Method originalMethod = class_getInstanceMethod([HookTool class], @selector(swizzle_decodeObjectForKey:));
    IMP function = method_getImplementation(originalMethod);
    id (*functionPoint)(id, SEL, id) = (id (*)(id, SEL, id)) function;
    id value = functionPoint(self, _cmd, key);
    
    // 保存图片名称
    if ([key isEqualToString:propKey]) {
        [_imageViewImageArray addObject:value];
    }
    
    // 保存 button 状态数据
    if ([key isEqualToString:btnKey]) {
        if ([value isKindOfClass:[NSDictionary class]]) {
            _buttonImageDictionary = value;
        }
    }
    
    return value;
}


#pragma mark - UIImageView

- (id)swizzle_imageView_initWithCoder:(NSCoder *)aDecoder
{
    // 执行顺序：initWithCoder -》DecoderWithKey -》setImage：，所以每次给 imageView 设置图片时，需要将之前的置空。
    // tabbarItem 的图片设置不会执行 initWithCoder，如果不置空，会导致 imageView 设置成和 tabbarItem 一样的图片。
    [_imageViewImageArray removeAllObjects];

    UIImageView * instance = (UIImageView *)[self swizzle_imageView_initWithCoder:aDecoder];

    // 设置 image
    if (_imageViewImageArray.count > 0) {
        UIImage * normalImage = [HookTool imageAfterSearch:_imageViewImageArray[0]];
        if (normalImage) {
            instance.image = normalImage;
        }
    }
    // 设置 highlightedImage
    if (_imageViewImageArray.count > 1) {
        UIImage * highlightedImage = [HookTool imageAfterSearch:_imageViewImageArray[1]];
        if (highlightedImage) {
            instance.highlightedImage = highlightedImage;
        }
    }
    
    return instance;
}


#pragma mark - UIButton

- (id)swizzle_button_initWithCoder:(NSCoder *)aDecoder
{
    // 执行顺序：initWithCoder -》DecoderWithKey -》setImage：，所以每次给 button 设置图片时，需要将之前的置空。
    // tabbarItem 的图片设置不会执行 initWithCoder，如果不置空，会导致 button 设置成和 tabbarItem 一样的图片。
    [_imageViewImageArray removeAllObjects];
    _buttonImageDictionary = nil;
    
    UIButton * instance = (UIButton *)[self swizzle_button_initWithCoder:aDecoder];
    
    @autoreleasepool {
        [_buttonImageDictionary enumerateKeysAndObjectsUsingBlock:^(id _Nonnull key,
                                                                    id  _Nonnull obj,
                                                                    BOOL * _Nonnull stop) {
            
            if (_imageViewImageArray.count == 0) {
                *stop = YES;
            }
            else {
                switch ([key integerValue]) {
                    case ButtonImageOrder_Normal:
                        [HookTool setImageForButton:instance object:obj state:UIControlStateNormal];
                        break;
                    case ButtonImageOrder_Highlighted:
                        [HookTool setImageForButton:instance object:obj state:UIControlStateHighlighted];
                        break;
                    case ButtonImageOrder_Selected:
                        [HookTool setImageForButton:instance object:obj state:UIControlStateSelected];
                        break;
                    case ButtonImageOrder_Disabled:
                        [HookTool setImageForButton:instance object:obj state:UIControlStateDisabled];
                        break;
                }
            }
        }];
    }

    return instance;
}
```

更详细的代码：[ImageTool](https://github.com/BiYJ/ImageTool)

如上可见，这种动态方式对于编码并不友好，资源必须要特定的 bundle，一旦资源路径出错，轻则图片未加载，重则程序崩溃。

所以需要研究下如果使用静态方式 pod 子模块代码。

操作步骤：

1. 修改各个子模块的 podspec 文件，删除依赖，修改资源的路径

	```oc
	#    s.resource_bundles = { 'xx' => ['Classes/**/*.{xib,storyboard,Bundle,png,gif,jpg,jpeg,txt,js}', 'Resource/**/*'] }
	     s.resources = ['Classes/**/*.{xib,storyboard,Bundle,png,gif,jpg,jpeg,txt,js}', 'Resource/**/*']

	#    s.dependency 'Realm', '3.11.2'
	#    s.dependency 'RealmSwift', '3.11.2'

	     ## 依赖私有库
	#    s.dependency 'xxx'
	```

2. 删除 use_frameworks!

	```oc
	#use_frameworks!
	```

	重新执行 pod install，等 Pod installation complete! 之后，运行工程。报错，一个一个的解决。

> \#import \<AlipaySDK/AlipaySDK.h\>

修改为 \#import "AlipaySDK.h"

> dyld: Library not loaded: @rpath/Realm.framework/Realm

现在不用 pod 导入 realm，而是将 realm.framework 拖入 basicModule 工程。这里找了[官方最新的](https://static.realm.io/downloads/objc/realm-objc-3.13.1.zip?_ga=2.256310156.625208557.1551076206-43540051.1551076206) realm.framework，它分为静态版和动态版，添加到工程的 ``Embedded Binaries``，编译时报错 ``Unknown type name namespace``。
	
不管通过修改 .h 为 .hpp，还是修改 build settings -> Compile Sources As -> Objectoive-C++ 都没有效果，无计可施之时想到了，可以将 use\_frameworks! 时 cocoapods 生成 的 Realm.framework 拷贝一份，拖入工程死马等活马医。

编译运行，这个问题解决了~

> dyld: Library not loaded: @rpath/libswiftCore.dylib
  Referenced from: /Users/.../CYKJMain.app/Frameworks/RealmSwift.framework/RealmSwift
  Reason: image not found
  
在主工程中创建 xx.swift，并自动创建桥接文件。

<center>
![](https://coding-net-production-file-ci.codehub.cn/3a187b30-cb3c-11e9-9601-1f29a8550414.png?sign=PIV6MKaupFKQfCsBa6GO7abI1ONhPTEyNTcyNDI1OTkmaz1BS0lEYXk4M2xGbWFTNlk0TFRkek1WTzFTZFpPeUpTTk9ZcHImZT0xNTY3Mzk1NTI0JnQ9MTU2NzE3OTUyNCZyPTgwNzc5NTAmZj0vM2ExODdiMzAtY2IzYy0xMWU5LTk2MDEtMWYyOWE4NTUwNDE0LnBuZyZiPWNvZGluZy1uZXQtcHJvZHVjdGlvbi1maWxl)
</center>

> Check dependencies
> Argument list too long: recursive header expansion failed

Search Paths -> Header Search Paths，去掉 ``$(PODS_ROOT)/**``，去掉不必要的 ``recursive`` search。原因是：参数列表太长，在递归展开的时候失败，当工程根目录的层级比较深时，Pods里面的层级也比较多时，导致路径太长，超出范围。

> duplicate symbol xx in

工程中出现重复的类或者 storyboard 文件等，删除或修改名称即可。

<center>
![](https://coding-net-production-file-ci.codehub.cn/5df4d200-cb3d-11e9-a629-cfad1d180165.png?sign=gDRLjswgEokgwfTYCbRPGKB9ASxhPTEyNTcyNDI1OTkmaz1BS0lEYXk4M2xGbWFTNlk0TFRkek1WTzFTZFpPeUpTTk9ZcHImZT0xNTY3Mzk2MDE0JnQ9MTU2NzE4MDAxNCZyPTE5NjczMzM4JmY9LzVkZjRkMjAwLWNiM2QtMTFlOS1hNjI5LWNmYWQxZDE4MDE2NS5wbmcmYj1jb2RpbmctbmV0LXByb2R1Y3Rpb24tZmlsZQ==)
</center>

	
## 二、文章

[关于Argument list too long的问题](https://juejin.im/post/5bea871ef265da612e282f54)

