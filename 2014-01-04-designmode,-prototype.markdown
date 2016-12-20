---
layout: post
title: "DesignMode、Prototype"
date: 2014-01-04 19:26:22 +0800
comments: true
categories: 
---

###原型模式、通过对一个对象进行`复制`后，得到一个相同类型的对象，而不需要关心: `被复制的对象类型或实现的接口`

原型模式分为两种:

- 简单形式的复制
- 登记形式的复制

***

###简单形式

- 对象复制的功能抽象

```objc
#import <Foundation/Foundation.h>

/**
 *  对象复制功能的抽象
 */
@protocol Clonable <NSObject>

- (id)clone;

@end
```

- 让类完成对象复制的接口方法

```objc
#import <Foundation/Foundation.h>

//实现可复制接口
#import "Clonable.h"

@interface Car : NSObject <Clonable>

@end
```

```objc
#import "Car.h"

@implementation Car

- (id)clone {
    return [Car new];//其实这里直接可以使用NSCopying协议实现浅拷贝对象
}

@end
```

***

###登记形式，就是在前面基础上再多加一个 `管理器` 

```objc
#import <Foundation/Foundation.h>

/**
 *  传入登记的对象类，必须实现的接口
 */
#import "Clonable.h"

/**
 *  使用唯一标识  管理  原型对象（复制出来的对象）
 */
@interface CarPrototypeManager : NSObject

+ (instancetype)sharedInstance;

/**
 *  使用一个标识符来保存一个原型对象
 */
- (void)setPrototype:(id<Clonable>)prototype
          Identifier:(NSString *)identy;

/**
 *  获取一个原型对象
 */
- (id<Clonable>)getPrototypeWithIdentifier:(NSString *)identy;

/**
 *  移除一个原型对象
 */
- (void)removePrototypeWithIdentifier:(NSString *)identy;

@end
```

```objc
#import "CarPrototypeManager.h"

@interface CarPrototypeManager ()

@property (nonatomic, strong) NSMutableDictionary *mappigs;

@end

@implementation CarPrototypeManager

+ (instancetype)sharedInstance {
    static CarPrototypeManager *manager = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        manager = [[CarPrototypeManager alloc] init];
    });
    return manager;
}

- (NSMutableDictionary *)mappigs {
    if (!_mappigs) {
        _mappigs = [[NSMutableDictionary alloc] init];
    }
    return _mappigs;
}

- (void)setPrototype:(id<Clonable>)prototype Identifier:(NSString *)identy {
    [self.mappigs setObject:prototype forKey:identy];
}

- (id<Clonable>)getPrototypeWithIdentifier:(NSString *)identy {
    return [self.mappigs objectForKey:identy];
}

- (void)removePrototypeWithIdentifier:(NSString *)identy {
    [self.mappigs removeObjectForKey:identy];
}

@end
```

###使用demo

```objc
#import "PrototypeTest.h"

//导入原型登记管理容器
#import "CarPrototypeManager.h"

#import "Car.h"

@implementation PrototypeTest

- (void)test {
    
    //1.
    CarPrototypeManager *manager = [CarPrototypeManager sharedInstance];
    
    //2.
    Car *car = [Car new];
    
    //3. 登记原型对象
    Car *clone = [car clone];
    [manager setPrototype:clone Identifier:@"prototype1"];
    
    //4. 获取登记的原型对象
    clone = [manager getPrototypeWithIdentifier:@"prototype1"];
    
    //5. 移除登记的原型对象
    [manager removePrototypeWithIdentifier:@"prototype1"];
}

@end
```