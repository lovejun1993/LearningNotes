---
layout: post
title: "Runtime Part2 SEL"
date: 2014-09-22 18:04:03 +0800
comments: true
categories: 
---

###什么是消息选择器？

- (1) 消息选择器是一种`文本字符串`，用于消息传递的过程

- (2) 消息选择器是一种分为多段的文本字符串，每个段以冒号结尾且后面可以跟方法执行参数

- (3) 运行时系统根据消息选择器，找到对应的`方法实现`完成方法调用

- (4) 消息选择器主要是标识区别`某个对象/某个类`中的某一个`对象方法/类方法`

- (5) 在Objective-C中，消息选择器直接与一个或多个类/对象中方法声明格式对应


加入有如下OC类，其对象方法声明如下

```objc
@interface Cat : NSObject
- (void)helloWorld;
- (void)helloWorldWithArgument:(id)arg;
@end
```

那么根据上面声明的两个对象方法，则对应的消息选择器也有两个:

```objc
helloWorld
helloWorldWithArgument:
```

注意选择器并不是下面这样的，下面这样其实是将选择器转换成了SEL数据类型之后的东西了。

```objc
SEL sel1 = @selector(helloWorld);
SEL sel2 = @selector(helloWorldWithArgument:);
```

然后在程序`编译`期间，创建各种c数据结构，来解析我们所编写的所有Objective-C代码:

- 类结构、`objc_class`
- 对象结构、`objc_object`
- 方法结构、`objc_method`
- 函数实现结构、`IMP`
- 实例变量结构、`objc_ivar`
- 属性结构、`objc_property`
- 分类结构、`objc_category`


在程序`运行`期间查询消息选择器对应的方法实现IMP:

- (1) 首先找到`对象/类`所对应的`objc_class实例/objc_object实例`
- (2) **然后将选`择器转`换成`SEL类型`数据**
- (3) 再使用得到的`SEL类型`数据，查询`objc_class实例/objc_object实例`中的`method_list`，找到对应的`objc_method实例`
- (4) 最后根据找到的`objc_method实例`的IMP指针变量，找到最终的函数实现存放的内存地址，完成方法实现调用

****

###什么是SEL？上面第(3)步中会将选择器转换SEL类型的数据.

- (1) SEL的一些基础

	- SEL是一种特殊的Objective-C的`数据类型`
	- 可以根据这个唯一标识，可以找到对应的`objc_method`结构体实例
	- SEL 是c结构体 `objc_selector` 类型的一个指针

	```
	typedef struct objc_selector   *SEL;
	```

	- `objc_selector` 结构体声明

	```
	typedef const struct objc_selector   
	{  
	    void *sel_id;  
	    const char *sel_types;  
	};
	```
	
- (2) 通常我们使用`SEL sel1 = @selector(helloWorld);`得到一个SEL

其实就是创建了一个 objc_selector 结构体实例，并完成了成员变量的数据初始化。

- (3) 为什么需要将选择器转换成`objc_selector 结构体实例`了？

正是因为我们编写的所有`Objective-C方法`在程序编译期间，会被编译器自动编译成一个个的`c函数`。

但是最终编译生成的`c函数`如何和我们编写的`Objective-C方法`对应起来了？正是通过`objc_method`的一个个实例来映射。

```objc
struct objc_method {

	//1. 一个 objc_selector结构体实例
    SEL method_name;
    
    //2. OC方法的type encodings
    char *method_types;
    
    //3. 最终编译生成的c函数实现在运行时内存中存放的地址
    IMP method_imp;
} 
```

只能通过一个`SEL实例`，来查询到一个`objc_method实例`，继而取出c函数实现存放的内存地址`IMP`。

而`objc_method`实例又归属于`objc_class`实例中。一个OC类都有一个与之对应的独一无二的`objc_class`实例，所以无需担心相同的方法实现导致覆盖或冲突的问题，因为他们属于不同的`objc_class`实例中:

```objc
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class super_class;
    const char *name;
    long version;
    long info;
    long instance_size;
    struct objc_ivar_list *ivars;
    struct objc_method_list **methodLists;//存放所有出现在该OC类中的OC方法
    struct objc_cache *cache;
    struct objc_protocol_list *protocols;
#endif

}
```

***

###Objective-C中所谓的消息发送，其实就是调用`objc_msgSend()`这个c函数

- OC中的数组添加元素

```
[array insertObject:foo atIndex:5];
```

- 编译成c语言版本的消息发送代码

```
objc_msgSend(array, @selector(insertObject:atIndex:), foo, 5);
```

- 总结出格式

```
objc_msgSend(消息接收者, 消息名SEL值, 参数1, 参数2, ... 参数n)
```

***

###方法签名（ method signature）: 定义了目标方法实现执行时，所需参数的数据类型、以及函数执行后的返回值的数据类型

当对象接收到消息之后，经过一系列查询找到最终的`c函数实现`然后完成方法调用，但是有几个问题:

- (1) 运行时系统，如何确定被调用的c函数 `需要什么样的参数类型` 了？
- (2) 运行时系统，如何确定被调用的c函数 `函数返回值的类型` 了？

只有通过首先获取得到OC方法的`签名`，从而让运行时系统根据`签名`推测出最终被调用的c函数的格式:

- (1) 需要什么样的参数类型
- (2) 函数返回值的类型


如果获取OC方法的`签名`失败，那么在最终执行c函数实现时就会出现程序崩溃，因为运行时系统根本不知道传什么类型的数据、也不知道返回值的类型。


***

###OC消息的种类

- (1) 对象消息，最终消息发送给一个`OC类的对象`来处理
- (2) 类消息，最终消息发送给一个`OC类`来处理

一般用的最多的还是(1)，因为对象消息可以重写消息转发的一系列函数。但如果是类消息，就不好重写消息转发函数实现了，因为类方法的实现我们直接获取不到，而是在所谓的Meta元类中。

***

###动态绑定

```objc
@interface Animal : NSObject

@property (nonatomic, copy) NSString *name;

- (instancetype)initWithName:(NSString *)name;
- (void)debugName;

@end


@interface Monkey : Animal

@end
```

```objc
@implementation Animal

- (instancetype)initWithName:(NSString *)name {
    if (self = [super init]) {
        _name = [name copy];
    }
    return self;
}

- (void)debugName {
    NSLog(@"name = %@", _name);
}

@end

@implementation Monkey

@end
```

测试代码

```objc
- (void)test12 {
    
    id ani1 = [[Animal alloc] initWithName:@"animal01"];
    [ani1 debugName];
    
    id ani2 = [[Monkey alloc] initWithName:@"animal02"];
    [ani2 debugName];
//断点log
}
```

运行结果

```
2015-07-31 17:24:36.030 Demos[21074:299508] name = animal01
2015-07-31 17:24:36.030 Demos[21074:299508] name = animal02
(lldb) fr v
(Animal *) ani1 = 0x00007fead971ec10
(Monkey *) ani2 = 0x00007fead971f310
(lldb) 
```


可以看到，即使使用id万能指针，最终仍然能正确的识别出对象的真实类型，从而调用对应的方法实现。

重点是，在程序运行时，会首先找到ani1和ani2真正的target，然后才会去调用`objc_msgSend()`将消息发送给找到的target。