---
layout: post
title: "NSURLProtocol"
date: 2015-01-07 23:19:07 +0800
comments: true
categories: 
---

###NSURLProtocol可以让我们拦截程序中的一切网络请求，主要进行如下拦截处理:

- 自定义请求 和 响应

- 过滤掉某些请求不让其发起、以及修改

- 提供 自定义的全局缓存 逻辑

- 重定向 网络请求

- 提供 HTTP Mocking (方便前期测试)

***

###首先，继承NSURLProtocol编写自己的子类，然后在App程序启动时，注册到程序中

```
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions 
{
    [NSURLProtocol registerClass:[MyURLProtocol class]];
    return YES;
}
```

***

###接下来就是编写自己的NSURLProtocol子类，重写一些方法

***

###重写方法一、整个URL Loading System的入口，进行请求的过滤，筛选出需要进行处理的请求


```
+ (BOOL)canInitWithRequest:(NSURLRequest *)request
```

- 返回YES，表示要拦截处理，继续执行下面的其他函数

- 返回NO，表示不拦截处理，直接让其发起请求

- 注意，需要防止 `无限循环`

***

###重写方法二、请求重定向、修改请求头

```
+ (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request
```

###重写方法三、让被拦截的请求执行

```
- (void)startLoading
```

###重写方法四、取消执行请求

```
- (void)stopLoading 
```

###重写方法五、可用来使用缓存数据结束此次网络请求，如果不考虑自定义缓存逻辑，可以不重写此方法

```
+ (BOOL)requestIsCacheEquivalent:(NSURLRequest *)a toRequest:(NSURLRequest *)b
```

***

###demo1、自定义NSURLProtol拦截请求后进行处理，然后使用NSURLSession继续执行网络请求

```
#import <Foundation/Foundation.h>

@interface MySessionURLProtocol : NSURLProtocol

@end
```

```
#import "MySessionURLProtocol.h"

//防止循环拦截的标识
#define protocolKey @"SessionProtocolKey"

@interface MySessionURLProtocol ()<NSURLSessionDataDelegate>

@property (nonatomic, strong) NSURLSession * session;

@end
```

```
@implementation MySessionURLProtocol

/**
 *  是否拦截处理指定的请求
 *
 *  @param request 指定的请求
 *
 *  @return 返回YES表示要拦截处理，返回NO表示不拦截处理
 */
+ (BOOL)canInitWithRequest:(NSURLRequest *)request {
    
    /*
     防止无限循环，因为一个请求在被拦截处理过程中，也会发起一个请求，这样又会走到这里，如果不进行处理，就会造成无限循环
     如果已经拦截处理了，就不再进行拦截了，直接让其发起网络请求.
     */
    if ([NSURLProtocol propertyForKey:protocolKey inRequest:request]) {
        return NO;
    }
    
    // URL的全路径字符串
    NSString * url = request.URL.absoluteString;
    
	//只处理http和https请求
	NSString *scheme = [[request URL] scheme];
	if ( ([scheme caseInsensitiveCompare:@"http"] == NSOrderedSame ||
     [scheme caseInsensitiveCompare:@"https"] == NSOrderedSame))
    {
    	return YES;
    }
    
    return NO;
}

/**
 *  如果需要对请求进行重定向，添加指定头部等操作，可以在该方法中进行
 *
 *  @param request 原请求
 *
 *  @return 修改后的请求
 */
+ (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request {
    
    // 修改了请求的头部信息
    NSMutableURLRequest * mutableReq = [request mutableCopy];
    
    NSMutableDictionary * headers = [mutableReq.allHTTPHeaderFields mutableCopy];
    [headers setObject:@"BBBB" forKey:@"Key2"];
    
    mutableReq.allHTTPHeaderFields = headers;
    
    //返回修改请求头后的Request
    return [mutableReq copy];
    
    //还可以修改Request.url进行重定向
    //...
}

/**
 *  开始加载，在该方法中，加载一个请求
 */
- (void)startLoading {

	
    NSMutableURLRequest * request = [self.request mutableCopy];
    
    // 标记当前传入的Request已经被拦截处理过，
    //防止在最开始又继续拦截处理
    [NSURLProtocol setProperty:@(YES) forKey:protocolKey inRequest:request];
    
    //如下使用NSURLSession让拦截的请求进行请求网络
    NSURLSessionConfiguration * config = [NSURLSessionConfiguration defaultSessionConfiguration];
    self.session = [NSURLSession sessionWithConfiguration:config delegate:self delegateQueue:[NSOperationQueue mainQueue]];
    NSURLSessionDataTask * task = [self.session dataTaskWithRequest:request];
    [task resume];
}

/**
 *  取消请求
 */
- (void)stopLoading {
    [self.session invalidateAndCancel];
    self.session = nil;
}

#pragma mark - NSURLSessionDataDelegate

-(void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error
{
    if (error) {
        [self.client URLProtocol:self didFailWithError:error];
    } else {
        [self.client URLProtocolDidFinishLoading:self];
    }
}

-(void)URLSession:(NSURLSession *)session
         dataTask:(NSURLSessionDataTask *)dataTask
didReceiveResponse:(NSURLResponse *)response
completionHandler:(void (^)(NSURLSessionResponseDisposition))completionHandler
{
    [self.client URLProtocol:self
          didReceiveResponse:response
          cacheStoragePolicy:NSURLCacheStorageNotAllowed];
    
    completionHandler(NSURLSessionResponseAllow);
}

-(void)URLSession:(NSURLSession *)session
         dataTask:(NSURLSessionDataTask *)dataTask
   didReceiveData:(NSData *)data
{
    [self.client URLProtocol:self didLoadData:data];
}

- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
 willCacheResponse:(NSCachedURLResponse *)proposedResponse
 completionHandler:(void (^)(NSCachedURLResponse *cachedResponse))completionHandler
{
    completionHandler(proposedResponse);
}

@end
```

