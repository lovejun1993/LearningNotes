---
layout: post
title: "Block模拟链式语法"
date: 2015-12-04 00:44:38 +0800
comments: true
categories: 
---

###有点类似Java中的链式语法

```
New Person().wakeup().eatRice().sleeping();
```

***

###首先定义一个Person类

```
#import <Foundation/Foundation.h>

@interface Person : NSObject

- (void)wakeup;

- (void)eatRice;

- (void)sleeping;

@end
```

```
@implementation Person

- (void)wakeup {
    
    NSLog(@"我起床拉...\n");
}

- (void)eatRice {
    
    NSLog(@"我吃饭拉...\n")
}

- (void)sleeping {

    NSLog(@"我睡觉拉...\n");
}

@end
```

***

###测试代码

```
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    Person *person = [[Person alloc] init];
    
    //1. 起床
    [person wakeup];
    
    //2. 吃饭
    [person eatRice];
    
    //3. 睡觉
    [person sleeping];
    
}

@end
```

如上这样做的话，不能达到之前给出的那种链式点语法的操作.

***

###对Person类的三个方法返回值做修改

```
#import <Foundation/Foundation.h>

@interface Person : NSObject

- (Person *)wakeup;

- (Person *)eatRice;

- (Person *)sleeping;

@end
```

```
@implementation Person

- (Person *)wakeup {
    
    NSLog(@"我起床拉...\n");
    
    return self;
}

- (Person *)eatRice {
    
    NSLog(@"我吃饭拉...\n")
    
    return self;
}

- (Person *)sleeping {

    NSLog(@"我睡觉拉...\n");
    
    return self;
}

@end
```

###测试代码


```
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    Person *person = [[Person alloc] init];
    
    // 起床 -> 吃饭 -> 睡觉
    [[[person wakeup] eatRice] sleeping];
}

@end
```

OK，和链式点语法有点接近了，但是造成很多的`方括号消息代码`.

***

###继续修改Person三个方法的返回值类型、以及方法实现

```
#import <Foundation/Foundation.h>

@interface Person : NSObject

//1. 所有函数的返回值都是一个Block

//2. Block的结构定义:
//2.1 Block的返回值类型是 Person对象
//2.2 Block中可以定义回传值参数，如下暂时未使用带参数的

- (Person *(^)())wakeup;

- (Person *(^)())eatRice;

- (Person *(^)())sleeping;

@end
```

```
@implementation Person

- (Person *(^)())wakeup {
    
    return ^() {
        
        //1. 处理代码
        NSLog(@"我起床拉...");
        
        //2. 返回当前对象
        return self;
    };
}

- (Person *(^)())eatRice {
    
    return ^() {
        
        //1. 处理代码
        NSLog(@"我吃饭拉...");
        
        //2. 返回当前对象
        return self;
    };
}

- (Person *(^)())sleeping {

    return ^() {
        
        //1. 处理代码
        NSLog(@"我睡觉拉...");
        
        //2. 返回当前对象
        return self;
    };
}

@end
```

###测试代码

```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    Person *person = [[Person alloc] init];
    
    // 起床 -> 吃饭 -> 睡觉
	//[[[person wakeup] eatRice] sleeping];
    
    //第一种
    person.wakeup().eatRice().sleeping();
    
    
    //第二种
    person.wakeup().eatRice().sleeping().wakeup().eatRice().sleeping();
}
```

###成员私有变量的getter与setter方法（属性方法）

- 如果成员私有变量为

```
类型 _variableName;
```

- 那么getter函数为如下

```
- (类型)variableName {
	
	return _variableName;
}
```

- 那么setter函数为如下

```
- (void)setVariableName:(类型)aVariableName {
	_variableName = aVariableName;
}
```

- 如上看出，其实getter函数名字就是`成员私有变量`的名字中`去掉前面的下划线`的后面一部分.

- 那么，任何的`对象方法funcName`，都被当成成员私有变量`_funcName`的getter方法

```
- (void)funcName {

}

或

- (id)funcName {
	return nil;
}
```

- 因此，所有的`对象方法funcName`，都可以通过`对象.funcName`，被当做属性getter方法调用。

***

###解析如上`person.wakeup()`执行过程

- 第一步、执行`person.wakeup`，类似调用使用`@property(. , .) id wakeup;`生成的`getter方法`
	- 也就是说任何方法都可以通过`对象.方法SEL`来调用
	- 因为当做成一个员变量的`getter`方法来调用
		- 成员变量其实没有，只是Xcode编译器认为是一个变量为`方法名字`的getter方法

- 第二步、wakeup方法被当成`getter方法`执行，返回一个`Block`
	- Block是带有返回值的
	- Block返回值就是当前对象自己

- 第三步、`person.wakeup()`执行接收到返回的Block，分解为如下两步
	- Block block = person.wakeup;
	- block();

- 注意: 
	- 每一个`方法的返回值`都是一个Block
	- Block是带有返回值的
	- 返回值就是当前对象的类型，返回当前操作的对象自己


***

###测试使用`对象.方法名`来调用一个方法（发送消息）

- Person类中有一个对象方法，方法名字是`test`

```
#import <Foundation/Foundation.h>

@interface Person : NSObject

- (void)test;

@end
```

```
@implementation Person

- (void)test {
    NSLog(@"test");
}

@end
```

- 测试代码

```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    Person *person = [[Person alloc] init];
    
    //1. 确实可以完成方法调用
    //2. 但是Xcode会给出一个黄色的警告: Property access result unused - getters should not be used for side effects
    person.test;
    
}
```

- 使用`person.test`虽然可以完成调用Person对象的test方法，但是有一个编译警告

```
Property access result unused - getters should not be used for side effects
```

大概意思是:

```
属性没有被使用
```

通常情况下，假设有一个name属性

```
@property (nonatomic ,copy) NSString *name;
```

那么经常如下使用这个属性时

```
self.name = @"hahaha";
```

而一旦向下面这么写，也会出现上面的那个`属性未使用的警告`

```
self.name;
```

- 就是说调用的成员变量`_name`的getter方法`name`获取到了值
- 但是又没有做其他的操作，那么Xcode认为这个getter方法调用是没有价值的
- 因此编译器给出一个黄色警告，提醒开发者使用获取到的变量
- 所以，消除这个警告的方法 -- `使用这个通过getter方法得到的值`

***

###带参数的链式语法函数

```

#import <Foundation/Foundation.h>

@interface Person : NSObject


// Block的结构定义:
// 1. Block的返回值类型是 Person对象
// 2. Block中可以定义回传值参数，如下暂时未使用带参数的

- (Person *(^)(NSString *name))dead;

@end
```

```
@implementation Person

- (Person *(^)(NSString *))dead {
    
    return ^(NSString *name) {
        
        //1. 操作外界执行Block时，传入的参数值
        NSLog(@"name = %@\n", name);
        
        //2. 返回当前对象
        return self;
    };
}

@end
```

测试代码如下

```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    //1.
    Person *person = [[Person alloc] init];
    
    //2. 
    person.wakeup().eatRice().sleeping().dead(@"nimade");
    
}
```