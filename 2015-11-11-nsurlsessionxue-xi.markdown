---
layout: post
title: "NSURLSession学习"
date: 2015-11-11 02:45:09 +0800
comments: true
categories: 
---

###AFNetworking这几天从2.x迁移到了3.x，使用的一些Api发生了变化。而我App代码里面都是使用的`AFHttpRequestOperationManager`，所以必须都修改成`AFHttpSessionManager`相关Api。

- 注意 iOS SDK支持

	- iOS7+ ，才支持 NSURLSession.

	- iOS7- ，还是只能使用 NSURLConnection.

***

###首先总结下iOS完成网络操作的相关Api

- NSURLRequest、NSMutableURLRequest: 
	- 针对某个NSURL、请求方法、请求参数
	- 指向某一个服务器Host下某一个path路径.

- NSURLResponse: 
	- 用来`描述`网络请求响应response data
	- 如: MIME类型、期望的Content-Length、编码格式、响应的URL..等描述此次响应的信息
	- `只存储响应的用于描述的元数据，而不存储响应数据本身`

- NSURLCredential:
	- 封装了由`认证信息`和`持久化行为`组成的`证书`

- NSURLProtectionSpace:
	- 表示`需要`特定证书的`区域`
	- 一个`保护区域`可以 限制到 `单独的URL`

- NSURLCredientialStorage
	- 一般是一个共享实例
	- 用于管理 `存储` NSURLCredential 证书对象
	- 以及完成 `NSURLCredential对象`与`NSURLProductionSpace对象`的`映射`

- NSURLAuthenticationChallenge:
	- 封装了 `认证` 一个请求的 `NSURLProtocol实现` 所需要的所有信息
		- 建议的证书
		- 保护空间
		- 错误信息
		- 认证结果响应
		- 认证尝试次数
		- 等等...
	- 发送认证请求对象，必须实现`NSURLAuthenticationChallengeSender 协议`
	- 主要用于 `NSURLProtocol的子类` 来告诉 URL加载系统 需要 `认证`

- NSURLProtocol
	- 是一个 `抽象类`，暂时不知道这个类做神马的，后面再学习吧..

- NSURLCache
	- iOS系统通过默认的NSURLCache完成对网络请求数据的缓存
	- 但是我们可以重写这个类两个方法，完成自己定制的网络数据缓存
		- 发起请求之前拦截掉，可以进行查询缓存response data
		- 发起请求之后也会拦截，可以将获取到的response data缓存到磁盘或内存

- NSCache
	- 它使用键值对存储`key-value`，这一点类似于`NSDictionary`类
	- 常用使用缓存来`临时` 存储 `短时间`使用但`创建昂贵`的对象
	- 并且当系统遇到内存警告时，会自动释放掉缓存的内存对象
	- 存储在NSCache对象中的对象，必须实现`NSDiscardableContent协议`
	- NSCache是`线程安全`的，我们可以在不同的线程中添加、删除和查询缓存中的对象，而不需要锁定缓存区域
	- 并且提供对象存储、删除时的 回调处理

- NSHTTPCookieStorage

- NSURLCredentialStorage

- NSURLConnection （如今苹果使用NSURLSession体系来替代）

***

###从AFN在iOS7以下使用`NSURLConnection`可以看出一些总结:

- 使用NSURLConenction进行请求，没有明显的区分为如下类型的任务，都是统一使用NSURLConnection进行网络数据传输.
	- 小数据任务（GET）
	- 上传数据任务 
	- 下载数据任务 

- NSURLConnection主要的回调方法协议声明

	- NSURLConnectionDelegate，主要处理:
		- 请求错误（`请求结束执行方式一`）
		- web服务挑战
	- NSURLConnectionDataDelegate，主要处理:
		- 请求 重定向
		- 接收到 响应描述
		- 接收到 响应数据
		- 上传数据
		- 缓存 响应描述
		- 网络请求结束（`请求结束执行方式二`）
	- NSURLConnectionDownloadDelegate，主要完成下载
		- `一次性`下载
		- `断点下载` resume download，需要在响应头设置每一次的下载地址
		- 下载结束
		- 缺点: 需要自己手动的将接收到的NSData分段写入磁盘文件


- NSURLConnection 只能够通过 `NSURLRequest` 发起请求

- 所有网络请求操作都会使用同一个`NSURLConnection`配置，无法做到不同配置
	- 缓存策略
	- 通信协议
	- cookie
	- 证书政策（credential policies）