***

###demo2、简单的重定向处理

```
@implementation MySessionURLProtocol

...

+ (NSURLRequest *) canonicalRequestForRequest:(NSURLRequest *)request {
    NSMutableURLRequest *mutableReqeust = [request mutableCopy];
    mutableReqeust = [self redirectHostInRequset:mutableReqeust];
    return mutableReqeust;
}

...

+(NSMutableURLRequest*)redirectHostInRequset:(NSMutableURLRequest*)request
{
    if ([request.URL host].length == 0) {
        return request;
    }
    
    //原始url全路径
    NSString *originUrlString = [request.URL absoluteString];
    
    //原始url的host
    NSString *originHostString = [request.URL host];
    
    //获取host的range
    NSRange hostRange = [originUrlString rangeOfString:originHostString];
    
    if (hostRange.location == NSNotFound) {
        return request;
    }
    
    //重定向到bing搜索主页
    NSString *ip = @"cn.bing.com";
    
    // 替换开始的host
    NSString *urlString = [originUrlString stringByReplacingCharactersInRange:hostRange withString:ip];
    
    //得到重定向后的url
    NSURL *url = [NSURL URLWithString:urlString];
    
    //将新的url赋值给Request
    request.URL = url;

    //返回修改host重定向的request
    return request;
}

...

@end
```

***

###demo3、自定义NSURLProtol拦截请求后进行处理，然后使用NSURLConnection继续执行网络请求

