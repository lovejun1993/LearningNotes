---
layout: post
title: "DesignMode、Decorator"
date: 2014-01-07 23:01:46 +0800
comments: true
categories: 
---

###装饰模式、在不修改原始类代码的情况下，动态的给`对象`添加一些额外的功能

- 适配器模式，针对`类方法`和`对象方法`都可以进行扩展
- 装饰器模式，只针对`对象方法`进行扩展，又有点类似`代理模式`

***

如下举个简单的例子，假设Manager项目管理者本身只有管理项目的职责，但是需要扩展写代码的职责，但是又不能够修改Manager类的代码。

那么可以使用装饰器模式，来给一个`Manager类对象`来扩展具备写代码的职责.

> 只针对对象方法，类方法不行.

###码农的抽象

```objc
#import <Foundation/Foundation.h>

/**
 *  具备写代码能力的抽象
 */
@protocol Coder <NSObject>

/** 写代码 */
- (void)coding;

@end
```

###一个具体的码农

```objc
#import <Foundation/Foundation.h>

#import "Coder.h"

/**
 *  一个具体的码农CoderA
 */
@interface CoderA : NSObject <Coder>

@end
```

```objc
#import "CoderA.h"

@implementation CoderA

- (void)coding {
    NSLog(@"我是码农，正在写代码...");
}

@end
```

现在将给 `码农的一个具体类对象` 添加额外的 `项目管理的一个具体类对象` 上去.

***

###项目管理能力的抽象

```objc
#import <Foundation/Foundation.h>

/**
 *  项目管理者的抽象
 */
@protocol ProjectManager <NSObject>

/** 项目分析 */
- (void)projectAnalyze;

/** 项目设计 */
- (void)projectDesign;

/** 项目上线 */
- (void)projectOnline;

@end
```

###项目管理能力的默认基础实现（抽象类）

```objc
#import <Foundation/Foundation.h>

//能够 项目管理的人
#import "ProjectManager.h"

/**
 *  能够项目管理的抽象人
 */
@interface AbstractProjectManager : NSObject <ProjectManager>


@end
```

```objc
#import "AbstractProjectManager.h"

@implementation AbstractProjectManager

// 下面是抽象方法的默认实现

- (void)projectAnalyze {
    NSLog(@"项目分析");
}

- (void)projectDesign {
    NSLog(@"项目设计");
}

- (void)projectOnline {
    NSLog(@"项目上线");
}

@end
```

###项目管理者Manager对象来包装一个传入的码农Coder对象

```objc
#import <Foundation/Foundation.h>

//继承得到项目能力的基础实现
#import "AbstractProjectManager.h"

//能够 写代码的人
#import "Coder.h"

/**
 *  Manager对象包装一个Coder对象
 *  Manager对象实现Coder接口，具备码农写代码的职责
 */
@interface CoderDecorator : AbstractProjectManager <Coder>

/**
 *  要添加项目管理能力的Coder对象
 */
@property (nonatomic, strong) id<Coder> target;

/**
 *  传入需要包装的Coder对象
 */
- (instancetype)initWithCoder:(id<Coder>)coder;

@end
```

```objc
#import "CoderDecorator.h"

@implementation CoderDecorator

- (instancetype)initWithCoder:(id<Coder>)coder {
    self = [super init];
    if (self) {
        _target = coder;
    }
    return self;
}

// Manager对象的写代码职责，实际上由传入的Coder对象最终来完成
- (void)coding {
    
    //1. 项目分析
    [self projectAnalyze];
    
    //2. 项目设计
    [self projectDesign];
    
    //3. 调用原始Coder对象写代码
    [_target coding];
    
    //4. 项目上线
    [self projectOnline];
}


@end
```

###demo示例

```objc
#import "DecoratorTest.h"

//原始码农
#import "CoderA.h"

//码农包装器，添加项目管理的能力
#import "CoderDecorator.h"

@implementation DecoratorTest

- (void)test {
    
    //1. 创建一个原始码农
    id<Coder> coder = [CoderA new];
    
    //2. 创建原始码农的 包装器
    coder = [[CoderDecorator alloc] initWithCoder:coder];
    
    //3. 调用包装器的coding方法，内部附加其他方法
    [coder coding];
}

@end
```

***

这样的话，就不用在Manager类中去耦合很多的关于Coder的实现细节，通过包装一个Coder对象的方式，在运行时动态的完成Manager对象的方法扩展.

如果是需要对`类方法`进行扩展，使用`适配器模式`.