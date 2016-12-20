---
layout: post
title: "Runtime Part12 Intercept"
date: 2014-09-25 00:44:46 +0800
comments: true
categories: 
---

###方法拦截的途径

- 方法一、执行一个SEL不存在Method的消息
	- 然后不让方法继续执行，直接拦截掉 （resolve）
	- 转发给其他具有这个Method实例的Target （forward）

- 方法二、使用runtime api，直接替换掉SEL指向的Method实例，让其找到一个假的函数实现

- 方法三、写一个子类去继承，然后重写父类方法

第一种方法都已经有文章记录了，这里就主要记录下第二种与第三种，实现方法拦截.

***

###第二种、主动使用Runtime Api，在`运行时替换掉两个方法的实现指针IMP（确切的是替换掉SEL指向的Method实例）`，达到拦截目标方法执行的目的

![Hook目标方法的核心思路](http://i13.tietuku.com/8bad815bcfca153c.png)


####拦截目标方法执行的思想: `交换方法SEL对应的方法实现IMP指针`
	
- 第一步、运行时替换掉两个方法的`SEL指向`的函数实现指针`IMP`（确切的是`Method`实例）
	
![Hook目标方法的核心思路](http://i13.tietuku.com/8bad815bcfca153c.png)			
- 第二步、执行完拦截方法之后，重新找到原始方法的实现

![](http://i12.tietuku.com/2feae62cbaf7ad93.png)


####目标方法`执行前`、`执行后`，动态插入的着一些`代码`，而这些动态插入的代码是可以单独封装的，也称为一个切面`Aspect`。

####那么一个Aspect的作用

- 这些Aspect随时可以插入到某个目标方法的执行前或执行后
- 让目标方法根本`不知道`的情况下，给他`附加`的做一些事情

####关于hook封装比较好的开源库，

- `Aspect`，很简单的完成方法拦截，让开发者更专注于切面代码的开发
- `JRSwizzle`	，稍微封装了下使用Runtime Api替换SEL执行的Method实例，不像`Aspect`封装的那么完美，但是也基本够用了

####Runtime应用、写一个`面向切面`的代码结构示例

- 先写一个原始类（要被拦截方法执行的类）

```
#import <Foundation/Foundation.h>

@interface TargetClass : NSObject

- (void)show;

@end
```

```
#import "TargetClass.h"

@implementation TargetClass

- (void)show {
    NSLog(@"I am SwizzleClass ... \n");
}

@end
```

- 再写切面Aspect类（用于封装一些需要在拦截目标方法时，动态插入的代码）

```
#import <Foundation/Foundation.h>

@interface AspectClass : NSObject

/**
 *	 拦截到目标方法执行前，做的事情...
 */
+ (void)beforeTarget;

/**
 *	拦截到目标方法执行后，做的事情...
 */
+ (void)afterTarget;

@end
```

```
#import "AspectClass.h"

@implementation AspectClass

+ (void)beforeTarget {
    NSLog(@"我是Aspect，在目标方法【执行前】做的事情...\n");
}

+ (void)afterTarget {
    NSLog(@"我是Aspect，在目标方法【执行后】做的事情...\n");
}

@end
```

- 然后是在`原始类的一个分类`，并重写其`load`时交换方法实现（在类的load方法交互，为了保证只交换一次，否则就会很乱）

	- 在分类里面做交换操作，为了不影响原始类里面的逻辑

```
#import "TargetClass.h"

@interface TargetClass (Swizzle)

@end
```

```
#import "TargetClass+Swizzle.h"

//导入切面类
#import "AspectClass.h"

//导入实现交换函数实现的助手类
#import "JRSwizzle.h"

@implementation TargetClass (Swizzle)

//load方法会在类第一次加载的时候被调用
//调用的时间比较靠前，适合在这个方法里做方法交换
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        
        NSError *error = nil;
        
        [TargetClass jr_swizzleMethod:@selector(show)
                           withMethod:@selector(aspect_show)
                                error:&error];
    });
}

/**
 * 被替换成的方法实现
 */
- (void)aspect_show {
    
    //1. 目标方法执行前 (统计、对目标方法的参数进行处理..)
    [AspectClass beforeTarget];
    
    //2. 目标方法执行
    [self aspect_show];
    
    //3. 目标方法执行后 (统计...)
    [AspectClass afterTarget];
}

@end
```

- 最终上面的代码执行后的结果

```
1. 我是Aspect，在目标方法【执行前】做的事情...
2. I am TargetClass ... 
3. 我是Aspect，在目标方法【执行后】做的事情...
```


***

###第三种、写一个`子类`继承自要拦截方法的所在父类，然后`重写`其要拦截的方法

- 主要的步骤
	- 第一步、重写要拦截的父类方法
	- 第二步、重写方法中，先做自己的代码操作
	- 第三步、重写方法中，最后执行父类的这个方法

现在我要拦截UINavigationController的`pushViewController:animated:`方法

- 首先，定义一个继承自UINavigationController子类

```
@interface ZSYBaseNavController : UINavigationController

@end
```

```
#pragma mark - 重写系统方法，进行拦截

//拦截pushViewController方法
- (void)pushViewController:(UIViewController *)viewController animated:(BOOL)animated 
{
	//1. 做点什么
	//...
	
	//2, 最后执行父类的push操作
	[super pushViewController:viewController animated:animated];
}

//拦截popToViewController操作
- (NSArray *)popToViewController:(UIViewController *)viewController animated:(BOOL)animated 
{
	//1. 做点什么
	//...
	
	//2, 最后执行父类的pop操作
	return [super popToViewController:viewController animated:animated];
}

//拦截popToRootViewControllerAnimated操作
- (NSArray *)popToRootViewControllerAnimated:(BOOL)animated 
{
    //1. 做点什么
	//...
	
	//2, 最后执行父类的pop操作
    return [super popToRootViewControllerAnimated:animated];
}

//拦截popViewControllerAnimated操作
- (UIViewController *)popViewControllerAnimated:(BOOL)animated 
{
   //1. 做点什么
	//...
	
	//2, 最后执行父类的pop操作    
	return [super popViewControllerAnimated:animated];
}

//拦截setViewControllers操作
- (void)setViewControllers:(NSArray *)viewControllers animated:(BOOL)animated 
{
    //1. 做点什么
	 //...
	
	//2, 最后执行父类的set操作
    [super setViewControllers:viewControllers animated:animated];
}

@end
```

- 然后，程序代码中使用我们自己定义的`UINavigationController子类`即可

***

###使用Runtime时交换函数实现，防止`数组操作不当`崩溃的问题

- 例子1. 写一个`NSArray+Safe`的分类完成交换方法实现，并且防止数组越界处理
	- 注意: 只需要写好这个分类就可以了，并不需要手动的在项目里#import

```
#import <Foundation/Foundation.h>

@interface NSArray (Safe)

- (id)safe_objectAtIndex:(NSUInteger)index;

@end
```

```
#import "NSArray+Safe.h"
#import "JRSwizzle.h"

@implementation NSArray (Safe)

+ (void)load {
    
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        
        //1. 获取要替换方法实现的Class
        Class arrayClass = [[NSArray array] class];
        
        //2. 替换方法实现
        NSError *error = nil;
        [arrayClass jr_swizzleMethod:@selector(objectAtIndex:)
                          withMethod:@selector(safe_objectAtIndex:)
                               error:&error];
    });
}

/**
 * 拦截掉系统方法，进行数组越界处理
 */
- (id)safe_objectAtIndex:(NSUInteger)index {
    
    if (index < self.count) {
        
        //让被拦截的系统方法继续执行
        return [self safe_objectAtIndex:index];
        
    } else {
        
        NSLog(@"数组越界，取消系统函数执行...\n");
        
        //停止系统方法执行
        return nil;
    }
}

@end
```

- 例子2. 写一个`NSMutableArray+Safe`分类，在运行时防止`NSMutableArray添加nil`导致崩溃的实现

```
#import <Foundation/Foundation.h>

@interface NSMutableArray (Safe)

- (void)safe_addObject:(id)anObject;

@end
```

```
#import "NSMutableArray+Safe.h"
#import "JRSwizzle.h"

@implementation NSMutableArray (Safe)

+ (void)load {
    
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        
        //1. 获取Class
        Class arrayClass = [[NSMutableArray array] class];
        
        //2. 调用类方法完成方法实现交换
        NSError *error = nil;
        [arrayClass jr_swizzleMethod:@selector(addObject:)
                          withMethod:@selector(safe_addObject:)
                               error:&error];
    });
}

- (void)safe_addObject:(id)anObject {
    
    if ((anObject == nil) || [anObject isEqual:[NSNull null]]) {
        NSLog(@"当前待添加到数组的数据不合法...\n");
    } else {
        [self safe_addObject:anObject];
    }
}

@end
```

- 小结如上两个例子:
	
	- 1) 使用Runtime来交换`系统方法实现`与`分类中我们自己写的方法实现`
	- 2) 这样做可以不用在其他地方写`#import "NSArray+safe.h"`导入头文件
	- 3) 对于其他任何地方，使用`NSArray`的相关代码并没有任何改变，他们并不知道给他替换了函数实现
	- 4) 所有的修改逻辑，都在一个分类里面

