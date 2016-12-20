---
layout: post
title: "Runtime Part6 MetaClass"
date: 2016-02-12 23:46:40 +0800
comments: true
categories: 
---

##Meta Class 元类
	
年前在看`YYModel`源码，刚好碰到关于MetaClass相关东西，之前也一直模模糊糊。过年在家休息没事就学习学习，新的码农一年又开始了...

****


###Objective-c 中的 每一个类 `如: Person类` 其实包含两部分

#### 一、普通类（类本身）

- 如、Person类本身(我们创建的类)

#### 二、元类（系统创建）

- Meta Person类，由系统默认创建
- 只不过这个创建Meta Person类的过程，我们不知道而已

***

###先不管神马的，看看关于meta class的系统方法

- 判断一个objc_class实例是否是Meta类的Class

```c
BOOL class_isMetaClass(Class cls)
```

- 获取一个NSObject类对应的Meta类的Class

```c
Class objc_getMetaClass(const char *name)
```

我就只找到这两个相关的方法.

****

##类 与 Meta类

- 类，存放`对象`的相关数据
	- 实例成员变量
	- 实例方法

- Meta类，存放`类`的先关数据
	- 类的成员变量（static 类成员变量）
	- 类方法（+开头的方法）

- 区分 对象方法 与 类方法 调用过程

	- 向一个`对象`发送消息

		- 通过`对象的 isa指针`找到 对象所属类对应的的 objc_class结构体实例
		- 然后开始查找`objc_class实例`的 `method_list` 查找 Method

	- 向一个`类`发送消息

		- 通过`类Class的 isa指针`找到 Meta类对应的 objc_class结构体实例
		- 然后从`objc_class实例`的 `method_list` 查找 Method



***

##运行时动态创建一个NSObejct子类，并 添加方法Method、添加变量Ivar、获取Ivar以及值.

```objc
//
//  ViewController.m
//
//  Created by xiongzenghui on 14/1/14.
//  Copyright © 2014年 xiongzenghui. All rights reserved.
//

#import "ViewController.h"

void ReportFunction(id self, SEL _cmd)
{
    //1. 对象
    NSLog(@"This object is %p.", self);
    
    //2. 对象所属的类
    NSLog(@"Class is %@", [self class]);
    
    //3. 所属类的父类
    NSLog(@"super is %@.",[self superclass]);
    
    //4. 每一个类都有两部分
    //类的第一部分、类本身
    //类的第二部分、元类
    Class currentClass = [self class];
    for (int i = 1; i < 10; i++)//i的次数随便改都可以
    {
        NSLog(@"Following the isa pointer times = %d, ClassValue = %@, ClassAddress = %p", i, currentClass, currentClass);
        
        //通过Class的 isa指针 找到 MetaClass
        currentClass = object_getClass(currentClass);
    }
    
    //5. NSObject类本身
    NSLog(@"NSObject's class is %p", [NSObject class]);
    
    //6. NSObject类的元类
    NSLog(@"NSObject's meta class is %p", object_getClass([NSObject class]));
}

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
   [super viewDidLoad];
	
	//运行时创建类 	
	[self createClass];   
}

- (void)createClass
{
    //1. 创建一个Class
    Class MyClass = objc_allocateClassPair([NSObject class],
                                           "myclass",
                                           0);
    
    //2. 添加一个NSString的变量，第四个参数是对其方式，第五个参数是参数类型
    if (class_addIvar(MyClass, "itest", sizeof(NSString *), 0, "@")) {
        NSLog(@"add ivar success");
    }
    
    //3. 添加一个函数
    class_addMethod(MyClass,
                    @selector(report),
                    (IMP)ReportFunction,
                    "v@:");
    
    //4. 注册这个类到runtime系统中就可以使用他了
    objc_registerClassPair(MyClass);
    
    //5. 测试创建的类
    [self test:MyClass];
}

- (void)report {
    //什么都不做，只是为了OC对象能够调用到c函数
}

- (void)test:(Class)class {
    
    //1.
    id obj = [[class alloc] init];
    
    //2.
    [obj report];
}

@end
```

