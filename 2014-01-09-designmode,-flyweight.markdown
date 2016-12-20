---
layout: post
title: "DesignMode、Flyweight"
date: 2014-01-09 10:28:07 +0800
comments: true
categories: 
---

###享元模式、我理解就类似`缓存`优化，先取缓存，没有就创建新的，再存入缓存.

- 简单 享元
- 复合 享元

***

###抽象享元接口定义

```objc
/**
 *  抽象享元接口
 */
@protocol Flyweight <NSObject>

/**
 *  功能函数定义
 */
- (void)operation:(NSString *)state;

@end
```

###具体享元实现

```objc
#import <Foundation/Foundation.h>

//实现抽象享元接口
#import "Flyweight.h"

/**
 *  一个具体享元
 */
@interface ConcreteFlyweight : NSObject <Flyweight>

/**
 *  使用一个唯一表示实例化
 */
- (instancetype)initWithIdentifier:(NSString *)identifier;

@end
```

```objc
#import "ConcreteFlyweight.h"

@interface ConcreteFlyweight ()

@property (nonatomic, copy) NSString *identifier;

@end

@implementation ConcreteFlyweight

- (instancetype)initWithIdentifier:(NSString *)identifier {
    self = [super init];
    if (self) {
        _identifier = identifier;
    }
    return self;
}

- (void)operation:(NSString *)state {
    NSLog(@"外部state = %@\n", state);
    NSLog(@"内部state = %@\n", _identifier);
}

@end
```

###所有具体享元对象的工厂

```objc
//
//  FlyWeightFactory.h
//  demos

#import <Foundation/Foundation.h>

//享元抽象接口
#import "Flyweight.h"

/**
 *  所有具体享元的工厂
 */
@interface FlyWeightFactory : NSObject

/**
 *  单例工厂
 */
+ (instancetype)sharedInstacne;

/**
 *  传入唯一标识查询享元
 */
- (id<Flyweight>)concreteFlyWeightWithIdentifier:(NSString *)identifer;

@end
```

```objc
//
//  FlyWeightFactory.m
//  demos
//
#import "FlyWeightFactory.h"

#import "ConcreteFlyweight.h"

@interface FlyWeightFactory ()

@property (nonatomic, strong) NSMutableDictionary *mapping;

@end

@implementation FlyWeightFactory

+ (instancetype)sharedInstacne {
    static FlyWeightFactory *factory = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        factory = [[FlyWeightFactory alloc] init];
    });
    return factory;
}

- (NSMutableDictionary *)mapping {
    if (!_mapping) {
        _mapping = [[NSMutableDictionary alloc] init];
    }
    return _mapping;
}

- (id<Flyweight>)concreteFlyWeightWithIdentifier:(NSString *)identifer
{
    id<Flyweight> flyweight = nil;
    
    //1. 首先从缓存查询
    flyweight = [self.mapping objectForKey:identifer];
    
    //2. 如果缓存不存在，就创建一个新的，并存入缓存
    if (!flyweight) {
        
        //2.1
        flyweight = [[ConcreteFlyweight alloc] initWithIdentifier:identifer];
        
        //2.2
        [self.mapping setObject:flyweight forKey:identifer];
    }
    
    return flyweight;
}

@end
```

###demo示例

```objc
#import "FlyweightTest.h"

//只导入 享元工厂
#import "FlyWeightFactory.h"

@implementation FlyweightTest

- (void)test {
    
    //1. 从享元工厂查询一个标识对应的享元对象
    id<Flyweight> flyweight = [[FlyWeightFactory sharedInstacne] concreteFlyWeightWithIdentifier:@"identifier"];
    
    //2. 调用接口定义的方法
    [flyweight operation:@"hahahah"];
}

@end
```

****

###复合享元、让一些简单的享元对象`组合`在一起

####添加复合享元的实现，复杂享元不需要存储到缓存，所以不需要identifier.

```objc
//
//  ConcreteCompositeFlyweight.h
//  demos
//
#import <Foundation/Foundation.h>

//实现抽象享元接口
#import "Flyweight.h"

@interface ConcreteCompositeFlyweight : NSObject <Flyweight>

/**
 *  继续可以添加 子享元对象
 */
- (void)addFlayweight:(id<Flyweight>)flyweight
       WithIdentifier:(NSString *)identifier;


@end
```

