---
layout: post
title: "DesignMode、 Template"
date: 2014-01-11 12:32:29 +0800
comments: true
categories: 
---

###模板模式、提供一些基础的模板方法、步骤性的逻辑代码框架给其他的子类.

之前BaseViewController使用过这个模式.

***

###抽象模板接口

```objc
//
//  Template.h
//  demos
//

#import <Foundation/Foundation.h>

@protocol Template <NSObject>


////////////////////////////////////////////////////////////
/////每一个步骤方法可以选择重写实现
////////////////////////////////////////////////////////////
- (void)step1;

- (void)step2;

- (void)step3;

- (void)step4;


////////////////////////////////////////////////////////////
/////业务逻辑步骤不能重写
////////////////////////////////////////////////////////////
- (void)business;

@end
```

###默认实现类作为抽象类

```objc
//
//  AbstractTemplate.h
//  demos
//

#import <Foundation/Foundation.h>

//实现模板接口
#import "Template.h"

@interface AbstractTemplate : NSObject <Template>

@end
```

```objc
//
//  AbstractTemplate.m
//  demos
//

#import "AbstractTemplate.h"

@implementation AbstractTemplate

- (void)step1 {}
- (void)step2 {}
- (void)step3 {}
- (void)step4 {}

- (void)business {
    [self step1];
    [self step2];
    [self step3];
    [self step4];
}

@end
```

###具体模板

```objc
//
//  ConcreatTemplate.h
//  demos
//

#import <Foundation/Foundation.h>

#import "AbstractTemplate.h"

@interface ConcreatTemplate : AbstractTemplate

@end
```

```objc
//
//  ConcreatTemplate.m
//  demos
//

#import "ConcreatTemplate.h"

@implementation ConcreatTemplate

- (void)step1 {
    NSLog(@"步骤1");
}

- (void)step2 {
    NSLog(@"步骤2");
}

- (void)step3 {
    NSLog(@"步骤3");
}

- (void)step4 {
    NSLog(@"步骤4");
}

@end
```

感觉有点像 `建造者` 模式.