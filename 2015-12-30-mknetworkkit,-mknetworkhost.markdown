---
layout: post
title: "MKNetworkKit、MKNetworkHost"
date: 2015-12-30 15:05:25 +0800
comments: true
categories: 
---

###首先看一下Host的使用

####创建一个MKNetworkHost实例

```objc
- (MKNetworkHost *)host {
    
    if (!_host) {
        
        // 创建Host，不要添加 http://
        _host = [[MKNetworkHost alloc] initWithHostName:@"www.weather.com.cn"];
        
        // 设置Host的端口
        //_host.portNumber = 1000;
        
        // 设置路径
        //_host.path = @"";
        
        // 设置默认的请求头参数
        //_host.defaultHeaders = @{};
        
        // 设置是否使用安全，如果是YES，那么在scheme前面添加 https://
        //_host.secureHost = YES;
        
        // 设置Host的回调代理对象
        _host.delegate = self;
        
        // 开启缓存方法一、使用MK提供的默认缓存目录，内存缓存没有大小限制
        [_host enableCache];
        
        // 开启缓存方法二、使用自己的缓存目录，可以指定内存缓存大小限制
        //[_host enableCacheWithDirectory:@"" inMemoryCost:50 *1024];
        
        // 设置Host的请求参数组织方式
        _host.defaultParameterEncoding = MKNKParameterEncodingURL;//默认
        //_host.defaultParameterEncoding = MKNKParameterEncodingJSON;
        //_host.defaultParameterEncoding = MKNKParameterEncodingPlist;
    }
    
    return _host;
}
```

####设置Host的代理后，Host内部创建 NSURLSessionConfiguration 时会回调

```objc
#pragma mark - MKNetworkHostDelegate

/**
 *  创建 默认 NSURLSessionConfiguration
 */
-(void) networkHost:(MKNetworkHost*) networkHost didCreateDefaultSessionConfiguration:(NSURLSessionConfiguration*) configuration
{
    NSLog(@"创建 默认 NSURLSessionConfiguration");
}

/**
 *  创建 内存 NSURLSessionConfiguration
 */
-(void) networkHost:(MKNetworkHost*) networkHost didCreateEphemeralSessionConfiguration:(NSURLSessionConfiguration*) configuration
{
    NSLog(@"创建 内存 NSURLSessionConfiguration");
}

/**
 *  创建 后台 NSURLSessionConfiguration
 */
-(void) networkHost:(MKNetworkHost*) networkHost didCreateBackgroundSessionConfiguration:(NSURLSessionConfiguration*) configuration
{
    NSLog(@"创建 后台 NSURLSessionConfiguration");
}
```

####然后使用Host创建，并发起一个MKNetworkRequest

```objc
- (void)requestNet {
    
    //1. 使用Host创建一个MKNetworkRequest实例
    _request = [self.host requestWithPath:@"/adat/cityinfo/101010100.html"
                                   params:nil
                               httpMethod:@"GET"];
    
    //2. 设置Request的回调
    [_request addCompletionHandler:^(MKNetworkRequest *completedRequest) {
        NSLog(@"response  = %@\n", completedRequest.responseData);
        NSLog(@"response  = %@\n", completedRequest.responseAsImage);
        NSLog(@"response  = %@\n", completedRequest.responseAsJSON);
        NSLog(@"response  = %@\n", completedRequest.responseAsString);
    }];
    
    //3. 使用Host开始执行这个Request
    [self.host startRequest:_request];
}
```

####如果是下载，使用Host实例的如下方法

```objc
-(void) startDownloadRequest:(MKNetworkRequest*) request;
```

####如果是上传，使用Host实例的如下方法

```objc
-(void) startUploadRequest:(MKNetworkRequest*) request;
```

****

###开始阅读`MKNetworkHost`源码

###首先 MKNetworkHost.m中，声明了一个`MKNetworkRequest 匿名分类`扩展方法，可以操作其私有属性变量

```objc
@interface MKNetworkRequest (/*Private Methods*/)

@property (readwrite) NSHTTPURLResponse *response;

@property (readwrite) NSData *responseData;


@property (readwrite) NSError *error;

@property (readwrite) MKNKRequestState state;

@property (readwrite) NSURLSessionTask *task;

-(void) setProgressValue:(CGFloat) updatedValue;


@end
```

###初始化工作

```objc
- (instancetype) initWithHostName:(NSString*) hostName {
  
  //1. 
  MKNetworkHost *engine = [[MKNetworkHost alloc] init];
  
  //2.
  engine.hostName = hostName;
  
  return engine;
}
```

