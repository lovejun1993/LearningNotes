---
layout: post
title: "Runtime Part8 Associate"
date: 2014-09-24 00:13:26 +0800
comments: true
categories: 
---

###在运行时，给对象关联一个其他对象，解决分类无法添加变量的缺陷.

- Category只能添加`方法`，而不能添加成员变量

- 动态添加成员变量，只能由`object associate`完成

- 主要是两个c函数

```
1. 把一个对象与另外一个对象进行关联

objc_setAssociatedObject的四个参数含义
> id object，给谁设置关联对象。
> const void *key，关联对象唯一的key，获取时会用到。
> id value，被关联对象。
> objc_AssociationPolicy，关联策略


void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
```

```
2. 获取相关联的对象

id objc_getAssociatedObject(id object, const void *key)
```

***

###使用Category+objc_associate结合使用的例子，给类动态添加成员变量+setter与getter

- `objc_associate`让一个对象A能够在运行时，使用一个key来绑定另一个对象B。

- `Category分类`提供`操作被绑定的对象`的函数入口

***

###eg1. 给Person类动态关联一个对象

- 原始 Person类 

```
#import <Foundation/Foundation.h>

@interface Person : NSObject

@property (nonatomic, copy) NSString *name;

@end

@implementation Person

@end
```

- `Person+Associate分类`，完成给原始类Person，动态添加成员变量，以及setter与getter

```
#import "Person.h"

@interface Person (Associate)

/**
 *	提供设置动态添加的变量值的入口
 */
- (void)setAddress:(NSString *)addr;

/**
 *	提供获取动态添加的变量值的入口
 */
- (NSString *)getAddress;

@end

```
	
```
#import "Person+Associate.h"
#import <objc/runtime.h>
	
static NSString *addressKey = nil;

@implementation Person (Associate)

- (void)setAddress:(NSString *)addr {
    
    objc_setAssociatedObject(self,
                             &addressKey,
                             addr,
                             OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    
}

- (NSString *)getAddress {
    return objc_getAssociatedObject(self, &addressKey);
}

@end
```

***

###eg2. 使用UIView分类，完成给任意UIView对象，添加手势点击操作，并接受Block回调

```
UIView+Associate.h

#import <UIKit/UIKit.h>

@interface UIView (Associate)

/**
 *  动态的给UIView实例，添加一个TapGesture操作
 */
- (void)x_setTapActionWithBlock:(void (^)(void))block;

//注意，没有写get TapGesture的函数入口，是因为不需要得到手势对象，只需要点击回调即可

@end
```

```
UIView+Associate.m

#import "UIView+Associate.h"
#import <objc/runtime.h>

//用来绑定Block成员变量的key
static NSString *kXActionHandlerTapBlockKey;

//用来绑定TapGesture成员变量的key
static NSString *kXActionHandlerTapGestureKey;

@implementation UIView (Associate)

- (void)x_setTapActionWithBlock:(void (^)(void))block {
    
    //1. 先获取与当前UIView实例绑定的手势对象
    UITapGestureRecognizer *gesture = objc_getAssociatedObject(self, &kXActionHandlerTapBlockKey);
    
    //2. 如果还没有绑定一个手势对象，那么先创建手势，后绑定给当前UIView实例
    if (!gesture) {
        
        //创建手势对象
        gesture = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(x_handleActionForTapGesture:)];
        
        //将手势添加到当前UIView实例
        [self addGestureRecognizer:gesture];
        
        //绑定给当前UIView实例
        objc_setAssociatedObject(self,
                                 &kXActionHandlerTapGestureKey,
                                 gesture,
                                 OBJC_ASSOCIATION_RETAIN);
    }
    
    //3. 将传入的Block绑定给当前UIView实例
    objc_setAssociatedObject(self,
                             &kXActionHandlerTapBlockKey,
                             block,
                             OBJC_ASSOCIATION_COPY_NONATOMIC);
}

/**
 *  手势识别对象的target和action
 */
- (void)x_handleActionForTapGesture:(UITapGestureRecognizer *)gesture {
    
    //判断手势操作结束
    if (gesture.state == UIGestureRecognizerStateRecognized) {
        
        //获取与当前UIView实例绑定的Block
        void (^block)(void) = objc_getAssociatedObject(self, &kXActionHandlerTapBlockKey);
        
        if (block) {
            block();
        }
    }
}

@end
```