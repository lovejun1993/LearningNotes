---
layout: post
title: "自定义NSOperation子类"
date: 2015-09-23 11:44:56 +0800
comments: true
categories: 
---

###昨天接到了蘑菇街的面试，在关于NSOperation的问题，觉得自己回答的不是太好，决定仔细看看.

###那么，就从自己写一个自定义电NSOperation开始，才知道一些具体细节.

***

###No1. 首先给出使用自己定义的operation的示例代码.

```
//1. 
_operationQueue = [[NSOperationQueue alloc] init];
    
//2. 
_myOperation = [[MyOperation alloc] init];
    
//3. 
[_myOperation setCompletionBlock:^{
    while (1) {
        NSLog(@"my operation is run ....\n");
    }
}];
    
//4. 
[_operationQueue addOperation:_myOperation];
    
//5. 
[self performSelector:@selector(cancelOperation) withObject:nil afterDelay:3.f];
```

***

###No2. 开始写`MyOperation.h`文件，向外暴露的方法与属性.

```
//定义operation的所有可能的状态
typedef NS_ENUM(NSInteger, MyOperationStatus) {
    MyOperationStatusReady          = 1, //准备就绪
    MyOperationStatusExcuting       = 2, //正在执行
    MyOperationStatusPaused         = 3, //已经暂停
    MyOperationStatusFinished       = 4, //已经结束
};

@interface MyOperation : NSOperation

//
@property (nonatomic, strong, readonly) NSSet *runLoopModes;

@end

```

OK，先让自定义的NSOperation能够跑起来把，就只先加这么多属性把...

***

###No3. 开始写`MyOperation.m` 具体实现.

主要是如下几点:

1. NSOperation实例的`状态改变`时，`需要手动发出KVO通知`，否则NSOperation不会起作用.
2. NSOperation具体工作函数（start、cancel、finish ....）
3. NSOperation执行一些任务代码
4. 完成后进行回调

***

###No4. 提供了两个c函数，分别获取NSOperation状态的工具函数.

第一个c函数，将`MyOperationStatus枚举值`转换成对应的`iOS系统认识的NSOperation KVO通知key`

```
/**
 *  根据status枚举值获取对应状态的字符串
 *
 *   这四个状态字符串 isReady、isExecuting、isPaused、isFinished 必须写对
 *
 */
static inline NSString * AFKeyPathFromOperationState(MyOperationStatus status) {
    
    switch (status) {
        case MyOperationStatusReady:
            return @"isReady";
            
        case MyOperationStatusExcuting:
            return @"isExecuting";
            
        case MyOperationStatusPaused:
            return @"isPaused";
            
        case MyOperationStatusFinished:
            return @"isFinished";
            
        default: {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wunreachable-code"
            return @"state";
#pragma clang diagnostic pop
        }
    }
}
```

第二个c函数，判断当前要修改NSOperation实例状态是否合法

```
/**
 *  从一个状态改变到另一个状态，是否合法
 */
static inline BOOL AFStateTransitionIsValid(MyOperationStatus fromStatus, MyOperationStatus toStatus, BOOL isCancelled) {
    
    switch (fromStatus) {
            
        //1. 当前状态为ready
        case MyOperationStatusReady:
            switch (toStatus) {
                case MyOperationStatusPaused:
                case MyOperationStatusExcuting:
                    return YES;
                case MyOperationStatusFinished:
                    return isCancelled;
                default:
                    return NO;
            }
            
        //2. 当前状态为excuting
        case MyOperationStatusExcuting:
            switch (toStatus) {
                case MyOperationStatusPaused:
                case MyOperationStatusFinished:
                    return YES;
                default:
                    return NO;
            }
            
        //3. 当前状态为finished
        case MyOperationStatusFinished:
            return NO;
       
        //4. 当前状态为paused
        case MyOperationStatusPaused:
            return toStatus == MyOperationStatusReady;
            
        //5. 默认情况
        default: {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wunreachable-code"
            switch (toStatus) {
                case MyOperationStatusPaused:
                case MyOperationStatusReady:
                case MyOperationStatusExcuting:
                case MyOperationStatusFinished:
                    return YES;
                default:
                    return NO;
            }
        }
#pragma clang diagnostic pop
    }
}
```

