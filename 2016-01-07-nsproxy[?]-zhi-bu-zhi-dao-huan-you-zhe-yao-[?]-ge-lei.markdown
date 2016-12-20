---
layout: post
title: "NSProxy一直不知道还有这么一个类"
date: 2016-01-07 21:21:34 +0800
comments: true
categories: 
---

###顾名思义，肯定是关于`代理`的一些鬼东西.

***

###代理模式

> 模拟房屋租借: 租房子的人(A)、房屋中介(C)、房东(B)

- 在远古时期是这样子的: `A-->B`
	- A千辛万苦寻遍了每一条街道
	- A终于找到了B，这个时候可以直接签合同租房子了
	- 如果B不在家或已经租出去了，总之又白跑了

- 在现代时期随着C的崛起，现在是这样子的: `A-->C-->B`
	- A可以花部分钱找到C
	- C手里有大把的房源B，正愁找不到挨宰的
	- C只需要在他房屋系统即可看到哪些适合A条件的B
	- OK，A只要有足够的钱，一上午就可以通过C找到B，租好房子了


- 看到区别:

	- A-->B，就是自己亲自去寻找数据，自己提供业务
	- A-->C-->B，让中介（代理）帮我寻找数据，帮我提供业务，自己只需要得到满意的结果即可

- 那么也就是在A与B中间，`多出了一层C`，而这层C可以做什么了？
	- 查询到 很多B数据 返回给 A
	- 提供让A找到B `之前` 的一些事情（如: 查询房屋系统..）
	- 提供让A找到B `之后` 的一些事情（如: 收取租客房屋租金的一半..）
	- 总之C 是可以拦截掉 A对B 的直接强引用调用，并且可以动态的添加一点什么

***

###代理 与 协议 的区别:

- 相同点:

	- 都是同样使用 @protocol 声明的一个抽象方法定义

- 不同点:

	- 代理，主要要来 `获取数据、回传数据、回调执行代码，减少一些不必要的代码耦合度`
	- 协议，通常是用来`描述 具有一组相关行为`的物体抽象定义

***

###NSProxy类实现了NSObject协议，NSObject类也实现了NSObject协议。

```objc
@interface NSProxy <NSObject> {
    Class	isa;
}

//...

@end
```

```objc
@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}

//....

@end
```

***

###举一个简单的例子，通过向外暴露Proxy类，来达到屏蔽内部使用的具体类.

***

###首先，大多是抽一个抽象协议接口，用来抽象某一类事物.

```objc
Animal.h 代表动物这一类

#import <Foundation/Foundation.h>

/**
 *  一个动物的所有行为的抽象
 */
@protocol Animal <NSObject>

/**
 *  动物的名字
 */
- (NSString *)name;

/**
 *  动物的类型
 */
- (NSString *)type;

/**
 *  叫
 */
- (void)call;

/**
 *  跑步
 */
- (void)run;

@end
```

###接下来那么就是接口实现咯，简单两个实现类 Dog 和 Cat

- Dog

```objc
#import <Foundation/Foundation.h>
#import "Animal.h"

@interface Dog : NSObject <Animal>

@end
```

```objc
#import "Dog.h"

@implementation Dog

- (NSString *)name {
    return @"狗";
}

- (NSString *)type {
    return @"哺乳动物";
}

- (void)call {
    NSLog(@"狗在叫...");
}

- (void)run {
    NSLog(@"狗在跑...");
}

@end
```

- Cat

```objc
#import <Foundation/Foundation.h>
#import "Animal.h"

@interface Cat : NSObject <Animal>

@end
```

```objc
#import "Cat.h"

@implementation Cat

- (NSString *)name {
    return @"猫";
}

- (NSString *)type {
    return @"哺乳动物";
}

- (void)call {
    NSLog(@"猫在叫...");
}

- (void)run {
    NSLog(@"猫在跑...");
}

@end
```

再接着就是AnimalProxy类，作为所有Animal这一类对象的代理对象

***

