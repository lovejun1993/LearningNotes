---
layout: post
title: "Class Cluster 类簇"
date: 2016-05-09 21:31:12 +0800
comments: true
categories: 
---

### 相信搞过iOS的都知道类簇的概念，比如 NSNumber 、 NSString 、 NSArray 等等都是类簇。那到底类簇是什么意思？用来干嘛？

类簇其实是一种设计模式，估计知道的人很少，使用的人就更少了。名字叫做 >>> 抽象工厂。

比如，最常用的就是NSNumber，我经常使用NSNumber的如下方法进行类型转换:

```objc
@interface NSNumber (NSNumberCreation)

+ (NSNumber *)numberWithChar:(char)value;
+ (NSNumber *)numberWithUnsignedChar:(unsigned char)value;
+ (NSNumber *)numberWithShort:(short)value;
+ (NSNumber *)numberWithUnsignedShort:(unsigned short)value;
+ (NSNumber *)numberWithInt:(int)value;
+ (NSNumber *)numberWithUnsignedInt:(unsigned int)value;
+ (NSNumber *)numberWithLong:(long)value;
+ (NSNumber *)numberWithUnsignedLong:(unsigned long)value;
+ (NSNumber *)numberWithLongLong:(long long)value;
+ (NSNumber *)numberWithUnsignedLongLong:(unsigned long long)value;
+ (NSNumber *)numberWithFloat:(float)value;
+ (NSNumber *)numberWithDouble:(double)value;
+ (NSNumber *)numberWithBool:(BOOL)value;
+ (NSNumber *)numberWithInteger:(NSInteger)value NS_AVAILABLE(10_5, 2_0);
+ (NSNumber *)numberWithUnsignedInteger:(NSUInteger)value NS_AVAILABLE(10_5, 2_0);

@end
```

其实NSNumber就是一个类簇类，内部管理如上这些基本数据类型。但是对外（开发者）使用时，可以只需要使用一个入口类NSNumber就可以了，那么这正是类簇这个设计模式的核心所在。

刚好在看AFNetworking基于NSURLSession实现中遇到类簇的问题，之前在Effect OC-2.0书上也看到过，只是觉得没什么用就没怎么仔细看，原来作用大着了。

在AFURLSessionManager.m中，有一个私有分类`_AFURLSessionTaskSwizzling`，用来swziizle系统NSURLSessionTask对象的`state`属性，然后进行Task的对象改变通知告诉框架调用者进行回调处理。

```objc
@interface _AFURLSessionTaskSwizzling : NSObject

@end

@implementation _AFURLSessionTaskSwizzling

+ (void)load {
    /**
     WARNING: Trouble Ahead
     https://github.com/AFNetworking/AFNetworking/pull/2702
     */

    if (NSClassFromString(@"NSURLSessionTask")) {
        /**
         iOS 7 and iOS 8 differ in NSURLSessionTask implementation, which makes the next bit of code a bit tricky.
         Many Unit Tests have been built to validate as much of this behavior has possible.
         Here is what we know:
            - NSURLSessionTasks are implemented with class clusters, meaning the class you request from the API isn't actually the type of class you will get back.
            - Simply referencing `[NSURLSessionTask class]` will not work. You need to ask an `NSURLSession` to actually create an object, and grab the class from there.
            - On iOS 7, `localDataTask` is a `__NSCFLocalDataTask`, which inherits from `__NSCFLocalSessionTask`, which inherits from `__NSCFURLSessionTask`.
            - On iOS 8, `localDataTask` is a `__NSCFLocalDataTask`, which inherits from `__NSCFLocalSessionTask`, which inherits from `NSURLSessionTask`.
            - On iOS 7, `__NSCFLocalSessionTask` and `__NSCFURLSessionTask` are the only two classes that have their own implementations of `resume` and `suspend`, and `__NSCFLocalSessionTask` DOES NOT CALL SUPER. This means both classes need to be swizzled.
            - On iOS 8, `NSURLSessionTask` is the only class that implements `resume` and `suspend`. This means this is the only class that needs to be swizzled.
            - Because `NSURLSessionTask` is not involved in the class hierarchy for every version of iOS, its easier to add the swizzled methods to a dummy class and manage them there.
        
         Some Assumptions:
            - No implementations of `resume` or `suspend` call super. If this were to change in a future version of iOS, we'd need to handle it.
            - No background task classes override `resume` or `suspend`
         
         The current solution:
            1) Grab an instance of `__NSCFLocalDataTask` by asking an instance of `NSURLSession` for a data task.
            2) Grab a pointer to the original implementation of `af_resume`
            3) Check to see if the current class has an implementation of resume. If so, continue to step 4.
            4) Grab the super class of the current class.
            5) Grab a pointer for the current class to the current implementation of `resume`.
            6) Grab a pointer for the super class to the current implementation of `resume`.
            7) If the current class implementation of `resume` is not equal to the super class implementation of `resume` AND the current implementation of `resume` is not equal to the original implementation of `af_resume`, THEN swizzle the methods
            8) Set the current class to the super class, and repeat steps 3-8
         */
        NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration ephemeralSessionConfiguration];
        NSURLSession * session = [NSURLSession sessionWithConfiguration:configuration];
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wnonnull"
        NSURLSessionDataTask *localDataTask = [session dataTaskWithURL:nil];
#pragma clang diagnostic pop
        IMP originalAFResumeIMP = method_getImplementation(class_getInstanceMethod([self class], @selector(af_resume)));
        Class currentClass = [localDataTask class];
        
        while (class_getInstanceMethod(currentClass, @selector(resume))) {
            Class superClass = [currentClass superclass];
            IMP classResumeIMP = method_getImplementation(class_getInstanceMethod(currentClass, @selector(resume)));
            IMP superclassResumeIMP = method_getImplementation(class_getInstanceMethod(superClass, @selector(resume)));
            if (classResumeIMP != superclassResumeIMP &&
                originalAFResumeIMP != classResumeIMP) {
                [self swizzleResumeAndSuspendMethodForClass:currentClass];
            }
            currentClass = [currentClass superclass];
        }
        
        [localDataTask cancel];
        [session finishTasksAndInvalidate];
    }
}

+ (void)swizzleResumeAndSuspendMethodForClass:(Class)theClass {
    Method afResumeMethod = class_getInstanceMethod(self, @selector(af_resume));
    Method afSuspendMethod = class_getInstanceMethod(self, @selector(af_suspend));

    if (af_addMethod(theClass, @selector(af_resume), afResumeMethod)) {
        af_swizzleSelector(theClass, @selector(resume), @selector(af_resume));
    }

    if (af_addMethod(theClass, @selector(af_suspend), afSuspendMethod)) {
        af_swizzleSelector(theClass, @selector(suspend), @selector(af_suspend));
    }
}

- (NSURLSessionTaskState)state {
    NSAssert(NO, @"State method should never be called in the actual dummy class");
    return NSURLSessionTaskStateCanceling;
}

- (void)af_resume {
    NSAssert([self respondsToSelector:@selector(state)], @"Does not respond to state");
    NSURLSessionTaskState state = [self state];
    [self af_resume];
    
    if (state != NSURLSessionTaskStateRunning) {
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNSURLSessionTaskDidResumeNotification object:self];
    }
}

- (void)af_suspend {
    NSAssert([self respondsToSelector:@selector(state)], @"Does not respond to state");
    NSURLSessionTaskState state = [self state];
    [self af_suspend];
    
    if (state != NSURLSessionTaskStateSuspended) {
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNSURLSessionTaskDidSuspendNotification object:self];
    }
}

@end
```

