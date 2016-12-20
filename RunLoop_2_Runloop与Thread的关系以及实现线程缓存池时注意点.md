##线程的种类

### 一次性 的线程对象

- (1) 使用默认的方法创建出来的线程对象，都只能`执行一次`任务代码
	- NSThread创建一次性线程
	- 注意: GCD Dispatch Queue底层会使用线程缓存池
	
- (2) 当一次性任务执行`完毕`之后，线程对象就会`被系统释放`所占用的内存资源

### 永久存活（常驻内存） 的线程对象

- (1) 有时候需要一个`永久`存活在后台的线程对象，为App程序在整个运行期间提供一些特定的数据服务、后台定时任务代码等等

- (2) 也就是说要让这个线程能`随时`处理事件、`无限次`处理事件、而且不能让线程对象`退出`执行


##如何实现一个永久存在的thread，并且可以随时不断的接收分配线程任务？

自定义一个NSThread

```objc
#import <Foundation/Foundation.h>

@interface MyThread : NSThread

@end
@implementation MyThread
- (void)dealloc {
    NSLog(@"MyThread dealloc >>>> %@", self);
}
@end
```

ViewController使用自定义的NSThread

```objc
@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    MyThread *t = [[MyThread alloc] initWithTarget:self selector:@selector(doWork) object:nil];
    [t start];
}

- (void)doWork {
    NSLog(@"MyThread doWork >>>> %@", [NSThread currentThread]);
}

@end
```

运行结果

```
2016-11-27 15:33:02.625 RunLoopBasic[1889:21240] MyThread doWork >>>> <MyThread: 0x7fa3fbc0d640>{number = 2, name = (null)}
2016-11-27 15:33:02.625 RunLoopBasic[1889:21240] MyThread dealloc >>>> <MyThread: 0x7fa3fbc0d640>{number = 2, name = (null)}
```

执行完线程体函数`doWork`之后，线程对象就被废弃了。

那么尝试，将线程对象使用成员变量保存起来。

```objc
@implementation ViewController{
    MyThread *_t;
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    _t = [[MyThread alloc] initWithTarget:self selector:@selector(doWork) object:nil];
    [_t start];
}

- (void)doWork {
    NSLog(@"MyThread doWork >>>> %@", [NSThread currentThread]);
}

@end
```

运行结果

```
2016-11-27 15:34:36.125 RunLoopBasic[1990:23159] MyThread doWork >>>> <MyThread: 0x7fad1bf07a80>{number = 2, name = (null)}
```

这样做的话，线程对象确实是不会被废弃了。那么试下能不能随时执行任务了？

```objc
@implementation ViewController{
    MyThread *_t;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    _t = [[MyThread alloc] initWithTarget:self selector:@selector(doInitThread) object:nil];
    [_t start];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
//    [self performSelector:@selector(doWork) onThread:_t withObject:nil waitUntilDone:YES]; waitUntilDone:YES 会崩溃
    [self performSelector:@selector(doWork) onThread:_t withObject:nil waitUntilDone:NO];
}

- (void)doInitThread {
    [_t setName:@"MyThread"];
    NSLog(@"MyThread doInitThread >>>> %@", [NSThread currentThread]);
}

- (void)doWork {
    NSLog(@"MyThread doWork >>>> %@", [NSThread currentThread]);
}

@end
```

运行之后，不论点击屏幕多少次只有一次初始化线程的log

```
2016-11-27 15:42:39.203 RunLoopBasic[2448:31255] MyThread doInitThread >>>> <MyThread: 0x7fcc4b50f760>{number = 2, name = MyThread}
```

只走了thread的init函数实现，但是performSelector的操作并没有被thread执行。

###既然NSThread对象没有被废弃，那为什么后续分配任务执行时，却无法响应了？

我大致怀疑可能是这样的:

- (1) 虽然NSThread对象的确是存在
- (2) 但是NSThread对象的状态可能是finished，造成无法再执行任务了


重新改写MyThread，添加description详细信息输出

