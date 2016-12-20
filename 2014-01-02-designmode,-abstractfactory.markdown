---
layout: post
title: "DesignMode、AbstractFactory"
date: 2014-01-02 17:49:47 +0800
comments: true
categories: 
---

###抽象工厂、简单工厂的升级版，解决简单工厂只针对`一个抽象接口`

- 同时有多个 `抽象接口`
- 每个抽象接口，有`多种实现情况`

****

###抽象工厂模式结构 >>> 延伸至类簇模式

- 两种抽象产品标准
	- ProductA
	- ProductB

- 抽象工厂
	- 生产ProductA类型的具体产品
	- 生产ProductB类型的具体产品


- 具体工厂A
	- 生产ProductA类型的具体产品	
	- 生产ProductB类型的具体产品

- 具体工厂B
	- 生产ProductA类型的具体产品	
	- 生产ProductB类型的具体产品

- 统一工厂C作为A工厂与B工厂的入口（类簇类）
	- **从而可以避免 A工厂 与 B工厂 直接向外暴露**

***

###产品A的标准

```objc
#import <Foundation/Foundation.h>

@protocol ProductA <NSObject>

- (void)name;

- (void)type;

@end
```

***

###产品B的标准

```objc
#import <Foundation/Foundation.h>

@protocol ProductB <NSObject>

- (void)name;

- (void)type;

@end
```

###产品A标准的 A1版本实现

```objc
#import <Foundation/Foundation.h>

//实现商品接口，成为一个商品
#import "ProductA.h"

@interface ProductA1 : NSObject <ProductA>

@end
```

```objc
#import "ProductA1.h"

@implementation ProductA1

- (void)name {
    NSLog(@"我是ProductA标准一实现版本1");
}

- (void)type {
    NSLog(@"我是ProductA标准一实现版本1");
}

@end
```

###产品A标准的 A2版本实现

```objc
#import <Foundation/Foundation.h>

//实现商品接口，成为一个商品
#import "ProductA.h"

@interface ProductA2 : NSObject <ProductA>

@end
```

```objc
#import "ProductA2.h"

@implementation ProductA2

- (void)name {
    NSLog(@"我是ProductA标准一实现版本2");
}

- (void)type {
    NSLog(@"我是ProductA标准一实现版本2");
}

@end
```

###产品B标准的 B1版本实现

```objc
#import <Foundation/Foundation.h>

//实现商品接口，成为一个商品
#import "ProductB.h"

@interface ProductB1 : NSObject <ProductB>

@end
```

```objc
#import "ProductB1.h"

@implementation ProductB1

- (void)name {
    NSLog(@"我是ProductB标准一实现版本1");
}

- (void)type {
    NSLog(@"我是ProductB标准一实现版本1");
}

@end
```

###产品B标准的 B2版本实现

```objc
#import <Foundation/Foundation.h>

//实现商品接口，成为一个商品
#import "ProductB.h"

@interface ProductB2 : NSObject <ProductB>

@end
```

```objc
#import "ProductB2.h"

@implementation ProductB2

- (void)name {
    NSLog(@"我是ProductB标准一实现版本2");
}

- (void)type {
    NSLog(@"我是ProductB标准一实现版本2");
}

@end
```

###工厂抽象标准，定义生产 `产品A标准与产品B标准` 的商品

```objc
#import <Foundation/Foundation.h>

//标准A类产品
#import "ProductA.h"

//标准B类产品
#import "ProductB.h"

@protocol AbstractFactory <NSObject>

/**
 *  生产 标准A类 产品
 */
+ (id<ProductA>)productA;

/**
 *  生产 标准B类 产品
 */
+ (id<ProductB>)productB;

@end
```

###生产 A1版本与B1版本 产品簇 的 `具体工厂1`

```objc
#import <Foundation/Foundation.h>

//实现工厂，接口成为一个工厂，只生产 A1版本与B1版本 的产品
#import "AbstractFactory.h"

/**
 *  生产:
 *      1. ProductA1 具体类
 *      2. ProductB1 具体类
 */
@interface Product1Factory : NSObject <AbstractFactory>

+ (id<ProductA>)productA;

+ (id<ProductB>)productB;

@end
```

```objc
#import "Product1Factory.h"

//A1与B1
#import "ProductA1.h"
#import "ProductB1.h"

@implementation Product1Factory

+ (id<ProductA>)productA {

	//返回A1版本
    return [ProductA1 new];
}

+ (id<ProductB>)productB {

	//返回B1版本
    return [ProductB1 new];
}

@end
```

###生产 A2版本与B2版本 产品簇 的 `具体工厂2`

```objc
#import <Foundation/Foundation.h>

//实现工厂，接口成为一个工厂，只生产 A1版本与B1版本 的产品
#import "AbstractFactory.h"

/**
 *  生产:
 *      1. ProductA2 具体类
 *      2. ProductB2 具体类
 */
@interface Product2Factory : NSObject <AbstractFactory>

+ (id<ProductA>)productA;

+ (id<ProductB>)productB;

@end
```

```objc
#import "Product2Factory.h"

//A2 与 B2
#import "ProductA2.h"
#import "ProductB2.h"

@implementation Product2Factory

+ (id<ProductA>)productA {
    return [ProductA2 new];
}

+ (id<ProductB>)productB {
    return [ProductB2 new];
}

@end
```

###向外只提供一个统一的工厂类，假设工厂分为便宜造价和昂贵造价之分，那么生成的产品也就有便宜与昂贵之分.

```objc
#import "AbstractFactory.h"

@interface FactoryClouster : NSObject <AbstractFactory>

@end
```

```objc
#import "FactoryClouster.h"

//内部管理各种具体工厂
#import "Product1Factory.h"
#import "Product2Factory.h"

@implementation FactoryClouster

+ (id<ProductA>)productA {
    if (选择便宜) {
        return [Product1Factory productA];
    } else {
        return [Product2Factory productA];
    }
}

+ (id<ProductB>)productB {
    if (选择便宜) {
        return [Product1Factory productB];
    } else {
        return [Product2Factory productB];
    }
}

@end
```

***

###小结:

- 有多种不同的产品标准共存
- 而每一种产品标准又有不同版本实现