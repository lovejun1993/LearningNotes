---
layout: post
title: "App Start Task Queue"
date: 2015-01-06 18:54:04 +0800
comments: true
categories: 
---

###通常App程序启动，我们需要做一些初始化事件:

- 选择最优网络通道
- 加载初始化服务器数据
- 版本更新监测

***

####但是希望有一个Manager类，来管理所有的Task，让每一种不同的业务单独封装到一个Task类中.

***


###那么，可以单独抽一个启动任务管理Manager类

接下来开始抽象出需要的角色.

***

###角色1、什么样的才是启动任务？

```
@protocol AppStartTask <NSObject>

/**
 *  Task的名字
 */
- (NSString *)taskName;

/**
 *  启动Task，回传实现了AppStartTaskDelegate协议的对象
 *  （此方法由AppTaskManager执行）
 */
- (void)startWithDelegate:(id<AppStartTaskDelegate>)delegate;

@end
```

###角色2、启动任务的需要 回传什么数据 或 回调执行什么代码？

```
@protocol AppStartTaskDelegate <NSObject>

/**
 *  告诉delegate Task正常结束
 */
-(void)startupTaskDidCompleted:(id<AppStartTask>)startupTask;

/**
 *  告诉delegate Task错误结束
 */
-(void)startupTaskDidFailed:(id<AppStartTask>)startupTask
                      error:(NSError*)error;

@end
```

###角色3、AppStartTaskManager 管理所有的 启动Task任务.

```
@interface AppStartTaskManager : NSObject <AppStartTaskManagerDelegate>

/**
 *  单例
 */
+ (instancetype)sharedInstance;

/**
 *  开始执行队列中的所有Task
 */
- (void)startAllTask;

/**
 *  添加Task
 */
-(void)addStartupTask:(id<AppStartTask>)task;

/**
 *  移除Task
 */
-(void)removeStartupTask:(id<AppStartTask>)task;

/*!
 *  移除所有Task
 */
-(void)removeAllStartupTask;

@end
```

AppStartTaskManager自己实现AppStartTaskManagerDelegate，自己作为回调对象.

****

###AppDelegate.m中使用TaskManager开始所有的Task

```
- (void)startupApplication {
    
    //弄一个遮盖加载View
    self.window.rootViewController = [VSLoadingViewController defaultView];
    
    //获取AppStartTaskManager单例
    AppStartTaskManager* startupManager = [AppStartTaskManager sharedInstance];
    
    //设置AppStartTaskManager单例的回调对象
    startupManager.delegate = self;
    
    //先将AppStartTaskManager单例的所有task删除
    //注意，remove方法中，将AppStartTaskManager单例对象，作为第一个Task对象
    [startupManager removeAllStartupTask];

	//向AppStartTaskManager单例，添加新的Task
    [startupManager addStartupTask:[Manager1 sharedInstance]];
    [startupManager addStartupTask:[Manager2 sharedInstance]];
    [startupManager addStartupTask:[Manager3 sharedInstance]];
    [startupManager addStartupTask:[Manager4 sharedInstance]];
    [startupManager addStartupTask:[Manager5 sharedInstance]];
    
	//......其他的Manager
	//注意:这些manager实现了task协议的，封装一些业务代码.

	//开始执行所有的Task
    [startupManager startup];
    
    // 启动时执行的附加任务
    //...
}
```

###AppStartTaskManager单例，remove掉所有的Task，并将自己作为第一个进入队列的Task任务.

```
@implementation AppStartTaskManager

- (void)removeAllStartupTask {
    
    dispatch_async(self.serialQueue, ^{
        
        //1. 先删除其他Task
        [self.taskQueue removeAllObjects];
        
        //2. 然后将当前Manager对象作为Task，加入到队列的第一个元素
        //为了执行当前Manager对象执行task开始的方法
        [self.taskQueue addObject:(id<AppStartTask>)self];
    });
}

@end

```

***

###AppStartTaskManager单例，开始执行所有的任务 `id<AppStartTask>`

```
@implementation AppStartTaskManager

- (void)startAllTask {
    
    __weak __typeof(self)weakSelf = self;
    
    dispatch_async(self.serialQueue, ^{
        
        //依次取出数组最前面的Task执行
        id<AppStartTask> task = ( self.taskQueue.count > 0 ) ? self.taskQueue.firstObject : nil;
        
        //如果Task存在就执行
        if (task) {
            
            //打印Task名字
            NSLog(@"task name = %@\n", [task taskName]);
            
            //执行Task，当前AppStartTaskManager是回传当前Manager对象	
            //在其他Manager就回传其他Manager对象
            [task startWithDelegate:weakSelf];
            
            //从TaskQueue移除当前已经执行的Task
            [self.taskQueue removeObjectAtIndex:0];
        }
    });
}

@end
```

***

###上面其实把AppStartTaskManager也当做一个实现了 AppStartTask协议 的Task。那么此时AppStartTaskManager有两个工作:

- 工作一、完成一些业务操作（版本监测），然后回传给AppDelegate对象进行处理.

