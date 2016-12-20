---
layout: post
title: "NSURLCache分析"
date: 2015-08-31 15:42:02 +0800
comments: true
categories: 
---


###NSURLCache如其名字针对请求URL进行缓存数据.但是NSURLCache并没有帮我完成response数据的缓存，而只是提供给我们什么时候可以让我们手动完成缓存数据.

> NSURLCache只是给我们提供了网络请求、响应两个时刻的回调函数.

***

###No1. 在App启动的时候，注册我们自己的NSURLCache对象.

```
在Appdelegate的 didFinishLaunching函数添加如下代码:

//1. 
NSURLCache *URLCache = [[NSURLCache alloc] initWithMemoryCapacity:4 * 1024 * 1024
                                                         diskCapacity:20 * 1024 * 1024
                                                             diskPath:nil];
    
//2. 
[NSURLCache setSharedURLCache:URLCache];
```

否则只会回调系统自己的NSURLCache对象.

***

###No2. 当发起一个请求后，得到了正确的response数据会触发如下方法执行:


> 将请求得到的response数据缓存到本地.

```
- (void)storeCachedResponse:(NSCachedURLResponse *)cachedResponse 
				 forRequest:(NSURLRequest *)request
```

***

###No3. 当一个NSURLRequest对象要执行请求时，首先尝试获取有没有对应的缓存response数据.就会触发如下方法:

> 从本地查找是否存在缓存数据.

```
- (NSCachedURLResponse *)cachedResponseForRequest:(NSURLRequest *)request;
```

如果查找到对应缓存数据，先返回缓存数据，再取请求最新的服务器数据并缓存下来.

***

###No4. iOS系统使用NSURLCache缓存的response数据保存的位置、形式.

* 缓存数据的形式

```
是使本地sqlite的形式存放.
```
* 缓存路径

```
/rom根目录/App/Library/Caches/AppBundle/Cache.db
```

* 完成SNURLCache的系统表.

![表](http://i1.tietuku.com/a3d4af2b2cb4ae1f.png)

四张表：

```
cfurl_cache_blob_data

```

```
cfurl_cache_receiver_data
```

```
cfurl_cache_response
```

```
cfurl_cache_schema_version
```

iOS系统默认使用这四张表存储cached response数据.

![表](http://i1.tietuku.com/925a316e79f42782.png)
![表](http://i1.tietuku.com/5ff8ba4e82ae5cc2.png)
![表](http://i1.tietuku.com/45edf95eddd47409.png)


***

###No5. 当我们使用NSURLCache之后，并设置了NSURLRequest.cachePolicy缓存策略.那么iOS系统就会使用这个本地数据库给我们完成数据缓存（完成缓存不是NSUELCache类）.

***

###No6. 我们可以写一个继承自NSURLCache的子类，并使用我们自己写的缓存response data、查找response data的逻辑代码.

有时间在再说...