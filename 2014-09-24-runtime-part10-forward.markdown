---
layout: post
title: "Runtime Part10 Forward"
date: 2014-09-24 00:26:56 +0800
comments: true
categories: 
---

###完整的消息转发阶段

- foward to a target 
- forward with NSInvocation

***

###当向对象或类发送一个没有对应方法实现的消息SEL时，由系统自动调用消息转发的几个系统函数。

注意： 必须是执行一个不存在方法实现的SEL，才会调用如下的几个函数.

> 可以作为实现方法拦截的思路之一，JSPatch正式利用这个系统消息转发函数进行获取目标方法执行所需要的所有的参数值，再自己手动构造NSInvocation发送消息.

本文介绍使用 `Forward 消息转发`的方式处理 没有找到 SEL对应的Method.


****

###当Resolve动态决议函数中返回NO，而foward消息转发分为两个阶段。如果第一个阶段返回一个不是nil的对象，那么就不会再进入第二个阶段。

- 消息转发阶段一、转发给一个具备方法实现的`对象`去处理
	- 如果该函数返回一个 非Nil或非nil的Target，那么就不会再进入第二个阶段
	- 反之，就会进入阶段二，让开发者自己手动创建NSInvocation完成消息转发

```
- (id)forwardingTargetForSelector:(SEL)aSelector;
```

- 消息转发阶段二、自己构造一个`NSInvocation消息对象`，再设置消息接收者`对象`，并设置参数值，并执行 -[NSInvocation invoke];完成消息发送.	
	- 能够获取到目标实现函数执行所需要的`所有的参数值`
	- 能得到最终转发消息执行的`返回值`
	- 并能够做出一些额外的修改
		- 修改传入的方法参数值（防止nil、null处理）
			- 追加参数
			- 删除参数
			- 对参数做预处理（做一些参数防止null、nil处理）
		- 修改返回值


```
//这种方法处理转发更为灵活，可以做很多处理
- (void)forwardInvocation:(NSInvocation *)anInvocation;

//同时需要重写这个函数，来提供目标实现函数的编码（提供: 返回值类型、参数类型、函数名...等等信息）
- (nullable NSMethodSignature *)methodSignatureForSelector:(SEL)sel;
```

***

###对于对象方法的消息转发

####阶段一、消息转发的举例


```
@implementation Person
	
- (void)dynamicAddMethodDemo {
    
    //1. 首先从外部隐式调用一个不存在的方法(一定是一个不存在实现的方法SEL)
    [self performSelector:@selector(noExistMethod:) withObject:@"test"];
}
	
//2.【重要】重写转发的方法，处理当前未找到实现的SEL调用，将消息转发给其他target处理（如此处的Car对象处理）
//注意: 
//这个方法只是由系统传入了被调用函数的SEL，
//无法拿到函数执行需要的参数值，所以就不能做预先处理等等
- (id)forwardingTargetForSelector:(SEL)aSelector {
    
	if ([NSStringFromSelector(aSelector) isEqualToString:@"noExistMethod:"]) {
	    
		 //返回一个具体当前SEL对应的方法实现的target（类或对象）
		 //但是替换的对象千万不要是self，那样会进入死循环
	    return [Car new];
	}
	    
	return [super forwardingTargetForSelector:aSelector];
}
```
	
```
//用来处理 SEL = @"noExistMethod:"的target
	
@interface Car : NSObject
	
//.h文件中，可以不用声明方法的定义
- (void)noExistMethod:(NSString *)str;

@end
	
@implementation Car

- (void)noExistMethod:(NSString *)str {
    //最终由Person对象转发到这里处理
}

@end
```

####阶段二、消息转发的举例

- 系统将方法调用包装成一个`NSInvocation对象`消息，包括 
	- 方法的SEL
	- 目标target
	- 方法执行的参数

