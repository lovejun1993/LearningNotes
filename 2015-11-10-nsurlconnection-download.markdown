---
layout: post
title: "NSURLConnection Download"
date: 2015-11-10 21:36:02 +0800
comments: true
categories: 
---




###学习NSURLSession的准备，要知道为啥NSURLSession优秀，就得知道NSURLConnection的缺点.

****

###首先看以下NSURLConnection用于回调代码的协议定义

####NSURLConnectionDelegate

```
@protocol NSURLConnectionDelegate <NSObject>

//1. 都是可选实现
//2. 这个协议主要是请求错误、证书校验安全相关

@optional

- (void)connection:(NSURLConnection *)connection didFailWithError:(NSError *)error;

- (BOOL)connectionShouldUseCredentialStorage:(NSURLConnection *)connection;

- (void)connection:(NSURLConnection *)connection willSendRequestForAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge;

- (BOOL)connection:(NSURLConnection *)connection canAuthenticateAgainstProtectionSpace:(NSURLProtectionSpace *)protectionSpace NS_DEPRECATED(10_6, 10_10, 3_0, 8_0, "Use -connection:willSendRequestForAuthenticationChallenge: instead.");

- (void)connection:(NSURLConnection *)connection didReceiveAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge NS_DEPRECATED(10_2, 10_10, 2_0, 8_0, "Use -connection:willSendRequestForAuthenticationChallenge: instead.");

- (void)connection:(NSURLConnection *)connection didCancelAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge NS_DEPRECATED(10_2, 10_10, 2_0, 8_0, "Use -connection:willSendRequestForAuthenticationChallenge: instead.");

@end
```

####NSURLConnectionDataDelegate

```
@protocol NSURLConnectionDataDelegate <NSURLConnectionDelegate>

//1. 都是可选实现
//2. 主要是关于数据获取: 接收到响应、接收到数据、发送数据、缓存respnse
//3. 可以看到对【小数据请求】、【下载请求】、【上传请求】这三种操作都定义在了这一个协议中.

@optional

- (nullable NSURLRequest *)connection:(NSURLConnection *)connection willSendRequest:(NSURLRequest *)request redirectResponse:(nullable NSURLResponse *)response;

- (void)connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response;

- (void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data;

- (nullable NSInputStream *)connection:(NSURLConnection *)connection needNewBodyStream:(NSURLRequest *)request;

- (void)connection:(NSURLConnection *)connection   didSendBodyData:(NSInteger)bytesWritten
                                                 totalBytesWritten:(NSInteger)totalBytesWritten
                                         totalBytesExpectedToWrite:(NSInteger)totalBytesExpectedToWrite;

- (nullable NSCachedURLResponse *)connection:(NSURLConnection *)connection willCacheResponse:(NSCachedURLResponse *)cachedResponse;

- (void)connectionDidFinishLoading:(NSURLConnection *)connection;
@end
```

***

###写了一个简单的下载类`NSURLConnectionManager`内部使用`NSURLConnection `

```
.h


#import <Foundation/Foundation.h>

@interface URLConnectionManager : NSObject

/**
 *  因为每一个请求，必须重新创建一个新的NSURLConection对象。
 *  所以，不使用单例方法
 */
//+ (instancetype)sharedInstance;

/**
 *  传入NSURLRequest，来创建一个NSURLConection对象
 */
- (instancetype)initWithRequest:(NSURLRequest *)request;

/**
 *  开始网络操作
 */
- (void)startWithCompletionBlock:(void (^)(NSData *data))block;

@end
```

