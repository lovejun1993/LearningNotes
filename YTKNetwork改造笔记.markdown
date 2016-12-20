---
layout: post
title: "YTKNetwork改装"
date: 2015-01-22 11:01:28 +0800
comments: true
categories: 
---

##AFNetworking创建NSMutableURLRequest的几种类型

```objc
@interface AFHTTPRequestSerializer : NSObject <AFURLRequestSerialization>

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
                                     error:(NSError * _Nullable __autoreleasing *)error;

@end
```

有两点:

- (1) `GET`, `HEAD`, `DELETE`这三种:
   - 首先将参数拼接成`&key=value`的字符串形式，然后再对字符串进行URL Encode
	- 最后将`&key=value`字符串，添加到request's url后面

- (2) `POST`, `PUT`这两种:
	- 首先将参数拼接成`&key=value`的字符串形式

	
	
下面是实现代码

```objc
@implementation AFHTTPRequestSerializer

- (NSMutableURLRequest *)requestWithMethod:(NSString *)method
                                 URLString:(NSString *)URLString
                                parameters:(id)parameters
                                     error:(NSError *__autoreleasing *)error
{
    // 后台接口请求路径path url
    NSURL *url = [NSURL URLWithString:URLString];

    // 创建NSMutableURLRequest请求对象
    NSMutableURLRequest *mutableRequest = [[NSMutableURLRequest alloc] initWithURL:url];
    mutableRequest.HTTPMethod = method;

    /**
     *  KVO关注当前AFHTTPRequestSerializer对象如下属性值改变，做出一些处理
     *  然后这里取出AFHTTPRequestSerializer对象中的如下属性值，再复制给NSMutableURLRequest请求对象
     *
     *  - (1) allowsCellularAccess
     *  - (2) cachePolicy
     *  - (3) HTTPShouldHandleCookies
     *  - (4) HTTPShouldUsePipelining
     *  - (5) networkServiceType
     *  - (6) timeoutInterval
     */
    for (NSString *keyPath in AFHTTPRequestSerializerObservedKeyPaths()) {
        if ([self.mutableObservedChangedKeyPaths containsObject:keyPath]) {
            [mutableRequest setValue:[self valueForKeyPath:keyPath] forKey:keyPath];
        }
    }

    /**
     *  完成具体的NSMutableURLRequest请求对象，这里分三种实现
     *  - (1) AFHTTPRequestSerializer
     *  - (2) AFJSONRequestSerializer
     *  - (3) AFJSONRequestSerializer
     */
    mutableRequest = [[self requestBySerializingRequest:mutableRequest withParameters:parameters error:error] mutableCopy];

	return mutableRequest;
}

@end
```

完成具体的NSMutableURLRequest请求对象的实现代码之、AFHTTPRequestSerializer

```objc
- (NSURLRequest *)requestBySerializingRequest:(NSURLRequest *)request
                               withParameters:(id)parameters
                                        error:(NSError *__autoreleasing *)error
{
    NSParameterAssert(request);

    NSMutableURLRequest *mutableRequest = [request mutableCopy];
	
	// 请求头参数设置
    [self.HTTPRequestHeaders enumerateKeysAndObjectsUsingBlock:^(id field, id value, BOOL * __unused stop) {
        if (![request valueForHTTPHeaderField:field]) {
            [mutableRequest setValue:value forHTTPHeaderField:field];
        }
    }];
	
	/**
     *  将所有的请求参数组装成参数字符串NSString
     *  - (1) 由AFN框架组装参数字符串 >>>> queryStringSerialization == nil;
     *  - (2) 由用户自己组装参数字符串 >>>> queryStringSerialization != nil;
     */
    NSString *query = nil;
    if (parameters) {
        if (self.queryStringSerialization) {
            NSError *serializationError;
            query = self.queryStringSerialization(request, parameters, &serializationError);

            if (serializationError) {
                if (error) {
                    *error = serializationError;
                }

                return nil;
            }
        } else {
            switch (self.queryStringSerializationStyle) {
                case AFHTTPRequestQueryStringDefaultStyle:
                    query = AFQueryStringFromParameters(parameters);
                    break;
            }
        }
    }

	/**
     *  区分不同的请求method，来决定不同形式参数设置
     *  - (1) GET\HEAD\DELETE 
     *      - 直接拼接成一个 key=value 字符串
     *      - 然后拼接在 NSURLRequest.url后面
     *  - (2) POST\PUT 
     *      - 先 拼接成 key=value 字符串
     *      - 然后将字符串转换成NSData
     *      - 最后将NSData设置给NSMutableURLRequest.HTTPBody属性值
     */
    if ([self.HTTPMethodsEncodingParametersInURI containsObject:[[request HTTPMethod] uppercaseString]]) {
    	//>>>>>>>>>>>>>>>>>>>>>>> GET\HEAD\DELETE
        if (query && query.length > 0) {
            mutableRequest.URL = [NSURL URLWithString:[[mutableRequest.URL absoluteString] stringByAppendingFormat:mutableRequest.URL.query ? @"&%@" : @"?%@", query]];
        }
    } else {
    	//>>>>>>>>>>>>>>>>>>>>>>> POST\PUT 
        // #2864: an empty string is a valid x-www-form-urlencoded payload
        if (!query) {
            query = @"";
        }
        if (![mutableRequest valueForHTTPHeaderField:@"Content-Type"]) {
            [mutableRequest setValue:@"application/x-www-form-urlencoded" forHTTPHeaderField:@"Content-Type"];
        }
        [mutableRequest setHTTPBody:[query dataUsingEncoding:self.stringEncoding]];
    }

    return mutableRequest;
}
```