```
@implementation Person
	
- (void)dynamicAddMethodDemo {
    
    //1. 首先从外部隐式调用一个不存在的方法(一定是一个不存在实现的方法SEL)
    [self performSelector:@selector(noExistMethod:) withObject:@"test"];
}
	
	
//2.【重要】必须重写如下方法，返回SEL对应的方法签名
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    
	// 从父类获取方法签名
	NSMethodSignature *signature = [super methodSignatureForSelector:aSelector];
	    
	// 从存在这个SEL对应方法实现的target中，获取SEL对应方法实现的签名（这里Car实例有这个SEL的方法实现，所以这里返回Car类中关于这个SEL的方法签名）
	if (!signature) {
	    if ([Car instancesRespondToSelector:aSelector]) {
	        signature = [Car instanceMethodSignatureForSelector:aSelector];
	    }
	}
	    
	return signature;
}
	
//3.【重要】重写该方法奖方法调用消息转发给其他的target来处理，并且可以修改方法参数、以及拿到函数执行的结果值
- (void)forwardInvocation:(NSInvocation *)anInvocation {

	//首先，我们可以对方法参数进行一些处理
	//如下对参数操作，都是传递的地址c指针，如果是OC对象，也必须取地址
    
    //获取方法执行的参数
    id arg1 = nil;
    [anInvocation getArgument:&arg1 atIndex:2];
    
    //修改方法执行的参数
    id arg2 = nil;
    [anInvocation setArgument:&arg2 atIndex:2];
    
    //追加方法执行的参数，也就是set到下一个下标的位置
    id arg3 = nil;
    [anInvocation setArgument:&arg3 atIndex:3];
			
	//转发给其他target，并处理消息执行方法
	if ([Car instancesRespondToSelector:anInvocation.selector]) {
	    
	    //待转发到的target
	    Car *target = [Car new];
	    
	    //转发消息给这个target，并执行
	    [anInvocation invokeWithTarget:target];
	    
	    //上面那句，等价于如下两句
		//[anInvocation setTarget:target];
		//[invocatin invoke];
	}
	
	//设置方法执行后返回值
    id retValue = nil;
    [anInvocation setReturnValue:&retValue];
    
    //获取方法执行后返回值
    [anInvocation getReturnValue:&retValue];
    
	//转发完毕，做其他的操作
	//....
}

```
	
```
//用来处理 SEL = @"noExistMethod:"的target
	
@interface Car : NSObject
	
//.h文件中，可以不用声明方法的定义
- (void)noExistMethod:(NSString *)str;

@end
	
@implementation Car

- (void)noExistMethod:(NSString *)str {
    //最终由Person对象转发到这里处理
}

@end
```

***

###对象方法可以进入forward消息转发阶段，但对于`类方法`就无法进入forward消息转发阶段了

因为NSObject类提供的消息转发就如下几种

- resolve

```objc
+ (BOOL)resolveClassMethod:(SEL)sel
```

```objc
+ (BOOL)resolveInstanceMethod:(SEL)sel
```

- forward

```objc
- (id)forwardingTargetForSelector:(SEL)aSelector
```

```objc
- (void)forwardInvocation:(NSInvocation *)anInvocation
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector 
```

如上能对`类方法`进行处理的就只有`+ (BOOL)resolveClassMethod:(SEL)sel`阶段中对于ClassMethod进行处理，在该函数实现中完成:

- 找到一个具备SEL对于实现的Target进行消息处理
- return NO; 不进入第二个阶段的消息转发

但是如果return YES;就会进入完整的消息转发阶段forwardXxx，但是对于forward系列的函数都是`对象方法`，显然只会对`对象方法`进行转发，而不会对`类方法`处理转发.

从如上的NSObject提供的消息转发函数来看，对于`类方法`只有一次机会进行处理。但是还是觉得不合理啊，再看看...

***

###如何让类方法也进入foward转发阶段，并完成转发了？

####我个人的蠢办法

- 因为类方法的实现，都是存放在Meta类中的，也就是Meta类的objc_class结构体实例中

- 那么是不是就是拿到Meta类来重写forward函数就可以了？

- 但是Meta类我们是看不到的，只有通过`object_getClass([类 class])`来获取到对应的Meta类的Class对象

- 然后再通过`Class_addMethod()、Class_repleaceMethod()`来重写Meta类中的forward函数完成消息转发

这仅仅是我个人突发奇想，没有任何的实现根据，觉得好笑也不是特别靠谱...

修正下，可以不用去Meta类对应的Objc_class获取类方法实现，直接可以通过

```objc
// 获取类方法实现
Method m = class_getClassMethod([self class], @selector(classMethod));
const char *type = method_getTypeEncoding(m);
    
// 获取类方法签名
NSMethodSignature *ms = [[self class] methodSignatureForSelector:@selector(class)];
```

####再看看有无其他方法让类方法也进入forward消息转发


- 首先看交换类方法的实现

```objc
#import <Foundation/Foundation.h>

@interface Swizzle : NSObject

+ (void)test1;
+ (void)test1_hook;

@end
```

```objc
#import "Swizzle.h"
#import <objc/runtime.h>

@implementation Swizzle

+ (void)load {
    Method method1 = class_getClassMethod([self class], @selector(test1));
    Method method2 = class_getClassMethod([self class], @selector(test1_hook));
    
    method_exchangeImplementations(method1, method2);
}

+ (void)test1 {
    NSLog(@"test1");
}

+ (void)test1_hook {
    NSLog(@"test1 pre");
    
    [self test1_hook];
    
    NSLog(@"test2 last");
}

@end
```