###先看AnimalProxy.h，骗过编译器实现了Animal协议，让这个代理类假装是Animal类.

```objc
#import <Foundation/Foundation.h>
#import "Animal.h"

/**
 *  专门用来做Animal这一类对象的代理
 *  代理也实现Animal协议
 */
@interface AnimalProxy : NSProxy <Animal>

+ (instancetype)sharedInstance;

@end
```

###再来看AnimalProxy.m

```objc
#import "AnimalProxy.h"
#import <objc/runtime.h>

#import "Dog.h"
#import "Cat.h"

@interface AnimalProxy ()

@property (nonatomic,strong) NSMutableDictionary *selHandlerDict;

@end

@implementation AnimalProxy

+ (instancetype)sharedInstance {
    
    static AnimalProxy *_sharedProxy = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        
        //NSProxy没有init方法
        _sharedProxy = [AnimalProxy alloc];
        
        //创建SEL与接口实现对象的关联
        _sharedProxy.selHandlerDict = [NSMutableDictionary dictionary];
        
        //绑定 协议 与 具体实现类对象
        [_sharedProxy _registerHandlerProtocol:@protocol(Animal) handler:[Dog new]];
//        [_sharedProxy _registerHandlerProtocol:@protocol(Animal) handler:[Cat new]];
    });
    return _sharedProxy;
}

- (void)_registerHandlerProtocol:(Protocol *)protocol handler:(id)handler {
    
    //记录protocol中的方法个数
    unsigned int numberOfMethods = 0;
    
    //获取protocol中定义的方法描述
    struct objc_method_description *methods = protocol_copyMethodDescriptionList(protocol,
                                                                                 YES,//是否必须实现
                                                                                 YES,//是否对象方法
                                                                                 &numberOfMethods);
    
    //遍历所有的方法描述，设置其Target对象
    for (unsigned int i = 0; i < numberOfMethods; i++) {
        
        //方法描述结构体实例
        struct objc_method_description method = methods[i];
        
        //方法的名字
        NSString *methodNameStr = NSStringFromSelector(method.name);
        
        //保存所有方法名对应的Target对象，最终接收处理消息
        [_selHandlerDict setValue:handler forKey:methodNameStr];
    }
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    
    //获取当前触发消息的SEL
    SEL sel = invocation.selector;
    
    //SEL的字符串
    NSString *methodNameStr = NSStringFromSelector(sel);
    
    //SEL字符串查询字典得到保存的消息接收者Target
    id target = [_selHandlerDict objectForKey:methodNameStr];
    
    //是否找到了要转发SEL执行的Target
    if (target && [target respondsToSelector:sel])
    {
        //找到Target，就让Target处理消息
        [invocation invokeWithTarget:target];
        
    } else {
        
        //未找到Target 或 Target未实现方法，交给super去转发消息
        [super forwardInvocation:invocation];
    }
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    
    //SEL的字符串
    NSString *methodNameStr = NSStringFromSelector(sel);
    
    //SEL字符串查询字典得到保存的消息接收者Target
    id target = [_selHandlerDict objectForKey:methodNameStr];
    
    //是否找到了要转发SEL执行的Target
    if (target && [target respondsToSelector:sel]) {
        
        //找到了Target，那么获取找到的Target的SEL对应的方法签名
        return [target methodSignatureForSelector:sel];
        
    } else {
        
        //未找到Target 或 Target未实现方法，交给super
        return [super methodSignatureForSelector:sel];
    }
}

@end
```

其实上面使用了消息转发阶段二来转发消息，其实不用这么复杂的，只是为了学习下而已，完全可以在消息转发阶段一的`forwardingTargetForSelector:`返回接口实现类对象即可。

****

###分解上面主要步骤:

- 创建代理对象时，绑定接口中每一个方法稍后处理的Target对象

- 因为当前代理对象，没有实现任何Animal协议方法。那么，当调用代理对象的Animal协议方法时，会由系统调用如下两种进行处理SEL找不到对应方法实现时情况，进入消息转发
	- 消息转发阶段一、resolvc instance/class method
	- 消息转发阶段二、forward target/invocation

