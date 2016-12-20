---
layout: post
title: "CocoaPods module source"
date: 2015-07-13 13:12:54 +0800
comments: true
categories: 
---

###使用CocoaPods来单独管理封装的可复用的控件

公司需要将套在App工程中的一些公共代码单独出来，并使用CocoaPods去管理。

***

###Cocoapods是非常好用的一个iOS依赖管理工具，使用它可以方便的管理和更新项目中所使用到的第三方库，以及将自己的项目中的公共组件交由它去管理。

比如，可以将一个项目中的各个业务模块解耦，然后分别做成一个pods源，在主工程通过pods导入，能够极大的解耦业务模块。

通常大家使用pods去在线下载导入依赖的第三方开源类库，但其实也可以导入配置在自己git服务器上的库代码，最终编译器编译生成一个.a或.framework导入到工程依赖中。

***

###安装完CocoaPods后，执行的一条命令 pos setup 的作用

- (1) 将github上的一个git库中的所有第三方库的podspec描述文件克隆到本地
	- 所有的项目的Podspec文件都托管在 `https://github.com/CocoaPods/Specs`
	- 可能有一些库并未收录其中

- (2) podspec文件在本地存放的位置`~/.cocoapods`

- (3) `pod list` 和 `pod search` 命令只搜索存在于本地`~/.cocoapods`文件夹的所有第三方库，并不会连接到远程服务器

- (4) `pod search --full query` 命令作更仔细的搜索，该命令不但搜索类库的名称，同时还搜索类库的描述文本，所以搜索速度也相对慢一些

- (5) 如果要从github服务器更新本地第三方库的描述文件，执行`pod repo update master`

***

###podspec文件

- (1) podspec文件的结构

```objc
Pod::Spec.new do |s|
  s.name             = "MyLib"
  s.version          = "0.1.0"
  s.summary          = "A short description of MyLib."
  
  s.description       = "具体的描述信息..."
  
  s.homepage         = "https://github.com/<GITHUB_USERNAME>/MyLib"
  # s.screenshots     = "www.example.com/screenshots_1", "www.example.com/screenshots_2"
  s.license          = 'MIT'
  s.author           = { "xiongzenghui" => "xiongzenghui@myhome.163.com" }
  s.source           = { :git => "https://github.com/<GITHUB_USERNAME>/MyLib.git", :tag => s.version.to_s }
  # s.social_media_url = 'https://twitter.com/<TWITTER_USERNAME>'

  s.platform     = :ios, '7.0'
  s.requires_arc = true
  
  # 存放 所有源文件 的目录  Pod/Classes/
  s.source_files = 'Pod/Classes/**/*'
  
  # 存放所有资源文件 Pod/Assets/
  s.resource_bundles = {
    'MyLib' => ['Pod/Assets/*.png']
  }

  # 指定 头文件 的 搜索位置
  # s.public_header_files = 'Pod/Classes/**/*.h'
  
  # 依赖的系统framework
  # s.frameworks = 'UIKit', 'MapKit'
  
  # 依赖的系统的framework
  # s.libraries  = 'xxx.framework', 'xxx.framework', 'xxx.framework'
  
  # 依赖的其他第三方开源库，如果有多个需要填写多个 s.dependency
  s.dependency 'AFNetworking', '~> 2.6.3'
  s.dependency 'JSONModel', '~> 2.6.3'
  
end 
```

从上面可以看出，这个文件的作用实际上就是指定pod的名字的、版本号、代码下载的git地址、以及指定依赖的系统Framework、和第三方其他的开源库。

所说这个文件的作用就是指定，要下载的pods源对于的代码在哪里，以及怎么下载进行依赖配置到工程中。

了解完基本概念之后，下面开始具体如何操作。

***

###去创建一个存放库代码的git库（简称git库1）

- (1) 将我们的库代码上传到这个git库1

```
https://git.coding.net/xiongzenghui/TestLib.git
```

- (2) 一定要对提交的commit打一个tag

```
dev_2_0
```

这样我们的代码库就OK了，接下来配置podspec的git库.

***

###配置podspec的git库（简称git库2）

- (1) 创建一个存在所有的podspec文件的git库

