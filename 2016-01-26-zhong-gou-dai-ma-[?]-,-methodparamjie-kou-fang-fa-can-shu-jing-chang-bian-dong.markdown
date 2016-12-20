---
layout: post
title: "重构代码一、MethodParam接口方法参数经常变动"
date: 2016-01-26 12:16:50 +0800
comments: true
categories: 
---



###重构别人的App代码时，发现了一个问题

- 就是一个业务接口的参数，可能会经常改变
- 但是又不好修改业务接口的参数
- 可能造成很多的地方都得修改

***

###解决方法: 将参数列表抽象成一个协议

下面开始demo

***

###登陆和注册业务接口需要的参数抽象 MyServiceRequestParameterProtocols.h

```objc
#import <Foundation/Foundation.h>

/**
 *  抽象 登陆 业务接口的参数
 */
@protocol LoginRequestParameterPtotocol <NSObject>

@required

/**
 *  账号
 */
- (NSString *)username;

/**
 *  密码
 */
- (NSString *)password;

@optional

@end


/**
 *  抽象 注册 业务接口的参数
 */
@protocol RegistRequestParameterPtotocol <NSObject>

//就是将一些参数抽象出来

@end
```

###然后是具体参数封装类，实现 参数抽象协议，提供参数值，MyServiceLoginRequestParamV1.h

.h

```objc
#import <Foundation/Foundation.h>

//导入参数抽象接口
#import "MyServiceRequestParameterProtocols.h"

//实现参数接口，提供参数
@interface MyServiceLoginRequestParamV1 : NSObject <LoginRequestParameterPtotocol>

@property (nonatomic, copy) NSString *loginName;
@property (nonatomic, copy) NSString *loginPassword;

@end
```

.m

```objc
#import "MyServiceLoginRequestParamV1.h"

@implementation MyServiceLoginRequestParamV1

//接口实现方法，提供参数值
- (NSString *)username {
    return self.loginName;
}

- (NSString *)password {
    return self.password;
}

@end
```

###业务类的业务方法声明，只依赖抽象的参数协议的一个实现类对象，而不是单个单个的所有参数

.h

```objc
#import <Foundation/Foundation.h>

//导入参数抽象接口
#import "MyServiceRequestParameterProtocols.h"

@interface MyService : NSObject

/**
 *  业务方法接收 参数接口的 一个实现类的对象
 *  打包所有的单个参数值
 */
- (void)login:(id<LoginRequestParameterPtotocol>)param;

@end
```

.m 

```objc
#import "MyService.h"

@implementation MyService

- (void)login:(id<LoginRequestParameterPtotocol>)param {
    
    //获取接口定义的参数方法
    NSString *username = [param username];
    NSString *password = [param password];
    
    //请求网络...
}

@end
```

###在ViewController使用业务类方法类

```objc
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
 	
 	//1.   
    MyService *service = ...;
    
    //2. 构造参数打包
    MyServiceLoginRequestParamV1 *loginParam = [[MyServiceLoginRequestParamV1 alloc] init];
    loginParam.loginName = @"zhansgan";
    loginParam.loginPassword = @"123456";
    
    //3. 传入打包好的参数给业务方法
    [service login:loginParam];
    
}
```

###假设要修改 登陆接口 的参数，比如添加一个参数xxx，那么首先修改参数接口抽象

```objc
#import <Foundation/Foundation.h>

/**
 *  抽象 登陆 业务接口的参数
 */
@protocol LoginRequestParameterPtotocol <NSObject>

@required

- (NSString *)username;

- (NSString *)password;

/**
 *  新添加的参数
 */
- (NSString *)xxx;

@optional


@end


@protocol RegistRequestParameterPtotocol <NSObject>

//就是将一些参数抽象出来

@end
```

###再对象修改参数协议实现类，添加新的参数方法的实现

```objc
#import <Foundation/Foundation.h>

//导入参数抽象接口
#import "MyServiceRequestParameterProtocols.h"

//实现参数接口，提供参数
@interface MyServiceLoginRequestParamV1 : NSObject <LoginRequestParameterPtotocol>

@property (nonatomic, copy) NSString *loginName;
@property (nonatomic, copy) NSString *loginPassword;

//新添加的方法实现
@property (nonatomic, copy) NSString *xxx;

@end
```

```
#import "MyServiceLoginRequestParamV1.h"

@implementation MyServiceLoginRequestParamV1

//接口实现方法，提供参数值
- (NSString *)username {
    return self.loginName;
}

- (NSString *)password {
    return self.password;
}

- (NSString *)xxx {
    return self.xxx;
}

@end
```

###最后修改业务类的业务方法内部具体实现，添加获取新的参数

```objc
#import "MyService.h"

@implementation MyService

- (void)login:(id<LoginRequestParameterPtotocol>)param {
    
    //获取接口定义的参数方法
    NSString *username = [param username];
    NSString *password = [param password];
    
    //获取新的参数
    NSString *xxx = [param xxx];
    
    //请求网络...
}

@end
```

###在ViewController使用业务类方法类

```objc
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
 	
 	//1.   
    MyService *service = ...;
    
    //2. 构造参数打包
    MyServiceLoginRequestParamV1 *loginParam = [[MyServiceLoginRequestParamV1 alloc] init];
    loginParam.loginName = @"zhansgan";
    loginParam.loginPassword = @"123456";
    
    /**
     *	 只需要多添加一个参数值
     */
     loginParam.xxx = @"xxx";
    
    //3. 传入打包好的参数给业务方法
    [service login:loginParam];
    
}
```


参数接口与实现还有一种写法，但是在Effective Objective-C 2.0书上不提倡

- 接口参数的抽象

```
#import <Foundation/Foundation.h>


/**
 *  所需参数抽象接口定义
 */
@protocol VSCashierDeskPayFailedHandlerArgumentProtocol <NSObject>


@property (nonatomic, strong) NSArray* arrOrders;
@property (nonatomic, assign) BOOL isPrePay;
@property (nonatomic, assign) BOOL isFinalPay;
@property (nonatomic, strong) NSArray* ordersArray;
@property(nonatomic, strong)  NSDate *orderAddTime;

@end
```

- 接口参数实现类

```
@interface VSCashierDeskPayFailedHandlerArgumentProtocolV1_0 : NSObject <VSCashierDeskPayFailedHandlerArgumentProtocol>

@end
```

```
@implementation VSCashierDeskPayFailedHandlerArgumentProtocolV1_0

@synthesize arrOrders;
@synthesize isPrePay;
@synthesize isFinalPay;
@synthesize ordersArray;
@synthesize orderAddTime;

@end
```

****

###小结参数变化优化:

- 所有参数使用一些协议抽象出来
- 所有的参数使用一个协议实现类对象，打包起来
- 业务方法只依赖这个协议的一个具体实现类的`对象`而不是单个单个的参数