## `dispatch_queue_t`

### 数据结构都是 `队列` ，都是保持 `先进先出` 来调度block

#### 分为两种类型的队列

- 串行 >>> `一个一个`接着调度 >>> `按照先后顺序`
- 并行 >>> `随机取出几个`一起调度 >>> `随机无序`

#### 串行队列:
	
- (1) `dispatch_main_queue` 系统主线程串行队列
- (2) `dispatch_queue_create("队列名", NULL)` 自行创建的串行队列

#### 全局并发队列: 

- `dispatch_get_global_queue()` 系统全部并发队列，其实分为如下四个不同的queue
	- (3) high queue
	- (4) default queue
	- (5) low queue
	- (6) backgroud queue
- (7) `dispatch_queue_create("队列名", DISPATCH_QUEUE_CONCURRENT)` 自行创建的并发队列

所以，总共分为`7`种队列。

## `iOS8之前`，四种权限的全局并发GCD队列优先级、以及获取方式

使用的是类似如下，`dispatch queue priority` 队列的优先级，来获取queue。

```c
#define DISPATCH_QUEUE_PRIORITY_HIGH 2
#define DISPATCH_QUEUE_PRIORITY_DEFAULT 0
#define DISPATCH_QUEUE_PRIORITY_LOW (-2)
#define DISPATCH_QUEUE_PRIORITY_BACKGROUND INT16_MIN
```

其调度权限，依次从上往下`降低`。

```c
// 注意，第二个参数一般默认都写0即可
dispatch_queue_t queue1 = dispatch_get_global_queue(2, 0);
dispatch_queue_t queue2 = dispatch_get_global_queue(0, 0);
dispatch_queue_t queue3 = dispatch_get_global_queue(-2, 0);
dispatch_queue_t queue4 = dispatch_get_global_queue(INT16_MIN, 0);
```

苹果对获取全局队列方法的注释:

```c
The well-known global concurrent queues may not be modified. Calls to
 * dispatch_suspend(), dispatch_resume(), dispatch_set_context(), etc., will
 * have no effect when used with queues returned by this function.
```

可以看到，对global queue 做`dispatch_suspend(), dispatch_resume(), dispatch_set_context()`这三种处理，是没有任何效果的。

## 刚开始我认为四种优先级的队列，都是`同一个对象`，只是权限不同而已，这是`错误`的。

```objc
- (void)testGlobalQueue {
    dispatch_queue_t queue1 = dispatch_get_global_queue(2, 0);
    dispatch_queue_t queue2 = dispatch_get_global_queue(0, 0);
    dispatch_queue_t queue3 = dispatch_get_global_queue(-2, 0);
    dispatch_queue_t queue4 = dispatch_get_global_queue(INT16_MIN, 0);
    
}
```

输出结构

```
(OS_dispatch_queue_root *) queue1 = 0x000000010dcfe3c0
(OS_dispatch_queue_root *) queue2 = 0x000000010dcfe240
(OS_dispatch_queue_root *) queue3 = 0x000000010dcfe0c0
(OS_dispatch_queue_root *) queue4 = 0x000000010dcfdf40
```

可以看到:

- (1) 真正的queue类型是`OS_dispatch_queue_root`
- (2) 4个queue的内存地址，都是`不同`的

所以说，四种权限级别的queue，就是四个不同的对象。

## 是否需要控制创建`dispatch_queue_t`实例的数量？

### 对于 GCD Concurrent Queue:

- (1) 最终GCD底层对于线程已经使用了线程池机制优化的
- (2) 一个线程执行完任务之后，不会立马被释放掉，而是先暂时缓存起来
- (3) 当调度任务进入时，从缓存中取出一个线程进行处理

所以，一个Concurrent Queue可能会创建`n个`子线程，所以应该也要避免随意的创建Concurrent Queue实例。

但是一般情况下，使用`自己创建`的GCD Concurrent Queue场景很少，一般都是直接使用`dispatch_get_global_queue(long identifier, unsigned long flags);`获取系统的全局并发队列即可。

### 对于 GCD Serial Queue:

- (1) 一个Serial Queue新的实例，就相当于一个`新的子线程`
- (2) 不断地创建 GCD Serial Queue 实例，就等同于不断的`创建新的子线程`

这样就完全绕过了`线程池复用`机制了，压根就不会复用原来的线程，就有可能造成大量的线程创建，降低CPU的性能。

所以，对于Serial Queue的创建，还是有必要进行限制。

## 到底是缓存`NSThread对象`，还是缓存`dispatch_queue_t实例` ？

### 直接对NSThread对象进行缓存是不行的，必须经过如下步骤:

- (1) 给thread获取一个RunLoop
- (2) 向RunLoop添加`RunLoop Source`
- (3) 开启thread的runloop

这样才能保证thread一直处于`executing`状态，后续也才能接受任务执行。

### 现在基本上都是操作`dispatch_queue_t`，很少去操作`NSThread`、`pthread` ...

对于`dispatch_queue_t`实例进行缓存，并不需要像缓存therad对象做那么多的处理。

想用的时候取出`dispatch_queue_t`实例，只要往里面分配block就能够执行，并且GCD底层实现了thread池可以复用。

### 使用例子测试缓存`dispatch_queue_t`是否可行

用于缓存`dispatch_queue_t`实例的Context:

```
// 保存dispatch_queue_t实例的容器
struct dispatch_queue_context {
    void **queues;
};

// 指针类型
typedef struct dispatch_queue_context* dispatch_queue_context_t;

// 全局context实例
static struct dispatch_queue_context context;
```

ViewController中初始化全局Context实例，然后从Context中获取queue使用：

```objc
@implementation ViewController

- (instancetype)initWithCoder:(NSCoder *)aDecoder {
    if (self = [super initWithCoder:aDecoder]) {
        [self setupContextQueues];
    }
    return self;
}

- (instancetype)initWithNibName:(NSString *)nibNameOrNil bundle:(NSBundle *)nibBundleOrNil {
    if (self = [super initWithNibName:nibNameOrNil bundle:nibBundleOrNil]) {
        [self setupContextQueues];
    }
    return self;
}

- (void)setupContextQueues {
    
    //1. 给context的queue数组分配内存
    context.queues = malloc(sizeof(dispatch_queue_t) * 5);
    
    //2. 分别创建queue并填充数组
    for (int i = 0; i < 5; i++) {
        dispatch_queue_t queue = dispatch_queue_create("queue", NULL);
        
        //【重点】一定要retain一下queue，否则后续取出queue进行类型转换的时候会崩溃
        context.queues[i] = (__bridge_retained void*)(queue);
    }
}

- (void)test_dispatch_queue_context {
    /**
     *  从context中取出queue
     */
    
    for (int i = 0; i < 5; i++) {
        dispatch_queue_t queue = (__bridge dispatch_queue_t)(context.queues[i]);
        dispatch_async(queue, ^{
            NSThread *t = [NSThread currentThread];
            NSLog(@"task %d, thread>>> isCanceled=%d, isExecuting=%d, isFinish=%d", i, t.isCancelled, t.isExecuting, t.isFinished);
        });
    }
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	    [self test_dispatch_queue_context];
}

@end
```

输出结果

```
2017-02-22 21:44:41.408 Demo[2011:31446] task 1, thread>>> isCanceled=0, isExecuting=1, isFinish=0
2017-02-22 21:44:41.408 Demo[2011:31445] task 0, thread>>> isCanceled=0, isExecuting=1, isFinish=0
2017-02-22 21:44:41.408 Demo[2011:31556] task 3, thread>>> isCanceled=0, isExecuting=1, isFinish=0
2017-02-22 21:44:41.408 Demo[2011:31447] task 2, thread>>> isCanceled=0, isExecuting=1, isFinish=0
2017-02-22 21:44:41.408 Demo[2011:31557] task 4, thread>>> isCanceled=0, isExecuting=1, isFinish=0
```

可以看到，最终分配的线程状态一直是`isExecuting=1`，表明线程一直是存活的。

所以可以直接针对`dispatch_queue_t`实例进行缓存，会更方便也更高效。就把一个`dispatch_queue_t`实例，看做成一个`thread`。

### 线程池中最大并发数量多少才会最好了？

```
http://ifeve.com/how-to-calculate-threadpool-size/
http://chuansong.me/n/2769744
```

看起来这个计算的过程还是比较复杂的，不是那么简单就可以得出一个数字。首先是，一般CPU并发线程控制的场景主要分为:

- (1) CPU密集型（对象创建、UI布局、绘图...）
- (2) IO型（加解密、压缩解压缩、搜索排序...）


直接拷贝下文章中总结出的最佳CPU线程并发数计算公式:

```
最佳线程数目 = （线程等待时间/线程占用CPU时间 + 1）* CPU总核数
```

一般来说，`非CPU密集型`的业务（加解密、压缩解压缩、搜索排序等业务是CPU密集型的业务），瓶颈都在`后端数据库`，本地CPU计算的时间很少，所以设置几十或者几百个工作线程也都是可能的。