- 注意的问题
	
	- 1) 必须要找对交换方法实现的Class
	- 2) 只能对函数实现交换一次
	
- 找到某个`系统类`的真正的Class的方法

```
//1. 先得到这个系统类的一个示例
系统类 *实例指针 = [系统类 获取示例的方法];

//2. 再通过这个实例得到Class
Class cls = [实例指针 class];
```

***

###通过运行时创建动态代理完成拦截

- (1) 对于拦截目标对象的某个方法实现执行时，封装所有拦截逻辑的抽象协议

```objc
@protocol Aspect <NSObject>

@required

- (void)preInvoke:(NSInvocation *)invocation withTarget:(id)target;
- (void)afterInvoke:(NSInvocation *)invocation withTarget:(id)target;

@end
```

- (2) 抽象切面的具体实现切面类

```objc
@interface ConcreateAspect : NSObject <Aspect>
@end

@implementation ConcreateAspect

- (void)preInvoke:(NSInvocation *)invocation withTarget:(id)target {
    NSLog(@"preInvoke: %@, target: %@", invocation, target);
}

- (void)afterInvoke:(NSInvocation *)invocation withTarget:(id)target {
    NSLog(@"afterInvoke: %@, target: %@", invocation, target);
}

@end
```

- (3) 所有切面的管理类、以及目标对象的代理类

