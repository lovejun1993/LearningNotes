##之前看到关于RunLoop的一些使用

在AFNetworking源码中对runloop的使用代码:


```objc
@implementation AFURLConnectionOperation

///创建单例NSThread常驻线程对象
+ (NSThread *)networkRequestThread {
	static NSThread *_networkRequestThread = nil;
	static dispatch_once_t oncePredicate;
	dispatch_once(&oncePredicate, ^{
	    
		//1. 创建单例NSThread对象
		_networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
		
		//2. 开启线程对象
		[_networkRequestThread start];
	});

	return _networkRequestThread;
}

///NSThread入口函数
+ (void)networkRequestThreadEntryPoint:(id)__unused object {
	@autoreleasepool {
		[[NSThread currentThread] setName:@"AFNetworking"];
			
		//1. 获取当前线程的runloop
		NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
		
		//2. 给当前线程的runloop注册监听的一个基于port的事件源
		//【重要】：这个port事件其实只是为了让runloop不会退出执行
		[runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
		
		//3. 开始运行runloop
		[runLoop run];
	}
}

///NSOperation的任务函数
- (void)start {
	[self.lock lock];
	
	if ([self isCancelled]) {
	
		//1. 在单例NSThread对象上执行函数
	    [self performSelector:@selector(cancelConnection) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
	} else if ([self isReady]) {
	    self.state = AFOperationExecutingState;
		
		//1. 在单例NSThread对象上执行函数
	    [self performSelector:@selector(operationDidStart) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
	}
	
	[self.lock unlock];
}

///给此次网络请求operation创建一个新的NSURLConnection，并开始网络请求
- (void)operationDidStart {
    [self.lock lock];
    if (![self isCancelled]) {
    
    	//1. 创建NSURLConnection，并将当前operation对象设置为delegate
		self.connection = [[NSURLConnection alloc] initWithRequest:self.request delegate:self startImmediately:NO];
		
		//2. 获取单例NSThread的runloop
		NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
		
		//3. 并在获取的runloop上注册监听NSURLConnection端口事件、NSOutputStream端口事件
		for (NSString *runLoopMode in self.runLoopModes) {
		    [self.connection scheduleInRunLoop:runLoop forMode:runLoopMode];
		    [self.outputStream scheduleInRunLoop:runLoop forMode:runLoopMode];
		}
		
		//4. 开始执行网络请求、输出流
		[self.outputStream open];
		[self.connection start];
	}
    [self.lock unlock];
	
    dispatch_async(dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingOperationDidStartNotification object:self];
    });
}

@end
```

从上述代码小结几点:

- (1) 创建了一个常驻后台的单例线程对象，并开启runloop接收各种基于NSPort的端口事件

- (2) 给单例NSThread对象的runloop 添加此时创建出来的如下两个对象的port事件源
	-  创建出来的 `self.outputStream` 对象的事件
	-  创建出来的 `self.connection` 对象的事件

- (3) 并在这个单例NSThread对象上，完成耗时的操作
	- NSOutputStream 完成向某个文件写入数据
	- NSURLConnection 完成对某一个url的网络请求

- (4) 当单例NSThread完成耗时操作之后，线程的runloop会接收到来自系统线程基于NSPort的端口事件，正是上面注册的两个对象的事件源
	- 接收到 `self.outputStream` 对象完成时的事件
	- 接收到 `self.outputStream ` 对象完成时的事件
	
那么对于AFN 2.x版本的一个网络请求实现大概思路就是上面这样的:

```
NSOperation 1
	- start >>> 单例NSThread对象上执行
		- 创建operation对象.outputStream = ....
		- 创建operation对象.connection = ....
		- 给单例NSThread对象.runloop注册上面两个对象的port事件
		- 在单例NSThread对象上开始执行operation对象.outputStream与operation对象.connection

NSOperation 2
	- start >>> 单例NSThread对象上执行
		- 创建operation对象.outputStream = ....
		- 创建operation对象.connection = ....
		- 给单例NSThread对象.runloop注册上面两个对象的port事件
		- 在单例NSThread对象上开始执行operation对象.outputStream与operation对象.connection

NSOperation 3
	- start >>> 单例NSThread对象上执行
		- 创建operation对象.outputStream = ....
		- 创建operation对象.connection = ....
		- 给单例NSThread对象.runloop注册上面两个对象的port事件
		- 在单例NSThread对象上开始执行operation对象.outputStream与operation对象.connection
```

