---
layout: post
title: "Callbacks回调方式"
date: 2015-05-16 23:09:05 +0800
comments: true
categories: 
---

###前阵子去面试然后问我: `模块之间的回调有哪些方式？`我当时还真没回答好，刚好晚上看代码的时候突然灵机一动想到了，还是记录下吧免得下次又忘记了..

***

###小结下可以完成回调的所有方法:

- Delegate
- Block 
- c 函数指针
- Target-Action模式（eg、UIButton添加回调处理）
- NSNotificationCenter 
- KVO属性值改变监测 

当时我的脑子里只有Delegate/Block这两种方法...至于其他的，有的是紧张忘记了，有的则是很陌生...感叹自己多菜逼啊！好好学习学习把...

关于Delegate/Block我就不说了，会iOS的都知道，只是注意几个问题:

- 对于Block就有一个`传递一个新的拷贝对象 or 传递对象的指针`的概念.

- 对于NSNotificationCenter，不会strong强引用通知观察者.

***


###从最原始的c函数指针完成回调记录起吧

- ViewController.m

```objc
#import "ViewController.h"

#import "Callback1.h"

static void callback1(NSString *arg1, NSString *arg2) {
    // 操作回传入方法的参数值
    NSLog(@"arg1 = %@, arg2 = %@", arg1, arg2);
}

static NSString* callback2(NSString *arg1, NSString *arg2) {
    //1. 操作回传入方法的参数值
    //2. return返回一个返回值
    return [NSString stringWithFormat:@"arg1 = %@ , arg2 = %@", arg1, arg2];
}

static NSInteger callback3(id<CallbackInterface> ret) {
    
    NSLog(@"name = %@", [ret name]);
    NSLog(@"address = %@", [ret address]);
    NSLog(@"postCode = %@", [ret postCode]);
    
    return 201;
}

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

    // 无返回值 函数指针回调
    GET_DATA1(@"www.baidu.com", callback1);
    
    // 有返回值 函数指针回调1
    GET_DATA2(@"www.baidu.com", callback2);
    
    // 有返回值 函数指针回调2
    GET_DATA3(@"www.baidu.com", callback3);
}

@end
```

- Callback.h与Callback.m

Callback.h

```objc
#import <Foundation/Foundation.h>

/**
 *  回调Context的抽象接口
 */
@protocol CallbackInterface <NSObject>

- (NSString *)name;
- (NSString *)address;
- (NSString *)postCode;

@end

// 不需要返回值的 回调方法
void GET_DATA1(NSString *url, void (*callback)(NSString *arg1, NSString *arg2));

// 需要返回值的 回调方法
void GET_DATA2(NSString *url, NSString* (*callback)(NSString *arg1, NSString *arg2));

// 需要返回值的 + 使用Context类打包回传值 回调方法
void GET_DATA3(NSString *url, NSInteger (*callback)(id<CallbackInterface> ret));
```

Callback.m

```objc
#import "Callback1.h"

/**
 *  回调Context的抽象接口实现类
 */
@interface CallbackContext : NSObject <CallbackInterface>

@property (nonatomic, copy) NSString *aName;
@property (nonatomic, copy) NSString *aAddress;
@property (nonatomic, copy) NSString *aPostCode;

@end

@implementation CallbackContext

- (NSString *)name {
    return _aName;
}

- (NSString *)address {
    return _aAddress;
}

- (NSString *)postCode {
    return _aPostCode;
}

@end

void GET_DATA1(NSString *url, void (*callback)(NSString *arg1, NSString *arg2)) {
    NSLog(@"接收到的参数URL = %@", url);
    
    // 执行接收到的函数指针完成回调
    if (callback) {
        callback(@"回传参数1", @"回传参数2");
    }
}

void GET_DATA2(NSString *url, NSString* (*callback)(NSString *arg1, NSString *arg2)) {
    NSLog(@"接收到的参数URL = %@", url);
    
    // 执行接收到的函数指针完成回调
    if (callback) {
        NSString *ret = callback(@"回传参数1", @"回传参数2");
        NSLog(@"接收到调用者返回值 = %@", ret);
    }

}

void GET_DATA3(NSString *url, NSInteger (*callback)(id<CallbackInterface> ret)) {
    NSLog(@"接收到的参数URL = %@", url);
    
    // 组装回传参数Context对象
    CallbackContext *ctx = [CallbackContext new];
    ctx.aName = @"XZH";
    ctx.aAddress = @"HuNan";
    ctx.aPostCode = @"415700";
    
    // 执行接收到的函数指针完成回调
    if (callback) {
        NSInteger code = callback(ctx);
        NSLog(@"接收到的code = %ld", code);
    }
}
```

多余就不解释了，一切尽在代码中...

****

###Target-Action（目标-动作）完成回调

- 不得不提经常使用的UIButton:UIControl，由UIControl提供了Target-Action完成回调的api.

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    UIButton *button = ....;
    [button addTarget:self action:@selector(test) forControlEvents:UIControlEventTouchUpInside];
}

