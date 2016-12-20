---
layout: post
title: "AFNetworking、AFURLRequestSerialization"
date: 2016-01-23 11:22:58 +0800
comments: true
categories: 
---

###AFURLRequestSerialization


> The `AFURLRequestSerialization` protocol is adopted by an object that encodes parameters for a specified HTTP requests. Request serializers may encode parameters as query strings, HTTP bodies, setting the appropriate HTTP header fields as necessary.

- AFURLRequestSerialization是一个protocol
- 给一个NSMutableURLRequest组装请求参数
- url?key1=value1&key2=value2
- 请求体 http body 参数 (`POST Method`)
- 请求头 http hreader 参数，默认添加的请求头参数
	- `Accept-Language`
	- `User-Agent`

> For example, a JSON request serializer may set the HTTP body of the request to a JSON representation, and set the `Content-Type` HTTP header field value to `application/json`.

- 举例 JSON Request 组装
- 设置 请求体 参数
- 然后设置 请求头 参数 `Content-Type` = `application/json`

***

###协议定义

```objc
NS_ASSUME_NONNULL_BEGIN

/**
 The `AFURLRequestSerialization` protocol is adopted by an object that encodes parameters for a specified HTTP requests. Request serializers may encode parameters as query strings, HTTP bodies, setting the appropriate HTTP header fields as necessary.

 For example, a JSON request serializer may set the HTTP body of the request to a JSON representation, and set the `Content-Type` HTTP header field value to `application/json`.
 */
@protocol AFURLRequestSerialization <NSObject, NSSecureCoding, NSCopying>

/**
 Returns a request with the specified parameters encoded into a copy of the original request.

 @param request The original request.
 @param parameters The parameters to be encoded.
 @param error The error that occurred while attempting to encode the request parameters.

 @return A serialized request.
 */
- (nullable NSURLRequest *)requestBySerializingRequest:(NSURLRequest *)request
                               withParameters:(nullable id)parameters
                                        error:(NSError * __nullable __autoreleasing *)error
#ifdef NS_SWIFT_NOTHROW
NS_SWIFT_NOTHROW
#endif
;
@end
```

###AFURLRequestSerialization协议实现

- AFHTTPRequestSerializer提供默认的基本实现，提供URL后组装key和value
	- 子类1、AFJSONRequestSerializer 提供组装JSON Request


###AFHTTPRequestSerializer实现AFURLRequestSerialization协议，提供一个组装Request的基本实现

> 可以自己子类化一个AFHTTPRequestSerializer，做一些附加的参数拼接、处理、验证...

```objc
@interface AFHTTPRequestSerializer : NSObject <AFURLRequestSerialization>

/**
 The string encoding used to serialize parameters. `NSUTF8StringEncoding` by default.
 
 1. 编码参数的格式
 2. NSUTF8StringEncoding为默认格式
 */
@property (nonatomic, assign) NSStringEncoding stringEncoding;

/**
 Whether created requests can use the device’s cellular radio (if present). `YES` by default.

 1. 是否可以使用蜂窝网
 2. 默认YES，可以使用

 @see NSMutableURLRequest -setAllowsCellularAccess:
 */
@property (nonatomic, assign) BOOL allowsCellularAccess;

/**
 The cache policy of created requests. `NSURLRequestUseProtocolCachePolicy` by default.

 1. 设置Request的缓存策略
 2. NSURLRequestUseProtocolCachePolicy 为默认策略

 @see NSMutableURLRequest -setCachePolicy:
 */
@property (nonatomic, assign) NSURLRequestCachePolicy cachePolicy;

/**
 Whether created requests should use the default cookie handling. `YES` by default.

 1. 是否处理Cookie
 2. 默认YES，处理

 @see NSMutableURLRequest -setHTTPShouldHandleCookies:
 */
@property (nonatomic, assign) BOOL HTTPShouldHandleCookies;

/**
 Whether created requests can continue transmitting data before receiving a response from an earlier transmission. `NO` by default

 1. 是否重用已经存在的网络连接
 2. 默认为NO
 3. POST和PUT会修改服务器上的实体，所以不建议流水线操作发送POST和PUT请求

 @see NSMutableURLRequest -setHTTPShouldUsePipelining:
 */
@property (nonatomic, assign) BOOL HTTPShouldUsePipelining;

/**
 The network service type for created requests. `NSURLNetworkServiceTypeDefault` by default.

 1. 网络服务质量
 2. 枚举定义

 typedef NS_ENUM(NSUInteger, NSURLRequestNetworkServiceType)
{
    NSURLNetworkServiceTypeDefault = 0,	// Standard internet traffic
    NSURLNetworkServiceTypeVoIP = 1,	// Voice over IP control traffic
    NSURLNetworkServiceTypeVideo = 2,	// Video traffic
    NSURLNetworkServiceTypeBackground = 3, // Background traffic
    NSURLNetworkServiceTypeVoice = 4	   // Voice data
};

 @see NSMutableURLRequest -setNetworkServiceType:
 */
@property (nonatomic, assign) NSURLRequestNetworkServiceType networkServiceType;

/**
 The timeout interval, in seconds, for created requests. The default timeout interval is 60 seconds.
 
 1. 设置请求的超时时间
 2. 默认是60秒

 @see NSMutableURLRequest -setTimeoutInterval:
 */
@property (nonatomic, assign) NSTimeInterval timeoutInterval;

///---------------------------------------
/// @name Configuring HTTP Request Headers
///---------------------------------------

/**
 Default HTTP header field values to be applied to serialized requests. By default, these include the following:

 - `Accept-Language` with the contents of `NSLocale +preferredLanguages`
 - `User-Agent` with the contents of various bundle identifiers and OS designations

 1. 设置NSURLRequest的请求头参数
 2. 框架默认添加 `Accept-Language` 和 `User-Agent` 这两个参数
 3. 不要直接设置这个字典，而是使用提供的方法 `setValue:forHTTPHeaderField:`

 @discussion To add or remove default request headers, use `setValue:forHTTPHeaderField:`.
 */
@property (readonly, nonatomic, strong) NSDictionary *HTTPRequestHeaders;

/**
 Creates and returns a serializer with default configuration.
 
 获取一个实例
 
 */
+ (instancetype)serializer;

/**
 Sets the value for the HTTP headers set in request objects made by the HTTP client. If `nil`, removes the existing value for that header.

 1. 用来设置请求参数
 2. 会设置给所有由这个request serializer组装的Request

 @param field The HTTP header to set a default value for
 @param value The value set as default for the specified header, or `nil`
 */
- (void)setValue:(nullable NSString *)value
forHTTPHeaderField:(NSString *)field;

/**
 Returns the value for the HTTP headers set in the request serializer.
 
 获取请求参数值

 @param field The HTTP header to retrieve the default value for

 @return The value set as default for the specified header, or `nil`
 */
- (nullable NSString *)valueForHTTPHeaderField:(NSString *)field;

/**
 Sets the "Authorization" HTTP header set in request objects made by the HTTP client to a basic authentication value with Base64-encoded username and password. This overwrites any existing value for this header.

 1. 该Request对应的服务器URL，需要提供账号密码认证
 2. 设置账号与密码

 @param username The HTTP basic auth username
 @param password The HTTP basic auth password
 */
- (void)setAuthorizationHeaderFieldWithUsername:(NSString *)username
                                       password:(NSString *)password;

/**
 @deprecated This method has been deprecated. Use -setValue:forHTTPHeaderField: instead.
 
 废弃
 
 */
- (void)setAuthorizationHeaderFieldWithToken:(NSString *)token DEPRECATED_ATTRIBUTE;


/**
 Clears any existing value for the "Authorization" HTTP header.
 
 清除认证的账号密码信息
 
 */
- (void)clearAuthorizationHeader;

///-------------------------------------------------------
/// @name Configuring Query String Parameter Serialization
///-------------------------------------------------------

/**
 HTTP methods for which serialized requests will encode parameters as a query string. `GET`, `HEAD`, and `DELETE` by default.
 
 1. 使用字符串key=value形式组装的在URL后面
 2. 包括: GET、HEAD、DELETE
 
 */
@property (nonatomic, strong) NSSet *HTTPMethodsEncodingParametersInURI;

/**
 Set the method of query string serialization according to one of the pre-defined styles.

 @param style The serialization style.

 @see AFHTTPRequestQueryStringSerializationStyle
 */
- (void)setQueryStringSerializationWithStyle:(AFHTTPRequestQueryStringSerializationStyle)style;

/**
 Set the a custom method of query string serialization according to the specified block.

 设置用户自己组装NRequest的请求参数回调Block

 @param block A block that defines a process of encoding parameters into a query string. This block returns the query string and takes three arguments: the request, the parameters to encode, and the error that occurred when attempting to encode parameters for the given request.
 */
- (void)setQueryStringSerializationWithBlock:(nullable NSString * (^)(NSURLRequest *request, id parameters, NSError * __autoreleasing *error))block;

///-------------------------------
/// @name Creating Request Objects
///-------------------------------

/**
 @deprecated This method has been deprecated. Use -requestWithMethod:URLString:parameters:error: instead.
 */
- (NSMutableURLRequest *)requestWithMethod:(NSString *)method
                                 URLString:(NSString *)URLString
                                parameters:(id)parameters DEPRECATED_ATTRIBUTE;

/**
 Creates an `NSMutableURLRequest` object with the specified HTTP method and URL string.

 If the HTTP method is `GET`, `HEAD`, or `DELETE`, the parameters will be used to construct a url-encoded query string that is appended to the request's URL. Otherwise, the parameters will be encoded according to the value of the `parameterEncoding` property, and set as the request body.

 @param method The HTTP method for the request, such as `GET`, `POST`, `PUT`, or `DELETE`. This parameter must not be `nil`.
 @param URLString The URL string used to create the request URL.
 @param parameters The parameters to be either set as a query string for `GET` requests, or the request HTTP body.
 @param error The error that occurred while constructing the request.

 @return An `NSMutableURLRequest` object.
 */
- (NSMutableURLRequest *)requestWithMethod:(NSString *)method
                                 URLString:(NSString *)URLString
                                parameters:(nullable id)parameters
                                     error:(NSError * __nullable __autoreleasing *)error;

/**
 @deprecated This method has been deprecated. Use -multipartFormRequestWithMethod:URLString:parameters:constructingBodyWithBlock:error: instead.
 */
- (NSMutableURLRequest *)multipartFormRequestWithMethod:(NSString *)method
                                              URLString:(NSString *)URLString
                                             parameters:(NSDictionary *)parameters
                              constructingBodyWithBlock:(void (^)(id <AFMultipartFormData> formData))block DEPRECATED_ATTRIBUTE;

/**
 Creates an `NSMutableURLRequest` object with the specified HTTP method and URLString, and constructs a `multipart/form-data` HTTP body, using the specified parameters and multipart form data block. See http://www.w3.org/TR/html4/interact/forms.html#h-17.13.4.2

 Multipart form requests are automatically streamed, reading files directly from disk along with in-memory data in a single HTTP body. The resulting `NSMutableURLRequest` object has an `HTTPBodyStream` property, so refrain from setting `HTTPBodyStream` or `HTTPBody` on this request object, as it will clear out the multipart form body stream.

 @param method The HTTP method for the request. This parameter must not be `GET` or `HEAD`, or `nil`.
 @param URLString The URL string used to create the request URL.
 @param parameters The parameters to be encoded and set in the request HTTP body.
 @param block A block that takes a single argument and appends data to the HTTP body. The block argument is an object adopting the `AFMultipartFormData` protocol.
 @param error The error that occurred while constructing the request.

 @return An `NSMutableURLRequest` object
 */
 
 //上传Request组装，提供文件的NSData数据
- (NSMutableURLRequest *)multipartFormRequestWithMethod:(NSString *)method
                                              URLString:(NSString *)URLString
                                             parameters:(nullable NSDictionary *)parameters
                              constructingBodyWithBlock:(nullable void (^)(id <AFMultipartFormData> formData))block
                                                  error:(NSError * __nullable __autoreleasing *)error;

/**
 Creates an `NSMutableURLRequest` by removing the `HTTPBodyStream` from a request, and asynchronously writing its contents into the specified file, invoking the completion handler when finished.

 @param request The multipart form request. The `HTTPBodyStream` property of `request` must not be `nil`.
 @param fileURL The file URL to write multipart form contents to.
 @param handler A handler block to execute.

 @discussion There is a bug in `NSURLSessionTask` that causes requests to not send a `Content-Length` header when streaming contents from an HTTP body, which is notably problematic when interacting with the Amazon S3 webservice. As a workaround, this method takes a request constructed with `multipartFormRequestWithMethod:URLString:parameters:constructingBodyWithBlock:error:`, or any other request with an `HTTPBodyStream`, writes the contents to the specified file and returns a copy of the original request with the `HTTPBodyStream` property set to `nil`. From here, the file can either be passed to `AFURLSessionManager -uploadTaskWithRequest:fromFile:progress:completionHandler:`, or have its contents read into an `NSData` that's assigned to the `HTTPBody` property of the request.

 @see https://github.com/AFNetworking/AFNetworking/issues/1398
 */
 //上传Request组装，提供文件的读取路径
- (NSMutableURLRequest *)requestWithMultipartFormRequest:(NSURLRequest *)request
                             writingStreamContentsToFile:(NSURL *)fileURL
                                       completionHandler:(nullable void (^)(NSError * __nullable error))handler;

@end
```