```objc
//
//  ConcreteCompositeFlyweight.m
//  demos
//
#import "ConcreteCompositeFlyweight.h"

@interface ConcreteCompositeFlyweight ()

@property (nonatomic, strong) NSMutableDictionary *mapping;

@end

@implementation ConcreteCompositeFlyweight

- (NSMutableDictionary *)mapping {
    if (!_mapping) {
        _mapping = [[NSMutableDictionary alloc] init];
    }
    return _mapping;
}

- (void)addFlayweight:(id<Flyweight>)flyweight
       WithIdentifier:(NSString *)identifier
{
    //保存到字典中
    [self.mapping setObject:flyweight forKey:identifier];
}

- (void)operation:(NSString *)state {
    
    //遍历保存的所有享元对象
    [self.mapping enumerateKeysAndObjectsUsingBlock:^(NSString *  _Nonnull key,
                                                      id<Flyweight>  _Nonnull obj,
                                                      BOOL * _Nonnull stop)
    {
        //调用每一个子享元对象
        [obj operation:state];
    }];
}

@end
```

###修改享元工厂，添加创建复合享元的接口方法

```objc
//
//  FlyWeightFactory.h
//  demos
//
#import <Foundation/Foundation.h>

//享元抽象接口
#import "Flyweight.h"

/**
 *  所有具体享元的工厂
 */
@interface FlyWeightFactory : NSObject

/**
 *  单例工厂
 */
+ (instancetype)sharedInstacne;

/**
 *  获取简单享元
 */
- (id<Flyweight>)concreteFlyWeightWithIdentifier:(NSString *)identifer;

/**
 *  获取复杂享元
 */
- (id<Flyweight>)compositeFlyWeightWithIdentifier:(NSArray<NSString *> *)identifers;

@end
```

```objc
//
//  FlyWeightFactory.m
//  demos
//

#import "FlyWeightFactory.h"

//简单享元实现
#import "ConcreteFlyweight.h"

//复杂享元实现
#import "ConcreteCompositeFlyweight.h"

@interface FlyWeightFactory ()

@property (nonatomic, strong) NSMutableDictionary *mapping;

@end

@implementation FlyWeightFactory

+ (instancetype)sharedInstacne {
    static FlyWeightFactory *factory = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        factory = [[FlyWeightFactory alloc] init];
    });
    return factory;
}

- (NSMutableDictionary *)mapping {
    if (!_mapping) {
        _mapping = [[NSMutableDictionary alloc] init];
    }
    return _mapping;
}

- (id<Flyweight>)concreteFlyWeightWithIdentifier:(NSString *)identifer
{
    id<Flyweight> flyweight = nil;
    
    //1. 首先从缓存查询
    flyweight = [self.mapping objectForKey:identifer];
    
    //2. 如果缓存不存在，就创建一个新的，并存入缓存
    if (!flyweight) {
        
        //2.1
        flyweight = [[ConcreteFlyweight alloc] initWithIdentifier:identifer];
        
        //2.2
        [self.mapping setObject:flyweight forKey:identifer];
    }
    
    return flyweight;
}

- (id<Flyweight>)compositeFlyWeightWithIdentifier:(NSArray<NSString *> *)identifers
{
    //1. 创建一个复杂享元对象
    ConcreteCompositeFlyweight *compositeFlyweight = [ConcreteCompositeFlyweight new];
    
    //2. 一次创建（查询）子享元对象
    id<Flyweight> flyweight = nil;
    for (NSString *identifier in identifers) {
        
        //2.1 <最重要>: 查询循环获取简单享元对象
        flyweight = [self concreteFlyWeightWithIdentifier:identifier];
        
        //2.2 注册到 复合享元中
        [compositeFlyweight addFlayweight:flyweight
                           WithIdentifier:identifier];
    }
    
    //3.
    return compositeFlyweight;
}

@end
```

###demo示例

```objc
#import "FlyweightTest.h"

//只导入 享元工厂
#import "FlyWeightFactory.h"

@implementation FlyweightTest

- (void)test {
    
    //1. 定义组合的简单享元的标识
    NSArray<NSString *> *identifiers = @[@"identifier1", @"identifier2", @"identifier3"];
    
    //2. 从享元工厂获取复杂享元
    id<Flyweight> flyweight = [[FlyWeightFactory sharedInstacne] compositeFlyWeightWithIdentifier:identifiers];
    
    //3. 调用复合享元
    [flyweight operation:@"hahahah"];
}

@end
```