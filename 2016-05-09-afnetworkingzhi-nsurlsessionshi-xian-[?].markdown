---
layout: post
title: "AFNetworking之NSURLSession实现"
date: 2016-05-09 13:27:11 +0800
comments: true
categories: 
---


###准备自己做一个类似YTKNetwork这样的AFNetworking的二次封装，为什么了？

- 小说App的文字处理方面已经OK了，接下来需要做一套网络类库的模子，也可以方便的在以后其他的App中一样使用.

- YTKNetwork就是一个壳子，套住一些第三方开源的网络类库，便于随时切换
	- 具体YTKNetwork的一些细节，请参考其源码实现
	- 简单的理解就是把一个网络请求抽象成一个个独立的Model对象（我想大多数都是这么弄的...）
	- 我看过一些对于AFNetworking的封装:
		- 比较挫的: 直接对AFRequestManager封装一下，其实仍然大面积的依赖AFNetworking
		- 比较好的: 对一个网络请求的全过程进行抽象，较小面积的依赖AFNetworking
			- 实体1、网络请求 >>> Request（单个、批量、链式）
			- 实体2、请求调度者 >>> Dispatcher
			- 关系、创建的Request交给Dispatcher内部调度（队列机制、添加优先级）

- YTKNetwork套住的是AFNetworking，而AFNetworking在iOS7之前和iOS7之后使用的iOS系统api有区别
	- NSOperation + 单例NSThread开启NSRunLoop + NSURLConnection 
	- NSURLSession默认后台子线程执行、请求可以暂停和恢复

所以使用对于iOS7之后，使用NSURLSession完成网络操作比使用NSURLConnection强大多了.

- YTKNetwork仍然使用的是AFNetworking在iOS7以下的实现方式
	- NSURLConnection
	- 所以想按着YTKNetwork的套路，基于NSURLSession的AFNetworking版本实现进行封装

****

###AFJSONResponseSerializer源码中不支持 `text/html` 的response数据类型解析

OK，还是半年多之前看过AFNetworking基于NSOperation + NSURLConnection的实现过，那么接下来捋一捋基于NSURLSession的实现过程吧:


好吧，随手写了如下一段请求代码运行报错，卧槽....

```objc
{
    //普通GET DataTask数据请求
    NSString *url = @"http://php.weather.sina.com.cn/iframe/index/w_cl.php?code=js&day=0&city=&dfc=1&charset=utf-8";
    NSDictionary *param = @{@"key1" : @"value1",@"key2" : @"value2" };
    
    [[RDMySessionManager manager] GET:url parameters:param progress:nil
                              success:^(NSURLSessionDataTask * _Nonnull task, id  _Nullable responseObject) {
                                  NSLog(@"success>>> %@", responseObject);
                              } failure:^(NSURLSessionDataTask * _Nullable task, NSError * _Nonnull error) {
                                  NSLog(@"fail >>> %@", [error localizedDescription]);
                              }];
}
    
{
    // 使用NSURLSessionDataTask进行下载
    NSString *url = @"http://m3.pc6.com/xuh2/wimoweh.dmg";
    
    [[RDMySessionManager manager] GET:url parameters:nil progress:nil
                              success:^(NSURLSessionDataTask * _Nonnull task, id  _Nullable responseObject) {
                                  NSLog(@"success>>> %@", responseObject);
                              } failure:^(NSURLSessionDataTask * _Nullable task, NSError * _Nonnull error) {
                                  NSLog(@"fail >>> %@", [error localizedDescription]);
                              }];
}
    
{
    // 直接使用NSURLSessionDownloadTask进行下载
    NSString *url = @"http://m3.pc6.com/xuh2/wimoweh.dmg";
    NSURLRequest *request = [NSURLRequest requestWithURL:[NSURL URLWithString:url]];
    
    [[RDMySessionManager manager] downloadTaskWithRequest:request progress:^(NSProgress * _Nonnull downloadProgress) {
        
    } destination:^NSURL * _Nonnull(NSURL * _Nonnull targetPath, NSURLResponse * _Nonnull response) {
    	
    	//1. 将系统临时路径下的文件拷贝到指定目录，如果不拷贝会被自动删除
    	//...
    		
        //2. 返回自己存放下载文件的路径
        return targetPath;
    } completionHandler:^(NSURLResponse * _Nonnull response, NSURL * _Nullable filePath, NSError * _Nullable error) {
        
    }];
}
```

错误提示如下，最主要的一句是: `unacceptable content-type: text/html`，半年之前遇到过记得就是加上 text/html解析方式就OK了，但是忘记在哪里加了...

```
Error Domain=com.alamofire.error.serialization.response Code=-1016 "Request 
failed: unacceptable content-type: text/html" 
UserInfo={com.alamofire.serialization.response.error.response=<NSHTTPURLResponse: 0x7fed61f202a0> { URL: http://php.weather.sina.com.cn/iframe/index/w_cl.php?

....省略
```

> 注意，记录下免的又忘记了，AFN中对于请求参数组装和响应数据解析有不同的类型支持.

####请求参数组装的类型

- AFURLRequestSerialization 接口协议
- 实现一、AFHTTPRequestSerializer 使用key=value形式组装参数
- 实现二、AFJSONRequestSerializer 继承自 AFHTTPRequestSerializer，将参数转成json格式
- 实现三、AFPropertyListRequestSerializer 继承自 AFHTTPRequestSerializer，将参数转成xml格式

####响应数据的解析类型

- AFURLResponseSerialization 接口协议
- 实现一、AFHTTPResponseSerializer 将response data解析成二进制数据NSData类型
- 实现二、AFJSONResponseSerializer 继承自 AFHTTPResponseSerializer，将将response data解析成json
- 实现三、..... 将将response data解析成xml字符串
- 实现四、..... 将将response data解析成plist字符串
- 实现四、..... 将将response data解析成Image图像

网上搜了下，直接修改AFHTTPResponseSerializer的init的代码，添加text/html方式，如:

```
// 源码
self.acceptableContentTypes = [NSSet setWithObjects:@"application/json", @"text/json", @"text/javascript", nil];

// 修改后
self.acceptableContentTypes = [NSSet setWithObjects:@"application/json", @"text/json", @"text/javascript", @"text/html", nil];
```

