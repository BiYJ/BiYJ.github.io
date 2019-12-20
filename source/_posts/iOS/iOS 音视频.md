---
title: iOS 音视频
categories: iOS
---

[https://www.cnblogs.com/jiayayao/p/10023951.html](https://www.cnblogs.com/jiayayao/p/10023951.html)


[分辨率、帧率和码率三者之间的关系](https://blog.csdn.net/qq_39759656/article/details/80701965)

> 码率如果为 10Mb/s，代表 1 秒钟有 10M bit 的视频数据，对于 YUV422 格式的 1080P 视频而言，一帧图像是 1920\*1080\*2\*8/1024/1024 = 31.64Mbit，1 秒钟 30 帧图像的话，则有 949.2Mb/s。
> 
> 游戏追求高帧率的目的是为了尽可能让 3D 模型渲染出来的运动效果更加接近真实运动轨迹


## 音视频源

#### AVFoundation

[Still and Video Media Capture](https://developer.apple.com/library/archive/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/04_MediaCapture.html)

![](https://developer.apple.com/library/archive/documentation/AudioVideo/Conceptual/AVFoundationPG/Art/captureOverview_2x.png)
![](https://developer.apple.com/library/archive/documentation/AudioVideo/Conceptual/AVFoundationPG/Art/captureDetail_2x.png)

[OC之AVCaptureDevice](https://www.jianshu.com/p/155efb36e041)


## 美颜

#### GPUImage

![](https://upload-images.jianshu.io/upload_images/304825-5faefe55f9296071.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

> source(视频、图片源) -> filter（滤镜） -> final target (处理后视频、图片)

![](https://upload-images.jianshu.io/upload_images/304825-84ff54fb516a70ce.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

[基于GPUImage的实时美颜滤镜](https://www.jianshu.com/p/945fc806a9b4)
[GPUImage 实现自定义相机](https://www.jianshu.com/p/e66fd9cac733)

![](https://upload-images.jianshu.io/upload_images/784630-88b91bf193866d12.png?imageMogr2/auto-orient/strip|imageView2/2/w/1012)


## 流媒体服务器

#### nginx

> Nginx 是一个非常出色的 HTTP 服务器，特点是占有内存少，并发能力强

```
$ brew tap denji/homebrew-nginx   # 下载 Nginx 到本地

$ brew install nginx-full --with-rtmp-module   # 安装 Nginx 服务器和 rtmp 模块

$ nginx   # 开启 nginx 服务器，接着浏览器输入 http://localhost:8080 

$ brew info nginx-full   # /usr/local/etc/nginx/nginx.conf

$ nginx -s reload   # 重新加载 nginx 的配置文件
```

```
rtmp {
    server {
        listen 1990;
        application liveApp {
            live on;
            record off;
        }
    }
}
```

* `application xx`  流媒体上应用名称，可以随意填
* `record off`   不记录数据

[搭建基于RTMP的本地Nginx服务器](https://blog.csdn.net/qq_24904667/article/details/80063031)


## ffmpeg

```
$ brew install ffmpeg    # 安装 ffmpeg

$ ffmpeg -re -i [本地视频全路径] -vcodec copy -f flv [rtmp 路径]
```

> ffmpeg -re -i /Users/xx/Desktop/1.mp4 -vcodec copy -f flv rtmp://localhost:1990/liveApp/room

* \-re

	按照帧率发送。
	
	需要延时发送流媒体的数据。不然的话，FFmpeg 处理数据速度很快，瞬间就能把所有的数据发送出去，流媒体服务器是接受不了的。因此需要按照视频实际的帧率发送数据。
	
* \-i

	输入文件

* \-vcodec copy:

	强制使用 codec 编解码方式，要加，否则 ffmpeg 会重新编码输入的 H.264 裸流。

* \-f

	强制转换格式，flv 为格式

* rtmp://localhost:1990/liveApp/room

	需要跟配置的一一对应，端口、应用名称，room 可以随便写


[ffmpeg参数中文详细解释](http://blog.csdn.net/leixiaohua1020/article/details/12751349)

```
ffmpeg -i abc.mp4 -b:v 640k abc.flv   # .mp4 转码成 .flv
```

[iOS 利用FFmpeg 开发音视频流（一）——Mac 系统上编译 FFmpeg](https://www.jianshu.com/p/12941473a61d)
[iOS 利用FFmpeg 开发音视频流（二）——Mac 系统上编译 iOS 可用的FFmpeg 库](https://www.jianshu.com/p/ec432a8f5729)


## 播放器


#### ijkplayer

[iOS中集成ijkplayer视频直播框架](https://www.jianshu.com/p/1f06b27b3ac0)


## 直播


[【如何快速的开发一个完整的iOS直播app】(播放篇)](https://www.jianshu.com/p/7b2f1df74420)
[【如何快速的开发一个完整的iOS直播app】(采集篇)](https://www.jianshu.com/p/c71bfda055fa)
[【如何快速的开发一个完整的iOS直播app】(美颜篇)](https://www.jianshu.com/p/4646894245ba)
[【如何快速的开发一个完整的iOS直播app】(推流篇)](https://www.jianshu.com/p/53059be61546)
[【如何快速的开发一个完整的iOS直播app】(搭建Web服务器)](https://www.jianshu.com/p/d76ecf5ed690)

<center>
![](https://upload-images.jianshu.io/upload_images/304825-5481594e6e2a9d56.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)
![](https://upload-images.jianshu.io/upload_images/304825-54974199408c0cc1.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)
![](https://upload-images.jianshu.io/upload_images/304825-9b64e9596f3ccdce.jpeg?imageMogr2/auto-orient/strip|imageView2/2)
</center>


## 编解码

[视频软解码和硬解码的区别](https://blog.csdn.net/qq_15807167/article/details/52262559)
[视频文件格式、视频封装格式、视频编码方式](https://blog.csdn.net/electrombile/article/details/51067344)

<center>

|文件格式|封装格式|编码格式|
|:------:|:-----:|:-------:|
|AVI|AVI|MPEG-2，DIVX，XVID，AC-1，H.264|
|WMV|WMV|WMV，AC-1|
|RM、RMVB|Real Video|RV，RM|
|MOV|QuickTime File Format|MPEG-2，XVID，H.264|
||TS/PS|MPEG-2，H.264，MPEG-4|
|MPG、MPEG、VOB、DAT、3GP、MP4|MPEG||
|MKV|Matroska|可以封装所有的视频编码格式|
|FLV|Flash Video||

</center>

![](https://upload-images.jianshu.io/upload_images/1073278-998c16363298004e.jpg)

## 推流

[iOS 直播 —— 推流](https://www.jianshu.com/p/27b3bb4695aa)

<center>
![](https://upload-images.jianshu.io/upload_images/784630-410c22a906db5ba9.png?imageMogr2/auto-orient/strip|imageView2/2/w/1016)
![](https://upload-images.jianshu.io/upload_images/784630-206124c2ad59164b.png?imageMogr2/auto-orient/strip|imageView2/2/w/373)
![](https://upload-images.jianshu.io/upload_images/784630-93465c366c7a601c.png?imageMogr2/auto-orient/strip|imageView2/2/w/859)
</center>

> FLV

> [[Flv Header][Metainfo Tag][Video Tag][Audio Tag][Video Tag][Audio Tag][Other Tag]...]


[iOS视频直播（推流原理篇）](https://www.jianshu.com/p/a6a2db3c1ec2)

## 直播协议

[直播协议的选择：RTMP vs. HLS](http://www.samirchen.com/ios-rtmp-vs-hls/)

> 相对于常见的流媒体直播协议，例如 RTMP 协议、RTSP 协议等，HLS 最大的不同在于直播客户端获取到的并不是一个完整的数据流，而是连续的、短时长的媒体文件，客户端不断的下载并播放这些小文件

HLS 的缺点：

* 通常 HLS 直播延时会达到 20~30s，而高延时对于需要实时互动体验的直播来说是不可接受的。
* HLS 基于短连接 HTTP，HTTP 是基于 TCP 的，这就意味着 HLS 需要不断地与服务器建立连接，TCP 每次建立连接时的三次握手、慢启动过程、断开连接时的四次挥手都会产生消耗。

HLS 的优点：

* 数据通过 HTTP 协议传输，所以采用 HLS 时不用考虑防火墙或者代理的问题。
* 使用短时长的分片文件来播放，客户端可以平滑的切换码率，以适应不同带宽条件下的播放。
* HLS 是苹果推出的流媒体协议，在 iOS 平台上可以获得天然的支持，采用系统提供的 AVPlayer 就能直接播放，不用自己开发播放器。

![](http://www.samirchen.com/images/ios-rtmp-vs-hls/HLS.png)

> 采用 RTMP 协议时，从采集推流端到流媒体服务器再到播放端是一条数据流，因此在服务器不会有落地文件。

RTMP 优点：

* 延时较小，通常为 1~3s。
* 基于 TCP 长连接，不需要多次建连。

RTMP 缺点：

* iOS 平台没有提供原生支持 RTMP 或 HTTP-FLV 的播放器，这就需要开发支持相关协议的播放器。

![](https://upload-images.jianshu.io/upload_images/304825-f92e85515845e107.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

## 直播优化

[直播中累积延时的优化](http://www.samirchen.com/live-delay-optimization/)

前后台切换导致的累计延时：

1. 视频同步音频的策略
2. 重连直播流

网络状况导致的卡顿延时：

1. 使用 CDN 分发网络。
2. 合理采用 CDN 边缘节点推流。
3. 推流端、播放端使用 HTTPDNS 选择网络状况最好的节点接入。
4. 推流端实现码率自适应策略，在网络状况不佳的情况下，降低推流码率来降低上行带宽压力。
5. 流媒体服务器提供多档位直播流服务，与此同时，播放端实现直播流多档位切换策略，在网络状况不佳的情况下，切换到低档位直播流来降低下行带宽压力。

各端缓冲区策略：

1. 在「卡顿」和「累积延时」这两项体验指标上寻找一个平衡点，在各端设置合适的缓冲区大小。
2. 在各端实现一些丢帧策略，当缓冲区超过一定阈值时，开始丢帧。
3. 在播放端的缓冲区过大时，尝试断开重连。

[iOS直播优化方案](https://www.jianshu.com/p/c9a6ff3690a4)

直播列表刷新机制：

* 全量刷新
* 顶部刷新，按滑动屏幕的偏移量。
* 局部刷新，对直播流进行分组，只更新用户当前界面的直播流，便于服务端实施缓存策略。
* 刷新的时间间隔可配置。
* 用户当前正在滑动屏幕时，延迟刷新当前页面。

进入直播间速度的优化：

* 在直播间大厅预先解析视频流地址，从而加快获取视频数据的速度。
* 对房间内部分 UI 模块使用懒加载方式，如：公聊、用户列表等。

直播间内的优化：

![](https://upload-images.jianshu.io/upload_images/1896520-c9743644432b03de.png?imageMogr2/auto-orient/strip|imageView2/2/w/764)

直播间内长连接消息的处理：

* 将处理长链接消息的 SocketIO 库中的 block 执行线程设置为非主线程，缓解主线程的 CPU 占用；
* 实时统计单位时间内收到的消息数目，在性能较低的机型上，动态调整公聊列表的刷新频率；
* 使用队列暂存消息，每条公聊消息到来时不直接刷新列表；
* 批量地 pop 队列中的消息，只保留最近收到的一些消息进行滚动刷新。

动画流畅：

* 动画进行队列分优先级排列，对于同个时间段内的动画根据优先级高的的优先显示，其余可延迟或者抛弃动画显示。

App 崩溃：

* 开始直播时保存推流地址信息，App 崩溃后，再次启动可以继续直播，恢复效果上相当于一次切后台的操作。

[直播卡顿优化](https://blog.csdn.net/weixin_40763897/article/details/95751375)
[如何优化视频卡顿](https://cloud.tencent.com/document/product/454/7946)
[iOS直播技术分享-延迟优化（五）](https://www.jianshu.com/p/2e1614d216ac)