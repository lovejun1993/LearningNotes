---
layout: post
title: "multithread"
date: 2014-12-17 16:17:12 +0800
comments: true
categories: 
---

##操作系统启动一个App程序时

- (1) 创建一个唯一的`进程`分配给这个App程序使用
- (2) 这个进程中可能会有n多个`线程`
- (3) 最终执行代码的正是这n多个`线程`
	- 主线程、使我们用的最多的线程
	- 其他后台子线程、系统创建或我们自己创建

比如说，Xcode debug 可以attch另外一个正在运行的程序进程:

![](http://i4.piimg.com/4851/c67924e8c1862c30.png)

然后输出日志也可以得到调试App程序进程id:

```
2015-09-17 16:33:30.446 Demos[8099:63713] ........ 
```

当App程序退出结束执行后，分配的这个8099号进程就会被系统回收掉。


##并发与并行的区别

一般说的`并发`这个词，其实还有一个与之对应的单词是`并行`，可能大多数初级开发者并不知道二者有什么区别，比如我...

###并发 >>> 操作系统只使用了`一个`cpu硬件 >>> 线程

- 一个cpu在一个时间点，只能运行`一个程序进程`（QQ）
- 但一个程序`进程`内部，又包含若n个的`子线程`
- 在整个程序进程执行期间，所有的`子线程`都好像是同时执行
- 但是在某一个时间点确是按照先后顺序一个一个接着执行，只是切换的速度非常快

###并行 >>> 操作系统使用了`二个或以上`cpu硬件 >>> 进程

- 如果有`两个cpu硬件`，那么同一时间点，可以运行`两个程序进程`（QQ、FireFox浏览器）
- 两个cpu，又各自负责对应的程序进程内部的子线程的调度
	- `cpu 1` 来负责 程序A进程内 所有子线程 的切换
	- `cpu 2` 来负责 程序B进程内 所有子线程 的切换
	- 那么对于每一个cpu，其实就是上面`并发`的情况

个人觉得的话，并发应该是并行最大的区别就是: `一个CPU还是多个CPU`。然后一个主要针对是线程，另一个主要针对进程。


###目前来说苹果主流iOS移动设备，基本上都是`单cpu硬件 + 双核心`

在iOS程序开发中，`并发`是说的最多的话题，因为目前移动设备基本上都还处于`单CPU硬件`时代:

- (1) 一个程序进程在一个cpu调度运行期间
- (2) 该进程内所有的多线程看起来好像是同时执行的
- (3) 但是在一个唯一时间点时只有一个线程执行，也就是说并不是真正意义上的一个时间点同时执行多个线程的意思。

那么一个cpu硬件是如何给一个进程内所有的线程进行运行时间分配了？

- (1) cpu计算出启动的进程所占用的总运行时间
- (2) 将总时间分成若干个小时间片
- (3) 而每一个小的时间片，cpu将切换到进程内某一个线程去执行
- (4) 那么在这个时间片内，没有被cpu切换执行的线程，都会处于`挂起`状态，等待唤醒
- (5) 当某一个线程执行时间到了之后，CPU自动切换到下一个线程执行
- (6) 切换线程会参考`线程的 优先级`，来决定是否先让某一个等待的线程唤醒
	- iOS8之前，使用 `thread priority`和`dispatch queue priority` 来描述
	- iOS8以及之后，使用 `Quality Of Service` 来描述

后续的讨论都是基于`单个CPU硬件 + 多核心`基础上，即单CPU硬件下的多核心`并发多线程编程`。通常来说，比如4核心，那么最理想的`线程`并发数就是4或5。

- (1) 密集型、比较常用的CPU工作模式
- (2) IO耗时型、适用于大文件读写时的模式

##个人觉得一个线程根据其功能所划分成的种类

- (1) `非独立线程`

	依附于进程或者创建这个线程的父线程。

- (2) `独立线程`

	不依赖于其他任何线程，从创建到运行完毕之后就会释放，且只能执行一次性任务。这是用的最多的情况的线程。

- (3) `常驻后台线程`

	一个线程单例对象存在于进程，只要进程在该线程就一直存在。一般是配合runloop来做一个长时间接收事件的后台线程存在。比如，`AFURLConnectionOperation.m`中就做了一个单例的NSThread对象，负责所有的NSURLConnection事件的调度处理。


## 目前iOS平台提供多线程编程的api种类

- (1) 直接操作Thread线程对象
	-  pthread 
	-  NSThread

- (2) 使用`队列`数据结构，来屏蔽直接使用Thread线程对象
	- Grand Dispatch Queue (GCD) 任务调度队列
	- NSOperationQueue 操作队列

显然使用的更多的是(2)来操作Thread线程对象。此种情况下，一个队列的实例就相当于是一个Thread对象:

- (1) `dispatch_queue_t` 实例
- (2) NSOperationQueue 实例


创建如上两种类型的一个实例，就相当于创建了一个Thread对象。

###下面是pthread创建使用多线程的demo

```c
#import <pthread.h>
#include <limits.h>

void createThread()
{
    pthread_t       thread_id;//线程id
    pthread_attr_t  thread_attr;//线程属性
    size_t          stack_size;//线程栈大小
    int             status;//创建线程结果
    
    status = pthread_attr_init(&thread_attr);
    
    if (status != 0) {
        perror("创建线程属性失败!\n");
    }
    
    status = pthread_attr_setdetachstate (&thread_attr, PTHREAD_CREATE_DETACHED);
    
    if (status != 0) {
        perror("创建线程属性失败!\n");
    }
    
    // 获取当前线程的 栈大小
    status = pthread_attr_getstacksize(&thread_attr, &stack_size);
    
    if (status != 0) {
        perror("获取线程的栈空间大小失败!\n");
    }
    
    printf ("Default stack size is %zu; minimum is %u\n",
            stack_size, PTHREAD_STACK_MIN);

    // 设置当前的线程的栈大小
    status = pthread_attr_setstacksize(&thread_attr, PTHREAD_STACK_MIN * 1024);
    
    if (status != 0) {
        perror("设置线程的栈空间大小失败!\n");
    }
    
    // 获取当前线程的 栈大小
    status = pthread_attr_getstacksize(&thread_attr, &stack_size);
    
    if (status != 0) {
        perror("获取线程的栈空间大小失败!\n");
    }
    
    printf ("Default stack size is %zu; minimum is %u\n",
            stack_size, PTHREAD_STACK_MIN);
    
    // 按照之前的设置，创建线程
    status = pthread_create(&thread_id, &thread_attr, &threadPoint, NULL);
    
    if (status != 0) {
        perror("创建线程失败!\n");
    }
    
    
}

//线程入口函数
void* threadPoint(void* ptr)
{
    //do somethings ..
    return NULL;
}
```

###下面是NSThread创建使用多线程模拟卖票

```objc
@interface ViewController () {
    
    int tickets;//总票数
    int count;//售出票数
    
    NSThread* ticketsThreadone;
    NSThread* ticketsThreadtwo;
    NSThread* ticketsThreadthree;
    NSThread* ticketsThreadfour;
    
    NSCondition *conditon;
}
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];

	//1.
    tickets = 10;
    count = 0;
    
    //3.
    conditon = [[NSCondition alloc] init];
    
    //4.
    ticketsThreadone = [[NSThread alloc] initWithTarget:self selector:@selector(threadEntry) object:nil];
    [ticketsThreadone setName:@"Thread-1"];
    [ticketsThreadone start];
    
    //5.
    ticketsThreadtwo = [[NSThread alloc] initWithTarget:self selector:@selector(threadEntry) object:nil];
    [ticketsThreadtwo setName:@"Thread-2"];
    [ticketsThreadtwo start];
    
    //6.
    ticketsThreadthree = [[NSThread alloc] initWithTarget:self selector:@selector(threadEntry) object:nil];
    [ticketsThreadthree setName:@"Thread-3"];
    [ticketsThreadthree start];
    
    //7. 用于唤醒前面三个子线程的 工作线程
    ticketsThreadfour = [[NSThread alloc] initWithTarget:self selector:@selector(threadNotify) object:nil];
    [ticketsThreadfour setName:@"Thread-4"];
    [ticketsThreadfour start];
}

- (void)threadEntry{
    
//    NSLog(@"thredEntry thread: %@", [NSThread currentThread].name);
    
    while (TRUE) {
    
        //让所有线程阻塞并排队等待信号
        [conditon lock];
        [conditon wait];
        
        if(tickets > 0){
            
            [NSThread sleepForTimeInterval:0.09];
            count++;
            tickets--;
            NSLog(@"当前票数是:%d,售出:%d,线程名:%@",tickets,count,[[NSThread currentThread] name]);
        }else{
            break;
        }
        
        [conditon unlock];
    }
}

-(void)threadNotify {
    
//    NSLog(@"threadNotify thread: %@", [NSThread currentThread].name);
    
    while (YES) {
        [conditon lock];
        [conditon signal];//发送一个信号
        [conditon unlock];
    }
}

@end
```

使用NSThread创建线程时需要注意的地方:

- (1) 手动创建的新的NSThread实例，默认是不会打开NSRunloop运行回环，即只能完成一次任务，然后就会被释放废弃

- (2) 不会自动创建 autoreleasepool ，那么可能会造成内存泄露。最好使用`@autoreleasepool{ ..代码.. } `来包裹线程的整个入口函数实现

###复杂度排名: pthread > NSThread > NSOperationQueue > GCD

个人觉得GCD是最简单易用的，且看很多代码基本上都是使用GCD。因为很明显使用`pthread、NSThread `创建线程、多线程同步等等都是比较麻烦的，pthread比NSthread更麻烦。

而NSOperationQueue本是基于GCD封装出来的，但是好像很少有人去使用，我也是觉得有点好笑，那么相比GCD与NSOoperationQueue:

- GCD与NSOoperationQueue的共同点:
	- 都屏蔽了直接使用pthread、NSThread来创建使用多线程
	- 都提供了`队列`数据结构让开发者只需要往里面扔需要执行的线程任务代码即可
	- 线程统一由iOS底层系统api完成: 线程创建、线程释放、线程废弃。即开发者无序再解除线程这个东西了
	- 并且GCD底层实现了`线程复用`的机制（并发线程队列）

- GCD有的优点NSOperationQueue全都有，那为什么用的人却少？
	- 一般App代码中不需要太复杂的线程之间关系，直接使用GCD的异步api即可完成任务
	- 对于NSOperationQueue提供GCD没有的功能
		- 随便取消任务（`iOS8`之后GCD也提供了取消的功能）
		- 任务之间的依赖关系
		- 随时查看任务的状态（isExecuteing、isCanceled、isFinished）
		- 自定义NSOperation子类完成更复杂的线程任务
			- (1) 继承自NSOperation实现一个子类，比如:AFURLConnectionOperation 
			- (2) 重写`start`方法实现，完成线程执行的任务代码
			- (3) 重写几个状态改变方法实现，手动发出KVO通知
				- 是否正在执行（isExecuted）
				- 是否结束（isFinished）
				- 是否取消（isCanceld）


大部分App业务层代码并不需要太过复杂的线程代码，所以一般情况下都是优先使用GCD。但是对于像需要完成AFNetworking这样太个性化的多线程任务代码时候，就需要使用NSOperationQueue和自定义NSOperation了。

##NSOperationQueue是否可以取消执行正在运行的NSOperation？

```objc
@interface ViewController ()

@property (nonatomic, strong)NSOperationQueue *operationQueue;
@property (nonatomic, strong)NSOperation *operation;

@end
```

####代码1: 同步执行operation

```
- (void)viewDidLoad {
    [super viewDidLoad];
    
	//1. 创建一个operation，死循环输出信息，用来测试是否被取消掉
	_operation = [NSBlockOperation blockOperationWithBlock:^{
	    
	    while (1) {
	        NSLog(@"operation正在执行...\n");
	    }
	}];
	    
	//2. 直接执行operation的start方法，直接就在当前主线程开始执行，并不会创建新的线程
	[_operation start];
	    
	//3. 延迟执行取消operation的方法
	[self performSelector:@selector(cancelOperation) withObject:nil afterDelay:3.f];
	
}

//4. 取消operation的函数
- (void)cancelOperation {
    [_operation cancel];
}
```

如上结果: `一直打印，没有停止`。

步骤分析:

- `operation`在当前主线程执行，而`performSelector`代码也是在主线程上执行
- operation的工作代码，是一段死循环，所以会一直卡住所在线程（主线程）.
- 那么就导致，无法执行`[_operation start];`这句代码`后面`的代码
- 所以流程就根本不会走到`cancelOperation`方法实现中去，那么operation根本取消不了.
- 所以说这种情况下，operation根本没有执行取消方法，因为没有根本取消执行.

####代码2: 异步执行operation

```
- (void)viewDidLoad {
    [super viewDidLoad];
	
	//1.
    _operationQueue = [[NSOperationQueue alloc] init];
    
    //2.
    _operation = [NSBlockOperation blockOperationWithBlock:^{
        while (1) {
            NSLog(@"operation正在执行...\n");
        }
    }];
    
    //3.
    [_operationQueue addOperation:_operation];
    
    //4.
    [self performSelector:@selector(cancelOperation) withObject:nil afterDelay:3.f];
}

//4. 取消operation的函数
- (void)cancelOperation {
    [_operation cancel];
}

```

如上结果: `还是一直在打印，没有停止.`

步骤分析:

- 现在解决了`[operation start]`实现执行和`performSelector`实现执行，分别处于不同的线程上
- 断点到`cancelOperation方法`，发现该函数已经被执行了，但是还是在打印.
- 已经执行了`-[NSOperation cancel]`，但是仍然还是在打印，为什么？
- 找了一下相关资料，是这样解释的
	- operation确实已经被cancel执行了
	- 但是这个operation被分配的内存资源有可能一段时间内无法回收。
	- 就造成取消执行operation后，但是其入口函数代码依然在执行，让我们误认为取消不了正在执行的operaton的错误理解
- 所以解决执行`-[NSOperation cancel]`方法后，operation的内存资源暂时无法回收继而继续执行的方法:
	- operation的任务代码中，需要判断当前`operation的状态`是否已经被cancel

####代码3: 异步执行operation + operation状态判断

```
- (void)viewDidLoad {
    [super viewDidLoad];
    
	//1.
    _operationQueue = [[NSOperationQueue alloc] init];
    
    //2.【重要】operation中的任务代码，需要判断当前operation的状态，做不同的处理
    __weak ViewController *wself = self;
    _operation = [NSBlockOperation blockOperationWithBlock:^{
    
    	//【重要】: 只有当operation没有执行cancel取消，才执行我们的任务代码
        while (wself.operation.isCancelled == NO) {
            NSLog(@"operation正在执行...\n");
        }
    }];
    
    //3.
    [_operationQueue addOperation:_operation];
    
    //4.
    [self performSelector:@selector(cancelOperation) withObject:nil afterDelay:3.f];
}

//4. 取消operation的函数
- (void)cancelOperation {
    [_operation cancel];
}
```

总结`取消执行正在执行的operation`:

- 需要`异步`执行的operation.
- operaiton任务代码中，需要`判断operation当前的状态`来进行分情况处理

这样当`operation 执行 cancel方法之后`，才会取消执行任务代码.

	
##pthread具体使用

POSIX是一套基于c语言的Api，使用POSIX API编写的多线程代码，是可以很容易的做到跨平台.


在使用`pthread_create()`创建一个`分离线程`时偶尔产生的严重问题

- 大多时的正常情况:
	- 使用`pthread_create()`创建一个线程.
	- 然后接收`pthread_create()`创建完毕得到的新线程实例.
	- 线程开始执行.
	- 执行完毕.

- 偶尔发生的非正常情况:
	- 使用`pthread_create()`创建返回一个线程时.
	- 然而这个线程执行速度很快，没等到`pthread_create()`返回创建的线程时，这个线程已经执行完毕并且释放了.
	- 而此时这个释放掉的线程占用的`线程Id`和`系统资源`很有可能`移交给其他线程`使用，
	- 这样一来`pthread_create()`返回的`线程Id`并不是`新创建的线程的Id`，而是其他已经存在的线程的Id与资源.

- 使用`pthread_cond_timewait()`函数解决上述在创建线程时，发生线程已经执行完毕的情况.


###示例1，创建一个`分离(detach)线程`，自己管理自己的声明周期

分离线程（detached），靠自己.

```
1. `一个分离的线程是不能被其他线程回收或杀死的.`
2. `它的存储器资源在线程自己终止时，由系统自动释放.`
```


```objc
void createPthread()
{
    //1. 线程属性（调度策略、优先级等）都在这里设置，如果为NULL则表示用默认属性
    pthread_attr_t attr;
    
    //2. 返回 创建线程id
    pthread_t posixThreadId;
    
    //3. 保存线程创建是否成功（0:成功，-1:失败）
    int returnVal;
    
    //4. 初始化线程属性
    returnVal = pthread_attr_init(&attr);
    
    //5. 如果线程创建失败
    assert(!returnVal);
    
    //5. 设置要创建出的线程 是 `分离线程`
    returnVal = pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
    
    //6. 创建出一个线程，传入参数:线程id、线程属性、线程入口函数
    int threadError = pthread_create(&posixThreadId, &attr, &threadFunc, NULL);
    
    //7.【重要】 不再使用线程属性，将其销毁
    returnVal = pthread_attr_destroy(&attr);
    
    assert(!returnVal);
    
    if (threadError != 0) {
        //线程创建错误
    }
}

//线程的入口函数
void* threadFunc(void* params)
{
    NSLog(@"%@", [NSThread currentThread]);
    
    return NULL;
}

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    createPthread();
}

@end
```

###示例2，创建一个`可结合(joinable)线程`，由创建这个线程的所在线程管理生命周期

类型二、可结合线程（joinable），靠别人.

```
1. `这个线程是由其他线程创建出来的.`
2. `只能被创建他的线程等待终止.`
3. `在被其他线程回收之前，它的存储器资源（如栈）是不释放的.`
4. `所在线程会一直阻塞，等待joinable线程执行完毕.`
```

> join()方法，使其他线程等待当前线程终止.

```objc
#import "ViewController.h"
#import <pthread.h>

// 打印线程信息
//static void printids()
//{
//    pid_t       pid;
//    pthread_t   tid;
//    
//    pid = getpid();//线程id
//    tid = pthread_self();//线程
//    
//    printf("pid %u tid %u (0x%x)\n", (unsigned int)pid,
//           (unsigned int)tid, (unsigned int)tid);
//}

void *thr_fn1(void *arg)
{
    printf("thread 1 returning.\n");
    return((void *)1);
}

void *thr_fn2(void *arg)
{
    printf("thread 2 exiting.\n");
    return((void *)2);
}

@implementation ViewController

- (void)exam1 {
    
    pthread_t tid1,tid2;
    
    void *tret;
    
    // 创建两个默认线程（joinable）
    pthread_create(&tid1, NULL, thr_fn1, NULL);
    pthread_create(&tid2, NULL, thr_fn2, NULL);
    
    NSLog(@"%@", [NSThread currentThread]);
    
    //【重要】在当前主线程上，等待 tid1 执行结束
    pthread_join(tid1, &tret);
    printf("thread 1 exit code %d\n",(int)tret);
    
    NSLog(@"%@", [NSThread currentThread]);
    
    //【重要】在当前主线程上，等待 tid2 执行结束
    pthread_join(tid2,&tret);
    printf("thread 2 exit code %d\n",(int)tret);
    
    NSLog(@"%@", [NSThread currentThread]);
    
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self exam1];
    
}

@end
```

输出信息

```
thread 1 returning.
thread 2 exiting.
2016-06-29 18:06:51.678 demo[9887:1715916] <NSThread: 0x7f80e2405b20>{number = 1, name = main}
thread 1 exit code 1
2016-06-29 18:06:51.679 demo[9887:1715916] <NSThread: 0x7f80e2405b20>{number = 1, name = main}
thread 2 exit code 2
2016-06-29 18:06:51.679 demo[9887:1715916] <NSThread: 0x7f80e2405b20>{number = 1, name = main}
```

###示例3，可结合线程之间，使用`pthread_exit()`函数结束线程并回传值的例子

```objc
#import "ViewController.h"
#import <pthread.h>

void thread1(char s[])
{
    printf("This is a pthread1.\n");
    printf("%s\n",s);
    pthread_exit("Hello first!");  //【重要】结束线程，返回一个值。
}

void thread2(char s[])
{
    printf("This is a pthread2.\n");
    printf("%s\n",s);
    pthread_exit("Hello second!");//【重要】结束线程，返回一个值。
}

@implementation ViewController

- (void)exam2 {
    
    pthread_t id1,id2;
    void *a1,*a2;
    int ret1,ret2;
    
    char s1[] = "传递给thread1的参数";
    char s2[] = "传递给thread2的参数";
    
    ret1 = pthread_create(&id1, NULL, (void *) thread1, s1);
    ret2 = pthread_create(&id2, NULL, (void *) thread2, s2);
    if (ret1 != 0 || ret2 != 0) {
        // 创建线程失败
        return;
    }
    
    // 在当前主线等待 线程1 执行完毕
    pthread_join(id1,&a1);
    printf("接收到线程1 的返回值: %s\n",(char*)a1);
    
    // 在当前主线等待 线程2 执行完毕
    pthread_join(id2,&a2);
    printf("接收到线程2 的返回值: %s\n",(char*)a2);
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
//    [self exam1];
    [self exam2];
}

@end
```

输出信息

```
This is a pthread1.
传递给thread1的参数
This is a pthread2.
传递给thread2的参数
接收到线程1 的返回值: Hello first!
接收到线程2 的返回值: Hello second!
```

###在Cocoa程序上面使用POSIX线程注意

- 在Cocoa框架上，使用POSIX线程，Cocoa框架默认是不会创建锁机制去确保线程安全的.

- 为了让 Cocoa 知道你的程序正在使用用多线程，让Cocoa框架确保线程安全，那么可以这么做`生成一个NSThread线程，可以立即退出，线程的主体入口点不需要做任何事情.`这样后，Cocoa框架就会保证多线程安全执行.

- 检测程序是否是多线程环境? `+[NSThread isMultiThreaded]`来查看

- 不要混合 `POSIX` 和 `Cocoa` 的锁. `用POSIX API创建出的锁，就必须使用POSIX API操作这个锁。如果是Cocoa API创建出的锁，就必须使用Coacoa API 操作这个锁.`

###读取和修改，线程的`堆栈`空间大小

- 任何声明为 `static/extern` 的变量，可以被`进程内所有的线程`读与写. 

- 每个线程可以通过`暴露自己的堆栈地址`的方法，让`其他线程`进行共享自己的堆栈.

- 如果当前线程要执行一个`大数据量`的任务，那么就有必要给创建出的线程分配一个比较大的内存块.

- 但是线程`栈空间`的最大值，在线程创建的时刻就已经被固定了。如果访问超过栈空间地址，就会出现访问未授权的内存区域的错误.

下面给出demo示例代码，获取系统默认线程的栈大小、修改线程栈的默认大小、然后创建修改属性后的线程.

```objc
#import <pthread.h>
#include <limits.h>	一定要导入这个头文件

void* threadPoint(void* ptr)
{
    
    //线程入口函数
    
    return NULL;
}

void createThread()
{	
	//1. 线程Id
    pthread_t thread_id;
    
    //2. 线程属性
    pthread_attr_t thread_attr;
    
    //3. 保存线程栈大小的变量
    size_t stack_size;
    
    //4. 保存每一次POSIX Api操作的结果值（0:成功，-1:失败）
    int status;
    
    //5. 初始化线程属性
    status = pthread_attr_init(&thread_attr);
    
    if (status != 0) {
        perror("创建线程属性失败!\n");
    }
    
    //6. 设置要创建出的线程是【分离线程】
    status = pthread_attr_setdetachstate (&thread_attr, PTHREAD_CREATE_DETACHED);
    
    if (status != 0) {
        perror("创建线程属性失败!\n");
    }
    
    //7. 获取当前线程默认设置栈大小
    status = pthread_attr_getstacksize(&thread_attr, &stack_size);
    
    if (status != 0) {
        perror("获取线程的栈空间大小失败!\n");
    }
    
    printf ("Default stack size is %zu; minimum is %u\n",
            stack_size, PTHREAD_STACK_MIN);

    //8. 手动修改系统默认分配的栈栈大小
    status = pthread_attr_setstacksize(&thread_attr, PTHREAD_STACK_MIN * 1024);
    
    if (status != 0) {
        perror("设置线程的栈空间大小失败!\n");
    }
    
    //9. 获取修改之后的线程栈大小
    status = pthread_attr_getstacksize(&thread_attr, &stack_size);
    
    if (status != 0) {
        perror("获取线程的栈空间大小失败!\n");
    }
    
    printf ("Default stack size is %zu; minimum is %u\n",
            stack_size, PTHREAD_STACK_MIN);
    
    //10. 按照之前的设置，创建线程
    status = pthread_create(&thread_id, &thread_attr, &threadPoint, NULL);
    
    if (status != 0) {
        perror("创建线程失败!\n");
    }
    
    
}
```

###每个pthread线程都有自己的临时存储值得一个字典对象threadDictionary

- `NSThread`实例也有一个`threadDictionary`可变字典属性

```
NSThread *t = [[NSThread alloc] init];
    
[t.threadDictionary setObject:@"value" forKey:@"key"];

```

- POSIX线程，通过如下三个c函数

```objc
pthread_key_create()	// 创建字典
pthread_setspecific()	// 设置值到字典
pthread_getspecific()	// 从字典获取值
```

```objc
#include "Demo3.h"
#include<stdio.h>
#include<pthread.h>
#include<string.h>


//声明一个key
pthread_key_t p_key;

void *thread_func(void *args)
{
    
    //不同线程的读取自己的私有存储器中的p_key对应的值
    pthread_setspecific(p_key,args);
    
    //输出
    int *tmp = (int*)pthread_getspecific(p_key);
    printf("%d is runing in %s\n",*tmp,__func__);
    
    //修改p_key对应的值
    *tmp = (*tmp)*100;
    
    //输出
    int *tmp2 = (int*)pthread_getspecific(p_key);
    printf("%d is runing in %s\n",*tmp2,__func__);
    
    return (void*)0;
}

int main()
{
    //两个线程Id
    pthread_t pa, pb;
    
    //各自的参数
    int a=1;
    int b=2;
    
    //创建线程的私有存储有一个p_key
    pthread_key_create(&p_key,NULL);
    
    //创建线程
    pthread_create(&pa, NULL,thread_func,&a);
    pthread_create(&pb, NULL,thread_func,&b);
    
    //等待线程执行结束
    pthread_join(pa, NULL);
    pthread_join(pb, NULL);
    
    return 0;
}
```

##GCD具体使用及细节

###一个`dispatch_queue_t`与一个`线程`的关系

![](http://i1.piimg.com/4851/42558425db6fbe0d.png)

可以看到`dispatch_queue_t`最终还是直接操作Thread完成线程的事情。并且底层所使用的Thread线程还做了`缓存池`的性能优化。

###`dispatch_queue_t`类型划分

- (1) serial queue 串行队列
	- (1.1) dispatch main serial queue 系统创建
	- (1.2) dispatch serial queue 开发者自己创建

- (2) concurrent queue 并发队列
	- (2.1) dispatch gloabel concurrent queue 系统创建，然后根据线程的优先级/QOS，又分为如下几种:
		- `high` priority/qos queue
		- `default` priority/qos queue
		- `low` priority/qos queue
		- `backgroud` priority/qos queue
	- (2.2) dispatch concurrent queue 开发者自己创建
- (2) 全局派发队列 global dispatch queue 按照`线程执行权限` 细分为四种
	

具体细分的话，可以分为7种队列。

###一个`serial dispatch queue`实例可以看做是`一个`线程。而对于一个`concurrent dispatch queue`实例则可以看做是`多个`线程。

![](http://i1.piimg.com/4851/2427128a9d1662ac.png)

- (1) serial dispatch queue: 

	- 从队列中依次按照顺序取出头部的`dispatch_block_t`实例(block任务)
	- 都是在`同一个`线程上`依次顺序`执行完毕。当一个任务在线程上执行完毕之后，才去开始执行下一个任务。

- (2) concurrent dispatch queue: 
	
	- 可以指定一个最大并发数，
	- 从队列中`随机`取出几个（`dispatch_block_t`实例)，执行顺序没有先后之分
	- 每一个`dispatch_block_t`，可能分配到`不同的`线程上去执行

一般其实很少自己去创建`concurrent dispatch queue`，用的最多的可能还是`serial dispatch queue`，比如:

- (1) dispatch main queue 操作UI对象
- (2) 自己创建一个`单例 serial dispatch queue`来完成一些大任务又需要多线程同步

###`dispatch_queue_create() == [Thread alloc]`

- (1) serial dispatch queue 会一直`重复使用一个唯一`的线程去执行任务，不再创建第二个子线程

- (2) 所以就跟无限制创建线程对象一样，也需要避免无限制创建`serial dispatch queue`实例

- (3) 比如、YYDispatchQueuePool实现了一个`serial dispatch queue`实例的缓存池，来管理所有的`serial dispatch queue`实例
	- 框架实例化时，根据cpu核心数创建一定数量的`serial dispatch queue`实例并缓存起来
	- 从缓存中`随机`取出一个`serial dispatch queue`实例来调度block任务
	- 适用于`并发无序`的异步线程任务代码

###在主线程上`disatch_sync(并发队列, ^(){...});`

```objc
@implementation ViewController
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	NSLog(@"*******************开始*******************");
	
	dispatch_queue_t queue = dispatch_queue_create("用户队列", DISPATCH_QUEUE_CONCURRENT);
	
	dispatch_sync(queue, ^{
	    NSLog(@"Block任务1, 所在Thread: %@\n", [NSThread currentThread]);
	});
	
	NSLog(@"*******************任务1执行结束*******************");
	
	dispatch_sync(queue, ^{
	    NSLog(@"Block任务2, 所在Thread: %@\n", [NSThread currentThread]);
	});
	
	NSLog(@"*******************任务2执行结束*******************");
	
	dispatch_sync(queue, ^{
	    NSLog(@"Block任务3, 所在Thread: %@\n",[NSThread currentThread]);
	});
	
	NSLog(@"*******************任务3执行结束*******************");
	
	dispatch_sync(queue, ^{
	    NSLog(@"Block任务4, 所在Thread: %@\n",[NSThread currentThread]);
	});
	
	NSLog(@"*******************任务4执行结束*******************");
	
	dispatch_sync(queue, ^{
	    NSLog(@"Block任务5, 所在Thread: %@\n",[NSThread currentThread]);
	});
	
	NSLog(@"*******************任务5执行结束*******************");
}
@end
```

运行结果

```
2016-07-03 15:52:22.485 Demos[8985:119037] *******************开始*******************
2016-07-03 15:52:22.485 Demos[8985:119037] Block任务1, 所在Thread: <NSThread: 0x7f9301405540>{number = 1, name = main}
2016-07-03 15:52:22.486 Demos[8985:119037] *******************任务1执行结束*******************
2016-07-03 15:52:22.486 Demos[8985:119037] Block任务2, 所在Thread: <NSThread: 0x7f9301405540>{number = 1, name = main}
2016-07-03 15:52:22.486 Demos[8985:119037] *******************任务2执行结束*******************
2016-07-03 15:52:22.486 Demos[8985:119037] Block任务3, 所在Thread: <NSThread: 0x7f9301405540>{number = 1, name = main}
2016-07-03 15:52:22.486 Demos[8985:119037] *******************任务3执行结束*******************
2016-07-03 15:52:22.486 Demos[8985:119037] Block任务4, 所在Thread: <NSThread: 0x7f9301405540>{number = 1, name = main}
2016-07-03 15:52:22.486 Demos[8985:119037] *******************任务4执行结束*******************
2016-07-03 15:52:22.487 Demos[8985:119037] Block任务5, 所在Thread: <NSThread: 0x7f9301405540>{number = 1, name = main}
2016-07-03 15:52:22.487 Demos[8985:119037] *******************任务5执行结束*******************
```

可以看到，虽然是将这些block任务`disaptch_sync`到了自己创建的线程队列中，但是最终block执行还是在当前执行`disaptch_sync`操作的主线程上。

且`disaptch_sync`到一个并发队列中，仍然是按照顺序一个一个执行的。

###将如上代码放在在子线程上执行了?

```objc
@implementation ViewController
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

	// 将所有的dispatch_sync操作放到一个子线程上
    dispatch_async(dispatch_get_global_queue(0, 0), ^() {
        NSLog(@"*******************开始*******************");
        
        // 创建一个新的线程队列
        dispatch_queue_t queue = dispatch_queue_create("用户队列", DISPATCH_QUEUE_CONCURRENT);
        
        dispatch_sync(queue, ^{
            NSLog(@"Block任务1, 所在Thread: %@\n", [NSThread currentThread]);
        });
        
        NSLog(@"*******************任务1执行结束*******************");
        
        dispatch_sync(queue, ^{
            NSLog(@"Block任务2, 所在Thread: %@\n", [NSThread currentThread]);
        });
        
        NSLog(@"*******************任务2执行结束*******************");
        
        dispatch_sync(queue, ^{
            NSLog(@"Block任务3, 所在Thread: %@\n",[NSThread currentThread]);
        });
        
        NSLog(@"*******************任务3执行结束*******************");
        
        dispatch_sync(queue, ^{
            NSLog(@"Block任务4, 所在Thread: %@\n",[NSThread currentThread]);
        });
        
        NSLog(@"*******************任务4执行结束*******************");
        
        dispatch_sync(queue, ^{
            NSLog(@"Block任务5, 所在Thread: %@\n",[NSThread currentThread]);
        });
        
        NSLog(@"*******************任务5执行结束*******************");
    });
}
@end
```

运行结果

```
2016-11-16 13:39:36.986 Demo[12546:3358613] *******************开始*******************
2016-11-16 13:39:36.987 Demo[12546:3358613] Block任务1, 所在Thread: <NSThread: 0x17407bd40>{number = 2, name = (null)}
2016-11-16 13:39:36.988 Demo[12546:3358613] *******************任务1执行结束*******************
2016-11-16 13:39:36.988 Demo[12546:3358613] Block任务2, 所在Thread: <NSThread: 0x17407bd40>{number = 2, name = (null)}
2016-11-16 13:39:36.988 Demo[12546:3358613] *******************任务2执行结束*******************
2016-11-16 13:39:36.988 Demo[12546:3358613] Block任务3, 所在Thread: <NSThread: 0x17407bd40>{number = 2, name = (null)}
2016-11-16 13:39:36.989 Demo[12546:3358613] *******************任务3执行结束*******************2016-11-16 13:39:36.989 Demo[12546:3358613] *******************任务4执行结束*******************
2016-11-16 13:39:36.990 Demo[12546:3358613] Block任务5, 所在Thread: <NSThread: 0x17407bd40>{number = 2, name = (null)}
2016-11-16 13:39:36.990 Demo[12546:3358613] *******************任务5执行结束*******************
```

```objc
@implementation ViewController
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    dispatch_async(dispatch_get_global_queue(0, 0), ^() {
        NSLog(@"*******************开始*******************");
        
        // 创建一个新的线程队列
        dispatch_queue_t queue = dispatch_queue_create("用户队列", DISPATCH_QUEUE_SERIAL);
        
        dispatch_sync(queue, ^{
            NSLog(@"Block任务1, 所在Thread: %@\n", [NSThread currentThread]);
        });
        
        NSLog(@"*******************任务1执行结束*******************");
        
        dispatch_sync(queue, ^{
            NSLog(@"Block任务2, 所在Thread: %@\n", [NSThread currentThread]);
        });
        
        NSLog(@"*******************任务2执行结束*******************");
        
        dispatch_sync(queue, ^{
            NSLog(@"Block任务3, 所在Thread: %@\n",[NSThread currentThread]);
        });
        
        NSLog(@"*******************任务3执行结束*******************");
        
        dispatch_sync(queue, ^{
            NSLog(@"Block任务4, 所在Thread: %@\n",[NSThread currentThread]);
        });
        
        NSLog(@"*******************任务4执行结束*******************");
        
        dispatch_sync(queue, ^{
            NSLog(@"Block任务5, 所在Thread: %@\n",[NSThread currentThread]);
        });
        
        NSLog(@"*******************任务5执行结束*******************");
    });
}
@end
```

运行结果

```
2016-11-16 13:40:39.743 Demo[12554:3358997] *******************开始*******************
2016-11-16 13:40:39.744 Demo[12554:3358997] Block任务1, 所在Thread: <NSThread: 0x170074540>{number = 3, name = (null)}
2016-11-16 13:40:39.744 Demo[12554:3358997] *******************任务1执行结束*******************
2016-11-16 13:40:39.745 Demo[12554:3358997] Block任务2, 所在Thread: <NSThread: 0x170074540>{number = 3, name = (null)}
2016-11-16 13:40:39.745 Demo[12554:3358997] *******************任务2执行结束*******************
2016-11-16 13:40:39.746 Demo[12554:3358997] Block任务3, 所在Thread: <NSThread: 0x170074540>{number = 3, name = (null)}
2016-11-16 13:40:39.747 Demo[12554:3358997] *******************任务3执行结束*******************
2016-11-16 13:40:39.748 Demo[12554:3358997] Block任务4, 所在Thread: <NSThread: 0x170074540>{number = 3, name = (null)}
2016-11-16 13:40:39.748 Demo[12554:3358997] *******************任务4执行结束*******************
2016-11-16 13:40:39.749 Demo[12554:3358997] Block任务5, 所在Thread: <NSThread: 0x170074540>{number = 3, name = (null)}
```

可以看到使用`dispatch_sync(queue, block);`时，不管传入的queue是 `串行队列` or `无序队列`:

- (1) `不会创建新的子线程`来执行block
	- 就是不会使用传入的queue所对应的线程，去执行传入的block
	- 而是直接在当前线程执行block
- (2) 只是将block临时放到这个queue的`末尾`进行排队，执行的时候再从这个队列取出来
- (3) block执行线程是在执行`dispatch_sync(queue, block);`操作所在的当前线程上
- (4) 必须走完`dispatch_sync(queue, block);` 中的block代码，才会走后面的代码，常常也可以用于多线程同步

如果只是一些内存数据的同步访问，可以使用`dispatch_sync(queue, block);`。但如果对于一些大文件的读写代码，还是要避免使用`dispatch_sync(queue, block);`，否则会造成线程卡顿的，最好还是使用`dispatch_async(queue, block);`异步的形式。


###`dispatch_sync(queue, block);`造成死锁的代码

```objc
@implementation ViewController

// 在主线程上执行
- (void)viewDidLoad {
    [super viewDidLoad];

    NSLog(@"1");
    
    // 将NSLog(@"2");这句代码，放到主线程队列的队尾最后执行
    dispatch_sync(dispatch_get_main_queue(), ^{
        NSLog(@"2");
    });
    
    // 又因为上面使用的是sync，表示会等待上面的block任务（NSLog(@"2");）执行完毕，才会开始执行
    NSLog(@"3");
}

@end
```

输出结果

```
2016-07-03 16:01:03.621 Demos[9498:128820] 1
```

只输出了1，但是2和3都没有被执行，这是为什么了？其实这就是主线程发生死锁了，已经被卡主了，无法继续往下执行。

解决的办法就是必须让执行`dispatch_sync(queue, block)`的线程与传入的`queue`代表的线程`不一样`。

###`disatch_async(concurrentQueue, block)`

```objc
dispatch_queue_t queue = dispatch_queue_create("用户队列", DISPATCH_QUEUE_CONCURRENT);

NSLog(@"1");

dispatch_async(queue, ^{
    NSLog(@"Block任务1, 所在Thread: %@\n", [NSThread currentThread]);
});

NSLog(@"2");

dispatch_async(queue, ^{
    NSLog(@"Block任务2, 所在Thread: %@\n", [NSThread currentThread]);
});

NSLog(@"3");

dispatch_async(queue, ^{
    NSLog(@"Block任务3, 所在Thread: %@\n",[NSThread currentThread]);
});

NSLog(@"4");

dispatch_async(queue, ^{
    NSLog(@"Block任务4, 所在Thread: %@\n",[NSThread currentThread]);
});

dispatch_async(queue, ^{
    NSLog(@"Block任务5, 所在Thread: %@\n",[NSThread currentThread]);
});

dispatch_async(queue, ^{
    NSLog(@"Block任务6, 所在Thread: %@\n",[NSThread currentThread]);
});

dispatch_async(queue, ^{
    NSLog(@"Block任务7, 所在Thread: %@\n",[NSThread currentThread]);
});

dispatch_async(queue, ^{
    NSLog(@"Block任务8, 所在Thread: %@\n",[NSThread currentThread]);
});
```

运行结果

```
2016-07-03 16:39:06.888 Demos[11598:161121] 1
2016-07-03 16:39:06.888 Demos[11598:161121] 2
2016-07-03 16:39:06.888 Demos[11598:161416] Block任务1, 所在Thread: <NSThread: 0x7fb4b0f09130>{number = 7, name = (null)}
2016-07-03 16:39:06.888 Demos[11598:161121] 3
2016-07-03 16:39:06.888 Demos[11598:161414] Block任务2, 所在Thread: <NSThread: 0x7fb4b0f09070>{number = 6, name = (null)}
2016-07-03 16:39:06.889 Demos[11598:161121] 4
2016-07-03 16:39:06.889 Demos[11598:161416] Block任务3, 所在Thread: <NSThread: 0x7fb4b0f09130>{number = 7, name = (null)}
2016-07-03 16:39:06.889 Demos[11598:161416] Block任务5, 所在Thread: <NSThread: 0x7fb4b0f09130>{number = 7, name = (null)}
2016-07-03 16:39:06.889 Demos[11598:161414] Block任务4, 所在Thread: <NSThread: 0x7fb4b0f09070>{number = 6, name = (null)}
2016-07-03 16:39:06.889 Demos[11598:161459] Block任务7, 所在Thread: <NSThread: 0x7fb4b0e09090>{number = 8, name = (null)}
2016-07-03 16:39:06.889 Demos[11598:161460] Block任务6, 所在Thread: <NSThread: 0x7fb4b0c3f9d0>{number = 9, name = (null)}
2016-07-03 16:39:06.889 Demos[11598:161416] Block任务8, 所在Thread: <NSThread: 0x7fb4b0f09130>{number = 7, name = (null)}
```

async是`异步`执行block任务的，什么意思了？就是说block任务一旦`扔到队列队尾`开始排队，那么代码就开始往下执行。而不会等待block任务执行完毕才会返回再往下执行，存在一个`顺序`的问题。

以前我就蠢到写如下的代码:

```objc
@interface ViewController ()
@property (nonatomic, copy) NSString *name;
@end

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	
	//1. 
	dispatch_async(dispatch_get_main_queue(), ^{
        self.name = @"hahahah";
    });
        
	//2.            
    NSLog(@"self.name = %@", self.name);
}

@end
```

运行结果

```
2016-07-03 16:27:52.282 Demos[10972:150277] self.name = (null)
```

我还纳闷了很久，为什么娶不到值...........但如果一定要拿到异步修改的name值，就必须要将获取name值的代码也放入到这个队列中排队执行

```objc
@interface ViewController ()
@property (nonatomic, copy) NSString *name;
@end

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	
	//1. 
	dispatch_async(dispatch_get_main_queue(), ^{
        self.name = @"hahahah";
    });
        
	//2. 
	dispatch_async(dispatch_get_main_queue(), ^{
		NSLog(@"self.name = %@", self.name);
    });
}

@end
```

###使用信号量 dispatch semephore 来让一个异步任务同步等待执行

semephore值为1的时候，可以用来当做多线程同步。如果值>1，可以用来完成类似生产者与消费者这样的联动的多线程处理任务。

```objc
static dispatch_semephore_t semephore

@interface ViewController ()
@property (nonatomic, copy) NSString *name;
@end

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	
	//1. 信号量初始化
	semephore = dispatch_semaphore_create(0);
	
	//2. 异步任务
	__block NSString *name = nil;
	dispatch_async(dispatch_get_main_queue(), ^{
        name  = [self.name copy];
        
        // 4. 异步任务完成之后，发出信号通知线程往下走
        dispatch_semaphore_signal(semephore);
    });
        
	//3. 卡主线程往下走，等待Block执行完毕再往下走
	dispatch_semaphore_wait(semephore, DISPATCH_TIME_FOREVER);
    NSLog(@"name = %@", name);
}

@end
```

还有个结合网络请求更具体点的例子:

```objc
static dispatch_semaphore_t semephore;

static NSString *URL = @"http://www.weather.com.cn/data/sk/101110101.html";

// 只要不是要成员Block变量保存Block，就能避免Block被替换掉的问题
@interface CallbackManager : NSObject
- (void)doWorkWithCompletionBlock:(void (^)(void))block;
@end
@implementation CallbackManager
- (void)doWorkWithCompletionBlock:(void (^)(void))block {
    NSURLSessionTask *task = [[NSURLSession sharedSession] dataTaskWithURL:[NSURL URLWithString:URL] completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error)
                              {
                                  if (block) {
                                      block();
                                  }
                              }];
    
    [task resume];
}
@end

@implementation CallbackViewController {
    CallbackManager *_manager;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    _manager = [[CallbackManager alloc] init];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
//    [self test1];
//    [self test2];
    [self test3];
}

- (void)test3 {
    semephore = dispatch_semaphore_create(0);
    NSLog(@"task begin");
    [_manager doWorkWithCompletionBlock:^{
        NSLog(@">>>>>>>> doWork <<<<<<<<<<<");
        NSLog(@"task end");
        dispatch_semaphore_signal(semephore);
    }];
    dispatch_semaphore_wait(semephore, DISPATCH_TIME_FOREVER);
    NSLog(@"next code");
}

@end
```

输出结果

```
2016-09-19 00:24:49.613 Demos[7485:72749] task begin
2016-09-19 00:24:49.725 Demos[7485:72944] >>>>>>>> doWork <<<<<<<<<<<
2016-09-19 00:24:49.725 Demos[7485:72944] task end
2016-09-19 00:24:49.726 Demos[7485:72749] next code
```

假设一个类对象的api是异步返回的，但是如果一定需要同步等待返回:

```objc
@interface Cache : NSObject
- (void)objectForKey:(NSString *)key completionBlock:(void (^)(id obj))block;
@end
@implementation Cache
- (void)objectForKey:(NSString *)key completionBlock:(void (^)(id obj))block {
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        if (block) {block([NSObject new]);}
    });
}
@end
```

如上这个类对象api是通过异步子线程方式返回一个对象，如下两种方法获取:

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    Cache *cache = [[Cache alloc] init];
    __block id obj1 = nil;
    
    //1. 只能异步获取
    [cache objectForKey:@"key" completionBlock:^(id obj2) {
        obj1 = obj2;
        NSLog(@"obj2 = %@", obj2);
    }];
    
    //2. 这里是获取不到值的
    NSLog(@"obj1 = %@", obj1);
}
```

输出结果

```
2016-09-13 10:49:13.861 Demos[685:122112] obj1 = (null)
2016-09-13 10:49:13.862 Demos[685:122205] obj2 = <NSObject: 0x140004bd0>
```

那么现在一定要这个api是同步等待返回了? 可以使用`dispatch_semaphore_t`来做到:

```objc
static dispatch_semaphore_t semaphore;
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    Cache *cache = [[Cache alloc] init];
    __block id obj1 = nil;
    semaphore = dispatch_semaphore_create(0);//1.
    [cache objectForKey:@"key" completionBlock:^(id obj2) {
        obj1 = obj2;
        NSLog(@"obj2 = %@", obj2);
        dispatch_semaphore_signal(semaphore);//2.
    }];
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);//3.
    NSLog(@"obj1 = %@", obj1);
}
```

输出结果

```
2016-09-13 10:51:57.825 Demos[690:122837] obj2 = <NSObject: 0x15e71d610>
2016-09-13 10:51:57.830 Demos[690:122805] obj1 = <NSObject: 0x15e71d610>
```

这样就可以等待异步子线程代码全部走完之后，再走后面的代码。

参照TMCache的代码，可以对Cache类做如下改造，即把上面的信号量代码封装到Cache类去:

```objc
static dispatch_semaphore_t semephore;