完成具体的NSMutableURLRequest请求对象的实现代码之、AFJSONRequestSerializer

```objc
@implementation AFJSONRequestSerializer

- (NSURLRequest *)requestBySerializingRequest:(NSURLRequest *)request
                               withParameters:(id)parameters
                                        error:(NSError *__autoreleasing *)error
{
    // GET、HEAD、DELETE，直接调用AFHTTPRequestSerializer进行key=value字符串构建与拼接
    if ([self.HTTPMethodsEncodingParametersInURI containsObject:[[request HTTPMethod] uppercaseString]]) {
        return [super requestBySerializingRequest:request withParameters:parameters error:error];
    }
    
    // 下面是POST、PUT的情况
    NSMutableURLRequest *mutableRequest = [request mutableCopy];

    // 请求头参数设置
    [self.HTTPRequestHeaders enumerateKeysAndObjectsUsingBlock:^(id field, id value, BOOL * __unused stop) {
        if (![request valueForHTTPHeaderField:field]) {
            [mutableRequest setValue:value forHTTPHeaderField:field];
        }
    }];
    
    /**
     *  设置接口调用需要的请求参数
     */
    if (parameters) {
        
        //1. 先设置 `request.Content-Type = application/json`
        if (![mutableRequest valueForHTTPHeaderField:@"Content-Type"]) {
            [mutableRequest setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
        }
        
        /** 2. 请求参数NSDictionary对象处理
         *  - (1) 将传入的请求参数NSDictionary转换成JSON格式的NSData数据
         *      - 对于json化处理时对objc对象的要求: https://developer.apple.com/reference/foundation/jsonserialization
         *  - (2) 再设置给NSMutableURLRequest.HTTPBody属性值
         */
        [mutableRequest setHTTPBody:[NSJSONSerialization dataWithJSONObject:parameters options:self.writingOptions error:error]];
    }

    return mutableRequest;
}

@end
```

完成具体的NSMutableURLRequest请求对象的实现代码之、AFPropertyListRequestSerializer

```objc
@implementation AFPropertyListRequestSerializer

- (NSURLRequest *)requestBySerializingRequest:(NSURLRequest *)request
                               withParameters:(id)parameters
                                        error:(NSError *__autoreleasing *)error
{
    NSParameterAssert(request);

    if ([self.HTTPMethodsEncodingParametersInURI containsObject:[[request HTTPMethod] uppercaseString]]) {
        return [super requestBySerializingRequest:request withParameters:parameters error:error];
    }

    NSMutableURLRequest *mutableRequest = [request mutableCopy];

    [self.HTTPRequestHeaders enumerateKeysAndObjectsUsingBlock:^(id field, id value, BOOL * __unused stop) {
        if (![request valueForHTTPHeaderField:field]) {
            [mutableRequest setValue:value forHTTPHeaderField:field];
        }
    }];

    if (parameters) {
        if (![mutableRequest valueForHTTPHeaderField:@"Content-Type"]) {
            [mutableRequest setValue:@"application/x-plist" forHTTPHeaderField:@"Content-Type"];
        }
        
        /**
         *  与AFJSONRequestSerializer稍微不同的地方
         *  - (1) 先将参数NSDictionary对象，转换成plist格式的NSData数据
         *  - (2) 同样再设置给NSMutableURLRequest.HTTPBody属性值
         */
        [mutableRequest setHTTPBody:[NSPropertyListSerialization dataWithPropertyList:parameters format:self.format options:self.writeOptions error:error]];
    }

    return mutableRequest;
}

@end
```

