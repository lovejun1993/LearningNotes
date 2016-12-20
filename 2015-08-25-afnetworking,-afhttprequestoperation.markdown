---
layout: post
title: "AFNetworking、AFHTTPRequestOperation"
date: 2015-08-25 14:02:24 +0800
comments: true
categories: 
---

###AFHttpRequestOperation继承自AFURLConnectionOperation，主要完成如下事情:

- 设置 response serailizer
	- 可以设置自己的 response serializer，写一个子类

- 回传 response object

- 设置 success block 与 fail block

***

###AFHttpRequestOperation.h

```objc
#import <Foundation/Foundation.h>
#import "AFURLConnectionOperation.h"

@interface AFHTTPRequestOperation : AFURLConnectionOperation

/**
 *	响应描述，只读
 */
@property (readonly, nonatomic, strong) NSHTTPURLResponse *response;

/**
 *	响应数据解析器，可以设置为自己的 response serializer
 * 
 */
@property (nonatomic, strong) AFHTTPResponseSerializer <AFURLResponseSerialization> * responseSerializer;

/**
 *	响应数据
 */
@property (readonly, nonatomic, strong) id responseObject;

/**
 *	设置传入的回调block
 * 1. 请求成功之后的执行block
 * 2. 请求失败之后的执行block
 */
- (void)setCompletionBlockWithSuccess:(void (^)(AFHTTPRequestOperation *operation, id responseObject))success
                              failure:(void (^)(AFHTTPRequestOperation *operation, NSError *error))failure;

@end
```

****

###AFHttpRequestOperation.m 文件，c方法创建一个单例`并发`队列

```objc
static dispatch_queue_t http_request_operation_processing_queue() {
    static dispatch_queue_t af_http_request_operation_processing_queue;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        af_http_request_operation_processing_queue = dispatch_queue_create("com.alamofire.networking.http-request.processing", DISPATCH_QUEUE_CONCURRENT);
    });

    return af_http_request_operation_processing_queue;
}
```

****

###AFHttpRequestOperation.m 文件，c方法创建一个单例Group

```objc
static dispatch_group_t http_request_operation_completion_group() {
    static dispatch_group_t af_http_request_operation_completion_group;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        af_http_request_operation_completion_group = dispatch_group_create();
    });

    return af_http_request_operation_completion_group;
}
```

***

###AFHttpRequestOperation.m 文件，修改AFURLConnectionOperation中的`只读`权限属性，修改为`readwrite`

```objc
@interface AFURLConnectionOperation ()

@property (readwrite, nonatomic, strong) NSURLRequest *request;

@property (readwrite, nonatomic, strong) NSURLResponse *response;

@end
```

****

###AFHttpRequestOperation.m匿名分类

```objc
@interface AFHTTPRequestOperation ()

@property (readwrite, nonatomic, strong) 
NSHTTPURLResponse *response;

@property (readwrite, nonatomic, strong) id responseObject;

@property (readwrite, nonatomic, strong) NSError *responseSerializationError;

@property (readwrite, nonatomic, strong) NSRecursiveLock *lock;

@end
```

***

###AFHTTPRequestOperation的initWithRequest:方法

```objc
@implementation AFHTTPRequestOperation

- (instancetype)initWithRequest:(NSURLRequest *)urlRequest {
    self = [super initWithRequest:urlRequest];
    if (!self) {
        return nil;
    }

    //默认是二进制NSData响应数据解析格式
    self.responseSerializer = [AFHTTPResponseSerializer serializer];

    return self;
}

@end
```

***

###框架使用者传入的回调Block的处理过程.

####AFHttpRequestOperation向外暴露的设置success与fail的两种block的方法，经过处理后塞给NSOperation的completionBlock

