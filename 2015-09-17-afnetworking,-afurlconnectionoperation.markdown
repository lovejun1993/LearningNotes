---
layout: post
title: "AFNetworking、AFURLConnectionOperation"
date: 2015-09-17 14:02:33 +0800
comments: true
categories: 
---

###AFURLConnectionOperation提供:

- 实现NSOperation的状态管理

- 创建 NSURLConnection，进行网络请求、回调，单例Thread上进行schedule

- 可以暂停pause和继续执行resume

- 各种回调block设置


***

###先看下AFURLConnectionOperation.h向外暴露的一些入口

***

###AFURLConnectionOperation实现的所有协议

```objc
@interface AFURLConnectionOperation : NSOperation <NSURLConnectionDelegate, NSURLConnectionDataDelegate, NSSecureCoding, NSCopying>

....
```

- 协议1、NSURLConnectionDelegate 网络数据请求
- 协议2、NSURLConnectionDataDelegate 网络数据请求
- 协议3、NSSecureCoding 对象归档
- 协议4、NSCopying 对象拷贝

***

###设置 RunLoop 接收NSURLConnection事件源的 modes，默认mode是 `NSRunLoopCommonModes`，即在所有mode下都接收事件源.

```objc
@property (nonatomic, strong) NSSet *runLoopModes;
```

***

###封装请求数据的request

- 向外只读
- 对内读写

```objc
@property (readonly, nonatomic, strong) NSURLRequest *request;
```

***

###让外部读取 响应描述对象

```objc
@property (readonly, nonatomic, strong, nullable) NSURLResponse *response;
```

***

###让外部读取 请求错误

```objc
@property (readonly, nonatomic, strong, nullable) NSError *error;
```

***

###让外部获取 响应数据的二进制格式

```objc
@property (readonly, nonatomic, strong, nullable) NSData *responseData;
```

***

###让外部获取 响应数据的字符串格式


```objc
@property (readonly, nonatomic, copy, nullable) NSString *responseString;
```

***

###响应数据解析的编码格式，外部只读，默认是UTF8

.h

```objc
@property (readonly, nonatomic, assign) NSStringEncoding responseStringEncoding;
```

.m

```objc
@implementation AFURLConnectionOperation

- (NSStringEncoding)responseStringEncoding {
    
    //1. Lock
    [self.lock lock];
    
    //2. 得到编码格式
    if (!_responseStringEncoding && self.response)
    {
        //2.1 首先设置默认的编码格式 UTF8
        NSStringEncoding stringEncoding = NSUTF8StringEncoding;
        
        //2.2 使用响应的编码格式
        if (self.response.textEncodingName)
        {
            //2.2.1
            CFStringEncoding IANAEncoding = CFStringConvertIANACharSetNameToEncoding((__bridge CFStringRef)self.response.textEncodingName);
            
            //2.2.2
            if (IANAEncoding != kCFStringEncodingInvalidId)
            {
                stringEncoding = CFStringConvertEncodingToNSStringEncoding(IANAEncoding);
            }
        }

        //2.3 设置最终的编码格式
        self.responseStringEncoding = stringEncoding;
    }
    
    //3. Un Lock
    [self.lock unlock];

    return _responseStringEncoding;
}

@end
```

***

###让外部设置 是否使用Storate存储 web服务挑战时创建的凭证 NSURLCreditial对象，以便后续使用，默认YES

.h

```objc
@property (nonatomic, assign) BOOL shouldUseCredentialStorage;
```

.m 实现 `NSURLConnectionDelegate` 相关函数中使用这个BOOL值

```objc
@implementation AFURLConnectionOperation

- (BOOL)connectionShouldUseCredentialStorage:(NSURLConnection __unused *)connection {
    return self.shouldUseCredentialStorage;
}

@end
```

***

###让外部设置 接收到web服务挑战时的 需要的凭证

.h

```objc
@property (nonatomic, strong, nullable) NSURLCredential *credential;
```

.m 当接收到需要接收挑战的回调时，`NSURLConnectionDelegate`协议中的如下函数会被调用