最终控制台输出如下

```
2016-02-13 22:49:52.725 XZHRequest[1862:23623] add ivar success
2016-02-13 22:49:52.726 XZHRequest[1862:23623] This object is 0x7fed59c8aea0.
2016-02-13 22:49:52.726 XZHRequest[1862:23623] Class is myclass
2016-02-13 22:49:52.726 XZHRequest[1862:23623] super is NSObject.
2016-02-13 22:49:52.726 XZHRequest[1862:23623] Following the isa pointer times = 1, ClassValue = myclass, ClassAddress = 0x7fed59c84fa0
2016-02-13 22:49:52.726 XZHRequest[1862:23623] Following the isa pointer times = 2, ClassValue = myclass, ClassAddress = 0x7fed59c87560
2016-02-13 22:49:52.726 XZHRequest[1862:23623] Following the isa pointer times = 3, ClassValue = NSObject, ClassAddress = 0x10cc15198
2016-02-13 22:49:52.726 XZHRequest[1862:23623] Following the isa pointer times = 4, ClassValue = NSObject, ClassAddress = 0x10cc15198
2016-02-13 22:49:52.727 XZHRequest[1862:23623] Following the isa pointer times = 5, ClassValue = NSObject, ClassAddress = 0x10cc15198
2016-02-13 22:49:52.727 XZHRequest[1862:23623] Following the isa pointer times = 6, ClassValue = NSObject, ClassAddress = 0x10cc15198
2016-02-13 22:49:52.727 XZHRequest[1862:23623] Following the isa pointer times = 7, ClassValue = NSObject, ClassAddress = 0x10cc15198
2016-02-13 22:49:52.727 XZHRequest[1862:23623] Following the isa pointer times = 8, ClassValue = NSObject, ClassAddress = 0x10cc15198
2016-02-13 22:49:52.782 XZHRequest[1862:23623] Following the isa pointer times = 9, ClassValue = NSObject, ClassAddress = 0x10cc15198
2016-02-13 22:49:52.782 XZHRequest[1862:23623] NSObject's class is 0x10cc15170
2016-02-13 22:49:52.783 XZHRequest[1862:23623] NSObject's meta class is 0x10cc15198
```

- 方法`object_getClass(id obj)`会根据 `Class的私有成员变量isa指针` 找到一个类，有两种情况:
	- 对象的isa >>> 所属类
	- 类的isa >>> Meta 类
	- Meta类的isa >>> Meta NSObject
	- Meta NSObject的isa >>> 永远指向自己，形成环路

- times = 2时、根据`myclass->isa`指针，找到其对应的 `Meta myclass`

- times = 3时、根据 `Meta myclass->isa` 指针，找到了 `Meta NSObject`

- times = 4，5，6，7，8...、根据 `元类NSObject` 的 isa指针，最后`都是指向自己`

***

###再记录一个例子，使用`imp_implementationWithBlock()`替代上面例子需要一个辅助的static c函数来完成运行时创建Class.

