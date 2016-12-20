---
layout: post
title: "最近面试都是问你时怎么做项目架构设计的"
date: 2015-09-23 17:55:51 +0800
comments: true
categories: 
---


###如题，最近又开始面试了，然后基本上都会问到你的项目是如何做架构设计的？



***

###主要涉及的模块:

- 网络数据交互
	- 提供二次封装隔离第三方开源框架实现
	- 网络数据传输优化

- 本地数据存储
	- 实体类数据
		- SQLite/FMDB/CoreData/NSCocing
		- 一些自定义缓存类库，eg: Kache、YYCache、TMCache
	- 网络数据response data
		- NSData >>> NSKeyedAchiver/NSKeydUnAchiver to/from Disk file 

- MVVM业务架构模式
	- ViewModel单独封装业务代码
		- 提供统一的BaseViewModel提供一些基础性的功能代码
		- 提供统一的业务抽象
		- 再内部管理众多版本的业务具体实现
		- 通过开源Ioc依赖注入框架Objection来管理 `@protocol : 具体NSObject实现类`之间的关系
		- 以及

		
- BaseViewController
	- 结构化、步骤化的模板方法
	- 统一push管理
	- 统一的拦截处理

- BaseCell、BaseTableViewCell



***

###网络请求框架选择

网络请求这一块一直是我比较关心的，从最开始使用`ASIHttpRequest`全部基于delegate执行回调，而且源码n多，又是mrc，实在是有一点吃不消.

然后我尝试在`github`查找了一下网络类库，发现一个叫做`MKNetworkKit`这个类库。首先看了他介绍的用法，也就是从这里我才开始慢慢接触到了Block使用。用了一段时间之后，决定看一下源码，而看了之后，发现他的代码还是很少，就几个文件而已。而且已经自己做了`responseData`缓存】、并且支持`断网时将网络请求operation归档到本地，当网络恢复后继续执行`..反正就研究了一段时间.

然而，在`github`上又发现了一个飙升的网络类库`AFNetworking`。然后去了他主页看了一下，发现也是全部基于Block实现回调。并且发现他其实有1.x和2.x，出于好奇，看了一下他的1.x版本，原来也就是基于delegate实现回调.我猜想是不是1.x的开发人员，看到了`MKNetworkKit`这个框架之后，而做出大修改。用了一段时间之后，也决定看一下源码，看了后发现，代码也是挺少的，而且代码分类更细致，代码更易懂。OK，从此之后，基本上都是用`AFNetworking`，虽然没有自动缓存的功能，不过最近发现他的git库已经有关于dis_cache功能开的分支了，哈哈。


####网络请求框架的二次封装

首先，正如上面我说的网络类库淘汰率太高了。可能今天你用的类库，明天又出了一个新的类库，代码更容易使用，更安全，能够符合iOS SDK最新的需求...总之，网络类库我觉得还是不要直接将她的代码，嵌入到我们自己App代码中，否则一旦切换成其他的类库时，可能需要改动的地方就很多了。

再次，一直想着自己封装一下`AFNetworking`，但是一直没有找到好的思路。自己从事iOS开发一年半多，感觉在这些问题上还是有点菜。于是不断的在github找符合自己意思的源码，于是终于发现了一个叫做`YTKNetwork`的代码，对`AFNetworking`进行了二次封装。这样的话，可以通过`YTKNetwork `中的一个类`YTKNetworkAgant`去直接耦合`AFNetworking `，一旦要切换框架时，只需要修改这一个类的代码，而我们App的上层代码基本不需要做任何变动。

####对网络请求整个过程进行抽象

- 对网络请求抽象出实体类 >>> 单独的Request类
- 对网络请求发起的步骤 >>> RequestAgant类
- RequestAgant类 直接耦合 AFNetworking

####网络层框架封装设计时，应尽可能少的使用block，而多使用delegate

- 原因一、block很难追踪，难以维护

```objc
- (void)someFunctionWithBlock:(SomeBlock *)block
{
    ... ...

 -> block();  //当你单步走到这儿的时候，要想知道block里面都做了哪些事情的话，就很麻烦。

    ... ...
}
```

- 原因二、block会延长相关对象的生命周期（经常会出现retain cycle）

> 也就是说核心功能类最好使用deelgate回调，至于上层业务、UI事件等等可以使用block.

####关于网络传输优化

- 直接使用ip地址，减少dns域名解析

- 使用缓存手段减少请求的发起次数（规定几秒内连续网络请求，只使用本地缓存response data数据）

- 每次App启动从服务器拿一份IP列表并缓存，这些IP是所有提供API的服务器的IP
	- 每次应用启动的时候，针对这个列表里的所有IP获取ping延时时间
	- 然后取`延时时间最小`的那个IP作为今后发起请求的IP地址
	- 这个IP地址列表一般是每天第一次启动的时候读一次API，然后更新到本地
	- 可以考虑使用`NSURLProtocol`来达到目的

- 对传输的数据进行压缩，减小传输数据的体积

***

###本地数据存储.

- NSUserDefault，只适合存储一些简单的数据: 标记位等等
- KeyChain，即使删除App，只要iOS系统不重装，数据依然存在
- 磁盘文件，Plist、archive、Stream
- 磁盘SQLite数据库，FMDB、CoreData

以前做的项目，都是使用的sqlite存储，然后使用的`FMDB`类库。但是一直有一个问题，就是一旦responseJSON属性变化，那就导致实体类`Model`添加属性或者修改属性，而此时去读sqlite的时候，就会崩掉。

后来，我搜了一下，最好的办法是做`sqlite数据版本迁移`，但是看了一下代码，觉得还是比较复杂的。然后我尝试开始接触`CoreData`，这个东西提供了一些简单的属性添加、实体类增添、关系变化等等自动修改表结构的功能，觉得还是比较方便了，虽然看很多高手都说性能不行...但是我觉得，对于我现在的这个项目，应该是足够了。

然后了，学习了一下`CoreData`，操作还是有一点麻烦。于是又在github查找了一下，找到了一些关于`CoreData`的封装代码，然后clone下来看了一下实现代码，发现....哇，原来还可以封装成类似J2EE开发中的`ORM`完全面向对象的Api用法.

