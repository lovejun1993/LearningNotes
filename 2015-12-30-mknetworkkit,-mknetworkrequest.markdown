---
layout: post
title: "MKNetworkKit、MKNetworkRequest"
date: 2015-12-30 17:57:40 +0800
comments: true
categories: 
---

###MKNetworkRequest有点类似YTKRequest的封装思路.

***

###首先看到支持 三种 请求参数拼接方式

```
typedef enum {
  
  //1. 拼接在url后面，以key-value形式
  MKNKParameterEncodingURL = 0, // default
  
  //2. json形式
  MKNKParameterEncodingJSON,
  
  //3. plist形式
  MKNKParameterEncodingPlist
  
} MKNKParameterEncoding;
```

***

###然后看到定义了 七种 MKNetworkRequest的状态

```
typedef enum {
	
	//1. 就绪
	MKNKRequestStateReady = 0,
	
	//2. 开始
	MKNKRequestStateStarted,
	
	//3. 响应数据来自缓存，并且可用
	MKNKRequestStateResponseAvailableFromCache,
	
	//4. 响应数据来自缓存，但是不可用（过期）
	MKNKRequestStateStaleResponseAvailableFromCache,
	
	//5. 取消
	MKNKRequestStateCancelled,
	
	//6. 成功结束
	MKNKRequestStateCompleted,
	
	//7. 错误结束
	MKNKRequestStateError
	
} MKNKRequestState;
```

***

###定义了全局Block类型

```
typedef void (^MKNKHandler)(MKNetworkRequest* completedRequest);
```

###MKNetworkRequest初始化

```
- (instancetype)initWithURLString:(NSString *) aURLString
                           params:(NSDictionary*) params
                         bodyData:(NSData *) bodyData
                       httpMethod:(NSString *) httpMethod
{
    if(self = [super init]) {
        
        //使用数组保存所有的变化状态
        self.stateArray = [NSMutableArray array];
      
        //请求全路径字符串
        self.urlString = aURLString;
      
        //参数字典
        if(params) {
            self.parameters = params.mutableCopy;
        } else {
            self.parameters = [NSMutableDictionary dictionary];
        }
    
        //请求体body
        self.bodyData = bodyData;
        
        //请求方法
        self.httpMethod = httpMethod;
    
        //请求头参数字典
        self.headers = [NSMutableDictionary dictionary];
    
        //普通 请求回调block数组
        self.completionHandlers = [NSMutableArray array];
        
        //上传 请求回调block数组
        self.uploadProgressChangedHandlers = [NSMutableArray array];
        
        //下载 请求回调block数组
        self.downloadProgressChangedHandlers = [NSMutableArray array];
    
        //所有下载文件NSData数组
        self.attachedData = [NSMutableArray array];
        
        //所有下载文件数组
        self.attachedFiles = [NSMutableArray array];
    }
  
    return self;
}
```

- 使用 懒加载 创建 `NSURLRequest`，对 GET 和 POST 分情况进行参数拼接处理