```objc
@implementation MyThread
- (void)dealloc {
    NSLog(@"MyThread dealloc >>>> %@", self);
}
- (NSString *)description {
    return [NSString stringWithFormat:@"<%@ - %p>: isExecuting = %d, isCancelled = %d, isFinished = %d", [self class], self, self.isExecuting, self.isCancelled, self.isFinished];
}
- (NSString *)debugDescription {
    return [self description];
}
@end
```

改写UIViewController的测试代码

```objc
@implementation ViewController{
    MyThread *_t;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    _t = [[MyThread alloc] initWithTarget:self selector:@selector(doInitThread) object:nil];
    [_t start];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
//    [self performSelector:@selector(doWork) onThread:_t withObject:nil waitUntilDone:YES]; waitUntilDone:YES 会崩溃
    
    // 再给thread分配任务之前和之后添加状态输出的log
    NSLog(@"pre performSelector >>>> %@", _t);
    
    [self performSelector:@selector(doWork) onThread:_t withObject:nil waitUntilDone:NO];
    
    NSLog(@"after performSelector >>>> %@", _t);
}

- (void)doInitThread {
    [_t setName:@"MyThread"];
    NSLog(@"MyThread doInitThread >>>> %@", [NSThread currentThread]);
}

- (void)doWork {
    NSLog(@"MyThread doWork >>>> %@", [NSThread currentThread]);
}

@end
```

运行后，点击屏幕输出

```objc
2016-11-27 15:51:13.051 RunLoopBasic[2938:40303] MyThread doInitThread >>>> <MyThread - 0x7f83d2f6b040>: isExecuting = 1, isCancelled = 0, isFinished = 0
2016-11-27 15:52:12.663 RunLoopBasic[2938:40266] pre performSelector >>>> <MyThread - 0x7f83d2f6b040>: isExecuting = 0, isCancelled = 0, isFinished = 1
2016-11-27 15:52:12.663 RunLoopBasic[2938:40266] after performSelector >>>> <MyThread - 0x7f83d2f6b040>: isExecuting = 0, isCancelled = 0, isFinished = 1
```

可以看到点击屏幕的函数时，`thread.isFinished = 1`。说明thread已经执行结束了，那说明线程执行结束，就不会再响应分配给的任务了吗？

###尝试开启NSThread的runloop事件循环监听，看能否解决thread无法再响应任务的问题

```objc
@implementation ViewController{
    MyThread *_t;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    _t = [[MyThread alloc] initWithTarget:self selector:@selector(doInitThread) object:nil];
    [_t start];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
//    [self performSelector:@selector(doWork) onThread:_t withObject:nil waitUntilDone:YES]; waitUntilDone:YES 会崩溃
    
    NSLog(@"pre performSelector >>>> %@", _t);
    
    [self performSelector:@selector(doWork) onThread:_t withObject:nil waitUntilDone:NO];
    
    NSLog(@"after performSelector >>>> %@", _t);
}

- (void)doInitThread {
    //1.
    [_t setName:@"MyThread"];
    
    //2. CoreFoundation版本、开启当前线程的runloop
//    CFRunLoopRef runloop = CFRunLoopGetCurrent();
    CFRunLoopRun();
    
    //3. Foundation版本、开启当前线程的runloop
//    NSRunLoop *runloop = [NSRunLoop currentRunLoop];
//    [runloop run];
    
    NSLog(@"MyThread doInitThread >>>> %@", [NSThread currentThread]);
}

- (void)doWork {
    NSLog(@"MyThread doWork >>>> %@", [NSThread currentThread]);
}

@end
```

运行点击屏幕3次的输出

