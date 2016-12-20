---
layout: post
title: "Runtime Part4 Ivar Property"
date: 2014-09-23 19:21:10 +0800
comments: true
categories: 
---

###成员变量 与 属性 是不同的

- 成员变量 Ivar

> 仅仅只是一个对象的内部使用的 变量

- 属性 Property

> 首先包括成员变量，并且还包括对成员变量操作的 setter方法与getter方法

***

###成员变量的相关操作

####Ivar 结构体定义

```
typedef struct objc_ivar *Ivar;
```

```
struct objc_ivar {
	char *ivar_name OBJC2_UNAVAILABLE; // 变量名
	char *ivar_type OBJC2_UNAVAILABLE; // 变量类型
	int ivar_offset OBJC2_UNAVAILABLE; // 基地址偏移字节
	#ifdef __LP64__
	int space OBJC2_UNAVAILABLE;
	#endif
}
```

***

###属性的相关操作

#### Property 结构体定义

```
typedef struct objc_property *objc_property_t;
```

####获取类实例的`属性列表`

```
id PersonClass = objc_getClass("Person");
    
unsigned int outCount;
objc_property_t *properties = class_copyPropertyList(PersonClass, &outCount);
    
for (int i = 0; i < outCount; i++) {
    
    objc_property_t property = properties[i];
    
    const char* propertyName = property_getName(property);
    const char* propertyAttribute = property_getAttributes(property);
    
    fprintf(stdout, "属性名 = %s , 属性类型 = %s\n", propertyName, propertyAttribute);
}
```

```
如上代码执行结果:

属性名 = car  , 属性类型 = T@"Car",&,N,V_car
属性名 = name , 属性类型 = T@"NSString",C,N,V_name
属性名 = age  , 属性类型 = Tq,N,V_age
```


###关于属性的attributes类型

常见属性的编码字符串

```
属性类型 = T@"Car",&,N,V_car
属性类型 = T@"NSString",C,N,V_name
属性类型 = Tq,N,V_age
```

一个属性的编码结构体（编码字符串的拆分结构）

```c
typedef struct {
     const char *name; // 特性名
     const char *value; // 特性值
} objc_property_attribute_t;
```

****

###安全的使用KVC

```objc
//
//  NSObject+Properties.h
//  XControllerManager
//
//  Created by xiongzenghui on 16/1/25.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import <Foundation/Foundation.h>

/**
 *  控制key与value的安全使用KVC
 */
@interface NSObject (Properties)

- (BOOL)hasPropertyWithName:(NSString *)name;

- (BOOL)canSetValueForPropertyWithName:(NSString *)name;

- (void)setValuesAndKeys:(NSDictionary *)valuesAndKeys;

@end
```

```objc
//
//  NSObject+Properties.m
//  XControllerManager
//
//  Created by xiongzenghui on 14/1/25.
//  Copyright © 2014年 xiongzenghui. All rights reserved.
//

#import "NSObject+Properties.h"
#import <objc/runtime.h>

@implementation NSObject (Properties)

- (BOOL)hasPropertyWithName:(NSString *)name
{
    objc_property_t property = [self _propertyWithName:name];
    
    if (property == NULL) {
        return NO;
    } else {
        return YES;
    }
}

- (BOOL)canSetValueForPropertyWithName:(NSString *)name
{
    if (![self hasPropertyWithName:name]) {
        return NO;
    }
    
    //1. 获取指定名字的属性
    objc_property_t property = [self _propertyWithName:name];
    
    //2. 获取属性的 编码字符串 中是否包含 "R" （只读readonly）
    char *attr = property_copyAttributeValue(property, "R");
    
    return !attr;
}

- (void)setValuesAndKeys:(NSDictionary *)valuesAndKeys
{
    for (id key in valuesAndKeys) {
        
        id value = [valuesAndKeys objectForKey:key];
        
        //判断属性是否存在、是否可写
        if (![self canSetValueForPropertyWithName:key]) {
            continue;
        }
        
        //KVC赋值
        [self setValue:value forKey:key];
    }
}

- (objc_property_t)_propertyWithName:(NSString *)name
{
    if (!name || [name isEqualToString:@""])
        return NULL;
    
    objc_property_t property = class_getProperty([self class], [name UTF8String]);
    
    return property;
}

@end
```

###Type Encodings，之前输出的类似`T@"Car",&,N,V_car`


> https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html


####我们使用的一些`数据类型`，实际上最终编译器会解析成类似`T@"Car",&,N,V_car`这样的一串字符

```objc
Person类的所有属性（Ivar）的 `最终数据类型` 由编译器编译成一串特定格式的字符

属性名 = car  , 属性类型 = T@"Car",&,N,V_car
属性名 = name , 属性类型 = T@"NSString",C,N,V_name
属性名 = age  , 属性类型 = Tq,N,V_age
```

```
Car *car;自定义类型 ---> T@"Car",&,N,V_car
```

```
NSString *name; ----> T@"NSString",C,N,V_name
```

```
NSInteger age; ----> Tq,N,V_age
```

总而言之，苹果会对我们在代码中使用的所有数据类型，进行额外的一次编码，编码成最终iOS系统所能识别的数据类型.

####Objective-C 中的所有的 `数据类型` 的系统编码格式

| 最终苹果的编码格式 | iOS代码中对应的数据类型 | 
|:-------------:|:-------------:| 
| c | char | 
| i | int | 
| s | short |
| l | long (注意: long is treated as a 32-bit quantity on 64-bit programs) |
| q | long long  |
| C | unsigned char |
| I | unsigned int |
| S | unsigned short |
| L | unsigned long |
| Q | unsigned long long |
| f | float |
| d | double |
| B | C++ bool or a C99 _Bool |
| v （小写） | void |
| * | character string (char *) |
| @ | 自定义类型的对象，An object (whether statically typed or typed id) |
| # | class object (Class) |
| : | A method selector (SEL) |
| [array type] | array数组 |
| {name=type...} |  structure 结构体 |
| (name=type...) | union 共用体 |
| bnum | A bit field of num bits |
| ^type | A `pointer` to type |
| ? | An unknown type (among other things, this code is used for `function pointers 函数指针`) |

####举例编码格式

- 假设存在如下的结构体

```
typedef struct example {
    id   anObject;
    char *aString;
    int  anInt;
} Example;
```

- 使用@encode()快速获取数据类型的编码格式

```
@encode(Example);
```

- 结构体Example 的数据类型编码

```
{example=@*i}
```

- 结构体Example  `指针` 的数据类型编码

```
^{example=@*i}
```

####Objective-C `method 方法` encodings

| 最终苹果的编码格式 | iOS代码中对应的数据类型 | 
|:-------------:|:-------------:| 
| r | const | 
| n | in | 
| N | inout | 
| o | out | 
| O | bycopy | 
| R | byref | 
| V | oneway |