###子类一、AFJSONRequestSerializer

```objc
/**
 `AFJSONRequestSerializer` is a subclass of `AFHTTPRequestSerializer` that encodes parameters as JSON using `NSJSONSerialization`, setting the `Content-Type` of the encoded request to `application/json`.
 */
@interface AFJSONRequestSerializer : AFHTTPRequestSerializer

/**
 Options for writing the request JSON data from Foundation objects. For possible values, see the `NSJSONSerialization` documentation section "NSJSONWritingOptions". `0` by default.
 */
@property (nonatomic, assign) NSJSONWritingOptions writingOptions;

/**
 Creates and returns a JSON serializer with specified reading and writing options.

 @param writingOptions The specified JSON writing options.
 */
+ (instancetype)serializerWithWritingOptions:(NSJSONWritingOptions)writingOptions;

@end
```

###子类二、AFPropertyListRequestSerializer

```objc
/**
 `AFPropertyListRequestSerializer` is a subclass of `AFHTTPRequestSerializer` that encodes parameters as JSON using `NSPropertyListSerializer`, setting the `Content-Type` of the encoded request to `application/x-plist`.
 */
@interface AFPropertyListRequestSerializer : AFHTTPRequestSerializer

/**
 The property list format. Possible values are described in "NSPropertyListFormat".
 */
@property (nonatomic, assign) NSPropertyListFormat format;

/**
 @warning The `writeOptions` property is currently unused.
 */
@property (nonatomic, assign) NSPropertyListWriteOptions writeOptions;

/**
 Creates and returns a property list serializer with a specified format, read options, and write options.

 @param format The property list format.
 @param writeOptions The property list write options.

 @warning The `writeOptions` property is currently unused.
 */
+ (instancetype)serializerWithFormat:(NSPropertyListFormat)format
                        writeOptions:(NSPropertyListWriteOptions)writeOptions;

@end
```

***

###录下一个对字符串做base64压缩的c函数

```objc
static NSString * AFBase64EncodedStringFromString(NSString *string) {
    
    NSData *data = [NSData dataWithBytes:[string UTF8String]
                                  length:[string lengthOfBytesUsingEncoding:NSUTF8StringEncoding]];
    
    NSUInteger length = [data length];
    
    NSMutableData *mutableData = [NSMutableData dataWithLength:((length + 2) / 3) * 4];
    
    uint8_t *input = (uint8_t *)[data bytes];
    
    uint8_t *output = (uint8_t *)[mutableData mutableBytes];
    
    for (NSUInteger i = 0; i < length; i += 3) {
        
        NSUInteger value = 0;
        
        for (NSUInteger j = i; j < (i + 3); j++) {
            value <<= 8;
            if (j < length) {
                value |= (0xFF & input[j]);
            }
        }
        
        static uint8_t const kAFBase64EncodingTable[] = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";
        
        NSUInteger idx = (i / 3) * 4;
        output[idx + 0] = kAFBase64EncodingTable[(value >> 18) & 0x3F];
        output[idx + 1] = kAFBase64EncodingTable[(value >> 12) & 0x3F];
        output[idx + 2] = (i + 1) < length ? kAFBase64EncodingTable[(value >> 6)  & 0x3F] : '=';
        output[idx + 3] = (i + 2) < length ? kAFBase64EncodingTable[(value >> 0)  & 0x3F] : '=';
    }
    
    return [[NSString alloc] initWithData:mutableData encoding:NSASCIIStringEncoding];
}
```