```
2016-11-27 15:59:34.955 RunLoopBasic[3416:48136] MyThread doInitThread >>>> <MyThread - 0x7fba98e0b410>: isExecuting = 1, isCancelled = 0, isFinished = 0


2016-11-27 15:59:37.305 RunLoopBasic[3416:48098] pre performSelector >>>> <MyThread - 0x7fba98e0b410>: isExecuting = 0, isCancelled = 0, isFinished = 1
2016-11-27 15:59:37.305 RunLoopBasic[3416:48098] after performSelector >>>> <MyThread - 0x7fba98e0b410>: isExecuting = 0, isCancelled = 0, isFinished = 1


2016-11-27 15:59:39.649 RunLoopBasic[3416:48098] pre performSelector >>>> <MyThread - 0x7fba98e0b410>: isExecuting = 0, isCancelled = 0, isFinished = 1
2016-11-27 15:59:39.650 RunLoopBasic[3416:48098] after performSelector >>>> <MyThread - 0x7fba98e0b410>: isExecuting = 0, isCancelled = 0, isFinished = 1


2016-11-27 15:59:41.678 RunLoopBasic[3416:48098] pre performSelector >>>> <MyThread - 0x7fba98e0b410>: isExecuting = 0, isCancelled = 0, isFinished = 1
2016-11-27 15:59:41.678 RunLoopBasic[3416:48098] after performSelector >>>> <MyThread - 0x7fba98e0b410>: isExecuting = 0, isCancelled = 0, isFinished = 1
```

现在可以让线程后续继续接受分配的任务了，但是发现`thread.isFinished = 1`。

这就很奇怪了，按照常理来说，一个东西都结束运行之后，就不能够再继续工作了。怎么还能可以继续完成任务了？？？？

总之可以看到开启了thread的runloop之后，线程的确就可以继续接受任务了。

###对比AFURLConnectionOperation创建单例NSThread的代码，发现除了开启runloop之外，还给runloop添加了一个NSMachPort事件源

尝试也给MyThread添加一个NSMachPort事件源

```objc
@implementation ViewController{
    MyThread *_t;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    _t = [[MyThread alloc] initWithTarget:self selector:@selector(doInitThread) object:nil];
    [_t start];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
//    [self performSelector:@selector(doWork) onThread:_t withObject:nil waitUntilDone:YES]; waitUntilDone:YES 会崩溃
    
    NSLog(@"pre performSelector >>>> %@", _t);
    
    [self performSelector:@selector(doWork) onThread:_t withObject:nil waitUntilDone:NO];
    
    NSLog(@"after performSelector >>>> %@", _t);
}

- (void)doInitThread {
    //1.
    [_t setName:@"MyThread"];
    
    //2. CoreFoundation版本、开启当前线程的runloop
//    {
//        CFRunLoopRef runloop = CFRunLoopGetCurrent();
//        CFMachPortContext ctx = {0, NULL, NULL, NULL, NULL};
//        CFMachPortRef mach_port = CFMachPortCreate(kCFAllocatorDefault, NULL, &ctx, NULL);
//        CFRunLoopSourceRef port_source = CFMachPortCreateRunLoopSource(kCFAllocatorDefault, mach_port, 0);
//        CFRunLoopAddSource(runloop, port_source, kCFRunLoopDefaultMode);
//        CFRunLoopRun();
//    }
    
    //3. Foundation版本、开启当前线程的runloop
    {
        NSRunLoop *runloop = [NSRunLoop currentRunLoop];
        [runloop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runloop run];
    }
    
    //4. 
    NSLog(@"MyThread doInitThread >>>> %@", [NSThread currentThread]);
}

- (void)doWork {
    NSLog(@"MyThread doWork >>>> %@", [NSThread currentThread]);
}

@end
```

程序运行后点击屏幕3次的输出

```
2016-11-27 16:22:50.699 RunLoopBasic[4685:69270] pre performSelector >>>> <MyThread - 0x7fae2b623390>: isExecuting = 1, isCancelled = 0, isFinished = 0
2016-11-27 16:22:50.700 RunLoopBasic[4685:69270] after performSelector >>>> <MyThread - 0x7fae2b623390>: isExecuting = 1, isCancelled = 0, isFinished = 0
2016-11-27 16:22:50.700 RunLoopBasic[4685:69314] MyThread doWork >>>> <MyThread - 0x7fae2b623390>: isExecuting = 1, isCancelled = 0, isFinished = 0


2016-11-27 16:22:59.532 RunLoopBasic[4685:69270] pre performSelector >>>> <MyThread - 0x7fae2b623390>: isExecuting = 1, isCancelled = 0, isFinished = 0
2016-11-27 16:22:59.532 RunLoopBasic[4685:69270] after performSelector >>>> <MyThread - 0x7fae2b623390>: isExecuting = 1, isCancelled = 0, isFinished = 0
2016-11-27 16:22:59.533 RunLoopBasic[4685:69314] MyThread doWork >>>> <MyThread - 0x7fae2b623390>: isExecuting = 1, isCancelled = 0, isFinished = 0


2016-11-27 16:23:01.344 RunLoopBasic[4685:69270] pre performSelector >>>> <MyThread - 0x7fae2b623390>: isExecuting = 1, isCancelled = 0, isFinished = 0
2016-11-27 16:23:01.345 RunLoopBasic[4685:69270] after performSelector >>>> <MyThread - 0x7fae2b623390>: isExecuting = 1, isCancelled = 0, isFinished = 0
2016-11-27 16:23:01.345 RunLoopBasic[4685:69314] MyThread doWork >>>> <MyThread - 0x7fae2b623390>: isExecuting = 1, isCancelled = 0, isFinished = 0
```

