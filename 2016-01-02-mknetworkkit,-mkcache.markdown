---
layout: post
title: "MKNetworkKit、MKCache"
date: 2016-01-02 22:25:47 +0800
comments: true
categories: 
---

###首先看下`MKNetworkHost.m`中使用 `MKCache`的代码

****

###缓存相关参数设置

```objc
NSUInteger const kMKNKDefaultCacheDuration = 600; // 10 minutes
NSUInteger const kMKNKDefaultImageCacheDuration = 3600*24*7; // 7 days
NSString *const kMKCacheDefaultDirectoryName = @"com.mknetworkkit.mkcache";
```

***

###MKNetworkHost实例初始化时 创建了两个 MKCache实例，用来缓存 `响应描述` 和 `响应数据`

```objc
@interface MKNetworkHost (/*Private Methods*/) <NSURLSessionDelegate>

/**
 *  用来缓存 response data
 */
@property MKCache *dataCache;

/**
 *  用来缓存 response描述
 */
@property MKCache *responseCache;

@end
```

***

###MKNetworkHost实例，开启缓存功能

```objc
-(void) enableCache {
  [self enableCacheWithDirectory:kMKCacheDefaultDirectoryName inMemoryCost:0];
}

-(void) enableCacheWithDirectory:(NSString*) cacheDirectoryPath inMemoryCost:(NSUInteger) inMemoryCost
{
    //1. data Cache
    NSString *dataCacgePath = [NSString stringWithFormat:@"%@/data", cacheDirectoryPath];
    self.dataCache = [[MKCache alloc] initWithCacheDirectory:dataCacgePath
                                                inMemoryCost:inMemoryCost];
    
    //2. response Cache
    NSString *responseCachePath = [NSString stringWithFormat:@"%@/responses", cacheDirectoryPath];
    self.responseCache = [[MKCache alloc] initWithCacheDirectory:responseCachePath
                                                  inMemoryCost:inMemoryCost];
}
```

***

###执行MKNetworkRequest时，使用缓存的逻辑

- 对于 `上传、下载` MKNetworkRequest实例，`不`使用缓存

