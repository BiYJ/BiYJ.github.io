---
title : 静态、动态 pod
categories : iOS原理
---

## 一、静态和动态

在项目中使用 pod 实现模块化，对于子模块和第三类库的导入方式存在两种：静态库、动态库。

当在 podfile 中指定 use\_frameworks! 时，子模块和第三方类库将被打包成 ``.framework`` 动态库，模块之间的代码不能直接引用，需要添加依赖；

<center>
![](http://dzliving.com/DynamicModule2.png)
</center>

反之（默认情况）将打包成 ``.a`` 静态库。

<center>
![](http://dzliving.com/StaticModule3.png)
</center>

动态库和静态库的区别：

- 资源加载方式
- 包的大小 
- 编译速度

#### 1.1 资源加载方式

1. s.dependency 'xx’

	静态方式中各模块的 podspec 文件不用设置依赖，就可以直接 #import 其他模块的类头文件。
	
	<center>
￼	![](http://dzliving.com/StaticModule1.png)
	</center>
	
	而动态方式则会报错。

	<center>
	![](http://dzliving.com/DynamicModule1.png)
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
![](http://dzliving.com/StaticModule2.png)
</center>

在图中，上面的是使用 use\_frameworks! 的动态包， 下面的是默认（或使用 use\_modular\_headers!）的静态包，几次验证，都是<font color=#cc0000>动态的更小</font>。

#### 1.3 编译速度

动态库 Pod 方式：

```
Total pre-main time: 819.46 milliseconds (100.0%)
         dylib loading time: 542.43 milliseconds (66.1%)
        rebase/binding time:  35.86 milliseconds (4.3%)
            ObjC setup time:  57.79 milliseconds (7.0%)
           initializer time: 183.27 milliseconds (22.3%)
           slowest intializers :
             libSystem.B.dylib :  10.33 milliseconds (1.2%)
    libMainThreadChecker.dylib :  28.76 milliseconds (3.5%)
                  AFNetworking :  83.46 milliseconds (10.1%)
                         Realm :  27.37 milliseconds (3.3%)
               CYKJBasicModule :  17.54 milliseconds (2.1%)
```

静态库 Pod 方式：

```
Total pre-main time: 591.00 milliseconds (100.0%)
         dylib loading time: 447.06 milliseconds (75.6%)
        rebase/binding time:  30.51 milliseconds (5.1%)
            ObjC setup time:  20.26 milliseconds (3.4%)
           initializer time:  93.04 milliseconds (15.7%)
           slowest intializers :
             libSystem.B.dylib :   7.67 milliseconds (1.2%)
    libMainThreadChecker.dylib :  25.41 milliseconds (4.3%)
                         Realm :  21.13 milliseconds (3.5%)
                      CYKJMain :  57.87 milliseconds (9.7%)
```


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

如上可见，这种动态方式对于编码并不友好，资源必须要特定的 bundle，一旦资源路径出错，轻则图片未加载，重则程序崩溃。

所以需要研究下如果使用静态方式 pod 子模块代码。

首先将 use\_frameworks! 删除，重新执行 pod install，等 Pod installation complete! 之后，运行工程，报错，一个一个的解决。

1. dyld: Library not loaded: @rpath/Realm.framework/Realm

	现在不用 pod 导入 realm，而是将 realm.framework 拖入 basicModule 工程。这里找了[官方最新的](https://static.realm.io/downloads/objc/realm-objc-3.13.1.zip?_ga=2.256310156.625208557.1551076206-43540051.1551076206) realm.framework，它分为静态版和动态版，添加到工程的 ``Embedded Binaries``，编译时报错 ``Unknown type name namespace``。
	
	不管通过修改 .h 为 .hpp，还是修改 build settings -> Compile Sources As -> Objectoive-C++ 都没有效果，无计可施之时想到了，可以将 use\_frameworks! 时 cocoapods 生成 的 Realm.framework 拷贝一份，拖入工程死马等活马医。
	
	编译运行，这个问题解决了~

2. Argument list too long: recursive header expansion failed

	Search Paths -> Header Search Paths，去掉 ``$(PODS_ROOT)/**``，去掉不必要的 ``recursive`` search。
	

其余的就是解决一些资源加载问题，资源重名问题，动态库的引用问题 #import “” 改为 #import <>。

<center>
![](http://dzliving.com/DuplicateFile.png)
</center>	


## 二、文章

[关于Argument list too long的问题](https://juejin.im/post/5bea871ef265da612e282f54)