```objc
@interface AspectProxy : NSProxy

/**
 *  被拦截的目标对象
 */
@property (nonatomic, strong) id proxyTarget;

/**
 *  拦截之后执行逻辑的切面
 */
@property (nonatomic, strong) id<Aspect> aspect;

/**
 *  目标对象被拦截的所有消息sel
 */
@property (nonatomic, strong, readonly) NSMutableArray *selectors;

// 传入代理哪一个目标对象、要拦截目标对象哪些sel的消息、执行拦截的切面对象、
- (instancetype)initWithObject:(id)object andAspect:(id<Aspect>)aspect;
- (instancetype)initWithObject:(id)object selectors:(NSArray *)selectors andAspect:(id<Aspect>)aspect;

// 添加要拦截的消息对应的sel
- (void)registerSelector:(SEL)selector;

@end

@implementation AspectProxy

- (instancetype)initWithObject:(id)object
                     selectors:(NSArray *)selectors
                     andAspect:(id<Aspect>)aspect
{
    // 保存被拦截的目标对象
    _proxyTarget = object;
    
    // 拦截之后执行处理的切面对象
    _aspect = aspect;
    
    // 要拦截的消息对应的sel
    _selectors = [selectors mutableCopy];
    
    return self;
}

- (instancetype)initWithObject:(id)object
                     andAspect:(id<Aspect>)aspect
{
    return [self initWithObject:object
                      selectors:nil
                      andAspect:aspect];
}

#pragma mark - 重写消息转发函数，完成目标对象方法实现拦截，并执行切面逻辑

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    
    // 从被代理的目标对象查找SEL对应的方法签名
    return [_proxyTarget methodSignatureForSelector:sel];
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    
    //1. 调用前拦截
    if ([_aspect respondsToSelector:@selector(preInvoke:withTarget:)]) {
        if (_selectors) {
            SEL sel = [invocation selector];
            for (NSValue *value in _selectors) {
                if (sel == [value pointerValue]) {
                    [_aspect preInvoke:invocation withTarget:_proxyTarget];
                }
            }
        } else {
            [_aspect preInvoke:invocation withTarget:_proxyTarget];
        }
    }
    
    //2. 目标方法执行
    [invocation invokeWithTarget:_proxyTarget];
    
    //3. 调用后拦截
    if ([_aspect respondsToSelector:@selector(afterInvoke:withTarget:)]) {
        if (_selectors) {
            SEL sel = [invocation selector];
            for (NSValue *value in _selectors) {
                if (sel == [value pointerValue]) {
                    [_aspect afterInvoke:invocation withTarget:_proxyTarget];
                }
            }
        } else {
            [_aspect afterInvoke:invocation withTarget:_proxyTarget];
        }
    }
}

- (void)registerSelector:(SEL)selector {
    NSValue *value = [NSValue valueWithPointer:selector];
    [_selectors addObject:value];
}

@end
```

