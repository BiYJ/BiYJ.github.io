---
title: iOS 应用签名
categories: iOS 原理


## 一、密码学简介

#### 1.1 base64

* Base64 是一种通过查表的编码方法，不能用于加密，即使使用自定义的编码表也不行。
* Base64 适用于小段内容的编码，比如数字证书签名、Cookie 的内容等。
* 由于 = 字符也可能出现在 Base64 编码中，但 = 用在 URL、Cookie 里面会造成歧义，所以，很多Base64编码后会把 = 去掉，解码时，需要加上 = 把 Base64 字符串的长度变为 4 的倍数，再进行解码。


使用 mac 自带的 base64 编码，格式：`base64 [文件路径] -o [保存文件路径]`

```
$ base64 a.png
$ base64 a.png -o a.txt
```

iOS 代码编写：

```
NSData * data = [@"A" dataUsingEncoding:NSUTF8StringEncoding];

// base64 编码
NSLog(@"%@", [data base64EncodedStringWithOptions:NSDataBase64Encoding64CharacterLineLength]);

// base64 解码
NSData * newData = [[NSData alloc] initWithBase64EncodedString:@"QQ==" options:NSDataBase64DecodingIgnoreUnknownCharacters];
NSLog(@"%@", [[NSString alloc] initWithData:newData encoding:NSUTF8StringEncoding]);

QQ==
A
```

base64 对照表