###小结下AFHTTPRequestSerializer、AFJSONRequestSerializer、AFPropertyListRequestSerializer当请求method是GET\HEAD\DELETE\POST\PUT时候，分别对传入的请求参数NSDictionary字典对象不同的处理情况

| 三种RequestSerializer | GET、HEAD、DELETE |  POST、PUT |
| :------------- |:-------------:| -------------:|
| AFHTTPRequestSerializer | 将请求参数拼接成url?key1=value1&key2=value2...这样的字符串格式，直接拼接到NSURLRequest.url后面 | 将请求参数拼接成url?key1=value1&key2=value2...这样的字符串格式，然后直接转换成NSData数据，设置给NSMutableURLRequest.HTTPBody属性 |
| AFJSONRequestSerializer | 将请求参数拼接成url?key1=value1&key2=value2...这样的字符串格式，直接拼接到NSURLRequest.url后面 | 将请求参数拼接成url?key1=value1&key2=value2...这样的字符串格式，然后直接转换成`JSON格式`的NSData数据，设置给NSMutableURLRequest.HTTPBody属性 |
| AFPropertyListRequestSerializer | 将请求参数拼接成url?key1=value1&key2=value2...这样的字符串格式，直接拼接到NSURLRequest.url后面 | 将请求参数拼接成url?key1=value1&key2=value2...这样的字符串格式，然后直接转换成`Plist格式`的NSData数据，设置给NSMutableURLRequest.HTTPBody属性 |

###对于AFURLSessionManager提供的`uploadTask...`完成上传的接口，需要自手动构造NSURLRequest对象

AFURLSessionManager的如下这个函数，用于数据上传这种特殊的网络请求

```objc
- (NSMutableURLRequest *)multipartFormRequestWithMethod:(NSString *)method
                                              URLString:(NSString *)URLString
                                             parameters:(NSDictionary *)parameters
                              constructingBodyWithBlock:(void (^)(id <AFMultipartFormData> formData))block
                                                  error:(NSError *__autoreleasing *)error
```

使用例子

```objc
//1. 
NSMutableURLRequest *request = [[AFHTTPRequestSerializer serializer] multipartFormRequestWithMethod:@"POST" URLString:kUploadHOST parameters:nil constructingBodyWithBlock:^(id<AFMultipartFormData> formData) {
    [formData appendPartWithFileData:model.filedata name:@"filename" fileName:stuffix mimeType:@"file"];
} error:nil];


//2. 
__weak __typeof(self)weakSelf = self;
NSURLSessionUploadTask *task = [_manager uploadTaskWithStreamedRequest:request progress:^(NSProgress * _Nonnull uploadProgress) {
    __strong __typeof(weakSelf)strongSelf = weakSelf;

 } completionHandler:^(NSURLResponse * _Nonnull response, id  _Nullable responseObject, NSError * _Nullable error) {


 }];

//3. 
[task resume];
```

如上这个构建multi part form request的函数实现:

```objc
- (NSMutableURLRequest *)multipartFormRequestWithMethod:(NSString *)method
                                              URLString:(NSString *)URLString
                                             parameters:(NSDictionary *)parameters
                              constructingBodyWithBlock:(void (^)(id <AFMultipartFormData> formData))block
                                                  error:(NSError *__autoreleasing *)error
{
    NSParameterAssert(method);
    NSParameterAssert(![method isEqualToString:@"GET"] && ![method isEqualToString:@"HEAD"]);

    NSMutableURLRequest *mutableRequest = [self requestWithMethod:method URLString:URLString parameters:nil error:error];

    __block AFStreamingMultipartFormData *formData = [[AFStreamingMultipartFormData alloc] initWithURLRequest:mutableRequest stringEncoding:NSUTF8StringEncoding];

    if (parameters) {
        for (AFQueryStringPair *pair in AFQueryStringPairsFromDictionary(parameters)) {
            NSData *data = nil;
            if ([pair.value isKindOfClass:[NSData class]]) {
                data = pair.value;
            } else if ([pair.value isEqual:[NSNull null]]) {
                data = [NSData data];
            } else {
                data = [[pair.value description] dataUsingEncoding:self.stringEncoding];
            }

            if (data) {
                [formData appendPartWithFormData:data name:[pair.field description]];
            }
        }
    }

    if (block) {
        block(formData);
    }
	
	// 最终请求参数设置
    return [formData requestByFinalizingMultipartFormData];
}
```

```objc
- (NSMutableURLRequest *)requestByFinalizingMultipartFormData {
    if ([self.bodyStream isEmpty]) {
        return self.request;
    }

    // Reset the initial and final boundaries to ensure correct Content-Length
    [self.bodyStream setInitialAndFinalBoundaries];
    [self.request setHTTPBodyStream:self.bodyStream];

    [self.request setValue:[NSString stringWithFormat:@"multipart/form-data; boundary=%@", self.boundary] forHTTPHeaderField:@"Content-Type"];
    [self.request setValue:[NSString stringWithFormat:@"%llu", [self.bodyStream contentLength]] forHTTPHeaderField:@"Content-Length"];

    return self.request;
}
```

可以看到最终是设置的`NSMutableURLRequest.HTTPBodyStream`这个文件输入输出流对象属性。


##使用缓存容器来持有YTKRequest临时对象，即不用再自己使用实例变量去持有创建的YTKRequest对象

```objc
- (void)test2 {

	// 临时的Request对象
	Wearther2Request *req2 = [[Wearther2Request alloc] initWithkeyWord:@"苏州市"];
	
	// 发起网络请求并执行回调
    [req2 startWithSuccessComplet:^(KernelBaseRequest *request) {
        
    } failComplet:^(KernelBaseRequest *request) {
        
        NSLog(@"error = %@", request.responseError);
    }];
}
```

如上代码创建的是一个`临时`的Request对象，但是依然可以出了方法块之后完成最终的网络请求回调Block。

正是因为有一个静态单例类Agant，里面有一个缓存块，用来缓存这些`临时`的Request对象:

- (1) 当请求start时，将这个`临时`的Request对象，缓存起来，让其不备提前废弃

- (2) 当请求finish时，从缓存中移除`临时`的Request对象，让这个`临时`的Request对象release释放一次

- (3) 当请求执行cancel时，也从缓存中移除`临时`的Request对象，让这个`临时`的Request对象release释放一次

###YTKNetwork主要基于文件缓存，然后我将其做了一些优化:

- (1) 主要依赖于内存缓存，在程序运行时，基本上只使用内存缓存
- (2) 并提供内存缓存数据的存取多线程同步安全
- (3) 内存缓存淘汰策略使用`LRU算法`
- (4) 缓存查询，优先查询内存缓存，如果内存没有再去文件缓存读取
- (5) App后台、退出、接收内存警告，才将内存缓存写入磁盘文件

##其中可能出现Manager/ViewModel对象与Request对象之间的强引用，就会造成互相不被释放继而内存泄漏。虽然可以通过`__weak`来转换指针引用来解决，但是YTK中给出了一种可以写`self`的另外一种解决办法更方便.

> 先看第一种情形，两个局部对象互相强引用

- (1) ViewModel类

```objc
#import <Foundation/Foundation.h>
#import "Request.h"

@interface ViewModel : NSObject

@property (nonatomic, strong) Request *req;

- (void)log;

@end

@implementation ViewModel

- (void)dealloc {
    NSLog(@"ViewModel Dealloc %@ - %p", self, self);
}

- (void)log {
    NSLog(@"ViewModel work...");
}

@end
```

- Request类

```objc
#import <Foundation/Foundation.h>

@interface Request : NSObject

@property (nonatomic, copy) void (^block)(void);

- (void)clearBlocks;

@end

@implementation Request

- (void)dealloc {
    NSLog(@"Request Dealloc %@ - %p", self, self);
}

- (void)clearBlocks {
    _block = nil;
}

@end
```