```objc
- (instancetype) init {
  
  if((self = [super init])) {
    
    //1. 使用一个 串行队列，统一执行Request
    self.runningTasksSynchronizingQueue = dispatch_queue_create("com.mknetworkkit.cachequeue", DISPATCH_QUEUE_SERIAL);
    
    //2. 在串行队列，初始化用来保存所有NSURLSessionTask的数组
    dispatch_async(self.runningTasksSynchronizingQueue, ^{
      self.activeTasks = [NSMutableArray array];
    });
  }
  
  return self;
}
```

###维护了三个NSURLSession单例

```objc
@interface MKNetworkHost (/*Private Methods*/) <NSURLSessionDelegate>

@property (readonly) NSURLSession *defaultSession;
@property (readonly) NSURLSession *ephemeralSession;
@property (readonly) NSURLSession *backgroundSession;

...
```

```objc
-(NSURLSession*) backgroundSession {
  
  static dispatch_once_t onceToken;
  
  static NSURLSessionConfiguration *backgroundSessionConfiguration;
  static NSURLSession *backgroundSession;
  
  dispatch_once(&onceToken, ^{
  
  	//1. 使用当前工程的包路径作为configuration的identifier
    backgroundSessionConfiguration =
    [NSURLSessionConfiguration backgroundSessionConfigurationWithIdentifier:
     [[NSBundle mainBundle] bundleIdentifier]];
    
    //2. 使用delegate形式，回调Host的代理，通知session创建完毕
    if([self.delegate respondsToSelector:@selector(networkHost:didCreateBackgroundSessionConfiguration:)]) 
    {
      [self.delegate networkHost:self didCreateBackgroundSessionConfiguration:backgroundSessionConfiguration];
    }
    
    //3. 创建NSURLSession实例，使用一个新的NSOperationQueue作为回调操作队列
    backgroundSession = [NSURLSession sessionWithConfiguration:backgroundSessionConfiguration
                                                           delegate:self
                                                      delegateQueue:[[NSOperationQueue alloc] init]];
  });
  
  return backgroundSession;
}
```

```objc
-(NSURLSession*) defaultSession {
  
  static dispatch_once_t onceToken;
  
  static NSURLSessionConfiguration *defaultSessionConfiguration;
  static NSURLSession *defaultSession;
  
  dispatch_once(&onceToken, ^{
    
    //1. default configuration
    defaultSessionConfiguration = [NSURLSessionConfiguration defaultSessionConfiguration];

	//2.
    if([self.delegate respondsToSelector:@selector(networkHost:didCreateDefaultSessionConfiguration:)]) 
    {
      [self.delegate networkHost:self didCreateDefaultSessionConfiguration:defaultSessionConfiguration];
    }
    
    //3. 注意sesion回调使用的 main queue
    defaultSession = [NSURLSession sessionWithConfiguration:defaultSessionConfiguration
                                                      delegate:self
                                                 delegateQueue:[NSOperationQueue mainQueue]];
  });
  
  return defaultSession;
}
```

```objc
-(NSURLSession*) ephemeralSession {
  
  static dispatch_once_t onceToken;
  static NSURLSessionConfiguration *ephemeralSessionConfiguration;
  static NSURLSession *ephemeralSession;
  dispatch_once(&onceToken, ^{
    
    //1. ephemeral configuration
    ephemeralSessionConfiguration = [NSURLSessionConfiguration ephemeralSessionConfiguration];

	//2.
    if([self.delegate respondsToSelector:@selector(networkHost:didCreateEphemeralSessionConfiguration:)]) {
      [self.delegate networkHost:self didCreateEphemeralSessionConfiguration:ephemeralSessionConfiguration];
    }
    
    //3. 注意sesion回调使用的 main queue
    ephemeralSession = [NSURLSession sessionWithConfiguration:ephemeralSessionConfiguration
                                                   delegate:self
                                              delegateQueue:[NSOperationQueue mainQueue]];
  });
  
  return ephemeralSession;
}
```

###MKNetworkHost开启缓存

- 使用两个MKCache实例

```objc
/**
 *  缓存response data 具体数据
 */
@property MKCache *dataCache;

/**
 *  缓存response 描述数据
 */
@property MKCache *responseCache;
```

- 使用MK提供的缓存目录，以及不限制内存缓存大小

```objc
-(void) enableCache;
```

```objc
-(void) enableCache {
  [self enableCacheWithDirectory:kMKCacheDefaultDirectoryName inMemoryCost:0];
}
```

```
MK的默认缓存目录
NSString *const kMKCacheDefaultDirectoryName = @"com.mknetworkkit.mkcache";
```

- 使用开发者自己设置的缓存目录，使用开发者设置的缓存大小来限制

```objc
-(void) enableCacheWithDirectory:(NSString*) cacheDirectoryPath inMemoryCost:(NSUInteger) inMemoryCost;
```

