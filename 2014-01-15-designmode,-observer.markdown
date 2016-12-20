---
layout: post
title: "DesignMode、Observer"
date: 2014-01-15 15:42:40 +0800
comments: true
categories: 
---

###观察者、某个对象的值改变时，通知所有注册过的对象做出处理，类似`响应式`.

***

###假设某个页面获取了钱包的最新钱数量，那么返回到其他页面同样也需要保持这个最新钱数量，要不然就造成钱数量不统一。

***

###那么在多个页面同步最新网络数据办法:

- 每个页面都发一次请求，获取最新数据

- 一个页面触发请求，获取最新数据，然后`通知`其他页面自动更新为最新状态数据

####显然第二种方式更好，可以让其他页面自动更新为最新数据 -- 响应式

***

###下面举一个简单的例子，某个页面触发请求网络最新数据之后，立刻通知其他监听者（页面、业务对象）跟着更新数据.

***

###.h 文件

```
#import <Foundation/Foundation.h>

@class UserInfoManager;

/**
 *  定义协议，让其他对象实现，回传最新数据
 */
@protocol UserInfoManagerProtocol <NSObject>

- (void)userInfoDidChanged:(UserInfoManager *)manager;

@end

@interface UserInfoManager : NSObject


/**
 *  回传的参数值: 待支付数量
 */
@property(nonatomic, readonly) NSInteger repayNumber;

/**
 *  回传的参数值: 待收货数量
 */
@property(nonatomic, readonly) NSInteger recieveNumber;

/**
 *  获取Manager单例
 */
+ (instancetype)sharedInstance;

/**
 *  同步服务器上的最新数据
 */
- (void)synchronServerData;

/**
 *  添加实现了UserInfoManagerProtocol协议的监听者
 *  可以添加多个监听者，使用数组保存，并且使用同步多线程
 */
- (void)addObserver:(id<UserInfoManagerProtocol>) observer;

/**
 *  移除监听者
 */
- (void)removeObserver:(id<UserInfoManagerProtocol>) observer;

@end
```

###.m 文件

```
#import "UserInfoManager.h"

@interface UserInfoManager ()

/**
 *  保存所有的监听者数组
 */
@property (nonatomic, strong) NSMutableArray *observerArray;

/**
 *  使用一个串行队列执行多线程
 */
@property (nonatomic, strong) dispatch_queue_t serialQueue;

@end

@implementation UserInfoManager

- (void)dealloc {
    
    //1. 释放对象
    _serialQueue = nil;
    
    //2. 取消网络请求
    //...
    
}

+ (instancetype)sharedInstance {
    static UserInfoManager *manager = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        manager = [[self alloc] init];
    });
    return manager;
}

- (NSMutableArray *)observerArray {
    if (!_observerArray) {
        _observerArray = [[NSMutableArray alloc] init];
    }
    return _observerArray;
}

- (dispatch_queue_t)serialQueue {
    if (!_serialQueue) {
        _serialQueue = dispatch_queue_create("com.cn.UserInfoManager", DISPATCH_QUEUE_SERIAL);
    }
    return _serialQueue;
}

- (void)synchronServerData {
    
    //1. 请求服务器最新数据
    //...
    
    //2. 然后通知所有监听者，更新最新数据
    [self _notifyAllObservers];
}

- (void)addObserver:(id<UserInfoManagerProtocol>)observer {
    
    //异步执行+串行队列调度，同步多线程
    dispatch_async(self.serialQueue, ^{
        [self.observerArray addObject:observer];
    });
}

- (void)removeObserver:(id<UserInfoManagerProtocol>)observer {
    
    //异步执行+串行队列调度，同步多线程
    dispatch_async(self.serialQueue, ^{
        [self.observerArray removeObject:observer];
    });
}

- (void)_notifyAllObservers {
    
    //让所有监听者执行实现的代理函数，获取最新数据
    
    [self.observerArray makeObjectsPerformSelector:@selector(userInfoDidChanged:)
                                        withObject:self];
}

@end
```

###测试，假设在某个ViewController中

- 要想成为监听者，必须实现协议方法

```
#import "UserInfoManager.h"

@interface ViewController () <UserInfoManagerProtocol>

//...

@end
```

```
@implementation ViewController

//4.
- (void)dealloc {

    [[UserInfoManager sharedInstance] removeObserver:self];
}

- (void)viewDidLoad {
    [super viewDidLoad];

    //1. 添加监听者
    [[UserInfoManager sharedInstance] addObserver:self];
    
    //2. 开始同步数据
    [[UserInfoManager sharedInstance] synchronServerData];
    
}

//3.
- (void)userInfoDidChanged:(UserInfoManager *)manager {
    
    //回调方法中，读取UserInfoManager实例的最新数据
    
    NSLog(@"待支付数量 = %ld", manager.repayNumber);
    NSLog(@"待收货数量 = %ld", manager.recieveNumber);
    
    //还可以更新UI上的数据
    //...
}

@end
```

