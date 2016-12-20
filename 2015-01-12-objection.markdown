---
layout: post
title: "Objection"
date: 2015-01-12 00:37:19 +0800
comments: true
categories: 
---

***

###Objection能干什么？

- objection 是一个轻量级的`依赖注入`的实现框架.

- `依赖注入`的概念

	- 角色1、`接口`，即抽象方法集合.

	- 角色2、`接口的实现`，定义一个具体类，具体实现接口中定义的所有的抽象方法

	- 角色3、`注入器或工厂`，负责完成如下几件事

		- 事情1、管理一系列的`接口`，向外暴露我能做哪些业务功能
		- 事情2、管理一些列`接口的具体实现`，一个接口不同版本的具体实现类
		- 事情3、管理`接口`与`实现`的对应关系
		- 事情4、接收外界传入的一个`接口`来查询到对应的`具体实现类`的一个实例.

****

###JSObjectionModule，管理一个业务模块的所有相关接口与实现的映射关系

####首先，可以抽一个公共的`BaseModule`，提供一些公共代码

```
#import <Foundation/Foundation.h>

//导入Objection类库
#import <JSObjection.h>

@interface BaseModule : JSObjectionModule

//接口与实现绑定器
@property (strong, nonatomic) JSObjectionInjector *injector;

//单例方法
+ (instancetype)module;

//子类Module重写，完成对应的接口与实现的绑定
- (void)bindConfig;

@end
```

```
#import "BaseModule.h"

@implementation BaseModule

+ (instancetype)module {
    static id module  = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        module = [[[self class] alloc] init];
    });
    return module;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        [self bindConfig];
    }
    return self;
}

- (void)bindConfig
{
    //子类覆写
}

@end
```

****

###提供一个所有的Module的统一管理的Manager单例类

```
#import <Foundation/Foundation.h>
#import <JSObjection.h>
#import <JSObjectFactory.h>

@interface ObjectInjectorManager : NSObject

/* 对象依赖注入器 */
@property (strong, nonatomic) JSObjectionInjector *injector;

/* 对象工厂 */
@property (strong, nonatomic) JSObjectFactory *factory;

+ (instancetype)manager;

@end
```

```
#import "ObjectInjectorManager.h"

//1. 全局App都使用的公告模块
#import "ApplicationModule.h"

//2. ViewController模块
#import "ViewControllerModule.h"

//3. Login模块
#import "LoginHandleModule.h"

@implementation ObjectInjectorManager

+ (instancetype)manager {
    static ObjectInjectorManager *injector = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        injector = [[ObjectInjectorManager alloc] init];
    });
    return injector;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        
        [self initInjector];
        [self initFactory];
    }
    return self;
}

- (void)initInjector {
    
    //1. 创建对象注入器
    _injector = [JSObjection defaultInjector];
    _injector = _injector ? : [JSObjection createInjector];
    
    //2. 创建所有模块对象的Module
    ApplicationModule *appModule = [ApplicationModule module];
    ViewControllerModule *controllerModule = [[ViewControllerModule alloc] init];
    LoginHandleModule *loginModule = [[LoginHandleModule alloc] init];
    
    //3. 将所有Module注册到Injector
    _injector = [_injector withModules:appModule, controllerModule, loginModule, nil];
    
    //3. 设置全局对象注入器
    [JSObjection setDefaultInjector:_injector];
}

- (void)initFactory {
    
    //4. 工厂是使用对象注入器完成创建的
    _factory = [[JSObjectFactory alloc] initWithInjector:_injector];
}

@end
```

****

###第一个JSObjectionModule: `ApplicationModule`，来负责一些`依赖对象之间的注入`

####ApplicationModule代码

```
#import <Foundation/Foundation.h>
#import "BaseModule.h"

@interface ApplicationModule : BaseModule

@end
```

```
#import "ApplicationModule.h"
#import "Car.h"

@implementation ApplicationModule

- (void)bindConfig
{
    //1. 管理一个多实例（普通）的类对象
    [self bindClass:[Car class] inScope:JSObjectionScopeNormal];
    
    //2. 管理一个单例类对象
    [self bindClass:[Car class] inScope:JSObjectionScopeSingleton];
}

@end
```

