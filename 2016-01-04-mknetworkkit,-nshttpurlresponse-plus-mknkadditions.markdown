---
layout: post
title: "MKNetworkKit、NSHTTPURLResponse+MKNKAdditions"
date: 2016-01-04 11:59:50 +0800
comments: true
categories: 
---

###这个分类主要是扩展`NSHTTPURLResponse`，封装了一些缓存超时的逻辑，以及响应数据描述的一些快捷方法.

> 核心: HTTP 1.1 Cache Control 响应头/请求参数

***

###首先看头文件向外暴露的api

```objc
@interface NSHTTPURLResponse (MKNKAdditions)

/**
 *  响应数据类型是否是图片
 */
@property (readonly) BOOL isContentTypeImage;

/**
 *  响应描述是否 没有 Cache-Control 指令
 */
@property (readonly) BOOL hasDoNotCacheDirective;

/**
 *  响应描述是否包含 需要重新认证 的请求头参数
 */
@property (readonly) BOOL hasRequiredRevalidationHeaders;

/**
 *  响应描述是否包含 缓存相关 的请求头参数
 */
@property (readonly) BOOL hasHTTPCacheHeaders;

/**
 *  缓存过期的时间
 */
@property (readonly) NSDate* cacheExpiryDate;

@end
```

###那么现在就需要了解下 `HTTP 1.1 Cache Control`相关知识.

***

###首先列举下可以用来缓存数据的东西:

- 1) 内存使用一个`数组/字典`，充当缓存容器

- 2) 客户端与web服务之间的session

- 3) 磁盘上Cookie文件

- 4) cpu或硬件，内部的高速缓存Cache

- 5) 基于`http 1.1`规定的 服务端与客户端的缓存

本人主要基于 第五种 缓存.

****

###那么所谓的`缓存`又分为两种:

- 客户端 缓存

- 服务端 缓存

既然iOS是搞客户端开发，所以主要着重于`客户端 缓存`的学习.

***

###客户端缓存 又称为 浏览器缓存

- 大多的浏览器，会 `自动` 缓存数据以及缓存管理

- 当打开一个网页显示出来后，浏览器会自动将这个网页上的所有数据（css样式、html代码、内部的数据）缓存到电脑上的磁盘文件

***

###`服务端` 可以告诉 `客户端` 是否需要进行缓存、缓存多长时间 等

- 服务端 通过设置 发送给客户端的 `响应头`，包含 是否需要让客户端进行缓存的 `参数设置`

- 客户端接收到响应时，`取出响应头中的指定参数的值`，就知道该不该缓存当前的请求响应数据

本文主要基于`HTTP 1.1 缓存标准`时，服务端 通过 `响应头` 告诉 客户端是否进行缓存.


###哪些请求缓存，哪些请求不缓存

- 服务端通过`响应头参数`告诉客户端是否进行缓存

- 浏览器一般只缓存`GET`方法的请求

- 不会对`POST`方法的请求进行缓存

- 不会对`需要认证`或者`安全加密`的请求进行缓存

- 如果响应中没有任何参数 来说明 缓存的过期，也不会进行缓存

***

###通过HTTP规定的一些`响应头参数`，告诉客户端如何缓存数据，那么主要的控制缓存的响应头参数:

- Expires

- Cache-Control Header

- Last-Modified/If-Modified-Since

- ETag/If-None-Match


主要为了告诉客户端此次请求使用`本地缓存数据` or 重新请求服务器获取最新数据

***

###列举一个简单的，服务端发送给客户端的 响应头

```objc
CacheControl = no-cache 
Pragma=no-cache 
Expires = -1 
```

大致就是类似这样的一个JSON字典类型的结构.

***

### 缓存控制头 Cache-Control 主要的指令:

|Cache-directive|说明|
|:----------:|---------|
|public|所有内容都将被缓存|
|private|内容只缓存到私有缓存中|
|no-cache|所有内容都不会被缓存|
|no-store|所有内容都不会被缓存到缓存或 Internet 临时文件中|
|must-revalidation/proxy-revalidation|如果缓存的内容失效，请求必须发送到服务器/代理以进行重新验证|
|max-age=xxx|缓存的内容将在 xxx 秒后失效, 这个选项只在HTTP 1.1可用, 并如果和Last-Modified一起使用时, 优先级较高|

***

### 如上命令 在不同的情形下，浏览器（客户端）是`将请求重新发送到服务器` or `使用缓存的内容`?