```
-(NSMutableURLRequest*) request {
  
    NSURL *url = nil;
    
    //1. 针对 GET、DELETE、HEAD 这三种请求，直接拼接请求参数到url
    if (([self.httpMethod.uppercaseString isEqualToString:@"GET"] || \
         [self.httpMethod.uppercaseString isEqualToString:@"DELETE"] || \
         [self.httpMethod.uppercaseString isEqualToString:@"HEAD"]) && \
        (self.parameters.count > 0))
    {
        // 将 参数字典 转换成 name=zhangsan&pwd=@"123456" 字符串个数
        NSString *keyAndValues = [self.parameters urlEncodedKeyValueString];

        // 将参数字符串拼接到请求路径后面
        url = [NSURL URLWithString:[NSString stringWithFormat:@"%@?%@", self.urlString,
                                    keyAndValues]];
    } else {
        
        //没有请求参数
        url = [NSURL URLWithString:self.urlString];
    }
  
    //2. 没有请求url
    if(url == nil)
    {
        NSAssert(@"Unable to create request %@ %@ with parameters %@",
              self.httpMethod, self.urlString, self.parameters);
        return nil;
    }
  
    //3. 使用 url 创建 NSMutableURLRequest实例
    NSMutableURLRequest *createdRequest = [NSMutableURLRequest requestWithURL:url];
    
    //4. 设置 请求头 字典
    [createdRequest setAllHTTPHeaderFields:self.headers];
    
    //4. 设置 请求方法
    [createdRequest setHTTPMethod:self.httpMethod];
    
    //5. 保存请求体body字符串
    NSString *bodyStringFromParameters = nil;
    
    //6. 告诉服务器参数类型为NSUTF8StringEncoding
    NSString *charset = (__bridge NSString *)CFStringConvertEncodingToIANACharSetName(CFStringConvertNSStringEncodingToEncoding(NSUTF8StringEncoding));
  
    //7. 按照不同的方式 拼接 参数字符串
    switch (self.parameterEncoding) {
      
        //7.1 key=value格式
        case MKNKParameterEncodingURL: {
            
            NSString *headerValue = [NSString stringWithFormat:@"application/x-www-form-urlencoded; charset=%@", charset];
            
            [createdRequest setValue:headerValue
                  forHTTPHeaderField:@"Content-Type"];
           
            //body字符串为 key=value&key=value格式的 字符串
            bodyStringFromParameters = [self.parameters urlEncodedKeyValueString];
        }
          break;
            
        //7.2 JSON格式
        case MKNKParameterEncodingJSON: {
            
            NSString *headerValue = [NSString stringWithFormat:@"application/json; charset=%@", charset];
            
            [createdRequest setValue:headerValue
                  forHTTPHeaderField:@"Content-Type"];
            
            //body字符串为 json格式的 字符串
            bodyStringFromParameters = [self.parameters jsonEncodedKeyValueString];
        }
          break;
            
        //7.3 Plist格式字符串
        case MKNKParameterEncodingPlist: {
            
            NSString *headerValue = [NSString stringWithFormat:@"application/x-plist; charset=%@", charset];
            
            [createdRequest setValue:headerValue
                  forHTTPHeaderField:@"Content-Type"];
            
            //body字符串为 plist格式的 字符串
            bodyStringFromParameters = [self.parameters plistEncodedKeyValueString];
        }
    }
  
    //8. 只有POST请求，
    //才将 请求参数转换为 NSData
    //设置给 NSMutableURLRequest的httpBodt
    if (!([self.httpMethod.uppercaseString isEqualToString:@"GET"] ||
        [self.httpMethod.uppercaseString isEqualToString:@"DELETE"] ||
        [self.httpMethod.uppercaseString isEqualToString:@"HEAD"]))
    {
        //将body字符串 转换成 NSData
        NSData *bodyParam = [bodyStringFromParameters dataUsingEncoding:NSUTF8StringEncoding];
        
        //将NSData 设置给 NSMutableURLRequeat的httpBody属性
        [createdRequest setHTTPBody:bodyParam];
    }
  
    //9. 如果有传入的body data，那么就是要外界的body data
    if(self.bodyData) {
        [createdRequest setHTTPBody:self.bodyData];
    }
  
    //10. 上传多个文件
    if(self.attachedFiles.count > 0 || self.attachedData.count > 0)
    {
        //10.1 告诉服务器文件数据编码格式
        NSString *charset = (__bridge NSString *)CFStringConvertEncodingToIANACharSetName(CFStringConvertNSStringEncodingToEncoding(NSUTF8StringEncoding));
    
        //10.2 上传文件类型
        NSString *headerValue = [NSString stringWithFormat:@"multipart/form-data; charset=%@; boundary=%@",
         charset, kBoundary];

        [createdRequest setValue:headerValue forHTTPHeaderField:@"Content-Type"];
    
        //10.3 上传文件长度
        //HEAVY OPERATION!
        NSString *contentLength = [NSString stringWithFormat:@"%lu", (unsigned long) [self.multipartFormData length]];

        [createdRequest setValue:contentLength forHTTPHeaderField:@"Content-Length"];
  }
  
  return createdRequest;
}
```