```
https://git.coding.net/xiongzenghui/podspecgit.git
```

- (2) 使用`pod repo add`将我们创建的git库2注册到Cocoapods的库中

```
pod repo add TestLibSpecRepo https://git.coding.net/xiongzenghui/podspecgit.git
```

- (3) 查看添加的自己的spec库

```
cd ~/.cocoapods/repos
```

![](http://i4.piimg.com/8311/6c1074ba4444f653.png)

- (4) 在其他需要pods这个spec库的机器上同样执行上面的注册命令，才能pods下载指定的git库1的代码

```
pod repo add TestLibSpecRepo https://git.coding.net/xiongzenghui/podspecgit.git
```

***

###进入到git库1根目录下，编写podspec文件

- (1) 执行`pod spec create TestLibSpec` 创建spec模板文件

![](http://i4.piimg.com/8311/cdac816d83f5b6e0.png)

![](http://i4.piimg.com/8311/9625776a670d44f8.png)

- (2) 编辑生成的podspec文件

```objc
#
#  Be sure to run `pod spec lint TestLibSpec.podspec' to ensure this is a
#  valid spec and to remove all comments including this before submitting the spec.
#
#  To learn more about Podspec attributes see http://docs.cocoapods.org/specification.html
#  To see working Podspecs in the CocoaPods repo see https://github.com/CocoaPods/Specs/
#

Pod::Spec.new do |s|

  # ―――  Spec Metadata  ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  These will help people to find your library, and whilst it
  #  can feel like a chore to fill in it's definitely to your advantage. The
  #  summary should be tweet-length, and the description more in depth.
  #

  s.name         = "TestLibSpec"
  s.version      = "1.0.0"
  s.summary      = "这是一个测试的pods源."

  # This description is used to generate tags and improve search results.
  #   * Think: What does it do? Why did you write it? What is the focus?
  #   * Try to keep it short, snappy and to the point.
  #   * Write the description between the DESC delimiters below.
  #   * Finally, don't worry about the indent, CocoaPods strips it!
  s.description  = "这是描述这是描述这是描述这是描述这是描述这是描述这是描述"

  s.homepage     = "http://www.baidu.com"
  # s.screenshots  = "www.example.com/screenshots_1.gif", "www.example.com/screenshots_2.gif"


  # ―――  Spec License  ――――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  Licensing your code is important. See http://choosealicense.com for more info.
  #  CocoaPods will detect a license file if there is a named LICENSE*
  #  Popular ones are 'MIT', 'BSD' and 'Apache License, Version 2.0'.
  #

  s.license      = "MIT"
  # s.license      = { :type => "MIT", :file => "FILE_LICENSE" }


  # ――― Author Metadata  ――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  Specify the authors of the library, with email addresses. Email addresses
  #  of the authors are extracted from the SCM log. E.g. $ git log. CocoaPods also
  #  accepts just a name if you'd rather not provide an email address.
  #
  #  Specify a social_media_url where others can refer to, for example a twitter
  #  profile URL.
  #

  s.author             = { "zain" => "xiongzenghui@gmail.com" }
  # Or just: s.author    = "zain"
  # s.authors            = { "zain" => "xiongzenghui@gmail.com" }
  # s.social_media_url   = "http://twitter.com/zain"

  # ――― Platform Specifics ――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  If this Pod runs only on iOS or OS X, then specify the platform and
  #  the deployment target. You can optionally include the target after the platform.
  #

  # s.platform     = :ios
  s.platform     = :ios, "6.0"

  #  When using multiple platforms
  # s.ios.deployment_target = "5.0"
  # s.osx.deployment_target = "10.7"
  # s.watchos.deployment_target = "2.0"
  # s.tvos.deployment_target = "9.0"


  # ――― Source Location ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  Specify the location from where the source should be retrieved.
  #  Supports git, hg, bzr, svn and HTTP.
  #

  s.source       = { :git => "https://git.coding.net/xiongzenghui/TestLib.git", :tag => "1.0.0" }


  # ――― Source Code ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  CocoaPods is smart about how it includes source code. For source files
  #  giving a folder will include any swift, h, m, mm, c & cpp files.
  #  For header files it will include any header in the folder.
  #  Not including the public_header_files will make all headers public.
  #

  s.source_files  = "HahaView/Lib/**/*.{h,m}"
  #s.exclude_files = "Classes/Exclude"

  # s.public_header_files = "Classes/**/*.h"


  # ――― Resources ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  A list of resources included with the Pod. These are copied into the
  #  target bundle with a build phase script. Anything else will be cleaned.
  #  You can preserve files from being cleaned, please don't preserve
  #  non-essential files like tests, examples and documentation.
  #

  # s.resource  = "icon.png"
  # s.resources = "Resources/*.png"

  # s.preserve_paths = "FilesToSave", "MoreFilesToSave"


  # ――― Project Linking ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  Link your library with frameworks, or libraries. Libraries do not include
  #  the lib prefix of their name.
  #

  # s.framework  = "SomeFramework"
  # s.frameworks = 'CFNetwork', 'SystemConfiguration', 'Security'
  # s.xcconfig = {'CLANG_CXX_LIBRARY' => 'libstdc++'}
  # s.xcconfig = { 'OTHER_LDFLAGS' => '-ObjC' }
  # s.library   = "iconv"
  # s.libraries = "iconv", "xml2"
  # s.ios.library = 'c++', 'stdc++', 'z'


  # ――― Project Settings ――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  If your library depends on compiler flags you can set them in the xcconfig hash
  #  where they will only apply to your library. If you depend on other Podspecs
  #  you can include multiple dependencies to ensure it works.

  s.requires_arc = true

  # s.xcconfig = { "HEADER_SEARCH_PATHS" => "$(SDKROOT)/usr/include/libxml2" }
  # s.dependency "JSONKit", "~> 1.4"

end
```

- (3) `pod spec lint`检查podspec文件是否正确

![](http://i1.piimg.com/8311/83f7e4c2c3abf848.png)

我遇到的报错:

```
fenqiledeiMac:MyLib fenqile$ pod spec lint

 -> TestLibSpec (1.0.2)
    - ERROR | [iOS] unknown: Encountered an unknown error ([!] /Applications/Xcode.app/Contents/Developer/usr/bin/git clone https://git.coding.net/xiongzenghui/TestLib.git /var/folders/jr/8nt_pqp16z5g83z3v4f78j280000gn/T/d20160713-19790-1caabet --template= --single-branch --depth 1 --branch v1.0.2

Cloning into '/var/folders/jr/8nt_pqp16z5g83z3v4f78j280000gn/T/d20160713-19790-1caabet'...
warning: Could not find remote branch v1.0.2 to clone.
fatal: Remote branch v1.0.2 not found in upstream origin
) during validation.

Analyzed 1 podspec.

[!] The spec did not pass validation, due to 1 error.
```

就是要在git库上存在这个`1.0.2`这个`Tag`，这个spec从这个分支拉去代码。其他的错误都很简单，按照提示解决即可。

还有一些构建错误要注意:

```
1. 对于一些引用的其他第三方库头文件，最好不要在 .h 中使用，放到 .m 中去
2. 如果使用到了，一定要使用 <AFNetworking/AFNetworking.h>，如果使用"AFNetworking.h"那么在pod lib lint会构建错误
```

- (4) 将上面编写好的podspec文件push到spec文件git库

```
pod repo push TestLibSpecRepo TestLibSpec.podspec
```

成功后提示信息如下

```
fenqiledeiMac:MyLib fenqile$ pod repo push TestLibSpecRepo TestLibSpec.podspec

Validating spec
 -> TestLibSpec (1.0.2)

Updating the `TestLibSpecRepo' repo

Your configuration specifies to merge with the ref 'refs/heads/master'
from the remote, but no such ref was fetched.

Adding the spec to the `TestLibSpecRepo' repo

 - [Add] TestLibSpec (1.0.2)

Pushing the `TestLibSpecRepo' repo

To https://git.coding.net/xiongzenghui/podspecgit.git
 * [new branch]      master -> master
```

- (5) `pod search TestLibSpec` 在本地spec的git库查询刚才pods库名字

```
-> TestLibSpec (1.0.2)
   这是一个测试的pods源.
   pod 'TestLibSpec', '~> 1.0.2'
   - Homepage: http://www.baidu.com
   - Source:   https://git.coding.net/xiongzenghui/TestLib.git
   - Versions: 1.0.2 [TestLibSpecRepo repo]
```

- (6) 去CocoaPods的spec目录下查看我们添加的spec文件、以及spec git库发生的变化

```
fenqiledeiMac:TestLibSpecRepo fenqile$ cd ~/.cocoapods/
fenqiledeiMac:.cocoapods fenqile$ ls
repos
fenqiledeiMac:.cocoapods fenqile$ cd repos/
fenqiledeiMac:repos fenqile$ ls
TestLibSpecRepo	master
fenqiledeiMac:repos fenqile$ cd TestLibSpecRepo/
fenqiledeiMac:TestLibSpecRepo fenqile$ ls
TestLibSpec
fenqiledeiMac:TestLibSpecRepo fenqile$ cd TestLibSpec/
fenqiledeiMac:TestLibSpec fenqile$ ls
1.0.2
fenqiledeiMac:TestLibSpec fenqile$ cd 1.0.2/
fenqiledeiMac:1.0.2 fenqile$ ls
TestLibSpec.podspec
fenqiledeiMac:1.0.2 fenqile$
```

![](http://i4.piimg.com/8311/bb9f11bab3801011.png)

![](http://i4.piimg.com/8311/a7493f3fc6d6b18b.png)

***

###新建立demo项目pods上马的库

```objc
# Uncomment this line to define a global platform for your project
# platform :ios, '9.0'

target 'PodsDemo' do
  # Uncomment this line if you're using Swift or would like to use dynamic frameworks
  # use_frameworks!

  pod 'TestLibSpec', '~> 1.0.2'

  target 'PodsDemoTests' do
    inherit! :search_paths
    # Pods for testing
  end

  target 'PodsDemoUITests' do
    inherit! :search_paths
    # Pods for testing
  end

end
```

但是报错了:


```
fenqiledeiMac:PodsDemo fenqile$ pod install
Analyzing dependencies
[!] Unable to find a specification for `TestLibSpec (~> 1.0.2)`
```

原因是，pod install只会在master下`/Users/fenqile/.cocoapods/repos/master`搜索，但是我们的spec文件在`/Users/fenqile/.cocoapods/repos/TestLibSpecRepo/`目录下

解决方法，在podfile文件最上添加指定加载spec的git库

```
#第一个地址: 指定搜索spec文件的git库地址
source "https://git.coding.net/xiongzenghui/podspecgit.git"

#第二个地址: 其他仍然从github下载的代码
source "https://github.com/CocoaPods/Specs.git"

# platform :ios, '7.0'

target 'PodsDemo' do
  # Uncomment this line if you're using Swift or would like to use dynamic frameworks
  # use_frameworks!

  pod 'TestLibSpec', '~> 1.0.2'

end
```

运行后成功如下

```
fenqiledeiMac:PodsDemo fenqile$ pod install
Analyzing dependencies
Downloading dependencies
Installing TestLibSpec (1.0.2)
Generating Pods project
Integrating client project

[!] Please close any current Xcode sessions and use `PodsDemo.xcworkspace` for this project from now on.
Sending stats
Pod installation complete! There is 1 dependency from the Podfile and 1 total pod installed.
```

在别的机器上进行pod时，使用`pod update`

如果CocoaPods软件版本过低，执行命令后会报错如下:

```
Downloading dependencies
Installing TestLibSpec (1.0.2)
Generating Pods project
2016-07-13 15:41:48.280 ruby[16797:10297231] [MT] DVTAssertions: ASSERTION FAILURE in /Library/Caches/com.apple.xbs/Sources/IDEFrameworks/IDEFrameworks-9548.1/IDEFoundation/Initialization/IDEInitialization.m:590
Details:  Assertion failed: _initializationCompletedSuccessfully
Function: BOOL IDEIsInitializedForUserInteraction()
Thread:   <NSThread: 0x7f985d0d1480>{number = 1, name = main}
Hints: None
Backtrace:
  0  0x0000000105f70e1f -[DVTAssertionHandler handleFailureInFunction:fileName:lineNumber:assertionSignature:messageFormat:arguments:] (in DVTFoundation)
  1  0x0000000105f705ac _DVTAssertionHandler (in DVTFoundation)
  2  0x0000000105f70818 _DVTAssertionFailureHandler (in DVTFoundation)
  3  0x0000000105f7077a _DVTAssertionFailureHandler (in DVTFoundation)
  4  0x00000001074462cd IDEIsInitializedForUserInteraction (in IDEFoundation)
  5  0x000000010a064631 +[PBXProject projectWithFile:errorHandler:readOnly:] (in DevToolsCore)
  6  0x000000010a0661b6 +[PBXProject projectWithFile:errorHandler:] (in DevToolsCore)
  7  0x00007fff8810af44 ffi_call_unix64 (in libffi.dylib)
Abort trap: 6
```

更新CocoaPods能够解决:

```
$ sudo gem update --system // 先更新gem，国内需要切换源
$ gem sources --remove https://rubygems.org/
$ gem sources -a https://ruby.taobao.org/
$ gem sources -l
\*\*\* CURRENT SOURCES \*\*\*
https://ruby.taobao.org/
$ sudo gem install cocoapods // 安装cocoapods
$ pod setup
```

###subspec使用

```
s.subspec 'NetWorkEngine' do |networkEngine|
  networkEngine.source_files = 'Pod/Classes/NetworkEngine/**/*'
  networkEngine.public_header_files = 'Pod/Classes/NetworkEngine/**/*.h'
  networkEngine.dependency 'AFNetworking', '~> 2.3'
end
 
s.subspec 'DataModel' do |dataModel|
  dataModel.source_files = 'Pod/Classes/DataModel/**/*'
  dataModel.public_header_files = 'Pod/Classes/DataModel/**/*.h'
end
 
s.subspec 'CommonTools' do |commonTools|
  commonTools.source_files = 'Pod/Classes/CommonTools/**/*'
  commonTools.public_header_files = 'Pod/Classes/CommonTools/**/*.h'
  commonTools.dependency 'OpenUDID', '~> 1.0.0'
end
 
s.subspec 'UIKitAddition' do |ui|
  ui.source_files = 'Pod/Classes/UIKitAddition/**/*'
  ui.public_header_files = 'Pod/Classes/UIKitAddition/**/*.h'
  ui.resource = "Pod/Assets/MLSUIKitResource.bundle"
  ui.dependency 'PodTestLibrary/CommonTools'
end
```

###维护更新podspec以及pod库代码步骤

- (1) 修改pods库源代码，修改完之后打上一个`Tag`和代码一起push到git库

- (2) 修改podspec文件中的`s.source:tag`对应commit的pods库源码（s.version.to_s表示最新commit）

- (3) 使用pod lib lint 或 pod spec lint 验证Tag对于的pods库代码是否正确

- (4) 如果验证错误，可以删除之前提交的Tag，然后按照提示继续修改pods库代码然后再次提交打上Tag

- (5)删除Tag的步骤:
	- 删除本地Tag、git tag -d Tag名字
	- 删除远程服务器上的Tag、git push origin :refs/tags/Tag名字

- (6) 如果验证成功，先修改spec文件的版本（`s.version`），然后将spec文件push到spec文件的git服务器上

```
pod repo push fenqile_pod_spec_repo xxx.podspec 
```

###推送到CocoaPods服务器的pods库

找了好久没有找到`私有库`之间进行依赖的，所以索性就做做`共有库`。

-  首次使用trunk的时候，需要注册自己的电脑

```
pod trunk register [E-mail] [User Name]
```

- 执行完成之后，会受到一封验证邮件，按邮件提示完成验证即可。注册流程完成之后，可以使用验证一下自己是否注册成功

```
pod trunk me
```

- 将制作好的pods spec文件 推送到CocoaPods服务器

```
pod trunk push xxxxx.podspec
```

如果是一些不是特别严重的warn警告

```
pod trunk push xxxxx.podspec --allow-warnings
```

###当推送spec文件到CocoaPods服务器之后，提示`Unable to find a pod with name, author, summary, or descriptionmatching`解决

- 首先尝试删除缓存文件

```
rm ~/Library/Caches/CocoaPods/search_index.json 
```

```
pod search xxx
```

- 如果还是报错，尝试更新pods

```
pod setup
```

###在将项目中大部分互相关联的代码移除时，发现如上只适合类似AFNetworking这样很独立的库。因为互相关联的代码可能有一些代码拖不出来，就导致库工程报错。而一旦报错，CocoaPods不允许上传到服务器。

所以，只能使用重新建立一个`静态库`工程。因为静态库工程`只做代码编译`，不做代码链接，就是说不回去查找这些声明的方法到底有没有真的实现函数。

- (1) 问题1、App工程已经pods了AFNetworking，但是创建的静态库工程中代码也使用到了AFNetworking，那静态库工程也需要pods一次AFNetworking吗？（文件）

很明显，在静态库工程中，不应该继续pods下载AFNetworking代码了，要不然在App工程中就会有两份AFNetworking的代码了，虽然可能会由CocoaPods自动去改掉前缀。

静态库工程既然只做编译，那么就可以将静态库工程中依赖的类文件的`头文件.h`拖到静态库工程中，就可以通过编译了。

- (2) 问题2、静态库中使用到了App工程中`定义 的变量`（变量）

在静态库工程中，再附加上在App工程中所定义的变量的`声明`。注意，是声明，而不是定义。

声明一个变量:

```
extern const DDLogLevel ddLogLevel;
```

```
FOUNDATION_IMPORT const DDLogLevel ddLogLevel;
```

***

###Mac OS10.11 执行 “sudo gem install cocoapods” 的时候出现问题：ERROR: While executing gem ... (Errno::EPERM) 解决

rvm是什么？为什么要安装rvm呢，因为rvm可以让你拥有多个版本的Ruby，并且可以在多个版本之间自由切换。
第一步：安装rvm

```
curl -L get.rvm.io | bash -s stable
```

```
source ~/.rvm/scripts/rvm
```

等待终端加载完毕,后输入：


```
rvm -v
```

如果能显示版本好则安装成功了。

第二步：安装ruby（cocoapods需要2.2.0以上的噢）

列出ruby可安装的版本信息

```
rvm list known
```

安装一个ruby版本


```
rvm install 2.1.4
```

如果想设置为默认版本，可以用这条命令来完成


```
rvm use 2.1.4 --default
```

查看已安装的ruby

```
rvm list
```

卸载一个已安装ruby版本

```
rvm remove 2.1.4
```

第三步：更换源

查看已有的源

```
gem source
```

显示会如下：

CURRENT SOURCES
http://rubygems.org/
然后我们需要来修改更换源（由于国内被墙）所以要把源切换至淘宝镜像服务器 在终端执行以下命令

 
```
gem update --system
gem uninstall rubygems-update
gem sources -r http://rubygems.org/
gem sources -a http://ruby.taobao.org
```

第四步：
安装好ruby 更新完gem后，就可以执行 

```
sudo gem install -n /usr/local/bin cocoapods --pre
```

***

###但是发现CocoaPods不适合当前我的情况，因为我是把一些互相关联的代码只是移到静态库工程，但是有一些代码还是需要依赖App工程中却无法移到静态库工程。静态库工程就会缺失一些方法实现的.m文件，虽然是可以通过编译，但是最后使用CocoaPods验证的时候就会报错，无法走面的流程。

所以，适用于CocoaPods管理的库代码，只能是能够被编译、链接、运行通过的代码。即使有依赖，也只能通过CocoaPods来下载引入CocoaPods自己识别的源对于的代码。

无法通过CocoaPods来处理，通过git的submodule来解决，自动让主App工程的git库拉去我们静态库工程的git库代码。

***

###Git Submodule

首先是当前App项目中一些公共代码进行结构划分:

- (1) App主工程，有一个git库
- (2) 封装所有Commom层代码的静态库工程，也有一个git库
- (3) 以后还可能陆续添加的各种子模块的git库
	- Network Request
	- Font Manager
	- 等等这些就可以做成一个单独独立的CocoaPods源

被抽出来的一套公用代码库，可以被多个App工程中引入调用。如果是手动进行拷贝到多个App工程中，还是比较麻烦的，每次公用代码工程修改后，又必须手动导入一次。对于公用代码的版本也不好管理。


Git提供了这种情况的解决方法 >>> git submodule（git子模块）。

- (1) 对应的每一个公用代码工程由一个远程git服务器仓库管理
- (2) git submodule则允许一个git仓库下，包含其他n多个`子git仓库`
- (3) 可以让多个公用库的代码保持独立，按需拉区其各自git上的更新

下面是对git submodule的使用步骤记录。

***

###在主工程git库上注册一个子模块git库 >>> git submodule add

- (1) 在主App工程的git库中，添加 submudule git库，会将子git库clone到主App工程的指定的目录下


```
git submodule add 仓库地址 路径
```

```
fenqiledeiMac:MyLib fenqile$ git submodule add http://xxxxx.git Commom
Cloning into 'Commom'...
remote: 对象计数中: 550, 完成.
remote: 压缩对象中: 100% (414/414), 完成.
remote: Total 550 (delta 132), reused 472 (delta 107)
Receiving objects: 100% (550/550), 430.91 KiB | 0 bytes/s, done.
Resolving deltas: 100% (132/132), done.
Checking connectivity... done.
```

会在当前工程根路径下生成一个名为“.gitmodules”的文件，其中记录了子模块git库的信息

```
[submodule "Commom"]
	path = Commom
	url = http://xxxx.git
```

- (2) 对主工程git库提交前面发生的变化

```
cd 主工程根下的.git位置
git add .
git commit -m "...."
git push origin 分支
```

- (3) 登陆主工程的git管理后台查看是否push成功

![](http://i2.piimg.com/8311/db2d4e6d567809cc.png)

可以查看此次push的一些什么东西？

```
[submodule "Commom"]
	path = Commom
	url = http://..........................git
```

```
Subproject commit ........................7c58
```

截图如下所示:

![](http://i2.piimg.com/8311/c0c035a6d7d30520.png)


也就是说在主工程中，使用了一个名字为`Commom`文件，记录了`子工程git库`中的某一个commit提交id，和`子工程git库`的url地址。

至此，submodule git库已经成功添加到主工程的git库中了。

***

###在主工程的git库，拉取子模块git库拉的更新 

当添加完子模块后，过段时间后子模块有更新，这时候就需要更新了。

- (1) 第一次clone整个主工程项目

```
git clone App主工程git库 存放目录

// 自动拉去submodule git仓库master分支最新代码
git submodule add http://xxxxx.git Commom
```

- (2) 非首次clone整个主工程项目

```
//获取静态库工程git库中最新commit id
git submodule init

//将最新comimit id 写入到主工程目录下的记录文件中
git submodule update

//拉取静态库工程git库最新代码
git submodule foreach git pull / git submodule foreach git pull origin master 
```

对于submodule的代码，最好都是放在`master分支`上。

- (3) 主工程拉取了子模块git库拉的更新后，会修改之前的`Commom`文件保存最新的submodule的commit

![](http://i4.piimg.com/8311/9fc2ad4f6f2e7c12.png)

***

###对submodule git库代码的修改

- (1) 进入到子工程的目录下进行修改代码
- (2) 修改完后，提交到submodule自己的git上得master分支

***

###移除submodule

```
$ cd your_project
$ git rm --cached another_project
$ rm -rf another_project
$ vim .gitmodules
...remove another_project...
$ vim .git/config
...remove another_project...
$ git commit -a -m 'Remove another_project submodule'
```

有几步的删除操作要手动完成。


###导入嵌套的静态库工程，导致主工程无法Archive

解决，将静态库中的所有源文件拖到Project区域

***

###学习来源

```
http://www.cocoachina.com/ios/20150228/11206.html
http://re-reference.iteye.com/blog/1755097
http://www.tuicool.com/articles/jqiEJzU
http://www.cocoachina.com/industry/20130509/6161.html
http://blog.csdn.net/bailyzheng/article/details/33405103
组内的微信大大高手亲手指点
```