```
/**
 *	定义了一个CoreData实体类，保存用户的一些不是特别要紧的数据、标识
 */

//标记要走哪一种业务流程（有时候做完其他流程之后，需要回到之前的流程）
typedef NS_ENUM(NSInteger, ZSYBusinessOperation) {
    
    ZSYBusinessOperationNone                                    = 0,
    
    ZSYBusinessOperationLoginPasswordValid                      = 4,
    ZSYBusinessOperationLoginPasswordModify                     = 5,
    ZSYBusinessOperationLoginPasswordForget                     = 6,
    
    ZSYBusinessOperationTradePasswordCreate                     = 7,
    ZSYBusinessOperationTradePasswordModify                     = 8,
    ZSYBusinessOperationTradePasswordForget                     = 9,
    
    //...
}

typedef NS_ENUM(NSInteger, ZSYGesturePasswordOperation) {
    ZSYGesturePasswordOperationNone                     = 0,
    ZSYGesturePasswordOperationCreate                   = 1,
    ZSYGesturePasswordOperationModify                   = 2,
    ZSYGesturePasswordOperationForget                   = 3,
    
    //...
};

typedef NS_ENUM(NSInteger, ZSYEnableGesturePassword) {
    ZSYGesturePasswordDisable = 0,
    ZSYGesturePasswordEnable  = 1,
};

@interface ZSYEntityCustomer : NSManagedObject

//创建一个数据库实例
+ (ZSYEntityCustomer *)createNewInstance;

//获取当前业务流程类型
+ (ZSYBusinessOperation)businessOperationForCurrent:(NSString *)mobile;

//移除业务流程
+ (void)removeBusinessOperationForCurrentByClass:(NSString *)mobile;
- (void)removeBusinessOperationForCurrentByInstance;

//添加业务流程
+ (void)setBusinessOperationForCurrentByClass:(ZSYBusinessOperation)operation Mobile:(NSString *)mobile;
- (void)setBusinessOperationForCurrentByInstance:(ZSYBusinessOperation)operation;

//....

@end
```

还有一些比较常用的使用`TMCache`保存，可以 内存+磁盘缓存.
比较保密的使用的`keychain`保存.

```
/**
 *	将使用`TMCache`和`keychain`代码，统一提供入口
 *
 *	实现中，分情况使用`TMCache`还是`keychain`
 */

@interface ZSYObjectCacheManager : NSObject

+ (void)saveObject:(id)obj forKey:(NSString *)key;
+ (id)getObjectForKey:(NSString *)key;
+ (void)removeObjectForKey:(NSString *)key;

//设置各种标记位
//是否出现
+ (BOOL)isGestureShowed;

//设置已经出现
+ (void)setGestureHadShow;

//设置已经移除
+ (void)setGestureHadentShow;

+ (void)setCookie:(NSString *)cookie;
+ (void)removeCookie;
+ (NSString *)getCookie;

//清除所有保存的数据
+ (void)clearAllDatas;

@end
```



####数据缓存.

方式一、【使用`CoreData`以`sqlite`的形式】保存一部分数据，主要是基于磁盘形式缓存.必须要求实体类继承`NSManagedObject`，且发现实现`NSCoding`协议好像有一些问题.

方法二、【直接缓存responseData】使用`YTKNetwork`可以按照`YTKRequest`指定的`version`或`过期时间`两种方法管理缓存responseData.

方式三、【缓存解析后的实体类对象】找到了一个`kache`的对象缓存框架，欣喜若狂。于是clone下来，运行他的demo，结果不是直接蹦，就是卡在那界面没任何反应...哎，一看最后提交日期是2013年。但是我看了一下他的介绍`提供对象的内存和磁盘缓存，支持哈希，队列和时间池`，看介绍还是觉得掉渣天的感觉，于是还是断点调试了一下哪里有问题，并且学习了一下代码是如何封装的.

####对于一个实体类Model如果包含1:n的关系，那么向外提供`不可变`数组，对内使用`可变`数组，只向外提供`方法api`来操作内部`可变`数组

- 书签类

```objc
#import <Foundation/Foundation.h>

@interface BookMarkModel : NSObject
@property (nonatomic, copy) NSString *text;
@end
```

```objc
#import "BookMarkModel.h"

@implementation BookMarkModel

@end
```

- 书类与书签，1:n的关系

```objc
#import <Foundation/Foundation.h>

// 实体类
#import "BookMarkModel.h"

@interface BookModel : NSObject

@property (nonatomic, copy) NSString *bookName;
@property (nonatomic, assign) NSUInteger bookId;

// 向外提供不可变数组
@property (nonatomic, strong) NSArray *bookmarks;

// 提供操作内部可变数组的api
- (void)addBookmark:(BookMarkModel *)mark;
- (void)removeBookMark:(BookMarkModel *)mark;
- (NSArray *)allBookmarks;

@end
```

```objc
#import "BookModel.h"

@interface BookModel ()

@property (nonatomic, strong) NSMutableArray *mutableBookMarks;

@end

@implementation BookModel

- (instancetype)init
{
    self = [super init];
    if (self) {
        _mutableBookMarks = [NSMutableArray new];
    }
    return self;
}

- (void)addBookmark:(BookMarkModel *)mark {
    [_mutableBookMarks addObject:mark];
}

- (void)removeBookMark:(BookMarkModel *)mark {
    [_mutableBookMarks removeObject:mark];
}

- (NSArray *)allBookmarks {
    return [_mutableBookMarks copy];
}

@end
```

####还可以提供各种各种功能的Model分类，比如持久化到SQLite的Model分类

```objc
#import "BookModel.h"

@interface BookModel (DAO)

// 从SQLite查询所有的BookModel表记录
+ (NSArray *)db_allBooks;

// 将当前BookModel对象保存到数据
- (BOOL)db_saveBook;

// 从SQLite表移除BookMarkModel对象
+ (BOOL)db_removeBook:(BookMarkModel *)book;

@end
```

提供这样等等分类，将操作实体类的各种各种的功能，全部分隔开来，避免写在一个文件中，以后阅读困难。

***

###业务类`ViewModel`负责完成: 网络请求、步骤化逻辑代码、输入校验等UI逻辑..


> 首先，抽象出`每一个实体类`对应的业务方法的协议.

