---
layout: post
title: "DesignMode、Chain of Responsibility"
date: 2014-01-12 12:48:25 +0800
comments: true
categories: 
---

###责任链、将多个对象组装在一起，按照顺序依次执行处理.

***

###抽象Request类

```objc
//
//  AbstractChainRequest.h
//  demos
//

#import <Foundation/Foundation.h>

@interface AbstractChainRequest : NSObject

/**
 *  下一个处理器
 */
@property (nonatomic, strong) AbstractChainRequest *next;

/**
 *  当前对象的处理
 */
- (void)handle;

@end
```

```objc
//
//  AbstractChainRequest.m
//  demos
//

#import "AbstractChainRequest.h"

@implementation AbstractChainRequest

- (void)handle{}

@end
```

###具体Request1

```objc
//
//  Request1.h
//  demosht © 2016年 xiongzenghui. All rights reserved.
//

#import "AbstractChainRequest.h"

@interface Request1 : AbstractChainRequest

@end
```

```objc
//
//  Request1.m
//  demos
//

#import "Request1.h"

@implementation Request1

- (void)handle {
    
    //1.
    NSLog(@"Request1 Handle");
    
    //2.
    [self.next handle];
}

@end
```

###具体Request2

```objc
//
//  Request2.h
//  demos
//

#import "AbstractChainRequest.h"

@interface Request2 : AbstractChainRequest

@end
```

```objc
//
//  Request2.m
//  demos
//

#import "Request2.h"

@implementation Request2

- (void)handle {
    
    //1.
    NSLog(@"Request2 Handle");
    
    //2.
    [self.next handle];
}

@end
```

###具体Request3

```objc
//
//  Request3.h
//  demos
//

#import "AbstractChainRequest.h"

@interface Request3 : AbstractChainRequest

@end
```

```objc
//
//  Request3.m
//  demos
//

#import "Request3.h"

@implementation Request3

- (void)handle {
    
    //1.
    NSLog(@"Request3 Handle");
    
    //2.
    [self.next handle];
}

@end
```

###demo示例

```objc
#import "ChainRequestTest.h"

#import "Request1.h"
#import "Request2.h"
#import "Request3.h"

@implementation ChainRequestTest

- (void)test {
    
    //1. 所有的Request对象创建
    Request1 *req1 = [Request1 new];
    Request2 *req2 = [Request2 new];
    Request3 *req3 = [Request3 new];
    
    //2. 建立Reuqest对象之间的链接关系
    // （req1->req2->req3->nil）
    req1.next = req2;
    req2.next = req3;
    [req1 handle];
}

@end
```