- 工作二、管理 其他的Task 进行执行.

***

###AppStartTaskManager单例，中对Task添加、Task删除，使用`dispatch_async + 串行队列`完成同步多线程.

- 注意:`全部删除Task情况不同`.

```
@implementation AppStartTaskManager

- (instancetype)init {
    self = [super init];
    if (self) {
        
        //同步多线程执行数组创建
        dispatch_async(self.serialQueue, ^{
            self.taskQueue = [[NSMutableArray alloc] init];
        });
    }
    return self;
}

- (dispatch_queue_t)serialQueue {
    if (!_serialQueue) {
        _serialQueue = dispatch_queue_create("com.cn.UserInfoManager", DISPATCH_QUEUE_SERIAL);
    }
    return _serialQueue;
}

- (void)addStartupTask:(id<AppStartTask>)task {
    
    dispatch_async(self.serialQueue, ^{
        [self.taskQueue addObject:task];
    });
}

- (void)removeStartupTask:(id<AppStartTask>)task {
    
    dispatch_async(self.serialQueue, ^{
        [self.taskQueue removeObject:task];
    });
}


@end
```

***

###AppStartTaskManager单例，是taskQueue中的第一个Task`id<AppStartTask>`，然后被执行.

虽然主要是用来管理所有的Task类对象的执行，但其实也是一个Task `id<AppStartTask>`，那么也可以当做一个Task执行。那么AppStartTaskManager要当做Task，就必须实现`AppStartTask 协议`.

```
/**
 *  1. 当前Manager对象要作为所有Task的delegate
 *     所以实现AppStartTaskDelegate协议方法
 *
 *  2. 当前Manager也是一个Task，所以实现AppStartTask协议
 */
@interface AppStartTaskManager () <AppStartTask, AppStartTaskDelegate>

//临时保存每一个被执行的Task的回调对象delegate
@property (nonatomic, weak) id<AppStartTaskDelegate> taskDelegate;

@end
```

```
@implementation AppStartTaskManager

#pragma mark - AppStartTask 协议实现

- (NSString *)taskName {
    return @"我是应用程序的启动任务";
}

- (void)startWithDelegate:(id<AppStartTaskDelegate>)delegate {//delegate其实就是当前AppStartTaskManager对象
    
    //1. 保存传入的delegate，方便在网络异步回调时使用
    self.taskDelegate = delegate;
    
    //2. 发起网络请求
    //比如做App版本监测...等等
    //然后异步回调通知AppDelegate做出一些处理
    
    //3. （假设为异步回调执行的）网络请求回调完毕时，通知Task成功执行完毕
    //此处的taskDelegate其实就是当前Manager对象
    //通知Manager对象
    if ([self.taskDelegate respondsToSelector:@selector(startupTaskDidCompleted:)])
    {
        //AppTaskManager监测版本任务结束
        //执行Task回调对象
        [self.taskDelegate startupTaskDidCompleted:self];
        
        //清除delegate，因为每次启动Task都会传入delegate
        self.taskDelegate = nil;
    }
    
    //错误执行就不写了...
    
    //4. （假设为异步回调执行的）通知Manager的delegate，Task成功结束执行
    //此处的managerDelegate其实就是AppDelegate单例对象
    //通知AppDelegate单例对象
    if ([self.managerDelegate respondsToSelector:@selector(taskDidCompleted)])
    {
        //告诉Manager对象的代理，有一个Task执行完毕
        [self.managerDelegate taskDidCompleted];
    }
    
    //错误执行就不写了...
}

@end
```

###如上是AppStartTaskManager单例提供了`App的版本监测业务`，所以AppStartTaskManager单例被当做Task执行完之后，需要将监测版本回传给AppDelegate单例进行处理.

- 使用定义protocol + delegate 方式回传AppDelegate数据

```
/**
 *  为了给AppDelegate单例回传Task执行情况
 */
@protocol AppStartTaskManagerDelegate <NSObject>

@optional

/**
 *  告诉AppDelegate单例，Task正常启动完毕
 */
- (void)taskDidCompleted;

/**
 *  告诉AppDelegate单例，Task启动错误结束
 */
- (void)taskManagerDidErrorCompleted:(id<AppStartTask>)task
                               Error:(NSError *)error;

@end
```

- 创建AppStartTaskManager单例时，设置为该代理的实现对象

```
AppDelegate.m

//获取AppStartTaskManager单例
AppStartTaskManager* startupManager = [AppStartTaskManager sharedInstance];
    
//设置AppStartTaskManager单例的回调对象
startupManager.delegate = self;
```

- 那么AppStartTaskManager单例当做Task执行完毕之后，回传给AppDelegate

