---
title: SSH 的前世今生
categories: [运维]
---
SSH 是一个用来解决通信安全的一个协议，1995 年，芬兰学者 Tatu Ylonen 设计了 SSH 协议，将登录信息全部加密，成为互联网安全的一个基本解决方案，迅速在全世界获得推广，目前已经成为 Linux 系统的标准配置。
> SSH仅仅是一协议标准，其具体的实现有很多，既有开源实现的OpenSSH，也有商业实现方案。使用范围最广泛的当然是开源实现OpenSSH。现在操作系统基本都预装了 OpenSSH

## OpenSSH 概述
SSH 的软件架构是服务器-客户端模式（Server - Client）。在这个架构中，SSH 软件分成两个部分：向服务器发出请求的部分，称为客户端（client），OpenSSH 的实现为 ssh；接收客户端发出的请求的部分，称为服务器（server），OpenSSH 的实现为 sshd。
另外，OpenSSH 还提供一些辅助工具软件（比如 ssh-keygen 、ssh-agent）和专门的客户端工具（比如 scp 和 sftp）
scp 是 SSH 提供的一个客户端程序，用来在两台主机之间加密传送文件（即复制文件）
sftp 是 SSH 提供的一个客户端应用程序，主要用来安全地访问 FTP。因为 FTP 是不加密协议，很不安全，sftp 就相当于将 FTP 放入了 SSH。
## 原理
 	实现安全通信首先想到的实现方案肯定是对数据进行加密。加密的方式主要有两种：
> - 对称加密（也称为秘钥加密）
> - 非对称加密（也称公钥加密）