- (void)test {
    //按钮点击回调
}
```

前面的一篇关于按钮防止连续点击多次的文章已经记录，最后是由UIControl的`- (void)sendAction:(SEL)action to:(nullable id)target forEvent:(nullable UIEvent *)event;`方法完成方法调用的消息发送:

- 而发送的消息包含:

	- 1、Target >>> 消息处理者
	- 2、SEL >>> 具体处理的函数实现对应的标识符
	- 3、传入方法的参数

所以，Target-Action模式，最终是通过发送消息来完成回调处理，但是消息最终还是使用c的函数指针来完成的，所以任何形式的回调最后归根结底都是通过c函数指针完成。

####下面写一个使用Target-Action完成回调的简单例子

- ViewController.m

```objc
#import "ViewController.h"

#import "Callback2.h"

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
//    [self callback_demo_1];
    [self callback_demo_2];
}

- (void)callback_demo_2 {
    Callback2 *callback = [Callback2 new];
    
    //回调格式1
    [callback addTarget:self action:@selector(callback2_1)];
    
    //回调格式2
//    [callback addTarget:self action:@selector(callback2_1WithRet1:)];

    //回调格式3
//    [callback addTarget:self action:@selector(callback2_1WithRet1:Ret2:)];
    
    [callback doWork];
}

- (void)callback2_1 {
    NSLog(@"callback2_1 执行回调");
}

- (NSInteger)callback2_1WithRet1:(NSString *)ret1 {
    NSLog(@"callback2_1WithRet1: 执行回调");
    
    return 1;
}

- (NSInteger)callback2_1WithRet1:(NSString *)ret1 Ret2:(NSInteger)ret2 {
    NSLog(@"callback2_1WithRet1:Ret2: 执行回调");
    
    return 1;
}

@end
```

- Callback2

Callback2.h

```objc
#import <Foundation/Foundation.h>

@interface Callback2 : NSObject

- (void)addTarget:(id)target action:(SEL)sel;

- (void)doWork;

@end
```

Callback2.m

```objc
#import "Callback2.h"

#import <objc/message.h>
#import <objc/runtime.h>

@interface Callback2 ()

@property (nonatomic, weak) id target;
@property (nonatomic, assign) SEL sel;

@end

@implementation Callback2

- (void)addTarget:(id)target action:(SEL)sel {
    _target = target;
    _sel = sel;
}

- (void)doWork {
    NSLog(@"Callback2 执行完毕");
    
    //TODO: 需要通过解析Method TypeEncodings来分情况，针对函数的参数定义调用objc_msgSend()
    Method m = class_getInstanceMethod([_target class], _sel);
    const char *typeEncodins = method_getTypeEncoding(m);

/////////////////////////////如下消息方法需要通过解析Method TypeEncodings来分情况/////////////////////////////
    
    //eg1、不需要回传参数值、不需要返回值
    ((void (*)(id, SEL))(void *) objc_msgSend)(_target, _sel);
    
    //eg2、回传一个NSString参数值，需要返回值
    NSInteger ret1 = ((NSInteger (*)(id, SEL, NSString*))(void *) objc_msgSend)(_target, _sel, @"haha");
    
    //eg3、回传两个值，需要返回值
    NSInteger ret2 = ((NSInteger (*)(id, SEL, NSString*, BOOL))(void *) objc_msgSend)(_target, _sel, @"haha", YES);
}

@end
```

对于objc_method实例的type encoding解析来发送不同参数格式的SEL消息，有时间再看看怎么弄...

补充下上面的问题，具体来说是获取objc_method实例的argument type字符串，然后根据其编码字符串来确定每一个参数的类型.

```c
static id callSelector(NSString *className, NSString *selectorName, JSValue *arguments, JSValue *instance, BOOL isSuper);
```

可以根据JSPatch中的如上方法的套路去模仿，主要即使根据type encodings字符来找到对应的数据类型.

| 最终编码 | 开发时使用的对应数据类型 | 
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
| @ | 自定义类型的对象，NSObject * |
| # | class object (Class) |
| : | A method selector (SEL) |
| [array type] | array数组 |
| {name=type...} |  structure 结构体 |
| (name=type...) | union 共用体 |
| bnum | A bit field of num bits |
| ^type | A `pointer` to type |
| ? | An unknown type (among other things, this code is used for `function pointers 函数指针`) |
| ^f | float * |
| ^v | void * |
| ^@ | NSError ** |
| [5i] | int[] |
| [3f] | float[] |


***

###使用KVO完成回调

> 核心就是 添加KVO属性观测


比如前面一篇文章记录NSProgress回传进度值，就是通过KVO属性值改变之后，然后在`- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSString *,id> *)change context:(void *)context{ ... }`实现中不断的调用传入的block完成回调.

例子就不写了，可以参看前面的关于NSProgress的文章.


