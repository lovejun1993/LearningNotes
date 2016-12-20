---
layout: post
title: "DesignMode、Adapter Class/Object"
date: 2014-01-05 19:58:02 +0800
comments: true
categories: 
---

###适配器、将 `一个类的接口` 转换成客户希望的 `另外一个接口`，使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。

***

###适配器模式有3种

- 面向 `接口` 的适配器模式 
- 面向 `类` 的适配器模式
- 面向 `对象` 的适配器模式

***


###先看下 面向`接口` 适配器

####语言能力抽象接口

```objc
#import <Foundation/Foundation.h>

/**
 *  语言能力抽象
 */
@protocol Language <NSObject>

//说日语
- (void)speakJapanese;

//说英语
- (void)speakEnglish;

@end
```

####Person类只具备`说日语`和`说英语`功能

```objc
#import <Foundation/Foundation.h>

//导入语言能力接口
#import "Language.h"

@interface Person : NSObject <Language>

@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *sex;

@end
```

```objc
#import "Person.h"

@implementation Person

- (void)speakJapanese {
    NSLog(@"说日语");
}

- (void)speakEnglish {
    NSLog(@"说英语");
}

@end
```


####但是现在Person类型，需要具备`说法语`的新功能。但是不能修改已有的接口，所以只能通过`增加新接口`来声明`说法语`的功能，并且还得包括`已有接口所有的功能定义`

```objc
#import "Language.h"

/**
 *  作为Language接口的适配器，继承自原来的抽象接口
 *  从而具备了说日语、英语的抽象定义
 */
@protocol NewLanguage <Language> 

//说法语
- (void)speakFrance;

@end
```

####将Person类之前实现的接口，改为新的接口类型 `NewLanguage`

```objc
#import <Foundation/Foundation.h>

//导入语言能力接口
#import "NewLanguage.h"

//改为实现新的接口 NewLanguage
@interface Person : NSObject <NewLanguage>

@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *sex;

@end
```

```objc
#import "Person.h"

@implementation Person

- (void)speakJapanese {
    NSLog(@"说日语");
}

- (void)speakEnglish {
    NSLog(@"说英语");
}

- (void)speakFrance {
    NSLog(@"说法语");
}

@end
```

####新接口类型使用

```objc
#import "AdaptorTest.h"

#import "Person.h"

@implementation AdaptorTest

- (void)test {
    
    //1. 老接口类型的对象
    id<Language> oldLanguagePerson = [Person new];
    
    //老接口类型的使用
    [oldLanguagePerson speakEnglish];
    [oldLanguagePerson speakJapanese];
    
    //2. 转换成新接口类型的对象
    id<NewLanguage> newLanuagePerson = (id<NewLanguage>)oldLanguagePerson;

    //新接口类型的使用    
    [newLanuagePerson speakFrance];
}

@end
```

####如上有个缺点: 会直接修改Person类实现的接口类型，可能会造成一些影响.

***

###再看下 `类` 的适配器

####语言能力接口（老接口）

```objc
#import <Foundation/Foundation.h>

/**
 *  语言能力抽象
 */
@protocol Language <NSObject>

//说日语
- (void)speakJapanese;

//说英语
- (void)speakEnglish;

@end
```

####Man类 实现 老版本接口

```objc
#import <Foundation/Foundation.h>

#import "Language.h"

@interface Man : NSObject <Language>

@end
```

```objc
#import "Man.h"

@implementation Man

- (void)speakJapanese {
    NSLog(@"说日语");
}

- (void)speakEnglish {
    NSLog(@"说英语");
}

@end
```

####提供新的具备`法语功能`的新接口

```objc
#import "Language.h"

@protocol NewLanguage <NSObject>

//说法语
- (void)speakFrance;

@end
```

####现在想对Man类添加 `说法语` 的语言能力

- 第一步、写一个`子类`继承自Man类，获取说日语与英语的能力
- 第二步、然后让`子类`去实现`新语言接口`，具备说法语的新能力

```objc
#import "Man.h"

#import "NewLanguage.h"

/**
 *  作为Man类的适配器
 */
@interface ManAdaptor : Man <NewLanguage>

@end
```

```objc
#import "ManAdaptor.h"

@implementation ManAdaptor

- (void)speakFrance {
    NSLog(@"由Adaptor类，提供的说法语");
}

@end
```

####这样通过继承原始Person类之后，可以不用直接对Person类修改.

```objc
//1. 使用适配器类的对象 
ManAdaptor *adaptor = [ManAdaptor new];

//2. 原始接口方法
[adaptor speakJapanese];
[adaptor speakEnglish];

//3. 新接口方法
[adaptor speakFrance];
```

***

###`对象` 适配器

####老接口

```objc
#import <Foundation/Foundation.h>

/**
 *  语言能力抽象
 */
@protocol Language <NSObject>

//说日语
- (void)speakJapanese;

//说英语
- (void)speakEnglish;

@end
```

####`新接口` 继承自 `老接口`

```objc
#import "Language.h"

@protocol NewLanguage <Language>

//说法语
- (void)speakFrance;

@end
```

###创建一个适配器类 实现 新接口，并接收一个 老接口实现类对象

```objc
#import <Foundation/Foundation.h>

//实现新的接口，包含新添加的法语功能
#import "NewLanguage.h"

//老版本接口实现类
#import "Person.h"

//适配器类实现新版本接口
@interface ObjectAdaptor : NSObject <NewLanguage>

/**
 *  接收一个 老接口Language的 实现类 对象
 *  并作为其对象的适配器
 */
- (instancetype)initWithPerson:(Person *)person;

@end
```

```objc
#import "ObjectAdaptor.h"

@interface  ObjectAdaptor ()

//保存 被适配 的原始对象
@property (nonatomic, strong) Person *target;

@end

@implementation ObjectAdaptor

- (instancetype)initWithPerson:(Person *)person {
    self = [super init];
    
    if (self) {
        
        /** 保存适配的原始对象 */
        _target = person;
    }
    
    return self;
}

//////////////////////////////////////////////////////
/////原始接口的功能
//////////////////////////////////////////////////////

- (void)speakJapanese {
    //调用传入的对象说日语
    [_target speakJapanese];
}

- (void)speakEnglish {
    //调用传入的对象说英语
    [_target speakEnglish];
}

//////////////////////////////////////////////////////
/////新接口的功能
//////////////////////////////////////////////////////

- (void)speakFrance {
    NSLog(@"由Adaptor对象，提供说法语的功能");
}

@end
```

****

###默认适配器(抽象模板)

当想实现一个接口，但是又不想实现所有接口方法，只想去实现一部分方法时

####语言能力接口

```objc
#import <Foundation/Foundation.h>

/**
 *  语言能力抽象
 */
@protocol Language <NSObject>

//说日语
- (void)speakJapanese;

//说英语
- (void)speakEnglish;

@end
```

####所有接口方法的默认实现类（抽象模板类）

```objc
#import <Foundation/Foundation.h>

#import "Language.h"

@interface DefaultImplement : NSObject <Language>

@end
```

```objc
#import "DefaultImplement.h"

@implementation DefaultImplement

- (void)speakJapanese {}

- (void)speakEnglish {}

@end
```

####子类选择性实现

```objc
#import <Foundation/Foundation.h>

#import "DefaultImplement.h"

@interface SomeImpl : DefaultImplement

@end
```

```objc
#import "SomeImpl.h"

@implementation SomeImpl

- (void)speakEnglish {
    //选择性实现部分接口
}

@end
```