```objc
-(void) enableCacheWithDirectory:(NSString*) cacheDirectoryPath inMemoryCost:(NSUInteger) inMemoryCost
{
    //1. dataCache
    NSString *dataCacgePath = [NSString stringWithFormat:@"%@/data", cacheDirectoryPath];
    self.dataCache = [[MKCache alloc] initWithCacheDirectory:dataCacgePath
                                                inMemoryCost:inMemoryCost];
    
    //2. responseCache
    NSString *responseCachePath = [NSString stringWithFormat:@"%@/responses", cacheDirectoryPath];
    self.responseCache = [[MKCache alloc] initWithCacheDirectory:responseCachePath
                                                  inMemoryCost:inMemoryCost];
}
```

***

###所有的MKNetworkRequest都由MKNetworkHost创建

```objc
-(MKNetworkRequest*) requestWithURLString:(NSString*) urlString;

-(MKNetworkRequest*) requestWithPath:(NSString*) path;

-(MKNetworkRequest*) requestWithPath:(NSString*) path
                              params:(NSDictionary*) params;

-(MKNetworkRequest*) requestWithPath:(NSString*) path
                              params:(NSDictionary*) params
                          httpMethod:(NSString*) httpMethod;

-(MKNetworkRequest*) requestWithPath:(NSString*) path
                              params:(NSDictionary*) params
                          httpMethod:(NSString*)method
                                body:(NSData*) bodyData
                                 ssl:(BOOL) useSSL;
```

```objc
//根据传入一个完整的 字符串url 直接发起请求
-(MKNetworkRequest*) requestWithURLString:(NSString*) urlString
{
    //1. 创建 MKNetworkRequest，默认使用 GET
    MKNetworkRequest *request = [[MKNetworkRequest alloc] initWithURLString:urlString
                                                                   params:nil
                                                                 bodyData:nil
                                                               httpMethod:@"GET"];
    //2. 设置MKNetworkRequest请求头，与缓存相关
    [self prepareRequest:request];
    
    return request;
}
```

```
//传入path路径
//是否使用https:// 由Host决定
-(MKNetworkRequest*) requestWithPath:(NSString*) path
{
  return [self requestWithPath:path
                        params:nil
                    httpMethod:@"GET"
                          body:nil
                           ssl:self.secureHost];
}
```

```objc
//传入path路径
//传入请求参数字典
//是否使用https:// 由Host决定
-(MKNetworkRequest*) requestWithPath:(NSString*) path
                              params:(NSDictionary*) params
{
  
  return [self requestWithPath:path
                        params:params
                    httpMethod:@"GET"
                          body:nil
                           ssl:self.secureHost];
}
```

```objc
//传入path路径
//传入请求参数字典
//传入请求方法
//是否使用https:// 由Host决定
-(MKNetworkRequest*) requestWithPath:(NSString*) path
                              params:(NSDictionary*) params
                          httpMethod:(NSString*) httpMethod
{
  
  return [self requestWithPath:path
                        params:params
                    httpMethod:httpMethod
                          body:nil
                           ssl:self.secureHost];
}
```

```objc
-(MKNetworkRequest*) requestWithPath:(NSString*) path
                              params:(NSDictionary*) params
                          httpMethod:(NSString*) httpMethod
                                body:(NSData*) bodyData
                                 ssl:(BOOL) useSSL
{
    
    //1. 使用传入path创建Request时，必须提前设置Host的 hostName 指定服务器主机域名
    if(self.hostName == nil)
    {
        NSLog(@"Hostname is nil, use requestWithURLString: method to create absolute URL operations");
        return nil;
    }
    
    //2. 判断Host使用 https:// 或 http://
    //然后拼接 hostName
    NSMutableString *urlString = [NSMutableString stringWithFormat:@"%@://%@",
                                useSSL ? @"https" : @"http",
                                self.hostName];
    
    //3. 拼接端口号
    if(self.portNumber != 0) {
        [urlString appendFormat:@":%lu", (unsigned long)self.portNumber];
    }

    //4. 拼接path路径
    if(self.path) {
        [urlString appendFormat:@"/%@", self.path];
    }
    
    //5. 对 path 前面的斜杠处理
    //如果 path 是 /adat/cityinfo/101010100.html，那么不做处理
    //如果 path 是 adat/cityinfo/101010100.html，那么在其最前面加上一个 /
    if(![path isEqualToString:@"/"])
    {
        // fetch for root?
        if(path.length > 0 && [path characterAtIndex:0] == '/') // if user passes /, don't prefix a slash
          [urlString appendFormat:@"%@", path];
        else if (path != nil)
          [urlString appendFormat:@"/%@", path];
    }

    //6. 创建 MKNetworkRequest
    MKNetworkRequest *request = [[MKNetworkRequest alloc] initWithURLString:urlString
                                                                   params:params
                                                                 bodyData:bodyData
                                                               httpMethod:httpMethod.uppercaseString];
    
    //7. MKNetworkRequest实例默认组织请求参数的方式
    request.parameterEncoding = self.defaultParameterEncoding;
    
    //8. 将Host对象的 请求头参数字典 设置给 MKNetworkRequest对象
    [request addHeaders:self.defaultHeaders];
    
    //9. MKNetworkRequest 缓存相关的 请求头参数设置
    [self prepareRequest:request];
    
    return request;
}
```