***

###然后看看 MKNetworkRequest的所有属性

- 最终发起网络请求最终需要的 请求数据 来源

```
@property (readonly) NSMutableURLRequest *request;
```

- 响应数据描述

```
@property (readonly) NSHTTPURLResponse *response;
```

- 请求参数拼接方式

```
@property MKNKParameterEncoding parameterEncoding;
```

- 状态，重写了setter方法，修改设置状态时，顺便执行回调Block

```
注意，虽然此处是不允许外界修改，
但是MKNetworkHost通过声明了一个MKNetworkRequest分类，
覆盖了MKNetworkRequest内部的readonly
那么也就是说只有MKNetworkHost可以readwrite这个属性，
而其他类对象都是readonly

@property (readonly) MKNKRequestState state;
```

```
-(void) setState:(MKNKRequestState)state {
  
  //1. 切换最新状态
  _state = state;
  
  //2. 数组保存 当前传入的状态
  [self.stateArray addObject:@(state)];
  
  //3. 依次判断当前状态，执行不同的操作
  if(state == MKNKRequestStateStarted) {
    
    //3.1 开始请求服务器获取最新数据
    
    NSAssert(self.task, @"Task missing");
    
    //让保存的 NSURLSessionTask开始执行
    [self.task resume];
    
    //累加当前总请求数
    [self incrementRunningOperations];
      
  } else if(state == MKNKRequestStateResponseAvailableFromCache ||
     state == MKNKRequestStateStaleResponseAvailableFromCache) {
    
    //3.2 读取缓存数据
    
    //直接执行当前MKNetworkRequest保存的所有block
    //回传缓存数据
    [self.completionHandlers enumerateObjectsUsingBlock:^(MKNKHandler handler, NSUInteger idx, BOOL *stop) {
      	
      	//回传缓存数据
		 handler(self);
    }];
    
  } else if(state == MKNKRequestStateCompleted ||
            state == MKNKRequestStateError) {

	//3.3 请求正常结束 or 请求错误结束
	
	//让请求总记录数 - 1
    [self decrementRunningOperations];
    
    //执行回调block
    [self.completionHandlers enumerateObjectsUsingBlock:^(MKNKHandler handler, NSUInteger idx, BOOL *stop) {
      
      //回传缓存数据
      handler(self);
    }];
    
  } else if(state == MKNKRequestStateCancelled) {
    
    //3.4 请求取消执行
    
    //让请求总记录数 - 1
    [self decrementRunningOperations];
  }
}
```

- 需要使用用户名与密码进行服务器挑战认证的

```
@property NSString *username;
@property NSString *password;
```

- 需要使用证书进行认证

```
@property NSString *clientCertificate;
@property NSString *clientCertificatePassword;
```

- 下载url路径

```
@property NSString *downloadPath;
```

- 是否需要认证

```
这个属性其实是读取 
1. 使用用户名与密码进行服务器挑战认证的
2. 使用证书进行认证
只有使用如上任意一种认证，那么就返回YES，否则返回NO

@property (readonly) BOOL requiresAuthentication;
```

```
-(BOOL) requiresAuthentication {
  
  return (self.username != nil ||
          self.password != nil ||
          self.clientCertificate != nil ||
          self.clientCertificatePassword != nil);
}
```

- 是否使用 https:// 协议

```
@property (readonly) BOOL isSSL;
```

```
-(BOOL) isSSL {
  
  return [self.request.URL.scheme.lowercaseString isEqualToString:@"https"];
}
```

- 关于缓存使用策略，有点类似 `NSURLRequestCachePolicy`

