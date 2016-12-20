---
layout: post
title: "Runtime应用防止按钮连续点击"
date: 2016-04-22 15:09:25 +0800
comments: true
categories: 
---

###好久之前就看到过使用Runtime解决按钮的连续点击的问题，一直觉得没啥好记录的。刚好今天旁边同时碰到这个问题，看他们好捉急而且好像很难处理，于是我先自己看看...


前面自己也学习了很多Runtime的东西，一直觉得这个按钮连续点击其实很简单，就使用Runtime交换SEL实现IMP即可，但其实没明白解决这个问题的过程.

虽然直接可以在github搜到解决方法，但是还是有必要学习一下解决这个问题的一步一步的思路，给出这个作者的git:

```
https://github.com/strivever/UIButton-touch
```

***

```objc
@implementation ViewController

- (void)btnDidClick:(id)sender {
    NSLog(@"我被点击了....");
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    MyButton *btn = [[MyButton alloc] init];
    [btn setTitle:@"点我啊" forState:UIControlStateNormal];
    [btn setTitleColor:[UIColor blackColor] forState:UIControlStateNormal];
    btn.layer.borderWidth = 1;
    btn.frame = CGRectMake(50, 100, 100, 50);
    [self.view addSubview:btn];
    
    [btn addTarget:self action:@selector(btnDidClick:) forControlEvents:UIControlEventTouchUpInside];
}
```

如上代码是最简单的UIBUtton使用代码，但是有一个问题就是，按钮可以无限制、没有间隔时间、连续的n次点击都会触发处理函数.

****

###iOS中的按钮事件机制 >>> Target-Action机制

- 用户点击时，产生一个按钮点击事件消息
- 这个消息发送给注册的Target处理
- Target接收到消息，然后查找自己的SEL对应的具体实现IMP正儿八经的去处理点击事件


实际上该点击消息包含三个东西:

- Target处理者
- SEL方法Id
- 按钮事件当时触发时的状态
	
```
所有的按钮事件状态

typedef NS_OPTIONS(NSUInteger, UIControlState) {
    UIControlStateNormal       = 0,
    UIControlStateHighlighted  = 1 << 0,                  // used when UIControl isHighlighted is set
    UIControlStateDisabled     = 1 << 1,
    UIControlStateSelected     = 1 << 2,                  // flag usable by app (see below)
    UIControlStateFocused NS_ENUM_AVAILABLE_IOS(9_0) = 1 << 3, // Applicable only when the screen supports focus
    UIControlStateApplication  = 0x00FF0000,              // additional flags available for application use
    UIControlStateReserved     = 0xFF000000               // flags reserved for internal framework use
};
```

****

###已经知道点击按钮时候，会产生一个包装了Target、SEL、按钮事件状态三个东西的消息发送给Target处理

> 问题: 是谁来包装UIButton的点击事件消息，并且完成发送消息了？

这个是解决连续点击按钮的关键问题所在，必须搞清楚。因为如果搞清楚具体包装和发送按钮点击时间消息的地方和时机，那么可以拦截这个地方执行，然后加入是否在指定的间隔时间内决定是否让其继续执行发送消息的操作。

那么问题不就解决了吗，我都不让他发送消息了，他还能执行？

***

###首先从UIButton.h头文件中查找，是否有`send message` 、`send Action` ...等等包含`send`的方法

无法找到.

****

###UIButton继承自UIControl，而UIControl又负责很多的UI事件处理，那么可以继续从UIControl.h中查找

找到两个send相关的函数:

```objc
// send the action. the first method is called for the event and is a point at which you can observe or override behavior. it is called repeately by the second.
- (void)sendAction:(SEL)action to:(nullable id)target forEvent:(nullable UIEvent *)event;
```

```objc
- (void)sendActionsForControlEvents:(UIControlEvents)controlEvents;                        // send all actions associated with events
```

没看懂注释有什么意思，那么代码直接试把.

####前面我用自己的UIButton子类是有原因的，可以重写父类方法完成父类方法Hook的效果.

尝试进行hook UIControl的 `sendAction:to: forEvent:`

```objc
#import <UIKit/UIKit.h>

@interface MyButton : UIButton

@end
```

```objc
#import "MyButton.h"

@implementation MyButton

- (void)sendAction:(SEL)action to:(id)target forEvent:(UIEvent *)event {
    
    NSLog(@"Pre sendAction >>>> action = %@", NSStringFromSelector(action));
    
    [super sendAction:action to:target forEvent:event];
    
    NSLog(@"After sendAction >>>> action = %@", NSStringFromSelector(action));
}

@end
```

然后在ViewController中也进行下修改，确定按钮响应函数与这个`sendAction:to: forEvent:`执行的顺序.

