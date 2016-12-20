---
layout: post
title: "Runtime Part11 NSInvocation"
date: 2014-09-25 00:39:25 +0800
comments: true
categories: 
---

###NSInvocation学习记录

首先看一个简单的示例代码

```
//1. 创建要调用方法的SEL标识
SEL myMethod = @selector(myLog:parm:parm:);

//2. 设置消息执行的SEL
[invocatin setSelector:myMethod2];

//3. 从具有该SEL方法实现的Target获取到方法的签名
NSMethodSignature * sig  = [某个类 instanceMethodSignatureForSelector:myMethod];

//4. 使用方法签名，创建消息对象，确定消息调用的函数由哪些参数、返回值..
NSInvocation * invocatin = [NSInvocation invocationWithMethodSignature:sig];

//5. 给消息设置方法执行时需要的参数值
//注意：设置参数的Index 需要从2开始，因为前两个被selector和target占用
int a=1;
int b=2;
int c=3;
[invocatin setArgument:&a atIndex:2];//必须从2开始
[invocatin setArgument:&b atIndex:3];
[invocatin setArgument:&c atIndex:4];

//6. 设置消息的接收者
[invocatin setTarget:self];

//7. 向系统发送这个消息
[invocatin invoke];
```

如上有个方法的签名`NSMethodSignature`干什么的？
- 主要是描述一个方法的具体信息
	- 方法的参数
	- 方法的返回值类型
	- 等等其他有关方法相关的信息

****

###通过如上的代码大致可以得出`NSInvocation`做什么的？
- NSInvocation也是一种发送消息的方式
	- 在消息转发机制中，最后一个阶段，就是使用NSInvocation

- 其他我们常用的消息发送方式
	- 使用`[类/对象 SEL] or 对象.sel`发送消息
	- 使用`[类/对象 performSelector:OnThread:withArgs: ...]`发送消息

- 最原始的发送消息方式
	- 使用`objc_msgSend(Target, SEL, args)`发送消息

***


###使用`performSelector:withObject:`与`NSInvocation`发送消息的区别？
	
- 使用`performSelector:withObject:`对方法参数个数上有限制

- 而使用`NSInvocation`不限制方法参数个数，还可以做一下事情
	- 1) 对原始方法参数，进行增、删、改、
	- 2) 获取方法执行的返回值、以及修改

- 所以说，图简单就用`performSelector:withObject:`，要完成复杂的方法调用就需要使用`NSInvocation`

****

###如下是调用多参数的简单模板

```
- (void)viewDidLoad {
    [super viewDidLoad];
   
    SEL myMethod = @selector(myLog:parm:parm:);
   
    NSMethodSignature * sig  = [[self class] instanceMethodSignatureForSelector:myMethod];
   
    NSInvocation * invocatin = [NSInvocation invocationWithMethodSignature:sig];
   
    [invocatin setTarget:self];
   
    [invocatin setSelector:myMethod2];
   
    int a=1;
    int b=2;
    int c=3;
    [invocatin setArgument:&a atIndex:2];
    [invocatin setArgument:&b atIndex:3];
    [invocatin setArgument:&c atIndex:4];
    
    [invocatin invoke];
}

//该函数有三个参数
-(void)myLog:(int)a parm:(int)b parm:(int)c{
    NSLog(@"MyLog%d:%d:%d",a,b,c);
}
```

###使用NSInvocation来扩展`-[NSObject performSelector:withObject:withObject:]`存在的限制

- 可以同时设置 `多个参数`
- 同时可以获取 `返回值`

