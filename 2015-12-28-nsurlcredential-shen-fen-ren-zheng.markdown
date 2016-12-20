---
layout: post
title: "NSURLCredential 身份认证"
date: 2015-12-28 15:27:21 +0800
comments: true
categories: 
---

###当时也`https://`连接进行客户端与服务端的数据传输时，web服务器接收到客户端请求时可能需要先`验证客户端`是否正常用户，再决定是否返回该接口的真实数据

```
比如: 
web服务的某一些URL的访问，
需要提供 `身份认证、一定权限`才能通过，
否则直接返回 `401系统错误`，

称这种情况为，服务端 要求 客户端 接收挑战.
```

认证过程

- 1）web服务接收到来自客户端的请求

- 2）web服务并不直接返回数据，而是要求客户端提供认证信息，也就是说`挑战是服务器端向客户端发起的`
	- 要客户端提供用户名与密码接受挑战 >>> NSInternetPassword
	- 要求客户端提供客户端证书 >>> NSClientCertificate
	- 要求客户端信任该服务器（会弹出一个提示框，要求客户端信任该Https服务器的连接） >>> NSServerTrust

- 3）客户端回调执行，接收到需要提供认证信息，然后提供认证信息，并再次发送给 web服务

- 4）web服务验证认证信息
	- 认证成功，将最终的`结果数据`发送刚给客户端
	- 认证失败，`错误结束`此次请求，返回错误码 `401`


****

### web服务需要验证客户端网络请求


- NSURLConnectionDelegate 提供的接收挑战

```
- (void)connection:(NSURLConnection *)connection willSendRequestForAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge;
```

- NSURLSessionDelegate 提供的接收挑战，`会话级别`

```objc
- (void)URLSession:(NSURLSession *)session didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
                                             completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential * __nullable credential))completionHandler;
```


- NSURLSessionTaskDelegate 提供的接收挑战，`任务级别`

```objc
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
                            didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge 
                              completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential * __nullable credential))completionHandler;
```

- 如果同时实现 会话级别 与 任务级别，那么会由 `会话级别` 处理.

****

###如上回调函数都是接收到一个 `NSURLAuthenticationChallenge`，这个类是认证挑战类，也就是web服务 要求 客户端 进行挑战，最终生成一个 `挑战凭证 NSURLCredential实例`


####那么如上三种回调函数中，也就是说接收挑战，要做的几件事情如下:

我自己实现的形式

```objc
- (void)URLSession:(NSURLSession *)session didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential * _Nullable credential))completionHandler
{
    //1. 判断服务器返回的证书是否是服务器信任的
    if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust])
    {   
        //2. 根据服务器返回的受保护空间中的信任，创建一个挑战凭证
        NSURLCredential *credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
        
        //3. 完成挑战
        completionHandler(NSURLSessionAuthChallengeUseCredential , credential);
    }
    
}
```

摘录自AFNetworking中AFURLSessionManager的接收挑战实现

```objc
- (void)URLSession:(NSURLSession *)session
didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
 completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential *credential))completionHandler
{
    NSURLSessionAuthChallengeDisposition disposition = NSURLSessionAuthChallengePerformDefaultHandling;
    __block NSURLCredential *credential = nil;

    if (self.sessionDidReceiveAuthenticationChallenge) {
	    //回调用户自己处理挑战
    
        disposition = self.sessionDidReceiveAuthenticationChallenge(session, challenge, &credential);
    } else {
    
    	//AFN框架自己处理挑战
        if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
            if ([self.securityPolicy evaluateServerTrust:challenge.protectionSpace.serverTrust forDomain:challenge.protectionSpace.host]) {
                credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
                if (credential) {
                    disposition = NSURLSessionAuthChallengeUseCredential;
                } else {
                    disposition = NSURLSessionAuthChallengePerformDefaultHandling;
                }
            } else {
                disposition = NSURLSessionAuthChallengeCancelAuthenticationChallenge;
            }
        } else {
            disposition = NSURLSessionAuthChallengePerformDefaultHandling;
        }
    }

    if (completionHandler) {
        completionHandler(disposition, credential);
    }
}
```

***

###创建 NSURLCredential实例 的几种方法

- 通过 `账号和密码`，适用于返回 `401错误`的挑战

```objc
@interface NSURLCredential(NSInternetPassword)

- (instancetype)initWithUser:(NSString *)user password:(NSString *)password persistence:(NSURLCredentialPersistence)persistence;

+ (NSURLCredential *)credentialWithUser:(NSString *)user password:(NSString *)password persistence:(NSURLCredentialPersistence)persistence;

@property (nullable, readonly, copy) NSString *user;

@property (nullable, readonly, copy) NSString *password;

@property (readonly) BOOL hasPassword;

@end
```

- 通过 `证书` （keyChain中存储）

```objc
@interface NSURLCredential(NSClientCertificate)

- (instancetype)initWithIdentity:(SecIdentityRef)identity certificates:(nullable NSArray *)certArray persistence:(NSURLCredentialPersistence)persistence NS_AVAILABLE(10_6, 3_0);

+ (NSURLCredential *)credentialWithIdentity:(SecIdentityRef)identity certificates:(nullable NSArray *)certArray persistence:(NSURLCredentialPersistence)persistence NS_AVAILABLE(10_6, 3_0);

@property (nullable, readonly) SecIdentityRef identity;

@property (readonly, copy) NSArray *certificates NS_AVAILABLE(10_6, 3_0);

@end
```

- 通过 `信任`

```objc
@interface NSURLCredential(NSServerTrust)

- (instancetype)initWithTrust:(SecTrustRef)trust NS_AVAILABLE(10_6, 3_0);

+ (NSURLCredential *)credentialForTrust:(SecTrustRef)trust NS_AVAILABLE(10_6, 3_0);

@end
```

```
SecTrustRef结构体: 用来描述，信任某个证书，完成一件事情
```

***

###NSURLCredentialPersistence ，区分`凭证NSURLCredential实例`的 保存策略

```objc
NSURLCredentialPersistenceNone, //不保存，只请求一次。
NSURLCredentialPersistenceForSession, //只在本次会话中有效
NSURLCredentialPersistencePermanent //永久有效,保存在钥匙串中，其他也有效
```

****

###有如下这几种挑战的形式

- NSURLAuthenticationMethodHTTPBasic

```
1. 需要用户名和密码

2. 使用 credentialWithUser:password:persistence:方法创建凭证
```

- NSURLAuthenticationMethodHTTPDigest

```
1. 也是需要用户名和密码。（digest（文摘）是自动生成

2. 使用 credentialWithUser:password:persistence:方法创建凭证
```

- NSURLAuthenticationMethodServerTrust

```
1. 需要一个 Trust 信任，来自受保护将空间

2. 使credentialForTrust:方法创建凭证
```

###创建了NSURLCredentical对象之后

- NSURLSession

```
使用提供的控制完成的handler block，
把这个对象传递给authentication challenge‘s的发送者。
```

- NSURLConnection和NSURLDownload

```
使用useCredential:forAuthenticationChallenge:方法把对象传递给authentication challenge’s sender。
```

****

###关于iOS9要求https协议通信

- 服务器使用的 `TLS版本` 必须大于等于 `1.2`