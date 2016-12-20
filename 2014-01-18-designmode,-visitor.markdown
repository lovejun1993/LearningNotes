---
layout: post
title: "DesignMode、Visitor"
date: 2014-01-18 23:29:02 +0800
comments: true
categories: 
---

###访问者、将对一个实体类的业务逻辑，封装到一个Visitor中.

***

###假设有两个实体

- 书

- 水果

***

###各自有各自的业务逻辑

- 书本需要计算最终的`价格` >>> float返回值类型

- 水果需要计算最终的`价格` >>> float返回值类型

可以看到是相同类型的业务，具体说是相同类型返回值.

****

###可以被访问的抽象节点

```objc
//
//  Element.h
//  demos
//

#import <Foundation/Foundation.h>

@protocol Visitor;

/**
 *  实体类抽象
 */
@protocol Element <NSObject>

/**
 *  当前Elemnt对象 转发给 传入的Vistor对象 进行处理
 *  （被传入的Visitor对象访问）
 *  
 *  注意: 此函数定义了`返回值`的类型
 */
- (float)acceptVistor:(id<Visitor>)vistor;


@end
```

###具体节点1、书

```objc
//
//  BookService.h
//  demos
//

#import <Foundation/Foundation.h>

#import "Element.h"

/**
 *  计算书的价格
 */
@interface Book : NSObject <Element>

@property (nonatomic, assign) float bookPrice;

@property (nonatomic, copy) NSString *bookNo;

@end
```

```objc
//
//  BookService.m
//  demos
//

#import "Book.h"

#import "Visitor.h"

@implementation Book

- (float)acceptVistor:(id<Visitor>)vistor {
    
    //1. 对当前BookService对象的条件进行过滤
    if ([self.bookNo isEqualToString:@""] || \
        !self.bookNo)
    {
        return 0.0;
    }
    
    //2. 让Vistor处理
    return [vistor visitWithFruit:self];
}

@end
```

###具体节点2、水果

```objc
//
//  Fruit.h
//  demos
//

#import <Foundation/Foundation.h>

#import "Element.h"

/**
 *  计算水鬼的价格
 */

@interface Fruit : NSObject <Element>

@property (nonatomic, assign) float fruitPrice;

@property (nonatomic, copy) NSString *fruitNo;

@end
```

```objc
//
//  Fruit.m
//  demos
//

#import "Fruit.h"

#import "Visitor.h"

@implementation Fruit

- (float)acceptVistor:(id<Visitor>)vistor {
    
    //1. 对当前FruitService对象的条件进行过滤
    if ([self.fruitNo isEqualToString:@""] || \
        !self.fruitNo)
    {
        return 0.0;
    }
    
    //2. 让Vistor处理
    return [vistor visitWithFruit:self];
}

@end
```

###抽象Visitor

```objc
//
//  Visitor.h
//  demos
//

#import <Foundation/Foundation.h>

@protocol Element;

/**
 *  抽象访问者
 */
@protocol Visitor <NSObject>

/**
 *  业务1:
 *  计算书价格业务
 *  返回值为float
 */
- (float)visitWithBook:(id<Element>)book;

/**
 *  业务2:
 *  计算水果价格业务
 *  返回值为float
 */
- (float)visitWithFruit:(id<Element>)fruit;

@end
```

###具体Visitor

```objc
//
//  ConcreateVistor.h
//  demos
//

#import <Foundation/Foundation.h>

#import "Visitor.h"

@interface ConcreateVistor : NSObject <Visitor>

@end
```

```objc
//
//  ConcreateVistor.m
//  demos
//

#import "ConcreateVistor.h"

/** 导入具体的Elment实现类 */
#import "Book.h"
#import "Fruit.h"

@implementation ConcreateVistor

- (float)visitWithBook:(id<Element>)book {
    
    //1.
    Book *bs = (Book *)book;
    
    //2.
    float price = bs.bookPrice;
    
    //3. 书价打折
    return price * 0.6;
    
    /** 优化: 可以单独抽一个Manager类提供这个处理业务 */
    //...
}

- (float)visitWithFruit:(id<Element>)fruit {
    
    //1.
    Fruit *fs = (Fruit *)fruit;
    
    //2.
    float price = fs.fruitPrice;
    
    //3. 水果打折
    return price * 0.85;
    
    /** 优化: 可以单独抽一个Manager类提供这个处理业务 */
    //...
}

@end
```

###demo实例

```objc
//
//  VistorTest.m
//  demos
//

#import "VistorTest.h"

//商品类
#import "Book.h"
#import "Fruit.h"

//访问者
#import "ConcreateVistor.h"

@implementation VistorTest

+ (void)test {

    //1. 实体1
    Book *book = [Book new];
    book.bookPrice = 99.0;
    book.bookNo = @"book11111";
    
    //2. 实体2
    Fruit *fruit = [Fruit new];
    fruit.fruitPrice = 129.0;
    fruit.fruitNo = @"fruit2222";
    
    //3. 访问者（业务统一调用者）
    id<Visitor> vistor = [ConcreateVistor new];
    
    //4. 将Book交给Visitor对象完成计算总价的
    float price1 = [book acceptVistor:vistor];
    
    //5. 将Fruit交给Visitor对象完成计算总价的
    float price2 = [fruit acceptVistor:vistor];
    
    //6. 总价
    float sum = price1 + price2;
}

@end
```