以后用得着的...

***

###再一个使用NSCharSet替换掉字符串中的特殊字符

```objc
static NSString * AFPercentEscapedStringFromString(NSString *string) {
    
    //对URL中包含的特殊字符（:#[]@）替换成 %
    // does not include "?" or "/" due to RFC 3986 - Section 3.4
    static NSString * const kAFCharactersGeneralDelimitersToEncode = @":#[]@";
    
    static NSString * const kAFCharactersSubDelimitersToEncode = @"!$&'()*+,;=";
    
    //创建一个用于URL的字符集
    NSMutableCharacterSet * allowedCharacterSet = [[NSCharacterSet URLQueryAllowedCharacterSet] mutableCopy];
    
    //:#[]@ + !$&'()*+,;=
    NSString *temp = [kAFCharactersGeneralDelimitersToEncode stringByAppendingString:kAFCharactersSubDelimitersToEncode];
    
    //从字符集中移除上面的所有 特殊字符
    [allowedCharacterSet removeCharactersInString:temp];
    
    // FIXME: https://github.com/AFNetworking/AFNetworking/pull/3028
    // return [string stringByAddingPercentEncodingWithAllowedCharacters:allowedCharacterSet];
    
    //下面是从传入的string中
    static NSUInteger const batchSize = 50;
    
    NSUInteger index = 0;
    NSMutableString *escaped = @"".mutableCopy;
    
    while (index < string.length) {
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wgnu"
        NSUInteger length = MIN(string.length - index, batchSize);
#pragma GCC diagnostic pop
        NSRange range = NSMakeRange(index, length);
        
        // To avoid breaking up character sequences such as 👴🏻👮🏽
        range = [string rangeOfComposedCharacterSequencesForRange:range];
        
        NSString *substring = [string substringWithRange:range];
        
        //从substring中 替换掉 不存在于allowedCharacterSet字符集中的 其他字符
        NSString *encoded = [substring stringByAddingPercentEncodingWithAllowedCharacters:allowedCharacterSet];
        
        [escaped appendString:encoded];
        
        index += range.length;
    }
    
    return escaped;
}
```

***

- 然后使用默认的`请求参数组装器是二进制.

```
注意: 这里使用的默认HttpRequestSerializer【以key对value的键值对完成拼接请求参数字符串】
//www.baidu.com?loginName=xiongzenghui 这种.
self.requestSerializer = [AFHTTPRequestSerializer serializer];

//这种会使用JSON格式组装请求参数.
self.requestSerializer = [AFJSONRequestSerializer serializer];

//可以自己设置其他类型的请求组装器
self.requestSerializer = [AFPropertyListRequestSerializer serializer];
```

- 使用默`JSON格式解析器`作为默认response解析格式

```
self.responseSerializer = [AFJSONResponseSerializer serializer];

//可以自己设置其他类型的响应解析器
self.responseSerializer = [AFPropertyListResponseSerializer serializer];
self.responseSerializer = [AFXMLParserResponseSerializer serializer];
self.responseSerializer = [AFXMLDocumentResponseSerializer serializer];
self.responseSerializer = [AFImageResponseSerializer serializer];
self.responseSerializer = [AFCompoundResponseSerializer serializer];
```


#### 后面的JSON和Plist的请求组装器，都是AFHTTPRequestSerializer的子类.

****

- 先来看下`AFHTTPRequestSerializer`如何组装将传入的 `请求URL` ，`请求path`，`请求方法`，`请求参数`组装成`NSURLRequest实例` ？


- 分两种组装类型:
1. GET，DELETE，HEAD		直接添加到`NSURLRequest.URL`
2. POST					设置到`NSURLRequest.httpBody`

OK，往下找到AFURLRequestSerialization.h中的如下方法.

```
- (NSMutableURLRequest *)requestWithMethod:(NSString *)method
                                 URLString:(NSString *)URLString
                                parameters:(id)parameters
                                     error:(NSError *__autoreleasing *)error
{
	//...
}
```

- 我摘录一下上面方法的重要的部分

```
//1. 创建一个Request（传入URL、请求方法）
NSMutableURLRequest *mutableRequest = [[NSMutableURLRequest alloc] initWithURL:url];
    mutableRequest.HTTPMethod = method;

//2. RequestSerializer单例对象，使用KVO机制观测属性值改变，此时数组保存就是值被修改的属性
//（这个KVO属性的小逻辑后续说）
for (NSString *keyPath in AFHTTPRequestSerializerObservedKeyPaths()) {
    if ([self.mutableObservedChangedKeyPaths containsObject:keyPath]) {
        [mutableRequest setValue:[self valueForKeyPath:keyPath] forKey:keyPath];
    }
}

//3. 继续调用自身的其他的一个方法，完成添加【拼接完成的请求参数】.
mutableRequest = [[self requestBySerializingRequest:mutableRequest withParameters:parameters error:error] mutableCopy];
```

- 再走到添加请求参数的方法内部看看. requestBySerializingRequest:withParameters:error:

```
- (NSURLRequest *)requestBySerializingRequest:(NSURLRequest *)request
                               withParameters:(id)parameters
                                        error:(NSError *__autoreleasing *)error
{
    NSParameterAssert(request);
	
	//1. 深拷贝之前创建的request
    NSMutableURLRequest *mutableRequest = [request mutableCopy];

	//2. 给request设置请求头参数
	//【重要】所以说给当前请求设置请求头参数，只需要设置给requestSerializer的HTTPRequestHeaders
	//但是这个属性是只读，不能直接修改这个属性的值.
	//通过一个方法【 - (void)setValue:(nullable NSString *)value
forHTTPHeaderField:(NSString *)field】 来设置.

    [self.HTTPRequestHeaders enumerateKeysAndObjectsUsingBlock:^(id field, id value, BOOL * __unused stop) {
        if (![request valueForHTTPHeaderField:field]) {
            [mutableRequest setValue:value forHTTPHeaderField:field];
        }
    }];

	//3. 开始组装请求参数
    if (parameters) {
        NSString *query = nil;
        
        if (self.queryStringSerialization) {
        //3.1 框架使用者自己组装请求参数.
        
            NSError *serializationError;
            query = self.queryStringSerialization(request, parameters, &serializationError);

            if (serializationError) {
                if (error) {
                    *error = serializationError;
                }

                return nil;
            }
        } else {
        //3.2 框架来组装请求参数.
            switch (self.queryStringSerializationStyle) {
                case AFHTTPRequestQueryStringDefaultStyle:
                
                	//调用了一个C方法，进行请求参数的拼接.（如: page=1&size=5 这个样子）
                    query = AFQueryStringFromParameters(parameters);
                    break;
            }
        }
	
		//4. 是否包含当前类型的网络请求(GET,HEAD,DELETE,POST)
        if ([self.HTTPMethodsEncodingParametersInURI containsObject:[[request HTTPMethod] uppercaseString]]) {
        
        	//4.1 GET,HEAD,DELETE这三种类型请求的参数拼接.
            mutableRequest.URL = [NSURL URLWithString:[[mutableRequest.URL absoluteString] stringByAppendingFormat:mutableRequest.URL.query ? @"&%@" : @"?%@", query]];
        } else {
        
        	//4.2 POST请求类型的参数组装.
            if (![mutableRequest valueForHTTPHeaderField:@"Content-Type"]) {
                [mutableRequest setValue:@"application/x-www-form-urlencoded" forHTTPHeaderField:@"Content-Type"];
            }
            [mutableRequest setHTTPBody:[query dataUsingEncoding:self.stringEncoding]];
        }
    }

    return mutableRequest;
}