但是对于`CPU密集型`操作，我觉得`线程等待时间` 和 `线程占用CPU时间`应该都是很短的，而等待时间估计会更短，所以将上面的公式如果在密集型CPU类型时，可以简化成如下:

```
最佳线程数目 = CPU总核数;
```

也就是直接等于`CPU的总核心数`。

## 串行队列 和 并发队列，哪种类型需要进行缓存？

### 第一种、单独完成某一块业务代码的多线程同步

比如，网络请求数据的文件缓存，创建一个单例的queue实例，所有的请求缓存数据文件的读写操作，都必须在这个单例的queue上排队执行。

对于这种串行队列，就不太适合做全局的缓存队列，因为他只适用于这一个业务木块的同步代码逻辑。

### 第二种、能够随机的取出一个队列来执行一个任务

这种适合`随机`的一个任务进行调度，随机的从缓存中取出一个队列进行任务调度执行。

比如，大量的CoreText、CoreGraphics等等任务代码，并不要求有`执行循序`，也不要求一定要在哪一个线程上执行。

对于这种，只需要随机获取一个队列进行任务调度的操作，就需要一种随机获取队列的缓存。

### 第三种、全局并发队列

这种队列，开发者不能自己创建，就不考虑。

## 结合线程最大并发数，设计`dispatch_queue_t`的缓存结构

首先看到系统的`global concurrent dispatch queue` 全局并发队列，按照`dispatch queue priority`线程队列优先级，进行划分的结构。

### 一种优先级下，只存在一个并发queue实例

```c
- (1) High Priority
	- dispatch high priority queue 实例1
- (2) Default Priority
	- dispatch default priority queue 实例1
- (3) Low Priority
	- dispatch low priority queue 实例1
- (4) Backgroud Priority
	- dispatch backgroud priority queue 实例1
```

可以看到，每一个优先级下，只存在一个并发队列实例。

试想一下，我有4个任务，需要在`Low Priority`下的queue实例进行任务调度。

对于并发队列可以同时让多个线程执行任务。但是这个线程数不太好控制，有可能是2个，也可能是3个。

总之，无法保证同时百分百的跑满4个线程。而CPU一般都有2到4个核心，每一个核心都能跑一个线程。而系统并发队列，无法总是跑满4个线程，所以很多时候都会让1个或2个核心处于休闲状态...

### 一种优先级下，只存在`CPU核心数个`并发queue实例

```c
- (1) High Priority
	- dispatch high priority queue 实例1
	- dispatch high priority queue 实例2
	- dispatch high priority queue 实例3
	- dispatch high priority queue 实例4
- (2) Default Priority
	- dispatch default priority queue 实例1
	- dispatch default priority queue 实例2
	- dispatch default priority queue 实例3
	- dispatch default priority queue 实例4
- (3) Low Priority
	- dispatch low priority queue 实例1
	- dispatch low priority queue 实例2
	- dispatch low priority queue 实例3
	- dispatch low priority queue 实例4
- (4) Backgroud Priority
	- dispatch backgroud priority queue 实例1
	- dispatch backgroud priority queue 实例2
	- dispatch backgroud priority queue 实例3
	- dispatch backgroud priority queue 实例4
```

这样来说，虽然是可以保证绝对4个线程了。因为一个并发队列，至少产生一个线程。

但是，4个并发队列，产生的线程的有太多了。所以，需要将`并发`队列替换为`串行`队列，就刚好`4`个线程了，并且永远使用这4个线程。

OK，最终的结构如下，其中的queue都是`串行`的：

```c
- (1) High Priority
	- dispatch high priority 串行 queue 实例1
	- dispatch high priority 串行 queue 实例2
	- dispatch high priority 串行 queue 实例3
	- dispatch high priority 串行 queue 实例4
- (2) Default Priority
	- dispatch default priority 串行 queue 实例1
	- dispatch default priority 串行 queue 实例2
	- dispatch default priority 串行 queue 实例3
	- dispatch default priority 串行 queue 实例4
- (3) Low Priority
	- dispatch low priority 串行 queue 实例1
	- dispatch low priority 串行 queue 实例2
	- dispatch low priority 串行 queue 实例3
	- dispatch low priority 串行 queue 实例4
- (4) Backgroud Priority
	- dispatch backgroud priority 串行 queue 实例1
	- dispatch backgroud priority 串行 queue 实例2
	- dispatch backgroud priority 串行 queue 实例3
	- dispatch backgroud priority 串行 queue 实例4
```

## `dispatch_get_global_queue();`在iOS8之前与iOS8及之后，传入参数的区别

```c
/*!
 * @function dispatch_get_global_queue
 *
 * @abstract
 * Returns a well-known global concurrent queue of a given quality of service
 * class.
 *
 * @discussion
 * The well-known global concurrent queues may not be modified. Calls to
 * dispatch_suspend(), dispatch_resume(), dispatch_set_context(), etc., will
 * have no effect when used with queues returned by this function.
 *
 * @param identifier
 * A quality of service class defined in qos_class_t or a priority defined in
 * dispatch_queue_priority_t.
 *
 * It is recommended to use quality of service class values to identify the
 * well-known global concurrent queues:
 *  - QOS_CLASS_USER_INTERACTIVE
 *  - QOS_CLASS_USER_INITIATED
 *  - QOS_CLASS_DEFAULT
 *  - QOS_CLASS_UTILITY
 *  - QOS_CLASS_BACKGROUND
 *
 * The global concurrent queues may still be identified by their priority,
 * which map to the following QOS classes:
 *  - DISPATCH_QUEUE_PRIORITY_HIGH:         QOS_CLASS_USER_INITIATED
 *  - DISPATCH_QUEUE_PRIORITY_DEFAULT:      QOS_CLASS_DEFAULT
 *  - DISPATCH_QUEUE_PRIORITY_LOW:          QOS_CLASS_UTILITY
 *  - DISPATCH_QUEUE_PRIORITY_BACKGROUND:   QOS_CLASS_BACKGROUND
 *
 * @param flags
 * Reserved for future use. Passing any value other than zero may result in
 * a NULL return value.
 *
 * @result
 * Returns the requested global queue or NULL if the requested global queue
 * does not exist.
 */
dispatch_queue_t dispatch_get_global_queue(long identifier, unsigned long flags);
```

从注释中可以得到: 

- 1) identifier参数，在iOS8以下与iOS8及以上时，传入的类型不同
- 2) 苹果更推荐使用 `quality of service`，代替`priority`

那么这里，就引出一个问题，我们创建将要缓存的串行队列时，也有两种方法：

- (1) priority
- (2) quality of service

那么，需要做一个系统版本兼容：

- (1) iOS8之前，使用 priority
- (2) iOS8及之后，使用 quality of service

### identifier参数，代表`两种`类型枚举值:
	
- (1) 服务质量: iOS8、iOS8+ 

```c
a quality of service class defined in qos_class_t
```
	
- (2) 队列优先级: iOS7、iOS7-

```c
a priority defined in dispatch_queue_priority_t
```

### 苹果更推荐使用 `服务质量` 代替 `优先级`，原因是如下:

| 队列描述（服务质量 or 优先级）| 最终执行线程描述 | 
| :-------------: |:-------------:| 
| 服务质量 Quality Of Service | thread.qualityOfService = NSQualityOfServiceUserInteractive | 
| 优先级 Queue Priority | thread.threadPriority = 0.5 | 
	
	
对于优先级，最终被执行的线程是通过一个`数值`来表示，可能有时候根本无法分别0.5和0.6之间有什么区别。但是对于服务质量就很容易区分了。

## iOS8以前，使用 `队列优先级 dispatch_queue_priority_t `

### 获取队列时传入的参数


```c
#define DISPATCH_QUEUE_PRIORITY_HIGH 2
#define DISPATCH_QUEUE_PRIORITY_DEFAULT 0
#define DISPATCH_QUEUE_PRIORITY_LOW (-2)
#define DISPATCH_QUEUE_PRIORITY_BACKGROUND INT16_MIN
```

也就是说，只有4种queue实例。

### 反映到最终NSThread上的参数

```c
@interface NSThread : NSObject
+ (double)threadPriority;
@end
```

```c
double num = -[NSThread threadPriority];
```

就是一个数值，可能有时候根本无法分别0.5和0.6之间有什么区别。

## iOS8以及之后，使用 `服务质量 qos_class_t`

### 获取队列时传入的参数