|Cache-directive|打开一个新的浏览器窗口|在原窗口中单击 Enter 按钮|刷新页面|单击 Back 按钮|
|:------:|-----|-----|-----|------|
|public|浏览器呈现来自缓存的页面|浏览器呈现来自缓存的页面|浏览器重新发送请求到服务器|浏览器呈现来自缓存的页面|
|private|浏览器重新发送请求到服务器|第一次，浏览器重新发送请求到服务器；此后，浏览器呈现来自缓存的页面|浏览器重新发送请求到服务器|浏览器呈现来自缓存的页面|
|no-cache/no-store|浏览器重新发送请求到服务器|浏览器重新发送请求到服务器|浏览器重新发送请求到服务器|浏览器重新发送请求到服务器|
|must-revalidation/proxy-revalidation|浏览器重新发送请求到服务器|第一次，浏览器重新发送请求到服务器；此后，浏览器呈现来自缓存的页面|浏览器重新发送请求到服务器|浏览器呈现来自缓存的页面|
|max-age=xxx|在 xxx 秒`后`，浏览器重新发送请求到服务器|在 xxx 秒后，浏览器重新发送请求到服务器|浏览器重新发送请求到服务器|在 xxx 秒后，浏览器重新发送请求到服务器|

****

###Cache-Control 指令小结

- 用来控制 浏览器（客户端）缓存 的最重要的设置

- 会`覆盖`其他设置，比如 Expires 和 Last-Modified

- 处理跨浏览器缓存问题的最有效的方法，因为不同的浏览器的行为基本相同

***

###过期头 (Expires)

- 在响应头中，提供一个日期和时间，用来告诉浏览器，响应在该日期和时间后被认为`失效`

- 缓存 失效 之后，一般都不再被浏览器缓存

- 注意: Cache-Control中的`max-age`指令 和 `s-maxage`指令 将`覆盖` Expires 头部参数

- Expires字段 接收的日期字符串格式：`Expires: Sun, 08 Nov 2009 03:37:26 GMT`

- 浏览器查看 Expires字段日期，与当前时间比较
	- 如果当前日期 < Expires字段日期，认为该内容`有效`并从缓存中提取出来
	- 如果当前日期 > Expires字段日期，认为该内容`失效`，浏览器将`采取一些措施`处理失效的缓存

***

###不同的浏览器，在不同的情形下，处理`缓存过期失效`时，是不一样的