我没有直接去修改源码，不太喜欢修改类库源码，尤其是pods下载的，修改后pods更新了可能修改的代码就没了，首先理一下思路:

- `[AFHTTPSessionManager manager]` 完成的事情:
	- 1、创建NSURLSession、operationQueue....
	- 2、创建`AFJSONResponseSerializer`作为默认的response data解析器
	- 3、创建AFHTTPRequestSerializer作为默认的请求参数组装器
	- 4、创建AFSecurityPolicy完成https安全校验
	- 5、创建AFNetworkReachabilityManager完成网络状态改变监测
	- 6、[self.session getTasksWithCompletionHandler..] 继续执行没有完成的Task任务
		- DataTask
		- UploadTask
		- DownloadTask

- 在第二步创建出来的AFJSONResponseSerializer实例，不支持text/html方式

- 想办法执行完[AFHTTPSessionManager manager]后，继续执行添加AFJSONResponseSerializer支持text/html方式
	- **必须在进行网络请求之前完成**

> 写一个AFHTTPSessionManager子类，重写manager方法即可.

```objc
#import <AFNetworking/AFNetworking.h>

@interface RDMySessionManager : AFHTTPSessionManager

@end
```

```objc
#import "RDMySessionManager.h"

@implementation RDMySessionManager

+ (instancetype)manager {
    
    // 调用AFHTTPSessionManager创建实例
    RDMySessionManager *manager = [super manager];
    
    // 创建一个支持text/html解析的response serializer
    // （其实AFHTTPResponseSerializer也可以子类化进行个性化定制）
    AFHTTPResponseSerializer *serial = [AFHTTPResponseSerializer serializer];
    serial.acceptableContentTypes = [NSSet setWithObjects:@"application/json", @"text/json", @"text/javascript", @"text/html", nil];
    
    //将response serializer 设置给 AFHTTPSessionManager实例
    manager.responseSerializer = serial;
    return manager;
}

@end
```

测试代码

```objc
NSString *url = @"http://php.weather.sina.com.cn/iframe/index/w_cl.php?code=js&day=0&city=&dfc=1&charset=utf-8";
    NSDictionary *param = @{@"key1" : @"value1",@"key2" : @"value2" };
    
[[RDMySessionManager manager] GET:url parameters:param progress:nil
 success:^(NSURLSessionDataTask * _Nonnull task, id  _Nullable responseObject) {
    NSLog(@"success>>> %@", responseObject);
} failure:^(NSURLSessionDataTask * _Nullable task, NSError * _Nonnull error) {
    NSLog(@"fail >>> %@", [error localizedDescription]);
}];
```

OK，流程可以走通了，接下来就是断点看看内部实现。如果还是失败，那么就是iOS9的ATS对于https的要求了，各位可以网上搜很简单...

注：如下直接贴源码，高手直接路过，我只是便于自己的随时查阅就直接贴源码.

****

###进入到AFHTTPSessionManager方法

- 发起GET请求的入口

```objc
- (NSURLSessionDataTask *)GET:(NSString *)URLString
                   parameters:(id)parameters
                     progress:(void (^)(NSProgress * _Nonnull))downloadProgress
                      success:(void (^)(NSURLSessionDataTask * _Nonnull, id _Nullable))success
                      failure:(void (^)(NSURLSessionDataTask * _Nullable, NSError * _Nonnull))failure
{

	// 继续调用自己的其他方法完成创建 NSURLSessionDataTask
    NSURLSessionDataTask *dataTask = [self dataTaskWithHTTPMethod:@"GET"
                                                        URLString:URLString
                                                       parameters:parameters
                                                   uploadProgress:nil
                                                 downloadProgress:downloadProgress
                                                          success:success
                                                          failure:failure];

	// 启动task执行
    [dataTask resume];

    return dataTask;
}
```

- 所有GET/POST/PUT/DELETE类型的Task创建的统一入口

```objc
- (NSURLSessionDataTask *)dataTaskWithHTTPMethod:(NSString *)method
                                       URLString:(NSString *)URLString
                                      parameters:(id)parameters
                                  uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgress
                                downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgress
                                         success:(void (^)(NSURLSessionDataTask *, id))success
                                         failure:(void (^)(NSURLSessionDataTask *, NSError *))failure
{
    NSError *serializationError = nil;
    
    // 调用AFHTTPRequestSerializer完成NSMutableURLRequest的创建（1.AFHTTPRequestSerializer完成参数拼接 2.用户自己完成参数拼接）
    NSMutableURLRequest *request = [self.requestSerializer requestWithMethod:method URLString:[[NSURL URLWithString:URLString relativeToURL:self.baseURL] absoluteString] parameters:parameters error:&serializationError];
    
    // 参数拼接如果出错
    if (serializationError) {
        if (failure) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wgnu"

			// 执行错误回调，结束Task执行
            dispatch_async(self.completionQueue ?: dispatch_get_main_queue(), ^{
                failure(nil, serializationError);
            });
#pragma clang diagnostic pop
        }

        return nil;
    }

	// 调用AFURLSessionManager完成创建Task，并传入所有的回调block
    __block NSURLSessionDataTask *dataTask = nil;
    dataTask = [self dataTaskWithRequest:request
                          uploadProgress:uploadProgress
                        downloadProgress:downloadProgress
                       completionHandler:^(NSURLResponse * __unused response, id responseObject, NSError *error) {
        if (error) {
            if (failure) {
                failure(dataTask, error);
            }
        } else {
            if (success) {
                success(dataTask, responseObject);
            }
        }
    }];

    return dataTask;
}
```

###AFURLSessionManager的方法使用NSURLSession创建出Task

> 此处有一个iOS7的系统bug，iOS8+已经修复.

```
NSURLSessionTask, creating tasks on a concurrent queue can cause incorrect completionHandlers to get called

（在并发环境时创建Task可能造成获取不到对应的completionHandlers）

https://github.com/AFNetworking/AFNetworking/issues/2093


```


