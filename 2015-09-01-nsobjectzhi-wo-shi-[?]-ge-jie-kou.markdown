---
layout: post
title: "NSObject之我是一个接口"
date: 2015-09-01 21:21:23 +0800
comments: true
categories: 
---

###无意中打开了一个帖子，看到一句话说NSObject是一个@protocol.我震惊了之前一直认为是一个具体类，不行我的看看到底是不是.

***

###No1. 我打开一个Xcode项目，找到 <objc/NSObject.h>，发现确实是有一个NSObject的protocol，难道我一直集成的父类是一个protocol？协议还可以当具体类使用？


NSObject的protocol定义如下:

```
#include <objc/objc.h>
#include <objc/NSObjCRuntime.h>
@class NSString, NSMethodSignature, NSInvocation;
```

```
@protocol NSObject

//---------------------------------------------

//对象描述
@property (readonly, copy) NSString *description;

//每个对象的hash值
@property (readonly) NSUInteger hash;

//父类Class
@property (readonly) Class superclass;

//---------------------------------------------

//对象的equal方法
- (BOOL)isEqual:(id)object;

//获取Class
- (Class)class;

//原来真的可以这么调用 [self self]或self.self
- (instancetype)self;

//---------------------------------------------

//向NSRunloop添加NSTimer后，像对象发送消息（简单的理解就是调用函数）.
- (id)performSelector:(SEL)aSelector;
- (id)performSelector:(SEL)aSelector withObject:(id)object;
- (id)performSelector:(SEL)aSelector withObject:(id)object1 withObject:(id)object2;

//---------------------------------------------

- (BOOL)isProxy;

//---------------------------------------------

- (BOOL)isKindOfClass:(Class)aClass;
- (BOOL)isMemberOfClass:(Class)aClass;
- (BOOL)conformsToProtocol:(Protocol *)aProtocol;

//---------------------------------------------

- (BOOL)respondsToSelector:(SEL)aSelector;

//---------------------------------------------

- (instancetype)retain OBJC_ARC_UNAVAILABLE;
- (oneway void)release OBJC_ARC_UNAVAILABLE;
- (instancetype)autorelease OBJC_ARC_UNAVAILABLE;
- (NSUInteger)retainCount OBJC_ARC_UNAVAILABLE;

//---------------------------------------------

- (struct _NSZone *)zone OBJC_ARC_UNAVAILABLE;

//---------------------------------------------

//可选实现的方法

@optional
@property (readonly, copy) NSString *debugDescription;

@end
```


继续往下找，这不科学，协议只是一个抽象的声明，没任何的实现代码，怎么能会可以直接使用.

***

###No2. 找到了NSObject Protocol的实现类，类的名字其实也叫做NSObject.

```
@interface NSObject <NSObject> {
	Class isa  OBJC_ISA_AVAILABILITY;
}

//...

@end
```

因为无法看到实现代码....
（后面找到查看代码的方式了，苹果已经开源runtime的相关代码.）

***

###No3. 从NSObject Protocol中分析一、oc中发送执行方法的消息只有2中方法:

第一种估计会OC的都知道.

```
[对象 方法名] 或 [类 方法名]

```

第二种貌似大部分也知道，只有我不知道...

```
- (id)performSelector:(SEL)aSelector;
- (id)performSelector:(SEL)aSelector withObject:(id)object;
- (id)performSelector:(SEL)aSelector withObject:(id)object1 withObject:(id)object2;
```

***

###No3. 从NSObject Protocol中分析二、方法签名

> 用来描述这个方法的所有信息: 参数类型、参数个数、返回值 ..等.


在NSObject Protocol中有如下2个方法定义，用于获取方法的签名.

```
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector;

+ (NSMethodSignature *)instanceMethodSignatureForSelector:(SEL)aSelector;

```

> 方法的SEL值 VS 方法的签名 ？

```
SEL: 只是一个字符串，仅仅能够唯一标示一个方法的IMP.
```

```
Siganature: 包含了方法的参数信息、返回值信息..等等.
```

当要执行一个方法时，iOS系统根据方法名字转化成SEL值。然后找到函数的实现指针IMP，继而执行函数。

而执行函数，就要确定对传入的参数如何处理，以及返回值如何处理？
这个时候，方法的签名就起作用了，告诉系统如何解决如上的2个问题.

搜了下NSMethodSignature，然后与之紧密关联的是一个叫做【NSInvocation】的东西，来看一下是什么？

***

###No4. 顺藤摸瓜，认识NSInvocation是什么？

```
（纯属个人理解）

就是让我们自己的代码，也可以【发出一个消息】，用于执行消息相关的函数.
```

NSInvocaion我们手动触发一个消息.

```
//1.获取一个方法的签名（详细描述信息:参数类型、返回值类型...）
NSMethodSignature *methodSignature = [[self class] instanceMethodSignatureForSelector:@selector(myLog:)];
    
//2. 使用NSMethodSignature对象，生成NSInvocation对象.
NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:methodSignature];
    
//3. 设置NSInvocation对象【向哪个对象发送消息】.
[invocation setTarget:self];
    
//4. 设置NSInvocation对象【发送哪一个消息】.
[invocation setSelector:@selector(myLog:)];
    
//5. 设置【函数最终执行需要的参数】（注意: index=0是target参数，index=1是SEL参数，我们自己的参数从index=2开始）
    
NSString *argument = @"哈士奇";
[invocation setArgument:(__bridge void *)(argument) atIndex:2];
    
//6. 手动的【发送消息】
[invocation invoke];
```

***

###No5. NSObject对象属性isa.

```
@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}
```

NSObejct类对象，存在一个属性叫做isa.

```
typedef struct objc_class *Class;
```

这一看，还是一个指向 objc_class结构体变量类型的指针变量.

> isa指针出现在两个地方: 1)NSObject类对象 2)objc_class结构体 


第一个出现点: NSObject

如下都是苹果开源的代码.

```
- (BOOL)isMemberOfClass:(Class)cls {
  return [self class] == cls;
}
```

注意: 这个 对象的 calss方法（还有一个类的class方法）

```
- (Class)class {
  return object_getClass(self);
}
```

```
Class object_getClass(id obj)
{
  return _object_getClass(obj);
}
```

```
static inline Class _object_getClass(id obj)
{
  if (obj) {
  		return obj->isa;
  } else {
	  return Nil;
  }
}
```

在上面的最后一个函数_object_getClass中，直接通过对象的isa属性，就可以获取对应的Class类型.

第二个出现点: 结构体 objc_class

```
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
```

从NSObject协议的实现类NSObject类中，可以找到还有一个类的class方法.

```
+ (Class)class;
```

然后找到开源的代码函数实现

```
+ (Class)class {
    return self;
}
```

代码中使用了self.
大多情况下，self表示当前对象（处于对象方法），而这里是类的方法.

是不是可以这么说===>>> 当前类NSObject也是一个对象.

那么self表示当前对象就不矛盾了.

那么NSObject这个类又是谁的对象（实例）了？
首先从runtime源码中未找到，完后百度了一下发现也没有啥有价值的帖子能够正确的描述出来.