- 重写消息转发函数，将消息重新发送给与SEL方法名绑定的Target

***

###ViewControlelr中使用AnimalProxy类对象，来间接的操作Dog对象.

```objc
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];

    
    NSLog(@"name = %@", [[AnimalProxy sharedInstance] name]);
    NSLog(@"type = %@", [[AnimalProxy sharedInstance] type]);

    [[AnimalProxy sharedInstance] call];
    [[AnimalProxy sharedInstance] run];
    
}
```

###输出结果

```
name = 狗
type = 哺乳动物
狗在叫...
狗在跑...
```

###可以看到使用的是Dog类对象，但是在ViewController这个类中，我们是使用的AnimalProxy这个类对象的Animal协议方法，根本没有出现过任何的Dog类代码。

到这里可能看不出任何优势，好继续...

***

###那么假设哪一天我们需要将Dog换成Cat，那么只需要修改AnimalProxy类绑定的代码

```objc
@implementation AnimalProxy

+ (instancetype)sharedInstance {
    
    static AnimalProxy *_sharedProxy = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        
        //NSProxy没有init方法
        _sharedProxy = [AnimalProxy alloc];
        
        //创建SEL与接口实现对象的关联
        _sharedProxy.selHandlerDict = [NSMutableDictionary dictionary];
        
        //绑定 协议 与 具体实现类对象
//        [_sharedProxy _registerHandlerProtocol:@protocol(Animal) handler:[Dog new]];
        [_sharedProxy _registerHandlerProtocol:@protocol(Animal) handler:[Cat new]];
    });
    return _sharedProxy;
}

//....
```

仅仅需要将Animal协议的实现类，从Dog切换到Cat即可，而外界ViewController中的代码几乎不用做任何修改。


而如果把Dog的代码暴露给ViewController的到处都是，那就得全部工程全局搜索一个一个的替换，还是有一定工作量的。这个思想其实并不新鲜，在搞Java服务器中是很普通的抽象封装，只是稍微利用在iOS客户端代码中而已。

****

###下面这个例子是我一时兴起想到的，在上面的基础上使用一个配置plist更换版本实现类

- (1) 首先接口抽象


```objc
@protocol GetData  <NSObject>
- (NSString *)getData1;
- (NSString *)getData2;
- (NSString *)getData3;
@end
```

- (2) 不同版本的实现类

```objc
@interface __GetDataProxy_fromServerA : NSObject <GetData>
@end
@implementation __GetDataProxy_fromServerA

- (NSString *)getData1 {
    return @"使用server A 1 的数据";
}

- (NSString *)getData2 {
    return @"使用server A 2 的数据";
}

- (NSString *)getData3 {
    return @"使用server A 3 的数据";
}

@end
```

```objc
@interface __GetDataProxy_fromServerB : NSObject <GetData>
@end
@implementation __GetDataProxy_fromServerB

- (NSString *)getData1 {
    return @"使用server B 1 的数据";
}

- (NSString *)getData2 {
    return @"使用server B 2 的数据";
}

- (NSString *)getData3 {
    return @"使用server B 3 的数据";
}

@end
```

- (3) 使用一个plist配置工程中使用的实现类

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>ImplementationClass</key>
	<string>__GetDataProxy_fromServerB</string>
</dict>
</plist>
```

- (4) GetDataProxy获取单例时读取plist配置的实现类进行与协议protocol绑定

```objc
@implementation GetDataProxy {
    NSMutableDictionary *_handlerDic;
}

+ (instancetype)sharedInstance {
    static GetDataProxy *_proxy = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _proxy = [GetDataProxy new];
    });
    return _proxy;
}

/**
 *  因为是在dispatch_once()块中执行的，所以肯定是线程安全的
 *  但如果不是在dispatch_once()中执行，需要同步多线程进行操作 _handlerDic 缓存字典对象
 */