```
//1. 不使用缓存
@property BOOL doNotCache;

//2. 一直使用缓存
@property BOOL alwaysCache;

//3. 忽略缓存
@property BOOL ignoreCache;

//4. 每一次都是发起一次新的请求
@property BOOL alwaysLoad;
```

- 请求方法

```
@property NSString *httpMethod;
```

- response描述是否来自缓存

```
@property (readonly) BOOL isCachedResponse;
```

```
-(BOOL) isCachedResponse {
  
  //1.未超时 
  //2.已超时
  //这两种都是来自缓存
  return self.state == MKNKRequestStateResponseAvailableFromCache || \
  		 self.state == MKNKRequestStateStaleResponseAvailableFromCache;
}
```

- 缓存response是否可用

```
@property (readonly) BOOL responseAvailable;
```

```
-(BOOL) responseAvailable {
 	
 	 //1. 未超时
 	 //2. 已超时
 	 //3. 请求结束
	return self.state == MKNKRequestStateResponseAvailableFromCache || \
	       self.state == MKNKRequestStateStaleResponseAvailableFromCache || \
	       self.state == MKNKRequestStateCompleted;
}
```

- 获取当前请求，要上传的数据 NSData

```
@property (readonly) NSData *multipartFormData;
```

```
懒加载获取最终上传的NSData数据

-(NSData*) multipartFormData {

    //1. 支持 多个NSData 上传
    //2. 支持 多个本地文件 上传
    if(self.attachedData.count == 0 && self.attachedFiles.count == 0) {
        return nil;
    }
    
    //3. 最终上传的NSData
    NSMutableData *formData = [NSMutableData data];
    
    //----------------------------- 开始参数拼接 -----------------------------
    //4. 拼接 请求参数
    [self.parameters enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
        
        
        NSString *thisFieldString = [NSString stringWithFormat:
                                     @"--%@\r\nContent-Disposition: form-data; name=\"%@\"\r\n\r\n%@",
                                     kBoundary, key, obj];

        [formData appendData:[thisFieldString dataUsingEncoding:NSUTF8StringEncoding]];
        
        [formData appendData:[@"\r\n" dataUsingEncoding:NSUTF8StringEncoding]];
    }];

    //5. 拼接 上传文件
    [self.attachedFiles enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
        
        //每一个上传文件项使用一个字典描述
        //描述1、name 文件名
        //描述2、filepath 文件路径
        //描述3、mimetype 文件类型
        NSDictionary *thisFile = (NSDictionary*) obj;
        
        NSString *thisFieldString = [NSString stringWithFormat:
                                     @"--%@\r\nContent-Disposition: form-data; name=\"%@\"; filename=\"%@\"\r\nContent-Type: %@\r\nContent-Transfer-Encoding: binary\r\n\r\n",
                                     kBoundary,
                                     thisFile[@"name"],
                                     [thisFile[@"filepath"] lastPathComponent],
                                     thisFile[@"mimetype"]];

    
        [formData appendData:[thisFieldString dataUsingEncoding:NSUTF8StringEncoding]];
   
        //读取文件路径指向的文件
        [formData appendData: [NSData dataWithContentsOfFile:thisFile[@"filepath"]]];
   
        [formData appendData:[@"\r\n" dataUsingEncoding:NSUTF8StringEncoding]];
    }];

    //6. 拼接 上传文件的NSData数据
    [self.attachedData enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {

        //与上面一样
        NSDictionary *thisDataObject = (NSDictionary*) obj;
        
        NSString *thisFieldString = [NSString stringWithFormat:
                                     @"--%@\r\nContent-Disposition: form-data; name=\"%@\"; filename=\"%@\"\r\nContent-Type: %@\r\nContent-Transfer-Encoding: binary\r\n\r\n",
                                     kBoundary,
                                     thisDataObject[@"name"],
                                     thisDataObject[@"filename"],
                                     thisDataObject[@"mimetype"]];

        [formData appendData:[thisFieldString dataUsingEncoding:NSUTF8StringEncoding]];

        [formData appendData:thisDataObject[@"data"]];

        [formData appendData:[@"\r\n" dataUsingEncoding:NSUTF8StringEncoding]];
    }];

    //7. 结束参数拼接
    [formData appendData: [[NSString stringWithFormat:@"--%@--\r\n", kBoundary] dataUsingEncoding:NSUTF8StringEncoding]];

    return formData;
}
```

