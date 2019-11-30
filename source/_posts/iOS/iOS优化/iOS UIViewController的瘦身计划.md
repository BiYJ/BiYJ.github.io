---
title: iOS UIViewController的瘦身计划
categories: iOS架构
---

1. 代码的组织结构，以及为何要这样写。
2. 那些场景适合使用子控制器，那些场景应该避免使用子控制器？
3. 分离UITableView的数据源和UITableViewDataSource协议。
4. MVVM的重点是ViewModel，不是响应函数式。
5. MVVM中，ReactiveCocoa或RXSwift实现数据绑定的带来的弊端。
6. 用策略模式替代if-else或switch这样判断比较多，不利于代码阅读的分支结构。并在特定场景下，用策略模式解决模块调用问题。
7. 为什么要较少模块间跨层数据交流。

##  一、代码结构

在说控制器瘦身之前，首先要做的的是保证代码结构的清晰化。良好的代码结构有利于代码的传承、可读性以及可维护性。通常笔者都是这样控制代码结构的：

```
#pragma mark - life cycle 

#pragma mark - Notification 

#pragma mark - UI

#pragma mark - Touch 

#pragma mark - UITableViewDelegate/DataSource

#pragma mark - HTTP

#pragma mark - SET & GET
```

1. 不要在除了 getter 之外的结构中设置 view 基本坐标、属性等。
2. 在viewDidAppear里面做Notification的监听之类的事情。
3. 每一个代理方法都对应上相应的协议，否则后期随着代码量的增加，很难找出某一代理方法对应的协议，不利于代码的可读性。
4. getter 和setter 方法写在代码的最后面。
5. getter方法中不要添加比较重要重要的业务逻辑，重要的业务逻辑应该单独拿出来，放在对应的pragra mark 下，否则对于代码的阅读者来说，比较难以定位逻辑的入口位置。实际开发中遇到过多次这样的情况，焦头烂额的寻找关键逻辑入口处，纵里寻她千百度，结果它却躺在 getter方法中。
6. UI布局可以说比较重要，也可以说不重要。重要是因为一个新手接手新项目，如果对布局还没有了解，业务逻辑便无从谈起；UI布局不重要是因为只要相关控件封装的足够好，页面UI布局通常会很简单；因为UI布局比较重要，所以笔者将它放在固定位置(setter&getter上面)，因为UI布局通常比较简单，所以将其放在代码中比较靠后的位置。


## 二、关于子控制器

对于相对比较复杂的界面，通常情况下还可以考虑添加子控制的实现方式。如实际开发中，在商品搜索模块中，将历史搜索标签和推荐搜索标签、搜索推荐词条以及搜索结果用三个控制器分别承载不同的逻辑，是不同的代码逻辑分离。

1. 优点

	把和该元素相关的业务逻辑分解一部分到子控制器中，主业务逻辑对应的代码量瞬间减少很多，代码封装和分离十分清晰。

2. 缺点

	这种做法最大的缺点就是父控制器和子控制器之间的消息传递有时需要做额外的处理，尤其是子控制器的消息回调。

所以，建议根据实际情况有选择的考虑，如果父控制器和子控制器之间的消息交互较少，完全可以考虑此种方式。如果父控制器和子控制器之间的消息交互较多，建议仔细考虑清楚再做取舍。

实际开发中，苹果专门提供了一个 UITableViewController 类，专门为 tableView 服务，但是实际开发中很少有人直接使用。该控制相对于普通的 UIViewController 的而言，直接实现了下拉刷新功能；除此之外，还能切换编辑模式、响应键盘通知。如果 UITableViewController 实现的标准刚好同你项目中的 tableView 一些需求很类似，就可以直接通过使用子控制器的方式，避免了写那些重复的代码。当然，实际开发中出现这种事情的概率非常小。


## 三、UITableView 的瘦身

