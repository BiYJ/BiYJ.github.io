---
title: iOS UmbrellaFramework
categories: iOS原理
---


## 一、umbrella framework

> 将几个已经封装好的 framework 封装成一个，封装的这种 framework 就是 umbrella framework。

Apple 的官方文档中明确提到了不建议自己去创建 umbrellaframework。首先先引入 Apple 的 Guidelins for Creating Frameworks 的一段：

>Don’t Create Umbrella Frameworks
>
>While it is possible to create umbrella frameworks using Xcode, doing so is unnecessary for most developers and is not recommended. Apple uses umbrella frameworks to mask some of the interdependencies between libraries in the operating system. In nearly all cases, you should be able to include your code in a single, standard framework bundle. Alternatively, if your code was sufficiently modular, you could create multiple frameworks, but in that case, the dependencies between modules would be minimal or nonexistent and should not warrant the creation of an umbrella for them

分三个部分逐步创建并使用 UmbrellaFramework：

1. SubFramework：创建一个基础 framework
2. UmbrellaFramework：framework 里封装 framework
3. UmbrellaFrameworkDemo：使用 demo

## 二、创建一个基础的 framework

[UmbrellaFramework（一）创建基础framework](https://www.jianshu.com/p/eb1bdd226ea9)

1. 创建一个 framework 工程：Subframework；

	<center>
	![](http://dzliving.com/iOSUmbrellaFramework_1.png)
	</center>
		
2. 添加 SubSayHello 类，添加 sayHello 方法；

	```
	@interface SubSayHello : NSObject
	- (void)sayHello;
	@end
	
	@implementation SubSayHello

	- (void)sayHello
	{
	    NSLog(@"say Hello");
	}
	
	@end
	```
	
3. 在 SubFramework.h 头文件中导入 SubSayHello.h

	```
	#import <Subframework/SubSayHello.h>
	```
	
4. 将 SubSayHello.h 添加到 Target -> Build Phases -> Headers -> Public，可手动拖拽；

5. Build Settings -> Mach-O Type 选择 Static Library 静态库

6. 生成通用 framework

	* 方式一：分别在模拟器和真机下编译工程，生成两个 framework，用命令行合并成一个通用的。
		
		![](http://dzliving.com/iOSUmbrellaFramework_0.png)
		
		```
		$ lipo -create [真机 Framework 二进制文件路径] [模拟器 Framework 二进制文件路径] -output [结果路径]
		```
		
		> $ lipo -create /Users/cykj/Library/Developer/Xcode/DerivedData/Subframework-hkwchwbjmtuhoseinwkzbtcjxpbj/Build/Products/Debug-iphoneos/Subframework.framework/Subframework /Users/cykj/Library/Developer/Xcode/DerivedData/Subframework-hkwchwbjmtuhoseinwkzbtcjxpbj/Build/Products/Debug-iphonesimulator/Subframework.framework/Subframework -output /Users/cykj/Desktop/Subframework
		
		注意：如果执行命令报错，可以将结果地址改为 `/Users/cykj/Desktop/Subframework.xx`，生成后再将后缀名去掉。 
		
		
	* 方式二：脚本生成

		* 为 SubFramework 工程添加 Target -> Aggregate

			![](http://dzliving.com/iOSUmbrellaFramework_2.png)
			
		* 在新添加的 Target 中添加脚本

			![](http://dzliving.com/iOSUmbrellaFramework_3.png)

		* 脚本内容

			```
			# Sets the target folders and the final framework product.
			FRAMEWORK_NAME=LibraryName
			FRAMEWORK_VERSION=1.0
			FRAMEWORK_CONFIG=Release
			
			# Install dir will be the final output to the framework.
			# The following line create it in the root folder of the current project.
			INSTALL_PATH=${PROJECT_DIR}/Products/
			INSTALL_DIR=${INSTALL_PATH}/${FRAMEWORK_NAME}.framework
			
			# Working dir will be deleted after the framework creation.
			WORK_DIR=build
			DEVICE_DIR=${WORK_DIR}/${FRAMEWORK_CONFIG}-iphoneos/${FRAMEWORK_NAME}.framework
			SIMULATOR_DIR=${WORK_DIR}/${FRAMEWORK_CONFIG}-iphonesimulator/${FRAMEWORK_NAME}.framework
			
			xcodebuild -configuration "${FRAMEWORK_CONFIG}" -target "${FRAMEWORK_NAME}" -sdk iphoneos
			
			echo "Build simulator"
			xcodebuild -configuration "${FRAMEWORK_CONFIG}" -target "${FRAMEWORK_NAME}" -sdk iphonesimulator
			
			# Creates install directory if it not exits.
			if [ ! -d "${INSTALL_DIR}" ]
			then
			mkdir -p "${INSTALL_DIR}"
			fi
			
			# Creates headers directory if it not exits.
			if [ ! -d "${INSTALL_DIR}/Headers" ]
			then
			mkdir -p "${INSTALL_DIR}/Headers"
			fi
			
			# Remove all files in the headers diectory.
			for file in `ls "${INSTALL_DIR}/Headers"`
			do
			rm "${INSTALL_DIR}/Headers/${file}"
			done
			
			# Remove binary library file.
			rm -f ${INSTALL_DIR}/${FRAMEWORK_NAME}
			
			# Copies the headers files to the final product folder.
			if [ -d "${DEVICE_DIR}/Headers" ]
			then
			for file in `ls "${DEVICE_DIR}/Headers"`
			do
			cp "${DEVICE_DIR}/Headers/${file}" "${INSTALL_DIR}/Headers/${file}"
			done
			fi
			
			# copy nibs to bundle,then copy bundle to final folder
			BUNDLE_DIR=${DEVICE_DIR}/${FRAMEWORK_NAME}.bundle
			
			if [ -d "${BUNDLE_DIR}" ];then
			if ls ${DEVICE_DIR}/*.nib >/dev/null 2>&1;then
			rm -rf ${BUNDLE_DIR}/*.nib
			cp -rf ${DEVICE_DIR}/*.nib ${BUNDLE_DIR}
			fi
			rm -rf "${INSTALL_DIR}/${FRAMEWORK_NAME}.bundle"
			cp -R "${BUNDLE_DIR}" "${INSTALL_DIR}/${FRAMEWORK_NAME}.bundle"
			fi
			
			echo "Merge with simulator"
			# Uses the Lipo Tool to merge both binary files (i386 + armv6/armv7) into one Universal final product.
			lipo -create "${DEVICE_DIR}/${FRAMEWORK_NAME}" "${SIMULATOR_DIR}/${FRAMEWORK_NAME}" -output "${INSTALL_DIR}/${FRAMEWORK_NAME}"
			
			open "${INSTALL_PATH}"
			
			# rm -r "${WORK_DIR}"
			```
			
7. 查看 framework 支持的架构

	```
	$ lipo -info [framework 二进制文件路径]
	```


## 三、framework 里封装 framework

[UmbrellaFramework（二）framework里封装framework](https://www.jianshu.com/p/50d69681d7ca)

1. 创建 Cocoa Touch Framework 工程 UmbrellaFramework

2. 导入 SubFramework

	![](http://dzliving.com/iOSUmbrellaFramework_4.png)
	
3. 选择 Target -> Build Phases -> 点击左上角+号 -> New Copy Files Phase 添加 Copy Files，将 SubFramework 添加到 Copy Files，选择 Destination 为 Frameworks。

	![](http://dzliving.com/iOSUmbrellaFramework_5.png)

4. 添加 UmbrellaSayHello 类，添加 sayHello 方法，并在 sayHello 方法中调用 SubFramework 的 sayHello 方法。

	```
	@interface UmbrellaSayHello : NSObject
	- (void)sayHello;
	@end

	
	#import <Subframework/SubSayHello.h>

	@implementation UmbrellaSayHello
	
	- (void)sayHello
	{
	    NSLog(@"%s", __func__);
	    
	    SubSayHello * ssh = [[SubSayHello alloc] init];
	    [ssh test];
	}
	
	@end
	```
	
5. UmbrellaFramework.h 头文件中导入将 UmbrellaSayHello.h
	
	```
	#import <UmbrellaFramework/UmbrellaSayHello.h>
	```

6. 将 UmbrellaSayHello.h 添加到 UmbrellaFramework 的公共 headers 中

7. Architectures 添加 armv7s

8. 连接选项 Mach-O Type 不用改，选择默认选项 Dynamic Library，这意味着外层的 UmbrellaFramework 是一个动态库。

9. 生成真机和模拟器都能用的 framework。见第二章。


## 四、使用 UmbrellaFramework

[UmbrellaFramework（三）使用demo](https://www.jianshu.com/p/895621ecfc32)

1. 创建工程 UmbrellaFrameworkDemo

2. 嵌入UmbrellaFramework，选择工程 -> General -> Embedded binaries，添加UmbrellaFramework。UmbrellaFramework 将会同时添加到 Linked Frameworks and Libraries.

	![](http://dzliving.com/iOSUmbrellaFramework_6.png)

3. 工程中使用

	```
	#import <UmbrellaFramework/UmbrellaFramework.h>

	- (void)viewDidLoad
	{
	    [super viewDidLoad];
	    
	    UmbrellaSayHello * ush = [[UmbrellaSayHello alloc] init];
	    [ush sayHello];
	}
	```

	