```objc
@protocol NSURLConnectionDelegate <NSObject>
@optional

...

- (void)connection:(NSURLConnection *)connection willSendRequestForAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge;

...

@end
```

然后根据传入的challenge创建凭证NSCreditial对象，并发送给服务器认证.

***

###用于数据传输的安全性

```objc
@property (nonatomic, strong) AFSecurityPolicy *securityPolicy;
```

***

###让外部设置 上传数据的 输入流

.h

```objc
@property (nonatomic, strong) NSInputStream *inputStream;
```

.m 中的getter，获取operation对象的输入流，实际上是从一个 `NSURLRequest对象`中读取输入流

```objc
- (NSInputStream *)inputStream {
    return self.request.HTTPBodyStream;
}
```

.m 中的setter，将新一个新的输入流 设置给 当前operation对象持有的NSURLRequest对象中

```objc
- (void)setInputStream:(NSInputStream *)inputStream {

	//1. 深拷贝当前operation持有的request对象
    NSMutableURLRequest *mutableRequest = [self.request mutableCopy];
    
    //2. 替换request对象的输入流
    mutableRequest.HTTPBodyStream = inputStream;
    
    //3. 替换当前operation对象的request对象
    self.request = mutableRequest;
}
```

***

###让外部设置 写入磁盘文件的 输出流

.h

```objc
@property (nonatomic, strong, nullable) NSOutputStream *outputStream;
```

.m 中getter，获取operation对象的输出流

```objc
- (NSOutputStream *)outputStream {
    if (!_outputStream) {
        
        //在内存中创建的一个临时输出流
        self.outputStream = [NSOutputStream outputStreamToMemory];
    }

    return _outputStream;
}
```

.m 中setter，替换operation对象的输出流

```objc
- (void)setOutputStream:(NSOutputStream *)outputStream {
    
    //1. 加锁
    [self.lock lock];
    
    //2. 替换输出流
    if (outputStream != _outputStream) {
        
        //2.1 先关闭之前的输出流
        if (_outputStream) {
            [_outputStream close];
        }

        //2.2 再替换新的输出流
        _outputStream = outputStream;
    }
    
    //3. 解锁
    [self.lock unlock];
}
```

***

###用户执行Callback Queue与Group

```objc
/**
 The dispatch queue for `completionBlock`. If `NULL` (default), the main queue is used.
 */
#if OS_OBJECT_USE_OBJC
@property (nonatomic, strong, nullable) dispatch_queue_t completionQueue;
#else
@property (nonatomic, assign, nullable) dispatch_queue_t completionQueue;
#endif

/**
 The dispatch group for `completionBlock`. If `NULL` (default), a private dispatch group is used.
 */
#if OS_OBJECT_USE_OBJC
@property (nonatomic, strong, nullable) dispatch_group_t completionGroup;
#else
@property (nonatomic, assign, nullable) dispatch_group_t completionGroup;
#endif
```

***

###可以让外界设置的 operation对象的 userInfo字典

```objc
/**
 The user info dictionary for the receiver.
 */
@property (nonatomic, strong) NSDictionary *userInfo;
// FIXME: It doesn't seem that this userInfo is used anywhere in the implementation.
```

暂时不知道这个字典可以用来干什么...

***

###创建AFURLConnectionOperation的`指定构造函数`

```objc
- (instancetype)initWithRequest:(NSURLRequest *)urlRequest NS_DESIGNATED_INITIALIZER;
```

***

###向外提供可以 `暂停执行` operation，以及查询是否处于暂停状态

```objc
/**
 1. 如果是对 `Ready`, `Executing`, `Finished` 这三种状态的operation执行 `isPause消息`，返回NO.
 2. 被执行pause的operation, 会被保留在operationQueue, 当被执行 `取消cancel` 和 `继续执行resume` 时，才会从queue移除.
 3. 对 finished, cancelled, paused 的 operation `进行pause`，没有任何效果.
 */
- (void)pause;
```

那么也就是说pause也只是把operation保留在queue中，具体实现后面再说.

