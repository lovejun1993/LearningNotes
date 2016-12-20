---
layout: post
title: "Runtime Part7 object_getClass()与[object class]的区别"
date: 2016-05-17 17:53:35 +0800
comments: true
categories: 
---

##刚好在看KVO的东西的时候，忽然遇到一个问题: 

```c
Class cls1 = object_getClass(object);
Class cls2 = [object class];
```

这两个Class有什么区别？一时间真的说不出来有什么区别，去补补...

##之前的runtime文章记录过: `类 Class` 、`元类 Meta-Class` 、`对象 object` 三者之间的关系

来自网络的一个结构图:

![](http://i3.piimg.com/f0be13b08322042d.png)

那么文字总结下:

- 对象的 isa指针 指向 所属类

- 每一个类，都有一个 `isa指针` 指向一个 `唯一的 MetaClass`

- 每一个Meta Class，拥有的 `isa指针` 都直接指向 `顶层 NSObject Meta Class`

- 最上层的Meta Class（Meta NSObject）的`isa指针`指向自己，形成一个 `回路`

- 每一个`类本身` 的 `super_class指针` 指向其父类，如果该类为根类则值为 NULL

- 每一个 `MetaClass` 的 `super_class指针` 都指向其 `Super MetaClass`
	- 特别情况: 最顶层 NSObject Meta Class 的 `super_class指针`，指向的却是 `NSObject 本身 Class`

##现在应该可以清晰的明白: Object（实例），Class（类）,Metaclass（元类），Rootclass(根类)，Rootclass's metaclass(根元类) 这些东西

- 对于NSObject的所有子类
	- Object（实例/对象）
	- Class（类本身）
	- Metaclass（元类）

- 对于NSObject类自己 
	- 类分为两个版本:（普通类、元类）
	- Rootclass(根类本身)
	- Rootclass's metaclass(根元类)

- 一个NSObject类其实分为两部分: 1)对象方法实现存放类 2)类方法实现存放类

虽然没经过权威证实，我们写的每一个NSObejct类最终都会被`objc_registerClassPair()`注册到runtime system系统中去。但是我觉得只有这么解释，才能说的通`已经存在源文件NSObject类是无法使用runtime api添加Ivar的`问题。


那么看下`objc_registerClassPair()`的demo代码:

```objc
// 1. 运行时创建一个类
Class MyClass = objc_allocateClassPair([NSObject class], "myclass", 0);
    
// 2. 添加一个NSString的实例变量，第四个参数是对其方式，第五个参数是参数类型编码
if(class_addIvar(MyClass, "itest", sizeof(NSString *), 0, "@")) {
    NSLog(@"add ivar success");
} else {
    NSLog(@"add ivar failed");
}
    
// 3. 向运行时系统注册创建的类
objc_registerClassPair(MyClass);
```

如上代码就是向runtime system注册一个没有源文件的NSObject类的代码。

`objc_allocateClassPair()`这个方法返回了一个Class实例。但是看其方法的名字`ClassPair`，我看了很多文章也是说其实是注册了`一对Class`，虽然没有经历过权威的证实，但是我觉得是非常有道理的。

况且前面也已经说了:

- (1) 对象的方法实现，存在于 `objc_class 实例 A` 的 method list
- (2) 类方法的实现，存在于 `objc_class 实例 B` 的 method list

这个确实是权威证明的，毋庸置疑。那所以，`objc_allocateClassPair()`这个方法确实是注册了`一对Class`:

- (1) 一个类 就是 类本身（我们经常调用的类）
- (2) 一个类 确实 元类（由苹果自己创建）

也就是肯定是有两个`objc_class实例`生成，但仅仅返回我们存放`对象的方法实现`的`objc_class实例`。但是我们可以通过如下runtime api获取得到存放`类方法的实现`的`objc_class实例`:

```objc
Class metaCls = object_getClass([MyClass class]);
```

这里先直接使用前面我不明白的函数`object_getClass()`，只是想先说说这个函数的作用。

到这里为止，我想对于一个NSObject类实际上在运行时分为两个版本的`objc_class`实例的理解有一定的明白了。


###通过例子测试一个NSObject类实际上在运行时分为两个版本的`objc_class`实例的结论

下面都是有这个NSObject实体类

```objc
@interface TestObjR : NSObject
@property (nonatomic, copy) NSString *name;
@end
@implementation TestObjR
@end
```