```
@protocol ZSYCustomerLogicProtocol : NSObject


#pragma mark - 一些简单的基本的Api请求

/**
 *  用户注册
 *
 *  @param mobile   手机号
 *  @param password 登录密码
 *  @param code     验证码
 */
- (ZSYRequest *)registAPIWithMobile:(NSString *)mobile
                           Password:(NSString *)password
                               Code:(NSString *)code;

/**
 *  用户登录
 *
 *  @param mobile   手机号
 *  @param password 登录密码
 */
- (ZSYRequest *)loginAPIWithMobile:(NSString *)mobile
                          Password:(NSString *)password;
                          
                          /**
 *  修改手势密码
 *
 *  @param mobile 手机号
 *  @param oldPwd 原手势密码
 *  @param newPwd 新手势密码
 */
- (ZSYRequest *)updateGesturePasswordAPIWithMobile:(NSString *)mobile
                                            OldPwd:(NSString *)oldPwd
                                            NewPwd:(NSString *)newPwd;
                                            
                                            
#pragma mark - 返回一些处理某中情况的业务逻辑代码

- (GesturePasswordBlock)saveGesturePasswordCallback;
- (GesturePasswordBlock)findGesturePasswordCallback;
- (GesturePasswordBlock)forgetGesturePasswordCallback;
- (GesturePasswordBlock)updateGesturePasswordCallback;

#pragma mark - 一些较复杂流程的逻辑代码，并代替执行Api去请求服务器

/**
 *	登录之后，判断是否要执行
 * 
 * 【忘记手势密码】、【创建手势密码】、【未绑卡】、【未设置交易密码】哪一个流程，
 * 
 * 然后提供Block让ViewController执行，让用户输入数据，然后ViewModel获取数据进行下一步
 * 的处理.
 *
 */
- (YTKChainRequest *)loginCustomerLogicWithMobile:(NSString *)mobile
                                         Password:(NSString *)password
                       ForgetGesturePasswordBlock:(void (^)(void))forgetGestureBlock
                       CreateGesturePasswordBlock:(void (^)(void))createGestureBlock
                               UnBandBanCardBlock:(void (^)(void))unBandCard
                            UnSetPayPasswordBlock:(void (^)(void))unSetPayPwd;

@end


#pragma mark - 获取Api回调数据，这样的好处是，不局限在Block内部获取状态

//是否登录
- (BOOL)isCustomerLogin;
- (NSString *)getGesturePassword;
- (NSString *)getLoginResultStr;

//是否注册
- (BOOL)isCustomerRegist;
- (BOOL)isCustomerRegistSuccess;
- (NSString *)getRegistResultStr;

//是否修改密码
- (BOOL)isCustomerModifySuccess;
- (NSString *)getModifyResultStr;
- (BOOL)isCustomerPassowrdLock;
- (BOOL)isCustomerLock;

//是否短信下发
- (BOOL)isSMSCodeSendSuccess;

//是否实名
- (BOOL)isCustomerHasRealName;

//是否有手势密码
- (BOOL)isCustomerHasGesturePassword;

//是否绑卡
- (BOOL)isCustomerHasBindCard;

//是够设置交易密码
- (BOOL)isCustomerHasPayPassword;
```


> 然后写协议实现类.

```
@interface ZSYCustomerViewModelV1 : ZSYBaseViewModel <ZSYCustomerLogicProtocol>

//....

@end
```

```

//列举一个流程处理逻辑，其他简单的代码就补写了.

- (YTKChainRequest *)loginCustomerLogicWithMobile:(NSString *)mobile
                                         Password:(NSString *)password
                       ForgetGesturePasswordBlock:(void (^)(void))forgetGestureBlock
                       CreateGesturePasswordBlock:(void (^)(void))createGestureBlock
                               UnBandBanCardBlock:(void (^)(void))unBandCard
                            UnSetPayPasswordBlock:(void (^)(void))unSetPayPwd
{

	//保存Block
    _forgetGestureBlock = forgetGestureBlock;
    _createGestureBlock = createGestureBlock;
    
    //发起一个链式请求
    YTKChainRequest *chain = [[YTKChainRequest alloc] init];
    
    //创建一个登录请求    
    ZSYCustomerLoginRequest *login = [[ZSYCustomerLoginRequest alloc] initWithLoginName:mobile Password:password];
    
    //发起登录请求
    @weakify(self);
    [chain addRequest:login callback:^(YTKChainRequest *chainRequest, YTKBaseRequest *baseRequest) {
        @strongify(self);
        
        ZSYCustomerLoginRequest *login = (ZSYCustomerLoginRequest *)baseRequest;
        
        //如果登录成功，就判断是否执行前面的哪一种流程
        if ([self handleResponseIsSuccess:login]) {
            
            [self _handleCustomerLoginSuccess:mobile];
            
            //            if ([ZSYObjectCacheManager isForgetGesture]) {
            if ([ZSYEntityCustomer isForgetGesturePasswordOperation:mobile]) {
                
                if (_forgetGestureBlock) {
                    _forgetGestureBlock();
                }
                
                //            } else if ([ZSYObjectCacheManager isModifyGesture]) {
            } else if ([ZSYEntityCustomer isModifyGesturePasswordOperation:mobile]) {
                //do nothing...
            } else {
                [self showHudTipStr:@"登录成功"];
                
                NSString *gesturePwd = [self getGesturePassword];//该账号在服务器，不存在手势密码，必须设置
                if (![gesturePwd  zsy_isValid]) {
                    if (_createGestureBlock) {
                        _createGestureBlock();
                    }
                } else {
                    ZSYEntityCustomer *customer = [ZSYEntityCustomer object:[NSString stringWithFormat:@"mobile = '%@'",mobile]];
                    customer.gesturePwd = gesturePwd;
                    [ZSYEntityCustomer commit];
                }
                
            }
            
            if (self.onSuccess) {
                self.onSuccess();
            }
            
            [self clearCompletionBlocks];
            
        } else {
            if (self.onFail) {
                self.onFail();
            }
            
            [self clearCompletionBlocks];
        }
    }];
    
    //开始链式请求
    [chain start];
    chain.delegate = self;
    return chain;

}

//还有一些关于输入校验...等等基础代码
//.....

```

> 编写所有的ViewModel的父类`BaseViewModel`提供`统一添加Block`、`统一处理请求错误`、`统一处理请求失败`

```
@interface ZSYBaseViewModel : NSObject

@property (copy, nonatomic) void (^onSuccess)(void);
@property (copy, nonatomic) void (^onError)(void);
@property (copy, nonatomic) void (^onFail)(void);
@property (copy, nonatomic) void (^requestFailedBlock)(id response);

@property (strong, nonatomic, readonly) id responseJson;
@property (assign, nonatomic, readonly) ZSYResponseType responseCode;

//处理成功（responseCode == 0，也有同时支持几个code，所以定义了一个数组）
- (BOOL)handleResponseIsSuccess:(YTKBaseRequest *)request;

- (BOOL)handleResponseIsSuccess:(YTKBaseRequest *)request
              ResponseCodeArray:(NSArray *)codes;
              
//处理失败
- (void)handleRequestFail:(YTKBaseRequest *)request;

//处理错误
- (void)handleResponseError:(YTKBaseRequest *)request;

@end

```