@interface Cache : NSObject
- (void)objectForKey:(NSString *)key completionBlock:(void (^)(id obj))block;
- (id)objectForKey:(NSString *)key;
@end
@implementation Cache
- (void)objectForKey:(NSString *)key completionBlock:(void (^)(id obj))block {
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        if (block) {block([NSObject new]);}
    });
}
- (id)objectForKey:(NSString *)key {
	//1. 
    semephore = dispatch_semaphore_create(0);
    
    __block id _obj = nil;
    [self objectForKey:key completionBlock:^(id obj) {
        _obj = obj;
        
        //3.
        dispatch_semaphore_signal(semephore);
    }];
    
    //2.
    dispatch_semaphore_wait(semephore, DISPATCH_TIME_FOREVER);
    return _obj;
}
@end
```

测试代码

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    Cache *cache = [[Cache alloc] init];
    id obj = [cache objectForKey:@"key"];
    NSLog(@"obj = %@", obj);
}
```

输出结果

```
2016-09-13 10:58:15.036 Demos[703:124432] obj = <NSObject: 0x13cd4c8d0>
```

###`disatch_async(serialQueue, ^(){....});`

```objc
dispatch_queue_t queue = dispatch_queue_create("用户队列", DISPATCH_QUEUE_SERIAL);

NSLog(@"1");

dispatch_async(queue, ^{
    NSLog(@"Block任务1, 所在Thread: %@\n", [NSThread currentThread]);
});

NSLog(@"2");

dispatch_async(queue, ^{
    NSLog(@"Block任务2, 所在Thread: %@\n", [NSThread currentThread]);
});

NSLog(@"3");

dispatch_async(queue, ^{
    NSLog(@"Block任务3, 所在Thread: %@\n",[NSThread currentThread]);
});

NSLog(@"4");

dispatch_async(queue, ^{
    NSLog(@"Block任务4, 所在Thread: %@\n",[NSThread currentThread]);
});

dispatch_async(queue, ^{
    NSLog(@"Block任务5, 所在Thread: %@\n",[NSThread currentThread]);
});

dispatch_async(queue, ^{
    NSLog(@"Block任务6, 所在Thread: %@\n",[NSThread currentThread]);
});

dispatch_async(queue, ^{
    NSLog(@"Block任务7, 所在Thread: %@\n",[NSThread currentThread]);
});

dispatch_async(queue, ^{
    NSLog(@"Block任务8, 所在Thread: %@\n",[NSThread currentThread]);
});
```

运行结果

```
2016-07-03 16:42:52.326 Demos[11824:165242] 1
2016-07-03 16:42:52.326 Demos[11824:165242] 2
2016-07-03 16:42:52.326 Demos[11824:165242] 3
2016-07-03 16:42:52.326 Demos[11824:165266] Block任务1, 所在Thread: <NSThread: 0x7fecd05148c0>{number = 2, name = (null)}
2016-07-03 16:42:52.327 Demos[11824:165242] 4
2016-07-03 16:42:52.327 Demos[11824:165266] Block任务2, 所在Thread: <NSThread: 0x7fecd05148c0>{number = 2, name = (null)}
2016-07-03 16:42:52.327 Demos[11824:165266] Block任务3, 所在Thread: <NSThread: 0x7fecd05148c0>{number = 2, name = (null)}
2016-07-03 16:42:52.328 Demos[11824:165266] Block任务4, 所在Thread: <NSThread: 0x7fecd05148c0>{number = 2, name = (null)}
2016-07-03 16:42:52.328 Demos[11824:165266] Block任务5, 所在Thread: <NSThread: 0x7fecd05148c0>{number = 2, name = (null)}
2016-07-03 16:42:52.328 Demos[11824:165266] Block任务6, 所在Thread: <NSThread: 0x7fecd05148c0>{number = 2, name = (null)}
2016-07-03 16:42:52.332 Demos[11824:165266] Block任务7, 所在Thread: <NSThread: 0x7fecd05148c0>{number = 2, name = (null)}
2016-07-03 16:42:52.334 Demos[11824:165266] Block任务8, 所在Thread: <NSThread: 0x7fecd05148c0>{number = 2, name = (null)}
```

block任务依然还是`异步`执行的，同样也会创建子线程。但和`无序`队列的区别是:

- 只会创建`一个`新的子线程 来执行block
- 所有的线程按照`入队的先后顺序`依次执行block

所以使用 `dispatch async` + `serial queue` 形式，也可以达到多线程同步的效果。

###`dispatch_async`与`dispatch_sync`同时在一个Serial GCD Queue调度

```objc
@implementation ViewController

- (void)testQeueu {
    dispatch_queue_t queue = dispatch_queue_create("serial", DISPATCH_QUEUE_SERIAL);
    
    dispatch_async(queue, ^{
        NSLog(@"async task 1 >>> thread: %@", [NSThread currentThread]);
    });
    dispatch_async(queue, ^{
        NSLog(@"async task 2 >>> thread: %@", [NSThread currentThread]);
    });
    dispatch_sync(queue, ^{
        NSLog(@"sync task 3 >>> thread: %@", [NSThread currentThread]);
    });
    dispatch_async(queue, ^{
        NSLog(@"async task 4 >>> thread: %@", [NSThread currentThread]);
    });
    dispatch_sync(queue, ^{
        NSLog(@"sync task 5 >>> thread: %@", [NSThread currentThread]);
    });
}

@end
```

输出结果

```
2016-12-12 16:16:03.722 XZHRuntimeDemo[19943:1384149] async task 1 >>> thread: <NSThread: 0x61800007e7c0>{number = 4, name = (null)}
2016-12-12 16:16:03.723 XZHRuntimeDemo[19943:1384149] async task 2 >>> thread: <NSThread: 0x61800007e7c0>{number = 4, name = (null)}
2016-12-12 16:16:03.723 XZHRuntimeDemo[19943:1376726] sync task 3 >>> thread: <NSThread: 0x60000007e740>{number = 1, name = main}
2016-12-12 16:16:03.723 XZHRuntimeDemo[19943:1384149] async task 4 >>> thread: <NSThread: 0x61800007e7c0>{number = 4, name = (null)}
2016-12-12 16:16:03.723 XZHRuntimeDemo[19943:1376726] sync task 5 >>> thread: <NSThread: 0x60000007e740>{number = 1, name = main}
```