绝大多数情况下，只要有控制器就会存在UITableView或UICollectionView（这里仅仅以tableView为例），所以对UITableView 的瘦身尤为重要。以下的分析主要参照该[文章](https://www.objc.io/issues/1-view-controllers/lighter-view-controllers/)。

#### 3.1 拆分出不重要的东西

毫无疑问，在 Controller 层中协调 View 和 Model 的工作是无法拆除的。那么除此之外，不是必须有 Controller 层承载的内容便可以被拆除，比如 tableView 的数据源和 UITableViewDataSource 协议。下面分两种情况说明，一种是将数据源和 UITableViewDataSource 协议都拆分出来，另一种是只拆分数据源。

1. 单一的 cell 和数据源

	关于这种情况，[文章](https://www.objc.io/issues/1-view-controllers/lighter-view-controllers/)中实现代码如下这样。

	控制器中代码
	
	```
	TableViewCellConfigureBlock configureCell = ^(PhotoCell *cell, Photo *photo) {
		[cell configureForPhoto:photo];
	};
    NSArray * photos = [AppDelegate sharedDelegate].store.sortedPhotos;
    self.photosArrayDataSource = [[ArrayDataSource alloc] initWithItems:photos cellIdentifier:PhotoCellIdentifier configureCellBlock:configureCell];
    self.tableView.dataSource = self.photosArrayDataSource;
    [self.tableView registerNib:[PhotoCell nib] forCellReuseIdentifier:PhotoCellIdentifier];
    ```
    
    抽离的数据源代码
    
    ```
	// .h文件
	typedef void (^TableViewCellConfigureBlock)(id cell, id item);
	@interface ArrayDataSource : NSObject <UITableViewDataSource>
	- (id)initWithItems:(NSArray *)anItems
	     cellIdentifier:(NSString *)aCellIdentifier
	 configureCellBlock:(TableViewCellConfigureBlock)aConfigureCellBlock;
	- (id)itemAtIndexPath:(NSIndexPath *)indexPath;
	@end
	
	//.m文件
	#import "ArrayDataSource.h"
	@interface ArrayDataSource ()
	@property (nonatomic, strong) NSArray *items;
	@property (nonatomic, copy) NSString *cellIdentifier;
	@property (nonatomic, copy) TableViewCellConfigureBlock configureCellBlock;
	@end
	
	@implementation ArrayDataSource
	
	- (id)init
	{
	    return nil;
	}
	
	- (id)initWithItems:(NSArray *)anItems
	     cellIdentifier:(NSString *)aCellIdentifier
	 configureCellBlock:(TableViewCellConfigureBlock)aConfigureCellBlock
	{
	    self = [super init];
	    if (self) {
	        self.items = anItems;
	        self.cellIdentifier = aCellIdentifier;
	        self.configureCellBlock = [aConfigureCellBlock copy];
	    }
	    return self;
	}
	
	- (id)itemAtIndexPath:(NSIndexPath *)indexPath
	{
	    return self.items[(NSUInteger) indexPath.row];
	}
	
	#pragma mark - UITableViewDataSource
	- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section
	{
	    return self.items.count;
	}
	
	- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
	{
	    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:self.cellIdentifier forIndexPath:indexPath];
	    id item = [self itemAtIndexPath:indexPath];
	    self.configureCellBlock(cell, item);
	    return cell;
	}
	@end
	```

	上述代码将 tableView 的数据源以及 UITableViewDataSource 协议都抽离到 ArrayDataSource中，而UITableViewDelegate依然保留在Controller层中。至于数据源的来源(实际中往往是从网络获取)，这里主要是通过Store类获取数据,具体体现代码NSArray *photos = [AppDelegate sharedDelegate].store.sortedPhotos;，该行代码你可以理解为实际项目中的网络请求的伪代码。

2. 多种数据源和多种cell(仅拆分数据源和Protocols)

	同时拆分数据源和UITableViewDataSource协议这种方式有一定的局限性，如果一个tableView中有多类型cell,下面的这个方法将很难设计，尤其是针对参数aCellIdentifie和aConfigureCellBlock。所以针对这种情况仅仅将tableView 的dataSource拆出来即可，实际这种拆分情况就是MVVM模式中的ViewModel。

	```
	- (id)initWithItems:(NSArray *)anItems
	     cellIdentifier:(NSString *)aCellIdentifier
	 configureCellBlock:(TableViewCellConfigureBlock)aConfigureCellBlock;
	```
	
3. 网络层放在那里？

	按照该文章的意思，网络层可以在封装之后放在Cotroller中，这种方案当然是可以。除此之外，按照MVVM的设计模式，网络层同样可以放在ArrayDataSource类中，该类对外提供网络请求接口，数据返回后，同步更新内部的数据源即可。

#### 3.2 cell 内部控制 cell 状态

通常控制 cell 的状态，我们可以实现如下两个代理方法。

```
- (void)tableView:(UITableView *)tableView
        didHighlightRowAtIndexPath:(NSIndexPath *)indexPath
{
    PhotoCell *cell = [tableView cellForRowAtIndexPath:indexPath];
    cell.photoTitleLabel.shadowColor = [UIColor darkGrayColor];
    cell.photoTitleLabel.shadowOffset = CGSizeMake(3, 3);
}

- (void)tableView:(UITableView *)tableView
        didUnhighlightRowAtIndexPath:(NSIndexPath *)indexPath
{
    PhotoCell *cell = [tableView cellForRowAtIndexPath:indexPath];
    cell.photoTitleLabel.shadowColor = nil;
}
```

按照这种方式会存在两个缺点:

1、这两个代理方法增加了Controller的代码量。
2、在Controller中显示通过tableView获取cell，再调用cell实现的细节方法。思路相对比较绕。
综上所述，文章中是在cell中控制cell的状态。代码如下。
- (void)setHighlighted:(BOOL)highlighted animated:(BOOL)animated{
    [super setHighlighted:highlighted animated:animated];
    if (highlighted) {
        self.photoTitleLabel.shadowColor = [UIColor darkGrayColor];
        self.photoTitleLabel.shadowOffset = CGSizeMake(3, 3);
    } else {
        self.photoTitleLabel.shadowColor = nil;
    }
}


#### 3.3 cell 初始化和更新分离

另外一种好的做法就是将 cell 的根据 model 更新的方法，拆分到分类中完成。实际开发中，可能存在复杂的cell代码量很大，此时可以借助分类的方法分离关注点。

#### 3.4 简单谈谈 MVVM

1. MVVM本质上也是从MVC中派生出来的思想，由M、V、VM、V四部分组成，主要是为了减少MVC中Controller承担的负荷。
2. 借助ViewModel可以降低View和Model的耦合度。
3. 虽然ViewModel是MVVM组成的一部分，但是MVC中依然能用，上述分离tableView的dataSource就是很好的说明。MVVM的关键是要有ViewModel。而不是ReactiveCocoa、RXSwift或RXJava等。
4. ReactiveCocoa或RXSwift只是能更好的体现能更好地体现MVVM的精髓。使用函数响应式框架能更好的实现数据和视图的双向绑定(ViewModel的数据可以显示到View上，View上的操作同样会引起ViewModel的变化),降低了ViewModel和View的耦合度。
5. ReactiveCocoa或RXSwift不应该因为他本身难以被理解而被神化。通过这两个框架可以实现ViewModel和View的双向绑定，但同样会存在几个比较重大的问题。 首先,ReactiveCocoa或RXSwift的学习成本很高；其次，数据绑定使得 Bug 很难被调试，当界面出现异常，可能是View的问题，也可能是数据ViewModel的问题。而数据绑定会使一个位置的bug传递到其他位置，难以定位；最后，数据绑定是需要消耗更多的内存，对于大型项目更是如此。只是结合自己所学知识谈谈理解，如果对RXSwift感兴趣，推荐这个链接。


## 四、合理拆分模块

#### 4.1 模块拆分大小要合理

如果模块被拆分的太粗糙，基本就是简单的封装，并没有进一步细化，只是将所有的功能集中在一起，这样做似乎没有太大意义

如果模块被拆分的很细，Controller中很执行相关模块的功能就要调用相关模块代码，似乎代码量并不会减少太多。比如在做即时通信应用开发时，支持的消息类型有文字、语音、图片、视频消息。其中后三种消息类型同文字消息不同，后三者要求发送消息的时候，首先要像后台请求上传资源的权限，获取上传资源权限后，返回对应的字段(该字段以实际情况不同，可能是id，也可能是token之类的），上传成功后获取资源对应的URL，再把资源的URL通过类似文字消息的发送方式发送出去。此时，可以拆分成三个模块数据发送（叫A模块）、上传资源申请（叫B模块）、内容上传（叫C模块）。如果要发送文字消息，直接在Controller中调用模块A即可；但是如果想发送其他消息，就要依次调用模块B、模块C、模块A，按照这种调用方式，Controller必然会膨胀。

#### 4.2 策略模式

在说合理拆分模块之前，先简单说下策略模式，因为接下来举的例子中涉及策略模式。

策略模式一般是指：

1. 可以实现目标的方案集合；
2. 根据形势发展而制定的行动方针和斗争方法；
3. 有斗争艺术，能注意方式方法。

switch，if-else 之类的分支语句,此类语句给人的直观感觉是判断条件明确，代码层次清晰；缺点可能是代码繁琐，杂乱无章，而且拆分困难。特别是到后期维护代码的时候，这种状况往往令人有食之无味，弃之可惜的感觉。使用策略模式可以代替 switch 或 if-else 之类的代码。

举个例子，以下是小明的计划安排：

>    周一打篮球
>    周二逛街
>    周三洗衣服
>    周四打游戏
>    周五唱歌
>    其他休息

借助策略模式我们可以这样实现代码：

```
@interface XiaoMing : NSObject
- (void)doSomethingWithDayStr:(NSString *)dayStr params:(NSDictionary *)paramsDict;
@end


#import "XiaoMing.h"

@interface XiaoMing()
@property(nonatomic,copy)NSDictionary *strategyDict;//策略
@property(nonatomic,copy)NSDictionary *paramDict;//参数
@end


@implementation XiaoMing

- (void)doSomethingWithDayStr:(NSString *)dayStr params:(NSDictionary *)paramsDict
{
    self.paramDict = paramsDict;
    if (self.strategyDict[dayStr]){
        NSInvocation * doWhat = self.strategyDict[dayStr];
        [doWhat invoke];
    }
    else {
        [self sleep];
    }
}

- (NSInvocation *)invocationWithMethod:(SEL)selector
{
    NSMethodSignature * signature = [[self class] instanceMethodSignatureForSelector:selector];
    
    if (signature == nil) {
        NSString *reason = [NSString stringWithFormat:@"提示：The method[%@] is not find", NSStringFromSelector(selector)];
        @throw [NSException exceptionWithName:@"错误" reason:reason userInfo:nil];
    }
    NSInvocation*invocation = [NSInvocation invocationWithMethodSignature:signature];
    invocation.target = self;
    invocation.selector = selector;
    NSDictionary *param = self.paramDict;
    //index表示第几个参数,注意0和1已经被占用了（self和_cmd），所以我们传递参数的时候要从2开始。
    [invocation setArgument:&(param) atIndex:2];
    
    return invocation;
}

- (void)playBasketball:(NSDictionary *)dict{
    NSLog(@"方法:%s 参数:%@",__FUNCTION__,dict);
}

- (void)shopping:(NSDictionary *)dict{
    NSLog(@"方法:%s 参数:%@",__FUNCTION__,dict);
}

- (void)washClothes:(NSDictionary *)dict{
    NSLog(@"方法:%s 参数:%@",__FUNCTION__,dict);
}

- (void)playGames:(NSDictionary *)dict{
    NSLog(@"方法:%s 参数:%@",__FUNCTION__,dict);
}

- (void)sing:(NSDictionary *)dict{
     NSLog(@"方法:%s 参数:%@",__FUNCTION__,dict);
}
- (void)sleep{
     NSLog(@"这是其他情况：%s",__FUNCTION__);
}

- (NSDictionary *)strategyDict
{
    if (_strategyDict == nil) {
        _strategyDict = @{
                          @"day1" : [self invocationWithMethod:@selector(playBasketball:)],
                          @"day2" : [self invocationWithMethod:@selector(shopping:)],
                          @"day3" : [self invocationWithMethod:@selector(washClothes:)],
                          @"day4" : [self invocationWithMethod:@selector(playGames:)],
                          @"day5" : [self invocationWithMethod:@selector(sing:)]
             };
    }
    return _strategyDict;
}
@end
```

外部调用可以完全不再使用 if-else 的判断了。

```
XiaoMing * xm = [[XiaoMing alloc]init];

//各种情况直接赋值给dayStr即可。
NSString * dayStr = @"day3s";
[xm doSomethingWithDayStr:dayStr params:@{@"key":@"test"}];
```

#### 4.3 合理应用策略模式和组合方式解决上述4.2问题

关于上述问题我们可以通过组合和策略模式解决。首先创建一个 MessageManager 类。对外提供的接口大概是这样的：

```
typedef NS_ENUM (NSUInteger, MessageSendStrategy){
    MessageSendStrategyText = 0,
    MessageSendStrategyImage = 1,
    MessageSendStrategyVoice = 2,
    MessageSendStrategyVideo = 3
}

@protocol MessageManagerDelegate<NSObject>
 @required
- (void)messageSender:(MessageSender *)messageSender
didSuccessSendMessage:(BaseMessage *)message
             strategy:(MessageSendStrategy)strategy;

- (void)messageSender:(MessageSender *)messageSender
   didFailSendMessage:(BaseMessage *)message
             strategy:(MessageSendStrategy)strategy
                error:(NSError *)error;
@end


@interface MessageManager : NSObject
@property (nonatomic, weak) id<MessageSenderDelegate> delegate;
@property(nonatomic,copy)NSDictionary *strategyDict;//主要在这里定义策略，内部通过Invoke唤起对应方法。
- (void)sendMessage:(BaseMessage *)message withStrategy:(MessageSendStrategy)strategy;
@end
```

外部调用形式大概是这样的，除此之外还要遵守 MessageManagerDelegate 协议并实现协议方法。

```
[self.messageManager sendMessage:message withStrategy:MessageSendStrategyText];
```

MessageManager.m 文件实现大概是这样的：

```
@interface MessageManager()

@end

@implementation MessageManager
- (void)sendMessage:(BaseMessage *)message withStrategy:(MessageSendStrategy)strategy
{
      .....
     if (self.strategyDict[@(strategy)]){
        NSInvocation *doWhat = self.strategyDict[@(strategy)];
        [doWhat invoke];
    }
    ......
    ......
}

......
......

@end
```

总的来说，基本形式和上面举出的例子类似。

#### 4.4 减少跨层数据交流

前面注意事项。那么接下来就是要使用模块了，使用模块的时候同样有需要注意的地方：减少跨层数据交流。举个例子，假如有模块一、模块二、模块三，按照正常的调用方式是外部使用模块一调用模块二的方法，模块二的方法再调用模块三的方法。但是随着模块功能的完善，突然有一天出现模块一直接调用模块三的情况，那么后续就很难避免其他开发人员可能直接拿模块一调用模块三方法。类似这种跨层的数据交流很不利于项目的后续维护。

## 五、其它

除此之外，控件的合理拼装也能在很大程度上减少控制器中的代码。另外还有一个要注意的地方，就是UIViewController继承的问题。关于这个问题，可以在笔者之前写的这篇文章的第二部分内容找到答案。

## 六、补充一

在讲 UIResponder 之前可以先温习下响应链。脑海中想象一下这样的视图层级结构:一个 UIViewController 上放置一个 tableView(supTableView) ， supTableView 的 cell 上放置一个 tableView(subTableView) , subTableView 的 cell 上有一个 UIButton 。即：

UIViewController -> SuperTable -> SuperCell -> SubTable -> SubCell -> UIButton
如果想在点击UIButton的时候在UIViewController中产生回调，一般可以借助delegate或block实现，但是由于层级太深，这样做的话会很繁琐。明智的方式是借助UIResponder。

只需要一个 UIResponder 的 category 就行

```
@interface UIResponder (Router)
- (void)routerEventWithSelectorName:(NSString *)selectorName object:(id)object userInfo:(NSDictionary *)userInfo;
@end


@implementation UIResponder (Router)
- (void)routerEventWithSelectorName:(NSString *)selectorName object:(id)object userInfo:(NSDictionary *)userInfo
{
    [[self nextResponder] routerEventWithSelectorName:selectorName object:object userInfo:userInfo];
}
@end
```

UIButton点击事件

```
- (IBAction)btnClick:(UIButton *)sender
{
    [self routerEventWithSelectorName:@"btnClick:userInfo:" object:sender userInfo:@{@"key":@"value"}];
}
```

## 七、补充二

在 Cell 内部获取父控制器，在 Cell 内部调用控制器的一些耦合性比较小的代码，一定程度上也能达到瘦身的目的。如在 Cell 中有个返回按钮，需要当前父视图控制器返回 Push 到它之前的控制器，那么就需要在自定义 Cell 中拿到当前的父视图控制器做 Pop 操作。

```
- (UIViewController *)viewController
{
    for (UIView* next = [self superview]; next; next = next.superview) {
        UIResponder *nextResponder = [next nextResponder];
        if ([nextResponder isKindOfClass:[UIViewController class]]) {
            return (UIViewController *)nextResponder;
        }
    }
    return nil;
}
```

## 八、小结

在说 UIViewController的瘦身计划之前，第一部分先说了合理的代码结构；第二部分单提了下关于子控制器，并简单的用UITableViewController举了个例子；第三部分重点介绍了UITableView的瘦身，并因此引申出了MVVM的一些内容；第四部分主要介绍了一些模块拆分中遇到的一个问题和解决方案，除此还说明了模块跨层数据交流的问题；最后，提了下控件的拼装和UIViewController继承的问题。


## 文章

[ZhengYaWei](https://www.jianshu.com/u/4a57fb76690f) - [UIViewController的瘦身计划（iOS架构思想篇）](https://www.jianshu.com/p/98fa80eebc52)