```

- (BOOL)handleResponseIsSuccess:(YTKBaseRequest *)request ResponseCodeArray:(NSArray *)codes {

	//1. 
    NSDictionary *json = request.responseJSONObject;
    
    //2. 
    if ([json hasKey:Code]) {
    	
    	//2.1 获取response code
        self.responseCode = [json[Code] integerValue];
        
        BOOL flag = NO;
        
        //2.2 判断response code 是否合法
        for (NSNumber *code in codes) {
            if (code.integerValue == self.responseCode) {
                flag = YES;
                break;
            }
        }
        
        
        if (flag)
        {
            
            //合法
            
            if ([json hasKey:Data]) {
                self.responseJson = json[Data];
            } else {
                self.responseJson = nil;//清空上一次的
            }
            
            return YES;
            
        } else {
        
        		//不合法，自动处理是哪一种不合法情况
            [self handleResponseError:request];
            
            return NO;
        }

    } else {
        DLog(@"%@ 没有 response code !.\n", request);
        return YES;
    }
}
```

```
- (void)handleRequestFail:(YTKBaseRequest *)request {

    DLog(@"Fail Request: %@\n", [[request.requestOperation error] localizedDescription]);
    
	//1. 将当前不合法的response code 保存为 error code
	self.errorCode = request.requestOperation.error.code;    
	
	//2. 判断是哪一种系统网络请求错误
	switch (_errorCode) {

        case NSURLErrorTimedOut: {
			 //...
        }
            break;

        case NSURLErrorCancelled: {
			 //...
        }
            break;
            
        case NSURLErrorCannotFindHost: {
			 //...
        }
            break;
            
        case NSURLErrorCannotConnectToHost: {
			//...
        }
            break;
            
        case NSURLErrorNetworkConnectionLost: {
			//...
        }
            break;
            
        case NSURLErrorResourceUnavailable: {
			//...
        }
            break;
            
        case NSURLErrorNotConnectedToInternet: {
			//...
        }
            break;
            
        case NSURLErrorBadServerResponse: {
			//...
        }
            break;
            
        //还可以继续添加
    }

    //3. 可以根据错误类型发出通知
    //....
    
    //4. 执行fail block
    if (self.onError) {
//        dispatch_async(dispatch_get_main_queue(), ^{
            self.onError();
            [self clearCompletionBlocks];
//        });
    } else {
        [self clearCompletionBlocks];
    }
}
```

```
- (void)handleResponseError:(YTKBaseRequest *)request {

	switch (self.responseCode) {
		
		case ZSYResponseSuccess: {

        }
            break;
            
       case ZSYResponseSystemError: {

        }
            break;
            
        //...
        
        //登录超时
        case ZSYResponseLoginExpiratError: {
            
            //发送通知
            [self instancePostNotify:ZSYCurrentUserLoginStatusNotifyKey
                              Object:nil
                            UserInfo:@{ZSYCurrentUserLoginStatusItemKey:@(ZSYLoginStatusExpirate)}
                          ThreadType:ZSYNotifyThreadTypeMain];
        }
            break;
            
        //....其他就不写了，与后台写代码的要约定好
	}
}
```

好了，基本上`BaseViewModel`就算完成任务了...

> 然后使用`objection`将`协议`与`实现类`绑定.

```
@interface ZSYModuleProvider : NSObject

@property (strong, nonatomic) JSObjectionInjector *injector;

+ (instancetype)module;

@end
```

```
@implementation ZSYModuleProvider

//...略


- (void)initInjector {
	
	//1. 获取当前默认的对象容器
    _injector = [JSObjection defaultInjector];
    
    //2. 如果没有，就创建一个新的
    _injector = _injector ? : [JSObjection createInjector];
    
    //3. 创建每一个module（不同的module管理一组协议与实现类的映射）
    ZSYViewModelModule *viewModelModule = [[ZSYViewModelModule alloc] init];
    //还可以创建其他module...
    
    //4. 将module注册到对象容器
    _injector = [_injector withModules:viewModelModule, nil];
    //_injector = [_injector withModules:viewModelModule, 其他module1, 其他module2, .... 其他modulen];
    
    //5. 设置为默认的对象容器
    [JSObjection setDefaultInjector:_injector];
}

@end
```

> 然后看看管理一组相关的`协议与实现类`的module，这样的好处是:【关于使用的具体类的所有细节，只有module类知道，外界调用者一概不知，尽可能减少耦合】

```
@interface ZSYViewModelModule : JSObjectionModule

//...

@end
```

```

//接口导入
#import "XxxViewModelLogicProtocol.h"
#import "XxxViewModelLogicProtocol.h"
#import "XxxViewModelLogicProtocol.h"
#import "XxxViewModelLogicProtocol.h"


//接口实现类导入
#import "XxxCustomerViewModelV1.h"
#import "XxxCustomerViewModelV2.h"
#import "XxxCustomerViewModelV3.h"

@implementation ZSYViewModelModule

- (instancetype)init {
    self = [super init];
    if (self) {
        [self bindClasses];
        [self bindProtocols];
    }
    return self;
}

//绑定类
- (void)bindClasses {
    
}

//绑定协议
- (void)bindProtocols {
    [self bindClass:[XxxViewModel class] toProtocol:@protocol(XxxViewModelLogicProtocol)];
    //继续绑定其他的 协议:实现类
    
    //还可以分情况绑定实现类的版本
    #if DEBUG
    	[self bindClass:[XxxViewModelV1.0 class] toProtocol:@protocol(XxxViewModelLogicProtocol)];
    #else
    	[self bindClass:[XxxViewModelV2.0 class] toProtocol:@protocol(XxxViewModelLogicProtocol)];
    #endif
}

```

####到此为止基本上关于`数据处理`相关都差不多了，接下来看看UI之类的.

***

###编写全局UIViewController公共的父类`BaseViewController`.

> 使用`Aspects`来hook系统的函数，实现一些动态加入的一些代码（日志、统计..）

```
@property (weak, nonatomic)id<AspectToken> aspectViewWillAppearBefore;
@property (weak, nonatomic)id<AspectToken> aspectViewWillAppearAfter;
@property (weak, nonatomic)id<AspectToken> aspectViewDidAppearBefore;
@property (weak, nonatomic)id<AspectToken> aspectViewDidAppearAfter;
@property (weak, nonatomic)id<AspectToken> aspectViewWillDisAppearBefore;
@property (weak, nonatomic)id<AspectToken> aspectViewWillDisAppearAfter;
```