> 第一个发现、给runloop添加一个NSMachPort事件源之后，thread的状态和之前不一样了

```c
- (1) thread.isExecuting = 1
- (2) thread.isFinished = 0
```

> 第二个发现，doInitThread实现中的第4句NSLog打印的代码没有被执行


###上面说明给runloop添加一个事件源source之后，那么线程就一直处于`isExecuting = 1`的状态，而且`[runloop run];`之后的代码都不会被执行，也就是说thread不会结束执行

测试下给runloop添加一个事件源source之后，`[runloop run];`之后的代码都不会被执行

```objc
//
//  ViewController.m
//  RunLoopBasic
//
//  Created by xiongzenghui on 16/11/24.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import "ViewController.h"
#import "AFNetworking.h"
#import "MyThread.h"

@interface ViewController ()
@end

@implementation ViewController{
    MyThread *_t;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    _t = [[MyThread alloc] initWithTarget:self selector:@selector(doInitThread) object:nil];
    [_t start];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
//    [self performSelector:@selector(doWork) onThread:_t withObject:nil waitUntilDone:YES]; waitUntilDone:YES 会崩溃
    
    NSLog(@"pre performSelector >>>> %@", _t);
    
    [self performSelector:@selector(doWork) onThread:_t withObject:nil waitUntilDone:NO];
    
    NSLog(@"after performSelector >>>> %@", _t);
}

- (void)doInitThread {
    //1.
    [_t setName:@"MyThread"];
    
    //2. CoreFoundation版本、开启当前线程的runloop
//    {
//        CFRunLoopRef runloop = CFRunLoopGetCurrent();
//        CFMachPortContext ctx = {0, NULL, NULL, NULL, NULL};
//        CFMachPortRef mach_port = CFMachPortCreate(kCFAllocatorDefault, NULL, &ctx, NULL);
//        CFRunLoopSourceRef port_source = CFMachPortCreateRunLoopSource(kCFAllocatorDefault, mach_port, 0);
//        CFRunLoopAddSource(runloop, port_source, kCFRunLoopDefaultMode);
//        CFRunLoopRun();
//    }
    
    //3. Foundation版本、开启当前线程的runloop
    {
        //3.1
        NSRunLoop *runloop = [NSRunLoop currentRunLoop];
        
        //3.2
        NSLog(@"doInitThread >>>> 添加port事件之前");
        
        //3.3 【重要】 注释下面这句添加事件源的代码
//        [runloop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        
        //3.4
        NSLog(@"doInitThread >>>> 添加port事件之后, runloop还未开启");
        
        //3.5
        [runloop run];
        
        //3.6   
        NSLog(@"doInitThread >>>> runlop开始执行");
    }
    
    //4. 
    NSLog(@"MyThread doInitThread >>>> %@", [NSThread currentThread]);
}

- (void)doWork {
    NSLog(@"MyThread doWork >>>> %@", [NSThread currentThread]);
}

@end
```

程序运行后以及点击屏幕2次后的输出

