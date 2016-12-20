---
layout: post
title: "代码重构五、续解耦ViewController方法之模拟URLScheme"
date: 2016-01-26 22:31:37 +0800
comments: true
categories: 
---

###避免在ViewControllerA中导入其他ViewControllerB.h、ViewControllerC.h。之前记录了两种方法:

- 使用 一个枚举值 映射 一个ViewController，提供一个父类ViewController
	- 此种方法，可能会造成 枚举值 越来越多

- 使用 抽象接口 隔离
	- 此种方法，可能会造成 抽象接口 数量过大

***

###本文记录另外一种方法，有点类似 URL Scheme 跳转其他App的形式

####URL Scheme 打开其他App程序:

- 应用程序A 通过一个特定的 `URL Scheme，如: xzh://` 
- 然后通过UIApplication来掉起对应的一个特定的 应用程序B

####如上过程中关键点分析

- 关键点1、一个特定的 `URL Scheme，如: xzh://` 对应 一个特定的应用程序B
	- 核心: **一个唯一的 URL Scheme 映射 一个唯一App程序**

- 关键点2、最后由`系统作为中间助手`，打开对应的应用程序B
	- 如上虽然是通过UIApplication的方法打开，最终肯定是通过`iOS系统`来打开其他程序的
	- 核心: **需要一个中间助手完成跳转其他程序**

****

###URL Scheme 解耦 `应用程序之间` 的强依赖

- 一个唯一的URL Scheme 映射 一个唯一App程序
- 需要一个 中间助手 完成跳转到 这个唯一Scheme 对应的 App程序

***

###按照 URL Scheme 套路来 解耦 `ViewController之间` 的强依赖

- 一个唯一的 URL Scheme 映射 一个唯一的ViewController类
- 需要一个 中间类 完成Push到 这个唯一Scheme 对应的 ViewController对象

****

###角色一、保存一个映射关系的实体类:

> 一个唯一的 URL Scheme 映射 一个唯一的ViewController类

```objc
//
//  XPushRelationShip.h
//  XControllerManager
//
//  Created by xiongzenghui on 16/1/25.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import <Foundation/Foundation.h>

/**
 *  描述 URL 与 ViewController 的映射关系模型
 */
@interface XPushRelationShip : NSObject

/** ULR Scheme*/
@property (nonatomic, copy) NSString *xscheme;

/** 类名 */
@property (nonatomic, copy) NSString *xclassName;

/** 描述 */
@property (nonatomic, copy) NSString *xdescription;

/** 配置需要的参数 */
@property (nonatomic, strong) NSDictionary *xarguments;

@end
```

```objc
//
//  XPushRelationShip.m
//  XControllerManager
//
//  Created by xiongzenghui on 16/1/25.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import "XPushRelationShip.h"

@implementation XPushRelationShip

@end
```

###角色二、统一管理所有Scheme映射ViewController的关系

- 映射关系的，增加、删除、修改、查询

```objc
//
//  XMappingManager.h
//  XControllerManager
//
//  Created by xiongzenghui on 16/1/25.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import <Foundation/Foundation.h>
#import "XPushRelationShip.h"

/**
 *  管理所有的映射关系
 * 
 *  key 是 URLScheme， 如 xzh://
 *  value 是 XPushRelationShip实例，保存ViewController的类名...等参数
 */
@interface XMappingManager : NSObject

+ (instancetype)xsharedInstance;

/** 添加一个映射规则 */
+ (void)xappendMapping:(XPushRelationShip *)ship;

/** 移除一个映射规则 */
+ (void)xremoveMappingWithURLString:(NSString *)urlStr;

/** 更新一个映射规则 */
+ (void)xupdateMappingWithURLString:(NSString *)scheme;

/** 使用scheme查询一个映射规则 */
+ (XPushRelationShip *)xrelationShipWithURLString:(NSString *)urlStr;

/** NSURL是否存在映射关系 */
+ (BOOL)xisAvailableForURL:(NSURL *)url;

/** Scheme是否存在映射关系 */
+ (BOOL)xisAvailableForScheme:(NSString *)scheme;

@end

@interface XMappingManager (Convinient)

/**
 *  快速添加一个 Scheme映射ViewController 的关系
 */
+ (void)xappendMappingWithScheme:(NSString *)scheme
         ViewControllerClassName:(NSString *)className
                     Description:(NSString *)des
                       Arguments:(NSDictionary *)arguments;


@end
```