```objc
@implementation AFHTTPRequestOperation

- (void)setCompletionBlockWithSuccess:(void (^)(AFHTTPRequestOperation *operation, id responseObject))success
                              failure:(void (^)(AFHTTPRequestOperation *operation, NSError *error))failure
{
    // completionBlock is manually nilled out in AFURLConnectionOperation to break the retain cycle.
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-retain-cycles"
#pragma clang diagnostic ignored "-Wgnu"

	// 构造block，然后设置给父类
    self.completionBlock = ^{
        
        //1. 开始进入一个 group 工作
        if (self.completionGroup) {
            dispatch_group_enter(self.completionGroup);
        }

        //2. 向 group 发送一个 异步任务
        dispatch_async(http_request_operation_processing_queue(), ^{
            
            
            //2.1 判断是 执行成功block or 执行失败block
            if (self.error)
            {
                //请求发生错误
                
                if (failure) {
                    dispatch_group_async(self.completionGroup ?: http_request_operation_completion_group(), self.completionQueue ?: dispatch_get_main_queue(), ^{
                        failure(self, self.error);
                    });
                }
                
            } else {
                
                //请求成功
                
                //读取响应数据
                id responseObject = self.responseObject;
                
                //如下的if-else有必要吗？能到这个else，说明没有error的
                if (self.error)
                {
                    //我觉得这个if块多余的
                    
                    if (failure) {
                        dispatch_group_async(self.completionGroup ?: http_request_operation_completion_group(), self.completionQueue ?: dispatch_get_main_queue(), ^{
                            failure(self, self.error);
                        });
                    }
                    
                } else {
                    
                    if (success)
                    {
                        dispatch_group_async(self.completionGroup ?: http_request_operation_completion_group(), self.completionQueue ?: dispatch_get_main_queue(), ^{
                            success(self, responseObject);
                        });
                    }
                }
            }

            //2.2 结束提交一个group 工作
            if (self.completionGroup) {
                dispatch_group_leave(self.completionGroup);
            }
        });
    };
#pragma clang diagnostic pop
}

@end
```

```
如上结构:

//1. 开始组
dispatch_group_enter(self.completionGroup);

//2. 开始一个异步任务
dispatch_async(http_request_operation_processing_queue(), ^{
	
	//2.1 异步block代码
	//...
	
	//2.2 结束组操作
	dispatch_group_leave(self.completionGroup);
});
```

- 判断是否发生错误，然后执行对应的block

- 判断是否错误以及执行block，这一系列的逻辑代码又放在一个`全局单例子线程队列上` 异步执行

- 可以传入执行回调block的`group`和`queue`
	- completionGroup 队列组，默认 `NULL`
	- completionQueue 队列，默认 `main queue`
	- 如上两个属性是`继承自AFURLConnectionOperation`得到的

- 最终由`AFURLConnectionOperation`重写`setCompletionBlock:`方法提供

***

###AFHttpRequestOperation直接获取response object

```objc
- (id)responseObject {
    
    //因为后续设计到修改 responseObject属性值
    //所以使用加锁
    [self.lock lock];
    
    if (!_responseObject && [self isFinished] && !self.error)
    {
        NSError *error = nil;
        
        //使用response serialzier 解析 response data
        self.responseObject = [self.responseSerializer responseObjectForResponse:self.response
                                                                            data:self.responseData
                                                                           error:&error];
        if (error) {
        	//如果解析出错
            self.responseSerializationError = error;
        }
    }
    
    [self.lock unlock];

    return _responseObject;
//    return self.responseObject; 使用 self. 会引起死循环
}
```

- 首先获取父类的response data
- 然后通过response serializer解析

###内部用来读取 解析response错误 或 请求错误

```objc
- (NSError *)error {
    if (_responseSerializationError) {
        return _responseSerializationError;
    } else {
        return [super error];
    }
}
```

###最后实现了NSSecureCoding协议，可以归档到磁盘

```objc
+ (BOOL)supportsSecureCoding {
    return YES;
}
```

```objc
- (id)initWithCoder:(NSCoder *)decoder {
    self = [super initWithCoder:decoder];
    if (!self) {
        return nil;
    }

    self.responseSerializer = [decoder decodeObjectOfClass:[AFHTTPResponseSerializer class] forKey:NSStringFromSelector(@selector(responseSerializer))];

    return self;
}

- (void)encodeWithCoder:(NSCoder *)coder {
    [super encodeWithCoder:coder];

    [coder encodeObject:self.responseSerializer forKey:NSStringFromSelector(@selector(responseSerializer))];
}
```

只是对 response serializer 对象 进行归档保存.

***

###最终负责完成网络请求的相关代码全都封装在其父类中.