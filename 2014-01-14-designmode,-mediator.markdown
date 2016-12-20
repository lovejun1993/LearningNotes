---
layout: post
title: "DesignMode、Mediator"
date: 2014-01-14 15:01:51 +0800
comments: true
categories: 
---

###中介者、通过`中间者`解耦 `消息发送者`与`消息接收者` 之间的强耦合.

***

###消息发送者和消息接受者都需要实现的协议

```objc
//
//  MediaContact.h
//  demos
//
//  Created by xiongzenghui on 14/2/1.
//  Copyright © 2014年 xiongzenghui. All rights reserved.
//

#import <Foundation/Foundation.h>

/**
 *  能够通过到中介者沟通的抽象
 */
@protocol MediaContact <NSObject>

/**
 *  已经发送消息
 */
- (void)didSendMessage;

/**
 *  接收到消息
 */
- (void)didReceiveMessage:(NSString *)message;

@end
```

###抽象中介者

```objc
//
//  AbstractMediator.h
//  demos
//

#import <Foundation/Foundation.h>

//消息发送者与消息接受者
#import "MediaContact.h"

/**
 *  抽象中介者
 */
@interface AbstractMediator : NSObject

/**
 *  单例
 */
+ (instancetype)sharedInstance;

/**
 *  由中介者发送消息
 *
 *  @param msg  消息
 *  @param from 发送方
 *  @param to   接收方
 */
- (void)contactMessage:(NSString *)msg
                  From:(id<MediaContact>)from
                    To:(id<MediaContact>)to;

@end
```

```objc
//
//  AbstractMediator.m
//  demos
//

#import "AbstractMediator.h"

@implementation AbstractMediator

+ (instancetype)sharedInstance {
    static id mediator = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        mediator = [[self alloc] init];
    });
    return mediator;
}

- (void)contactMessage:(NSString *)msg
                  From:(id<MediaContact>)from
                    To:(id<MediaContact>)to
{
    //空实现
}

@end
```

###交流方A


```objc
//
//  PersonA.h
//  demos
//

#import <Foundation/Foundation.h>

#import "MediaContact.h"

/**
 *  实现接口成为能够注册到中介者的能力
 */
@interface PersonA : NSObject <MediaContact>

@end
```

```objc
//
//  PersonA.m
//  demos
//

#import "PersonA.h"

@implementation PersonA

- (void)didSendMessage {
    NSLog(@"已发送消息回调");
}

- (void)didReceiveMessage:(NSString *)message {
    NSLog(@"接收到消息: %@", message);
}

@end
```

###交流方B

```objc
//
//  PersonB.h
//  demos
//

#import <Foundation/Foundation.h>

#import "MediaContact.h"

/**
 *  实现接口成为能够注册到中介者的能力
 */
@interface PersonB : NSObject <MediaContact>

@end
```

```objc
//
//  PersonB.m
//  demos
//

#import "PersonB.h"

@implementation PersonB

- (void)didSendMessage {
    NSLog(@"已发送消息回调");
}

- (void)didReceiveMessage:(NSString *)message {
    NSLog(@"接收到消息: %@", message);
}

@end
```

###具体中介者

```objc
//
//  MediatorA.h
//  demos
//

#import "AbstractMediator.h"

@interface MediatorA : AbstractMediator

@end
```

```objc
//
//  MediatorA.m
//  demos
//

#import "MediatorA.h"

@implementation MediatorA

/**
 *  完成具体消息功能
 *
 *  1. 发送消息
 *  2. 接收消息
 */
- (void)contactMessage:(NSString *)msg
                  From:(id<MediaContact>)from
                    To:(id<MediaContact>)to
{
    //1. 通知接收方收取消息
    [to didReceiveMessage:msg];
    
    //2. 通知发送方，接送方已经接收消息
    [from didSendMessage];
}

@end
```

###A与B通过中介者进行通信

```objc
//
//  MediatorTest.m
//  demos
//

#import "MediatorTest.h"

//沟通双方
#import "PersonA.h"
#import "PersonB.h"

//消息中间件
#import "MediatorA.h"

@implementation MediatorTest

- (void)test {
    
    //1. 交流方A
    id<MediaContact> A = [PersonA new];
    
    //2. 交流方B 
    //这里假设A或B，不能在A中直接导入B，应该要让消息中间件容器进行隔离
    //可以使用一个标识来代替
    //如下直接导入PersonB，只是为了代码简单方便
    id<MediaContact> B = [PersonB new];
    
    //3. A向B发送消息
    [[MediatorA sharedInstance] contactMessage:@"哈哈哈哈"
                                          From:A
                                            To:B];
}

@end
```


###最终B只是接到一个消息字符串，并不知道该消息是A发送的，完成了A与B的解耦.