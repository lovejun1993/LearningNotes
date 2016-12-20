---
layout: post
title: "AFNetworking、AFHTTPRequestOperationManager"
date: 2015-08-24 14:02:02 +0800
comments: true
categories: 
---

###iOS7以下使用AFHTTPRequestOperationManager+NSOperation.

***

###AFHTTPRequestOperationManager简单使用

```
[[AFHTTPRequestOperationManager manager] GET:@"http://api.1-blog.com/biz/bizserver/xiaohua/list.do?"
                                      parameters:@{@"size":@"5",@"page":@"1"}
                                         success:^(AFHTTPRequestOperation *operation, id responseObject) {
                                             NSLog(@"responseObject = %@\n", responseObject);
                                         }failure:^(AFHTTPRequestOperation *operation, NSError *error){
        
                                             NSLog(@"error = %@\n", error);
                                         }];
```

### AFHTTPRequestOperationManager.h

```
#import <Foundation/Foundation.h>
#import <SystemConfiguration/SystemConfiguration.h>
#import <Availability.h>

#if __IPHONE_OS_VERSION_MIN_REQUIRED
#import <MobileCoreServices/MobileCoreServices.h>
#else
#import <CoreServices/CoreServices.h>
#endif

#import "AFHTTPRequestOperation.h"
#import "AFURLResponseSerialization.h"
#import "AFURLRequestSerialization.h"
#import "AFSecurityPolicy.h"
#import "AFNetworkReachabilityManager.h"

#ifndef NS_DESIGNATED_INITIALIZER
#if __has_attribute(objc_designated_initializer)
#define NS_DESIGNATED_INITIALIZER __attribute__((objc_designated_initializer))
#else
#define NS_DESIGNATED_INITIALIZER
#endif
#endif

NS_ASSUME_NONNULL_BEGIN

/**
 `AFHTTPRequestOperationManager` 提供完成的事情:
 1. request creation
 2. response serialization
 3. network reachability monitoring
 4. security
 5. as well as request operation management.
 6. conforms `NSSecureCoding` and `NSCopying` protocols，allowing operations to be archived to disk
	6.1 Archives and copies of HTTP clients will be initialized with an empty operation queue
	6.2 NSSecureCoding cannot serialize / deserialize block properties
	6.3 so an archive of an HTTP client will not include any reachability callback block that may be set

 ## Subclassing Notes

 1. iOS 7 or Mac OS X 10.9 以及之上，使用 `AFHTTPSessionManager`
 2. iOS 6 or Mac OS X 10.8 以及之前，`AFHTTPRequestOperationManager` 
 
 ## Methods to Override

 To change the behavior of all request operation construction for an `AFHTTPRequestOperationManager` subclass, override `HTTPRequestOperationWithRequest:success:failure`.

 ## Serialization

1. requestSerializer，必须实现 `<AFURLRequestSerialization>` 协议.

 2. responseSerializer，必须实现`<AFURLResponseSerialization>`协议.

 ## Network Reachability Monitoring

 AFNetworkReachabilityManager提供如下:
 1. 检测到达情况
	1.1 不可达
	1.2 可达
	1.3 可达，via WWAN（2g、3g、4g）
	1.4 可达，via WiFi

 2. 监测网络连接变化，执行回调通知

@interface AFHTTPRequestOperationManager : NSObject <NSSecureCoding, NSCopying>

/**
1. The URL used to monitor reachability
2. construct requests from relative paths in methods like `requestWithMethod:URLString:parameters:`
3.  the `GET` / `POST` / et al
 */
@property (readonly, nonatomic, strong, nullable) NSURL *baseURL;

/**
 AFHTTPRequestSerializer 提供如下两种创建 NSURLRequest:
 
1. 创建普通数据请求（非上传）
	`requestWithMethod:URLString:parameters:`

2. 创建 上传数据请求
   `multipartFormRequestWithMethod:URLString:parameters:constructingBodyWithBlock:` 

3. 可以设置所有请求的 default headers dictinary

4. serialization specified by this property. By default, this is set to an instance of `AFHTTPRequestSerializer`, which serializes query string parameters for `GET`, `HEAD`, and `DELETE` requests, or otherwise URL-form-encodes HTTP message bodies.

 @warning `requestSerializer` must not be `nil`.
 */
@property (nonatomic, strong) AFHTTPRequestSerializer <AFURLRequestSerialization> * requestSerializer;

/**
有如下几种response data解析成的数据类型

1. JSON serializer，以及支持的MIME Types:

- `application/json`
- `text/json`
- `text/javascript`

2. AFXMLDocumentResponseSerializer，以及支持的MIME Types:

- `application/xml`
- `text/xml`

3. AFPropertyListResponseSerializer，以及支持的MIME Types:

- `application/x-plist`

4. AFImageResponseSerializer，以及支持的MIME Types:

- `image/tiff`
- `image/jpeg`
- `image/gif`
- `image/png`
- `image/ico`
- `image/x-icon`
- `image/bmp`
- `image/x-bmp`
- `image/x-xbitmap`
- `image/x-win-bitmap`

 */
@property (nonatomic, strong) AFHTTPResponseSerializer <AFURLResponseSerialization> * responseSerializer;

/**
 用来调度所有的AFHttpRequestOperation的队列queue.
 */
@property (nonatomic, strong) NSOperationQueue *operationQueue;

///-------------------------------
/// @name Managing URL Credentials
///-------------------------------

/**
 Whether request operations should consult the credential storage for authenticating the connection. `YES` by default.

 @see AFURLConnectionOperation -shouldUseCredentialStorage
 */
@property (nonatomic, assign) BOOL shouldUseCredentialStorage;

/**
 1. 当某个服务器url访问需要进行 authentication challenges 时使用
 
 2. 由AFURLConnectionOperation封装处理接收挑战，创建凭证credential
 */
@property (nonatomic, strong, nullable) NSURLCredential *credential;

///-------------------------------
/// @name Managing Security Policy
///-------------------------------

/**
1. server trust for secure connections
2. `AFHTTPRequestOperationManager` uses the `defaultPolicy` unless otherwise specified.
 */
@property (nonatomic, strong) AFSecurityPolicy *securityPolicy;

///------------------------------------
/// @name Managing Network Reachability
///------------------------------------

/**
 The network reachability manager. `AFHTTPRequestOperationManager` uses the `sharedManager` by default.
 */
@property (readwrite, nonatomic, strong) AFNetworkReachabilityManager *reachabilityManager;

///-------------------------------
/// @name Managing Callback Queues
///-------------------------------

/**
 1. The dispatch queue for the `completionBlock` of request operations. 
 2. If `NULL` (default), the `main queue` is used.
 */
#if OS_OBJECT_USE_OBJC
@property (nonatomic, strong, nullable) dispatch_queue_t completionQueue;
#else
@property (nonatomic, assign, nullable) dispatch_queue_t completionQueue;
#endif

/**
 1. The dispatch group for the `completionBlock` of request operations.
 2.  If `NULL` (default), a `private dispatch group` is used.
 */
#if OS_OBJECT_USE_OBJC
@property (nonatomic, strong, nullable) dispatch_group_t completionGroup;
#else
@property (nonatomic, assign, nullable) dispatch_group_t completionGroup;
#endif

///---------------------------------------------
/// @name Creating and Initializing HTTP Clients
///---------------------------------------------

/**
 Creates and returns an `AFHTTPRequestOperationManager` object.
 */
+ (instancetype)manager;

/**
 Initializes an `AFHTTPRequestOperationManager` object with the specified base URL.

 This is the designated initializer.

 @param url The base URL for the HTTP client.

 @return The newly-initialized HTTP client
 */
- (instancetype)initWithBaseURL:(nullable NSURL *)url NS_DESIGNATED_INITIALIZER;

///---------------------------------------
/// @name Managing HTTP Request Operations
///---------------------------------------

/**
 1. Creates an `AFHTTPRequestOperation` sington instance with 传入的 NSURLRequest.
 2. sets the `response serializer` to that of the HTTP client.
 */
- (AFHTTPRequestOperation *)HTTPRequestOperationWithRequest:(NSURLRequest *)request
                                                    success:(nullable void (^)(AFHTTPRequestOperation *operation, id responseObject))success
                                                    failure:(nullable void (^)(AFHTTPRequestOperation *operation, NSError *error))failure;

///---------------------------
/// @name Making HTTP Requests
///---------------------------

/**
 Creates and runs an `AFHTTPRequestOperation` with a `GET` request.
 */
- (nullable AFHTTPRequestOperation *)GET:(NSString *)URLString
                     parameters:(nullable id)parameters
                        success:(nullable void (^)(AFHTTPRequestOperation *operation, id responseObject))success
                        failure:(nullable void (^)(AFHTTPRequestOperation * __nullable operation, NSError *error))failure;

/**
 Creates and runs an `AFHTTPRequestOperation` with a `HEAD` request.
 */
- (nullable AFHTTPRequestOperation *)HEAD:(NSString *)URLString
                      parameters:(nullable id)parameters
                         success:(nullable void (^)(AFHTTPRequestOperation *operation))success
                         failure:(nullable void (^)(AFHTTPRequestOperation * __nullable operation, NSError *error))failure;

/**
 1. Creates and runs an `AFHTTPRequestOperation` with a `POST` request.
 2. 普通数据请求
 */
- (nullable AFHTTPRequestOperation *)POST:(NSString *)URLString
                      parameters:(nullable id)parameters
                         success:(nullable void (^)(AFHTTPRequestOperation *operation, id responseObject))success
                         failure:(nullable void (^)(AFHTTPRequestOperation * __nullable operation, NSError *error))failure;

/**
 1. Creates and runs an `AFHTTPRequestOperation` with a multipart `POST` request.
 2. 上传数据请求
 */
- (nullable AFHTTPRequestOperation *)POST:(NSString *)URLString
                      parameters:(nullable id)parameters
       constructingBodyWithBlock:(nullable void (^)(id <AFMultipartFormData> formData))block
                         success:(nullable void (^)(AFHTTPRequestOperation *operation, id responseObject))success
                         failure:(nullable void (^)(AFHTTPRequestOperation * __nullable operation, NSError *error))failure;

/**
 Creates and runs an `AFHTTPRequestOperation` with a `PUT` request.
 */
- (nullable AFHTTPRequestOperation *)PUT:(NSString *)URLString
                     parameters:(nullable id)parameters
                        success:(nullable void (^)(AFHTTPRequestOperation *operation, id responseObject))success
                        failure:(nullable void (^)(AFHTTPRequestOperation * __nullable operation, NSError *error))failure;

/**
 Creates and runs an `AFHTTPRequestOperation` with a `PATCH` request.
 */
- (nullable AFHTTPRequestOperation *)PATCH:(NSString *)URLString
                       parameters:(nullable id)parameters
                          success:(nullable void (^)(AFHTTPRequestOperation *operation, id responseObject))success
                          failure:(nullable void (^)(AFHTTPRequestOperation * __nullable operation, NSError *error))failure;

/**
 Creates and runs an `AFHTTPRequestOperation` with a `DELETE` request.
 */
- (nullable AFHTTPRequestOperation *)DELETE:(NSString *)URLString
                        parameters:(nullable id)parameters
                           success:(nullable void (^)(AFHTTPRequestOperation *operation, id responseObject))success
                           failure:(nullable void (^)(AFHTTPRequestOperation * __nullable operation, NSError *error))failure;

@end

NS_ASSUME_NONNULL_END
```