```c
#define __QOS_ENUM(name, type, ...) enum { __VA_ARGS__ }; typedef type name##_t
#define __QOS_CLASS_AVAILABLE_STARTING(...)

#if defined(__has_feature) && defined(__has_extension)
#if __has_feature(objc_fixed_enum) || __has_extension(cxx_strong_enums)
#undef __QOS_ENUM
#define __QOS_ENUM(name, type, ...) typedef enum : type { __VA_ARGS__ } name##_t
#endif
#if __has_feature(enumerator_attributes)
#undef __QOS_CLASS_AVAILABLE_STARTING
#define __QOS_CLASS_AVAILABLE_STARTING __OSX_AVAILABLE_STARTING
#endif
#endif

__QOS_ENUM(qos_class, unsigned int,
	QOS_CLASS_USER_INTERACTIVE
			__QOS_CLASS_AVAILABLE_STARTING(__MAC_10_10, __IPHONE_8_0) = 0x21,
	QOS_CLASS_USER_INITIATED
			__QOS_CLASS_AVAILABLE_STARTING(__MAC_10_10, __IPHONE_8_0) = 0x19,
	QOS_CLASS_DEFAULT
			__QOS_CLASS_AVAILABLE_STARTING(__MAC_10_10, __IPHONE_8_0) = 0x15,
	QOS_CLASS_UTILITY
			__QOS_CLASS_AVAILABLE_STARTING(__MAC_10_10, __IPHONE_8_0) = 0x11,
	QOS_CLASS_BACKGROUND
			__QOS_CLASS_AVAILABLE_STARTING(__MAC_10_10, __IPHONE_8_0) = 0x09,
	QOS_CLASS_UNSPECIFIED
			__QOS_CLASS_AVAILABLE_STARTING(__MAC_10_10, __IPHONE_8_0) = 0x00,
);

#undef __QOS_ENUM
```

简化为如下:

```c
typedef enum : unsigned int {
	QOS_CLASS_USER_INTERACTIVE = 0x21, //33，与用户交互的任务，这些任务通常跟UI有关，这些任务需要在`一瞬间`完成
	QOS_CLASS_USER_INITIATED = 0x19, //25，也是一些与UI相关的任务，但是对完成时间并不是需要一瞬间立刻完成，可以延迟一点点
	QOS_CLASS_UTILITY = 0x11, //17，一些可能需要花点时间的任务，这些任务不需要马上返回结果，可能需要几分钟
	QOS_CLASS_BACKGROUND = 0x09, //9，这些任务对用户不可见，比如后台进行备份的操作，执行时间可能很长
	QOS_CLASS_DEFAULT = 0x15, // 21，当没有 QoS信息时默认使用，苹果建议不要使用这个级别
	QOS_CLASS_UNSPECIFIED = 0x00,//0，这个是一个标记，没啥实际作用
} qos_class_t;
```

`0x`表示十六进制数，转换十进制数的方法：

```
0x21 = 2*16^1 + 1*16^0 = 32 + 1 = 33
0x200 = 2*16^2 + 0*16^1 + 0*16^0 = 512
```

### 反映到最终NSThread上的参数

```c
@interface NSThread : NSObject
@property NSQualityOfService qualityOfService NS_AVAILABLE(10_10, 8_0); // read-only after the thread is started
@end
```

```c
NSQualityOfService qos = -[NSThread qualityOfService];
```

下面代码测试最终分配给thread的qos值

```c
dispatch_queue_t USER_INTERACTIVE = dispatch_get_global_queue(QOS_CLASS_USER_INTERACTIVE, 0);
dispatch_queue_t USER_INITIATED = dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0);
dispatch_queue_t UTILITY = dispatch_get_global_queue(QOS_CLASS_UTILITY, 0);
dispatch_queue_t BACKGROUND = dispatch_get_global_queue(QOS_CLASS_BACKGROUND, 0);
dispatch_queue_t DEFAULT = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);

dispatch_async(BACKGROUND, ^{
    NSLog(@"BACKGROUND task >>> thread quality = %ld", [NSThread currentThread].qualityOfService);
});

dispatch_async(UTILITY, ^{
    NSLog(@"UTILITY task >>> thread quality = %ld", [NSThread currentThread].qualityOfService);
});

dispatch_async(DEFAULT, ^{
    NSLog(@"DEFAULT task >>> thread quality = %ld", [NSThread currentThread].qualityOfService);
});

dispatch_async(USER_INITIATED, ^{
    NSLog(@"USER_INITIATED task >>> thread quality = %ld", [NSThread currentThread].qualityOfService);
});

dispatch_async(USER_INTERACTIVE, ^{
    NSLog(@"USER_INTERACTIVE task >>> thread quality = %ld", [NSThread currentThread].qualityOfService);
});
```


运行结果有点`不太固定`，在大部分情况下是这样的

```
2016-08-14 16:00:59.795 Demos[11151:178723] USER_INTERACTIVE task >>> thread quality = 33
2016-08-14 16:00:59.795 Demos[11151:177932] USER_INITIATED task >>> thread quality = 25
2016-08-14 16:00:59.795 Demos[11151:178725] DEFAULT task >>> thread quality = -1
2016-08-14 16:00:59.795 Demos[11151:178727] UTILITY task >>> thread quality = 17
2016-08-14 16:00:59.798 Demos[11151:178726] BACKGROUND task >>> thread quality = 9
```


`USER_INTERACTIVE`和`USER_INITIATED`这两个队列的调度顺序谁先谁后都有可能，按照苹果注释来看，从理论来说应该是`USER_INTERACTIVE > USER_INITIATED`.

这里很奇怪，不太明白为什么`DEFAULT task >>> thread quality = -1`。后来我发现对应的Foundation定义:

```c
typedef NS_ENUM(NSInteger, NSQualityOfService) {
    NSQualityOfServiceUserInteractive = 0x21,
    NSQualityOfServiceUserInitiated = 0x19,
    NSQualityOfServiceUtility = 0x11,
    NSQualityOfServiceBackground = 0x09,
    NSQualityOfServiceDefault = -1
}
```

里面定义的 qos default 就是`-1`，难道是以这个枚举值为标准的？

```objc
- (void)testQOS2 {
    dispatch_queue_t DEFAULT1 = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
    dispatch_queue_t DEFAULT2 = dispatch_get_global_queue(NSQualityOfServiceDefault, 0);
    NSLog(@"");
}
```

输出结果

```
(OS_dispatch_queue_root *) DEFAULT1 = 0x0000000108722240
(dispatch_queue_t) DEFAULT2 = nil
```

可是发现，`NSQualityOfServiceDefault`这个枚举值根本就获取不到queue实例，所以还是以`0x15`为qos default。

从这里可以看到，使用`服务质量`来描述队列的调度级别绝对是比使用`优先级`描述好多了，不像`threadPriority`为 `0.5` `0.0` `0.2`，不能够很清楚理解有什么区别。


## `4种 dispatch queue priority` 与 `5种 QOS` 之间的关系

```objc
- (void)testQOSAndPriority {
    
    // 5种，服务质量，获取的全局队列
    dispatch_queue_t QOS_USER_INTERACTIVE = dispatch_get_global_queue(QOS_CLASS_USER_INTERACTIVE, 0);// 线程优先级无法获取的全局队列
    dispatch_queue_t QOS_USER_INITIATED = dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0);
    dispatch_queue_t QOS_UTILITY = dispatch_get_global_queue(QOS_CLASS_UTILITY, 0);
    dispatch_queue_t QOS_BACKGROUND = dispatch_get_global_queue(QOS_CLASS_BACKGROUND, 0);
    dispatch_queue_t QOS_DEFAULT = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
    
    // 4种，优先级，获取的全局队列
    dispatch_queue_t PRIORITY_HIGH = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);
    dispatch_queue_t PRIORITY_LOW = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);
    dispatch_queue_t PRIORITY_BACKGROUND = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);
    dispatch_queue_t PRIORITY_DEFAULT = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
}
```

输出结果

```
// 5种QOS的queue
(OS_dispatch_queue_root *) QOS_USER_INTERACTIVE = 0x0000000105868540
(OS_dispatch_queue_root *) QOS_USER_INITIATED = 0x00000001058683c0
(OS_dispatch_queue_root *) QOS_UTILITY = 0x00000001058680c0
(OS_dispatch_queue_root *) QOS_BACKGROUND = 0x0000000105867f40
(OS_dispatch_queue_root *) QOS_DEFAULT = 0x0000000105868240

// 4种prioerity的queue
(OS_dispatch_queue_root *) PRIORITY_HIGH = 0x00000001058683c0
(OS_dispatch_queue_root *) PRIORITY_LOW = 0x00000001058680c0
(OS_dispatch_queue_root *) PRIORITY_BACKGROUND = 0x0000000105867f40
(OS_dispatch_queue_root *) PRIORITY_DEFAULT = 0x0000000105868240
```

可以看到QOS中的`QOS_CLASS_USER_INTERACTIVE`获得的queue，在所有通过Priority获得的queue中，是无法对应的。而其他4种QOS获得queue都是能够对应的。

也就是说，iOS8之后，只有通过`QOS_CLASS_USER_INTERACTIVE`这个QOS才能获取得到这个queue实例。

## 两种类型下对于default队列有区别的:

- (1) QOS，苹果是不推荐使用`DEFAULT`，具体注释如下

```c
Default QoS indicates the absence of QoS information.
（QOS_CLASS_DEFAULT 表示一种缺省默认的 QOS类型）

Whenever possible QoS information will be inferred from other sources.
（该服务质量类型将会从其他类型来进行推断）

If such inference is not possible, a QoS between UserInitiated and Utility will be used
（可能不是确定的，可能选择 UserInitiated 或 Utility 进行代替）
```