```objc
- (BOOL)isPaused;
```

###向外提供可以 `恢复执行` operation

```objc
- (void)resume;
```

```objc
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
```

***

###设置operation后台执行的Block代码块

```objc
#if defined(__IPHONE_OS_VERSION_MIN_REQUIRED)
- (void)setShouldExecuteAsBackgroundTaskWithExpirationHandler:(nullable void (^)(void))handler NS_EXTENSION_UNAVAILABLE_IOS("Not available in app extensions.");
#endif
```

###给operation设置，上传回调block

```objc
- (void)setUploadProgressBlock:(nullable void (^)(NSUInteger bytesWritten, long long totalBytesWritten, long long totalBytesExpectedToWrite))block;
```

###给operation设置，下载回调block

```objc
- (void)setDownloadProgressBlock:(nullable void (^)(NSUInteger bytesRead, long long totalBytesRead, long long totalBytesExpectedToRead))block;
```

###给operation设置，接收到web服务挑战时的回调block

```objc
- (void)setWillSendRequestForAuthenticationChallengeBlock:(nullable void (^)(NSURLConnection *connection, NSURLAuthenticationChallenge *challenge))block;
```

###给operation设置，请求被重定向时的回调block（注册自己的NSURLProtocol子类完成请求重定向）

```objc
- (void)setRedirectResponseBlock:(nullable NSURLRequest * (^)(NSURLConnection *connection, NSURLRequest *request, NSURLResponse *redirectResponse))block;
```

###给operation设置，响应描述response被缓存时的回调block（`NSURLConnectionDelegate` method `connection:willCacheResponse:`被调用时，执行如下block）

```objc
- (void)setCacheResponseBlock:(nullable NSCachedURLResponse * (^)(NSURLConnection *connection, NSCachedURLResponse *cachedResponse))block;
```

***

###同时执行多个operation


```objc
+ (NSArray *)batchOfRequestOperations:(nullable NSArray *)operations
                        progressBlock:(nullable void (^)(NSUInteger numberOfFinishedOperations, NSUInteger totalNumberOfOperations))progressBlock
                      completionBlock:(nullable void (^)(NSArray *operations))completionBlock;
```


###AFOperation start与finish状态的通知key定义

.h文件

```objc
/**
 Posted when an operation begins executing.
 */
FOUNDATION_EXPORT NSString * const AFNetworkingOperationDidStartNotification;

/**
 Posted when an operation finishes.
 */
FOUNDATION_EXPORT NSString * const AFNetworkingOperationDidFinishNotification;
```

.m文件定义

```objc
NSString * const AFNetworkingOperationDidStartNotification = @"com.alamofire.networking.operation.start";
```

```objc
NSString * const AFNetworkingOperationDidFinishNotification = @"com.alamofire.networking.operation.finish";
```

****

###下面开始.m实现代码....

****

###c方法创建用于执行completion block的 group和queue


```objc
static dispatch_group_t url_request_operation_completion_group() {
    static dispatch_group_t af_url_request_operation_completion_group;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        af_url_request_operation_completion_group = dispatch_group_create();
    });

    return af_url_request_operation_completion_group;
}
```

```objc
static dispatch_queue_t url_request_operation_completion_queue() {
    static dispatch_queue_t af_url_request_operation_completion_queue;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        af_url_request_operation_completion_queue = dispatch_queue_create("com.alamofire.networking.operation.queue", DISPATCH_QUEUE_CONCURRENT );
    });

    return af_url_request_operation_completion_queue;
}
```

***

###AFNetworking使用 递归锁的名字

```objc
static NSString * const kAFNetworkingLockName = @"com.alamofire.networking.operation.lock";
```

****

###AFURLConnectionOperation 实例 init做的事情