###当同时对一个`NSObject类的对象`作用时、``object_getClass()`与`-[NSObject class]、+[NSObject class]`的区别

```objc
@implementation RuntimeController

- (void)test1 {
    TestObjR *obj1 = [TestObjR new];
    TestObjR *obj2 = [TestObjR new];
    TestObjR *obj3 = obj1;
    TestObjR *obj4 = obj2;
    
    //object_getClass()
    NSLog(@"object_getClass >>> obj1: %p", object_getClass(obj1));
    NSLog(@"object_getClass >>> obj2: %p", object_getClass(obj2));
    NSLog(@"object_getClass >>> obj3: %p", object_getClass(obj3));
    NSLog(@"object_getClass >>> obj4: %p", object_getClass(obj4));
    
    //[object class]
    NSLog(@"[object class] >>> obj1: %p", [obj1 class]);
    NSLog(@"[object class] >>> obj2: %p", [obj2 class]);
    NSLog(@"[object class] >>> obj3: %p", [obj3 class]);
    NSLog(@"[object class] >>> obj4: %p", [obj4 class]);
}

@end
```

输出如下:

```
2016-10-26 14:15:37.402 Demos[2129:508480] object_getClass >>> obj1: 0x1003685e8
2016-10-26 14:15:37.402 Demos[2129:508480] object_getClass >>> obj2: 0x1003685e8
2016-10-26 14:15:37.403 Demos[2129:508480] object_getClass >>> obj3: 0x1003685e8
2016-10-26 14:15:37.403 Demos[2129:508480] object_getClass >>> obj4: 0x1003685e8
2016-10-26 14:15:37.404 Demos[2129:508480] [object class] >>> obj1: 0x1003685e8
2016-10-26 14:15:37.404 Demos[2129:508480] [object class] >>> obj2: 0x1003685e8
2016-10-26 14:15:37.404 Demos[2129:508480] [object class] >>> obj3: 0x1003685e8
2016-10-26 14:15:37.405 Demos[2129:508480] [object class] >>> obj4: 0x1003685e8
```

从输出可以得到两点:

- 一个NSObject类生成的n个不同对象，得到的是`同一个 objc_class`结构体的实例.

- 当对一个`NSObject类生成的对象`作用时，`object_getClass()`与`[object class]`返回的`objc_class`结构体实例都是用一个地址，即同一个实例

###当同时对一个`NSObject类`作用时、``object_getClass()`与`-[NSObject class]、+[NSObject class]`的区别


看下`object_getClass()`方法的声明:

```objc
/** 
 * Returns the class of an object.
 * 
 * @param obj The object you want to inspect.
 * 
 * @return The class object of which \e object is an instance, 
 *  or \c Nil if \e object is \c nil.
 */
OBJC_EXPORT Class object_getClass(id obj) 
```

这个方法接收的参数类型是`id`，即任何的`类的对象`都可以传入。但是如下这么写确实报错的:

```objc
//object_getClass()获取Class
NSLog(@"object_getClass >>> person1: %p", object_getClass(Person));//这里会报错
```

我有点小迷糊，`一个NSObject类`为啥不能当做`object_getClass()`的参数了？不是说NSObject类也是一个对象吗？那为啥不能够传进去当做参数了？


```objc
// 错误
object_getClass(Person);
```

第一种直接传入一个`NSObject类`作为参数就会编译器报错。

```objc
// 正确
object_getClass([Person class]);
```

第二种`获取类的objc_class结构体实例`之后再传进去就通过编译了。但是NSObject类不也是对象吗？为啥就不能当做参数传入了？

想了一段时间，明白之后还是因为runtime的基本概念没弄明白，主要是以下几点:

- (1) Runtime这一套类库，都是基于`c语言代码`实现的。所以 Objective-C代码最终也是编译为`c语言的代码`，再编译生成Mach-O格式的可执行文件

- (3) 我们所编写的NSObject类并非是最终iOS系统运行的类，压根就不能直接运行OC类，其实都是在编译阶段将一个NSObject类切割成如下几部分:

```objc
一个NSObject类:
	- objc_class A
		- ivar list
			- ivar 1
			- ivar 2
		- method_list
			- method 1
			- method 2
	- objc_class B
		- method_list
			- method 1
			- method 2
```

全部切割成 c struct:

```c
objc_class
objc_object
objc_method
objc_ivar
objc_category
objc_property
.....
```

那所以不能直接使用`NSObject`

```objc
// 错误
object_getClass(Person);
```

