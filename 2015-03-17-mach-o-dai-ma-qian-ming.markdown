---
layout: post
title: "Mach-O 代码签名"
date: 2015-03-17 17:44:08 +0800
comments: true
categories: 
---

###一直对iOS安全相关的东西接触的比较少，在有一些面试中也碰过几次，觉得安全相关的东西还是要会。


***

###我们写的App程序，最后经过编译打包生成一个ipa文件

- 这个ipa文件其实就是一个压缩包

- ipa解压文件夹/Payload/xxx.app包/存放了所有程序资源如下
	- 主要的二进制可执行文件 Mach-O文件
	- 各种资源文件（图片、数据、xib、bundle资源包）
	- _CodeSignature记录所有文件的签名

- 最最要的就是.app内的`可执行的Mach-O格式的文件`
	- Mach-O的文件的结构组成可以参看前面一片记录

- 其Mach-O可执行二进制文件包含的内容:
	- App程序代码（.h与.m）编译后的 .o文件
	- 以及对Mach-O文件的`代码签名`

- 所有类型的可执行二进制文件**必须经过Apple的代码签名**之后，才可以运行在iOS系统中

	- **所有的文件其实都需要签名之后，才能被iOS系统使用**
	- Mach-O格式的二进制文件
	- .a / .framework 静态库二进制文件
	- .dylib/.framework 动态库二进制文件

- 如果当iOS系统被**越狱**之后
	- 可以运行没有通过代码签名的可执行文件
	- 突破沙盒的限制，可以随意访问其他App的沙盒下的数据

所以这里引出一个问题

> 对 可执行文件 进行 代码签名.

****

###代码签名机制的核心: 开发者证书，一个公钥，一个私钥

- 必须使用一个`开发者证书`来完成对文件的代码签名

- 但是这个证书有要求，就是你必须要有这个证书的`私钥`

- 而所有你具有私钥的证书，都在`钥匙串访问`程序中的`我的证书`选项下

