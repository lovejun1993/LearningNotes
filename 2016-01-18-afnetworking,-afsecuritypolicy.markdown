---
layout: post
title: "AFNetworking、AFSecurityPolicy"
date: 2016-01-18 14:05:21 +0800
comments: true
categories: 
---

###AFSecurityPolicy类封装SSL证书验证的相关逻辑，因为NSURLConnection并未提供SSL证书验证。AFSecurityPolicy完成:

- 与 客户端本地系统中的受信任证书颁发机构CA列表 进行验证
- 对比 本地存储的证书 进行验证（`SSL Pinning 本地证书校验`）

***

###使用 `SSL Pinning 本地证书校验`，只需要让AFSecurityPolicy实例读取本地证书cer文件即可.

```objc
//把服务端证书(需要转换成cer格式)放到APP项目资源里，
//AFSecurityPolicy会自动寻找根目录下所有cer文件

//1. 创建一个SSL Pinning校验本地证书的 AFSecurityPolicy实例
AFSecurityPolicy *securityPolicy = [AFSecurityPolicy policyWithPinningMode:AFSSLPinningModePublicKey];

//2. 是否允许 不受信任的证书通过验证，默认为NO
//如果是使用自制的SSL证书，就必须设置为YES
securityPolicy.allowInvalidCertificates = YES;

//3. 将AFSecurityPolicy实例设置给 manager
[AFHTTPRequestOperationManager manager].securityPolicy = securityPolicy;

//4. 然后发起网络请求
[manager GET:@"https://example.com/" parameters:nil success:^(AFHTTPRequestOperation *operation, id responseObject) {
} failure:^(AFHTTPRequestOperation *operation, NSError *error) {
}];
```

***

###AFSecurityPolicy.h定义了SSL Pinning的三种模式

```objc
typedef NS_ENUM(NSUInteger, AFSSLPinningMode) {
    AFSSLPinningModeNone,
    AFSSLPinningModePublicKey,
    AFSSLPinningModeCertificate,
};
```

###AFSSLPinningModeNone 验证模式

- 这个模式不做本地证书验证（不做SSL Pinning操作）
- 直接从客户端系统种的 受信任颁发机构CA列表中 去验证
- 此模式不适用于 自颁发证书，不会通过验证的

###AFSSLPinningModePublicKey 验证模式

- 客户端 需要一份证书文件的拷贝
- 验证时只验证证书里的 `公钥`，不验证证书的`有效期`等信息
- 即使伪造证书的 公钥，也不能解密 传输的数据，必须要 私钥

###AFSSLPinningModeCertificate 验证模式

- 客户端 需要一份证书文件的拷贝
- 第一步验证、先验证证书的 `域名/有效期等信息`
- 第二步验证、对比 服务端返回的证书 跟 客户端存储的证书 是否一致


***

###先看下AFSecurityPolicy的一些属性


####记录SSL证书验证的模式

```objc
@property (readonly, nonatomic, assign) AFSSLPinningMode SSLPinningMode;
```

####记录 每一个域名Host 对应的 本地SSL证书

```objc
@property (nonatomic, strong, nullable) NSArray *pinnedCertificates;
```

####是否允许验证SSL证书

```objc
@property (nonatomic, assign) BOOL allowInvalidCertificates;
```
此属性值默认是NO

####是否验证证书中的CN字段

```objc
@property (nonatomic, assign) BOOL validatesDomainName;
```

***

###然后就是提高两种获取AFSecurityPolicy实例的方法


####使用客户端系统的证书颁发机构CA列表验证服务器发过来的证书

```objc
+ (instancetype)defaultPolicy;
```

####使用本地证书验证服务器发过来的证书

```objc
+ (instancetype)policyWithPinningMode:(AFSSLPinningMode)pinningMode;
```

***

###对指定domain域名使用证书SecTrustRef验证

```objc
- (BOOL)evaluateServerTrust:(SecTrustRef)serverTrust
                  forDomain:(nullable NSString *)domain;
```

****

###AFSecurityPolicy使用一个单例数组保存工程内的所有SSL证书的.cer格式文件