###从上可以得到

- Visitor中封装了 `Book、Fruit实体` 的一些 `业务逻辑`

- Visitor提供了 `计算总价` 这一类业务逻辑的 `统一入口`
	- Book的 总价计算
	- Fruit的 总价计算


****

###个人凭感觉发挥了下，觉得上面的Vistor中处理多个不同的业务，而且也是通过强转数据类型获取要处理的数据，那么针对这两个问题进一步细化了下

####首先两种抽象Vistor协议定义

- Vistor1

```objc
//
//  Vistor1.h
//  demos
//
//  Created by xiongzenghui on 16/3/29.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import <Foundation/Foundation.h>

/**
 *  处理类型1的节点
 */
@protocol ServiceElement1;

@protocol Vistor1 <NSObject>

/* 处理节点1的 某一个业务 */
- (int)visitWithElement1:(id<ServiceElement1>)element1;

@end
```

- Vistor2

```objc
//
//  Vistor2.h
//  demos
//
//  Created by xiongzenghui on 16/3/29.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import <Foundation/Foundation.h>

/**
 *  处理类型2的节点
 */
@protocol ServiceElement2;

@protocol Vistor2 <NSObject>

/* 处理节点2的 某一个业务 */
- (float)visitWithElement2:(id<ServiceElement2>)element2;

@end
```

####然后两种抽象节点，分别接受对应的一个Vistor

- element1

```objc
//
//  ServiceElement.h
//  demos
//
//  Created by xiongzenghui on 16/3/29.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import <Foundation/Foundation.h>

@protocol Vistor1;

@protocol ServiceElement1 <NSObject>

/* 获取实体的参数，这里只是图简单写出一个字典，其实应该抽象出一个参数接口协议 */
- (NSDictionary *)argument;

/* 接收一个Vistor处理 */
- (int)acceptor1:(id<Vistor1>)vistor;

@end
```

- element2

```objc
//
//  ServiceElement2.h
//  demos
//
//  Created by xiongzenghui on 16/3/29.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import <Foundation/Foundation.h>

@protocol Vistor2;

@protocol ServiceElement2 <NSObject>

/* 获取实体的参数，这里只是图简单写出一个字典，其实应该抽象出一个参数接口协议 */
- (NSDictionary *)argument;

/* 接收一个Vistor处理 */
- (float)acceptor2:(id<Vistor2>)vistor;

@end
```

###具体Vistor实现

- 抽象Vistor1的具体实现

```objc
//
//  ConcreateVistor1.h
//  demos
//
//  Created by xiongzenghui on 16/3/29.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import <Foundation/Foundation.h>

//导入Vistor1 协议
#import "Vistor1.h"

@interface ConcreateVistor1 : NSObject <Vistor1>

@end
```

```objc
//
//  ConcreateVistor1.m
//  demos
//
//  Created by xiongzenghui on 16/3/29.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import "ConcreateVistor1.h"

//导入节点1 协议
#import "ServiceElement1.h"

@implementation ConcreateVistor1

- (int)visitWithElement1:(id<ServiceElement1>)element1 {
    
    //1. 获取要处理的参数（这里可以做出一个参数接口协议）
    NSDictionary *argument = [element1 argument];
    int total = [[argument objectForKey:@"totoalPrice1"] floatValue];
    
    //2. 执行当前Vistor封装的业务
    return total * 2;
}

@end
```

- 抽象Vistor2的具体实现

```objc
//
//  ConcreateVistor2.h
//  demos
//
//  Created by xiongzenghui on 16/3/29.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import <Foundation/Foundation.h>

//导入Vistor2 协议
#import "Vistor2.h"

@interface ConcreateVistor2 : NSObject <Vistor2>

@end
```

```objc
//
//  ConcreateVistor2.m
//  demos
//
//  Created by xiongzenghui on 16/3/29.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import "ConcreateVistor2.h"

//导入节点2协议
#import "ServiceElement2.h"

@implementation ConcreateVistor2

- (float)visitWithElement2:(id<ServiceElement2>)element2 {
    //1. 获取要处理的参数（这里可以做出一个参数接口协议）
    NSDictionary *argument = [element2 argument];
    float total = [[argument objectForKey:@"totoalPrice2"] floatValue];
    
    //2. 执行当前Vistor封装的业务
    return total * 0.9;
}

@end
```

####最后是实体类，根据需要实现抽象节点协议。不同的Element对应不同的Vistor

- 实体1

```objc
//
//  VEntity1.h
//  demos
//
//  Created by xiongzenghui on 16/3/29.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import <Foundation/Foundation.h>

//导入所有节点协议
#import "ServiceElement1.h"
#import "ServiceElement2.h"

//成为两种节点
@interface VEntity1 : NSObject <ServiceElement1, ServiceElement2>

@end
```