```
.m

#import "URLConnectionManager.h"

@interface URLConnectionManager () <NSURLConnectionDataDelegate>

/**
 *  runloop运行的模式，可以指定runloop在某一个特定的mode下才能接收事件源
 *
 *      1. NSDefaultRunLoopMode
 *      2. NSConnectionReplyMode
 *      3. NSModalPanelRunLoopMode
 *      4. UITrackingRunLoopMode
 *      5. NSRunLoopCommonModes（综合如上全部情况）
 *
 */
@property (nonatomic, strong) NSSet *runLoopModes;

@property (nonatomic, strong) NSRecursiveLock *lock;

@property (nonatomic, strong) NSURLRequest *request;

@property (nonatomic, strong) NSURLConnection *connection;

@property (nonatomic, strong) NSOutputStream *outputStream;
@property (nonatomic, strong) NSInputStream *inputStream;


@property (nonatomic, copy) void (^completion)(NSData *data);
@property (nonatomic, assign) NSMutableData *receiveData;
@property (nonatomic, assign) long long expectedContentLength;
@property (nonatomic, assign) long long currentLength;

@end

@implementation URLConnectionManager

//+ (instancetype)sharedInstance {
//    static URLConnectionManager *manager = nil;
//    static dispatch_once_t onceToken;
//    dispatch_once(&onceToken, ^{
//        manager = [[URLConnectionManager alloc] init];
//    });
//    return manager;
//}

- (instancetype)initWithRequest:(NSURLRequest *)request {
    self = [super init];
    if (self) {
        
        _runLoopModes = [NSSet setWithObjects:NSRunLoopCommonModes, nil];
        
        _lock = [[NSRecursiveLock alloc] init];
        _lock.name = @"URLConnectionManagerLock";
        
        _receiveData = [NSMutableData data];

        _request = request;
    }
    return self;
}

#pragma mark - 创建一个单例子线程，用来完成下载任务和回调代理方法执行

+ (NSThread *)networkThread {
    static NSThread *thread = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        
        //1. 创建一个子线程
        thread = [[NSThread alloc] initWithTarget:self
                                         selector:@selector(threadEntryPoint:)
                                           object:nil];
        
        //2. 一定要开启线程
        [thread start];
    });
    return thread;
}

+ (void)threadEntryPoint:(id)__unused object {
    
    //1. 设置线程
    NSThread *t = [NSThread currentThread];
    t.name = @"URLConnectionManagerThread";
    
    //2. 打开子线程的runloop
    NSRunLoop *runloop = [NSRunLoop currentRunLoop];
    
    //3. 给runloop注册一个port，用于iOS系统线程与当前子线程的通信
    [runloop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
    
    //4. 最后一定要开启runloop
    [runloop run];
}

#pragma mark - 创建 NSURLConnection

- (void)operationDidStart {
    
    //1 加锁
    [_lock lock];
    
    //2. 创建connection
    _connection = [[NSURLConnection alloc] initWithRequest:_request
                                                  delegate:self
                                          startImmediately:NO];
    
    //3.打印当前线程，可以看到是在前面创建的子线程
    NSLog(@"当前所在线程: %@\n", [NSThread currentThread]);
    
    //4. 获取当前子线程的RunLoop
    NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
    
    //5. 在之前设置的所有runloop的moede下，设置接收如下事件源
    for (NSString *runLoopMode in _runLoopModes) {
        
        //NSURLConnection的事件源
        [_connection scheduleInRunLoop:runLoop forMode:runLoopMode];
        
        //输出流的事件源
        [_outputStream scheduleInRunLoop:runLoop forMode:runLoopMode];
    }
    
    //6. 开始执行connection和outputStream
    [_outputStream open];
    [_connection start];
    
    //7. 解锁
    [_lock unlock];
}

#pragma mark - 向外暴露的开始下载的入口

- (void)startWithCompletionBlock:(void (^)(NSData *data))block {
    
    //1. 加锁
    [_lock lock];
    
    _completion = [block copy];
    
    //2. 让网络相关代码都在单例子线程上执行
    [self performSelector:@selector(operationDidStart)
                 onThread:[[self class] networkThread]
               withObject:nil
            waitUntilDone:NO
                    modes:[_runLoopModes allObjects]];
    
    //3. 解锁
    [_lock unlock];
}

#pragma mark - NSURLConnectionDataDelegate （如下回调函数，都是在单例子线程上执行）

/**
 *  接收到服务器的响应，这个方法只会被回调执行一次
 */
- (void)connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response {
    
    //1. 获取文件的大小
    NSLog(@"文件的大小: %lld\n", response.expectedContentLength);
    
    //2. 记录文件总大小
    _expectedContentLength = response.expectedContentLength;
    
    //3. 将当前下载记录大小清零
    _currentLength = 0;
}

/**
 *  接收到服务器的数据，这个方法会回调执行多次，每次回传当前下载的部分数据
 */
- (void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data {
    
    //1. 类加当前下载总长度
    _currentLength += [data length];
    
    //2. 计算得到当前下载的进度（NSURLConnection没有提供直接获取进度的Api，需要自己计算得到）
    float progress = (float) _currentLength / _expectedContentLength;
    NSLog(@"当前下载进度: %f\n", progress);
    
    //3. 保存下载到的数据
    [_receiveData appendData:data];
}

/**
 *  下载完毕
 */
- (void)connectionDidFinishLoading:(NSURLConnection *)connection {
    
    //1. 将最终保存的NSData写入磁盘文件
    [_receiveData writeToFile:@"" atomically:YES];
    
    //2. 释放内存
    _receiveData = nil;
    
    //3. 回调主线程回调执行
    __weak __typeof(self)weakSefl = self;
    dispatch_async(dispatch_get_main_queue(), ^{
        if (weakSefl.completion) {
            weakSefl.completion(_receiveData);
        }
    });
}

#pragama mark - NSURLConnectionDelegate

/**
 *  下载错误
 */
- (void)connection:(NSURLConnection *)connection didFailWithError:(NSError *)error {
    
    //1. 取出当前错的code
    NSInteger errCode = error.code;
    
    //2. 比对是什么code
    //所有的code都可以在NSURLError.h找到
    
    //3. 针对是哪一种error code处理
    //...
}

@end
```