```objc
@implementation AFURLConnectionOperation

- (instancetype)initWithRequest:(NSURLRequest *)urlRequest {
    NSParameterAssert(urlRequest);

    self = [super init];
    if (!self) {
		return nil;
    }

	//1.
    _state = AFOperationReadyState;

	//2.
    self.lock = [[NSRecursiveLock alloc] init];
    self.lock.name = kAFNetworkingLockName;

	//3. 默认mode为commom
    self.runLoopModes = [NSSet setWithObject:NSRunLoopCommonModes];
	
	//4. 保存传入的NSURLRequest对象
    self.request = urlRequest;

	//5. 默认使用creditial storage 存储
    self.shouldUseCredentialStorage = YES;

	//6. 默认使用的安全策略
    self.securityPolicy = [AFSecurityPolicy defaultPolicy];

    return self;
}

//不能使用init方法进行初始化
- (instancetype)init NS_UNAVAILABLE
{
    return nil;
}

@end
```

###AFURLConnectionOperation 实例 dealloc 做的事情

```objc
@implementation AFURLConnectionOperation

- (void)dealloc {

	//1. 关闭输出流
	//因为输出流是临时给operation对象创建的
	//输入流是NSURLRequest持有的，不需要operation对象管理
    if (_outputStream) {
        [_outputStream close];
        _outputStream = nil;
    }
    
    //2. 在即将被释放的时候，才执行后台任务block
    if (_backgroundTaskCleanup) {
        _backgroundTaskCleanup();
    }
}

@end
```

###AFURLConnectionOperation实例 设置 后台执行的block

```
/**
 *  1. 保存传入的一个 void (^)(void) 类型的block，作为后台任务
 *  2. 使用UIApplication单例开启一个后台任务
 *  3. 只能设置一次后台任务
 */
```


```objc
#if defined(__IPHONE_OS_VERSION_MIN_REQUIRED)
- (void)setShouldExecuteAsBackgroundTaskWithExpirationHandler:(void (^)(void))handler {
    
    //1.
    [self.lock lock];
    
    //2. 只能设置一次 后台任务block
    if (!self.backgroundTaskCleanup) {
        
        //2.1 单例application
        UIApplication *application = [UIApplication sharedApplication];
        
        //2.2 后台任务id
        UIBackgroundTaskIdentifier __block backgroundTaskIdentifier = UIBackgroundTaskInvalid;
        
        //2.3 weak self
        __weak __typeof(self)weakSelf = self;
        
        //2.4 构造最终operation对象释放时，执行的block
        //（用于告诉系统，提交结束执行后台task任务的block）
        self.backgroundTaskCleanup = ^(){
            
            //2.4.1 提交后台Task任务，执行完毕
            if (backgroundTaskIdentifier != UIBackgroundTaskInvalid) {
                [[UIApplication sharedApplication] endBackgroundTask:backgroundTaskIdentifier];
                backgroundTaskIdentifier = UIBackgroundTaskInvalid;
            }
        };
        
        //2.5 开始执行后台task
        backgroundTaskIdentifier = [application beginBackgroundTaskWithExpirationHandler:^{
            
            __strong __typeof(weakSelf)strongSelf = weakSelf;

            //2.5.1 执行传入的block
            if (handler) {
                handler();
            }

            //2.5.2 如果执行完后台任务，operation对象还没有被释放掉
            //那么此时必须告诉系统结束执行后台任务
            if (strongSelf) {
                
                //2.5.2.1 取消执行operation
                [strongSelf cancel];
                
                //2.5.2.2 告诉系统后台任务执行完毕
                strongSelf.backgroundTaskCleanup();
            }
        }];
    }
    
    //3.
    [self.lock unlock];
}
#endif
```

告诉 `后台任务 执行完毕` 时，operation有两种情况:

- 没有被释放
	- 再继续告诉系统，后台任务执行完毕

- 已经被释放
	- dealloc方法中，告诉系统，后台任务执行完毕

***

###重写系统的description方法

```objc
- (NSString *)description {
    [self.lock lock];
    NSString *description = [NSString stringWithFormat:@"<%@: %p, state: %@, cancelled: %@ request: %@, response: %@>", NSStringFromClass([self class]), self, AFKeyPathFromOperationState(self.state), ([self isCancelled] ? @"YES" : @"NO"), self.request, self.response];
    [self.lock unlock];
    return description;
}
```