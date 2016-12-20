---
layout: post
title: "Runtime Part5 Class"
date: 2014-09-23 22:57:14 +0800
comments: true
categories: 
---

###通常我们编写一个类并继承自NSObject，然后编写所有的变量、所有的方法、以及实现的所有的协议....

****

###Foundation中提供了两个根类

####NSObject、平时使用最多的根类，提供了很多我们常用的api

基本的功能

```objc
- (BOOL)isEqual:(id)object;
@property (readonly) NSUInteger hash;
@property (readonly, copy) NSString *description;

...等等很多
```

发送消息

```objc
- (id)performSelector:(SEL)aSelector;
- (id)performSelector:(SEL)aSelector withObject:(id)object;
- (id)performSelector:(SEL)aSelector withObject:(id)object1 withObject:(id)object2;
```

类的初始化

```objc
+ (void)initialize {
	if (self == [NyClass class]) {
		// 初始化逻辑
	}
}
```

####NSProxy、用的比较少，只有需要实现代理对象的时候才会使用

NSProxy和NSObject层级一样的类，并作为抽象模板类，通常用于实现代理模式。

比如，经常我需要给一个对象的某一个方法实现在`运行时`添加一些额外的处理逻辑，那么此时的办法有这几种（注意，是运行时，而不是通过修改目标类的代码达到）:

```
第一种、重写一个子类继承自目标类，然后重写其扩展逻辑的方法实现
第二种、使用一个代理对象去代理目标类的对象，让代理对象接收一切这个目标类对象的消息
```

对于第一种方式，就直接使用一个子类去重写方法实现就没啥说的了。对于第二种，比较灵活一些，不用去子类化重写其方法实现，实现代理主要的步骤就是重写消息转发函数，在消息转发函数中做一些处理之后然后将消息再次转发给目标对象:

实现形式一、

```objc
// 直接消息转发给某个对象
- (id)forwardingTargetForSelector:(SEL)aSelector;
```

实现形式二、

```objc
// 目标函数签名
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector;

// 完整的消息转发
- (void)forwardInvocation:(NSInvocation *)anInvocation;
```



因为NSProxy类是作为抽象类，所以NSProxy类中没有`init对象方法`的实现。

也意味着NSProxy的`子类`必须实现至少一个init方法，才能符合OC中对象初始化规则。



***

###我很久以前对NSObject类的一个疑惑？

在runtime的开源代码中，找到[NSObject class]的实现代码如下

> 苹果的runtime开源地址: http://opensource.apple.com/source/objc4/

```objc
+ (Class)class
{  
    return self;  
}  
```

- 疑点一、该方法返回的就是`NSObject类自己`
- 疑点二、`NSObject类自己`是`objc_class`结构体类型的值
- 疑点三、self一般都是作为当前对象，难道当前类是`objc_class`结构体的一个实例？


> 根据这些疑问，我想是否可以有这种假设:

一个NSObject类，就是`objc_class`结构体的一个实例。


****

###找到Class指针类型所指向的结构体类型是`objc_class`，其恰好完整的描述了一个`类`所应该具备的数据

图示objc_class结构