- (1) 不管是async还是sync的调度的block，都是按照`顺序`执行
- (2) async调度的block，会在queue对应的线程上执行
	- 且是`异步`执行（当block丢到队列末尾就往下走）
- (3) sync调度的block，直接在当前线程上执行
	- 且是`同步等待`执行，必须等待block执行完毕，才会走后面的代码

##小结: 对于dispatch async/sync + ConcurrentQueue/SerialQueue 的区别图示

![](http://i3.tietuku.com/90837468181d2cf0.png)

- (1) `dispatch_sync()`在任何对列下都是保持如下
	- **不创建新的线程**
	- 在当前`dispatch_sync()`代码所在线程，执行完block，才会让所在线程执行后面的代码
	- 也就是说会卡主当前`dispatch_sync()`代码所在线程

- (2) `dispatch_async()`需要区分队列情况
	- Serial Queue
		- `main queue`，不会再创建线程，而是直接在主线程上等待执行
		- `自创建 serial queue`
			- 创建**一个唯一**的线程，一个接着一个调度执行block
			- 也可以完成`多线程同步`，只不过block都是在另外一个`子线程`上依次按照`顺序`执行
	- Concurrent Queue
		- 会创建多个线程，并且会复用这些线程
		- block会被自动分配到其中的某一个线程上调度执行
	- 比`dispatch_sync()`多进行block对象的拷贝

###dispatch once来让一段代码只执行一次

```
+ (instancetype)person {

    static Person *p = nil;
    static dispatch_once_t onceToken;
    
//此处打一个断点    
    
    dispatch_once(&onceToken, ^{
        p = [[Person alloc] init];
    });
    
//此处打一个断点   
    
    return p;
}
```

在第一个断点的，我们输出下 onceToken

```
(dispatch_once_t) onceToken = 0
```

再跳到第二个断点，继续输出下 onceToken

```
(dispatch_once_t) onceToken = -1
```

发现onceToken变成一个`-1`了.

OK，大概知道了让代码执行一次的道理了:

- 代码没有执行之前，将标志位置为 0
- 代码第一次执行之后，就将标志位改为 -1
- 每次执行到`dispatch_once()`代码块时，判断标志位如果是`0`就执行Block，如果是`-1`就不执行Block.
- 而改变这个`onceToken`值的代码，肯定是做了多线程同步处理的

###iOS8之后提交到gcd队列中的block也可`取消`了

```c
dispatch_queue_t queue = dispatch_queue_create("queue", DISPATCH_QUEUE_SERIAL);

//注意，一定要保存这个Block任务，后续才能取消
dispatch_block_t block1 = dispatch_block_create(0, ^{
    NSLog(@"block1 begin");
    [NSThread sleepForTimeInterval:1];
    NSLog(@"block1 done");
});

dispatch_block_t block2 = dispatch_block_create(0, ^{
    NSLog(@"block2 ");
});

dispatch_async(queue, block1);
dispatch_async(queue, block2);

//取消执行
dispatch_block_cancel(block2);
```

###`dispatch_barrier_async`来同步等待`concurrent queue`中的一个队列中的block执行

```objc
@implementation ViewController

- (void)thread1 {
    
    dispatch_queue_t concurrentDiapatchQueue=dispatch_queue_create("com.test.queue", DISPATCH_QUEUE_CONCURRENT);
	
	//1. 并发无序的任务
    dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"1 - thread: %@", [NSThread currentThread]);});
    dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"2 - thread: %@", [NSThread currentThread]);});
    dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"3 - thread: %@", [NSThread currentThread]);});
    dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"4 - thread: %@", [NSThread currentThread]);});
	
	//2. 需要按照循序执行任务
    dispatch_barrier_async(concurrentDiapatchQueue, ^{
        sleep(5); NSLog(@"停止5秒我是同步执行 - thread: %@", [NSThread currentThread]);
    });
	
	//3. 并发无序的任务
    dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"6 - thread: %@", [NSThread currentThread]);});
    dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"7 - thread: %@", [NSThread currentThread]);});
    dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"8 - thread: %@", [NSThread currentThread]);});
    dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"9 - thread: %@", [NSThread currentThread]);});
    dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"10 - thread: %@", [NSThread currentThread]);});
}

@end
```

输出信息

```
2016-06-28 23:31:33.085 Demos[1696:24860] 2 - thread: <NSThread: 0x7ff6f2406d80>{number = 8, name = (null)}
2016-06-28 23:31:33.085 Demos[1696:25426] 1 - thread: <NSThread: 0x7ff6f2605820>{number = 11, name = (null)}
2016-06-28 23:31:33.085 Demos[1696:24859] 4 - thread: <NSThread: 0x7ff6f253cce0>{number = 7, name = (null)}
2016-06-28 23:31:33.085 Demos[1696:24935] 3 - thread: <NSThread: 0x7ff6f26012b0>{number = 10, name = (null)}
2016-06-28 23:31:38.089 Demos[1696:24935] 停止5秒我是同步执行 - thread: <NSThread: 0x7ff6f26012b0>{number = 10, name = (null)}
2016-06-28 23:31:38.090 Demos[1696:24859] 7 - thread: <NSThread: 0x7ff6f253cce0>{number = 7, name = (null)}
2016-06-28 23:31:38.090 Demos[1696:24935] 6 - thread: <NSThread: 0x7ff6f26012b0>{number = 10, name = (null)}
2016-06-28 23:31:38.090 Demos[1696:25426] 8 - thread: <NSThread: 0x7ff6f2605820>{number = 11, name = (null)}
2016-06-28 23:31:38.090 Demos[1696:24860] 9 - thread: <NSThread: 0x7ff6f2406d80>{number = 8, name = (null)}
2016-06-28 23:31:38.090 Demos[1696:26066] 10 - thread: <NSThread: 0x7ff6f2505f20>{number = 12, name = (null)}
```

可以看到会让这个`concurrent queue`对应的后面所有的block线程任务挂起，必须等待`dispatch_barrier_async()`分配的block执行完毕之后，才会继续执行`concurrent queue`对应后面入队的block线程任务。

###之前我还以为`dispatch_barrier_async()`可能会导致`主线程`也会被挂起，其实没明白`队列调度block`以及`block入队`的概念。

`dispatch_barrier_async(queue, block)`只是会让传入方法中的`queue`后续入队的block任务暂时挂起，即不去调度执行。

然后等待`dispatch_barrier_async()`入队的那个block任务彻底执行完毕之后，才会去开始调度之后入队的block任务，并分配线程执行。

所以，压根就跟`主线程、主线程队列`没有任何的关系，只和`dispatch_barrier_async(queue, block)`中传入的queue有关系。

但是我觉得还是不要对`系统的线程队列`去使用`dispatch_barrier_async()`，因为他会让后续入队的block调度临时挂起。很可能这个时候，其他往系统队列入队了一个block任务，但是没有及时被响应调度。

所以个人觉得，一般还是使用`自己创建的 concurrent queue`去做`dispatch_barrier_async()`操作吧。


###`dispatch_async_f` 完成对`C方法`的异步调用


```c
static void *MyContext = &MyContext;
```

```c
-(void)doDispatchAsyncF
{
    
    //1. 线程队列
    dispatch_queue_t queue=dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
    //2. 异步调用c方法
    dispatch_async_f(queue, &MyContext, logNum);
    
    NSLog(@"-------doDispatchAsyncF------");
}
```

```c
void logNum()
{
    NSLog(@"------logNum------");
}
```


###`dispatch_suspend/dispatch_resume` 挂起/恢复一个队列进行block调度

```objc
- (void)thread5 {
    
    dispatch_queue_t concurrentDiapatchQueue=dispatch_queue_create("com.test.queue", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_async(concurrentDiapatchQueue, ^{
        
        for (int i=1; i<11; i++)
        {
            NSLog(@"%d - thread: %@", i, [NSThread currentThread]);
            
            //i = 3, 6, 9 时，先暂停队列后休眠3秒，然后恢复队列执行
            if (i % 3 == 0)
            {
                dispatch_suspend(concurrentDiapatchQueue);
                
                sleep(3);
                
                dispatch_resume(concurrentDiapatchQueue);
            }
        }
    });
}
```

输出信息

```
2016-06-28 23:58:52.599 Demos[3496:58977] 1 - thread: <NSThread: 0x7fadebf128b0>{number = 2, name = (null)}
2016-06-28 23:58:52.600 Demos[3496:58977] 2 - thread: <NSThread: 0x7fadebf128b0>{number = 2, name = (null)}
2016-06-28 23:58:52.600 Demos[3496:58977] 3 - thread: <NSThread: 0x7fadebf128b0>{number = 2, name = (null)}
2016-06-28 23:58:55.606 Demos[3496:58977] 4 - thread: <NSThread: 0x7fadebf128b0>{number = 2, name = (null)}
2016-06-28 23:58:55.606 Demos[3496:58977] 5 - thread: <NSThread: 0x7fadebf128b0>{number = 2, name = (null)}
2016-06-28 23:58:55.607 Demos[3496:58977] 6 - thread: <NSThread: 0x7fadebf128b0>{number = 2, name = (null)}
2016-06-28 23:58:58.612 Demos[3496:58977] 7 - thread: <NSThread: 0x7fadebf128b0>{number = 2, name = (null)}
2016-06-28 23:58:58.613 Demos[3496:58977] 8 - thread: <NSThread: 0x7fadebf128b0>{number = 2, name = (null)}
2016-06-28 23:58:58.613 Demos[3496:58977] 9 - thread: <NSThread: 0x7fadebf128b0>{number = 2, name = (null)}
2016-06-28 23:59:01.618 Demos[3496:58977] 10 - thread: <NSThread: 0x7fadebf128b0>{number = 2, name = (null)}
```

###`dispatch_semaphore_t` 信号量

通常情况下，semephore被用于多线程同步，一般只初始化`一个信号值`，一次只能让一个线程去执行一段代码:

```objc
- (void)test4 {
    
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    
    // 一次只允许一个线程执行
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);
    
    NSMutableArray *array = [NSMutableArray array];
    for (int index = 0; index < 100000; index++) {
        
        // 多线程操作的代码
        dispatch_async(queue, ^(){
            
            // 信号值-1为0，让其他线程排队等待
            dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
            
            [array addObject:[NSNumber numberWithInt:index]];
            
            // 信号值+1，通知其他排队的线程选择一个进入执行
            dispatch_semaphore_signal(semaphore);
            
        });
        
    }
}
```

semephore其自面意思是`信号量`，所以其实主要用来当做一种信号，来让多个线程合理的并发运行

```objc
@implementation ViewController

- (void)thread6 {
    
    //1. 初始信号量为5，为了一次能让5个子线程同时执行
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(5);
    
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    
    // 模拟100个子线程并发访问互斥变量
    for (int i = 0; i <30; i++)
    {
        //2. 消耗一个信号值，信号量 - 1
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
        
        // 同步访问临界区
        dispatch_async(queue, ^{
            
            NSLog(@"%d - thread: %@", i, [NSThread currentThread]);
            
            //3. 访问完临界区，信号量 + 1，释放信号量
            dispatch_semaphore_signal(semaphore);
        });
    }
}

@end
```

即可以控制某一次并发执行指定个数的线程，有点类似`NSCondition`的效果。

关于信号的最关键的几步:

- (1) 初始化信号量
- (2) wait 消耗一个信号值
- (3) signal 增加一个信号值


在NSURLSession结构不提供同步请求的，但是又需要执行同步请求的话，可以利用信号量

比如，异步完成回调的写法:

```objc
+ (void)synchronizaDownloadPatchPatchModel:(FQLPatchModel *__nonnull)model completionBlock:(void (^__nullable)(NSData *fileData))block
{
    // 开始网络请求
    NSURLRequest *request = nil;
    NSURLSessionDownloadTask *task = [[NSURLSession sharedSession] dataTaskWithRequest:request completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error)
    {
        // 完成下载回调
        block(data);
    }];
    
    [task resume];
}
```

但比如App启动进行下载热修复的脚本文件时，就必须同步完成下载然后执行AppDelegate的构建UI等操作以及进入主UI。如果脚本文件没有下载成功、安装成功，那么就不应该直接进入App主UI。那么此时就需要`同步回调`

```objc
+ (void)synchronizaDownloadPatchPatchModel:(FQLPatchModel *__nonnull)model completionBlock:(void (^__nullable)(NSData *fileData))block
{
    //1. 创建一个0的信号量
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    
    // 开始网络请求
    NSURLRequest *request = nil;
    NSURLSessionDownloadTask *task = [[NSURLSession sharedSession] dataTaskWithRequest:request completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error)
                                      {
                                          // 完成下载回调之后
                                          block(data);
                                          
                                          //3. 释放信号量，信号值为1，让线程继续往下执行
                                          dispatch_semaphore_signal(semaphore);
                                      }];
    [task resume];
    
    //2. 开始网络请求后，让当前下载线程（主线程）卡顿，不往下继续执行
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
}
```

可以达到同步完成现在任务之后，再同步执行回调。


###`dispatch_after` 在给定的线程队列中，延迟执行block

```
//1. 延迟执行时间
double delayInSeconds = 2.0;
    
//2. 包装延迟时间
dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, (int64_t) (delayInSeconds * NSEC_PER_SEC));

//3. 指定延迟执行的Block任务，在哪一个线程队列调度
dispatch_after(popTime, dispatch_get_main_queue(), ^(void){
    NSLog(@"我是延迟2秒后执行的代码。。。\n");
});
```

注意，使用`dispatch_after`延迟执行的代码块是`不能取消执行`的。

###`dispatch_group_t`，将一个队列上的`多个block任务`组合起来，可以方便设置所有任务完成时候的回调.


```objc
- (void)thread7 {
    
    dispatch_group_t group = dispatch_group_create();
    
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    
    dispatch_group_async(group, queue, ^{
        NSLog(@"任务1");
    });
    
    dispatch_group_async(group, queue, ^{
        NSLog(@"任务2");
    });
    
    dispatch_group_async(group, queue, ^{
        NSLog(@"任务3");
    });
    
    dispatch_group_async(group, queue, ^{
        NSLog(@"任务4");
    });
    
    dispatch_group_async(group, queue, ^{
        NSLog(@"任务5");
    });
}
```

输出信息

```
2016-06-29 00:21:15.339 Demos[4878:81889] 任务1
2016-06-29 00:21:15.339 Demos[4878:81887] 任务3
2016-06-29 00:21:15.339 Demos[4878:81888] 任务2
2016-06-29 00:21:15.339 Demos[4878:82056] 任务4
2016-06-29 00:21:15.339 Demos[4878:82057] 任务5
```

group enter + group leave == `dispatch_group_async`

下面两张方式是等价的.

```objc
dispatch_group_async(group, queue, ^{ 

// 。。。 

}); 
```

```objc
dispatch_group_enter (group);

dispatch_async(queue, ^{

//。。。

dispatch_group_leave (group);

});
```

`dispatch_group_notify`组任务完成之后一定会执行的代码块

```objc
//1. 组
dispatch_group_t group = dispatch_group_create();
    
//2. 队列
dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    
//3. 如下提交三个组任务
dispatch_group_async(group, queue, ^{
    NSLog(@"任务1");
});
    
dispatch_group_async(group, queue, ^{
    NSLog(@"任务2");
});
    
dispatch_group_async(group, queue, ^{
    NSLog(@"任务3");
});

//4. 最后一定执行的任务
dispatch_group_notify(group, queue, ^{
    NSLog(@"组执行完毕");
});
```

输出结果

```
2015-09-18 00:40:48.805 Demo[18296:198228] 任务2
2015-09-18 00:40:48.805 Demo[18296:198229] 任务3
2015-09-18 00:40:48.805 Demo[18296:198227] 任务1
2015-09-18 00:40:48.806 Demo[18296:198227] 组执行完毕
```

使用 `dispatch_group_wait` 等待该group中所有的线程任务全部执行完毕

```objc
- (void)thread7 {
    
    dispatch_group_t group = dispatch_group_create();
    
    dispatch_queue_t queue = dispatch_queue_create("queue", DISPATCH_QUEUE_CONCURRENT);
    
    //将任务异步地添加到group中去执行
    dispatch_group_async(group,queue,^{NSLog(@"1 - thread: %@", [NSThread currentThread]);});
    dispatch_group_async(group,queue,^{NSLog(@"2 - thread: %@", [NSThread currentThread]);});
    dispatch_group_async(group,queue,^{NSLog(@"3 - thread: %@", [NSThread currentThread]);});
    dispatch_group_async(group,queue,^{NSLog(@"4 - thread: %@", [NSThread currentThread]);});
    dispatch_group_async(group,queue,^{NSLog(@"5 - thread: %@", [NSThread currentThread]);});
    
    //等待Group内所有的任务执行完毕，再往下执行
    dispatch_group_wait(group,DISPATCH_TIME_FOREVER);
    
    NSLog(@"go on");
}
```

输出结果

```
2015-09-18 00:41:36.629 Demo[18365:199651] 3 - thread: <NSThread: 0x7f96ca504900>{number = 4, name = (null)}
2015-09-18 00:41:36.629 Demo[18365:199782] 4 - thread: <NSThread: 0x7f96ca4051a0>{number = 5, name = (null)}
2015-09-18 00:41:36.629 Demo[18365:199646] 1 - thread: <NSThread: 0x7f96ca610aa0>{number = 2, name = (null)}
2015-09-18 00:41:36.629 Demo[18365:199645] 2 - thread: <NSThread: 0x7f96ca70e2a0>{number = 3, name = (null)}
2015-09-18 00:41:36.629 Demo[18365:199651] 5 - thread: <NSThread: 0x7f96ca504900>{number = 4, name = (null)}
2015-09-18 00:41:36.629 Demo[18365:199530] go on
```

###`dispatch_source_t` 事件源，用于注册到RunLoop

所有的source分类

```
DISPATCH_SOURCE_TYPE_DATA_ADD   变量增加
DISPATCH_SOURCE_TYPE_DATA_OR    变量OR
DISPATCH_SOURCE_TYPE_MACH_SEND  Mach端口发送
DISPATCH_SOURCE_TYPE_MACH_RECV  Mach端口接收
DISPATCH_SOURCE_TYPE_MEMORYPRESSURE 内存压力情况变化
DISPATCH_SOURCE_TYPE_PROC       与进程相关的事件
DISPATCH_SOURCE_TYPE_READ       可读取文件映像
DISPATCH_SOURCE_TYPE_SIGNAL     接收信号
DISPATCH_SOURCE_TYPE_TIMER      定时器事件
DISPATCH_SOURCE_TYPE_VNODE      文件系统变更
DISPATCH_SOURCE_TYPE_WRITE      可写入文件映像
```

主要的使用步骤:

- 1) 创建一个source

```
dispatch_source_create(dispatch_source_type_t type, uintptr_t handle, unsigned long mask, dispatch_queue_t queue);
```

- 2) 设置source触发时的回调处理

```
dispatch_source_set_event_handler(dispatch_source_t source, ^{
    //回调Block
});
```

- 3) 设置取消source时的回调处理

```
dispatch_source_set_cancel_handler(mySource, ^{ 
   close(fd); //关闭文件秒速符 
});
```

- 4)【重要】将source添加到某个线程的runloop，让runloop接收事件

```
dispatch_resume(dispatch_source_t source);//默认使用 Default RunLoop Mode 进行事件注册
```

比如，监视文件夹内文件变化

```objc
NSURL *directoryURL; // assume this is set to a directory
int const fd = open([[directoryURL path] fileSystemRepresentation], O_EVTONLY);
if (fd < 0) {
     char buffer[80];
     strerror_r(errno, buffer, sizeof(buffer));
     NSLog(@"Unable to open \"%@\": %s (%d)", [directoryURL path], buffer, errno);
     return;
}
dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_VNODE, fd,
DISPATCH_VNODE_WRITE | DISPATCH_VNODE_DELETE, DISPATCH_TARGET_QUEUE_DEFAULT);
dispatch_source_set_event_handler(source, ^(){
     unsigned long const data = dispatch_source_get_data(source);
     if (data & DISPATCH_VNODE_WRITE) {
          NSLog(@"The directory changed.");
     }
     if (data & DISPATCH_VNODE_DELETE) {
          NSLog(@"The directory has been deleted.");
     }
});
dispatch_source_set_cancel_handler(source, ^(){
     close(fd);
});
self.source = source;
dispatch_resume(self.source);
//还要注意需要用DISPATCH_VNODE_DELETE 去检查监视的文件或文件夹是否被删除，如果删除了就停止监听
```


###`dispatch_queue_set_specific()` 、`dispatch_get_specific()`

有点类似runtime中的 `objc_setAssociatedObject()`与`objc_getAssociatedObject()`，动态绑定一个对象。

FMDB解决线程死锁: 主要是防止在 `同一个队列` 上进行 dispatch block任务:


```objc
//1. 
static const void * const kDispatchQueueSpecificKey = &kDispatchQueueSpecificKey;

//2. 绑定一个queue给self
//创建一个串行队列来执行数据库的所有操作
_queue = dispatch_queue_create([[NSString stringWithFormat:@"fmdb.%@", self] UTF8String], NULL);

//通过key标示队列，设置context为self
dispatch_queue_set_specific(_queue, kDispatchQueueSpecificKey, (__bridge void *)self, NULL);

//3. 在inDatabase:方法中，取出绑定的queue
//判断queue是否与当前方法执行所在线程的队列一致
//如果一样，就不往下执行，否则会出现线程死锁
- (void)inDatabase:(void (^)(FMDatabase *db))block {
    FMDatabaseQueue *currentSyncQueue = (__bridge id)dispatch_get_specific(kDispatchQueueSpecificKey);
    assert(currentSyncQueue != self && "inDatabase: was called reentrantly on the same queue, which would lead to a deadlock");
    
    // 后面是dispatch_sync()的代码.....
}
```

###Dispatch IO，让多个子线程并发读取一个`大文件`，指定每个子线程读取文件的 `A字节 到 B字节`，最终完成整个大文件的读取.

大体步骤

```objc
dispatch_async(queue, ^{/* 线程1，读取0-99字节 */});
dispatch_async(queue, ^{/* 线程2，读取100-199字节 */});
dispatch_async(queue, ^{/* 线程3，读取200-299字节 */});
...
```

主要步骤

```
1. 创建一个IO
2. 设定一次读取的大小（分割的大小）
3. 使用Global Dispatch Queue开始并列读取
4. 当每个分割的文件块读取完毕时，会将含有文件数据的dispatch data返回到dispatch_io_read设定的block，
在block中需要分析传递过来的dispatch data进行合并处理
```

下面是代码示例

```objc
dispatch_queue_t errorQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
```

使用`dispatch_data_t`实例，保存最终读取文件的二进制数据

```objc
dispatch_data_t data = dispatch_data_create(charbuffer, 4 * sizeof(int), queue, NULL);
```

多线程同步的信号量

```objc
dispatch_semaphore_t sem = dispatch_semaphore_create(0);
```

设置要读取的文件

```objc
dispatch_fd_t fd = open("/tmp/data.txt", O_RDWR | O_CREAT | O_TRUNC, S_IRWXU | S_IRWXG | S_IRWXO);
```

创建`dispatch_io_t`实例，来开始一个IO文件流

```objc
dispatch_io_t io = dispatch_io_create(DISPATCH_IO_STREAM, fd, errorQueue, ^(int err){

    //出错时执行的handler
    close(fd);
});
```

设定一次读取的大小(分割大小)

```objc
dispatch_io_set_low_water(io, SIZE_MAX);
```

多线程并发互斥读取文件，并将读取到的部分data拼接到`dispatch_data_t实例`中

```objc
dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER);

dispatch_io_read(io, 0, SIZE_MAX, queue, ^(bool done, dispatch_data_t pipedata, int err)
{
    //1. 出现错误直接结束执行
    if (error)
        return;
    
    //2. 不断的读取小文件块
    if (err == 0)
    {
        //每次读取到数据进行数据的处理
        size_t len = dispatch_data_get_size(pipedata);
        
        //读取到了数据
        if (data != NULL)
        {
            //拼接数据
            if (_data == NULL)
            {	
            	// 文件读取刚开始
                _data = data;
            } else {
            	// 读取到文件的部分数据，添加到dispatch_data_t实例后
                _data = dispatch_data_create_concat(_data, data);
            }
        }
        
        dispatch_semaphore_signal(sem);
    }
    
    //3. 读取完毕
    if (done)
    {
        //并发读取完毕
        dispatch_semaphore_signal(sem);
        dispatch_release(io);
        dispatch_release(queue);
    }
});
```


###`dispatch_set_target_queue(子队列1, 集合队列)`

- (1) `dispatch_set_target_queue(子队列1, 集合队列)`将多个子队列组合到一个整体队列，让多个子队列一起完成一个任务


代码如下:

```objc
@implementation ViewController

- (void)thread {
    
    //1. 组织 其他所有串行队列的 最终串行队列
    dispatch_queue_t targetQueue = dispatch_queue_create("test.target.queue", DISPATCH_QUEUE_SERIAL);
    
    //2. 每一个执行子串行任务的 子串行队列
    dispatch_queue_t queue1 = dispatch_queue_create("test.1", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t queue2 = dispatch_queue_create("test.2", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t queue3 = dispatch_queue_create("test.3", DISPATCH_QUEUE_SERIAL);
    
    //3. 将queue1、queue2、queue3，组织到 targetQueue 中 串行执行
    dispatch_set_target_queue(queue1, targetQueue);
    dispatch_set_target_queue(queue2, targetQueue);
    dispatch_set_target_queue(queue3, targetQueue);
    
    //4. 给queue1、queue2、queue3分配任务
    dispatch_async(queue1, ^{
        NSLog(@"1 in");
        NSLog(@"thread = %@\n", [NSThread currentThread]);
        NSLog(@"1 out");
    });
    
    dispatch_async(queue2, ^{
        NSLog(@"2 in");
        NSLog(@"thread = %@\n", [NSThread currentThread]);
        NSLog(@"2 out");
    });
    
    dispatch_async(queue3, ^{
        NSLog(@"3 in");
        NSLog(@"thread = %@\n", [NSThread currentThread]);
        NSLog(@"3 out");  
    });
}

@end 
```

输出如下

```
2016-06-29 16:04:19.557 demo[4721:1127044] 1 in
2016-06-29 16:04:19.558 demo[4721:1127044] thread = <NSThread: 0x7fc550d10c10>{number = 2, name = (null)}
2016-06-29 16:04:19.558 demo[4721:1127044] 1 out
2016-06-29 16:04:19.558 demo[4721:1127044] 2 in
2016-06-29 16:04:19.559 demo[4721:1127044] thread = <NSThread: 0x7fc550d10c10>{number = 2, name = (null)}
2016-06-29 16:04:19.559 demo[4721:1127044] 2 out
2016-06-29 16:04:19.559 demo[4721:1127044] 3 in
2016-06-29 16:04:19.559 demo[4721:1127044] thread = <NSThread: 0x7fc550d10c10>{number = 2, name = (null)}
2016-06-29 16:04:19.559 demo[4721:1127044] 3 out
```

可以看到，队列也是按照串行顺序进行调度，然后队列中任务也是串行.


