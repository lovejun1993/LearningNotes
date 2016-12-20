---
layout: post
title: "DesignMode、Factory"
date: 2014-01-01 17:42:28 +0800
comments: true
categories: 
---

###工厂模式（简单工厂）、获取一个`抽象接口`所有实现类的实例

- `一个抽象接口`
- 多个实现类
- 工厂提供所有实现类的对象

重点是只有 `一个抽象接口`

***

###抽象接口（标准）的定义

```objc
#import <Foundation/Foundation.h>

@protocol Animal <NSObject>

- (NSString *)name;

- (NSString *)type;

- (void)cry;

@end
```

###接口实现类一、Dog

```objc
#import <Foundation/Foundation.h>

#import "Animal.h"

@interface Dog : NSObject <Animal>

@end
```

```objc
#import "Dog.h"

@implementation Dog

- (NSString *)name {
    return @"我是狗";
}

- (NSString *)type {
    return @"我是哺乳动物";
}

- (void)cry {
    NSLog(@"狗在叫");
}

@end
```

###接口实现类二、Cat

```objc
#import <Foundation/Foundation.h>

#import "Animal.h"

@interface Cat : NSObject <Animal>

@end
```

```objc
#import "Cat.h"

@implementation Cat

- (NSString *)name {
    return @"我是猫";
}

- (NSString *)type {
    return @"我是哺乳动物";
}

- (void)cry {
    NSLog(@"猫在叫");
}

@end
```

###工厂提供抽象接口所有实现版本的对象

```objc
#import <Foundation/Foundation.h>

//向外暴露工厂实例的抽象接口
#import "Animal.h"

@interface Factory : NSObject

+ (id<Animal>)animal;

@end
```

```objc
#import "Factory.h"

//管理所有的具体实现类
#import "Dog.h"
#import "Cat.h"

@implementation Factory

+ (id<Animal>)animal {
    
    //1. 返回动物1、猫
    return [Dog new];
    
    //2. 返回动物1、狗
//    return [Cat new];
}

@end
```