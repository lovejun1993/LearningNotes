## 之前用过Bugly的线程卡顿监控上报，但是一直想搞清楚原理

学习来源: http://www.tanhao.me/code/151113.html/


### 大概思路

- (1) 在一个`子线程`上创建一个observer，注册监控`主线程runloop`状态改变
- (2) 然后根据状态改变之间的`时间差`，来判断是否卡顿

- (3) 线程卡顿分两种
	- 多次，小时间，卡顿
	- 单次，长时间，卡顿


### 先看个例子，看runloop状态改变的规律

runloop状态枚举定义

```
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
	// 1
    kCFRunLoopEntry = (1UL << 0),	
    
    // 2
    kCFRunLoopBeforeTimers = (1UL << 1),
    
    // 4
    kCFRunLoopBeforeSources = (1UL << 2),
    
    // 32
    kCFRunLoopBeforeWaiting = (1UL << 5),
    
    // 64
    kCFRunLoopAfterWaiting = (1UL << 6),
    
    // 128
    kCFRunLoopExit = (1UL << 7),
    
    // 
    kCFRunLoopAllActivities = 0x0FFFFFFFU
};
```	

ViewController+监视主线程runloop状态值改变

- AppDelegate中监视主线程runloop

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    [[PerformanceMonitor sharedInstance] start];
    
    return YES;
}
```

- 简单的ViewController代码

```
@implementation ViewController

- (void)viewDidLoad
{
    [super viewDidLoad];
    
}

@end
```

- 具体监控主线程runloop的代码

```
@interface PerformanceMonitor ()
{
    //记录监控主线程runloop的观察者
    CFRunLoopObserverRef observer;
    
    //子线程计算时间差值的同步信号
    dispatch_semaphore_t semaphore;
    
    //保存当前主线程runloop变化成的状态
    CFRunLoopActivity activity;
}
@end

@implementation PerformanceMonitor

+ (instancetype)sharedInstance
{
    static id instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

static void runLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info)
{
    //1. 单例对象保存当前主线程runloop的状态
    PerformanceMonitor *moniotr = (__bridge PerformanceMonitor*)info;
    NSLog(@"pre state = %ld", moniotr->activity);
    
    moniotr->activity = activity;
    NSLog(@"after state = %ld", moniotr->activity);
    NSLog(@"------------------------------");
    
    //产生一个信号量，通知子线程计算时间差，判断是否卡顿
    dispatch_semaphore_signal(moniotr->semaphore);
}

- (void)start
{
    //1. 只产生一个观察者
    if (observer)
        return;
    
    //2. 初始信号为0，只有当runloop状态改变回调时才执行
    semaphore = dispatch_semaphore_create(0);
    
    //3. 设置Runloop Observer的运行环境
    CFRunLoopObserverContext context = {
        0,
        (__bridge void*)self,
        NULL,
        NULL,
        NULL
    };
    
    //4. 创建一个runloop观察者
    //第一个参数用于分配observer对象的内存
    //第二个参数用以设置observer所要关注的事件，详见回调函数myRunLoopObserver中注释
    //第三个参数用于标识该observer是在第一次(NO)进入run loop时执行还是每次(YES)进入run loop处理时均执行
    //第四个参数用于设置该observer的优先级
    //第五个参数用于设置该observer的回调函数
    //第六个参数用于设置该observer的运行环境
    observer = CFRunLoopObserverCreate(kCFAllocatorDefault,
                                       kCFRunLoopAllActivities,
                                       YES,
                                       0,
                                       &runLoopObserverCallBack,
                                       &context);
    
    //5. 向主线程runloop添加观察者
    if (observer) {
        //获取主线程runloop
        CFRunLoopRef mainLoop = CFRunLoopGetMain();
        
        //向runloop添加一个观察者，并制定runloop mode
        CFRunLoopAddObserver(mainLoop,
                             observer,
                             kCFRunLoopCommonModes
                             );
    }
}