```objc
//////////////////////////////////////////////////////
///创建一个类
//////////////////////////////////////////////////////
Class People = objc_allocateClassPair([NSObject class], "People", 0);
    
//////////////////////////////////////////////////////
///添加两个变量
//////////////////////////////////////////////////////

if (class_addIvar(People, "name", sizeof(NSString *), 0, @encode(NSString *))) {
    NSLog(@"add ivar name success");
}
if (class_addIvar(People, "age", sizeof(int), 0, @encode(int))) {
    NSLog(@"add ivar age success");
}
    
//////////////////////////////////////////////////////
///创建方法的SEL
//////////////////////////////////////////////////////
SEL selector = sel_registerName("talk:");
    
//////////////////////////////////////////////////////
///创建方法的IMP指针，并指向Block给出的代码
//////////////////////////////////////////////////////
IMP impl = imp_implementationWithBlock(^(id self, NSString *arg1){
    
    //age变量值
	//通过KVC
	//int age = (int)[[self valueForKey:@"age"] integerValue];
	
	//通过Ivar
    Ivar ageIvar = class_getInstanceVariable([self class], "age");
    int age  = (int)[object_getIvar(self, ageIvar) integerValue];
    
    
    //name变量值
    //通过KVC
	//NSString *name = [self valueForKey:@"name"];
	
	//通过Ivar
    Ivar nameIvar = class_getInstanceVariable([self class], "name");
    NSString *name = object_getIvar(self, nameIvar);
    
    NSLog(@"age = %d, name = %@, msgSay = %@", age, name, arg1);
});
    

//////////////////////////////////////////////////////
///添加一个方法, 将SEL与IMP组装成一个Method结构体实例，添加到Class中的 method_list数组
//////////////////////////////////////////////////////
class_addMethod(People, selector, impl, "v@:@");

//////////////////////////////////////////////////////
///注册这个类到系统
//////////////////////////////////////////////////////
objc_registerClassPair(People);
    

//////////////////////////////////////////////////////
///生成一个实例
//////////////////////////////////////////////////////
id instanceP = [[People alloc] init];
    
//////////////////////////////////////////////////////
///给Ivar赋值
//////////////////////////////////////////////////////

//通过KVC赋值
[instanceP setValue:@"变量字符串值" forKey:@"name"];
//[instanceP setValue:@19 forKey:@"age"];

//通知Ivar赋值
Ivar ageIvar = class_getInstanceVariable(People, "age");
object_setIvar(instanceP, ageIvar, @19);

//////////////////////////////////////////////////////
///发送消息
//////////////////////////////////////////////////////
((void (*)(id, SEL, NSString *))(void *) objc_msgSend)(instanceP, selector, @"参数值");
    
//////////////////////////////////////////////////////
///释放对象、销毁类
//////////////////////////////////////////////////////
instanceP = nil;
objc_disposeClassPair(People);
```

- 方法的编码格式: 
	- `v@:` >>> 返回值void，参数1:self,参数2:SEL >>> `- (void)funcName;`
	- `v@:@` >>> 返回值void，参数1:self,参数2:SEL, 参数3:NSString* >>>  `- (void)funcName:(NSSring *)name;`

- 对应的Objective-C方法的SEL也应该是 >>> `talk:`带一个参数

- imp_implementationWithBlock()接收一个添加方法被调用时的回调Block
	- 其格式为: `method_return_type ^(id self, method_args …)`

***

##Meta Class 元类

```
Class objc_allocateClassPair(Class superclass, 
							 const char *name, 
							 size_t extraBytes) 
```

- 函数名 `objc_allocateClassPair..` 中的 `ClassPair`意思是说`一对Class`.

- 但是该函数只返回了`一个Class`

- 没有被返回的`另一个Class`就是 `MetaClass 元类`，由系统创建.

- Meta Class与普通类Class 都是结构体 `objc_class`实例

***

##Objective-c中的 对象 与 类

###类 

经常写获取一个对象的类型Class的代码

```
Class cls = [Person class];
```

就用到了`Class`这个数据结构，其定义为如下，可以得到Class是 结构体objc_class实例的 `指针类型`

```
typedef struct objc_class *Class;
```

那么能够成为一个类（Class）的数据结构为

```
struct objc_class {

	/* 
		又指向一个结构体objc_class实例 
		类的isa指针，指向类的 元类 MetaClass
	*/
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__

	/**
		指向其父类(普通类)
	 */
    Class super_class                                        OBJC2_UNAVAILABLE;
    
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    
    /**
    	方法列表
     */
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    
    /**
    	方法缓存
     */

    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
/* Use `Class` instead of `struct objc_class *` */
```

而Class数据结构中最重要的一个数据项是 `Class isa;` 这样的isa指针.

