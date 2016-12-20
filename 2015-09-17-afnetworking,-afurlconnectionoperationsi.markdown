---
layout: post
title: "AFNetworking、AFURLConnectionOperation四"
date: 2015-09-17 23:27:21 +0800
comments: true
categories: 
---

###AFURLConnectionOperation对象中的NSURLConnection所有的回调处理.

- NSURLConnectionDelegate
- NSURLConnectionDataDelegate

```objc
@interface AFURLConnectionOperation : NSOperation <NSURLConnectionDelegate, NSURLConnectionDataDelegate, NSSecureCoding, NSCopying>

....
```

***

###NSURLConnectionDelegate 主要处理

- 请求错误

- 请求需要接收web服务挑战

- 是否使用系统storage存储接收挑战的凭证


```objc
@protocol NSURLConnectionDelegate <NSObject>
@optional
- (void)connection:(NSURLConnection *)connection didFailWithError:(NSError *)error;
- (BOOL)connectionShouldUseCredentialStorage:(NSURLConnection *)connection;
- (void)connection:(NSURLConnection *)connection willSendRequestForAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge;

/** 如下的函数是iOS8以下的函数 */
- (BOOL)connection:(NSURLConnection *)connection canAuthenticateAgainstProtectionSpace:(NSURLProtectionSpace *)protectionSpace NS_DEPRECATED(10_6, 10_10, 3_0, 8_0, "Use -connection:willSendRequestForAuthenticationChallenge: instead.");
- (void)connection:(NSURLConnection *)connection didReceiveAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge NS_DEPRECATED(10_2, 10_10, 2_0, 8_0, "Use -connection:willSendRequestForAuthenticationChallenge: instead.");
- (void)connection:(NSURLConnection *)connection didCancelAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge NS_DEPRECATED(10_2, 10_10, 2_0, 8_0, "Use -connection:willSendRequestForAuthenticationChallenge: instead.");
@end
```

###NSURLConnectionDataDelegate 主要处理如下

- 请求重定向

- 接收到 响应描述 response

- 接收到 响应数据 response data

- 输入流 上传数据

- 缓存 响应描述 response

- 请求结束


```objc
@protocol NSURLConnectionDataDelegate <NSURLConnectionDelegate>
@optional
- (nullable NSURLRequest *)connection:(NSURLConnection *)connection willSendRequest:(NSURLRequest *)request redirectResponse:(nullable NSURLResponse *)response;
- (void)connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response;

- (void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data;

- (nullable NSInputStream *)connection:(NSURLConnection *)connection needNewBodyStream:(NSURLRequest *)request;
- (void)connection:(NSURLConnection *)connection   didSendBodyData:(NSInteger)bytesWritten
                                                 totalBytesWritten:(NSInteger)totalBytesWritten
                                         totalBytesExpectedToWrite:(NSInteger)totalBytesExpectedToWrite;

- (nullable NSCachedURLResponse *)connection:(NSURLConnection *)connection willCacheResponse:(NSCachedURLResponse *)cachedResponse;

- (void)connectionDidFinishLoading:(NSURLConnection *)connection;
@end
```

###下面开始看AFURLConnectionOperation.m对如上delegate方法实现

***

###实现一、接收到web服务挑战

```objc
- (void)connection:(NSURLConnection *)connection
willSendRequestForAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge
{
    /** 执行让用户自己接受挑战的block */
    if (self.authenticationChallenge) {
        self.authenticationChallenge(connection, challenge);
        return;
    }

    
    /** 下面是由框架完成挑战 */
    
    //1. 判断接收挑战的方法类型
    if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust])
    {
        //1.1.1 挑战类型是 服务器信任
        
        //1.1.2 使用 AFSecurityPolicy 判断当前host是否是被服务器信任的
        if ([self.securityPolicy evaluateServerTrust:challenge.protectionSpace.serverTrust forDomain:challenge.protectionSpace.host])
        {
            //1.1.2.1 是信任的
            
            //1.1.2.2 使用信任创建creditial凭证，完成挑战
            NSURLCredential *credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
            [[challenge sender] useCredential:credential forAuthenticationChallenge:challenge];
        } else {
            
            //1.1.2.1 非信任的
            
            //1.1.2.2 取消挑战
            [[challenge sender] cancelAuthenticationChallenge:challenge];
        }
    } else {
        
        //1.1.1 挑战类型是 非服务器信任
        
        //1.1.2
        if ([challenge previousFailureCount] == 0)
        {
            //1.1.2.1 没有出现挑战失败
            
            //1.2.2.2 判断是否用户自己设置过挑战凭证
            if (self.credential) {
                
                //使用用户自己设置的凭证完成挑战
                [[challenge sender] useCredential:self.credential forAuthenticationChallenge:challenge];
            } else {
                
                //不使用凭证继续挑战，会报错code=401
                [[challenge sender] continueWithoutCredentialForAuthenticationChallenge:challenge];
            }
        } else {
            
            //1.1.2.2 出现过一次以上挑战失败

            //不使用凭证继续挑战，会报错code=401
            [[challenge sender] continueWithoutCredentialForAuthenticationChallenge:challenge];
        }
    }
}
```

主要流程:

- 判断1、判断当前使用哪一种挑战方法
	- 服务器信任
	- 其他方式（账号密码、证书）

- 判断2、如果使用信任方式进行挑战，那么如何判断服务器信任？
	- AFSecurityPolicy

- 判断3、如果使用信任方式进行挑战
	- 被信任
		- 使用信任创建凭证，完成挑战
	- 不被信任
		- 直接取消挑战，请求失败结束

- 判断4、如果使用非信任方式进行挑战
	- 需要自己设置凭证