***

###No6. `MyOperation.m` 匿名分类添加扩展属性与方法.

```
@interface MyOperation ()

//递归锁，防止多线程危险
@property (readwrite, nonatomic, strong) NSRecursiveLock *lock;

//保存当前operation的状态
@property (readwrite, nonatomic, assign) MyOperationStatus status;

/**
 * 执行要在operation执行的任务代码
 */
- (void)operationDidStart;

/**
 * 取消执行operation
 */
- (void)cancelOperation;


/**
 * 结束执行operation的
 */
- (void)finish;

/**
 *	执行operatin暂停时需要执行的代码
 */
- (void)operationDidPause;

@end
```

***

###No6. `MyOperation`实现@implementation部分代码: 写一个类方法，创建一个单子子线程，开启runloop.


```
//线程的入口函数，开启线程的runloop，并注册线程通信的端口
+ (void)networkRequestThreadEntryPoint:(id)__unused object {
    @autoreleasepool {
    	
    	//1. 
        [[NSThread currentThread] setName:@"MyOperationThread"];
        
        //2. 
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
    }
}

//创建线程
+ (NSThread *)networkRequestThread {

    static NSThread *_networkRequestThread = nil;
    static dispatch_once_t oncePredicate;

    dispatch_once(&oncePredicate, ^{
    
    	//创建线程
        _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
        
        //启动线程
        [_networkRequestThread start];
    });
    
    return _networkRequestThread;
}

```

***

###7. `MyOperation`实现@implementation部分代码: `-[MyOperation init]`


```
- (instancetype)init {
    self = [super init];
    if (self) {
        
        //初始化时，设置ready状态
        _status = MyOperationStatusReady;
        
        //初始化锁
        _lock = [[NSRecursiveLock alloc] init];
        _lock.name = @"MyOperatioLock";
        
        //设置runloop的modes
        self.runLoopModes = [NSSet setWithObject:NSRunLoopCommonModes];
        
    }
    return self;
}
```

***

###8. `MyOperation`实现@implementation部分代码: 获取当前operation状态，都是`重写NSOperation实例方法`.


```
- (BOOL)isReady {
    return self.status == MyOperationStatusReady && [super isReady];
}

- (BOOL)isExecuting {
    return self.status == MyOperationStatusExcuting;
}

- (BOOL)isFinished {
    return self.status == MyOperationStatusFinished;
}

- (BOOL)isConcurrent {
    return YES;//默认是并发
}
```

***

###9. `MyOperation`实现@implementation部分代码: operation被调度时由系统执行的`start`函数，`重写-[NSOperation start]`完成自己的开始执行oepration.

```
- (void)start {
    [self.lock lock];
    
    //1. 如下判断operation是否取消，只能是在operation没有被调度之前，就执行cancel.
    //2. 如下【执行operation】和【取消operation】函数都是在【另一个单例子线程上】执行，由runloop来激活线程.
    
    if ([self isCancelled]) {
       
        [self performSelector:@selector(cancelOperation)
                     onThread:[[self class] networkRequestThread]
                   withObject:nil
                waitUntilDone:NO
                        modes:[self.runLoopModes allObjects]];
        
    } else {
        
        //1.
        self.status = MyOperationStatusExcuting;
        
        //2.
        [self performSelector:@selector(operationDidStart)
                     onThread:[[self class] networkRequestThread]
                   withObject:nil
                waitUntilDone:NO
                        modes:[self.runLoopModes allObjects]];
    }
    
    [self.lock unlock];
}
```

***

###10. `MyOperation`实现@implementation部分代码: 执行用户传入的Block任务代码，执行完任务执行`-[MyOperation finish]`修改status并发出KVO通知，让系统结束执行operation.

```
- (void)operationDidStart {
    [self.lock lock];
    
    //1. 在当先子线程的runloop发送事件源，执行耗时任务代码
    //....
    
    //2. 【重要】执行完任务后，让operation结束执行
    [self finish];
    
    [self.lock unlock];
}
```

***