- 响应数据格式1、NSData

```
@property (readonly) NSData *responseData;
```

- 响应数据格式2、JSON

```
@property (readonly) id responseAsJSON;
```

```
-(id) responseAsJSON {
  
    if(self.responseData == nil)
        return nil;

    NSError *error = nil;

    id returnValue = [NSJSONSerialization JSONObjectWithData:self.responseData options:0 error:&error];

    if(!returnValue)
        NSLog(@"JSON Parsing Error: %@", error);

    return returnValue;
}
```

- 响应数据格式3、 UIImage

```
#if TARGET_OS_IPHONE

//图片解压缩
-(UIImage*) decompressedResponseImageOfSize:(CGSize) size;

@property (readonly) UIImage *responseAsImage;

#else

@property (readonly) NSImage *responseAsImage;
	
#endif
```

```
1. 单例创建此次请求对象的响应UIImage数据
2. 使用屏幕的比例创建UIImage

-(UIImage*) responseAsImage {
  
  static CGFloat scale = 2.0f;
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    scale = [UIScreen mainScreen].scale;
  });
  return [UIImage imageWithData:self.responseData scale:scale];
}
```

```
使用 CoreImage、ImageIO 完成图片解压缩

-(UIImage*) decompressedResponseImageOfSize:(CGSize) size {
  
  CGImageSourceRef source = CGImageSourceCreateWithData((__bridge CFDataRef)(self.responseData), NULL);
  CGImageRef cgImage = CGImageSourceCreateImageAtIndex(source, 0, (__bridge CFDictionaryRef)(@{(id)kCGImageSourceShouldCache:@(YES)}));
  UIImage *decompressedImage = [UIImage imageWithCGImage:cgImage];
  if(source)
    CFRelease(source);
  if(cgImage)
    CGImageRelease(cgImage);
  
  return decompressedImage;
}
```

- 响应数据格式4、NSString

```
@property (readonly) NSString *responseAsString;

-(NSString*) responseAsString {
  
  	// 首先使用 NSUTF8StringEncoding 编码格式
    NSString *string = [[NSString alloc] initWithData:self.responseData
                                           encoding:NSUTF8StringEncoding];

	// 如果前面一种编码格式失败，
	// 就使用 NSASCIIStringEncoding 编码格式
    if(self.responseData.length > 0 && !string)
    {
        string = [[NSString alloc] initWithData:self.responseData encoding:NSASCIIStringEncoding];
    }

    return string;
}
```

- 保存此次网络请求的错误

```
@property (readonly) NSError *error;
```

- 保存此次网络请求的Task

```
@property (readonly) NSURLSessionTask *task;
```

- 上传进度 或 下载进度

```
@property (readwrite) CGFloat progress;
```

- 当前请求是否可以使用缓存功能

```
@property (readonly) BOOL cacheable;
```

```
-(BOOL) cacheable {
  
    //1. 请求方法
    NSString *requestMethod = self.httpMethod.uppercaseString;
    
    //2. 只允许缓存 GET方式 的响应数据
    if(![requestMethod isEqualToString:@"GET"])
        return NO;
    
    //3. 如果请求自己设置了不允许缓存
    if(self.doNotCache)
        return NO;

    //4. 需要认证的请求，不能够缓存响应数据
    if(self.requiresAuthentication || self.isSSL) {
        return self.alwaysCache;
    } else {
        return YES;
    }
}
```