***

###从头文件导入看出RequestManager依赖如下组件

```
//1. 负责执行网络请求操作（内部使用了一个全局单例子线程负责网络数据请求）
#import "AFHTTPRequestOperation.h"

//2. 请求参数等等组装器，生成NSURLRequest对象
#import "AFURLResponseSerialization.h"

//3. 响应data解析器
#import "AFURLRequestSerialization.h"

//4. 服务器信任鉴权
#import "AFSecurityPolicy.h"

//5. 网络连接监测
#import "AFNetworkReachabilityManager.h"
```

###`[[AFHTTPRequestOperationManager alloc] initWithBaseURL:nil]`初始化.

```
- (instancetype)initWithBaseURL:(NSURL *)url {
    self = [super init];
    if (!self) {
        return nil;
    }

    // Ensure terminal slash for baseURL path, so that NSURL +URLWithString:relativeToURL: works as expected
    if ([[url path] length] > 0 && ![[url absoluteString] hasSuffix:@"/"]) {
        url = [url URLByAppendingPathComponent:@""];
    }

    self.baseURL = url;
	
	//1. NSURLRequest组装器
    self.requestSerializer = [AFHTTPRequestSerializer serializer];
    
    //2. responseObject解析器
    self.responseSerializer = [AFJSONResponseSerializer serializer];

	//3. 安全策略
    self.securityPolicy = [AFSecurityPolicy defaultPolicy];

	//4. 网络状态监测
    self.reachabilityManager = [AFNetworkReachabilityManager sharedManager];
	
	//5. 所有网络操作operation的单例队列（NSOperationQueue默认是并发队列）
    self.operationQueue = [[NSOperationQueue alloc] init];

	//6. 安全相关
    self.shouldUseCredentialStorage = YES;

    return self;
}
```

