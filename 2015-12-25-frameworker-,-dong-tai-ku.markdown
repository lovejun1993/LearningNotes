---
layout: post
title: "Framework二、动态库"
date: 2015-12-25 17:02:18 +0800
comments: true
categories: 
---

###iOS目前运行的几种动态更新的机制:

- 1 `UIWebView`加载H5页面（最古老的方式），4是基于此
- 2 `lua脚本`控制动态更新（最麻烦最糟糕的方式）
- 3 `React Native`框架，使用`js`动态更新（最时髦的方式）
- 4 `PhoneGap 或 Cordova`框架使用下载服务器H5压缩包到本地解压
- 5 `Framework`模块化后`动态库`式更新（最`危险`的方式）

####所以大部分都是使用 3、4

- 都是通过向服务器下载到 `JS包` 或 `H5包`到手机本地

- 然后解压缩之后加载js或h5源码

- H5源码是通过使用UIWebView实现

- 3 优于 4，因为iOS9已经支持js

####也就是说暂时没有应用，通过使用 `动态加载framework`来完成动态更新，但是framework提供原生代码，所以还是有很多便利性

***

###动态framework 与 静态framework

- 动态的 framework，可以在程序`运行`时，加载到内存使用
- 静态的 framework，只可以在编写代码时，#import库中的头文件，也就是在`编译`阶段
- 动态的 framework，也可以像 静态的 framework 一样，在编写代码时 #import导入库头文件，此时`动态的framework`其实没有产生作用，就像`静态的framework`一样

****

###运行时加载framework，并使用里面的类

- 首先，将framework 赋值到 main bundle

![](http://i4.tietuku.com/ce8e7634154fc51a.png)

- 代码中运行时获取framework

```
#import "ViewController.h"

#import <MyFramework/MyFramework.h>

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    

    [self testReadFrameworkFile];
}

- (void)testReadFrameworkFile {
    
    //1.  获取framework文件 所在的路径
    NSString *frameworkPath = [[NSBundle mainBundle] pathForResource:@"MyFramework.framework" ofType:nil];
    
    //2. 使用NSBundle加载 framework文件
    NSError *err = nil;
    NSBundle *bundle = [NSBundle bundleWithPath:frameworkPath];

    if ([bundle loadAndReturnError:&err]) {
        NSLog(@"bundle load framework success.");
    } else {
        NSLog(@"bundle load framework err:%@",err);
    }
    
    //3. 获取 framework里面的 Class 和 Object
    Class rootClass = NSClassFromString(@"Logger");
    
    //4. 调用对象方法
    if (rootClass) {
        id object = [[rootClass alloc] init];
        Logger *logger = (Logger *)object;
		 [Logger log];
    }
    
    //5. 调用类方法
    [rootClass log];
}

@end
```

****

###尽管可以通过 程序运行时 ，动态加载 Dynamic Framework 达到热更新的效果，但是苹果不建议这么搞，所以此种方式只适合完成一些`急时的Bug修复`。