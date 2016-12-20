---
layout: post
title: "DesignMode、Bridge"
date: 2014-01-06 21:20:22 +0800
comments: true
categories: 
---

###桥接模式、`组合`不同抽象类型的具体实现类的对象，个人觉得和`组合模式`和`策略模式`差不多，就是使用抽象类型来解耦具体类的关联.

***

###模拟 PC组合CPU ，使用 `桥接模式` 实现.

***

###CPU类型的抽象

```objc
#import <Foundation/Foundation.h>

/**
 *  CPU类型的抽象
 */
@protocol CpuAbility <NSObject>

/** CPU的效率 */
- (NSString *)abilityCpu;

@end
```

###ADM CPU

```objc
#import <Foundation/Foundation.h>

//实现CPU剪口
#import "CpuAbility.h"

@interface ADMCpu : NSObject <CpuAbility>

@end
```

```objc
#import "ADMCpu.h"

@implementation ADMCpu

- (NSString *)abilityCpu {
    return @"ADM CPU 跑分 999分";
}

@end
```

###Intel CPU

```objc
#import <Foundation/Foundation.h>

//实现CPU剪口
#import "CpuAbility.h"

@interface IntelCpu : NSObject <CpuAbility>

@end
```

```objc
#import "IntelCpu.h"

@implementation IntelCpu

- (NSString *)abilityCpu {
    return @"Intel CPU 跑分 99999999分";
}

@end
```

###抽象PC类，提供基础性公共代码

```objc
#import <Foundation/Foundation.h>

//PC包含一个CPU
#import "CpuAbility.h"

/**
 *  PC的抽象模板类
 */
@interface AbstractComputer : NSObject

/**
 *  组合一个CPU对象
 */
@property (nonatomic, strong) id<CpuAbility> cpu;

/**
 *  需要一个CPU对象
 */
- (instancetype)initWithCPU:(id<CpuAbility>)cpu;

/**
 *  CPU跑分
 */
- (void)checkPcAbility;

@end
```

```objc
#import "AbstractComputer.h"

@implementation AbstractComputer

- (instancetype)initWithCPU:(id<CpuAbility>)cpu {
    self = [super init];
    if (self) {
        _cpu = cpu;
    }
    return self;
}

- (void)checkPcAbility {//子类实现}

@end
```

###LenevoComputer

```objc
#import "AbstractComputer.h"

/**
 *  联想PC
 */
@interface LenevoComputer : AbstractComputer

@end
```

```objc
#import "LenevoComputer.h"

@implementation LenevoComputer

////////////////////////////////////////////////////////////
//////重写抽象方法
////////////////////////////////////////////////////////////

- (void)checkPcAbility {
    NSLog(@"Lenevo Computer CPU 跑分: %@", [self.cpu abilityCpu]);
}

@end
```

###ISWComputer

```objc
#import "AbstractComputer.h"

/**
 *  IBM PC
 */
@interface ISWComputer : AbstractComputer

@end
```

```objc
#import "ISWComputer.h"

@implementation ISWComputer

////////////////////////////////////////////////////////////
//////重写抽象方法
////////////////////////////////////////////////////////////

- (void)checkPcAbility {
    NSLog(@"IBM Computer CPU跑分: %@", [self.cpu abilityCpu]);
}

@end
```

###demo示例

```objc
#import "BridgeTest.h"

//所有PC
#import "LenevoComputer.h"
#import "ISWComputer.h"

//所有CPU
#import "ADMCpu.h"
#import "IntelCpu.h"

@implementation BridgeTest

- (void)test {
    
////////////////////////////////////////////////////////////
//////Intel CPU + Lenevo Computer
////////////////////////////////////////////////////////////
    id<CpuAbility> intelCPU = [IntelCpu new];
    LenevoComputer *pc1 = [[LenevoComputer alloc] initWithCPU:intelCPU];
    
////////////////////////////////////////////////////////////
//////ADM CPU + ISW Computer
////////////////////////////////////////////////////////////
    id<CpuAbility> admCPU = [ADMCpu new];
    ISWComputer *pc2 = [[ISWComputer alloc] initWithCPU:admCPU];
    
////////////////////////////////////////////////////////////
//////PC1 与 PC2 各自的跑分测试
////////////////////////////////////////////////////////////
    [pc1 checkPcAbility];
    [pc2 checkPcAbility];
}

@end
```