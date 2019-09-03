---
title: iOS 构建静态库
categories: iOS原理
---

## 一、.a 文件静态库打包

1. 打开 Xcode 创建一个新的 Static Library 工程，取名 MyStaticLibrary。

	<center>
	![image.png](https://upload-images.jianshu.io/upload_images/5294842-0325cf47ca37c98a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

2. 创建工程完毕后，系统自动创建了一个同名类，添加一个方法用于测试。

	<center>
	![4](https://upload-images.jianshu.io/upload_images/5294842-8df90a574aa45c9a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

	```
	#import <Foundation/Foundation.h>
	
	@interface MyStaticLibrary : NSObject
	
	+ (void)test;
	
	@end
	
	@implementation MyStaticLibrary
	
	+ (void)test
	{
	    NSLog(@"sssss");
	}
	
	@end
	```

3. Command + B 运行工程进行打包。运行完毕后，在工程中 Products 文件夹下的 libMyStaticLibrary.a 文件由红色变成了黑色。右键 show in finder 可以在其目录下找到它。这就是我们打包好的 .a 静态文件。

	<center>
	![5](https://upload-images.jianshu.io/upload_images/5294842-a1c49e3f8dca1c1d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

	接下来<font color=#cc0000>公开</font>些接口或者头文件供人调用。

4. 公开接口头文件

	targets -> Build Phases -> Copy Files -> "+" 添加你需要公开的头文件。可以多添加几个类。
	<center>
	![6](https://upload-images.jianshu.io/upload_images/5294842-ecfbaeaf49520e08.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

	公开头文件后，Command + B 重新运行打包，我们会得到一个 include 文件夹和一个 .a 静态库。
	
	<center>
	![7](https://upload-images.jianshu.io/upload_images/5294842-e866fe6bddccf47b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

5. 新建一个可运行的工程，把这两个打包好的文件拖入项目测试。

	<center>
	![8](https://upload-images.jianshu.io/upload_images/5294842-4f20ac5a057ea50b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>

	选择 iPhone7 模拟器运行，运行程序，看到日志输出没有问题，即打包.a静态库大功告成。
	
	别高兴的太早。当把模拟器切换成 iPhone5 运行时，编译直接不通过，报错如下：
	
	<center>
	![9](https://upload-images.jianshu.io/upload_images/5294842-a5036775a9941b3f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	</center>
		
	Undefined symbols for architecture i386 的意思是 libMyStaticLibrary.a 静态库不支持 i386 架构，即 32 位模拟器。
	
	iPhone5 模拟器正好是 i386 架构，打包的静态库不支持。但是 iPhone7 模拟器运行却没有问题，这说明打包的静态库支持 iPhone7 模拟器的 cpu 架构 x86\_64。如何查看静态库所支持的架构，请看下一步。

6. 终端查看静态库所支持的架构

	终端 -\> cd 进入库文件路径 -> lipo -info 库名
	
	<center>
	![10](https://upload-images.jianshu.io/upload_images/5294842-0aa38c672b37daf7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	<center>
	
	可以看到静态库仅支持 x86\_64 架构，所以运行 iPhone5 模拟器时，编译会报错。

7. 设置适配所有模拟器架构

	project -> buildSeting -> Build Active Architecture Only 设为 NO，Valid Architectures 添加 arm7、arm7s 等架构，注意工程 iOS Deployment Target 设置为较低版本，如 8.0，不然不会有 i386。
	
	![11](https://upload-images.jianshu.io/upload_images/5294842-b0ae6154bcac8780.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	
	设置完成后，重新 Command + B 运行打包静态库文件（这时你可随便选一个模拟器），按照上述第 6 步终端查看其支持的架构，我们可以看到终端输出的结果是同时支持 i386 和 x86_64，这也就意味着同时支持所有模拟器。
	
	![12](https://upload-images.jianshu.io/upload_images/5294842-96d17cd63ccc6b63.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	
	到这里打包 .a 静态库已经告一段落，但是按上述流程打包的只能在模拟器上跑，真机是不能运行的，因为 ios 真机设备跟模拟器的架构又不一样，继续处理。

8. 打包支持真机架构的静态库

	所有流程都跟上面的一样，只是我们运行打包时要选择真机运行，如下图你可以选择自己插上去的真机，也可以选择 Generic ios Devices。当然不要忘记了设置支持所有真机机型架构： Build Active Architecture Only 设为 NO。
	
	看下打包出来的终端查看结果如下：
	
	![13](https://upload-images.jianshu.io/upload_images/5294842-fbd399845a26206c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	
	可以看到同时支持 armv7 和 arm64，也就是支持所有 ios 设备。好了到此打包 .a 静态库算是告一段落。
	
	如果要同时支持模拟器和真机，请使用命令合成 .a 静态库：
	
	```
	lipo -create \[name1.a 所在路径\] \[name2.a 所在路径\] -output \[newname.a\]
	```
	
	![14](https://upload-images.jianshu.io/upload_images/5294842-23c07761cdb71163.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	
	
	
## 二、.frameworke 文件静态库打包

1. Xcode 创建一个新的工程 MyFrameworkLib，选择工程如下：

	![15](https://upload-images.jianshu.io/upload_images/5294842-513d09f5af9113fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	
	创建完成后我们可以看到，工程本身自带一个 MyFrameworkLib.h 文件，这是类似一个主头文件一样的东西
	
	![16](https://upload-images.jianshu.io/upload_images/5294842-8f198b178e234f0d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. 创建需要测试的类。

	```
	#import <Foundation/Foundation.h>
	
	@interface MyFramework : NSObject
	
	+ (void)test;
	
	@end
	
	@implementation MyFramework
	
	+ (void)test
	{
	    NSLog(@"sssss");
	}
	
	@end
	```

3. 设置支持所有模拟器架构或真机架构（和打包 .a 第 7 步骤一样）

4. 公开头文件

	target -> Build Phases -> Headers -> 把需要公开的头文件从 project 拖入 Public。
	
	![17](https://upload-images.jianshu.io/upload_images/5294842-b76eae52a25b3bc8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

5. 设置打包的是静态库。

	因为动态库也可以是以 framework 形式存在，所以需要设置，否则默认打出来的是动态库（注意：如果要上线 AppSotre，一定要改成静态库，否则审核通不过）
	
	target -> BuildSetting -> 搜索关键字 mach-> Mach-o Type 设为 Static Library（这个默认选项是动态的）
	
	![18](https://upload-images.jianshu.io/upload_images/5294842-f1177d9ac68f5e8e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

6. 选中真机或模拟器运行设备打包（与打包 .a 一样），完成后 Products 文件夹下的 MyFrameworkLib.framework 文件由红色变成了黑色，右键 show in finder 显示如下：

	![19](https://upload-images.jianshu.io/upload_images/5294842-f15f1227e7dff56c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	
	MyFrameworkLib.framework 拖入项目便可直接使用。此外还要补充的一点是，打包静态库的时候还需注意打包的是测试版（Debug）还是发布版（Release），这个根据你自己的需求决定，而如何进行设置请下一步骤。

7. 设置打包静态库的测试版和发布版（.a 和 .frameworke）

	product -> scheme -> Edit scheme -> Run -> 选择 Debug 或 Release。
	
	![20](https://upload-images.jianshu.io/upload_images/5294842-dda92708cab9dae6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	
	如果要同时支持多种架构，和 .a 类似，需要设置 Build Activ Architecture Only = NO 和 iOS Deployment Target。
	
	![21](https://upload-images.jianshu.io/upload_images/5294842-e2eebccf407d08d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	
	如果要同时支持模拟器和真机，和 .a 类似，使用命令合成 framework 库：lipo -create \[<name1>.framework/<name1>\] \[<name2>.framework/<name2>\] -output newname
	
	![22](https://upload-images.jianshu.io/upload_images/5294842-e22ed06672764395.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	
	将生成的 MyLib 替换掉任何一个里面的 MyFrameworkLib 文件。