```objc
//
//  XMappingManager.m
//  XControllerManager
//
//  Created by xiongzenghui on 16/1/25.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import "XMappingManager.h"
#import "NSURL+Addtions.h"
#import "XTools.h"

@interface XMappingManager ()

@property (nonatomic, strong) NSMutableDictionary *mapping;
@property (nonatomic, strong) dispatch_queue_t serialQueue;

@end

@implementation XMappingManager

- (void)dealloc {
    _mapping = nil;
    _serialQueue = nil;
}

- (dispatch_queue_t)serialQueue {
    if (!_serialQueue) {
        _serialQueue = dispatch_queue_create("com.cn.xiongzenghui.mapping.queue", DISPATCH_QUEUE_SERIAL);
    }
    return _serialQueue;
}

+ (instancetype)xsharedInstance
{
    static XMappingManager *manager = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        
        manager = [[XMappingManager alloc] init];
        [manager xloadDiskConfigFiles];
    });
    return manager;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        dispatch_async(self.serialQueue, ^{
            self.mapping = [[NSMutableDictionary alloc] init];
        });
    }
    return self;
}

- (void)xloadDiskConfigFiles
{
    //TODO: 加载磁盘默认配置文件
}

+ (void)xappendMapping:(XPushRelationShip *)ship
{
    if ([self xrelationShipWithURLString:[ship xscheme]]) {
        return;
    }
    
    dispatch_async([XMappingManager xsharedInstance].serialQueue, ^{
        [[XMappingManager xsharedInstance].mapping setObject:ship forKey:[ship xscheme]];
    });
}

+ (void)xremoveMappingWithURLString:(NSString *)urlStr
{
    XPushRelationShip *ship = [self xrelationShipWithURLString:urlStr];
    
    if (!ship) return;
    
    dispatch_async([XMappingManager xsharedInstance].serialQueue, ^{
        [[XMappingManager xsharedInstance].mapping removeObjectForKey:[ship xscheme]];
    });
}

+ (void)xupdateMappingWithURLString:(NSString *)scheme
{
    XPushRelationShip *ship = [self xrelationShipWithURLString:scheme];
    
    if (!ship) return;

    dispatch_async([XMappingManager xsharedInstance].serialQueue, ^{
        ship.xscheme = scheme;
    });
}

+ (XPushRelationShip *)xrelationShipWithURLString:(NSString *)urlStr
{
    __block XPushRelationShip *ship = nil;
    
    [[XMappingManager xsharedInstance].mapping enumerateKeysAndObjectsUsingBlock:^(id  _Nonnull key, id  _Nonnull obj, BOOL * _Nonnull stop) {
        
        if ([key isEqualToString:urlStr]) {
            
            ship = obj;
            
            *stop =YES;
        }
    }];
    
    return ship;
}

+ (BOOL)xisAvailableForURL:(NSURL *)url
{
    //取出URL的scheme
    NSString *scheme = url.xschemString;
    
    return [self xisAvailableForScheme:scheme];
}

+ (BOOL)xisAvailableForScheme:(NSString *)scheme
{
    //scheme不能为空
    if (!scheme || [scheme isEqualToString:@""]) {
        return NO;
    }
    
    //缓存的所有的Scheme
    NSArray *keys = [[XMappingManager xsharedInstance].mapping allKeys];
    
    //是否包含这个Scheme
    if ([keys containsObject:scheme]) {
        return YES;
    }
    
    return NO;
}

@end

@implementation XMappingManager (Convinient)

+ (void)xappendMappingWithScheme:(NSString *)scheme
         ViewControllerClassName:(NSString *)className
                     Description:(NSString *)des
                       Arguments:(NSDictionary *)arguments
{
    NSParameterAssert(scheme);
    
    //创建一个新的映射关系对象
    XPushRelationShip *ship = [[XPushRelationShip alloc] init];
    
    //保存传入的url scheme
    ship.xscheme = [scheme copy];
    
    //url scheme 对应的 类名
    ship.xclassName = [className copy];
    
    //映射关系的描述信息
    if (des) {
        ship.xdescription = [des copy];
    }
    
    //映射关系提供的默认参数
    if (arguments) {
        ship.xarguments = arguments;
    }
    
    //完成添加映射关系
    [self xappendMapping:ship];
}

@end
```

如上可以做的扩展:

- 读取磁盘配置文件，加载映射关系

***

###角色3、统一完成最后Scheme对应ViewController对象的push操作的类

