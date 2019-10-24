---
title: OpenSSL演示RSA加密解密
categories: IT
---

原文：[OXHO](https://www.jianshu.com/u/078bfe235e51) - [iOS---利用OpenSSL演示RSA加密解密，PEM。](https://www.jianshu.com/p/6e428c2f870d)


链接：<a href='https://www.jianshu.com/p/6e428c2f870d'></a>
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

主要命令：

1. `genrsa` 生成并输入一个 RSA 密钥；
2. `rsautl` 使用 RSA 密钥进行加密、解密、签名和验证等运算；
3. `rsa` 处理 RSA 密钥的格式转换等问题。

演示步骤：

1. 生成一个私钥 private.pem 文件
	
	```
	$ openssl genrsa -out private.pem 1024
	```
	
	1024 代表位数，默认是 2048，也可以是 512 的。

2. 提取公钥 public.pem

	```
	$ openssl rsa -in private.pem -pubout -out public.pem
	```
	
3. cat 命令查看 private.pem 文件

	```
	$ cat private.pem
	-----BEGIN RSA PRIVATE KEY-----
	MIICXQIBAAKBgQDAXb5xgXWKdGizJ6lFp61U8Mk6TdBP0HgP38ZeiwEysgNfnPS1
	T8Lf0+OXbkWRdTdLAxCEG6IXp/gwBfqA2yab1GNbtsJSch/KxCqmHlxqbNB54dZH
	6TvibZLIXVbGysIc/keqkBW0Q+BZ2/bqgsRqDHByVtb5wE8o7AEbv+OD7QIDAQAB
	AoGAH4J7gn4xMbe1urrhaE55/vLeE1XRrHE6aWc/SBc+3/32vX+pOdXR1vYPTqu6
	a4QmxXvABdO87mEOL1ebW+YJ4a3ktg31AjmCeYBbLUpx2V54kPS3g144zq2VgW2O
	UJW6+QPBpD7fxeHc9IVU4ecgqPaT25srT43bZ/oo8f4B5UUCQQDjYlg1J5ZiN+m/
	ELUsT8aOwEPWUt2dfZWmHECDTgTLWyu1lZumrZsND5SRvgvTxFDBTAPSUSrC9uJ9
	isfbRotnAkEA2JM46HrYttHYCJc4Fz9qUQItw2Nczf4kLozZ9MwD+1vP20rrP3Ju
	HnKQoN+UitWmCMGCT2q5hes5x2oK8gq1iwJAS3yxpevfi/nd+tVUIELXuzpvCu71
	rbwseznz3OOAyfYZe64QP3Rw/hQHEZ9TE9mfqZxPxHm3xYPqyxzQnqY7zwJBAIug
	fvQDH2zeZTVEqPjz/Ys0qGRrzw1PZ+xLCsn0Li0DyEJNTtWc5LnkirHs80t+6oiC
	mYyx9eINATc7essJdHkCQQC2qQfycn7Z43PmYtETznFNZV4NjenQ/EVvVSC0VCCr
	b4UT4rUcQriUsBBv1gGHP6kzAl3ttXxsoO2W59CDdf7A
	-----END RSA PRIVATE KEY-----
	```
	
	可以看出就是 base64 编码。

4. 把 private.pem 转成明文

	```
	$ openssl rsa -in private.pem -text -out private.txt
	```

	总结：OpenSSL 使用 PEM 文件格式存储证书和密钥。

5. 通过公钥来加密数据

	* 创建一个文本文件 a.txt，写入想要加密的数据
	
		```
		$ touch a.txt
		$ vim a.txt
		```
		
		<center>
		![](http://dzliving.com/openSSL_aTxt.png?imageView2/0/w/350)
		</center>

	* 利用公钥加密 a.txt，输出一个 enc.txt 文件
	
		```
		openssl rsautl -encrypt -in a.txt -inkey public.pem -pubin -out enc.txt
		```
		
		其中 `-encrypt` 代表加密（和私钥有差异，记得对比）

	* 公钥加密的 enc 文件，利用私钥解密，生成 dec.txt 文件

		```
		$ openssl rsautl -decrypt -in enc.txt -inkey private.pem -out dec.txt
		```
		
		<center>
		![](http://dzliving.com/openSSL_dec.png?imageView2/0/w/350)
		</center>

6. 通过私钥来签名（加密）数据

	* 利用私钥 private.pem 签名 a.txt
		
		```
		$ openssl rsautl -sign -in a.txt -inkey private.pem -out enc.bin
		```
		
		其中 `-sign` 代表签名，公钥是 `-encrypt` 加密。

		利用 “xxd enc.bin” 命令查看 bin 文件加密之后的信息：
		
		```
		$ xxd enc.bin
		00000000: 6d83 6157 cc0a 27f9 e961 365a 265e 21cc  m.aW..'..a6Z&^!.
		00000010: 38f0 6e12 d824 f66a 484e 70b4 ccdf 62a9  8.n..$.jHNp...b.
		00000020: 563f 5806 c34d 81bb 6bfb 3ba0 5114 6fe4  V?X..M..k.;.Q.o.
		00000030: 960c 14d5 91e6 7d0a 4e8f ec76 a8db 1bc4  ......}.N..v....
		00000040: ac55 5c16 2084 f4f7 bbd6 2795 c4c6 43ff  .U\. .....'...C.
		00000050: 217d d2ef e790 6980 6843 334c ee76 3315  !}....i.hC3L.v3.
		00000060: 4933 22e0 4b0b 9524 9fd9 358e 34b2 549c  I3".K..$..5.4.T.
		00000070: 0db4 2b6e 51c4 bbef 12e1 a899 fca3 30c4  ..+nQ.........0.
		```

	* 利用公钥 public.pem 解密 enc.bin

		```
		$ openssl rsautl -verify -in enc.bin -inkey public.pem -pubin -out dec.txt
		```