```
2016-11-27 16:34:15.102 RunLoopBasic[5320:79720] doInitThread >>>> 添加port事件之前
2016-11-27 16:34:15.102 RunLoopBasic[5320:79720] doInitThread >>>> 添加port事件之后, runloop还未开启
2016-11-27 16:34:15.103 RunLoopBasic[5320:79720] doInitThread >>>> runlop开始执行
2016-11-27 16:34:15.103 RunLoopBasic[5320:79720] MyThread doInitThread >>>> <MyThread - 0x7faf6b513400>: isExecuting = 1, isCancelled = 0, isFinished = 0


2016-11-27 16:34:25.511 RunLoopBasic[5320:79676] pre performSelector >>>> <MyThread - 0x7faf6b513400>: isExecuting = 0, isCancelled = 0, isFinished = 1
2016-11-27 16:34:25.512 RunLoopBasic[5320:79676] after performSelector >>>> <MyThread - 0x7faf6b513400>: isExecuting = 0, isCancelled = 0, isFinished = 1


2016-11-27 16:34:27.364 RunLoopBasic[5320:79676] pre performSelector >>>> <MyThread - 0x7faf6b513400>: isExecuting = 0, isCancelled = 0, isFinished = 1
2016-11-27 16:34:27.364 RunLoopBasic[5320:79676] after performSelector >>>> <MyThread - 0x7faf6b513400>: isExecuting = 0, isCancelled = 0, isFinished = 1
```

可以看到，只要不给runloop添加port事件源，直接开启runloop，那么就会:

- (1) `[runloop run]`后面的代码继续执行
- (2) thread进入init的时候，`thread.isExecuting = 1`
- (3) thread执行完init的时候，`thread.isFinished = 1`
- (4) 后续分配任务给thread时候，`thread.isFinished = 1`

OK，那将3.3句代码打开后，程序运行后以及点击屏幕2次后的输出

```
2016-11-27 16:36:49.665 RunLoopBasic[5455:81894] doInitThread >>>> 添加port事件之前
2016-11-27 16:36:49.666 RunLoopBasic[5455:81894] doInitThread >>>> 添加port事件之后, runloop还未开启

2016-11-27 16:36:52.801 RunLoopBasic[5455:81853] pre performSelector >>>> <MyThread - 0x7f9de3d18bc0>: isExecuting = 1, isCancelled = 0, isFinished = 0
2016-11-27 16:36:52.801 RunLoopBasic[5455:81853] after performSelector >>>> <MyThread - 0x7f9de3d18bc0>: isExecuting = 1, isCancelled = 0, isFinished = 0
2016-11-27 16:36:52.801 RunLoopBasic[5455:81894] MyThread doWork >>>> <MyThread - 0x7f9de3d18bc0>: isExecuting = 1, isCancelled = 0, isFinished = 0


2016-11-27 16:36:54.582 RunLoopBasic[5455:81853] pre performSelector >>>> <MyThread - 0x7f9de3d18bc0>: isExecuting = 1, isCancelled = 0, isFinished = 0
2016-11-27 16:36:54.583 RunLoopBasic[5455:81853] after performSelector >>>> <MyThread - 0x7f9de3d18bc0>: isExecuting = 1, isCancelled = 0, isFinished = 0
2016-11-27 16:36:54.583 RunLoopBasic[5455:81894] MyThread doWork >>>> <MyThread - 0x7f9de3d18bc0>: isExecuting = 1, isCancelled = 0, isFinished = 0
```


可以看到，只要先给runloop添加port事件源之后，再开启runloop，那么就会:

- (1) `[runloop run]`后面的代码`不会`再执行 
- (2) thread进入init的时候，`thread.isExecuting = 1`
- (3) thread执行完init的时候，`thread.isExecuting = 1`
- (4) 后续分配任务给thread时候，`thread.isExecuting = 1`

线程的状态，始终都是执行中。

##为何给runloop添加了source监听之后，runloop就会一直卡着代码执行，让线程的状态一直处于executing了？


去CFRunLoop版本开源代码中查看下开启runloop执行的代码