```
#import "MyConnectionURLProtocol.h"

#define protocolKey @"ConnectionProtocolKey"

@interface MyConnectionURLProtocol () <NSURLConnectionDataDelegate>

@property (nonatomic, strong) NSURLConnection * connection;

@end

@implementation MyConnectionURLProtocol

/**
 *  是否拦截处理指定的请求
 *
 *  @param request 指定的请求
 *
 *  @return 返回YES表示要拦截处理，返回NO表示不拦截处理
 */
+ (BOOL)canInitWithRequest:(NSURLRequest *)request {
    
    /* 
        防止无限循环，因为一个请求在被拦截处理过程中，也会发起一个请求，这样又会走到这里，如果不进行处理，就会造成无限循环
     */
    if ([NSURLProtocol propertyForKey:protocolKey inRequest:request]) {
        return NO;
    }
    
    NSString * url = request.URL.absoluteString;
    
    // 如果url已http或https开头，则进行拦截处理，否则不处理
    if ([url hasPrefix:@"http"] || [url hasPrefix:@"https"]) {
        return YES;
    }
    return NO;
}

/**
 *  如果需要对请求进行重定向，添加指定头部等操作，可以在该方法中进行
 *
 *  @param request 原请求
 *
 *  @return 修改后的请求
 */
+ (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request {
    
    // 修改了请求的头部信息
    NSMutableURLRequest * mutableReq = [request mutableCopy];
    NSMutableDictionary * headers = [mutableReq.allHTTPHeaderFields mutableCopy];
    [headers setObject:@"AAAA" forKey:@"Key1"];
    mutableReq.allHTTPHeaderFields = headers;
    NSLog(@"connection reset header");
    return [mutableReq copy];
}

/**
 *  开始加载，在该方法中，加载一个请求
 */
- (void)startLoading {
    NSMutableURLRequest * request = [self.request mutableCopy];
    // 表示该请求已经被处理，防止无限循环
    [NSURLProtocol setProperty:@(YES) forKey:protocolKey inRequest:request];
    
    self.connection = [NSURLConnection connectionWithRequest:request delegate:self];
}

/**
 *  取消请求
 */
- (void)stopLoading {
    [self.connection cancel];
}

#pragma mark - NSURLConnectionDelegate

- (void)connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response {
    [self.client URLProtocol:self didReceiveResponse:response cacheStoragePolicy:NSURLCacheStorageNotAllowed];
}

- (void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data {
    [self.client URLProtocol:self didLoadData:data];
}

- (void)connectionDidFinishLoading:(NSURLConnection *)connection {
    [self.client URLProtocolDidFinishLoading:self];
}

- (void)connection:(NSURLConnection *)connection didFailWithError:(NSError *)error {
    [self.client URLProtocol:self didFailWithError:error];
}

@end
```

###最后发起一个网络请求测试

```
NSURL *url = @"http://www.baidu.com";

NSURLRequest *request = [NSURLRequest requestWithURL:url];

[self.webView loadRequest:request];
```

OK，终于又弄清楚了一个东西...

****

###更多的NSURLProtocol作用

- 用来实现UIWebView的离线缓存

- 截取request并重新定向到新的地址
	- 替换成链接最快的ip地址，作为api请求的Host

- NSEtcHosts有时间看看代码实现

***

###首先定义缓存响应response的实体类

```objc
#import <Foundation/Foundation.h>
#import <CoreData/CoreData.h>


@interface CachedURLResponse : NSManagedObject

@property (nonatomic, retain) NSDate * timestamp;
@property (nonatomic, retain) NSString * url;
@property (nonatomic, retain) NSString * mimeType;
@property (nonatomic, retain) NSString * encoding;
@property (nonatomic, retain) NSData * data;


@end
```

```objc
#import "CachedURLResponse.h"

@implementation CachedURLResponse

@end
```

###自定义NSURLProtocol子类

