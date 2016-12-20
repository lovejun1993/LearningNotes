---
layout: post
title: "AFNetworking、AFURLConnectionOperation二"
date: 2015-09-17 18:31:50 +0800
comments: true
categories: 
---

###NSOperation的状态管理，NSOperation状态改变需要手动发出属性值改变的KVO通知.

***

###.m 中定义了operation的所有state

```objc
typedef NS_ENUM(NSInteger, AFOperationState) {
    AFOperationPausedState      = -1,
    AFOperationReadyState       = 1,
    AFOperationExecutingState   = 2,
    AFOperationFinishedState    = 3,
};
```

***

###使用一个static inline函数，将AFOperationState枚举值，转换成NSOperation属性的getter方法名字符串

```objc
static inline NSString * AFKeyPathFromOperationState(AFOperationState state) {
    
    switch (state) {
            
        case AFOperationReadyState:
            return @"isReady";
            
        case AFOperationExecutingState:
            return @"isExecuting";
            
        case AFOperationFinishedState:
            return @"isFinished";
            
        case AFOperationPausedState:
            return @"isPaused";
            
        default: {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wunreachable-code"
            return @"state";
#pragma clang diagnostic pop
        }
    }
}
```

注意没有对 系统状态`isCancel`处理，而是对自定义状态`isPause`处理.

###系统NSOperation主要的方法与属性

```objc
@interface NSOperation : NSObject

//------------------- 声明周期方法 ---------------------

//启动执行
- (void)start;

//线程体函数
- (void)main;

//取消执行
- (void)cancel;

//------------------- 状态属性 ---------------------

@property (readonly, getter=isCancelled) BOOL cancelled;

@property (readonly, getter=isExecuting) BOOL executing;

@property (readonly, getter=isFinished) BOOL finished;

@property (readonly, getter=isReady) BOOL ready;


//其他的系统属性和方法省略

@end
```

系统NSOperation是没有关于`pause暂停`的相关属性的.

***

###再定义了一个static inline函数，判断两个AFOperationState之间进行转换是否合法

```objc
static inline BOOL AFStateTransitionIsValid(AFOperationState fromState, AFOperationState toState, BOOL isCancelled) {
    
    switch (fromState) {
            
        //1. 从ready转变
        case AFOperationReadyState:
            switch (toState) {
                case AFOperationPausedState:
                case AFOperationExecutingState:
                    return YES;
                case AFOperationFinishedState:
                    return isCancelled;
                default:
                    return NO;
            }
            
            
        //2. 从executing转变
        case AFOperationExecutingState:
            switch (toState) {
                case AFOperationPausedState:
                case AFOperationFinishedState:
                    return YES;
                default:
                    return NO;
            }
            
        //3. 从finish转变
        case AFOperationFinishedState:
            return NO;
            
        //4. 从pause转变
        case AFOperationPausedState:
            return toState == AFOperationReadyState;
        default: {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wunreachable-code"
            switch (toState) {
                case AFOperationPausedState:
                case AFOperationReadyState:
                case AFOperationExecutingState:
                case AFOperationFinishedState:
                    return YES;
                default:
                    return NO;
            }
        }
#pragma clang diagnostic pop
    }
}
```

###实现文件声明需要的变量

```objc
@interface AFURLConnectionOperation ()

//其他省略...

@property (readwrite, nonatomic, assign) AFOperationState state;

@property (readwrite, nonatomic, strong) NSRecursiveLock *lock;

@end
```

***


###AFURLConnectionOperation初始化init时

```objc
@implementation AFURLConnectionOperation

- (instancetype)initWithRequest:(NSURLRequest *)urlRequest {
    NSParameterAssert(urlRequest);

    self = [super init];
    if (!self) {
		return nil;
    }

    _state = AFOperationReadyState;

    self.lock = [[NSRecursiveLock alloc] init];
    self.lock.name = kAFNetworkingLockName;
    
    //省略...
    
}
```

operation的state为Ready

***

###重写属性state的setter方法，修改属性值时，同时发出KVO通知