其实总共分为两种创建 MKNetworkRequest 方法:

- 直接传入`一个字符串 url`
- 使用Host的hostName，然后传入 `path、参数字典、请求方法、是否https`

- 两种情况创建完 MKNetworkRequest 之后，都调用了如下这个方法进行 `缓存请求头参数设置`

```objc
-(void) prepareRequest: (MKNetworkRequest*) request;
```

***

###对 MKNetworkRequest 设置缓存相关的请求头参数

####按照HTTP 1.1 Cache Control 规范，添加请求头参数

####主要是如下这两种 响应头参数/请求头参数 的配合

- Last-Modified 响应头 与 IF-MODIFIED-SINCE 请求头

- ETag 响应头 与 IF-NONE-MATCH 请求头


```objc
// You can override this method to tweak request creation
// But ensure that you call super
-(void) prepareRequest: (MKNetworkRequest*) request
{
    //1. MKNetworkRequest是否设置使用缓存
    if(!request.cacheable || request.ignoreCache)
        return;
    
    //2. 从MKCache实例 responseCache中，
    //使用MKNetworkRequest实例的hash值为key，查询缓存response元数据
    NSHTTPURLResponse *cachedResponse = self.responseCache[@(request.hash)];
    
    //3. 获取响应头中，描述此次响应的 最后时间
    NSString *lastModified = [cachedResponse.allHeaderFields objectForCaseInsensitiveKey:@"Last-Modified"];
    
    //4. 获取响应头中，描述此次响应的 eTag
    NSString *eTag = [cachedResponse.allHeaderFields objectForCaseInsensitiveKey:@"ETag"];

    //5. 添加 请求头参数 IF-MODIFIED-SINCE
    if(lastModified) {
        [request addHeaders:@{@"IF-MODIFIED-SINCE" : lastModified}];
    }
    
    //6. 添加 请求头参数 IF-NONE-MATCH
    if(eTag) {
        [request addHeaders:@{@"IF-NONE-MATCH" : eTag}];
    }
}
```

- 从作者注释看到，这个方法我们可以通过子类进行重写，但是需要调用这个super class.

- 如上方法作者主要是是在`请求头`添加参数，为了`向服务器询问`
	- 当前请求url对应的服务器文件是否做出修改
	- 也就是说能不能够使用本地缓存数据


***

###那么现在就是让MKNetworkRequest执行

###普通数据请求Request执行

