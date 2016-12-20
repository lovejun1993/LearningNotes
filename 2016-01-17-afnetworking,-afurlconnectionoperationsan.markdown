---
layout: post
title: "AFNetworking、AFURLConnectionOperation三"
date: 2016-01-17 21:34:38 +0800
comments: true
categories: 
---

###AFURLConnectionOperation实例的生命周期方法

- start
- cancel
- pause
- resume
- finish

***

###当AFURLConnectionOperation对象被所在NSOperationQueue对象调度执行时，其start方法被执行

```objc
@implementation AFURLConnectionOperation

- (void)start {
    [self.lock lock];
    
    if ([self isCancelled]) {//取消operation
    
    	// 在单例子线程上执行 cancelConnection
#         [self performSelector:@selector(cancelConnection) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
        
    } else if ([self isReady]) {//开始调度operation
    
    	// 修改operatin状态为 executing
        self.state = AFOperationExecutingState;

		// 在单例子线程上执行 operationDidStart
		// 并指定runloop mode 为 default
        [self performSelector:@selector(operationDidStart) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
    }
    
    [self.lock unlock];
}

@end
```

如上的`cancelConnection 方法`和`operationDidStart 方法`都是在全局单例子线程上执行.

***

###AFURLConnectionOperation中使用了一个全局单例子线程NSThread对象

```objc
@implementation AFURLConnectionOperation

//线程创建
+ (NSThread *)networkRequestThread {
    static NSThread *_networkRequestThread = nil;
    static dispatch_once_t oncePredicate;
    dispatch_once(&oncePredicate, ^{
        _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
        [_networkRequestThread start];
    });

    return _networkRequestThread;
}

//线程入口
+ (void)networkRequestThreadEntryPoint:(id)__unused object {

	//第一件事、创建自动释放池autorelease pool
    @autoreleasepool {
    
    	//第二件事、设置子线程名
        [[NSThread currentThread] setName:@"AFNetworking"];

		//第三件事、打开子线程的runloop，并添加一个port让runloop不会退出
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
    }
}

@end
```

###AFURLConnectionOperation 的 `operationDidStart 实例方法` 最终在单例子线程上执行 `NSURLConnection操作`

```objc
@implementation AFURLConnectionOperation

- (void)operationDidStart {

	//1. 
    [self.lock lock];
    
    //2. 首先判断是否已经被取消执行
    //（这里只能在operation执行前取消，运行中进行取消的，需要判断 isCanceled在执行block代码之前。）
    if (![self isCancelled]) {
    
    	//2.1 创建NSURLConnection对象，准备开始网络连接
    	//指定回调delegate为 `当前operation对象`
        self.connection = [[NSURLConnection alloc] initWithRequest:self.request delegate:self startImmediately:NO];
	
		//2.2 获取当前所在线程（单例子线程）的runloop
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        
        //2.3 让NSURLConnection对象、NSOutputStream对象的事件源，在单例子线程的runloop进行接收
        for (NSString *runLoopMode in self.runLoopModes) {
        
        	//给runloop注册事件源时，需要指定一个 `mode`
            [self.connection scheduleInRunLoop:runLoop forMode:runLoopMode];
            [self.outputStream scheduleInRunLoop:runLoop forMode:runLoopMode];
        }

		 //2.4 打开输出流
		 //（将接收到的网络数据，写入到手机内存中）
        [self.outputStream open];
        
        //2.5 在单例子线程上 `同步` 执行connection
        [self.connection start];
    }
    
    //3. 解锁同步
    [self.lock unlock];

	//4. 并且发送一个开始网络请求的通知，告知已经开始
	//是在 `主线程` 上发送的
    dispatch_async(dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingOperationDidStartNotification object:self];
    });
}

@end
```

我log了一下这个方法所在的线程:

```
<NSThread: 0x7fa2e0562020>{number = 3, name = AFNetworking}
```

可以看到，确实是在AFNetworking创建的 `单例子线程` 上执行网络请求操作的.

然后又在`NSURLConnectionDelegate`回调函数，中log了一下，发现也是在单例子线程上执行的.

```
<NSThread: 0x7fa2e0562020>{number = 3, name = AFNetworking}
```

得出的结论:

- 所有网络请求的过程，都是放在一个 `单独的子线程` 上完成的.
- 对我们 主线程、NSOperation线程、没有任何影响.



###如上最终使用的`-[NSURLConnection start]`执行同步网络请求，不会引起App性能问题吗？

> 不会。

- 首先每一个网络请求，是在不同的NSOperation对象上执行。也就是说在一个子线程上执行，完全不会影响主线程UI。

- 最终所有NSOperation的 `NSURLConnection操作` 都是在 `单例子线程上`完成的

- 虽然 NSURLConnection操作是 sync同步的方式，但是
	
	- 单例子线程开启了runloop
	- runloop注册接收 NSURLConnection事件源、NSOutputStream事件源
	- 所以没有这两个事件源时，会 `休眠所在线程`
	- 一旦接收到事件源时，又会 唤醒所在线程

- 一旦最终 单例子线程 完成了NSURLConnection操作
	- 首先由 系统线程 通知 单例子线程
	- 然后再由 单例子线程 通知 NSOperation对象子线程
	- 最后 NSOperation对象子线程 通知 主线程UI

那么就是说所有`每一个网络请求NSURLConnection操作`大部分操作都是在单例子线程上完成。而NSOperation对象线程只是等待单例子线程执行完毕网络操作。

****

###AFURLConnectionOperation对象取消执行cancel，仅仅只是取消执行operation对象.


```objc
@implementation AFURLConnectionOperation

- (void)cancel {
    
    //1.
    [self.lock lock];
    
    //2. 必须是 没有完成 与 没有取消过
    if (![self isFinished] && ![self isCancelled]) {
        
        //2.1 执行父类NSOperation实例的cancel方法，取消执行operation
        //到此为止，仅仅只是取消了operation执行
        //但是operation对象的NSURLConnection操作没有被取消
        [super cancel];

        //2.2 继续取消operation对象的NSURLConnection操作
        //因为 NSURLConnection操作、事件源 都是与 单例子线程 相关
        //所以也必须回到 单例子线程上
        if ([self isExecuting]) 
        {
        	//因为重写了 isExecuting函数，所以此时不是 executing 状态
        
            [self performSelector:@selector(cancelConnection)
                         onThread:[[self class] networkRequestThread]
                       withObject:nil
                    waitUntilDone:NO
                            modes:[self.runLoopModes allObjects]];
        }
    }
    
    //3.
    [self.lock unlock];
}

@end
```

所以还得关注cancelConnection方法.

###继续取消operation对象的NSURLConnection操作

```objc
@implementation AFURLConnectionOperation

- (void)cancelConnection {
    
    //1. 构造 userInfo字典，保存此次取消执行operation的相关信息
    NSDictionary *userInfo = nil;
    if ([self.request URL]) {
        userInfo = @{NSURLErrorFailingURLErrorKey : [self.request URL]};
    }
    
    //2. 使用前面的userInfo字典，构造取消执行operation的NSError
    NSError *error = [NSError errorWithDomain:NSURLErrorDomain
                                         code:NSURLErrorCancelled//注意错误code
                                     userInfo:userInfo];

    //3. 如果已经完成的operation
    //就不用执行取消connection
    //完成的operation会做connection取消操作
    if (![self isFinished])
    {
        //3.1 没有结束的operation
        
        if (self.connection)
        {
            //3.1.1 operation对象的connection存在
            
            //3.1.2 取消执行connection
            [self.connection cancel];
            
            //3.1.3 手动调用NSURLConnectionDelegate的错误回调函数
            [self performSelector:@selector(connection:didFailWithError:)
                       withObject:self.connection
                       withObject:error];
            
        } else {
           
            // Accommodate race condition where `self.connection` has not yet been set before cancellation
            //3.2（当做没有设置operation对象的connection）
            
            //3.2.1 保存错误
            self.error = error;
            
            //3.2.2 结束执行operation
            //通知执行completionBlock
            //然后回传这个保存的error
            [self finish];
        }
    }
}

@end
```

如上代码得知，结束执行operation对象的NSURLConnection操作时分两种情况:

- operation对象，拥有一个 NSURLConnection对象（`非空`）

- operation对象的connection为`空`

****

####operation对象，拥有一个 NSURLConnection对象（`非空`）

