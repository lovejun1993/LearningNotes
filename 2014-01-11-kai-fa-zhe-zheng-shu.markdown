---
layout: post
title: "开发者证书"
date: 2014-01-11 21:29:58 +0800
comments: true
categories: 
---

***

### 开发者`账号`的种类

- `个人`(individual) $99/year
	- 可以真机调试, 发布 appstore, 每年 最多为 100台设备分发。
	
- `公司`(company) $99/year
	- 与`个人帐号`类似, 只有这两种帐号（`个人和公司`）可以发布 appstore。
	- 但是比`个人账号`申请麻烦，需要申请`邓白氏编码`。
	- `公司账号优势`: 可以添加多个`开发者子账号`, 但只允许`主账号提交`, `发布`等操作, 在协同开发时比较灵活, 可以各自管理授权设备等。
	
- `企业`(enterprise) $299/year
	- 企业帐号`无法用于appstore发布`。
	- 但可以`不通过appstore发布`让任意iphone都可以安装应用。

- `大学`(University) free
	- 大学帐号不能发布 appstore, 主要拥有真机调试的权限。

	
***

###cer证书与描述文件provision的关系

- 证书(xxx.cer)

- 描述文件(xxx.provision)

- 二者之间的关系


***

###证书一、真机调试的证书

- 登陆开发者中心

- 生成`cer证书`
	- 1) 使用某一个mac机器生成一个csr文件。
	- 2) 使用csr文件，在开发者中心申请到一个调试cer证书，告诉苹果这一台mac机器需要真机调试。
			- 这个过程呢，实际上是生成了一对`公钥`和`私钥`
			- 然后保存在我们电脑上的钥匙串中
	- 3) 下载cer证书。
	- `注意`: 其他机器也要进行真机调试时，把申请cer证书的机器上的cer证书，导出成`p12证书`给其他机器安装即可。
		- 因为cer证书，是使用某一个机器的csr文件申请创建的。也就是说cer证书与某一个机器一一对应。

- 添加`App Id`
	- 告诉苹果需要真机调试哪一个App。
	- App Id分两种
		- 1) `明确Id`: 如果要进行 `远程推送`/`游戏中心`/`内购`等功能，必须要填写一个明确的`AppId`。
		- 2) `模糊Id`: 如果不进行 `远程推送`/`游戏中心`/`内购`等，就可以使用。

- 注册`真机设备`(iphone...)
	- 告诉苹果需要在哪一个设备上进行真机调试。

- 创建`Mobile Provision`文件
	- 让前面三步得到的东西，结合起来 
	- 1) 在某一台mac机器上
	- 2) 将某一个App
	- 3) 在mac机器链接的iOS设备上
	- 进行调试

- 图示

![](http://i13.tietuku.com/b21133f7f773bf53.png)

****

###证书二、代码打包成ipa的证书

- `应用安装到手机`只有两种途径
	- `AppStore`下载按照
	- 安装`打包`产生的ipa文件

- 不是任何mac机器都可以打包，只有拥有`打包证书`的mac机器才可以

- 让一台mac机器具备打包能力的步骤，必须申请`打包证书`
	- 登陆开发者中心
	- 使用某台mac机器的生产一个csr请求文件
	- 使用这个csr请求文件，申请创建打包证书
	- 把需要安装打包程序的设备UUID加入到开发者中心设备管理中

- 双击创建完打包cer证书进行安装
	- 将安装的证书导出为p12交换证书，让其他mac机器也可以打包。

- 申请打包cer证书的图示

![](http://i13.tietuku.com/c311b072de5eb077.png)

- 创建打包证书的`provision描述文件`的图示
	- 描述文件，确定如下:
		- 打包`哪一个app`
		- 打包后的App，可以安装在`哪些iOS设备`

![](http://i13.tietuku.com/7e4a0a2f2fb729b6.png)

- 打包App（归档）

![](http://i13.tietuku.com/63d5d10856d405a7.png)

- Archive成功后弹出的界面
	- 选择一、验证
	- 选择二、上传到AppStore
	- 选择三、导出为ipa文件

![](http://i13.tietuku.com/0fe8b90338ac668a.png)

- 导出为ipa文件有三个模式
	- 模式一、可以上传到AppStore的ipa。
	- 模式二、可以让`注册了UUID`的iOS设备安装。
	- 模式三、使用企业账号打包出ipa，可以让任意iOS设备安装，但是不能够提交到AppStore。

![](http://i13.tietuku.com/0ee389d47f808aba.png)


如果需要打包出，任意iOS设备都可以安装的ipa文件，需要使用`企业账号的打包证书`来打包出ipa文件，但是不能够上AppStore，仅作为内部测试。

****

###证书三、App发布的证书

- 首先创建发布证书cer，步骤与`创建打包证书`类似。

```
略...
```

- 然后创建发布证书对应的provision描述文件

![](http://i11.tietuku.com/b779cf29e2c6f800.png)

- 安装证书与描述文件

- 登陆`iTunes Connect`配置`即将发布的App` 或 查看`已上传App状态`

![](http://i11.tietuku.com/0ac7510d555c08bf.png)

- 选择`我的 App`进入查看App状态

![](http://i11.tietuku.com/7399792cf64d3b9d.png)

- 选择`我的 App`，还可以新建App

![](http://i11.tietuku.com/9ef1cd3389ed402d.png)

- 填写待发布App的描述信息

![](http://i13.tietuku.com/5a0faa00cb7961a4.png)

- 填写应用审核联系方式、以及测试账号

![](http://i13.tietuku.com/423b92e89d2c526b.png)

- 上传ipa

![](http://i13.tietuku.com/8794ef5b9e3d110c.png)

![](http://i13.tietuku.com/e2334d21d87ebe7f.png)

****

###Mac机器本地申请一个Certificate Signing Request（CSR）文件，然后上传这个csr文件，向苹果申请一个cer证书（开发者证书）


- 这个过程呢，实际上是生成了一对公钥和私钥，保存在我们电脑上的钥匙串中
- 代码的签名也就是使用这种基于非对称密钥的加密方式
	- 用私钥进行签名
	- 用公钥进行验证
我们的钥匙串中存储着相关的公钥和私钥，而证书里则包含了公钥
- 当证书的私钥丢失后，只能revoke删掉这个证书，重新申请一个新的证书

****

###Provisioning Profile

> 一个Provisioning Profile文件包含了刚刚我们上面讲的所有的内容：证书、App ID、设备


####试想一下，如果我们要打包或者在真机上运行一个应用程序

- 首先需要`使用开发者证书对我们的App进行代码签名`，用来标识这个应用程序是合法的、安全的、完整的等等
- 然后需要指明它的App ID，并且验证Bundle ID是否与其一致
- 还需要确定但其概念iOS设备是否具备调试的能力、以及调试这一个App Id对应的App程序的能力
- 而Provisioning Profile文件，就是将如上这些信息全部`打包`在一起
	- App Id
	- 具备调试的iOS设备UUID
	- 开发者证书（只有公钥）
- 而且这个Provisioning Profile文件会在`App打包`的时候会`嵌入 .ipa包`，如下图所示:

[![Snip20160318_7.md.png](http://img.ja.9xiqi.cn/2016/03/18/Snip20160318_7.md.png)](http://pic.9xiqi.cn/image/lfydG)

- 当App被安装到iOS设备时，会将App中的profile文件，拷贝到iOS设备上