```objc
//
//  XURLPushManager.h
//  XControllerManager
//
//  Created by xiongzenghui on 16/1/25.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import <Foundation/Foundation.h>
#import <UIKit/UIKit.h>

//这个protocol定义的是popViewController时
//通过delegate方式完成回调
#import "XURLPushOnPopCallBackProtocolization.h"

/**
 *  使用URL与UIViewController映射
 *  解决ViewController直接直接的耦合
 */
@interface XURLPushManager : NSObject

/** 单例 */
+ (instancetype)xSharedInstance;

/**
 *  最终负责push viewController
 *
 *  @param url      请求路径
 *  @param argument 请求参数
 *  @param block    回调block
 */
- (void)x_pushWithURL:(NSURL *)url Argument:(NSDictionary *)argument RootNav:(UINavigationController *)rootNav  OnPopDelegate:(id<XURLPushOnPopCallbackDelegate>)popDelegate;

/**
 *  创建一个ViewController，并设置传入的参数
 *
 *  @param url      请求路径
 *  @param argument 请求参数
 */
- (UIViewController *)x_viewControllerWithURL:(NSURL *)url Argument:(NSDictionary *)argument OnPopDelegate:(id<XURLPushOnPopCallbackDelegate>)popDelegate;

@end
```

```objc

//
//  XURLPushManager.m
//  XControllerManager
//
//  Created by xiongzenghui on 16/1/25.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import "XURLPushManager.h"
#import "XMappingManager.h"
#import "UIViewController+URLAssociate.h"
#import "NSURL+Addtions.h"
#import "XTools.h"

@implementation XURLPushManager

+ (instancetype)xSharedInstance {
    static XURLPushManager *manager = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        manager = [[XURLPushManager alloc] init];
    });
    return manager;
}

- (void)x_pushWithURL:(NSURL *)url
             Argument:(NSDictionary *)argument
              RootNav:(UINavigationController *)rootNav
        OnPopDelegate:(id<XURLPushOnPopCallbackDelegate>)popDelegate
{
    //1. 获取对应的ViewController实例
    id vc = [self x_viewControllerWithURL:url Argument:argument OnPopDelegate:popDelegate];
    
    //2. 调用即将被push的ViewController对象
    if ([vc isKindOfClass:[UIViewController class]]) {
        
        //2.1 询问即将被push的控制器，是否执行当前URL的push操作
        if ([vc respondsToSelector:@selector(shouldOpenViewControllerByURL:)]) {
            BOOL shuoldOpen = [vc shouldOpenViewControllerByURL:url];
            
            if (!shuoldOpen) {
                return;
            }
        }
        
        //2.2 通知目标控制器被push后作出的处理
        if ([vc respondsToSelector:@selector(openedFromViewControllerByURL:)]) {
            
            //2.2.1 执行push viewController操作
            [rootNav pushViewController:vc animated:YES];
            
            //2.2.2 通知目标控制器执行处理
            [vc openedFromViewControllerByURL:url];
        }
    }
    
    //扩展push其他控制器类型
    //....
}

- (UIViewController *)x_viewControllerWithURL:(NSURL *)url Argument:(NSDictionary *)argument OnPopDelegate:(id<XURLPushOnPopCallbackDelegate>)popDelegate
{
	//1. 获取Scheme字符串，如: xzh://
	//因为NSURL.scheme返回的字符串不包含 `://`，需要自己拼接
    NSString *schemeString = [url xschemString];
    
    //2. 判断当前NSURL的Scheme是否存在映射关系
    if (![XMappingManager xisAvailableForScheme:schemeString]) {
        return nil;
    }
    
    //3. 使用scheme来查询得到一个 映射关系（XPushRelationShip实例）
    XPushRelationShip *ship = [XMappingManager xrelationShipWithURLString:schemeString];
    
    if (!ship) {
        return nil;
    }
    
    //4. 从找到的映射关系对象中，获取目标ViewController的 `类名`字符串
    NSString *clsName = ship.xclassName;
    
    //5. 得到Class
    Class cls = NSClassFromString(clsName);
    
    //6. 参数处理（1、预先配置参数 2、当前传入的参数）
    NSMutableDictionary *mutableArgment = [NSMutableDictionary dictionary];
    if (argument) {
        [mutableArgment addEntriesFromDictionary:argument];
    }
    if (ship.xarguments) {
        [mutableArgment addEntriesFromDictionary:ship.xarguments];
    }
    argument = [mutableArgment copy];
    
    //7. 创建目标ViewController对象，
    //并初始化时，传入 打开的URL、参数
    UIViewController *vc = [[cls alloc] initWithURL:url Argument:argument CallBackDelegate:popDelegate];

    return vc;
}
@end
```

####小结下主要完成的事情:

- 找到当前NSURL对应的ViewController对象

- ViewController对象使用objc_associate动态绑定参数值
	- 当前打开的NSURL
	- 参数Argument字典
	- 目标ViewController对象被pop时的回调delegate

- 使用传入的UINavigationController完成pushViewController操作
	- 第一步、首先调用`shouldOpenViewControllerByURL:`方法的返回值，来确定是否打开当前NSURL的push操作
	- 第二步、如果前一步返回YES，那么继续调用`openedFromViewControllerByURL:`通知目标ViewController被push了

***

###角色4、提供动态绑定参数值，以及提供其他ViewController 重写 的对象方法（用于模拟Open URL的效果）

```objc
//
//  UIViewController+URLAssociate.h
//  XControllerManager
//
//  Created by xiongzenghui on 16/1/25.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import <UIKit/UIKit.h>
#import "XURLPushOnPopCallBackProtocolization.h"