从注释大概意思看到，`QOS_CLASS_DEFAULT`不是一种确认的模式，有时候可能是`UserInitiated`模式，也可能是`Utility`模式，所以在使用QOS时不要去使用`DEFAULT`模式。

- (2) Priority，貌似我们大部分情况下都在使用`DEFAULT`，具体注释如下

```c
the queue will be scheduled for execution after all high priority queues have been scheduled,
but before any low priority queues have been scheduled.

（当所有的高优先级队列调度完毕，再进行default级别队列调度，但是比一些low优先级队列先调度）
```

## 那么设计一个兼容`iOS8-`与`iOS8、iOS+`版本下，获取global queue不同的问题

### iOS8系统，会默认将传入的 `dispatch queue priority` 替换成对应的 `quality of service`

| QOS 服务质量 | Priority 优先级 |
| :-------------: |:-------------:| 
| `DISPATCH_QUEUE_PRIORITY_HIGH` | `QOS_CLASS_USER_INITIATED` |
|  `DISPATCH_QUEUE_PRIORITY_DEFAULT` | `QOS_CLASS_DEFAULT` |
| `DISPATCH_QUEUE_PRIORITY_LOW` | `QOS_CLASS_UTILITY` |
| `DISPATCH_QUEUE_PRIORITY_BACKGROUND` | `QOS_CLASS_BACKGROUND` |

注意，其中`dispatch_get_global_queue(QOS_CLASS_USER_INTERACTIVE, 0);`，在`iOS8`之前获取不到实例的。

## 现在统一使用QOS操作队列，但是如果是iOS8以下系统，就需要将QOS转换成Priority

自定义QOS的枚举类型，代替`iOS8`才能够使用的的`qos_class_t`

```c
typedef NS_ENUM(NSInteger, XZHQualityOfService) {
    XZHQualityOfServiceUserInteractive          = 0x21,
    XZHQualityOfServiceUserInitiated            = 0x19,
    XZHQualityOfServiceUtility                  = 0x11,
    XZHQualityOfServiceBackground               = 0x09,
    XZHQualityOfServiceDefault                  = 0x15,
};
```

编写工具函数当系统是iOS8以下时，将`XZHQualityOfService`转换为对应的`dispatch_queue_priority_t`

```c
static inline dispatch_queue_priority_t XZHQosToPriority(XZHQualityOfService qos)
{
    switch (qos) {
        case XZHQualityOfServiceUserInteractive: {return DISPATCH_QUEUE_PRIORITY_HIGH;}
        case XZHQualityOfServiceUserInitiated: {return DISPATCH_QUEUE_PRIORITY_HIGH;}
        case XZHQualityOfServiceUtility: {return DISPATCH_QUEUE_PRIORITY_LOW;}
        case XZHQualityOfServiceBackground: {return DISPATCH_QUEUE_PRIORITY_BACKGROUND;}
        case XZHQualityOfServiceDefault: {return DISPATCH_QUEUE_PRIORITY_DEFAULT;}
        default: {return DISPATCH_QUEUE_PRIORITY_DEFAULT;}
    }
}
```

## `NSQualityOfServiceDefault` 与 `QOS_CLASS_DEFAULT` 区别

```objc
- (void)testNSQOSAndCQOS {
    dispatch_queue_t QOS_1_USER_INTERACTIVE = dispatch_get_global_queue(QOS_CLASS_USER_INTERACTIVE, 0);
    dispatch_queue_t QOS_1_USER_INITIATED = dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0);
    dispatch_queue_t QOS_1_UTILITY = dispatch_get_global_queue(QOS_CLASS_UTILITY, 0);
    dispatch_queue_t QOS_1_BACKGROUND = dispatch_get_global_queue(QOS_CLASS_BACKGROUND, 0);
    dispatch_queue_t QOS_1_DEFAULT = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
    
    
    dispatch_queue_t QOS_2_USER_INTERACTIVE = dispatch_get_global_queue(NSQualityOfServiceUserInteractive, 0);
    dispatch_queue_t QOS_2_USER_INITIATED = dispatch_get_global_queue(NSQualityOfServiceUserInitiated, 0);
    dispatch_queue_t QOS_2_UTILITY = dispatch_get_global_queue(NSQualityOfServiceUtility, 0);
    dispatch_queue_t QOS_2_BACKGROUND = dispatch_get_global_queue(NSQualityOfServiceBackground, 0);
    dispatch_queue_t QOS_2_DEFAULT = dispatch_get_global_queue(NSQualityOfServiceDefault, 0);
    
    
}
```

输出如下

```
(OS_dispatch_queue_root *) QOS_1_USER_INTERACTIVE = 0x0000000110026540
(OS_dispatch_queue_root *) QOS_1_USER_INITIATED = 0x00000001100263c0
(OS_dispatch_queue_root *) QOS_1_UTILITY = 0x00000001100260c0
(OS_dispatch_queue_root *) QOS_1_BACKGROUND = 0x0000000110025f40
(OS_dispatch_queue_root *) QOS_1_DEFAULT = 0x0000000110026240

(OS_dispatch_queue_root *) QOS_2_USER_INTERACTIVE = 0x0000000110026540
(OS_dispatch_queue_root *) QOS_2_USER_INITIATED = 0x00000001100263c0
(OS_dispatch_queue_root *) QOS_2_UTILITY = 0x00000001100260c0
(OS_dispatch_queue_root *) QOS_2_BACKGROUND = 0x0000000110025f40
(dispatch_queue_t) QOS_2_DEFAULT = nil
```

可以看到 `NSQualityOfServiceDefault`是获取不到queue的。

### 所以，最好是将 NSQualityOfService 转换成`qos_class_t`

```c
static inline XZHQualityOfService XZHQosFromNSQos(NSQualityOfService qos) {
    switch (qos) {
        case NSQualityOfServiceUserInteractive: {return XZHQualityOfServiceUserInteractive;}
        case NSQualityOfServiceUserInitiated: {return XZHQualityOfServiceUserInitiated;}
        case NSQualityOfServiceUtility: {return XZHQualityOfServiceUtility;}
        case NSQualityOfServiceBackground: {return XZHQualityOfServiceBackground;}
        case NSQualityOfServiceDefault: {return XZHQualityOfServiceDefault;}
        default: {return XZHQualityOfServiceBackground;}
    }
}
```

## 使用QOS与Priority创建Queue时的区别

### iOS8之前，给自己创建的队列设置优先级，需要从其他系统队列复制

```c
//1. 从哪一个系统队列进行优先级复制
dispatch_queue_t source = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)

//2. 先随便创建一个串行队列（没有指定队列优先级的）
dispatch_queue_t queue = dispatch_queue_create("haha", DISPATCH_QUEUE_SERIAL);

//3. 将dispatch_set_target_queue()函数中，第二个队列的优先级，复制给第一个队列
dispatch_set_target_queue(queue, source);
```

无法直接在创建队列时，指定队列的优先级，只能通过`dispatch_set_target_queue()`函数进行优先级复制。

注意，系统创建出来的`4种全局并发队列`、`主线程队列`他们的优先级都是已经设置好了的。

### iOS8之后，可以直接在创建队列时，指定服务质量来描述队列的调度级别

```c
//1. 服务质量
qos_class_t qosClass = QOS_CLASS_DEFAULT;

//2. 使用服务质量，创建queue attr
dispatch_queue_attr_t queue_attr = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL, qosClass, -1);

//3. 使用queue attr，创建queue
dispatch_queue_t queue = dispatch_queue_create("haha", queue_attr);
```

### 使用QOS创建queue时，需要先创建一个attr: `dispatch_queue_attr_make_with_qos_class()`参数意义
		

```c
__OSX_AVAILABLE_STARTING(__MAC_10_10, __IPHONE_8_0)
dispatch_queue_attr_t dispatch_queue_attr_make_with_qos_class(dispatch_queue_attr_t attr,
		dispatch_qos_class_t qos_class, int relative_priority);
```

4个参数意思:

```
qos_class:   指定你的队列的服务质量的类值。这个上面说明过了，请查看第(2)条对dispatch_get_global_queue的介绍。

relative_priority: 一个负数，对那四个特定的服务质量优先级所代表的值的一个偏移，这个值必须小于0并且大于MIN_QOS_CLASS_PRIORITY。
```

苹果的注释使用demo:

```c
* Example:
 * <code>
 *	dispatch_queue_t queue;
 *	dispatch_queue_attr_t attr;
 *	attr = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL,
 *			QOS_CLASS_UTILITY, 0);
 *	queue = dispatch_queue_create("com.example.myqueue", attr);
 * </code>
```

## 对于优先使用iOS8推荐的QualityOfService描述线程队列，所以基于此大致得出将要实现的dispatch queue pool的内存结构

