---
layout: post
title: "AFNetworking、多线程回调对应"
date: 2016-01-02 22:37:59 +0800
comments: true
categories: 
---

###腾讯面试问的问题就是尖锐啊，AFNetworking我大部分实现都看过了，唯独在多个子线程回调是如何对应上却很少花心思去看，那天的问题没有回答出来。个人觉得这个问题，真的有必要弄懂...

****

###AFN分iOS7以下和iOS以上两个版本的实现

- iOS7及以下，使用NSOperation + NSURLConnection + 单例NSThread + NSRunLoop 实现

- iOS8及以上，使用 NSURLSession 直接完成

从结构上看，iOS8推出的NSURLSession强大太多了，所以上面的问题也得分两个版本去看。

****

###iOS7及以下，多个子线程回调的对应关系

####问题就是单例NSThread如何处理并发的NSOperation，以及回调对应的NSOperation？

接下来就是查看代码了去看怎么实现的...

####第一个问题、网络请求的并发数如何决定？

```objc
1. 取决于AFHTTPRequestOperationManager中的operationQueue属性

2. operationQueue.maxConcurrentOperationCount = 最大并发执行的NSOperation的个数;
```

####第二个问题、单例NSThread是如何正确的找到多个NSOperation线程的completion block

首先，一个网络请求的Operation的开始 >>> `-[AFURLConnectionOperation start]`

```objc
- (void)start {
    [self.lock lock];
    
    // 统一让后续的网络操作在单例子线程NSThread上执行
    if ([self isCancelled]) {
        [self performSelector:@selector(cancelConnection) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
    } else if ([self isReady]) {
        self.state = AFOperationExecutingState;

        [self performSelector:@selector(operationDidStart) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
    }
    
    
    [self.lock unlock];
}
```

然后，单例NSThread负责具体完成NSURLConnection进行网络请求

```objc
- (void)operationDidStart {

	//1. 线程加锁
    [self.lock lock];
    
    //2. 当前某一个线程进行网络操作
    if (![self isCancelled]) {
    
    	//2.1 创建NSURLConnection，回到delegate方法也是在单例NSThread上执行
        self.connection = [[NSURLConnection alloc] initWithRequest:self.request delegate:self startImmediately:NO];
		
		//2.2 获取单例子线程的Runloop事件循环
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        
        //2.3 在Runloop上注册事件
        for (NSString *runLoopMode in self.runLoopModes) {
        	// 2.3.1 事件一、NSURLConnection事件
            [self.connection scheduleInRunLoop:runLoop forMode:runLoopMode];

            //2.3.2 事件二、输出流事件
            [self.outputStream scheduleInRunLoop:runLoop forMode:runLoopMode];
        }

		//2.4 在单例子线程上执行网络具体操作
        [self.outputStream open];
        [self.connection start];
    }
    
    //3. 解除线程锁，让其他的NSOperation开始进入，执行网络操作
    [self.lock unlock];

	//4. 发送通知，告知当前的AFURLConnectionOperation对象已经开始执行
    dispatch_async(dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingOperationDidStartNotification object:self];
    });
}
```

从上面使用线程锁NSLock来互斥执行具体的每一个网络操作NSURLConnection对象创建以及执行，那么也就是说:

- 最外层的`AFHTTPRequestOperation`对象是并发在NSOperationQueue中执行的

- 但是并发的每一个`AFURLConnectionOperation`对象，在单例NSThread对象上却是`加锁互斥`执行的
	- 也就是`分先后`顺序去执行`-[NSOperation start]`方法
	- 必须前一个Operation执行完start方法之后，才会开始下一个Operation的start方法执行

那么Operation执行的顺序是分`先后`的，但是很有可能网络数据交换后的最终回调是乱序的，那么如何能找到正确的Operation了？

首先，我查看了AFURLConnectionOperation中实现的NSURLConnectionDelegate协议函数，我并没有发现使用map、dic之类找到对应Operation的代码、或者其他特殊的代码，我发现都没有。代码实现就是类似如下:

```objc
- (void)connection:(NSURLConnection __unused *)connection
didReceiveResponse:(NSURLResponse *)response
{
    self.response = response;
}
```

没有任何的lock加锁互斥、就是很简单的对当前AFURLConnectionOperation对象的属性进行赋值。

那这就很奇怪了，如果先后start的两个AFURLConnectionOperation对象，但是如果第一个是大数据下载请求，而后一个是简单的小数据get请求，那么显然应该是第二个先执行回调。一旦是这样的话，是否会出现获取到错误的Operation回调吗？

其实我想的过于复杂了，再看AFURLConnectionOperation中的NSURLConnectionDelegate协议函数的实现代码

```objc
- (void)connection:(NSURLConnection __unused *)connection
didReceiveResponse:(NSURLResponse *)response
{
    self.response = response;
}
```

回调协议函数中，直接操作的就是当前的AFURLConnectionOperation对象。那么当前AFURLConnectionOperation对象，就一定是当前网络操作回调时对应的Operation对象吗？ 答案，是的。原因很简单，如下代码

```objc
//AFURLConnectionOperation的start方法中，在创建NSURLConnection时，指定了回调的delegate对象。
//那么正是通过这个delegate指向的AFURLConnectionOperation对象，从而可以直接找到对应的回调Operation对象。

- (void)operationDidStart {
    [self.lock lock];
    
    ....
    
    // 指定了当前的NSURLConnection对象回调哪一个Operation对象
    self.connection = [[NSURLConnection alloc] initWithRequest:self.request delegate:self startImmediately:NO];
    
    ...
    
    [self.lock unlock];
    
    ....
}
```

那么举例步骤为如下：

- `Operation1对象`，先开始执行start，继而流程在单例子线程Thread上

- 然后单例子线程Thread加锁锁住，让后面的`Operation2对象`排队等待

- 创建`Operation1对象`对应的大数据下载任务的`NSURLConnection对象1`，并设置`Operation1对象`作为NSURLConnection事件的回调对象，创建完毕之后添加到runloop监听事件并开启网络请求

- 单例子线程Thread释放掉`Operation1对象`的操作锁，让后面的`Operation2对象`进入执行

- `Operation2对象`，慢开始执行start，创建对应小数据get请求的`NSURLConnection对象2`，并设置`Operation2对象`作为NSURLConnection事件的回调对象，创建完毕之后添加到runloop监听事件并开启网络请求

- NSURLConnection对象1 与 NSURLConnection对象2，分先后顺序在单例NSThread上开始执行，并由单例线程的runloop接收NSURLConnection事件源

- 肯定是 NSURLConnection对象2 小数据请求操作最先完成

- 那么此时单例子线程的runloop接收到`NSURLConnection对象2`对应的Connection事件源 

- **继而得到 NSURLConnection对象2.delegate 得到回到哪一个Operation2对象**

所以不管Operation完成的顺序如何，最终都通过设置给NSURLConnection对象.delegate来找到最终要回调的Operation对象

其实在单例子线程进行Lock加锁互斥，让并发执行的NSOperation对象一次排队进入`operationDidStart方法`的目的:

> 为了让当前的NSOperation对象 与 NSURLConnection对象能够保持正确的一对一的关系。如果不加锁互斥NSOperation对象执行，就可能出现乱序，导致NSOperation对象 与 NSURLConnection对象不对应造成最终的网络回调错误。

****

###iOS8及以上，多个子线程回调的对应关系

- SessionManager 内部写了一个AFURLSessionManagerTaskDelegate类，该类对象用来保存`某一次网络操作`（某一个NSURLSessionTask对象）请求后的网络数据

- SessionManager 对象内部又使用了一个NSMutableDictionary对象，来保存 `AFURLSessionManagerTaskDelegate对象` 与 `NSURLSessionTask对象`之间的对应关系

- 通过 `NSURLSession对象.delegate` 来指定统一回调 对象为 >>> `SessionManager 单例对象`

- 然后由SessionManager单例对象，根据当前完成的`NSURLSessionTask对象`找到对应的`AFURLSessionManagerTaskDelegate对象`来保存此次网络请求数据

****