```

- 如上有一处请求参数的拼接逻辑.

```
mutableRequest.URL = [NSURL URLWithString:[[mutableRequest.URL absoluteString] stringByAppendingFormat:mutableRequest.URL.query ? @"&%@" : @"?%@", query]];

```

为了防止Request.URL.query存在参数【可能会被覆盖】的情况.


小结: 【组装GET/HEAD/DELETE三种类型的NSURLRequest.URL请求参数】

1. 【NSMutableURLRequest对象.URL】 是最终用于网络请求操作的URL【全路径】.
2. 【NSURL对象.query】 用于获取到当前URL后面带着的【请求参数】.

如上代码是GET、DELETE、HEAD三种类型请求时，请求参数拼接的情况.

- 再来看一下`POST请求`如何处理

```
//1. 添加默认的Content-Type属性
if (![mutableRequest valueForHTTPHeaderField:@"Content-Type"]) {
	[mutableRequest setValue:@"application/x-www-form-urlencoded" forHTTPHeaderField:@"Content-Type"];
}

//2. 将拼接完成的请求参数【1.使用AFHTTPRequestSerializer 2.AFJSONRequestSerializer】设置到【请求体】
//（[NSMutableURLRequest setHTTPBody:(NSData*)]）
[mutableRequest setHTTPBody:[query dataUsingEncoding:self.stringEncoding]];

```

小结: `组装POST类型的NSURLRequest.httpBody请求参数`

* 如上setHTTPBody:方法的参数是一个NSData类型的二进制数据.
* 将拼接完成的参数（如: name=xiongzenghui&mobile=142374234）转化成NSData，再放到NSMutableURLRequest对象的【HTTPBody属性】，保存到请求体中。


####总之，一个NSMutableURLRequest创建完毕，也就是说确定了
* 哪一个服务器
* 哪一个路径
* 什么方法
* 哪些请求参数

****

- 然后到 `-[AFHTTPSessionManager GET:parameters:success:failure:]`，如下`POST、GET`等方法主要完成`启动task`网络操作

```
- (NSURLSessionDataTask *)GET:(NSString *)URLString
                   parameters:(id)parameters
                      success:(void (^)(NSURLSessionDataTask *task, id responseObject))success
                      failure:(void (^)(NSURLSessionDataTask *task, NSError *error))failure
{

	//1. 传入NSURLRequest实例，创建一个NSURLSessionDataTask对象，包装http请求操作.
    NSURLSessionDataTask *dataTask = [self dataTaskWithHTTPMethod:@"GET" URLString:URLString parameters:parameters success:success failure:failure];

	//2. 启动task
    [dataTask resume];

    return dataTask;
}
```

- `具体完成创建NSURLSessionDataTask`的代码在`-[AFHttpSessionManager dataTaskWithHTTPMethod:URLString:parameters:success:failure:] `
	- 1) NSMutableURLRequest组装
	- 2) 将外界传入的`success block`和`fail block`回调执行
	- 3) 调用`继承自AFURLSessionManager`的实例方法`dataTaskWithRequest:completionHandler:`

```
- (NSURLSessionDataTask *)dataTaskWithHTTPMethod:(NSString *)method
                                       URLString:(NSString *)URLString
                                      parameters:(id)parameters
                                         success:(void (^)(NSURLSessionDataTask *, id))success
                                         failure:(void (^)(NSURLSessionDataTask *, NSError *))failure
{
	//保存组装参数创建request时的报错信息
    NSError *serializationError = nil;
    
    //1. 【重点】组装完成用于最终网络请求的NSURLRequest（包含: GET/POST、请求路径、BaseURL、请求参数）
    NSMutableURLRequest *request = [self.requestSerializer requestWithMethod:method URLString:[[NSURL URLWithString:URLString relativeToURL:self.baseURL] absoluteString] parameters:parameters error:&serializationError];
    
    //2. 是否组装NSURLRequest.URL参数失败
    if (serializationError) {
        if (failure) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wgnu"

			//如果出错，执行回调Block，请求结束.
            dispatch_async(self.completionQueue ?: dispatch_get_main_queue(), ^{
                failure(nil, serializationError);
            });
#pragma clang diagnostic pop
        }

        return nil;
    }

	
	//3. 如下又调用  -[AFURLSessionManager dataTaskWithRequest:completionHandler:]  完成task的最终创建.
	
	//	 并且对框架使用者传入的success和fail两种请求结束Block处理执行.
	
    __block NSURLSessionDataTask *dataTask = nil;
    
    dataTask = [self dataTaskWithRequest:request completionHandler:^(NSURLResponse * __unused response, id responseObject, NSError *error) {
    
        if (error) {
        	
        	//请求错误结束
        	
            if (failure) {
                failure(dataTask, error);
            }
        } else {
        
        	//请求成功结束
        
            if (success) {
                success(dataTask, responseObject);
            }
        }
    }];

    return dataTask;
}
```

如上第3步，处理了框架调用者传入的`success`和`fail`两个回调Block。

***

###AFURLSessionManager 继承结构

* AFHttpSessionManager 继承自 `AFURLSessionManager`
* 类似 AFHttpRequestOperation继承自AFURLConenctionOperation一样.

* AFURLSessionManager提供所有具体完成请求网络的相关代码.
* AFHttpSessionManager只是调用AFURLSessionManager封装的代码完成操作，提供简单快捷Api入口。

***

- AFURLSessionManager实现了如下协议
 - 网络请求相关
	- NSURLSessionDelegate
	 - NSURLSessionTaskDelegate
	 - NSURLSessionDataDelegate
	 - NSURLSessionDownloadDelegate
 - 归档磁盘文件
 	- NSSecureCoding
 	- NSCopying

```
@interface AFURLSessionManager : NSObject <NSURLSessionDelegate, NSURLSessionTaskDelegate, NSURLSessionDataDelegate, NSURLSessionDownloadDelegate, NSSecureCoding, NSCopying>

//...省略

@end
```

主需实现了如下几个接口方法

- NSURLSessionDelegate
	- 主要处理`NSURLSession级别`的回调，比如: `鉴权`、`后台` ... 

- NSURLSessionTaskDelegate（继承自NSURLSessionDelegate）
	- 主要处理`Task级别`的【通用】的回调，比如: 相应`response`、`重定向`...

- NSURLSessionDataDelegate （继承自NSURLSessionTaskDelegate）
	- 主要处理`Task级别`的`获取数据`和`上传数据`的回调.

- NSURLSessionDownloadDelegate （继承自NSURLSessionTaskDelegate）
	- 主要处理`Task级别`的`下载数据`的回调.


- 再看看`AFURLSessionManager`实例的一些基础属性.

```
会话
@property (readonly, nonatomic, strong) NSURLSession *session;
```

```
任务队列
@property (readonly, nonatomic, strong) NSOperationQueue *operationQueue;
```

```
//...
@property (nonatomic, strong) id <AFURLResponseSerialization> responseSerializer;
```

```
//安全策略
@property (nonatomic, strong) AFSecurityPolicy *securityPolicy;
```

```
//网络状态监测
@property (readwrite, nonatomic, strong) AFNetworkReachabilityManager *reachabilityManager;
```

```
保存各种task的数组

/**
 The data, upload, and download tasks currently run by the managed session.
 */
@property (readonly, nonatomic, strong) NSArray *tasks;

/**
 The data tasks currently run by the managed session.
 */
@property (readonly, nonatomic, strong) NSArray *dataTasks;

/**
 The upload tasks currently run by the managed session.
 */
@property (readonly, nonatomic, strong) NSArray *uploadTasks;

/**
 The download tasks currently run by the managed session.
 */
@property (readonly, nonatomic, strong) NSArray *downloadTasks;
```

```
// 队列
#if OS_OBJECT_HAVE_OBJC_SUPPORT
@property (nonatomic, strong, nullable) dispatch_queue_t completionQueue;
#else
@property (nonatomic, assign, nullable) dispatch_queue_t completionQueue;
#endif

//队列组
#if OS_OBJECT_HAVE_OBJC_SUPPORT
@property (nonatomic, strong, nullable) dispatch_group_t completionGroup;
#else
@property (nonatomic, assign, nullable) dispatch_group_t completionGroup;
#endif