因为根据QualityOfService服务质量，将线程定义为5种类型，不用的类型执行的优先级、获取的系统资源、CPU时间都是不相同的。

而不同的`QualityOfService`服务质量下，必须存放对应的`dispatch_queue_t`实例。所以，必须要分开按照不同的`QualityOfService`类型，来缓存对应服务质量下的所有的`dispatch_queue_t`实例。

所以说需要有一个类似`上下文环境 Context`这么一个东西，来表示处于哪一种`QualityOfService`类型下的`dispatch_queue_t`实例缓存容器:

```c
/**
 *  对应一个QualityOfService/DispatchQueuePriority下得缓存容器，
 *  缓存这个模式下的所有的dispatch_queue_t实例
 */
typedef struct __XZHDispatchContext {
    
    /**
     *  服务质量，值参考qos_class_t
     */
    XZHQualityOfService     qos;
    
    /**
     *  缓存的所有的dispatch_queue_t实例数组
     */
    void                    *cachedQueues;
    
    /**
     *  缓存的dispatch_queue_t实例的总个数
     */
    NSUInteger              cachedQueuesCount;
    
    /**
     *  记录当前获取缓存的dispatch_queue_t实例进行调度任务的总操作次数，
     *  并且操作次数是累加的
     */
    NSUInteger              workQueueOffset;
    
}__XZHDispatchContext;
```

那么在iOS8+的系统，DispatchQueuePool对象的内部结构是如下这样:

```objc
- DispatchQueuePool 对象

	- (1) QOS_CLASS_USER_INTERACTIVE Dispatch Context 对象
		- 缓存的dispatch_queue_t 实例1
		- 缓存的dispatch_queue_t 实例1
		- ....
		- 缓存的dispatch_queue_t 实例n
	
	- (2) QOS_CLASS_USER_INITIATED Dispatch Context 对象
		- 缓存的dispatch_queue_t 实例1
		- 缓存的dispatch_queue_t 实例1
		- ....
		- 缓存的dispatch_queue_t 实例n
	
	- (3) QOS_CLASS_DEFAULT Dispatch Context 对象
		- 缓存的dispatch_queue_t 实例1
		- 缓存的dispatch_queue_t 实例1
		- ....
		- 缓存的dispatch_queue_t 实例n
	
	- (4) QOS_CLASS_UTILITY Dispatch Context 对象
		- 缓存的dispatch_queue_t 实例1
		- 缓存的dispatch_queue_t 实例1
		- ....
		- 缓存的dispatch_queue_t 实例n
	
	- (5) QOS_CLASS_BACKGROUND Dispatch Context 对象
		- 缓存的dispatch_queue_t 实例1
		- 缓存的dispatch_queue_t 实例1
		- ....
		- 缓存的dispatch_queue_t 实例n
```

然后也要考虑iOS8之前使用dispatch queue priority的情况，实际上将QualityOfService与dispatchQueuePriority做一个映射:

| QOS 服务质量 | Priority 优先级 |
| :-------------: |:-------------:| 
| `DISPATCH_QUEUE_PRIORITY_HIGH` | `QOS_CLASS_USER_INITIATED` |
|  `DISPATCH_QUEUE_PRIORITY_DEFAULT` | `QOS_CLASS_DEFAULT` |
| `DISPATCH_QUEUE_PRIORITY_LOW` | `QOS_CLASS_UTILITY` |
| `DISPATCH_QUEUE_PRIORITY_BACKGROUND` | `QOS_CLASS_BACKGROUND` |

在框架内部自动完成映射，然后对于iOS8-系统，DispatchQueuePool对象的内部结构是如下这样:

```objc
- DispatchQueuePool 对象
	
	- (1) QOS_CLASS_USER_INITIATED Dispatch Context 对象
		- 缓存的dispatch_queue_t 实例1
		- 缓存的dispatch_queue_t 实例1
		- ....
		- 缓存的dispatch_queue_t 实例n
	
	- (2) QOS_CLASS_DEFAULT Dispatch Context 对象
		- 缓存的dispatch_queue_t 实例1
		- 缓存的dispatch_queue_t 实例1
		- ....
		- 缓存的dispatch_queue_t 实例n
	
	- (3) QOS_CLASS_UTILITY Dispatch Context 对象
		- 缓存的dispatch_queue_t 实例1
		- 缓存的dispatch_queue_t 实例1
		- ....
		- 缓存的dispatch_queue_t 实例n
	
	- (4) QOS_CLASS_BACKGROUND Dispatch Context 对象
		- 缓存的dispatch_queue_t 实例1
		- 缓存的dispatch_queue_t 实例1
		- ....
		- 缓存的dispatch_queue_t 实例n
```

因为iOS8以下系统，`DispatchQueuePriority`中没有与`QOS_CLASS_USER_INTERACTIVE`能够对应的值，所以去掉这一个Context缓存，即只有4个Context。

但是为了不改变代码逻辑，可以将`QOS_CLASS_USER_INTERACTIVE`对应得Context在iOS8-系统时，替换成`QOS_CLASS_USER_INITIATED`对应得Context。

***

###那么对于上述构建整个Pool对象的步骤:

- (1) 先alloc一个Pool对象
- (2) Pool对象内部init中，再创建各种QOS的Context实例
- (3) 每一种QOS对应的Context实例，又继续创建自己缓存的n个`dispatch_queue_t`实例
- (4) 而具体创建`dispatch_queue_t`实例时，需要区分iOS7与iOS8不同创建
	- iOS7、`dispatch_queue_priority_t`
	- iOS8、`qos_class_t`

接下来就是一些具体的代码实现细节了...

***

###针对某一种`QualityOfService`时，在iOS7与iOS8不同系统，使用不同的方式来创建 `dispatch_queue_t` 实例

首先是，将QualityOfService转换成DispatchQueuePriority 

```c
static inline dispatch_queue_priority_t __XZHQualityOfServiceToPriority(XZHQualityOfService qos)
{
    switch (qos) {
        case XZHQualityOfServiceUserInteractive: {
            return DISPATCH_QUEUE_PRIORITY_HIGH;//TODO: 这个地方比较特殊
            break;
        }
        case XZHQualityOfServiceUserInitiated: {
            return DISPATCH_QUEUE_PRIORITY_HIGH;
            break;
        }
        case XZHQualityOfServiceUtility: {
            return DISPATCH_QUEUE_PRIORITY_LOW;
            break;
        }
        case XZHQualityOfServiceBackground: {
            return DISPATCH_QUEUE_PRIORITY_BACKGROUND;
            break;
        }
        case XZHQualityOfServiceDefault: {
            return DISPATCH_QUEUE_PRIORITY_DEFAULT;
            break;
        }
    }
}
```

等价转换可以参考如下映射关系:

| QOS 服务质量 | Priority 优先级 |
| :-------------: |:-------------:| 
| `DISPATCH_QUEUE_PRIORITY_HIGH` | `QOS_CLASS_USER_INITIATED` |
|  `DISPATCH_QUEUE_PRIORITY_DEFAULT` | `QOS_CLASS_DEFAULT` |
| `DISPATCH_QUEUE_PRIORITY_LOW` | `QOS_CLASS_UTILITY` |
| `DISPATCH_QUEUE_PRIORITY_BACKGROUND` | `QOS_CLASS_BACKGROUND` |

注意，`QOS_CLASS_USER_ACTIVE`没有与之对应的`DISPATCH_QUEUE_PRIORITY`，所以在iOS7时需要将`QOS_CLASS_USER_ACTIVE`按照`DISPATCH_QUEUE_PRIORITY_HIGH`来处理。


下面是在iOS8以下与iOS8以上，创建 `Dispatch Serial Queue` 的不同方式:

```c
static dispatch_queue_t __allocDispatchQueueCreateWithQualityOfService(XZHQualityOfService qos, NSString *queueName)
{
    
    /**
     *  iOS8，直接可以使用 QOS 创建出一个 具备优先级的 SerialQueue
     */
    if (XZH_LEAST_IOS8) {
        dispatch_queue_attr_t queue_attr = dispatch_queue_attr_make_with_qos_class (DISPATCH_QUEUE_SERIAL, (qos_class_t)qos, -1);
        return dispatch_queue_create([queueName UTF8String], queue_attr);
    }
    
    /**
     *  iOS7，需要先创建一个SerialQueue，然后复制GlobalQueue对应的优先级
     */
    dispatch_queue_t serialQueue = dispatch_queue_create([queueName UTF8String], DISPATCH_QUEUE_SERIAL);
    //TODO: iOS7时，需要将QOS转换成priority
    dispatch_queue_priority_t priority = __XZHQualityOfServiceToPriority(qos);
    dispatch_queue_t globalQueue = dispatch_get_global_queue(priority, 0);
    dispatch_set_target_queue(serialQueue, globalQueue);
    return serialQueue;
}
```

OK，关于`Serial dispsatch_queue_t`在不同系统版本下，进行创建时进行了通用性的适配。那么下面就不再关系如何去创建`dispsatch_queue_t`的问题了，该考虑如何去缓存了。