```objc
@implementation AFSecurityPolicy

+ (NSArray *)defaultPinnedCertificates {
    
    static NSArray *_defaultPinnedCertificates = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        
        //工程的包
        NSBundle *bundle = [NSBundle bundleForClass:[self class]];
        
        //读取工程内 `所有的 xxx.cer` 文件
        NSArray *paths = [bundle pathsForResourcesOfType:@"cer" inDirectory:@"."];
        
        //读取所有的cer文件，并使用数组保存
        NSMutableArray *certificates = [NSMutableArray arrayWithCapacity:[paths count]];
        for (NSString *path in paths) {
            NSData *certificateData = [NSData dataWithContentsOfFile:path];
            [certificates addObject:certificateData];
        }

        _defaultPinnedCertificates = [[NSArray alloc] initWithArray:certificates];
    });

    return _defaultPinnedCertificates;
}

@end
```


###AFSecurityPolicy获取default实例，使用`客户端本地的受信任证书颁发机构CA列表`来验证服务器发过来的证书

```objc
+ (instancetype)defaultPolicy {
    
    //1. 创建一个新的AFSecurityPolicy实例
    AFSecurityPolicy *securityPolicy = [[self alloc] init];
    
    //2. SSL证书验证模式为: 客户端本地的受信任证书颁发机构CA列表
    securityPolicy.SSLPinningMode = AFSSLPinningModeNone;

    return securityPolicy;
}
```

###使用指定的SSL证书验证模式创建AFSecurityPolicy实例

```objc
+ (instancetype)policyWithPinningMode:(AFSSLPinningMode)pinningMode {
    
    //1. 创建一个新的AFSecurityPolicy实例
    AFSecurityPolicy *securityPolicy = [[self alloc] init];
    
    //2. 传入指定的SSL证书验证模式
    securityPolicy.SSLPinningMode = pinningMode;

    //3. 加载程序包中的 .cer证书文件 到内存数组保存
    [securityPolicy setPinnedCertificates:[self defaultPinnedCertificates]];

    return securityPolicy;
}
```

如下方法是上面调用读取程序包内的所有.cer证书文件

```objc
- (void)setPinnedCertificates:(NSArray *)pinnedCertificates {
    
    //1. 使用NSOrderedSet去除重复的对象
    _pinnedCertificates = [[NSOrderedSet orderedSetWithArray:pinnedCertificates] array];

    //2. 如果有证书，就取出证书中的公钥
    if (self.pinnedCertificates)
    {
        //2.1 证书总数长度的数组，用来保存每一个证书的公钥
        NSMutableArray *mutablePinnedPublicKeys = [NSMutableArray arrayWithCapacity:[self.pinnedCertificates count]];
        
        //2.2 遍历证书数组中的每一个证书（是一个NSData对象）
        for (NSData *certificate in self.pinnedCertificates) {
            
            //2.2.1 读取证书的公钥
            //为了AFSSLPinningModePublicKey方式的验证服务器发过来的证书
            id publicKey = AFPublicKeyForCertificate(certificate);
            
            //2.2.2 如果证书存在公钥就保存到公钥数组
            if (!publicKey) {
                continue;
            }
            [mutablePinnedPublicKeys addObject:publicKey];
        }
        self.pinnedPublicKeys = [NSArray arrayWithArray:mutablePinnedPublicKeys];
    } else {
        self.pinnedPublicKeys = nil;
    }
}
```

如上有一个 NSOrderedSet 的使用，具体使用参加 NSOrderedSet文章.


###看看从证书读取公钥的代码实现