###对象

能够成为一个对象（Object）的数据结构如下，可以看到对象数据结构中最重要的就是`Class isa;`指针项

```
struct objc_object {

	/**
		对象的isa指针，指向其 所属类本身（不是元类）	
	 */
    Class isa  OBJC_ISA_AVAILABILITY;
};
```

Class是`struct objc_class`的指针类型，那么`struct objc_object`也有其对于的指针类型


###调用类方法与对象方法

- 给`对象`发送消息时，消息是在`对象所属类`的方法列表method_list或cache中查询Method

- 给`类`发消息时，消息是在`这个类的元类 meta class`的方法列表或缓存中查询Method

####所以也就是说`对象方法`与`类方法`存放地点的不同

- 对象方法、存放在 `类本身`
- 类方法、存放在 `类的元类MetaClass`

**所以，元类是必不可少的，它存储了`类的所有类方法`**

***

###对象、类、元类三者之间的关系图

![](http://i3.piimg.com/f0be13b08322042d.png)

![](http://i2.piimg.com/02e5f1ad0ed1a2e4.png)

- 对象的 isa指针 指向 所属类

- 每一个类，都有一个 `isa指针` 指向一个 `唯一的 MetaClass`

- 每一个Meta Class，拥有的 `isa指针` 都指向 `顶层 NSObject Meta Class`

- 最上层的Meta Class（Meta NSObject）的isa指针指向自己，形成一个 `回路`

- 类本身的 super_class 指向其父类，如果该类为根类则值为 NULL

- 每一个 MetaClass 的 `super_class指针` 都指向其 `原本Class的superClass对应的 MetaClass`
	- 特列: 但是最上层的 Meta Class 的 `super_class 指针`，指向的却是 `NSObject 原本 Class`

***

###所有的 `类 NSObject子类` 都是 `objc_class结构体` 的实例

NSObject类获取Class的源码如下:

```objc
+ (Class)class {
    return self;
}
```

- Class是 `objc_class结构体`实例的指针变量类型
- 那么`+[NSObject class]`返回就是一个 `objc_class结构体实例`
- 而 `self` 一般是 `一个指向对象地址的指针`
- 而此时的 self 所代表的就是 `NSObject类本身`

####那么可以猜测: **NSObject 是 objc_class的实例**

```objc
//名字
NSLog(@"%@", [Person class]);

//多次输出其地址
NSLog(@"%p", [Person class]);
NSLog(@"%p", [Person class]);
NSLog(@"%p", [Person class]);
```

输出结果

```
2016-02-13 22:44:41.509 XZHRequest[1571:18970] Person
2016-02-13 22:44:41.510 XZHRequest[1571:18970] 0x10c1853e0
2016-02-13 22:44:41.510 XZHRequest[1571:18970] 0x10c1853e0
2016-02-13 22:44:41.510 XZHRequest[1571:18970] 0x10c1853e0
```

可以得到

- 每一个NSObject类在运行阶段，是按照一个`单例`保存


****

###`-[NSObject isKindOfClass:]` 查找对象是否是某一个类的对象的源码

```objc
- (BOOL)isKindOfClass:(Class)aClass
{
	Class cls;
	
    for (`cls = isa`; cls; cls = `cls->super_class`) 
        if (`cls == (Class)aClass`)
            return YES;
            
    return NO;
}
```

- 首先是 `cls = isa`，保存 `当前对象的isa指针`

- 然后每一次使用`类的 super_class指针`找到其 父类Class

- 最终循环停止条件是 `NSObject类本身的 superClass = nil`

- 每一次遍历时直接比较`Class对象是否相等`

***

###`-[NSObject isMemberOfClass:]` 源码

```objc
- (BOOL)isMemberOfClass:(Class)aClass
{
	return isa == (Class)aClass;
}
```

- 获得`当前对象的isa指针指向的类`
- 然后直接一次性比较`Class对象是否相等`

****