从上面一大坨的注释中，有一个很大的问题，就是NSURLSessionTask的实现在`iOS7`与`iOS8、iOS8+`下有着很大的差别，所以在swizzle的时候也就需要分别对待。

- (1) NSURLSessionTask 最低必须在 `iOS7` 系统上运行

```objc
if (NSClassFromString(@"NSURLSessionTask")) {
	// execute swizzle NSURLSessionTask ... 
}
```

- (2) 系统提供的一系列NSURLSessionTask、NSURLSessionDataTask、NSURLSessionUploadTask、NSURLSessionDownloadTask、NSURLSessionStreamTask、这些task类统统都是`类簇类`

即这些类并不是最终系统真正使用的真实类型。而要进行swizzle目标类的方法实现，就必须要要找到这个类型的真实类型，继而得到他真实的`objc_class`实例，继而进行SEL的指向交换，才能起作用。

- (3) iOS7与iOS7+系统下，这些Task类最终私有内部类，继承结构有所不同

![](http://i4.buimg.com/4851/ff9ad2b9a4ee550c.png)

On iOS 7，`__NSCFLocalDataTask > __NSCFLocalSessionTask > __NSCFURLSessionTask`

On iOS 8，`__NSCFLocalDataTask > __NSCFLocalSessionTask > NSURLSessionTask`

- (4) iOS7，`__NSCFLocalSessionTask`与`__NSCFURLSessionTask`都实现了`resume`与`suspend`。但是`__NSCFLocalSessionTask`中这两个方法实现，都没有调用`super`，即没有调用`__NSCFLocalSessionTask`的两个方法实现

所以在iOS7时，`__NSCFLocalSessionTask`与`__NSCFURLSessionTask`这两个类，都需要进行一次swizzle。

- (5) iOS8、iOS8+，只有`NSURLSessionTask`实现了`resume`与`suspend`，所以只需要swizzle`NSURLSessionTask`的两个方法实现

- (6) 随便获取一个dataTask对象，然后找到如上情况的Task私有类，进行swizzle两个方法的实现

获取一个真实task对象

```objc
NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration ephemeralSessionConfiguration];
        NSURLSession * session = [NSURLSession sessionWithConfiguration:configuration];
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wnonnull"
        NSURLSessionDataTask *localDataTask = [session dataTaskWithURL:nil];
#pragma clang diagnostic pop
```

找到这个task对象的真实私有的内部类型（`__NSCFLocalDataTask`）

```objc
IMP originalAFResumeIMP = method_getImplementation(class_getInstanceMethod([self class], @selector(af_resume)));
Class currentClass = [localDataTask class];
```

根据iOS7和iOS7+区别，iOS7需要swizzle两个类的实现，iOS8只需要swizzle一个类的实现:

```objc
while (class_getInstanceMethod(currentClass, @selector(resume))) {
	// __NSCFLocalSessionTask、__NSCFURLSessionTask、NSURLSessionTask	
		
    Class superClass = [currentClass superclass];
    IMP classResumeIMP = method_getImplementation(class_getInstanceMethod(currentClass, @selector(resume)));
    IMP superclassResumeIMP = method_getImplementation(class_getInstanceMethod(superClass, @selector(resume)));
    if (classResumeIMP != superclassResumeIMP &&
        originalAFResumeIMP != classResumeIMP) {
        [self swizzleResumeAndSuspendMethodForClass:currentClass];
    }
    currentClass = [currentClass superclass];
}
```

到此为止，AFN中关于类簇类的就这样了。如果知道类簇概念的可能一看就知道，如果不知道类簇概念的童鞋，可能已经晕了。

## 那么有必要先看下抽象工厂模式

- 主要的核心概念:
	- 抽象工厂 : 具体工厂 == 1 : n（一个抽象工厂，多个具体工厂）
	- 不同等级（簇）类型的抽象产品生成

- 以电脑需要CPU、主板为例子画出抽象工厂的结构图
	- 有两种产品: 1) CPU 2) MainBoard
	- 计算机 需要依赖两个产品: 1) CPU接口抽象 2) MainBoard接口抽象
	- 计算机 依赖一个抽象工厂，来生成 1) CPU 2) MainBoard
	- 满足抽象工厂的标准: 1) 能够生成出CPU 2） 能够生成出MainBoard
		- 具体工厂1、`Intel 工厂`
			- 生产 CPU
			- 生产 MainBoard
		- 具体工厂2、`AMD 工厂`
			- 生产 CPU
			- 生产 MainBoard


