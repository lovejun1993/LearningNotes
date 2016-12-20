---
layout: post
title: "Runtime Part1 认识"
date: 2014-09-21 02:31:21 +0800
comments: true
categories: 
---

###Runtime运行时，以前搞java的时候，就觉得是一个很神秘、很复杂的东西...没想到现在搞iOS，还是不可避免的要接触到这个东西，好吧具体学习一下吧.

****


###运行时系统组成部分: 编译器 和 运行时动态链接库

> 编译器、做的事情

- (1) 在程序`编译`期间负责完成一些`Objetive-C代码`编译生成`c代码`
- (2) 而这些被编译的c代码不仅仅只是普通的c代码，而是使用了`运行时系统库中api`的代码

> 运行时链接库、组成结构

- (1) 类元素: 一个Objective-C类的结构抽象
	- 类 `objc_class`
	- 方法 `objc_method`
	- 协议  `objc_protocol`
	- 分类 `objc_category`
	- 属性 `objc_property `
	- 实例变量 `objc_ivar`

- (2) 类的对象: 一个Objective-C类的某一个具体对象的结构抽象
	- `objc_object`

- (3) 消息传递
	- 动态类型
	- 动态类型绑定

- (4) 消息转发
	- 阶段一、快速转发: resolve method 
	- 阶段二、完成转发: 
		- 子阶段一、forward target
		- 子阶段二、forward invocation

- (5) 动态加载
	- 加载程序包
	- 框架包（.a、.framework、.dylib）
	- 资源文件（boundle、文件、图片）

****

###通常我们写的`[target doMethodWith:var1];`成为Objective-C中的方法调用，其实不是真正的方法调用，而是所谓的`发送消息`.

####Objetive-C是基于c语言封装出来的，底层还是c语言代码。开发者使用OC语法写代码，在编译时由Xcode调用编译器将OC的代码全部翻译成c语言的代码。

```
[target doMethodWith:var1];
```

被编译器翻译成一行消息发送的c代码

```c
objc_msgSend(target,@selector(doMethodWith:),var1);
```

假设有如下OC类

```objc
#import <Foundation/Foundation.h>

@interface Person : NSObject

- (void)doMethodWith:(NSString *)args;

@end
```

```objc
#import "Person.h"

@implementation Person

- (void)doMethodWith:(NSString *)args {
    NSLog(@"haha");
}

@end
```

- 首先，Person这个NSObjct类，使用了一个`objc_class`结构体的实例在内存中表示.（可以参看Class）

- 然后doMethodWith:这个方法使用了一个`objc_method`结构体实例在内存中表示.（可以参看Method）
	- SEL >>> doMethodWith: ，找到一个 objc_method实例的id值
	- IMP >>> 被编译生成的c函数的函数名 或 地址.
	- TypeEncodings >>> OC函数与具体c函数对应的格式转换

- 最后OC函数被编译生成的c函数可能如下

```
void doMethodWith(NSString *) {
	....
}
```


***

###看下调用`c函数`的过程，修改main.m

```c

#import <UIKit/UIKit.h>
#import "AppDelegate.h"

static void func() {
    printf("Hello World~~~\n");
}

int main(int argc, char * argv[]) {
    @autoreleasepool {

        //直接函数调用
        func();
        
//        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

c语言中，直接就是函数调用，直接找到函数名（函数地址）就可以完成方法调用，即编译期间就知道了要调用哪一个函数.

***

###Objective-C并不是像c语言直接使用函数名（函数地址），而是将`c函数的调用`封装成了一个Message消息，在OC中常见语法为`[Target SEL1:参数1 SEL2:参数2 SEL3:参数3 ...]` 这样类型的.

- 使用objc_msgSend(Target, SEL, 参数) 向iOS系统Runtime发送一个消息

- iOS系统Runtime接收到消息之后做的事情:
	- 首先区分是调用 对象方法 or 类方法
	- 调用 对象方法
		- 向当前NSObjct类对应的objc_class实例，查询SEL对应的c函数实现IMP
	- 调用 类方法
		- 首先找到当前NSObjct类对应的`objc_class实例->isa`指向的meta_class
			- （注意: meta class也是一个objc_class实例）
		- 向当前 meta_class实例，查询SEL对应的c函数实现IMP

- 找到了最终要执行的c函数之后，然后将所有的参数传给该c函数，让其执行
	- 系统默认的两个参数:
		- id self >>>> 消息接收者
		- SEL self >>>> OC中发送消息的方法SEL
	- 后面接着其他的用户自定义的参数

***

###那么编译器识别了一个`消息`之后，做了什么？

- Xcode编译器自动替换成消息的`c语言`的实现代码

```
objc_msgSend(消息接收者, @selector(run), 参数1, 参数2 ..,参数n)
```

- 看一下`objc_msgSend()`函数的参数定义:

```
id objc_msgSend(id self, SEL op, ...)


