
## 关于NSNotificationCenter的小东西

主要的提纲

```
1. NSNotificationCenter不会对`addObserver:对象`进行retain
2. NSNotificationCenter直接使用的是observer对象的`地址`
3. A线程关注通知，B线程发送通知，则通知接收处理就是在B线程
4. 通过 `NSMachPort`线程之间的通信，解决 3. 的问题，强制让通知接收处理回到A线程
5. -[NSNotificationCenter post....] 同步发送通知，同步执行通知处理
6. 使用`NSNotificationQueue`可以完成异步发送通知，异步通知处理
```

## 1. NSNotificationCenter不会对`addObserver:对象`进行retain

```objc
@interface MyView : UIView
@end
@implementation MyView
- (id)init
{
    if (self = [super init]) {
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(test) name:@"test" object:nil];
    }
    return self;
}

- (void)test
{
    NSLog(@"=================");
}

- (void)dealloc{
    NSLog(@"MyView delloc ===> %p", self);
}

@end

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	MyView *view = [[MyView alloc] init];
}

@end
```

运行后点击两次，可以看到有两次dealloc打印如下:

```
2014-11-24 14:13:08.690 Demo[901:325600] MyView delloc ===> 0x17019fe40
2014-11-24 14:13:10.141 Demo[901:325600] MyView delloc ===> 0x17019f960
```

确实是不会retain传入的obsever对象。

## 2. NSNotificationCenter直接使用的是observer对象的地址

> 所以对于observer对象在dealloc时，一定要removeObserver，否则会出现野指针。

```objc
@interface MyView : UIView
@end
@implementation MyView
- (id)init
{
    if (self = [super init]) {
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(test) name:@"test" object:nil];
    }
    return self;
}

- (void)test
{
    NSLog(@"=================");
}

- (void)dealloc{
    NSLog(@"MyView delloc ===> %p", self);
}

@end

@implementation ViewController{
    MyView *_view;
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
   
    //1. 先让对象废弃
    _view = [[MyView alloc] init];
    _view = nil;
    
    //2. 再发送通知
    [[NSNotificationCenter defaultCenter] postNotificationName:@"test" object:nil];
}

@end
```

如上代码在iOS9之前的系统，都会`BAD_ACESS`崩溃。但是在iOS9以及之后的系统，都不会有问题。所以，应该是iOS9修复了这个问题吧。


## 3. A线程发送通知，B线程关注通知

- 主线程添加observer，子线程发送notification

```objc
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
 	
 	// 主线程添加observer
 	[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(_handle) name:key object:nil];  
} 

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

	// 子线程发送notification
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:key object:nil];
    });
}

- (void)_handle {
    NSLog(@"xxxxxxxxxxxxxxxxxx thread = %@", [NSThread currentThread]);
}

@end
```

输出结果

```
2014-11-24 12:58:20.133 Demo[819:308826] xxxxxxxxxxxxxxxxxx thread = <NSThread: 0x1740768c0>{number = 2, name = (null)}
```

可以看到接收通知处理是处于`子线程`上完成的，那么在dealloc时就需要removeObserver。

- 子线程添加observer，主线程发送notification

```objc
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 子线程添加observer
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
    	[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(_handle) name:key object:nil];
    });
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

	// 主线程发送notification
	[[NSNotificationCenter defaultCenter] postNotificationName:key object:nil];
}

- (void)_handle {
    NSLog(@"xxxxxxxxxxxxxxxxxx thread = %@", [NSThread currentThread]);
}

@end
```

运行结果

```
2014-11-24 13:06:19.048 Demo[831:311011] xxxxxxxxxxxxxxxxxx thread = <NSThread: 0x170070840>{number = 1, name = main}
```

可以看到接收通知处理是处于`主线程`，那么在dealloc时也需要removeObserver。

以前我一直认为在A线程发送通知，那么只能在A线程注册通知.....囧。

## 可以得到如下几点:

- (1) NSNotificationCenter是`多线程安全`的
- (2) 与`哪个线程进行addObserver`没啥关系（只是操作缓存dic对象而已）
- (3) 与`在哪个线程进行post通知`有关系 >>> 在A线程上进行post通知，就会在A线程上进行observer回调

所以，在哪一个线程上post一个通知，就会在哪一个线程上进行notification的转发，继而转发给observer进行处理。

所以，在`子线程`上post通知，从而回调中操作UI对象，就会造成程序崩溃。

## 4. 如果想要 `发送通知` 和 `接收通知处理` 这两个步骤强制处于一个线程上了？