###11. `MyOperation`实现@implementation部分代码: 添加结束执行operation的方法

```
- (void)finish {
    
    //1. 修改operation状态
    [self.lock lock];
    self.status = MyOperationStatusFinished;
    [self.lock unlock];
    
    //2. 发送通知，operation已经结束执行
    //...
}
```

***

###12. `MyOperation`实现@implementation部分代码: 重写`@property (readwrite, nonatomic, assign) MyOperationStatus status;`生成的setter方法，第一是`修改operation的status`，第二是手动`发出对应状态修改的KVO通知`，实现一个operation的状态机.

```
- (void)setStatus:(MyOperationStatus)status {
    
    //检查从当前status到目标status的变化，是否合法
    if (!AFStateTransitionIsValid(self.status, status, [self isCancelled])) {
        return;
    }
    
    [self.lock lock];
    
    //获取status枚举对应的字符串值
    NSString *oldStateKey = AFKeyPathFromOperationState(self.status);
    NSString *newStateKey = AFKeyPathFromOperationState(status);
    
    //手动发出KVO属性修改通知
    [self willChangeValueForKey:newStateKey];
    [self willChangeValueForKey:oldStateKey];
    _status = status;
    [self didChangeValueForKey:oldStateKey];
    [self didChangeValueForKey:newStateKey];
    
    [self.lock unlock];
}
```

注意:

不管是【开始执行operation】、【取消执行operation】、【暂停operation】、【执行完毕operation】，
所有的状态改变，必须手动发出KVO通知，否则系统不会执行`NSOperation实例对应的函数`。

***

###No13. `MyOperation`实现@implementation部分代码: 执行operation的comppletionBlock。

```
/**
 *	重写设置nsoperation的回调代码方法
 */
- (void)setCompletionBlock:(void (^)(void))completionBlock {
    
    [self.lock lock];
    
    if (!completionBlock) {
        [super setCompletionBlock:nil];
    } else {
        __weak __typeof(self)weakSelf = self;
        [super setCompletionBlock:^ {
            __strong __typeof(weakSelf)strongSelf = weakSelf;
            
            //判断当operation取消后，不执行传入的任务代码
            if (strongSelf.isCancelled == NO) {
                dispatch_async(dispatch_get_main_queue(), completionBlock);
            }
        }];
    }
    
    [self.lock unlock];
}
```

注意:

1. 这个`completionBlock`只有手动发出了`isFinish修改的KVO通知`之后，才会由系统执行.

2. 未解决的问题: 如上设置的completionBlock，在cancel掉operation之后，还是在执行.

***

###No14. `MyOperation`实现@implementation部分代码: 重写 `-[NSOperation cancel]`方法完成取消执行operation.

```
- (void)cancel {
    [self.lock lock];
    
    if (!self.isFinished && !self.isCancelled) {
        [super cancel];
        
        [self performSelector:@selector(cancelOperation)
                     onThread:[[self class] networkRequestThread]
                   withObject:nil
                waitUntilDone:NO
                        modes:[self.runLoopModes allObjects]];
    }
    
    [self.lock unlock];
}
```

```
- (void)cancelOperation {
    
    if (![self isFinished]) {
        
        //1. 做一些清除对象
        //比如: 取消NSURLConnection对象执行
        
        //2. 结束执行operation（应该写在NSURLConnectionDelegate回调函数中）
        [self finish];
    }
}
```

OK，到此为止，最早列出的使用`MyOperation`的代码，可以执行了也可以看到输出.

但是还有很多问题:

1. 取消执行operaiton
2. 暂停执行operation

有时间再来看吧...

***


###No15. 暂停执行operation

> 注意: 需要在operation上执行一些长时间耗时的任务，要不然operaiton就很快就结束了.

```
- (void)pause;
- (BOOL)isPaused;
- (void)resume;
```

```
- (BOOL)isPaused {
    return self.status == MyOperationStatusPaused;
}
```

