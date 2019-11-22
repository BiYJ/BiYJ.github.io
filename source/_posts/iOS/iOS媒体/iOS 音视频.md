---
title: iOS 音视频
categories: iOS
---

## ijkplayer

[iOS中集成ijkplayer视频直播框架](https://www.jianshu.com/p/1f06b27b3ac0)

## AVFoundation

[Still and Video Media Capture](https://developer.apple.com/library/archive/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/04_MediaCapture.html)

![](https://developer.apple.com/library/archive/documentation/AudioVideo/Conceptual/AVFoundationPG/Art/captureOverview_2x.png)
![](https://developer.apple.com/library/archive/documentation/AudioVideo/Conceptual/AVFoundationPG/Art/captureDetail_2x.png)

[OC之AVCaptureDevice](https://www.jianshu.com/p/155efb36e041)

## GPUImage

![](https://upload-images.jianshu.io/upload_images/304825-5faefe55f9296071.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

> source(视频、图片源) -> filter（滤镜） -> final target (处理后视频、图片)

![](https://upload-images.jianshu.io/upload_images/304825-84ff54fb516a70ce.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

[基于GPUImage的实时美颜滤镜](https://www.jianshu.com/p/945fc806a9b4)
[GPUImage 实现自定义相机](https://www.jianshu.com/p/e66fd9cac733)


## nginx

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


## 直播


[【如何快速的开发一个完整的iOS直播app】(原理篇)](https://www.jianshu.com/p/bd42bacbe4cc)
[【如何快速的开发一个完整的iOS直播app】(播放篇)](https://www.jianshu.com/p/7b2f1df74420)
[【如何快速的开发一个完整的iOS直播app】(采集篇)](https://www.jianshu.com/p/c71bfda055fa)
[【如何快速的开发一个完整的iOS直播app】(美颜篇)](https://www.jianshu.com/p/4646894245ba)
[【如何快速的开发一个完整的iOS直播app】(推流篇)](https://www.jianshu.com/p/53059be61546)
[【如何快速的开发一个完整的iOS直播app】(搭建Web服务器)](https://www.jianshu.com/p/d76ecf5ed690)

![](https://upload-images.jianshu.io/upload_images/304825-5481594e6e2a9d56.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)
![](https://upload-images.jianshu.io/upload_images/304825-54974199408c0cc1.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)
![](https://upload-images.jianshu.io/upload_images/304825-9b64e9596f3ccdce.jpeg?imageMogr2/auto-orient/strip|imageView2/2)
![](https://upload-images.jianshu.io/upload_images/304825-f92e85515845e107.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

[iOS 直播 —— 推流](https://www.jianshu.com/p/27b3bb4695aa)

![](https://upload-images.jianshu.io/upload_images/784630-410c22a906db5ba9.png?imageMogr2/auto-orient/strip|imageView2/2/w/1016)
![](https://upload-images.jianshu.io/upload_images/784630-206124c2ad59164b.png?imageMogr2/auto-orient/strip|imageView2/2/w/373)
![](https://upload-images.jianshu.io/upload_images/784630-93465c366c7a601c.png?imageMogr2/auto-orient/strip|imageView2/2/w/859)