```objc
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                               uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                             downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                            completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject,  NSError * _Nullable error))completionHandler {

    __block NSURLSessionDataTask *dataTask = nil;
    
    // 在并发环境时创建Task可能造成获取不到对应的completionHandlers
    url_session_manager_create_task_safely(^{
        dataTask = [self.session dataTaskWithRequest:request];
    });

	// 给Task分配一个TaskDelegate，负责其所有的NSURLSessionDelegate回调函数实现
    [self addDelegateForDataTask:dataTask uploadProgress:uploadProgressBlock downloadProgress:downloadProgressBlock completionHandler:completionHandler];

    return dataTask;
}
```

```objc
static void url_session_manager_create_task_safely(dispatch_block_t block) {

	// 如果是iOS7.x时，使用串行队列同步执行NSURLSession创建Task的操作
	// 如果是iOS8.x时，随便....
    if (NSFoundationVersionNumber < NSFoundationVersionNumber_With_Fixed_5871104061079552_bug) {
        // Fix of bug
        // Open Radar:http://openradar.appspot.com/radar?id=5871104061079552 (status: Fixed in iOS8)
        // Issue about:https://github.com/AFNetworking/AFNetworking/issues/2093
        dispatch_sync(url_session_manager_creation_queue(), block);
    } else {
        block();
    }
}
```

```objc
static dispatch_queue_t url_session_manager_creation_queue() {
    static dispatch_queue_t af_url_session_manager_creation_queue;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        af_url_session_manager_creation_queue = dispatch_queue_create("com.alamofire.networking.session.manager.creation", DISPATCH_QUEUE_SERIAL);
    });

    return af_url_session_manager_creation_queue;
}
```

###AFURLSessionManager的方法完成给NSURLSessionDataTask添加一对一的AFURLSessionManagerTaskDelegate对象完成其所有的回调函数实现

```objc
- (void)addDelegateForDataTask:(NSURLSessionDataTask *)dataTask
                uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
              downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
             completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
	//1. 实例化一个新的TaskDelegate
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] init];
    
    //2. 指定TaskDelegate属于哪一个AFSessionManager
    delegate.manager = self;
    
    //3. TaskDelegate保存请求完毕的success和fail 两个block
    delegate.completionHandler = completionHandler;

	//4. 设置Task的描述
    dataTask.taskDescription = self.taskDescriptionForSessionTasks;
    
    //5. Task与TaskDelegate建立关系
    [self setDelegate:delegate forTask:dataTask];

	//6. 设置上传下载回调block
    delegate.uploadProgressBlock = uploadProgressBlock;
    delegate.downloadProgressBlock = downloadProgressBlock;
}

// 获取Task的描述 >>> 当前Task对象的地址
- (NSString *)taskDescriptionForSessionTasks {
    return [NSString stringWithFormat:@"%p", self];
}

// Task与TaskDelegate建立关系
- (void)setDelegate:(AFURLSessionManagerTaskDelegate *)delegate
            forTask:(NSURLSessionTask *)task
{
    NSParameterAssert(task);
    NSParameterAssert(delegate);

    [self.lock lock];
    
    //1. 使用一个属性mutable dic来保存TaskDelegate于Task的对应关系
    self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)] = delegate;
    
    //2. 给TaskDelegate设置一个NSProgress用来跟踪Task的完成进度
    [delegate setupProgressForTask:task];
    
    //3. 添加对Task关注的通知
    [self addNotificationObserverForTask:task];


    [self.lock unlock];
}
```

关注Task发送的通知

```objc
- (void)addNotificationObserverForTask:(NSURLSessionTask *)task {
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(taskDidResume:) name:AFNSURLSessionTaskDidResumeNotification object:task];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(taskDidSuspend:) name:AFNSURLSessionTaskDidSuspendNotification object:task];
}
```

> 使用NSProgress来跟踪Task的执行进度。（重新写了一篇文章记录学习NSProgress这个类）




上面是DataTask添加delegate，还有给UploadTask和DownloadTask分别添加delegate的操作

```objc
- (void)addDelegateForUploadTask:(NSURLSessionUploadTask *)uploadTask
                        progress:(void (^)(NSProgress *uploadProgress)) uploadProgressBlock
               completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] init];
    delegate.manager = self;
    delegate.completionHandler = completionHandler;

    uploadTask.taskDescription = self.taskDescriptionForSessionTasks;

    [self setDelegate:delegate forTask:uploadTask];

    delegate.uploadProgressBlock = uploadProgressBlock;
}

- (void)addDelegateForDownloadTask:(NSURLSessionDownloadTask *)downloadTask
                          progress:(void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                       destination:(NSURL * (^)(NSURL *targetPath, NSURLResponse *response))destination
                 completionHandler:(void (^)(NSURLResponse *response, NSURL *filePath, NSError *error))completionHandler
{
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] init];
    delegate.manager = self;
    delegate.completionHandler = completionHandler;

    if (destination) {
        delegate.downloadTaskDidFinishDownloading = ^NSURL * (NSURLSession * __unused session, NSURLSessionDownloadTask *task, NSURL *location) {
            return destination(location, task.response);
        };
    }

    downloadTask.taskDescription = self.taskDescriptionForSessionTasks;

    [self setDelegate:delegate forTask:downloadTask];

    delegate.downloadProgressBlock = downloadProgressBlock;
}
```

移除Task对应的TaskDelegate


```objc
- (void)removeDelegateForTask:(NSURLSessionTask *)task {
    NSParameterAssert(task);

    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:task];
    [self.lock lock];
    [delegate cleanUpProgressForTask:task];
    [self removeNotificationObserverForTask:task];
    [self.mutableTaskDelegatesKeyedByTaskIdentifier removeObjectForKey:@(task.taskIdentifier)];
    [self.lock unlock];
}
```

****

###AFURLSessionManagerTaskDelegate对象与NSURLSessionTask对象的映射关系建立

- 建立Task对象与TaskDelegate对象的映射

```objc
- (void)setDelegate:(AFURLSessionManagerTaskDelegate *)delegate
            forTask:(NSURLSessionTask *)task
{
    NSParameterAssert(task);
    NSParameterAssert(delegate);

    [self.lock lock];
    
	// 使用成员对象可变字典来保存关系
	self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)] = delegate;
    [delegate setupProgressForTask:task];
    [self addNotificationObserverForTask:task];
    [self.lock unlock];
}
```

- 查找Task对象对应的TaskDelegate对象