![](http://i4.tietuku.com/7c649fc6993849fe.png)

![](http://i4.tietuku.com/f7a5f730a3be0b7d.png)

![](http://i4.tietuku.com/5e747bde4f81af19.png)

![](http://i4.tietuku.com/8b9664ba43109b11.png)

注意：所有浏览器都假定为使用`默认设置`运行.

***

###对上述做个小结

|操作|行为|
|----|----|
|打开新窗口|如果指定cache-control的值为`private`、`no-cache`、`must-revalidate`，那么打开`新窗口`访问时都会`重新访问服务器`。而如果指定了 `max-age`值,那么在此`值内`的时间里就`不会重新访问`服务器。例如：`Cache-control:max-age=5` 表示当访问此网页后的5秒内再次访问不会去服务器|
|在地址栏回车|如果值为private或must-revalidate,则只有第一次访问时会访问服务器,以后就不再访问。如果值为no-cache,那么每次都会访问。如果值为max-age，则在过期之前不会重复访问。|
|按后退按扭|如果值为private、must-revalidate、max-age，则不会重访问。而如果为`no-cache`,则每次都重复访问服务器。|
|按刷新按扭|无论为何值,都会重复访问服务器。|


****

###Last-Modified/If-Modified-Since 头参数，告诉客户端 `服务端的某个文件是否发生修改`，主要通过对比`文件的修改时间`


- 第一次浏览器打开某一个网页

- 网页所在的`服务器`，会将这个`网页文件 最后修改 的时间`，通过`Last-Modified`为参数名，存放在发送给浏览器的`响应头`中

- 浏览器 解析 `响应头`中的`Last-Modified`参数值，并保存下来

- 以后浏览器再次访问这个网页时，会在`请求头`中添加`If-Modified-Since`的参数，其参数值就是上面保存的 `服务器网页最后修改的时间`

- 服务器解析请求头中的`If-Modified-Since`参数值，服务器进行`两个`日期时间的比较
	- 日期1、当前被请求服务器网页，最后修改时间
	- 日期2、`If-Modified-Since`参数值日期时间

- 如果服务器发现两个日期时间是一样的，那就是说该网页文件到现在为止还没有做修改过，那么返回给浏览器`响应code=304`
	- code=304，服务端告诉客户端，访问的服务器文件 `没有做出修改`
	- 客户端就可以直接从本地缓存数据加载页面了
	- 就`不需要`再请求服务器获取数据了

- 如果服务器发现两个日期`不一样`
	- 那么就返回`最新数据`给客户端
	- 响应code=200（成功）、code=400+（错误）

	
- 小结:

	- Last-Modified参数，响应头
	- If-Modified-Since参数，请求头

***

###ETag/If-None-Match，作用类似上面，只是不通过对比文件的修改时间，而是通过对比文件的`特征码`

- 第一次浏览器打开某一个网页

- 网页所在的服务器，会对当前被访问的文件，`使用某种算法生成一个特征码，判断文件是否发生修改`，并使用`ETag`参数保存在`响应头`中，发送给客户端

- 客户端解析响应头中的`ETag`参数值，并保存下来

- 以后浏览器再次访问这个网页时，会在`请求头`中添加`If-None-Match`的参数，其参数值就是上面保存的 `该网页文件按照某种算法生成的特征码` ETag参数值.

- 然后服务器比较两个`特征码`
	- 特征码1、当前被请求服务器网页 的 特征码
	- 特征码2、`If-None-Match`参数值 特征码


- 如果对比一致，就只给客户端返回 响应code=304，告诉客户端该服务器网页文件没有做出修改，可以使用缓存数据

- 如果对比不一致，就返回`最新数据`给客户端以及最新的`ETag`响应头参数

- 小结:

	- ETag参数，响应头
	- If-None-Match参数，请求头

****

###对比如上两种用来告诉客户端是否可以使用缓存数据的方法区别:

- Last-Modified/If-Modified-Since，通过对比文件的`修改时间`

	- 时间是不能够精确到毫秒、纳秒...
	- 很可能在一秒内，发起很多次请求
	- 所以使用时间来判断，有点不是特别精准

- ETag/If-None-Match，通过对比文件的`特征码`

	- 一个文件一旦做出修改，那么其按照某种算法生成的`特征码`绝对是`不一样`的
	- 所以通过对比特征码，可以很精准的判断文件是否被修改

***

###到此为止扫清了基础的盲点，接着看具体的代码实现

***

###快速获取响应数据类型是否是图片

```objc
-(BOOL) isContentTypeImage {
    
    //1. 获取响应头中的Content-Type参数值
    NSString *contentType = [self.allHeaderFields objectForCaseInsensitiveKey:@"Content-Type"];
    
    //2. 判断是否是图片
    return ([contentType.lowercaseString rangeOfString:@"image"].location != NSNotFound);
}
```

###判断响应头中是否 包含 Cache-Control 缓存控制参数

```objc
-(BOOL) hasCacheDirective {
  
    //1. 获取响应头中的Cache-Control参数值
    NSString *cacheControl = [self.allHeaderFields objectForCaseInsensitiveKey:@"Cache-Control"];
    
    //2. 没有Cache-Control参数值
    if(!cacheControl)
        return NO;
    
    //3. Cache-Control参数值中，是否包含 no-cache
    //如果包含，表示每一次都必须重新请求服务器，不使用缓存
    if(([cacheControl.lowercaseString rangeOfString:@"no-cache"].location != NSNotFound))//不包含no-cache
        return NO;
    
    //4. Cache-Control参数值中，是否包含 max-age
    //如果包含，在这个时间段内使用缓存
    if(self.maxAge == 0)//为0，表示不包含max-age
        return NO;
    
    return YES;
}
```

稍微改了下作者的代码，感觉他写的有点不对劲.


***

###上面调用了获取 Cache-Control参数值中的 max-age参数值的方法

```objc
-(NSInteger) maxAge {
  
    __block NSInteger maxAge = 0;
    
    //获取响应头中的Cache-Control参数值
    NSString *cacheControl = [self.allHeaderFields objectForCaseInsensitiveKey:@"Cache-Control"];
    
    //使用 , 分开成多个部分，如: max-age, must-revalidate, no-cache
    NSArray *cacheControlEntities = [cacheControl componentsSeparatedByString:@","];
    
    //找到max-age参数
    [cacheControlEntities enumerateObjectsUsingBlock:^(NSString *substring, NSUInteger idx, BOOL *stop) {
    
        if([substring.lowercaseString rangeOfString:@"max-age"].location != NSNotFound) {
      
            //解析 max-age=5，后面的数字值5
            NSString *maxAge = nil;
            NSArray *array = [substring componentsSeparatedByString:@"="];
            if(array.count > 1) {
                maxAge = array[1];
                *stop = YES;
            }
        }
    }];
  
    return maxAge;
}
```

***

###判断是否包含 缓存控制的 相关响应头参数

```objc
-(BOOL) hasHTTPCacheHeaders {
  
    //1. 缓存控制参数
    NSString *cacheControl = [self.allHeaderFields objectForCaseInsensitiveKey:@"Cache-Control"];

    //2. 文件的特征码
    NSString *eTag = [self.allHeaderFields objectForCaseInsensitiveKey:@"ETag"];
    
    //3. 文件最后修改日期
    NSString *lastModified = [self.allHeaderFields objectForCaseInsensitiveKey:@"Last-Modified"];
  
    //4. 只要存在其中一种即可
    return (cacheControl || eTag || lastModified);
}
```

###判断是否需要 再次验证缓存是否 `可用`

- 主要查看响应头中，是否包含如下两个参数
	- Last-Modified 响应头参数
	- ETag 响应头参数 

- 注意: 只是验证缓存是否可用，并不是缓存的 `超时时间`

	
```objc
-(BOOL) hasRequiredRevalidationHeaders {
  
    NSString *lastModified = [self.allHeaderFields objectForCaseInsensitiveKey:@"Last-Modified"];
    
    NSString *eTag = [self.allHeaderFields objectForCaseInsensitiveKey:@"ETag"];

    return (eTag || lastModified);
}
```

###根据这些响应头参数: Cache-Control、Expires、max-age、no-cache，得到缓存过期的时间

####主要分为如下几个参数作为缓存失效时间:

- 1) 获取响应头中的`Expires`参数，作为缓存失效时间

- 2) 获取响应头中的`Cache-Control`参数
	- 使用`max-age`参数值作为缓存失效时间
	- 如果又包含`no-cahce`表示不使用缓存，缓存失效时间为当前系统时间


```objc
-(NSDate*) cacheExpiryDate
{
    //---------------------------------------------------------------------------------
    //第一种、获取缓存失效的时间 Expires参数
    
    //1.1 取出响应头中的 Expires参数
    NSString *expiresOn = [self.allHeaderFields objectForCaseInsensitiveKey:@"Expires"];
    
    //1.2 将 Expires参数日期值 转换成rfc1123日期格式
    __block NSDate *expiresOnDate = [NSDate dateFromRFC1123:expiresOn];
    
    //1.3 如果 Expires参数值存在，直接返回Expires参数值转换的日期
    if(expiresOnDate)
        return expiresOnDate;
  
    //---------------------------------------------------------------------------------
    //第二种、获取缓存失效的时间 Cache-Control指令
    
    //2.1 取出响应头中的 Cache-Control参数
    NSString *cacheControl = [self.allHeaderFields objectForCaseInsensitiveKey:@"Cache-Control"];
    
    //2.2 得到Cache-Control参数值中，所有命令（max-age、no-cache ... ）
    NSArray *cacheControlEntities = [cacheControl componentsSeparatedByString:@","];
    
    //2.3 对max-age、no-cache处理
    [cacheControlEntities enumerateObjectsUsingBlock:^(NSString *substring, NSUInteger idx, BOOL *stop) {
    
        //2.3.1 取出max-age参数值，作为缓存失效时间
        if([substring.lowercaseString rangeOfString:@"max-age"].location != NSNotFound) {
          
            // do some processing to calculate expiresOn
            NSString *maxAge = nil;
            NSArray *array = [substring componentsSeparatedByString:@"="];
            
            if(array.count > 1) {
                maxAge = array[1];
                
                //将max-age参数值，作为缓存失效时间
                expiresOnDate = [[NSDate date] dateByAddingTimeInterval:[maxAge intValue]];
            }
        }
        
        //2.3.2 如果包含no-cache参数，表示必须重新请求服务器，不使用缓存
        if([substring.lowercaseString rangeOfString:@"no-cache"].location != NSNotFound)
        {
            // 作者说不使用缓存，缓存失效时间是当前系统时间
            expiresOnDate = [NSDate date];
        }
    
        //作者说这里可以忽略对 must-revalidate指令的 处理
    }];
  
    return expiresOnDate;
}
```

###到这里就结束了，总结下基于`HTTP 1.1 Cache`.

- 响应头参数 与 请求头参数 的结合使用

- Cache-Control参数
	- max-age子参数
	- no-cache子参数
	- ...等等其他子参数

- 服务器告诉客户端，访问的服务器文件是否发生改变，是否能够使用缓存
	- Last-Modified/If-Modified-Since，对比文件的`修改时间`
	- ETag/If-None-Match，对比文件的`特征码`

***

###最后说下，本文记录的缓存访问，必须要服务器提供支持。如果服务器没有提供这些缓存参数在响应头中，那么可能造成过期时间间隔永远大于0，并且等于600秒。

所以，在服务器不支持缓存控制时，不要使用MKNetworkHost的缓存功能.