- (2) `dispatch_set_target_queue(队列1, 队列2)` 将`队列2 的权限复制给 队列1`

使用dispatch set target queue 复制队列的权限:

```objc
//1. 自己创建的队列
dispatch_queue_t serialQueue = dispatch_queue_create("队列名字",NULL);

//2. 系统队列
dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND,0);

//3. 将第二个队列的优先级 复制给 第一个队列
dispatch_set_target_queue(serialQueue, globalQueue);
```

因为iOS8之前，创建线程队列时，无法直接指定队列的优先级。所以只能是先创建出一个队列，然后将系统队列的对应优先级复制给创建的队列。

但是iOS8及之后，直接可以在创建队列的时，直接指定队列的服务质量QOS。

```objc
//1. 指定队列的服务质量
dispatch_queue_attr_t queue_attr = dispatch_queue_attr_make_with_qos_class (DISPATCH_QUEUE_SERIAL, QOS_CLASS_UTILITY, -1);

//2. 创建队列时，传入设置的服务质量
dispatch_queue_t queue = dispatch_queue_create("queue", queue_attr);
```



##多线程同步

如上创建出来的NSThread对象，当共同指向一个入口函数时，可能会发生一些多线程并发造成数据不统一的错误。比如，如下使用NSThread卖票的实例代码:

```objc
#import "ThreadDemo3.h"

static NSInteger Count = 10;

@implementation ThreadDemo3 {
    NSThread *_t1;
    NSThread *_t2;
    NSThread *_t3;
}

- (instancetype)init
{
    self = [super init];
    if (self) {
        _t1 = [[NSThread alloc] initWithTarget:self selector:@selector(saleTickets) object:nil];
        _t1.name = @"线程1";
        
        _t2 = [[NSThread alloc] initWithTarget:self selector:@selector(saleTickets) object:nil];
        _t2.name = @"线程2";
        
        _t3 = [[NSThread alloc] initWithTarget:self selector:@selector(saleTickets) object:nil];
        _t3.name = @"线程3";
    }
    return self;
}

- (void)test {
    [_t1 start];
    [_t2 start];
    [_t3 start];
}

- (void)saleTickets {
    while (1) {
        if (Count > 0) {
            sleep(1);
            Count--;
            NSLog(@"卖出一张票，余票: %ld, 所在线程:%@", Count, [NSThread currentThread]);
        } else {
            NSLog(@"已经卖完，所在线程:%@", [NSThread currentThread]);
            break;
        }
    }
}

@end
```

输出信息

```
2016-07-03 12:24:40.394 Demos[3758:56774] 卖出一张票，余票: 9, 所在线程:<NSThread: 0x7fe65be24730>{number = 3, name = 线程2}
2016-07-03 12:24:40.394 Demos[3758:56775] 卖出一张票，余票: 7, 所在线程:<NSThread: 0x7fe65be0cec0>{number = 4, name = 线程3}
2016-07-03 12:24:40.394 Demos[3758:56773] 卖出一张票，余票: 8, 所在线程:<NSThread: 0x7fe65be23930>{number = 2, name = 线程1}
2016-07-03 12:24:41.400 Demos[3758:56774] 卖出一张票，余票: 6, 所在线程:<NSThread: 0x7fe65be24730>{number = 3, name = 线程2}
2016-07-03 12:24:41.400 Demos[3758:56775] 卖出一张票，余票: 5, 所在线程:<NSThread: 0x7fe65be0cec0>{number = 4, name = 线程3}
2016-07-03 12:24:41.400 Demos[3758:56773] 卖出一张票，余票: 5, 所在线程:<NSThread: 0x7fe65be23930>{number = 2, name = 线程1}
2016-07-03 12:24:42.406 Demos[3758:56774] 卖出一张票，余票: 4, 所在线程:<NSThread: 0x7fe65be24730>{number = 3, name = 线程2}
2016-07-03 12:24:42.406 Demos[3758:56773] 卖出一张票，余票: 3, 所在线程:<NSThread: 0x7fe65be23930>{number = 2, name = 线程1}
2016-07-03 12:24:42.406 Demos[3758:56775] 卖出一张票，余票: 3, 所在线程:<NSThread: 0x7fe65be0cec0>{number = 4, name = 线程3}
2016-07-03 12:24:43.411 Demos[3758:56775] 卖出一张票，余票: 1, 所在线程:<NSThread: 0x7fe65be0cec0>{number = 4, name = 线程3}
2016-07-03 12:24:43.411 Demos[3758:56773] 卖出一张票，余票: 1, 所在线程:<NSThread: 0x7fe65be23930>{number = 2, name = 线程1}
2016-07-03 12:24:43.411 Demos[3758:56774] 卖出一张票，余票: 2, 所在线程:<NSThread: 0x7fe65be24730>{number = 3, name = 线程2}
2016-07-03 12:24:44.415 Demos[3758:56775] 卖出一张票，余票: 0, 所在线程:<NSThread: 0x7fe65be0cec0>{number = 4, name = 线程3}
2016-07-03 12:24:44.415 Demos[3758:56774] 卖出一张票，余票: -2, 所在线程:<NSThread: 0x7fe65be24730>{number = 3, name = 线程2}
2016-07-03 12:24:44.415 Demos[3758:56773] 卖出一张票，余票: -1, 所在线程:<NSThread: 0x7fe65be23930>{number = 2, name = 线程1}
2016-07-03 12:24:44.415 Demos[3758:56775] 已经卖完，所在线程:<NSThread: 0x7fe65be0cec0>{number = 4, name = 线程3}
2016-07-03 12:24:44.416 Demos[3758:56774] 已经卖完，所在线程:<NSThread: 0x7fe65be24730>{number = 3, name = 线程2}
2016-07-03 12:24:44.416 Demos[3758:56773] 已经卖完，所在线程:<NSThread: 0x7fe65be23930>{number = 2, name = 线程1}
```

从输出看，这段代码执行的是有问题的，截图下可能会更清楚:

![](http://i4.piimg.com/4851/1baf8ff2275bb800.png)

可以看到有三个错误的地方:

- 卖出第一张票时，只神修改8张了....这明显不对
- 卖出第二张和第三站时，剩余票还是9张....这也明显不对吧
- 最后票已经卖完了，却还在卖票，导致剩余票数为-1 .... 这就错的离谱了

而且不断的重新运行如上代码，每一次得到的发生错误卖票的顺序还是不一样的.

那么分析为何会出现如上卖票错误的情况，但是必须要搞清楚一个问题:

> 在单核系统环境，多线程并不是真正的同时运行，而是不断的被cpu切换着运行。

- 首先，三个子线程是虽然是依次start，但是都是执行一个方法，且都是一个while(1){}死循环，不断的执行同一个方法。

- 当线程1进入saleTickets方法时，此时票数=10，但是立马执行了`sleep(1);`，所以线程1休眠了，并没有往下执行`Count--;`。
	- 其实`sleep(1);`这句代码是造成如上代码出现错误的关键所在，可能会造成cpu切换发生错误。

- 那么线程1休眠了，cpu立马切换到`线程2`运行。此时线程2也执行`sleep(1);`，线程2休眠一秒。但是此时cpu并没有切换线程，那么让线程2继续执行`Count--;`，剩余票数=9。此时线程2执行完毕之后，cpu切换到线程3执行。

- 线程3进入saleTickets方法，但是此时休眠1秒的`线程1`可能开始执行。那么cpu切换到线程1，让线程1继续执行`Count--;`，那么此时剩余票数=8，线程1执行完后继续由cpu切换回线程3执行。线程3继续执行`Count--;`，此时剩余票数=7。

- 然后线程3执行完时，cpu又切换回线程1，执行`NSLog(@"卖出一张票，余票: %ld, 所在线程:%@", Count, [NSThread currentThread]);`，因为此时线程1拿到的Count并不是线程3处理后的值，而是线程1自己读取的Count=9，所以输出剩余票=8。

OK，其实后面的两个错误（1）剩余同样票 2）出现票数-1）和上面的原因是一样的。那么问题的关键就是:

- cpu随意的切换到另一个线程执行
- cpu在切换时并没有保证当前执行线程全部执行完毕
- 也即是说，当前线程在执行时，必须不能让其他线程进入执行状态，只能是等待中

###那么最终的问题简而言之:

- 多线程被cpu是`随意`切换
- 没有等待一个线程`完整`的执行一段逻辑，就已经被切换到另一个线程去执行了

而达到如上两点 ，就是经常所说的`多线程同步（多线程顺序执行）`的话题了。

## 多线程并发环境出现的问题: `临界区数据不统一`和`线程之间需要协同合作完成一件事`

- 问题1、临界区数据不统一

```
1. 多个线程并发执行，都有可能随时操作临界区数据.
2. 如果线程A读取临界区数据之后返回给用户，`同时`线程B又修改了临界区数据，这样就造成了临界区数据的不统一.
3. 引起临界区数据不统一的问题就是因为同时可以让多个线程对临界区数据进行读写.
```

此种情况造成临界区数据不同意的原因，正是多个线程`并发无序`执行。

- 问题2、线程之间需要协同合作完成一件事

```
1. 线程B进行处理时，恰好需要线程A处理完之后的结果数据，才能开始执行.
2. 那么也就是说先让线程A执行，再让线程B执行，并将线程A的结果数据传递给线程B.
```

此种情况其实也需要线程之间按照顺序执行，A->B->C->D ....。

那么也就是说，问题1与问题2，其实都涉及到了一个线程的`执行顺序`的问题，也就是说最开始都是并发，但是当某一个时刻永远都是按照`先后顺序`执行。

##在多线程之间被cpu不停的切换时，需要按照`先后顺序`来完成两件事:

- 第一件事、`临界资源的互斥访问` 
- 第二件事、`线程之间的协同合作`这两件事.

其实这两件事，都有专业的术语: 间接制约 和 直接制约。

- 间接制约、互斥访问临界数据

```
1. 线程A进入临界区进行读写
2. 线程B、C、D...等等其他线程必须在临界区外等待线程A执行完毕离开临界区
```

- 直接制约、按照顺序协同合作

```
1. 线程A将计算结果提供给线程B作进一步处理，那么线程B在线程A将数据送达之前都将处于阻塞状态
```

线程同步，为了完成两件事:

- 临界数据的互斥访问
- 多线程之间又需要协同合作（A线程等待B线程执行完毕再执行）


线程互斥，仅仅完成一件事:

- 临界数据的互斥访问


所以同步与互斥的关系如下:

- 同步包括互斥
- 互斥是同步中的一种情况


##多个线程同时执行某一个代码块时进行同步的方法:

- (1) 原子操作 
	- OSAtomic

- (2) 锁的层面: 
	- @synchronized(变量) {变量的操作代码;} 由编译器在最终编译成的c代码中添加 互斥锁
	- NSLock锁
	- NSRecursiveLock 递归锁
	- Pthread mutex lock

- (3) 信号的层面:
	- `dispatch_semaphore_t`
	- NSCondition/NSConditionLock

- (4) 队列的层面: 
	- dispatch async Serial Queue
	- dispatch barrier async Concurrent Queue

##最简单的方式: @synchronized(要同步操作的一个对象) { 变量的操作代码; }

使用 @synchronized(对象){}的作用和使用NSLock加锁同步效果是一样的，全局都使用同一个NSLock对象进行同步。

只不过@synchronized(对象){}是由编译器加上使用NSLock对象对`变量`的同步代码`[NSLock lock]和[NSLock unlock]`.

所以 @synchronized(对象){} 和 NSLock 都是属于 >>> 互斥锁，然后互斥锁的缺点:

- 必须是处于多个线程抢夺同一个数据资源时才起作用
- 全局默认只使用`一把锁`，就会造成效率低下（因为全局的同步代码块使用一把唯一的锁）


@synchronized(对象){} 使用模板，注意括号中传入的是一个对象.

```
- (void) func {

	@synchronized(self)  
	{  
		// 这段代码对多个线程使用互斥排队执行
	}  
}
```

先看个简单的例子，让一个对象的方法在多线程中依次按照顺序执行，因为有可能其中的这些方法使用的数据是来自上一个方法执行后产生的数据。

```objc
#import <Foundation/Foundation.h>

@interface Tool : NSObject

- (void)test1;
- (void)test2;
- (void)test3;
- (void)test4;

@end

@implementation Tool

- (void)test1 {
    NSLog(@"test1 method");
}

- (void)test2 {
    NSLog(@"test2 method");
}

- (void)test3 {
    NSLog(@"test3 method");
}

- (void)test4 {
    NSLog(@"test4 method");
}

@end
```

```objc
@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	
	// 主线程中创建对象
    Tool *tool = [Tool new];
    
    //线程1 先执行 Tool对象的test1方法
    dispatch_sync(dispatch_get_global_queue(0, 0), ^{
        @synchronized(tool) {
            [tool test1];
        }
    });
    
    //线程2 再执行 Tool对象的test2方法
    dispatch_sync(dispatch_get_global_queue(0, 0), ^{
        @synchronized(tool) {
            [tool test2];
        }
    });
    
    //线程3 再执行 Tool对象的test3方法
    dispatch_sync(dispatch_get_global_queue(0, 0), ^{
        @synchronized(tool) {
            [tool test3];
        }
    });
    
    //线程4 再执行 Tool对象的test4方法
    dispatch_sync(dispatch_get_global_queue(0, 0), ^{
        @synchronized(tool) {
            [tool test4];
        }
    });
}

@end
```

输出信息

```
2016-07-01 00:35:51.706 Demos[4391:62410] test1 method
2016-07-01 00:35:51.707 Demos[4391:62410] test2 method
2016-07-01 00:35:51.707 Demos[4391:62410] test3 method
2016-07-01 00:35:51.707 Demos[4391:62410] test4 method
```

可以看到`self`执行的对象的如上几个方法依次按照顺序执行。我觉得其实上面完成可以使用 `dispatch async serial queue` 来完成多线程的`顺序执行`更简单.

然后以火车卖票系统为例，看如果不使用线程同步时可能发生的错误:

```objc
//
//  ThreadDemo1.m
//  Demos
//
//  Created by xiongzenghui on 16/6/29.
//  Copyright © 2016年 xiongzenghui. All rights reserved.
//

#import "ThreadDemo1.h"

@interface ThreadDemo1 ()

@property (nonatomic, strong) dispatch_queue_t queue1;
@property (nonatomic, strong) dispatch_queue_t queue2;

@property (nonatomic, assign) NSInteger count;
@property (nonatomic, assign) NSInteger tickets;

@end

@implementation ThreadDemo1

- (instancetype)init {
    self = [super init];
    if (self) {
        _count = 0;
        _tickets = 20;
    }
    return self;
}

- (dispatch_queue_t)queue1 {
    if (!_queue1) {
        _queue1 = dispatch_queue_create("queue1", DISPATCH_QUEUE_CONCURRENT);
    }
    return _queue1;
}

- (dispatch_queue_t)queue2 {
    if (!_queue2) {
        _queue2 = dispatch_queue_create("queue2", DISPATCH_QUEUE_SERIAL);
    }
    return _queue2;
}

- (void)test {
    
    // 现在多个线程并发去写self.value
    dispatch_async(self.queue1, ^{
        [self decreateCount];
    });
    
    dispatch_async(self.queue1, ^{
        [self decreateCount];
    });

    
    dispatch_async(self.queue1, ^{
        [self decreateCount];
    });
}

- (void)decreateCount {
    while (1) {
        if (self.tickets > 0) {
            
            sleep(1);
            
            // 卖出一张票
            self.tickets--;
            
            // 当前售出票
            self.count = 20 - self.tickets;
            
            NSLog(@"当前票数是:%ld, 售出:%ld, 线程:%@", self.tickets, self.count, [NSThread currentThread]);
            
        } else {
            break;
        }
    }
}


@end
```

可能输出正确的信息

```
我就不贴结果了，就是一张一张正确的卖掉，不出现 -1票数，以及多次剩下同样的票数 两种情况.
```

也可能输出错误的信息（出现 self.tickets == -1 和 多次剩下同样的票数）

```
2016-06-29 23:50:13.177 Demos[6831:110264] 当前票数是:19, 售出:1, 线程:<NSThread: 0x7fbf19d0efc0>{number = 8, name = (null)}
2016-06-29 23:50:13.177 Demos[6831:110265] 当前票数是:17, 售出:3, 线程:<NSThread: 0x7fbf19f0a250>{number = 10, name = (null)}
2016-06-29 23:50:13.177 Demos[6831:110259] 当前票数是:18, 售出:2, 线程:<NSThread: 0x7fbf19f07f70>{number = 9, name = (null)}
2016-06-29 23:50:14.183 Demos[6831:110264] 当前票数是:16, 售出:4, 线程:<NSThread: 0x7fbf19d0efc0>{number = 8, name = (null)}
2016-06-29 23:50:14.183 Demos[6831:110265] 当前票数是:15, 售出:5, 线程:<NSThread: 0x7fbf19f0a250>{number = 10, name = (null)}
2016-06-29 23:50:14.183 Demos[6831:110259] 当前票数是:14, 售出:6, 线程:<NSThread: 0x7fbf19f07f70>{number = 9, name = (null)}
2016-06-29 23:50:15.185 Demos[6831:110265] 当前票数是:12, 售出:8, 线程:<NSThread: 0x7fbf19f0a250>{number = 10, name = (null)}
2016-06-29 23:50:15.185 Demos[6831:110259] 当前票数是:12, 售出:8, 线程:<NSThread: 0x7fbf19f07f70>{number = 9, name = (null)}
2016-06-29 23:50:15.185 Demos[6831:110264] 当前票数是:13, 售出:7, 线程:<NSThread: 0x7fbf19d0efc0>{number = 8, name = (null)}
2016-06-29 23:50:16.188 Demos[6831:110259] 当前票数是:10, 售出:10, 线程:<NSThread: 0x7fbf19f07f70>{number = 9, name = (null)}
2016-06-29 23:50:16.188 Demos[6831:110264] 当前票数是:10, 售出:10, 线程:<NSThread: 0x7fbf19d0efc0>{number = 8, name = (null)}
2016-06-29 23:50:16.188 Demos[6831:110265] 当前票数是:11, 售出:9, 线程:<NSThread: 0x7fbf19f0a250>{number = 10, name = (null)}
2016-06-29 23:50:17.193 Demos[6831:110265] 当前票数是:8, 售出:12, 线程:<NSThread: 0x7fbf19f0a250>{number = 10, name = (null)}
2016-06-29 23:50:17.193 Demos[6831:110259] 当前票数是:9, 售出:11, 线程:<NSThread: 0x7fbf19f07f70>{number = 9, name = (null)}
2016-06-29 23:50:17.193 Demos[6831:110264] 当前票数是:8, 售出:11, 线程:<NSThread: 0x7fbf19d0efc0>{number = 8, name = (null)}
2016-06-29 23:50:18.194 Demos[6831:110265] 当前票数是:7, 售出:13, 线程:<NSThread: 0x7fbf19f0a250>{number = 10, name = (null)}
2016-06-29 23:50:18.194 Demos[6831:110259] 当前票数是:6, 售出:14, 线程:<NSThread: 0x7fbf19f07f70>{number = 9, name = (null)}
2016-06-29 23:50:18.200 Demos[6831:110264] 当前票数是:5, 售出:15, 线程:<NSThread: 0x7fbf19d0efc0>{number = 8, name = (null)}
2016-06-29 23:50:19.196 Demos[6831:110259] 当前票数是:4, 售出:16, 线程:<NSThread: 0x7fbf19f07f70>{number = 9, name = (null)}
2016-06-29 23:50:19.196 Demos[6831:110265] 当前票数是:4, 售出:16, 线程:<NSThread: 0x7fbf19f0a250>{number = 10, name = (null)}
2016-06-29 23:50:19.207 Demos[6831:110264] 当前票数是:3, 售出:17, 线程:<NSThread: 0x7fbf19d0efc0>{number = 8, name = (null)}
2016-06-29 23:50:20.201 Demos[6831:110259] 当前票数是:2, 售出:18, 线程:<NSThread: 0x7fbf19f07f70>{number = 9, name = (null)}
2016-06-29 23:50:20.201 Demos[6831:110265] 当前票数是:1, 售出:19, 线程:<NSThread: 0x7fbf19f0a250>{number = 10, name = (null)}
2016-06-29 23:50:20.213 Demos[6831:110264] 当前票数是:0, 售出:20, 线程:<NSThread: 0x7fbf19d0efc0>{number = 8, name = (null)}
2016-06-29 23:50:21.204 Demos[6831:110259] 当前票数是:-1, 售出:21, 线程:<NSThread: 0x7fbf19f07f70>{number = 9, name = (null)}
2016-06-29 23:50:21.204 Demos[6831:110265] 当前票数是:-1, 售出:21, 线程:<NSThread: 0x7fbf19f0a250>{number = 10, name = (null)}
```

如上代码就是描述三个子线程并发的访问 self.count与self.tickets变量。当票数<0时，还在卖票。注意，是可能发生错误，也有可能不发生错误。就看多线程在一瞬间被cpu切换时，是否发生错误。

所以说，多线程并发执行一段代码时，虽然整体看起来是并发执行的，但是到某一个小数据/小代码访问时，仍然需要按照进入的先后顺序进行排队执行，这样才能保证临界数据的一致性。

分析如上代码发生错误的步骤:

- 如果`线程1`进入decreateCount方法时，self.ticket == 1，就是说只剩最后一张票了

- 而此时`线程1` 将要执行 `self.ticket--;`这句代码时，突然其他线程闯入执行，那么cpu立刻切换到`线程2`上
	- 那么就造成`线程1`并没有完整的执行完一个`self.ticket--;`的步骤

- 此时`线程2` 突然进入decreateCount方法
	- 而 self.ticket == 1（线程1没有执行完就被cpu切换了）
	- `线程2` 继续执行`self.ticket--;`
	- 那么此时self.ticket == 0
	- 而此时，可能cpu立马又切换回`线程1`上继续执行

- `线程1`继续执行`self.ticket--;`
	- 而此时因为 self.ticket == 0
	- 那么就造成 self.ticket == -1 的错误情况

> 所以说，是否出现错误，完全取决于cpu切换多线程时的问题。如果当线程执行完毕进行线程切换，那么这是正确的。而如果一个线程没有执行完整个步骤的代码，而临时被cpu切换，那么下一个线程进入读取到的数据就是错误的。

那么接入话题，使用 @synchronized(对象){} 阻止cpu随意的进行多线程切换，造成可能发生的多线程访问数据错误。

> 修改一、在上面发生多线程操作成员变量的decreateCount方法中，对当前对象self进行同步互斥.

```objc
- (void)decreateCount {
    while (1) {
        
        // 让编译器自动生成NSLock并加锁、解锁
        @synchronized(self) {
            if (self.tickets > 0) {
                
                sleep(1);
                
                // 卖出一张票
                self.tickets--;
                
                // 当前售出票
                self.count = 20 - self.tickets;
                
                NSLog(@"当前票数是:%ld, 售出:%ld, 线程:%@", self.tickets, self.count, [NSThread currentThread]);
                
            } else {
                break;
            }
        }
    }
}
```

这样就能正确让多线程进行:

- 线程1进入decreateCount方法后，线程2和线程3暂时不执行decreateCount方法

- 当线程1彻底离开decreateCount方法后，线程2和线程3竞争进入decreateCount方法

- 而如果线程2竞争成功进入decreateCount方法后，那么线程3仍然处于等待，而执行完的线程1也处于等待

> 修改二、不野蛮的对整个decreateCount方法同步，而针对需要同步的成员变量的setter/getter方法中进行同步.（`经证实是无效的...`）

```objc
#import "ThreadDemo1.h"

@interface ThreadDemo1 () {
    NSInteger _count;
    NSInteger _tickets;
}

@property (nonatomic, strong) dispatch_queue_t queue1;
@property (nonatomic, strong) dispatch_queue_t queue2;

@property (nonatomic, assign) NSInteger count;
@property (nonatomic, assign) NSInteger tickets;

@end

@implementation ThreadDemo1

- (instancetype)init {
    self = [super init];
    if (self) {
        _count = 0;
        _tickets = 20;
    }
    return self;
}

- (dispatch_queue_t)queue1 {
    if (!_queue1) {
        _queue1 = dispatch_queue_create("queue1", DISPATCH_QUEUE_CONCURRENT);
    }
    return _queue1;
}

- (dispatch_queue_t)queue2 {
    if (!_queue2) {
        _queue2 = dispatch_queue_create("queue2", DISPATCH_QUEUE_SERIAL);
    }
    return _queue2;
}

- (void)test {
    
    // 现在多个线程并发去写self.value
    dispatch_async(self.queue1, ^{
        [self decreateCount];
    });
    
    dispatch_async(self.queue1, ^{
        [self decreateCount];
    });

    
    dispatch_async(self.queue1, ^{
        [self decreateCount];
    });
}

- (void)setCount:(NSInteger)count {
    @synchronized(self) {
        _count = count;
    }
}

- (void)setTickets:(NSInteger)tickets {
    @synchronized(self) {
        _tickets = tickets;
    }
}

- (NSInteger)count {
    @synchronized(self) {
        return _count;
    }
}

- (NSInteger)tickets {
    @synchronized(self) {
        return _tickets;
    }
}

- (void)decreateCount {
    while (1) {
        if (self.tickets > 0) {
            
            sleep(1);
            
            // 卖出一张票
            self.tickets--;
            
            // 当前售出票
            self.count = 20 - self.tickets;
            
            NSLog(@"当前票数是:%ld, 售出:%ld, 线程:%@", self.tickets, self.count, [NSThread currentThread]);
            
        } else {
            break;
        }
    }
}

@end
```

开始以为这么做能行，运行后发现还是错误的。想了想，他虽然能够控制多线程对成员变量的操作，但是不能够控制多线程执行decreateCount方法时的线程切换的时机。也就是说无法让线程1未执行完时，不让其他线程进入decreateCount方法。

所以，对于使用@synchronized(对象){}或锁同步时，主要是包裹发生多线程同时操作某一个数据的`整个代码块`，而不仅仅只是`某一个变量的setter与getter方法实现`。

##使用NSLock来同步多线程并发执行某一段代码访问临界变量

###主要的Api

```
1. lock，加锁
2. unlock，解锁
3. tryLock，尝试加锁，获取成功后才会加锁，否则直接返回NO。当处于while/for等循环中进行加锁时，需要判断是否加锁，否则会出现死锁
4. lockBeforeDate:，在指定的date之前暂时阻塞线程（如果没有获取锁的话），如果到期还没有获取锁，则线程被唤醒，函数立即返回NO
```

代码使用模板

```objc
NSLock *theLock = [[NSLock alloc] init];

- (void) func { 

	[theLock lock];

	//线程互斥执行的代码
	//...

	[theLock unLock];
	
}
```

卖票的例子、使用NSLock 让 NSThread `并发` 访问一个函数时 进行多线程同步.

```objc
@interface ViewController () {
    
    int tickets;//总票数
    int count;//售出票数
    
    NSThread* ticketsThreadone;
    NSThread* ticketsThreadtwo;
    NSThread* ticketsThreadthree;
    NSThread* ticketsThreadfour;
    
    NSLock *theLock;
}

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self demo5]; 
}

- (void)demo5 {
    
    //1.
    tickets = 10;
    count = 0;
    
    //2.
    theLock = [[NSLock alloc] init];
    
    conditon = [[NSCondition alloc] init];
    
    //3.
    ticketsThreadone = [[NSThread alloc] initWithTarget:self selector:@selector(thredEntry) object:nil];
    [ticketsThreadone setName:@"Thread-1"];
    [ticketsThreadone start];
    
    //4.
    ticketsThreadtwo = [[NSThread alloc] initWithTarget:self selector:@selector(thredEntry) object:nil];
    [ticketsThreadtwo setName:@"Thread-2"];
    [ticketsThreadtwo start];
    
    //5.
    ticketsThreadthree = [[NSThread alloc] initWithTarget:self selector:@selector(thredEntry) object:nil];
    [ticketsThreadthree setName:@"Thread-3"];
    [ticketsThreadthree start];
    
    //6.
    ticketsThreadfour = [[NSThread alloc] initWithTarget:self selector:@selector(thredEntry) object:nil];
    [ticketsThreadfour setName:@"Thread-4"];
    [ticketsThreadfour start];
}

/**
 *	多个线程 并发执行的函数
 */
- (void)thredEntry{
    
    while (TRUE) {
        
        //1. 进入临界资源上锁
        [theLock lock];
        
        //2. 进入临界资源区，操作临街资源
        if(tickets > 0){
            
            [NSThread sleepForTimeInterval:0.09];
            count++;
            tickets--;
            NSLog(@"当前票数是:%d,售出:%d,线程名:%@",tickets,count,[[NSThread currentThread] name]);
        }else{
            break;
        }
        
        //3. 出去临界资源解锁
        [theLock unlock];
    }
}

@end
```

输出内容

```
Demo[10182:100527] 当前票数是:9,售出:1,线程名:Thread-1
Demo[10182:100528] 当前票数是:8,售出:2,线程名:Thread-2
Demo[10182:100530] 当前票数是:7,售出:3,线程名:Thread-4
Demo[10182:100529] 当前票数是:6,售出:4,线程名:Thread-3
Demo[10182:100527] 当前票数是:5,售出:5,线程名:Thread-1
Demo[10182:100528] 当前票数是:4,售出:6,线程名:Thread-2
Demo[10182:100530] 当前票数是:3,售出:7,线程名:Thread-4
Demo[10182:100529] 当前票数是:2,售出:8,线程名:Thread-3
Demo[10182:100527] 当前票数是:1,售出:9,线程名:Thread-1
Demo[10182:100528] 当前票数是:0,售出:10,线程名:Thread-2
```

从代码上可以看出和之前的@synchronize(){}进行多线程同步没有太大的区别。

注意:上面的例子看输出的线程名，可以看到多线程执行的顺序是`无序`的，那如果希望多线程是`顺序`执行了？

###如果是使用GCD的话，直接通过 `串行队列` 即可。但是在NSThread的环境，只能通过使用 `NSCondition` 让 NSThread `串行` 访问一个函数时 进行多线程同步。

```objc
@interface ViewController ()  {
    
    int tickets;//总票数
    int count;//售出票数
    
    // 三个访问数据的多线程
    NSThread* ticketsThreadone;
    NSThread* ticketsThreadtwo;
    NSThread* ticketsThreadthree;
    
    // 这个线程是控制上面三个线程串行的线程
    NSThread* ticketsThreadfour;
    
    // 条件锁，用来控制多线程串行执行
    NSCondition *conditon;
}

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self demo5]; 
}

- (void)demo5 {
    
    //1.
    tickets = 10;
    count = 0;
    
    //3.
    conditon = [[NSCondition alloc] init];
    
    //4.
    ticketsThreadone = [[NSThread alloc] initWithTarget:self selector:@selector(threadEntry) object:nil];
    [ticketsThreadone setName:@"Thread-1"];
    [ticketsThreadone start];
    
    //5.
    ticketsThreadtwo = [[NSThread alloc] initWithTarget:self selector:@selector(threadEntry) object:nil];
    [ticketsThreadtwo setName:@"Thread-2"];
    [ticketsThreadtwo start];
    
    //6.
    ticketsThreadthree = [[NSThread alloc] initWithTarget:self selector:@selector(threadEntry) object:nil];
    [ticketsThreadthree setName:@"Thread-3"];
    [ticketsThreadthree start];
    
    //7. 用于唤醒前面三个子线程的 工作线程
    ticketsThreadfour = [[NSThread alloc] initWithTarget:self selector:@selector(threadNotify) object:nil];
    [ticketsThreadfour setName:@"Thread-4"];
    [ticketsThreadfour start];
}

/**
 *	多个线程 并发执行 的代码块
 */
- (void)threadEntry{
    
    while (TRUE) {
        
        // 线程1、线程2、线程3 依次进入并获取到锁
        [conditon lock];
        
        //【关键代码】 线程1、线程2、线程3 排队等待信号才能往下执行
        [conditon wait];
        
        // 排在第一个位置的线程收到信号进入就绪状态，并获取到锁开始执行，并让信号量-1
        if(tickets > 0){

            [NSThread sleepForTimeInterval:0.09];
            count++;
            tickets--;
            NSLog(@"当前票数是:%d,售出:%d,线程:%@",tickets,count,[NSThread currentThread]);
        }else{
            break;
        }
        
        // 线程执行完毕之后释放锁，让其他排队的线程可以获得锁
        [conditon unlock];
    }
}

///线程4的线程体
-(void)threadNotify {
    
    while (YES) {
        
        // 线程4获取到锁，并开始执行（因为上面三个线程此时全部处于挂起）
        [conditon lock];
        
        //【关键代码】 发送一个信号，通知排在第一个的线程进入准备状态，并立刻往下执行
        [conditon signal];
        
        // 线程4解除锁，让其他线程可以获得锁
        [conditon unlock];
    }
}

@end
```

输出内容

```
2016-07-03 18:03:15.373 Demos[16235:230334] 当前票数是:9,售出:1,线程:<NSThread: 0x7fbaf1519250>{number = 2, name = Thread-1}
2016-07-03 18:03:15.470 Demos[16235:230335] 当前票数是:8,售出:2,线程:<NSThread: 0x7fbaf1725e70>{number = 3, name = Thread-2}
2016-07-03 18:03:15.561 Demos[16235:230336] 当前票数是:7,售出:3,线程:<NSThread: 0x7fbaf1724070>{number = 4, name = Thread-3}
2016-07-03 18:03:15.654 Demos[16235:230334] 当前票数是:6,售出:4,线程:<NSThread: 0x7fbaf1519250>{number = 2, name = Thread-1}
2016-07-03 18:03:15.747 Demos[16235:230335] 当前票数是:5,售出:5,线程:<NSThread: 0x7fbaf1725e70>{number = 3, name = Thread-2}
2016-07-03 18:03:15.840 Demos[16235:230336] 当前票数是:4,售出:6,线程:<NSThread: 0x7fbaf1724070>{number = 4, name = Thread-3}
2016-07-03 18:03:15.934 Demos[16235:230334] 当前票数是:3,售出:7,线程:<NSThread: 0x7fbaf1519250>{number = 2, name = Thread-1}
2016-07-03 18:03:16.025 Demos[16235:230335] 当前票数是:2,售出:8,线程:<NSThread: 0x7fbaf1725e70>{number = 3, name = Thread-2}
2016-07-03 18:03:16.123 Demos[16235:230336] 当前票数是:1,售出:9,线程:<NSThread: 0x7fbaf1724070>{number = 4, name = Thread-3}
2016-07-03 18:03:16.218 Demos[16235:230334] 当前票数是:0,售出:10,线程:<NSThread: 0x7fbaf1519250>{number = 2, name = Thread-1}
```

可以看到三个线程是按照顺序进行执行的，并且对于售票的代码和票变量的访问逻辑都是独立完成之后才会切换到其他的线程。

从上面使用NSCondition来控制三个线程串行执行的代码来看:

- wait方法、就是让所有等待的线程等待一个信号量，让线程进入阻塞
- signal方法、就是让NSCondition对象产生一个信号量，表示可以让`排在第一个`的线程进入准备，然后lock获取到锁之后进入执行

再看个使用NSCondition模拟生产者与消费者的代码:

```objc
#import "ThreadDemo2.h"

@interface ThreadDemo2 ()

@property(strong, nonatomic) NSCondition *condition;
@property(strong, nonatomic) NSMutableArray *products;

@end

@implementation ThreadDemo2

- (instancetype)init {
    self = [super init];
    if (self) {
        _products = [[NSMutableArray alloc] init];
        _condition = [[NSCondition alloc] init];
    }
    return self;
}

- (void)test {
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        [self product];
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        [self customer];
    });
}

// 生产线程
- (void)product
{
    while (1) {
        
        // 让消费线程处于阻塞
        [self.condition lock];
        
        // 完成生产，发送一个信号，准备唤醒消费线程
        NSLog(@"%@:已经生产一个产品", [NSThread currentThread]);
        [_products addObject:[[NSObject alloc] init]];
        [self.condition signal];
        
        // 解除锁，让其消费线程可以执行customer方法
        [self.condition unlock];
    }
}

// 消费线程
- (void)customer
{
    while (1) {
        // 让生产线程处于阻塞
        [self.condition lock];
        
        // 如果没有产品，释放锁让 生产线程生产产品
        while ([_products count] == 0) {
            NSLog(@"wait for products");
            [_condition wait];
        }
        
        // 消费一个产品
        [_products removeObjectAtIndex:0];
        NSLog(@"%@:已经消费一个产品", [NSThread currentThread]);
        
        // 接触锁，让生产线程执行product方法
        [self.condition unlock];
    }
}

@end
```

还有几种条件所、分不锁、递归锁就不贴了，不是太常使用就懒得去看了。

###在for/while等循环中加锁时，需要使用`tryLock`，不能使用`lock`，否则会出现线程死锁:

```objc
while (!finish) {

    //TODO: 防止在循环中使用pthread_mutex_lock()出现线程死锁，前面的锁没有解除，又继续加锁
    //pthread_mutex_trylock(): 尝试获取锁，获取成功之后才加锁，否则不执行加锁
    
    if (pthread_mutex_trylock(&_lock) == 0) {//获取锁成功，并执行加锁
    
        // 缓存数据读写
        //.....
        
        //解锁
        pthread_mutex_unlock(&_lock);
        
    } else {
    
        //获取锁失败，让线程休眠10秒后再执行解锁
        usleep(10 * 1000);
    }
    
    //解锁
    pthread_mutex_unlock(&_lock);
}
```

###FMDB大量使用`dispatch_sync(){}`，那么他用来防止死锁的办法

主要是如下函数，用来给disaptch queue绑定一个标示对象，来区分是否是指定的dispatch queue

```c
// 【重要】给queue绑定的一个标示
dispatch_queue_set_specific(dispatch_queue_t, 唯一标示, 绑定的标示对象, NULL);
```

```c
// 【重要】根据当前标示取出当前的queue，判断是不是我们的queue
NSObject *value = (__bridge NSObject *)(dispatch_get_specific(唯一标示));
if (value == ctx) {
    NSLog(@">>>>> 是我们的queue");
} else {
    NSLog(@">>>>> 不是我们的queue");
}
```

下面是demo实例

```objc
@implementation ViewController

- (void)testORM2_DeadLock {
    
    // 将给队列用来绑定一个标示值的key
    static void *queueKey1 = "queueKey1";
    
    // 我们自己的队列
    dispatch_queue_t queue1 = dispatch_queue_create(queueKey1, DISPATCH_QUEUE_SERIAL);
    
    // 给队列要绑定的标示值
    static NSObject *ctx = NULL;
    ctx = [NSObject new];
    
    // 【重要】给queue绑定的一个标示
    dispatch_queue_set_specific(queue1, queueKey1, (__bridge void *)(ctx), NULL);
    
    // 【重要】根据当前标示取出当前的queue，判断是不是我们的queue
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSObject *value = (__bridge NSObject *)(dispatch_get_specific(queueKey1));
        if (value == ctx) {
            NSLog(@"1>>>>> 是我们的queue");
        } else {
            NSLog(@"1>>>>> 不是我们的queue");
            
            dispatch_async(queue1, ^{
                NSObject *value = (__bridge NSObject *)(dispatch_get_specific(queueKey1));
                if (value == ctx) {
                    NSLog(@"2>>>>> 是我们的queue");
                } else {
                    NSLog(@"2>>>>> 不是我们的queue");
                }
            });
        }
    });
}

@end
```

运行结果

```
2016-12-07 16:12:06.407 XZHRuntimeDemo[16882:1394212] 1>>>>> 不是我们的queue
2016-12-07 16:12:06.407 XZHRuntimeDemo[16882:1393906] 2>>>>> 是我们的queue
```

这样来防止出现`dispatch_sync(queue,{})`处于`线程A`上，而又`dispatch_sync(线程A的queue,){}`造成线程死锁。


##在iOS系统中提供的 GCD dispatch queue 的种类

> 不管什么线程队列，其数据结构都是 `队列` ，也就是 `先进先出` 是其最基本的原则。

- (1) 全局并发队列: 根据四种线程权限又分为`四种`全局并发队列

四种线程执行权限

```objc
#define DISPATCH_QUEUE_PRIORITY_HIGH 2
#define DISPATCH_QUEUE_PRIORITY_DEFAULT 0
#define DISPATCH_QUEUE_PRIORITY_LOW (-2)
#define DISPATCH_QUEUE_PRIORITY_BACKGROUND INT16_MIN
```

对应的四种权限的线程队列获取方式

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

##就是说对于`全局并发队列`来执行 `dispatch_suspend(), dispatch_resume(), dispatch_set_context()` 这三种操作是没有任何效果的。

刚开始我认为如上都是一个队列，只是权限不同而已，这是错误的理解。运行上面四局代码结果，可以看到这个四个指针指向的内存地址都是`不同`的

```
(lldb)
(OS_dispatch_queue_root *) queue1 = 0x0000000110b493c0
(OS_dispatch_queue_root *) queue2 = 0x0000000110b49240
(OS_dispatch_queue_root *) queue3 = 0x0000000110b490c0
(OS_dispatch_queue_root *) queue4 = 0x0000000110b48f40
(lldb) 
```

- (2) 第五种队列、程序进程内唯一的一个 `主线程队列`，也是最重要的一个线程队列

```c
dispatch_get_main_queue()
```

主线程队列是串行按照顺序一个一个执行任务。

- (3) 第六种队列、自定义的`串行顺序`线程队列

```c
#define DISPATCH_QUEUE_SERIAL NULL
```

```c
dispatch_queue_create("队列的名字", NULL);
```

和主线程队列一样按照顺序一个一个执行任务，但不同的是最终是运行在一个唯一的子线程上

- (4) 第七种队列、自定义的`并发无序`线程队列

```c
dispatch_queue_create("线程队列名字", DISPATCH_QUEUE_CONCURRENT);
```

大体上和 global queue 差不多的。

###`dispatch_get_global_queue();`获取全局队列在iOS8之后提出的`服务质量`来描述与iOS8之前使用`优先级`来描述的区别

```c
dispatch_queue_t
dispatch_get_global_queue(long identifier, unsigned long flags);
```

对于第一个参数 `identifier` 苹果文档给出的注释:

```c
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
```

从注释中可以得到:

### identifier参数，将会代表`两种`类型枚举值:
	
- (1) 服务质量: iOS8及之`后`才能使用

```
a quality of service class defined in qos_class_t
```
	
- (2) 队列优先级: iOS8之`前`都可以使用

```
a priority defined in dispatch_queue_priority_t
```
	
	
###从注释中可以看到，苹果更推荐使用 `服务质量` 来描述线程，而不是 `优先级`，原因是如下:

| 队列描述（服务质量 or 优先级）| 最终执行线程描述 | 
| :-------------: |:-------------:| 
| 服务质量 Quality Of Service | thread.qualityOfService = NSQualityOfServiceUserInteractive | 
| 优先级 Queue Priority | thread.threadPriority = 0.5 | 
	
	
对于优先级，最终被执行的线程是通过一个`数值`来表示，可能有时候根本无法分别0.5和0.6之间有什么区别。但是对于服务质量就很容易区分了。
	
###quality of service （QOS...）服务质量

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
	QOS_CLASS_DEFAULT = 0x15, // -1，当没有 QoS信息时默认使用，苹果建议不要使用这个级别
	QOS_CLASS_UNSPECIFIED = 0x00,//0，这个是一个标记，没啥实际作用
} qos_class_t;
```

可以看到，在iOS8之后推出的队列`服务质量`，可以指定更细腻的线程执行权限，肯定比基于`优先级`的方式好多了。服务质量，核心点在于对于`线程执行时间的长短`进行划分，可以控制的更细。

### thread queue priority 

```c
#define DISPATCH_QUEUE_PRIORITY_HIGH 2
#define DISPATCH_QUEUE_PRIORITY_DEFAULT 0
#define DISPATCH_QUEUE_PRIORITY_LOW (-2)
#define DISPATCH_QUEUE_PRIORITY_BACKGROUND INT16_MIN
```

在iOS8以下，可以使用如上这些宏代表的队列优先级。

### 服务质量 与 队列优先级 之间具有映射关系的枚举值

| Priority 优先级 | QOS 服务质量 |
| :-------------: |:-------------:| 
| `DISPATCH_QUEUE_PRIORITY_HIGH` | `QOS_CLASS_USER_INITIATED` |
|  `DISPATCH_QUEUE_PRIORITY_DEFAULT` | `QOS_CLASS_DEFAULT` |
| `DISPATCH_QUEUE_PRIORITY_LOW` | `QOS_CLASS_UTILITY` |
| `DISPATCH_QUEUE_PRIORITY_BACKGROUND` | `QOS_CLASS_BACKGROUND` |


###对于QOS中的`NSQualityOfServiceUserInteractive` 在 QueuePriority中，是没有与之对应的枚举值。


```objc
- (void)testQOSAndPriority {
    
    // 服务质量，获取的全局队列
    dispatch_queue_t USER_INTERACTIVE = dispatch_get_global_queue(QOS_CLASS_USER_INTERACTIVE, 0);// 线程优先级无法获取的全局队列
    dispatch_queue_t USER_INITIATED = dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0);
    dispatch_queue_t UTILITY = dispatch_get_global_queue(QOS_CLASS_UTILITY, 0);
    dispatch_queue_t BACKGROUND = dispatch_get_global_queue(QOS_CLASS_BACKGROUND, 0);
    dispatch_queue_t DEFAULT = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
    
    // 优先级，获取的全局队列
    dispatch_queue_t PRIORITY_HIGH = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);
    dispatch_queue_t PRIORITY_LOW = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);
    dispatch_queue_t PRIORITY_BACKGROUND = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);
    dispatch_queue_t PRIORITY_DEFAULT = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
}
```

运行结果

```
(OS_dispatch_queue_root *) USER_INTERACTIVE = 0x0000000111a5d540
(OS_dispatch_queue_root *) USER_INITIATED = 0x0000000111a5d3c0
(OS_dispatch_queue_root *) UTILITY = 0x0000000111a5d0c0
(OS_dispatch_queue_root *) BACKGROUND = 0x0000000111a5cf40
(OS_dispatch_queue_root *) DEFAULT = 0x0000000111a5d240