```
AppStartTaskManager.m

- (void)startWithDelegate:(id<AppStartTaskDelegate>)delegate {
    
    //1. 保存传入的delegate，方便在网络异步回调时使用
    self.taskDelegate = delegate;
    
    //2. 发起网络请求
    //版本监测...
    
    //3. （假设为异步回调执行的）网络请求回调完毕时，通知Task成功执行完毕
    //...    
    //错误执行就不写了...
    
    //----------------------- 如下回调AppDelegate ------------------------
    //4. （假设为异步回调执行的）通知Manager的delegate，Task成功结束执行
    if ([self.managerDelegate respondsToSelector:@selector(taskDidCompleted)])
    {
        //告诉Manager对象的代理，有一个Task执行完毕
        [self.managerDelegate taskDidCompleted];
    }
    
    //错误执行就不写了...
}
```

***

###AppStartTaskManager单例，又是 AppStartTaskManager单例(作为Task) 的回调代理对象。那么就负责Task的回调操作:成功结束、错误结束。

> AppStartTaskManager 实现了 AppStartTaskDelegate协议

```
@implementation AppStartTaskManager 

#pragma mark - AppStartTaskDelegate 协议方法实现

-(void)startupTaskDidCompleted:(id<AppStartTask>)startupTask
{
    //Task成功执行完毕，那么就继续往下执行下一个Task
    [self startAllTask];
}

-(void)startupTaskDidFailed:(id<AppStartTask>)startupTask
                      error:(NSError*)error
{
    //Task执行失败（当前AppStartTaskManager监测版本或其他业务操作失败）
    //需要通知AppDelegate做出处理
    if ([self.managerDelegate respondsToSelector:@selector(taskManagerDidErrorCompleted:Error:)])
    {
        [self.managerDelegate taskManagerDidErrorCompleted:startupTask Error:error];
    }
}


@end
```

***

###AppDelegate.m中实现AppStartTaskManagerDelegate协议，接收AppStartTaskManager单例对象的回调，知道某个Task的执行情况.

```
#pragma mark - AppStartTaskManagerDelegate 协议实现

- (void)taskDidCompleted
{
    //接收到AppStartTaskManager处理版本检测 成功
    //..进行处理
}

- (void)taskManagerDidErrorCompleted:(id<AppStartTask>)task
                               Error:(NSError *)error
{
    //接收到AppStartTaskManager处理版本检测 失败
    //..进行处理
}
```

***

###那么到此AppStartTaskManager被当做Task执行完毕，也完成了通知代理AppDelegate对象，此时继续执行taskQueue中的下一个Task.

***

###将Task抽象、TaskDelegate抽象，从AppStartTaskManager.h中剥离出来，使用一个单独的.h文件描述.


> AppStartTaskProtocol.h


```
#import <Foundation/Foundation.h>

@protocol AppStartTaskDelegate;

/**
 *  Task的抽象
 */
@protocol AppStartTask <NSObject>

/**
 *  Task的名字
 */
- (NSString *)taskName;

/**
 *  启动Task，回传实现了AppStartTaskDelegate协议的对象
 *  （此方法由AppTaskManager执行）
 */
- (void)startWithDelegate:(id<AppStartTaskDelegate>)delegate;

@end

/**
 *  TaskDelegate的抽象
 */
@protocol AppStartTaskDelegate <NSObject>

/**
 *  Task正常结束
 */
-(void)startupTaskDidCompleted:(id<AppStartTask>)startupTask;

/**
 *  Task错误结束
 */
-(void)startupTaskDidFailed:(id<AppStartTask>)startupTask
                      error:(NSError*)error;

@end
```

***

###假设Manager1是作为下一个Task执行一些操作，那么Manager1必须实现 `AppStartTask协议`才能成为Task。

- Manager1.h

```
@interface Manager1 : NSObject<AppStartTask> //实现了AppStartTask协议才能成为Task

//...

@end
```

- Manager1.m

```
@interface Manager1 ()

//临时保存每一个被执行的Task的回调对象delegate
@property(nonatomic, weak) id<AppStartTaskDelegate> taskDelegate;

@end
```

```
@implementation Manager1

#pragma mark - AppStartTask 协议实现，作为一个Task的行为

- (NSString *)taskName {
    return @"Manager1 Task";
}

- (void)startWithDelegate:(id<AppStartTaskDelegate>)delegate {
    
    //1. 保存传入的delegate，方便后面使用
    self.taskDelegate = delegate;
    
    //2. 完成当前Manager作为task要完成的业务操作
    //...
    
    //3. 完成后，异步通知传入的delegate（AppStartTaskManager单例对象）执行结果
    //3.1 假设异步执行成功
    if ([self.taskDelegate respondsToSelector:@selector(startupTaskDidCompleted:)]) {
        [self.taskDelegate startupTaskDidCompleted:self];
    }
    
    //3.1 假设异步执行失败
    if ([self.taskDelegate respondsToSelector:@selector(startupTaskDidFailed:error:)]) {
        [self.taskDelegate startupTaskDidFailed:self error:nil];
    }
    
    //4. 那么最终会回到 AppStartTaskManager单例中，继续往下执行其他的Task
}

@end
```

***

###后续增加每一个启动Task，就添加一个实现`AppStartTask协议`的NSObject子类，然后添加到AppStartTaskManager单例的taskQueue数组。

***

###最后小结Task结构图

![](http://i4.tietuku.com/c8a58fa67e1fa771.jpg)