- (4) 任意一个将要被拦截的目标类

```objc
@interface Invoker : NSObject

- (void)doWork1;
- (void)doWork2;
- (void)doWork3;

@end

@implementation Invoker

- (void)doWork1 {
    NSLog(@"Invoker doWork 1");
}

- (void)doWork2 {
    NSLog(@"Invoker doWork 2");
}

- (void)doWork3 {
    NSLog(@"Invoker doWork 3");
}

@end
```

- (5) 拦截测试类

```objc
- (void)demo9 {
    
    //1. 目标类对象
    Invoker *target = [Invoker new];
    
    //2. 要拦截目标对象的哪一些sel的消息
    NSValue *value1 = [NSValue valueWithPointer:@selector(doWork1)];
    NSArray *selectors = @[value1];
    
    //3. 切面对象
    ConcreateAspect *aspect = [[ConcreateAspect alloc] init];
    
    //4. 管理切面的Proxy对象
    id proxy = [[AspectProxy alloc] initWithObject:target
                                         selectors:selectors
                                         andAspect:aspect];
    
    //5. 让Proxy对象进入消息转发
    [proxy doWork1];
    
    //6. 继续添加拦截另外的消息sel，并且也让Proxy对象进入消息转发
    [proxy registerSelector:@selector(doWork2)];
    [proxy doWork2];
    
    //7. 让Proxy对象进入消息转发，但是没有添加拦截的消息sel
    //因为未保存到AspectProxy对象的 _selectors数组中
    [proxy doWork3];
}
```

运行结果

```
2016-08-01 00:00:41.769 Demos[31990:463123] preInvoke: <NSInvocation: 0x7fddc1c2cd80>, target: <Invoker: 0x7fddc1c4cdc0>
2016-08-01 00:00:41.769 Demos[31990:463123] Invoker doWork 1
2016-08-01 00:00:41.769 Demos[31990:463123] afterInvoke: <NSInvocation: 0x7fddc1c2cd80>, target: <Invoker: 0x7fddc1c4cdc0>
2016-08-01 00:00:41.769 Demos[31990:463123] preInvoke: <NSInvocation: 0x7fddc1e2b170>, target: <Invoker: 0x7fddc1c4cdc0>
2016-08-01 00:00:41.770 Demos[31990:463123] Invoker doWork 2
2016-08-01 00:00:41.770 Demos[31990:463123] afterInvoke: <NSInvocation: 0x7fddc1e2b170>, target: <Invoker: 0x7fddc1c4cdc0>
2016-08-01 00:00:41.770 Demos[31990:463123] Invoker doWork 3
```

从结果可以看到，doWork1与doWork2这两个sel对应的方法实现调用时，之前和之后都加了拦截逻辑。但是doWork3函数实现执行没有加拦截的。

小结下吧:

- (1) 向Proxy类发送一个不存在实现的SEL消息
- (2) Proxy类对象的消息转发函数被系统调用
- (3) 在消息转发函数中，完成拦截逻辑调用
	- 之前做一点什么
	- `[invocation invokeWithTarget:被代理的目标对象]`将消息转发给目标对象去处理
	- 之后做一点什么