@end
```

App程序启动后输出如下

```
2016-03-24 15:17:15.422
2016-03-24 15:17:15.423 PerformanceMonitor[32671:216300] pre state = 1
2016-03-24 15:17:15.423 PerformanceMonitor[32671:216300] after state = 2
2016-03-24 15:17:15.423 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.423 PerformanceMonitor[32671:216300] pre state = 2
2016-03-24 15:17:15.424 PerformanceMonitor[32671:216300] after state = 4
2016-03-24 15:17:15.424 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.425 PerformanceMonitor[32671:216300] pre state = 4
2016-03-24 15:17:15.426 PerformanceMonitor[32671:216300] after state = 2
2016-03-24 15:17:15.426 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.427 PerformanceMonitor[32671:216300] pre state = 2
2016-03-24 15:17:15.427 PerformanceMonitor[32671:216300] after state = 4
2016-03-24 15:17:15.427 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.438 PerformanceMonitor[32671:216300] pre state = 4
2016-03-24 15:17:15.439 PerformanceMonitor[32671:216300] after state = 2
2016-03-24 15:17:15.439 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.439 PerformanceMonitor[32671:216300] pre state = 2
2016-03-24 15:17:15.439 PerformanceMonitor[32671:216300] after state = 4
2016-03-24 15:17:15.440 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.440 PerformanceMonitor[32671:216300] pre state = 4
2016-03-24 15:17:15.440 PerformanceMonitor[32671:216300] after state = 2
2016-03-24 15:17:15.441 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.441 PerformanceMonitor[32671:216300] pre state = 2
2016-03-24 15:17:15.442 PerformanceMonitor[32671:216300] after state = 4
2016-03-24 15:17:15.442 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.442 PerformanceMonitor[32671:216300] pre state = 4
2016-03-24 15:17:15.442 PerformanceMonitor[32671:216300] after state = 2
2016-03-24 15:17:15.442 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.442 PerformanceMonitor[32671:216300] pre state = 2
2016-03-24 15:17:15.442 PerformanceMonitor[32671:216300] after state = 4
2016-03-24 15:17:15.444 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.444 PerformanceMonitor[32671:216300] pre state = 4
2016-03-24 15:17:15.445 PerformanceMonitor[32671:216300] after state = 2
2016-03-24 15:17:15.445 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.445 PerformanceMonitor[32671:216300] pre state = 2
2016-03-24 15:17:15.445 PerformanceMonitor[32671:216300] after state = 4
2016-03-24 15:17:15.445 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.445 PerformanceMonitor[32671:216300] pre state = 4
2016-03-24 15:17:15.446 PerformanceMonitor[32671:216300] after state = 32
2016-03-24 15:17:15.446 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.789 PerformanceMonitor[32671:216300] pre state = 32
2016-03-24 15:17:15.789 PerformanceMonitor[32671:216300] after state = 64
2016-03-24 15:17:15.789 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.790 PerformanceMonitor[32671:216300] pre state = 64
2016-03-24 15:17:15.790 PerformanceMonitor[32671:216300] after state = 2
2016-03-24 15:17:15.790 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.790 PerformanceMonitor[32671:216300] pre state = 2
2016-03-24 15:17:15.790 PerformanceMonitor[32671:216300] after state = 4
2016-03-24 15:17:15.790 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.790 PerformanceMonitor[32671:216300] pre state = 4
2016-03-24 15:17:15.790 PerformanceMonitor[32671:216300] after state = 2
2016-03-24 15:17:15.790 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.790 PerformanceMonitor[32671:216300] pre state = 2
2016-03-24 15:17:15.791 PerformanceMonitor[32671:216300] after state = 4
2016-03-24 15:17:15.791 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.791 PerformanceMonitor[32671:216300] pre state = 4
2016-03-24 15:17:15.792 PerformanceMonitor[32671:216300] after state = 2
2016-03-24 15:17:15.792 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.793 PerformanceMonitor[32671:216300] pre state = 2
2016-03-24 15:17:15.793 PerformanceMonitor[32671:216300] after state = 4
2016-03-24 15:17:15.793 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.793 PerformanceMonitor[32671:216300] pre state = 4
2016-03-24 15:17:15.793 PerformanceMonitor[32671:216300] after state = 2
2016-03-24 15:17:15.793 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.794 PerformanceMonitor[32671:216300] pre state = 2
2016-03-24 15:17:15.794 PerformanceMonitor[32671:216300] after state = 4
2016-03-24 15:17:15.794 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.794 PerformanceMonitor[32671:216300] pre state = 4
2016-03-24 15:17:15.794 PerformanceMonitor[32671:216300] after state = 2
2016-03-24 15:17:15.794 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.795 PerformanceMonitor[32671:216300] pre state = 2
2016-03-24 15:17:15.795 PerformanceMonitor[32671:216300] after state = 4
2016-03-24 15:17:15.795 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:15.795 PerformanceMonitor[32671:216300] pre state = 4
2016-03-24 15:17:15.795 PerformanceMonitor[32671:216300] after state = 32
2016-03-24 15:17:15.795 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:16.815 PerformanceMonitor[32671:216300] pre state = 32
2016-03-24 15:17:16.816 PerformanceMonitor[32671:216300] after state = 64
2016-03-24 15:17:16.816 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:16.817 PerformanceMonitor[32671:216300] pre state = 64
2016-03-24 15:17:16.817 PerformanceMonitor[32671:216300] after state = 2
2016-03-24 15:17:16.817 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:16.817 PerformanceMonitor[32671:216300] pre state = 2
2016-03-24 15:17:16.818 PerformanceMonitor[32671:216300] after state = 4
2016-03-24 15:17:16.818 PerformanceMonitor[32671:216300] ------------------------------
2016-03-24 15:17:16.818 PerformanceMonitor[32671:216300] pre state = 4
2016-03-24 15:17:16.818 PerformanceMonitor[32671:216300] after state = 32
2016-03-24 15:17:16.818 PerformanceMonitor[32671:216300] ------------------------------
```

随便点击下屏幕，如下几种状态切换:

```
2016-03-24 15:19:54.500 PerformanceMonitor[32805:219499] pre state = 32
2016-03-24 15:19:54.501 PerformanceMonitor[32805:219499] after state = 64
2016-03-24 15:19:54.501 PerformanceMonitor[32805:219499] ------------------------------
2016-03-24 15:19:54.501 PerformanceMonitor[32805:219499] pre state = 64
2016-03-24 15:19:54.501 PerformanceMonitor[32805:219499] after state = 2
2016-03-24 15:19:54.501 PerformanceMonitor[32805:219499] ------------------------------
2016-03-24 15:19:54.501 PerformanceMonitor[32805:219499] pre state = 2
2016-03-24 15:19:54.501 PerformanceMonitor[32805:219499] after state = 4
2016-03-24 15:19:54.502 PerformanceMonitor[32805:219499] ------------------------------
2016-03-24 15:19:54.503 PerformanceMonitor[32805:219499] pre state = 4
2016-03-24 15:19:54.503 PerformanceMonitor[32805:219499] after state = 2
2016-03-24 15:19:54.504 PerformanceMonitor[32805:219499] ------------------------------
2016-03-24 15:19:54.504 PerformanceMonitor[32805:219499] pre state = 2
2016-03-24 15:19:54.504 PerformanceMonitor[32805:219499] after state = 4
2016-03-24 15:19:54.504 PerformanceMonitor[32805:219499] ------------------------------
2016-03-24 15:19:54.504 PerformanceMonitor[32805:219499] pre state = 4
2016-03-24 15:19:54.504 PerformanceMonitor[32805:219499] after state = 2
2016-03-24 15:19:54.504 PerformanceMonitor[32805:219499] ------------------------------
2016-03-24 15:19:54.505 PerformanceMonitor[32805:219499] pre state = 2
2016-03-24 15:19:54.505 PerformanceMonitor[32805:219499] after state = 4
2016-03-24 15:19:54.505 PerformanceMonitor[32805:219499] ------------------------------
2016-03-24 15:19:54.505 PerformanceMonitor[32805:219499] pre state = 4
2016-03-24 15:19:54.505 PerformanceMonitor[32805:219499] after state = 32
2016-03-24 15:19:54.505 PerformanceMonitor[32805:219499] ------------------------------
2016-03-24 15:19:54.596 PerformanceMonitor[32805:219499] pre state = 32
2016-03-24 15:19:54.597 PerformanceMonitor[32805:219499] after state = 64
2016-03-24 15:19:54.597 PerformanceMonitor[32805:219499] ------------------------------
2016-03-24 15:19:54.597 PerformanceMonitor[32805:219499] pre state = 64
2016-03-24 15:19:54.598 PerformanceMonitor[32805:219499] after state = 2
2016-03-24 15:19:54.598 PerformanceMonitor[32805:219499] ------------------------------
2016-03-24 15:19:54.598 PerformanceMonitor[32805:219499] pre state = 2
2016-03-24 15:19:54.598 PerformanceMonitor[32805:219499] after state = 4
2016-03-24 15:19:54.598 PerformanceMonitor[32805:219499] ------------------------------
2016-03-24 15:19:54.599 PerformanceMonitor[32805:219499] pre state = 4
2016-03-24 15:19:54.599 PerformanceMonitor[32805:219499] after state = 2
2016-03-24 15:19:54.599 PerformanceMonitor[32805:219499] ------------------------------
2016-03-24 15:19:54.599 PerformanceMonitor[32805:219499] pre state = 2
2016-03-24 15:19:54.599 PerformanceMonitor[32805:219499] after state = 4
2016-03-24 15:19:54.600 PerformanceMonitor[32805:219499] ------------------------------
2016-03-24 15:19:54.600 PerformanceMonitor[32805:219499] pre state = 4
2016-03-24 15:19:54.600 PerformanceMonitor[32805:219499] after state = 32
2016-03-24 15:19:54.600 PerformanceMonitor[32805:219499] ------------------------------
2016-03-24 15:19:55.254 PerformanceMonitor[32805:219499] pre state = 32
2016-03-24 15:19:55.254 PerformanceMonitor[32805:219499] after state = 64
2016-03-24 15:19:55.254 PerformanceMonitor[32805:219499] ------------------------------
2016-03-24 15:19:55.254 PerformanceMonitor[32805:219499] pre state = 64
2016-03-24 15:19:55.254 PerformanceMonitor[32805:219499] after state = 2
2016-03-24 15:19:55.254 PerformanceMonitor[32805:219499] ------------------------------
2016-03-24 15:19:55.254 PerformanceMonitor[32805:219499] pre state = 2
2016-03-24 15:19:55.254 PerformanceMonitor[32805:219499] after state = 4
2016-03-24 15:19:55.254 PerformanceMonitor[32805:219499] ------------------------------
2016-03-24 15:19:55.254 PerformanceMonitor[32805:219499] pre state = 4
2016-03-24 15:19:55.255 PerformanceMonitor[32805:219499] after state = 32
2016-03-24 15:19:55.255 PerformanceMonitor[32805:219499] ------------------------------
```


主要会出现卡顿情况的切换状态是如下两种:

```
kCFRunLoopBeforeWaiting >>> kCFRunLoopAfterWaiting
```

```
kCFRunLoopAfterWaiting >>> kCFRunLoopBeforeSources
```


那下面就是主要监视主线程runloop状态是如上两种状态切换时，查看其时间差是否超过一定数值，超过数值即视为卡顿.

## 编写一个单例类来封装监控主线程runloop

### AppDelegate启动回调函数开始监控

```objc
@implementation AppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    //启动主线程的runloop监控
    [[PerformanceMonitor sharedInstance] start];
    
    return YES;
}
```

###具体监控主线程runloop的单例类

声明辅助变量

```objc
@interface PerformanceMonitor ()
{
    //
    int timeoutCount;
    