```
- (void)pause {
    
    //暂停条件: 不能已经暂停 & 不能已经结束 & 不能已经取消
    if ([self isPaused] || [self isFinished] || [self isCancelled]) {
        return;
    }

    [self.lock lock];
    
    //如果正在执行ing
    if ([self isExecuting]) {
        
        //1. 执行暂停执行任务代码
        [self performSelector:@selector(operationDidPause)
                      onThread:[[self class] networkRequestThread]
                    withObject:nil
                 waitUntilDone:NO
                         modes:[self.runLoopModes allObjects]];
        
        //2. 发送通知，operation已经暂停
        //.....
    }
    
    //修改状态，发出KVO通知
    self.status = MyOperationStatusPaused;
    
    [self.lock unlock];
}
```

```
- (void)operationDidPause {
    [self.lock lock];
    
    //取消执行建立网络连接..等等
    //[self.conection cancel];
    NSLog(@"operation暂停执行..., 线程: %@\n", [NSThread currentThread]);
    
    //结束执行（应该写在NSURLConnectionDelegate回调函数中）
    [self finish];
    
    [self.lock unlock];
}
```

实际上pause暂停就是直接让operation结束执行，发出finish的KVO通知.

***

###No16. 恢复执行`resume`operation.

```
- (void)resume {
    
    //只有被paused的operation才可以恢复
    if(![self isPaused]) {
        return;
    }
    
    [self.lock lock];
    
    //1.
    self.status = MyOperationStatusReady;
    
    //2.
    [self start];
    
    [self.lock unlock];
}
```

实际上resume恢复operaiton操作，就是直接让operation直行start开始方法.

***

###No17. 实现断网`将operation归档到本地文件`，恢复网络从本地文件恢复.

1. NSOperation实现协议 `NSSecureCoding, NSCopying`
2. 判断网络状态
3. 归档或解档

***

###小结AFNetworking iOS7以下时进行网络通信的原理

- 第一个阶段，每一个独立的网络请求

	- 网络请求1 >>> 在一个NSOperation子线程执行
	- 网络请求2 >>> 在一个NSOperation子线程执行
	- 网络请求3 >>> 在一个NSOperation子线程执行
	- 网络请求4 >>> 在一个NSOperation子线程执行
	- ....

- 然后所有的operation最终通过一个单例NSThread上进行网络通信
	- 创建一个单例的`AFNetworking Thread`对象
	- 打开Thread对象的Runloop来接收事件源，唤醒Thread线程处理
	- 创建AutoReleasePool对象，管理对象释放
	- 最后让NSURLConnection连接、调度、delegate回调，都放在单例的`AFNetworking Thread`对象上执行

- 当网络操作完成时，单例的`AFNetworking Thread`对象的runloop接收到系统线程的事件源source，告知网络数据交换完毕

- 单例的`AFNetworking Thread`对象被runloop唤醒，并回调operation对象

- 最终operation对象又回调给用户线程进行后续处理

****

###NSOperationQueue的suspend暂停的示例代码

```objc
@interface ViewController ()

@property (strong, nonatomic) NSOperationQueue *queue;

@end

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self testQueueSuspendAndResume];
}

- (NSOperationQueue *)queue {
    if (!_queue) {
        
        //默认创建出来的队列，没有并发数限制的
        _queue = [[NSOperationQueue alloc] init];
        
        //设置最大并发数
        _queue.maxConcurrentOperationCount = 4;
    }
    return _queue;
}

- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    
    int i = 0;
    while (i < 10000) {
        [self.queue addOperationWithBlock:^{
            NSLog(@"%@", [NSThread currentThread]);
        }];
        i++;
    }

}

- (void)testQueueSuspendAndResume {
    
    if (self.queue.isSuspended) {
        //恢复
        [self.queue setSuspended:NO];
    } else {
        //暂停
        [self.queue setSuspended:YES];
    }
}

@end
```

###NSOperationQueue为`串行队列`时被suspend时，仍然会将当前正在执行中的operation执行完毕，再将后续正准备调度执行的operation暂停.