- 使用缓存主要针对 `普通数据请求 + GET方法` 的 MKNetworkRequest实例

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
        //由此可见，缓存超时逻辑 是由 NSHTTPURLResponse分类 控制的
        NSDate *cacheExpiryDate = cachedResponse.cacheExpiryDate;
        
        //2.3 过期时间 与 当前时间 的 时间差
        NSTimeInterval expiryTimeFromNow = [cacheExpiryDate timeIntervalSinceNow];
        
        //2.4 如果响应数据类型是 `图片`，并且不存在 过期时间，那么就计算一个过期的时间差
        if(cachedResponse.isContentTypeImage && !cacheExpiryDate)
        {
            //2.4.1 根据 cachedResponse 是否有 `缓存请求头参数` 设置，来决定其过期时间
            //存在，设置过期 时间差 为 kMKNKDefaultCacheDuration = 10分钟
            //不存在，设置过期 时间差 为 kMKNKDefaultImageCacheDuration = 7天
            expiryTimeFromNow = cachedResponse.hasRequiredRevalidationHeaders ? kMKNKDefaultCacheDuration : kMKNKDefaultImageCacheDuration;
        }
        
        //2.5 非图片响应类型的缓存 的 时间差
        if(cachedResponse.hasDoNotCacheDirective || !cachedResponse.hasHTTPCacheHeaders)
        {
            expiryTimeFromNow = kMKNKDefaultCacheDuration;
        }

        //2.6 获取对应缓存data数据
        NSData *cachedData = self.dataCache[@(request.hash)];

        //2.7 判断是否存在缓存 response data 数据
        if(cachedData) {
            
            //2.7.1 将 cachedData 填充到 当前MKNetworkRequest新实例
            request.responseData = cachedData;
            
            //2.7.2 将 cachedResponse 填充到 当前MKNetworkRequest新实例
            request.response = cachedResponse;

            //2.7.3 判断是否使用 缓存数据 结束此次请求
            if(expiryTimeFromNow > 0 && !request.alwaysLoad)
            {
                //( alwaysLoad 为 NO ) 并且 ( MKNetworkRequest实例没有设置每一次请求都必须 重新 加载最新数据 )
                //第一件事、修改request的状态，数据来源于缓存
				//第二件事、修改状态内部执行block回传缓存数据，结束此次请求
                request.state = MKNKRequestStateResponseAvailableFromCache;
                
                //使用缓存数据，就直接结束此次请求
                return; // don't make another request
                
            } else {
                
                //( alwaysLoad == YES ) 或 (expiryTimeFromNow <= 0)
                //第一件事、修改request的状态，标识来源于缓存，但是分为两种状态:
                //状态1: 未超时 + alwaysLoad = NO
                //状态2: 已超时
                //第二件事、修改状态内部执行block 先 回传缓存数据
                
                request.state = expiryTimeFromNow > 0 ? MKNKRequestStateResponseAvailableFromCache :
                MKNKRequestStateStaleResponseAvailableFromCache;
                
                //第三件事、然后再继续执行请求数据
                //往后继续执行代码请求服务器数据...
            }
        }
    }
    
    //--------------- 以下是 ( 不存在缓存数据 ) 或 ( 需要重新请求服务器数据 ) ---------------
    
    //3. 获取 默认 NSURLSession
    //注意，不一定最终使用这个 session 创建 Task，只是一个默认值 session
    NSURLSession *sessionToUse = self.defaultSession;
    
    //4. 使用 内存 session 的场景
    //场景1: 使用https://协议 网络通信
    //场景2: 请求需要进行认证
    if(request.isSSL || request.requiresAuthentication) {
        
        //使用 内存 NSURLSession 实例
        //可以保证数据的安全隐秘性，因为只有内存中缓存
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
        
        //5.2 response 响应为 nil，那么错误结束
        if(!response) {
            
            //填充数据
            request.response = (NSHTTPURLResponse*) response;
            request.responseData = data;
            request.error = error;
            
            //修改MKNetworkRequest的状态为error
            request.state = MKNKRequestStateError;
            
            return;
        }
        
        //5.3 response 存在，填充 response
        //替换之前的 缓存response
        request.response = (NSHTTPURLResponse*) response;
        
        //5.4 从 response描述中获取 响应code
        if(request.response.statusCode >= 200 && request.response.statusCode < 300)
        {
            //code: 200 -- 299，表示请求成功
            
            //替换之前的 缓存response data
            request.responseData = data;
            
            request.error = error;
            
        } else if(request.response.statusCode == 304) {
            
            // don't do anything
            
        } else if(request.response.statusCode >= 400) {

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
            //从作者的注释中，我们可以看出:
            //首先，我们自定义一个 MKNetworkHost 的子类
            //然后，我们重写 MKNetworkHost 的方法 errorForCompletedRequest:
            //最后，再 errorForCompletedRequest: 方法中，对请求错误NSError做更多的自定制
            request.error = [self errorForCompletedRequest:request];
        }
        
        //5.5 判断请求是否发生错误
        if(!request.error)
        {	
        	//未发生错误
        	
            //5.5.1 判断MKNetworkRequest 是否可以 缓存数据
            if(request.cacheable)
            {
                //将当前MKNetworkRequest对应的响应数据
                //使用MKCache实例缓存如下数据
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
        
        	//发生错误
            
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

###从上可以得到关于 MKCache 的基本使用

- 查询缓存response

```objc
NSHTTPURLResponse *cachedResponse = self.responseCache[@(request.hash)];
```

- 查询缓存data数据

```objc
NSData *cachedData = self.dataCache[@(request.hash)];
```

- 对 新的response和新的data 进行缓存

```objc
self.dataCache[@(request.hash)] = data;

self.responseCache[@(request.hash)] = response;
```

- 还有关于 `缓存过期逻辑` 是由`NSHTTPURLResponse+MKNKAdditions`封装的

```
后续再说吧，涉及的比较多.
```

****

###`MKCache.h`向外暴露的方法

```objc
@interface MKCache : NSObject

@property NSString *directoryPath;
@property NSUInteger cacheMemoryCost;

//传入 1)缓存文件存放目录 2)内存最大缓存长度 两个参数
-(instancetype) initWithCacheDirectory:(NSString*) cacheDirectory
                          inMemoryCost:(NSUInteger) inMemoryCost;

//如下两个方法非常有用
//可以让普通的NSObject对象，当成NSDictionry一样使用
//如: 取值， value = dict[@"key"];
//如: 设值， dict[@"key"] = value;
- (id) objectForKeyedSubscript:(id<NSCopying>) key;
- (void)setObject:(id<NSCoding>)obj forKeyedSubscript:(id<NSCopying>) key;

@end
```

###关于让任意NSObject可以像 `NSDictionary`一样读取`key-value`键值对

####自定义Person类，实现如下两个方法即可

- Person.h

```objc
#import <Foundation/Foundation.h>

@interface Person : NSObject

//类似字典读
- (id) objectForKeyedSubscript:(id<NSCopying>) key;

//类似字典写
- (void)setObject:(id<NSCoding>)obj forKeyedSubscript:(id<NSCopying>) key;

@end
```

- Person.m

```objc
#import "Person.h"

@implementation Person

- (id)objectForKeyedSubscript:(id<NSCopying>)key {
    
    //内部自己使用数组或字典控制
    //key与value的关系
    
    return nil;
}

- (void)setObject:(id<NSCoding>)obj forKeyedSubscript:(id<NSCopying>)key {
    
    //内部自己使用数组或字典控制
    //key与value的关系
}

@end
```

- 使用Person对象

```objc
Person *p = [[Person alloc] init];
    
p[@"key"] = @"value";
    
NSString *value = p[@"key"];
```

- 小结 keyedSubscript: 

	- 编译代码，可以编译成功.
	- 那么说明可以将 `任意的NSObject` 伪装成 `NSDictionary` 一样 管理 `键值对`

****

###开始阅读`MKCache.m`

***

###一些基础配置

```objc
//磁盘缓存文件的统一后缀名（.mkcache）
NSString *const kMKCacheDefaultPathExtension = @"mkcache";

//内存缓存最大的记录个数
NSUInteger const kMKCacheDefaultCost = 10;
```

***

###MKCache匿名分类

```objc
@interface MKCache (/*Private Methods*/)

//1. 内存缓存字典（缓存容器）
@property NSMutableDictionary *inMemoryCache;

//2. 保存最近查询缓存字典的key
@property NSMutableArray *recentlyUsedKeys;

//3. 同步多线程访问缓存的 串行 队列
@property dispatch_queue_t queue;
@end
```

***

###禁止使用 init 方法来初始化

```objc
-(instancetype) init {
  
    @throw [NSException exceptionWithName:NSInternalInconsistencyException
                                   reason:@"MKCache should be initialized with the designated initializer initWithCacheDirectory:inMemoryCost:"
                                 userInfo:nil];
    return nil;
}
```

其实还可以使用`UNAVAILABLE_ATTRIBUTE`在头文件中修饰init函数，来禁止使用.

###使用指定`缓存目录`与`缓存开销`的init函数进行初始化

```objc
-(instancetype) initWithCacheDirectory:(NSString*) cacheDirectory
                          inMemoryCost:(NSUInteger) inMemoryCost
{
    NSParameterAssert(cacheDirectory != nil);
  
    if(self = [super init]) {
    
        //1. 系统 Cache 目录
        NSArray *paths = NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES);
        
        //2. 缓存目录位于 系统Cache目录下/传入目录名
        self.directoryPath = [paths.firstObject stringByAppendingPathComponent:cacheDirectory];
        
        //3. 当前缓存最大长度，默认最大缓存10个对象
        self.cacheMemoryCost = inMemoryCost ? inMemoryCost : kMKCacheDefaultCost;
    
        //4. 创建缓存长度大小的可变字典作为缓存容器
        self.inMemoryCache = [NSMutableDictionary dictionaryWithCapacity:self.cacheMemoryCost];
        
        //5. 创建一个数组保存最近用来查询缓存字典的key
        self.recentlyUsedKeys = [NSMutableArray arrayWithCapacity:self.cacheMemoryCost];
    
        //6. 判断缓存目录是否已经存在，如果不存在就创建缓存目录
        BOOL isDirectory = YES;
        BOOL directoryExists = [[NSFileManager defaultManager] fileExistsAtPath:self.directoryPath
                                                                    isDirectory:&isDirectory];
        if(!isDirectory) {
            NSError *error = nil;
            if(![[NSFileManager defaultManager] removeItemAtPath:self.directoryPath error:&error]) {
                NSLog(@"%@", error);
            }
            directoryExists = NO;
        }
    
        if(!directoryExists)
        {
            NSError *error = nil;
            if(![[NSFileManager defaultManager] createDirectoryAtPath:self.directoryPath
                                          withIntermediateDirectories:YES
                                                           attributes:nil
                                                                error:&error])
            {
                NSLog(@"%@", error);
            }
        }
    
        //7. 创建一个全局串行队列，同步多线程完成缓存容器的读写
        self.queue = dispatch_queue_create("com.mknetworkkit.cachequeue", DISPATCH_QUEUE_SERIAL);
        
        //8. 注册通知，接收到如下通知时，刷新缓存容器
        //通知一、接收到系统内存警告
        //通知二、应用程序即将进入后台
        //通知三、应用程序即将退出
#if TARGET_OS_IPHONE
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(flush)
                                                     name:UIApplicationDidReceiveMemoryWarningNotification
                                                   object:nil];
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(flush)
                                                     name:UIApplicationDidEnterBackgroundNotification
                                                   object:nil];
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(flush)
                                                     name:UIApplicationWillTerminateNotification
                                                   object:nil];
    
#elif TARGET_OS_MAC
    
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(flush)
                                                     name:NSApplicationWillHideNotification
                                                   object:nil];
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(flush)
                                                     name:NSApplicationWillResignActiveNotification
                                                   object:nil];
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(flush)
                                                     name:NSApplicationWillTerminateNotification
                                                   object:nil];
    
#endif
    }
  
    return self;
}
```

###那么看看接收到系统通知时，如何刷新缓存容器？

```objc
/**
 *  将内存最新的缓存数据，
 *  全部写入磁盘文件
 */
-(void) flush {
  
    //1. 遍历 内存中 的 缓存字典 的每一个缓存记录
    //首先删除掉对应的磁盘缓存文件
    //然后再将内存最新的缓存数据写入磁盘缓存文件
    [self.inMemoryCache enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
        
        //1.1 内存中的每一个缓存key
        NSString *stringKey = [NSString stringWithFormat:@"%@", key];
        
        //1.2 使用key得到缓存文件路径，如: /..系统路径../MKNetworkCache/books.mkcache（缓存文件后缀名为.mkcache）
        NSString *filePath = [[self.directoryPath stringByAppendingPathComponent:stringKey]
                                stringByAppendingPathExtension:kMKCacheDefaultPathExtension];
        
        //1.3 先删掉 内存中缓存项目 对应的 磁盘缓存文件
        //也就是说以内存缓存为准
        if([[NSFileManager defaultManager] fileExistsAtPath:filePath])
        {
            NSError *error = nil;
            if(![[NSFileManager defaultManager] removeItemAtPath:filePath error:&error])
            {
                NSLog(@"%@", error);
            }
        }

        //1.4 取出内存缓存key对应的当前最新对象
        NSData *dataToBeWritten = nil;
        id objToBeWritten = self.inMemoryCache[key];
        
        //1.5 使用归档的方式写入磁盘文件
        //注意: 缓存的对象Class必须实现NSCoding协议，才能使用归档
        dataToBeWritten = [NSKeyedArchiver archivedDataWithRootObject:objToBeWritten];
        [dataToBeWritten writeToFile:filePath atomically:YES];
    }];

    //2. 清除内存缓存字典
    [self.inMemoryCache removeAllObjects];
    
    //3. 清除内存缓存最近使用key
    [self.recentlyUsedKeys removeAllObjects];
}
```

###MKCache对象销毁时移除监听通知

```objc
-(void) dealloc {
  
#if TARGET_OS_IPHONE
  [[NSNotificationCenter defaultCenter] removeObserver:self name:UIApplicationDidReceiveMemoryWarningNotification object:nil];
  [[NSNotificationCenter defaultCenter] removeObserver:self name:UIApplicationDidEnterBackgroundNotification object:nil];
  [[NSNotificationCenter defaultCenter] removeObserver:self name:UIApplicationWillTerminateNotification object:nil];
#elif TARGET_OS_MAC
  [[NSNotificationCenter defaultCenter] removeObserver:self name:NSApplicationWillHideNotification object:nil];
  [[NSNotificationCenter defaultCenter] removeObserver:self name:NSApplicationWillResignActiveNotification object:nil];
  [[NSNotificationCenter defaultCenter] removeObserver:self name:NSApplicationWillTerminateNotification object:nil];
#endif
}
```

***

###下面是最关键的两个步骤了，对象的存入 和 对象的读取

***

###对象的存入

```objc
- (void)setObject:(id <NSCoding>) obj forKeyedSubscript:(id <NSCopying>) key {
    
    /**
     * 全部的缓存读取操作，都在一个 串行队列 上 异步执行
     * 
     * 1. 所有的操作都会在一个 子线程 上执行（注意: 只有一个子线程，因为是串行队列）
     * 2. 不在当前线程上执行，这样就提高了效率
     * 3. 所有入队调度的block会按照进入队列的顺序依次执行，很好的完成多线程同步
     */
    dispatch_async(self.queue, ^{
        
        //1. 将传入的对象使用传入的key，存入到缓存字典
        self.inMemoryCache[key] = obj;

        //2.-------------------- 使用队列机制，管理缓存key -------------------
        /** 也就是说将传入的缓存key放到队列尾部，作为最后一个使用的key */
        /**
         * 但是这里作者并没有真正使用queue数据结构，只是简单的数组表示
         *  
         *  最新使用的位置: recentlyUsedKeys[0]元素
         *  最久使用的位置: [recentlyUsedKeys数组 lastObject]
         */
        
        //2.1 获取当前缓存key位于 最近使用key缓存数组 的位置
        NSUInteger index = [self.recentlyUsedKeys indexOfObject:key];
        
        //2.2 从当前位置移除
        if(index != NSNotFound) {
            [self.recentlyUsedKeys removeObjectAtIndex:index];
        }

        //2.3 放到数组第0个位置，作为最后（最新）使用的缓存项
        [self.recentlyUsedKeys insertObject:key atIndex:0];

        //3. 清理超出长度的缓存对象，处理最后被使用的缓存对象
        if(self.recentlyUsedKeys.count > self.cacheMemoryCost)
        {
            //3.1 获取最后被使用的缓存key
            id<NSCopying> lastUsedKey = self.recentlyUsedKeys.lastObject;
            
            //3.2 将内存中缓存的最后一个被使用的缓存对象，使用归档api转换成NSData数据
            id objectThatNeedsToBeWrittenToDisk = [NSKeyedArchiver archivedDataWithRootObject:self.inMemoryCache[lastUsedKey]];
            
            //3.3 从内存移除掉最后使用的缓存对象
            [self.inMemoryCache removeObjectForKey:lastUsedKey];

            //3.4 得到NSData数据即将写入磁盘的文件路径
            NSString *stringKey = [NSString stringWithFormat:@"%@", lastUsedKey];
            NSString *filePath = [[self.directoryPath stringByAppendingPathComponent:stringKey] stringByAppendingPathExtension:kMKCacheDefaultPathExtension];
            
            //3.5 先删除已经存在的磁盘缓存文件
            if([[NSFileManager defaultManager] fileExistsAtPath:filePath]) {
                NSError *error = nil;
                if(![[NSFileManager defaultManager] removeItemAtPath:filePath error:&error])
                {
                    NSLog(@"Cannot remove file: %@", error);
                }
            }
            
            //3.6 将最新的NSData数据写入磁盘缓存文件
            [objectThatNeedsToBeWrittenToDisk writeToFile:filePath atomically:YES];
            
            //3.7 移除掉最久被使用的缓存key
            [self.recentlyUsedKeys removeLastObject];
        }
    });
}
```

####总体上大致分为三步，全部是`同步顺序`执行:

- 1) 将 新的缓存项 存入到 缓存字典

- 2) 将当前使用的缓存key，位置调整到 recentlyUsedKeys数组 的第0号位置，作为最新使用的缓存key

- 3) 清理超出归档长度的缓存对象，将其写入磁盘缓存文件