### 对称加密
对称加密指加密解密使用同一套秘钥。常用的对称加密算法有 AES、Blowfish、3DES、CAST128、以及Arcfour。原理图如下：
![](https://cdn.nlark.com/yuque/0/2022/webp/23060187/1670330917336-b9b38e52-9c5c-4453-bfbd-5b7603631ca1.webp#averageHue=%23f4f4f4&clientId=u6ad43652-4976-4&from=paste&id=u2081d1f8&originHeight=190&originWidth=764&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=stroke&taskId=u45cda712-dfee-47b1-9d6d-d09303ca229&title=)
![](https://cdn.nlark.com/yuque/0/2022/webp/23060187/1670330924205-79e43633-b340-4c52-8fb1-3cfced6cffec.webp#averageHue=%23f4f4f4&clientId=u6ad43652-4976-4&from=paste&id=u2a78e038&originHeight=191&originWidth=764&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=stroke&taskId=ufa3a095d-46c3-4b92-9231-936dd8edc8f&title=)

### 非对称加密
1976年，美国学者Dime和Henman为解决信息公开传送和密钥管理问题，提出一种新的密钥交换协议，允许在不安全的媒体上的通讯双方交换信息，安全地达成一致的密钥，这就是“公开密钥系统”。
与对称加密算法不同，非对称加密算法需要两个密钥：公开密钥（publickey）和私有密钥（privatekey）。公开密钥与私有密钥是一对，如果用公开密钥对数据进行加密，只有用对应的私有密钥才能解密；如果用私有密钥对数据进行加密，那么只有用对应的公开密钥才能解密。因为加密和解密使用的是两个不同的密钥，所以这种算法叫作非对称加密算法。
数据的加密和解密过程是通过密码体制和密钥来控制的。密码体制的安全性依赖于密钥的安全性，现代密码学不追求加密算法的保密性，而是追求加密算法的完备，即：使攻击者在不知道密钥的情况下，没有办法从算法找到突破口。根据加解密算法所使用的密钥是否相同，或能否由加(解)密密钥简单地求得解(加)密密钥。密码体制可分为对称密码体制和非对称密码体制。
非对称密码体制也叫公钥加密技术，该技术是针对私钥密码体制(对称加密算法)的缺陷被提出来的。与对称密码体制不同，公钥加密系统中，加密和解密是相对独立的，加密和解密会使用两把不同的密钥，加密密钥(公开密钥)向公众公开，谁都可以使用，解密密钥(秘密密钥)只有解密人自己知道，非法使用者根据公开的加密密钥无法推算出解密密钥，这样就大大加强了信息保护的力度。[公钥密码体制](https://baike.baidu.com/item/%E5%85%AC%E9%92%A5%E5%AF%86%E7%A0%81%E4%BD%93%E5%88%B6/10392876?fromModule=lemma_inlink)不仅解决了密钥分配的问题，它还为签名和认证提供了手段。
非对称密码算法有很多，其中比较典型的是[RSA算法](https://baike.baidu.com/item/RSA%E7%AE%97%E6%B3%95/263310?fromModule=lemma_inlink)，它的数学原理是大素数的分解

非对称加密如何被应用呢？以用户 TopGun 要尝试登录服务器为例，流程图如下

![](https://cdn.nlark.com/yuque/0/2022/webp/23060187/1670330934150-641b8b18-08df-4b73-81b3-9a0c61cdab94.webp#averageHue=%2372cc9a&clientId=u6ad43652-4976-4&from=paste&id=u563ee0aa&originHeight=542&originWidth=569&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=stroke&taskId=u40e30782-79fb-475d-b31c-0a2a6a12f03&title=)

登录流程：

1. 远程Server收到Client端用户TopGun的登录请求，Server把自己的公钥发给用户。
2. Client使用这个公钥，将密码进行加密。
3. Client将加密的密码发送给Server端。
4. 远程Server用自己的私钥，解密登录密码，然后验证其合法性。
5. 若验证结果，给Client相应的响应。
> 补充：这一种登陆方式是通过口令的方式登陆。需要每次输入一次密码，密码输入后通过服务端的公钥加密后进行传输到服务端，服务端的私钥进行解密。 

### 中间人攻击
上面过程本身是安全的，但是实施的时候存在一个风险：
> Client端如何保证接受到的公钥就是目标Server端的？，如果一个攻击者中途拦截Client的登录请求，向其发送自己的公钥，Client端用攻击者的公钥进行数据加密。攻击者接收到加密信息后再用自己的私钥进行解密，不就窃取了Client的登录信息了吗？

如果有人截获了登录请求，然后冒充远程主机，将伪造的公钥发给用户，那么用户很难辨别真伪。因为不像https协议，SSH协议的公钥是没有证书中心（CA）公证的，也就是说，都是自己签发的。可以设想具体场景，如果攻击者插在用户与远程主机之间（比如在公共的wifi区域），用伪造的公钥，获取用户的登录密码。再用这个密码登录远程主机，那么SSH的安全机制就荡然无存了。这种风险就是著名的"中间人攻击"(Man-in-the-middle attack)。
![](https://cdn.nlark.com/yuque/0/2022/webp/23060187/1670330943900-e7a79e9b-fab2-4b9e-b2c2-5c4bc4dbeaaf.webp#averageHue=%23cfe5db&clientId=u6ad43652-4976-4&from=paste&id=u12c6e64a&originHeight=481&originWidth=623&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=stroke&taskId=uf23e19ea-042f-469b-bc59-e8a501e0b31&title=)
### SSH解决中间人攻击
#### 1、口令登录（基于口令的认证）
从上面的描述可以看出，问题就在于如何对Server的公钥进行认证？在https中可以通过CA来进行公证，可是SSH的publish key和private key都是自己生成的，没法公证。只能通过Client端自己对公钥进行确认。
通常在第一次登录的时候，系统会出现下面提示信息：
```xml
$ ssh user@host
The authenticity of host 'host (12.18.429.21)' can't be established.
RSA key fingerprint is 98:2e:d7:e0:de:9f:ac:67:28:c2:42:2d:37:16:58:4d.
Are you sure you want to continue connecting (yes/no)?
```
上面的信息说的是：无法确认主机host(12.18.429.21)的真实性，不过知道它的公钥指纹，询问你是否继续连接？所谓"公钥指纹"，是指公钥长度较长（这里采用RSA算法，长达1024位），很难比对，所以对其进行MD5计算，将它变成一个128位的指纹。上例中是`98:2e:d7:e0:de:9f:ac:67:28:c2:42:2d:37:16:58:4d` ，再进行比较，就容易多了。
> 很自然的一个问题就是，用户怎么知道远程主机的公钥指纹应该是多少？
> 回答是没有好办法，远程主机必须在自己的网站上贴出公钥指纹，以便用户自行核对，或者自己甄别host地址是否正确。

```xml
// 1. 假定经过风险衡量以后（一般用户直接就选择yes吧），用户决定接受这个远程主机的公钥。
　　Are you sure you want to continue connecting (yes/no)? yes
// 2. 系统会出现一句提示，表示host主机已经得到认可。
　　Warning: Permanently added 'host,12.18.429.21' (RSA) to the list of known hosts.
//  3. 然后，会要求输入密码。
　　Password: (enter password)
//  如果密码正确，就可以登录了。
```
当远程主机的公钥被接受以后，它就会被保存在文件$HOME/.ssh/known_hosts之中。下次再连接这台主机，系统就会认出它的公钥已经保存在本地了，从而跳过警告部分，直接提示输入密码。每个SSH用户都有自己的known_hosts文件，此外系统也有一个这样的文件，通常是/etc/ssh/ssh_known_hosts，保存一些对所有用户都可信赖的远程主机的公钥。

#### 2、公钥登录（基于公钥的认证）
使用密码登录，每次都必须输入密码，非常麻烦。好在SSH提供了另外一种可以免去输入密码过程的登录方式：公钥登录。
> 所谓"公钥登录"，原理很简单，就是用户将自己的公钥储存在远程主机上。
> 登录的时候，远程主机会向用户发送一段随机字符串，用户用自己的私钥加密后，再发回来。远程主机用事先储存的公钥进行解密，如果成功，就证明用户是可信的，直接允许登录shell，不再要求密码。

以用户 TopGun 为例，流程如下：
公钥认证流程：
![公钥认证流程图 1](https://cdn.nlark.com/yuque/0/2022/png/23060187/1670144598451-1df25a2b-4b50-4c6c-9216-4f36bda4e91e.png#averageHue=%23020202&clientId=u502bb770-0544-4&from=paste&height=706&id=uce12715b&originHeight=635&originWidth=1142&originalType=binary&ratio=1&rotation=0&showTitle=true&size=50539&status=done&style=stroke&taskId=u402a7b8d-c32a-4549-bc0d-34ad3c65a29&title=%E5%85%AC%E9%92%A5%E8%AE%A4%E8%AF%81%E6%B5%81%E7%A8%8B%E5%9B%BE%201&width=1268.888922502966 "公钥认证流程图 1")
![公钥认证流程图 2](https://cdn.nlark.com/yuque/0/2022/webp/23060187/1670330960539-68ba9d77-471c-4cf8-aada-c6f8698dc94f.webp#averageHue=%23a8afa1&clientId=u6ad43652-4976-4&from=paste&id=u2adc57e6&originHeight=836&originWidth=1200&originalType=url&ratio=1&rotation=0&showTitle=true&status=done&style=stroke&taskId=ub43ef4d6-a4d8-423f-8ab6-b3a3449db90&title=%E5%85%AC%E9%92%A5%E8%AE%A4%E8%AF%81%E6%B5%81%E7%A8%8B%E5%9B%BE%202 "公钥认证流程图 2")

1. Client 端用户 TopGun 将自己的公钥存放在 Server 上，追加在文件 authorized_keys 中。
2. Server 收到登录请求后，服务器的公钥验证通过后, 双方会依据 Diffie-Hellman 算法生成一个共享 `session key` （此时双方都有 `session key`）会话密钥将用于加密整个会话。之后 Server  随机生成一个字符串 str，并发送给 Client。
3. 客户端通过私钥解密随机数字后，将获取的随机数与共享会话密钥结合，计算该值的 MD5 哈希值。发送给服务器
4. 服务器使用  `session key` 它发送给客户端的随机 str 来自行计算 MD5 值。它将自己的计算与客户端发回的计算进行比较。如果这两个值匹配，则证明客户端拥有私钥并且客户端已通过身份验证

> 在步骤1中，Client将自己的公钥存放在Server上。需要用户手动将公钥copy到server上。这就是在配置ssh的时候进程进行的操作。下图是GitLab上SSH keys设置视图：
> ![](https://cdn.nlark.com/yuque/0/2022/png/23060187/1670143271055-5e7c0e9a-614f-4d15-a4d0-046a988fa39c.png#averageHue=%23fbfbfb&clientId=u502bb770-0544-4&from=paste&id=ua362a4a6&originHeight=936&originWidth=2496&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=stroke&taskId=ufb0eff60-a48f-40dc-88dc-146c6f7d036&title=)
> 

这种方法要求用户必须提供自己的公钥。如果没有现成的，可以直接用ssh-keygen生成一个：
```shell
ssh-keygen
```
运行上面的命令以后，系统会出现一系列提示，可以一路回车。其中有一个问题是，要不要对私钥设置口令（passphrase），如果担心私钥的安全，这里可以设置一个。
运行结束以后，在$HOME/.ssh/目录下，会新生成两个文件：id_rsa.pub 和 id_rsa 。前者是你的公钥，后者是你的私钥。
比如，我需要用用户user登录host服务器，只需再输入下面的命令，将公钥传送到远程主机host上面：
```shell
 ssh-copy-id user@host
```
好了，从此你再登录，就不需要输入密码了
如果还是不行，就打开远程主机的/etc/ssh/sshd_config这个文件，检查下面几行前面"#"注释是否取掉。
```shell
 RSAAuthentication yes
 PubkeyAuthentication yes
 AuthorizedKeysFile .ssh/authorized_keys
```
然后，重启远程主机的ssh服务。
```shell
 // ubuntu系统
 service ssh restart
 // debian系统
 /etc/init.d/ssh restart
```

什么是authorized_keys文件？
远程主机将用户的公钥，保存在登录后的用户主目录的$HOME/.ssh/authorized_keys文件中。公钥就是一段字符串，只要把它追加在authorized_keys文件的末尾就行了。
这里不使用上面的ssh-copy-id命令，改用下面的命令，解释公钥的保存过程：
```shell
$ ssh user@host 'mkdir -p .ssh && cat >> .ssh/authorized_keys' < ~/.ssh/id_rsa.pub
```
这条命令由多个语句组成，依次分解开来看：
（1）"$ ssh user@host"，表示登录远程主机；
（2）单引号中的mkdir .ssh && cat >> .ssh/authorized_keys，表示登录后在远程shell上执行的命令：
（3）"$ mkdir -p .ssh"的作用是，如果用户主目录中的.ssh目录不存在，就创建一个；
（4）'cat >> .ssh/authorized_keys' < ~/.ssh/id_rsa.pub的作用是，将本地的公钥文件~/.ssh/id_rsa.pub，重定向追加到远程文件authorized_keys的末尾。
写入authorized_keys文件后，公钥登录的设置就完成了。我本地的ahtorized_keys文件。添加后内容如下：
```shell
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDQsqm2aqI3MCUcDlcqnU9JfE30TYfUjayKGf/+WvLOpFVUqhwnrAFVT7VaCW5G5Lh3IlWzGCflhl0Yjasb3BTELV+W+zHpefgEwJaNB725yfzdPvk/aGrHv7UusggMqqR11JDW2zlISo/xUysEzJ4pTnjz6AcdKUCE8biB3s37yTSa8k78g2j06tzLglr+np179jCiq9RJfKCs/omeCHbU+pyhcCk6DoL/SRHzNczfHIcJBvhWvY7CPKHb2vqD9k5d9cS2renGvB2/fCkyyNdnVZgKc3r5ufJUD+yWJ6yMHNT/hMn0uybEHuVz8s/zXjdH3xgx40UrfLs8eNKTKpAN app@bjxg-ap-27-7
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCk7sNBV8ag8KKApN4mapTQTkDEbcA3VNFIhSuO+YLTbkO29KU6FSQhlT2It2hmmmdaVJXEMx6JmQS7+62DNHIpHnzaKimEU+tKosAdOwHoZPcskMw6QlvjkhicKg7+kNsFdeSuAtE3V4ShbFPUfRNAs/9a/b56/FqqGIk6oPVu19CSvMyLnnqTht9rMQdIw4uOynogEKivgdwsAFh4qayJSz8VwklN9dx+8dqw8e9e/CxXEDWuZ2T/DtEa+gIRy0bvJaDcgzcpvxMkf+71HTpYlvkziF1YPmPX60gcyCkRkLnmM1fSkHXhAmBxZuSLc3l1VKXXWkH9rsgvxLzKuML/ app@bjxg-ap-28-7
```

## 配置 Windows Terminal
#### 1.下载 Windows Terminal
在微软商店（Microsoft Store）搜索下载Windows Terminal。
![](https://cdn.nlark.com/yuque/0/2022/png/23060187/1670145414520-801211fc-07dd-4c01-bcea-203c5eb649bc.png#averageHue=%23f7f7f7&clientId=u502bb770-0544-4&from=paste&id=u78066f12&originHeight=198&originWidth=631&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=stroke&taskId=ud0c0c304-cb1a-4695-b3ed-e67c52d9634&title=)
#### 2. 安装 OpenSSH 客户端
在设置-->应用-->可选功能中添加OpenSSH客户端
![](https://cdn.nlark.com/yuque/0/2022/png/23060187/1670145434758-af59935a-6bb2-45a4-956e-b5fd81a30276.png#averageHue=%23f2f5f8&clientId=u502bb770-0544-4&from=paste&id=ub1438820&originHeight=504&originWidth=1003&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=stroke&taskId=ua4d40afc-6b9a-4085-a230-002cecf1458&title=)
#### 3. 测试 ssh 和 scp 命令功能
![](https://cdn.nlark.com/yuque/0/2022/png/23060187/1670145450357-48be18e5-c7a6-44ea-bb03-20d4ae413649.png#averageHue=%23282828&clientId=u502bb770-0544-4&from=paste&id=ubb7315b5&originHeight=386&originWidth=743&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=stroke&taskId=u7f0a66ab-4a45-426c-a9c4-d93c74baaec&title=)
#### 4. 生成密钥
在 CMD 中执行命令 ssh-keygen
![](https://cdn.nlark.com/yuque/0/2022/png/23060187/1670145473739-dde00269-37dc-4903-ab6b-ebb6d4a3abe2.png#averageHue=%23161616&clientId=u502bb770-0544-4&from=paste&id=u98c34e26&originHeight=404&originWidth=692&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=stroke&taskId=ufdf573bc-7675-4fc3-9fa7-f029ca34b7e&title=)
#### 5. 使用 scp 命令将 Windows 公钥发送至服务器
```
scp -P 22 C:\Users\tangsc\.ssh\id_rsa.pub root@1.117.62.185:~/.ssh/authorized_keys
```
![](https://cdn.nlark.com/yuque/0/2022/png/23060187/1670145494586-3d2d6d6b-6a33-4db2-b9eb-e21537877695.png#averageHue=%231c1c1c&clientId=u502bb770-0544-4&from=paste&id=u6ede6e19&originHeight=168&originWidth=1044&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=stroke&taskId=uc3f659d4-e1f5-4cb7-9d7e-0e4f5baa66d&title=)
#### 6. 在 Windows Terminal 设置登录配置文件
复制一份命令提示符的配置文件，修改命令行。
![](https://cdn.nlark.com/yuque/0/2022/png/23060187/1670145510105-96966bf9-c9e2-458a-a977-870e94d1433f.png#averageHue=%23f7f7f7&clientId=u502bb770-0544-4&from=paste&id=u9e4c7d6f&originHeight=558&originWidth=476&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=stroke&taskId=u6213d79e-daf3-4987-aafb-fc6bb7a8d9e&title=)


更新时间

1. 2022年12月4日16点02分

参考文档

1. [使用Windows Terminal免密登录Linux服务器 – 运维之路](https://www.oaroad.com/software/3748878.html)
2. [SSH Secure Shell home page, maintained by SSH protocol inventor Tatu Ylonen. SSH clients, servers, tutorials, how-tos.](https://www.ssh.com/academy/ssh)
3. [SSH 基本知识](https://wangdoc.com/ssh/basic)
4. [ssh原理以及与https的区别_PeipeiQ的博客-CSDN博客_ssh和https](https://blog.csdn.net/PeipeiQ/article/details/80702514)
5. [非对称加密_百度百科](https://baike.baidu.com/item/%E9%9D%9E%E5%AF%B9%E7%A7%B0%E5%8A%A0%E5%AF%86/9874417?fr=aladdin)
6. [图解SSH原理](https://www.jianshu.com/p/33461b619d53)