```objc
@interface ViewController ()

@property (strong, nonatomic) NSOperationQueue *queue;

@end

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self testQueueSuspendAndResume];
}

- (NSOperationQueue *)queue {
    if (!_queue) {
        
        //默认创建出来的队列，没有并发数限制的
        _queue = [[NSOperationQueue alloc] init];
        
        //设置最大并发数
        _queue.maxConcurrentOperationCount = 4;
    }
    return _queue;
}

- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];

//    [self queueData1];
    [self queueData2];
}

- (void)queueData2 {
    
    //修改为串行队列
    self.queue.maxConcurrentOperationCount = 1;
    
    [self.queue addOperationWithBlock:^{
        for (NSInteger i = 0; i < 10000; i++) {
            NSLog(@"Thread 1: %ld", i);
        }
    }];
    
    [self.queue addOperationWithBlock:^{
        for (NSInteger i = 0; i < 10000; i++) {
            NSLog(@"Thread 2: %ld", i);
        }
    }];
    
    [self.queue addOperationWithBlock:^{
        for (NSInteger i = 0; i < 10000; i++) {
            NSLog(@"Thread 3: %ld", i);
        }
    }];
}

- (void)testQueueSuspendAndResume {
    
    if (self.queue.isSuspended) {
        //恢复
        [self.queue setSuspended:NO];
    } else {
        //暂停
        [self.queue setSuspended:YES];
    }
}

@end
```

程序运行起来立刻点击屏幕，那么会看到关于`Thread 1`一直输出到9999，而看不到`Thread 2`与`Thread 3`相关的输出，即使队列被suspend，但是仍然把`Thread 1`相关的任务代码全部执行完毕，再将队列中后续的任务挂起.

***

###Operation的cancel取消，一个operation即使被cancel，但是并不会立刻被系统回收释放掉，所以该operation内部的任务代码仍然会被继续执行，所以需要在内部任务中，需要`时时监测operation是否已经被cancel掉`.

```objc
@interface ViewController ()

@property (strong, nonatomic) NSOperationQueue *queue;
@property (strong, nonatomic) NSOperation *op;

@end

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self testOperationCancel];
}

- (void)invocationBlockCallback {
    
    //耗时任务1
    for (NSInteger i = 0; i < 10000; i++) {
        //判断op是否已经被取消
        if ([_op isCancelled] != YES) {
            NSLog(@"Operation did execute: %ld", i);
        }
    }
    
    //耗时任务2
    for (NSInteger i = 0; i < 10000; i++) {
        //判断op是否已经被取消
        if ([_op isCancelled] != YES) {
            NSLog(@"Operation did execute: %ld", i);
        }
    }
    
    //耗时任务3
    for (NSInteger i = 0; i < 10000; i++) {
        //判断op是否已经被取消
        if ([_op isCancelled] != YES) {
            NSLog(@"Operation did execute: %ld", i);
        }
    }
}

- (void)testOperationCancel {
    
    //1.
    _op = [[NSInvocationOperation alloc] initWithTarget:self
                                               selector:@selector(invocationBlockCallback)
                                                 object:nil];
    //2.
    [self.queue addOperation:_op];
    
    //3. 5秒后取消operaiton执行
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(5 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        [_op cancel];
    });
}

@end
```

在所有的block任务代码块中，包含监测`if(![op isCancel]) { do some work ...;}`这样的判断代码，来响应operation被执行cancel.

####但是上述在每一此循环中都会判断operation是否已经被取消，这样还是不太好，`苹果建议执行完一段耗时任务之后再去判断operation的取消状态`

将上去的block任务回调函数实现修改如下

```objc
- (void)invocationBlockCallback {
    
    //耗时任务1
    for (NSInteger i = 0; i < 10000; i++) {
        NSLog(@"Operation did execute: %ld", i);
    }
    
    if ([_op isCancelled]) return;//判断是否已经被取消
    
    //耗时任务2
    for (NSInteger i = 0; i < 10000; i++) {
        NSLog(@"Operation did execute: %ld", i);
    }

    if ([_op isCancelled]) return;//判断是否已经被取消
    
    //耗时任务3
    for (NSInteger i = 0; i < 10000; i++) {
        NSLog(@"Operation did execute: %ld", i);
    }

    if ([_op isCancelled]) return;//判断是否已经被取消
}
```

即执行完一段耗时任务之后，再去判断operation是否已经被取消.

***