既然是在单例NSThread线程上创建的NSOutputStream对象和NSURLConnection对象，那么应该也是在单例NSThread线程上执行`AFURLConnectionOperation实现的NSURLConnectionDelegate协议函数`。

```objc
@implementation ViewController

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];

    AFJSONResponseSerializer *serializer = [AFJSONResponseSerializer serializer];
    serializer.acceptableContentTypes = [NSSet setWithObject:@"text/html"];
    AFHTTPRequestOperationManager *manager = [AFHTTPRequestOperationManager manager];
    manager.responseSerializer = serializer;
    [manager GET:@"http://www.weather.com.cn/data/sk/101010100.html" parameters:nil success:^(AFHTTPRequestOperation * _Nonnull operation, id  _Nonnull responseObject) {
        NSLog(@"success = %@", responseObject);
    } failure:^(AFHTTPRequestOperation * _Nullable operation, NSError * _Nonnull error) {
        NSLog(@"failure = %@", error);
    }];
}

@end
```

在AFURLConnectionOperation中如下几个方法实现中添加log，打印下当前所在线程对象，来清楚看一下线程流转情况:

```objc
@implementation AFURLConnectionOperation

///1. operation的start
- (void)start {
    NSLog(@"start >>> current thread = %@", [NSThread currentThread]);
    
    [self.lock lock];
    if ([self isCancelled]) {
        [self performSelector:@selector(cancelConnection) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
    } else if ([self isReady]) {
        self.state = AFOperationExecutingState;

        [self performSelector:@selector(operationDidStart) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
    }
    [self.lock unlock];
}

@end

///2. operation对象delegate实现
- (void)connection:(NSURLConnection __unused *)connection
didReceiveResponse:(NSURLResponse *)response
{
    self.response = response;
    NSLog(@"connection:didReceiveResponse: >>>> current thread = %@", [NSThread currentThread]);
}

///3. operation对象结束执行
- (void)finish {
	NSLog(@"finish >>> current thread = %@", [NSThread currentThread]);
	
	// 调用 setState: 实现告诉系统这个opertaion执行结束了
    [self.lock lock];
    self.state = AFOperationFinishedState;
    [self.lock unlock];

	// 回调主线程发送通知
    dispatch_async(dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingOperationDidFinishNotification object:self];
    });
}

///4. operation的completion block执行所在线程
- (void)setCompletionBlock:(void (^)(void))block {
    [self.lock lock];
    if (!block) {
        [super setCompletionBlock:nil];
    } else {
        __weak __typeof(self)weakSelf = self;
        [super setCompletionBlock:^ {
            __strong __typeof(weakSelf)strongSelf = weakSelf;

			NSLog(@"CompletionBlock >>> current thread = %@", [NSThread currentThread]);
				
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wgnu"
            dispatch_group_t group = strongSelf.completionGroup ?: url_request_operation_completion_group();
            dispatch_queue_t queue = strongSelf.completionQueue ?: dispatch_get_main_queue();
#pragma clang diagnostic pop

            dispatch_group_async(group, queue, ^{
                block();
            });

            dispatch_group_notify(group, url_request_operation_completion_queue(), ^{
                [strongSelf setCompletionBlock:nil];
            });
        }];
    }
    [self.lock unlock];
}
```

输出结果

```
2016-11-25 00:15:44.890 RunLoopBasic[6789:99215] start >>> current thread = <NSThread: 0x7fbe40f124d0>{number = 2, name = (null)}
2016-11-25 00:15:44.986 RunLoopBasic[6789:99223] connection:didReceiveResponse: >>>> current thread = <NSThread: 0x7fbe40f16100>{number = 3, name = AFNetworking}
2016-11-25 00:15:44.987 RunLoopBasic[6789:99223] finish >>> current thread = <NSThread: 0x7fbe40f16100>{number = 3, name = AFNetworking}
2016-11-25 00:15:44.999 RunLoopBasic[6789:99219] CompletionBlock >>> current thread = <NSThread: 0x7fbe40e1bce0>{number = 5, name = (null)}
```

可以看到operation其实也是在一个子线程上进行执行start方法实现的。可以看下`-[AFHTTPRequestOperationManager GET:parameters:success:failure:]`中对创建出来的operation是如何执行的