![d900ff4dc0a2c2f2.png](http://imgchr.com/images/d900ff4dc0a2c2f2.png)

![0b9364c290966bac.png](http://imgchr.com/images/0b9364c290966bac.png)

到这里可以看出`具体工厂`的作用:

- 第一点、绝对是使用一个工厂，来屏蔽如何生产 CPU、MainBoard 的具体过程.
	- 通俗的说就是减轻调用者对 `内部私有类` 的直接依赖.
		- Intel xxx1.0 CPU、Intel xxx2.0 CPU、Intel xxx3.0 CPU ...
		- Intel xxx1.0 主板、Intel xxx2.0 主板、Intel xxx3.0 主板 ...

- 第二点、对于每一种产品，有`不同的工厂（提供商）`来竞争
	- Intel工厂可以生产 CPU、主板
	- AMD工厂也可以生产 CPU、主板
	- 甚至其他XXX工厂也能生产

那么`抽象工厂`了？	上面最终会产生`Intel工厂、AMD工厂、XXX工厂...`，但是突然有一天，就一家牛逼的工厂`XXOO工厂`将这些工厂全部都收购了，但是仍然让这些工厂继续运营。

那么XXOO工厂面临一个问题，所有的外部订单，必须统一经由XXOO工厂，然后分发到对应的`Intel工厂、AMD工厂、XXX工厂...`去完成生产。

这个由`XXOO工厂`统一`分发`到`内部具体工厂`的过程，其实就是抽象工厂模式的核心。


使用代码来模拟下上面的结构:

- CPU接口抽象

```objc
#import <Foundation/Foundation.h>

@protocol CPU <NSObject>

- (void)cpu_work;

@end
```

- MainBoard接口抽象

```objc
#import <Foundation/Foundation.h>

@protocol MainBoard <NSObject>

- (void)mainboard_work;

@end
```

- 抽象工厂类

```objc
#import <Foundation/Foundation.h>

// CPU产品抽象
#import "CPU.h"

// MainBoard产品抽象
#import "MainBoard.h"

/**
 *  抽象工厂类定义
 */
@interface AbstractComputerFactory : NSObject

// 生产CPU
- (id<CPU>)generateCPU;

// 生产MainBoard
- (id<MainBoard>)generateMainBoard;

@end
```

```objc
#import "AbstractComputerFactory.h"

@implementation AbstractComputerFactory

/* 让子类具体工厂完成重写 */
- (id<CPU>)generateCPU {return nil;}
- (id<MainBoard>)generateMainBoard {return nil;}

@end
```

- 具体工厂类1（Intel工厂，AMD工厂）

```objc
#import "AbstractComputerFactory.h"

/**
 *  Intel工厂
 */
@interface IntelFactory : AbstractComputerFactory

@end
```

```objc
#import "IntelFactory.h"

////////////////////////// 内部管理自己的CPU具体产品 ////////////////////////////

// intel 型号1 cpu
@interface __IntelCpu1_1_0 : NSObject <CPU>

@end

@implementation __IntelCpu1_1_0

- (void)cpu_work {
    NSLog(@"Intel cpu work...");
}

@end

// intel 型号2 cpu
@interface __IntelCpu1_2_0 : NSObject <CPU>

@end

@implementation __IntelCpu1_2_0

- (void)cpu_work {
    NSLog(@"Intel cpu work...");
}

@end

//////////////////////// 内部管理自己的MainBoard具体产品 ////////////////////////

// intel 型号1 mainboard
@interface __IntelMainboard_1_1_0 : NSObject <MainBoard>

@end

@implementation __IntelMainboard_1_1_0

- (void)mainboard_work {
    NSLog(@"intel mainboard work...");
}

@end

// intel 型号2 mainboard
@interface __IntelMainboard_1_2_0 : NSObject <MainBoard>

@end

@implementation __IntelMainboard_1_2_0

- (void)mainboard_work {
    NSLog(@"intel mainboard work...");
}

@end

////////////////////////////////// Intel 工厂 ////////////////////////////////
////// 随时可以切换 内部使用的 CPU型号、MainBoard型号

@implementation IntelFactory

- (id<CPU>)generateCPU {
//    return [__IntelCpu1_1_0 new];           // 型号1 CPU
    return [__IntelCpu1_2_0 new];             // 型号2 CPU
}

- (id<MainBoard>)generateMainBoard {
//    return [__IntelMainboard_1_1_0 new];    // 型号1 主板
    return [__IntelMainboard_1_2_0 new];      // 型号2 主板
}

@end
```

- 最后是计算Computer类耦合一个具体工厂

```objc
#import <Foundation/Foundation.h>

@interface Computer : NSObject 

- (void)work;

@end
```

```objc
#import "Computer.h"

// 导入一个具体工厂
#import "IntelFactory.h"

@implementation Computer {
    IntelFactory *_factory;
}

- (void)work {
    
    //1. 获取工厂
    _factory = [IntelFactory new];
    
    //2. 从工厂获取CPU
    id<CPU> cpu = [_factory generateCPU];
    
    //3. 从工厂获取MainBoard
    id<MainBoard> mainBoard = [_factory generateMainBoard];
    
    //4. 操作CPU和MainBoard
    [cpu cpu_work];
    [mainBoard mainboard_work];
}

@end
```
还有一个AMD工厂

```objc
#import "AbstractComputerFactory.h"

@interface AMDFactory : AbstractComputerFactory
@end

........省略
```

那么现在有两个具体工厂类，可以进一步使用一个抽象工厂类，来向外屏蔽直接暴露Intel和AMD这两个具体工厂，也就是值暴露收购Intel和AMD的工厂

```objc
#import "AbstractComputerFactory.h"

@interface XXOOFactory : AbstractComputerFactory

@end
```

```objc
#import "XXOOFactory.h"

//具体工厂类
#import "IntelFactory.h"
#import "AMDFactory.h"

@implementation XXOOFactory {
    AbstractComputerFactory *_concreateFactory;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        if (是否使用Intel工厂生产) {
            _concreateFactory = [IntelFactory new];
        } else {
            _concreateFactory = [AMDFactory new];
        }
    }
    return self;
}

- (id<CPU>)generateCPU {
    return [_concreateFactory generateCPU];
}

- (id<MainBoard>)generateMainBoard {
    return [_concreateFactory generateMainBoard];
}

@end
```

这样的话，外界就不用再关心是由Intel还是AMD加工的，只需要知道是收购之后的XXOO工厂得到即可。

***

###Foundation中的类簇类、NSArray与NSMuatbleArray

首先看下NSArray与NSMuatbleArray的真实类型，即内部私有类叫什么

```objc
Class cls1 = objc_getClass("NSArray");
NSLog(@">>> %@ >>>> %p", cls1, cls1);
    
Class cls2 = NSClassFromString(@"NSArray");
NSLog(@">>> %@ >>> %p", cls2, cls2);
    
Class cls3 = [NSArray class];
NSLog(@">>> %@ >>> %p", cls3, cls3);
    
Class cls4 = objc_getClass("NSMutableArray");
NSLog(@">>> %@ >>> %p", cls4, cls4);
    
Class cls5 = NSClassFromString(@"NSMutableArray");
NSLog(@">>> %@ >>> %p", cls4, cls5);
    
Class cls6 = [NSMutableArray class];
NSLog(@">>> %@ >>> %p", cls6, cls6);
    
id array1 = @[@"111", @"222", @"3333", @"4444"];
    
Class cls7 = [array1 class];
NSLog(@">>> %@ >>> %p", cls7, cls7);
    
Class cls8 = object_getClass(array1);
NSLog(@">>> %@ >>> %p", cls8, cls8);
    
Class cls9 = object_getClass([array1 class]);
NSLog(@">>> %@ >>> %p", cls9, cls9);
    
id array2 = [array1 mutableCopy];
    
Class cls10 = [array2 class];
NSLog(@">>> %@ >>> %p", cls10, cls10);
    
Class cls11 = object_getClass(array2);
NSLog(@">>> %@ >>> %p", cls11, cls11);
    
Class cls12 = object_getClass([array2 class]);
NSLog(@">>> %@ >>> %p", cls12, cls12);
```

运行结果

```
2016-08-21 23:38:38.294 Demos[3872:34836] >>> NSArray >>>> 0x1104c7900
2016-08-21 23:38:38.295 Demos[3872:34836] >>> NSArray >>> 0x1104c7900
2016-08-21 23:38:38.295 Demos[3872:34836] >>> NSArray >>> 0x1104c7900

2016-08-21 23:38:38.295 Demos[3872:34836] >>> NSMutableArray >>> 0x1104c7978
2016-08-21 23:38:38.296 Demos[3872:34836] >>> NSMutableArray >>> 0x1104c7978
2016-08-21 23:38:38.296 Demos[3872:34836] >>> NSMutableArray >>> 0x1104c7978

2016-08-21 23:38:38.296 Demos[3872:34836] >>> __NSArrayI >>> 0x1104c78b0
2016-08-21 23:38:38.296 Demos[3872:34836] >>> __NSArrayI >>> 0x1104c78b0
2016-08-21 23:38:38.296 Demos[3872:34836] >>> __NSArrayI >>> 0x1104c79f0

2016-08-21 23:38:38.296 Demos[3872:34836] >>> __NSArrayM >>> 0x1104c78d8
2016-08-21 23:38:38.296 Demos[3872:34836] >>> __NSArrayM >>> 0x1104c78d8
2016-08-21 23:38:38.296 Demos[3872:34836] >>> __NSArrayM >>> 0x1104c7a18
```

可以看到内部的私有类型是

- `__NSArrayI` >>>> NSArray内部私有版本
- `__NSArrayM` >>>> NSMutableArray内部私有版本

那么从结果输出，我想在对Foundation大部分类型做比较时，需要判断这个类是否是类簇，然后分情况做类型比较

对Foudnation所有集合类进行类型比较的错误情况（集合类基本上都是类簇）

```objc
- (void)compare {
    id array = @[@"111", @"222", @"3333", @"4444"];
    
    // 错误比较
    if ([array class] == [NSArray class]) {
       NSLog(@"属于数组");// 其实永远都不会执行
    }
    
    // 正确比较
    if ([array isKindOfClass:[NSArray class]]) {
        NSLog(@"属于数组");
    }
}
```

但是如果是针对我们自己编写的一些基本类，并非是类簇的情况下，可以采用上述两种方法进行比较类型，可以使用如下这几种都可以

```objc
- (void)compare {

    Person *person = [Person new];
    
    /**
     *  因为一个NSObejct类在运行时对应的objc_class结构体实例是一个 全局存在的单例
     */
    if ([person class] == [Person class]) {
        NSLog(@"属于Person类");
    }
    
    if ([person isKindOfClass:[Person class]]) {
        NSLog(@"属于Person类");
    }
    
    if (object_getClass(person) == [Person class]) {
        NSLog(@"属于Person类");
    }
    
    if (object_getClass(person) == objc_getClass("Person")) {
        NSLog(@"属于Person类");
    }
}
```

但其实关于数组的私有类还有如下几个:

- (1) `__NSPlaceholderArray`
- (2) `__NSArray0`


先看看下`__NSPlaceholderArray`是用来干嘛的，从字面意义看就是一个占位的。然后还有`__NSArray0`干嘛的

```objc
- (void)testNSArrayAlloc {
    
    /**
     *  都是对象，所以使用[obj class]与object_getClass(obj)都是一样
     */
    
    id arr1 = [NSArray alloc];
    id arr2 = [arr1 init];
    id arr3 = [NSMutableArray alloc];
    id arr4 = [arr3 init];
    
}
```

输出结果

```
(__NSPlaceholderArray *) arr1 = 0x00007fab0b401310
(__NSArray0 *) arr2 = 0x00007fab0b501890
(__NSPlaceholderArray *) arr3 = 0x00007fab0b400740
(__NSArrayM *) arr4 = 0x00007fab0b617fd0 @"0 objects"
```

可以看到`[NSArray alloc]` 与 `[NSMutableArray alloc]` 得到的对象类型是`__NSPlaceholderArray`，并不是经常使用的`NSArray`和`NSMutableArray`类型的对象。

> 并且仔细看，这两次`__NSPlaceholderArray`对象的地址是不同的。

但是通过 `[[NSArray alloc] init]` 或 `[[NSMutableArray alloc] init]` 之后就可以得到 `__NSArrayI、__NSArray0` 或 `__NSArrayM` 类型的对象了。

`__NSArray0`其实就是一个不包含任何子数据的不可变版本的数组对象。


也就是说由 `-[__NSPlaceholderArray init]`后，可以得到最终使用的两种类型数组对象:

- 不可变数组、`__NSArrayI、__NSArray0`类型的对象
- 可变数组、`__NSArrayM`类型的对象

那`-[__NSPlaceholderArray init]`方法的实现，恰好就相当于之前实现抽象工厂类`-[XXOOFactory init]`方法实现中所做的内部私有类版本判断的逻辑基本是一直的。

```objc
#import "XXOOFactory.h"

//具体工厂类
#import "IntelFactory.h"
#import "AMDFactory.h"

@implementation XXOOFactory {
    AbstractComputerFactory *_concreateFactory;
}

/**
 *	 在init中，判断内部使用哪一个版本的私有实现类对象
 */
- (instancetype)init {
    self = [super init];
    if (self) {
        if (是否使用Intel工厂生产) {
            _concreateFactory = [IntelFactory new];
        } else {
            _concreateFactory = [AMDFactory new];
        }
    }
    return self;
}

- (id<CPU>)generateCPU {
    return [_concreateFactory generateCPU];
}

- (id<MainBoard>)generateMainBoard {
    return [_concreateFactory generateMainBoard];
}

@end
```

那`-[__NSPlaceholderArray init]`是如何判断该生产可变数组还是不可变数组了？即如何记录之前到底是`[NSArray alloc]`还是`[NSMutableArray alloc]`？

看了下`《iOS高级内存管理编程指南》`中关于类对象类簇的部分内容，得到如下的伪代码实现:

- (1) `__NSPlacehodlerArray`全局使用一个单例对象A，来记录`[NSArray alloc]`的操作

```c
static __NSPlacehodlerArray *GetPlaceholderForNSArray() {
    static __NSPlacehodlerArray *instanceForNSArray;
    if (!instanceForNSArray) {
        instanceForNSArray = [__NSPlacehodlerArray alloc];
    }
    return instanceForNSArray;
}
```

- (2) `__NSPlacehodlerArray`全局使用一个单例对象B、 来记录`[NSMutableArray alloc]`的操作

```c
static __NSPlacehodlerArray *GetPlaceholderForNSMutableArray() {
    static __NSPlacehodlerArray *instanceForNSMutableArray;
    if (!instanceForNSMutableArray) {
        instanceForNSMutableArray = [__NSPlacehodlerArray alloc];
    }
    return instanceForNSMutableArray;
}
```

- (3) NSArray的alloc方法实现

```objc
+ (id)alloc
{
    if (self == [NSArray class]) {
        return GetPlaceholderForNSArray();//获取单例
    }
}
```

- (4) NSMutableArray的alloc方法实现

```objc
+ (id)alloc
{
    if (self == [NSMutableArray class]) {
        return GetPlaceholderForNSMutableArray();//获取单例
    }
}
```

- (5) `-[__NSPlacehodlerArray init]`方法实现

```objc
- (id)init
{
	// 判断[NSArray alloc]和[NSMutableArray alloc]得到的内存是哪一个类型的__NSPlacehodlerArray单例内存
	// 来创建对应的真实数据类型: __NSArrayI 或  __NSArrayM
	
    if (self == GetPlaceholderForNSArray()) {
    	/**
    	 *	重新创建一个__NSArrayI对象返回
    	 */
        self = [[__NSArrayI alloc] init];
    }
    else if (self == GetPlaceholderForNSMutableArray()) {
    	/**
    	 *	重新创建一个__NSArrayM对象返回
    	 */
        self = [[__NSArrayM alloc] init];
    }
    return self;
}
```

从 `-[__NSPlacehodlerArray init]`方法实现，可以看出 :

第一点、`__NSPlacehodlerArray` 内部管理了 `__NSArrayI 和 __NSArrayM` 这两个类，但并没有向外界公开，外界拿到的只是一个`NSArray的子类`，然后按照`NSArray.h`定义的方法api操作，其实根本不知道操作的是 `__NSArrayI 还是 __NSArrayM`。

第二点、`__NSPlacehodlerArray`全局存在两个不同的单例对象，用来控制是创建如下哪一种类型对象

- (1) `__NSArrayI`
- (2) `__NSArrayM`

第三点，注意最终`-[__NSPlacehodlerArray init]`创建的都是`新`的`__NSArrayI`或`__NSArrayM`类型的对象。


用代码证实一下 `-[NSArray alloc]` 和 `-[NSMuableArray alloc]`得到的是两个不同的`__NSPlacehodlerArray`单例对象

```objc
- (void)test2 {
    id obj1 = [NSArray alloc];
    id obj2 = [NSArray alloc];
    id obj3 = [NSArray alloc];
    id obj4 = [NSArray alloc];
    
    id obj5 = [NSMutableArray alloc];
    id obj6 = [NSMutableArray alloc];
    id obj7 = [NSMutableArray alloc];
    id obj8 = [NSMutableArray alloc];
    
}
```

断点后fr v输出如下

```
(__NSPlaceholderArray *) obj1 = 0x00007ff7e0f01160
(__NSPlaceholderArray *) obj2 = 0x00007ff7e0f01160
(__NSPlaceholderArray *) obj3 = 0x00007ff7e0f01160
(__NSPlaceholderArray *) obj4 = 0x00007ff7e0f01160

(__NSPlaceholderArray *) obj5 = 0x00007ff7e0f00ab0
(__NSPlaceholderArray *) obj6 = 0x00007ff7e0f00ab0
(__NSPlaceholderArray *) obj7 = 0x00007ff7e0f00ab0
(__NSPlaceholderArray *) obj8 = 0x00007ff7e0f00ab0
```

此外Foundation对`不可变版本的数组`的`空数组`也做了个小优化，即使用同一个对象内存来表示:

```objc
NSArray *arr1 = [[NSArray alloc] init];
NSArray *arr2 = [[NSArray alloc] init];
NSArray *arr3 = @[];
NSArray *arr4 = @[];
NSArray *arr5 = @[@1];
```

断点后fr v输出如下

```
(__NSArray0 *) arr1 = 0x00007fc640c02a30 @"0 objects"
(__NSArray0 *) arr2 = 0x00007fc640c02a30 @"0 objects"
(__NSArray0 *) arr3 = 0x00007fc640c02a30 @"0 objects"
(__NSArray0 *) arr4 = 0x00007fc640c02a30 @"0 objects"

(__NSArrayI *) arr5 = 0x00007fc640c0c920 @"1 object"
```

- 可以看到`前4个`的内存地址是一样的，后面一直内存地址不同.
- 若干个不可变的空数组间没有任何特异性，所以返回一个静态对象也理所应当.

小结下NSArray、NSMutableArray作为类簇:

- (1) 通过`NSArray`创建的对象，其实根本不是NSArray类的对象

- (2) 而是NSArray内部屏蔽的私有`子类`，注意类簇最好使用`子类化`来实现
	- `__NSArray0`
	- `__NSArrayI`
	- `__NSArrayM`

- (3) 这样做的目的为了避免将很多的不同版本子类直接暴露给用户使用，造成如果后续代码升级，用户就需要手动修改跟多的子类使用的代码

- (4) 通过使用一个统一的外部类簇类，来提供对应版本的操作api，让用户只需要操作这一个类就可以达到操作内部n多个子类


***

###类簇简单应用、在不同的iOS系统版本下，对一些不同系统版本的系统函数进行适配

通常一个类的某些方式实现在iOS7以下和iOS7以上或者未来的iOS9，是不同的实现方式，就造成某一些方法不能同时在iOS不同的系统版本上运行，导致程序crash.

如果一个类需要进行iOS7以下、iOS7以上、又或者iOS8、iOS9进行针对性的适配，有如下两种办法:

- 第一种、
	- `针对iOS6、iOS7、iOS8及以上分别提供一个单独的实现类`
	- 将所有的实现类全部暴露给给客户，让客户自己按需调用

- 第二种、
	- `针对iOS6、iOS7、iOS8及以上分别提供一个单独的实现类`
	- 提供一个`统一的入口类`给客户调用，客户不需要自己针对不同的系统版本调用对应的实现类，客户只需要操作暴露给他的统一入口来即可。
	- 不同版本的实现类判断以及调用，由统一入口类内部完成


对比如上两种方法，显然第二种方法会更好，写一个方法二的简单模板。

####暴露给客户的统一入口类（类簇）

```objc
#import <Foundation/Foundation.h>

@interface MyTool : NSObject

// 客户只需要关心调用这个方法即可
- (void)doWork;

@end
```

```objc
#import "MyTool.h"

// 其他私有的不同系统版本的实现子类
// 我们可以把它写在 .m 中，而不必写成单独的 类文件
#import "MyTool_iOS6.h"
#import "MyTool_iOS7.h"
#import "MyTool_iOS8.h"

@implementation MyTool

// 判断当前系统版本，选择对应系统版本下的实现类
+ (instancetype)alloc {
    if (self == [MyTool class]) {
        if (floor(NSFoundationVersionNumber) <= NSFoundationVersionNumber_iOS_6_1) {
            //iOS6
            return [MyTool_iOS6 alloc];
        } else if (floor(NSFoundationVersionNumber) > NSFoundationVersionNumber_iOS_6_1 && floor(NSFoundationVersionNumber) < NSFoundationVersionNumber_iOS_8_0) {
            //iOS7
            return [MyTool_iOS7 alloc];
        } else if (floor(NSFoundationVersionNumber) > NSFoundationVersionNumber_iOS_7_1){
            //iOS8及以上
            return [MyTool_iOS8 alloc];
        }
    }
    return [super alloc];
}

- (void)doWork { /* 空实现，由具体子类去实现 */ }

@end
```

### 入口类内部依赖的不同系统下的实现类

- iOS6下的实现类

```objc
#import "MyTool.h"

@interface MyTool_iOS6 : MyTool

@end
```

```objc
#import "MyTool_iOS6.h"

@implementation MyTool_iOS6

- (void)doWork {
    NSLog(@"iOS6 work");
}

@end
```

- iOS7的实现类

```objc
#import "MyTool.h"

@interface MyTool_iOS7 : MyTool

@end
```

```objc
#import "MyTool_iOS7.h"

@implementation MyTool_iOS7

- (void)doWork {
    NSLog(@"iOS7 work");
}

@end
```

- iOS8及以上的实现类

```objc
#import "MyTool.h"

@interface MyTool_iOS8 : MyTool

@end
```

```objc
#import "MyTool_iOS8.h"

@implementation MyTool_iOS8

- (void)doWork {
    NSLog(@"iOS8 work");
}

@end
```

### 最后是客户只需要找到入口类MyTool即可，而不需要知道如上的具体版本下的某一个实现类.

```objc
- (void)test3 {
	
	//1. 
    MyTool *tool = [[MyTool alloc] init];
    
    //2. 
    [tool doWork];
}
```

> 注意点，类簇的内部私有类，一般情况下都是直接`继承自类簇类`来实现。

到这里我想应该可以总结出： NSArray作为入口类（类簇）作用，其实就类似上面例子的MyTool的作用.

> 核心: 向调用者屏蔽内部所有的私有的具体实现类的信息，只需要当做类簇类一样的使用即可。


### 再记录一个来自Effect OC-2.0书上的一个问题，就是比较一个NSArray的对象的类型

如果已经弄明白NSArray/MyTool作为类簇的作用，那么你应该能立刻判断出如下两种写法哪一种是错误的。

- 情况一

```objc
NSArray *array = @[@"111", @"222", @"333"];
或
NSArray *array = [NSArray arrayWithObjects:@"111", @"222", @"333", nil];
    
if ([array class] == [NSArray class]) {
    NSLog(@"YES");
} else {
    NSLog(@"NO");
}
```

- 情况二

```objc
NSArray *array = @[@"111", @"222", @"333"];
或
NSArray *array = [NSArray arrayWithObjects:@"111", @"222", @"333", nil];

if ([array isKindOfClass:[NSArray class]]) {
    NSLog(@"YES");
} else {
    NSLog(@"NO");
}
```

> 答案: 情况一的Class比较是错误的。情况二是正确的。

### 下面是原因（如果之前的弄懂了其实已经不用看原因了）:

- 1) NSArray是一个类簇类，也就是说只是一个入口类
- 2) 最终创建返回的对象绝对不是NSArray的对象，而是如下三种类型:
	- `__NSArray0、__NSArrayI` 不可变数组
	- `__NSArrayM` 可变数组