***

###前面说过一个Pool对象，内部需要有5种`QualityOfService`各自的`dispsatch_queue_t`的容器，那么此时可以封装一个单独的类来作为某一种`QualityOfService`下的`dispsatch_queue_t`的容器

YY作者直接使用了c代码，写了一个Context类来表示，可能考虑到性能的问题吧，尽量的避免使用OC去写。

```c
/**
 *  对应一个QualityOfService/DispatchQueuePriority下得缓存容器，
 *  缓存这个模式下的所有的dispatch_queue_t实例
 */
typedef struct __XZHDispatchContext {
    
    /**
     *  服务质量，值参考qos_class_t
     */
    XZHQualityOfService     qos;
    
    /**
     *  缓存的所有的dispatch_queue_t实例
     */
    void**                  queues;
    /**
     *  缓存的dispatch_queue_t实例的总个数
     */
    NSUInteger              num_queues;
    
    /**
     *  记录当前获取缓存的dispatch_queue_t实例进行调度任务的总操作次数，
     *  并且操作次数是累加的
     */
    int32_t                 workQueueOffset;
    
} __XZHDispatchContext;
```

对作者源码稍微修改了一下，我比较喜欢把一些不暴露的类、方法前面加上`__类`、`_方法名`。

可以看到一个Context实例包含的东西:

- (1) 对应哪一种`QualityOfService` 服务质量
- (2) 缓存的所有的`dispsatch_queue_t`的数组
- (3) 缓存的`dispsatch_queue_t`总数
- (4) 当前从Context缓存`dispsatch_queue_t`池中，取出`dispsatch_queue_t`使用的`次数`，这个`次数`是累加的，来计算获取哪一个index指向的`dispsatch_queue_t`

***

###现在设计下Pool这个OC类向外暴露的api

```objc
#ifndef XZHDispatchQueuePool_h
#define XZHDispatchQueuePool_h

/**
 *  代替iOS8的 NSQualityOfService与qos_class_t
 */
typedef NS_ENUM(NSInteger, XZHQualityOfService) {
    XZHQualityOfServiceUserInteractive          = 0x21,
    XZHQualityOfServiceUserInitiated            = 0x19,
    XZHQualityOfServiceUtility                  = 0x11,
    XZHQualityOfServiceBackground               = 0x09,
    XZHQualityOfServiceDefault                  = 0x00,//发现NSQualityOfServiceDefault获取不到dispatch_queue_t实例
};

/**
 * dispatch_queue_t pool 缓存池，在主线程上进行创建和初始化
 */
@interface XZHDispatchQueuePool : NSObject

/**
 *  默认创建所有QualityOfService类型的缓存队列容器
 *
 *  @param poolName   缓存池名
 *  @param queueCount 队列总数，传入小于1分配最优的线程数，否则创建指定的线程数
 *
 *  @return Pool创建工程返回Pool对象，否则返回nil
 */
- (instancetype)initWithPoolName:(NSString *)poolName queueCount:(NSInteger)queueCount;

/**
 *  释放Pool对象，以及内部缓存的所有dispatch_queue_t实例
 */
- (void)xzh_releaseDispatchQueuePool;

/**
 *  传入NSQualityOfService类型，从Pool中查询一个空闲的dispatch_queue_t
 *
 *  @param qos NSQualityOfService，因为苹果在iOS8之后推荐使用QOS来描述线程队列，
 *所以在iOS8之后优先使用QOS。但是在iOS8以下，仍然使用dispatch_queue_priority_t，
 *转换逻辑由内部控制
 *
 *  @return dispatch_queue_t实例
 */
- (dispatch_queue_t)xzh_queueFromPoolWithQualityOfService:(XZHQualityOfService)qos;

// 默认的全局单例Pool对象
+ (instancetype)xzh_defaultPool;
+ (dispatch_queue_t)xzh_defaultDispatchQueueWithQualityOfService:(XZHQualityOfService)qos;

@end
```

主要包含两种类型操作:

- (1) 自行创建Pool对象，然后从这个Pool对象中获取缓存的`dispatch_queue_t`
- (2) 直接使用`Default Pool`对象，由框架自动创建


然后是看下实现文件中，Pool对象的内部私有变量结构:

```c
@implementation XZHDispatchQueuePool {
    @package
    
    /**
     *  缓存池的名字
     */
    NSString                                        *_poolName;
    
    /**
     *  对应五种qos的Context对象的数组
     *  [0] useractive
     *  [1] userinitialed
     *  [2] ulity
     *  [3] backgroud
     *  [4] default
     */
    __XZHDispatchContext                            **_contextArray;
}
```

***

###从Pool缓存池类的初始化方法`initWithPoolName:queueCount:`继续切入，显然此时需要创建Pool对象内部持有的所有的`__XZHDispatchContext`缓存容器。


```c
- (instancetype)initWithPoolName:(NSString *)poolName queueCount:(NSInteger)queueCount {
    self = [super init];
    if (self) {
        _poolName = [_poolName copy];
        
        /**
         *  创建 __XZHDispatchContext 结构体实例数组，五种QOS，所以就有5个实例
         */
        _contextArray = __allocDispatchContextArrayCreateWithAllQualityOfService(queueCount);
        if (NULL == _contextArray) return nil;//创建失败
    }
    return self;
}
```

***

###继续进入`__allocDispatchContextArrayCreateWithAllQualityOfService()`这个c方法实现，具体如何创建`dispatch_queue_t`的缓存容器`__XZHDispatchContext 实例数组`

```c
/**
 *  创建某一个Pool对象所管理的 所有的 Context实例，每一种QualityOfService对应一个Context实例，所以最多5个Context实例
 *
 *  @param queueCount Context下对应的缓存的dispatch_queue_t实例个数
 */
static __XZHDispatchContext** __allocDispatchContextArrayCreateWithAllQualityOfService(NSInteger queueCount)
{
    /**
     *  分配一个长度为5的__XZHDispatchContext结构体实例数组，用来存放所有的QOS对应的缓存容器
     */
    __XZHDispatchContext **contextArray = (__XZHDispatchContext **)malloc(5 * sizeof(__XZHDispatchContext));
    if (NULL == contextArray) return NULL;
    
    /**
     *  1. 创建 QualityOfServiceUserInteractive Context
     */
    contextArray[0] = __allocDispatchContextCreateWithSpeciaQualityOfService(queueCount, XZHQualityOfServiceUserInteractive, @"ServiceUserInteractive");
    
    /**
     *  2. 创建 QualityOfServiceUserInitiated Context
     */
    contextArray[1] = __allocDispatchContextCreateWithSpeciaQualityOfService(queueCount, XZHQualityOfServiceUserInitiated, @"ServiceUserInitiated");
    
    /**
     *  3. 创建 QualityOfServiceUtility Context
     */
    contextArray[2] = __allocDispatchContextCreateWithSpeciaQualityOfService(queueCount, XZHQualityOfServiceUtility, @"ServiceUtility");
    
    /**
     *  4. 创建 QualityOfServiceBackground Context
     */
    contextArray[3] = __allocDispatchContextCreateWithSpeciaQualityOfService(queueCount, XZHQualityOfServiceBackground, @"ServiceBackground");
    
    /**
     *  5. 创建 QualityOfServiceDefault Context
     */
    contextArray[4] = __allocDispatchContextCreateWithSpeciaQualityOfService(queueCount, XZHQualityOfServiceDefault, @"ServiceDefault");
    
    return contextArray;
}
```

***

###继续进入`__allocDispatchContextCreateWithSpeciaQualityOfService()`这个c函数，如何具体创建一个`__XZHDispatchContext`实例


注意，这里有个小问题，就是给这个Context实例到底创建多少个`Serial dispatch_queue_t`实例？

因为一个`Serial  dispatch_queue_t实例 == 一个Thread线程对象`，所以实际上就是创建多少个线程的问题。前面已经说过了，这里按照CPU密集型，所以最优的线程数为如下:

```c
#define kXZHBestDispatchQueueCount  ([NSProcessInfo processInfo].activeProcessorCount + 1)
```

然后是创建一个`__XZHDispatchContext`实例的c函数实现:

```c
/**
 *  创建某一个QualityOfService质量下的 Context实例
 *
 *  @param queueCount queueCount Context下对应的缓存的dispatch_queue_t实例个数
 *  @param qos        QualityOfService
 *  @param queueName  Queue名字
 */
static __XZHDispatchContext* __allocDispatchContextCreateWithSpeciaQualityOfService(NSInteger queueCount, XZHQualityOfService qos, NSString *queueName)
{
    /**
     *  1. 根据当前CPU核心数创建对应的dispatch_queue_t实例个数，按照密集型CPU操作进行计算
     */
    if (queueCount < 1) queueCount = kXZHBestDispatchQueueCount;
    if (queueCount > kXZHMaxDispatchQueueCount) queueCount = kXZHMaxDispatchQueueCount;
    
    /**
     *  2. 创建并初始化一个Context实例
     */
    __XZHDispatchContext *context = (__XZHDispatchContext *)malloc(sizeof(__XZHDispatchContext));
    if (NULL == context) return NULL;
    context->queues = malloc(queueCount * sizeof(void *));
    if (NULL == context->queues) return NULL;
    context->num_queues = queueCount;
    context->qos = qos;
    context->workQueueOffset = 0;
    
    /**
     *  3. 创建这个Context下，所有缓存的dispatch_queue_t实例
     */
    for (int i = 0; i < queueCount; i++) {
        NSString *name = [NSString stringWithFormat:@"%@.%d", queueName, i+1];
        
        //【重要一】 调用前面写好的创建具体dispatch_queue_t的c函数完成创建
        dispatch_queue_t queue = __allocDispatchQueueCreateWithQualityOfService(qos, name);
#if DEBUG
        NSLog(@"创建队列 %@", queue);
#endif
        //【重要二】 类型转换时，对对象retain一次，防止废弃释放成为野指针，意味着废弃时不需要进行`(__bridge_transfer CF)(queue)` 让其release一次
        context->queues[i] = (__bridge_retained void*)(queue);
    }
    
    return context;
}
```

OK，到此为止，Pool对象持有5个Context实例，一个Context实例内部持有多个queue实例了。已经完成了在`不同的QualityOfService`下，如何的缓存queue实例了。

***

###如何完成从Pool对象内部持有的哪一个Context中，取出哪一个`dispatch_queue_t`实例了？

```objc
@interface XZHDispatchQueuePool : NSObject

/**
 *  传入NSQualityOfService类型，从Pool中查询一个空闲的dispatch_queue_t
 *
 *  @param qos NSQualityOfService，因为苹果在iOS8之后推荐使用QOS来描述线程队列，
 *所以在iOS8之后优先使用QOS。但是在iOS8以下，仍然使用dispatch_queue_priority_t，
 *转换逻辑由内部控制
 *
 *  @return dispatch_queue_t实例
 */
- (dispatch_queue_t)xzh_queueFromPoolWithQualityOfService:(XZHQualityOfService)qos;

@end
```

```objc
- (dispatch_queue_t)xzh_queueFromPoolWithQualityOfService:(XZHQualityOfService)qos {
    dispatch_queue_t queue = NULL;
    
    /**
     *  从对应QOS的Context实例中，查询queue实例
     */
     
    switch (qos) {
        case XZHQualityOfServiceUserInteractive: {
            return __XZHGetCachedDispatchQueueFromContext(_contextArray[0]);
            break;
        }
        case XZHQualityOfServiceUserInitiated: {
            return __XZHGetCachedDispatchQueueFromContext(_contextArray[1]);
            break;
        }
        case XZHQualityOfServiceUtility: {
            return __XZHGetCachedDispatchQueueFromContext(_contextArray[2]);
            break;
        }
        case XZHQualityOfServiceBackground: {
            return __XZHGetCachedDispatchQueueFromContext(_contextArray[3]);
            break;
        }
        case XZHQualityOfServiceDefault: {
            return __XZHGetCachedDispatchQueueFromContext(_contextArray[4]);
            break;
        }
    }
}
```

***

###`__XZHGetCachedDispatchQueueFromContext()`这个c函数，完成从传入的Context中查询得到一个`dispatch_queue_t`实例

```c
static dispatch_queue_t __XZHGetCachedDispatchQueueFromContext(__XZHDispatchContext *context) {
    if (NULL == context) return 0;
    dispatch_queue_t queue = 0;
    
    // 1. 使用原子性同步多线程，进行context属性值修改，计算当前使用缓存queue的操作次数
    int32_t offset = OSAtomicIncrement32(&(context->workQueueOffset));
    
    // 2. 使用计算得到的offset 取余 总缓存queue数，得到取出queue的index
    int32_t curQueueIndex = offset % context->num_queues;
    
    // 3. 从数组顺序取出一个queue，取出实例只进行类型转换
    //【重要】注意这里指针类型转换，不要使用 __bridge_transfer，否则会对指针进行release，造成程序崩溃
    queue = (__bridge id)(context->queues[curQueueIndex]);
    
    // 4. 防止queue=nil，dispatch_astync(nil, block);崩溃
    if (!queue) {
#if DEBUG
        NSLog(@"获取缓存队列失败");
#endif
        queue = dispatch_get_global_queue(0, 0);
    } else {
#if DEBUG
        NSLog(@"获取缓存队列成功: queue name = %s", dispatch_queue_get_label(queue));
#endif
    }
    
    return queue;
}
```

使用`递增使用次数 % 队列总数 = 当前要使用的队列index` ，这样保证每一次取到的都不是同一个队列，让所有缓存的`dispatch_queue_t`平分分配执行的任务。


***

###接下来就是如何释放废弃整个Pool对象，以及内部的Context实例数组，以及每一个Context实例保存的所有的`dispatch_queue_t`实例

```objc
@interface XZHDispatchQueuePool : NSObject

/**
 *  释放Pool对象，以及内部缓存的所有dispatch_queue_t实例
 */
- (void)xzh_releaseDispatchQueuePool;

@end
```

```objc
- (void)xzh_releaseDispatchQueuePool {
    if (NULL == _contextArray) return;
    
    // 依次释放5个Context实例
    __XZHDispatchContextRelease(_contextArray[0]);
    __XZHDispatchContextRelease(_contextArray[1]);
    __XZHDispatchContextRelease(_contextArray[2]);
    __XZHDispatchContextRelease(_contextArray[3]);
    __XZHDispatchContextRelease(_contextArray[4]);
}
```

调用了`__XZHDispatchContextRelease()`来释放一个具体的Context实例:

```c
static void __XZHDispatchContextRelease(__XZHDispatchContext *context)
{
    if (NULL == context) return;
    
    // 1. 【重要】使用 __bridge_transfer在进行指针类型转换时，顺便做一次与之前 __bridge_retained对应的release操作，保持retainCount平衡
    for (int i = 0; i < context->num_queues; i++) {
    	// __bridge_transfer 使用
        id queue = (__bridge_transfer dispatch_queue_t)context->queues[i];
#if DEBUG
        NSLog(@"release queue = %s", dispatch_queue_get_label(queue));
#endif
    }
    
	//2. 再对 dispatch_queue 数组进行依次释放release
    if (context->queues) free(context->queues);
    context->queues = NULL;
    
    //3. 最终对整个Context实例进行释放release
    if (context) free(context);
    context = NULL;
}
```

大部分功能就已经实现了，可以再额外添加一个使用默认的Pool对象功能。

***

###默认的Pool对象功能

```objc
@interface XZHDispatchQueuePool : NSObject

+ (instancetype)xzh_defaultPool;

+ (dispatch_queue_t)xzh_defaultDispatchQueueWithQualityOfService:(XZHQualityOfService)qos;

@end
```

```objc
+ (instancetype)xzh_defaultPool {
    static XZHDispatchQueuePool *_GlobalPool = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _GlobalPool = [[XZHDispatchQueuePool alloc] initWithPoolName:@"XZHDefaultDispatchQueuePool" queueCount:0];
    });
    return _GlobalPool;
}

+ (dispatch_queue_t)xzh_defaultDispatchQueueWithQualityOfService:(XZHQualityOfService)qos {
    return XZHGetDefaultDispatchQueueWithQualityOfService(qos);
}
```

使用了一个`XZHGetDefaultDispatchQueueWithQualityOfService()`这个c函数来完成从默认的Pool单例中获取`dispatch_queue_t`实例

```c
dispatch_queue_t XZHGetDefaultDispatchQueueWithQualityOfService(XZHQualityOfService qos) {

	//1. 获取的默认的Pool单例
    XZHDispatchQueuePool *defaultPool = [XZHDispatchQueuePool xzh_defaultPool];
    
    //2. 
    if ([XZHDispatchQueuePool xzh_defaultPool]) {
        __XZHDispatchContext *context = NULL;
        
        // 根据不同的QOS获取不同的Context队列缓存容器
        switch (qos) {
            case XZHQualityOfServiceUserInteractive: {
                context = defaultPool->_contextArray[0];
                break;
            }
            case XZHQualityOfServiceUserInitiated: {
                context = defaultPool->_contextArray[1];
                break;
            }
            case XZHQualityOfServiceUtility: {
                context = defaultPool->_contextArray[2];
                break;
            }
            case XZHQualityOfServiceBackground: {
                context = defaultPool->_contextArray[3];
                break;
            }
            case XZHQualityOfServiceDefault: {
                context = defaultPool->_contextArray[4];
                break;
            }
        }
        
        //3. 调用之前的查询Context缓存队列的方法，查询Context内部的queue实例
        dispatch_queue_t queue = __XZHGetCachedDispatchQueueFromContext(context);
        
        //4. 查询缓存queue成功
        if (queue) return queue;
    }
    
#if DEBUG
    NSLog(@"获取缓存队列失败");
#endif
	//5. 查询缓存queue失败，返回一个系统queue，防止程序崩溃
    return dispatch_get_global_queue(0, 0);
}
```