    //记录监控主线程runloop的观察者
    CFRunLoopObserverRef observer;
    
    //子线程计算时间差值的同步信号
    dispatch_semaphore_t semaphore;
    
    //保存当前主线程runloop变化成的状态
    CFRunLoopActivity activity;
}
@end
```

单例初始化

```
@implementation PerformanceMonitor

+ (instancetype)sharedInstance
{
    static id instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

@end
```

开始检测主线程runloop

```
- (void)start
{
    //1. 只产生一个观察者
    if (observer)
        return;
    
    //2. 初始信号为0，只有当runloop状态改变回调时才执行
    semaphore = dispatch_semaphore_create(0);
    
    //3. 设置Runloop Observer的运行环境
    CFRunLoopObserverContext context = {
        0,
        (__bridge void*)self,
        NULL,
        NULL,
        NULL
    };
    
    //4. 创建一个runloop观察者
    //第一个参数用于分配observer对象的内存
    //第二个参数用以设置observer所要关注的事件，详见回调函数myRunLoopObserver中注释
    //第三个参数用于标识该observer是在第一次(NO)进入run loop时执行还是每次(YES)进入run loop处理时均执行
    //第四个参数用于设置该observer的优先级
    //第五个参数用于设置该observer的回调函数
    //第六个参数用于设置该observer的运行环境
    observer = CFRunLoopObserverCreate(kCFAllocatorDefault,
                                       kCFRunLoopAllActivities,
                                       YES,
                                       0,
                                       &runLoopObserverCallBack,
                                       &context);
    
    //5. 向主线程runloop添加观察者
    if (observer) {
        //获取主线程runloop
        CFRunLoopRef mainLoop = CFRunLoopGetMain();
        
        //向runloop添加一个观察者，并制定runloop mode
        CFRunLoopAddObserver(mainLoop,
                             observer,
                             kCFRunLoopCommonModes
                             );
    }
    
    //6. 新开一个子线程来不断的计算时间差
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        while (YES)
        {
            //设定等待超时时间为50ms
//            long flag = dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
            long flag = dispatch_semaphore_wait(semaphore, dispatch_time(DISPATCH_TIME_NOW, 50*NSEC_PER_MSEC));
            
            //flag为非0表示等待超过50ms
            if (flag != 0)
            {
                if (activity == kCFRunLoopBeforeSources || activity == kCFRunLoopAfterWaiting)
                {
                    //连续超时次数小于5，不处理为卡顿
                    if (++timeoutCount < 5)
                        continue;
                    
                    //连续超时次数大于5，就处理为卡顿
                    [self doReport];
                }
            }
            
            //连续超过5次超时后，清零
            timeoutCount = 0;
        }
    });
}

- (void)doReport {
    //1. 得到当前造成卡顿的函数调用信息
    //2. 上报给服务器
    NSLog(@"有点卡顿");
}
```

- 对于连续的短卡顿 和 一次长时间卡顿 的处理
	- 对连续5次超过50ms的小卡顿处理
	- 对一次长时间的卡顿250ms做出卡顿处理
		- 将250ms分成5次等待，每次50ms


检测到主线程runloop状态改变的callback函数

```c
static void runLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info)
{
    //1. 先记录当前变成的 主线程runloop的状态值
    PerformanceMonitor *moniotr = (__bridge PerformanceMonitor*)info;
    moniotr->activity = activity;
    
    //2. 再发送信号量，通知子线程进行状态切换过程的时间差值
    dispatch_semaphore_signal(moniotr->semaphore);
}
```


停止监控

```objc
- (void)stop
{
    if (!observer)
        return;
    
    //移除runloop观察者
    CFRunLoopRemoveObserver(CFRunLoopGetMain(), observer, kCFRunLoopCommonModes);
    CFRelease(observer);
    observer = NULL;
}
```