- 总结下一些缺点
	
	- 1) 需要`自己创建一个单独的线程`来完成所有的`下载有关操作`，否则就会造成主线程卡顿
			- 操作1、NSURLConnection实例在子线程上执行
			- 操作2、NSURLConnectionDataDelegate的回调也再子线程上执行
			- 操作3、单例线程收runloop激活和休眠
	- 2) 不会区分`小数据获取数据`、`上传数据`、`下载数据`这三种网络任务，统一定义在一个协议中
	- 3) 在完成`下载或上传`时，没有提供获取`进度`的Api，而是需要我们自己去手动的计算得到比例
	- 4) `下载`时，也没有帮我们完成文件的写入，同样需要我们自己完成文件的写入操作
	- 5） 只能完成`start`和`cancel`，没有`suspend`和`resume`

####对如上代码稍微修改: `让下载得到的每一点NSData进行分段写入到磁盘文件`，而不是向上面一样最后一次性的全部写入。当文件很大的时候，一次性写入就会造成性能问题。

- 在上的基础之上，列出修改的地方

```
@interface URLConnectionManager () <NSURLConnectionDataDelegate>

//...略

//记录响应的文件名
@property (nonatomic, copy) NSString *filename;

//磁盘文件分段写入器
@property (nonatomic, strong) NSFileHandle *fileHandle;

@end
```

```
@implementation URLConnectionManager

//...略

#pragma mark - NSURLConnectionDataDelegate ， 如下回调函数，都是在单例子线程上执行

/**
 *  接收到服务器的响应，这个方法只会被回调执行一次
 */
- (void)connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response {
    
    //...略
    
    //4. 记录文件名
    _filename = [response suggestedFilename];
    
    //5. 创建文件分段写入方式
    NSString *LibPath = [NSSearchPathForDirectoriesInDomains(NSLibraryDirectory, NSUserDomainMask, YES) objectAtIndex:0];
    NSString *abPath = [LibPath stringByAppendingPathComponent:_filename];
    _fileHandle = [NSFileHandle fileHandleForWritingAtPath:abPath];//设置写入的路径
}

/**
 *  接收到服务器的数据，这个方法会回调执行多次，每次回传当前下载的部分数据
 */
- (void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data {
    
    //...略
    
    //3. 追加的形式写入磁盘文件
    [_fileHandle seekToEndOfFile];//先移到文件内容末尾处
    [_fileHandle writeData:data];//再写入追加的data
}

/**
 *  下载完毕
 */
- (void)connectionDidFinishLoading:(NSURLConnection *)connection {
    
    //使用分段式写入磁盘文件，所以就不要这里代码了
    //-------------------------------------------
    /*
    //1. 将最终保存的NSData写入磁盘文件
    [_receiveData writeToFile:@"" atomically:YES];
    
    //2. 释放内存
    _receiveData = nil;
    */
    //------------------------------------------- 
    
    //1. 关闭分段写入
    [_fileHandle closeFile];
    
    
    //2. 回调主线程回调执行
    __weak __typeof(self)weakSefl = self;
    dispatch_async(dispatch_get_main_queue(), ^{
        if (weakSefl.completion) {
            weakSefl.completion(_receiveData);
        }
    }); 
}

@end
```