(OS_dispatch_queue_root *) PRIORITY_HIGH = 0x0000000111a5d3c0
(OS_dispatch_queue_root *) PRIORITY_LOW = 0x0000000111a5d0c0
(OS_dispatch_queue_root *) PRIORITY_BACKGROUND = 0x0000000111a5cf40
(OS_dispatch_queue_root *) PRIORITY_DEFAULT = 0x0000000111a5d240
```

从运行结果可以看出来，真的是有对应关系的，除了`USER_INTERACTIVE `没有对应的。

###两种类型下对于default队列有区别的:

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

- (2) QueuePriority，貌似我们大部分情况下都在使用`DEFAULT`，具体注释如下

```c
the queue will be scheduled for execution after all high priority queues have been scheduled,
but before any low priority queues have been scheduled.

（当所有的高优先级队列调度完毕，再进行default级别队列调度，但是比一些low优先级队列先调度）
```

###根据队列`优先级`如何调度任务分配线程执行，以及最终分配到某一个具体子线程的`线程优先级`值是多少

```c
dispatch_queue_t PRIORITY_HIGH = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);
dispatch_queue_t PRIORITY_LOW = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);
dispatch_queue_t PRIORITY_BACKGROUND = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);
dispatch_queue_t PRIORITY_DEFAULT = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
dispatch_async(PRIORITY_BACKGROUND, ^{
    NSLog(@"PRIORITY_BACKGROUND task >>>> thread priority = %lf", [NSThread currentThread].threadPriority);
});
    
dispatch_async(PRIORITY_LOW, ^{
    NSLog(@"PRIORITY_LOW task >>>> thread priority = %lf", [NSThread currentThread].threadPriority);
});
    
dispatch_async(PRIORITY_DEFAULT, ^{
    NSLog(@"PRIORITY_DEFAULT task >>>> thread priority = %lf", [NSThread currentThread].threadPriority);
});
    
dispatch_async(PRIORITY_HIGH, ^{
    NSLog(@"PRIORITY_HIGH task >>>> thread priority = %lf", [NSThread currentThread].threadPriority);
});
```

输出结果

```
2016-08-14 15:49:35.944 Demos[10548:166948] PRIORITY_HIGH task >>>> thread priority = 0.500000
2016-08-14 15:49:35.944 Demos[10548:166947] PRIORITY_DEFAULT task >>>> thread priority = 0.500000
2016-08-14 15:49:35.944 Demos[10548:166931] PRIORITY_LOW task >>>> thread priority = 0.500000
2016-08-14 15:49:35.945 Demos[10548:166929] PRIORITY_BACKGROUND task >>>> thread priority = 0.000000
```

所以在大部分情况下，使用队列优先级时，队列进行调度的顺序 : `high queue > defalt queue > low queue > backgroud queue`。但是可以看到线程的`threadPriority`值是 `0.5 或 0`，好像和`队列的优先级`扯不上太多的关系。


###根据队列`服务质量`如何调度任务分配线程执行的

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

然后还可以看到线程的`qualityOfService`值依次为: `33` `25` `-1` `17` `9` 刚好就是 `qos_class_t`中5个枚举值的十进制数值。

从这里可以看到，使用`服务质量`来描述队列的调度级别绝对是比使用`优先级`描述好多了，因为最终子线程的执行级别的数值很清楚。不像`threadPriority`为 `0.5` `0.0` `0.2`...不能够很清楚理解有什么区别。

###通过优先级获取的全局队列，最终执行线程的服务质量`quality`值有吗？

```c
dispatch_queue_t PRIORITY_HIGH = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);
dispatch_queue_t PRIORITY_LOW = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);
dispatch_queue_t PRIORITY_BACKGROUND = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);
dispatch_queue_t PRIORITY_DEFAULT = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

dispatch_async(PRIORITY_BACKGROUND, ^{
    NSLog(@"PRIORITY_BACKGROUND task >>>> thread quality = %ld", [NSThread currentThread].qualityOfService);
});

dispatch_async(PRIORITY_LOW, ^{
    NSLog(@"PRIORITY_LOW task >>>> thread quality = %ld", [NSThread currentThread].qualityOfService);
});

dispatch_async(PRIORITY_DEFAULT, ^{
    NSLog(@"PRIORITY_DEFAULT task >>>> thread quality = %ld", [NSThread currentThread].qualityOfService);
});

dispatch_async(PRIORITY_HIGH, ^{
    NSLog(@"PRIORITY_HIGH task >>>> thread quality = %ld", [NSThread currentThread].qualityOfService);
});
```

运行结果

```
2016-08-14 16:20:31.524 Demos[12301:195566] PRIORITY_HIGH task >>>> thread quality = 25
2016-08-14 16:20:31.524 Demos[12301:195534] PRIORITY_DEFAULT task >>>> thread quality = -1
2016-08-14 16:20:31.524 Demos[12301:195529] PRIORITY_LOW task >>>> thread quality = 17
2016-08-14 16:20:31.525 Demos[12301:195532] PRIORITY_BACKGROUND task >>>> thread quality = 9
```

可以看到，最终线程的`quality`值也是存在的，并且和`qos_class_t`的枚举值一一对应。

###所以说iOS8系统会根据如下映射关系会将 `传入的 dispatch queue priority` 替换成对应的 `quality of service`:

| QOS 服务质量 | Priority 优先级 |
| :-------------: |:-------------:| 
| `DISPATCH_QUEUE_PRIORITY_HIGH` | `QOS_CLASS_USER_INITIATED` |
|  `DISPATCH_QUEUE_PRIORITY_DEFAULT` | `QOS_CLASS_DEFAULT` |
| `DISPATCH_QUEUE_PRIORITY_LOW` | `QOS_CLASS_UTILITY` |
| `DISPATCH_QUEUE_PRIORITY_BACKGROUND` | `QOS_CLASS_BACKGROUND` |

##iOS8之前，给自己创建的队列设置优先级

```c
//1. 指定一个long类型的优先级数值
    dispatch_queue_priority_t identifier = DISPATCH_QUEUE_PRIORITY_DEFAULT;

//2. 先随便创建一个串行队列（没有指定队列优先级的）
dispatch_queue_t queue = dispatch_queue_create("haha", DISPATCH_QUEUE_SERIAL);

//3. 将dispatch_set_target_queue()函数中，第二个队列的优先级，复制给第一个队列
dispatch_set_target_queue(queue, dispatch_get_global_queue(identifier, 0));
```

无法直接在创建队列时，指定队列的优先级，只能通过`dispatch_set_target_queue()`函数进行优先级复制。

注意，系统创建出来的`4种全局并发队列`、`主线程队列`他们的优先级都是已经设置好了的。


###iOS8之后，可以直接在创建队列时，指定服务质量来描述队列的调度级别

```c
//1. 服务质量
qos_class_t qosClass = QOS_CLASS_DEFAULT;

//2. 创建串行队列，并给队列指定服务质量
dispatch_queue_attr_t queue_attr = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL, qosClass, 0);

//3. 返回创建完毕的队列
dispatch_queue_t queue = dispatch_queue_create("haha", queue_attr);
```

###在iOS8之后用来描述队列调度快慢使用`QOS_CLASS`是苹果推荐的，而在iOS8之前只能使用`DISPATCH_QUEUE_PRIORITY`来描述，这里就涉及一个如何在iOS8前后进行适配

首先，`QOS_CLASS`中服务质量类型 与 `DISPATCH_QUEUE_PRIORITY`中定义的调度优先级对应关系:

| QOS 服务质量 | Priority 优先级 |
| :-------------: |:-------------:| 
| `DISPATCH_QUEUE_PRIORITY_HIGH` | `QOS_CLASS_USER_INITIATED` |
|  `DISPATCH_QUEUE_PRIORITY_DEFAULT` | `QOS_CLASS_DEFAULT` |
| `DISPATCH_QUEUE_PRIORITY_LOW` | `QOS_CLASS_UTILITY` |
| `DISPATCH_QUEUE_PRIORITY_BACKGROUND` | `QOS_CLASS_BACKGROUND` |

注意对于`qos_class_t`还有一个OC版本的枚举定义:

```c
typedef NS_ENUM(NSInteger, NSQualityOfService) {
    NSQualityOfServiceUserInteractive = 0x21,
    NSQualityOfServiceUserInitiated = 0x19,
    NSQualityOfServiceUtility = 0x11, 
    NSQualityOfServiceBackground = 0x09,
    NSQualityOfServiceDefault = -1
} NS_ENUM_AVAILABLE(10_10, 8_0);
```

尝试`qos_class_t` 与 `NSQualityOfService` 是否有区别:

```c
dispatch_queue_t USER_INTERACTIVE = dispatch_get_global_queue(QOS_CLASS_USER_INTERACTIVE, 0);
dispatch_queue_t USER_INITIATED = dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0);
dispatch_queue_t UTILITY = dispatch_get_global_queue(QOS_CLASS_UTILITY, 0);
dispatch_queue_t BACKGROUND = dispatch_get_global_queue(QOS_CLASS_BACKGROUND, 0);
dispatch_queue_t DEFAULT = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);