###接下来看下MKNetworkRequest向外暴露的方法

- 添加 请求参数字典

```
@property NSMutableDictionary *parameters;

-(void) addParameters:(NSDictionary*) paramsDictionary;
```

```
-(void) addParameters:(NSDictionary*) paramsDictionary {
  
  [self.parameters addEntriesFromDictionary:paramsDictionary];
}
```

- 添加 请求头参数字典

```
@property NSMutableDictionary *headers;

-(void) addHeaders:(NSDictionary*) headersDictionary;
```

```
-(void) addHeaders:(NSDictionary*) headersDictionary {
  
  [self.headers addEntriesFromDictionary:headersDictionary];
}
```

- 添加 认证类型的 请求头参数

```
-(void) setAuthorizationHeaderValue:(NSString*) token forAuthType:(NSString*) authType;
```

```
-(void) setAuthorizationHeaderValue:(NSString*) value forAuthType:(NSString*) authType {
    
    //1.
    NSString *param = [NSString stringWithFormat:@"%@ %@", authType, value];
    
    //2.
    self.headers[@"Authorization"] = param;
}
```

- 添加要上传指定路径的文件

```
/**
 *  @param filePath 文件所在磁盘路径
 *  @param key      上传到服务器指定的name参数，需要与服务器开发协商
 *  @param mimeType 文件类型
 */
-(void) attachFile:(NSString*) filePath
            forKey:(NSString*) key
          mimeType:(NSString*) mimeType;
```

```
-(void) attachFile:(NSString*) filePath
            forKey:(NSString*) key
          mimeType:(NSString*) mimeType
{    
    NSDictionary *dict = @{
    						@"filepath": filePath,
                         	@"name": key,//这个name参数必须和服务器协商好
                         	@"mimetype": mimeType
                         	};

	//支持多个文件上传
    [self.attachedFiles addObject:dict];
}
```

- 添加要上传文件的NSData数据

```
/**
 *  @param data     文件的NSData数据
 *  @param key      上传到服务器指定的name参数，需要与服务器开发协商
 *  @param mimeType 文件类型
 *  @param fileName 保存到服务器的文件名
 */
-(void) attachData:(NSData*) data
            forKey:(NSString*) key
          mimeType:(NSString*) mimeType
 suggestedFileName:(NSString*) fileName;
```

```
-(void) attachData:(NSData*) data
            forKey:(NSString*) key
          mimeType:(NSString*) mimeType
 suggestedFileName:(NSString*) fileName
{
    // 如果没有设置服务器文件名，使用传入的name值作为文件名
    if(!fileName)
        fileName = key;
  
    NSDictionary *dict = @{
                         @"data": data,
                         @"name": key,
                         @"mimetype": mimeType,
                         @"filename": fileName
                         };
  
    [self.attachedData addObject:dict];
}
```

- 添加 普通网络请求 的回调Block

```
-(void) addCompletionHandler:(MKNKHandler) completionHandler;
```

```
-(void) addCompletionHandler:(MKNKHandler) completionHandler {
 	
 	//数组保存block 
	[self.completionHandlers addObject:completionHandler];
}
```

- 添加 上传网络请求 的回调Block

```
-(void) addUploadProgressChangedHandler:(MKNKHandler) uploadProgressChangedHandler;
```

```
-(void) addUploadProgressChangedHandler:(MKNKHandler) uploadProgressChangedHandler {
    
    [self.uploadProgressChangedHandlers addObject:uploadProgressChangedHandler];
}
```

- 添加 下载网络请求 的回调Block

```
-(void) addDownloadProgressChangedHandler:(MKNKHandler) downloadProgressChangedHandler;
```

```
-(void) addDownloadProgressChangedHandler:(MKNKHandler) downloadProgressChangedHandler {
  
  [self.downloadProgressChangedHandlers addObject:downloadProgressChangedHandler];
}
```

- 回传 下载/上传 进度值