- 注意，由于分段写入是`追加`的形式不断的写入文件，所以当开始下载的时候，要防止错误的追加原来的文件进行写入

	- 接收响应的文件名时，删掉磁盘已经存在的文件


####再对上述稍加修改，完成下载进度回调

```

#import <Foundation/Foundation.h>

@interface URLConnectionManager : NSObject

/**
 *  设置进度回调Block
 */
@property (nonatomic, copy) void (^onProgress)(float progress);

//...略

@end
```

```
@implementation URLConnectionManager

//...略

#pragma mark - NSURLConnectionDataDelegate ， 如下回调函数，都是在单例子线程上执行

//...略

/**
 *  接收到服务器的数据，这个方法会回调执行多次，每次回传当前下载的部分数据
 */
- (void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data {
    
    //...略
    
    //4. 回跳主线程执行回传进度
    if (self.onProgress) {
        __weak __typeof(self)weakSefl = self;
        dispatch_async(dispatch_get_main_queue(), ^{
            weakSefl.onProgress(progress);
        });
    }
}



@end
```

####注意，在最开始有段代码，完成在子线程中创建NSURLConnection，并设置由子线程的runloop接收事件，最后在子线程上start.

```
- (void)startWithCompletionBlock:(void (^)(NSData *data))block {
    
    //1. 加锁
    [_lock lock];
    
    _completion = [block copy];
    
    //2. 将NSURLConnection的创建以及运行，都在单例子线程上完成
    [self performSelector:@selector(operationDidStart)
                 onThread:[[self class] networkThread]
               withObject:nil
            waitUntilDone:NO
                    modes:[_runLoopModes allObjects]];
    
    //3. 解锁
    [_lock unlock];
}
```

####而上面的`operationDidStart`方法中，可以看到之前的代码，最终使用的是一个`开启了runloop`的子线程来完成所有的NSURLConnection的操作.

####那么有如下问题.
	
- 问题一、为什么在子线程的runloop上schedule NSURLConnection实例？（`让runloop接收这个类型的事件`）

- 问题二、NSMatchPort如何完成线程之间的通信？


解决这些问题，就必须彻底弄清楚`NSRunLoop`，感觉要弄清楚的东西比较多，还是单独写一篇文章记录把.

***

###NSURLConnection断点下载

- NSURLConnection没有提供直接完成断点下载的Api，只有开始和取消
- 那么要完成断点下载，需要人为的设置`请求头的range属性`告诉服务器要下载哪一部分数据

****

###NSURLConnection大文件多线程并发下载

- 第一步、获取到服务器文件的`总大小`

- 第二步、根据`每一个下载任务大小`，分成多个子线程任务完成下载
	
	- 线程1，完成下载0字节 -- 100字节
	- 线程2，完成下载100字节 -- 200字节
	- 线程3，完成下载200字节 -- 300字节
	- 线程4，完成下载300字节 -- 400字节

- 第三步、手机端磁盘创建一个与`下载文件总大小一样`的临时文件

- 第四步、每个下载任务线程，将下载得到NSData写入对应的部分

***

###本文仅针对 App运行在 前台 时完成下载.