####ApplicationModule使用，负责一些基础的依赖对象之间的注入

> 场景: Car汽车需要Engine引擎与Brake轮胎才能工作.

- Engine实体类

```
#import <Foundation/Foundation.h>

@interface Engine : NSObject

- (void)doSomething;

@end
```

```
#import "Engine.h"

@implementation Engine

- (void)doSomething {
    NSLog(@"Engine ....\n");
}

@end
```

- Brakes实体类

```
#import <Foundation/Foundation.h>

@interface Brakes : NSObject

- (void)doSomething;

@end
```

```
#import "Brakes.h"

@implementation Brakes

- (void)doSomething {
    NSLog(@"Brakes ....\n");
}

@end
```

- Car实体类

```
#import <Foundation/Foundation.h>

#import "Engine.h"
#import "Brakes.h"

@interface Car : NSObject

//下面两个指针引用的对象由注入器创建 
@property(nonatomic, strong) Engine *engine;
@property(nonatomic, strong) Brakes *brakes;

//标志当前对象是否被容器创建完毕
@property(nonatomic) BOOL awake;

//指定的构造器
- (id)initWithName:(NSString *)name Num:(NSNumber *)num;

- (void)doSomething;

@end
```

```
#import "Car.h"

#import <Objection.h>

@interface Car () {
    NSString *_name;
    NSInteger _num;
}

@end


@implementation Car


@synthesize engine, brakes, awake;

//依赖注入对象
objection_requires(@"engine", @"brakes")

//指定执行init函数
objection_initializer(initWithName:Num:)

- (id)initWithName:(NSString *)name Num:(NSNumber *)num {
    
    NSLog(@"这个构造器由容器调用..\n");
    
    self = [super init];
    if (self) {
        _name = name;
        _num = num;
    }
    return self;
}

/* 该对象是否被Objection创建完毕 */
- (void)awakeFromObjection {
    
    NSLog(@"Car类被容器创建完毕..\n");
    
    awake = YES;
}

- (void)doSomething {
    
    NSLog(@"【Car】name = %@ , num = %ld\n", _name, _num);
    
    [engine doSomething];
    [brakes doSomething];
}

@end
```

- ViewController测试

```
@implementation Controller0

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.title = @"依赖对象的注入";
    self.view.backgroundColor = [UIColor whiteColor];
    
    
	[self test1];
	[self test2];
	[self test3];
}

/**
 *  通过Injector获取容器管理的对象
 *  并由容器填充这个对象的其他依赖的对象属性
 */
- (void)test1 {
    
    //1. 获取Manager单例
    ObjectInjectorManager *manager = [ObjectInjectorManager manager];
    
    //2. 从对象注入器中，根据一个Class，查询得到对应的对象
    Car *car = [manager.injector getObject:[Car class]];
    
    //3. 测试
    [car doSomething];
}

/**
 *  通过Factory获取依赖对象
 *  有一点类似test1
 */
- (void)test2 {
    
    //1.
    ObjectInjectorManager *manager = [ObjectInjectorManager manager];
    
    //2.
    Car *car = [manager.factory getObject:[Car class]];
    
    [car doSomething];
}

/**
 *  通Factory获取依赖对象
 *  并执行指定的初始化构造器函数
 */
- (void)test3 {
    
    //1.
    ObjectInjectorManager *manager = [ObjectInjectorManager manager];
    
    //2. 执行指定的初始化构造器函数，并传入初始化参数
    Car *car = [manager.factory getObjectWithArgs:[Car class], @"Car", @19, nil];
    
    //3.
    [car doSomething];
}

@end
```

***

###第二个JSObjectionModule: `ViewControllerModule`，来降低ViewController之间的耦合关系

> 场景: 当前UIViewController某个按钮点击之后，想要跳转到一个显示最新书籍列表的UIViewController2显示数据，但是不想直接`#import "UIViewController2.h"`的方式导入.