***

###对象的读取

```objc
-(id <NSCoding>) objectForKeyedSubscript:(id <NSCopying>) key {
  
    /** 读取没有进行多线同步 */
    
    //1. 先从内存缓存字典读取
    id cacheData = self.inMemoryCache[key];
    
    if(cacheData)
        return cacheData;
  
    //2. 如果内存缓存没有找到，再从磁盘缓存读取
    NSString *stringKey = [NSString stringWithFormat:@"%@", key];

    NSString *filePath = [[self.directoryPath stringByAppendingPathComponent:stringKey]
                          stringByAppendingPathExtension:kMKCacheDefaultPathExtension];
  
    if([[NSFileManager defaultManager] fileExistsAtPath:filePath])
    {
        //从磁盘缓存文件 反序列化成 内存对象
        cacheData = [NSKeyedUnarchiver unarchiveObjectWithData:[NSData dataWithContentsOfFile:filePath]];
        
        //然后保存到缓存字典
        //这里其实又会调用 setObject:forKeyedSubscript: 方法 写入内存缓存
        //所以同样也做了缓存长度超过处理
        self.inMemoryCache[key] = cacheData;
        
        return cacheData;
    }
  
    //内存缓存 与 磁盘文件缓存，都没有找到
    return nil;
}
```

####总体上大致分为三步，不使用同步多线程:

- 1) 先从 内存缓存 查找

- 2) 如果内存没找到，再从磁盘缓存文件查找
	- 如果找到了磁盘缓存文件
	- 将磁盘文件 反序列化 为 内存对象
	- 并将反序列化出来的内存对象 载入到内存
	- 调用`setObject:forKeyedSubscript:`载入内存，做长度超长处理

- 3) 如果都没找到，返回nil