```objc
- (void)setState:(AFOperationState)state {
    
    //1. 验证当前state能否变化成传入的新state
    if (!AFStateTransitionIsValid(self.state, state, [self isCancelled])) {
        return;
    }

    //2. 加锁
    [self.lock lock];
    
    //3. 获取当前state的系统属性getter方法名
    NSString *oldStateKey = AFKeyPathFromOperationState(self.state);
    
    //4. 获取传入的state的系统属性getter方法名
    NSString *newStateKey = AFKeyPathFromOperationState(state);

    //5. 修改state属性值，发出KVO通知
    [self willChangeValueForKey:newStateKey];
    [self willChangeValueForKey:oldStateKey];
    _state = state;
    [self didChangeValueForKey:oldStateKey];
    [self didChangeValueForKey:newStateKey];
    
    //6. 解锁
    [self.lock unlock];
}
```

###当operation执行完毕时，需要执行传入的completionBlock，但是系统不知道operation是否执行完毕。

###那么如何让系统知道operation执行完毕了？

```objc
@interface AFURLConnectionOperation ()

- (void)finish;

@end
```

```objc
@implementation AFURLConnectionOperation

- (void)finish {
    
    //1. 加锁同步
    [self.lock lock];
    
    //2. 修改为Finish状态
    self.state = AFOperationFinishedState;
    
    //3. 解锁
    [self.lock unlock];

    //4. 告诉operation执行完毕
    dispatch_async(dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingOperationDidFinishNotification object:self];
    });
}

@end
```

因为重写了state属性的setter，所以最终会发出`_isFinish`属性修改KVO通知，那么此时系统就知道这个operation执行完毕了。

***

###重写了NSOperation的设置completion block的方法，对传入的block稍加处理

```objc
@implementation AFURLConnectionOperation

- (void)setCompletionBlock:(void (^)(void))block {

	//1. 加锁同步
    [self.lock lock];
    
    //2. 设置block
    if (!block) {
    
    	//2.1 传入的block为nil
        [super setCompletionBlock:nil];
        
    } else {
    
    	//2.2 传入的block非nil
    	
    	/**
    	 *  如下做了 block 内部使用self指针造成retain cycle的问题处理
    	 */
        __weak __typeof(self)weakSelf = self;
        [super setCompletionBlock:^ {
            __strong __typeof(weakSelf)strongSelf = weakSelf;

#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wgnu"
			//2.2.1 获取group
            dispatch_group_t group = strongSelf.completionGroup ?: 
            
            //2.2.2 获取queue
            url_request_operation_completion_group();
            dispatch_queue_t queue = strongSelf.completionQueue ?: dispatch_get_main_queue();
#pragma clang diagnostic pop

			//2.2.3 使用group + queue的async任务
            dispatch_group_async(group, queue, ^{
                block();
            });

			//2.2.4 使用group_notify最终执行代码块
			//释放掉operation对象引用的block，那么block也就不会再指向operation对象
			//这样就解决了 `operation对象` 与 `block` 之间的循环强引用
			//并且这里是在另一个线程上执行的设置completion block，所以setCompletionBlock:需要使用 Lock 来同步
            dispatch_group_notify(group, url_request_operation_completion_queue(), ^{
                [strongSelf setCompletionBlock:nil];
            });
        }];
    }
    
	//3. 解锁
    [self.lock unlock];
}

@end
```

如上了解到，有一些需要执行完一些异步操作之后，额外再做一些操作时，可以考虑使用

- dispatch group async 任务
- dispatch group notify 最终代码块

***

###最后就是重写NSOperation的状态相关函数

```objc
@implementation AFURLConnectionOperation

- (BOOL)isReady {
    return self.state == AFOperationReadyState && [super isReady];
}

- (BOOL)isExecuting {
    return self.state == AFOperationExecutingState;
}

- (BOOL)isFinished {
    return self.state == AFOperationFinishedState;
}

- (BOOL)isConcurrent {
    return YES;
}

@end
```

###附加的关于暂停状态的函数

```objc
@implementation AFURLConnectionOperation

- (BOOL)isPaused {
    return self.state == AFOperationPausedState;
}

@end
```