- 首先定义一个接口，表示`完成指定类型数据加载`的功能

```
/**
 *  定义能够完成显示最新书籍列表数据的UIViewController
 */
- (void)loadNewBooks;

@end
```

- 接口实现类1，`Controller2`

```
#import <UIKit/UIKit.h>

#import "ControllerPushProtocol.h"

//实现接口
@interface Controller2 : UIViewController <ControllerPushProtocol>

@end
```

```
#import "Controller2.h"

@implementation Controller2


#pragma mark- 接口实现

- (void)loadNewBooks {
    NSLog(@"Controller2 ....\n");
}

- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    
    //加载数据，然后显示在列表
    [self loadNewBooks];
}

@end
```

- 接口实现类2，`Controller3`

```
#import <UIKit/UIKit.h>

#import "ControllerPushProtocol.h"

@interface Controller3 : UIViewController <ControllerPushProtocol>

@end
```

```
#import "Controller3.h"

@implementation Controller3

#pragma mark- 接口实现

- (void)loadNewBooks {
    NSLog(@"Controller3 ....\n");
}

- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    
    //加载数据，然后显示在列表
    [self loadNewBooks];
}

@end
```

- `ViewControllerModule`管理接口与实现的映射关系

```
#import "BaseModule.h"

//接口
#import "ControllerPushProtocol.h"

/* 管理控制器对象的管理 */
@interface ViewControllerModule : BaseModule

@end
```

```
#import "ViewControllerModule.h"

//接口实现类1
#import "Controller2.h"

//接口实现类2
#import "Controller3.h"


@implementation ViewControllerModule


- (void)bindConfig {
    
    //如下可以灵活切换成另一种接口的实现类对象
    //来提供最新书籍列表数据显示的方式
    
    //1. 接口实现类1
	//[self bindClass:[Controller2 class] toProtocol:@protocol(ControllerPushProtocol)];
    
    //2. 接口实现类2
    [self bindClass:[Controller3 class] toProtocol:@protocol(ControllerPushProtocol)];
}


@end
```

- 当前控制器ViewController1，通过一个接口向容器查询到一个`能够显示最新书籍列表的ViewController对象`

```
#import "Controller1.h"

//容器管理类
#import "ObjectInjectorManager.h"

//最新书籍列表的接口
#import "ControllerPushProtocol.h"

@implementation Controller1


- (void)click {
    
    //1. 获取全局Injector
    ObjectInjectorManager *manager = [ObjectInjectorManager manager];
    
    //2. 使用id<接口> 来声明指针，避免直接依赖其他具体类名
    id<ControllerPushProtocol> vc = [manager.injector getObject:@protocol(ControllerPushProtocol)];

    //3. 跳转到这个显示最新书籍列表的控制器
    [self.navigationController pushViewController:(UIViewController *)vc animated:YES];

}

@end
```

####这么做的好处:

- ViewConroller1根本不知道显示最新书籍列表的是`ViewController2` 还是 `ViewController3`

- 随时可以从`ViewController2` 切换到 `ViewController3`

****

###第三个JSObjectionModule: `LoginHandleModule`，管理登陆业务模块的接口与实现映射关系


> 场景: 登陆有三种登陆方式: 1)输入账号与密码登陆 2)手势密码登陆 3)电话号码登陆


- 定义业务方法的抽象方法集合（接口）

```
#import <Foundation/Foundation.h>

FOUNDATION_EXPORT NSString *const UserNameLoginKey;
FOUNDATION_EXPORT NSString *const GestureLoginKey;
FOUNDATION_EXPORT NSString *const PhonumLoginKey;

#pragma mark - 接口

@protocol LoginProtocol <NSObject>

- (void)handleLogin;

@end
```

- 写一个接口默认实现类，作为其他具体类型实现的`父类`

```
@interface LoginHandlebase : NSObject <LoginProtocol>

@end
```