![](http://i4.tietuku.cn/d63a2cea73b6189e.png)


- 如果是要备份（导出）这个持有私钥的证书，注意要记得`展开`证书那一条`显示出私钥`并将`两行`都选中

![](http://i4.tietuku.cn/5d79c8857e14b512.png)

- 快速地显示出你的系统中能用来对代码进行签名的认证的方法的命令如下:

```
security find-identity -v -p codesigning
```

***

###数字签名（digital signature）

- 对指定的数据内容使用`哈希算法`，得到一个`固定长度`的信息摘要
- 然后再使用 `私钥` （注意必须是私钥）对该摘要加密
- 得到的`加密数据`就是所谓的数字签名

###App中所有文件签名流程

- 首先是苹果自己持有的WWDR私钥，对我们申请得到的`开发者证书`进行签名
	- 防止开发者证书被篡改
- 我们在打包ipa包时，使用我们自己的`开发者证书的私钥`对App进行签名
	- 过程中，苹果会验证开发者证书是否被篡改
	- 如果证书正常，那么苹果使用我们证书的私钥完成签名操作

****

###证书到底是什么？

- 一个证书是一个`公钥`加上许多`附加信息`.
	
- 一个证书的组成结构为如下:
	- 证书包含的数据部分
		- **开发者证书的 公钥**
		- `私钥`被安装到Mac机器的本地系统的钥匙串中
		- 以及其他数据信息
			- 用户个人信息
			- 证书颁发机构信息
			- 证书有效期等信息

	- 证书自己被苹果进行的签名
		- 这个签名，是被某个`认证机构`（Certificate Authority 简称 CA）进行签名认证过的，认证这个证书中的信息是准确无误的
		- 而对于iOS应用开发来说，这个认证机构，就是苹果的认证部门 Apple Worldwide Developer Relations CA.
		- 认证的签名，是有一个有效期的，且直接对比当前计算机设置的时间.
		- 这也是为什么将系统时间设定到过去会对 iOS 造成多方面破坏的原因之一.

****

###图示证书组成结构的

![](http://i13.tietuku.cn/cc9c6ad5ad0df35b.png)

注意:上述WWDR机构 >>> Apple Worldwide Developer Relations Certification Authority（苹果全球开发者证书认证机构）

****

###苹果官方对我们的开发者证书进行签名的过程

- WWDR将上述`证书本身内容`使用`哈希算法`得到一个`固定长度`的信息摘要
	- hash(证书用户信息, 颁发机构信息, 证书有效期 ... )

- 然后使用`WWDR自己的私钥`对该`信息摘要`加密生成`数字签名`

![](http://ww3.sinaimg.cn/large/74311666jw1f20vp2tpevj211m0c0dgs.jpg)

****

###向苹果申请一个开发者证书的过程

- 本地Mac机器上创建一个csr文件（Certificate Signing Request）

- 提交给苹果的 Apple Worldwide Developer Relations Certification Authority(WWDR)证书认证中心进行签名

- 最后从苹果开发者中心下载得到一个cer证书文件，并且双击安装到本地
	- 该证书中包含一个公钥
	- 该公钥对应当前`开发者用户自己的 公钥`

- 这个过程中还会产生一个对应`开发者用户自己的 私钥`
	- **私钥不会保存在证书中**
	- 私钥而是直接安装到Mac机器系统上的钥匙串中

- `证书`存放在keychain中，而`私钥`在所属证书的扩展栏目下面所示

****

###证书存在的意义

- 证书本身只是一个中间媒介（容器），用来包含数据与数字签名
	- 数据内容
		- **开发者证书的 公钥**
		- 等等数据
	- 数字签名
		- 用于校验当前证书是否合法
		- 如果证书中的数据内容（公钥）被篡改掉了
		- 那么如下两个摘要信息肯定是不相同的
			- 摘要信息A >>> 证书内容，通过hash算法得到
			- 摘要信息B >>> iOS系统采用内置的`WWDR公钥`，解密证书中的签名，得到的摘要
				- `证书中的数字签名`: 是在申请开发者证书时，由苹果官方使用WWDR私钥加密后得到的，并写入到证书文件中的
		- **最终决定是否使用证书中包含 开发者自己的公钥**

- iOS系统对证书文件本身，其实并不关心
	- 最后关心的是，`是否应该使用`证书中包含的 `开发者自己的公钥`

- **iOS系统是如何得到开发者证书的了？**

****

###Mac机器上安装的开发者证书会拷贝到手机上

- 数字证书中只会带有`公钥`

- 数字证书会被包含在 `Provisioning` 描述文件中

- Provisioning文件，在打包ipa时，会由Xcode嵌入到ipa包中

[![Snip20160318_7.md.png](http://img.ja.9xiqi.cn/2016/03/18/Snip20160318_7.md.png)](http://pic.9xiqi.cn/image/lfydG)

- 而当ipa包被iOS设备安装的时候，又会将Provisioning文件拷贝到iOS设备中

- 总之，最后iOS系统是能够得到`开发者证书的 公钥`，而`开发者证书的  私钥`始终是存放在`Mac机器中的钥匙串访问`中

- **.app包内的所有文件，是根据 开发者证书的私钥 进行签名的**

- 那么只要有了`开发者证书的 公钥`，就可以验证（解密）签名是否合法

****

###证书的校验的过程

![](http://ww4.sinaimg.cn/large/74311666jw1f20xmp6hwsj21300f8gn8.jpg)

- **在iOS系统中已经内置了WWDR的的`公钥`**
- 先对证书内部的内容，按照上面的`哈希算法`得到一个`固定长度`的信息摘要A
- 然后再使用`公钥`对证书中的数字签名进行`解密`，得到解密后的信息摘要B
- 然后再比较 信息摘要A 与 信息摘要B，如果`内容`一致就说明该证书可信
- 在验证了证书是可信的以后
	- iOS系统就可以获取到证书中包含的`开发者自己的 公钥`（不是WWDR的公钥）
	- 并使用该`开发者公钥`来判断`代码签名`的可用性

****

###对于在iOS中用到的证书，会分成两种:

- 一个带有前缀 iPhone `Developer` 的证书
	- 用于使应用可以在你的测试设备上运行，真机调试
	
- 另一个带有前缀 iPhone `Distribution` 的证书
	- 用于将归档出来的ipa提交应用到 AppStore

- 一个证书的用途取决于它所包含的内部信息
	- Apple Developer Certificate (Submission) 
	- Apple Developer Certificate (Development)

- iOS系统会利用这个信息来判断你的应用是运行模式
	- 开发模式
	- 发布模式
	- 并据此判断以`切换`应用的`运行规则`

***

###只有同时具备`公钥与私钥`的开发者证书才能完成对二进制文件进行代码签名（对App中任何文件签名）

![](http://i12.tietuku.cn/504cbc3acfa7beff.png)

####开发者证书的公钥与私钥

- 开发者证书的 私钥 >>> 通过`加密`算法，给App内所有的文件进行代码签名
- 开发者证书的 公钥 >>> 通过`解密`算法，验证App内所有文件的签名是否合法，继而决定是否能在iOS系统中使用这个文件

###Xcode在打包ipa时，读取设置的Code Signing指向的本地Mac系统安装的开发者证书

- 通常我们在Xcode中真机测试或打包发布的时候，在Xcode中设置中选择使用哪一种签名

-  Xcode 只允许你在有限的选项中进行选择，这些选项都是你既拥有公钥也拥有私钥的证书
-  所以如果在选项中没有出现你想要的那一个，那么你需要检查的第一件事情就是你是否拥有这个证书的`私钥`
-  真机调试、测试，那么应该使用 `开发证书` 中的那一对秘钥来签名
-  发布应用，那么应该使用 `发布证书` 中的那一对秘钥来签名
- **签名过程本身是由命令行工具 codesign命令 来完成的**

####给没有设置过签名的App进行签名

```
codesign -s '证书名' Example.app
```

####重新设置签名，必须带上 -f 参数，这样 codesign 会用你选择的签名替换掉已经存在的那一个

```
codesign -f -s '证书名' Example.app
```

如果在 Xcode 中编译一个应用，这个应用构建完成之后会自动调用 codesign 命令进行签名

****

###对一个App进行签名，实际上是对编译生成的`可执行二进制文件（Unix excutable）`以及App中所有的资源文件，使用Mac机器本地安装的开发者证书中的`私钥`进行签名


> 对一个程序进行签名，会修改它的 主执行二进制文件（Mach-O格式文文件）.

- 通过Apple认证合法的证书对二进制文件进行签名
- 签名后的可执行二进制文件，iOS系统才能让其够执行的文件
- **iOS系统内核在调用二进制可执行文件之前，会检查其签名是否合法**
	- 通过检测可执行二进制文件中的`LC_CODE_SIGNATURE Segement段`是否有效和可信任
	- 越狱后的iOS系统，即禁止掉了检测如上的值，即可直接执行
- 在iOS系统中可以正常合法放行让其执行

****

###使用带有`私钥`的开发者证书，对二进制可执行文件签名后产生的信息，再次写入二进制文件中

> 设置签名的过程实际上会 `改动` 可执行文件的文件内容，将签名数据`写入`二进制文件中.

- 比如shell脚本进行签名后，对其签名的数据，存放在该脚本文件的`系统扩展属性`中
- Mach-O类型的二进制可执行文件进行签名后，对其签名的数据，会再次写入Mach-O格式文件中
	- 存放在 `LC_CODE_SIGNATURE` 这个 Segment段中
- 所以，OS X 和 iOS 上的`任何类型的 可执行 二进制文件`都可以被设置签名
	- 动态库、静态库
	- 命令行工具、脚本
	- .app 后缀的程序包

****

###用codesign能显示某个二进制文件的数字签名详细信息

```
codesign -dvvvv 二进制文件名
```

例子输出信息:

```
xiongzenghuideMacBook-Pro:Ape.app xiongzenghui$ codesign -dvvvv Ape

可执行二进制文件的路径
Executable=/Users/xiongzenghui/Desktop/猿题库 5.9.0/Payload/Ape.app/Ape

//App程序包名
Identifier=com.fenbi.ape.gz

//Mach-O格式文化，支持cpu的架构类型
Format=bundle with Mach-O universal (armv7 arm64)

//???
CodeDirectory v=20200 size=34960 flags=0x0(none) hashes=1739+5 location=embedded
Hash type=sha1 size=20
CDHash=8ba02908da7084d25c3a6fd2d80ffb384411e7da
Signature size=3487

//首先检查这三行
Authority=Apple iPhone OS Application Signing
Authority=Apple iPhone Certification Authority
Authority=Apple Root CA
Info.plist entries=39
TeamIdentifier=29QXUV376B
Sealed Resources version=2 rules=14 files=43
Internal requirements count=1 size=96
```

如上测试的二进制文件，就是Mach-O第一篇文章中的`Ape.app/Ape`文件

```
/Users/xiongzenghui/Desktop/猿题库 5.9.0/Payload/Ape.app/Ape
```

###用codesign列出一些有关 Example.app 的签名信息

```
codesign -vv -d Example.app
```

比如输出如下:

```
Executable=/Users/toto/Library/Developer/Xcode/DerivedData/Example-cfsbhbvmswdivqhekxfykvkpngkg/Build/Products/Debug-iphoneos/Example.app/Example  
Identifier=ch.kollba.example  
Format=bundle with Mach-O thin (arm64)  
CodeDirectory v=20200 size=26663 flags=0x0(none) hashes=1324+5 location=embedded  
Signature size=4336  
Authority=iPhone Developer: Thomas Kollbach (7TPNXN7G6K)  
Authority=Apple Worldwide Developer Relations Certification Authority  
Authority=Apple Root CA  
Signed Time=29.09.2014 22:29:07  
Info.plist entries=33  
TeamIdentifier=DZM8538E3E  
Sealed Resources version=2 rules=4 files=120  
Internal requirements count=1 size=184 
```

注意:

- 是对类似`/Users/xiongzenghui/Desktop/猿题库 5.9.0/Payload/Ape.app`执行如上命令


```
codesign -vv -d Ape.app
```

###对如上两个命令输出的信息中，优先查看的字段

- 第一件事是以 `Authority` 开头的 那三行

	- 第一行，告诉你到底是 `哪一个证书` 为这个 app 设置了签名
	- 第二行，我们使用给App签名的证书，又被`Apple Worldwide Developer Relations Certification Authority`这个证书签名
	- 第三行，`Apple Worldwide Developer Relations Certification Authority`这个证书，又最终被`Apple Root CA`设置了签名（苹果根证书）

- 第二件事，查看 `format` 行
	- Example.app 并`不单单`是一个可执行文件，它是一个`程序包（bundle）`
	- 只支持arm64 cpu架构
	- 从 Executable 中的路径信息你可以看出，这是一个以测试为目的的打包，所以是一个 `Mach-O thin` 的二进制文件

- 第三件事，查看 `Identifier` 行
	- 是我在 Xcode 中设置的 `bundle identifier` 程序包名

- 第四件事，查看 `TeamIdentifier`行
	- 标识我的工作组（系统会用这个来判断应用是否是由`同一个开发者`发布)
	- 此外用于发布应用的证书中也包含这种标识，这种标识在`区分同一名称下 的 不同证书`时非常有用


****

###对一个编译好的ipa检查其签名

> 依次找到 ipa/Payload/Ape.app

```
codesign --verify Example.app
```

执行后，如果没有任何输出，代表签名是完好的.

***

###一旦.app/Mach-O可执行二进制文化的签名发生修改

再执行上述检查命令时，就会输出如下

```
Example.app: main executable failed strict validation 
```

****

###.app包内所有类型的文件都会被签名，以确保是iOS系统认可的合法文件

- 各种资源文件
	- 国际化语言文件
	- XIB/NIB 文件
	- 存档文件(archives)
	- SSL证书文件
- 可执行二进制文件
	- 图片文件
	- 脚本文件
	- Mach-O格式文件

***

###如何达到为所有文件设置签名？

> 执行签名操作的过程中，会在程序包（.app包下）中新建一个叫做 _CodeSignatue/CodeResources 的文件

比如我将一个ipa拷贝到桌面后，依次获取得到上面目录的位置:

```
/Users/xiongzenghui/Desktop/猿题库 5.9.0/Payload/Ape.app/_CodeSignature/CodeResources文件
```

打开这个CodeResources文件看看里面的内容

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>files</key>
	<dict>
		<key>14767c52889c771@2x.jpg</key>
		<data>
		vMGaaOS9Kth2/0CVZaYyfF3hMqg=
		</data>
		<key>14767c5a49daa27@2x.jpg</key>
		<data>
		Hkuov4kJ5avSH2GcRvaMKpmYGg4=
		</data>
		<key>14767c5d992517c@2x.jpg</key>
		<data>
		aGyhVr6PmKVu5hqONz/rh3wxdbg=
		</data>
		<key>14767c610e896ad@2x.jpg</key>
		<data>
		9DeVQpnKQgbagn/KH57zqnnDPRw=
		</data>
		<key>14767c6425ed6b4@2x.jpg</key>
		<data>
		zYLp1RkCKH4CA7Dmzp4JBsCgjLw=
		</data>
		<key>AccountInfoItem.nib</key>
		<data>
		OicEumVrkzg4zhHnaNNJZyTq3aU=
		</data>
		<key>AppIcon60x60@2x.png</key>
		<data>
		QAO44VmEUSWT6wnA6pVJinz93ns=
		</data>
		<key>AppIcon60x60@3x.png</key>
		<data>
		ZOi2W9JGYReH1Vicq5ScG/mpM2w=
		</data>
		<key>AppIcon76x76@2x~ipad.png</key>
		<data>
		+A8V09qcT1X6hFKPUQjqx/kocs4=
		</data>
		<key>AppIcon76x76~ipad.png</key>
		<data>
		Ac2PB5XBHw46C3eGa61/11A2cdg=
		</data>
		<key>Assets.car</key>
		<data>
		PGjsyRbSC3m4iQVp5Zt9eAwPTP8=
		</data>
		<key>CourseConfigDD.dat</key>
		<data>
		mx9cfResHUe6YsbC4aDNRGEO0CE=
		</data>
		<key>DistrictChuzhong.dat</key>
		<data>
		nhL5RFQ3qTCX5JgvNr+lNDLXy3A=
		</data>
		<key>DistrictGaozhong.dat</key>
		<data>
		iJlDCMXStmZ7Enx1UxEI7DkYEDM=
		</data>
		<key>ExerciseReport.nib</key>
		<data>
		IFZEb+gFZ6+HzQFZwdsv3jkj5+w=
		</data>
		<key>HelveticaLT35Thin.ttf</key>
		<data>
		iBH+NaSBm9BpIwfcpO03F/pAxb0=
		</data>
		<key>Info.plist</key>
		<data>
		2WSQ1NlJqWH40CkW/JbyqHC17rg=
		</data>
		<key>LaunchGuide.nib</key>
		<data>
		JELArC4GW7iH5xpcZa53GpG5Fo0=
		</data>
		<key>LaunchImage-700-568h@2x.png</key>
		<data>
		Z8BB0hssm+yitXG5CzauUbfFBHg=
		</data>
		<key>LaunchImage@2x.png</key>
		<data>
		O++zOZb9yvP1MW9il8TQdMpnVc4=
		</data>
		<key>LaunchScreen.nib</key>
		<data>
		qhkkvoXiJFJoTyLsnPbPRpsQJ2U=
		</data>
		<key>MajorKeypointTreeDD.dat</key>
		<data>
		xd4SdFTvAmUx6vJApi0jewRCg8k=
		</data>
		<key>MockExerciseGuide.nib</key>
		<data>
		pIWTtennvm80M7TUOdBi7BLuebI=
		</data>
		<key>MockItemView.nib</key>
		<data>
		dsd5NJ/1aIWsz6Bwv+jK/y5RrU8=
		</data>
		<key>NimbusPhotos.bundle/gfx/default.png</key>
		<data>
		yQdhJSSWjA5+vRfebmjyD0MK8+8=
		</data>
		<key>NimbusPhotos.bundle/gfx/next.png</key>
		<data>
		vSy4Vr39udmaZKMJdU2zcezZRfA=
		</data>
		<key>NimbusPhotos.bundle/gfx/next@2x.png</key>
		<data>
		u4Im8LjCWGcbsYrbUrs+GZnf+0k=
		</data>
		<key>NimbusPhotos.bundle/gfx/previous.png</key>
		<data>
		qfkv/2uO2UoQcD2dynm9Umg6KA8=
		</data>
		<key>NimbusPhotos.bundle/gfx/previous@2x.png</key>
		<data>
		k5/GmSz4Z9A3rV5pHskx5yNXf1g=
		</data>
		<key>PkgInfo</key>
		<data>
		n57qDP4tZfLD1rCS43W0B4LQjzE=
		</data>
		<key>PromotionDD.dat</key>
		<data>
		b30GWu/m5v5FBmr7vScb3KUcywQ=
		</data>
		<key>QuickExerciseGuide.nib</key>
		<data>
		mNRIt55rQ06ux0xYfwsYSHqmCdk=
		</data>
		<key>SC_Info/Manifest.plist</key>
		<data>
		5ncRuCR5wUbZMYk5yaXxEqiHHp0=
		</data>
		<key>SoupText.dat</key>
		<data>
		UuFg44I39CG6XsumGpMtbmN8EIY=
		</data>
		<key>SubjectDD.dat</key>
		<data>
		BZShljREWsl63QZZf+ESRh9Vjns=
		</data>
		<key>TencentOpenApi_IOS_Bundle.bundle/error.png</key>
		<data>
		lE6HVRWTNcbl/H/551a0SJGgvDU=
		</data>
		<key>TencentOpenApi_IOS_Bundle.bundle/local.html</key>
		<data>
		0qea+8Co8p/V0X2rKQGJo9/3B98=
		</data>
		<key>TencentOpenApi_IOS_Bundle.bundle/qqicon.png</key>
		<data>
		bKCi641iRnJMix2uh9xNGwBODt4=
		</data>
		<key>TencentOpenApi_IOS_Bundle.bundle/success.png</key>
		<data>
		HzVLiMbCKqmJewuPsHSehTjKXCY=
		</data>
		<key>UserReport.nib</key>
		<data>
		vhxk1FZ1UZmGcUxrBcYHlgVb/FI=
		</data>
		<key>UserSubjectDD.dat</key>
		<data>
		xsE2Sb2UUDGSXp8azQnXruuUZ/E=
		</data>
		<key>apetex-no-ch.zip</key>
		<data>
		PIyxh3dkQ3RbuVq6Q+4j2QLs8+o=
		</data>
		<key>archived-expanded-entitlements.xcent</key>
		<data>
		C6ZN6bCzPEqg7ZAHiq2MKYAJuPg=
		</data>
		<key>beep-beep.aiff</key>
		<data>
		w9mTKEUfkR76cIpX+U3+bMki22E=
		</data>
		<key>fbw.dat</key>
		<data>
		za0sGkUjo5AFoCu6rP36LRBgutM=
		</data>
	</dict>
	<key>files2</key>
	<dict>
		<key>14767c52889c771@2x.jpg</key>
		<data>
		vMGaaOS9Kth2/0CVZaYyfF3hMqg=
		</data>
		<key>14767c5a49daa27@2x.jpg</key>
		<data>
		Hkuov4kJ5avSH2GcRvaMKpmYGg4=
		</data>
		<key>14767c5d992517c@2x.jpg</key>
		<data>
		aGyhVr6PmKVu5hqONz/rh3wxdbg=
		</data>
		<key>14767c610e896ad@2x.jpg</key>
		<data>
		9DeVQpnKQgbagn/KH57zqnnDPRw=
		</data>
		<key>14767c6425ed6b4@2x.jpg</key>
		<data>
		zYLp1RkCKH4CA7Dmzp4JBsCgjLw=
		</data>
		<key>AccountInfoItem.nib</key>
		<data>
		OicEumVrkzg4zhHnaNNJZyTq3aU=
		</data>
		<key>AppIcon60x60@2x.png</key>
		<data>
		QAO44VmEUSWT6wnA6pVJinz93ns=
		</data>
		<key>AppIcon60x60@3x.png</key>
		<data>
		ZOi2W9JGYReH1Vicq5ScG/mpM2w=
		</data>
		<key>AppIcon76x76@2x~ipad.png</key>
		<data>
		+A8V09qcT1X6hFKPUQjqx/kocs4=
		</data>
		<key>AppIcon76x76~ipad.png</key>
		<data>
		Ac2PB5XBHw46C3eGa61/11A2cdg=
		</data>
		<key>Assets.car</key>
		<data>
		PGjsyRbSC3m4iQVp5Zt9eAwPTP8=
		</data>
		<key>CourseConfigDD.dat</key>
		<data>
		mx9cfResHUe6YsbC4aDNRGEO0CE=
		</data>
		<key>DistrictChuzhong.dat</key>
		<data>
		nhL5RFQ3qTCX5JgvNr+lNDLXy3A=
		</data>
		<key>DistrictGaozhong.dat</key>
		<data>
		iJlDCMXStmZ7Enx1UxEI7DkYEDM=
		</data>
		<key>ExerciseReport.nib</key>
		<data>
		IFZEb+gFZ6+HzQFZwdsv3jkj5+w=
		</data>
		<key>HelveticaLT35Thin.ttf</key>
		<data>
		iBH+NaSBm9BpIwfcpO03F/pAxb0=
		</data>
		<key>LaunchGuide.nib</key>
		<data>
		JELArC4GW7iH5xpcZa53GpG5Fo0=
		</data>
		<key>LaunchImage-700-568h@2x.png</key>
		<data>
		Z8BB0hssm+yitXG5CzauUbfFBHg=
		</data>
		<key>LaunchImage@2x.png</key>
		<data>
		O++zOZb9yvP1MW9il8TQdMpnVc4=
		</data>
		<key>LaunchScreen.nib</key>
		<data>
		qhkkvoXiJFJoTyLsnPbPRpsQJ2U=
		</data>
		<key>MajorKeypointTreeDD.dat</key>
		<data>
		xd4SdFTvAmUx6vJApi0jewRCg8k=
		</data>
		<key>MockExerciseGuide.nib</key>
		<data>
		pIWTtennvm80M7TUOdBi7BLuebI=
		</data>
		<key>MockItemView.nib</key>
		<data>
		dsd5NJ/1aIWsz6Bwv+jK/y5RrU8=
		</data>
		<key>NimbusPhotos.bundle/gfx/default.png</key>
		<data>
		yQdhJSSWjA5+vRfebmjyD0MK8+8=
		</data>
		<key>NimbusPhotos.bundle/gfx/next.png</key>
		<data>
		vSy4Vr39udmaZKMJdU2zcezZRfA=
		</data>
		<key>NimbusPhotos.bundle/gfx/next@2x.png</key>
		<data>
		u4Im8LjCWGcbsYrbUrs+GZnf+0k=
		</data>
		<key>NimbusPhotos.bundle/gfx/previous.png</key>
		<data>
		qfkv/2uO2UoQcD2dynm9Umg6KA8=
		</data>
		<key>NimbusPhotos.bundle/gfx/previous@2x.png</key>
		<data>
		k5/GmSz4Z9A3rV5pHskx5yNXf1g=
		</data>
		<key>PromotionDD.dat</key>
		<data>
		b30GWu/m5v5FBmr7vScb3KUcywQ=
		</data>
		<key>QuickExerciseGuide.nib</key>
		<data>
		mNRIt55rQ06ux0xYfwsYSHqmCdk=
		</data>
		<key>SC_Info/Manifest.plist</key>
		<data>
		5ncRuCR5wUbZMYk5yaXxEqiHHp0=
		</data>
		<key>SoupText.dat</key>
		<data>
		UuFg44I39CG6XsumGpMtbmN8EIY=
		</data>
		<key>SubjectDD.dat</key>
		<data>
		BZShljREWsl63QZZf+ESRh9Vjns=
		</data>
		<key>TencentOpenApi_IOS_Bundle.bundle/error.png</key>
		<data>
		lE6HVRWTNcbl/H/551a0SJGgvDU=
		</data>
		<key>TencentOpenApi_IOS_Bundle.bundle/local.html</key>
		<data>
		0qea+8Co8p/V0X2rKQGJo9/3B98=
		</data>
		<key>TencentOpenApi_IOS_Bundle.bundle/qqicon.png</key>
		<data>
		bKCi641iRnJMix2uh9xNGwBODt4=
		</data>
		<key>TencentOpenApi_IOS_Bundle.bundle/success.png</key>
		<data>
		HzVLiMbCKqmJewuPsHSehTjKXCY=
		</data>
		<key>UserReport.nib</key>
		<data>
		vhxk1FZ1UZmGcUxrBcYHlgVb/FI=
		</data>
		<key>UserSubjectDD.dat</key>
		<data>
		xsE2Sb2UUDGSXp8azQnXruuUZ/E=
		</data>
		<key>apetex-no-ch.zip</key>
		<data>
		PIyxh3dkQ3RbuVq6Q+4j2QLs8+o=
		</data>
		<key>archived-expanded-entitlements.xcent</key>
		<data>
		C6ZN6bCzPEqg7ZAHiq2MKYAJuPg=
		</data>
		<key>beep-beep.aiff</key>
		<data>
		w9mTKEUfkR76cIpX+U3+bMki22E=
		</data>
		<key>fbw.dat</key>
		<data>
		za0sGkUjo5AFoCu6rP36LRBgutM=
		</data>
	</dict>
	<key>rules</key>
	<dict>
		<key>^</key>
		<true/>
		<key>^(Frameworks/[^/]+\.framework/|PlugIns/[^/]+\.appex/|PlugIns/[^/]+\.appex/Frameworks/[^/]+\.framework/|())SC_Info/[^/]+\.(sinf|supf|supp)$</key>
		<dict>
			<key>omit</key>
			<true/>
			<key>weight</key>
			<integer>10000</integer>
		</dict>
		<key>^.*\.lproj/</key>
		<dict>
			<key>optional</key>
			<true/>
			<key>weight</key>
			<real>1000</real>
		</dict>
		<key>^.*\.lproj/locversion.plist$</key>
		<dict>
			<key>omit</key>
			<true/>
			<key>weight</key>
			<real>1100</real>
		</dict>
		<key>^Watch/[^/]+\.app/(Frameworks/[^/]+\.framework/|PlugIns/[^/]+\.appex/|PlugIns/[^/]+\.appex/Frameworks/[^/]+\.framework/)SC_Info/[^/]+\.(sinf|supf|supp)$</key>
		<dict>
			<key>omit</key>
			<true/>
			<key>weight</key>
			<integer>10000</integer>
		</dict>
		<key>^version.plist$</key>
		<true/>
	</dict>
	<key>rules2</key>
	<dict>
		<key>.*\.dSYM($|/)</key>
		<dict>
			<key>weight</key>
			<real>11</real>
		</dict>
		<key>^</key>
		<dict>
			<key>weight</key>
			<real>20</real>
		</dict>
		<key>^(.*/)?\.DS_Store$</key>
		<dict>
			<key>omit</key>
			<true/>
			<key>weight</key>
			<real>2000</real>
		</dict>
		<key>^(Frameworks/[^/]+\.framework/|PlugIns/[^/]+\.appex/|PlugIns/[^/]+\.appex/Frameworks/[^/]+\.framework/|())SC_Info/[^/]+\.(sinf|supf|supp)$</key>
		<dict>
			<key>omit</key>
			<true/>
			<key>weight</key>
			<integer>10000</integer>
		</dict>
		<key>^(Frameworks|SharedFrameworks|PlugIns|Plug-ins|XPCServices|Helpers|MacOS|Library/(Automator|Spotlight|LoginItems))/</key>
		<dict>
			<key>nested</key>
			<true/>
			<key>weight</key>
			<real>10</real>
		</dict>
		<key>^.*</key>
		<true/>
		<key>^.*\.lproj/</key>
		<dict>
			<key>optional</key>
			<true/>
			<key>weight</key>
			<real>1000</real>
		</dict>
		<key>^.*\.lproj/locversion.plist$</key>
		<dict>
			<key>omit</key>
			<true/>
			<key>weight</key>
			<real>1100</real>
		</dict>
		<key>^Info\.plist$</key>
		<dict>
			<key>omit</key>
			<true/>
			<key>weight</key>
			<real>20</real>
		</dict>
		<key>^PkgInfo$</key>
		<dict>
			<key>omit</key>
			<true/>
			<key>weight</key>
			<real>20</real>
		</dict>
		<key>^Watch/[^/]+\.app/(Frameworks/[^/]+\.framework/|PlugIns/[^/]+\.appex/|PlugIns/[^/]+\.appex/Frameworks/[^/]+\.framework/)SC_Info/[^/]+\.(sinf|supf|supp)$</key>
		<dict>
			<key>omit</key>
			<true/>
			<key>weight</key>
			<integer>10000</integer>
		</dict>
		<key>^[^/]+$</key>
		<dict>
			<key>nested</key>
			<true/>
			<key>weight</key>
			<real>10</real>
		</dict>
		<key>^embedded\.provisionprofile$</key>
		<dict>
			<key>weight</key>
			<real>20</real>
		</dict>
		<key>^version\.plist$</key>
		<dict>
			<key>weight</key>
			<real>20</real>
		</dict>
	</dict>
</dict>
</plist>
```

- 这个文件中，是一个 plist 格式文件，其组成结构如下

| key        | type           | value  | 用途 |
| :---------: |:-------:| :-------:| :-------: |
| Root | Dictionary | 4 items| plist文件根节点 |
| files | Dictionary | 45 items | Mac os X 10.9.5之`前`，保存被签名文件的字典 |
| files2 | Dictionary | 43 items | Mac os X 10.9.5之`后`，保存被签名文件的字典 |
| rules | Dictionary | 6 items | Mac os X 10.9.5之`前`，签名文件的规则 |
| rules2 | Dictionary | 14 items | Mac os X 10.9.5之`后`，签名文件的规则 |

- 该plist文件包含的两种数据:

	- files/files2字典、存储了被签名的程序包内，所有文件对应的签名数值
		- key: 文件名
		- data: 签名数值
	- rules/rules2字典、还包含了一系列`规则`
		- 决定了哪些资源文件应当被设置签名

****

###Mac OS X 10.9.5 之前与之后，文件签名的变化

- rules 和 files 是为老版本准备
	- 通过在`被设置签名的程序包`中添加一个名为 `ResourceRules.plist` 的文件
	- 这个plist文件，会规定`哪些资源文件`在`检查代码签名`是否完好时应该被`忽略`
- files2 和 rules2是为新的`第二版`的代码签名准备的
	- 新版本中你无法再将某些资源文件`排除`在代码签名之外
	- 新版本的代码签名中，所有的文件都必须设置签名，不再可以有例外
		- 代码文件
		- 资源文件
		- 脚本文件

****

###查看一个应用被授予的哪些权限

> 授权文件（entitlements）

举例，进入到 `/Users/xiongzenghui/Desktop/猿题库 5.9.0/Payload/`

执行如下命令

```
codesign -d --entitlements - Ape.app
```

****

###查看 描述文件（provisioning file）的内容

- Xcode 将从开发者中心下载的全部配置文件都放在了这里

```
~/Library/MobileDevice/Provisioning Profiles
```

- 以XML格式查看某一个 provisioning文件内容

```
security cms -D -i example.mobileprovisio
```

****

###小结:

- Mac机器本地安装的开发者证书中的`私钥`用来在本地对打包后的ipa包内的 .app 进行代码签名

- 在打包ipa包时，Xcode默认会将 开发者证书（只包含`公钥`）打包到 provisioning描述文件中，而最后provisioning描述文件又会被嵌入到ipa包中
	- 在App安装时，provisioning描述文件又会被拷贝到iOS设备中
	- 便于后期读取开发者证书中的公钥，进行代码验证签名

- iOS系统如何确认读取的开发者证书是否合法了？（可能会被篡改过）
	- 证书中包含一个数字签名，这个数字签名是由 WWDR（苹果认证中心）使用自己的 `私钥`加密过的
	- **WWDR私钥，只有苹果官方自己持有**
	- WWDR公钥，默认内置在iOS系统中
	- iOS系统通过使用内置的`WWDR公钥`对证书中被`WWDR私钥`加密后的数字签名，进行解密得到一个解密后的消息摘要A
	- 然后又对证书内部的数据内容，使用同样的哈希算法得到一个固定长度的消息摘要B
	- 然后对比这两个消息摘要，如果内容完全一致，就说明该证书是合法的、完整的

- 然后取出证书内部的`公钥`，用于对 .app包内所有被签名过的文件，进行签名的验证（解密）

所以说，当iOS设备非越狱的情况下，只有通过苹果官方认可的开发者证书私钥进行签名后的App，才能运行在iOS设备上的系统中，即便证书的公钥被篡改，也没任何作用，因为最终的WWDR私钥只存放在苹果官方服务器上，不会对外开放.



****

学习来源: 

- http://objccn.io/issue-17-2/
- http://www.cocoachina.com/ios/20141017/9949.html
- 《iOS安全攻防与实战》