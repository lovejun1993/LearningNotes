---
layout: post
title: "RSA Encrypt"
date: 2015-01-18 11:07:46 +0800
comments: true
categories: 
---

iOS中使用RSA非对称加密

***

###先查看当前Mac系统安装的OpenSSL版本

```
openssl version
```

默认Mac系统安装的是OpenSSL 0.9.8zh 14 Jan 2016。版本已经很老了，需要更新下OpenSSL程序。

****

###更新OpenSSL

- (1) 使用brew更新

```
brew install openssl
```

- (2) link 安装更新的 openssl

```
brew link openssl --force
```

- (3) 设置环境变量


```
fenqiledeiMac:~ fenqile$ echo $SHELL
/bin/bash
fenqiledeiMac:~ fenqile$ vim ~/.bash_profile
fenqiledeiMac:~ fenqile$ source ~/.bash_profile
fenqiledeiMac:~ fenqile$ openssl version
OpenSSL 1.0.2h  3 May 2016
```

在`~/.bash_profile`文件中加入如下环境变量

```
export PATH=/usr/local/bin:$PATH
```

***

###RSA加解密算法原理

- (1) RSA算法是一种非对称加密算法,常被用于加密数据传输
- (2) 在加密解密数据前,需要先生成公钥(public key)和私钥(private key)
- (3) 公钥(public key): 用于加密数据. 用于公开, 一般存放在数据提供方, 例如iOS客户端
- (4) 私钥(private key): 用于解密数据. 必须保密, 私钥泄露会造成安全问题，一般只存放在Server端

***

###生成秘钥文件

```
openssl genrsa -out private.pem 1024
```

- (1) openssl:是一个自由的软件组织,专注做加密和解密的框架
- (2) genrsa:指定了生成了算法使用RSA
- (3) -out:后面的参数表示生成的key的输入文件
- (4) 1024:表示的是生成key的长度,单位字节(bits)

在当前目录下生成一个private.pem文件

```
fenqiledeiMac:~ fenqile$ ls -l
total 8
drwx------+ 27 fenqile  staff   918  7 18 11:17 Desktop
drwx------+  7 fenqile  staff   238  7  8 16:16 Documents
drwx------+ 78 fenqile  staff  2652  7 18 12:42 Downloads
drwx------@ 58 fenqile  staff  1972  7 18 10:38 Library
drwx------+  3 fenqile  staff   102  6  6 09:56 Movies
drwx------+  4 fenqile  staff   136  6  7 20:32 Music
drwx------+  5 fenqile  staff   170  7 14 13:03 Pictures
drwxr-xr-x+  5 fenqile  staff   170  6  6 09:56 Public
drwxr-xr-x  37 fenqile  staff  1258  6  7 13:14 Reader
drwxr-xr-x  15 fenqile  staff   510  6  7 11:23 YTKNetwork
drwxr-xr-x   3 fenqile  staff   102  6 28 11:22 bin
drwxr-xr-x  14 fenqile  staff   476  7 18 10:23 fenqile
-rw-r--r--   1 fenqile  staff   891  7 18 12:49 private.pem
drwxr-xr-x   6 fenqile  staff   204  7 12 15:33 tiqianle
```

***

###创建证书请求 

```
openssl req -new -key private.pem -out rsacert.csr
```

输入命令后，会依次叫你输入很多配置项目，依次按照提示填写完毕即可，有一项`A challenge password`就需要记住了，这个是设置证书的密码。

所有输入完毕之后，当前目录下会多出一个文件`rsacert.csr`。

还可以拿着这个文件去数字证书颁发机构（即CA）申请一个数字证书。CA会给你一个新的文件cacert.pem,那才是你的数字证书。(要收费的)

***

###生成证书并签名，有效期10年
 
```
fenqiledeiMac:~ fenqile$ openssl x509 -req -days 3650 -in rsacert.csr -signkey private.pem -out rsacert.crt
Signature ok
subject=/C=cn/ST=ShenZhen/L=ShenZhen/O=Fenqile/OU=Fenqile/CN=Fenqile/emailAddress=zain@fenqile.com
Getting Private key
```

- (1) 509是一种非常通用的证书格式
- (2) 将用上面生成的密钥privkey.pem和rsacert.csr证书请求文件生成一个数字证书rsacert.crt（公钥）

当前目录下会多出一个文件`rsacert.crt`，就是公钥。

***

###转换格式 将 CRT 格式文件 转换成 DER 格式

```
openssl x509 -outform der -in rsacert.crt -out rsacert.der
```

当前目录下会多出一个文件`rsacert.der`，转换文件格式后的公钥。

为什么要转换成DER格式了？

因为在iOS开发中,公钥是不能使用base64编码的。上面的命令是将公钥的base64编码字符串转换成二进制数据。

***

###导出 私钥PEM文件的 P12文件

```
openssl pkcs12 -export -out p.p12 -inkey private.pem -in rsacert.crt
```

在iOS中，不能直接使用私钥PEM文件，所以需要导出一个p12文件。
执行完命令后，会多出一个名字`p.p12`的p12格式文件，这个就是PEM私钥。

***

###执行完上面的这些步骤，总共可以生成如下五个文件

```
private.pem    	PEM格式私钥
rsacert.csr		证书请求文件
rsacert.crt		CRT格式公钥
rsacert.der		DER格式公钥
p.p12			p12格式私钥
```


真正在加解密中起作用的只有两个文件:

```
rsacert.der		DER格式公钥
p.p12			p12格式私钥
```

***

###Demo工程中测试生成的公钥与秘钥进行加解密


***

###使用OpenSSL-for-iOS提供的shell脚本自动集成OpenSSL库代码，并生成对应的静态库.a或.framework


 下载shell脚本，然后执行脚本，等待下载完所有的openssl库代码然后编译完毕就可以了，自动生成.a还可以生成.framework。
 
***

###最后就是用一个封装了RSA操作的开源库来完成加解密

https://github.com/ideawu/Objective-C-RSA
 
 