***

###执行GET请求

```objc
- (AFHTTPRequestOperation *)GET:(NSString *)URLString
                     parameters:(id)parameters
                        success:(void (^)(AFHTTPRequestOperation *operation, id responseObject))success
                        failure:(void (^)(AFHTTPRequestOperation *operation, NSError *error))failure
{
    AFHTTPRequestOperation *operation = [self HTTPRequestOperationWithHTTPMethod:@"GET"
                                                                       URLString:URLString
                                                                      parameters:parameters
                                                                         success:success
                                                                         failure:failure];

    [self.operationQueue addOperation:operation];

    return operation;
}
```

调用最终创建operation的方法.

***

###执行POST请求，非上传

```objc
- (AFHTTPRequestOperation *)POST:(NSString *)URLString
                      parameters:(id)parameters
                         success:(void (^)(AFHTTPRequestOperation *operation, id responseObject))success
                         failure:(void (^)(AFHTTPRequestOperation *operation, NSError *error))failure
{
    AFHTTPRequestOperation *operation = [self HTTPRequestOperationWithHTTPMethod:@"POST"
                                                                       URLString:URLString
                                                                      parameters:parameters
                                                                         success:success
                                                                         failure:failure];

    [self.operationQueue addOperation:operation];

    return operation;
}
```