- 3) 所以 [array class] 实际上得到的类型是 >>>> `__NSArray0、__NSArrayI`

- 4) 而 [NSArray class] 到的类型是 >>>>  `NSArray`

- 5) 对于每一个类（类对象）都是一个 `objc_class结构体`的一个`全局单例对象`，很显有如下关系，而且是`永远都成立`如下如下:
	- [NSArray class] != [__NSArray0 class]
	- [NSArray class] != [__NSArrayI class]
	- [NSArray class] != [__NSArrayM class]

### 所以对于一些`类簇类`比较类型时:

- 不要使用 `if( [对象 class] == [类簇 class] ) {}`来对类簇返回的类对象的类型进行比较，因为`永远都不会成立`

- 需要使用 `[array isKindOfClass:[NSArray class]]` 来比较类型
	- isKindOfClass: 会一直通过 `类对象的isa指针` 查找 `super_class`
	- 而 `__NSArrayI、__NSArray0、__NSArrayM` 都是 继承自 `类簇NSArray` 

## 那么到此为止，类簇的作用已经很明显 >>> 向外屏蔽内部使用的各种私有的具体实现类.

### 那么前面的一个问题，类簇和抽象工厂模式有什么关系？

看下前面关于MyTool进行不同的iOS系统版本适配的的结构图