```

***

- -[AFURLSessionManager initWithSessionConfiguration:] 初始化做的一些事情，类似`AFURLRequestOperationManager`

```
- (instancetype)initWithSessionConfiguration:(NSURLSessionConfiguration *)configuration {
   
    self = [super init];
    if (!self) {
        return nil;
    }
	
	//1. 设置默认的 NSURLSessionConfiguration.
    if (!configuration) {
        configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    }
    self.sessionConfiguration = configuration;
	
	//2. operationQueue初始化
    self.operationQueue = [[NSOperationQueue alloc] init];
    self.operationQueue.maxConcurrentOperationCount = 1;//最大并发数

	//3. 创建一个会话【NSURLSession后面再说】
    self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration delegate:self delegateQueue:self.operationQueue];
	
	//4. 请求参数组装器
    self.responseSerializer = [AFJSONResponseSerializer serializer];

	//5. 安全策略
    self.securityPolicy = [AFSecurityPolicy defaultPolicy];

	//6. 网络状态监测
#if !TARGET_OS_WATCH
    self.reachabilityManager = [AFNetworkReachabilityManager sharedManager];
#endif

	//7. 保存所有在后台执行task的id.
    self.mutableTaskDelegatesKeyedByTaskIdentifier = [[NSMutableDictionary alloc] init];

	//8. 初见线程锁
    self.lock = [[NSLock alloc] init];
    self.lock.name = AFURLSessionManagerLockName;

	//9. 看这个session中，是否还有未执行完毕的task。如果有未完成的，那么给他分配一个TaskDelegate实例。
    [self.session getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) {
    
    	//9.1 处理普通Task
        for (NSURLSessionDataTask *task in dataTasks) {
            [self addDelegateForDataTask:task completionHandler:nil];
        }

		//9.2 处理上传Task
        for (NSURLSessionUploadTask *uploadTask in uploadTasks) {
            [self addDelegateForUploadTask:uploadTask progress:nil completionHandler:nil];
        }

		//9.3 处理下载Task
        for (NSURLSessionDownloadTask *downloadTask in downloadTasks) {
            [self addDelegateForDownloadTask:downloadTask progress:nil destination:nil completionHandler:nil];
        }
    }];

	//10. 注册事件 1）task恢复执行 2)task暂停执行
	 [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(taskDidResume:) name:AFNSURLSessionTaskDidResumeNotification object:nil];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(taskDidSuspend:) name:AFNSURLSessionTaskDidSuspendNotification object:nil];

    return self;
}
```

- 从第9步，看出NSURLSestionTask有三个子类:

* NSURLSession`Data`Task
	*  主要是数据任务，它会在`较短`的时间内给予反馈（数据片段传输）
	*  Datatask不能用于Background sessions
	
* NSURLSession`Upload`Task
	* 上传的网络操作

* NSURLSession`Download`Task
	* 下载的网络操作


- 看一下这个方法 -[AFURLSessionManager addDelegateForDataTask:completionHandler:]。 当每次创建一个SessionTask时，就创建一个完成其回调函数的AFURLSessionManagerTaskDelegate回调函数的实例.

```
- (void)addDelegateForDataTask:(NSURLSessionDataTask *)dataTask
             completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
	//1. 创建一个AFURLSessionManagerTaskDelegate对象（和AFHttpURLConectionOperation不同的地方）
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] init];
    
    //2. 设置回调方法的对象
    delegate.manager = self;
    
    //3. 回调Block
    delegate.completionHandler = completionHandler;

	//4. 将当前传入的dataTask的地址作为其描述
    dataTask.taskDescription = self.taskDescriptionForSessionTasks;
    
    //5. 
    [self setDelegate:delegate forTask:dataTask];
}
```

- 类似还有其他两个task类型

```
[self addDelegateForUploadTask:uploadTask progress:nil completionHandler:nil];
```

```
[self addDelegateForDownloadTask:downloadTask progress:nil destination:nil completionHandler:nil];
```

- 多线程下并发访问，互斥写task保存的字典，可能有多个请求并发执行.

```
- (void)setDelegate:(AFURLSessionManagerTaskDelegate *)delegate
            forTask:(NSURLSessionTask *)task
{
    NSParameterAssert(task);
    NSParameterAssert(delegate);

    [self.lock lock];
    
    //以task的地址值为key，TaskDelegate实例为value.
    self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)] = delegate;
    
    [self.lock unlock];
}
```

- 返回的是当前AFURLSessionManager实例的地址

```
- (NSString *)taskDescriptionForSessionTasks {
    return [NSString stringWithFormat:@"%p", self];
}
```

***

###接下来看看AFURLSessionManagerTaskDelegate干什么的？

- 先看下定义

```
//注意这个类实现的协议.
@interface AFURLSessionManagerTaskDelegate : NSObject <NSURLSessionTaskDelegate, NSURLSessionDataDelegate, NSURLSessionDownloadDelegate>

//弱引用持有AFURLSessionManager单例对象
@property (nonatomic, weak) AFURLSessionManager *manager;

//保存接收到的服务器二进制数据
@property (nonatomic, strong) NSMutableData *mutableData;

//进度
@property (nonatomic, strong) NSProgress *progress;

//下载文件的网络地址
@property (nonatomic, copy) NSURL *downloadFileURL;

//下载完成后的回调代码块
@property (nonatomic, copy) AFURLSessionDownloadTaskDidFinishDownloadingBlock downloadTaskDidFinishDownloading;

//网络请求操作结束后的回调代码块
@property (nonatomic, copy) AFURLSessionTaskCompletionHandler completionHandler;

@end
```

- 执行下载操作时的回调Block定义:

```
//参数1: 会话
//参数2: 任务
//参数3: 写入本地哪个文件

typedef NSURL * (^AFURLSessionDownloadTaskDidFinishDownloadingBlock)(NSURLSession *session, NSURLSessionDownloadTask *downloadTask, NSURL *location);

```

- 网络请求操作结束后回调Block定义:

```
//参数1: 响应
//参数2: 响应body（还可以取出header）
//参数3: 写入本地哪个文件
typedef void (^AFURLSessionTaskCompletionHandler)(NSURLResponse *response, id responseObject, NSError *error);
```

- 如上看出，`AFURLSessionManagerTaskDelegate`实现了前面提到的三个抽象接口，用于封装执行网络请求Api后，系统回调方法的通用代码。
- 单独抽一个类封装协议实现的代码


> 这个地方我觉得比之前AFURLConnectionOperation设计的比较好的地方。
使用了一个单独的类去封装这些协议的接口方法，就不用把这些方法写在operation的代码中。
这样就是说每个Task执行后的系统回调方法，都在对应的AFURLSessionManagerTaskDelegate对象里面。

- 再往下看看AFURLSessionManagerTaskDelegate实现的协议方法:

* NSURLSessionTaskDelegate协议定义的抽象方法

```
- (void)URLSession:(__unused NSURLSession *)session
              task:(__unused NSURLSessionTask *)task
   didSendBodyData:(__unused int64_t)bytesSent
    totalBytesSent:(int64_t)totalBytesSent
