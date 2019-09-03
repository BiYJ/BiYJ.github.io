---
title: iOS 构建动态库
categories: iOS原理
---

## 一、构建步骤

1. 创建一个动态库 MyDynamicFramework

	<center>
	![1](https://upload-images.jianshu.io/upload_images/5294842-1d96752661c51928.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

2. 创建一个测试类
	
	<center>
	![2](https://upload-images.jianshu.io/upload_images/5294842-19c8607f757caffa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

3. 在 MyDynamicFramework.h（默认生成，可统一暴露头文件） 中 \#import "Person.h"

	```
	#import <UIKit/UIKit.h>
	
	//! Project version number for MyDynamicFramework.
	FOUNDATION_EXPORT double MyDynamicFrameworkVersionNumber;
	
	//! Project version string for MyDynamicFramework.
	FOUNDATION_EXPORT const unsigned char MyDynamicFrameworkVersionString[];
	
	// In this header, you should import all the public headers of your framework using statements like #import <MyDynamicFramework/PublicHeader.h>
	
	#import "Person.h"
	```

4. 点击工程 -> Targets -> Build Phases -> Headers。

	动态库中新建的文件会自动添加到 project 列表，MyDynamicFramework.h 文件是处于 Public 列表中。由于动态库外部使用者需要调用 Person.h 中的方法，所以也需要将 Person.h 拖拽到 Public 列表。

	<center>
	![3](https://upload-images.jianshu.io/upload_images/5294842-a0c3255d6f8e059e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

5. 编译动态库

	选择动态库对应的 Scheme，选择 Generic iOS Device 或真机编译出对应真机的动态库，Command + B 编译。在 Xcode 工程中的 Products（这个目录不是工程源文件目录，而是编译后生成对应的沙盒目录）找到 MyDynamicFramework.framework 文件，右键 show in finder。

	<center>
	![4](https://upload-images.jianshu.io/upload_images/5294842-fa2e1c628bf5d0ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

6. 利用 lipo -info 查看动态库所支持的 CPU 指令集。

	<center>
	![5](https://upload-images.jianshu.io/upload_images/5294842-f9636efca8e466c4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

	新建工程后所编译出来的动态库所支持的 CPU 指令集是 arm 7、arm64。

	需要注意：
	```
	lipo -info [文件]
	```

	后面跟的是文件路径，而<font color=#cc0000>不是 .framework 路径</font>。

7. 指令集种类

	* armv7｜armv7s｜arm64 都是 ARM 处理器的指令集 
	* i386｜x86_64 是 iOS 模拟器的指令集
    
	理论上指令集是<font color=#cc0000>向下兼容</font>的，比如连接设备为 arm64，那么是有可能编译出的动态库所支持的指令集为 armv7s 或者是 armv7。但是向下兼容并不是说一个 armv7s 的动态库可以用在 arm64 架构的设备上，如果连接的设备是 arm64 的，而导入的动态库是没有支持 arm64，那么在编译阶段即会报错。

8. Xcode 指令集的编译选项

	打开 Target -> Build Setting -> Architectures

	<center>
	![6](https://upload-images.jianshu.io/upload_images/5294842-62927892def45aa0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

	* Architectures：指明选定 Target 要求被编译生成的二进制包所支持的指令集
	* Build Active Architecture Only：指明是否只编译当前连接设备所支持的指令集，如果为 YES，那么只编译出连接设备所对应的指令集；如果为 NO，则编译出所有其它有效的指令集（由 Architectures 和 Valid Architectures决定）
	* Valid Architectures：指明可能支持的指令集并非 Architectures 列表中指明的指令集都会被支持
    
    编译产生的动态库所支持的指令集将由上面三个编译选项所影响，首先一个动态库要成功编译，则需要这三个编译选项的交集不为空。
    

9. 制作支持各机型的动态库

	* Build Active Architecture Only 统一为 NO  
	* Architectures 和 Valid Architectures 都设置为 armv7、armv7s、arm64、arm64e
    
	真机 Command + B 则生成支持 armv7、armv7s、arm64 的动态库，模拟器运行，则生成支持 i386、x86_64 的动态库。

10. 合并模拟器和真机动态库

	<center>
	![7](https://upload-images.jianshu.io/upload_images/5294842-0ddeffdaf5249655.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

	使用 <font color=#cc0000>lipo -create -output</font> 命令合动态库，注意路径是文件路径，不是 .framework 的路径。

11. 使用脚本合并

	* 新建一个 target 脚本。

	<center>
	![8](https://upload-images.jianshu.io/upload_images/5294842-2706f7ee2b9fc8ff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>
	
	* 粘贴以下脚本内容到指定位置
    
	<center>
	![9](https://upload-images.jianshu.io/upload_images/5294842-17a6d2b9aa9c6f96.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

	```
	if [ "${ACTION}" = "build" ]
	then
	INSTALL_DIR=${SRCROOT}/Products/${PROJECT_NAME}.framework
	
	DEVICE_DIR=${BUILD_ROOT}/${CONFIGURATION}-iphoneos/${PROJECT_NAME}.framework
	
	SIMULATOR_DIR=${BUILD_ROOT}/${CONFIGURATION}-iphonesimulator/${PROJECT_NAME}.framework
	
	
	if [ -d "${INSTALL_DIR}" ]
	then
	rm -rf "${INSTALL_DIR}"
	fi
	
	mkdir -p "${INSTALL_DIR}"
	
	cp -R "${DEVICE_DIR}/" "${INSTALL_DIR}/"
	#ditto "${DEVICE_DIR}/Headers" "${INSTALL_DIR}/Headers"
	
	# 使用lipo命令将其合并成一个通用framework  
	# 最后将生成的通用framework放置在工程根目录下新建的Products目录下  
	lipo -create "${DEVICE_DIR}/${PROJECT_NAME}" "${SIMULATOR_DIR}/${PROJECT_NAME}" -output "${INSTALL_DIR}/${PROJECT_NAME}"
	
	#open "${DEVICE_DIR}"
	#open "${SRCROOT}/Products"
	fi
	```

	* 编译新 target
	* 编译完成后生成的 framework 位于工程源代码根目录下的 Products 文件夹下面，通过 lipo -info 可以看到动态库已经支持 i386、x86_64、armv7、armv7s、arm64。
    
	注意：是工程目录，不是沙盒目录。

	<center>
	![10](https://upload-images.jianshu.io/upload_images/5294842-8b8ce94fac9201b6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	
	![11](https://upload-images.jianshu.io/upload_images/5294842-b6bdf42669309d73.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

12. 使用动态库

	在新工程的 target -> General -> Embedded Binaries 中添加 MyDynamicFramework.framework。

	<center>
	![12](https://upload-images.jianshu.io/upload_images/5294842-6ddf27a9f7b57855.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>


## 二、动态使用

#### 2.1 使用别人提供的动态库遇到的坑

①、第三方库所支持的 CPU 指令集不全。
②、运行过程中出现 image not found 异常或者控制台没有异常输出。
原因：没有往 Embedded Binaries 中添加 xxx.framework

#### 2.2 动态库动态更新问题

能否用动态库来动态更新 AppStore 上的版本呢？
	
framework 本来是苹果专属的内部提供的动态库文件格式，但是自从 2014 年 WWDC 之后，开发者也可以自定义创建framework 实现动态更新（绕过 apple store 审核，从服务器发布更新版本）的功能，这与苹果限定的上架的 app 必须经过apple store 的审核制度是冲突的，所以含有自定义的 framework 的 app 是无法在商店上架的，但是如果开发的是企业内部应用，就可以考虑尝试使用动态更新技术来将多个独立的 app 或者功能模块集成在一个 app 上面。
	
企业内部使用的 app，将企业官网中的板块开发成 4 个独立的 app，然后将其改造为 framework 文件最终集成在一款平台级的 app 当中进行使用，这样就可以在一款 app 上面使用原本 4 个 app 的全部功能。
	
使用自定义的动态库的方式来动态更新只能用在 in house（企业发布）和 develop 模式却但不能在使用到 AppStore，因为在上传打包的时候，苹果会对我们的代码进行一次 Code Singing，包括 app 可执行文件和所有 Embedded 的动态库。因此，只要你修改了某个动态库的代码，并重新签名，那么 MD5 的哈希值就会不一样，在加载动态库的时候，苹果会检验这个 hash 值，当苹果监测到这个动态库非法时，就会造成 Crash。

#### 2.3 iOS 如何使用 framework 来进行动态更新？

重要参考文档：[iOS 利用 Framework 进行动态更新](http://nixwang.com/2015/11/09/ios-dynamic-update/)

#### 2.4 谈谈 Mach-O

* 在制作 framework 的时候需要选择这个 Mach-O Type，确定 static、dynamic 类型库.
* 为 Mach Object 文件格式的缩写，它是一种用于可执行文件，目标代码、动态库、内核转储的文件格式。作为 a.out 格式的替代，Mach-O 提供了更强的扩展性，并提升了符号表中信息的访问速度。  
    

#### 2.5 自己创建的动态库

自建的动态库和系统的动态库有什么区别呢？
	
我们创建的动态库是在自己应用的 .app 目录里面，只能自己的 App Extension 和 APP 使用。而系统的动态库是在系统目录里面，所有的程序都能使用。
	
可执行文件和自己创建的动态库位置：
	
一般我们得到的 iOS 程序包是 .ipa 文件。其实就是一个压缩包，解压缩 .ipa 后里面会有一个 payload 文件夹，文件夹里有一个 .app 文件，右键显示包内容，然后找到一个一般体积最大的、与 .app 同名的文件，那个文件就是可执行文件。

<center>
![20](https://upload-images.jianshu.io/upload_images/5294842-6c9126111deeb112.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 
![21](https://upload-images.jianshu.io/upload_images/5294842-6b14325efeab9f6d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

在模拟器上运行的时候用 \[\[NSBundle mainBundle\] bundlePath\]; 就能得到 .app 的路径。可执行文件就在 .app 里面。

而我们自己创建的动态库就在 .app 目录下的 Framework 文件夹里。
	
<center>
![22](https://upload-images.jianshu.io/upload_images/5294842-3d47ed70dbd5d025.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>
	
我们可以看一下可执行文件中对动态库的链接地址。用[MachOView](https://link.jianshu.com?t=http://ofcckdrlc.bkt.clouddn.com/MachOView.app.zip)查看可执行文件。其中 @rpth 这个路径表示的位置可以查看[Xcode 中的链接路径问题](https://www.jianshu.com/p/cd614e080078)，而现在表示的其实就是 .app 下的 Framework 文件夹。
	
<center>
![23](https://upload-images.jianshu.io/upload_images/5294842-90fe4c9c69836d46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	
![24](https://upload-images.jianshu.io/upload_images/5294842-58535b001a052566.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>
	
下图表示了静态库、自建的动态库和系统动态库：
	
![25](https://upload-images.jianshu.io/upload_images/5294842-2488a9514d240879.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	

## 三、文章
[iOS 动态库制作以及遇到的坑](https://www.jianshu.com/p/87a412b07d5d)