```objc
/**
 *  读取从程序包种的载入的cer文件的NSData数据
 *  解析为证书，再读取证书中的公钥
 */
static id AFPublicKeyForCertificate(NSData *certificate) {
    
    // 存放读取的公钥
    id allowedPublicKey = nil;
    
    // 解析NSData对应的证书实例
    SecCertificateRef allowedCertificate;
    
    // 证书实例数组，就一个元素，存放上面的证书实例
    SecCertificateRef allowedCertificates[1];
    
    // 临时数组
    CFArrayRef tempCertificates = nil;
    
    // 安全策略
    SecPolicyRef policy = nil;
    
    // 信任
    SecTrustRef allowedTrust = nil;
    
    // 信任结果
    SecTrustResultType result;

    //1. 使用NSData --> CFDataRef之后，创建证书实例
    allowedCertificate = SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificate);
    
    //2. 如果allowedCertificate为空，流程走到 _out处 （c语言中的goto:）
    __Require_Quiet(allowedCertificate != NULL, _out);

    //3. 从证书中取出 允许的信任
    
    //3.1 使用数组保存解析的证书实例
    allowedCertificates[0] = allowedCertificate;
    
    //3.2 c数组转换为CFArrayRef数组
    tempCertificates = CFArrayCreate(NULL, (const void **)allowedCertificates, 1, NULL);

    //3.3 国际电信证书标准格式的安全策略
    policy = SecPolicyCreateBasicX509();
    
    //3.4 使用如上三个数据，获取到信任 &allowedTrust
    __Require_noErr_Quiet(SecTrustCreateWithCertificates(tempCertificates, policy, &allowedTrust), _out);
    
    //3.5 信任评估的结果，如果出错也走到 _out处
    __Require_noErr_Quiet(SecTrustEvaluate(allowedTrust, &result), _out);

    //4. 从得到的允许的信任中得到公钥
    allowedPublicKey = (__bridge_transfer id)SecTrustCopyPublicKey(allowedTrust);

    //5. _out流程主要是释放内存
_out:
    if (allowedTrust) {
        CFRelease(allowedTrust);
    }

    if (policy) {
        CFRelease(policy);
    }

    if (tempCertificates) {
        CFRelease(tempCertificates);
    }

    if (allowedCertificate) {
        CFRelease(allowedCertificate);
    }

    //6. 返回解析得到的证书公钥
    return allowedPublicKey;
}
```


主要分三步:

- 从NSData解析出 证书SecCertificateRef实例
- 从SecCertificateRef实例取出 信任SecTrustRef实例
- 从信任SecTrustRef实例取出 公钥SecKeyRef实例

***

###验证对应域名的服务器信任中的证书是否合法，有三种验证方式

- AFSSLPinningModeNone
	- 适用于从一些公认受信任证书颁发机构CA获取的证书
	- 使用客户端本地系统种的 被信任证书颁发结构CA的根证书 验证

- AFSSLPinningModePublicKey
	- 本地证书验证一，适用于自制证书，以及App移动应用
	- `验证复杂性低于下面的验证二`
	- 使用本地客户端证书比较
	- 对比服务器证书的公钥 与 本地证书的公钥 是否一致

- AFSSLPinningModeCertificate
	- 本地证书验证二，适用于自制证书，以及App移动应用
	- 使用本地客户端证书比较
	- 首先判断服务器证书是否合法
	- 再对比本地证书内容是否一致


