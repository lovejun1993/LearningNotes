---
layout: post
title: "BaseViewController设计"
date: 2015-08-09 18:21:00 +0800
comments: true
categories: 
---



##基于OOP的思想，我想所有的控制器根据需要去继承自我们的BaseViewController。

###这么做的好处有:
* 很多都要用的功能代码不会散落到每一个ViewController
* 可以提供一些模板方法（代码规则）让其他子类去遵守
* 便于统一添加/移除一些UI效果、网络状态监听
* 统一管理控制器的push/present，让控制器类之间彻底解耦
* 其他的...

***

###No1. 指定一种代码规范让其他程序员遵守，便于维护代码


在baseViewController加上如下方法

```objc

//用于取消请求的代码都写在这
- (void)destroyRequest;

//创建subviews的代码都写在这
- (void)createSubviews;
- (void)destroySubviews;
- (void)reloadSubviews;

//创建变量的代码都写在这
- (void)createVariables;
- (void)destroyVariables;

//绑定ViewModel的代码都写在这
- (void)bindViewModel;
- (void)unbindViewModel;
- (void)enableViewModel;
- (void)disableViewModel;

//获取数据的代码都写在这
- (void)loadData;
- (void)loadDataFromHost;
- (void)reloadDataFromHost;
- (void)loadDataFromLocal;
- (void)saveDataToLocal;

//等等其他就不举例了... 

```

###No2. 封装一些公共的工具性代码

如各种自定义通知 【用户登录状态、网络状态 ... 】

```objc
- (void)handleNotReachableHost;
- (void)handleReachableViaWiFi;
- (void)handleReachableViaWWAN;

- (void)handleUserNotLogin;
- (void)handleUserLogin;
- (void)handleUserLogout;
- (void)handleUserLoginExpirate;

```

###No3. 统一处理所有控制器对象的push/present

第一步、规定每一个ViewController对应一个数值

```objc
typedef NS_ENUM(NSInteger, ZSYViewControllerId) {

	 ZSYHomePage                         = 1,
    ZSYCustomerRegistAndLogin ,
    ZSYCustomerLogin ,
    ZSYCustomerRegist ,
    ZSYCustomerFindPwd ,
    ZSYCustomerIdentiVery ,

	//....
}

```


第二步、写一个封装的push方法

```objc

- (id)pushWithControllerId:(ZSYViewControllerId)controllerId
                 isAnimate:(BOOL)isAnimate;

- (id)pushWithControllerId:(ZSYViewControllerId)controllerId
                     Title:(NSString *)title
                  Argument:(NSDictionary *)argument
                 isAnimate:(BOOL)isAnimate;

```

自定义push方法做的事情:

```objc

	//1. 获取类文件名
    className = [self viewControllerclassNameById:controllerId];
    
    //2. 获取当前控制器的NavigationController
    ZSYBaseNavController *nav = [self findNavigationControllerByCurrentVC];
    
    //3. 查找当前导航控制器是否已经存在该控制
    //...就不贴代码了
    
    //4. 判断使用present 还是 push
    //...
    
    
    //5. 使用push的情况
    //5.1 创建一个控制器对象
    //5.2 填充参数值
    
    //6. push 控制器
    
    //7. 返回 控制器对象
    
    
```