```objc
//
//  VEntity1.m
//  demos
//
//  Created by xiongzenghui on 16/3/29.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import "VEntity1.h"

//导入访问者
#import "Vistor1.h"
#import "Vistor2.h"

@implementation VEntity1

- (NSDictionary *)argument {
    return @{
             @"参数1" : @"value1",
             @"参数2" : @"value2",
             };
}

- (int)acceptor1:(id<Vistor1>)vistor {
    return [vistor visitWithElement1:self];
}

- (float)acceptor2:(id<Vistor2>)vistor {
    return [vistor visitWithElement2:self];
}

@end
```

- 实体2

```objc
//
//  VEntity2.h
//  demos
//
//  Created by xiongzenghui on 16/3/29.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import <Foundation/Foundation.h>

//导入所有节点协议
#import "ServiceElement1.h"
#import "ServiceElement2.h"

//成为两种节点
@interface VEntity2 : NSObject <ServiceElement1, ServiceElement2>

@end
```

```objc
//
//  VEntity2.m
//  demos
//
//  Created by xiongzenghui on 16/3/29.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import "VEntity2.h"

//导入访问者
#import "Vistor1.h"
#import "Vistor2.h"

@implementation VEntity2

- (NSDictionary *)argument {
    return @{
             @"参数1" : @"value1",
             @"参数2" : @"value2",
             };
}

- (int)acceptor1:(id<Vistor1>)vistor {
    return [vistor visitWithElement1:self];
}

- (float)acceptor2:(id<Vistor2>)vistor {
    return [vistor visitWithElement2:self];
}


@end
```

####现在要新增一个其他类型数据的处理业务

####声明对应的Element节点协议 与 Visitor协议

- Element协议

```
#import <Foundation/Foundation.h>

@protocol Vistor3;

@protocol ServiceElement3 <NSObject>

/* 获取实体的参数，这里只是图简单写出一个字典，其实应该抽象出一个参数接口协议 */
- (NSString *)uname;
- (NSString *)upassword;

/* 接收一个Visitor业务处理对象 */
- (NSNumber *)acceptor3:(id<Vistor3>)vistor;

@end
```

- Vistitor协议

```
#import <Foundation/Foundation.h>

@protocol ServiceElement3;

@protocol Visitor3 <NSObject>

- (NSNumber *)visitWithElement3:(id<ServiceElement3>)element3;

@end
```

####让需要成为该新节点的实体类实现Element协议，具备这个类型业务处理的能力

```objc
#import <Foundation/Foundation.h>

//导入所有节点协议
#import "ServiceElement1.h"
#import "ServiceElement2.h"
#import "ServiceElement3.h"

//成为两种节点
@interface VEntity1 : NSObject <ServiceElement1, ServiceElement2, ServiceElement3>

@end
```

```objc
//
//  VEntity1.m
//  demos
//
//  Created by xiongzenghui on 16/3/29.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import "VEntity1.h"

//导入访问者
#import "Vistor1.h"
#import "Vistor2.h"
#import "Visitor3.h"

@implementation VEntity1 {
    NSString *_uname;
    NSString *_upassword;
}

/* Element1节点处理需要的参数 */
- (NSDictionary *)argument1 {
    return @{
             @"参数1" : @"value1",
             @"参数2" : @"value2",
             };
}

/* Element2节点处理需要的参数 */
- (NSDictionary *)argument2 {
    return @{
             @"参数1" : @"value1",
             @"参数2" : @"value2",
             };
}


/* Element3节点处理需要的参数 */
- (NSString *)uname {
    return _uname;
}

- (NSString *)upassword {
    return _upassword;
}


#pragma mark - Element1 节点处理

- (int)acceptor1:(id<Vistor1>)vistor {
    return [vistor visitWithElement1:self];
}

#pragma mark - Element2 节点处理

- (float)acceptor2:(id<Vistor2>)vistor {
    return [vistor visitWithElement2:self];
}

#pragma mark - Element3 节点处理

- (NSNumber *)acceptor3:(id<Visitor3>)vistor {
    return [vistor visitWithElement3:self];
}

@end
```

####小结Visitor模式编写注意点:

- 实体类根据需要，选择性的实现Element协议，成为对应节点类型从而具备该节点对应的业务处理能力

- 一类Element 应该只对应的 一类Vistor

- 一个Element结构
	- 接收一个实现了依赖的抽象Visitor接口的具体Visitor对象
	- 提供该Elment对应的Vistor进行业务处理时需要的各种参数
	- 调用通过传入的Visitor对象，来进行最终的业务处理

- 一个Vistor只负责处理`一种`业务，保持职责单一
	- 多种业务分成不同的Visitor单独封装

****


###Visitor模式的缺点:

- Element中的`accept`方法的`返回值类型` 必须事先确定，并且与Visitor中的`visitXxx`方法的`返回值类型`要一致.
	- 不同实体类，但是相同业务的处理，也就是返回值相同

- 访问者模式，只适合`实体类型确定、数据类型确定、后期代码重构`的时候使用