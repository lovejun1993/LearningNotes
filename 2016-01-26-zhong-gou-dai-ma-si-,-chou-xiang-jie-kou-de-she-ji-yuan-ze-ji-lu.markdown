---
layout: post
title: "重构代码四、抽象接口的设计原则记录"
date: 2016-01-26 12:45:34 +0800
comments: true
categories: 
---

###对于一切 `可能发生变化的功能点` 使用 抽象接口 隔离

重构代码过程中，发现过去的一些封装思路好像都忘记的差不多了，还是回头记录下，经常温习把.

***

###接口设计原则一、接口隔离变化

- 可能发生的变化点

	- 流程性逻辑 变化
	- 子模块内部实现 变化
	- 可见视图 变化

- 将会经常变化的一些`相关联性的功能函数`，集中使用一个 `抽象协议` 封装.


```objc
/**
 *  动物类的抽象
 */
@protocol Animal <NSObject>

/** 动物睡觉 */
- (void)slepp;

/** 动物跑步 */
- (void)run;

@end
```

每一种动物 都有各自的 奔跑姿势、睡觉姿势...

![](http://i4.tietuku.com/3aedcb0ac09c34c0.png)

实现类一、猫

```objc
@interface Cat : NSObject <Animal>

@end

@implementation Cat

- (void)slepp {
    NSLog(@"猫睡觉...");
}

- (void)run {
    NSLog(@"猫跑步...");
}

@end
```

实现类二、狗

```objc
@interface Dog : NSObject <Animal>

@end

@implementation Dog

- (void)slepp {
    NSLog(@"狗睡觉...");
}

- (void)run {
    NSLog(@"狗跑步...");
}

@end
```

在需要一个 `动物` 的地方，使用 `抽象接口` 隔离 具体实现

```objc
//1. 由中间容器获取一个实现类对象
id<Animal> animal = [中间容器 getInstanceWithProtocol:@protocol(Aniaml)];

//2. 然后调用 `抽象接口给出` 的方法
[animal run];
[animal sleep];
```

***

###接口设计原则二、职责单一

- 一个接口 应该只负责 一个模块 的功能点抽象，不应该加入 `其它不相关模块` 的功能点函数

- 一个接口的功能函数要尽量少

- `不要` 建立 `臃肿庞大` 的接口，尽量建立细粒度的抽象接口

- 接口之间还可以组合

![](http://i4.tietuku.com/a3370193957460b6.png)


首先对 动物类 的 `基本功能` 抽象

```objc
#import <Foundation/Foundation.h>

/**
 *  动物类抽象
 */
@protocol Animal <NSObject>

//全部必须实现
@required

/**
 *  动物可以跑步
 */
- (void)run;


/**
 *  动物可以睡觉
 */
- (void)sleep;

@end
```

`独立功能1` 的抽象接口定义

```objc
#import <Foundation/Foundation.h>

/**
 *  动物表演杂技功能的单独抽象
 */
@protocol AnimalPerfomance <NSObject>

//可选实现
@optional

/**
 *  表演跳火圈
 */
- (void)tiaoHuoQuan;

@end
```

`独立功能2` 的抽象接口定义

```objc
#import <Foundation/Foundation.h>

/**
 *  动物卖萌功能的单独抽象
 */
@protocol AnimalMaiMeng <NSObject>

//可选实现
@optional

/**
 *  卖萌功能
 */
- (void)maimeng;

@end
```

`独立功能3` 的抽象接口定义

```objc
#import <Foundation/Foundation.h>

/**
 *  动物看家功能的单独抽象
 */
@protocol AnimalKanJia <NSObject>

//可选实现
@optional

/**
 *  看家功能
 */
- (void)kanjia;

@end
```

如上抽象接口的全部实现

```objc
//
//  Dog.h
//  demos
//
//  Created by xiongzenghui on 16/1/29.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import <Foundation/Foundation.h>

//导入所有需要实现的抽象接口
#import "Animal.h"
#import "AnimalPerfomance.h"
#import "AnimalMaiMeng.h"
#import "AnimalKanJia.h"

//实现所有抽象接口的方法
@interface Dog : NSObject <Animal, AnimalPerfomance, AnimalKanJia, AnimalMaiMeng>

@end
```

```objc
//
//  Dog.m
//  demos
//
//  Created by xiongzenghui on 16/1/29.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import "Dog.h"

@implementation Dog


/////////////////////////////////////////////////////////////
///Animal 协议

- (void)run {
    NSLog(@"狗正在跑...");
}

- (void)sleep {
    NSLog(@"狗正在睡觉...");
}

/////////////////////////////////////////////////////////////
///AnimalPerfomance 协议

- (void)tiaoHuoQuan {
    NSLog(@"狗正在表演跳火圈...");
}

/////////////////////////////////////////////////////////////
///AnimalKanJia 协议

- (void)kanjia {
    NSLog(@"狗正在看家...");
}

/////////////////////////////////////////////////////////////
///AnimalMaiMeng 协议

- (void)maimeng {
    NSLog(@"狗正在卖萌...");
}

@end
```

将 `实现类` 与 `前面的所有抽象接口` 进行绑定，后续获取这四个接口的实现类对象都是Dog对象.

```objc
bind("Dog", @protocol(Animal));
bind("Dog", @protocol(AnimalPerfomance));
bind("Dog", @protocol(AnimalKanJia));
bind("Dog", @protocol(AnimalMaiMeng));
```

后续就根据需要 使用 `不同的抽象接口` 来泛化 实现类对象，指明`只能调用其抽象接口中定义`的方法

- 假设需要调用 动物基础功能

```objc
//1. 使用 Animal主抽象接口类型 指向 实现类对象
id<Animal> animal = [中间容器 getInstanceWithProtocol:@protocol(Aniaml)];

//2. 调用 Animal接口 中的定义的方法
[animal run];
[animal sleep];
```

- 假设需要调用 动物表演杂技的功能

```objc
//1. 使用 AnimalPerfomance抽象接口类型 指向 实现类对象
id<AnimalPerfomance> animal = [中间容器 getInstanceWithProtocol:@protocol(AnimalPerfomance)];

//2. 调用 AnimalPerfomance接口 中的定义的方法
[animal tiaoHuoQuan];
```

- 假设需要调用 动物卖萌的功能

```objc
//1. 使用 AnimalMaiMeng抽象接口类型 指向 实现类对象
id<AnimalMaiMeng> animal = [中间容器 getInstanceWithProtocol:@protocol(AnimalMaiMeng)];

//2. 调用 AnimalMaiMeng接口 中的定义的方法
[animal maimeng];
```

- 假设需要调用 动物看家的功能

```objc
//1. 使用 AnimalKanJia抽象接口类型 指向 实现类对象
id<AnimalKanJia> animal = [中间容器 getInstanceWithProtocol:@protocol(AnimalKanJia)];

//2. 调用 AnimalKanJia接口 中的定义的方法
[animal kanjia];
```


***

###接口设计原则三、抽象父类 提供 一些基础`模板方法`、以及这些模板方法组成的一套`逻辑步骤`

```objc
#import <Foundation/Foundation.h>

/**
 *  抽象父类，提供一些基础的方法模块
 *  这些模板方法可以有基础实现，也可以是空实现
 *  具体由每一个不同的子类去各自实现
 */
@interface MyAbstractClass : NSObject

#pragma mark - 模板方法

- (void)run;

- (void)sleep;

- (void)dead;

#pragma mark - 提供一些基本流程

- (void)life;

@end
```

```objc
#import "MyAbstractClass.h"

@implementation MyAbstractClass


////////////////////////////////////////////
//////空实现
////////////////////////////////////////////

- (void)run{}

- (void)sleep{}


////////////////////////////////////////////
//////默认基础实现
////////////////////////////////////////////

- (void)dead {
    NSLog(@"我挂了...");
}

////////////////////////////////////////////
//////提供一些基本流程
////////////////////////////////////////////

- (void)life {
    
    //1. 走路
    [self run];
    
    //2. 睡觉
    [self sleep];
    
    //3. 挂了
    [self dead];
}

@end
```

- 子类可以重写 抽象父类的
	- `模板方法`

- 子类`最好不要`重写 抽象父类的
	- `默认方法实现` ，默认方法实现最好是能否提供给所有子类的 `通用`功能
	- `逻辑步骤`

***


###接口设计原则四、里氏替换原则

- 使用 `父类` 引用 `子类的对象`
	- 可以随时替 换成 `其它子类的对象`
	- 适合做一些 `抽象父类` 提供一些 `基本的方法模块`
	- 供子类实现 `特殊` 的功能

- 使用 `抽象接口` 引用 `接口实现类的对象`
	- 可以随时替 换成 `其它接口实现类的对象`


动物抽象接口实现一、猫

```objc
@interface Cat : NSObject <Animal>

@end

@implementation Cat

- (void)slepp {
    NSLog(@"猫睡觉...");
}

- (void)run {
    NSLog(@"猫跑步...");
}

@end
```

动物抽象接口实现二、猫

```objc
@interface Dog : NSObject <Animal>

@end

@implementation Dog

- (void)slepp {
    NSLog(@"狗睡觉...");
}

- (void)run {
    NSLog(@"狗跑步...");
}

@end
```

使用抽象接口定义对象的类型

```objc
id<Animal> animal = [Cat new];
//id<Animal> animal = [Dog new];
```

***

###接口设计原则五、依赖倒置原则（面向抽象/接口）

- 上层 只依赖 下层暴露出来的 `抽象接口`

- 依赖的几种写法

写法一、构造函数传递依赖对象

```objc
@protocol Animal <NSObject>

- (void)run;

@end
```

```objc
@interface ViewController ()

@property (nonatomic, strong) id<Animal> animal;

@end
```

```objc
@implementation ViewController

- initWithAnimal:(id<Animal>)animal
{
    self = [super init];
    
    if (self) {
        self.animal = animal;
    }
    
    return self;
}

@end
```

写法二、由中间容器注入依赖对象


```objc
Objection开源库使用
```


写法三、接口声明依赖对象

```objc
@interface ViewController ()

@property (nonatomic, strong) id<Animal> animal;

@end
```

####对于面向接口编程的最佳实现状态

- 状态一: 每个类 尽量都有 `接口或抽象类`，或者 `抽象类和接口两者同时具备`

使用前面的抽象类 `MyAbstractClass`，新添加一个抽象接口

```objc
/**
 *  可以在马戏团表演
 */
@protocol AnimalPerformance <NSObject>

/** 表演跳火圈 */
- (void)tiaoHuoQuan;

@end
```

具体类Monkey先`继承自抽象类`，然后额外实现一个`功能抽象接口`

```objc
//1. Monkey先`继承自抽象类`
//2. 额外实现一个`功能抽象接口`，来具备一个功能
@interface Monkey : MyAbstractClass <AnimalPerformance>

@end

@implementation Monkey

#pragma mark - 抽象类方法实现

- (void)run {
    NSLog(@"猴子跑..");
}

- (void)sleep {
    NSLog(@"猴子睡觉..");
}

- (void)dead {
    //1. 可以做自己的实现
    //2. 也可以使用父类的默认实现
}

//切记不要重写抽象类提供的 `步骤性` 的方法
//- (void)life {
//    .....
//}

#pragma mark - 抽象接口实现

- (void)tiaoHuoQuan {
    NSLog(@"猴子表演跳火圈了...");
}

@end
```

- 状态二: 任何类 都不应该 从 `具体类` 派生

这点好像有点太过理想化，有些东西感觉还是直接继承来的方便.

- 状态三: 尽量 `不要覆写` 父类 的方法

而将需要`变化实现`的方法 抽象隔离到 接口协议中，使用`实现接口的形式`的重写

***

###接口设计原则六、开闭原则

- 只能 添加 `新的实现类 或 接口`
- 不能 修改 `已有实现实现类 或 接口`