在`-[BaseViewController init]`中完成hook，拦截到`viewWillAppear:`、`viewDidAppear:`、`viewWillDisappear:`...等等方法.

```
- (void)_swizleMethods {
    @weakify(self);
    
    _aspectViewWillAppearBefore = [self aspect_hookSelector:@selector(viewWillAppear:)
                                                withOptions:AspectPositionBefore
                                                 usingBlock:^(id<AspectInfo> aspectInfo, BOOL animated) {
                                                     @strongify(self);
                                                     [self log];
                                                     [self statistic];
                                                 } error:NULL];
    
//    _aspectViewWillAppearAfter = [self aspect_hookSelector:@selector(viewWillAppear:)
//                                               withOptions:AspectPositionAfter
//                                                usingBlock:^(id<AspectInfo> aspectInfo, BOOL animated) {
//
//                                                } error:NULL];
    
//    _aspectViewDidAppearBefore = [self aspect_hookSelector:@selector(viewDidAppear:)
//                                               withOptions:AspectPositionBefore
//                                                usingBlock:^(id<AspectInfo> aspectInfo) {
//                                                } error:NULL];
    
    
//    _aspectViewDidAppearAfter = [self aspect_hookSelector:@selector(viewDidAppear:)
//                                              withOptions:AspectPositionAfter
//                                               usingBlock:^(id<AspectInfo> aspectInfo) {
////                                                   @strongify(self);
////                                                   [self _configTabBar];
//                                               } error:NULL];
    
//    _aspectViewWillDisAppearBefore = [self aspect_hookSelector:@selector(viewWillDisappear:)
//                                                   withOptions:AspectPositionBefore
//                                                    usingBlock:^(id<AspectInfo> aspectInfo) {
////                                                        @strongify(self);
////                                                        [self _configNavigationBarBackButton];
//                                                    } error:NULL];
    
    _aspectViewWillDisAppearAfter = [self aspect_hookSelector:@selector(viewWillDisappear:)
                                                  withOptions:AspectPositionAfter
                                                   usingBlock:^(id<AspectInfo> aspectInfo) {
                                                       @strongify(self);
                                                       [self.view endEditing:YES];
                                                   } error:NULL];
}

```

最后在`-[UIViewController dealloc]`移除hook.

```
- (void)_undoSwizz {
    [_aspectViewWillAppearBefore remove];
    [_aspectViewDidAppearAfter remove];
    
    [_aspectViewDidAppearBefore remove];
    [_aspectViewDidAppearAfter remove];
    
    [_aspectViewWillDisAppearBefore remove];
    [_aspectViewWillDisAppearAfter remove];
}
```

> 提供一些子类化的ViewController一些基本的代码

```
- (void)dealloc {
    DLog(@"%@ dealloc...\n", NSStringFromClass([self class]));
    [self removeNotifications];
    [self destroySubviews];
    [self destroyVariables];
    [self unbindViewModel];
    [self destroyRequest];
    //    [self _undoSwizz];
}
```

```
- (instancetype)initWithNibName:(NSString *)nibNameOrNil bundle:(NSBundle *)nibBundleOrNil {
    self = [super initWithNibName:nibNameOrNil bundle:nibBundleOrNil];
    if (self) {
        [self _swizleMethods];
        [self createVariables];
        [self addNotifications];
        [self bindViewModel];
    }
    return self;
}
```

在`viewDidLoad `调用`让子类重写的createSubviews方法`，让子类控制器实例化自己的subviews.

```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self.view setBackgroundColor:kZSYVCViewBackgroundColor];
    
    //调用当前子类控制器的创建subveiws的方法
    [self createSubviews];
    
    // iOS7及以后的版本支持，self.view.frame.origin.y会下移64像素至navigationBar下方。
    self.edgesForExtendedLayout = UIRectEdgeNone;
}

```

每一个控制器接收到内存警告时，统一清除内存.

```
- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    
    //1. 
    [self removeNotifications];
    
    //2. 
    [self destroyVariables];
    
    //3. 
    [self destroySubviews];
    
    //4.
    [self destroyRequest];
    
    //5.
    [[NSURLCache sharedURLCache] removeAllCachedResponses];
}
```

> BaseViewController统一关注用户登录状态改变的通知、以及默认处理，或者子类重写对应方法做特殊的处理.

```
- (void)handleUserNotLogin;
- (void)handleUserLogin;
- (void)handleUserLogout;
- (void)handleUserLoginExpirate;
```

在`-[BaseViewController init]`添加通知关注.

```
- (void)addNotifications  {
    //1. currnt Class
    if ([self isNeedObservenNetworkStatusNotify]) {//关注网络状态改变
        [self _addNotificationsForNetwork];
    }
    
    if ([self isneedObservenLoginStatusNotify]) {//关注登录状态改变
        [self _addNotificationsForCustomLogin];
    }
    
    //2. 子类重写
}
```

如上，子类可以重写`isNeedObservenNetworkStatusNotify`和`isneedObservenLoginStatusNotify`方法，来决定关不关注通知.

```
- (void)_addNotificationsForCustomLogin {
    NSNotificationCenter *center = [NSNotificationCenter defaultCenter];
    
    [center addObserver:self selector:@selector(_didReceiveNotidications:)
                   name:ZSYCurrentUserLoginStatusNotifyKey object:nil];
}
```

然后通知回调处理函数

```
- (void)_handleUserLoginStatusChanged:(ZSYLoginStatus)status {
    switch (status) {
        case ZSYLoginStatusNotLogined: {
            [self handleUserNotLogin];
            break;
        }
            
        case ZSYLoginStatusSuccess: {
            [self handleUserLogin];
        }
            break;
            
        case ZSYLogoutStatusSuccess: {
            [self handleUserLogout];
        }
            break;
            
        case ZSYLoginStatusExpirate: {
            [self handleUserLoginExpirate];
        }
            break;
    }
}
```

子类可以重写如下方法，完成自己需要的处理.

```
- (void)handleUserNotLogin {
    // 处理用户没有登录,子类重写
}

- (void)handleUserLogin {
    // 处理用户已经登录,子类重写
}

- (void)handleUserLogout {
    // 处理用户已经退出,子类重写
}

- (void)handleUserLoginExpirate {
    // 处理用户登录超时,子类重写
}
```

> 写一些用于划分功能代码的一些模板抽象函数，用于子类重写，方便维护代码时，到对应的函数就可以知道是做什么的.