```objc
-(void) startRequest:(MKNetworkRequest*) request {
  
    //1. 如果传入nil 或 MKNetworkRequest没有NSMutableURLRequest实例
    if(!request || !request.request)
    {
        NSLog(@"Request is nil, check your URL and other parameters you use to build your request");
        return;
    }

    //2. 如果 MKNetworkRequest 可以使用缓存
    if(request.cacheable && !request.doNotCache)
    {
        //2.1 使用 request.hash值，查询缓存容器，得到 response描述数据
        NSHTTPURLResponse *cachedResponse = self.responseCache[@(request.hash)];
        
        //2.2 获取 cachedResponse 的 过期时间
        NSDate *cacheExpiryDate = cachedResponse.cacheExpiryDate;
        
        //2.3 过期时间 与 当前时间 的差
        NSTimeInterval expiryTimeFromNow = [cacheExpiryDate timeIntervalSinceNow];
        
        //2.4 如果响应数据是 图片，并且不存在 过期时间，设置一个过期时间
        if(cachedResponse.isContentTypeImage && !cacheExpiryDate)
        {
            //2.4.1 根据 cachedResponse 是否有缓存请求头参数设置，来决定其过期时间
            //存在，设置过期时间间隔为 kMKNKDefaultCacheDuration = 10分钟
            //不存在，设置过期时间间隔为 kMKNKDefaultImageCacheDuration = 7天
            expiryTimeFromNow = cachedResponse.hasRequiredRevalidationHeaders ? kMKNKDefaultCacheDuration : kMKNKDefaultImageCacheDuration;
        }
        
        //2.5 如果缓存response不包含如下参数，那么就使用默认的过期时间间隔600秒
        //Cache-Control缓存控制参数
        //以及ETag、Last-Modified缓存控制参数
        if(cachedResponse.hasDoNotCacheDirective || !cachedResponse.hasHTTPCacheHeaders)
        {
            expiryTimeFromNow = kMKNKDefaultCacheDuration;
        }

        //2.6 获取对应缓存data数据
        NSData *cachedData = self.dataCache[@(request.hash)];

        //2.7 判断是否存在缓存 response data 数据
        if(cachedData) {
            
            //将 cachedData 填充到 MKNetworkRequest
            request.responseData = cachedData;
            
            //将 cachedResponse 填充到 MKNetworkRequest
            request.response = cachedResponse;

            //判断缓存数据是否过期
            if(expiryTimeFromNow > 0 && !request.alwaysLoad)
            {
                //缓存数据没有超时，并且 MKNetworkRequest 没有 设置每一次必须重新请求
                //alwaysLoad = NO
                
                //修改request的状态，数据来源于缓存
                request.state = MKNKRequestStateResponseAvailableFromCache;
                
                //如果使用缓存数据，直接结束此次请求
                return; // don't make another request
                
            } else {
                
                //alwaysLoad = YES，每一次都必须重新请求
                //修改request的状态，标识来源于缓存，但是分为两种状态:
                //状态1: 未超时
                //状态2: 已超时
                request.state = expiryTimeFromNow > 0 ? MKNKRequestStateResponseAvailableFromCache :
                MKNKRequestStateStaleResponseAvailableFromCache;
            }
        }
    }
    
    //3. 下面情况是，不存在 缓存数据
    
    //4. 获取 默认 NSURLSession
    NSURLSession *sessionToUse = self.defaultSession;
    
    //4. 如果使用了 https://协议 或 请求需要认证
    if(request.isSSL || request.requiresAuthentication) {
        
        //那么就使用 内存 NSURLSession
        sessionToUse = self.ephemeralSession;
    }

    //5. 由最终的 NSURLSession 创建 NSURLSessionTask
    NSURLSessionDataTask *task = [sessionToUse
                                  dataTaskWithRequest:request.request
                                  completionHandler:^(NSData *data, NSURLResponse *response, NSError *error)
    {
        //5.1 MKNetworkRequest是否已经取消
        if(request.state == MKNKRequestStateCancelled)
        {
            //获取response描述
            request.response = (NSHTTPURLResponse*) response;
            
            //获取请求错误
            if(error) {
                request.error = error;
            }
            
            //获取响应数据
            if(data) {
                request.responseData = data;
            }
            
            //结束往下执行
            return;
        }
        
        //5.2 response 响应为 nil
        if(!response) {
            
            //填充响应数据
            request.response = (NSHTTPURLResponse*) response;
            request.responseData = data;
            request.error = error;
            
            //修改MKNetworkRequest的状态为error
            request.state = MKNKRequestStateError;
            
            return;
        }
        
        //5.3 response 存在，填充 request的response
        request.response = (NSHTTPURLResponse*) response;
        
        //5.4 从 response描述中获取 响应code
        if(request.response.statusCode >= 200 && request.response.statusCode < 300)
        {
        	//5.4.1
            //code: 200 -- 299，表示请求成功
            request.responseData = data;
            request.error = error;
            
        } else if(request.response.statusCode == 304) {
            //5.4.2
            // don't do anything
            //服务器告诉客户端，可以使用缓存数据
            
        } else if(request.response.statusCode >= 400) {
			//5.4.3
            //code: 400以上，表示网络请求错误
            request.responseData = data;
            
            //构造字典，如下两个数据
            NSMutableDictionary *userInfo = [NSMutableDictionary dictionary];
            
            if(response)
                userInfo[@"response"] = response;
            
            if(error)
                userInfo[@"error"] = error;
            
            //构造NSError
            NSError *httpError = [NSError errorWithDomain:@"com.mknetworkkit.httperrordomain"
                                                     code:request.response.statusCode
                                                 userInfo:userInfo];
            
            //将构造的NSError设置给 MKNetworkRequest
            request.error = httpError;
            
            // if subclass of host overrides errorForRequest: they can provide more insightful error objects by parsing the response body.
            // the super class implementation just returns the same error object set in previous line
            //作者的注释说名，我们可以完成:
            //首先，我们自定义一个 MKNetworkHost 的子类
            //然后，我们重写 MKNetworkHost 的方法 errorForCompletedRequest:
            //最后，再 errorForCompletedRequest: 方法中，对请求错误NSError做更多的自定制
            request.error = [self errorForCompletedRequest:request];
        }
        
        //5.5 判断请求是否发生错误
        if(!request.error)
        {
        	//5.5.1 请求正常结束
        	
            //判断MKNetworkRequest是否可以缓存数据
            //必须是GET、且没有加密、没有认证的请求才能被缓存
            if(request.cacheable)
            {
                //将当前MKNetworkRequest对应的响应数据，使用Host对象的MKCache实例缓存起来
                self.dataCache[@(request.hash)] = data;
                self.responseCache[@(request.hash)] = response;
            }
            
            //同步多线程执行 activeTasks数组 删除 当前MKNetworkRequest，执行完毕不需要了
            dispatch_sync(self.runningTasksSynchronizingQueue, ^{
                [self.activeTasks removeObject:request];
            });
            
            //修改MKNetworkRequest状态为 执行完毕
            request.state = MKNKRequestStateCompleted;
            
        } else {
            
            //5.5.2 请求发生错误
            //同步多线程执行 删除 当前MKNetworkRequest，执行完毕不需要了
            dispatch_sync(self.runningTasksSynchronizingQueue, ^{
                [self.activeTasks removeObject:request];
            });
            
            //修改MKNetworkRequest状态为 发生错误
            request.state = MKNKRequestStateError;
        }
    }];

    //6. 将创建出来的NSURLSessionTask实例，交给 MKNetworkRequest保存
    request.task = task;

    //7. 同步多线程执行 activeTasks数组 添加 当前MKNetworkRequest 到数组
    dispatch_sync(self.runningTasksSynchronizingQueue, ^{
        [self.activeTasks addObject:request];
    });
    
    //8. 修改MKNetworkRequest状态为 开始执行
    request.state = MKNKRequestStateStarted;
}
```