```objc
- (BOOL)evaluateServerTrust:(SecTrustRef)serverTrust
                  forDomain:(NSString *)domain
{
    //1. 如果需要验证自制SSL证书，必须使用本地存放SSL证书来验证
    //必须使用 AFSSLPinningModePublicKey或者AFSSLPinningModeCertificate
    if (domain && \
        self.allowInvalidCertificates && \
        self.validatesDomainName && \
        (self.SSLPinningMode == AFSSLPinningModeNone || [self.pinnedCertificates count] == 0)
        )
    {
        // https://developer.apple.com/library/mac/documentation/NetworkingInternet/Conceptual/NetworkingTopics/Articles/OverridingSSLChainValidationCorrectly.html
        //  According to the docs, you should only trust your provided certs for evaluation.
        //  Pinned certificates are added to the trust. Without pinned certificates,
        //  there is nothing to evaluate against.
        //
        //  From Apple Docs:
        //          "Do not implicitly trust self-signed certificates as anchors (kSecTrustOptionImplicitAnchors).
        //           Instead, add your own (self-signed) CA certificate to the list of trusted anchors."
        NSLog(@"In order to validate a domain name for self signed certificates, you MUST use pinning.");
        return NO;
    }

    //2. 如果是要验证服务器域名，就需要添加 SSL安全策略，否则就是x509基本安全策略
    NSMutableArray *policies = [NSMutableArray array];
    
    if (self.validatesDomainName) {
        [policies addObject:(__bridge_transfer id)SecPolicyCreateSSL(true, (__bridge CFStringRef)domain)];
    } else {
        [policies addObject:(__bridge_transfer id)SecPolicyCreateBasicX509()];
    }
    
    //3. 给传入的需要验证的证书信任，添加安全策略
    SecTrustSetPolicies(serverTrust, (__bridge CFArrayRef)policies);

    //4. 判断如果是使用客户端本地受信任颁发机构CA列表验证
    if (self.SSLPinningMode == AFSSLPinningModeNone) {
        
        //情况一、使用客户端本地受信任颁发机构CA列表验证
        return self.allowInvalidCertificates || AFServerTrustIsValid(serverTrust);
        
    } else if (!AFServerTrustIsValid(serverTrust) && !self.allowInvalidCertificates) {
        
        //情况二、AFSSLPinningModePublicKey或者AFSSLPinningModeCertificate这两种验证模式
        //传入的验证证书不被信任
        //不允许自制证书
        
        return NO;
    }

    //5. 确认传入的验证证书是被信任的之后，再使用本地证书验证传入的信任中的证书
    
    //5.1 从传入的信任中的获取证书（服务器证书）
    NSArray *serverCertificates = AFCertificateTrustChainForServerTrust(serverTrust);
    
    //5.2 本地证书验证服务器证书，必须是AFSSLPinningModeCertificate和AFSSLPinningModePublicKey
    switch (self.SSLPinningMode) {
            
        //4.2.1 不处理
        case AFSSLPinningModeNone:
        default:
            return NO;
        
        //4.2.2 使用本地证书 比较 服务器证书
        case AFSSLPinningModeCertificate: {
            
            //获取之前加载的所有本地证书
            NSMutableArray *pinnedCertificates = [NSMutableArray array];
            for (NSData *certificateData in self.pinnedCertificates) {
                [pinnedCertificates addObject:(__bridge_transfer id)SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificateData)];
            }
            
            //将本地的证书 设置为 根证书
            SecTrustSetAnchorCertificates(serverTrust, (__bridge CFArrayRef)pinnedCertificates);

            //首先判断服务器证书是否可信任
            if (!AFServerTrustIsValid(serverTrust)) {
                return NO;
            }

            //然后与当前本地的所有证书，逐个比较，看有没有一样的
            NSUInteger trustedCertificateCount = 0;
            for (NSData *trustChainCertificate in serverCertificates) {
                if ([self.pinnedCertificates containsObject:trustChainCertificate]) {
                    trustedCertificateCount++;
                }
            }
            
            //只要找到一个一样的，就表示验证通过
            return trustedCertificateCount > 0;
        }
            
        //4.2.3 比较证书的公钥是否一致
        case AFSSLPinningModePublicKey: {
            
            NSUInteger trustedPublicKeyCount = 0;
            
            //取出服务器证书的公钥
            NSArray *publicKeys = AFPublicKeyTrustChainForServerTrust(serverTrust);

            //再与本地证书的公钥进行比较
            for (id trustChainPublicKey in publicKeys) {
                for (id pinnedPublicKey in self.pinnedPublicKeys) {
                    if (AFSecKeyIsEqualToKey((__bridge SecKeyRef)trustChainPublicKey, (__bridge SecKeyRef)pinnedPublicKey)) {
                        trustedPublicKeyCount += 1;
                    }
                }
            }
            
            //只要有一个相同，就表示通过验证
            return trustedPublicKeyCount > 0;
        }
    }
    
    return NO;
}
```

会从传入的服务器信任SecTrustRef实例中，取出服务器证书，然后验证证书是否合法.


###再看AFURLConnectionOperation实现NSURLConnectionDelegate中的接受web服务挑战的函数中，对AFSecurityPolicy的使用

```objc
@implementation AFURLConnectionOperation

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
        
        //1.1.1 挑战类型是 非服务器信任，使用自己的凭证完成挑战
        //使用账号+密码创建凭证
        //使用证书创建凭证
        
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

@end
```

###AFURLSessionManager中使用

```objc
- (void)URLSession:(NSURLSession *)session
didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
 completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential *credential))completionHandler
{
    NSURLSessionAuthChallengeDisposition disposition = NSURLSessionAuthChallengePerformDefaultHandling;
    __block NSURLCredential *credential = nil;

    if (self.sessionDidReceiveAuthenticationChallenge) {
        disposition = self.sessionDidReceiveAuthenticationChallenge(session, challenge, &credential);
    } else {
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