- ViewController中持有一个ViewModel对象，而ViewModel

第一种不会出现循环引用的写法

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    ViewModel *vm = [ViewModel new];
    Request *req = [Request new];
}
```

第二种会出现循环引用的写法

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
 
     ViewModel *vm = [ViewModel new];// vm retainCount == 1
     
     Request *req = [Request new];// req retainCount == 1
     
     vm.req = req;// vm retainCount == 1, req retainCount == 2
     
     req.block = ^() { [vm log]; };// vm retainCount == 2, req retainCount == 2
 
}// vm retainCount == 1, req retainCount == 1 >>> 内存泄漏
```

从注释中可以看出为什么不释放的原因了，因为retainCount没有执行-1，就是说互相持有对方造成双方的retainCount都不会-1。

![](http://7xrn7f.com1.z0.glb.clouddn.com/16-7-22/63405488.jpg)

> 此时解决办法是: 必须主动让其中一方释放对对方的持有。

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    ViewModel *vm = [ViewModel new];// vm retainCount == 1
    
    Request *req = [Request new];// req retainCount == 1
    
    vm.req = req;// vm retainCount == 1, req retainCount == 2
    
    req.block = ^() { [vm log]; };// vm retainCount == 2, req retainCount == 2
    
    // 主动释放Request对象对ViewModel对象的持有
    [req clearBlocks];// vm retainCount == 1, req retainCount == 2
    
}// vm retainCount == 0, req retainCount == 1 >>> req retainCount == 0
```

最后一步`[req clearBlocks]`解决循环引用的步骤:
	
- (1) 在最后清除掉Request对象的Block属性指向的Block对象
- (2) 那么Block对象失去了唯一的strong指针，那么Block对象就被废弃了
- (3) 而Block对象被废弃，就意味着ViewModel对象失去了一个strong指针，即会发送消息 `[VieModel对象 release]`
- (4) 那么此时ViewModel对象的strong指针，就只有局部函数中的一个临时strong指针了
- (5) 当临时指针超出方法范围，临时指针无效，那么继续发送`[VieModel对象 release]`，至此`ViewModel的retainCount == 0`，即被废弃了
- (6) 而ViewModel对象废弃，又造成Request对象没有了strong引用，接着`[Request对象 release]`

> 第二种情况，在遵循MVVM结构时，可能通常在ViewController中绑定一个ViewModel对象，然后通过ViewModel暴露接口方法获取网络数据并执行json解析成实体类，再返回给UI对象使用。

- (1) ViewModel类

```objc
#import <Foundation/Foundation.h>

@interface ViewModel : NSObject

/**
 *  提供给UI层对象的接口
 */
- (void)apiCall:(void (^)(id entity))block;

@end

#import "Request.h"
@implementation ViewModel {
    
    // 成员属性强引用
    Request __strong *_req;
}

- (void)apiCall:(void (^)(id entity))block {
    
    // 创建一个请求，模拟网络请求
    _req = [Request new];
    NSLog(@"当前创建的Request对象: %p", _req);
    
    // 设置请求回调
    _req.block = ^() {
        
        // 解析得到的json
        id entity = [self parseJSON];//注意，为了测试这里不要写weakSelf，要不然出现不了循环引用
        
        // 回传解析成的实体对象
        if (block) block(entity);
    };
    
    // 开始网络请求
    [_req start];
}

- (void)dealloc {
    NSLog(@"ViewModel Dealloc %@ - %p", self, self);
}

/**
 *  解析json成为实体类对象
 */
- (id)parseJSON {
    return [NSObject new];
}

@end
```

- (2) Request类

```objc
#import <Foundation/Foundation.h>

@interface Request : NSObject

@property (nonatomic, copy) void (^block)(void);

- (void)clearBlocks;

- (void)start;

@end

@implementation Request

- (void)dealloc {
    NSLog(@"当前废弃的Request对象: %p", self);
}

- (void)clearBlocks {
    _block = nil;
}

- (void)start {
    NSLog(@"开始网络请求");
}

@end
```

- (3) ViewController中的测试代码

```objc
#import "ViewModel.h"

