---
title: SSH
categories: IT
---

## 一、SSH

> [SSH](https://baike.baidu.com/item/ssh/10407?fr=aladdin) 是一种协议标准，其目的是实现<font color=#cc0000>安全远程登录</font>以及其它<font color=#cc0000>安全网络服务</font>。

传统的网络服务程序，如：ftp、pop 和 telnet 在本质上都是不安全的，因为它们在网络上用明文传送口令和数据。

SSH 是目前较可靠，专为远程登录会话和其他网络服务提供安全性的协议。利用 SSH 协议可以有效防止远程管理过程中的信息泄露问题。既能防止中间人攻击，也能够防止 DNS 欺骗和 IP 欺骗。使用 SSH，还有一个额外的好处就是传输的数据是经过压缩的，所以可以加快传输的速度。

SSH 是一个协议标准，其具体的实现有很多，既有开源实现的 OpenSSH，也有商业实现方案。使用范围最广泛的当然是开源实现 OpenSSH。

SSH 主要由三部分组成：

1. 传输层协议 [SSH-TRANS]

	提供了服务器认证，保密性及完整性。此外它有时还提供压缩功能。 SSH-TRANS 通常运行在 TCP/IP 连接上，也可能用于其它可靠数据流上。 SSH-TRANS 提供了强力的加密技术、密码主机认证及完整性保护。该协议中的认证基于主机，并且该协议不执行用户认证。更高层的用户认证协议可以设计为在此协议之上。

2. 用户认证协议 [SSH-USERAUTH]

	用于向服务器提供客户端用户鉴别功能。它运行在传输层协议 SSH-TRANS 上面。当 SSH-USERAUTH 开始后，它从低层协议那里接收会话标识符（从第一次密钥交换中的交换哈希 H）。会话标识符唯一标识此会话并且适用于标记以证明私钥的所有权。 SSH-USERAUTH 也需要知道低层协议是否提供保密性保护。

3. 连接协议 [SSH-CONNECT]

	将多个加密隧道分成逻辑通道。它运行在用户认证协议上。它提供了交互式登录话路、远程命令执行、转发 TCP/IP 连接和转发 X11 连接。


## 二、SSH 工作原理

在讨论 SSH 的原理和使用前，我们需要分析一个问题：**为什么需要 SSH？**

SSH 和 telnet、ftp 等协议主要的区别在于<font color=#cc0000>安全性</font>。这就引出下一个问题：

> 如何实现数据的安全呢？

首先想到的实现方案肯定是对数据进行加密。加密的方式主要有两种：

1. 对称加密（也称为秘钥加密）
2. 非对称加密（也称公钥加密）

所谓对称加密，指加密解密使用同一套秘钥。如下图所示：

<center>
![对称加密-Client端](http://dzliving.com/SSH_0.png)
![对称加密-Server端](http://dzliving.com/SSH_1.png)
</center>

对称加密的加密强度高，很难破解。但是在实际应用过程中不得不面临一个棘手的问题：

> 密钥本身存在安全因素，密钥本身存在泄密可能性。如何安全的保存密钥呢？

尤其是考虑到数量庞大的 Client 端，很难保证密钥不被泄露。一旦一个 Client 端的密钥被窃据，那么整个系统的安全性也就不复存在。为了解决这个问题，非对称加密应运而生。非对称加密有两个密钥：“公钥”和“私钥”。

> 两个密钥的特性：公钥加密后的密文，只能通过对应的私钥进行解密。而通过公钥推理出私钥的可能性微乎其微。

下面看下使用非对称加密方案的登录流程：

<center>
![非对称加密登录流程](http://dzliving.com/SSH_2.png)
</center>

1. 远程 Server 收到 Client 端用户 TopGun 的登录请求，Server 把自己的公钥发给用户。
2. Client 使用这个公钥，将密码进行加密。
3. Client 将加密的密码发送给 Server 端。
4. 远程 Server 用自己的私钥，解密登录密码，然后验证其合法性。
5. 若验证结果，给 Client 相应的响应。

私钥是 Server 端独有，这就保证了 Client 的登录信息即使在网络传输过程中被窃据，也没有私钥进行解密，保证了数据的安全性，这充分利用了非对称加密的特性。

> 这样就一定安全了吗？

上述流程会有一个问题：Client 端如何保证接受到的公钥就是目标 Server 端的？如果一个攻击者中途拦截 Client 的登录请求，向其发送自己的公钥，Client 端用攻击者的公钥进行数据加密。攻击者接收到加密信息后再用自己的私钥进行解密，不就窃取了 Client 的登录信息了吗？这就是所谓的中间人攻击。

<center>
![中间人攻击](http://dzliving.com/SSH_3.png)
</center>

SSH中是如何解决这个问题的？

#### 2.1 基于口令的认证

从上面的描述可以看出，问题就在于如何对 Server 的公钥进行认证？在 https 中可以通过 CA 来进行公证，可是 SSH 的 publish key 和 private key 都是自己生成的，没法公证。只能通过 Client 端自己对公钥进行确认。通常在第一次登录的时候，系统会出现下面提示信息：

```
The authenticity of host 'ssh-server.example.com (12.18.429.21)' can't be established.
RSA key fingerprint is 98:2e:d7:e0:de:9f:ac:67:28:c2:42:2d:37:16:58:4d.
Are you sure you want to continue connecting (yes/no)? 
```

上面的信息说的是：无法确认主机 ssh-server.example.com（12.18.429.21）的真实性，不过知道它的公钥指纹，是否继续连接？

之所以用 <font color=#cc0000>fingerprint 代替 key</font>，主要是 key 过于长（RSA算法生成的公钥有 <font color=#cc0000>1024</font> 位），很难直接比较。所以，<font color=#cc0000>对公钥进行 hash 生成一个 128 位的指纹</font>，这样就方便比较了。

如果输入 yes 后，会出现下面信息：

```
Warning: Permanently added 'ssh-server.example.com,12.18.429.21' (RSA) to the list of known hosts. 
Password: (enter password) 
```

该 host 已被确认，并被追加到文件 <font color=#cc0000>known\_hosts</font> 中，然后就需要输入密码，之后的流程就正常进行。

#### 2.2 基于公钥认证

在上面介绍的登录流程中可以发现，每次登录都需要输入密码，很麻烦。SSH 提供了另外一种可以免去输入密码过程的登录方式：公钥登录。流程如下：

<center>
![公钥认证流程](http://dzliving.com/SSH_4.png)
</center>

1. Client 将自己的公钥存放在 Server 上，追加在文件 <font color=#cc0000>authorized\_keys</font> 中。
2. Server 端接收到 Client 的连接请求后，会在 authorized\_keys 中匹配到 Client 的公钥 pubKey，并<font color=#cc0000>生成随机数 R</font>，用 Client 的公钥对该随机数进行加密得到 pubKey(R)，然后将加密后信息发送给 Client。
3. Client 端通过私钥进行解密得到随机数 R，然后对随机数 R 和本次会话的 SessionKey 利用 <font color=#cc0000>MD5</font> 生成摘要 Digest1，发送给 Server 端。
4. Server 端会也会对 R 和 SessionKey 利用同样摘要算法生成 Digest2。
5. Server 端会最后比较 Digest1 和 Digest2 是否相同，完成认证过程。

在步骤 1 中，Client 将自己的公钥存放在 Server 上。需要用户手动将公钥 copy 到 server 上。这就是在配置 ssh 的时候进程进行的操作。下图是 GitHub 上 SSH keys 设置视图：

<center>
![GitHub中SSH keys设置](http://dzliving.com/SSH_5.png)
</center>

Server 端根据什么信息在 authorized\_keys 中进行查找的呢？主要是根据 Client 在认证的开始会发送一个 <font color=#cc0000>KeyID</font> 给Server，这个 KeyID 是唯一对应 Client 的一个 PublicKey，Server 就是通过该 KeyID 在 authorized\_keys 进行查找对应的 PublicKey。

## 三、密钥协商算法

通过 Diffie-Hellman 算法来实现，具体过程：

1. 服务端和客户端共同选定一个大素数，叫做种子值；
2. 服务端和客户端各自独立选择另外一个只有自己才知道的素数
3. 双方使用相同的加密算法(AES)，由种子值和各自私有的素数生成一个密钥值，并将这个值发送给对方
4. 在收到密钥后，服务端和客户端根据种子值和自己的私有素数，计算出一个最终的密钥
5. 双方使用上一步得到的结果作为密钥来加密和解密通信内容

## 四、SSH 实践

1. 生成密钥操作

	经过上面的原理分析，下面三行命令的含义应该很容易理解了：

	```
	$ ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
	$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
	$ chmod 0600 ~/.ssh/authorized_keys
	```

	ssh-keygen 是用于生产密钥的工具。

	* -t：指定生成密钥类型（rsa、dsa、ecdsa 等）
	* -P：指定 passphrase，用于确保私钥的安全
	* -f：指定存放密钥的文件（公钥、私钥文件默认在同目录下，但存放公钥的文件名需要加上后缀 .pub）

	首先看下面 ~/.ssh 中的四个文件：
	
	<center>
	![SSH-涉及文件](http://dzliving.com/SSH_6.png)
	</center>
	
	* id_rsa：保存私钥
	* id_rsa.pub：保存公钥
	* authorized_keys：保存已授权的客户端公钥
	* known_hosts：保存已认证的远程主机 ID

	四个角色的关系如下图所示：
	
	<center>
	![SSH 结构简图](http://dzliving.com/SSH_7.png)
	</center>
	
	注意：一台主机可能既是 Client，也是 Server。所以会同时拥有 authorized\_keys 和 known\_hosts。

2. 登录操作

	```
	# 以用户名 user，登录远程主机 host
	$ ssh user@host
	
	# 本地用户和远程用户相同，则用户名可省去
	$ ssh host
	
	# SSH 默认端口 22，可以用参数 p 修改端口
	$ ssh -p 2017 user@host
	```

## 五、known_hosts机制

1. known\_hosts 中存储的内容是什么？

	known\_hosts 中存储是已认证的远程主机 host key，每个 SSH Server 都有一个 `secret, unique ID, called a host key`。

2. host key 何时加入 known\_hosts 的？

	当我们第一次通过 SSH 登录远程主机的时候，Client 端会有如下提示：

	```
	Host key not found from the list of known hosts.
	Are you sure you want to continue connecting (yes/no)?
	```

	此时，如果我们选择 yes，那么该 host key 就会被加入到 Client 的 known\_hosts 中，格式如下：

	```
	# domain name+encryption algorithm+host key
	example.hostname.com ssh-rsa AAAAB4NzaC1yc2EAAAABIwAAAQEA...
	```

3. 为什么需要 known\_hosts？

	这个文件主要是通过 Client 和 Server 的双向认证，从而避免中间人攻击，每次 Client 向 Server 发起连接的时候，不仅仅 Server 要验证 Client 的合法性，Client 同样也需要验证 Server 的身份，SSH client 就是通过 known\_hosts 中的 host key 来验证 Server 的身份的。

这种方案足够安全吗？当然不，比如第一次连接一个未知 Server 的时候，known\_hosts 还没有该 Server 的 host key，这不也可能遭到中间人攻击吗？这可能只是安全性和可操作性之间的折中吧。

## 六、问题

> 如果 client 生成多个密钥对，把对应的公钥全部传到 server 端，这时候 server 端如何决定用哪个公钥对随机数进行加密？

多密钥场景下，需要在 `~/.ssh/config` 下进行配置，显示指定主机和 `key` 的对应关系，格式如下：

```
Host server01
HostName example1.com
Port 1000
IdentityFile ~/.ssh/id_rsa_xxx
User user01

Host server02
HostName example2.com
Port 1000
IdentityFile ~/.ssh/id_rsa_zzz
User user02
```

## 七、文章

[TopGun_Viper](https://www.jianshu.com/u/aee2b91e3398) - [图解SSH原理](https://www.jianshu.com/p/33461b619d53)
[SSH](https://www.jianshu.com/p/3c9f3252f568)
[Linux ssh命令详解](https://www.cnblogs.com/ftl1012/p/ssh.html)