```c
SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    CHECK_FOR_FORK();
    
    //1. 
    if (__CFRunLoopIsDeallocating(rl)) return kCFRunLoopRunFinished;
    __CFRunLoopLock(rl);
    
    //2. 根据传入的CFStringRef mode 字符串，找到CFRunLoopModeRef实例
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, modeName, false);
    
    //3.【重要】 如果mode下不存在 sources/timers/observers
    if (NULL == currentMode || __CFRunLoopModeIsEmpty(rl, currentMode, rl->_currentMode)) {
		Boolean did = false;
		if (currentMode) __CFRunLoopModeUnlock(currentMode);
		__CFRunLoopUnlock(rl);
		
		//【重要】 直接结束函数执行了
		return did ? kCFRunLoopRunHandledSource : kCFRunLoopRunFinished;
    }
    
    //4. 切换runloop为传入的mode
    volatile _per_run_data *previousPerRun = __CFRunLoopPushPerRunData(rl);
    CFRunLoopModeRef previousMode = rl->_currentMode;
    rl->_currentMode = currentMode;
    int32_t result = kCFRunLoopRunFinished;

	//5. 告诉所有的observer，runloop开始执行了
	if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
	
	//6. 继续调用私有函数真正启动runloop >>> 开启一个do-while(){}循环
	result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
	
	//7. 告诉所有的observer，runloop即将执行结束
	if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
	
    return result;
}
```

```c
typedef CF_ENUM(SInt32, CFRunLoopRunResult) {
    kCFRunLoopRunFinished = 1,				// 所有的Sources都被移除
    kCFRunLoopRunStopped = 2,				// 手动调用CFRunLoopStop() 
    kCFRunLoopRunTimedOut = 3,				// 超时 
    kCFRunLoopRunHandledSource = 4			// 处理完事件就结束执行
};
```

从代码中可以看到，第`3`步如果为YES，那么就不会走到第`6`步调用内部私有函数正式的开启runloop。

> 在执行runloop的start之前，必须要预先给runloop添加一个或以上的source，才能够成功开启runloop。

所以其实之前如果注释掉`doInitThread`实现中的`3.3`句代码，其实根本不会开启thread的runloop。


ok，到此为止，对于如何正确的开启一个thread的runloop以及让一个线程长期存活以待不断的接收任务执行的使用就是如上了。

##一个thread与一个runloop之间的关系

苹果没有给出任何api直接创建一个RunLoop对象，只能通过如下系统函数获取得到一个创建完毕的runloop

```objc
[NSRunLoop currentRunLoop]、[NSRunLoop mainRunLoop]
```

```c
CFRunLoopGetCurrent()、CFRunLoopGetMain()
```

可以从苹果的CFRunLoop开源代码中，找到`CFRunLoopGetCurrent()、CFRunLoopGetMain()`这两个函数最终调用的内部私有函数的实现:


```c
// 全局缓存dic ===> <thread:runloop>
static CFMutableDictionaryRef __CFRunLoops = NULL;

// 同步锁
static CFLock_t loopsLock = CFLockInit;

// should only be called by Foundation
// t==0 is a synonym for "main thread" that always works
CF_EXPORT CFRunLoopRef _CFRunLoopGet0(pthread_t t) {

	// 1. 
    if (pthread_equal(t, kNilPthreadT)) {
		t = pthread_main_thread_np();
    }
    
    //2. 
    __CFLock(&loopsLock);
    
    //3. 
    if (!__CFRunLoops) {
        __CFUnlock(&loopsLock);

        // 全局缓存dic还不存在
		CFMutableDictionaryRef dict = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, &kCFTypeDictionaryValueCallBacks);

        // 创建主线程的runloop
        CFRunLoopRef mainLoop = __CFRunLoopCreate(pthread_main_thread_np());

        // 建立thread与runloop的映射关系，保存到dic
        CFDictionarySetValue(dict, pthreadPointer(pthread_main_thread_np()), mainLoop);

        // 把临时创建的dic实例，让 __CFRunLoops 指向
        if (!OSAtomicCompareAndSwapPtrBarrier(NULL, dict, (void * volatile *)&__CFRunLoops)) {
	       CFRelease(dict);//如果指向设置失败，则释放掉局部的dic实例
        }

        // runloop已经由 __CFRunLoops 指向的 字典实例所引用，此时retainCount=2，所以让retainCount-1恢复平衡
	    CFRelease(mainLoop);

        __CFLock(&loopsLock);
    }

    //3. 从全局缓存字典中使用 thread线程作为key 来查询是否存在对应的runloop
    CFRunLoopRef loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t))
    __CFUnlock(&loopsLock);

    //4. 如果thread不存在对应的runloop 
    if (!loop) {

        //4.1 给当前线程创建一个新的runloop
        CFRunLoopRef newLoop = __CFRunLoopCreate(t);
        __CFLock(&loopsLock);
	
        //4.3 如果不存在对应的缓存runloop，则将新创建的runloop缓存起来
        CFDictionarySetValue(__CFRunLoops, pthreadPointer(t), newLoop);
        loop = newLoop;

        // don't release run loops inside the loopsLock, because CFRunLoopDeallocate may end up taking it
        __CFUnlock(&loopsLock);
	    CFRelease(newLoop);
    }

    //5. 注册一个回调，当线程销毁时，顺便也销毁其对应的 RunLoop。
    if (pthread_equal(t, pthread_self())) {
        _CFSetTSD(__CFTSDKeyRunLoop, (void *)loop, NULL);
        if (0 == _CFGetTSD(__CFTSDKeyRunLoopCntr)) {
            _CFSetTSD(__CFTSDKeyRunLoopCntr, (void *)(PTHREAD_DESTRUCTOR_ITERATIONS-1), (void (*)(void *))__CFFinalizeRunLoop);
        }
    }

    return loop;
}
```