1. id self: 指向消息的接收者（eg. Person对象）
2. SEL op: 指向一个方法的 `具体实现` 的 `IMP指针`，所以此刻的SEL并不是真正的方法实现
3. ...: 调用函数传入的参数
```

- 所谓的消息，oc中有一个版本 >>> `NSInvocation`
	- 估计NSInvocation最终还是通过 `objc_msgSend()` 完成最终目标c函数调用

****

###在传递给IMP指针指向的最终c函数实现的参数中，除了方法本身的我们传入的参数之外，还有`两个隐藏参数`:

- 第一个位置参数: `self` >>> 其编码是 `@` >>> id类型
	- 消息接收者

- 第二个位置参数: `_cmd` >>> 其编码是 `:` >>> SEL类型
	- OC中执行的SEL

- 如上两个隐藏参数，是在`编译期间`由编译器自动插入到函数的实现代码中得

###综上所述runtime最核心的概念 -- `消息`

那么runtime能够做什么？

- 运行时，修改内存中的数据的结构
	- 动态的在内存中创建一个 `类`
	- 给类增加一个属性
	- 给类增加一个协议实现
	- 给类增加一个方法实现IMP
	- 替换、交换已经存在的方法实现IMP
	- 遍历一个 类 的所有 `成员变量/属性/方法`

- 关于应用
	- JSON 转成 实体类对象
	- 方法拦截，实现AOP面向切面编程
	- JSPatch替换已有的OC方法实现


***

###`objc_msgSend()`使用一，发送void返回值类型且void形参类型的消息

```objc
#import <Foundation/Foundation.h>

@interface MsgSendDemo : NSObject

- (void)cry;
- (NSString *)cryWithWord:(NSString *)word;

@end

@implementation MsgSendDemo

- (void)cry {
    NSLog(@"不带参数的OC消息发送");
}

- (NSString *)cryWithWord:(NSString *)word {
    NSLog(@"带参数以及返回值的OC消息发送，接收到word = %@", word);
    return @"I am return value";
}

@end
```

测试类

```objc
- (void)test13 {
    
    //1.
    Class cls = objc_getClass("MsgSendDemo");
    
    //2.
    id obj = [[cls alloc] init];
    
    //3.
    SEL sel1 = @selector(cry);
    ((void (*)(id, SEL)) (void *) objc_msgSend)(obj, sel1);
    
    //4.
    SEL sel2 = @selector(cryWithWord:);
    NSString *ret = ((NSString* (*)(id, SEL, NSString*)) (void *) objc_msgSend)(obj, sel2, @"我是参数");
    NSLog(@"ret = %@", ret);
}
```

运行结果

```
2015-07-31 18:06:02.711 Demos[23341:331724] 不带参数的OC消息发送
2015-07-31 18:06:02.711 Demos[23341:331724] 带参数以及返回值的OC消息发送，接收到word = 我是参数
2015-07-31 18:06:02.712 Demos[23341:331724] ret = I am return value
```

可以看到，照样完成了我们经常使用OC方括号完成的消息发送。


注意如上使用`objc_msgSend()`函数时，前面的一大坨函数指针的代码的作用:

- Xcode6之后，默认objc_msgSend()函数`不接收任何参数`

```
#if !OBJC_OLD_DISPATCH_PROTOTYPES

//objc_msgSend()函数声明为不接受参数类型
OBJC_EXPORT void objc_msgSend(void /* id self, SEL op, ... */ )
    __OSX_AVAILABLE_STARTING(__MAC_10_0, __IPHONE_2_0);
    
//objc_msgSendSuper()函数声明为不接受参数类型
OBJC_EXPORT void objc_msgSendSuper(void /* struct objc_super *super, SEL op, ... */ )
    __OSX_AVAILABLE_STARTING(__MAC_10_0, __IPHONE_2_0);
#else
```

可以看到只要 `OBJC_OLD_DISPATCH_PROTOTYPES`这个宏的值 == `NO`，就会走如上的宏编译成 `不接受参数` 的方法定义，而接受参数的方法定义如下

```
OBJC_EXPORT id objc_msgSend(id self, SEL op, ...)
    __OSX_AVAILABLE_STARTING(__MAC_10_0, __IPHONE_2_0);
```

```
OBJC_EXPORT id objc_msgSendSuper(struct objc_super *super, SEL op, ...)
    __OSX_AVAILABLE_STARTING(__MAC_10_0, __IPHONE_2_0);