totalBytesExpectedToSend:(int64_t)totalBytesExpectedToSend
{
	//.....此处省略代码，后续分析，下面如上.
}
```

```
- (void)URLSession:(__unused NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error
{
	//.....
}
```

* NSURLSessionDataTaskDelegate协议定义的抽象方法

```
- (void)URLSession:(__unused NSURLSession *)session
          dataTask:(__unused NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data
{
	//.....
}
```

* NSURLSessionDownloadTaskDelegate协议定义的抽象方法

```
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
didFinishDownloadingToURL:(NSURL *)location
{
	//....
}
```

```
- (void)URLSession:(__unused NSURLSession *)session
      downloadTask:(__unused NSURLSessionDownloadTask *)downloadTask
      didWriteData:(__unused int64_t)bytesWritten
 totalBytesWritten:(int64_t)totalBytesWritten
totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite
{
	//....
}
```

```
- (void)URLSession:(__unused NSURLSession *)session
      downloadTask:(__unused NSURLSessionDownloadTask *)downloadTask
 didResumeAtOffset:(int64_t)fileOffset
expectedTotalBytes:(int64_t)expectedTotalBytes
{
	//.....
}
```

OK，先说一下大致的作用就可以，后续再来看看具体怎么做的.

***

- 绕了一圈，继续回到  看看创建`NSURLSessionDataTask`实例.
	- 1) 代码出现的地方`-[AFURLSessionManager dataTaskWithRequest:completionHandler:]`
	- 2) 主要使用`NSURLSession`实例来创建一个Task实例
		- 创建出来的Task实例的一些配置，来自`NSURLSession`实例
	
```
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                            completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
	//1. 
    __block NSURLSessionDataTask *dataTask = nil;
    
    //2. 使用GCD同步执行代码块 
    dispatch_sync(url_session_manager_creation_queue(), ^{
        dataTask = [self.session dataTaskWithRequest:request];
    });

	//3. 这里给当前创建的task实例，分配一个队对应处理所有NSURLSessionXxxDelegate函数的TaskDelegate实例。
    [self addDelegateForDataTask:dataTask completionHandler:completionHandler];

    return dataTask;
}

```

***

- 再看一下给当前`SessionTask`对象，同时创建一个TaskDelegate对象完成后续回调操作。
	- 代码所在地: `-[AFURLSessionManager addDelegateForDataTask:completionHandler:]`

```
- (void)addDelegateForDataTask:(NSURLSessionDataTask *)dataTask
             completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
	//1. 创建一个新的TaskDelegate实例.
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] init];
    
    //2. TaskDelegate实例指向当前AFURLSessionManager单例.
    delegate.manager = self;
    
    //3. 保存传入的回调Block.
    delegate.completionHandler = completionHandler;

	//4. 给TaskDelegate实例指定一个唯一标示Id.
    dataTask.taskDescription = self.taskDescriptionForSessionTasks;
    
    //5. 让`TaskDelegate实例` 与 `SessionTask实例` 建立关系，方便后续可以通过Task找到对应的TaskDelegate.
    [self setDelegate:delegate forTask:dataTask];
}
```

***

- 再看一下AFURLSessionManager实例化时，创建`NSURLSesion`实例。

```
self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration 			
											 delegate:self 
										delegateQueue:self.operationQueue];
```

- 看一下NSURLSession优秀与NSURLConnecetion的地方:
	- `后台`上传下载、网络操作的`暂停`、`恢复`、`断点续传`等。
	- App程序可以存在`多个Session实例`，可以给每个Session实例做不同的配置(`http header，Cache，Cookie，protocal，Credential`)。
	- 不再在整个App层面共享配置，还可以子类化并支持私有配置的NSURLSession实例。

- 对鉴权的回调做了改进
	- 此前NSURLConnection的`鉴权回调无法和请求进行匹配`，该回调可能来自任意的请求
	- 现在`每个请求`都可以在指定的代理方法中对进行匹配处理


- NSURLSession是用于替代NSURLConnection的一套Api体系
	- NSURLSession包含的一套Api关系图.

![](http://i3.tietuku.com/f5818dee50366632.png)

- 注意: 如果用户强制`将程序关闭`，`NSURLSession会断掉`。而其他情况都可以执行`后台下载上传`.

- NSURLSession全局实例，是可以让当前手机iOS系统内部`所有App程序`使用.

![](http://i1.tietuku.com/630d009dffeee2f7.png)

- 使用NSURLConnection与NSURLSession进行网络请求时的结构不同之处.

![](http://i1.tietuku.com/411c8927818fd677.png)


- 再具体看看NSURLSession.h 有一些什么？


```
#import <Foundation/NSObject.h>
#import <Foundation/NSURLRequest.h>
#import <Foundation/NSHTTPCookieStorage.h>

#include <Security/SecureTransport.h>
```

```
依赖的基本工具类

@class NSString;
@class NSURL;
@class NSError;
@class NSArray;
@class NSDictionary;
@class NSInputStream;
@class NSData;
@class NSOperationQueue;
```

```
依赖的NSURLConnection体系的相关API

//1. 完成请求/响应两个时刻的拦截
@class NSURLCache;

//2. 网络请求后得到的响应
@class NSURLResponse;

//3. 由NSURLResponse转换后的响应
@class NSHTTPURLResponse;

//4. Cookie对象（iOS自动给我们的请求添加本地Cookie，网络缓存）
@class NSHTTPCookie;

//5. 被缓存过的response
@class NSCachedURLResponse;

//6. 证书相关的
@class NSURLAuthenticationChallenge;

//7. 证书相关
@class NSURLProtectionSpace;

//8. 证书相关
@class NSURLCredential;

//9. 证书相关
@class NSURLCredentialStorage;
```

-  直接获取全局共享的单例。

```
+ (NSURLSession *)sharedSession;
```

- 传入配置对象，并使用默认的delegateQueue，创建自己的session

```
+ (NSURLSession *)sessionWithConfiguration:(NSURLSessionConfiguration *)configuration;
```

```
+ (NSURLSession *)sessionWithConfiguration:(NSURLSessionConfiguration *)configuration delegate:(id <NSURLSessionDelegate>)delegate delegateQueue:(NSOperationQueue *)queue;
```

- 看下有哪些暴露的属性@property.

```
@property (readonly, retain) NSOperationQueue *delegateQueue;

@property (readonly, retain) id <NSURLSessionDelegate> delegate;

@property (readonly, copy) NSURLSessionConfiguration *configuration;

@property (copy) NSString *sessionDescription;
```

- 操作所有Tasks

```
//释放掉当前Session实例，但是对全局Session单例不起作用.
- (void)finishTasksAndInvalidate;

//关闭使用Session实例.
- (void)invalidateAndCancel;

- (void)resetWithCompletionHandler:(void (^)(void))completionHandler; 

- (void)flushWithCompletionHandler:(void (^)(void))completionHandler;  

- (void)getTasksWithCompletionHandler:(void (^)(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks))completionHandler;
```

- 创建不同类型的Task实例.

```
/* Creates a data task with the given request.  The request may have a body stream. */
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request;

/* Creates a data task to retrieve the contents of the given URL. */
- (NSURLSessionDataTask *)dataTaskWithURL:(NSURL *)url;

/* Creates an upload task with the given request.  The body of the request will be created from the file referenced by fileURL */
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromFile:(NSURL *)fileURL;

/* Creates an upload task with the given request.  The body of the request is provided from the bodyData. */
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromData:(NSData *)bodyData;

/* Creates an upload task with the given request.  The previously set body stream of the request (if any) is ignored and the URLSession:task:needNewBodyStream: delegate will be called when the body payload is required. */
- (NSURLSessionUploadTask *)uploadTaskWithStreamedRequest:(NSURLRequest *)request;

/* Creates a download task with the given request. */
- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request;

/* Creates a download task to download the contents of the given URL. */
- (NSURLSessionDownloadTask *)downloadTaskWithURL:(NSURL *)url;

/* Creates a download task with the resume data.  If the download cannot be successfully resumed, URLSession:task:didCompleteWithError: will be called. */
- (NSURLSessionDownloadTask *)downloadTaskWithResumeData:(NSData *)resumeData;
```



-  NSURLSession实际上应该理解成`一组相关的类库集合`包括了: 
	- NSURLRequest、NSMutableURLRequest
	- NSURLCache
	- NSURLResponse
	- NSURLProtocol
	- NSHTTPCookieStorage
	- NSURLCredentialStorage
	- 等等...
	
- 不管是`NSURLSession` 还是 `NSURLConnection` 要发起网络请求，一样都是首先创建`NSURLRequest`的实例，来确定要访问的服务器、请求参数、请求方法、请求路径...

```
//1.
NSURL *requestURL = [NSURL URLWithString:@"http://请求服务器郁闷"];
    
//2.
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:requestURL
                                                       cachePolicy:1
                                                   timeoutInterval:15.f];
    
//4. 设置POST请求
request.HTTPMethod = @"POST";
    
//5. 构造请求体参数
NSString *argument = [NSString stringWithFormat:@"loginname=zhangsan&password=123456"];
    