大体的结构相当于如下

```
- 全局缓存dic
	- item 1
		- key: thread 1
		- value : runloop 1
	- item 2
		- key: thread 2
		- value : runloop 2
	- item 3
		- key: thread 3
		- value : runloop 3
	- item 4
		- key: thread 4
		- value : runloop 4
```

从上面的`_CFRunLoopGet()`函数实现可以明白:

- (1) 线程Thread 和 运行循环RunLoop 完全是不同的东西，但是二者之间又是一一对应的。

- (2) 我们自己创建的线程对象在 `刚创建时` 并没有 RunLoop，必须要主动获取让这个线程对象调用RunLoop对象的方法`_CFRunLoopGet(线程对象)`之后，其RunLoop对象才会被创建

- (3) 只能处于在指定线程时，才能获取指定线程的 RunLoop对象。但是`主线程`的RunLoop对象 是可以在`任意线程`上进行获取。


runloop的结构的抽象定义:

```c
struct __CFRunLoop {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;			/* locked for accessing mode list */
    __CFPort _wakeUpPort;			// used for CFRunLoopWakeUp 
    Boolean _unused;
    volatile _per_run_data *_perRunData;              // reset for runs of the run loop
    
    // 【重要】RunLoop对应的线程对象
    pthread_t _pthread;
    
    uint32_t _winthread;
    CFMutableSetRef _commonModes;
    CFMutableSetRef _commonModeItems;
    CFRunLoopModeRef _currentMode;
    CFMutableSetRef _modes;
    
    struct _block_item *_blocks_head;
    struct _block_item *_blocks_tail;
    CFAbsoluteTime _runTime;
    CFAbsoluteTime _sleepTime;
    CFTypeRef _counterpart;
};
```

可以看到，`__CFRunLoop`实例有一个成员变量叫做`_pthread`，并且指向一个`struct _opaque_pthread_t *`这个结构体类型的实例。

> 所以`__CFRunLoop`实例一直持有一个thread实例，那么所以只要`__CFRunLoop`实例存在，那么thread实例就一直会存在。

那么所以，全局缓存dic、runloop、thread三者之间的串联关系:

```c
全局缓存dic ---> runloop ---> thread
```

下面证实下，只要thread开启runloop，那么thread就不会被废弃:

```objc
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 局部thread对象
    MyThread *t = [[MyThread alloc] initWithTarget:self selector:@selector(doInitThread) object:nil];
    [t start];
}

- (void)doInitThread {
    //1.
    NSThread *t = [NSThread currentThread];
    
    //2.
    [t setName:@"MyThread"];
    
    //3. Foundation版本、开启当前线程的runloop
    {
        //3.1
        NSRunLoop *runloop = [NSRunLoop currentRunLoop];
        
        //3.2
        NSLog(@"doInitThread >>>> 添加port事件之前");
        
        //3.3 【重要】 注释下面这句添加事件源的代码
        [runloop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        
        //3.4
        NSLog(@"doInitThread >>>> 添加port事件之后, runloop还未开启");
        
        //3.5
        [runloop run];
        
        //3.6   
        NSLog(@"doInitThread >>>> runlop开始执行");
    }
    
    //4. 
    NSLog(@"MyThread doInitThread >>>> %@", [NSThread currentThread]);
}

@end
```