```
- (BOOL)isNeedLogin;
- (void)destroyRequest;
- (void)createVariables;
- (void)destroyVariables;
- (void)createSubviews;
- (void)destroySubviews;
- (void)reloadSubviews;
- (void)showLoadingView;
- (void)hideLoadingView;
- (void)bindViewModel;
- (void)unbindViewModel;
- (void)enableViewModel;
- (void)disableViewModel;
- (void)handleRefreshPullDown;
- (void)handleRefreshPullUp;
- (void)endAllRefresh;
- (void)loadData;
- (void)loadDataFromHost;
- (void)loadDataFromLocal;
```

然后BaseViewController只是做了简单的默认实现或空实现.

> 还提供统一的`push viewController`的入口，统一管理控制器的push.

```
- (id)pushWithControllerId:(ZSYViewControllerId)controllerId
                 isAnimate:(BOOL)isAnimate;

- (id)pushWithControllerId:(ZSYViewControllerId)controllerId
                     Title:(NSString *)title
                  Argument:(NSDictionary *)argument
                 isAnimate:(BOOL)isAnimate;
```

```
- (id)pushWithControllerId:(ZSYViewControllerId)controllerId
                     Title:(NSString *)title
                  Argument:(NSDictionary *)argument
                 isAnimate:(BOOL)isAnimate 
{
	//1. 根据传入的id获取对的className字符串
	className = [ZSYBaseViewController viewControllerclassNameById:controllerId];
	
	//2. 获取当前控制器的控制器栈
	ZSYBaseNavController *nav = [self findNavigationControllerByCurrentVC];
	
	//3. 查找要push的目标控制器是否已经在栈中
	UIViewController *findVC = [nav pushToViewController:NSClassFromString(className)
                                                Animated:isAnimate];
                                                
	//4. 如果已经存在，直接popTo
	if (findVC) {
        [ZSYBaseViewController p_fillArgumentForInstance:findVC
                                               arguments:argument
                                                   title:title];
        return findVC;
    }
    
    //5. 如果登录木块使用present
    if (controllerId == ZSYCustomerRegistAndLogin) {
        vcInstance = [self handleWhenControllerIdIsLoginModules:className
                                                       Argument:argument
                                                         Tiltle:title
                                                      isAnimate:isAnimate];
        
        return vcInstance;
    }

	//6. 根据分析得到的className，然后实例化目标控制器对象
	vcInstance = [self createNormalViewControllerInstanceExcludeLoginModolesWithControllerId:controllerId
                                                                                   ClassName:className];
                                                                                   
	//7. 将传入的参数填充给创建的控制器对象
	vcInstance = [ZSYBaseViewController p_fillArgumentForInstance:vcInstance
                                                        arguments:argument
                                                            title:title];
                                                            
	//8. push
	[nav pushViewController:vcInstance animated:isAnimate];
	
	//9. 返回
	return vcInstance;
	
}
```

如上步骤的一些子方法实现就不贴了...

> 统一给所有ViewController添加`loading动画View`

```
- (void)showLoadingView;
- (void)hideLoadingView;
```

```
@property (nonatomic, strong) UIView *defaultLoadingView;
```

```
- (void)showLoadingView {
    
    if (!_defaultLoadingView) {
        
        _defaultLoadingView = [[UIView alloc] init];
        _defaultLoadingView.backgroundColor = [UIColor whiteColor];
        
        //执行一些动画效果
        //.....
        
    }
    
    [self.view bringSubviewToFront:_defaultLoadingView];
}
```

```
- (void)hideLoadingView {
    [self.view sendSubviewToBack:_defaultLoadingView];
}
```

***

###编写全局UITableViewController的公共父类`BaseTableViewController`

```
@interface ZSYBaseTableViewController : ZSYBaseViewController<UITableViewDelegate, UITableViewDataSource>

//提供设置某一个section、某一个row的cell的回调Block
@property (copy, nonatomic) void (^OnConfigCell)(NSIndexPath *indexPath, id data, ZSYBaseCell *cell);

//提供Cell被点击的回调Block
@property (copy, nonatomic) void (^OnItemSelect)(NSIndexPath *indexPath, id data, UITableViewCell *cell);

//统一创建下拉刷新
- (BOOL)isNeedPullDownRefresh;
- (BOOL)isNeedPullUpRefresh;
- (BOOL)isNeedRefreshWhenAppear;

//提供个性化TableView相关显示
- (NSInteger)sectionCount;
- (NSInteger)rowCountForSection:(NSInteger)section;
- (NSArray *)dataSource;
- (Class)cellClass;
- (CGFloat)cellDefaultHeight;
- (NSString *)cellId;

//设置cell出现时候的动画类型
- (ZSYTableViewAnimateType)zsyAnimateType;

//显示当dataSource.count==0时，显示默认背景View
- (void)showNoDataImage:(NSString *)image Message:(NSString *)message;
-(void)removeNoDataImage;

//其他就不列举了
//...

@end
```

```
@implementation ZSYBaseTableViewController

//继承自BaseViewController，会自动执行这个方法
- (void)createSubviews {
    [self setupTableView];
    [self setupTableFooterView];
    
    if ([self isNeedPullDownRefresh]) {
        [self createPullDownRefresh];
    }
    
    if ([self isNeedPullUpRefresh]) {
        [self createPullUpRefresh];
    }
    
    [self registCell];
    [self setupNodatasView];
}

- (void)setupTableView {

	//...创建TableView
}

```

```
默认实现模板方法


- (BOOL)isNeedPullDownRefresh {
    return NO;
}

- (BOOL)isNeedPullUpRefresh {
    return NO;
}

- (BOOL)isNeedRefreshWhenAppear {
    return YES;
}

- (NSInteger)sectionCount {
    return 1;
}

- (NSInteger)rowCountForSection:(NSInteger)section {
    return [[self dataSource] count];
}

- (NSArray *)dataSource {
    return nil;
}

- (Class)cellClass {
    return [ZSYBaseCell class];
}

- (CGFloat)cellDefaultHeight {
    return [[self cellClass] defaultCellSize].height;
}

- (NSString *)cellId {
    return NSStringFromClass([self class]);
}

- (UITableViewStyle)tableViewStyle {
    return UITableViewStylePlain;
}

- (ZSYTableViewAnimateType)zsyAnimateType {
    return ZSYTableViewAnimateTypeNone;
}

- (BOOL)isPerDisplayAnimate {
    return NO;
}


```

TableView datasource and delegate