//6. 将请求参数字符串转化成NSData
NSData *requestBody = [argument dataUsingEncoding:NSUTF8StringEncoding];
    
//7. 设置POST请求请求体
request.HTTPBody = requestBody;
```

- 先来看一下使用`NSURLConnection`执行一个简单的网络请求的步骤:

```
[NSURLConnection sendAsynchronousRequest:request
                                   queue:[NSOperationQueue mainQueue]
                       completionHandler:^(NSURLResponse *response, NSData *data, NSError *error) {
			 // 请求回调Block
}];
```

```
AFN是在一个单例子线程的runloop上schedule执行NSURLConection，如上只是列举一个简单的使用NSURLConection例子
```

- 再看一下使用`NSURLSession`执行一个简单的网络请求的步骤:

```
//1. NSURLSession实例
NSURLSession *session = [NSURLSession sharedSession];

//2. NSURLSession实例，创建出一个Task实例，并使用NSURSession的分类提供的遍历方法，使用Block形式回调接收回传值，就不用实现代理方法了.
NSURLSessionDataTask *task = [session dataTaskWithRequest:request
                                     completionHandler:
 ^(NSData *data, NSURLResponse *response, NSError *error) {
     // ...
 }];

//3. 让Task实例开始执行
[task resume];

```

- 小结NSURLSession使用步骤:

1. 定义一个NSURLRequest。
2. 定义一个NSURLSessionConfiguration，`配置`各种网络参数。
3. 使用NSURLSession的工厂方法获取一个所需类型的NSURLSession。
4. 使用定义好的NSURLRequest和NSURLSession构建一个NSURLSessionTask。
5. 使用Delegate或者CompletionHandler处理任务执行过程的所有事件。

***

- 好大概知道了`NSURLSession`，然后再看看上面的AFN的代码:

```
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                            completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    //1. 
    __block NSURLSessionDataTask *dataTask = nil;
    
	//2. 
    dispatch_sync(url_session_manager_creation_queue(), ^{
        dataTask = [self.session dataTaskWithRequest:request];
    });

    //3. 
    [self addDelegateForDataTask:dataTask completionHandler:completionHandler];

    return dataTask;
}
```

- Task实例创建完毕之后，就要开始执行了`resume`

```
- (NSURLSessionDataTask *)GET:(NSString *)URLString
                   parameters:(id)parameters
                      success:(void (^)(NSURLSessionDataTask *task, id responseObject))success
                      failure:(void (^)(NSURLSessionDataTask *task, NSError *error))failure
{
    NSURLSessionDataTask *dataTask = [self dataTaskWithHTTPMethod:@"GET" URLString:URLString parameters:parameters success:success failure:failure];
	
	//开始执行Task
    [dataTask resume];

    return dataTask;
}
```

OK，到这里位置，一个网络请求操作已经开始执行了。

****

###AFQueryStringPair这个内部类，用来封装`键值对参数 key=value`的

```objc
@interface AFQueryStringPair : NSObject
/**
 *  键
 */
@property (readwrite, nonatomic, strong) id field;

/**
 *  值
 */
@property (readwrite, nonatomic, strong) id value;

/**
 *  key - value
 */
- (id)initWithField:(id)field value:(id)value;

/**
 *  替换掉value中的 URL字符集之外的 字符串
 */
- (NSString *)URLEncodedStringValue;

@end

@implementation AFQueryStringPair

- (id)initWithField:(id)field value:(id)value {
    self = [super init];
    if (!self) {
        return nil;
    }

    self.field = field;
    self.value = value;

    return self;
}

- (NSString *)URLEncodedStringValue {
    if (!self.value || [self.value isEqual:[NSNull null]]) {
    	//value空
        return AFPercentEscapedStringFromString([self.field description]);
    } else {
    	//key=value组织
        return [NSString stringWithFormat:@"%@=%@", AFPercentEscapedStringFromString([self.field description]), AFPercentEscapedStringFromString([self.value description])];
    }
}
```

关于这个`AFQueryStringPair`封装键值对参数的具体，有专门的文章记录《键值对参数的封装》

***

###AFMultipartFormData 协议，抽象出一个用于`上传`网络请求时使用到的拼接上传数据的工具


```objc
/**
 The `AFMultipartFormData` protocol defines the methods supported by the parameter in the block argument of `AFHTTPRequestSerializer -multipartFormRequestWithMethod:URLString:parameters:constructingBodyWithBlock:`.
 */
@protocol AFMultipartFormData

/**
1. Appends the HTTP header 
2. 参数形式 = `Content-Disposition: file; filename=#{generated filename}; name=#{name}"` and `Content-Type: #{generated mimeType}`
3. followed by the encoded file data and the multipart form boundary.
4. The filename and MIME type for this data in the form will be automatically generated, using the last path component of the `fileURL` and system associated MIME type for the `fileURL` extension, respectively.

 @param fileURL The URL corresponding to the file whose content will be appended to the form. This parameter must not be `nil`.
 @param name The name to be associated with the specified data. This parameter must not be `nil`.
 @param error If an error occurs, upon return contains an `NSError` object that describes the problem.

 @return `YES` if the file data was successfully appended, otherwise `NO`.
 */
- (BOOL)appendPartWithFileURL:(NSURL *)fileURL
                         name:(NSString *)name
                        error:(NSError * __nullable __autoreleasing *)error;

/**
1. Appends the HTTP header
2. 参数形式 = `Content-Disposition: file; filename=#{filename}; name=#{name}"` and `Content-Type: #{mimeType}`
3. followed by the encoded file data and the multipart form boundary.

 @param fileURL The URL corresponding to the file whose content will be appended to the form. This parameter must not be `nil`.
 @param name The name to be associated with the specified data. This parameter must not be `nil`.
 @param fileName The file name to be used in the `Content-Disposition` header. This parameter must not be `nil`.
 @param mimeType The declared MIME type of the file data. This parameter must not be `nil`.
 @param error If an error occurs, upon return contains an `NSError` object that describes the problem.

 @return `YES` if the file data was successfully appended otherwise `NO`.
 */
- (BOOL)appendPartWithFileURL:(NSURL *)fileURL
                         name:(NSString *)name
                     fileName:(NSString *)fileName
                     mimeType:(NSString *)mimeType
                        error:(NSError * __nullable __autoreleasing *)error;

/**
1. Appends the HTTP header
2. 参数形式 = `Content-Disposition: file; filename=#{filename}; name=#{name}"` and `Content-Type: #{mimeType}`
3. followed by the data from the input stream and the multipart form boundary.

 @param inputStream The input stream to be appended to the form data
 @param name The name to be associated with the specified input stream. This parameter must not be `nil`.
 @param fileName The filename to be associated with the specified input stream. This parameter must not be `nil`.
 @param length The length of the specified input stream in bytes.
 @param mimeType The MIME type of the specified data. (For example, the MIME type for a JPEG image is image/jpeg.) For a list of valid MIME types, see http://www.iana.org/assignments/media-types/. This parameter must not be `nil`.
 */
- (void)appendPartWithInputStream:(nullable NSInputStream *)inputStream
                             name:(NSString *)name
                         fileName:(NSString *)fileName
                           length:(int64_t)length
                         mimeType:(NSString *)mimeType;

/**
1. Appends the HTTP header 
2. 参数形式 = `Content-Disposition: file; filename=#{filename}; name=#{name}"` and `Content-Type: #{mimeType}`
3.  followed by the encoded file data and the multipart form boundary.

 @param data The data to be encoded and appended to the form data.
 @param name The name to be associated with the specified data. This parameter must not be `nil`.
 @param fileName The filename to be associated with the specified data. This parameter must not be `nil`.
 @param mimeType The MIME type of the specified data. (For example, the MIME type for a JPEG image is image/jpeg.) For a list of valid MIME types, see http://www.iana.org/assignments/media-types/. This parameter must not be `nil`.
 */
- (void)appendPartWithFileData:(NSData *)data
                          name:(NSString *)name
                      fileName:(NSString *)fileName
                      mimeType:(NSString *)mimeType;

/**
1. Appends the HTTP headers 
2. 参数形式 = `Content-Disposition: form-data; name=#{name}"`,
3. followed by the encoded data and the multipart form boundary.

 @param data The data to be encoded and appended to the form data.
 @param name The name to be associated with the specified data. This parameter must not be `nil`.
 */

