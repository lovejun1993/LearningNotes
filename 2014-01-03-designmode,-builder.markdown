---
layout: post
title: "DesignMode、Builder"
date: 2014-01-03 18:49:55 +0800
comments: true
categories: 
---


###建造者、向外屏蔽内部复杂的流程步骤

- 存在很多固定的步骤话操作
- 且不希望外界知道内部这么多的步骤

****

###对要构造的房子这类事物的抽象

```objc
#import <Foundation/Foundation.h>

/**
 *  房子具备的行为
 */
@protocol House <NSObject>

- (void)养狗;//能够养狗

- (void)做饭;//能够做饭

- (void)睡觉;//能够睡觉

@end
```

###具体的一种房子

```objc
#import <Foundation/Foundation.h>

//实现房子接口
#import "House.h"

@interface WealthHouse : NSObject <House>

@end
```

```objc
#import "WealthHouse.h"

@implementation WealthHouse

- (void)养狗 {
    //养哈士奇
}

- (void)做饭 {
    //做四川菜
}

- (void)睡觉 {
    //躺着睡
}

@end
```

###对建造一个房子的所有步骤抽象出一个统一的模板

```objc
#import <Foundation/Foundation.h>

/**
 *  建造房子的所有步骤抽象
 */
@protocol HouseBuidler <NSObject>

//第一步
- (void)挖地基;

//第二步
- (void)浇筑地梁;

//第三步
- (void)砌筑基本结构;

//第四步
- (void)填土砸夯;

//第五步
- (void)封顶;

@end
```

###一个具体的房子建造者

```objc
#import <Foundation/Foundation.h>
#import "House.h"

/**
 *  具体的房子建造者
 */
@interface HouseBuider : NSObject 

//最终建造完毕的房子
- (id<House>)result;

@end
```

```objc
#import "HouseBuider.h"

//实现房子建造者接口
#import "Buidler.h"

//房子实现类1
#import "WealthHouse.h"

// .m中内部实现建造步骤
@interface HouseBuider () <Buidler>

/**
 *  一个Builder对象，完成一个House对象，的所有构造过程
 */
@property (nonatomic, strong) id<House> house;

@end

@implementation HouseBuider

#pragma mark - 具体的建造步骤

- (void)挖地基 {
//    完成 _house 玩地基...
}

- (void)浇筑地梁 {
//    完成 _house 浇筑地梁...
}

- (void)砌筑基本结构 {
//    完成 _house 砌筑基本结构...
}

- (void)填土砸夯 {
//    完成 _house 填土砸夯...
}

- (void)封顶 {
//    完成 _house 封顶...
}

#pragma mark - 向外提供的统一建造入口

- (id<House>)result {
    
    //1. 创造一个具体房子
    _house = [WealthHouse new];
    
    //2. 完成房子的所有构造步骤
    [self 挖地基];
    [self 浇筑地梁];
    [self 砌筑基本结构];
    [self 填土砸夯];
    [self 封顶];
    
    //3. 返回构造完毕的房子
    return _house;
}

@end
```

优化点是可以通过更灵活的方式来拿到`其他的House接口实现类对象`来建造.