```
- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView {

	//1. 获取子类TableViewController重写的实例方法，获取section count
    NSInteger sectionCount = [self sectionCount];
    
    //2. 根据list.count显示默认背景View
    if (sectionCount < 1) {
        [self showNoDataImage:[self nodataImage] Message:[self nodataTitle]];
    } else {
        [self removeNoDataImage];
    }
    return sectionCount;
}
```

```
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {

	//1. 
    NSInteger rows = [self rowCountForSection:section];
    
    //2. 
    if (rows < 1) {
        [self showNoDataImage:[self nodataImage] Message:[self nodataTitle]];
    } else {
        [self removeNoDataImage];
    }

    return rows;
}
```

```
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
	
	
    ZSYBaseCell *cell = [tableView dequeueReusableCellWithIdentifier:[self cellId] forIndexPath:indexPath];
    
    cell.tableView = tableView;
    cell.indexPath = indexPath;
    cell.shouldShowLine = YES;
//    cell.hidden = NO;
   
   //获取子类TableViewController重写的实例方法，获取 数据源List
    NSArray *dataList = [self dataSource];
   
    id itemData = nil;
    
    //判断是一维数组 or 二维数组，分情况处理
    if ([self sectionCount] == 1) {
        itemData = dataList[[indexPath row]];
        
        //【让所有cell实现一个协议，来设置当前要显示的实体类对象】
        [cell setupDataItem:itemData];
    } else {
        
        id sectionData = dataList[indexPath.section];
        
        if ([sectionData isKindOfClass:[NSArray class]]) {
            NSArray *sectionArray = (NSArray *)sectionData;
            itemData = sectionArray[indexPath.row];
            
            //【让所有cell实现一个协议，来设置当前要显示的实体类对象】
            [cell setupDataItem:itemData];
        } else {
            itemData = sectionData;
            
            //【让所有cell实现一个协议，来设置当前要显示的实体类对象】
            [cell setupDataItem:itemData];
        }
    }
    
    //执行回调Block，执行子类预先设置Cell的代码
    if (_OnConfigCell) {
        _OnConfigCell(indexPath, itemData, cell);
    }
    
    //
    cell.selectedBackgroundView = [[UIView alloc] initWithFrame:cell.frame];
    cell.selectedBackgroundView.backgroundColor = kZSYWMTableViewCellBGColor;
    return cell;
}
```

```
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
    UITableViewCell *cell = [tableView cellForRowAtIndexPath:indexPath];
    NSArray *dataList = [self dataSource];
    id data = nil;
    if ([self sectionCount] == 1) {
        data = dataList[[indexPath row]];
    } else {
        
        id sectionData = dataList[indexPath.section];
        
        if ([sectionData isKindOfClass:[NSArray class]]) {
            NSArray *sectionArray = (NSArray *)sectionData;
            data = sectionArray[indexPath.row];
            
        } else {
            data = sectionData;
        }
    }
    
    if (_OnItemSelect) {
        _OnItemSelect(indexPath, data, cell);
    }
}
```

如下这个返回cell高度的函数分两种: 1)所有高度一致  2)高度不一样，且使用`AutoLayout`设置的约束 3)子类如果使用frame自己计算高度，那么重写`tableView:heightForRowAtIndexPath:`返回自己计算出来的高度即可

```
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    
    if (![[self cellClass] isDynamic]) {
        return [self cellDefaultHeight];
    }
        
    __weak typeof(self) weakSelf = self;

	//获取得到当前TableViewController显示的Cell给出的默认数据
    CGSize defaultSize = [[self cellClass] defaultCellSize];

	//调用BaseCell的计算约束得到Cell高度的方法
    CGSize cellSize = [[self cellClass] sizeForCellWithDefaultSize:defaultSize
                                                    setupCellBlock:^id(id<ZSYAutoLayoutCellProtocol> cellToSetup)
    {
        NSArray *dataList = [weakSelf dataSource];
        id obj = nil;
        
        if ([self sectionCount] == 1) {
            obj = dataList[[indexPath row]];
            [cellToSetup setupDataItem:obj];
        } else {
            id sectionData = dataList[indexPath.section];
            
            if ([sectionData isKindOfClass:[NSArray class]]) {
                NSArray *sectionArray = (NSArray *)sectionData;
                id obj = sectionArray[indexPath.row];
                [cellToSetup setupDataItem:obj];
            }
        }
        return cellToSetup;
    }];
    
    return cellSize.height;

}
```

如上计算cell高度约束的代码，写在了`BaseCell`中，稍后再说.


总之，其他所有的子类TableViewController基本上不用再谢所有的`TableView datasource and delegate`相关代码了...

那么给出一个子类TableViewController代码..

```
@interface ZSYHomePage_RootViewController : ZSYBaseTableViewController

@end
```

```
@implementation ZSYHomePage_RootViewController


//重写父类方法，让父类执行创建下拉刷新的UI
- (BOOL)isNeedPullDownRefresh {
    return YES;
}

- (BOOL)isNeedRefreshWhenAppear {
    return NO;
}

- (void)createSubviews {
    [super createSubviews];
    
    //1. 现在加当前控制器的subviews
    
    //设置tableView
    [self settingTableView];
    
    //2. 加载完子类所有subviews后，执行此句代码显示加载View
    [self showLoadingView];
    
    //3. 网络请求
    //...
}

//配置父类的TableView，调整位置
- (void)settingTableView {
    
    //设置TableView点击回调、headerView
    [self.tableView setContentInset:UIEdgeInsetsMake(0, 0, kZSYTabBarHeight + 2, 0)];
    
    
    self.tableView.scrollEnabled = YES;
    self.tableView.bounces = YES;
    
    //给父类传入回调Block，处理Cell点击的处理代码
    @weakify(self);
    self.OnItemSelect = ^(NSIndexPath *indexPath, id data, UITableViewCell *cell) {
        @strongify(self);
        
        //跳转详情
        [self handleForwardToProductDetail:data cell:cell];
    };
    
    //创建广告轮播View
    [self createAdvertiseView];
}

//重写提供Cell显示的一些模板方法
- (Class)cellClass {
    return [ZSYHomepageCell2 class];
}

- (NSArray *)dataSource {
    return _dataList;
}

- (NSInteger)rowCountForSection:(NSInteger)section {
    return 1;
}

- (NSInteger)sectionCount {
    return [_dataList count];
}

//也可以直接重写父类的 TableViewDatasource方法或TableViewDelegate方法.
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
	
	//1. 先让父类的方法执行完
    ZSYBaseCell *cell = (ZSYBaseCell *)[super tableView:tableView cellForRowAtIndexPath:indexPath];
    
    //2. 然后对得到的Cell再做不同的处理
    cell.shouldShowLine = NO;
    [cell showBottomLine];
    cell.selectedBackgroundView = [[UIView alloc] initWithFrame:cell.frame];
    cell.selectedBackgroundView.backgroundColor = kZSYWMTableViewCellBGColor;
    return cell;
}

//其他代码就省略了...

@end
```