```objc
@implementation AFHTTPRequestOperationManager

- (AFHTTPRequestOperation *)GET:(NSString *)URLString
                     parameters:(id)parameters
                        success:(void (^)(AFHTTPRequestOperation *operation, id responseObject))success
                        failure:(void (^)(AFHTTPRequestOperation *operation, NSError *error))failure
{
	//1. operation 创建
    AFHTTPRequestOperation *operation = [self HTTPRequestOperationWithHTTPMethod:@"GET" URLString:URLString parameters:parameters success:success failure:failure];

	//2. 【重要】使用queue异步子线程调度operation
    [self.operationQueue addOperation:operation];

    return operation;
}

@end
```

那么所以其实使用AFHTTPRequestOperationManager提供的发起请求时，并不需要再自己包一层GCD异步子线程的代码了。因为内部已经是使用了NSOperationQueue去异步子线程执行operation对象了。


回调执行`-[AFURLConnectionOperation connection:didReceiveResponse:]`所在的线程正是开始创建的那个单例的NSThread线程。那么回调是在单例NSThread线程上，那如何回到NSOperation原来所在的线程上了？

```objc
@implementation AFURLConnectionOperation

///网络任务完成后的回调
- (void)connectionDidFinishLoading:(NSURLConnection __unused *)connection {
    self.responseData = [self.outputStream propertyForKey:NSStreamDataWrittenToMemoryStreamKey];

    [self.outputStream close];
    if (self.responseData) {
       self.outputStream = nil;
    }

    self.connection = nil;

	//【重要】结束执行operation
    [self finish];
}

@implementation AFURLConnectionOperation

- (void)finish {

	// 调用 setState: 实现告诉系统这个opertaion执行结束了
    [self.lock lock];
    self.state = AFOperationFinishedState;
    [self.lock unlock];

	// 回调主线程发送通知
    dispatch_async(dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingOperationDidFinishNotification object:self];
    });
}

- (void)setState:(AFOperationState)state {
    if (!AFStateTransitionIsValid(self.state, state, [self isCancelled])) {
        return;
    }
	
	// 手动发出KVO通知，告诉系统当前operation对象正常结束执行
    [self.lock lock];
    NSString *oldStateKey = AFKeyPathFromOperationState(self.state);
    NSString *newStateKey = AFKeyPathFromOperationState(state);

    [self willChangeValueForKey:newStateKey];
    [self willChangeValueForKey:oldStateKey];
    _state = state;
    [self didChangeValueForKey:oldStateKey];
    [self didChangeValueForKey:newStateKey];
    [self.lock unlock];
}


@end
```

此时就会由系统执行`-[NSOperation completionBlock]`，设置给operation对象的block任务。AFN重写了`-[NSOperation setCompletionBlock:]`方法实现，为了让最终block任务在主线程或者用户自己设置的线程上执行。


小结下线程的流转:


```
1. NSOperation被NSOperationQueue子线程异步调度执行所在 `线程A` 执行start

2. 然后start内部又让流程走到了 `单例NSThread线程B` 完成具体的网络请求

3. 完成网络请求之后， 仍然是在 `单例NSThread线程B` 完成所有的 NSConnectionDelegate回调函数

4. opertaion被执行finish方法，发出KVO通知告诉系统结束执行。然后operation对象的completionBlock被随机分配到另外一个 `子线程C` 上执行

5. completion内部，又将流程带到了用户自己设置的 `线程D（默认是主线程）` 上执行
```

注意，对于NSOperation对象，按照上述网络请求的代码的话，是在主线程上进行创建的。

##最终由系统的线程完成网络请求数据交换

但是后来发现最终的网络数据请求是在苹果系统内部的单独的子线程上，使用`CFSocket`相关的函数api完成的，而且还有两个系统子线程:
	
```
- (1) `com.apple.CFSocket.private` 子线程 >>> 完成最终的网络数据交换
- (2) `com.apple.NSURLConnectionLoader` 子线程 >>> 负责向其他线程发送事件源，多线程之间的通信
```

从大牛博客中发现，而且可以从断点的调用栈中发现这两个线程的确是存在的。

从上述可以得到最终是由一个叫做`AFNetworking`的单例NSThread完成网络请求的创建、开始、回调监听，但是最终的网络请求操作确是系统的线程。


