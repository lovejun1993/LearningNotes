---
layout: post
title: "framework一、创建"
date: 2015-12-24 17:30:17 +0800
comments: true
categories: 
---

###首先一般的解释行语言的程序的运行过程

- 比如c语言的程序运行过程

```
预编译 >> 编译(Compile) >> 链接(Link) >> 运行(Run)
```

- 其中的连接Link，就是将一些`系统Lib库`以及`用户自己开发的Lib库`链接到c程序中

- 虽然Objective-C为`消息型`语言，但是也少不了如上四个步骤

- 只是Objective-C将 `编译 >> 连接`这个过程，推迟到`运行时`

###为什么需要 静态库 或者 动态库？

- 库不提供`源码`给调用者。

- 库只提供一些没有main函数的程序代码集合，经过编译后产生的`二进制 可执行文件`。

- 为了不想让别人看到内部代码的实现（Host、参数、加密、Api..），所以`库`实际上就是一种`闭源`的方法。

- 那么对于`开源 实现代码`形式的组件，可以使用`CocoaPods源`。

- 也可以是一种 `模块化` 开发App的方法，将一些重用性较高的代码单独做成一个framework。

****

###`静`态库

- 主要实现方式
	- 实现方式一、`.a`包 + `暴露头文件`
	- 实现方式二、`staic framework`

- 需要`编译`到目标工程中，就造成`库` 与 `主App工程` 无法解耦

- 每次库代码更新之后，就必须让主App重新引入库，然后进行编译，再发布最终编译生成的App

- 静态库，`链接时`，系统会完整地`复制到`可执行文件中， 被多次使用就有多份冗余拷贝