@interface UIViewController (URLAssociate)


////////////////////////////////////////////////////////////////////////
////// 初始化传入:
////// 1. 打开目标控制器的URL
////// 2. 传给目标控制器的参数字典
////// 3. 目标控制器被pop时回调delegate
////////////////////////////////////////////////////////////////////////

- (instancetype)initWithURL:(NSURL *)url
                   Argument:(NSDictionary *)argument
           CallBackDelegate:(id<XURLPushOnPopCallbackDelegate>)delegate;

////////////////////////////////////////////////////////////////////////
////// 动态绑定变量
////////////////////////////////////////////////////////////////////////

- (NSURL *)URL;
- (void)setURL:(NSURL *)url;

- (void)setArgument:(NSDictionary *)argument;
- (NSDictionary *)argument;

- (id<XURLPushOnPopCallbackDelegate>)popDelegate;
- (void)setPopDelegate:(id<XURLPushOnPopCallbackDelegate>)delegate;

- (void)setBackdata:(id)data;
- (id)backdata;

////////////////////////////////////////////////////////////////////////
////// 如下是提供给其他ViewController 重写 的对象方法
////////////////////////////////////////////////////////////////////////

/**
 *  被其他控制器push时触发
 */
- (void)openedFromViewControllerByURL:(NSURL *)aUrl;

/**
 *  是否允许当前URL的push操作
 */
- (BOOL)shouldOpenViewControllerByURL:(NSURL *)aUrl;


@end
```

```objc
//
//  UIViewController+URLAssociate.m
//  XControllerManager
//
//  Created by xiongzenghui on 16/1/25.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import "UIViewController+URLAssociate.h"
#import <objc/runtime.h>

NSString *UIViewControllerURLKey            = @"UIViewControllerURLKey";
NSString *UIViewControllerArgumentKey       = @"UIViewControllerArgumentKey";
NSString *UIViewControllerPopCallbackDelegateKey       = @"UIViewControllerXURLPushOnPopCallbackDelegateKey";
NSString *UIViewControllerBackDataKey       = @"UIViewControllerBackDataKey";

@implementation UIViewController (URLAssociate)


- (instancetype)initWithURL:(NSURL *)url
                   Argument:(NSDictionary *)argument
           CallBackDelegate:(id<XURLPushOnPopCallbackDelegate>)delegate
{
    self = [self init];
    if (self) {
        [self setURL:url];
        [self setArgument:argument];
        [self setPopDelegate:delegate];
    }
    return self;
}

- (NSURL *)URL {
    return objc_getAssociatedObject(self, &UIViewControllerURLKey);
}