![Snip20160521_4.png](http://imgchr.com/images/Snip20160521_4.png)

代码显示objc_class结构

```objc
struct objc_class {

	/* 指向该类的 MetaClass */
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__

	/* 指向其父类，如果该类为`RootClass`则值为 NULL */
    Class super_class                                        OBJC2_UNAVAILABLE;
    
    /* 类名字 */
    const char *name                                         OBJC2_UNAVAILABLE;
    
    /* 版本信息，默认为0 */
    long version                                             OBJC2_UNAVAILABLE;
    
    /* 类信息 */
    long info                                                OBJC2_UNAVAILABLE;
    
    /* 实例的总大小 */
    long instance_size                                       OBJC2_UNAVAILABLE;
    
    /* 实例变量Ivar链表 */
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    
    /* 实例Method链表 */
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    
    /* 实例Method的缓存 */
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    
    /* 实现的所有协议链表 */
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
/* Use `Class` instead of `struct objc_class *` */
```

 再从上面扣下 `objc_class` 中经常使用的属性

```c
/* 实例变量Ivar链表 */
struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    
/* 实例Method链表 */
struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    
/* 实例Method的缓存 */
struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    
/* 实现的所有协议链表 */
struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
```

从`objc_class`结构体的定义来看，我们所编写的`NSObject子类`的结构和`objc_class`结构体的结构几乎是完全吻合的（Ivar、Method、Property...等等）。


***

###继续回到`+[NSObject class]`这个函数实现

```objc
+ (Class)class
{  
    return self;  
}  
```

从表面意思来看:

- self代表的是当前NSObject类，也就是说当前NSObject类是一个对象

- 而对象的类型就是Class指针指向的类型 >>> `objc_class结构体`类型

结合起来，就是说: 

```
我们编写的一个NSObject类，就是一个objc_class结构体的实例。
```

乍听起来有点别扭，我们编写的类源文件就他妈一个本地文件，怎么就成了一个内存对象了？我是想了好久，也没弄明白啥意思。

关于NSObject类到底是谁的对象，我搜了很多的帖子和苹果文档都没有找到确切的答案。这样的话，可能就要去看苹果的底层源码了，这样的话对于我这样的菜鸟真的是太难了。

但是可以换一个角度去思考:

- 在`代码编写`阶段，我们所创建的一个NSObject类都是死的，就是Xcode工程里面的一个源文件而已
	- 没有分配内存的东西都是死的

- 而编`译器编译阶段`时，将我们写的 `NSObject类` 编译成 `objc_class结构体`这样的c语言源码，继而编译成二进制可执行文件

- 可执行`程序运行阶段`时，创建出objc_class实例并为其分配内存，来保存我们编写的NSObject类包含的信息:
	- Ivar成员变量
	- Proeprty属性
	- Method方法实现
	- 实现的Protocol
	- ....等等，都记录在内存中的一个 `objc_class`结构体实例中

那么从这个角度来看，得到两个点:

```
1. NSObject类是死的
2. 而程序运行时在内存中，用来表示NSObject类的objc_class结构体实例才是真正活的
```

那么一个代码中`死的NSObejct类`如何去与一个内存中的`objc_class实例`产生关系了？

我想可能是这样的，`NSObject类` 和 `objc_class结构体`之间可能通过某种方法进行互相转换。而且，在C++中的class与struct可以互相继承。

但都只是个人的一些猜测，那么`一个NSObject类是一个objc_class结构体实例`这句话好像是有点牵强，但可以肯定的是:`一个NSObject类在内存中运行时，一定对应一个唯一的objc_class实例`.

就目前个人能力，我无法彻底搞清楚NSObject类是如何作为一个objc_class结构体实例的过程，那可能得去看苹果更深的底层代码，可是暂时没这个时间，先放着吧....

但总之，我们编写的`一个NSObject类文件`是死的，但其对应的内存中的表示形式就是一个`objc_class结构体实例`。那我想，对于我这种喜欢走捷径的懒货来说，理解到这样的程度也已经够了。

***

###`NSObjct类`中的isa指针 与 `NSObject对象`中的isa指针:


- NSObjct类 中的 isa指针
	- 指向 当前 类 的 `Meta Class 元类`

- NSObjct对象 中的 isa指针
	- 指向 当前 对象 的 `所属类（NSObject子类）`

***


### objc_cache 结构体 对 `使用SEL 查询 Method List` 做的缓存优化

```
struct objc_cache {

	//总缓存长度
	unsigned int mask /* total = mask + 1 */ OBJC2_UNAVAILABLE; 
	
	//当前缓存长度
	unsigned int occupied OBJC2_UNAVAILABLE; 
	
	//保存Method实例的数组
	Method buckets[1] OBJC2_UNAVAILABLE; 
};
```


####搜索SEL对应的Method过程:

- 注意:
	- 发送对象消息、查询 `所属类的 method_list`
	- 发送类消息、查询 `Meta Class的 method_list`

- 1) 对象接到一个消息（包含SEL，参数列表）
- 2) 根据 `对象的isa指针` 查找到 所属类（NSObject子类）
- 3) 如果`objc_cache`存在这个SEL对应的Method，直接方法调用，并结束后面的步骤
- 4) 如果`objc_ objc_cache`不存在，那么就开始遍历isa指针指向的所属类的`method_list`
- 5) 找到SEL对应的Method
- 6) 找到函数实现的IMP
- 7) 完成函数执行
- 8)（最重要）将当前查找过的Method保存到 `objc_cache`

***

###区分 调用对象方法 or 调用类方法

- 对象方法

	- 根据 `对象 的isa指针`，找到所属的 `类`
	- 然后查询其 `method_list 或 cache`


- 类方法

	- 根据 `类 的isa指针`，找到所属的 `Meta Class 元类`
	- 然后查询其 `method_list 或 cache`


****

###看下经常使用类的几个系统方法的源码实现

- Objective-C中的

```objc
- (BOOL)isMemberOfClass:(Class)cls { 
    return [self class] == cls; 
}
```

```objc
- (Class)class { 
    return object_getClass(self); 
} 
```

- 最后调用的runtime c函数

```objc
//c函数
Class object_getClass(id obj) 
{ 
    return _object_getClass(obj); 
} 
```

```objc
//静态内联c函数
static inline Class _object_getClass(id obj) 
{ 
    if (obj) 
    	return obj->isa;//对象isa执行的所属类，类返回MetaClass 
    else
    	return Nil; 
} 
```

所以也就是说`[self class]`返回的就是当前`对象->isa`指向的`类`型

```objc
//对象的方法
- (Class)class
{  
	//返回对象的isa指针
    return (id)isa;   
}  

//类的方法
+ (Class)class
{  
	//返回自己
    return self;  
}  
```

****


####重写父类的方法
		
- `重写父类的方法`之后，并`不是覆盖掉了父类的方法`，只是在当前`子类`对象中找到了这个方法后就`不会再去父类`中找了。
		
- 但是如果，想执行了子类的方法之后，还要去执行父类的方法，那么只需要执行`[super 方法的SEL]`