```objc
@implementation NSObject (NSInvocation)

- (id _Nonnull)performSelector:(SEL _Nonnull)aSelector withObjects:(id _Nullable)object, ... {
    
    //1. 获取SEL对应的方法签名
    NSMethodSignature *signature = [self methodSignatureForSelector:aSelector];
    
    //2. 必须找到方法签名，才能使用NSInvocation手动发送消息
    if (signature) {
        
        //2.1 使用方法签名创建一个对应的消息对象
        NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:signature];
        
        //2.2 设置消息接收者
        [invocation setTarget:self];
        
        //2.3 设置消息对应的触发SEL
        [invocation setSelector:aSelector];
        
/////////////////////////////////////////////////////////////////////////////
/////////2.4 多个参数一起打包处理
/////////////////////////////////////////////////////////////////////////////
        
        //声明一个新的可变参数列表
        va_list args;
        
        //指向传入的 `可变参数列表 object` 的第一个参数的地址
        va_start(args, object);
        
        //可变参数起始下标为2（除开系统的两个参数）
        [invocation setArgument:&object atIndex:2];
        
        //保存后续的每一个方法参数值
        id arg = nil;
        
        //后续方法参数值下标为3开始
        int index = 3;
        
        //遍历取出args参数列表中的每一个参数
        while ((arg = va_arg(args, id))) {
            
            //从下标3开始依次往下设置
            [invocation setArgument:&arg atIndex:index];
            
            index++;
        }
        
        //清空va_list可变参数列表
        va_end(args);
        
/////////////////////////////////////////////////////////////////////////////
        
        //2.5 触发消息
        [invocation invoke];
        
        //2.6 目标方法执行后的返回值获取
        if (signature.methodReturnLength) {
            
            //从NSInvocation获取返回值
            id anObject;
            [invocation getReturnValue:&anObject];
            
            return anObject;
            
        } else {
            
            return nil;
        }
        
    } else {
        
        return nil;
    }
}

@end
```

- NSInvocation完成`多参数设置`、`返回值获取`
- `可变参数` 设置为 NSInvocation参数的下标值 = `2`
- 具体的`每一个方法参数` 设置为 NSInvocation参数的下标值 = `3++`

****

###如下是从JSPatch中的callSelector()函数中，抽出来的一个NSInvocation使用模板

```objc
//1. 获取要执行方法的签名
NSMethodSignature  *signature = [类 instanceMethodSignatureForSelector:方法SEL];

//2. 未找到方法实现
if (!signature) {
	//...处理...
	return;
}

//2. 创建NSInvocation
NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:signature];

//3. 设置消息接收者
invocation.target = ....;

//4. 消息中执行方法的SEL
invocation.selector = 方法sel;

//5. 方法执行参数设置，需要考虑: 外界传递参数个数 != 方法的参数个数
//方法参数的个数，除去 self与_cmd两个系统参数
NSUInteger argsCount = signature.numberOfArguments - 2;

//外界传递的参数个数
NSUInteger arrCount = objects.count;

//取前面两个参数个数的最小值
NSUInteger count = MIN(argsCount, arrCount);

//对每个参数进行NSNull处理，并设置给NSInvocation
for (int i = 0; i < count; i++) {
    id obj = objects[i];
    if ([obj isKindOfClass:[NSNull class]]) {
        obj = nil;
    }
    
	[invocation setArgument:&obj atIndex:i + 2];//除开self、_cmd
}

//6. 返回值的处理
const char *returnType = [methodSignature methodReturnType];
id returnValue;
    
if (strncmp(returnType, "v", 1) != 0) {
    //方法返回值不是void
    
    if (strncmp(returnType, "@", 1) == 0) {
        //返回值类型是 Foundation类型（NSObject*）
        
        //从NSInvocation获取方法执行后的返回值
        void *result;
        [invocation getReturnValue:&result];
        
        //返回值方式一、将Core Foundation的对象转换为Objective-C对象，同时将内存管理权交给ARC
        returnValue = (__bridge_transfer id)result;
        
        //返回值方式二、只是类型转换成Objective-C对象
        returnValue = (__bridge id)result;
        
        return returnValue;
        
	} else {
		//基本数据类型
	    //SEL
	    //struct
	    //char* 与 ^其他类型
	    //Class
	    
	    分别按照如上类型，调用对应的方法，将从invocation获取的返回值，转换成对应的类型的值
	    returnValue = .....;
	    
		return returnValue;
	}
}

//返回值类型void，返回nil
return nil;
```