```
-(void) setProgressValue:(CGFloat) progressValue {
  
    //1. 让 KMNetworkRequest实例 保存最新进度值
    self.progress = progressValue;
  
    //2. 上传操作、回传 进度值（如果不是上传操作，那么downloadProgressChangedHandlers数组没有block）
    [self.downloadProgressChangedHandlers enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop)
    {
        MKNKHandler handler = obj;
        
        handler(self);
    }];
    
    //3. 下载操作、回传 进度值（同上）
    [self.uploadProgressChangedHandlers enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop)
    {
        MKNKHandler handler = obj;
        
        handler(self);
    }];
}
```

- 取消执行 MKNetworkRequest

```
-(void) cancel;
```

```
-(void) cancel {
    
    //只有 已经开始的request 才可以取消
    if(self.state == MKNKRequestStateStarted)
    {
        //取消执行 NSURLSessionTask
        [self.task cancel];
        
        //修改 request 状态，并执行回调block
        self.state = MKNKRequestStateCancelled;
    }
}
```

###私有函数

- 累加 总请求数

```
-(void) incrementRunningOperations {
  
#ifdef TARGET_OS_IPHONE
  dispatch_async(dispatch_get_main_queue(), ^{
    
    //累加
    numberOfRunningOperations ++;
    
    //显示菊花转圈
    if(numberOfRunningOperations > 0)
      [UIApplication sharedApplication].networkActivityIndicatorVisible = YES;
  });
#endif
}
```

- 递减 总请求数

```
-(void) decrementRunningOperations {
  
#ifdef TARGET_OS_IPHONE
  dispatch_async(dispatch_get_main_queue(), ^{
    
    //NSLog(@"%@", self.stateArray);
    
    //递减
    numberOfRunningOperations --;
    
    //隐藏菊花
    if(numberOfRunningOperations == 0)
      [UIApplication sharedApplication].networkActivityIndicatorVisible = NO;
    if(numberOfRunningOperations < 0) {
      NSLog(@"Number of operations is below zero. State Changes [%@]", self.stateArray); // FIX ME
    }
    
  });
#endif
}
```

- 使用MKNetworkRequest实例的hash值比较是否同一个请求
	- 但是 不针对 POST和PATCH

```
-(BOOL) isEqualToRequest:(MKNetworkRequest*) request {
    return [self hash] == [request hash];
}
```

###重写父类方法

- NSObject实例的 hash 方法

```
/*
 In case of requests,
 we assume that only two GET/DELETE/PUT/OPTIONS/HEAD requests
 POST and PATCH methods are not idempotent. So we return random numbers as hash
 */

-(NSUInteger) hash {
  
    if(!([self.httpMethod.uppercaseString isEqualToString:@"POST"] || \
         [self.httpMethod.uppercaseString isEqualToString:@"PATCH"]))
    {
        //GET DELETE PUT HEAD

        NSMutableString *str = [NSMutableString stringWithFormat:@"%@ %@",
                                self.httpMethod.uppercaseString,
                                self.request.URL.absoluteString];

        if(self.username)
            [str appendString:self.username];
        
        if(self.password)
            [str appendString:self.password];
        
        if(self.clientCertificate)
            [str appendString:self.clientCertificate];
        
        if(self.clientCertificatePassword)
            [str appendString:self.clientCertificatePassword];

        return [str hash];
        
    } else {
        
        //POST 和 PATCH 这两种请求，返回一个随机数
        //所有 这两种请求 的hash值 肯定不一样
        return arc4random();
    }
}
```

- NSObject实例的 isEqual: 方法

```
- (BOOL)isEqual:(id)object {
  
    //比较self指针
    if (self == object)
        return YES;
    
    //比较Class
    if (![object isKindOfClass:[MKNetworkRequest class]])
        return NO;
    
    //比较hash值
    return [self isEqualToRequest:(MKNetworkRequest*) object];
}
```