而必须要拿到runtime system认识的类对象 >>> `objc_class`实例

```objc
Class cls = [Person class];
object_getClass(cls);
```

所以最终runtime system只认识这些`objc_`开头的实例。

###证明NSObject类对应的在运行时阶段的`objc_class`结构体实例是单例


```objc
@implementation RuntimeController

- (void)test2 {
    TestObjR *obj1 = [TestObjR new];
    TestObjR *obj2 = [TestObjR new];
    TestObjR *obj3 = [TestObjR new];
    
    Class cls1 = [obj1 class];
    Class cls2 = [obj2 class];
    Class cls3 = [obj3 class];
    Class cls4 = [TestObjR class];
    
    NSLog(@"%p", cls1);
    NSLog(@"%p", cls2);
    NSLog(@"%p", cls3);
    NSLog(@"%p", cls4);
}
```

@end

输出结果

```
2016-10-26 14:38:50.620 Demos[2150:514216] 0x100420620
2016-10-26 14:38:50.623 Demos[2150:514216] 0x100420620
2016-10-26 14:38:50.623 Demos[2150:514216] 0x100420620
2016-10-26 14:38:50.623 Demos[2150:514216] 0x100420620
```

从输出的地址上看，都是指向同一个地址，所以NSObject类对应的`objc_class`结构体实例确实是单例，全局只存在一个内存空间来存放。

###demo

```objc
@implementation RuntimeController

- (void)test3 {
    
    NSLog(@"============== 1. 对类使用object_getClass() =================");
    
    for (int i = 0; i < 4; i++) {
        Class cls = object_getClass([TestObjR class]);
        NSLog(@"cls = %@, cls: %p", cls, cls);
    }
    
    NSLog(@"============== 2. 对类使用object_getClass() =================");
    
    for (int i = 0; i < 4; i++) {
        Class cls = object_getClass([[TestObjR new] class]);
        NSLog(@"cls = %@, cls: %p", cls, cls);
    }
    
    NSLog(@"============== 3. 对类使用 +[NSObject class] =================");
    
    for (int i = 0; i < 4; i++) {
        Class cls = [TestObjR class];
        NSLog(@"cls = %@, cls: %p", cls, cls);
    }
}

@end
```

输出结果

```
2016-10-26 15:48:41.374 Demos[2252:531051] ============== 1. 对类使用object_getClass() =================
2016-10-26 15:48:41.377 Demos[2252:531051] cls = TestObjR, cls: 0x1003e8688
2016-10-26 15:48:41.377 Demos[2252:531051] cls = TestObjR, cls: 0x1003e8688
2016-10-26 15:48:41.378 Demos[2252:531051] cls = TestObjR, cls: 0x1003e8688
2016-10-26 15:48:41.378 Demos[2252:531051] cls = TestObjR, cls: 0x1003e8688
2016-10-26 15:48:41.379 Demos[2252:531051] ============== 2. 对类使用object_getClass() =================
2016-10-26 15:48:41.380 Demos[2252:531051] cls = TestObjR, cls: 0x1003e8688
2016-10-26 15:48:41.380 Demos[2252:531051] cls = TestObjR, cls: 0x1003e86882016-10-26 15:48:41.381 Demos[2252:531051] cls = TestObjR, cls: 0x1003e8688
2016-10-26 15:48:41.384 Demos[2252:531051] ============== 3. 对类使用 +[NSObject class] =================
2016-10-26 15:48:41.385 Demos[2252:531051] cls = TestObjR, cls: 0x1003e86b0
2016-10-26 15:48:41.386 Demos[2252:531051] cls = TestObjR, cls: 0x1003e86b0
2016-10-26 15:48:41.386 Demos[2252:531051] cls = TestObjR, cls: 0x1003e86b0
2016-10-26 15:48:41.387 Demos[2252:531051] cls = TestObjR, cls: 0x1003e86b0

```

虽然Class都是`TestObjR`，但是得到的`objc_class`实例的地址是`不同`的。


##从opensource查看`object_getClass()`、`+[NSObject class]`、`-[NSObject class]`


###NSObject常用的方法实现

http://opensource.apple.com/source/objc4/objc4-237/runtime/Object.m