总而言之，可以看出这个TableViewController中少了很多的关于TableView的基础性代码了，因为都在Base里面.

####如上，可能注意到，并没写关于`如何设置Cell显示对应实体类对象的代码`，哈哈...接着继续讲到.

***

###编写全局Cell的公共父类`BaseCell`

> 首先定义一个协议，所有Cell都必须实现，用于将传入的实体对象，设置到Cell的subViews上.

```
@protocol CellProtocol <NSObject>

@property (nonatomic, strong) id data;

- (void)setupDataItem:(id)data;

@end
```

> BaseCell对 `setupDataItem:` 做一个空实现，让子类去自己设置自己的实体对象。再就是提供根据Cell设置的所有的约束，计算得到cell的高度的工具函数.

```
@interface ZSYBaseCell : UITableViewCell <ZSYAutoLayoutCellProtocol>

//...

@end
```

```
@implementation ZSYBaseCell

- (void)setupDataItem:(id)data {
    //子类重写
}


//如下方法提供计算设置了约束的Cell的高度
+ (CGSize)sizeForCellWithDefaultSize:(CGSize)defaultSize setupCellBlock:(setupCellBlock)block {
    __block ZSYBaseCell *cell = nil;
    
    [cellArray enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
        if ([obj isKindOfClass:[[self class] class]]) {
            cell = obj;
            *stop = YES;
        }
    }];
    
    if (!cell) {
        cell = [[[self class] alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:@"XZHAutoLayoutCellIdentifier"];
        cell.frame = CGRectMake(0, 0, defaultSize.width, defaultSize.height);
        [cellArray addObject:cell];
    }
    
    [cell resetSubviews];
    cell = block((id<ZSYAutoLayoutCellProtocol>) cell);
    
    [cell setNeedsLayout];
    [cell layoutIfNeeded];
    
    CGSize size = [cell.contentView systemLayoutSizeFittingSize:UILayoutFittingCompressedSize];
    size.height += 1.0f;
    
    return size;
}

@end
```

> 编写我们自己的子类Cell

```
@interface HomepageCell : ZSYBaseCell

//...

@end
```

```
@implementation HomepageCell

//实现协议方法，将传入的实体类数据显示到subViews
- (void)setupDataItem:(id)data {
    
    if ([data isKindOfClass:[WMModelProduct class]]) {
        [_subView setImage:[UIImage imageNamed:@"img_home_cell_1"]];
        //首页列表图加载默认（默认加载图片名字）
    }
    
    if ([data isKindOfClass:[ZSYEntityAdItem class]]) {
        ZSYEntityAdItem *item = (ZSYEntityAdItem *)data;
        UIImage *defImage = [UIImage imageNamed:@"首页列表图加载默认"];
        [_subView sd_setImageWithURL:[NSURL URLWithString:item.imageURL] placeholderImage:defImage options:SDWebImageRefreshCached];
    }
}

//其他省略...

@end
```

如上看出，在BaseTableViewController中的代理函数中，自动取出要显示的实体类对象之后，然后调用Cell协议中的方法`setupDataItem:`，让cell完成subveiws赋值.

这些代码，都是在Cell自己的代码中，并没有分散到其他的地方.

****

###App的crash跟踪、log

- Log如果文件保存，使用的开源库`CocoaLumberjack`
- crash跟踪使用的是`bugly`
	- dSYM符号表文件来还原CrashLog日志中的堆栈

****

###App与服务端的通信安全

####一、设计签名

- 服务端需要给客户端一个密钥（为了不让别人也获取到这个密钥，最好直接写死在代码里面）
	- 或者更好的方法是将秘钥放到一个编译出来的`静态库.a`中

- 每次调用服务器API时，使用这个`密钥`再加上`API路径`和`API请求参数`通过指定算法得到一个值，然后请求的时候带上这个值

- 服务端收到请求之后，按照同样的密钥`同样的算法`（服务端与客户端需要协商一致），计算得到一个值，然后比较客户端带过来的算法值是否一致
	- 如果一致，表明是自己的app
	
- 另外适当增加一下求Hash的算法的复杂度，那就是各种Hash算法（比如MD5）加点盐，再回炉跑一次Hash啥的

####二、数据传输安全一、直接使用https通信

那就很方便了，https提供了一个SSL安全数据传输层，已经能够避免数据传输过程中的窃取、监听、篡改，基本上已经很安全了。

####二、数据传输安全二、部分仍然使用http通信

- 比较高的安全做法
	- 首先对原始数据撒盐
	- 然后再对撒要后的数据MD5/SHA...等等算法
	- 然后在乱序
	- 再MD5/SHA来回几次
	- 最后将数据进行http传输

- 特别高的安全做法
	- 使用RSA加密算法
	- 首先使用openssl生成秘钥和公钥
	- 秘钥放服务端，公钥放客户端
	- 可以使用openssl的一个封装库来完成更简单
	- 客户端使用`公钥`解密服务器数据、使用公钥加密发送给服务器的数据
	- 服务端使用`私钥`解密客户端数据、使用私钥加密发送给客户端的数据
	- 一般只对数据json部分进行RSA加密，其余部分json数据不需要

比如，一般json数据格式:

```
{  
    "status": "000000",              ----返回状态，六个0表示成功  
    "desc": null,                    ----返回结果描述，六个0表示成功
    "data" : {
    	//api请求对应的json数据
    }
    
    //....其他key
```

只需要对data数据项进行RSA加解密.

***

###关于类簇的使用 >>> 减小外部对内部众多的类直接依赖

后面已经有文字讲解类簇了，就不写了...

***


	
***

###还有的话就是一些工具类了.

1. UIButton+Factory 分类提供全局统一样式的按钮实例

2. FontManager 提供统一设置UILabel、TextView、等title字体样式，样式采用plist文件配置.

3. LocationManager 定位工具类，采用的开源库`INTULocationManager`.

4. ThirdPartyManager 封装所有第三方SDK相关代码.

5. 然后就是其他各种的工具分类了，太多了就不列举了...

****

大概就这么多了，以后遇到好的设计再加进来...