只是method是post，与上面代码一致.

***

###GET请求、数据POST请求，最终都是走私有方法创建operation函数，主要将请求参数组装成NSURLRequest对象.

```objc
- (AFHTTPRequestOperation *)HTTPRequestOperationWithHTTPMethod:(NSString *)method
                                                     URLString:(NSString *)URLString
                                                    parameters:(id)parameters
                                                       success:(void (^)(AFHTTPRequestOperation *operation, id responseObject))success
                                                       failure:(void (^)(AFHTTPRequestOperation *operation, NSError *error))failure
{
    NSError *serializationError = nil;
    
    //1. 传入请求url的字符串
    NSString *urlStr = [[NSURL URLWithString:URLString relativeToURL:self.baseURL] absoluteString];
    
    //2. 调用requestSerializer完成NSMutableURLRequest对象创建
    NSMutableURLRequest *request = [self.requestSerializer requestWithMethod:method
                                                                   URLString:urlStr
                                                                  parameters:parameters
                                                                       error:&serializationError];
    //3. 是否构造出错
    if (serializationError) {
        if (failure) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wgnu"
            
            //3.1 构造出错，执行回调通知错误
            dispatch_async(self.completionQueue ?: dispatch_get_main_queue(), ^{
                failure(nil, serializationError);
            });
#pragma clang diagnostic pop
        }

        //3.2 错误结束此次请求
        return nil;
    }

	//4. 使用得到的request对象，创建operation对象
    return [self HTTPRequestOperationWithRequest:request success:success failure:failure];
}
```

***

###执行POST请求，上传，直接调用Request Serilalizer构造 `multipart` request

```objc
- (AFHTTPRequestOperation *)POST:(NSString *)URLString
                      parameters:(id)parameters
       constructingBodyWithBlock:(void (^)(id <AFMultipartFormData> formData))block
                         success:(void (^)(AFHTTPRequestOperation *operation, id responseObject))success
                         failure:(void (^)(AFHTTPRequestOperation *operation, NSError *error))failure
{
    NSError *serializationError = nil;
    
    //1. 调用 Request Serialzier 组装 NSURLRequest对象
    NSString *urlStr = [[NSURL URLWithString:URLString relativeToURL:self.baseURL] absoluteString];
    
    //注意，调用创建的是 multipart request
    NSMutableURLRequest *request = [self.requestSerializer multipartFormRequestWithMethod:@"POST"
                                                                                URLString:urlStr
                                                                               parameters:parameters constructingBodyWithBlock:block
                                                                                    error:&serializationError];
    
    //2. 是否组装出错
    if (serializationError) {
        if (failure) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wgnu"
            
            //组装出错，执行错误回调
            dispatch_async(self.completionQueue ?: dispatch_get_main_queue(), ^{
                failure(nil, serializationError);
            });
#pragma clang diagnostic pop
        }

        //结束此次请求
        return nil;
    }

    //3. 使用组装完毕的NSURLRequest创建AFHTTPRequestOperation对象
    AFHTTPRequestOperation *operation = [self HTTPRequestOperationWithRequest:request
                                                                      success:success
                                                                      failure:failure];
    
    //4. operation入队manager的queue
    [self.operationQueue addOperation:operation];

    return operation;
}
```