###上传请求Request执行

```objc
-(void) startUploadRequest:(MKNetworkRequest*) request {
  
    //1. 必须满足条件
    if(!request || !request.request) {
    
        NSAssert((request && request.request),
             @"Request is nil, check your URL and other parameters you use to build your request");
        return;
    }
  
    //2. 直接使用 后台 session 创建 上传Task
    //上传的数据NSData，使用 MKNetworkRequest保存的
    request.task = [self.backgroundSession uploadTaskWithRequest:request.request
                                                      fromData:request.multipartFormData];
    
    //3. 同步多线程执行 将request 添加到 activeTasks数组
    dispatch_sync(self.runningTasksSynchronizingQueue, ^{
        [self.activeTasks addObject:request];
    });
  
    //4. 修改 MKNetworkRequest的状态为 开始执行
    request.state = MKNKRequestStateStarted;
}
```

###下载请求Request执行

```objc
-(void) startDownloadRequest:(MKNetworkRequest*) request {
  
    //1. 获取当前应用程序的AppDelegate实例，是否实现了后台下载的代理函数
    //application:handleEventsForBackgroundURLSession:completionHandler:
    static dispatch_once_t onceToken;
    static BOOL methodImplemented = YES;
    dispatch_once(&onceToken, ^{
        methodImplemented = [[[UIApplication sharedApplication] delegate]
                             respondsToSelector:
                             @selector(application:handleEventsForBackgroundURLSession:completionHandler:)];
    });
    
    //2. 如果没有实现，所以让MKNetworkKit使用后台session完成下载，需要做如下两步:
    //步骤1: AppDelegate实现 application:handleEventsForBackgroundURLSession:completionHandler:函数
    //步骤2: 再此实现函数中，将传入的completionBlock交给MKNetworkHost实例保存
    if(!methodImplemented) {
        NSLog(@"application:handleEventsForBackgroundURLSession:completionHandler: is not implemented in your application delegate. Download tasks might not work properly. Implement the method and set the completionHandler value to MKNetworkHost's backgroundSessionCompletionHandler");
    }

    //3. 必须创建request、和request的NSURLRequest
    if(!request || !request.request) {
        NSLog(@"Request is nil, check your URL and other parameters you use to build your request");
        return;
    }

    //4. 由后台session创建 下载Task
    request.task = [self.backgroundSession downloadTaskWithRequest:request.request];
    
    //5. 同步多线程执行 activeTasks数组 添加 MKNetworkRequest
    dispatch_sync(self.runningTasksSynchronizingQueue, ^{
        [self.activeTasks addObject:request];
    });
    
    //6. 修改 MKNetworkRequest的状态为 开始执行
    request.state = MKNKRequestStateStarted;
}
```

###重写MKNetworkHost方法完成请求错误时更多的自定义

```objc
-(NSError*) errorForCompletedRequest: (MKNetworkRequest*) completedRequest {
	
	//1. 
	NSError * error = completedRequest.error;
	
	//2. 对error更多的设置（这个方法主要完成的事）
	//....
	
	//3. 返回最后定制化的error
	return error;
}
```

***

###NSURLSessionDelegate 协议方法实现