OK，默认的Pool也完成了。

***

###最后就是使用一下上面完成的Pool吧

```objc
@interface ViewController () {
    XZHDispatchQueuePool *_pool;
}

@end

@implementation ViewController

- (instancetype)init {
    self = [super init];
    if (self) {
//        _pool = [[XZHDispatchQueuePool alloc] initWithPoolName:@"TestPool" queueCount:-1];
//        _pool = [[XZHDispatchQueuePool alloc] initWithPoolName:@"TestPool" queueCount:3];
//        _pool = [[XZHDispatchQueuePool alloc] initWithPoolName:@"TestPool" queueCount:5];
//        _pool = [[XZHDispatchQueuePool alloc] initWithPoolName:@"TestPool" queueCount:25];
        _pool = [[XZHDispatchQueuePool alloc] initWithPoolName:@"TestPool" queueCount:0];
    }
    return self;
}

- (void)testPool {

	dispatch_queue_t queue1 = [_pool xzh_queueFromPoolWithQualityOfService:XZHQualityOfServiceUserInteractive];
    dispatch_async(queue1, ^{
        NSLog(@"hahahaha 1");
    });

    dispatch_queue_t queue2 = [_pool xzh_queueFromPoolWithQualityOfService:XZHQualityOfServiceUserInteractive];
    dispatch_async(queue2, ^{
        NSLog(@"hahahaha 2");
    });
    
    dispatch_queue_t queue3 = [_pool xzh_queueFromPoolWithQualityOfService:XZHQualityOfServiceUserInteractive];
    dispatch_async(queue3, ^{
        NSLog(@"hahahaha 3");
    });
    
    dispatch_queue_t queue4 = [_pool xzh_queueFromPoolWithQualityOfService:XZHQualityOfServiceUserInteractive];
    dispatch_async(queue4, ^{
        NSLog(@"hahahaha 4");
    });
    
    dispatch_queue_t queue5 = XZHGetDefaultDispatchQueueWithQualityOfService(XZHQualityOfServiceUserInteractive);
    dispatch_async(queue5, ^{
        NSLog(@"hahahaha 5");
    });
}

@end
```

- (1) 程序输出一、`_pool = [[XZHDispatchQueuePool alloc] initWithPoolName:@"TestPool" queueCount:0];`

```
2016-08-28 20:54:18.861 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceUserInteractive.1[0x7f995aec9b70]>
2016-08-28 20:54:18.862 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceUserInteractive.2[0x7f995ad269c0]>
2016-08-28 20:54:18.862 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceUserInteractive.3[0x7f995aec6230]>
2016-08-28 20:54:18.862 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceUserInteractive.4[0x7f995ae14580]>
2016-08-28 20:54:18.862 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceUserInteractive.5[0x7f995aec4cc0]>
2016-08-28 20:54:18.862 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceUserInitiated.1[0x7f995ad1e470]>
2016-08-28 20:54:18.863 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceUserInitiated.2[0x7f995ad23a40]>
2016-08-28 20:54:18.863 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceUserInitiated.3[0x7f995ad21220]>
2016-08-28 20:54:18.863 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceUserInitiated.4[0x7f995aec3420]>
2016-08-28 20:54:18.863 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceUserInitiated.5[0x7f995aec4ba0]>
2016-08-28 20:54:18.863 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceUtility.1[0x7f995ad1ae10]>
2016-08-28 20:54:18.863 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceUtility.2[0x7f995ad23530]>
2016-08-28 20:54:18.863 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceUtility.3[0x7f995ad254c0]>
2016-08-28 20:54:18.898 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceUtility.4[0x7f995ae07ab0]>
2016-08-28 20:54:18.898 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceUtility.5[0x7f995ad2c270]>
2016-08-28 20:54:18.898 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceBackground.1[0x7f995aec62e0]>
2016-08-28 20:54:18.898 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceBackground.2[0x7f995aec4980]>
2016-08-28 20:54:18.898 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceBackground.3[0x7f995ad2e790]>
2016-08-28 20:54:18.898 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceBackground.4[0x7f995ad17f50]>
2016-08-28 20:54:18.899 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceBackground.5[0x7f995ad2ad90]>
2016-08-28 20:54:18.899 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceDefault.1[0x7f995ad29e90]>
2016-08-28 20:54:18.899 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceDefault.2[0x7f995ad33570]>
2016-08-28 20:54:18.899 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceDefault.3[0x7f995af08670]>
2016-08-28 20:54:18.899 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceDefault.4[0x7f995aec4a10]>
2016-08-28 20:54:18.899 Demo[5170:72471] 创建队列 <OS_dispatch_queue: ServiceDefault.5[0x7f995ad34890]>
```

每一种context创建了`5`个队列，因为我是`模拟器`跑的，所以使用的是我Mac电脑上的CPU核心数。而我Mac电脑的CPU是`i5双核四线程`，所以`[NSProcessInfo processInfo].activeProcessorCount == 4`，然后再`+1`就是`5`咯。

- (2) 程序输出二、上面async分配的线程任务

```
2016-08-28 20:58:05.850 Demo[5170:72471] 获取缓存队列成功: queue name = ServiceUserInteractive.1
2016-08-28 20:58:05.850 Demo[5170:72471] 获取缓存队列成功: queue name = ServiceUserInteractive.2
2016-08-28 20:58:05.850 Demo[5170:76471] hahahaha 1
2016-08-28 20:58:05.850 Demo[5170:72471] 获取缓存队列成功: queue name = ServiceUserInteractive.3
2016-08-28 20:58:05.850 Demo[5170:76477] hahahaha 2
2016-08-28 20:58:05.850 Demo[5170:72471] 获取缓存队列成功: queue name = ServiceUserInteractive.4
2016-08-28 20:58:05.850 Demo[5170:76471] hahahaha 3
2016-08-28 20:58:05.850 Demo[5170:72471] 获取缓存队列成功: queue name = ServiceUserInteractive.3
2016-08-28 20:58:05.850 Demo[5170:76477] hahahaha 4
2016-08-28 20:58:05.851 Demo[5170:76477] hahahaha 5
```

也可以看到缓存队列是依次按照`offset累计使用次数 % 缓存queue总数`来按顺序获取的，最终线程任务也可以正常执行。

- (3) 程序输出三、释放废弃Pool对象

```
2016-08-28 20:59:40.724 Demo[5170:72471] release queue = ServiceUserInteractive.1
2016-08-28 20:59:40.724 Demo[5170:72471] release queue = ServiceUserInteractive.2
2016-08-28 20:59:40.724 Demo[5170:72471] release queue = ServiceUserInteractive.3
2016-08-28 20:59:40.725 Demo[5170:72471] release queue = ServiceUserInteractive.4
2016-08-28 20:59:40.725 Demo[5170:72471] release queue = ServiceUserInteractive.5
2016-08-28 20:59:40.726 Demo[5170:72471] release queue = ServiceUserInitiated.1
2016-08-28 20:59:40.726 Demo[5170:72471] release queue = ServiceUserInitiated.2
2016-08-28 20:59:40.726 Demo[5170:72471] release queue = ServiceUserInitiated.3
2016-08-28 20:59:40.726 Demo[5170:72471] release queue = ServiceUserInitiated.4
2016-08-28 20:59:40.726 Demo[5170:72471] release queue = ServiceUserInitiated.5
2016-08-28 20:59:40.726 Demo[5170:72471] release queue = ServiceUtility.1
2016-08-28 20:59:40.726 Demo[5170:72471] release queue = ServiceUtility.2
2016-08-28 20:59:40.726 Demo[5170:72471] release queue = ServiceUtility.3
2016-08-28 20:59:40.727 Demo[5170:72471] release queue = ServiceUtility.4
2016-08-28 20:59:40.727 Demo[5170:72471] release queue = ServiceUtility.5
2016-08-28 20:59:40.727 Demo[5170:72471] release queue = ServiceBackground.1
2016-08-28 20:59:40.727 Demo[5170:72471] release queue = ServiceBackground.2
2016-08-28 20:59:40.727 Demo[5170:72471] release queue = ServiceBackground.3
2016-08-28 20:59:40.727 Demo[5170:72471] release queue = ServiceBackground.4
2016-08-28 20:59:40.727 Demo[5170:72471] release queue = ServiceBackground.5
2016-08-28 20:59:40.727 Demo[5170:72471] release queue = ServiceDefault.1
2016-08-28 20:59:40.727 Demo[5170:72471] release queue = ServiceDefault.2
2016-08-28 20:59:40.728 Demo[5170:72471] release queue = ServiceDefault.3
2016-08-28 20:59:40.728 Demo[5170:72471] release queue = ServiceDefault.4
2016-08-28 20:59:40.728 Demo[5170:72471] release queue = ServiceDefault.5
```

可以看到前面创建的`5 * 5 == 25`个queue实例全部已经释放废弃了。