```objc
- (AFURLSessionManagerTaskDelegate *)delegateForTask:(NSURLSessionTask *)task {
    NSParameterAssert(task);

    AFURLSessionManagerTaskDelegate *delegate = nil;
    [self.lock lock];
    delegate = self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)];
    [self.lock unlock];

    return delegate;
}
```

###看看 AFURLSessionManagerTaskDelegate 负责实现NSURLSessionTaskDelegate的一些回调函数

> 这么做的好处: 不用分散NSURLSessionTaskDelegate所有的函数实现，以及一个Task可以对应的一个TaskDelegate.

###AFURLSessionManager.m中有个私有类 `_AFURLSessionTaskSwizzling`，主要用来拦截NSURLSessionTask对象的resume消息、suspend消息的发送，然后发送对应的通知.

> 注意: NSURLSessionTask这个类在iOS8 SDK才有，所以进行hook拦截NSURLSessionTask的工作也只能在iOS8及之后的SDK进行操作.

下图是从对应issue中截图，展示iOS7与iOS8之后的，NSURLSessionTask体系的区别:

[![Snip20160519_4.png](http://imgchr.com/images/Snip20160519_4.png)](http://imgchr.com/image/PuZ)

```objc
@interface _AFURLSessionTaskSwizzling : NSObject

@end

@implementation _AFURLSessionTaskSwizzling

+ (void)load {
    /**
     WARNING: Trouble Ahead
     https://github.com/AFNetworking/AFNetworking/pull/2702
     */

	// 只在iOS8及以上的SDk环境进行hook
    if (NSClassFromString(@"NSURLSessionTask")) {
        /**
         iOS 7 and iOS 8 differ in NSURLSessionTask implementation, which makes the next bit of code a bit tricky.
         Many Unit Tests have been built to validate as much of this behavior has possible.
         Here is what we know:
            - NSURLSessionTasks are implemented with class clusters, meaning the class you request from the API isn't actually the type of class you will get back.
            - Simply referencing `[NSURLSessionTask class]` will not work. You need to ask an `NSURLSession` to actually create an object, and grab the class from there.
            - On iOS 7, `localDataTask` is a `__NSCFLocalDataTask`, which inherits from `__NSCFLocalSessionTask`, which inherits from `__NSCFURLSessionTask`.
            - On iOS 8, `localDataTask` is a `__NSCFLocalDataTask`, which inherits from `__NSCFLocalSessionTask`, which inherits from `NSURLSessionTask`.
            - On iOS 7, `__NSCFLocalSessionTask` and `__NSCFURLSessionTask` are the only two classes that have their own implementations of `resume` and `suspend`, and `__NSCFLocalSessionTask` DOES NOT CALL SUPER. This means both classes need to be swizzled.
            - On iOS 8, `NSURLSessionTask` is the only class that implements `resume` and `suspend`. This means this is the only class that needs to be swizzled.
            - Because `NSURLSessionTask` is not involved in the class hierarchy for every version of iOS, its easier to add the swizzled methods to a dummy class and manage them there.
        
         Some Assumptions:
            - No implementations of `resume` or `suspend` call super. If this were to change in a future version of iOS, we'd need to handle it.
            - No background task classes override `resume` or `suspend`
         
         The current solution:
            1) Grab an instance of `__NSCFLocalDataTask` by asking an instance of `NSURLSession` for a data task.
            2) Grab a pointer to the original implementation of `af_resume`
            3) Check to see if the current class has an implementation of resume. If so, continue to step 4.
            4) Grab the super class of the current class.
            5) Grab a pointer for the current class to the current implementation of `resume`.
            6) Grab a pointer for the super class to the current implementation of `resume`.
            7) If the current class implementation of `resume` is not equal to the super class implementation of `resume` AND the current implementation of `resume` is not equal to the original implementation of `af_resume`, THEN swizzle the methods
            8) Set the current class to the super class, and repeat steps 3-8
         */
        NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration ephemeralSessionConfiguration];
        NSURLSession * session = [NSURLSession sessionWithConfiguration:configuration];
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wnonnull"
        NSURLSessionDataTask *localDataTask = [session dataTaskWithURL:nil];
#pragma clang diagnostic pop
        IMP originalAFResumeIMP = method_getImplementation(class_getInstanceMethod([self class], @selector(af_resume)));
        Class currentClass = [localDataTask class];
        
        while (class_getInstanceMethod(currentClass, @selector(resume))) {
            Class superClass = [currentClass superclass];
            IMP classResumeIMP = method_getImplementation(class_getInstanceMethod(currentClass, @selector(resume)));
            IMP superclassResumeIMP = method_getImplementation(class_getInstanceMethod(superClass, @selector(resume)));
            if (classResumeIMP != superclassResumeIMP &&
                originalAFResumeIMP != classResumeIMP) {
                [self swizzleResumeAndSuspendMethodForClass:currentClass];
            }
            currentClass = [currentClass superclass];
        }
        
        [localDataTask cancel];
        [session finishTasksAndInvalidate];
    }
}

+ (void)swizzleResumeAndSuspendMethodForClass:(Class)theClass {
    Method afResumeMethod = class_getInstanceMethod(self, @selector(af_resume));
    Method afSuspendMethod = class_getInstanceMethod(self, @selector(af_suspend));

    if (af_addMethod(theClass, @selector(af_resume), afResumeMethod)) {
        af_swizzleSelector(theClass, @selector(resume), @selector(af_resume));
    }

    if (af_addMethod(theClass, @selector(af_suspend), afSuspendMethod)) {
        af_swizzleSelector(theClass, @selector(suspend), @selector(af_suspend));
    }
}

- (NSURLSessionTaskState)state {
    NSAssert(NO, @"State method should never be called in the actual dummy class");
    return NSURLSessionTaskStateCanceling;
}

- (void)af_resume {
    NSAssert([self respondsToSelector:@selector(state)], @"Does not respond to state");
    NSURLSessionTaskState state = [self state];
    [self af_resume];
    
    if (state != NSURLSessionTaskStateRunning) {
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNSURLSessionTaskDidResumeNotification object:self];
    }
}

- (void)af_suspend {
    NSAssert([self respondsToSelector:@selector(state)], @"Does not respond to state");
    NSURLSessionTaskState state = [self state];
    [self af_suspend];
    
    if (state != NSURLSessionTaskStateSuspended) {
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNSURLSessionTaskDidSuspendNotification object:self];
    }
}
@end
```

如上代码主要完成的事情:

- 交换NSURLSessionTask、以及NSURLSessionTask所有父类的resume方法实现、suspend方法实现

- 达到hook之后，发送对应的通知

```objc
[[NSNotificationCenter defaultCenter] postNotificationName:AFNSURLSessionTaskDidResumeNotification object:self];
```

```objc
[[NSNotificationCenter defaultCenter] postNotificationName:AFNSURLSessionTaskDidSuspendNotification object:self];
```

从这里也可以得到一些iOS版本判断的小技巧:

- 第一种、NSClassFromString(@"对应iOS SDK版本才有的类")

```objc
if (NSClassFromString(@"NSURLSessionTask")) {
	// iOS8 及以上版本
} else {
	// iOS8以下
}
```

- 第二种、[对象 responseToSelector:] or class_respondsToMethod(<#__unsafe_unretained Class cls#>, <#SEL sel#>)

- 第三种、NSFoundationVersionNumber < NSFoundationVersionNumber_iOS_8_0

- 第四种、max_allowed宏比较low了

####补充关于load方法 >>> 只要 xxx.m 文件被读取时就会被调用，且该 xxx.m 中所有的类的 load方法 都会被调用，且按照类的定义从上到下的顺序调用

```objc
#import "ViewController.h"

@interface _MyClass : NSObject

@end

@implementation _MyClass

+ (void)load {
    NSLog(@"%@", NSStringFromClass([self class]));
}

@end


@interface ViewController ()

@end

@implementation ViewController

+ (void)load {
    NSLog(@"%@", NSStringFromClass([self class]));
}

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
}

@end
```

输出信息

```
2016-05-09 21:26:16.238 Demos[36425:484470] _MyClass
2016-05-09 21:26:16.239 Demos[36425:484470] ViewController
```

对上面的类定义换下顺序后为如下

```objc
#import "ViewController.h"

@interface ViewController ()

@end

@implementation ViewController

+ (void)load {
    NSLog(@"%@", NSStringFromClass([self class]));
}

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
}

@end

@interface _MyClass : NSObject

@end

@implementation _MyClass

+ (void)load {
    NSLog(@"%@", NSStringFromClass([self class]));
}

@end
```

输出信息

```
2016-05-09 21:28:18.303 Demos[36554:487713] ViewController
2016-05-09 21:28:18.304 Demos[36554:487713] _MyClass
```

OK，是按照类定义的从上至下的顺序依次调用所有类的load方法，只会调用一次.

****


###AFURLSessionManagerTaskDelegate

```objc
@interface AFURLSessionManagerTaskDelegate : NSObject <NSURLSessionTaskDelegate, NSURLSessionDataDelegate, NSURLSessionDownloadDelegate>

// SessionManager
@property (nonatomic, weak) AFURLSessionManager *manager;

// 下载的数据
@property (nonatomic, strong) NSMutableData *mutableData;

// 上传和下载进度管理
@property (nonatomic, strong) NSProgress *uploadProgress;
@property (nonatomic, strong) NSProgress *downloadProgress;

// 下载完成后保存数据的临时文件位置
@property (nonatomic, copy) NSURL *downloadFileURL;

// 各种回调block
@property (nonatomic, copy) AFURLSessionDownloadTaskDidFinishDownloadingBlock downloadTaskDidFinishDownloading;
@property (nonatomic, copy) AFURLSessionTaskProgressBlock uploadProgressBlock;
@property (nonatomic, copy) AFURLSessionTaskProgressBlock downloadProgressBlock;
@property (nonatomic, copy) AFURLSessionTaskCompletionHandler completionHandler;

@end
```

- 实现了 NSURLSessionTaskDelegate, NSURLSessionDataDelegate, NSURLSessionDownloadDelegate 三个协议

- NSURLSessionTaskDelegate: 
	- 重定向回调
	- 接收服务器挑战回调
	- 上传数据回调
	- 请求结束回调

```objc
/*
 * Messages related to the operation of a specific task.
 */
@protocol NSURLSessionTaskDelegate <NSURLSessionDelegate>
@optional

/* An HTTP request is attempting to perform a redirection to a different
 * URL. You must invoke the completion routine to allow the
 * redirection, allow the redirection with a modified request, or
 * pass nil to the completionHandler to cause the body of the redirection 
 * response to be delivered as the payload of this request. The default
 * is to follow redirections. 
 *
 * For tasks in background sessions, redirections will always be followed and this method will not be called.
 */
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
                     willPerformHTTPRedirection:(NSHTTPURLResponse *)response
                                     newRequest:(NSURLRequest *)request
                              completionHandler:(void (^)(NSURLRequest * __nullable))completionHandler;

/* The task has received a request specific authentication challenge.
 * If this delegate is not implemented, the session specific authentication challenge
 * will *NOT* be called and the behavior will be the same as using the default handling
 * disposition. 
 */
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
                            didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge 
                              completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential * __nullable credential))completionHandler;

/* Sent if a task requires a new, unopened body stream.  This may be
 * necessary when authentication has failed for any request that
 * involves a body stream. 
 */
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
                              needNewBodyStream:(void (^)(NSInputStream * __nullable bodyStream))completionHandler;

/* Sent periodically to notify the delegate of upload progress.  This
 * information is also available as properties of the task.
 */
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
                                didSendBodyData:(int64_t)bytesSent
                                 totalBytesSent:(int64_t)totalBytesSent
                       totalBytesExpectedToSend:(int64_t)totalBytesExpectedToSend;

/* Sent as the last message related to a specific task.  Error may be
 * nil, which implies that no error occurred and this task is complete. 
 */
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
                           didCompleteWithError:(nullable NSError *)error;

@end
```

- NSURLSessionDataDelegate: 
	- 接收到response描述回调
	- DataTask转换成DownloadTask回调
	- 接收到response data数据回调
	- 缓存response回调

```objc
/*
 * Messages related to the operation of a task that delivers data
 * directly to the delegate.
 */
@protocol NSURLSessionDataDelegate <NSURLSessionTaskDelegate>
@optional
/* The task has received a response and no further messages will be
 * received until the completion block is called. The disposition
 * allows you to cancel a request or to turn a data task into a
 * download task. This delegate message is optional - if you do not
 * implement it, you can get the response as a property of the task.
 *
 * This method will not be called for background upload tasks (which cannot be converted to download tasks).
 */
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
                                 didReceiveResponse:(NSURLResponse *)response
                                  completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler;

/* Notification that a data task has become a download task.  No
 * future messages will be sent to the data task.
 */
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
                              didBecomeDownloadTask:(NSURLSessionDownloadTask *)downloadTask;

/*
 * Notification that a data task has become a bidirectional stream
 * task.  No future messages will be sent to the data task.  The newly
 * created streamTask will carry the original request and response as
 * properties.
 *
 * For requests that were pipelined, the stream object will only allow
 * reading, and the object will immediately issue a
 * -URLSession:writeClosedForStream:.  Pipelining can be disabled for
 * all requests in a session, or by the NSURLRequest
 * HTTPShouldUsePipelining property.
 *
 * The underlying connection is no longer considered part of the HTTP
 * connection cache and won't count against the total number of
 * connections per host.
 */
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
                                didBecomeStreamTask:(NSURLSessionStreamTask *)streamTask;

/* Sent when data is available for the delegate to consume.  It is
 * assumed that the delegate will retain and not copy the data.  As
 * the data may be discontiguous, you should use 
 * [NSData enumerateByteRangesUsingBlock:] to access it.
 */
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
                                     didReceiveData:(NSData *)data;

/* Invoke the completion routine with a valid NSCachedURLResponse to
 * allow the resulting data to be cached, or pass nil to prevent
 * caching. Note that there is no guarantee that caching will be
 * attempted for a given resource, and you should not rely on this
 * message to receive the resource data.
 */
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
                                  willCacheResponse:(NSCachedURLResponse *)proposedResponse 
                                  completionHandler:(void (^)(NSCachedURLResponse * __nullable cachedResponse))completionHandler;

@end
```

- NSURLSessionDownloadDelegate:
	- 数据下载完毕回调
	- 接收下载进度回调
	- 断点下载回调

	
```objc
/*
 * Messages related to the operation of a task that writes data to a
 * file and notifies the delegate upon completion.
 */
@protocol NSURLSessionDownloadDelegate <NSURLSessionTaskDelegate>

/* Sent when a download task that has completed a download.  The delegate should 
 * copy or move the file at the given location to a new location as it will be 
 * removed when the delegate message returns. URLSession:task:didCompleteWithError: will
 * still be called.
 */
- (void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask
                              didFinishDownloadingToURL:(NSURL *)location;

@optional
/* Sent periodically to notify the delegate of download progress. */
- (void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask
                                           didWriteData:(int64_t)bytesWritten
                                      totalBytesWritten:(int64_t)totalBytesWritten
                              totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite;

/* Sent when a download has been resumed. If a download failed with an
 * error, the -userInfo dictionary of the error will contain an
 * NSURLSessionDownloadTaskResumeData key, whose value is the resume
 * data. 
 */
- (void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask
                                      didResumeAtOffset:(int64_t)fileOffset
                                     expectedTotalBytes:(int64_t)expectedTotalBytes;

@end
```

- AFURLSessionManagerTaskDelegate中对NSURLSessionTask对象属性的KVO监测、NSProgress进度的KVO监测以及NSProgress进度`初始化`

```objc
- (void)setupProgressForTask:(NSURLSessionTask *)task {
    __weak __typeof__(task) weakTask = task;

	//1. 初始化NSProgress
    self.uploadProgress.totalUnitCount = task.countOfBytesExpectedToSend;
    self.downloadProgress.totalUnitCount = task.countOfBytesExpectedToReceive;
    [self.uploadProgress setCancellable:YES];
    [self.uploadProgress setCancellationHandler:^{
        __typeof__(weakTask) strongTask = weakTask;
        [strongTask cancel];
    }];
    [self.uploadProgress setPausable:YES];
    [self.uploadProgress setPausingHandler:^{
        __typeof__(weakTask) strongTask = weakTask;
        [strongTask suspend];
    }];
    if ([self.uploadProgress respondsToSelector:@selector(setResumingHandler:)]) {
        [self.uploadProgress setResumingHandler:^{
            __typeof__(weakTask) strongTask = weakTask;
            [strongTask resume];
        }];
    }

    [self.downloadProgress setCancellable:YES];
    [self.downloadProgress setCancellationHandler:^{
        __typeof__(weakTask) strongTask = weakTask;
        [strongTask cancel];
    }];
    [self.downloadProgress setPausable:YES];
    [self.downloadProgress setPausingHandler:^{
        __typeof__(weakTask) strongTask = weakTask;
        [strongTask suspend];
    }];

    if ([self.downloadProgress respondsToSelector:@selector(setResumingHandler:)]) {
        [self.downloadProgress setResumingHandler:^{
            __typeof__(weakTask) strongTask = weakTask;
            [strongTask resume];
        }];
    }

	//2. 监听Task对象属性值改变
    [task addObserver:self
           forKeyPath:NSStringFromSelector(@selector(countOfBytesReceived))
              options:NSKeyValueObservingOptionNew
              context:NULL];
    [task addObserver:self
           forKeyPath:NSStringFromSelector(@selector(countOfBytesExpectedToReceive))
              options:NSKeyValueObservingOptionNew
              context:NULL];

    [task addObserver:self
           forKeyPath:NSStringFromSelector(@selector(countOfBytesSent))
              options:NSKeyValueObservingOptionNew
              context:NULL];
    [task addObserver:self
           forKeyPath:NSStringFromSelector(@selector(countOfBytesExpectedToSend))
              options:NSKeyValueObservingOptionNew
              context:NULL];

	//3. 监听NSProgress对象的进度值改变
    [self.downloadProgress addObserver:self
                            forKeyPath:NSStringFromSelector(@selector(fractionCompleted))
                               options:NSKeyValueObservingOptionNew
                               context:NULL];
    [self.uploadProgress addObserver:self
                          forKeyPath:NSStringFromSelector(@selector(fractionCompleted))
                             options:NSKeyValueObservingOptionNew
                             context:NULL];
}
```

- AFURLSessionManagerTaskDelegate中监听到KVO通知回调函数做的事情

```objc
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSString *,id> *)change context:(void *)context {
    
    if ([object isKindOfClass:[NSURLSessionTask class]] || [object isKindOfClass:[NSURLSessionDownloadTask class]])
    {
        // 被修改的是 Task对象
        if ([keyPath isEqualToString:NSStringFromSelector(@selector(countOfBytesReceived))]) {
            self.downloadProgress.completedUnitCount = [change[NSKeyValueChangeNewKey] longLongValue];
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(countOfBytesExpectedToReceive))])
         {
         	// Task对象的countOfBytesExpectedToReceive属性值被修改
            self.downloadProgress.totalUnitCount = [change[NSKeyValueChangeNewKey] longLongValue];
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(countOfBytesSent))]) 
        {
        	// Task对象的countOfBytesSent属性值被修改
            self.uploadProgress.completedUnitCount = [change[NSKeyValueChangeNewKey] longLongValue];
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(countOfBytesExpectedToSend))]) 
        {
			// Task对象的countOfBytesExpectedToSend属性值被修改
            self.uploadProgress.totalUnitCount = [change[NSKeyValueChangeNewKey] longLongValue];
        }
    }
    else if ([object isEqual:self.downloadProgress]) {
        // 被修改的是 下载progress对象
        if (self.downloadProgressBlock) {
            self.downloadProgressBlock(object);
        }
    }
    else if ([object isEqual:self.uploadProgress]) {
        // 被修改的是 上传progress对象
        if (self.uploadProgressBlock) {
            self.uploadProgressBlock(object);
        }
    }
}
```


主要就是监测到属性值修改之后，做出一些回调处理:

```
1. Task的属性值被修改、去修改NSProgress的总工作量、已经完成量
2. NSProgress进度值修改、直接执行回调block，回传进度值
```

- 实现 `NSURLSessionTaskDelegate`中，`请求结束` 的回调delegate函数

```objc
- (void)URLSession:(__unused NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error
{
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wgnu"
    __strong AFURLSessionManager *manager = self.manager;

    __block id responseObject = nil;

    __block NSMutableDictionary *userInfo = [NSMutableDictionary dictionary];
    userInfo[AFNetworkingTaskDidCompleteResponseSerializerKey] = manager.responseSerializer;

    // 拷贝一份下载数据data，并释放持有的data
    NSData *data = nil;
    if (self.mutableData) {
        data = [self.mutableData copy];
        //We no longer need the reference, so nil it out to gain back some memory.
        self.mutableData = nil;
    }

    // 判断下载完毕是否写入临时文件
    if (self.downloadFileURL) {
        // 如果有，回传临时文件路径
        userInfo[AFNetworkingTaskDidCompleteAssetPathKey] = self.downloadFileURL;
    } else if (data) {
        // 如果没有，回传下载的data
        userInfo[AFNetworkingTaskDidCompleteResponseDataKey] = data;
    }

    // 判断是否发生错误
    if (error) {
        
        // 回传错误
        userInfo[AFNetworkingTaskDidCompleteErrorKey] = error;

        // group async 回调Task结束
        dispatch_group_async(manager.completionGroup ?: url_session_manager_completion_group(), manager.completionQueue ?: dispatch_get_main_queue(), ^{
            
            // 执行回调block，回传数据
            if (self.completionHandler) {
                self.completionHandler(task.response, responseObject, error);
            }

            // 发送Task结束通知
            dispatch_async(dispatch_get_main_queue(), ^{
                [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingTaskDidCompleteNotification object:task userInfo:userInfo];
            });
        });
    } else {
        //
        dispatch_async(url_session_manager_processing_queue(), ^{
            
            // 二进制respoonse data >>> 使用SessionManager注册的response serializer解析成对应的响应数据
            NSError *serializationError = nil;
            responseObject = [manager.responseSerializer responseObjectForResponse:task.response data:data error:&serializationError];

            // 如果是下载数据，responseObject就是下载数据之后的临时文件路径
            if (self.downloadFileURL) {
                responseObject = self.downloadFileURL;
            }

            if (responseObject) {
                userInfo[AFNetworkingTaskDidCompleteSerializedResponseKey] = responseObject;
            }

            if (serializationError) {
                userInfo[AFNetworkingTaskDidCompleteErrorKey] = serializationError;
            }

            // 将执行回调block和发送通知使用group结合在一起执行
            dispatch_group_async(manager.completionGroup ?: url_session_manager_completion_group(), manager.completionQueue ?: dispatch_get_main_queue(), ^{
                
                if (self.completionHandler) {
                    self.completionHandler(task.response, responseObject, serializationError);
                }

                dispatch_async(dispatch_get_main_queue(), ^{
                    [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingTaskDidCompleteNotification object:task userInfo:userInfo];
                });
            });
        });
    }
#pragma clang diagnostic pop
}
```

- 实现`NSURLSessionDataDelegate`中， `接收到响应数据data` 的回调delegate函数

```objc
- (void)URLSession:(__unused NSURLSession *)session
          dataTask:(__unused NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data
{
	// 不断的拼接下载得到的部分数据
    [self.mutableData appendData:data];
}
```

- 实现`NSURLSessionDownloadDelegate`中， `下载完毕` 的回调delegate函数

> 注意需要调用 -[AFURLSessionManager downloadTaskWithRequest:progress:destination:completionHandler:] 方法创建NSURLSessionDownloadTask，才会触发这个delegate函数

```objc
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
didFinishDownloadingToURL:(NSURL *)location
{
    NSError *fileManagerError = nil;
    
    // 清空之前一次下载数据临时路径
    self.downloadFileURL = nil;

    // 执行设置的下载完毕的block
    if (self.downloadTaskDidFinishDownloading) {//必须执行 -[AFURLSessionManager downloadTaskWithRequest:progress:destination:completionHandler:] ，这个block才会存在
        
        // 回传系统存放下载文件的路径，框架调用者可以自己修改，并将临时文件保存到其他路径下
        self.downloadFileURL = self.downloadTaskDidFinishDownloading(session, downloadTask, location);
        
        // 下载完后的清除操作
        if (self.downloadFileURL) {
            
            // 移除临时文件
            [[NSFileManager defaultManager] moveItemAtURL:location toURL:self.downloadFileURL error:&fileManagerError];

            // 发送通知临时文件已经被移除
            if (fileManagerError) {
                [[NSNotificationCenter defaultCenter] postNotificationName:AFURLSessionDownloadTaskDidFailToMoveFileNotification object:downloadTask userInfo:fileManagerError.userInfo];
            }
        }
    }
}
```

OK，AFURLSessionManagerTaskDelegate就做了着一些事情，AFURLSessionManagerTaskDelegate观测了Task对象属性值修改、观测NSProgress属性值修改并回传进度值、实现了三个session delegate中`部分（必须实现）`的回调函数，那么大部分session delegate回调实现还是放在URLSessionManager中.

****

###AFURLSessionManager、从NSURLSession获取所有的类型的Task

```objc

/**
 The data, upload, and download tasks currently run by the managed session.
 */
@property (readonly, nonatomic, strong) NSArray <NSURLSessionTask *> *tasks;

/**
 The data tasks currently run by the managed session.
 */
@property (readonly, nonatomic, strong) NSArray <NSURLSessionDataTask *> *dataTasks;

/**
 The upload tasks currently run by the managed session.
 */
@property (readonly, nonatomic, strong) NSArray <NSURLSessionUploadTask *> *uploadTasks;

/**
 The download tasks currently run by the managed session.
 */
@property (readonly, nonatomic, strong) NSArray <NSURLSessionDownloadTask *> *downloadTasks;
```

```objc
- (NSArray *)tasksForKeyPath:(NSString *)keyPath {
    __block NSArray *tasks = nil;
    
    // 信号量0
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    
    // 从NSURLSession获取所有的类型的Task
    [self.session getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) {
        if ([keyPath isEqualToString:NSStringFromSelector(@selector(dataTasks))]) {
            // 数据请求
            tasks = dataTasks;
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(uploadTasks))]) {
            // 上传请求
            tasks = uploadTasks;
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(downloadTasks))]) {
            // 下载请求
            tasks = downloadTasks;
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(tasks))]) {
            // 合并所有类型的请求
            tasks = [@[dataTasks, uploadTasks, downloadTasks] valueForKeyPath:@"@unionOfArrays.self"];
        }

        // 信号量1
        dispatch_semaphore_signal(semaphore);
    }];

    // 一直等待信号，否则一直卡在这里，不会return返回
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);

    return tasks;
}

- (NSArray *)tasks {
    return [self tasksForKeyPath:NSStringFromSelector(_cmd)];
}

- (NSArray *)dataTasks {
    return [self tasksForKeyPath:NSStringFromSelector(_cmd)];
}

- (NSArray *)uploadTasks {
    return [self tasksForKeyPath:NSStringFromSelector(_cmd)];
}

- (NSArray *)downloadTasks {
    return [self tasksForKeyPath:NSStringFromSelector(_cmd)];
}
```

###AFURLSessionManager、取消session执行所有的tasks

```objc
- (void)invalidateSessionCancelingTasks:(BOOL)cancelPendingTasks;
```

###AFURLSessionManager、创建 Data Task 网络请求

```objc
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                            completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject,  NSError * _Nullable error))completionHandler;
```

###AFURLSessionManager、创建 Upload Task 网络请求

- 传入上传文件名

```objc
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request
                                         fromFile:(NSURL *)fileURL
                                         progress:(nullable void (^)(NSProgress *uploadProgress))uploadProgressBlock
                                completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject, NSError  * _Nullable error))completionHandler;
```

- 传入上传文件NSData

```objc
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request
                                         fromData:(nullable NSData *)bodyData
                                         progress:(nullable void (^)(NSProgress *uploadProgress))uploadProgressBlock
                                completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject, NSError * _Nullable error))completionHandler;
```

###AFURLSessionManager、创建 Download Task 网络请求

- 一般下载任务

```objc
- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request
                                             progress:(nullable void (^)(NSProgress *downloadProgress))downloadProgressBlock
                                          destination:(nullable NSURL * (^)(NSURL *targetPath, NSURLResponse *response))destination
                                    completionHandler:(nullable void (^)(NSURLResponse *response, NSURL * _Nullable filePath, NSError * _Nullable error))completionHandler;
```

- 断点下载任务

```objc
- (NSURLSessionDownloadTask *)downloadTaskWithResumeData:(NSData *)resumeData
                                                progress:(nullable void (^)(NSProgress *downloadProgress))downloadProgressBlock
                                             destination:(nullable NSURL * (^)(NSURL *targetPath, NSURLResponse *response))destination
                                       completionHandler:(nullable void (^)(NSURLResponse *response, NSURL * _Nullable filePath, NSError * _Nullable error))completionHandler;
```

###AFURLSessionManager、最后就是实现NSURLSessionTaskDelegate、NSURLSessionDataDelegate、NSURLSessionDownloadDelegate、这三个协议中AFURLSessionManagerTaskDelegate没有实现的代理回调函数

- 接收到response描述的回调实现

```objc
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
didReceiveResponse:(NSURLResponse *)response
 completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler
{
	// 当前Task的改变的状态（DataTask->DownLoadTask）
    NSURLSessionResponseDisposition disposition = NSURLSessionResponseAllow;

	// 回传response描述
    if (self.dataTaskDidReceiveResponse) {
        disposition = self.dataTaskDidReceiveResponse(session, dataTask, response);
    }

	// 告诉iOS系统是否转变task的类型
    if (completionHandler) {
        completionHandler(disposition);
    }
}
```

- 接收到响应Data数据

```objc
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data
{
	//1. 获取任务Task对象 对应的 TaskDelegate对象
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:dataTask];
    
    //2. 调用TaskDelegate实现的 URLSession:dataTask:didReceiveData:
    [delegate URLSession:session dataTask:dataTask didReceiveData:data];

	//3. 回传响应Data
    if (self.dataTaskDidReceiveData) {
        self.dataTaskDidReceiveData(session, dataTask, data);
    }
}
```

SessionManager自己实现了一次，然后TaskDelegate也实现了一次，只不过SessionManager调用TaskDelegate的实现.


其他就不贴了，和这个的套路差不多，那基本上AFN基于NSURLSession的实现就这样了，具体的细节用到的时候再去看吧...

