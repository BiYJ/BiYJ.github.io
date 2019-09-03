---
title: iOS 项目发布
category: iOS原理
---

## 一、Apple开发者账号

#### 1.1 开发者账号类型

- 个人级
- 公司级
- 企业级

公司和企业的可多人协作。

在苹果的开发者平台登录后，可在 ``People`` 界面邀请其他人员协作开发，邀请的人需要注册一个 appleId 账号，并且可以设置开发人员的权限。

![](http://dzliving.com/People.png)


#### 1.2 辨别账号类型

1. 个人级

	账户的 Apple Developer Program 下方只有 ``Certificates，identifiers & Profiles`` 和 ``App Store Connect`` 两个图标。

	![](http://dzliving.com/Personal.png)
	

2. 公司级

	账户的 Apple Developer Program 下方包含 ``People``、``Certificates，identifiers & Profiles`` 和 ``App Store Connect`` 三个图标。
	
	> 图标一：邀请其他开发人员的入口
	>
	> 图标二：开发者证书、App ID 和描述文件生成的入口
	>
	> 图标三：将 APP 上传到 App Store 的入口

	![](http://dzliving.com/Company.png)


3. 企业级

	账户的 Apple Developer <font color=#cc0000>Enterprise</font> Program 下方有``People``、``Certificates，identifiers & Profiles`` 两个图标，第一个图标邀请其他开发人员的入口，第二个图标是开发者证书、App ID和描述文件生成的入口。

	![](http://dzliving.com/Enterprise.png)

#### 1.3 账号对比

1. 个人（Individual）

	- 99 美元/年  
	- 只能上架到 App Store
	- 最大支持 100 台设备   
	- 1 人使用

	“个人”开发者可以申请<font color=#cc0000>升级</font>成“公司”，通过拨打苹果公司客服电话（400\-6701\-855）进行咨询办理。

2. 公司（Company）

	- 99美元/年
	- 只能上架到 App Store
	- 最大支持 100 台设备
	- 多人协作

	允许多个开发者进行协作开发，比个人账号多一些账号管理设置，可以设置多个 Apple ID，分为 4 种级别的权限。
	
	申请时需要填写公司的邓白氏码（DUNS Number）。
	
3. 企业（Enterprise）

	- 299美元/年
	- 不能上架到 App Store，只能企业内部使用。通过 item-services 分发来下载
	- 不限制设备数
	- 多人协作

	允许多个开发者进行协作开发，比个人账号多一些账号管理设置，可以设置多个 Apple ID，分为4种级别的权限。
	
	申请时需要填写公司的邓白氏码（DUNS Number）。
	
	只能企业内部使用，否则有被<font color=#cc0000>封号</font>的风险。
	
4. 总结
	
	- 个人级和公司级只能上架 App Store 供用户下载，企业级不能上架 App Store，只能放在自己的服务器或者三方平台；

		个人/公司级账号可以将<font color=#cc0000>开发环境</font>的包上传到自己的服务器，或者第三方平台。首先在开发者账号添加最多 100 个手机的 UUID，创建开发 developer 证书和描述文件，将它两下载到电脑上，双击安装，在 Xcode 选择开发证书打包。这种可调式（Debug）包的有效期也是一年，一年内需要重新创建证书，并更新 ipa 包，否则 APP 将闪退。注意：每次添加设备 UUID 需要重新生成描述文件。	

	- 企业级账号，每年至少重新打包 ipa 一次，更新 ipa 包中的证书信息，证书的有效期是一年，一年不更新，APP将闪退，无法进入。个人/公司级别的没有限制，只要成功上架到 App Store，如果没有特殊情况，理论可以放到苹果公司倒闭也不用去更新。

	- 安装企业级账号分发的包后，需要去设置中信任 APP，系统级限制，无法跳过。个人/公司级没有该限制。
	
	- 个人/公司必须经过苹果的人工审核才能在 App Store 上架，而企业级发布到自己服务器或者第三方平台是不需要审核的。


## 二、证书相关

#### 2.1 开发者证书

总共有两种类型：

- Developer（开发证书）
- Distribution（发布证书）

![](http://dzliving.com/Certificates.png)

不论是真机调试，还是上传到 appstore 都是需要的，是一个基证书，用来证明自己开发者身份的。

#### 2.2 App ID

> 是一个 APP 的唯一标示，相当于人的身份证号

<center>![](http://dzliving.com/Identifiers.png)</center>

- Description 是一个 App ID 的描述，可以随便
- App ID Prefix 苹果自动填了，可以看出，它是一个团队的 ID Suffix
- App ID Suffix 有两个选项：
	- Explicit App ID : 一个明确的 App ID，什么意思呢？可以这么来解释：我们做项目时的Bundle Identifier (Bundle ID)是用来标示我们的 app 的。我们的 App ID 也是用来标示我们的app 的。这两个有什么联系呢？Explicit App ID 就是要我们确定一个唯一的 Bundle ID，用来标示我们的 app，使它有一个<font color=#cc0000>固定的</font>身份。可以发现，当提交应用到苹果时，如果我们一开始没写 Explicit App ID，苹果会自动帮我们生成一个与我们发布时填的一样的 Bundle ID 到我们的 App ID 中。填写 Explicit App ID 的格式为：com.xxx.yyy。
	- Wildcard App ID : 一个通配符的 App ID。这个 App ID 能够在所有能够匹配的应用中使用。填写 WildcardApp ID 的格式为：com.xxx.*。

手机安装 app 时，会先查找是否已有相同 App ID 的 app，如果没有，则直接进行安装；如果已有，则进行覆盖安装。

#### 2.3 CSR

> 在苹果开发者中心生成证书时需要一个 CSR（证书签名请求）文件。

当创建 CSR 时，电脑系统实际上会生成公钥和私钥对。<font color=#cc0000>CSR 包含公钥</font>。

<center>![](http://dzliving.com/CSRContent.png)</center>

Apple 使用该公钥来制作证书。证书或多或少是一种发布公钥的方式以及关于该密钥的一些相关信息，并且验证发布实体，表示该信息是有效的。

由于每个证书都有自己的公钥 - 私钥对，因此推送证书、开发发证书和发布证书都可以使用不同的 CSR，每个证书用于验证和保护该特定项目。

生成 CSR 文件步骤：

<center>
	![](http://dzliving.com/KeyAccess.png)
	![](http://dzliving.com/CSR.png)
	![](http://dzliving.com/CSR2.png)
</center>

1. 用户电子邮件地址随便填写，并不一定非要填写邮件格式，无实际作用
2. 常用名字使用默认的就可以，也可以修改
3. 选择存储到磁盘。
4. 选择继续，保存到指定位置即可。最终得到一个 ``CertificateSigningRequest.certSigningRequest`` 文件，也就是 CSR 文件。该文件后缀名不要更改，文件名可自由指定。

CSR文件尽量每个证书都制作一次，将常用名称区分开来，因为该常用名称是证书中的密钥的名字。

<center>![](http://dzliving.com/P12CSR.png)</center>

#### 2.4 .cer

> 在苹果开发者中心生成的证书，导出后就是 .cer 文件。.cer 证书仅包含公钥。

<center>![](http://dzliving.com/CreateCer.png)</center>

提交 CSR 文件后就会生成一个 cer 证书，有效期为一年。

证书不可轻易删除，可能会造成相应的 Provisioning Profiles 失效，尤其是企业级的发布证书，删除后已经安装在手机上的 APP 将会闪退。如果是一个团队大家都在用，把这个证书删除了，会导致别人的描述文件失效。


#### 2.5 .p12

> .p12证书可能既包含公钥也包含私钥。

双击安装下载后的 .cer 文件，都可以在``钥匙串访问``工具中导出 .p12 文件。

为什么要导出.p12文件？

开发证书和发布证书是有数量限制的，当超过数量后再也申请不了。

多台电脑开发程序时，没必要生成很多的证书，导出证书生成的 .p12 文件，供给所有的 mac 设备使用，使新设备不需要在苹果开发者网站重新申请开发和发布证书，就能使用。

点击 .p12 文件加入钥匙串中，使我们的电脑<font color=#cc0000>具备开发的证明</font>。

注意：一般 .p12 文件是给别人使用的，本机必须已经有一个带秘钥的证书才可以生成.p12文件。

#### 2.6 描述文件

> Profiles 将 App ID、开发者证书、硬件 Device 绑定到一块儿。

<center>![](http://dzliving.com/Mobile.png)</center>

在开发者中心配置好后可以添加到 Xcode 上，也可以直接在 Xcode 上连接开发者中心生成，真机调试时需要在描述文件中添加真机的 UDID。

#### 2.7 邓白氏码

> 邓氏编码（D-U-N-S® Number，是 Data Universal Numbering System的缩写）。它是一个独一无二的 9 位数字全球编码系统，相当于企业的身份识别码，被广泛应用于企业识别、商业信息的组织及整理。可以帮助识别和迅速定位全球 2.4 亿家企业的信息。


## 三、打包上传

#### 3.1 payload

适用于开发阶段。操作步骤：

1. 选中 target -> edit Scheme，修改 run 操作的 Build Configuration 是 budeg/release。

	<center>![](http://dzliving.com/SchemeRun.png)</center>

2. 真机运行工程，在左侧工程目录 -> Products 找到 .app 文件，show in Finder。

	<center>![](http://dzliving.com/DemoApp.png)</center>

3. 桌面生成一个名为 Payload 的空文件夹，将 .app 文件拖入其中。

4. 压缩 Payload 文件夹，修改后缀名为 .ipa，此时就可以将包上传 fir.im 网站，进行提测。

	<center>![](http://dzliving.com/Payload.png)</center>


#### 3.2 Archive

1. 常规打包方式

	运行设备选择 ``Generic iOS Device`` 或者真机，顶部 Xcode 导航栏 -> Product -> Archive。

2. 命令行使用 xcodebuild 自动打包

	①、clean 工程
	
	```c
	$ cd Desktop/CYKJ/CYKJMain/CYKJMain/
	
	$ xcodebuild clean -project CYKJMain.xcodeproj -scheme CC -configuration Release
	
	$ xcodebuild clean -workspace CYKJMain.xcworkspace -scheme CC -configuration Release
	```
	如果工程使用了 pod，选择 ``-workspace``，否则使用 ``-project``。
	
	上面的命令中：
	
	> \-project CYKJMain.xcodeproj : 编译项目名称  
	>
	> \-workspace CYKJMain.xcworkspace : 编译工作空间名称  
	>
	> \-scheme CC : scheme 名称
	> 
	> \-configuration Release : (Debug/Release)

	clean 成功：
	
	<center>![](http://dzliving.com/XCodeBuildClean.png)</center>
	
	②、archive 导出 .xcarchive 文件
	
	```
	$ xcodebuild archive -project CYKJMain.xcodeproj -scheme CC -archivePath /Users/cykj/Desktop/cc.xcarchive

	$ xcodebuild archive -workspace CYKJMain.xcworkspace -scheme CC -archivePath /Users/cykj/Desktop/cc.xcarchive
	```
	上面的命令中：
	
	> \-archivePath /Users/cykj/Desktop/cc.xcarchive : 导出 .xcarchive 文件的目录以及文件名称

	archive 成功：

	<center>
		![](http://dzliving.com/XCodeBuildArchive.png)
		![](http://dzliving.com/CCArchive.png)	
	</center>

	③、导出 ipa 包

	```c
	$ xcodebuild -exportArchive -archivePath /Users/cykj/Desktop/cc.xcarchive -exportPath /Users/cykj/Desktop/cc -exportFormat ipa -exportProvisioningProfile "developmentProfile"
	```
	
	上面的命令中:  
	
	> -archivePath /Users/cykj/Desktop/cc.xcarchive : 刚刚导出的 .xcarchive 文件的目录  
	>
	> -exportPath /Users/cykj/Desktop/cc : 将要导出的 ipa 文件的目录以及文件名
	>
	> -exportFormat ipa : 导出为ipa文件
	>
	> -exportProvisioningProfile "developmentProfile" : 工程配置的 profile 文件的名称
	
	<font color=#cc0000>``xcodebuild: error: invalid option ‘-exportFormat’``</font>
	
	报错原因：Xcode8 之后，对之前的 exportFormat 方式不再支持。
	
	解决方法：不使用 exportFormat ipa 引入 ``exportOptionsPlist``。

	```
	$ xcodebuild -exportArchive -archivePath "${ARCHIVEPATH}/${TARGET_NAME}.xcarchive" -exportPath ${EXPORTPATH} -exportOptionsPlist ${EXPORTOPTIONSPLIST}
	```
	
	直接使用 xcodebuild -help 查看 exportOptionsPlist。
	
	```
	Available keys for -exportOptionsPlist:

	compileBitcode : Bool

   		For non-App Store exports, should Xcode re-compile the app from bitcode? Defaults to YES.

	embedOnDemandResourcesAssetPacksInBundle : Bool

		For non-App Store exports, if the app uses On Demand Resources and this is YES, asset packs are embedded in the app bundle so that the app can be tested without a server to host asset packs. Defaults to YES unless onDemandResourcesAssetPacksBaseURL is specified.

	iCloudContainerEnvironment

		For non-App Store exports, if the app is using CloudKit, this configures the "com.apple.developer.icloud-container-environment" entitlement. Available options: Development and Production. Defaults to Development.

	manifest : Dictionary

		For non-App Store exports, users can download your app over the web by opening your distribution manifest file in a web browser. To generate a distribution manifest, the value of this key should be a dictionary with three sub-keys: appURL, displayImageURL, fullSizeImageURL. The additional sub-key assetPackManifestURL is required when using on demand resources.

	method : String

		Describes how Xcode should export the archive. Available options: app-store, ad-hoc, package, enterprise, development, and developer-id. The list of options varies based on the type of archive. Defaults to development.

	onDemandResourcesAssetPacksBaseURL : String

		For non-App Store exports, if the app uses On Demand Resources and embedOnDemandResourcesAssetPacksInBundle isn't YES, this should be a base URL specifying where asset packs are going to be hosted. This configures the app to download asset packs from the specified URL.

	teamID : String

		The Developer Portal team to use for this export. Defaults to the team used to build the archive.

	thinning : String

		For non-App Store exports, should Xcode thin the package for one or more device variants? Available options: <none> (Xcode produces a non-thinned universal app), <thin-for-all-variants> (Xcode produces a universal app and all available thinned variants), or a model identifier for a specific device (e.g. "iPhone7,1"). Defaults to <none>.

	uploadBitcode : Bool

		For App Store exports, should the package include bitcode? Defaults to YES.

	uploadSymbols : Bool

		For App Store exports, should the package include symbols? Defaults to YES.
	```
	
	plist 格式：

	```
	<?xml version="1.0" encoding="UTF-8"?>
	<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
	<plist version="1.0">
	<dict>
		<key>teamID</key>
			<string>xxxx</string>
		<key>method</key>
			<string>app-store</string>
		<key>compileBitcode</key>
			<false/>
		<key>provisioningProfiles</key>
			<dict>
				<key>com.xxx.yyy</key>
					<string>{iOS Provisioning Profiles Name}</string>
			</dict>
	</dict>
	</plist>
	```
	
	命令行：
	
	```
	$ xcodebuild -exportArchive -exportOptionsPlist /Users/cykj/Desktop/cc.plist -archivePath /Users/cykj/Desktop/cc.xcarchive -exportPath /Users/cykj/Desktop/cc
	```
	
	导出 ipa 成功：
	
	<center>![](http://dzliving.com/XCodeBuildIPA.png)</center>
	
	<font color=#cc0000>``Error Domain=IDEProfileLocatorErrorDomain Code=4 "No "iOS App Development"``</font>
	
	问题原因：plist 文件中提供的 mehtod 的 value 不对。

#### 3.3 Application Loader

当已经生成了 ipa 包时，可以通过 Application Loader 将包 upload 至 AppStore，操作步骤如下：

1. 登录 Application Loader。路径：Xcode -> Open Developer Tool -> Application Loader
	
	<center>
	![](http://dzliving.com/ApplicationLoaderLogin.png)
	</center>
	
	这里需要注意的是，密码不是与 Apple ID 对应的用于登录 AppStore 的密码，而是 <font color=#cc0000>``App 专用密码``</font>，获取地址：[https://appleid.apple.com/account/manage](https://appleid.apple.com/account/manage)
	
	<center>
	![](http://dzliving.com/ApplicationLoaderPassword.png)
	</center>

2. 上传 ipa 包

	登录成功后，需要导入 ipa 包，经过 Application Loader 的检查之后，即可上传 AppStore。

	<center>
	![](http://dzliving.com/ApplicationLoaderIPA.png)
	</center>

3. 报错

	<center>
		![](http://dzliving.com/ApplicationLoaderError.png)
	</center>
	
	bundle = 209 的包已经在 itunes connect 上有了，新上传的包需要在此 bundle 号的基础上增加，然后重新 upload。

## 四、打包审核注意

1. 检查 Version
2. 修改 BundleID
3. 修改 Build 号
4. 修改 pch 文件及其他宏定义。可以处理成代码自动根据 BundleID 识别当前为开发环境/发布环境
5. 修改证书及描述文件为发布环境
6. 询问设计人员是否要更换启动图、应用图标、市场图


## 五、Apple Store Connect

#### 5.1 新建 APP

<center>
![](http://dzliving.com/CreateApp.png)
</center>

* 名称
	
	<center>
	![](http://dzliving.com/AppStoreName.png)
	</center>
	
* 套装 ID：即 Bundle ID，显示在开发者中心中创建好 App ID。

	<center>
	![](http://dzliving.com/TaoZhuangID.png)
	</center>
	
* SKU：与 Bundle ID 一样即可

注意：这些信息在创建好 App 后是不能修改的，需要修改的话，只能新建一个 App 替代。[点击更多阅读](https://www.jianshu.com/p/791ca505e3d1)

#### 5.2 添加 App 版本

填写信息：

* 此版本的新增内容。<font color=#cc0000>如果当前是第一个上线版本，则没有这一项。</font>

	<center>
	![](http://dzliving.com/AppNewVersion.png)
	</center>
	
* App 预览和屏幕快照。每个版本发布时，记得询问设计人员市场图是否有更新。
	
	<center>
	![](http://dzliving.com/AppScreenShot.png)
	</center>

* 关键词。在市场中搜索时有用，可以做优化。

	<center>
	![](http://dzliving.com/AppKeyword.png)
	</center>
	
* 技术支持网址。可以填写公司官网地址。

	<center>
	![](http://dzliving.com/AppSupportURL.png)
	</center>

* 描述。对 app 进行说明，可以包括公司概述、功能介绍、联系方式等。

	<center>
	![](http://dzliving.com/AppDesc.png)
	</center>
	
* 版权

* 商务代表联系信息

* App 审核信息。


#### 5.3 导入此构建版本时出错

upload 包后，在 itunes connect 中等待处理后，查看发现报错：

<center>
![](http://dzliving.com/UploadIpaError.png)
</center>

报错后，着急去排查工程里面是否有配置问题，排查一圈之后，准备重新打包 upload。就在这时，刷新 itunes connect 网页看一下状态，发现报错的已经正常了，直接用那个包提交，不用重新打包了。[点击更多阅读](https://www.jianshu.com/p/b3f024d9fd81)

#### 5.4 IDFA

提交审核时，IDFA 选择“否”，报错：

<center>
![](http://dzliving.com/IDFA.jpg)
</center>

百度查找文章发现原因

<center>
![](http://dzliving.com/IDFASeach.png)
</center>

着手排查工程中是否使用了 IDFA 并引入 AdSupport.framework，如果有则移除。终端使用命令：

```
$ cd 项目目录
$ grep -r advertisingIdentifier .
```

用这条命令检测自己的工程，如果没有查到相关引用，那么就不要勾选使用 IDFA，如果查到了相关引用，并且这些文件是用于展现广告的用途，那么勾选使用了 IDFA。

<center>
![](http://dzliving.com/IDFAGrep.jpg)
</center>

log 显示极光中有使用。百度搜索“极光 IDFA”问题，跳转到极光社区，找到文章：[iOS审核时需要勾选IDFA吗？](https://community.jiguang.cn/t/ios-idfa/13099)

<center>
![](http://dzliving.com/JiGuangIDFA.png)
</center>

根据官方人员的说明，使用不带用 advertisingIdentifier 字段的方法。兴致勃勃的重新打包，等待了 10-20 分钟左右的时间，重新提交审核，IDFA 选择“否”，依然报错。

已经删除了工程中导入的 AdSupport.framework，不应该啊。

再次进行排查。将 ipa 包导出到本地，修改后缀名为 <font color=#cc0000>.zip</font>，解压后使用终端命令：

```
$ cd ipa解压后的文件目录
$ grep -r AdSupport.framework .
```

<center>
![](http://dzliving.com/AdSupportGrep.jpg)
</center>

log 显示百度统计 sdk 中导入了 AdSupport.framework，百度搜索后跳转官方网站：[iOS SDK采集IDFA注意事项](https://mtj.baidu.com/web/help/article?id=286&type=0)

<center>
![](http://dzliving.com/BaiduAdSupport.png)
</center>

百度官方的意思是设置 IDFA 为 YES 并勾选。

考虑到上线紧迫，最后决策：<font color=#cc0000>移除百度统计相关代码，打包 upload</font>。


## 六、文章

[Benjamin丶](https://www.jianshu.com/u/fd1e679c3ac1) & [Apple开发者账号介绍及证书配置说明](https://www.jianshu.com/p/8190cf4a8172)
[PersonChen_QJ](https://www.jianshu.com/u/80f9260b9fb7) & [iOS申请邓白氏编码图文流程](https://www.jianshu.com/p/da6663c8fd5f)
[用于创建开发人员或分发证书和推送证书的CSR（证书签名请求）文件是否必须相同？](https://cloud.tencent.com/developer/ask/205801)
[2019年最新苹果企业开发者账号创建证书完整流程](http://www.sohu.com/a/324460196_120174355)
[MrCoderLin](https://www.jianshu.com/u/bb0d2c0a0880) & [Payload文件压缩法打包ipa](https://www.jianshu.com/p/87a1eba7caeb)
[咖啡绿茶1991](https://www.jianshu.com/u/ced5bf319bfe) & [iOS命令行自动打包(archive)](https://www.jianshu.com/p/347056c3f49c)
[iOS自动化打包](https://blog.csdn.net/zhangxiweicaochen/article/details/72730355)
[Xcode9 xcodebuild 命令行打包遇到的坑与解决方案](https://blog.csdn.net/yuanmengong886/article/details/78214978)
[一键打包完整Shell脚本xcodebuild archive](https://www.jianshu.com/p/36d2c6d65aa7)
[iOS 如何填App Store Connect信息](https://www.jianshu.com/p/1c9ad924c79b)
[iOS提交审核：您的 App 正在使用广告标识符 (IDFA)](https://www.jianshu.com/p/56892880e003)