####接收服务器挑战

```objc
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
 completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential *credential))completionHandler
{
    //1. 判断当前挑战方式，是否为 服务器信任
    if([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust])
    {
        //使用服务器信任 创建挑战凭证
        NSURLCredential *credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
        
        //将凭证发送给服务端，完成挑战
        completionHandler(NSURLSessionAuthChallengeUseCredential,credential);
    }
    
    //----------------------- 下面是 非 信任形式的 挑战 -----------------------
    
    //2. 从保存的所有 MKNetworkRequest中，找到当前需要接收挑战的 MKNetworkRequest
    __block MKNetworkRequest *matchingRequest = nil;
    [self.activeTasks enumerateObjectsUsingBlock:^(MKNetworkRequest *request, NSUInteger idx, BOOL *stop)
    {
        if([request.task isEqual:task]) {
          matchingRequest = request;
          *stop = YES;
        }
    }];
  
    //3. Basic 和 Digest 这两种形式的挑战，其实可以看做成一种
    //Basic形式挑战，需要用户名和密码
    //Digest形式挑战，需要用户名和密码，摘要由系统自动生成
    if([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodHTTPBasic] ||
     [challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodHTTPDigest])
    {
        //挑战失败次数是否过多，规定最多3次
        if([challenge previousFailureCount] == 3)
        {
            //不提交凭证，完成挑战
            completionHandler(NSURLSessionAuthChallengeRejectProtectionSpace, nil);
            
        } else {
          
            //使用用户名与密码，创建凭证
            //并且不缓存凭证，只使用一次
            NSURLCredential *credential = [NSURLCredential credentialWithUser:matchingRequest.username
                                                                   password:matchingRequest.password
                                                                persistence:NSURLCredentialPersistenceNone];
            //发送凭证，完成挑战
            if(credential) {
                completionHandler(NSURLSessionAuthChallengeUseCredential, credential);
            } else {
                completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
            }
        }
    }
}
```

***

###Task错误结束

```objc
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error {
    
    //1. 遍历找到当前Task对应的MKNetworkRequest实例
    __block MKNetworkRequest *matchingRequest = nil;
    [self.activeTasks enumerateObjectsUsingBlock:^(MKNetworkRequest *request, NSUInteger idx, BOOL *stop) {

        if([request.task isEqual:task]) {
          
            //清空响应数据
            request.responseData = nil;
            
            //保存响应描述
            request.response = (NSHTTPURLResponse*) task.response;
            
            //保存响应错误
            request.error = error;
            
            //修改MKNetworkRequest状态
            //注意: 修改state的方法中，会完成 completion block执行
            //也就是说回调会执行完毕
            if(error) {
                request.state = MKNKRequestStateError;
            } else {
                request.state = MKNKRequestStateCompleted;
            }
            
            //记录找到的MKNetworkRequest
            matchingRequest = request;
            
            //结束遍历
            *stop = YES;
        }
    }];

    //2. 等待上面的代码执行完毕，也就是等待MKNetworkRequest实例的block执行完毕
    //然后再删掉 MKNetworkRequest实例
    dispatch_sync(self.runningTasksSynchronizingQueue, ^{
        [self.activeTasks removeObject:matchingRequest];
    });
}
```

####上传Task的不断回传当前下载的部分数据

```objc
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
   didSendBodyData:(int64_t)bytesSent
    totalBytesSent:(int64_t)totalBytesSent
totalBytesExpectedToSend:(int64_t)totalBytesExpectedToSend
{
    //1. 计算上传进度
    float progress = (float)(((float)totalBytesSent) / ((float)totalBytesExpectedToSend));
    
    //2. 遍历找到当前Task的MKNetworkRequest实例
    //然后不断的 执行 设置当前 最新的 上传进度
    [self.activeTasks enumerateObjectsUsingBlock:^(MKNetworkRequest *request, NSUInteger idx, BOOL *stop) {
    
        if([request.task isEqual:task]) {
            
            //在set方法中，执行block回传进度
            [request setProgressValue:progress];
            *stop = YES;
        }
    }];
}
```

###每一个下载Task执行完毕