```objc
@implementation ViewController

- (void)btnDidClick:(id)sender {
    NSLog(@"我被点击了 >>> %@", NSStringFromSelector(_cmd));
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    MyButton *btn = [[MyButton alloc] init];
    [btn setTitle:@"点我啊" forState:UIControlStateNormal];
    [btn setTitleColor:[UIColor blackColor] forState:UIControlStateNormal];
    btn.layer.borderWidth = 1;
    btn.frame = CGRectMake(50, 100, 100, 50);
    [self.view addSubview:btn];
    
    [btn addTarget:self action:@selector(btnDidClick:) forControlEvents:UIControlEventTouchUpInside];
}
```

最后输出结果如下

```
2016-04-22 15:43:14.181 RuntimeDemo[35850:366849] Pre sendAction >>>> action = btnDidClick:
2016-04-22 15:43:14.183 RuntimeDemo[35850:366849] 我被点击了 >>> btnDidClick:
2016-04-22 15:43:14.183 RuntimeDemo[35850:366849] After sendAction >>>> action = btnDidClick:
```

从如上的输出结果，分析一下:

- 当点击按钮时，立刻执行我们自己MyButton的`sendAction:to: forEvent:`方法实现

- 当继续执行`[UIControl sendAction:to: forEvent:]`时，就会完成如下工作，将流程走到ViewController对象这个Target
	- 按钮点击事件的消息包装
	- 发送给消息处理这Target

- 当Target接收到消息，进行处理
	- 即执行ViewController对象的`btnDidClick:`

- 当最后Target处理完消息，继续执行`[super sendAction:action to:target forEvent:event];`后面的一句打印

OK，理清楚从`按钮点击` ~ `消息包装与发送` ~ `消息处理` 这三个步骤，那么防止按钮连续点击就有突破口了.

> 最后摘录自来源文字关于UIControl的`sendAction:to:forEvent:`这个方法的作用:

- 对于一个给定的事件，UIControl会调用sendAction:to:forEvent:来将行为消息转发到`UIApplication`对象

- 再由UIApplication对象调用其sendAction:to:fromSender:forEvent:方法来将消息分发到指定的target上

***

###最终突破口 >>> UIControl完成按钮点击事件消息的包装与发送的阶段，可以做一些间隔时间处理点击消息发送

> 我们可以在UIControl的`sendAction:to:forEvent:`做防止按钮连续处理.

####那么大概有如下几种做法:

- 第一种、自定义我们的UIButton类，以后程序中都使用我们UIButton类（只适合新项目，不太适合老项目，用的地方太多了）

- 第二种、使用UIButton Category封装防止按钮连续点击处理的逻辑（这种挺好，对原来的UIButton使用代码绿色无公害）

- 第三站、直接在main.m中执行main()之前，就替换掉UIControl的`sendAction:to:forEvent:`具体实现（稍微有点复杂）

***

###首先看下使用UIButton子类实现

```objc
#import <UIKit/UIKit.h>

@interface MyButton : UIButton

/**
 *  按钮点击的间隔时间
 */
@property (nonatomic, assign) NSTimeInterval time;

@end
```

```objc
#import "MyButton.h"

// 默认的按钮点击时间
static const NSTimeInterval defaultDuration = 3.0f;

// 记录是否忽略按钮点击事件，默认第一次执行事件
static BOOL _isIgnoreEvent = NO;

// 设置执行按钮事件状态
static void resetState() {
    _isIgnoreEvent = NO;
}

@implementation MyButton

- (void)sendAction:(SEL)action to:(id)target forEvent:(UIEvent *)event {
    
    //1. 按钮点击间隔事件
    _time = _time == 0 ? defaultDuration : _time;
    
    //2. 是否忽略按钮点击事件
    if (_isIgnoreEvent) {
        //2.1 忽略按钮事件
        
        // 直接拦截掉super函数进行发送消息
        return;
        
    } else if(_time > 0) {
        //2.2 不忽略按钮事件
        
        // 后续在间隔时间内直接忽略按钮事件
        _isIgnoreEvent = YES;
        
        // 间隔事件后，执行按钮事件
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(_time * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            resetState();
        });
        
        // 发送按钮点击消息
        [super sendAction:action to:target forEvent:event];
    }
}

@end
```

ViewController中测试

```objc
@implementation ViewController

- (void)btnDidClick:(id)sender {
    NSLog(@"我被点击了 >>> %@", NSStringFromSelector(_cmd));
}

- (void)viewDidLoad {
    [super viewDidLoad];
	
	MyButton *btn = [[MyButton alloc] init];
    [btn setTitle:@"点我啊" forState:UIControlStateNormal];
    [btn setTitleColor:[UIColor blackColor] forState:UIControlStateNormal];
    btn.layer.borderWidth = 1;
    btn.frame = CGRectMake(50, 100, 100, 50);
    [self.view addSubview:btn];
    
    // 设置按钮的点击间隔时间
    btn.time = 2.f;
    
    [btn addTarget:self action:@selector(btnDidClick:) forControlEvents:UIControlEventTouchUpInside];
}
```

运行程序后狂点按钮后的log如下