```
#import "Loginlization.h"

NSString *const UserNameLoginKey = @"UserNameLoginKey";
NSString *const GestureLoginKey = @"GestureLoginKey";
NSString *const PhonumLoginKey = @"PhonumLoginKey";

@implementation LoginHandlebase

- (void)handleLogin {//空实现，由子类具体完成自己的实现}

@end
```

- 具体子类实现1、`UserNameLogin`

```
#import "Loginlization.h"

//继承自接口默认实现类
@interface UserNameLogin : LoginHandlebase

@end
```

```
#import "UserNameLogin.h"

@implementation UserNameLogin

- (void)handleLogin {
    NSLog(@"用户名密码登陆...\n");
}

@end
```

- 具体子类实现2、`GestureLogin`

```
#import "Loginlization.h"

//继承自接口默认实现类
@interface GestureLogin : LoginHandlebase

@end
```

```
#import "GestureLogin.h"

@implementation GestureLogin

- (void)handleLogin {
    NSLog(@"手势登陆...\n");
}

@end
```

- 具体子类实现3、`PhonumLogin`

```
#import "Loginlization.h"

//继承自接口默认实现类
@interface PhonumLogin : LoginHandlebase

@end
```

```
#import "PhonumLogin.h"

@implementation PhonumLogin

- (void)handleLogin {
    NSLog(@"手机号码登陆...\n");
}

@end
```

- `LoginHandleModule`完成对接口与n个具体实现类，使用`不同的name`进行映射，获取接口实现时也通过不同的name获取到不同的实现类对象.

```
#import "BaseModule.h"

//继承自BaseModule
@interface LoginHandleModule : BaseModule

@end
```

```
#import "LoginHandleModule.h"

//业务接口
#import "Loginlization.h"

//业务接口实现类1
#import "UserNameLogin.h"

//业务接口实现类2
#import "GestureLogin.h"

//业务接口实现类3
#import "PhonumLogin.h"

@implementation LoginHandleModule

- (void)bindConfig {
    
    //一个接口可以同时绑定多个不同的实现类对象
    //使用不同的name值来区分
    
    [self bindClass:[UserNameLogin class]
         toProtocol:@protocol(LoginProtocol)
            inScope:JSObjectionScopeSingleton//单例存在
              named:UserNameLoginKey];
    
    [self bindClass:[GestureLogin class]
         toProtocol:@protocol(LoginProtocol)
            inScope:JSObjectionScopeNormal//多实例
              named:GestureLoginKey];
    
    [self bindClass:[PhonumLogin class]
         toProtocol:@protocol(LoginProtocol)
            inScope:JSObjectionScopeNone//多实例
              named:PhonumLoginKey];
    
}

@end
```

- LoginViewController测试类，任意切换登陆方式

```
- (void)UserNameAndPasswordLogin 
{
	//1. 
	ObjectInjectorManager *manager = [ObjectInjectorManager manager];
            
	//使用不同的name，通过Injector查询到接口实现类对象
    id<LoginProtocol> userNameLogin = [manager.injector getObject:@protocol(LoginProtocol)
                                                            named:UserNameLoginKey
                                                     argumentList:nil];
    
    //3. 
    [userNameLogin handleLogin];

}
```

```
- (void)gesturePasswordLogin 
{
	
	ObjectInjectorManager *manager = [ObjectInjectorManager manager];
            
	//使用不同的name，通过Injector查询到接口实现类对象
    id<LoginProtocol> gestureLogin = [manager.injector getObject:@protocol(LoginProtocol)
                                                           named:GestureLoginKey
                                                    argumentList:nil];
    
    [gestureLogin handleLogin];
}
```

```
- (void)mobileLogin 
{

	ObjectInjectorManager *manager = [ObjectInjectorManager manager];
	    
	//使用不同的name，通过Injector查询到接口实现类对象
	id<LoginProtocol> phonumLogin = [manager.injector getObject:@protocol(LoginProtocol)
	                                                        named:PhonumLoginKey
	                                                 argumentList:nil];
	    
	[phonumLogin handleLogin];
            
}
```

****

OK，大概的功能就这么多了，有时间再看...