```objc
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
    didFinishDownloadingToURL:(NSURL *)location
{
    //1. 找到下载Task对应的MKNetworkRequest
    [self.activeTasks enumerateObjectsUsingBlock:^(MKNetworkRequest *request, NSUInteger idx, BOOL *stop) {

        if([request.task.currentRequest.URL.absoluteString isEqualToString:downloadTask.currentRequest.URL.absoluteString]) {
          
            //将下载的临时文件，复制到 MKNetworkRequest对象指定的路径
            NSError *error = nil;
            BOOL isOk = [[NSFileManager defaultManager] moveItemAtPath:location.path
                                                                toPath:request.downloadPath
                                                                 error:&error];
            if(!isOk) {
                NSLog(@"Failed to save downloaded file at requested path [%@] with error %@", request.downloadPath, error);
            }

            *stop = YES;
        }
    }];
  
    //2. 获取后台session中 所有未解决的 task
    [self.backgroundSession getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) {
        
        //2.1 数据请求个数 + 上传请求个数 + 下载请求个数
        NSUInteger count = dataTasks.count + uploadTasks.count + downloadTasks.count;
        
        //2.2 当所有任务完成的时候执行AppDelegate传入的Block
        if (count == 0) {
            
            //2.2.1 获取AppDelegate设置给MKNetworkHost实例的block
            void (^backgroundSessionCompletionHandlerCopy)() = self.backgroundSessionCompletionHandler;
            
            //2.2.2 执行block
            if (self.backgroundSessionCompletionHandler) {
                self.backgroundSessionCompletionHandler = nil;
                backgroundSessionCompletionHandlerCopy();
            }
        }
    }];
}
```

###下载任务每一次接收部分数据

```objc
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
      didWriteData:(int64_t)bytesWritten
 totalBytesWritten:(int64_t)totalBytesWritten
totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite
{
    //1. 当前下载进度
    float progress = (float)(((float)totalBytesWritten) / ((float)totalBytesExpectedToWrite));
    
    //2. 遍历找到当前Task的MKNetworkRequest实例
    //然后回传设置当先上传进度
    [self.activeTasks enumerateObjectsUsingBlock:^(MKNetworkRequest *request, NSUInteger idx, BOOL *stop) {

        if([request.task.currentRequest.URL.absoluteString isEqualToString:downloadTask.currentRequest.URL.absoluteString])
        {
            //在set方法中，执行block回传进度
            [request setProgressValue:progress];
        }
    }];
}
```

### `setProgressValue:`更新 上传/下载 进度

```
1. 这个方法是给 MKNetworkRequest 使用分类 扩展的方法
2. 用于回传进度值
```

```
代码实现可见 MKNetworkRequest 源码分析文章.
```

###NSURLSession结束工作

```objc
- (void)URLSession:(NSURLSession *)session didBecomeInvalidWithError:(NSError *)error {
  
    if(session == self.backgroundSession) {
        NSLog(@"Session became invalid with error: %@", error);
    }
}
```

***

###如果需要完成如上后台下载任务，需要在 `AppDelegate.m`实现如下代理回调函数（可以看NSURLSession Background Task 这篇文章）

```objc
#import "MKNetworkHost.h"

@interface AppDelegate ()

@property (nonatomic, strong) MKNetworkHost *networkHost;

@end
```

```objc
- (void)application:(UIApplication *)application
handleEventsForBackgroundURLSession:(NSString *)identifier
  completionHandler:(void (^)())completionHandler
{
    //将传入的block穿给Host后续处理
    self.networkHost.backgroundSessionCompletionHandler = completionHandler;
}
```

***

###最后我们可以 子类化MKNetwokHost ，重写如下方法提供更多的逻辑

####提供请求头个性化设置

```objc
-(void) prepareRequest: (MKNetworkRequest*) networkRequest;
```

####提供请求错误的个性化设置

```objc
-(NSError*) errorForCompletedRequest: (MKNetworkRequest*) completedRequest;
```

****

###`runningTasksSynchronizingQueue`这个队列用来管理 所有的MKNetworkRequest实例.

- 注意是一个 `串行` 队列，为了 `同步多线程`

- 同步 执行 数组创建

```objc
dispatch_async(self.runningTasksSynchronizingQueue, ^{
  self.activeTasks = [NSMutableArray array];
});
```

- 同步 执行 数组添加

```objc
dispatch_sync(self.runningTasksSynchronizingQueue, ^{
    [self.activeTasks addObject:request];
});
```

- 同步 执行 数组删除

```objc
dispatch_sync(self.runningTasksSynchronizingQueue, ^{
    [self.activeTasks removeObject:request];
});
```

作者使用这个串行队列就是为了同步多线程管理Request对象的`增删`...我还以为是如何管理`请求的发起`，就类似之前NSOperationQueue那样，看来是我想多了..

***

###问题1: 在创建上传和下载 MKNetworkRequest的时候，没有设置Task的代理实现对象

```objc
NSURLSessionDownloadDelegate
NSURLSessionUploadDelegate

中的协议方法会被执行？
```

###问题2: NSURLSession中delegate的持有方式是使用`retain`强引用

```objc
没有看到对NSURLSession对象结束执行的代码

- [NSURLSession finishTasksAndInvalidate]
或
- [NSURLSession invalidateAndCancel]
```