***

###模拟[cat cryWithWord:@"传入的值"];

Cat类

```objc
#import <Foundation/Foundation.h>

@interface Cat : NSObject

- (NSString *)cryWithWord:(NSString *)word;

@end
```

```objc
#import "Cat.h"

@implementation Cat

- (NSString *)cryWithWord:(NSString *)word {
    NSLog(@"%s word =%@", __FUNCTION__, word);
    return [NSString stringWithFormat:@"modify by cryWithWord: %@", word];
}

@end
```

ViewController测试类

```objc
#import <objc/runtime.h>
#import <objc/message.h>

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
//    [self testSendMsg1];
//    [self testSendMsg2];
    [self testInvocation];
//    [self testInvocation2];
}

- (void)testInvocation {
    
    /**
     *  方法实现的签名 >>> 告诉系统实现函数的定义
     *  - 返回值类型 >>> id >>> @
     *  - 第一个形参是消息接收者 >>> self >>> id >>> @
     *  - 第二个形参是SEL >>> _cmd >>> :
     *  - 后面的是形参其他用法参数
     */
    NSMethodSignature *mSign = [NSMethodSignature signatureWithObjCTypes:"@@:@"];
    
    //2.
    NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:mSign];
    
    //3.
    Class cls = objc_getClass("Cat");
    id cat = [[cls alloc] init];
    [invocation setTarget:cat];
    
    //4.
    [invocation setSelector:@selector(cryWithWord:)];
    
    //5.
    NSString *arg = @"传入的值";
    [invocation setArgument:&arg atIndex:2];
    
    //6.
    [invocation invoke];
    
    //7.
    void *result;
    [invocation getReturnValue:&result];
    
    //8.
    NSString *ret = (__bridge NSString *)(result);
    NSLog(@"%s 接收到的返回值 >>> %@", __FUNCTION__, ret);
    
}

@end
```

运行后输出如下

```
-[Cat cryWithWord:] word =传入的值
-[ViewController testInvocation] 接收到的返回值 >>> modify by cryWithWord: 传入的值
```

****

###

Cat类

```objc
#import <Foundation/Foundation.h>

@interface Cat : NSObject

- (NSString *)cryWithWordA:(NSString *)wordA wordB:(NSString *)wordB;

@end
```

```objc
#import "Cat.h"

@implementation Cat

- (NSString *)cryWithWordA:(NSString *)wordA wordB:(NSString *)wordB {
    NSLog(@"%s 接收到的形参wordA = %@, wordB = %@", __FUNCTION__, wordA, wordB);
    return [NSString stringWithFormat:@"The word = %@_%@", wordA, wordB];
}

@end
```

ViewController测试类

```objc
#import <objc/runtime.h>
#import <objc/message.h>

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
//    [self testSendMsg1];
//    [self testSendMsg2];
//    [self testInvocation];
    [self testInvocation2];
}

- (void)testInvocation2 {
    
    NSMethodSignature *mSign = [NSMethodSignature signatureWithObjCTypes:"@@:@@"];
    
    //2.
    NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:mSign];
    
    //3.
    Class cls = objc_getClass("Cat");
    id cat = [[cls alloc] init];
    [invocation setTarget:cat];
    
    //4.
    [invocation setSelector:@selector(cryWithWordA:wordB:)];
    
    //5.
    NSString *arg1 = @"Hello";
    NSString *arg2 = @"World";
    [invocation setArgument:&arg1 atIndex:2];
    [invocation setArgument:&arg2 atIndex:3];
    
    //6.
    [invocation invoke];
    
    //7.
    void *result;
    [invocation getReturnValue:&result];
    
    //8.
    NSString *ret = (__bridge NSString *)(result);
    NSLog(@"%s 接收到的返回值 >>> %@", __FUNCTION__, ret);
}
```

输出如下

```
-[Cat cryWithWordA:wordB:] 接收到的形参wordA = Hello, wordB = World
-[ViewController testInvocation2] 接收到的返回值 >>> The word = Hello_World
```