直接可以通过`Method class_getClassMethod(Class cls, SEL name)`获取到类方法的实现，就不用像之前说的先获取Meta类的Class，再获取类方法的实现了。

但是NSObject类提供的`forwardingTargetForSelector:`和`forwardInvocation:`消息转发函数，都是针对`对象方法`，如果将一个类的`对象方法`实现与`类方法`实现进行交换，好像不太合理了，毕竟两个方法的类型都不同，而且一个存放类本身，一个是存放Meta类中。

对于`类方法`和`对象方法`不能交叉进行替换实现，必须对象方法之间替换，类方法之前替换，比如如下代码:

```objc
分情况进行 1)类方法交换实现 2)对象方法交换实现

/**
 *  交换一个类中的两个方法SEL指向的IMP
 *
 *  @param cls       类
 *  @param origSEL   原始方法的SEL
 *  @param newSEL    交换后新方法的SEL
 */
void Swizzle(Class cls, SEL origSEL, SEL newSEL)
{
    
    //1. 首先尝试获取对象方法的实现
    Method origMethod = class_getInstanceMethod(cls, origSEL);
    Method newMethod = nil;
    
    //2. 判断originMethod是`类`方法还是`对象`方法
    if (!origMethod) {
        
        //2.1 交换类方法实现
        
        // 获取类方法
        origMethod = class_getClassMethod(cls, origSEL);
        
        if (!origMethod) {
            return;
        }
        
        //说明是交换类方法，则查找新的方法(拦截的方法)
        newMethod = class_getClassMethod(cls, newSEL);
        
        //类中不存在新方法实现
        if (!newMethod) {
            return;
        }
        
    }else{
        
        //2.2 交换对象方法
        
        //获取新方法实现
        newMethod = class_getInstanceMethod(cls, newSEL);
        
        //未找到新方法实现
        if (!newMethod) {
            return;
        }
    }
    
    /**
		1. 这里有一个注意的问题 >>> 要交换的方法有两种情况:
        
        情况一、原始方法的实现，没有出现在当前类，而是出现在当前类的父类，也就是当前类中并未重新实现
        情况二、原始方法已经在当前类重新实现
        
        2. 如果是情况一，直接使用exchange交换SEL，那么可能造成`父类`的方法实现IMP被交换，就会导致其他所有子类都会受到影响。那显然我们只需要修改当前类，而不想要影响其他子类。
     
        3. 那么结合上面两种情况:
        	
        	- 首先，使用原始方法SEL，添加要替换成的新的实现IMP
          
          	- 如果添加成功 >>> 情况一
          		- 第一步、向当前类 添加`新的`方法实现，使用newSEL
          		- 第二步、`修改 replace`新方法SEL指向原始方法实现IMP，为了执行newSEL时回到原来的方法实现中

			- 如果添加失败 >>> 情况二
				- 直接`交换 exchange`原始方法与新方法的SEL指向的IMP
     
     */
    if(class_addMethod(cls, origSEL, method_getImplementation(newMethod), method_getTypeEncoding(newMethod)))
    {
        //情况一: 为了执行newSEL时回到原来的方式实现
        class_replaceMethod(cls, newSEL, method_getImplementation(origMethod), method_getTypeEncoding(origMethod));
    
    } else {
    
        //情况二: 直接交换了方法实现
        method_exchangeImplementations(origMethod, newMethod);
    }
}
```


####参看JSPatch中为了拿到目标函数执行时所有的参数值实现

> 核心: 直接交换目标函数实现，从而直接进入到系统实现的消息转发c函数（_objc_msgForward 或 _objc_msgForward_stret），从而获取到传入的NSInvocation对象，继而获取到所有的目标方法执行的参数值。

- `_objc_msgForward` 和  `_objc_msgForward_stret`这两个c函数，是c底层负责消息转发的具体函数.

```c
id _objc_msgForward(id receiver, SEL sel, ...) 
```

```c
void _objc_msgForward_stret(id receiver, SEL sel, ...) 
```

- JSPatch中获取目标方法执行时所有的参数的c函数