- (void)setURL:(NSURL *)url {
    
    objc_setAssociatedObject(self,
                             &UIViewControllerURLKey,
                             url,
                             OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (NSDictionary *)argument {
    return objc_getAssociatedObject(self, &UIViewControllerArgumentKey);
}

- (void)setArgument:(NSDictionary *)argument {
    objc_setAssociatedObject(self,
                             &UIViewControllerArgumentKey,
                             argument,
                             OBJC_ASSOCIATION_COPY);
}

- (id<XURLPushOnPopCallbackDelegate>)popDelegate {
    return objc_getAssociatedObject(self, &UIViewControllerPopCallbackDelegateKey);
}

- (void)setPopDelegate:(id<XURLPushOnPopCallbackDelegate>)delegate {
    objc_setAssociatedObject(self,
                             &UIViewControllerPopCallbackDelegateKey,
                             delegate,
                             OBJC_ASSOCIATION_ASSIGN);
}

- (id)backdata {
    return objc_getAssociatedObject(self, &UIViewControllerBackDataKey);
}

- (void)setBackdata:(id)data {
    objc_setAssociatedObject(self,
                             &UIViewControllerBackDataKey,
                             data,
                             OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

#pragma mark - 空实现，由其他具体ViewController根据需要完成自己的实现

- (void)openedFromViewControllerByURL:(NSURL *)aUrl {}

- (BOOL)shouldOpenViewControllerByURL:(NSURL *)aUrl {return YES;}

@end
```

###到此为止，基本上可以完成ViewController中，使用一个Scheme就可以push到一个具体的ViewControllerA中，而不需要导入ViewControllerA.h


- ViewController.m

```objc
#import "ViewController.h"
#import "XMappingManager.h"

@interface ViewController () <XURLPushOnPopCallbackDelegate>

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    //1. `注册 URL Scheme` 与 ViewController Protol 的映射
    [XMappingManager xappendMappingWithScheme:@"xzh://"
                      ViewControllerClassName:@"ViewControllerA"
                                  Description:@"最新新闻列表"
                                    Arguments:@{@"key1":@"value1"}];
    
    
    //2. push时，可以传入URL的形式:
    //形式一、xzh://
    //形式二、xzh://AAA
    //形式三、xzh://AAA?key=value
    [self xpushWithURL:@"xzh://" RootNav:self.navigationController OnPopDelegate:self];   
}

#pragma mark - pop时回调带来方法实现

- (void)dismissedFromViewControllerForURL:(NSURL *)aUrl backData:(id)data
{
    NSLog(@"pop掉的控制器URL: %@, 接收的回传值: %@", aUrl, data);
}

@end
```


- ViewControllerA.m

```objc
//
//  ViewControllerA.m
//  XControllerManager
//
//  Created by xiongzenghui on 16/1/25.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import "ViewControllerA.h"
#import "UIViewController+URLAssociate.h"


@implementation ViewControllerA

- (void)viewDidLoad {
    [super viewDidLoad];
    
}

////////////////////////////////////////////////////////////
/////告诉XURLPushManager，是否需允许当前URL完成push操作（被打开）
////////////////////////////////////////////////////////////
- (BOOL)shouldOpenViewControllerByURL:(NSURL *)aUrl {
    
    /**
     *  登陆操作
     */
    if ([aUrl.absoluteString isEqualToString:@"xzh://xiongzenghu/login"]) {
        //判断是否打开，进行业务操作1
        //...
        return YES;
    }
    
    /**
     *  注册操作
     */
    if ([aUrl.absoluteString isEqualToString:@"xzh://xiongzenghu/regist"]) {
        //判断是否打开，进行业务操作2
        //...
        return YES;
    }

    
    //对于其他URL不允许被push
    return NO;
}

////////////////////////////////////////////////////////////
/////由XURLPushManager完成push后的回调函数
////////////////////////////////////////////////////////////
- (void)openedFromViewControllerByURL:(NSURL *)aUrl {
    
    //目标控制器被push打开
    
    //1. 获取传过来的请求参数
    NSDictionary *params = [self argument];
    NSLog(@"参数: %@", params);
    
    //2. 获取打开的URL
    NSLog(@"url = %@", aUrl);
    NSLog(@"url = %@", [self URL]);
    
    //3. 处理完业务后，使用delegate形式回传参数
    [self setBackdata:@"哈哈回传值"];
}


@end
```

再看下两个Controller.m依赖的文件

- ViewController.m依赖的

```
//提供映射关系管理
#import "XMappingManager.h"

//提供调用XURLPushManger的便利方法
#import "UIViewController+PushConvinient.h"
```

- ViewControllerA.m依赖的

```
//提供动态绑定URL、参数字典、回调delegate的方法
#import "UIViewController+URLAssociate.h"
```

如上我们需要单独抽离出 `ViewController.h与ViewController.m`遇到的依赖

```
//提供映射关系管理
#import "XMappingManager.h"

//提供调用XURLPushManger的便利方法
#import "UIViewController+PushConvinient.h"
```

仅仅只是牵涉到自己封装的类库Api，没有牵涉到`ViewControllerA.h`，那么被单独拖到其他工程中时没有太多的具体类依赖，就是这样的效果...

***

###继续完成ViewControllerA被pop时，回调ViewController对象

```objc
//
//  UINavigationController+Intercept.h
//  XControllerManager
//
//  Created by xiongzenghui on 16/1/25.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import <UIKit/UIKit.h>

@interface UINavigationController (Intercept) <UINavigationControllerDelegate>

@end
```

```objc
//
//  UINavigationController+Intercept.m
//  XControllerManager
//
//  Created by xiongzenghui on 16/1/25.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import "UINavigationController+Intercept.h"
#import "NSObject+XSwizzle.h"
#import "XControllerManager.h"
#import "UIViewController+URLAssociate.h"

@implementation UINavigationController (Intercept)

+ (void)load {
    
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        
        [self xnavigationController_exchangedImplemetations];
        
    });
}


+ (void)xnavigationController_exchangedImplemetations {
    NSError *error = nil;
    
    [self x_swizzleMethod:@selector(viewDidLoad)
               withMethod:@selector(_xCustom_viewDidLoad)
                    error:&error];
    
    if (error) {
        XLog(@"XControllerManagerError : %@\n", error);
    } else {
        error = nil;
    }
    
    [self x_swizzleMethod:@selector(pushViewController:animated:)
               withMethod:@selector(_xCustom_pushViewController:animated:)
                    error:&error];
    
    if (error) {
        XLog(@"XControllerManagerError : %@\n", error);
    } else {
        error = nil;
    }
    
    [self x_swizzleMethod:@selector(popViewControllerAnimated:)
               withMethod:@selector(_xCustom_popViewControllerAnimated:)
                    error:&error];

    if (error) {
        XLog(@"XControllerManagerError : %@\n", error);
    } else {
        error = nil;
    }
    
//    [self x_swizzleMethod:@selector(popToViewController:animated:)
//               withMethod:@selector(_xCustom_popToViewController:animated:)
//                    error:&error];
    
    if (error) {
        XLog(@"XControllerManagerError : %@\n", error);
    } else {
        error = nil;
    }
    
//    [self x_swizzleMethod:@selector(popToRootViewControllerAnimated:)
//               withMethod:@selector(_xCustom_popToRootViewControllerAnimated:)
//                    error:&error];
    
    if (error) {
        XLog(@"XControllerManagerError : %@\n", error);
    } else {
        error = nil;
    }

}

- (void)_xCustom_viewDidLoad
{
    [self _xCustom_viewDidLoad];
    
    self.delegate = self;
}

- (void)_xCustom_pushViewController:(UIViewController *)viewController animated:(BOOL)animated
{
    [self _xCustom_pushViewController:viewController animated:animated];
}

- (UIViewController *)_xCustom_popViewControllerAnimated:(BOOL)animated
{
    //1. 获取当前被pop掉的控制器
    id poped = [self _xCustom_popViewControllerAnimated:animated];
    
    //2. 通知deelegate即将pop
    if ([poped isKindOfClass:[UIViewController class]] && \
        [poped popDelegate] && \
        [[poped popDelegate] respondsToSelector:@selector(dismissedFromViewControllerForURL:backData:)])
    {
        //2.1 取出当前被pop的控制器保存的 `URL`
        NSURL *url = [(UIViewController *)poped URL];
        
        //2.2 取出当前被pop的控制器保存的 `回传值`
        id backdata = [(UIViewController *)poped backdata];
        
        //2.3 通过delegate形式完成回调
        [[(UIViewController *)poped popDelegate] dismissedFromViewControllerForURL:url backData:backdata];
    }
    
    return poped;
}

#pragma mark - UINavigationControllerDelegate

- (void)navigationController:(UINavigationController *)navigationController willShowViewController:(UIViewController *)viewController animated:(BOOL)animated
{
    
}

- (void)navigationController:(UINavigationController *)navigationController didShowViewController:(UIViewController *)viewController animated:(BOOL)animated
{
    
}

@end
```

通过Method Swizzle方式HOOK系统UINavigationController的方法完成.

***

###大致类记录完毕，当需要将ViewController拖到其他工程时

- 不会有任何的`其他具体ViewController.h`依赖
- 仅仅只是依赖 `完成这一系列映射关系的封装代码库`
- 把封装的代码库也一起拖到其他工程即可，然后完成`修改Scheme与ViewController的映射关系`即可

***

###当然如上代码只是涉及一些简单的情况，一种解耦思路吧.