- 要自己手动创建`一个子线程`，然后在其runloop去schele调度`NSURLConnection实例`进行网络请求，否则会造成`主线程卡顿`

- NSURLConnection发起的 网络请求操作，是不能够被 `取消后继续恢复`执行，只能取消，不能恢复执行.

***

###NSURLSession的一些总结:

- NSURLSession主要的回调方法协议声明

	- NSURLSessionDelegate、管理`session级别`的回调，主要负责:
		- Session实例失效
		- web服务挑战认证（`session统一管理web服务挑战`）
		- 后台`上传/下载`处理

	- NSURLSessionTaskDelegate、管理`task级别`的回调，主要负责:
		- 请求重定向
		- web服务挑战认证（`每个Task也可以单独设置web服务挑战`）
			- 注意: 如果Task也接受挑战，Session也接受挑战，那么最终由Session接受挑战.
		- 请求结束执行（可能是错误，也可能是正常结束）将NSURLConnection区分错误和正常结束的方法`合二为一`。在其回调函数中判断传入error是否为nil:
			- 如果是nil，正常结束
			- 如果非nil，发生错误

		- NSURLSessionDelegate 又有三个子协议
			- NSURLSessionDataDelegate
			- NSURLSessionDownloadDelegate
			- NSURLSessionStreamDelegate

- NSURLSession可以只使用`NSURL`发起请求，可以不创建`NSURLRequest`

- 每一个NSURLSession实例，可以配置一个单独的`NSURLSessionConfiguration`实例。来为每一个Session配置不同的设置:
	- 请求超时
	- 请求缓存策略
	- 验证证书
	- 等等

- NSURLSession创建出来的所有的Task，默认都是运行在`新子线程`上

- NSURLSession的下载和上传代理方法中，已经回传了能够快速计算得到`进度`的数值

- NSURLSession的下载代理中，已经有接受完下载数据的`临时文件`，我们只需要将这个临时文件copy到新的文件目录下即可

- 由NSURLSession创建出的所有的网络请求Task，都可以被
	- 取消执行（都可以...）
	- 暂停执行（取消执行后可以继续恢复执行）
	- 恢复执行

- 使用NSURLSession创建出来的Task，不会直接被resume执行
	- 只是将该 task 对象先返回
	- 允许我们进一步的配置之后
	- 让开发者自己选择进行resume执行

****

###首先看下`NSURLSession`整体Api的结构图