***

###实现二、是否使用系统storage存储挑战凭证

```objc
- (BOOL)connectionShouldUseCredentialStorage:(NSURLConnection __unused *)connection {
    return self.shouldUseCredentialStorage;
}
```

***

###实现三、处理网络请求被重定向

```objc
/**
 *  @param connection       网络连接
 *  @param request          原始网络请求
 *  @param redirectResponse 重定向到的请求对应的响应
 *
 *  @return 返回给NSURLConnection要发起的网络请求NSURLRequest对象
 */
- (NSURLRequest *)connection:(NSURLConnection *)connection
             willSendRequest:(NSURLRequest *)request
            redirectResponse:(NSURLResponse *)redirectResponse
{
    
    
    if (self.redirectResponse) {
        
        // 执行回调block，让用户对将要发起的网络请求进行修改
        // 最终交给NSURLConnection一个修改之后的请求
        /** 其实还可以通过NSURLProtocol实现 */
        return self.redirectResponse(connection, request, redirectResponse);
        
        
    } else {
        
        //直接交给NSURLConnection的是 当前要发起的请求
        
        return request;
    }
}
```

###实现四、上传数据回调

```objc
/**
 *  @param bytesWritten              已经上传的长度
 *  @param totalBytesWritten         总上传长度
 *  @param totalBytesExpectedToWrite 预期要上传的长度
 */
- (void)connection:(NSURLConnection __unused *)connection
   didSendBodyData:(NSInteger)bytesWritten
 totalBytesWritten:(NSInteger)totalBytesWritten
totalBytesExpectedToWrite:(NSInteger)totalBytesExpectedToWrite
{
    //主线程队列，回传进度值
    dispatch_async(dispatch_get_main_queue(), ^{
        
        if (self.uploadProgress) {
            
            self.uploadProgress((NSUInteger)bytesWritten,
                                totalBytesWritten,
                                totalBytesExpectedToWrite);
        }
    });
}
```

###实现五、接收到响应数据data，一种是一次性数据请求，另一种是下载请求

```objc
/**
 *  接收到响应数据，然后将数据写入到输出流中
 * 
 *  第一类、一次性数据请求
 *  第二类、下载请求，会多次回调
 */
- (void)connection:(NSURLConnection __unused *)connection
    didReceiveData:(NSData *)data
{
    //1. 响应数据总长度
    NSUInteger length = [data length];
    
    //2. 读取所有响应数据
    while (YES)
    {
        //记录当前总共写入的文件大小
        NSInteger totalNumberOfBytesWritten = 0;
        
        //判断输出流指向的区域，还能否被写入数据
        if ([self.outputStream hasSpaceAvailable])
        {
            //可以继续写入
            
            //开辟一个接收响应data数据的内存
            const uint8_t *dataBuffer = (uint8_t *)[data bytes];

            //记录写入当前接收到的data数据的写入大小
            NSInteger numberOfBytesWritten = 0;
            
            /** 不断的将接收到的data，写入到输出流对象中*/
            while (totalNumberOfBytesWritten < (NSInteger)length)
            {
                numberOfBytesWritten = [self.outputStream write:&dataBuffer[(NSUInteger)totalNumberOfBytesWritten]
                                                      maxLength:(length - (NSUInteger)totalNumberOfBytesWritten)];
                if (numberOfBytesWritten == -1) {
                    break;
                }

                totalNumberOfBytesWritten += numberOfBytesWritten;
            }

            break;
            
        } else {
            
            //不可以继续写入数据
            
            [self.connection cancel];
            
            if (self.outputStream.streamError)
            {
                [self performSelector:@selector(connection:didFailWithError:)
                           withObject:self.connection
                           withObject:self.outputStream.streamError];
            }
            
            return;
        }
    }

    //3. 主线程队列，回传当前接收的总数据
    /** 如果是下载请求，会多次回调 */
    dispatch_async(dispatch_get_main_queue(), ^{
        
        //累加长度
        self.totalBytesRead += (long long)length;

        //回传下载进度
        if (self.downloadProgress) {
            self.downloadProgress(length,
                                  self.totalBytesRead,
                                  self.response.expectedContentLength);
        }
    });
}
```

###实现六、网络请求结束

```objc
- (void)connectionDidFinishLoading:(NSURLConnection __unused *)connection {
    
    //1. 从输出流中获取data数据
    self.responseData = [self.outputStream propertyForKey:NSStreamDataWrittenToMemoryStreamKey];

    //2. 关闭输出流
    [self.outputStream close];
    
    if (self.responseData) {
       self.outputStream = nil;
    }

    //释放掉网络连接
    self.connection = nil;

    //结束operation执行
    [self finish];
}
```

###实现七、请求错误

```objc
- (void)connection:(NSURLConnection __unused *)connection
  didFailWithError:(NSError *)error
{
    self.error = error;

    [self.outputStream close];
    
    if (self.responseData) {
        self.outputStream = nil;
    }

    self.connection = nil;

    [self finish];
}
```

###实现八、缓存response

```objc
- (NSCachedURLResponse *)connection:(NSURLConnection *)connection
                  willCacheResponse:(NSCachedURLResponse *)cachedResponse
{
    //判断是否由用户缓存response
    //1. 回调用户设置的缓存block，让用户缓存
    //2. 交给NSURLConenction进行缓存
    
    if (self.cacheResponse)
    {
        return self.cacheResponse(connection, cachedResponse);
        
    } else {
        
        if ([self isCancelled]) {
            return nil;
        }

        return cachedResponse;
    }
}
```

***

其中关于 web服务挑战 使用 `AFSecurityPolicy` 验证是否被服务器信任，不知道具体怎么做的，稍后看吧.