给出一个父类与子类重写方法的模板，通常用于拦截父类的方法执行，在之前和之后做一些hook处理:
	

```objc
@interface Tool : NSObject

- (void)doWork;

@end
@implementation Tool

- (void)doWork {
    NSLog(@"Tool的方法实现");
}

@end
```

有时候由于Tool这个类的代码是闭源的拿不到，但是又需要测试一下，或者拦截做一些额外的处理，此时就可以继承自这个Tool类得到一个子类来重写要拦截的方法实现:

```objc
@interface ToolIntercepltor : Tool

@end
@implementation ToolIntercepltor

- (void)doWork {
    
    //1.
    NSLog(@"Tool doWork previous");
    
    //2.
    [super doWork];
    
    //3.
    NSLog(@"Tool doWork after");
}

@end
```

这样即可拦截Tool的doWork方法实现，在之前和之后做一些自己的处理。或者直接不让Tool的doWork方法实现执行。

***

###举例获取一个Block的最终真实Class类型

```objc
static force_inline Class XZHNSBlockClassGet() {
    static Class cls;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        
        //1. 获取一个NSBlock的对象
        void (^block)(void) = ^(){};
        
        //2. 获取当前Block的Class
        //通过对象的isa指针
        Class cls = [(NSObject *)block class];
        
        //3. 一直找到Block最终的superClass
        while (class_getSuperclass(cls) != [NSObject class]) {
            cls = class_getSuperclass(cls);
        }
    });
    return cls;
}
```

注意，后面知道这样获取得到是Block的`类簇类`NSBlock，但其实不是真正的Block对象类型。

****

###一直查找一个对象的类的`super_class`，结束条件是null

```objc
@implementation ViewController

- (void)test {

	Class cls = [self class];
    
    while (1) {
        NSLog(@"%@", cls);
        cls = class_getSuperclass(cls);
    }
}

@end
```

输出结果

```
2016-03-30 12:03:35.008 JSPatchDeme[12220:83909] ViewController
2016-03-30 12:03:35.009 JSPatchDeme[12220:83909] UIViewController
2016-03-30 12:03:35.009 JSPatchDeme[12220:83909] UIResponder
2016-03-30 12:03:35.009 JSPatchDeme[12220:83909] NSObject
2016-03-30 12:03:35.009 JSPatchDeme[12220:83909] (null)
2016-03-30 12:03:35.009 JSPatchDeme[12220:83909] (null)
2016-03-30 12:03:35.010 JSPatchDeme[12220:83909] (null)
2016-03-30 12:03:35.010 JSPatchDeme[12220:83909] (null)
2016-03-30 12:03:35.010 JSPatchDeme[12220:83909] (null)
...无限输出 null
```

也就是说OC中的顶层父类是NSObject，再获取super_class就是null了。


***

###运行时创建并注册一个类

```objc
#import "DynamicRegistClass.h"
#import <objc/runtime.h>
#import <objc/message.h>


static Class WidgetClass;

static void display(id self, SEL _cmd) {
    NSLog(@"被添加的方法实现被调用");
    
    //7. 设置实例变量值一、直接使用KVC
    [self setValue:@(0.99) forKey:@"height"];
    NSLog(@"height = %@", [self valueForKey:@"height"]);
    
    //8. 设置实例变量值一、runtime函数
    Ivar iavr = class_getInstanceVariable([self class], "height");
    object_setIvar(self, iavr, @(1.9999));
    float num = [object_getIvar(self, iavr) floatValue];
    NSLog(@"height = %f", num);
}

@implementation DynamicRegistClass

+ (void)initialize {
    
    //1. 创建一个继承自NSObject的子类
    WidgetClass = objc_allocateClassPair([NSObject class], "Widget", 0);
    
    //2. 向类添加一个OC方法
    const char *typeEncodings = "v@:";//void、id、sel
    class_addMethod(WidgetClass, @selector(display), (IMP)display, typeEncodings);
    
    //3. 向类添加一个实例变量
    const char *height = "height";
    class_addIvar(WidgetClass, height, sizeof(float), 0, @encode(float));
    
    //4. 注册类到运行时系统
    objc_registerClassPair(WidgetClass);
}

- (void)test {
    
    //1. 得到一个对象
    id obj = [[WidgetClass alloc] init];
    
    //2. 调用对象方法
    ((void (*)(id, SEL))(void *) objc_msgSend)(obj, @selector(display));
}

// 这个函数只是为了 @selector(display) 取消警告
- (void)display {}

@end
```

运行结果

```
2016-07-31 23:10:32.005 Demos[29027:419821] 被添加的方法实现被调用
2016-07-31 23:10:32.006 Demos[29027:419821] height = 0.99
2016-07-31 23:10:32.006 Demos[29027:419821] height = 1.999900
```

OK，和我们手动创建一个NSObject类文件的效果其实是一样的。

> 注意，只有通过运行时创建注册的类，才可以添加Ivar。如果是已经存在类文件.h和.m的类，是不能够再添加Ivar的。原因是对象的Ivar实例变量的内存布局是按照顺序一个接着一个排列的，如果运行时修改Ivar布局，就会导致全部的Ivar布局错乱。


