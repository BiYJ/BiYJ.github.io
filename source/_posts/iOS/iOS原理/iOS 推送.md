---
title: iOS 推送
categories: iOS原理
---


## 一、推送原理

当用户打开应用程序的通知中心之后，苹果远程推送服务器就能把消息推送到装有该应用的设备上，具有强制性、实时性的特点，并且用户无需打开应用都能收到推送的消息。

#### 1.1 名词介绍

* Provider：消息提供者，一般是我们的后台服务器或者第三方推送服务器后台
* APNs(Apple Push Notification service)：苹果推送通知服务。
* APNs Server(Apple Push Notification service Server)：苹果推送通知服务的服务器。
* notification：需要推送给 iOS 客户端(iPhone或者是iPad)上的消息
* Client App：客户端 App,一般是安装在iPhone或者是iPad上的应用程序(App)
* deviceToken：是由 APNs 根据设备和App来生成的唯一的一串数据。deviceToken 在以下三种情况下会发生改变：
	* 同一个设备上重新安装同一款应用
	* 同一个应用安装在不同的设备上
	* 设备重新安装了系统，同一个应用对应的 deviceToken 也会改变


#### 1.2 推送原理

<center>
![](http://dzliving.com/Notificaiton_3.jpg)
![](http://dzliving.com/Notification_20.png)
</center>


从图中可以很清楚的看出来推送的原理主要分为以下几步：

<center>
![](http://dzliving.com/Notification_1.png)
</center>

1. 由 App 向 iOS 设备发送一个注册通知，用户需要同意系统发送推送；
2. iOS 向 APNs 远程推送服务器发送 App 的 Bundle Id 和设备的 UDID；
3. APNs 根据设备的 UDID 和 App 的 Bundle Id 生成 deviceToken 再发回给 App；
4. App 再将 deviceToken 发送给远程推送服务器(自己的服务器), 由服务器保存在数据库中。

	<center>
	![](http://dzliving.com/Notification_0.png)
	
	![](http://dzliving.com/Notification_2.png)
	</center>
	
5. 当自己的服务器想发送推送时，在远程推送服务器中输入要发送的消息并选择发给哪些用户的deviceToken，由远程推送服务器发送给 APNs。
6. APNs 根据 deviceToken 发送给对应的用户。

详细流程：

1. 在今日头条 App 的 AppDelegate 的 `didFinishLaunchingWithOptions` 方法中注册远程推送通知，此时只要 iOS 设备正常联网能够访问到外网，iOS 设备默认就会和 APNs 服务器<font color=#cc0000>维持一个基于 TCP 的长连接</font>，就会把 iOS 设备的 UDID(Unique Device Identifier:唯一设备标识码，用来标识唯一一台苹果设备)和 App 的 Bundle Identifier 通过长连接发送给 APNs 服务器，然后苹果通过这两个的值根据一定的加密算法得出 deviceToken，并将 deviceToken 返回给 iOS 设备。(注：<font color=#cc0000>APNs服务器会留有 UDID+Bundle Identifier+deviceToken 的映射表</font>)

2. 实现 UIApplicationDelegate 代理中的有关于注册远程通知的相关方法，包括注册成功、注册失败、对接收到通知的处理等。

3. 如果注册成功，实现注册成功的代理方法，就能够接收到 deviceToken，并将 deviceToken 发送给 App 服务器，App 服务器将此 deviceToken 存储在数据库中(一般如果是及时通讯类应用那么还会与用户的账号进行映射)。

4. 如果注册失败，那么实现注册失败的协议方法，处理失败后的事情。

5. app 服务器接收到 deviceToken 之后，就可以根据这些 deviceToken 向 APNs 发送推送消息。

6. APNs 接收到 deviceToken 和消息之后，根据 deviceToken 查找映射表找到对应的 UDID 和 Bundle Identifier，根据 UDID 找到唯一一台苹果设备，再在找到的苹果设备上根据 Bundle Identifier 找到唯一的应用，然后推送消息。

7. 当设备接收到消息的时候，如果 App 在前台，那么不会在设备上方弹出横幅（如果使用了音效，还会触发音效的播放），直接调用我们实现的 UIApplicationDelegate 中的接收消息的方法；如果 App 在后台或者未运行时就会在设备的上方弹出横幅（如果使用了音效，还会触发音效的播放），点击横幅才会触发调用我们实现的 UIApplicationDelegate 中的接收消息的方法，这个时候你直接点击应用图标进来是不会调用的。

## 二、信息包

信息包结构图：

<center>
![](http://dzliving.com/Notification_4.png)
</center>

上图显示的这个消息体就是我们的应用服务器（Provider）发送给 APNs 服务器的消息结构，APNs 验证这个结构正确并提取其中的信息后，再将消息推送到指定的 iOS 设备。

这个结构体包括五个部分

1. 第一部分是命令标示符
2. 第二部分是 devicetoken 的长度
3. 第三部分是 devicetoken 字符串
4. 第四部分是推送消息体（Payload）的长度
5. 最后一部分也就是真正的消息内容了，里面包含了推送消息的基本信息，比如消息内容，应用 Icon 右上角显示多少数字以及推送消息到达时所播放的声音等

Payload（消息体）的结构:

```
{
	“aps”:{
		“alert”:“CSDN给您发送了新消息”,
		“badge”:1,
		“sound”:“default”
	},
}
```

这其实就是个 JSON 结构体，alert 标签的内容就是会显示在用户手机上的推送信息，badge 显示的数量（注意是整型）是会在应用 Icon 右上角显示的数量，提示有多少条未读消息等，sound 就是当推送信息送达是手机播放的声音，传 defalut 就标明使用系统默认声音。


## 三、证书

* 应用的调试证书、描述文件

	[iOS- 最全的真机测试教程](https://www.jianshu.com/p/c8e86f62687a)

* 应用的发布证书、描述文件

	[iOS-最全的App上架教程](https://www.jianshu.com/p/cea762105f7c)

* 推送的调试证书和发布证书

	[](https://www.jianshu.com/p/9e98af89cecc)


## 四、后台接收通知

<center>
![](http://dzliving.com/Notification_6.png)
</center>

开启推送。

<center>
![](http://dzliving.com/Notification_5.png?imageView2/0/w/400)
</center>

当推送信息中包含 `content-available` 字段，并且等于 1

```
{
    "_j_business" = 1;
    "_j_msgid" = 29273432613945685;
    "_j_uid" = 31254343846;
    aps =     {
        alert = 11111;
        badge = 1;
        "content-available" = 1;
        sound = default;
    };
}
```

app 即使在后台也能在 appDelegate 中触发代理回调

```
- (void)application:(UIApplication *)application 
didReceiveRemoteNotification:(NSDictionary *)userInfo
fetchCompletionHandler: (void (^)(UIBackgroundFetchResult))completionHandler 
{

}
```

## 五、Notification Extension

[iOS10推送通知进阶(Notification Extension）](https://www.jianshu.com/p/78ef7bc04655)

<center>
![](http://dzliving.com/Notification_7.png)
</center>

1. UNNotificationContentExtension（通知内容扩展）给通知创建一个自定义的用户界面；
2. UNNotificationServiceExtension（通知服务扩展）是在收到通知后，展示通知前，做一些事情的。比如：增加附件，网络请求等。

#### 5.1 UNNotificationServiceExtension - 通知服务扩展

如果经常使用 iMessage 的朋友们，就会经常收到一些信息，附带了一些照片或者视频，所以推送中能附带这些多媒体是非常重要的。如果推送中包含了这些多媒体信息，可以使用户不用打开 app，不用下载就可以快速浏览到内容。众所周知，推送通知中带了 push payload，即使去年苹果已经把 payload 的 size 提升到了 <font color=#cc0000>4k bites</font>，但是这么小的容量也无法使用户能发送一张高清的图片，甚至把这张图的缩略图包含在推送通知里面，也不一定放的下去。在 iOS X 中，我们可以使用新特性来解决这个问题。我们可以通过新的 service extensions 来解决这个问题。

iOS10 给通知添加附件有两种情况：本地通知和远程通知。

1. 本地推送通知

	只需给 content.attachments 设置 UNNotificationAttachment 附件对象

2. 远程推送通知

	需要实现 UNNotificationServiceExtension（通知服务扩展），在回调方法中处理 推送内容时设置 request.content.attachments（请求内容的附件）属性，之后调用 contentHandler 方法即可。

UNNotificationServiceExtension 提供在远程推送将要被 push 出来前，处理推送显示内容的机会。此时可以对通知的 `request.content` 进行内容添加，如添加附件、userInfo 等。下图显示了Notification Service Extension 的流程：

<center>
![](http://dzliving.com/Notification_8.jpg)
</center>

处理的细节如下：

1. 为了能在 service extension 里面的 `attachment`，必须给 `apns` 增加 `"mutable-content":1` 字段，使你的推送通知是动态可变的。

	```
	{
	     "aps":{
	         "alert":"Testing.. (34)",
	         "badge":1,
	         "sound":"default",
	         "mutable-content":1
	      }
	}
	```

2. 给项目新建一个 Notification Service Extension 的扩展。自动生成下列文件。

	<center>
	![](http://dzliving.com/Notification_10.png)
	</center>

3. 在 -didReceiveNotificationRequest:withContentHandler: 方法中处理request.content，用来给通知的内容做修改。如下面代码示例了收到通知后，给通知增加图片附件：

	```
	- (void)didReceiveNotificationRequest:(UNNotificationRequest *)request withContentHandler:(void (^)(UNNotificationContent * _Nonnull))contentHandler {
	    self.contentHandler = contentHandler;
	    self.bestAttemptContent = [request.content mutableCopy];
	    self.bestAttemptContent.title = [NSString stringWithFormat:@"%@ [modified]", self.bestAttemptContent.title];
	    
	    //1. 下载
	    NSURL *url = [NSURL URLWithString:@"http://img1.gtimg.com/sports/pics/hv1/194/44/2136/138904814.jpg"];
	    NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
	    NSURLSession *session = [NSURLSession sessionWithConfiguration:config];
	    NSURLSessionDataTask *task = [session dataTaskWithURL:url completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
	        if (!error) {
	        
	            //2. 保存数据
	            NSString *path = [NSSearchPathForDirectoriesInDomains(NSLibraryDirectory, NSUserDomainMask, YES).firstObject
	                              stringByAppendingPathComponent:@"download/image.jpg"];
	            UIImage *image = [UIImage imageWithData:data];
	            NSError *err = nil;
	            [UIImageJPEGRepresentation(image, 1) writeToFile:path options:NSAtomicWrite error:&err];
	            
	            //3. 添加附件
	            UNNotificationAttachment *attachment = [UNNotificationAttachment attachmentWithIdentifier:@"remote-atta1" URL:[NSURL fileURLWithPath:path] options:nil error:&err];
	            if (attachment) {
	                self.bestAttemptContent.attachments = @[attachment];
	            }
	        }
	        
	        //4. 返回新的通知内容
	        self.contentHandler(self.bestAttemptContent);
	    }];
	    
	    [task resume];
	}
	```

使用 UNNotificationServiceExtension，你有 <font color=#cc0000>30</font> 秒的时间处理这个通知，可以同步下载图像和视频到本地，然后包装为一个 UNNotificationAttachment 扔给通知，这样就能展示用服务器获取的图像或者视频了。

注意：如果数据处理失败、超时，extension 会报一个崩溃信息，但是通知会用默认的形式展示出来，app不会崩溃。

附件通知所带的附件格式大小都是有限的，并不能做所有事情，视频的前几帧作为一个通知的附件是个不错的选择。

UNNotificationAttachment：attachment 支持

1. 音频 5M（kUTTypeWaveformAudio/kUTTypeMP3/kUTTypeMPEG4Audio/kUTTypeAudioInterchangeFileFormat）
2. 图片10M（kUTTypeJPEG/kUTTypeGIF/kUTTypePNG）
3. 视频50M（kUTTypeMPEG/kUTTypeMPEG2Video/kUTTypeMPEG4/kUTTypeAVIMovie）


#### 5.2 UNNotificationContentExtension - 通知内容扩展

要想创建一个自定义的用户界面，需要用到 Notification Content Extension（通知内容扩展）。

Notification Content Extension（通知内容扩展）允许开发者加入自定义的界面，在这个界面里面，你可以绘制任何你想要的东西。但是有一个最重要的限制就是，<font color=#cc0000>这个自定义的界面没有交互</font>。它们不能接受点击事件，用户并不能点击它们。但是推送通知还是可以继续与用户进行交互，因为用户可以使用 notificaiton 的 actions。

注意：extension 也可以处理这些 actions。

1. 推送界面的组成

	<center>
	![](http://dzliving.com/Notification_9.png)
	</center>

	* header 的 UI 是系统提供的一套标准的 UI。这套 UI 会提供给所有的推送通知。
	* header 下面的 custom content 是自定义的内容，就是 Notification Content Extension。在这里，就可以显示任何你想绘制的内容了。你可以展示任何额外的有用的信息给用户。
	* default content 是系统的界面。这也就是 iOS 9 之前的推送的样子。
	* notification action 用户可以触发一些操作。并且这些操作还会相应的反映到上面的自定义的推送界面 content extension 中。
	
2. 创建 Notification Content Extension

	创建一个新的 Notification Content 的 target。Xcode 自动生成一个新的模板以及下列文件。
	
	<center>
	![](http://dzliving.com/Notification_11.png)
	</center>
	
	然后打开这里的 ViewController。
	
	```
	#import "NotificationViewController.h"
	#import <UserNotifications/UserNotifications.h>
	#import <UserNotificationsUI/UserNotificationsUI.h>
	
	@interface NotificationViewController () <UNNotificationContentExtension>
	
	@property IBOutlet UILabel *label;
	
	@end
	
	@implementation NotificationViewController
	
	- (void)viewDidLoad {
	    [super viewDidLoad];
	    // Do any required interface initialization here.
	}
	
	- (void)didReceiveNotification:(UNNotification *)notification {
	    self.label.text = notification.request.content.body;
	}
	
	@end
	```
	
	发现这里的 ViewController 就是一个普通的 UIViewController, 但是它实现了 UNNotificationContentExtension 协议。
	
	UNNotificationContentExtension 协议有一个 required方法 didReceiveNotification:。当收到指定 categroy 的推送时，didReceiveNotification: 方法会随着 ViewController 的生命周期方法，一起被调用，这样就能接受 notification object，更新UI。

#### 5.3 配置category

接下来就是要让推送到达后，系统怎样找到自定义的 UI。这时候就需要配置 extension 的 info.plist 文件。
	
<center>
![](http://dzliving.com/Notification_12.png)
</center>

这里和我们给 notification actions 注册 category 一样，给这个通知扩展指定相应的 category。在 UNNotificationExtensionCategory 字段里写入相应的 category id。值得提到的一点是，这里对应的 category 是可以为一个数组的，里面可以为多个 category，这样做的目的是多个 category 共用同一套 UI。

<center>
![](http://dzliving.com/Notification_13.png)
</center>
	
上图中 category id 为 myNotificationCategory1 和 myNotificationCategory2 的通知就共用了一套 UI。
	
设置了 category 后，只要在通知里面增加 category 字段，值是上面在 extension 的 plist 里面配置的 category id，收到的通知就会通过自定义的样式显示。
	
远程通知在 apns 里面增加 category 字段。
	
```
{
    "aps":{
        "alert":"Testing.. (34)",
        "badge":1,
        "sound":"default",
        "category":"myNotificationCategory1"
     }
}
```

#### 5.4 自定义UI

然后开始写自定义UI。

```
- (void)didReceiveNotification:(UNNotification *)notification 
{
    self.label.text = [NSString stringWithFormat:@"%@ [modified]", notification.request.content.title];
    self.subLabel.text = [NSString stringWithFormat:@"%@ [modified]", notification.request.content.body];
    self.imageView.image = [UIImage imageNamed:@"hong.png"];
}
```

可以在 ViewController 中增加一些 Label 和 ImageView，收到通知的时候，提取想要的内容，或者添加额外的内容，设置到我们自定义的 View 上。

![](https://upload-images.jianshu.io/upload_images/1124181-cd4dbbd7380741f1.png)


#### 5.5 优化

1. 发现是自定义界面的大小很不美观

	这时候可以通过设置 ViewController 的 preferredContentSize大小，控制自定义视图的大小。也可以通过约束，控制自定义视图的大小。
	
	```
	- (void)viewDidLoad 
	{
	    [super viewDidLoad];
	    self.preferredContentSize = CGSizeMake(CGRectGetWidth(self.view.frame), 100);
	}
	```

2. 视图恢复成正确的尺寸前，先展示有一大片空白的样子，然后变成正确的样子。当通知展示出来之后，它的大小并不是正常的我们想要的尺寸。iOS 系统会去做一个动画来 Resize 它的大小。这样体验很差。
 
	<center>
	![](http://dzliving.com/Notification_14.gif)
	</center>
		
	会出现上面这张图的原因是，在推送送达的那一刻，iOS 系统需要知道我们推送界面的最终大小。但是我们自定义的extension在系统打算展示推送通知的那一刻，并还没有启动。所以这个时候，在我们代码都还没有跑起来之前，我们需要告诉iOS系统，我们的View最终要展示的大小。
	
	为了解决这个问题，我们需要在 extension 的 info.plist 里设置一个 content size ratio。增加字段 UNNotificationExtensionInitialContentSizeRatio。

	<center>
	![](http://dzliving.com/Notification_15.png)
	</center>

	这个属性定义了宽和高的比例。当然设置了这个比例以后，也并不是万能的。因为你并不知道你会接受到多长的content。当你仅仅只设置比例，还是不能完整的展示所有的内容。有些时候如果我们可以知道最终的尺寸，那么我们固定尺寸会更好。

3. 这时候我们发现我们自定义的界面显示的内容(custom content)和系统默认的内容(default content)重复了。

	可以在 extension 的 info.plist 里设置，把系统默认的样式隐藏。增加字段UNNotificationExtensionDefaultContentHidden。
	
	<center>
	![](http://dzliving.com/Notification_16.png)
	</center>
	
	将系统内容隐藏后效果如下：
	
	<center>
	![](http://dzliving.com/Notification_17.png)
	</center>

#### 5.6 自定义操作

iOS8 开始引入的 action 的工作原理：

默认系统的 Action 的处理是：当用户点击的按钮，就把 action 传递给 app，与此同时，推送通知会立即消失。这种做法很方便。

但是有的情况是，希望用户点击 action 按钮后，效果及时响应在我们自定义的 UI 上。这个时候，用户点击完按钮，我们把这个 action 直接传递给 extension，而不是传递给 app。当 actions 传递给 extension 时，它可以延迟推送通知的消失时间。在这段延迟的时间之内，我们就可以处理用户点击按钮的事件了，并且更新 UI，一切都处理完成之后，我们再去让推送通知消失掉。

这里我们可以运用 UNNotificationContentExtension 协议的第二个方法，这方法是 Optional

```
- (void)didReceiveNotificationResponse:(UNNotificationResponse *)response completionHandler:(void (^)(UNNotificationContentExtensionResponseOption option))completion
{
    if ([response.actionIdentifier isEqualToString:@"action-like"]) {
        self.label.text = @"点赞成功~";
    }
    else if ([response.actionIdentifier isEqualToString:@"action-collect"]){
        self.label.text = @"收藏成功~";        
    }
    else if ([response.actionIdentifier isEqualToString:@"action-comment"]){
        self.label.text = [(UNTextInputNotificationResponse *)response userText];
    }
    
    //这里如果点击的action类型为UNNotificationActionOptionForeground，
    //则即使completion设置成Dismiss的，通知也不能消失
    
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.5 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        completion(UNNotificationContentExtensionResponseOptionDismiss);
    });
}
```

在这个方法里判断所有的 action，更新界面，并延迟 1.5 秒后让通知消失。真实情况可能是，点击“赞”按钮后，发送请求给服务器，根据服务器返回结果，展示不同的UI效果在通知界面上，然后消失。如果是评论，则将评论内容更新到界面上。

如果还想把这个 action 传递给 app，最后消失的参数应该这样：

```
completion(UNNotificationContentExtensionResponseOptionDismissAndForwardAction);
```

但是我实际运行遇见这种情况，如果点击的 action 类型为 UNNotificationActionOptionForeground，则即使 completion 设置成 Dismiss 的，通知也不能消失，也没有启动 app。

#### 5.7 自定义输入型操作

action 有 2 种类型：

1. UNNotificationAction 普通按钮样式
2. UNTextInputNotificationAction 输入框样式

UNTextInputNotificationAction 的样式如下：

<center>
![](http://dzliving.com/Notification_18.png)
</center>

系统的输入样式的 action，只有在点击发送按钮时，才能接受到 action 的响应回调。（比如上面的didReceiveNotificationResponse:completionHandler: 方法）。但有的时候系统的样式或者功能不能满足需求，这时候可以自定义键盘上面的 inputAccessoryView。

首先，重写ViewController的下面两个方法：

```
- (BOOL)canBecomeFirstResponder
{
    return YES;
}

- (UIView *)inputAccessoryView
{
    return self.customInputView;
}
```

自定义 inputAccessoryView，以绘制自定义的输入样式。

```
- (void)didReceiveNotificationResponse:(UNNotificationResponse *)response completionHandler:(void (^)(UNNotificationContentExtensionResponseOption option))completion
{
    ...
    
    }
    else if ([response.actionIdentifier isEqualToString:@"action-comment"]){
        self.label.text = [(UNTextInputNotificationResponse *)response userText];
        [self becomeFirstResponder];
        [self.textField becomeFirstResponder];
        
        self.completion = completion;
    }
}
```

实现了点击评论按钮，ViewController 成为第一响应者，使自定义的输入样式显示出来。然后，让textField成为第一响应者，使键盘弹出。

这里将操作的completion保存，以便在需要的时候调用。比如，可以在点击键盘右下的send按钮时，调用completion，使通知消失。

```
- (BOOL)textFieldShouldReturn:(UITextField *)textField
{
    [textField resignFirstResponder];
    self.label.text = textField.text;
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.5 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        self.completion(UNNotificationContentExtensionResponseOptionDismiss);
    });
    return YES;
}
```

实现效果如下：

<center>
![](http://dzliving.com/Notification_19.png)
</center>

#### 5.8 结合使用两个扩展

可以在 content extension 里面绘制界面时，通过 notification.request.content.attachments 获取附件放到自定义控件里面。

```
- (void)didReceiveNotification:(UNNotification *)notification {

    ...
    
    UNNotificationAttachment * attachment = notification.request.content.attachments.firstObject;
    if (attachment) {
        if ([attachment.URL startAccessingSecurityScopedResource]) {
            self.imageView.image = [UIImage imageWithContentsOfFile:attachment.URL.path];
            dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.5 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
                [attachment.URL stopAccessingSecurityScopedResource];
            });
        }
    }
}
```

我们可以提取 content 的 attachments。前文提到过，attachment 是由系统管理的，系统会把它们单独的管理，这意味着它们存储在我们 sandbox 之外。所以这里我们要使用 attachment 之前，我们需要告诉 iOS 系统，我们需要使用它，并且在使用完毕之后告诉系统我们使用完毕了。对应上述代码就是 `-startAccessingSecurityScopedResource和-stopAccessingSecurityScopedResource` 的操作。当我们获取到了 attachment 的使用权之后，我们就可以使用那个文件获取我们想要的信息了。

#### 5.9 关于调试

很多人在开发 iOS extension 时遇到了调试的问题，可以看这里的[解决方法](https://link.jianshu.com/?t=https://github.com/liuyanhongwl/ios_common/blob/master/files/App-Extension-Tips.md)，如果还不能有效解决您的问题，欢迎评论留言。

[Demo](https://link.jianshu.com/?t=https://github.com/liuyanhongwl/UserNotification)
[【WWDC2016 Session】iOS 10 推送Notification新特性](http://www.cocoachina.com/articles/16833)
[iOS- 实现APP前台、后台、甚至杀死进程下收到通知后进行语音播报（金额）。](https://www.jianshu.com/p/601426a247e6)


## 文章

官方文档：[Local and Remote Notification Programming Guide](https://developer.apple.com/library/archive/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/APNSOverview.html#//apple_ref/doc/uid/TP40008194-CH8-SW1)
[iOS中使用本地通知为你的APP添加提示用户功能](https://link.jianshu.com/?t=https%3A%2F%2Fmy.oschina.net%2Fu%2F2340880%2Fblog%2F405491)
[iOS远程推送之(一)：APNs原理和基本配置](https://www.jianshu.com/p/9e98af89cecc)
[iOS远程推送之(二)：角标applicationIconNumber设置](https://www.jianshu.com/p/59878fd8053c)
[iOS远程推送之(三)：点击通知横幅启动应用](https://www.jianshu.com/p/d7372b166a0b)
[iOS 远程消息推送 APNS推送原理和一步一步开发详解篇](https://www.jianshu.com/p/032bfc949917)
[SmartPush](https://link.jianshu.com/?t=https%3A%2F%2Fgithub.com%2Fshaojiankui%2FSmartPush)
[iOS远程推送原理及实现过程](https://blog.csdn.net/weixin_36162680/article/details/83863718)
[iOS 推送通知及通知扩展](https://www.jianshu.com/p/4ffb56716a5c)
[iOS10 推送通知 UserNotifications](https://www.jianshu.com/p/bb89d636f989)
[iOS 推送全解析，你不可不知的所有 Tips！](https://www.jianshu.com/p/e9c313df746f)