#endif
```

```
//用于返回值是一些c结构体类型
OBJC_EXPORT void objc_msgSend_stret(id self, SEL op, ...)
    __OSX_AVAILABLE_STARTING(__MAC_10_0, __IPHONE_2_0)
    OBJC_ARM64_UNAVAILABLE;
```


- 解决办法一、Xcode中设置一下

百度搜...

- 解决办法二、使用 `函数类型` 强制转换 （建议使用这种方法）

假设有如下两个c函

```c
id test1(id self, SEL sel, ...) {
    
    return nil;
}

void test2(id self, SEL sel) {
    
}
```

分别对如上两个c函数的类型强制转换并执行

```c
id returnValue = ((id (*)(id, SEL, id))(void *) test1)(self, @selector(aUser), @"dawdwa");
```

```c
((void (*)(id, SEL))(void *) test2)(self, @selector(aUser));
```

于是可以得到`objc_msgSend()`函数类型转换格式并调用函数的语法格式如下:

> id和SEL两个参数是系统固定的参数，所以必须要写上

```objc
((函数返回值类型 (*)(id, SEL, 参数1, 参数2, 参数3 ...))(void *) objc_msgSend)(消息接收者, SEL, 参数1, 参数2, 参数3 ...);
```

如下举几个强转`objc_msgSend()`函数的例子:

```objc
#import "ViewController.h"
#import <objc/message.h>
#import <objc/runtime.h>

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    //1. 传递多余的参数，不会引发崩溃
    ((void (*)(id, SEL))(void *) objc_msgSend)(self, @selector(func1));
    ((void (*)(id, SEL, NSString*, NSString*))(void *) objc_msgSend)(self, @selector(func1), @"args1", @"asrg2");
    ((void (*)(id, SEL, NSString*, NSString*, NSString*))(void *) objc_msgSend)(self, @selector(func1), @"args1", @"asrg2", @"args3");
    
    //2.
    ((void (*)(id, SEL, NSString*, NSString*))(void *) objc_msgSend)(self, @selector(func1WithArgs1:args2:), @"args1", @"asrg2");
    
    //3.
    ((void (*)(id, SEL, NSString*, NSString*, NSString*))(void *) objc_msgSend)(self, @selector(func1WithArgs1:args2:args3:), @"args1", @"asrg2", @"args3");
    
    //4.
    NSString *ret = ((NSString* (*)(id, SEL))(void *) objc_msgSend)(self, @selector(func2));
    NSLog(@"%@", ret);
    
    //5.
    ret = ((NSString* (*)(id, SEL, NSString*, NSString*))(void *) objc_msgSend)(self, @selector(func2WithArgs1:args2:), @"参数1", @"参数2");
    NSLog(@"%@", ret);
    
}

- (void)func1 {
    NSLog(@"func1");
}

- (void)func1WithArgs1:(NSString *)args1 args2:(NSString *)args2 {
    NSLog(@"args1 = %@ , args2 = %@", args1, args2);
}

- (void)func1WithArgs1:(NSString *)args1 args2:(NSString *)args2 args3:(NSString *)args3 {
    NSLog(@"args1 = %@ , args2 = %@ , args3 = %@", args1, args2, args3);
}

- (NSString *)func2 {
    return @"func2";
}

- (NSString *)func2WithArgs1:(NSString *)args1 args2:(NSString *)args2 {
    return @"func2";
}

@end
```

输出如下

```
func1
func1
func1
args1 = args1 , args2 = asrg2
args1 = args1 , args2 = asrg2 , args3 = args3
func2
func2
```

***

###运行时系统提供了获取如下各项数据的函数api

```objc
- (void)test14 {
    
    //1 类定义
    Class cls1 = objc_getClass("MsgSendDemo");
    
    //2. 父类定义
    Class cls2 = class_getSuperclass([Animal class]);
    
    //3. 元类定义
    Class cls3 = objc_getMetaClass("MsgSendDemo");
    
    //4. 类名
    const char *className = class_getName(cls1);
    
    //5. 类对象所有实例变量
    unsigned int outCount = 0;
    Ivar *ivars = class_copyIvarList(cls2, &outCount);
//    free(ivars);
    
    //6. 所有方法
    outCount = 0;
    Method *methods = class_copyMethodList(cls1, &outCount);
//    free(methods);
    
    //7. 实现的所有协议
    outCount = 0;
    Protocol * __unsafe_unretained * protocols = class_copyProtocolList(cls1, &outCount);
//    free(protocols);
    
    //8. 类对象所有的属性
    outCount = 0;
    objc_property_t *propertys = class_copyPropertyList(cls2, &outCount);
//    free(propertys);

//断点 
}
```