dispatch_queue_t USER_INTERACTIVE_oc = dispatch_get_global_queue(NSQualityOfServiceUserInteractive, 0);
dispatch_queue_t USER_INITIATED_oc = dispatch_get_global_queue(NSQualityOfServiceUserInitiated, 0);
dispatch_queue_t UTILITY_oc = dispatch_get_global_queue(NSQualityOfServiceUtility, 0);
dispatch_queue_t BACKGROUND_oc = dispatch_get_global_queue(NSQualityOfServiceBackground, 0);
dispatch_queue_t DEFAULT_oc = dispatch_get_global_queue(NSQualityOfServiceDefault, 0);
```

运行结果

```
(OS_dispatch_queue_root *) USER_INTERACTIVE = 0x000000010fd94540
(OS_dispatch_queue_root *) USER_INITIATED = 0x000000010fd943c0
(OS_dispatch_queue_root *) UTILITY = 0x000000010fd940c0
(OS_dispatch_queue_root *) BACKGROUND = 0x000000010fd93f40
(OS_dispatch_queue_root *) DEFAULT = 0x000000010fd94240

(OS_dispatch_queue_root *) USER_INTERACTIVE_oc = 0x000000010fd94540
(OS_dispatch_queue_root *) USER_INITIATED_oc = 0x000000010fd943c0
(OS_dispatch_queue_root *) UTILITY_oc = 0x000000010fd940c0
(OS_dispatch_queue_root *) BACKGROUND_oc = 0x000000010fd93f40
(dispatch_queue_t) DEFAULT_oc = nil
```

那么现在可以明白YYDispatchQueuePool中对`NSQualityOfServiceDefault`转换成`QOS_CLASS_DEFAULT`:

- (1) iOS8以及之后的系统，苹果推荐使用 `qos_class 服务质量` 来代替 `queue_priority 优先级` 来描述队列。而现在大多数iOS系统都是9.0+的，所以优先使用qos，但是仍需要适配priority.

- (2) 在iOS8之前的系统，`NSQualityOfServiceDefault`获取不到队列实例。需要将`QOS_CLASS_DEFAULT`转换成`DISPATCH_QUEUE_PRIORITY_DEFAULT`，再获取对应的队列实例.

###那么如何对iOS8之后和iOS8之前，将`队列优先级`与`队列服务质量`进行一个适配了？并且在目前不部分iOS系统都是`9.x`，所以更应该主要使用`服务质量`。

YYDispatchQueuePool提供了两个c函数，用于将NSQualityOfService类型，根据当前iOS系统版本，转换成`队列优先级` 或 `队列服务质量`。

- (1) iOS8之前、NSQualityOfService >>>> 队列优先级 `dispatch_queue_priority_t` 

```c
static inline dispatch_queue_priority_t NSQualityOfServiceToDispatchPriority(NSQualityOfService qos) {
    switch (qos) {
    
    	// 用户界面行为都是`高优先级`，因为NSQualityOfServiceUserInteractive没有与之对应的 队列优先级
        case NSQualityOfServiceUserInteractive: return DISPATCH_QUEUE_PRIORITY_HIGH;
        case NSQualityOfServiceUserInitiated: return DISPATCH_QUEUE_PRIORITY_HIGH;
        
        // ServiceUtility == PRIORITY_LOW
        case NSQualityOfServiceUtility: return DISPATCH_QUEUE_PRIORITY_LOW;
        
        // ServiceBackground == PRIORITY_BACKGROUND
        case NSQualityOfServiceBackground: return DISPATCH_QUEUE_PRIORITY_BACKGROUND;
        
        // ServiceDefault == PRIORITY_DEFAULT
        case NSQualityOfServiceDefault: return DISPATCH_QUEUE_PRIORITY_DEFAULT;
        
        // 默认返回 PRIORITY_DEFAULT
        default: return DISPATCH_QUEUE_PRIORITY_DEFAULT;
    }
}
```

- (2) iOS8之后、NSQualityOfService >>>> GCD版本的 `qos_class_t`

```c
static inline qos_class_t NSQualityOfServiceToQOSClass(NSQualityOfService qos) {
    switch (qos) {
        case NSQualityOfServiceUserInteractive: return QOS_CLASS_USER_INTERACTIVE;
        case NSQualityOfServiceUserInitiated: return QOS_CLASS_USER_INITIATED;
        case NSQualityOfServiceUtility: return QOS_CLASS_UTILITY;
        case NSQualityOfServiceBackground: return QOS_CLASS_BACKGROUND;
        case NSQualityOfServiceDefault: return QOS_CLASS_DEFAULT;
        default: return QOS_CLASS_UNSPECIFIED;
    }
}
```

###那么结合上面两个c转换函数，可以封装出一个用于适配iOS8之前和之后，获取`自己创建`的串行队列实例的方法:

因为获取系统的线程队列，比如: global queue。因为目前仍然需要适配iOS6、iOS7这些系统，所以不能够使用传入QOS来获取队列（只能够`iOS8+`的系统中使用）。

所以的话，目前来说在使用`dispatch_get_global_queue(long identifier, unsigned long flags);`获取系统全局队列实例时，还是要使用`优先级`（2， 0 ， -2， INT16_MIN）的方式，要失去适配iOS6、iOS7。

但是如果是使用`自己创建`的线程队列时，那么就可以考虑使用`服务质量 qos_class_t`的方式来描述队列调度级别，但仍然需要考虑适配 `iOS6/iOS7/iOS8+` 的问题。


如下是摘录自YYDispatchQueuePool中的片段代码，通过传入`NSQualityOfService枚举值`来根据当前iOS系统版本进行划分:

- (1) iOS8以及之后的系统版本，使用 `QOS` 描述创建的队列
- (2) iOS之前的系统，使用 `dispatch_queue_priority_t` 描述创建的队列

```objc
static inline dispatch_queue_priority_t NSQualityOfServiceToDispatchPriority(NSQualityOfService qos) {
    switch (qos) {
        case NSQualityOfServiceUserInteractive: return DISPATCH_QUEUE_PRIORITY_HIGH;
        case NSQualityOfServiceUserInitiated: return DISPATCH_QUEUE_PRIORITY_HIGH;
        case NSQualityOfServiceUtility: return DISPATCH_QUEUE_PRIORITY_LOW;
        case NSQualityOfServiceBackground: return DISPATCH_QUEUE_PRIORITY_BACKGROUND;
        case NSQualityOfServiceDefault: return DISPATCH_QUEUE_PRIORITY_DEFAULT;
        default: return DISPATCH_QUEUE_PRIORITY_DEFAULT;
    }
}

static inline qos_class_t NSQualityOfServiceToQOSClass(NSQualityOfService qos) {
    switch (qos) {
        case NSQualityOfServiceUserInteractive: return QOS_CLASS_USER_INTERACTIVE;
        case NSQualityOfServiceUserInitiated: return QOS_CLASS_USER_INITIATED;
        case NSQualityOfServiceUtility: return QOS_CLASS_UTILITY;
        case NSQualityOfServiceBackground: return QOS_CLASS_BACKGROUND;
        case NSQualityOfServiceDefault: return QOS_CLASS_DEFAULT;
        default: return QOS_CLASS_UNSPECIFIED;
    }
}

@implementation MultithreadViewController

- (dispatch_queue_t)queueFromPoolWithQOSSerive:(NSQualityOfService)servie {
    if ([UIDevice currentDevice].systemVersion.floatValue >= 8.0) {// 使用服务质量描述队列
        
        //1. 服务质量
        qos_class_t qosClass = NSQualityOfServiceToQOSClass(servie);
        
        //2. 创建串行队列，并给队列指定服务质量
        dispatch_queue_attr_t queue_attr = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL, qosClass, 0);
        
        //3. 返回创建完毕的队列
        return dispatch_queue_create("haha", queue_attr);
        
    } else {// 队列优先级描述队列

        //1. 优先级
        dispatch_queue_priority_t identifier = NSQualityOfServiceToDispatchPriority(servie);
        
        //2. 先随便创建一个串行队列（没有指定队列优先级的）
        dispatch_queue_t queue = dispatch_queue_create("haha", DISPATCH_QUEUE_SERIAL);
        
        //3. 再给创建出来的串行队列，设置优先级
        dispatch_set_target_queue(queue, dispatch_get_global_queue(identifier, 0));
        
        return queue;
    }
}

- (void)testGetQueue {
    id queue1 = [self queueFromPoolWithQOSSerive:NSQualityOfServiceUserInteractive];
    id queue2 = [self queueFromPoolWithQOSSerive:NSQualityOfServiceUserInitiated];
    id queue3 = [self queueFromPoolWithQOSSerive:NSQualityOfServiceUtility];
    id queue4 = [self queueFromPoolWithQOSSerive:NSQualityOfServiceBackground];
    id queue5 = [self queueFromPoolWithQOSSerive:NSQualityOfServiceDefault];
    
    dispatch_async(queue5, ^{
        NSLog(@"queue5 thread quality = %ld", [NSThread currentThread].qualityOfService);
    });
    
    dispatch_async(queue4, ^{
        NSLog(@"queue4 thread quality = %ld", [NSThread currentThread].qualityOfService);
    });
    
    dispatch_async(queue3, ^{
        NSLog(@"queue3 thread quality = %ld", [NSThread currentThread].qualityOfService);
    });
    
    dispatch_async(queue2, ^{
        NSLog(@"queue2 thread quality = %ld", [NSThread currentThread].qualityOfService);
    });
    
    dispatch_async(queue1, ^{
        NSLog(@"queue1 thread quality = %ld", [NSThread currentThread].qualityOfService);
    });
}

@end
```

运行结果

```
2016-08-14 16:28:28.765 Demos[12764:202374] queue1 thread quality = 33
2016-08-14 16:28:28.765 Demos[12764:202458] queue2 thread quality = 25
2016-08-14 16:28:28.765 Demos[12764:202377] queue5 thread quality = -1
2016-08-14 16:28:28.765 Demos[12764:202457] queue3 thread quality = 17
2016-08-14 16:28:28.771 Demos[12764:202371] queue4 thread quality = 9
```

####如何选择使用 队列`优先级` or 队列`服务质量` 来描述线程队列了？

- (1) 获取`系统`的全局队列时 >>> 目前来说，还是需要时 `优先级`
	- 没有必要对`dispatch_get_global_queue()`系统函数进行封装适配iOS8+/iOS8-:
		- iOS8以上，`dispatch_get_global_queue()`获取到的队列就是使用	qos 描述的
		- iOS8以下，`dispatch_get_global_queue()`获取到的队列仍然使用 priority 描述的
	
	- 所以对于`dispatch_get_global_queue()`获取系统全局队列，使用`QOS`还是`Priority`来获取，其实没啥区别
	
	- 如果以后不再需要适配`iOS8以下`的系统时，才可以直接使用`QOS`

- (2) `自己`创建队列时 >>> 最好是进行一个封装适配，根据当前系统版本分情况使用`优先级`与`服务质量`

	- `iOS8及以上`系统，使用 `QOS` 描述队列

	- `iOS8以下`系统，使用 `dispatch_queue_priority_t` 描述队列

##使用`dispatch_sync` gcd queue 完成多线程同步

看下不进行多线程同步时，多线程访问一个缓存对象出现的错误:

```objc
@interface MultithreadViewController ()
@property (nonatomic, strong) NSMutableDictionary *cache;
@end

@implementation MultithreadViewController

- (NSMutableDictionary *)cache {
    if (!_cache) {
        _cache = [NSMutableDictionary new];
    }
    return _cache;
}

- (void)testSync {

dispatch_async(dispatch_get_global_queue(0, 0), ^() {
//        dispatch_sync(SerialQueue(), ^{
            [self.cache setObject:@"value1" forKey:@"key1"];
            NSLog(@"set value1 thread = %@", [NSThread currentThread]);
//        });
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^() {
//        dispatch_sync(SerialQueue(), ^{
            [self.cache setObject:@"value2" forKey:@"key1"];
            NSLog(@"set value2 thread = %@", [NSThread currentThread]);
//        });
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^() {
//        dispatch_sync(SerialQueue(), ^{
            NSLog(@"value = %@, thread = %@", [self.cache objectForKey:@"key1"],[NSThread currentThread]);
//        });
    });
}
```

运行时而正确，时而错误，有时候甚至出现`野指针`导致`崩溃`，比如错误的输出:

```
2016-08-14 17:59:58.586 Demos[16405:250373] set value2 thread = <NSThread: 0x7fde48d6c790>{number = 3, name = (null)}
2016-08-14 17:59:58.586 Demos[16405:250371] set value1 thread = <NSThread: 0x7fde48dbc5c0>{number = 4, name = (null)}
2016-08-14 17:59:58.586 Demos[16405:250374] value = value2, thread = <NSThread: 0x7fde48e13010>{number = 2, name = (null)}
```

进行了`set value1`之后，读取缓存字典得到仍然是`value2`，这明显是错误的。

那么，将上面代码中的`dispatch_sync`代码注释代开之后，可以尝试运行n次都是如下结果:

```
2016-08-14 18:05:03.761 Demos[16708:256296] set value1 thread = <NSThread: 0x7f8963fc17e0>{number = 6, name = (null)}
2016-08-14 18:05:03.761 Demos[16708:256297] set value2 thread = <NSThread: 0x7f8963e6a690>{number = 7, name = (null)}
2016-08-14 18:05:03.762 Demos[16708:256076] value = value2, thread = <NSThread: 0x7f8963e68c30>{number = 4, name = (null)}
```

从结果看到这才是正确的情况，先执行了`set value1`，继续执行`set value2`，最后读取得到`value2`。

注意，`dispatch_sync()`会在当前执行`dispatch_sync()`操作的所在线程上执行同步代码块，但是会让其他并发竞争的线程处于`阻塞`。

可以将上面的方法改成如下，增加并发线程的打印信息:

```objc
- (void)testSync {
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^() {
        NSLog(@"并发线程1: %@", [NSThread currentThread]);
        
        dispatch_sync(SerialQueue(), ^{
            [self.cache setObject:@"value1" forKey:@"key1"];
            NSLog(@"set value1 thread = %@", [NSThread currentThread]);
        });
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^() {
        NSLog(@"并发线程2: %@", [NSThread currentThread]);
        
        dispatch_sync(SerialQueue(), ^{
            [self.cache setObject:@"value2" forKey:@"key1"];
            NSLog(@"set value2 thread = %@", [NSThread currentThread]);
        });
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^() {
        NSLog(@"并发线程3: %@", [NSThread currentThread]);
        
        dispatch_sync(SerialQueue(), ^{
            NSLog(@"value = %@, thread = %@", [self.cache objectForKey:@"key1"],[NSThread currentThread]);
        });
    });
}
```

运行结果

```
2016-08-14 18:07:36.189 Demos[16917:259623] 并发线程3: <NSThread: 0x7f9452d47190>{number = 4, name = (null)}
2016-08-14 18:07:36.189 Demos[16917:259620] 并发线程1: <NSThread: 0x7f9452ca09f0>{number = 2, name = (null)}
2016-08-14 18:07:36.189 Demos[16917:259625] 并发线程2: <NSThread: 0x7f9452f10030>{number = 3, name = (null)}
2016-08-14 18:07:36.189 Demos[16917:259623] value = (null), thread = <NSThread: 0x7f9452d47190>{number = 4, name = (null)}
2016-08-14 18:07:36.190 Demos[16917:259620] set value1 thread = <NSThread: 0x7f9452ca09f0>{number = 2, name = (null)}
2016-08-14 18:07:36.190 Demos[16917:259625] set value2 thread = <NSThread: 0x7f9452f10030>{number = 3, name = (null)}
```

可以看到`dispatch_sync()`如何调度的:

- (1) 现将所有的sync任务，全部扔到`SerialQueue()`创建的串行队列`尾部`，按照FIFO先进先出依次`入队`

- (2) 入队完所有调度任务之后，开始取出`队头`的Block开始调度执行
	- Block调度执行，**不会创建新的子线程**
	- 而是直接在执行`dispatch_sync(){}`所在子线程上执行

- (3) 只有当一个Block任务执行`完毕`之后，才会开始调度后面的Block任务

##弄清楚`队列优先级与队列服务质量`之后，再看看使用 `dispatch_sync(queue, block)` 时产生 线程死锁 场景

###测试例子1，会产生`线程死锁`

- ViewController中的测试代码

```objc
@interface AtomicViewController : UIViewController

@end

@implementation AtomicViewController

- (void)func {

	//【重要】该方法执行在`主线程`上
    
    //代码块1
    NSLog(@"1");
    
    //代码块2，在主线程队列同步等待执行一个Block任务
    dispatch_sync(dispatch_get_main_queue(), ^{
        NSLog(@"2");
    });

    //代码块3
    NSLog(@"3");
}

@end
```

运行结果: 

```
2016-08-14 16:57:45.985 Demos[14436:224130] 1
```

从输出结果看，没有输出 2、3。所以可以断定方法所在线程（主线程）已经发生`死锁`。

主线程死锁之后的表现就是，屏幕无法响应任何的事件，就好像玩游戏卡死了，没有任何的反应。

分析产生线程死锁的步骤及原因:

- (1) 执行完代码块1，开始执行`sync同步`等待执行一个Block任务

- (2) 而Block任务使用`sync`方式丢到`main queue`

- (3) 任何线程队列 是`FIFO`，所以`NSLog(@"2");`这个Block任务被丢到了`main queue` 队列的 `末尾` 最后执行

- (4) 而下面的`代码块3`，又需要等待处于`main queue` 队列中的`代码块2`执行完毕，才能往下执行
 
- (5) 但是此时处于`main queue 队尾`的任务Block`NSLog(@"2");`最后一个执行，又需要等待`代码块3`执行完毕

- (6) 这样就产生了这样一个**你等我我等你**的死循环
	- `代码块3`需要等待`代码块2`执行完毕
	- `代码块2`需要等待`代码块3`执行完毕

出现死锁的步骤图示

![](http://i4.tietuku.com/1af73b27703f8746.png)

![](http://i4.tietuku.com/a655a184cba8f9b1.png)

出现上面问题的关键几点:
	
- 点1、`dispatch_sync`操作处于 `主线程`
- 点2、`dispatch_sync` 将Block任务 丢到了 `主线程的队列` 的 队尾
	

在对上面两点继续分析，那就是 在`主线程`进行了`dispatch_sync`操作，并且`dispatch_sync`到了`主线程队列的末尾`.

那么可以总结出来出现如上线程死锁的产生情况了:
	
- (1) `dispatch_sync` 操作 处于 `线程A`
- (2) `dispatch_sync(线程A队列, block任务);`

所以解决，就是在`A线程上，执行dispatch_sync(线程B队列, block任务);`即可

```objc
- (dispatch_queue_t)serialQueue {
    if (!_serialQueue) {
        
        //创建一个串行队列
        _serialQueue = dispatch_queue_create("myQueue1", DISPATCH_QUEUE_SERIAL);
    }
    return _serialQueue;
}

- (void)func {
    
    //代码块1
    NSLog(@"1");
    
    //代码块2，修改在主线程的队列进行丢Block任务
    //（改成另一个自己的队列，只要不是主线程队列都可以）
    dispatch_sync(self.serialQueue, ^{
        NSLog(@"2");
    });

    //代码块3
    NSLog(@"3");
}
```

运行结果:

```
2016-08-14 17:48:26.190 Demos[15580:238369] 1
2016-08-14 17:48:26.191 Demos[15580:238369] 2
2016-08-14 17:48:26.191 Demos[15580:238369] 3
```

可以看到正确执行了，图示正确的结构:

![](http://i4.tietuku.com/709ef52f66581df6.png)

其他形式使用`dispatch_sync`产生线程死锁的情况，根如上都是差不多的，只是可能不是发生在主线程，而是其他的子线程以及对于的子线程队列。

其实也可以明白一点，`dispatch_sync(queue , block)` 以及 `dispatch_async(queue , block)`:

- (1) 都是将Block任务，扔到了`queue`队列的`末尾`
- (2) sync: 立刻执行末尾的Block，但是处于`当前sync操作的线程`上执行
- (3) async: 等待某个时刻才会去依次执行Block，会分配后台子线程去执行
	- 就是让`子线程1`彻底执行完同步代码之后，才会让`子线程2`开始执行同步代码块
	- 就是说执行同步代码块，位于`不同的子线程`上

##使用 dispatch async + `串行队列`， 完成多线程同步

- 会创建子线程，并且只会创建`一个唯一的子线程`，因为是`串行`队列
- 所有的操作都会在同一个 `子线程` 上执行
- 不在当前线程上执行，这样就提高了效率
- 如果主线程也参与多线程竞争时，且同步代码块执行时间较长，就应该考虑使用`dispatch async + 串行队列`完成多线程同步

```objc
dispatch_async(串行队列, ^() {
	
	//多线程同步的代码
	//代码1
});

dispatch_async(串行队列, ^() {
	
	//多线程同步的代码
	//代码2
});

dispatch_async(串行队列, ^() {
	
	//多线程同步的代码
	//代码3
});
```

> 比如、使用GCD Serial Queue 来完成对 getter/setter方法 内代码的多线程同步，而避免使用锁、信号等机制.

```objc
@interface ViewController ()

@property (nonatomic, strong) dispatch_queue_t serialQueue;

@end

@implementation ViewController

- (dispatch_queue_t)serialQueue {
    if (!_serialQueue) {
        _serialQueue = dispatch_queue_create("com.xzh.serail.queue", DISPATCH_QUEUE_SERIAL);
    }
    return _serialQueue;
}

//setter
- (void)modifyUserName:(NSString *)name {
    
    //使用同步执行方法派发到串行队列（这里可以改成async优化）
    dispatch_sync(self.serialQueue, ^{
        self.user.name = [name copy];
    });
}

//getter
- (NSString *)getUserName {
    
    //使用同步派发block任务，为了让所有的setter代码执行完毕，拿到每一次的最新的值
    __block returnName = nil;
    dispatch_sync(self.serialQueue, ^{
        returnName = self.user.name;
    });
    
    return returnName;
}

@end
```

对上述setter方法改进，因为调用`setterName:`时，不需要任何的返回值，那么就不必原地等待setterName:实现执行完毕，才做其他的事情.

```objc
//setter
- (void)modifyUserName:(NSString *)name {
    
	//异步派发block任务，因为不需要得到返回值
    dispatch_async(self.serialQueue, ^{
        self.user.name = [name copy];
    });
}
```

dispatch sync 与 dispatch async 都是将block任务代码丢到`队列的最后面`等待调度，但区别是: 

- sync、会等待blokc任务执行完毕之后，才会做下面的事情
- async、知道block任务放入队尾后，立刻返回做后面其他的事情

但是队列是`串行`的就可以保证block任务是依次按照循序执行就OK了，那么也就是:

- 如果需要等到block任务执行的返回值，才能往下走 >>> sync
- 如果是Void返回值类型的block任务代码执行 >>> async

##dispatch barrier async + `无序队列`，也能够完成多线程同步

- 尽量使用`并发`队列
	- 让大部分不需要顺序执行的Block任务可以并发执行
	- 而只针对需要依赖顺序的Block任务使用同步等待

```objc
- (dispatch_queue_t)concurrntQueue {
    if (!_concurrntQueue) {
        
        //创建一个并发队列
        _concurrntQueue = dispatch_queue_create("myQueue2", DISPATCH_QUEUE_CONCURRENT);
    }
    return _concurrntQueue;
}
```

- 尽量使用`异步 async`派发block线程任务
	- 在需要`线程同步`执行的代码块使用`dispatch_barrier_async`在并发队列中完成一个`同步等待完成的任务`，并且所有的任务都是在`子线程`上完成.
	- 因为使用`dispatch_sync`不会创建新的子线程来完成Block任务，所以性能会比`dispatch_barrier_async `差.

```objc
dispatch_async(self.concurrentQueue, ^(){
    NSLog(@"dispatch-1");
});
    
dispatch_async(self.concurrentQueue, ^(){
    NSLog(@"dispatch-2");
});
    
dispatch_barrier_async(self.concurrentQueue, ^(){
    NSLog(@"dispatch_barrier-3");
});
    
dispatch_barrier_async(self.concurrentQueue, ^(){
    NSLog(@"dispatch_barrier-4");
});
    
dispatch_barrier_async(self.concurrentQueue, ^(){
    NSLog(@"dispatch_barrier-5");
});
    
dispatch_async(self.concurrentQueue, ^(){
    NSLog(@"dispatch-6");
});

dispatch_async(self.concurrentQueue, ^(){
    NSLog(@"dispatch-7");
});
```

运行结果

```
2015-12-16 13:51:12.233 Demo[2353:73281] dispatch-2
2015-12-16 13:51:12.234 Demo[2353:73281] dispatch-1
2015-12-16 13:51:12.236 Demo[2353:73281] dispatch_barrier-3
2015-12-16 13:51:12.237 Demo[2353:73281] dispatch_barrier-4
2015-12-16 13:51:12.237 Demo[2353:73281] dispatch_barrier-5
2015-12-16 13:51:12.237 Demo[2353:73281] dispatch-6
2015-12-16 13:51:12.252 Demo[2353:73281] dispatch-7
```

可以看出:

- 1,2 无序执行
- 3,4,5 是依次按照顺序一个一个接着完成
- 6,7 也可能并发无序完成


那么所以，对于setter与getter函数中对多线程同步，可以改为如下:

```objc
- (dispatch_queue_t)concurrentQueue {
    if (!_concurrentQueue) {
        _concurrentQueue = dispatch_queue_create("com.xzh.concurrnt.queue", DISPATCH_QUEUE_CONCURRENT);
    }
    return _concurrentQueue;
}
```

前面使用 async + serial queue完成同步，这里使用 barrier async + concurrent queue 完成同步，最大的区别是无序并发线程，可能比串行队列效率高一些

```objc
- (void)modifyUserName:(NSString *)name {
    
	//使用栅栏同步setter操作
    dispatch_barrier_async((self.concurrentQueue, ^{
        self.user.name = [name copy];
    });
}
```

如下getter也是使用 sync 到 并发队列完成.

```objc
- (NSString *)getUserName {
    
    __block returnName = nil;
    dispatch_sync(self.concurrentQueue, ^{
        returnName = self.user.name;
    });
    
    return returnName;
}
```

- barrier 栅栏 在并发队列派发执行block任务的作用

	- 首先等待栅栏之前的所有并发block任务执行完毕
	- 单独执行栅栏block块
	- 等待栅栏block块执行完毕之后，才会继续之后后面的并发block任务

![](http://i13.tietuku.cn/8e4ba1744d51a4ab.png)

##使用GCD队列完成多线程同步是书上推荐的方式，但是我看到很多牛人博客上推荐使用pthread mutex

pthread mutex是最底层的c语言，自然效率肯定会高一些。而且我看了YYCache中，也使用到了`pthread_mutex_lock`作为线程互斥同步的方式。

- AFNetworking 使用的NSRecursiveLock
- YYModel中使用 dispatch semophore 信号
- TMCache使用的 GCD Queue 

我看到多线程编程指南书上也是推荐使用GCD Queue的方式，那我想YY没有用GCD Queue的方式，可能是追求底层效率把。

##在一个while循环内不要直接lock，而是使用 `try_lock`，防止产生线程死锁

eg、列举使用 `pthread_mutext_lock` 的例子

```objc
while (!finish) {

        //TODO: 防止在循环中使用pthread_mutex_lock()出现线程死锁，前面的锁没有解除，又继续加锁
        //pthread_mutex_trylock(): 尝试获取锁，获取成功之后才加锁，否则不执行加锁
        
        if (pthread_mutex_trylock(&_lock) == 0) {//获取锁成功，并执行加锁
        
            if (_LRU->_totalCount > countLimit) {
                //不断移除链表尾部节点
                _LinkMapNode *node = [_LRU removeTailNode];
                
                //添加到数组hold住，后面使用线程队列异步释放
                [holder addObject:node];
            } else {
                finish = YES;
            }
            
            //解锁
            pthread_mutex_unlock(&_lock);
            
        } else {
        
            //获取锁失败，尝试10秒后再获取锁
            usleep(10 * 1000);
        }
        
        pthread_mutex_unlock(&_lock);
    }

```

##最近公司使用MJExtensions在多个子线程并发执行json解析时较大几率导致程序崩溃，并且就是MJ中的代码导致崩溃

原因就是因为多线程同时访问一个`公共缓存数据`的问题导致代程序崩溃。

以往在给UI对象做的一些缓存是不需要使用多线程同步的，因为所有的UI对象只能在`主线程`这一个线程上执行操作。所以根本就不会形成`多个线程`上操作UI对象的缓存数据，所以对于`UI对象`的`缓存数据`是`不需要`多线程同步的。

那么对于其他类型的缓存数据，比如: 请求数据缓存、业务对象缓存、DB对象缓存，这样的缓存块数据就得必须使用多线程同步了。

比如，如下一个就会产生崩溃的代码，在多线程环境下访问一个缓存块数据:

```objc
@interface MultithreadViewController ()
@property (nonatomic, strong) NSMutableDictionary *cache;
@end

@implementation MultithreadViewController

-(NSMutableDictionary *)cache {
    if (!_cache) {
        _cache = [NSMutableDictionary new];
    }
    return _cache;
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 100; i++) {
            [self.cache setObject:@"1111" forKey:@"1111"];
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 100; i++) {
            [self.cache setObject:@"2222" forKey:@"2222"];
        }    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 100; i++) {
            [self.cache setObject:@"3333" forKey:@"3333"];
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 100; i++) {
            [self.cache setObject:@"4444" forKey:@"4444"];
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 100; i++) {
            [self.cache setObject:@"55555" forKey:@"55555"];
        }
    });
}