- (void)appendPartWithFormData:(NSData *)data
                          name:(NSString *)name;


/**
1. Appends HTTP headers
2. followed by the encoded data and the multipart form boundary.

 @param headers The HTTP headers to be appended to the form data.
 @param body The data to be encoded and appended to the form data. This parameter must not be `nil`.
 */
- (void)appendPartWithHeaders:(nullable NSDictionary *)headers
                         body:(NSData *)body;

/**
 Throttles request bandwidth by limiting the packet size and adding a delay for each chunk read from the upload stream.

 1. When uploading over a 3G or EDGE connection
 当在3G或2.5G网络连接下，进行上传网络请求时
 
 2. requests may fail with "request body stream exhausted".
 可能会出现请求错误，提示为 `request body stream exhausted` 请求传输流被耗尽，因为超过其网络连接的最大带宽
 
 3. Setting a maximum packet size and delay according to the recommended values (`kAFUploadStream3GSuggestedPacketSize` and `kAFUploadStream3GSuggestedDelay`) lowers the risk of the input stream exceeding its allocated bandwidth.
 设置一个最大数据包大小和延迟事件，可以根据推荐值(“kAFUploadStream3GSuggestedPacketSize”和“kAFUploadStream3GSuggestedDelay”)降低了输入流的风险超过其分配带宽

 4. Unfortunately, there is no definite way to distinguish between a 3G, EDGE, or LTE connection over `NSURLConnection`. As such, it is not recommended that you throttle bandwidth based solely on network reachability. Instead, you should consider checking for the "request body stream exhausted" in a failure block, and then retrying the request with throttled bandwidth.
 不幸的是，没有明确的方法来区分3g、2.5G、LTE网络质量，在使用NSURLConnection网络连接时.
 因此，不建议您 节流带宽 完全基于 网络可达性.
 相反，应该考虑在网络请求失败的回调block中检查错误信息是否是`request body stream exhausted`
 然后重新以 节流宽带的方式 进行网络请求

 @param numberOfBytes Maximum packet size, in number of bytes. The default packet size for an input stream is 16kb.
 @param delay Duration of delay each time a packet is read. By default, no delay is set.
 */
- (void)throttleBandwidthWithPacketSize:(NSUInteger)numberOfBytes
                                  delay:(NSTimeInterval)delay;

@end
```

###AFMultipartFormData 抽象上传网络请求协议的 实现类 AFStreamingMultipartFormData

- AFStreamingMultipartFormData类声明

```objc
@interface AFStreamingMultipartFormData : NSObject <AFMultipartFormData>

/**
 *  传入的一个请求对象NSURLRequest
 *  默认编码为UFT8
 */
- (instancetype)initWithURLRequest:(NSMutableURLRequest *)urlRequest
                    stringEncoding:(NSStringEncoding)encoding;

/**
 *  重新设置 NSURLRequest.header 请求头参数
 * 
 *  1. 上传数据类型 Content-Type
 *  2. 上传数据大小 Content-Length
 */
- (NSMutableURLRequest *)requestByFinalizingMultipartFormData;

@end
```

- AFStreamingMultipartFormData 匿名分类声明辅助对象

```objc
@interface AFStreamingMultipartFormData ()

/**
 *  网络请求对象NSURLRequest
 */
@property (readwrite, nonatomic, copy) NSMutableURLRequest *request;

/**
 *  内容编码格式
 */
@property (readwrite, nonatomic, assign) NSStringEncoding stringEncoding;

/**
 *  拼接上传请求头参数的 分割符
 */
@property (readwrite, nonatomic, copy) NSString *boundary;

/**
 *  AFNetworking封装的一个输入流NSInputStream对象
 */
@property (readwrite, nonatomic, strong) AFMultipartBodyStream *bodyStream;

@end
```

如上涉及到了一个AFNetworking封装的输入流类 `AFMultipartBodyStream`，稍后说.

- AFStreamingMultipartFormData 具体实现implemtation部分

首先是之前AFStreamingMultipartFormData @interface中声明的initXxx方法实现

```objc
@implementation AFStreamingMultipartFormData

- (id)initWithURLRequest:(NSMutableURLRequest *)urlRequest
          stringEncoding:(NSStringEncoding)encoding
{
    self = [super init];
    if (!self) {
        return nil;
    }

    //1. 保存要包装的请求对象
    self.request = urlRequest;
    
    //2. 编码格式
    self.stringEncoding = encoding;
    
    //3. 保存分割符，使用的一个静态c函数计算得到
    self.boundary = AFCreateMultipartFormBoundary();
    
    //4. 创建此次上传Request请求对应的 `输入流`对象
    self.bodyStream = [[AFMultipartBodyStream alloc] initWithStringEncoding:encoding];

    return self;
}

@end
```

```objc
/**
 *	 1. 获取一个分隔符，使用随机数
 *	 
 *	 2. 输出数字格式 %08X
 *	 	2.1 %x 代表16进制输出的字母符号为 小写.
 *	  	2.2 %X 代表16进制输出的字母符号为 大写.
 *	   2.3 08 指定数据的最小输出位数为8位，若不够8位，则补零，若大于8位，则按照原位数输出.
 */
static NSString * AFCreateMultipartFormBoundary() {
    return [NSString stringWithFormat:@"Boundary+%08X%08X", arc4random(), arc4random()];
}
```

###先看看 AFStreamingMultipartFormData依赖的一个输入流类 AFMultipartBodyStream

####AFMultipartBodyStream @interface声明

```objc
@interface AFMultipartBodyStream : NSInputStream <NSStreamDelegate>

@property (nonatomic, assign) NSUInteger numberOfBytesInPacket;

@property (nonatomic, assign) NSTimeInterval delay;

@property (nonatomic, strong) NSInputStream *inputStream;

@property (readonly, nonatomic, assign) unsigned long long contentLength;

@property (readonly, nonatomic, assign, getter = isEmpty) BOOL empty;

- (id)initWithStringEncoding:(NSStringEncoding)encoding;

- (void)setInitialAndFinalBoundaries;

- (void)appendHTTPBodyPart:(AFHTTPBodyPart *)bodyPart;

@end
```

其中AFMultipartBodyStream又依赖一个类AFHTTPBodyPart.

####AFMultipartBodyStream匿名分类 与 NSStream匿名分类

```objc
/**
 *  1. 让NSStream（流）系统属性读写权限修改为readwrite
 *  2. 系统默认是readonly
 *  3. NSStream（流）层级
 *  	3.1 NSInputStream（输入流） : NSStream（流）
 *  	3.2 NSOutputStream（输出流） : NSStream（流n）
 */
@interface NSStream ()

/**
 *  输入/输出流的状态
 */
@property (readwrite) NSStreamStatus streamStatus;

/**
 *  输入/输出流出现的错误
 */
@property (readwrite, copy) NSError *streamError;

@end

@interface AFMultipartBodyStream () <NSCopying>

@property (readwrite, nonatomic, assign) NSStringEncoding stringEncoding;

@property (readwrite, nonatomic, strong) NSMutableArray *HTTPBodyParts;

@property (readwrite, nonatomic, strong) NSEnumerator *HTTPBodyPartEnumerator;

@property (readwrite, nonatomic, strong) AFHTTPBodyPart *currentHTTPBodyPart;

@property (readwrite, nonatomic, strong) NSOutputStream *outputStream;

@property (readwrite, nonatomic, strong) NSMutableData *buffer;

@end
```

####AFMultipartBodyStream具体实现 @implementation

```objc
@implementation AFMultipartBodyStream

/** 消除警告 */
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wimplicit-atomic-properties"
#if (defined(__IPHONE_OS_VERSION_MAX_ALLOWED) && __IPHONE_OS_VERSION_MAX_ALLOWED >= 80000) || (defined(__MAC_OS_X_VERSION_MAX_ALLOWED) && __MAC_OS_X_VERSION_MAX_ALLOWED >= 1100)
@synthesize delegate;
#endif

@synthesize streamStatus;
@synthesize streamError;
#pragma clang diagnostic pop


/** init初始化 */
```