```objc
static NSString *key = @"haha";

@interface ViewController () <NSMachPortDelegate>

@property (nonatomic) NSMutableArray    *notifications;         // 通知队列
@property (nonatomic) NSThread          *notificationThread;    // 期望线程
@property (nonatomic) NSLock            *notificationLock;      // 用于对通知队列加锁的锁对象，避免线程冲突
@property (nonatomic) NSMachPort        *notificationPort;      // 用于向期望线程发送信号的通信端口

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 初始化
    self.notifications = [[NSMutableArray alloc] init];
    self.notificationLock = [[NSLock alloc] init];
    
    // 设置接收通知处理的线程
    self.notificationThread = [NSThread currentThread];
    
    // 给当前【主线程runloop】注册一个监听的事件端口
    self.notificationPort = [[NSMachPort alloc] init];
    self.notificationPort.delegate = self;
    [[NSRunLoop currentRunLoop] addPort:self.notificationPort forMode:(__bridge NSString *)kCFRunLoopCommonModes];
    
    // 关注通知
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(processNotification:) name:key object:nil];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

	// 子线程post通知
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:key object:nil];
    });
}

#pragma mark - NSMachPortDelegate 

- (void)handleMachMessage:(void *)msg {
    [self.notificationLock lock];
    
    // 将数组中保存的notifications，重定向到指定线程上进行处理
    while ([self.notifications count]) {
        NSNotification *notification = [self.notifications objectAtIndex:0];
        [self.notifications removeObjectAtIndex:0];
        [self.notificationLock unlock];
        [self processNotification:notification];
        [self.notificationLock lock];
    };
    
    [self.notificationLock unlock];
}

- (void)processNotification:(NSNotification *)notification {
    
    if ([NSThread currentThread] != _notificationThread) {
		
		// 将不符合当前线程处理的通知，添加到重定向的数组中临时保存
        [self.notificationLock lock];
        [self.notifications addObject:notification];
        [self.notificationLock unlock];
        
        // 发送一个port事件，通知其对应线程，即执行NSMachPortDelegate协议方法
        [self.notificationPort sendBeforeDate:[NSDate date]
                                   components:nil
                                         from:nil
                                     reserved:0];
    } else {
    
		// 符合当前规定线程处理notification
        NSLog(@"xxxxxxxxxxxxxxxxxx thread = %@", [NSThread currentThread]);
    }
}

@end
```

运行结果

```
2016-11-24 13:50:48.029 Demo[883:321794] xxxxxxxxxxxxxxxxxx thread = <NSThread: 0x17006fc40>{number = 1, name = main}
2016-11-24 13:50:50.010 Demo[883:321794] xxxxxxxxxxxxxxxxxx thread = <NSThread: 0x17006fc40>{number = 1, name = main}
2016-11-24 13:50:50.193 Demo[883:321794] xxxxxxxxxxxxxxxxxx thread = <NSThread: 0x17006fc40>{number = 1, name = main}
2016-11-24 13:50:50.393 Demo[883:321794] xxxxxxxxxxxxxxxxxx thread = <NSThread: 0x17006fc40>{number = 1, name = main}
2016-11-24 13:50:50.560 Demo[883:321794] xxxxxxxxxxxxxxxxxx thread = <NSThread: 0x17006fc40>{number = 1, name = main}
2016-11-24 13:50:50.743 Demo[883:321794] xxxxxxxxxxxxxxxxxx thread = <NSThread: 0x17006fc40>{number = 1, name = main}
2016-11-24 13:50:50.893 Demo[883:321794] xxxxxxxxxxxxxxxxxx thread = <NSThread: 0x17006fc40>{number = 1, name = main}
```

即使在子线程post通知，仍然是在主线程处理通知回调。小结下大致的步骤:


- (1) 子线程post 一个 notification

- (2) 执行接收通知回调函数
	- (2.1) current thread == 主线程
	- (2.2) current thread == 子线程

- (3) current thread == 主线程
	- 直接处理接收到的Notification

- (4) current thread == 子线程
	- 临时将接收到的notification添加到数组保存
	- 通过MachPort向主线程发送一个端口事件
	
- (5) `主线程runloop`接收到端口事件
	- 【重要】流程转到`主线程`上执行
	- 执行NSMachPortDelegate协议函数，而此时已经处于`主线程`了
	- 取出数组中保存的notification进行处理


如上只是一个实现通知重定向的最简单的思路，可以将这些代码封装成一个NSNotificationCenter的子类去完成。

## 通过NSNotificationQueue异步发送通知

```objc
- (void)test4 {
    
    //1. 创建center
    _myCenter = [[NSNotificationCenter alloc] init];
    
    //2. 创建NSNotificationQueue
    _notificationQueue = [[NSNotificationQueue alloc] initWithNotificationCenter:_myCenter];
    
    //3. 创建NSNotification
    NSNotification *notification = [[NSNotification alloc] initWithName:@"myKey" object:nil userInfo:nil];
    
    //4.
    _queue = [[NSOperationQueue alloc] init];
    
    //5.
    [_myCenter addObserverForName:@"myKey" object:nil queue:_queue usingBlock:^(NSNotification * _Nonnull note)
     {
         NSLog(@"通知回调: %@", [NSThread currentThread]);
     }];
    
    //6. 发送通知
    NSLog(@"发送通知之前");
    [_notificationQueue enqueueNotification:notification
                               postingStyle:NSPostWhenIdle
                               coalesceMask:NSNotificationNoCoalescing
                                   forModes:@[NSDefaultRunLoopMode]];
    NSLog(@"发送通知之后");
    
    /*
     typedef NS_ENUM(NSUInteger, NSPostingStyle) {
        NSPostWhenIdle = 1,     当runloop没有其他事件接收时，才去发送这个通知（异步）
        NSPostASAP = 2,         当runloop的当前的此次循环执行完毕，就去发送这个通知（异步）
        NSPostNow = 3,          立马去发送这个通知（同步）
     };
     
     typedef NS_OPTIONS(NSUInteger, NSNotificationCoalescing) {
        NSNotificationNoCoalescing = 0,         不合并通知，每个通知都发送一次
        NSNotificationCoalescingOnName = 1,     合并同名的通知，只会发送一条这样的通知
        NSNotificationCoalescingOnSender = 2,   合并来自同一个发送者的通知，只会发送一条这样的通知
     };
     */
    
    //7. 移除这个通知（从不执行合并的操作队列顶部移除通知）
//    [_notificationQueue dequeueNotificationsMatching:notification coalesceMask:NSNotificationNoCoalescing];
}
```