```objc
@implementation Object 

+ initialize
{
    return self; 
}

- init
{
    return self;
}

- self
{
    return self; 
}

- class
{
    return (id)isa; 
}

+ class 
{
    return self;
}

+ superclass 
{ 
    return ((struct objc_class *)self)->super_class; 
}

- superclass 
{ 
    return ((struct objc_class *)isa)->super_class; 
}

- (BOOL)isKindOf:aClass
{
    register Class cls;
    for (cls = isa; cls; cls = ((struct objc_class *)cls)->super_class) 
        if (cls == (Class)aClass)
            return YES;
    return NO;
}

- (BOOL)isMemberOf:aClass
{
    return isa == (Class)aClass;
}

+ (BOOL)instancesRespondTo:(SEL)aSelector 
{
    return class_respondsToMethod((Class)self, aSelector);
}

- (BOOL)respondsTo:(SEL)aSelector 
{
    return class_respondsToMethod(isa, aSelector);
}

- (IMP)methodFor:(SEL)aSelector 
{
    return class_lookupMethod(isa, aSelector);
}

+ (IMP)instanceMethodFor:(SEL)aSelector 
{
    return class_lookupMethod(self, aSelector);
}

@end
```

着重看

```objc
- class
{
    return (id)isa; 
}

+ class 
{
    return self;
}
```

- `+[NSObject class]`

返回的就是Person这个NSObject类对应的`objc_class`结构体实例。并且返回的是`self`.....

我觉得一个NSObject类，其实就是可以看做运行时环境中的一个`objc_class`结构体实。

所以简单的理解就是，NSObject类就是`objc_class的实例`，或者具备被一一对应的关系。

总之，NSObject类最终在程序运行时，就是由一个与之一一对应的`objc_class的实例`来承载表示。

- `-[Person对象 class]`

首先根据`对象->isa`指针找到所属类Person对应的`objc_class`结构体实例。

###`object_getClass()`实现


http://opensource.apple.com/source/objc4/objc4-680/runtime/objc-class.mm

```c
Class object_getClass(id obj)
{
    if (obj) return obj->getIsa();
    else return Nil;
}

Class object_setClass(id obj, Class cls)
{
    if (!obj) return nil;

    // Prevent a deadlock between the weak reference machinery
    // and the +initialize machinery by ensuring that no 
    // weakly-referenced object has an un-+initialized isa.
    // Unresolved future classes are not so protected.
    if (!cls->isFuture()  &&  !cls->isInitialized()) {
        _class_initialize(_class_getNonMetaClass(cls, nil));
    }

    return obj->changeIsa(cls);
}

BOOL object_isClass(id obj)
{
    if (!obj) return NO;
    return obj->isClass();
}
```

- 首先传入的是NSObject类所对应的`objc_class`结构体实例

- 获取到传入的`objc_class实例->isa`所指向的`另一个objc_class`结构体实例`

##所以当对于一个`NSObject类`时，有如下不等式永远都是成立的

```
object_getClass([NSObject类 class]) != [NSObject类 class] 
object_getClass([NSObject类 class]) != [NSObject对象 class]
```

因为

```
object_getClass([NSObject类 class]) >>> Meta Class
```

```
[NSObject类 class] == [NSObject对象 class] >>> Class本身
```

##那么我想可以回答本篇文章开头的疑问了: `object_getClass()与[object class]` 有什么区别？

- `object_getClass()`方法，获取类对象的isa指针指向的另外一个objc_class结构体实例

- 而对于`NSObject类`的isa指针，指向的是`Meta元类`，已经不是`NSObject类本身`了

- `[Person class]与[Person对象 class]`获取都是`Person类本身`的`objc_class`结构体实例

- 所以当两种方法同时去操作一个`类`时，得到的objc_class结构体实例是`不一样`的
	- 前者是 `Meta类` 的objc_class结构体实例
	- 后者是 `类本身` 的objc_class结构体实例
	- 所以地址是`不相等`的

##还有一个与`object_getClass()`类似的方法: `objc_getClass()`

```c
// 接收一个c字符串
Class objc_getClass(const char *name)
```

以一个小例子看其区别:

```objc
NSLog(@"%p", [Person class]);
NSLog(@"%p", object_getClass([Person class]));
NSLog(@"%p", objc_getClass("Person"));
```

结果输出

```
2016-06-22 16:12:29.007 demo[10437:1581222] 0x104e53c28
2016-06-22 16:12:29.008 demo[10437:1581222] 0x104e53c00
2016-06-22 16:12:29.008 demo[10437:1581222] 0x104e53c28
```

应该知道了吧..


##学习资源

```
http://www.jianshu.com/p/ae5c32708bc6
```