@end
```

其中加了for循环，是为了模拟一些耗时的代码的回调。程序运行后，立刻会出现崩溃，可以自己运行最终崩溃到`objc_msgSend()`，并且提示`BAD_ACCESS`，所以基本上可以肯定是`野指针`导致的。

从代码上看这个类型的对象，只有是`self.cache`指向的可变字典对象编程野指针可能性最大。

`__strong`修饰的指针指向一个对象时，如果这个对象被废弃，但是`指针不会自动赋值为nil`，但是`__weak`修饰的指针就会自动复制nil，从而不会产生使用野指针的问题。

那么崩溃发生的步骤是如下:

- (1) 4号线程Block执行时，当执行`self.cache`之后，就切换到了5号线程

- (2) 切换到5号线程后，同样也执行`self.cache`
	- 很可能因为切换的时间很快，造成线程4执行`self.cache`是，其实`_cache`仍然是等于nil
	- 那么在线程5上，也会走重新创建可变字典对象的代码，然后使用`_cache`指向了新的可变字典对象

- (3) 那么问题来了: `_cache`指向的原来可变字典就失去了strong指针，所以就被废弃了。但是strong指针指向的对象被废弃时，不会自动赋值为nil，也就是成为了一个野指针

- (4) 而线程一旦又切换回4号线程时，很可能还没等待`_name`指向新的字典对象时，即指向的是正在被废弃的对象，那么此时使用`objc_msgSend(字典对象野指针，sel, 参数)`就会必崩

OK，引起崩溃的过程大致就是这样了，那么这一个过程中最主要的结果问题:

- (1) 没有等待`执行完`一个固定的操作，线程就被随时随地的进行切换了 >>>> 一个步骤必须要完整执行完毕，才能切换其他线程

- (2) 对于`_cache`指针没有互斥使用，因为很可能线程1读取到的数据，并不是线程2最终操作完成后的数据，而只是线程1被切换线程时的临时数据 >>>> 避免读取到脏数据，不是最新修改的数据

那么如上两个问题就是一个问题 >>>> 多线程同步。


OK，那么使用多线程同步来解决上述代码的崩溃:

- (1) 改法一、将所有的关于`self.cache`使用的代码，全部放到一个串行的线程队列中。GCD的串行队列是Effective OC书上最为推荐的一种多线程同步方式

```objc
@interface MultithreadViewController ()
@property (nonatomic, strong) NSMutableDictionary *cache;
@property (nonatomic, strong) dispatch_queue_t serialQueue;
@end

@implementation MultithreadViewController

- (dispatch_queue_t)serialQueue {
    if (!_serialQueue) {
        _serialQueue = dispatch_queue_create("serial", DISPATCH_QUEUE_SERIAL);
    }
    return _serialQueue;
}

-(NSMutableDictionary *)cache {
    if (!_cache) {
        _cache = [NSMutableDictionary new];
    }
    return _cache;
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    dispatch_async(self.serialQueue, ^{
        for (int i = 0; i < 100; i++) {
            [self.cache setObject:@"1111" forKey:@"1111"];
        }
    });
    
    dispatch_async(self.serialQueue, ^{
        for (int i = 0; i < 100; i++) {
            [self.cache setObject:@"2222" forKey:@"2222"];
        }
    });
    
    dispatch_async(self.serialQueue, ^{
        for (int i = 0; i < 100; i++) {
            [self.cache setObject:@"3333" forKey:@"3333"];
        }
    });
    
    dispatch_async(self.serialQueue, ^{
        for (int i = 0; i < 100; i++) {
            [self.cache setObject:@"4444" forKey:@"4444"];
        }
    });
    
    dispatch_async(self.serialQueue, ^{
        for (int i = 0; i < 100; i++) {
            [self.cache setObject:@"55555" forKey:@"55555"];
        }
    });
}

@end
```

如上是将字典创建、字典使用都放到了一个串行线程队列排队执行。

- (2) 改法二、使用信号量或pmutex_lock作为同步锁，并将上述代码稍加修改

使用pmutex_lock同步锁

```objc
#import <pthread.h>

static pthread_mutex_t mutex;

@interface MultithreadViewController ()
@property (nonatomic, strong) NSMutableDictionary *cache;
@property (nonatomic, strong) dispatch_queue_t serialQueue;
@end

@implementation MultithreadViewController

- (void)setObject:(NSString *)obj forKey:(nonnull id<NSCopying>)aKey {
    pthread_mutex_lock(&mutex);
    if (!_cache) _cache = [NSMutableDictionary new];
    [_cache setObject:obj forKey:aKey];
    pthread_mutex_unlock(&mutex);
}

- (id)objectForkey:(NSString *)key {
    pthread_mutex_lock(&mutex);
    if (!_cache) _cache = [NSMutableDictionary new];
    NSString *obj = [_cache objectForKey:key];
    pthread_mutex_unlock(&mutex);
    return obj;
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 100; i++) {
            [self setObject:@"1111" forKey:@"2222"];
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 100; i++) {
            [self setObject:@"2222" forKey:@"2222"];
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 100; i++) {
            [self setObject:@"3333" forKey:@"3333"];
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 100; i++) {
            [self setObject:@"4444" forKey:@"4444"];
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 100; i++) {
            [self setObject:@"55555" forKey:@"55555"];
        }
    });
}

@end
```

使用信号量

```objc
//static pthread_mutex_t mutex;
static dispatch_semaphore_t semaphore;

@interface MultithreadViewController ()
@property (nonatomic, strong) NSMutableDictionary *cache;
@property (nonatomic, strong) dispatch_queue_t serialQueue;
@end

@implementation MultithreadViewController

//- (void)setObject:(NSString *)obj forKey:(nonnull id<NSCopying>)aKey {
//    pthread_mutex_lock(&mutex);
//    if (!_cache) _cache = [NSMutableDictionary new];
//    [_cache setObject:obj forKey:aKey];
//    pthread_mutex_unlock(&mutex);
//}
//
//- (id)objectForkey:(NSString *)key {
//    pthread_mutex_lock(&mutex);
//    if (!_cache) _cache = [NSMutableDictionary new];
//    NSString *obj = [_cache objectForKey:key];
//    pthread_mutex_unlock(&mutex);
//    return obj;
//}

- (void)setObject:(NSString *)obj forKey:(nonnull id<NSCopying>)aKey {
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    if (!_cache) _cache = [NSMutableDictionary new];
    [_cache setObject:obj forKey:aKey];
    dispatch_semaphore_signal(semaphore);
}

- (id)objectForkey:(NSString *)key {
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    if (!_cache) _cache = [NSMutableDictionary new];
    NSString *obj = [_cache objectForKey:key];
    dispatch_semaphore_signal(semaphore);
    return obj;
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    // 首先在主线程上，创建一个信号值
    semaphore = dispatch_semaphore_create(1);
    
    // 后续线程争夺这一个信号值
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 100; i++) {
            [self setObject:@"1111" forKey:@"2222"];
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 100; i++) {
            [self setObject:@"2222" forKey:@"2222"];
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 100; i++) {
            [self setObject:@"3333" forKey:@"3333"];
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 100; i++) {
            [self setObject:@"4444" forKey:@"4444"];
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 100; i++) {
            [self setObject:@"55555" forKey:@"55555"];
        }
    });
}

@end
```

首先需要保证信号量必须先创建完毕，才能开始多线程的任务，这一点就没有直接使用`pthread_mutex_t`来的方便。

- (3) 改法三，最后还是与GCD串行队列尝试下

```objc
//static pthread_mutex_t mutex;
//static dispatch_semaphore_t semaphore;

@interface MultithreadViewController ()
@property (nonatomic, strong) NSMutableDictionary *cache;
@property (nonatomic, strong) dispatch_queue_t serialQueue;
@end

@implementation MultithreadViewController

//- (void)setObject:(NSString *)obj forKey:(nonnull id<NSCopying>)aKey {
//    pthread_mutex_lock(&mutex);
//    if (!_cache) _cache = [NSMutableDictionary new];
//    [_cache setObject:obj forKey:aKey];
//    pthread_mutex_unlock(&mutex);
//}
//
//- (id)objectForkey:(NSString *)key {
//    pthread_mutex_lock(&mutex);
//    if (!_cache) _cache = [NSMutableDictionary new];
//    NSString *obj = [_cache objectForKey:key];
//    pthread_mutex_unlock(&mutex);
//    return obj;
//}

//- (void)setObject:(NSString *)obj forKey:(nonnull id<NSCopying>)aKey {
//    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
//    if (!_cache) _cache = [NSMutableDictionary new];
//    [_cache setObject:obj forKey:aKey];
//    dispatch_semaphore_signal(semaphore);
//}
//
//- (id)objectForkey:(NSString *)key {
//    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
//    if (!_cache) _cache = [NSMutableDictionary new];
//    NSString *obj = [_cache objectForKey:key];
//    dispatch_semaphore_signal(semaphore);
//    return obj;
//}

- (void)setObject:(NSString *)obj forKey:(nonnull id<NSCopying>)aKey {
    dispatch_async(_serialQueue, ^{
        if (!_cache) _cache = [NSMutableDictionary new];
        [_cache setObject:obj forKey:aKey];
    });
}

- (id)objectForkey:(NSString *)key {
    __block NSString *obj;
    dispatch_sync(_serialQueue, ^{
        if (!_cache) _cache = [NSMutableDictionary new];
        obj = [_cache objectForKey:key];
    });
    return obj;
}


- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    // 首先在主线程上，创建一个信号值
//    semaphore = dispatch_semaphore_create(1);
    
    // 首先在主线程创建一个串行队列
    _serialQueue = dispatch_queue_create("serial", DISPATCH_QUEUE_SERIAL);
    
    // 后续线程争夺这一个信号值
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 100; i++) {
            [self setObject:@"1111" forKey:@"11111"];
            NSLog(@"1111 = %@", [self objectForkey:@"1111"]);
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 100; i++) {
            [self setObject:@"2222" forKey:@"2222"];
            NSLog(@"1111 = %@", [self objectForkey:@"2222"]);
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 100; i++) {
            [self setObject:@"3333" forKey:@"3333"];
            NSLog(@"1111 = %@", [self objectForkey:@"3333"]);
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 100; i++) {
            [self setObject:@"4444" forKey:@"4444"];
            NSLog(@"1111 = %@", [self objectForkey:@"4444"]);
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 100; i++) {
            [self setObject:@"55555" forKey:@"55555"];
            NSLog(@"1111 = %@", [self objectForkey:@"5555"]);
        }
    });
}

@end
```

和使用信号量进行多线程同步一样，需要`预先`创建完毕一个`串行线程队列`之后，必须保证这个串行队列是在所有多线程任务开始之前已经创建完毕，然后再来调度所有的多线程的任务。


> 总之，对于`除开在 主线程`上使用的缓存对象，一定要多线程同步互斥访问缓存块数据，以及缓存快的逻辑代码也要同步。


##使用原子性进行多线程同步

比如说，@property如果不写`nontomic`，那么默认使用`atomic`原子性进行多线程同步。但是效率会比`nontomic`差，因为原子性规定在任何时刻，一次只能让一个线程去访问。

如果对于直接声明的Ivar实例变量，无法使用@property提供的`atomic`，那么可以通过`<libkern/OSAtomic.h>`库提供的原子性的api来自己完成对变量的原子性操作。


`libkern/OSAtomic.h（原子操作）`提供并发编程中`线程同步`的一种方法

- 这个头文件包含是大量关于`原子操作`和`同步数据`的函数
- 提供的所有函数都是`线程安全`的
- 没有阻断其它线程对函数的访问
- 分别提供了在`32位/64位`cpu处理器下能够使用的数学与逻辑运算的函数，这些操作依赖于`硬件`
	- 比较接近底层，所以效率肯定比`Cocoa锁，如:NSLock等`要高很多
	- 但是使用有点麻烦，也没有很多相关的资料


那么什么是原子操作？

- 对于nonatomic 非原子性、线程非安全，但是效率快
	
	- 多个线程并发执行任务
	- 非原子性，让cpu不断的切换其他线程，不会保证当前线程是否执行完毕
	- 所以就会出现临界数据线程不安全的问题
		- 必须对临界资源`加锁`同步访问


- 对于atomic 原子性、线程安全，但是影响访问效率

	- 多个线程并发执行任务
	- 原子操作是保证`当前线程的任务没有完成之前`，让cpu停止切换线程的操作
	- 从而保证`一个线程` 能够 `单独完成` 整体全部操作之后，才让cpu执行切换其他线程的操作
	- 所以，不再需要我们自己加锁，已经由操作系统来完成线程安全的问题

	
使用 `volatile` 修饰某个变量

- 没有使用`volatile`修饰的变量，编译器对这个变量会做一些`内存读取优化`

	- 避免 `过多次数` 的访问内存，编译器会为变量作一个 `缓存内存块`
	- 缓存内存块中存放的是变量值的一份copy值
	- 以后对该变量的读取，优先从缓存内存块中读取copy值
	- 但是可能存在取到的不是最新的值这样的问题


- 而如果变量使用`volatile`修饰，那么

	- 编译器不会做如上的缓存优化
	- 每次读取这个变量，都是直接访问真实的内存块
	- 可以保证数据永远的真实性，但是会造成效率低下
	- 只适用于一些小数据

Memory Barriers（内存屏障 或 内存栅栏）

- 内存屏障的作用

	- 多cpu时代，多个cpu的内部存储器之间，进行`数据同步`会有问题
	- `CPU内部`存储器单元 要远超过 `主存储`器单元 的访问速度
	- 内存屏障阻碍了CPU采用优化技术来降低内存操作延迟，降低性能
	- 使用内存屏障能够让cpu内部所有核的存储单元数据进行同步

- 内存屏障的种类

	- `编译器`引起的内存屏障
	- `缓存`引起的内存屏障
	- `乱序执行`引起的内存屏障


如上都是一些原理性的东西，其实理解就好，毕竟不是搞专业研究的。下面就是一些简单的OSAtomic库中额函数api使用:


- 初始化一个整形变量

```
//假设运行再64位系统

int64_t a = 0;

OSAtomicAdd64Barrier(4, &a);
```

- 给一个整形变量加上一个数值

```
OSAtomicAdd64(3, &a);
```

- 自增

```
OSAtomicIncrement64(&a);//++
```

- 自减

```
OSAtomicDecrement64(&a);//--
```

- 设置一个内存屏障完成`同步`

```
1. 使用 OSMemoryBarrier();
2. 可以用于 读操作，也可以用于 写操作.
```

```
- (void)func {
    
    //1. 不需要同步的代码
    //...
    
    //2. 设置一个内存屏障
    OSMemoryBarrier();
    
    //3. 需要同步的代码
    //...
}
```

##多线程之间的通信

###常见的用于多个线程之间进行交互主要有如下:

- NSThread
- -[NSObject performSelector:onThread:object: .... ]
- NSOperation Queue
- GCD Dispatch Queue
- NSMatchPort、NSMessagePort
- UIPastBoard（还适用于不同`进程`）

###`进程` 之间的通信

- UIPastBoard 剪贴板
- URL Scheme 吊起目标App程序
- NSMatchPort、NSMessagePort（本地进程）
- iOS9推出的 App Extension

###[NSObject对象的**performSelectorOnThread:某一个Thread** ] 

- 在A线程上，将一个函数放到B线程上去执行
- 并传入带过去的必要的一些参数值
- 可以使用`NSInocation`附加n个参数值
- B线程对象，就得到了A线程对象的数据

```
在主线程做事情:
performSelectorOnMainThread:withObject:waitUntilDone:
performSelectorOnMainThread:withObject:waitUntilDone:modes:
```

```
在指定线程中做事情:
performSelector:onThread:withObject:waitUntilDone:
performSelector:onThread:withObject:waitUntilDone:modes:
```

```
在当前线程中做事情:
performSelector:withObject:afterDelay:
performSelector:withObject:afterDelay:inModes:
```

```
取消发送给当前线程的某个消息:
cancelPreviousPerformRequestsWithTarget:
cancelPreviousPerformRequestsWithTarget:selector:object:
```

###PerfromSelectorOnThread方法和NSMatcherPort完成 不同线程对象之间的数据传递:

- performSelectorOnThread使用`更简单`

- NSMatchPort更适合`异步`、`复杂`的线程对象之间数据传输

###关于performSelectorXxx函数做了一些什么？

- 创建一个`NSTimer`对象

- 并指定NSTimer对象的回调函数，为performSelector:传入的函数的SEL值

- 开启当前代码执行在的线程的 runLoop（我们自己创建的子线程，runLoop是默认关闭的）

- 将NSTimer添加到 runLoop

- runLoop 开始调度 内部不同mode下的 timers/observers/sources
	- mode1
		- timers/observers/sources
	- mode2
		- timers/observers/sources
	- mode3
		- timers/observers/sources
	- mode4
		- timers/observers/sources

- 执行完之后，结束执行NSTimer


##UIPastBoard剪贴板完成线程或进程之间的数据交换

- 存入数据

```
NSPasteboard *pasteboard = [NSPasteboard generalPasteboard];
[pasteboard clearContents];
[pasteboard writeObjects:@[image]];
```

- 读取数据

```
NSPasteboard *pasteboard = [NSPasteboard generalPasteboard];

if ([pasteboard canReadObjectForClasses:@[[NSImage class]] options:nil]) {
    NSArray *contents = [pasteboard readObjectsForClasses:@[[NSImage class]] options: nil];
    NSImage *image = [contents firstObject];
}
```

###使用NSMachPort

- NSMachPort有三个子类，其中iOS平台使用的只有一种
	- NSSocketPort，iOS平台`不`支持
	- NSMessagePort，iOS平台`不`支持
	- **NSMachPort**，iOS平台支持

- 使用NSMachPort实现不同线程对象之间数据传递的步骤:
	
	- 第一步、线程对象`A`，使用自己的RunLoop向系统注册一个用于线程交互的`NSMatchPort端口`

	- 第二步、线程对象`B`，向这个`NSMatchPort端口`所在的RunLoop发送`需要线程交互的请求`，需要与线程A交互

	- 第三步、`iOS系统线程`接收到`需要线程交互的请求`，根据传入的`NSMatchPort端口`，查找到对应的RunLoop

	- 第四步、`iOS系统线程`将线程对象B发送的方法调用、传入数据等等，`发送`给找到的RunLoop
	
	- 第五步、线程对象A的**RunLoop**，接收到其他线程交互请求与数据，**RunLoop**唤醒线程对象A，开始执行任务


###CoreFoundation中的 RunLoop 启动源码

> http://opensource.apple.com/source/CF/CF-1153.18/CFRunLoop.c

从如上连接进入，可以查看runloop启动的具体代码实现，里面可以找到一个函数: `mach_msg()`

- 通过 `mach_msg()`函数，接收基于MachPort（端口）的消息
- 如果没有别人发送 port 消息过来，内核 会将线程置于`等待`状态

调用 `mach_msg()`函数 完成 Port消息的 发送与接收

- 用户空间（用户线程）
- 内核空间（内核系统线程）
- 由系统完成 自动切换

该函数实现了如上三个空间的切换。

###`主线程` 的 RunLoop 已经由系统创建了AutoRelease Pool，但是自己创建的子线程需要自己创建

- App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，其回调都是 `_wrapRunLoopWithAutoreleasePoolHandler()`。

- 第一个 Observer监视的事件是 `Entry` (即将进入Loop)
	- 其回调内会调用 `_objc_autoreleasePoolPush()`创建自动释放池
	- 其order是`-2147483647`，优先级`最高`，保证创建释放池发生在其他所有回调`之前`

- 第二个 Observer 监视了两个事件： `BeforeWaiting` (准备进入休眠) 和 `Exit` (即将退出Loop)
	- BeforeWaiting 事件回调
		- 调用 _objc_autoreleasePoolPop() 释放旧的池
		- 调用 _objc_autoreleasePoolPush() 创建新的池
	- Exit 事件回调
		- 调用 _objc_autoreleasePoolPop() 释放自动释放池
	- 这个Observer的order是2147483647，优先级`最低`，保证其释放池子发生在其他所有回调`之后`

- 所以`主线程`不会出现内存泄漏（`仅针对放入释放池管理的对象`），开发者也不必显示创建 Pool

##NSPort 的三个子类

- NSMessagePort (iOS平台不可用)
	- 在一台机器内部操作系统内部

- NSMachPort 
	- 在一台机器内部操作系统内部

- NSSocketPort (iOS平台不可用)
	- 本地和远程两种通讯
	- 但是本地通讯比较耗资源

****

###CococaFoundation中 NSMachPort 线程通信

- ViewController 中按钮点击，创建一个子线程并传入主线程的port

```objc
@implementation ViewController


- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    
    UIButton *b = [[UIButton alloc] initWithFrame:CGRectMake(20, 80, 120, 50)];
    [b setTitle:@"New Thread" forState:UIControlStateNormal];
    [b setBackgroundColor:[UIColor blueColor]];
    [b addTarget:self action:@selector(launchThread) forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:b];
}



//启动线程
- (void)launchThread{
    
    //1. 创建主线程的port
    // 子线程通过此端口发送消息给主线程
    NSPort *myPort = [NSMachPort port];
    
    //2. 设置port的代理回调对象
    myPort.delegate = self;
    
    //3. 把port加入runloop，接收port消息
    [[NSRunLoop currentRunLoop] addPort:myPort forMode:NSDefaultRunLoopMode];
    
    //4. 启动次线程,并传入主线程的port
    MyWorkerClass *work = [[MyWorkerClass alloc] init];
    [NSThread detachNewThreadSelector:@selector(launchThreadWithPort:)
                             toTarget:work
                           withObject:myPort];
}

#pragma mark - NSPortDelegate

#define kMsg1 100
#define kMsg2 101

/**
 *  接收到子线程的port 消息
 */
- (void)handlePortMessage:(NSMessagePort*)message{

    NSLog(@"接到子线程传递的消息！");
    
    //1. 消息id
    NSUInteger msgId = [[message valueForKeyPath:@"msgid"] integerValue];
    
    //2. 当前主线程的port
    NSPort *localPort = [message valueForKeyPath:@"localPort"];
    
    //3. 接收到消息的port（来自其他线程）
    NSPort *remotePort = [message valueForKeyPath:@"remotePort"];
  
    if (msgId == kMsg1)
    {
        //向子线的port发送消息
        [remotePort sendBeforeDate:[NSDate date]
                             msgid:kMsg2
                        components:nil
                              from:localPort
                          reserved:0];
        
    } else if (msgId == kMsg2){
        NSLog(@"操作2....\n");
    }
    
}

@end
```

- MyWorkerClass.m 完成创建子线程，并且开启子线程的runloop，保存主线程传入的port

```objc
#import "MyWorkerClass.h"

@interface MyWorkerClass() <NSMachPortDelegate> {
    NSPort *remotePort;
    NSPort *myPort;
}
@end

#define kMsg1 100
#define kMsg2 101

@implementation MyWorkerClass


- (void)launchThreadWithPort:(NSPort *)port {
    
    @autoreleasepool {
        
        //1. 保存主线程传入的port
        remotePort = port;
        
        //2. 设置子线程名字
        [[NSThread currentThread] setName:@"MyWorkerClassThread"];
        
        //3. 开启runloop
        [[NSRunLoop currentRunLoop] run];
        
        //4. 创建自己port
        myPort = [NSPort port];
        
        //5.
        myPort.delegate = self;
        
        //6. 将自己的port添加到runloop
        //作用1、防止runloop执行完毕之后推出
        //作用2、接收主线程发送过来的port消息
        [[NSRunLoop currentRunLoop] addPort:myPort forMode:NSDefaultRunLoopMode];
        
        
        
        //7. 完成向主线程port发送消息
        [self sendPortMessage];
        
        
    }
}

/**
 *	完成向主线程发送port消息
 */
