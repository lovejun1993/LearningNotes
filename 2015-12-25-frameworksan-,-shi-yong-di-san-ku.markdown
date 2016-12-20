---
layout: post
title: "Framework三、使用第三库"
date: 2015-12-25 18:21:27 +0800
comments: true
categories: 
---

###使用 `Cocoapods`来管理framework的第三方依赖库.

http://guides.cocoapods.org/making/using-pod-lib-create

***

###创建 `库` 工程

```
pod lib create lib工程名字
```

***

###然后根据提示步骤，选择如下

- 程序语言
- 是否添加Test Framework
- 是否添加UI Testing
- 工程的前缀是什么

****

###选择完毕之后Cococpods自动创建Lib工程，添加pods配置文件 `xcworkspace`

![](http://i4.tietuku.com/a5949d9d84a1ce4f.png)

****

###然后创建一个git仓库，来管理这个Lib工程

- 使用github创建一个仓库
- 然后将远程git仓库，clone到本地
- 然后将前面创建的Lib工程目录下的所有文件拷贝到本地仓库目录下
- 最后push到远程git仓库

```
git add.
git commit -m "info"
git push origin master
```

****

###那么一个问题来了，就是`库工程`和`主App工程`引用了同一个第三方开源库，会造成编译报错

- 如: 库工程使用了AFN、主App工程也使用了AFN

```
那么需要将 库工程中的 所有的AFN源码文件，都加上一个 前缀，来区分开来.
```

- 上面是我们自己手动修改，太麻烦。可以使用`CocoaPods`来给我们完成上述的 `添加前缀` 的体力劳动

```
CocoaPods打包库工程的时候，会默认将库工程依赖的所有 第三方库中的源码文件，都统一添加一个 之前创建Lib工程时 指定的 前缀.
```

- 最后`CocoaPods`将AFN的源码打包成一个`库`，添加到App工程

![](http://i4.tietuku.com/31eb396391d2ccac.png)

****

###进入 Lib工程的根目录下，编辑`MyLib.podspec`添加 库工程依赖的其他第三库

```
Pod::Spec.new do |s|
  s.name             = "MyLib"
  s.version          = "0.1.0"
  s.summary          = "A short description of MyLib."
  
  s.description 	    = "具体的描述信息..."
  
  s.homepage         = "https://github.com/<GITHUB_USERNAME>/MyLib"
  # s.screenshots     = "www.example.com/screenshots_1", "www.example.com/screenshots_2"
  s.license          = 'MIT'
  s.author           = { "xiongzenghui" => "xiongzenghui@myhome.163.com" }
  s.source           = { :git => "https://github.com/<GITHUB_USERNAME>/MyLib.git", :tag => s.version.to_s }
  # s.social_media_url = 'https://twitter.com/<TWITTER_USERNAME>'

  s.platform     = :ios, '7.0'
  s.requires_arc = true
	
  # 存放 所有源文件 的目录	Pod/Classes/
  s.source_files = 'Pod/Classes/**/*'
  
  # 存放所有资源文件 Pod/Assets/
  s.resource_bundles = {
    'MyLib' => ['Pod/Assets/*.png']
  }

  # 指定 头文件 的 搜索位置
  # s.public_header_files = 'Pod/Classes/**/*.h'
  
  # 依赖的系统framework
  # s.frameworks = 'UIKit', 'MapKit'
  
  # 依赖的其他自定义库
  # s.libraries  = 'xxx.framework', 'xxx.framework', 'xxx.framework'
  
  # 依赖的第三方开源库，如果有多个需要填写多个 s.dependency
  s.dependency 'AFNetworking', '~> 2.6.3'
  
end
```

****

###编写完之后，验证`podspec文件`是否正确是使用命令  在podspec所在目录下执行  `pod lib lint`

- 成功之后的输出

![](http://i4.tietuku.com/20155f67fd1872b3.png)

- 如果失败，按照提示完成所有的操作即可

- 注意: `一定要先创建远程git仓库，来管理Lib工程`

***

###进入 `Lib工程/Example/`，执行`pod install` 或 `pod install --no-repo-update`

```
找到包含 Podfile文件 的目录
```

![](http://i4.tietuku.com/61e9b6052226bfd1.png)

****

###添加库工程的源码文件

- 添加源码的路径 `Xcode工程路径`

![](http://i4.tietuku.com/c6da7bae86f05970.png)

- 添加源码的路径 `本地路径`

```
MyLib
  ├── .travis.yml
  ├── _Pods.xcproject
  ├── Example
  │   ├── MyLib
  │   ├── MyLib.xcodeproj
  │   ├── MyLib.xcworkspace
  │   ├── Podfile
  │   ├── Podfile.lock
  │   ├── Pods
  │   └── Tests
  ├── LICENSE
  ├── MyLib.podspec
  ├── Pod
  │   ├── Assets	添加所有的资源文件
  │   └── Classes   添加所有的源码文件
  │     └── RemoveMe.[swift/m]
  └── README.md
```

- 在Xcode工程中的: `Pods/Development Pods/MyLib/Pod/Classes`目录下，添加 `库源码`

```
#import <Foundation/Foundation.h>


@interface NetManager : NSObject

+ (void)log;

@end
```

```
#import "NetManager.h"

//使用AFN第三方库
#import <AFNetworking/AFNetworking.h>

@implementation NetManager

+ (void)log {
    AFHTTPSessionManager *manager = [AFHTTPSessionManager manager];
    NSLog(@"manager = %@\n", manager);
}

@end
```

- 如果要添加`私有`的库源码，不向外暴露

![](http://i4.tietuku.com/1708b8c777a4fa7a.png)


****

###只要 `新增` 类、资源文件、依赖的三方库、`podspec文件`修改、都需要重新运行`pod install --no-repo-update`来更新配置信息

****

###在`App工程`中的代码中使用`库工程的代码`

```
#import "XZHViewController.h"

//导入库工程的代码
#import <MyLib/NetManager.h>


@interface XZHViewController ()

@end

@implementation XZHViewController

- (void)viewDidLoad
{
    [super viewDidLoad];
    
    //使用库代码
    [NetManager log];
}

@end
```

****

###库代码修改完毕之后提交到远程仓库，一定要打tag，然后将tag也推送到远程仓库

- 提交库源码

```
git add .

当前提交的版本
git commit -a -m 'v0.1.0' 

添加远端仓库（clone下来的工程可以不用执行）
git remote add origin git地址

推送到远程仓库
git push origin master
```

- 给此次提交打一个tag，以及将tag推到服务器

```
使用当前版本号打一个tag
git tag -a 0.1.0 -m 'v0.1.0'

推送tag到远端仓库
git push --tags     
```

****

###库代码提交到远程git仓库之后，编辑库工程根目录下的`podspec文件`

- 设置远程git仓库、以及此次打包需要的哪一个tag

```
s.source           = { :git => "https://github.com/xiongzenghuidegithub/MyLib.git", :tag => "0.1.0"}
```

****

###验证`podspec文件`是否编写正确，在库工程的根目录下执行

```
pod lib lint
```

***

###打包生成类库

####安装打包插件 `cocoapods-packager`，这个插件对引用的三方库进行`重命名`很好的解决了类库命名冲突的问题

```
sudo gem install cocoapods-packager
```

####打包成 .a 或 .framework

- 打包成 `.a`

```
pod package BZLib.podspec --library --force
```

- 打包出 `.framework`，不添加 `--library`参数即可

```
pod package BZLib.podspec --force
```

![](http://i4.tietuku.com/133d9a88e7ab8ada.png)

- 在库工程的根目录下，可以找到新增加的目录，其目录下就是生成的framework文件，如下所示:

![](http://i4.tietuku.com/65c4504d75b88441.jpg)

![](http://i4.tietuku.com/6dc1e0876f1953a4.png)

![](http://i4.tietuku.com/197ff814ce2f27b1.png)

- 注意，生成的framework，默认由CocoaPods完成了`多cpu架构`的编译，已经完成支持所有的cpu架构

```
i386, x86_64, armv7, armv7s, arm64
```

需要特别强调的是，该插件通过对引用的三方库进行重命名很好的解决了类库命名冲突的问题。

OK，到此位置就可以轻松的生成framework了。

****

###如果需要使用 `pod install`来安装我们的库

****

###首先需要创建一个用于存放 `podspec文件` 的远程git仓库

```
https://git.coding.net/xiongzenghui/SpecRepo.git
```

****

###添加一个自己的所有PodSpect文件的`远程git仓库`，然后执行如下命令将自己的SpecRepo添加到系统CocoaPods的Spec库

```
pod repo add XiongSpecRepo https://git.coding.net/xiongzenghui/SpecRepo.git
```

****

###进入 `~/.cocoapods/repos`目录下，查看我们添加的spec仓库

```
xiongzenghuideMacBook-Pro:MyLib xiongzenghui$ cd ~/.cocoapods/repos
xiongzenghuideMacBook-Pro:repos xiongzenghui$ ls
XiongSpecRepo master
```

如果其他人员也需要操作这个spec仓库，需要给予`权限`.

****

###将库工程下的`PodSpect`文件提交到`本地SpecRepo`，如下命令同时将这个podspec文件也Push到`远程SpecRepo`

```
pod repo push XiongSpecRepo MyLib.podspec
```

****

###进入到 `~/.cocoapods/repos/XiongSpecRepo/`查看添加的

```
MyLib
```

****

###然后执行`pod search 库名`会从本地系统CocoaPods的所有的SpecRepo搜索这个库，最终查询到我们自己的SpecRepo `XiongSpecRepo`


```
xiongzenghuideMacBook-Pro:XiongSpecRepo xiongzenghui$ pod search MyLib

-> MyLib (0.1.1)
   测试pod生成framework的YohunlUtilsPod.
   pod 'MyLib', '~> 0.1.1'
   - Homepage: https://github.com/xiongzenghuidegithub/MyLib
   - Source:   https://github.com/xiongzenghuidegithub/MyLib.git
   - Versions: 0.1.1 [XiongSpecRepo repo]
```

####注意: 团队内其他项目组需要使用，那么必须手动将之前的`XiongSpecRepo`添加到本地CocoaPods的Spec系统库中.

****

###小结主要步骤:

- 建立 `两个` git仓库
	- 远程git仓库1，存放 `库工程源码`
	- 远程git仓库2，存放 `所有的 PodSpec 文件`

- `PodSpec`文件用于生成库文件（.a 或 .framework）时，读取的配置文件
	- 指定从哪个git远程库，拉取 库源码
	- 获取哪个tag的源码
	- 以及源码位置
	- 依赖系统framework
	- 依赖的其他静态库
	- 依赖的第三方开源库

- `pod lib create 库工程名字` 创建一个库工程
	

- 然后编写spec文件 `库工程名字.podspec`
	- 做出如上所有修改之后
	- 执行命令 `pod lib lint`监测podspec文件是否语法正确

- 执行 `pod install`下载依赖类库、以及更新系统所有配置（各种库文件的依赖..）
	- 注意: 每一次库工程发生变动之后，都需要执行

- 然后编写 库工程的 所有源码

- 提交`库工程源码`到git远程仓库、以及提交`tag`到git远程仓库
	- push 所有的源码
	- 对此次提交设置一个tag
	- push tag

- 再次编写`PodSpec`文件
	- 修改对应的 库源码的 git地址
	- 修改对应的 库源码的 tag

- 然后将 `库工程的 PodSpec 文件` 提交到CocoaPods的Spec库

- 本机直接可以通过 `pod install 库名`安装库

- 其他机器需要首先添加之前的`PodSpec git 库`到自己的本地CocoaPods的Spec库，然后再执行上面的安装命令

***

###可以看到CocoaPods其实是

- 首先将github or 其他git仓库中，所有的 `podspec文件`，拷贝到`本地的Spec Repo`中
- 然后再查询`本地podspec文件`找到要下载源码库的 `git地址、以及tag、依赖关系` 等

****

学习来源: http://www.cnblogs.com/brycezhang/p/4117180.html