```objc
//
//  MyURLProtocol.m
//  NSURLProtocolExample
//
//  Created by Rocir Marcos Leite Santiago on 11/29/13.
//  Copyright (c) 2013 Rocir Santiago. All rights reserved.
//

#import "MyURLProtocol.h"

// AppDelegate
#import "AppDelegate.h"

// Model
#import "CachedURLResponse.h"

static NSString * const MyURLProtocolHandledKey = @"MyURLProtocolHandledKey";

@interface MyURLProtocol () <NSURLConnectionDelegate>

@property (nonatomic, strong) NSURLConnection *connection;
@property (nonatomic, strong) NSMutableData *mutableData;
@property (nonatomic, strong) NSURLResponse *response;

@end

@implementation MyURLProtocol

+ (BOOL)canInitWithRequest:(NSURLRequest *)request {
    
    if ([NSURLProtocol propertyForKey:MyURLProtocolHandledKey inRequest:request]) {
        return NO;
    }
    
    return YES;
}

+ (NSURLRequest *) canonicalRequestForRequest:(NSURLRequest *)request {
    return request;
}

- (void) startLoading {
    
    //1. 获取缓存的response
    CachedURLResponse *cachedResponse = [self cachedResponseForCurrentRequest];
    
    //2. 判断缓存response是否存在
    if (cachedResponse) {
        
        //2.1 如果存在
        
        //2.2 获取response的属性
        NSData *data = cachedResponse.data;
        NSString *mimeType = cachedResponse.mimeType;
        NSString *encoding = cachedResponse.encoding;
        
        //2.3 构造一个新的response
        NSURLResponse *response = [[NSURLResponse alloc] initWithURL:self.request.URL
                                                            MIMEType:mimeType
                                               expectedContentLength:data.length
                                                    textEncodingName:encoding];

        //2.4 将新的response作为request对应的response
        [self.client URLProtocol:self
              didReceiveResponse:response
              cacheStoragePolicy:NSURLCacheStorageNotAllowed];
        
        //2.5 设置request对应的 响应数据 response data
        [self.client URLProtocol:self didLoadData:data];
        
        //2.6 标记请求结束
        [self.client URLProtocolDidFinishLoading:self];
        
    } else {
        
        //2.1 不存在
        
        //2.2 拷贝新一个新的request
        NSMutableURLRequest *newRequest = [self.request mutableCopy];
        
        //2.3 可以对request做一些附加修改
        //请求头参数
        //请求URL替换
        //...
        
        //2.4 标记这个request已经被拦截处理过
        [NSURLProtocol setProperty:@YES
                            forKey:MyURLProtocolHandledKey
                         inRequest:newRequest];
        
        //2.5 创建一个新的NSURLConnection网络请求
        self.connection = [NSURLConnection connectionWithRequest:newRequest delegate:self];
        
    }
    
}

- (void) stopLoading {
    
    [self.connection cancel];
    self.mutableData = nil;
    
}

#pragma mark - NSURLConnectionDelegate

- (void) connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response {
    
    /** 必写 */
    [self.client URLProtocol:self
          didReceiveResponse:response
          cacheStoragePolicy:NSURLCacheStorageNotAllowed];
    
    self.response = response;
    
    self.mutableData = [[NSMutableData alloc] init];
}

- (void) connection:(NSURLConnection *)connection didReceiveData:(NSData *)data {

    /** 必写 */
    [self.client URLProtocol:self didLoadData:data];
    
    [self.mutableData appendData:data];
}

- (void) connectionDidFinishLoading:(NSURLConnection *)connection {
    
    /** 必写 */
    [self.client URLProtocolDidFinishLoading:self];
    
    [self saveCachedResponse];
}

- (void)connection:(NSURLConnection *)connection didFailWithError:(NSError *)error {
    
    /** 必写 */
    [self.client URLProtocol:self didFailWithError:error];
}

#pragma mark - 使用CoreData缓存CachedURLResponse实体类

- (CachedURLResponse *) cachedResponseForCurrentRequest {
    
    AppDelegate *delegate = [[UIApplication sharedApplication] delegate];
    NSManagedObjectContext *context = delegate.managedObjectContext;
    
    NSFetchRequest *fetchRequest = [[NSFetchRequest alloc] init];
    NSEntityDescription *entity = [NSEntityDescription entityForName:@"CachedURLResponse"
                                              inManagedObjectContext:context];
    [fetchRequest setEntity:entity];
    
    //缓存查询key
    NSString *cacheKey = [NSString stringWithFormat:@"SELF MATCHES %@", self.request.URL.absoluteString];
    
    NSPredicate *predicate = [NSPredicate predicateWithFormat:cacheKey];
    
    [fetchRequest setPredicate:predicate];
    
    NSError *error;
    NSArray *result = [context executeFetchRequest:fetchRequest error:&error];
    
    if (result && result.count > 0) {
        return result[0];
    }
    
    return nil;
    
}

- (void) saveCachedResponse {
    
    AppDelegate *delegate = [[UIApplication sharedApplication] delegate];
    NSManagedObjectContext *context = delegate.managedObjectContext;
    
    CachedURLResponse *cachedResponse = [NSEntityDescription insertNewObjectForEntityForName:@"CachedURLResponse"
                                                                      inManagedObjectContext:context];
    cachedResponse.data = self.mutableData;
    cachedResponse.url = self.request.URL.absoluteString;
    cachedResponse.timestamp = [NSDate date];
    cachedResponse.mimeType = self.response.MIMEType;
    cachedResponse.encoding = self.response.textEncodingName;
    
    NSError *error;
    [context save:&error];
    if (error) {
        NSLog(@"Could not cache the response.");
    }
    
    
}

@end
```

###AppDelegte注册自己的URLProtocol

```objc
@implementation AppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    // Override point for customization after application launch.
    
    [NSURLProtocol registerClass:MyURLProtocol.class];
    
    return YES;
}

@end	
```