![](http://i4.tietuku.com/cf619460a30842b2.png)

可以看到 程序1 与 程序2 ，都含有一份 静态库内可执行文件的拷贝

***

###`动`态库 

- 是在`App程序运行`时，可以选择性加载

- 运行时，将动态库载入到`内存`，并使用同一个内存块，使得工程中到处都可以使用，而无需重复开辟内存，提高系统性能

- 动态库 可以动态的`更新`，然后动态的加载，不影响App版本发布

- `iOS 8`中已允许开发者可以创建自己动态

- 主要实现方式
	- 实现方式一、`dynamic framework`
		- iOS9以后，苹果支持此种动态库
	- 实现方式二、`.dylib`
		- 苹果不允许创建这种动态库

- 动态库，`链接时`，`不会被复制`，程序运行时由系统动态`加载到内存`，供程序调用，系统`只加载一次`，`多个程序共用`，节省内存

![](http://i4.tietuku.com/82153ca7107fd602.png)


****

###Frmaework 与 .a 的区别:

- 之间的关系

```
Xxx.a + Xxx.h + 各种资源文件 = Xxx.framework
```

- 不管是 .a还是.framework，都是包含`实现代码编译之后的二进制文件`
	- .a 还需要单独提供 实现代码的 .h接口声明
	- .framework就不需要再提供，因为直接打包进去了

- 建议用framework来实现`模块化`开发App

- 可能遇到的问题: `静态库中的Category分类中的方法可能无法找到`
	- 解决: 在静态库Lib工程中配置`other linker flags`的值为-`ObjC`

****

### `模拟器` 和 `真机` 的cpu架构区别

- 常见在一个SDK拖入Xcode工程之后，编译之后报如下的错误:

```
undefined symbols for architecture x86_64
``` 

```
undefined symbols for architecture i386
```

```
undefined symbols for architecture armv7
```

```
undefined symbols for architecture armv7s
```

```
undefined symbols for architecture arm64
```

- 其中的英文单词 `architecture` 的意思是: `cpu 架构`

- 那么如上错误也就是说: `没有找到如上这5种 cpu 架构 支持的 可执行文件`
	- 此时，就需要提供不同cpu架构下的，可执行文件
	- 然后合并为一个`通用`的可执行文件

- `模拟器` 的 cpu 架构

	- i386  ---> 32位系统: iphonesimulator 3gs 到 iphonesimulator 5
	- x86_64 ---> 64位系统: iphonesimulator 5s 到 iphonesimulator 6 plus

- `真机设备` 的 cpu 架构

	- armv6 ---> iPhone 2G/3G，iPod 1G/2G
	- armv7 ---> iPhone 3GS/4/4s，iPod 3G/4G，iPad 1G/2G/3G
	- armv7s ---> iphone5，iphone5c
	- arm64 ---> iphone5s，iphone6plus，iphone6s

- 原因: `因为不同的cpu架构下，支持的cpu指令长度就不同`


```
armv7 , armv7s对应真机的32位处理器，
arm64对应真机的64位模拟器，
i386对应模拟器的32位模拟器；
x86_64对应模拟器的64位模拟器
```

***

###下面开始记录创建一个 静态库 的步骤，如下主要使用`.a 二进制包` + `公开暴露头文件`形式.

***

####创建一个 `Cocoa Touch Static Library 工程`

![](http://i4.tietuku.com/510bb90cc4774b90.png)

****

####一个静态库工程，由 `头文件` 和 `实现文件` 组成，这些文件最终经过编译之后，生成库

![](http://i4.tietuku.com/b9fa08963a38b02b.png)

****

####配置向外暴露、向外隐藏的源文件

- 首先选中 静态库工程的 target

![](http://i4.tietuku.com/e605d40ff748a56c.png)

- 然后添加 `copy headers`

![](http://i4.tietuku.com/dfb73b2e31f280c4.png)

- Headers 内的三种 访问权限

	- Public 是你希望暴露给外部调用者的
	- Private 下的头文件依然是`可以暴露`出来的
	- Project下的头文件对你的工程来说才是`私有`的
	- 那么一般公开就放在Public下，私有就放在Project下

![](http://i1.tietuku.com/8b4506bdf8f531a9.png)

- 将project中的想要暴露的头文件.h，添加到public.

![](http://i1.tietuku.com/5ba3ecce2ce1c12d.png)

![](http://i4.tietuku.com/0a54e1c368b594a4.png)

****

####在 静态库工程 中创建一个测试类，仅仅执行NSLog打印

![](http://i4.tietuku.com/98817e2714a51724.png)

![](http://i4.tietuku.com/c13d0bc71acb77a2.png)

****

####让外部调用者可以使用我们的`Logger`代码

- 首先，调整访问权限

![](http://i4.tietuku.com/33bc84c06fea9492.png)

- 然后，在 `MyStaticFramework.h` 统一暴露

![](http://i4.tietuku.com/c52a716d18355d29.png)

***

###接下来配置 静态库工程的 Target 中的 `Build Settings`

***

###指定一个`目录`名，表示你将把拷贝的 `公共头文件` 存放到哪里

- 选中 静态库工程的 Target
- 选择 Build Setting
- 搜索 public header
- 双击 Public Headers Folder Path
- 输入 `include/$(PROJECT_NAME)`

![](http://i4.tietuku.com/7fad83e500f3f2c8.png)

****

###接着改变一些其他的设置，尤其是那些在`二进制库`中遗留下的设置，编译器提供给你一个选项，来`消除无效代码`，也可以移除掉一些`debug用符号`

和之前一样，使用搜索框，改变下述设置
	
- Dead Code Stripping 设置为 NO

![](http://i4.tietuku.com/bd229e73df19a281.png)

- Strip Debug .. 全部设置为 NO

![](http://i4.tietuku.com/25d4f0fc346802ca.png)

- Strip Style 设置为 Non-Global Symbols

![](http://i4.tietuku.com/70d8399a6ca68bd2.png)

****

###然后选择目标为 `iOS Device`，按下command + B进行编译。编译成功之后，工程导航栏中`Product目录`下`libMyStaticFramework.a文件`将从 `红色` 变为 `黑色`，表明现在该文件`已经存在`了。

- 红色 表示 不存在
- 黑色 表示 存在

****

###右键单击`libMyStaticFramework.a`，选择 Show in Finder，可以找到 

- .a包，将源码编译之后二进制文件库
- 公开的头文件，所有暴露出去的.h文件

####真机编译目录 与 模拟器编译目录

- /Users/`用户名`/Library/Developer/Xcode/DerivedData/`项目名`-`随机值`/
	- Build/
		- Products/
			- Debug-iphoneos/  ---> 真机编译的目录
			- Debug-iphonesimulator/  ---> 模拟器编译的目录

- 注意:

```
1. 如果选择的是debug模式:
那么编译出来的就是Debug-iphoneos目录与Debug-iphonesimulator目录
```

```
2. 如果选择的是Release模式:
那么编译出来的就是Release-iphoneos目录与Release-iphonesimulator目录
```

####由于之前是使用`iOS Device`是基于`真机`cpu架构编译的，所以先看下`真机编译目录xxx-iphoneos`下生成的文件

- Debug-`iphoneos`/
	- include/
		- MyStaticFramework/
			- MyStaticFramework.h
			- Looger.h
	- libMyStaticFramework.a


OK，到这里就得到了一个适用于`真机`的 `.a`包

****

###然后创建一个依赖开发（Dependent Development）工程，来测试 静态库工程

- 1) Xcode选择 `File\Close Project` 关闭之前的 `静态库工程`

- 2) 使用File\New\Project创建一个新的`App工程`

- 3) 最后将新创建的`App项目工程`保存到和之前的`静态库工程`相同的目录下

![](http://i4.tietuku.com/b17b3b810f4b8e48.png)

- 4) 将`静态库工程`的xcodeproj文件，拖动到`App工程`中，方便在`App工程`中直接修改`静态库工程`的代码

![](http://i4.tietuku.com/b17b3b810f4b8e48.png)

![](http://i4.tietuku.com/6635120f21ac944c.png)

- 注意: 

```
你无法将同一工程在两个Xcode窗口中同时打开，
如果你发现你无法在你的工程中导航到库工程的话，
检查一下是否库工程在其他Xcode窗口中打开了。
```

****

###给App工程添加 静态库 作为依赖库

- 选中App工程
- 继续选中 App工程 的 Target
- Build Phases
- Target Dependencies，点击 `+` 按钮调出选择器

![](http://i4.tietuku.com/8faa17c75ebf4c9d.png)

![](http://i4.tietuku.com/360b4bfc0478eeef.png)

![](http://i4.tietuku.com/ccfb4757874c2edc.png)

****

###将生成的 .a包 目录下的 `公开的头文件` 拖入到App工程

![](http://i4.tietuku.com/b72625136bf1fa91.png)


****

###在App工程中测试静态库

```
#import "ViewController.h"

//导入头文件
#import "MyStaticFramework.h"

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    //测试静态库
    [Looger log];
}

@end
```

然后看到控制器输出

```
2015-12-24 23:16:47.953 FrameworkAppDemo[8489:395857] Framework Code execute...
```

####OK，到此为止是使用`.a` + `暴露头文件`的形式完成静态库

***

###接下来创建一个 `Framework` 来代替 `.a` + `暴露头文件`

####一个framework其实就是 `静态库 .a包` + `一组头文件`，大致上可以任务framework是对这二者的一个包装

####framework也有几点不同之处

- 目录结构:

	- Framework有一个能被Xcode识别的特殊的目录结构
	- 我们创建一个`build task`，由它来创建这种特殊结构

- 片段（Slice）:

	- 也就是frameowork能够支持的`cpu架构`（architecture）
	- architecture 例如: i386、x86_64、arm7、armv7、arm64
	- 需要针对所有的cpu架构，`都构建一次`
	- 然后将构建得到的所有的情况，都添加到framework中


####首先看下 Framework 的 特殊结构

- Bugly.framework 
	- Headers/  点击会进入到 Versions/Current/Headers/
	- Bugly/   点击会进入到 Versions/Current/Headers/Bugly
	- Versions/
		- A/
			- Headers/
				- 业务接口类1.h
				- 业务接口类2.h
				- 业务接口类3.h
				- ....所有会`暴露的`头文件
			- Bugly
		- Current/  点击会进入到 A/


- 这个是转载自其他技术博客的结构示意图

![](http://i4.tietuku.com/41dd9d29c857574d.png)


****

### 创建一个新的 `Cocoa Touch Framework 工程`，来完成framework

- 注意前面使用的是 `Cocoa Touch Static Library 工程`，生成的是 `.a包` 与 `公共头文件`

- 而现在是创建一个`Cocoa Touch Framework 工程`

![](http://i4.tietuku.com/0e13656d750aa7f4.png)

- 默认创建的是 `动态库` framework，如果需要修改成 `静态库`，因为Xcode5之前的项目只支持 `静态库`形式的framework

![](http://i4.tietuku.com/111b6cdc481b6504.png)

****

###在多种不同cpu架构下的编译生成静态库

- 目前需要支持的所有cpu架构

	- 真机
		- arm7: 在最老的支持iOS7的设备上使用
		- arm7s: 在iPhone5和5C上使用
		- arm64: 运行于iPhone5S的64位 ARM 处理器 上
	
	- 模拟器
		- i386: 32位模拟器上使用
		- x86_64: 64为模拟器上使用

- 每一种CPU架构，都需要 `不同` 的 `二进制数据`

***

###在静态库工程下，新创建一个Target -- `Aggregate`，使用Aggregate代替前面创建script生成framework.

- 首先，给 静态库工程 添加一个 `Aggregate Target`

![](http://i4.tietuku.com/2ff2e5d12cb55d0e.png)

![](http://i4.tietuku.com/44d4351e627f1aa6.png)

![](http://i4.tietuku.com/e0dbea1a4288fc0b.png)

- 然后，选中 静态库工程 的 `Aggregate Target`，添加 静态库工程的 `主Target` 依赖

![](http://i4.tietuku.com/5b5835ceed37019b.png)

![](http://i4.tietuku.com/d84f76249111779a.png)

- 在`Aggregate Target`下，选择Build Phases栏，点击Editor/Add Build Phase/Add Run Script Build Phase，创建一个新的`Run Script Build Phase`脚本，完成`多种cpu架构下编译静态库`

![](http://i4.tietuku.com/765e8c7719ded01e.png)

![](http://i4.tietuku.com/4ff322cf4c0a0f99.png)

- 编写脚本代码区域

```
# Sets the target folders and the final framework product.
# 如果工程名称和Framework的Target名称不一样的话，要自定义FMKNAME
# 例如: FMK_NAME = "MyFramework"
FMK_NAME=${PROJECT_NAME}
# Install dir will be the final output to the framework.
# The following line create it in the root folder of the current project.
INSTALL_DIR=${SRCROOT}/Products/${FMK_NAME}.framework
# Working dir will be deleted after the framework creation.
WRK_DIR=build
DEVICE_DIR=${WRK_DIR}/Release-iphoneos/${FMK_NAME}.framework
SIMULATOR_DIR=${WRK_DIR}/Release-iphonesimulator/${FMK_NAME}.framework
# -configuration ${CONFIGURATION}
# Clean and Building both architectures.
xcodebuild -configuration "Release" -target "${FMK_NAME}" -sdk iphoneos clean build
xcodebuild -configuration "Release" -target "${FMK_NAME}" -sdk iphonesimulator clean build
# Cleaning the oldest.
if [ -d "${INSTALL_DIR}" ]
then
rm -rf "${INSTALL_DIR}"
fi
mkdir -p "${INSTALL_DIR}"
cp -R "${DEVICE_DIR}/" "${INSTALL_DIR}/"
# Uses the Lipo Tool to merge both binary files (i386 + armv6/armv7) into one Universal final product.
lipo -create "${DEVICE_DIR}/${FMK_NAME}" "${SIMULATOR_DIR}/${FMK_NAME}" -output "${INSTALL_DIR}/${FMK_NAME}"
rm -r "${WRK_DIR}"
open "${INSTALL_DIR}"
```

![](http://i4.tietuku.com/7a5d4723529510fa.png)

- 注意: 如上针对 `release`模式生成的framework

****

###选择 `Aggregate Target`，按下 cmd+B 编译生成framework，会自动弹出 framework的文件夹所在目录

![](http://i4.tietuku.com/f229c73b4c67fc7d.png)

****

###查看生成的framework支持的cpu架构

- 进入到framework的文件夹下

```
cd /Users/xiongzenghui/Desktop/FrameworkDemo/MyFramework/Products/MyFramework.framework
```

- 使用lipo命令查看framework支持的cpu架构

```
lipo -info MyFramework（生成的二进制文件，就是库）
```

- 可以看到如下信息，说明只支持 i386、x86_64、armv7、arm64

```
Architectures in the fat file: MyFramework are: i386 x86_64 armv7 arm64 
```

![](http://i4.tietuku.com/3395dbef459cdd09.png)

- 那么想要支持 iphone 5c了？也就是支持 `armv7s` 架构

![](http://i4.tietuku.com/111b6cdc481b6504.png)

然后重新编译 Agregate Target 生成的 framework就可以了

- 使用lipo命令查看framework支持的cpu架构

![](http://i4.tietuku.com/0a29c1df79ad3116.png)

****

###在framework工程中，添加测试类

- Logger.h

```
#import <Foundation/Foundation.h>

@interface Logger : NSObject

+ (void)log;

@end
```

- Logger.m

```
#import "Logger.h"

@implementation Logger

+ (void)log {
    NSLog(@"ahahahahahahhahaahah.....\n");
}

@end
```

- 将`Logger.h`添加到 `public`区域

- 将`Logger.h`在framework统一向外暴露的头文件中暴露出去

```
#import <UIKit/UIKit.h>

//! Project version number for MyFramework.
FOUNDATION_EXPORT double MyFrameworkVersionNumber;

//! Project version string for MyFramework.
FOUNDATION_EXPORT const unsigned char MyFrameworkVersionString[];

// In this header, you should import all the public headers of your framework using statements like #import <MyFramework/PublicHeader.h>

//此后面，导入我们在工程中的public区域的公共头文件.h
#import <MyFramework/Logger.h>
```

- 然后选中 `Agregate Target`重新 command+B 编译，然后生成新的framework包

****

###新建一个App工程，测试生成的framework，拖到`Embedded Binaries`区域

- 将生成的framework包，直接拖到App工程中

![](http://i4.tietuku.com/b9bd501f5297abc7.png)

- 在App工程的ViewController中测试`Logger`类

```
#import "ViewController.h"

//导入framework中的公共头文件
#import <MyFramework/MyFramework.h>

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    //调用framework提供的类Api
    [Logger log];
}

@end

```

![](http://i4.tietuku.com/98f77c8b95153c8f.png)

***

###小结下制作framework当中的问题

- 支持的cpu架构不全
- framework工程没有配置 `Code Signing Identify`，设置`开发者证书 的 描述Provisioning`
- Category方法找不到，看前面的解决方法

***

###OK，到此为止使用framework打包源码，开发一个framework库的方式记录完毕。