如上几个主要是得到 `NSURLRequest对象`

***

###最终使用NSURLRequest对象，创建Operation对象的函数。这个函数是向外暴露的，所以开发者可以直接使用一个request来创建operation。


```objc
- (AFHTTPRequestOperation *)HTTPRequestOperationWithRequest:(NSURLRequest *)request
                                                    success:(void (^)(AFHTTPRequestOperation *operation, id responseObject))success
                                                    failure:(void (^)(AFHTTPRequestOperation *operation, NSError *error))failure
{
	//1. 创建operation，使用request
    AFHTTPRequestOperation *operation = [[AFHTTPRequestOperation alloc] initWithRequest:request];
    
    //2. 
    operation.responseSerializer = self.responseSerializer;
    
    //3. 
    operation.shouldUseCredentialStorage = self.shouldUseCredentialStorage;
    
    //4.
    operation.credential = self.credential;
    
    //5.
    operation.securityPolicy = self.securityPolicy;

	//6. 调用AFHTTPRequestOperation重写父类的方法，完成回调block的保存
    [operation setCompletionBlockWithSuccess:success failure:failure];
    
    //7.
    operation.completionQueue = self.completionQueue;
    
    //8.
    operation.completionGroup = self.completionGroup;

    return operation;
}
```


***

###NSOperation被Queue调度时，会被执行`start`函数.

***

###AFHTTPRequestOperation继承自AFURLConnectionOperation，关于线程的方法全都由AFURLConnectionOperation提供.

***

###AFHTTPRequestOperationManager重写了description方法

```objc
- (NSString *)description {
    return [NSString stringWithFormat:@"<%@: %p, baseURL: %@, operationQueue: %@>", NSStringFromClass([self class]), self, [self.baseURL absoluteString], self.operationQueue];
}
```

***

###AFHTTPRequestOperationManager实现了NSSecureCoding协议，所以可以归档到磁盘文件

```objc
+ (BOOL)supportsSecureCoding {
    return YES;
}
```

```objc
- (id)initWithCoder:(NSCoder *)decoder {
    
    //1.
    NSURL *baseURL = [decoder decodeObjectForKey:NSStringFromSelector(@selector(baseURL))];

    //2.
    self = [self initWithBaseURL:baseURL];
    if (!self) {
        return nil;
    }

    //3.
    self.requestSerializer = [decoder decodeObjectOfClass:[AFHTTPRequestSerializer class] forKey:NSStringFromSelector(@selector(requestSerializer))];
    self.responseSerializer = [decoder decodeObjectOfClass:[AFHTTPResponseSerializer class] forKey:NSStringFromSelector(@selector(responseSerializer))];

    return self;
}
```

```objc
- (void)encodeWithCoder:(NSCoder *)coder {

	//1.
    [coder encodeObject:self.baseURL forKey:NSStringFromSelector(@selector(baseURL))];
    
    //2.
    [coder encodeObject:self.requestSerializer forKey:NSStringFromSelector(@selector(requestSerializer))];
    
    //3.
    [coder encodeObject:self.responseSerializer forKey:NSStringFromSelector(@selector(responseSerializer))];
}
```

如果是归档后，那么恢复后的manager对象的operationQueue是空的，因为没有被归档.

- operatinQueue为空
- completionQueue为空
- completionGroup为空

***


###总之，一个`AFHTTPRequestOperation : NSOperation`被创建完毕

- 包含一个NSURLRequest对象，请求数据
- 包含一个responseSerializer
- web服务挑战
- 包含success block， fail block，回调队列和组