![](http://i5.tietuku.com/5ea7dccedc6eae2c.png)

###所有的具体网络数据操作封装到了单独的Task类，看下`NSURLSessionTask`的 继承结构图


![](http://i3.tietuku.com/bbb357f36fff3543.png)

- `NSURLSession`相比做出的优化，是将原本所有混在`NSURLConnection`里面做的所有事情细化成如下多个类
	
	- `NSURLSession`，抽象出一个客户端与服务端的会话任务，App可以同时存在`多个会话任务 （对不同Server Host的网络请求控制块）`

	- `NSURLSessionConfiguration`，给所在会话提供一些配置信息（缓存、Cookie、证书...)

	- `NSURLSessionTask`，完成具体网络请求
		- 子类出了三个Task类，分别完成不同的网络操作
		- Task支持任务的`暂停、取消、恢复`
		- Task默认任务运行在其他`非主线程`之上
		- 所有task一开始都处于`挂起`状态，必须手动执行`resume`

	- NSURLSessionDownloadTask 继承自 NSURLSessinDataTask
		- GET数据请求: 可以看做是一种 小数据下载 任务
		- 下载请求: 可以看做是一种 大数据（文件）任务
	
*****

###图示一下NSURLSession 与 NSURLConnection 代码结构

![](http://i1.tietuku.com/411c8927818fd677.png)


****

###看下Api的基本使用

```
//1. 组装成一个request（虽然可以直接使用URL字符串，但是还是建议组成成request，因为有更多的设置）
NSURLRequest *request=[NSURLRequest requestWithURL:url];

//2. 获取全局会话session
NSURLSession *session=[NSURLSession sharedSession];

//3. 使用session创建Task，并传入请求结束后的回调Block
NSURLSessionDataTask *dataTask=[session dataTaskWithRequest:request completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) 
{
    if (!error) {
        NSString *dataStr=[[NSString alloc]initWithData:data encoding:NSUTF8StringEncoding];
        NSLog(@"%@",dataStr);
    }else{
        NSLog(@"error is :%@",error.localizedDescription);
    }
}];

//4. 执行Task（可能是启动执行，也可能是恢复执行）
[dataTask resume];
```

***

###NSURLSession的工作模式、以及各种参数配置的载体 -- `NSURLSessionConfiguration`

- 默认模式: 工作模式与之前NSURLConnection类似
	- 使用 `磁盘文件` 缓存网络请求数据、Cookies
	- 使用 `keyChain` 保存证书进行授权认证

	```
	[NSURLSessionConfiguration defaultSessionConfiguration]:
	```

- 临时模式: 
	- 使用 `内存` 缓存所有的数据，`保证数据的安全性`，当进程结束掉之后所有缓存数据会被丢失
		- 网络请求数据
		- 证书Cookies

	```
	[NSURLSessionConfiguration ephemeralSessionConfiguration]:
	```

- 后台任务模式: 让App运行在后台时，也能进行`网络操作`
	- 只能进行 `上传` 和 `下载` 的网络操作
	- 创建时需要传入的一个唯一标识字符串，区分每一个后台session对象
	- 使用`后台session`创建出的 任务 在 `外部进程` 中运行，而并`不是`在当前`App进程`中执行
	- 所以即使`App进程 挂起、崩溃、被杀死`，后台任务 依然会执行

	```
	[NSURLSessionConfiguration backgroundSessionConfiguration:@"session的唯一标识"]
	```

***


###NSURLSessionConfiguration 一些属性

- 来唯一标识一个后台session

```
@property (nullable, readonly, copy) NSString *identifier;
```

- 网络请求缓存策略，可以注册自己的`NSURLCache实例`

```
@property NSURLRequestCachePolicy requestCachePolicy
```

- 网络请求超时

```
@property NSTimeInterval timeoutIntervalForRequest;
```

- 设置一个网络请求操作的最大持续时间

```
@property NSTimeInterval timeoutIntervalForResource;
```

- 是否允许使用蜂窝连接

```
@property BOOL allowsCellularAccess;
```

- 是否由系统自己选择最佳的网络连接配置

```
@property (getter=isDiscretionary) BOOL discretionary NS_AVAILABLE(10_10, 7_0);
```

- 指定一个共享容器的标示符,然后我们就可以通过该标示符获取到共享容器

```
@property (nullable, copy) NSString *sharedContainerIdentifier NS_AVAILABLE(10_10, 8_0);
```

- 指定 `后台 session`是否可以从后台启动

```
@property BOOL sessionSendsLaunchEvents NS_AVAILABLE(NA, 7_0);
```

- 确定 session 是否支持 `SSL 协议`

```
@property SSLProtocol TLSMinimumSupportedProtocol;
@property SSLProtocol TLSMaximumSupportedProtocol;
```

- 用于开启 HTTP `管线化`（HTTP pipelining），这可以显着降低请求的加载时间，但是由于`没有被服务器广泛支持`，默认是禁用的

```
@property BOOL HTTPShouldUsePipelining;
```

- 是否由系统存储cookies，使用磁盘文件形式

```
@property BOOL HTTPShouldSetCookies;
```

- 设置如何缓存cookies

```
@property NSHTTPCookieAcceptPolicy HTTPCookieAcceptPolicy;
```

```
typedef NS_ENUM(NSUInteger, NSHTTPCookieAcceptPolicy) {
    NSHTTPCookieAcceptPolicyAlways,
    NSHTTPCookieAcceptPolicyNever,
    NSHTTPCookieAcceptPolicyOnlyFromMainDocumentDomain
};
```

- 指定了一组默认的可以设置`出站请求（访问其他服务器Host）`（outbound request）的数据头。这对于 `跨 session 共享信息`时，如内容类型、语言、用户代理、身份认证，是很有用的。

> 解决页面需要认证的情况，比如登录TP-link的路由器，提示你需要用户名和密码

```
@property (nullable, copy) NSDictionary *HTTPAdditionalHeaders;
```

```
例子: 

NSString *userPasswordString = [NSString stringWithFormat:@"%@:%@", user, password];

NSData * userPasswordData = [userPasswordString dataUsingEncoding:NSUTF8StringEncoding];

NSString *base64EncodedCredential = [userPasswordData base64EncodedStringWithOptions:0];

NSString *authString = [NSString stringWithFormat:@"Basic %@", base64EncodedCredential];

NSString *userAgentString = @"AppName/com.example.app (iPhone 5s; iOS 7.0.2; Scale/2.0)";

某个configuration.HTTPAdditionalHeaders = @{
												@"Accept": 												@"application/json",
												@"Accept-Language": @"en",
												@"Authorization": authString,
												@"User-Agent": userAgentString
												};
```

- 设置连接到某一个服务器Host主机的 `连接数`

```
@property NSInteger HTTPMaximumConnectionsPerHost;
```

- session使用这个类实例进行存储cookie，默认使用 `[NSHTTPCookieStorage sharedHTTPCookieStorage]`获取

```
@property (nullable, retain) NSHTTPCookieStorage *HTTPCookieStorage;
```

- 存储了 session 所使用的 `证书`，默认使用 `[NSURLCredentialStorage sharedCredentialStorage]`获取

```
@property (nullable, retain) NSURLCredentialStorage *URLCredentialStorage;
```

- 持有一个NSURLCache用来缓存网络请求响应数据，默认使用 `[NSURLCache sharedURLCache]`获取

```
@property (nullable, retain) NSURLCache *URLCache;
```

- 使用后台空闲时间创建TCP协议的Socket连接，系统会根据这个属性BOOL值，决定当App进入后台时，是否 `长时间打开` 或 `延迟恢复` socket连接.

```
@property BOOL shouldUseExtendedBackgroundIdleMode NS_AVAILABLE(10_11, 9_0);
```

- 用来配置特定某个 session 所使用的 `自定义网络通信协议`（该协议是 NSURLProtocol 的子类）的数组

```
@property (nullable, copy) NSArray<Class> *protocolClasses;
```

****

###NSURLSessionTask的三个子类Task

- NSURLSessionDataTask 短时间即可响应的数据请求（也称 小数据下载任务）
		
- NSURLSessionUploadTask 上传操作，`只支持POST`
		
- NSURLSessionDownloadTask 下载操作（大文件数据下载）

***

###NSURLSession存在一个强引用的问题

- NSURLSession的类方法传入一个实现NSURLSessionDelegate协议的对象

```
+ (NSURLSession *)sessionWithConfiguration:(NSURLSessionConfiguration *)configuration delegate:(nullable id <NSURLSessionDelegate>)delegate delegateQueue:(nullable NSOperationQueue *)queue;
```

- 然而在NSURLSession中，将传入的代理对象使用`retain`，导致传入的代理对象无法释放
	- 由因为代理对象无法释放，那么Session对象也无法释放

```
@property (nullable, readonly, retain) id <NSURLSessionDelegate> delegate;
```

- 图示强引用问题

	![](http://i12.tietuku.com/1cc67c1a12dd3f4a.jpg)
	
	- 不会释放的原因是，我们自己的SessionManager对象，有两个强引用的指针指向

- 一般正常的两个对象之间的引用必须是`一强一弱`，就不会引起循环引用

	![](http://i12.tietuku.com/c5a92618adca391e.jpg)

	- 一旦A对象释放掉B对象，那么B对象就会被释放，因为B对象只有一个强引用指针
	- 一旦B对象被释放，那么C对象也会被释放，因为也只有一个强引用指针

- 解决`互相强引用`的方法，必须是图示中，我们自己的SessionManager必须让NSURLSession对象对自己的delegate做一次释放操作

	- 比如`AFURLSessionManager`中提供了一个`取消执行`或`结束执行`session的入口函数

	```
	- (void)invalidateSessionCancelingTasks:(BOOL)cancelPendingTasks {
	    dispatch_async(dispatch_get_main_queue(), ^{
	        if (cancelPendingTasks) {
	            [self.session invalidateAndCancel];
	        } else {
	            [self.session finishTasksAndInvalidate];
	        }
	    });
	}
	```

- 所以说执行如下两个方法让session停止执行，继而session才会对delegate进行release

```
//1. 在网络操作结束、不需要再使用的session的时候，结束或取消session
[self.session invalidateAndCancel];
或
[self.session finishTasksAndInvalidate];

//2. 
self.session = nil;

//3. 最终代理对象的retainCount就会恢复正常
```

- AFURLSessionManager可以这么使用

```
//1. 
[self.sessionManager invalidateSessionCancelingTasks];

//2.
self.sessionManager = nil;
```

####如上两部就是让NSURLSession对象，对自己内部的delegate做一次释放，来减少一个指向AFURLSessionManager对象的强指针

***

###DataTask回调执行时，分别对不同的URLSession进行数据存储

```
- (void)URLSession:(NSURLSession *)session 
          dataTask:(NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data
{
	if (session == self.defaultSession) {
		//...
	} else if (session == self.ephemeralSession) {
		//...	
	} else if (session == self.customesession1) {
		//...
	} else if (session == self.customesession2) {
		//...
	}
}
```

