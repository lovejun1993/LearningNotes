## CocoaPods更新

 先切换gem源

```
gem sources --remove https://rubygems.org/
```

```
gem source -a https://gems.ruby-china.org
```

 升级了cocoapods

```
sudo gem install -n /usr/local/bin cocoapods --pre
```

 查看cocoapods版本

```
pod --version
```

## 存在私有pods源时，pod install报错解决

```
[!] Unable to add a source with url `git@git.elenet.me:eleme.mobile.ios/ios-specs.git` named `mobile-ios-specs`.
You can try adding it manually in `~/.cocoapods/repos` or via `pod repo add`.
```

需要添加私有pods源的spec git库

```
pod repo add mobile-ios-specs git@git.elenet.me:eleme.mobile.ios/ios-specs.git
```

格式

```
pod repo add spec库名 spec库的git地址
```

## git clone SSH地址时报错解决

```
Cloning into 'mobile-ios-specs'...
Permission denied (publickey).
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

解决: Git SSH Key 生成步骤

```
git config --global user.name "xuhaiyan"
git config --global user.email "haiyan.xu.vip@gmail.com"
```

```
cd ~/.ssh
```

```
ssh-keygen -t rsa -C "git库注册的邮箱"
```

```
pbcopy < ~/.ssh/id_rsa.pub  
```

```
最后添加到git库的后台界面的SSHKeys中.
```

如果还是不行，就是没有权限访问这个spc git库。

## CocoaPods库指令

### podfile、target xxx do .... end

在使用CocoaPods时，pod install默认只能为xcode工程的第一个target添加依赖库支持。

假设有两个不同的target:TargetName1和TargetName2:

第一种、所有的target使用相同的第三方依赖配置

```c
link_with ‘TargetName1’, ‘TargetName2’

platform :iOS, ‘6.1’

pod ‘MKNetworkKit’ 
pod ‘MBProgressHUD’ 
pod ‘IQKeyboardManager’ 
pod ‘Toast’
```

第二种、不同的target使用不同的第三方依赖配置

```c
platform :ios, ‘6.1’

target :TargetName1 do 
pod ‘MKNetworkKit’ 
pod ‘MBProgressHUD’ 
pod ‘IQKeyboardManager’ 
pod ‘Toast’ 
end

target :TargetName2 do 
pod ‘MKNetworkKit’ 
pod ‘MBProgressHUD’ 
pod ‘IQKeyboardManager’ 
pod ‘Toast’ 
end 
```

最后还要打开使用pods的Xcode主工程，为每个target的，build setting里的四个地方，增加$(inherited)

```c
Other Link Flags 
Library search Paths
Header search Paths
Framework search Paths
```

### podfile、注释写法

```
=begin
这里写注释
=end
```

### podfile、`inhibit_all_warnings!`、`use_frameworks!`

```c
platform :ios, '7.0'

inhibit_all_warnings!// 在全局指定不显示所有所引用的库中的警告信息

use_frameworks!//通过指定use_frameworks!要求生成的是framework而不是静态库

.....
```

### podfile、workspace

指定下载pods源代码之后创建的心的Xcode工程配置文件的名字。

默认情况下，我们不需要指定，直接使用与Podfile所在目录的工程名一样就可以了。如果要指定另外的名称，而不是使用工程的名称，可以这样指定:

```c
workspace 'MyWorkspace'
```

一个项目可以有多个Target ，如果我们的项目名称和Target 的名称一致就只会产生一个 Pod.debug 的配置文件，也就不会产生一个和Target 一致的Pod-target.debug 的配置文件。

（pod.debug 也是一个默认的配置文件）。

### podfile、定义依赖宏

```c
def declare_pods
  pod 'NVMFakeModulesManager'
  pod 'NVMCommonService'
  pod 'NVMEnvironment'
  pod 'NVMUIKitCore', '~> 0.1'
  pod 'NVMUIKit', '~> 5.10'
  pod 'NVMFoundation'
  pod 'NVMNetwork'
end

target 'ds_ios' do
  shared_pods//使用宏
end
```

### podfile、project

指定某一个target，使用哪一个Xcode作为主工程。默认情况下是没有指定的，当没有指定时，会使用Podfile目录下与target同名的工程。

```c
target 'NVMBreakfastModule' do
  project 'NVMBreakfastModule'
  declare_pods
  pod 'NVMShellProject'
end

target 'NVMBreakfastModuleDemo' do
  project 'Demo/NVMBreakfastModuleDemo/NVMBreakfastModuleDemo'
end
```

对上述配置的Xcode工程目录结构:

```
- NVMBreakfastModule
	- Demo
		- NVMBreakfastModuleDemo
			- NVMBreakfastModuleDemo
				- 项目代码
			- NVMBreakfastModuleDemo.xcodeproj
	- NVMBreakfastModule
	- NVMBreakfastModule.xcodeproj
	- NVMBreakfastModule.xcworkspace
	- Podfile
	- Pods
```

### podfile、source

source是指定pod的来源。如果不指定source，默认是使用CocoaPods官方的source。通常我们没有必要添加。


如果不想使用官方的，而是在别的地方也有，可以这样指定:

```c
source 'https://github.com/artsy/Specs.git' 
```

默认是官方的source source 

```c
https://github.com/CocoaPods/Specs.git'
```

### podfile、Hooks

- plugin
- pre_install
- post_install

### podfile、pod中用于指定范围的符号如下

```c
>= version 要求版本大于或者等于version，当有新版本时，都会更新至最新版本
< version 要求版本小于version，当超过version版本后，都不会再更新
<= version 要求版本小于或者等于version，当超过version版本后，都不会再更新
~> version 比如上面说明的version=1.1.0时，范围在[1.1.0, 2.0.0)。注意2.0.0是开区间，也就是不包括2.0.0。
```

### podfile、使用本地库

```c
pod 'AFNetworking', :path=>'~/Documents/AFNetworking'
```

### podfile、制定本地的podspec

```c
podspec :path=>'/Documents/PrettyKit/PrettyKit.podspec'
```

### podfile、从外部podspec引入

```
pod 'JSONKit', :podspec => 'https://example.com/JSONKit.podspec'
```

### podfile、针对每个target设置不同的宏, 比如debug版本, 那么X=0, release版本X=1

```c
GCC_PREPROCESSOR_DEFINITIONS
```

第一种，针对整个工程:

```c
Pod::Spec.new do |s|

.....

s.xcconfig = { 
 "GCC_PREPROCESSOR_DEFINITIONS" => '$(inherited) X=1' 
}

....

end
```

第二种，针对某一个target

```c
Pod::Spec.new do |s|

.....

s.pod_target_xcconfig = { 'GCC_PREPROCESSOR_DEFINITIONS' => '$(inherited) X=1' }

....

end
```