@implementation RetainCycleViewController {
    ViewModel __strong *_vm;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    _vm = [ViewModel new];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [_vm apiCall:^(id entity) {
        NSLog(@"接收到的响应实体数据: %@", entity);
    }];
    
    // 测试ViewModel对象是否能被废弃
    _vm = nil;
}

@end
```

运行后无论点击屏幕多少次只会输出如下log

```
2016-07-22 13:37:30.958 Demo[5049:562544] 当前创建的Request对象: 0x7fb311f405b0
2016-07-22 13:37:30.958 Demo[5049:562544] 开始网络请求
```

但是并没有看到`ViewModel对象`的废弃的log，那么就是说即使最后对ViewController对象持有的ViewModel对象减少了一个storng指针，但仍然没有被废弃。


但是点击第二次，第三次...第N次都没有打印`开始网络请求`，是因为当前ViewController对象的`_vm`指针已经赋值`nil`了，即Viewmodel对象仍然存在，但`_vm`指针已经不再指向了，也就无法操做ViewMidel对象了。


没有看到任何关于`ViewModel对象废弃`的log，那就只能说明 >>> `还有其他的strong指针，仍然指向这个ViewModel对象`。

其实仍然指向ViewModel对象的指针就是`Request对象中的Block属性对于的实例变量`，ViewModel对象不被废弃主要是因为ViewModel中的`- (void)apiCall:(void (^)(id entity))block`方法中对`ViewModel对象的retainCount+1`:


```objc
- (void)apiCall:(void (^)(id entity))block {
    

    _req = [Request new];//req.retainCount == 1
    NSLog(@"当前创建的Request对象: %p", _req);
    
    _req.block = ^() {
        id entity = [self parseJSON];//viewModel.retainCount == 2
        if (block) block(entity);
    };
    
    [_req start];

}//因为request对象是当前VieModel对象的实例变量指向，那么ViewModel本身不被废弃，就不会释放实例变量指向的对象
```

经过这个方法之后，ViewModel对象的retainCount == 2。

就算ViewController对象释放掉一个strong指向之后，ViewModel对象的retainCount == 1。

而这个最后的一个指向，永远都不可能自动释放，继而ViewModel对象不会被废弃。

> 对上面的解决: 主动让ViewModel对象 或 Request对象 其中的一方，释放掉对另外一方的持有。


可以看到，Request类中有个`clearBlocks`对象方法，就是用来主动释放掉Block对象的，继而可以释放掉队Block对象对ViewModel对象的持有:

```objc
- (void)clearBlocks {
    _block = nil;
}
```

OK，那么只需要对ViewModel的`apiCall:`在上面实现的末尾加上调用Request对象的clearBlocks方法实现即可解除循环引用

```objc
- (void)apiCall:(void (^)(id entity))block {
    

    _req = [Request new];//req.retainCount == 1
    NSLog(@"当前创建的Request对象: %p", _req);
    
    _req.block = ^() {
        id entity = [self parseJSON];//viewModel.retainCount == 2
        if (block) block(entity);
    };
    
    [_req start];
    
    // 解除循环引用
    [_req clearBlocks];//viewModel.retainCount == 1

}
```

运行后点击屏幕的log:

```
2016-07-22 13:55:37.570 Demo[5207:608777] 开始网络请求
2016-07-22 13:55:37.570 Demo[5207:608777] ViewModel Dealloc <ViewModel: 0x7ff51b536d00> - 0x7ff51b536d00
2016-07-22 13:55:37.570 Demo[5207:608777] 当前废弃的Request对象: 0x7ff51b564360
```

从log可以看到:

- (1) Viewmodel对象最先被废弃掉
- (2) 然后接着废弃掉ViewModel对象持有的Request对象

OK，那么这才是正常的效果了。再小结如上ViewModel对象的retainCount变化:

- (1) `_req.block` 持有了VieModel对象 >>> VieModel对象.retainCount + 1

- (2) `[_req clearBlocks]` >>> VieModel对象.retainCount - 1

所以说当一段代码对一个`对象retainCount + 1`之后，当在即将执行完毕时必须让`对象retainCount - 1`保持平衡，不打破之前对象的retainCount的平衡性。

也就是说，对一个对象的有retain，也就必须有release，必须成对出现。