程序运行后的输出

```
2016-11-27 17:08:26.499 RunLoopBasic[8294:108202] doInitThread >>>> 添加port事件之前
2016-11-27 17:08:26.500 RunLoopBasic[8294:108202] doInitThread >>>> 添加port事件之后, runloop还未开启
```

可以看到:

- (1) 局部创建的thread对象，并没有被废弃
- (2) 同样doInitThread实现内的流程会卡在3.5句代码`[runloop run];`地方，不会再往下执行


###那么不能简单的缓存NSThread对象，那么如果是简单的缓存`dispatch_queue_t`实例了？

```c
static dispatch_queue_t SingletonQueue() {
    static dispatch_queue_t queue = NULL;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        queue = dispatch_queue_create("myQueue", NULL);
    });
    return queue;
}
```

```objc
@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	dispatch_async(SingletonQueue(), ^{
        NSThread *t = [NSThread currentThread];
        NSLog(@"time = %@, thread.isExecuting = %d, thread.isFinished = %d", [NSDate date], t.isExecuting, t.isFinished);
    });
}

@end
```

多次运行后输出结果

```
2016-12-15 00:05:58.564 Demo[9816:96607] time = 2016-12-14 16:05:58 +0000, thread.isExecuting = 1, thread.isFinished = 0
2016-12-15 00:06:02.352 Demo[9816:96607] time = 2016-12-14 16:06:02 +0000, thread.isExecuting = 1, thread.isFinished = 0
2016-12-15 00:06:04.487 Demo[9816:96607] time = 2016-12-14 16:06:04 +0000, thread.isExecuting = 1, thread.isFinished = 0
2016-12-15 00:06:04.708 Demo[9816:96607] time = 2016-12-14 16:06:04 +0000, thread.isExecuting = 1, thread.isFinished = 0
2016-12-15 00:06:04.905 Demo[9816:96607] time = 2016-12-14 16:06:04 +0000, thread.isExecuting = 1, thread.isFinished = 0
2016-12-15 00:06:05.068 Demo[9816:96607] time = 2016-12-14 16:06:05 +0000, thread.isExecuting = 1, thread.isFinished = 0
2016-12-15 00:06:05.224 Demo[9816:96607] time = 2016-12-14 16:06:05 +0000, thread.isExecuting = 1, thread.isFinished = 0
2016-12-15 00:06:05.388 Demo[9816:96607] time = 2016-12-14 16:06:05 +0000, thread.isExecuting = 1, thread.isFinished = 0
2016-12-15 00:06:05.544 Demo[9816:96607] time = 2016-12-14 16:06:05 +0000, thread.isExecuting = 1, thread.isFinished = 0
2016-12-15 00:06:05.708 Demo[9816:96607] time = 2016-12-14 16:06:05 +0000, thread.isExecuting = 1, thread.isFinished = 0
2016-12-15 00:06:05.864 Demo[9816:96607] time = 2016-12-14 16:06:05 +0000, thread.isExecuting = 1, thread.isFinished = 0
2016-12-15 00:07:15.003 Demo[9816:98378] time = 2016-12-14 16:07:15 +0000, thread.isExecuting = 1, thread.isFinished = 0
```

可以看到最终代码执行所在的thread状态一直是`isExecuting == 1`，所以可以通过缓存`dispatch_queue_t实例`来代替之前缓存`NSThread对象`就简单的多了。

如果想要缓存`NSThread对象`，必须完成如下步骤:

- (1) 获取线程对应的runloop，也就是让系统完成创建runloop并于线程绑定
- (1) 给runloop添加`CFRunLoopSource事件源`，如果不添加那么runloop无法开启
- (2) 开启`runloop`

那么所以说，通过缓存`dispatch_queue_t实例`来完成线程池还是方便的多了。