- 手动调用NSURLConnectionDelegate的错误回调函数

```
//1. 
[self.connection cancel];

//2.
[self performSelector:@selector(connection:didFailWithError:)
                       withObject:self.connection
                       withObject:error];
```

- 被调用的NSURLConnectionDelegate函数

```objc
@implementation AFURLConnectionOperation

#pragma mark - NSURLConnectionDelegate

- (void)connection:(NSURLConnection __unused *)connection
  didFailWithError:(NSError *)error
{
	//1. 保存请求错误
    self.error = error;

	//2. 关闭operation对象的输出流
    [self.outputStream close];
    
    //3. 如果存在responseData数据，说明输出流使用过
    if (self.responseData) {
    
    	//如果存在输出流对象，还需要释放内存
        self.outputStream = nil;
    }

	//4. 释放掉connection
    self.connection = nil;

	//5. 结束执行operation，执行回调completionblock
    [self finish];
}

@end
```

此种情况，还需要 关闭 接收网络数据的 NSOutputStream对象，然后再结束执行operation.


###operation对象的connection为`空`

- 因为没有connection对象了，所以直接结束执行operation.

```
//1.
self.error = error;

//2.
[self finish];
```

***

###AFURLConnectionOperation对象结束执行

```objc
@implementation AFURLConnectionOperation

- (void)finish {
    
    //1. 加锁同步
    [self.lock lock];
    
    //2. 修改为Finish状态
    self.state = AFOperationFinishedState;
    
    //3. 解锁
    [self.lock unlock];

    //4. 告诉operation执行完毕
    dispatch_async(dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingOperationDidFinishNotification object:self];
    });
}

@end
```

修改state后，发出 `_isFinish`属性修改的KVO通知，最终被系统监听收到，知道这个operation执行完毕了，然后就会执行其completionBlock，最终完成了向用户回调.

***

###AFURLConnectionOperation对象暂停执行

```objc
@implementation AFURLConnectionOperation

- (void)pause {
    
    //1. 如果当前operation处于 暂停、结束、取消，不能暂停
    if ([self isPaused] || [self isFinished] || [self isCancelled]) {
        return;
    }

    //2. 同步加锁
    [self.lock lock];
    
    //3. 只有处于 `执行中` 的operation可以被pause
    if ([self isExecuting])
    {
        //3.1 一样的 单例子线程上 执行暂停operation
        [self performSelector:@selector(operationDidPause)
                     onThread:[[self class] networkRequestThread]
                   withObject:nil
                waitUntilDone:NO
                        modes:[self.runLoopModes allObjects]];

        //3.2 发送通知，告知operation暂停
        dispatch_async(dispatch_get_main_queue(), ^{
            NSNotificationCenter *notificationCenter = [NSNotificationCenter defaultCenter];
            [notificationCenter postNotificationName:AFNetworkingOperationDidFinishNotification object:self];
        });
    }

    //4. 修改state为pause暂停
    self.state = AFOperationPausedState;
    
    //5. 解锁同步
    [self.lock unlock];
}

@end
```

```objc
@implementation AFURLConnectionOperation

- (void)operationDidPause {
    
    //1.
    [self.lock lock];
    
    //2. 只是取消NSURLConnection对象执行
    //那么runloop也不再接收事件
    [self.connection cancel];
    
    //3.
    [self.lock unlock];
}

@end
```

###AFURLConnectionOperation对象恢复执行

```objc
@implementation AFURLConnectionOperation

- (void)resume {
    
    //1. 只有被pause的operation，才可以被resume
    if (![self isPaused]) {
        return;
    }

    //2. 加锁
    [self.lock lock];
    
    //3. operation state 为 ready
    self.state = AFOperationReadyState;

    //4. 手动调用 operation对象的 start方法
    //（start方法是开始网络操作的入口）
    [self start];
    
    //5. 解锁
    [self.lock unlock];
}

@end
```

###图示总结线程关系

- 每一个网络请求，都是一个新的NSOperation子线程
- 但是最终完成网络数据获取的是一个 单例子线程NSThread
- 单例子线程NSThread 回调 每一个新的NSOperation子线程

![](http://i8.tietuku.com/5ef41fd770dcefdb.png)