[![Snip20160512_2.png](http://imgchr.com/images/Snip20160512_2.png)](http://imgchr.com/image/Ply)

### 其实 MyTool这个类，就是一个工厂，且是一个具体工厂（因为没有实现接口的形式暴露出去），而他生成的产品就是实现了 `- (void)doWork;`方法的具体类的一个对象:

- MyTool_iOS6 实现类一
- MyTool_iOS7 实现类二
- MyTool_iOS8 实现类三

我自己的理解就是，

```
类簇模式 = 一个工厂 + n多个该工厂管理的且实现了同一个抽象产品接口的具体产品类
```

不像抽象工厂，存在多种不同的产品类型共存的情况.

## 前面还说过NSString也是类簇，那么也来看看

### [NSString alloc] 与 [NSMutableString alloc] 返回的内部真实的私有类型

```objc
- (void)string1 {
    
    id obj1 = [NSString alloc];
    id obj2 = [NSMutableString alloc];
}
```

断点输出如下

```
(NSPlaceholderString *) obj1 = 0x00007ff378701ee0
(NSPlaceholderMutableString *) obj2 = 0x00007ff378701f30
```

### [NSPlaceholderString对象 init] 与 [NSPlaceholderMutableString init] 返回的最终私有类型

```objc
- (void)string1 {
    
    id obj1 = [NSString alloc];
    id obj2 = [NSMutableString alloc];

    NSString *obj3 = [obj1 init];
    NSMutableString *obj4 = [obj2 init];
    
}
```

断点输出如下

```
(__NSCFConstantString *) obj3 = 0x000000010048f390 @""
(__NSCFString *) obj4 = 0x00007f97ea6b9dd0
```

- 不可变字符串类型最终的使用类型 >>> __NSCFConstantString
- 可变字符串最终使用类型 >>> __NSCFString


### NSString内部管理私有类 >>> __NSCFConstantString（不可变、常量字符串）

```objc
- (void)string1 {
   
    NSString *str1 = @"Hello World~~";
    NSString *str2 = @"Hello World~~";
    NSString *str3 = @"Hello World~~";
    
    NSLog(@"str1 = %p, str1 class = %@",str1, NSStringFromClass([str1 class]));
    NSLog(@"str2 = %p, str2 class = %@",str2, NSStringFromClass([str2 class]));
    NSLog(@"str3 = %p, str3 class = %@",str3, NSStringFromClass([str3 class]));
}
```

运行后的输出

```
2016-05-13 15:02:21.325 Demos[1209:12412] str1 = 0x1072a93c0, str1 class = __NSCFConstantString
2016-05-13 15:02:21.325 Demos[1209:12412] str2 = 0x1072a93c0, str2 class = __NSCFConstantString
2016-05-13 15:02:21.326 Demos[1209:12412] str3 = 0x1072a93c0, str3 class = __NSCFConstantString
```

可以看到两点:

- 指向同一个内存地址 >>> 对于 `@"Hello World~~"` 这样的字符串会单独使用`一个唯一的内存空间`存放

- 其类型是 `__NSCFConstantString ` >>> 常量字符串

### NSString内部管理私有类 >>> __NSCFString（可变字符串）

```
NSMutableString *str2 = [@"1111" mutableCopy];
```

```
NSMutableString *str3 = [[NSMutableString alloc] initWithString:@"Haha"];
```

### NSString内部管理私有类 >>> NSTaggedPointerString

```
NSString *str1 = [NSString stringWithFormat:@"Haha %d", 111];
```

还有一个path的string暂时找不到事怎么弄出来的...

## 类簇 >>> 提供一个统一入口类，来隐藏内部n多的真实的私有类

个人觉得 类簇 与 抽象工厂 还是有一定的区别:

- 类簇 是 抽象工厂的 `简化版`
	- 类簇 一般使用 直接`继承`，而不需要使用`实现接口`的形式，不然就显得过于啰嗦了
	- 一个类簇，完成`一个工厂`的作用，仅提供对于`一个类型`下的不同版本实现类

- 而抽象工厂
	- 产品抽象、工厂抽象
	- 具体工厂 内部管理 `不同类型的` 产品抽象 的实现类（类簇一般只针对一个类型的多个实现）
	- 可以有`多个不同的工厂`并存（类簇一般是一个工厂）


学习资料来源

```
http://www.cocoachina.com/ios/20141219/10696.html
http://www.cocoachina.com/industry/20140109/7681.html
http://mobile.51cto.com/hot-436710.htm
```