```
2016-04-22 16:58:39.998 RuntimeDemo[40146:474695] 我被点击了 >>> btnDidClick:
2016-04-22 16:58:42.308 RuntimeDemo[40146:474695] 我被点击了 >>> btnDidClick:
2016-04-22 16:58:44.545 RuntimeDemo[40146:474695] 我被点击了 >>> btnDidClick:
2016-04-22 16:58:46.783 RuntimeDemo[40146:474695] 我被点击了 >>> btnDidClick:
2016-04-22 16:58:49.046 RuntimeDemo[40146:474695] 我被点击了 >>> btnDidClick:
2016-04-22 16:58:51.281 RuntimeDemo[40146:474695] 我被点击了 >>> btnDidClick:
2016-04-22 16:58:53.526 RuntimeDemo[40146:474695] 我被点击了 >>> btnDidClick:
2016-04-22 16:58:55.886 RuntimeDemo[40146:474695] 我被点击了 >>> btnDidClick:
```

可以看到点击间隔最小是2秒

****

###使用UIButton Category封装防止按钮连续点击的具体实现

> 其实大体上逻辑和上面的实现差不多，只是因为在Category分类里面，无法完成`重写sendAction:to:forEvent:`对应的实现，只能通过`运行时替换掉sendAction:to:forEvent:具体实现`之后拦截到UIButton的sendAction:to:forEvent:方式执行时，将上面例子的逻辑加进来.

- UIButton分类完成按钮防止连续点击的代码实现

```objc
#import <UIKit/UIKit.h>

@interface UIButton (Helper)

/**
 *  按钮点击的间隔时间
 */
@property (nonatomic, assign) NSTimeInterval clickDurationTime;

@end
```

```objc
#import "UIButton+Helper.h"
#import <objc/runtime.h>

// 默认的按钮点击时间
static const NSTimeInterval defaultDuration = 3.0f;

// 记录是否忽略按钮点击事件，默认第一次执行事件
static BOOL _isIgnoreEvent = NO;

// 设置执行按钮事件状态
static void resetState() {
    _isIgnoreEvent = NO;
}

@implementation UIButton (Helper)

@dynamic clickDurationTime;

+ (void)load {
    SEL originSEL = @selector(sendAction:to:forEvent:);
    SEL mySEL = @selector(my_sendAction:to:forEvent:);
    
    Method originM = class_getInstanceMethod([self class], originSEL);
    const char *typeEncodinds = method_getTypeEncoding(originM);
    
    Method newM = class_getInstanceMethod([self class], mySEL);
    IMP newIMP = method_getImplementation(newM);

    if (class_addMethod([self class], mySEL, newIMP, typeEncodinds)) {
        class_replaceMethod([self class], originSEL, newIMP, typeEncodinds);
    } else {
        method_exchangeImplementations(originM, newM);
    }
}

- (void)my_sendAction:(SEL)action to:(id)target forEvent:(UIEvent *)event {
    
    // 保险起见，判断下Class类型
    if ([self isKindOfClass:[UIButton class]]) {
        
        //1. 按钮点击间隔事件
        self.clickDurationTime = self.clickDurationTime == 0 ? defaultDuration : self.clickDurationTime;
        
        //2. 是否忽略按钮点击事件
        if (_isIgnoreEvent) {
            //2.1 忽略按钮事件
            return;
        } else if(self.clickDurationTime > 0) {
            //2.2 不忽略按钮事件
            
            // 后续在间隔时间内直接忽略按钮事件
            _isIgnoreEvent = YES;
            
            // 间隔事件后，执行按钮事件
            dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(self.clickDurationTime * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
                resetState();
            });
            
            // 发送按钮点击消息
            [self my_sendAction:action to:target forEvent:event];
        }

    } else {
        [self my_sendAction:action to:target forEvent:event];
    }
}

#pragma mark - associate

- (void)setClickDurationTime:(NSTimeInterval)clickDurationTime {
    objc_setAssociatedObject(self, @selector(clickDurationTime), @(clickDurationTime), OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (NSTimeInterval)clickDurationTime {
    return [objc_getAssociatedObject(self, @selector(clickDurationTime)) doubleValue];
}

@end
```

对作者的代码稍微做了一些修改，将一些不必要的oc函数直接写成c函数、c全局变量.

- 使用分类的UIButton类

```objc
#import <UIKit/UIKit.h>

//导入分类即可
#import "UIButton+Helper.h"

@interface MyButton : UIButton

@end
```

```objc
#import "MyButton.h"

@implementation MyButton

@end
```

我们的按钮类不需要做任何的事情，完全不知道被拦截附加完成了防止连续点击的逻辑.

- 最后ViewController测试类

基本上不需要做什么修改，可以导入UIButton分类，对该按钮设置点击间隔时间.


OK，这个问题就到此为止了解决了，以及整个分析的过程记录完毕.

***

###学习来源


```
http://www.cocoachina.com/ios/20160111/14932.html
```