![](http://dzliving.com/Base64_0.jpeg)

#### 1.2 hash

hash 又称为散列函数，在 iOS 中使用到的 hash 算法有：

* MD5
* SHA1/256/512
* HMAC

这些算法都具有以下特点：

* 算法都是公开的；
* 对相同的数据进行加密，得到的结果是相同的；
* 对不同的数据进行加密，得到的结果是定长的。如：MD5 32 个字符；
* 不能反算。所以常用来做信息摘要，信息“指纹”，用来做信息识别。

1. md5 

	```
	$ md5 -s "12345678"
	MD5 ("12345678") = 25d55ad283aa400af464c76d713c07ad
	```
	
	应用场景：
	
	> 用户登录应用时，注册填入的密码，经过 hash 算法加密后，传给服务器，服务器只存储 hash 值。当用户再次登录时，只需要验证登录密码的 hash 值与服务器保存的是否相同。
	
	上面的应用场景足够安全吗？来看下图（网址：[https://www.cmd5.com/](https://www.cmd5.com/)）
	
	<center>
	![](http://dzliving.com/iOSSafe_0.png)
	</center>
	
	由于相同的数据得到的 hash 值是相同的，当记住很多的原始数据和 hash 值的对应关系之后，是可以<font color=#cc0000>查询</font>出来原始数据的。注意：这里是查询而不是反算。
	
	要想更加安全一些，可以采取策略：“加盐”。
	
	>加盐：在原始数据中加入一段额外的字符串，降低查询成功率。
	
	```
	$ md5 -s "12345678&&%#**#7*&@&#W#^E"
	MD5 ("12345678&&%#**#7*&@&#W#^E") = 28e2829463d71bb8931eac4517436a2e
	```
	
	<center>
	![](http://dzliving.com/iOSSafe_1.png)
	</center>
	
	加盐之后是否就能高枕无忧呢？显然不能，如果开发者将盐固定写死在 app，是有可能泄露的，应用的安全性也就随之降低。
	
	在此基础上，引申出动态“加盐”的 HMAC 算法。

2. HMAC

	> 使用 KEY 进行明文加密，然后进行两次 Hash 算法。Key 是服务器随机生成的，对应的是一个账号一个 Key。
	
	HMAC 是一种加密策略，两次使用的 hash 算法可以 MD5，也可以是 SHa.


#### 1.3 对称加密算法

* DES 加密算法简单，基本不使用；
* 3DES 执行 3 次 DES 算法，基本不使用。
* AES（高级密码标准），iOS中的钥匙串访问就是使用 AES

算法的特点：<font color=#cc0000>可以反算</font>。

1. 加解密过程

	* 明文 -> 密钥加密 -> 密文
	* 密文 -> 密钥解密 -> 明文

2. 加密方式

	* EBC

		电子代码本，每一个数据块进行独立加密。
	
		![](http://dzliving.com/iOSSafe_2.png)
		
		a.txt 文件原始数据：

		```
		11111111111111
		11111111111111
		11111111111111
		11111111111111
		```
		
		终端执行命令：
		
		```
		$ openssl enc -des-ecb -K 616263 -nosalt -in a.txt -out b.bin
		$ xxd b.bin
		```
		
		上面采用 `ecb` 算法，`-nosalt` 不需要加盐，`616263` 是 abc 的 ASCII 码，`.bin` 是二进制文件，`xxd` 查看二进制文件。
		
		a.txt 文件修改后数据：
		
		```
 		11111111111111
		11111111111111
		11111111111100
		11111111111111
		```
		
		终端执行同样的命令后，可以看到差别，如图：
		
		![](http://dzliving.com/iOSSafe_5.png)		
	* CBC

		密码块链，使用一个密钥和一个初始化向量对数据进行加密。
		
		![](http://dzliving.com/iOSSafe_3.png)
		
		这种加密方式可以有效的保证数据的完整性。也就是说，如果有一个数据块丢失，就会导致后面所有的数据块都无法解密。一般这种技术用于防范窃听。
		
		```
		$ openssl enc -des-cbc -iv 0102030405060708 -K 616263 -nosalt -in a.txt -out b.bin
		$ xxd b.bin
		```
		
		![](http://dzliving.com/iOSSafe_6.png)		
	由上可以看出，EBC 加密方式修改数据后，只影响一个数据块的加密结果；CBC 则影响之后的数据块。

		


#### 1.4 非对称加密算法

[RSA算法](https://baike。baidu。com/item/RSA%E7%AE%97%E6%B3%95/263310?fr=aladdin)
[RSA - 原理、特点（加解密及签名验签）及公钥和私钥的生成](https://blog。csdn。net/kikajack/article/details/80703894)

* RSA 是非对称加密算法，算法是公开的；
* 公钥加密数据，私钥解密数据；
* 私钥加密数据，公钥解密数据；
* 公钥可以有多个，私钥只要一个。

以下进行 openssl 终端演示

1. 常用指令

	* genrsa 生成并输入一个 RSA 私钥
	* rsautl 使用 RSA 密钥进行加密、解密、签名和验证等运算
	* rsa 处理 RSA 密钥的格式转换等问题
	
2. 加密过程

	* 生成 RSA 私钥

		```
		$ openssl genrsa -out private.pem 1024
		Generating RSA private key， 1024 bit long modulus
		。。。。。。。。。。。。。。。。。。。。。++++++
		。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。++++++
		e is 65537 (0x10001)
		```
	
		openssl 默认生成 512 位的私钥，上面指定的是 1024 位的。需要注意的是，愈长的私钥被破解的机率愈低，但是相对地，我们在使用加密与解密的时间也会愈长。使用者可以自行评量。
	
	* 查看 RSA 私钥

		```
		$ cat private.pem
		-----BEGIN RSA PRIVATE KEY-----
		MIICXAIBAAKBgQDQNiXkM7Sw3EwCHp1TPEFvKLe5kP7MWr+N/mf4NDlgEZa08xIT
		htAI7T7I/e6rW6Ho7HAgbucCb/cx1GEx+kH0ldRCKZ7NxS4Ueo9YjqArbsRlQDx5
		w5cVj58g5E1ZuIvR600uXWYZXAK9LS65HQoWZiWtMmuy4qPW5NtgdALQLwIDAQAB
		AoGAEby7J6CcAQETXI7dGi0k6eJPHHjUq/YDIYaNtuBEDwIQk6OtY4p1iH0lfxva
		zDBHL7+Mocaw2U1Ogqk0CnzmR1drO+ubpRRI15KKYkuS918xXqlPNPszowXns7z8
		THjpjeLgWfRAcuvVs7bHPu4C/b2smXwNBptTAtpIS2/6VSECQQDvoHA8cUgL+8Nn
		ovPMhcsK7NXoZ/gnSWh3LqECK9JIvDlkQao8mzd8SpZBf5Xya9esqCUYib6gzvlG
		/RxpUpG/AkEA3nAxicmTe5K1EBUed3zH1ALxquwy/7s1tsi87brvDlNAGi2MFDXO
		AZ9R9H1mVHxqnyekCrpCvgUv6QPsoex9kQJAedzc11BA9J8vy9fKJrvv+3lge5XM
		VKZ3cw0KouEISyc2BK+EVNgXCqWf7mVlK2j+wPauDuGWSY+YpCp6tXFhXwJBAMPQ
		pntyvWdybfx7averHErSUKa0Ce1Ag/el3VO2VU4aEXs6D2+XMgQRmdcOMXA8mqwC
		/JEJCUo4TMXnU3/0LVECQHKC4ARiTNGtCf0e+HZjT8XanqHR+O6QUMZ90m4aQGRg
		VaBLs5uNFf945FyyPpvQSJFqTZ3j17Go8dRpgd5k0q0=
		-----END RSA PRIVATE KEY-----
		```
	
	* 转成明文并查看
	
		```
		$ openssl rsa -in private.pem -text -out private.txt
		writing RSA key
		
		$ cat private.txt
		Private-Key: (1024 bit)
		modulus:
		    00:d0:36:25:e4:33:b4:b0:dc:4c:02:1e:9d:53:3c:
		    41:6f:28:b7:b9:90:fe:cc:5a:bf:8d:fe:67:f8:34:
		    39:60:11:96:b4:f3:12:13:86:d0:08:ed:3e:c8:fd:
		    ee:ab:5b:a1:e8:ec:70:20:6e:e7:02:6f:f7:31:d4:
		    61:31:fa:41:f4:95:d4:42:29:9e:cd:c5:2e:14:7a:
		    8f:58:8e:a0:2b:6e:c4:65:40:3c:79:c3:97:15:8f:
		    9f:20:e4:4d:59:b8:8b:d1:eb:4d:2e:5d:66:19:5c:
		    02:bd:2d:2e:b9:1d:0a:16:66:25:ad:32:6b:b2:e2:
		    a3:d6:e4:db:60:74:02:d0:2f
		publicExponent: 65537 (0x10001)
		privateExponent:
		    11:bc:bb:27:a0:9c:01:01:13:5c:8e:dd:1a:2d:24:
		```
		
	* 从私钥中提取公钥

		```
		$ openssl rsa -in private.pem -pubout -out public.pem
		writing RSA key
		
		$ cat public.pem
		-----BEGIN PUBLIC KEY-----
		MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDQNiXkM7Sw3EwCHp1TPEFvKLe5
		kP7MWr+N/mf4NDlgEZa08xIThtAI7T7I/e6rW6Ho7HAgbucCb/cx1GEx+kH0ldRC
		KZ7NxS4Ueo9YjqArbsRlQDx5w5cVj58g5E1ZuIvR600uXWYZXAK9LS65HQoWZiWt
		Mmuy4qPW5NtgdALQLwIDAQAB
		-----END PUBLIC KEY-----
		```
	
	* 通过公钥加密数据

		```
		$ openssl rsautl -encrypt -in a.txt -inkey public.pem -pubin -out enc.txt
		
		$ cat enc.txt
		1��，~��~Dn�P�Q�
		               ]X
		��wa�؆���ޅ��T���sJ��z���，l쩨�l��，Y�i�����vD9^Y��)D�:�
		```
		
		RSA 非对称式加解演算法因为先天的限制，无法加密过大的档案，若遇到这个问题时，OpenSSL 会输出如下的错误讯息
		
		```
		RSA operation error
140736003920840:error:0406D06E:rsa routines:RSA_padding_add_PKCS1_type_2:data too large for key size:/BuildRoot/Library/Caches/com。apple。xbs/Sources/libressl/libressl-22.50。3/libressl/crypto/rsa/rsa_pk1.c:158:
		```
		
	* 通过私钥解密数据

		```
		$ openssl rsautl -decrypt -in enc.txt -inkey private.pem -out dec.txt
		$ cat dec.txt
		11111111111111
		11111111111111
		11111111111111
		11111111111111
		```
		
	* 通过私钥加密数据

		```
		$ openssl rsautl -sign -in dec.txt -inkey private.pem -out enc2.txt
		$ cat enc2.txt
		]n�&h`�，ܪ8ۭ��`��X�fL8。�^����G
		\��3��P����t�Xj���&�f��_7�K�[2wK7@ʼ�_�Qh�N
		                                          �x��。
		                                               �uɮ�i����@����ݛ���ɧ]l�
		```
		
	* 通过公钥解密数据

		```
		$ openssl rsautl -verify -in enc2.txt -inkey public.pem -pubin -out dec2.txt
		$ cat dec2.txt
		11111111111111
		11111111111111
		11111111111111
		11111111111111
		```

3. 加密特点

	* 加密的安全性非常高，加密解密使用不一样的密钥；
	* 运算的效率非常低，一般只加密小数据，用于数字签名。


## 二、数字签名

数字信息的识别码。

1. 用户在支付宝付费 10 元

	![](http://dzliving.com/iOSSafe_7.png)
	
2. 直接传递数据是不安全的，黑客可以随意将“消费 10 元”改成“消费 100 元”，支付宝无法验证用户的数据是否被篡改，考虑后，决定将“消费 10 元”的数据连同它的 hash 值一起发送给服务器，服务器端通过对数据进行 hash 运算，与携带的 hash 值进行比较，相同则没有被篡改过。

	![](http://dzliving.com/iOSSafe_8.png)

3. 以为这样就安全了吗？黑客通过中间拦截，将“消费 10 元”及其 hash 值一并修改。

	![](http://dzliving.com/iOSSafe_9.png)
	
4. 有没有方式可以让 hash 值无法被修改呢？答案是对 hash 值数据进行非对称（RSA）加密，这就是数字签名。

	![](http://dzliving.com/iOSSafe_10.png)


## 三、代码签名

>代码签名(Code signing)是对可执行文件或脚本进行数字签名，用来确认软件在签名后未被修改或损坏的措施。和数字签名原理一样，只不过签名的数据是代码。

#### 3.1 简单的代码签名

在 iOS 出来之前，以前的主流操作系统(Mac/Windows)软件随便从哪里下载都能运行，系统安全存在隐患，盗版软件，病毒入侵，静默安装等等。那么苹果希望解决这样的问题，要保证每一个安装到 iOS 上的 APP 都是经过苹果官方允许的，怎样保证呢？就是通过代码签名。

如果要实现验证，其实最简单的方式就是通过苹果官方生成非对称加密的一对公私钥。在 iOS 的系统中内置一个公钥，私钥由苹果后台保存，我们传 APP 到 AppStore 时，苹果后台用私钥对 App 数据进行签名，iOS 系统下载这个 APP 后，用公钥验证这个签名，若签名正确，这个 App 肯定是由苹果后台认证的，并且没有被修改过，也就达到了苹果的需求：保证安装的每一个 App 都是经过苹果官方允许的。

如果我们 iOS 设备安装 App 只从 App Store 这一个入口这件事就简单解决了，没有任何复杂的东西，一个数字签名搞定。但是实际上 iOS 安装 App 还有其他渠道：比如在开发 App 时直接真机调试的，而且苹果还开放了企业内部分发的渠道，企业证书签名的 App 也是需要顺利安装的。

苹果需要开放这些方式安装APP，这些需求就无法通过简单的代码签名来办到了。那么我们来分析一下，它有些什么需求。

1. 安装包不需要上传到 App Store，可以直接安装到手机上；
2. 苹果为了保证系统的安全性，又必须对安装的 App 有绝对的控制权
	* 经过苹果允许才可以安装
	* 不能被滥用，导致非开发 App 也能被安装

为了实现这些需求，iOS 签名的复杂度也就开始增加了，苹果这里给出的方案是双层签名。

#### 3.2 iOS 的双层代码签名

iOS 的双层代码签名流程这里简单梳理一下，这也不是最终的 iOS 签名原理。iOS 的最终签名在这个基础上还要稍微加点东西，文末会讲。

首先这里有两个角色：

1. iOS 系统
2. Mac 系统

因为 iOS 的 App 开发环境在 Mac 系统下，所以这个依赖关系成为了苹果双层签名的基础。

1. 在 Mac 系统中生成非对称加密算法的一对公钥/私钥(Xcode 代办了)。这里称为公钥 M、私钥 M。

	<center>
	![](http://dzliving.com/iOSSafe_11.png)
	</center>

2. 苹果自己有固定的一对公私钥，跟之前 App Store 原理一样，私钥在苹果后台，公钥在每个 iOS 系统中。这里称为公钥 A，私钥 a.

	<center>
	![](http://dzliving.com/iOSSafe_12.png)
	</center>

3. 把公钥 M 以及一些你开发者的信息，传到苹果后台(这个就是 CSR 文件)，用苹果后台里的私钥 A 去签名公钥 M。得到一份数据包含了公钥 M 以及其签名，把这份数据称为<font color=#cc0000>证书</font>。

	<center>
	![](http://dzliving.com/iOSSafe_13.png)
	</center>

4. 在开发时，编译完一个 APP 后，用本地的私钥 M（今后你导出的P12) 对这个 APP 进行签名，同时把第三步得到的证书一起打包进 APP 里，安装到手机上。

	<center>
	![](http://dzliving.com/iOSSafe_14.png)
	</center>
	
5. 在安装时，iOS 系统取得证书，通过系统内置的公钥 A，去验证证书的数字签名是否正确。

6. 验证证书后确保了公钥 M 是苹果认证过的，再用公钥 M 去验证 APP 的签名，这里就间接验证了这个 APP 安装行为是否经过苹果官方允许。（这里只验证安装行为，不验证 APP 是否被改动，因为开发阶段 APP 内容总是不断变化的，苹果不需要管。）

	<center>
	![](http://dzliving.com/iOSSafe_15.png)
	</center>

有了上面的过程，已经可以保证开发者的认证，和程序的安全性了。 但是，你要知道 iOS 的程序，主要渠道是要通过 APP Store 才能分发到用户设备的，如果只有上述的过程，那岂不是只要申请了一个证书，就可以安装到所有 iOS 设备了？那么为了防止滥用，苹果再加了几个限制。


## 四、描述文件

应用签名方式并不能解决应用滥用的问题，所以苹果又加了两个限制：

1. 在苹果后台注册过的设备才可以安装
2. 签名只能针对某一个具体的 App

并且苹果还想控制 App 里面的 iCloud/PUSH/后台运行/调试器附加这些权限，所以苹果把这些权限开关统一称为 Entitlements(授权文件)。并将这个文件放在了一个叫做 Provisioning Profile（描述文件）文件中。描述文件是在 AppleDevelop 网站创建的（在 Xcode 中填上 AppleID 它会代办创建)，Xcode 运行时会打包进入 APP 内。

所以我们使用 CSR 申请证书时，我们还要申请描述文件。流程如下：

<center>
![](http://dzliving.com/iOSSafe_16.png)
</center>

这个描述文件里面就是描述：

1. 可以安装的设备有哪些
2. APP 的 ID 是什么
3. 权限有哪些

在开发时，编译完一个 APP 后，用本地的私钥 M 对这个 APP 进行签名，同时把从苹果服务器得到的  Provisioning Profile 文件打包进 APP 里，文件名为 embedded.mobileprovision，把 APP 安装到手机上。

<center>
![](http://dzliving.com/iOSSafe_17.png)
</center>

我们可以利用 `$security cms -D -i embedded.mobileprovision` 命令查看 Provisioning profile 内容，这些 Xcode 创建的 Profile 文件都存放在 `~/Library/MobileDevice/Provisioning Profiles/` 目录下

<center>
![](http://dzliving.com/iOSSafe_18.png)
</center>

注意：每次我们新建项目其实会生成一个描述文件，选择运行到手机上，我们只需要编译一下，在 App 包里面就可以看到。

<center>
![](http://dzliving.com/iOSSafe_19.png)
</center>

那么为了便于我们查看信息! 我们可以通过 Xcode 来查看!!

<center>
![](http://dzliving.com/iOSSafe_20.png)
</center>

当然，Provisioning profile 本身也是通过签名认证的，所以别想着你可以更改里面的东西来达到扩充权限或者设备的目的。只有老老实实的去网站向 Apple 申请一份更多权限、设备的 profile。


## 五、整体的流程

首先总结一下刚才的一些名词

* 证书：内容是公钥或者私钥，由认证机构对其签名组成的数据包。可以使用钥匙串访问看到

	<center>
	![](http://dzliving.com/iOSSafe_21.png)
	</center>

* P12：就是本地私钥，可以导入到其他电脑
* Entitlements：权限文件，包含了 App 一些权限的 plist 文件
* CertificateSigningRequest：CSR 文件包含了本地公钥的数据文件
* Provisioning Profile：描述文件，包含了证书、Entitlements 等数据，并由苹果后台私钥签名的数据包。

流程如下:

1. keychain 里的 “从证书颁发机构请求证书”，这里就本地生成了一对公私钥，保存的 CertificateSigningRequest 里面就包含公钥，私钥保存在本地电脑里。
2. 向苹果申请对应把 CSR 传到苹果后台生成证书。
3. 把证书下载到本地。这时本地有两个证书。一个是第 1 步生成的私钥，一个是这里下载回来的证书，keychain 会把这两个证书关联起来，因为他们公私钥是对应的，在 XCode 选择下载回来的证书时，实际上会找到 keychain 里对应的私钥去签名。这里私钥只有生成它的这台 Mac 有，如果别的 Mac 也要编译签名这个 App 怎么办？答案是把私钥导出给其他 Mac 用，在 keychain 里导出私钥，就会存成 .p12 文件，其他 Mac 打开后就导入了这个私钥。

	<center>
	![](http://dzliving.com/iOSSafe_22.png)
	</center>

4. 在苹果网站上操作，配置 AppID/权限/设备等，最后下载 Provisioning Profile 文件。
5. XCode 会通过第 3 步下载回来的证书（存着公钥），在本地找到对应的私钥（第一步生成的），用本地私钥去签名 App，并把 Provisioning Profile 文件命名为 embedded.mobileprovision 一起打包进去。所以任何本地调试的 App，都会有一个 embedded.mobileprovision(描述文件)从 App Store 下载的没有。


## 六、APP签名的数据

这里对 App 的签名数据保存分两部分

1. Mach-O 可执行文件会把签名直接写入文件里

	<center>
	![](http://dzliving.com/iOSSafe_23.png)
	</center>

2. 其他资源文件则会保存在 _CodeSignature 目录下在 App 包里。

	<center>
	![](http://dzliving.com/iOSSafe_24.png)
	</center>


## 文章

[逻辑教育视频：](https://www.bilibili.com/video/av34227931)
[逻辑教育视频：双向验证](https://www.bilibili.com/video/av34261047)
[请叫我Hank](https://wwwjianshu.com/u/fa45480a8444) - [iOS应用签名(上)](https://www.jianshu.com/p/02034d1a91b5)、[iOS应用签名(下)](https://www.jianshu.com/p/3c9e2055ae5b)