```c
static void overrideMethod(Class cls,                   
                           NSString *selectorName,      
                           JSValue *function,           
                           BOOL isClassMethod,          
                           const char *typeDescription) 
{
	//1. 要被替换实现的方法SEL
    SEL selector = NSSelectorFromString(selectorName);
    
    //2. 被替换实现的方法的编码
    if (!typeDescription) {
        Method method = class_getInstanceMethod(cls, selector);//这里
        
        typeDescription = (char *)method_getTypeEncoding(method);
    }
    
    //3. 被替换方法实现的IMP
    IMP originalImp = class_respondsToSelector(cls, selector) ? class_getMethodImplementation(cls, selector) : NULL;

    //4. 准备系统消息转发函数的IMP
    //4.1 如果cpu是arm64架构
    IMP msgForwardIMP = _objc_msgForward;
    //4.2 针对 `非arm64` cpu架构时，消息转发使用 _objc_msgForward_stret系统函数
    #if !defined(__arm64__)
        if (typeDescription[0] == '{') {
            //In some cases that returns struct, we should use the '_stret' API:
            //http://sealiesoftware.com/blog/archive/2008/10/30/objc_explain_objc_msgSend_stret.html
            //NSMethodSignature knows the detail but has no API to return, we can only get the info from debugDescription.
            NSMethodSignature *methodSignature = [NSMethodSignature signatureWithObjCTypes:typeDescription];
            if ([methodSignature.debugDescription rangeOfString:@"is special struct return? YES"].location != NSNotFound) {
                msgForwardIMP = (IMP)_objc_msgForward_stret;
            }
        }
    #endif
    
    //5. 让原来方法执行时，从而直接进入`系统消息转发函数（forwardInvocation:）`的实现中
    class_replaceMethod(cls, selector, msgForwardIMP, typeDescription); 
    
    //6. 拦截系统的消息转发函数（forwardInvocation:）执行
    #pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wundeclared-selector"
    if (class_getMethodImplementation(cls, @selector(forwardInvocation:)) != (IMP)JPForwardInvocation) {
    
    	// 替换掉当前类自己的forwardInvocation:方法实现
    	IMP originalForwardImp = class_replaceMethod(cls, @selector(forwardInvocation:), (IMP)JPForwardInvocation, "v@:@");
    	
    	// 然后增加一个新的方法，用来回到类自己的forwardInvocation:方法实现，完成系统的消息转发工作
    	class_addMethod(cls, @selector(ORIGforwardInvocation:), originalForwardImp, "v@:@");
    }
#pragma clang diagnostic pop
    
	.....
    
}
```

第一次看，觉得仍然只是针对类的对象方法进行消息转发。但是又想了一下，如果是需要拦截类方法的话，是不是只需要传入MetaClass就行了？

```objc
//1. 获取交换类的MetaClass
Class metaCls = objc_getMetaClass("NSObject子类名字");

//2. 传入MetaClass进行类方法交换
overrideMethod(metaCls, sel, ....);
```

我觉得这样就一点能够让类方法也进入到forwardInvocation:函数了，经过demo测试确实也是可以的。

***

###使用消息转发实现OC类的`多继承`

- (1) 有三个抽象接口

```objc
@protocol SayChinese <NSObject>
- (void)sayChinese;
@end

@protocol SayEnglish <NSObject>
- (void)sayEnglish;
@end

@protocol SayFranch <NSObject>
- (void)sayFranch;
@end
```

- (2) 我内部拥有三个具体实现类，但是我希望不对外暴露实现，只是我内部知道具体实现

```objc
@interface __SayChineseImpl : NSObject <SayChinese>
@end

@implementation __SayChineseImpl
- (void)sayChinese {
    NSLog(@"说中国话");
}
@end
```

```objc
@interface __SayEnglishImpl : NSObject <SayEnglish>
@end

@implementation __SayEnglishImpl
- (void)sayEnglish {
    NSLog(@"说英语话");
}
@end
```

```objc
@interface __SayFranchImpl : NSObject <SayFranch>
@end

@implementation __SayFranchImpl
- (void)sayFranch {
    NSLog(@"说法国话");
}
@end
```

- (3) 统一向外暴露一个类，来操作上面三个类所有方法实现，即多继承的效果

```objc
/**
 *  模拟继承多种语言
 */
@interface SpeckManager : NSObject <SayChinese, SayEnglish, SayFranch>

@end

@implementation SpeckManager {
    id<SayChinese> _sayC;
    id<SayEnglish> _sayE;
    id<SayFranch> _sayF;
}

- (instancetype)init
{
    self = [super init];
    if (self) {
        _sayC = [__SayChineseImpl new];
        _sayE = [__SayEnglishImpl new];
        _sayF = [__SayFranchImpl new];
    }
    return self;
}

- (id)forwardingTargetForSelector:(SEL)aSelector {
    
    if (aSelector == @selector(sayChinese)) {
        return _sayC;
    }
    
    if (aSelector == @selector(sayEnglish)) {
        return _sayE;
    }
    
    if (aSelector == @selector(sayFranch)) {
        return _sayF;
    }
    
    return [super forwardingTargetForSelector:aSelector];
}

@end
```

代码看起来还是挺简单的，最终就是将消息直接转发给对应的实现类对象即可。