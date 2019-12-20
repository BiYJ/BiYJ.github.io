---
title: iOS 重签名
categories: iOS原理
---

* 查看签名信息

	```
	$ codesign -vv -d WeChat.app/
	Executable=/Users/cykj/Desktop/WeChat/Payload/WeChat.app/WeChat
	Identifier=com.tencent.xin
	Format=app bundle with Mach-O universal (armv7 arm64)
	CodeDirectory v=20500 size=1068251 flags=0x0(none) hashes=16686+7 location=embedded
	Signature size=4390
	Authority=Apple iPhone OS Application Signing
	Authority=Apple iPhone Certification Authority
	Authority=Apple Root CA
	Info.plist entries=63
	TeamIdentifier=88L2Q4487U
	Sealed Resources version=2 rules=27 files=1267
	Internal requirements count=1 size=96

	$ codesign -d -v Demo.app
	Executable=/Users/cykj/Desktop/Demo.app/Demo
	Identifier=com.DemoDD
	Format=app bundle with Mach-O thin (arm64)
	CodeDirectory v=20400 size=718 flags=0x0(none) hashes=14+5 location=embedded
	Signature size=4717
	Signed Time=2019年11月8日 09:43:11
	Info.plist entries=26
	TeamIdentifier=8XF9U638XV
	Sealed Resources version=2 rules=13 files=7
	Internal requirements count=1 size=176
	```
	
	当 `Authority =(unavailable)` 时，说明已破壳。

* 查看电脑里有的证书

	```
	$ security find-identity -v -p codesigning
	  1) 61C19811D348A79062E2C2EA06BB7BE55CD739E4 "iPhone Developer: Hua Liu (YKR9SMF4G2)"
	  2) 23ED623C792844BFC041845033A367CC0B2644A4 "iPhone Distribution: Centrin Health Technology Co.,Ltd (CXACXM8Y8A)"
	  3) B6A5FC0E8150A96210DC7245D0831E5D8E8FB9C6 "iPhone Developer: 1070223935@qq.com (343672HBQC)" (CSSMERR_TP_CERT_REVOKED)
	  4) 61239D581B122C79D1A7023BC6F4AB2FB6885062 "iPhone Developer: Fan Ha (JX6YWJXDG6)" (CSSMERR_TP_CERT_REVOKED)
	  5) 8E875D5DBCD27796F14DC64766209B6EAE663A6F "iPhone Developer: Fan Ha (JX6YWJXDG6)"
	  6) 43B891317B4EF3B921C4A701376AACA65C83E7D4 "iPhone Distribution: Beijing Medical Clinic Co., Ltd. (SVUKZT3N6V)"
	  7) 3DB2F9B422B3E780D6D917E454A1058285113B76 "iPhone Developer: 1070223935@qq.com (343672HBQC)"
	     7 valid identities found
	```

* 查看可执行文件的信息
	
	```
	$ cd Demo.app
	$ otool -l Demo
	Demo:
	Mach header
	      magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
	 0xfeedfacf 16777228          0  0x00           2    22       2896 0x00200085
	Load command 0
	      cmd LC_SEGMENT_64
	  cmdsize 72
	  segname __PAGEZERO
	   vmaddr 0x0000000000000000
	   vmsize 0x0000000100000000
	  fileoff 0
	 filesize 0
	  maxprot 0x00000000
	 initprot 0x00000000
	   nsects 0
	    flags 0x0
	Load command 1
	      cmd LC_SEGMENT_64
	  cmdsize 792
	  segname __TEXT
	   vmaddr 0x0000000100000000
	   vmsize 0x0000000000008000
	  fileoff 0
	 filesize 32768
	  maxprot 0x00000005
	 initprot 0x00000005
	   nsects 9
	    flags 0x0
	Section
	  sectname __text
	   segname __TEXT
	      addr 0x0000000100006508
	      size 0x00000000000005a8
	    offset 25864
	    
	...
	```

* 将可执行文件信息转存到文件
	
	```
	$ otool -l Demo > ~/Desktop/1.txt
	```

* 查看可执行文件是否加密，在 1.txt 文件中查找 cryptid：

	![](http://dzliving.com/iOSCodeSign_0.png)
	
	crypt == 0 可用；crypt == 1 未解密，不可用。


https://www.jianshu.com/p/e111b4477ef1
https://www.jianshu.com/p/1938eeddd2f3
https://www.jianshu.com/p/c8a7302babc2