- (void)sendPortMessage {
    
    //发送消息到主线程，操作1
    [remotePort sendBeforeDate:[NSDate date]
                         msgid:kMsg1
                    components:nil
                          from:myPort
                      reserved:0];
    
    //发送消息到主线程，操作2
//    [remotePort sendBeforeDate:[NSDate date]
//                         msgid:kMsg2
//                    components:nil
//                          from:myPort
//                      reserved:0];
}


#pragma mark - NSPortDelegate

/**
 *  接收到主线程port消息
 */
- (void)handlePortMessage:(NSPortMessage *)message
{
    NSLog(@"接收到父线程的消息...\n");
    
//    unsigned int msgid = [message msgid];
//    NSPort* distantPort = nil;
//    
//    if (msgid == kCheckinMessage)
//    {
//        distantPort = [message sendPort];
//        
//    }
//    else if(msgid == kExitMessage)
//    {
//        CFRunLoopStop((__bridge CFRunLoopRef)[NSRunLoop currentRunLoop]);
//    }
}

@end
```

####如上代码

- 完成的: 子线程对象的port 能够向 父线程对象的port 发送消息
- 有问题: 无法让 `父线程上对象的port` 向 `子线程上对象的

****

###解决让让 `父线程上对象的port` 向 `子线程上对象的port` 发送消息

####如果希望父线程也可以向子线程发消息 主要步骤
	
1. 子线程可以先向父线程发个 特殊的消息
2. 而这个特殊消息包含，子线程的 NSMachPort对象
3. 父线程便持有了子线程创建的MachPort对象
4. 父线程可以通过向持有的这个MachPort对象发送消息

####修改上面`MyWorkerClass`这个子线程对象的发送port消息的方法`sendPortMessage`

> 子线程可以先向父线程发个特殊的消息，传过来的是自己创建的另一个NSMachPort对象，这样父线程便持有了子线程创建的port对象了，可以向这个子线程的port对象发送消息了.


```objc
- (void)sendPortMessage 
{

//     如下代码是创建一个 port消息 ，将当前子线程上对象的port，发送给父线程的port
//     但是由于iOS7+屏蔽了使用NSPortMessage，所以如下代码会编译报错.
    
//    1. 构造一个port消息，传入当前子线程对象的port
//    NSPortMessage* messageObj = [[NSPortMessage alloc] initWithSendPort:主线程对象的port
//                                                            receivePort:当前子线程对象的port
//    2. 设置消息的id
//    [messageObj setMsgid:kCheckinMessage];
//    
//    3. 发送port消息
//    [messageObj sendBeforeDate:[NSDate date]];
}
```

####那么 主线程对象 的 `NSPortDelegate`回调方法`- (void)handlePortMessage:(NSMessagePort*)message`做如下事情，即可完成主线程对象的port发送消息:

- 1) 首先取出 接收到的port消息中 的子线程的 port
- 2) 保存子线线程 port
- 3) 然后使用 `[子线程port sendMessage...]` 完毕

****

###额外记录下使用NSRunLoopObserver用法

```objc
/**
 * 定义一个运行在 kCFRunLoopDefaultMode 模式下的观察者
 */
- (void)observerRunLoop
{
    NSRunLoop *curRunLoop = [NSRunLoop currentRunLoop];
    
    //创建observer的运行环境
    CFRunLoopObserverContext  context = {0, (__bridge void *)(self), NULL, NULL, NULL};
    
    //创建observer对象
    CFRunLoopObserverRef observer = CFRunLoopObserverCreate(kCFAllocatorDefault,    //分配内存
                                                            kCFRunLoopAllActivities,//设置观察类型
                                                            YES,                    //标识该observer是在第一次进入run loop时执行还是每次进入run loop处理时均执行
                                                            0,                      //设置该observer的优先级
                                                            &myRunLoopObserver,     //设置该observer的回调函数
                                                            &context);              //设置该observer的运行环境
    
    //将Cocoa的NSRunLoop类型转换成Core Foundation的CFRunLoopRef类型
    if (observer) 
    {	
    	// 获取CoreFoundation环境下的runLoop结构体实例
        CFRunLoopRef cfRunLoop = [curRunLoop getCFRunLoop];
        
        //将新建的observer加入到当前thread的run loop
        CFRunLoopAddObserver(cfRunLoop, observer, kCFRunLoopDefaultMode);
    }
}
```

```objc
/**
 * observer关注runloop所有状态改变后的回调处理函数
 */
void myRunLoopObserver(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info) {
    switch (activity) {
        case kCFRunLoopExit:
            NSLog(@"Run Loop 终止");
            break;
        case kCFRunLoopBeforeTimers:
            NSLog(@"将要处理一个Timer事件源");
            break;
        case kCFRunLoopBeforeWaiting:
            NSLog(@"即将进入休眠");
            break;
        case kCFRunLoopAfterWaiting:
            NSLog(@"被唤醒的时刻,但在唤醒它的事件被处理之前");
            break;
        case kCFRunLoopEntry:
            NSLog(@"RunLoop 进入");
            break;
        case kCFRunLoopBeforeSources:
            NSLog(@"即将处理一个 Input Source");
            break;
        default:
            break;
    }
}
```

***

###使用 CFMessagePort 完成线程之间的通信

> 此部分来源于`王中周的技术博客`.

- CFMessagePort 只能用于 `本地（一部手机系统）`里面的进程通信 、以及一个进程内的所有线程之间的通信
- 在`iOS7及以后`系统中，CFMessagePort的通信机制`不再可用`
- CFMessagePort是基于 `CFMachPortRef通信机制`进行封装的

###CFMessagePort使用关键点

- 线程A上对象，创建一个`本地 MessagePort`，并且关注则个port的消息接收，使用一个`唯一的标识`来区分这个port
- 线程B上对象，创建一个`远程 MessagePort`，而这个 远程port标识就是前面线程A上创建的port的标识
- 线程B上对象，使用`CFMessagePortSendRequest()`函数向远程port发送消息
- 线程A上对象的接收port消息的函数被回调执行

> 实际上最终还是通过 MachPort 完成线程通信，系统根据 CFMessagePort 的唯一标识，找到对应的 MachPort.

####代码示例

- ViewController 主线程上注册port，用于接收子线程发送的消息

```objc
@ViewController


@implementation AtomicViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    //测试
    [self launchThread];
}

- (void)launchThread {
    
    //1. 主线程对象，创建一个用于接收消息的 message port ref
    [self createLocalMessagePort];
    
    //2. 创建一个子线程，在子线程上执行对象方法，完成子线程对象向主线程port发送消息
    _work = [[MyWorkerClass alloc] init];
    NSThread *t = [[NSThread alloc] initWithTarget:_work
                                          selector:@selector(LaunchThread)
                                            object:nil];
    
    //3. 开启线程
    [t start];
}

#pragma mark - CFMessagePortRef 

/**
 *	创建一个监听的本地port，接收消息
 /
- (void)createLocalMessagePort {
    
    //1.
    if (_listenerPort != 0 && CFMessagePortIsValid(_listenerPort))
    {
        // 停止当前所有的 port消息的发送和接收操作
        CFMessagePortInvalidate(_listenerPort);
    }
    
    //2. 创建一个本地CFMessagePortRef对象，设置一个字符串来唯一标识这个port，以及回调函数
    _listenerPort = CFMessagePortCreateLocal(kCFAllocatorDefault,
                                             CFSTR("MyMachPortName"),
                                             onRecvMessageCallBack,
                                             NULL,
                                             NULL);
    
    //3.
    CFRunLoopSourceRef source = CFMessagePortCreateRunLoopSource(kCFAllocatorDefault,
                                                                 _listenerPort,
                                                                 0);
    
    //4.
    CFRunLoopAddSource(CFRunLoopGetCurrent(), source, kCFRunLoopCommonModes);
}


/**
 *	接收到消息的回调函数
 */
CFDataRef onRecvMessageCallBack(CFMessagePortRef local,SInt32 msgid,CFDataRef cfData, void*info)
{
    NSLog(@"onRecvMessageCallBack is called");
    
    NSString *strData = nil;
    
    //1. 将接收到的消息，转换成自定义实体类对象数据格式
    if (cfData)
    {
        const UInt8  * recvedMsg = CFDataGetBytePtr(cfData);
        
        strData = [NSString stringWithCString:(char *)recvedMsg encoding:NSUTF8StringEncoding];
        /**
         
         实现数据解析操作
         
         **/
        
        NSLog(@"receive message:%@",strData);
    }
    
    //2. 生成返回数据，返回给发送消息的子线程的port
    NSString *returnString = [NSString stringWithFormat:@"i have receive:%@",strData];
    const char* cStr = [returnString UTF8String];
    NSUInteger ulen = [returnString lengthOfBytesUsingEncoding:NSUTF8StringEncoding];
    CFDataRef sgReturn = CFDataCreate(NULL, (UInt8 *)cStr, ulen);
    
    return sgReturn;
}

@end
```

- `MyWorkerClass`子线程上执行的类对象

```objc
@implementation MyWorkerClass

- (void)LaunchThread {
    
    @autoreleasepool {
        
        //1.
        NSThread *t = [NSThread currentThread];
        
        //2.
        NSRunLoop *currentRunLoop = [NSRunLoop currentRunLoop];
        
        //3.
        [currentRunLoop run];
        
        //4. 向主线程的port 发送消息
        [self sendMessageToDameonWith:@{@"key":@"value"} msgID:12345];
    }
}

-(NSString *)sendMessageToDameonWith:(id)msgInfo msgID:(SInt32)msgid
{
    // 生成Remote port
    CFMessagePortRef bRemote = CFMessagePortCreateRemote(kCFAllocatorDefault,
                                                         CFSTR("MyMachPortName"));
    if (nil == bRemote) {
        NSLog(@"bRemote create failed");
        return nil;
    }
    
    // 构建发送数据（string）
    NSString    *msg = [NSString stringWithFormat:@"%@",msgInfo];
    NSLog(@"send msg is :%@",msg);
    const char *message = [msg UTF8String];
    CFDataRef data,recvData = nil;
    data = CFDataCreate(NULL, (UInt8 *)message, strlen(message));
    
    // 执行发送操作
    CFMessagePortSendRequest(bRemote, msgid, data, 0, 100 , kCFRunLoopDefaultMode, &recvData);
    if (nil == recvData) {
        NSLog(@"recvData date is nil.");
        CFRelease(data);
        CFMessagePortInvalidate(bRemote);
        CFRelease(bRemote);
        return nil;
    }
    
    // 接收并解析 主线程port 返回的数据
    const UInt8  * recvedMsg = CFDataGetBytePtr(recvData);
    if (nil == recvedMsg) {
        NSLog(@"receive date err.");
        CFRelease(data);
        CFMessagePortInvalidate(bRemote);
        CFRelease(bRemote);
        return nil;
    }
    
    NSString    *strMsg = [NSString stringWithCString:(char *)recvedMsg encoding:NSUTF8StringEncoding];
    NSLog(@"%@",strMsg);
    
    CFRelease(data);
    CFMessagePortInvalidate(bRemote);
    CFRelease(bRemote);
    CFRelease(recvData);
    
    return strMsg;
}
```


最后的小结:

- MachPort、MessagePort，在iOS7之后苹果不再面向开发者，因为其使用太过复杂、麻烦.
- 苹果推荐使用如下进行线程通信
	- GCD DisPach Queue
	- NSOperation Queue
	- NSObject分类的各种的 performSelecter:onThread:...方法


##YYDispatchQueuePool实现了一个serial dispatch queue 实例的缓存池

###使用`线程池`的意义是什么？

- (1) 复用线程对象，避免重复性不断的创建新的线程

通常情况下，我们使用`GCD dispatahc queue` 或 `NSOperationQueue` 更或者直接使用`NSThread、pthread、-[NSObject performSelecter:onThread:]`来让一段代码在一个新的子线程上进行执行。

当现场一旦执行完任务，就会退出执行，被系统回收所占用的资源。而下一次需要执行子线程任务时，又要重新的创建子线程。

当某个线程正在执行一顿耗时的代码，导致需要占用过多的CPU时间。但是此时又创建了一个新的线程做其他的事情，那么同样会竞争CPU时间，就导致本来就很紧张的CPU时间更加紧张。

而且当某一个线程发生死锁时，可能造成`无限制`的创建子线程，占用过高的系统资源，导致主线程卡死

- (2) 子线程数过多，会导致CPU性能下降。根据系统的承受能力，调整线程池中工作线线程的数目

因为CPU核数有限，所以CPU能够处理的最大并发线程数也是有限的。

而且进程内如果出现大量的线程处理请求，造成CPU大量耗费于`上下文切换`、以及内存溢出等后果。

通过线程池技术可以控制`子线程的最大并发数`，保证CPU不会超负荷运行。并不是说子线程越多越好，越多速度越快。不断的增加子线程的同时，也对系统性能增加更多的消耗。


###什么是线程池？

以前我搞java的时候，记得最深的就是jdbc数据库连接池，觉得非常的牛逼但是一直没弄明白。

那么线程池是用来:

- (1) 控制系统中线程的数量
- (2) 让外界将要执行的任务进行排队等待，一个一个按照入队顺序执行
- (3) 一个任务执行完毕，再从`任务队列`的中取最前面的任务开始执行
- (4) 若当前线程池已经没有可以工作的线程了，那么就让这个任务先处于等待，等待一个空闲的线程来服务

参考了一些基于c实现的线程缓存池，因为是直接操作的线程thread而不是GCD队列，所以他们的实现中还提供了一个`任务队列`。但是这个东西，我觉得GCD队列已经完全实现了，就是Block任务入队都是按照FIFO实现完毕了，所以关于`任务队列`我觉得就没必要再做了，可以像YYDispatchQueuePool那样，直接将缓存的`dispatch_queue_t`实例暴露出去就行了


###在iOS中，一个 `dispatch_queue_t`实例 就相当于是 一个`新的子线程`

- (1) serial queue >>> 生成一个子线程
- (2) concurrent queue >>> 生成多个子线程

如下是每次都创建一个新的 serial dispatch queue 实例，然后进行线程任务调度

```objc
/**
 *  每次创建一个新的serial queue >>> 创建一个新的子线程
 */
    
dispatch_async(dispatch_queue_create("queue1", DISPATCH_QUEUE_SERIAL), ^{
    NSLog(@"queue1 thread: %@", [NSThread currentThread]);
});
    
dispatch_async(dispatch_queue_create("queue2", DISPATCH_QUEUE_SERIAL), ^{
    NSLog(@"queue2 thread: %@", [NSThread currentThread]);
});
    
dispatch_async(dispatch_queue_create("queue3", DISPATCH_QUEUE_SERIAL), ^{
    NSLog(@"queue3 thread: %@", [NSThread currentThread]);
});
    
dispatch_async(dispatch_queue_create("queue4", DISPATCH_QUEUE_SERIAL), ^{
    NSLog(@"queue4 thread: %@", [NSThread currentThread]);
});
    
dispatch_async(dispatch_queue_create("queue5", DISPATCH_QUEUE_SERIAL), ^{
    NSLog(@"queue5 thread: %@", [NSThread currentThread]);
});
    
dispatch_async(dispatch_queue_create("queue6", DISPATCH_QUEUE_SERIAL), ^{
    NSLog(@"queue6 thread: %@", [NSThread currentThread]);
});
    
dispatch_async(dispatch_queue_create("queue7", DISPATCH_QUEUE_SERIAL), ^{
    NSLog(@"queue7 thread: %@", [NSThread currentThread]);
});
    
dispatch_async(dispatch_queue_create("queue8", DISPATCH_QUEUE_SERIAL), ^{
    NSLog(@"queue8 thread: %@", [NSThread currentThread]);
});
    
dispatch_async(dispatch_queue_create("queue9", DISPATCH_QUEUE_SERIAL), ^{
    NSLog(@"queue9 thread: %@", [NSThread currentThread]);
});
    
dispatch_async(dispatch_queue_create("queue10", DISPATCH_QUEUE_SERIAL), ^{
    NSLog(@"queue10 thread: %@", [NSThread currentThread]);
});
```

运行结果

```
2016-08-18 12:49:58.748 Demo[4374:422577] queue3 thread: <NSThread: 0x7f8141c28580>{number = 4, name = (null)}
2016-08-18 12:49:58.748 Demo[4374:422574] queue2 thread: <NSThread: 0x7f8141e23700>{number = 3, name = (null)}
2016-08-18 12:49:58.748 Demo[4374:422582] queue7 thread: <NSThread: 0x7f8141f922f0>{number = 8, name = (null)}
2016-08-18 12:49:58.748 Demo[4374:422580] queue6 thread: <NSThread: 0x7f8141c1c150>{number = 7, name = (null)}
2016-08-18 12:49:58.748 Demo[4374:422576] queue1 thread: <NSThread: 0x7f8141f90b40>{number = 2, name = (null)}
2016-08-18 12:49:58.748 Demo[4374:422578] queue4 thread: <NSThread: 0x7f8141e060b0>{number = 5, name = (null)}
2016-08-18 12:49:58.748 Demo[4374:422579] queue5 thread: <NSThread: 0x7f8141f15770>{number = 6, name = (null)}
2016-08-18 12:49:58.748 Demo[4374:422583] queue8 thread: <NSThread: 0x7f8141f1b1c0>{number = 9, name = (null)}
2016-08-18 12:49:58.748 Demo[4374:422584] queue9 thread: <NSThread: 0x7f8141d1bfa0>{number = 10, name = (null)}
```

从运行结果来看，每一个任务都是在一个新的子线程上执行。

###复用一个线程队列，比如获取系统创建的全局单例并发线程队列

```objc
/**
 *  重用某一个线程队列，内部会使用GCD底层的线程池进行优化
 */

dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"thread1: %@", [NSThread currentThread]);
});

dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"thread2: %@", [NSThread currentThread]);
});

dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"thread3: %@", [NSThread currentThread]);
});

dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"thread4: %@", [NSThread currentThread]);
});

dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"thread5: %@", [NSThread currentThread]);
});

dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"thread6: %@", [NSThread currentThread]);
});

dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"thread7: %@", [NSThread currentThread]);
});

dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"thread8: %@", [NSThread currentThread]);
});

dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"thread9: %@", [NSThread currentThread]);
});

dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"thread10: %@", [NSThread currentThread]);
});
```

运行结果

```
2016-08-18 12:53:32.201 Demo[4437:439465] thread1: <NSThread: 0x7faa1ae1aae0>{number = 2, name = (null)}
2016-08-18 12:53:32.201 Demo[4437:439458] thread3: <NSThread: 0x7faa1ad0fb90>{number = 3, name = (null)}
2016-08-18 12:53:32.201 Demo[4437:439629] thread5: <NSThread: 0x7faa1ac36040>{number = 6, name = (null)}
2016-08-18 12:53:32.201 Demo[4437:439628] thread4: <NSThread: 0x7faa1ad01fc0>{number = 4, name = (null)}
2016-08-18 12:53:32.201 Demo[4437:439484] thread2: <NSThread: 0x7faa1ac06110>{number = 5, name = (null)}
2016-08-18 12:53:32.201 Demo[4437:439458] thread6: <NSThread: 0x7faa1ad0fb90>{number = 3, name = (null)}
2016-08-18 12:53:32.202 Demo[4437:439629] thread7: <NSThread: 0x7faa1ac36040>{number = 6, name = (null)}
2016-08-18 12:53:32.202 Demo[4437:439628] thread9: <NSThread: 0x7faa1ad01fc0>{number = 4, name = (null)}
2016-08-18 12:53:32.202 Demo[4437:439465] thread8: <NSThread: 0x7faa1ae1aae0>{number = 2, name = (null)}
2016-08-18 12:53:32.202 Demo[4437:439484] thread10: <NSThread: 0x7faa1ac06110>{number = 5, name = (null)}
```

从运行结果看，这10个线程任务是通过2、3、4、5、6这5个线程循环利用来完成。

而且多点击几次运行，还可以发现底层线程的number也是`递增`的。当前面的一些线程废弃回收之后，后面的心的子线程的number不会使用之前的线程的number，而是通过递增的方式。

也就是说或，对于GCD`并发`线程队列，底层是做了线程池的优化了的。那么这是对系统的gloabal concurrent queue的测试，那对我们自己创建的concurrent queue也有线程池优化吗？

```objc
dispatch_queue_t concurrentQueue = dispatch_queue_create("cocurrent.queue", DISPATCH_QUEUE_CONCURRENT);

dispatch_async(concurrentQueue, ^{
    NSLog(@"thread1: %@", [NSThread currentThread]);
});

dispatch_async(concurrentQueue, ^{
    NSLog(@"thread2: %@", [NSThread currentThread]);
});

dispatch_async(concurrentQueue, ^{
    NSLog(@"thread3: %@", [NSThread currentThread]);
});

dispatch_async(concurrentQueue, ^{
    NSLog(@"thread4: %@", [NSThread currentThread]);
});

dispatch_async(concurrentQueue, ^{
    NSLog(@"thread5: %@", [NSThread currentThread]);
});

dispatch_async(concurrentQueue, ^{
    NSLog(@"thread6: %@", [NSThread currentThread]);
});

dispatch_async(concurrentQueue, ^{
    NSLog(@"thread7: %@", [NSThread currentThread]);
});

dispatch_async(concurrentQueue, ^{
    NSLog(@"thread8: %@", [NSThread currentThread]);
});

dispatch_async(concurrentQueue, ^{
    NSLog(@"thread9: %@", [NSThread currentThread]);
});

dispatch_async(concurrentQueue, ^{
    NSLog(@"thread10: %@", [NSThread currentThread]);
});
```

运行结果

```
2016-08-18 12:59:17.576 Demo[4540:467914] thread4: <NSThread: 0x7fd56b62c510>{number = 4, name = (null)}
2016-08-18 12:59:17.576 Demo[4540:467925] thread1: <NSThread: 0x7fd56b5058a0>{number = 2, name = (null)}
2016-08-18 12:59:17.576 Demo[4540:468613] thread3: <NSThread: 0x7fd56b5971f0>{number = 3, name = (null)}
2016-08-18 12:59:17.577 Demo[4540:467971] thread2: <NSThread: 0x7fd56b41acd0>{number = 5, name = (null)}
2016-08-18 12:59:17.577 Demo[4540:467914] thread5: <NSThread: 0x7fd56b62c510>{number = 4, name = (null)}
2016-08-18 12:59:17.577 Demo[4540:467925] thread6: <NSThread: 0x7fd56b5058a0>{number = 2, name = (null)}
2016-08-18 12:59:17.577 Demo[4540:468613] thread7: <NSThread: 0x7fd56b5971f0>{number = 3, name = (null)}
2016-08-18 12:59:17.577 Demo[4540:467971] thread8: <NSThread: 0x7fd56b41acd0>{number = 5, name = (null)}
2016-08-18 12:59:17.577 Demo[4540:467914] thread9: <NSThread: 0x7fd56b62c510>{number = 4, name = (null)}
2016-08-18 12:59:17.577 Demo[4540:467925] thread10: <NSThread: 0x7fd56b5058a0>{number = 2, name = (null)}
```

可以看到，同样是也是利用2、3、4、5、6这5个线程来完成10个任务。所以GCD对任何的`并发线程队列 concurrent queue`在底层线程上都做了`线程池`机制优化了的。

###小结 GCD Concurrent Queue 与 GCD Serial Queue 

对于使用GCD Concurrent Queue

- (1) 最终GCD底层对于线程已经使用了线程池机制优化的
- (2) 一个线程执行完任务之后，不会立马被释放掉，而是先暂时缓存起来
- (3) 当调度任务进入时，从缓存中取出一个线程进行处理
- (4) 当然也应该进来的避免自己创建很多的 GCD Concurrent Queue 实例

但是对于GCD Serial Queue

- (1) 一个Serial Queue新的实例，就相当于一个新的子线程
- (2) 如果不断地创建 GCD Serial Queue 实例，就等同于不断的强制性创建新的子线程
- (3) 那么这样一来，就完全绕过了线程池复用线程机制了，就有可能造成大量的线程创建，降低CPU的性能

###只有控制创建GCD Serial Queue实例的数量，来避免底层不断的创建新的子线程

> 对于 GCD Concurrent Queue:

虽然不断的创建GCD Concurrent Queue实例，也是不断的创建线程，那么也应该控制创建GCD Concurrent Queue实例的个数。但是一般情况下，使用`自己创建`的GCD Concurrent Queue场景很少，一般都是直接使用`dispatch_get_global_queue(long identifier, unsigned long flags);`获取系统的全局并发队列即可。所以对于GCD Concurrent Queue实例个数进行控制，就不是那么的重要了。


> 对于 GCD Serial Queue:

上面已经通过不断的创建GCD Serial Queue实例时，底层也会不断的创建新的子线程，而不会进行任何的线程复用处理。

所以在整个App程序代码中，就必要进行控制GCD Serial Queue实例的创建个数了。

iOS6之后，就已经将让开发者直接使用线程`thread`的方式，转换成使用队列`queue`的形式了，让开发者不必再直接操作底层的`thread`，而是转变成操作`queue`这种FIFO形式的数据结构。

那么此时，我们可以不必直接面对`thread`，而是更多的面对`queue`。以前的话，是尽量的缓存`thread`，那么现在就应该是尽量的缓存`queue`。

可以像`YYDispatchQueuePool`实现一个缓存GCD Serial Queue的一个队列缓存容器，来很好的避免无限制的创建GCD Serial Queue实例。

本篇文字也是学习了`YYDispatchQueuePool`源码实现，其主要实现了一个类似线程池的容器，只不过缓存的是各种`dispatch_queue_t`实例，而不是一个最终的底层`thread`。

作者是直接定义了缓存`32`个`serial dispatch_queue_t`实例，作为线程池内的工作线程。

但是，我有一个疑问，为什么固定创建32个线程队列了？而不是4、8、16 甚至1024个了？

而且我查阅了一些资料，创建多少个线程存放到缓存池，也与当前CPU的核心数是有关系的。且需要判断CPU处理任务的类型: (1) 密集型 (2) I/O型。对于这两种类型的CPU，然后再结合CPU的核心数、线程任务处理时间、线程等待时间...通过一个公式计算得到最适合的最大线程并发数。

只有调试找到适合CPU最大的一个线程并发数，才能彻底发挥出CPU的并发处理性能。如果线程少了，就浪费CPU的并发数。如果线程多了，又造成CPU并发压力大，线程切换消耗。

###线程池用来控制系统中最大并发的线程数量，那么到底控制到多少才会最好了？

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
最佳线程数目 = 1 * CPU总核数
```

也就是直接等于`CPU的总核心数`。但是从另外一篇文章中，得到这样的解释:

> 对于计算密集型任务，在拥有Ncpu个处理器的系统上，当线程池大小为`CPU核心数 + 1`时，通常能实现最优的利用率。因为当计算密集型任务时，可能偶尔由于`页缺失`故障或者其他原因而暂停时，这个`额外的线程`就能够确保CPU的时钟周期不会被浪费。http://www.iteye.com/problems/95917。

觉得还是有一定的道理的，所以我决定将预先缓存生成的`dispatch_queue_t`的个数设置为`CPU核心数 + 1`。

****

###对于优先使用iOS8推荐的QualityOfService描述线程队列，所以基于此大致得出将要实现的dispatch queue pool的内存结构

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