- (instancetype)init
{
    self = [super init];
    if (self) {
        _handlerDic = [NSMutableDictionary new];
        
        // 获取配置的实现类
        NSString *file = [[NSBundle mainBundle] pathForResource:@"GetSystemDataProxy" ofType:@"plist"];
        NSDictionary *configDic = [NSDictionary dictionaryWithContentsOfFile:file];
        
        // 实例化配置的实现类对象
        NSString *clsName = configDic[@"ImplementationClass"];
        Class cls = NSClassFromString(clsName);
        id<GetData> getData = [[cls alloc] init];
        
        // 绑定得到对象作为协议实现类对象
        [self _registerHandlerProtocol:@protocol(GetData) handler:getData];
    }
    return self;
}

/**
 *  绑定协议中所有定义方法对应的实现类对象
 *
 *  @param protocol 协议
 *  @param handler  协议中每一个方法发送的消息处理对象
 */
- (void)_registerHandlerProtocol:(Protocol *)protocol handler:(id)handler {
    
    //记录protocol中的方法个数
    unsigned int numberOfMethods = 0;
    
    //获取protocol中定义的方法描述
    struct objc_method_description *methods = protocol_copyMethodDescriptionList(protocol,
                                                                                 YES,//是否必须实现
                                                                                 YES,//是否对象方法
                                                                                 &numberOfMethods);
    
    //遍历所有的方法描述，设置其Target对象
    for (unsigned int i = 0; i < numberOfMethods; i++) {
        
        //方法描述结构体实例
        struct objc_method_description method = methods[i];
        
        //方法的名字
        NSString *methodNameStr = NSStringFromSelector(method.name);
        
        //保存所有方法名对应的Target对象，最终接收处理消息
        [_handlerDic setValue:handler forKey:methodNameStr];
    }
}

- (id)forwardingTargetForSelector:(SEL)aSelector {
    
    NSString *selString = NSStringFromSelector(aSelector);
    id<GetData> getData = [_handlerDic objectForKey:selString];
    if (getData) {
        return getData;
    }
    
    return [super forwardingTargetForSelector:aSelector];
}

@end
```

- (5) ViewController中测试

```objc
@implementation RuntimeController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self demo6];
}

- (void)demo6 {
    NSLog(@"data 1 = %@", [[GetDataProxy sharedInstance] getData1]);
    NSLog(@"data 2 = %@", [[GetDataProxy sharedInstance] getData2]);
    NSLog(@"data 3 = %@", [[GetDataProxy sharedInstance] getData3]);
}

@end
```

输出log

```
2016-07-23 23:46:25.396 Demos[18839:229499] data 1 = 使用server B 1 的数据
2016-07-23 23:46:25.396 Demos[18839:229499] data 2 = 使用server B 2 的数据
2016-07-23 23:46:25.396 Demos[18839:229499] data 3 = 使用server B 3 的数据
```

- (6) 修改plist配置的实现类，再次运行程序

```objc
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>ImplementationClass</key>
	<string>__GetDataProxy_fromServerA</string>
</dict>
</plist>
```

修改完plist配置文件之后直接运行程序即可，无序修改Proxy内任何代码和ViewController外部调用代码

```
2016-07-23 23:48:15.427 Demos[19121:236373] data 1 = 使用server A 1 的数据
2016-07-23 23:48:15.427 Demos[19121:236373] data 2 = 使用server A 2 的数据
2016-07-23 23:48:15.427 Demos[19121:236373] data 3 = 使用server A 3 的数据
```

***

###小结NSProxy类使用

- 可以提供类似Aop方法拦截，以及面向切面编程思路

- 屏蔽内部具体实现类，随时切换实现类

- 主要重写NSProxy如下两个方法，在处理消息转发时，将消息转发给真正的Target处理

```objc
- (void)forwardInvocation:(NSInvocation *)invocation;
- (nullable NSMethodSignature *)methodSignatureForSelector:(SEL)sel 
```