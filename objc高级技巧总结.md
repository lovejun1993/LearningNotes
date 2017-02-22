
## 苹果开源代码

```
https://opensource.apple.com/tarballs/CF/
https://github.com/opensource-apple/CF
https://github.com/opensource-apple/objc4
```

## 记录下苹果文档查询的路径:

```
https://developer.apple.com/search/?q=
```

## 最好不要在load方法中写过多耗时的代码

因为App启动时，会等待所有objc类的load方法实现全部执行完，才会走后面的代码逻辑。

## 一段代码执行时间计算

```c
int count = 100000;
double date_s = CFAbsoluteTimeGetCurrent();

......

double date_current = CFAbsoluteTimeGetCurrent() - date_s;
NSLog(@"consumeTime: %f μs",date_current * 11000 * 1000);
```

## sendAction 发送UIEvent事件

- UIApplication

```c
sendAction:to:from:forEvent:
```

- UIControl

```c
sendAction:to:forEvent:
```

可以通过替换SEL指向的IMP，来拦截这两个负责发送UI事件的方法实现，来做一些额外的处理。

## 异步子线程完成图像绘制

```c
- (void)display {

	//1. 
    dispatch_async(backgroundQueue, ^{
        CGContextRef ctx = CGBitmapContextCreate(...);
        // draw in context...
        CGImageRef img = CGBitmapContextCreateImage(ctx);
        CFRelease(ctx);
        
        //2.
        dispatch_async(mainQueue, ^{
            layer.contents = img;
        });
    });
}
```

先子线程完成图像的绘制渲染，最后回到主线程设置处理后的图像。

## 将一些不太重要的代码放在 `idle（空闲）` 时去执行

```objc
- (void)idleNotificationMethod { 
    // do something here 
} 
 
- (void)registerForIdleNotification  
{ 
	//1. 关注通知 IdleNotification
    [[NSNotificationCenter defaultCenter] addObserver:self 
        selector:@selector(idleNotificationMethod) 
        name:@"IdleNotification" 
        object:nil]; 
        
    //2. 构建通知
    NSNotification *notification = [NSNotification 
        notificationWithName:@"IdleNotification" object:nil]; 

	//3. 异步发送通知，并使用空闲模式
    [[NSNotificationQueue defaultQueue] enqueueNotification:notification 
    postingStyle:NSPostWhenIdle]; 
}  
```

##  在自定义线程上使用释放池

```objc
+ (NSThread *)networkRequestThread {
	static NSThread *_networkRequestThread = nil;
	static dispatch_once_t oncePredicate;
	dispatch_once(&oncePredicate, ^{
	    
		//1. 创建单例NSThread对象
		_networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
		
		//2. 开启线程对象
		[_networkRequestThread start];
	});

	return _networkRequestThread;
}

///NSThread入口函数
+ (void)networkRequestThreadEntryPoint:(id)__unused object {

	// 使用释放池进行包裹
	@autoreleasepool {
		[[NSThread currentThread] setName:@"AFNetworking"];
		NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
		[runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
		[runLoop run];
	}
}
```

## 对象关联时所有内存管理策略

| 关联对象指定的内存策略 | 等效的OC对象内存管理修饰符  | 
| :-------------: |:-------------:| 
| `OBJC_ASSOCIATION_ASSIGN` | atomic + assign |
| `OBJC_ASSOCIATION_RETAIN_NONATOMIC` | nonatimic + retain |
| `OBJC_ASSOCIATION_COPY_NONATOMIC` | nonatimic + copy |
| `OBJC_ASSOCIATION_RETAIN` | atomic + retain |
| `OBJC_ASSOCIATION_COPY` | atomic + copy |

没有指定`nonatomic`，默认就是`atomic`原子属性同步多线程。

## 如果一个方法在一个循环次数非常多的循环中使用，在进入循环前使用 `methodForSelector:` 获取该方法 IMP，然后缓存起来，以后每次调用该oc函数时，直接使用IMP，这种技术成为IMP Caching.

首先，有一个测试类:

```objc
@interface Person : NSObject
+ (void)logName1:(NSString *)name;
- (void)logName2:(NSString *)name;
@end
@implementation Person
+ (void)logName1:(NSString *)name {
    NSLog(@"log1 name = %@", name);
}
- (void)logName2:(NSString *)name {
    NSLog(@"log2 name = %@", name);
}
@end
```

然后ViewController测试IMP Caching:

```c
#import <objc/runtime.h>

static id PersonClass = nil;
static SEL PersonSEL1;
static SEL PersonSEL2;
static IMP PersonIMP1;
static IMP PersonIMP2;

@implementation ViewController

+ (void)initialize {
    PersonClass = [Person class];
   
    PersonSEL1 = @selector(logName1:);
    PersonSEL2 = @selector(logName2:);
    
    //获取类方法实现
    PersonIMP1 = [PersonClass methodForSelector:PersonSEL1];
    
    //获取对象方法实现
    PersonIMP2 = method_getImplementation(class_getInstanceMethod(PersonClass, PersonSEL2));
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

    //1. 调用类方法实现
    ((void (*)(id, SEL, NSString*)) (void *) PersonIMP1)(PersonClass, PersonSEL1, @"我是参数");
    
    //2. 调用对象方法实现
    ((void (*)(id, SEL, NSString*)) (void *) PersonIMP2)([Person new], PersonSEL2, @"我是参数");
    
    NSLog(@"");
}

@end
```

输出结果

```
2017-02-08 22:47:46.586 Test[805:25490] log1 name = 我是参数
2017-02-08 22:47:46.587 Test[805:25490] log2 name = 我是参数
```

##  `_objc_msgForward` iOS系统消息转发c函数指针

还有与之差不多意思的:

```c
_objc_msgForward_stret
```

jspatch恰恰就是利用的这个`_objc_msgForward`c方法实现，达到交换任意Method的SEL指向的IMP。

当一个oc类中，找不到某一个SEL对应的IMP时，会进入到系统的消息转发函数。

下面测试下，`_objc_msgForward`到底是如何转发消息的？

首先，有如下测试类:

```objc
@interface Person : NSObject
+ (void)logName1:(NSString *)name;
- (void)logName2:(NSString *)name;
@end
@implementation Person
+ (void)logName1:(NSString *)name {
    NSLog(@"log1 name = %@", name);
}
- (void)logName2:(NSString *)name {
    NSLog(@"log2 name = %@", name);
}
@end
```

ViewController中随便执行一个Perosn对象不存在实现的SEL消息:

```objc
@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

    // 此处打一个断点，然后执行下面的lldb调试命令后，再往下执行代码
    static Person *person;
    person = [Person new];
    [person performSelector:@selector(hahahaaha)];
    
    NSLog(@"");
}

@end
```

断点后，在lldb中输入如下调试命令，会打印出所有运行时发送的消息:

```c
(lldb) call (void)instrumentObjcMessageSends(YES)
```

程序崩溃后，进入Mac电脑系统如下目录:

```c
cd /tmp/
```

找到该目录下类似如下结构的文件，然后打开

```
msgSends-901
```

打开文件后，只看与`Person`相关的信息大概为如下:

```objc
+ Person NSObject initialize
+ Person NSObject new
- Person NSObject init
- Person NSObject performSelector:
+ Person NSObject resolveInstanceMethod:
+ Person NSObject resolveInstanceMethod:
- Person NSObject forwardingTargetForSelector:
- Person NSObject forwardingTargetForSelector:
- Person NSObject methodSignatureForSelector:
- Person NSObject methodSignatureForSelector:
- Person NSObject class
- Person NSObject doesNotRecognizeSelector:
- Person NSObject doesNotRecognizeSelector:
- Person NSObject class
```

从`Person NSObject performSelector:`开始执行一个不存在实现SEL消息后，依次开始执行:

- (1) `resolveInstanceMethod:`
- (2) `forwardingTargetForSelector:`
- (3) `methodSignatureForSelector:`

所以，`_objc_msgForward`这个指针指向的c函数的作用，就是进入到消息转发阶段，从阶段1到阶段2，如果最后阶段仍然无法处理消息，就产生异常让程序退出。

## 内存对象的优化

### 一、大的对象在`子线程`创建

- (1) 文件读入为对象
    - plist
    - NSCoding
    - 图片
    - 各种资源文件
- (2) sqlite表数据读入为对象


### 二、大量的对象在`子线程`释放废弃

测试类打印dealloc信息

```objc
@interface Cat : NSObject
@property (nonatomic, copy) NSString *name;
@end
@implementation Cat
-(void)dealloc {
    NSLog(@"Cat %@ dealloc on thead %@", _name, [NSThread currentThread]);
}
@end
```

ViewController测试异步释放数组对象

```objc
@implementation ViewController

- (void)asyncReleaseArray {
    
    //1. 假设存在一个很多子对象的数组
    NSMutableArray *dataSource = [NSMutableArray new];
    for (int i = 0; i < 3; i++) {
        Cat *cat = [Cat new];
        cat.name = [NSString stringWithFormat:@"cat%d", i+1];
        [dataSource addObject:cat];
    }
    
///////////////2. 现在要在子线程完成对数组对象的释放，如下固定三步/////////////////
    
    //第一步、增加一个临时数组持有所有的子对象，此时存在2个数组（dataSource、holder）持有所有的子对象
    NSArray *holder = [[NSArray alloc] initWithArray:dataSource];
    
    //第二步、释放掉之前的数组对象，此时只有holder数组持有所有的子对象
    dataSource = nil;
    
    //第三步、将holder数组的释放操作，异步放到子线程
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        [holder count];//当在子线程执行完方法后，该临时指针变量被废弃，于是holder指向的数组对象被废弃
    });
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
	[self asyncReleaseArray]; 
    NSLog(@"");
}

@end
```

输出信息

```
2017-02-08 23:32:25.567 Test[1107:57254] Cat cat1 dealloc on thead <NSThread: 0x7f9322598970>{number = 2, name = (null)}
2017-02-08 23:32:25.567 Test[1107:57254] Cat cat2 dealloc on thead <NSThread: 0x7f9322598970>{number = 2, name = (null)}
2017-02-08 23:32:25.567 Test[1107:57254] Cat cat3 dealloc on thead <NSThread: 0x7f9322598970>{number = 2, name = (null)}
```

可以看到Cat对象都是在子线程完成废弃的，并不会全部压在主线程进行废弃。

### YYDispatchQueuePool中的写法


```objc
- (void)removeAll {
    _totalCost = 0;
    _totalCount = 0;
    _head = nil;
    _tail = nil;
    
    if (CFDictionaryGetCount(_nodeMap) > 0) {
        CFMutableDictionaryRef holder = _nodeMap;//dic retainCount = 2
        _nodeMap = CFDictionaryCreateMutable(CFAllocatorGetDefault(), 0, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);//dic retainCount = 1
        
        /**
         *  所有的node缓存节点内存释放与废弃
         *  - 异步释放与废弃
         *      - 主线程
         *      - 子线程
         *  - 同步释放与废弃
         *      - 当前创建node对象的所在线程（node对象的创建基本上都处于主线程）
         */
        
        if (_releaseAsynchronously) {
            if (_releaseOnMainThread && !pthread_main_np()) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    CFRelease(holder);//主线程上异步释放所有的对象, dic retainCount = 0
                });
            } else {
                dispatch_queue_t queue = _releaseOnMainThread ? dispatch_get_main_queue() : XZHMemoryCacheGetReleaseQueue();
                dispatch_async(queue, ^{
                    CFRelease(holder);//子线程上异步释放所有的对象, dic retainCount = 0
                });
            }
        } else {
            CFRelease(holder);// 当前线程同步释放对象, dic retainCount = 0
        }
    }
}
@end
```

### 通常是在子线程创建大体积对象，然后回调主线程使用。或者将主线程上不再使用的对象，异步放到子线程完成释放废弃。

- (1) 主线程创建对象 >>> 子线程异步释放与废弃
- (1) 子线程创建对象（读取文件为对象） >>> 主线程使用完毕 >>> 主线程异步释放与废弃

(1)、(2)的做法都是正确的。因为:

```
一个对象 >>> 一个内存块。
一个线程 >>> 一个代码执行路径。
```

那么任意的代码执行路径，都能够访问到某一块内存块，所以任意的线程是能够操作内存中任意的内存块数据。

所以也就是活，线程与对象是没有啥关系的，任意的对象可以在任意的线程上进行创建与释放，但是除开`UIKit`对象。

但是(2)不合常理，会影响主线程的执行效率，所以不推荐。

## 使用 `__unsafe_unretained`来修饰指针变量，不要滥用`__weak`会导致对象自动注册到自动释放池，从而影响运行速度

- (1) 如果一个对象确定是不会被废弃，或者调用完成之前不会被废弃

- (2) 那么，声明一个指向该对象的指针变量时，就是要`__unsafe_unretained`来修饰

- (3) `__unsafe_unretained`就是简单的拷贝`地址`，不进行任何的`对象内存管理`
    - 即不修改retainCount


## struct实例去持有 Foundation对象，当struct实例废弃时，要让Foundation对象在子线程上异步释放废弃

### 核心主要牵涉三个用于`c实例`与`objc对象`进行转换的东西

一、`(__bridge_retained CoreFoundation实例)Foundation对象`

- (1) Foundation对象 >>> CoreFoundation实例
- (2) [Foundation对象 retain]

二、`(__bridge_transfer Foundation对象)CoreFoundation实例`

- (1) CoreFoundation实例 >>> Foundation对象 
- (2) [转换后的Foundation对象 release]

三、`__bridge` 

- (1) Foundation对象 >>> CoreFoundation实例
- (2) CoreFoundation实例 >>> Foundation对象 
- (3) 不会执行任何的`retain/release`效果，仅仅只是类型的转换

### 下面demo测试

Foundation 类

```objc
@interface Dog : NSObject
@property (nonatomic, copy) NSString *name;
@end
@implementation Dog
- (void)dealloc {
    NSLog(@"废弃Dog对象，name = %@ on thread = %@", _name, [NSThread currentThread]);
}
@end
```

struct实例 使用 `void*` 万能指针类型持有 Foundation对象

```c
typedef struct DogsContext {
    void    *dogs;//持有oc数组对象
}DogsContext;
```

ViewController测试代码

```objc
static DogsContext *_dogsCtx = NULL;

@implementation ViewController

// 测试结构体实例持有oc对象
- (void)testARCBridge1 {
    
    //1. c结构体实例
    _dogsCtx = malloc(sizeof(DogsContext));
    _dogsCtx->dogs = NULL;
    
    //2. 创建测试的oc数组对象
    NSMutableArray *dogs = [NSMutableArray new];
    for (int i = 0; i < 3; i++) {
        Dog *dog = [Dog new];
        dog.name = [NSString stringWithFormat:@"name_%d", (i + 1)];
        NSLog(@"创建Dog对象，name = %@", dog.name);
        [dogs addObject:dog];
    }
    
    //3. struct实例 持有 NSFoundation对象，并对oc对象进行retain，防止oc对象被废弃
    _dogsCtx->dogs = (__bridge_retained void*)dogs;
}

// 测试从结构体实例中取出oc对象使用，然后不再需要的时候全部一起废弃
- (void)testARCBridge2 {
    
    //1. 先取出c struct实例持有的 NSFoundation对象使用，不进行任何retain/release
    NSMutableArray *array1 = (__bridge NSMutableArray*)_dogsCtx->dogs;
    for (Dog *dog in array1) {
        NSLog(@"使用Dog对象，name = %@", dog.name);
    }
    
    //2. 释放struct实例持有的NSMutableArray数组，继而释放掉了NSMutableArray数组持有的所有的Dogs对象
    //2.1 对oc数组对象进行release
    NSMutableArray *array2 = (__bridge_transfer NSMutableArray*)_dogsCtx->dogs;
    //2.2 解决结构体实例指向oc数组对象，并废弃结构体实例
    _dogsCtx->dogs = NULL;
    free(_dogsCtx);
    _dogsCtx = NULL;
    //2.3 子线程异步释放废弃oc数组内的其他子对象
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        [array2 class];
    });
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

        [self testARCBridge1];
    [self testARCBridge2];
    
    NSLog(@"");
}

@end
```

输出信息

```
2017-02-08 23:54:34.433 Test[1262:71331] 创建Dog对象，name = name_1
2017-02-08 23:54:34.434 Test[1262:71331] 创建Dog对象，name = name_2
2017-02-08 23:54:34.434 Test[1262:71331] 创建Dog对象，name = name_3

2017-02-08 23:54:34.434 Test[1262:71331] 使用Dog对象，name = name_1
2017-02-08 23:54:34.434 Test[1262:71331] 使用Dog对象，name = name_2
2017-02-08 23:54:34.434 Test[1262:71331] 使用Dog对象，name = name_3

2017-02-08 23:54:34.435 Test[1262:71700] 废弃Dog对象，name = name_1 on thread = <NSThread: 0x7ff0c8e68a50>{number = 2, name = (null)}
2017-02-08 23:54:34.435 Test[1262:71700] 废弃Dog对象，name = name_2 on thread = <NSThread: 0x7ff0c8e68a50>{number = 2, name = (null)}
2017-02-08 23:54:34.435 Test[1262:71700] 废弃Dog对象，name = name_3 on thread = <NSThread: 0x7ff0c8e68a50>{number = 2, name = (null)}
```

## 限制无限去创建 gcd dispatch queue

YYDispatchQueuePool的核心几点:

- (1) iOS8之前使用Priority，iOS8及之后使用QualityOfService，来操作`dispatch_queue_t`

- (2) 根据 Priority或iOS8及之后使用QualityOfService，不同的各种等级，分别创建一个Context内存块

- (3) 每一个Context内存块，保存`[NSProcessInfo processInfo].activeProcessorCount`个 dispatch serial queue实例

- (4) 根据对应的等级，从对应的Context中，随机取出一个dispatch serial queue实例，进行任务调度

- (5) 一个`dispatch serial queue实例`，就是`一个`底层线程

- (6) 一个`dispatch concurrent queue实例`，就是`n个`底层线程

这样避免无限制的创建dispatch queue实例，导致底层线程也会无限制的创建，没有办法复用。

并且每一种Context下，预先缓存`当前CPU硬件激活的核心数`个dispatch queue实例，可以让CPU在该Context等级下，满负荷运行，让CPU充分利用。

而不会因为线程太多，导致CPU在多线程之间切换、竞争消耗无畏的资源。


## `@package`块声明ivar

让一些Ivar只想在`静态类库`或`framework`中的类对象中的代码可以访问。


## 主线程上常见比较耗时的代码类型:

- (1) Layout 布局计算
	- 计算文本内容的 宽度计算、高度计算
	- UI对象的 frmae计算、frmae设置、frmae调整

- (2) Rendering 显示数据渲染
	- 文本内容的渲染
	- 图片的解码
	- 图形的绘制

- (3) UIKit Obejcts UI对象
	- UI对象的属性值调整
	- UI对象的创建
	- UI对象的销毁（废弃）

## 在`for/while`等循环中`加锁`时，需要使用`tryLock`，不能使用`lock`，否则可能会出现线程死锁

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


注意是执行`加锁`处理的才需要使用`tryLock`:

- (1) NSLock

```c
BOOL isGetLock = -[NSLock tryLock];
```


- (2) `pthread_mutex_t`

```c
pthread_mutex_t mutex;

......

BOOL isGetLock = pthread_mutex_trylock(&mutex);
```

## 实现弱引用objc对象

### 方法一、使用NSValue提供的方法

```objc
//1. NSValue弱引用方式包装一个objc对象
NSValue *value = [NSValue valueWithNonretainedObject:@"objc对象"];

//2. 从NSValue获取弱引用的objc对象
id weakObj = [value nonretainedObjectValue];
```

### 方法二、借助Block对象，内部持有一个使用了`__weak`修饰的objc对象

- (1) 返回值类型是id，参数类型是void，的block类型定义

```c
typedef id (^WeakReferenceBlcok)(void);
```

- (2) 将外界传入的需要弱引用处理的对象，借助block，进行`__weak`处理

```c
WeakReferenceBlcok makeWeakReference(id obj) {
    
    //1. 对外界传入的对象进行弱引用
    id __weak weakObj = obj;
    
    //2. 返回一个Block，执行Block后，让外界拿到 __weak 处理后的弱引用用对象
    return ^() {
        return weakObj;
    };
}
```

- (3) 对NSMutableDictionary稍加封装，添加如上的处理代码

```objc
@interface XZHDic : NSObject {
    NSMutableDictionary *_dic;//TODO: 初始化代码那些就不写了.....
}

- (void)weak_setObject:(id)anObject forKey:(NSString *)aKey;
- (id)weak_getObjectForKey:(NSString *)key;

@end
@implementation XZHDic

- (void)weak_setObject:(id)anObject forKey:(NSString *)aKey {
    //1.
    WeakReferenceBlcok block = makeWeakReference(anObject);
    
    //2.
    [_dic setObject:block forKey:aKey];
}

- (id)weak_getObjectForKey:(NSString *)key {
    //1.
    WeakReferenceBlcok block = [_dic objectForKey:key];
    
    //2.
    return (block ? block() : nil);
}

@end
```

### NSProxy定义weak属性 + 消息转发

- (1) 抽象事物的接口

```objc
#import <Foundation/Foundation.h>

@protocol Human <NSObject>
- (void)run;
@end
```

- (2) 一个具体实现类

```objc
#import <Foundation/Foundation.h>
#import "Human.h"

@interface XiaoMing : NSObject <Human>
@end
@implementation XiaoMing
- (void)run {
    NSLog(@"XiaoMing run....");
}
@end
```

- (3) NSProxy代理上面具体实现类的一个对象，并且使用weak修饰的属性

```objc
#import <Foundation/Foundation.h>
#import "Human.h"

@interface XZHProxy : NSProxy <Human>

/**
 *  被代理的弱引用对象
 */
@property (nonatomic, weak, readonly) id<Human> target;

/**
 *  传入要被弱引用的对象
 */
- (instancetype)initWithTarget:(id<Human>)target;

@end
@implementation XZHProxy

- (instancetype)initWithTarget:(id<Human>)target {
    _target = target;
    return self;
}

- (id)forwardingTargetForSelector:(SEL)selector {
    return _target;
}

- (BOOL)respondsToSelector:(SEL)aSelector {
    return [_target respondsToSelector:aSelector];
}

@end
```

外界使用代码

```objc
//1. target
XiaoMing *xiaoming = [[XiaoMing alloc] init];

//2. Proxy
XZHProxy *proxy = [[XZHProxy alloc] initWithTarget:xiaoming];

//3. 使用Proxy
[proxy run];
```

输出

```
2016-11-16 00:07:33.180 demo[9202:125702] XiaoMing run....
```

## `hitTest:withEvent:` 与 `pointInside:withEvent:`

### 关于`-[UIView hitTest:withEvent:]`的源码大致实现

```objc
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event
{
    // 1. 判断自己是否能够接收触摸事件（是否打开事件交互、是否隐藏、是否透明）
    if (self.userInteractionEnabled == NO || self.hidden == YES || self.alpha <= 0.01) return nil;
    
    // 2. 调用 pointInside:withEvent:， 判断触摸点在不在自己范围内（frame）
    if (![self pointInside:point withEvent:event]) return nil;
    
    // 3. 从`上到下`（最上面开始）遍历自己的所有子控件，看是否有子控件更适合响应此事件
    int count = self.subviews.count;
    for (int i = count - 1; i >= 0; i--) {
        UIView *childView = self.subviews[i];
        
        // 将产生事件的坐标，转换成当前相对subview自己坐标原点的坐标
        CGPoint childPoint = [self convertPoint:point toView:childView];
        
        // 又继续交给每一个subview去hitTest
        UIView *fitView = [childView hitTest:childPoint withEvent:event];
        
        // 如果childView的subviews存在能够处理事件的，就返回当前遍历的childView对象作为事件处理对象
        if (fitView) {
            return fitView;
        }
    }
    
    //4. 没有找到比自己更合适的view
    return self;
}
```

可以看到这个`hitTest:withEvent:`函数实现，主要就是测试这个UIView对象，到底能不能够处理这个UI触摸事件。

事件查找顺序是这样的:

```
- UIApplication
	- UIWindow
		- RootView
			- Subviews
```

而最终Subviews是从`最上面`的subview开始hitTest:。

### 使用Category Associate 扩大UI的事件响应区域

```objc
#import <UIKit/UIKit.h>

@interface UIButton (EnlargeTouchArea)

/**
 *  设置按钮上下左右的扩展响应区域
 */
- (void)setEnlargeEdgeWithTop:(CGFloat)top
                        right:(CGFloat)right
                       bottom:(CGFloat)bottom
                         left:(CGFloat)left;

@end
```

```
#import "UIButton+EnlargeTouchArea.h"
#import <objc/runtime.h>

static void *kButtonUpKey = &kButtonUpKey;
static void *kButtonLeftKey = &kButtonLeftKey;
static void *kButtonDownKey = &kButtonDownKey;
static void *kButtonRightKey = &kButtonRightKey;

@implementation UIButton (EnlargeTouchArea)

- (void)setEnlargeEdgeWithTop:(CGFloat)top right:(CGFloat)right bottom:(CGFloat)bottom left:(CGFloat)left
{
    objc_setAssociatedObject(self, kButtonUpKey, @(top), OBJC_ASSOCIATION_ASSIGN);
    objc_setAssociatedObject(self, kButtonLeftKey, @(left), OBJC_ASSOCIATION_ASSIGN);
    objc_setAssociatedObject(self, kButtonDownKey, @(bottom), OBJC_ASSOCIATION_ASSIGN);
    objc_setAssociatedObject(self, kButtonRightKey, @(right), OBJC_ASSOCIATION_ASSIGN);
}

- (CGRect) enlargedRect
{
    NSNumber* topEdge = objc_getAssociatedObject(self, &kButtonUpKey);
    NSNumber* rightEdge = objc_getAssociatedObject(self, &kButtonRightKey);
    NSNumber* bottomEdge = objc_getAssociatedObject(self, &kButtonDownKey);
    NSNumber* leftEdge = objc_getAssociatedObject(self, &kButtonLeftKey);
    
    if (topEdge && rightEdge && bottomEdge && leftEdge)
    {
        // 上下左右分别扩大响应区域
        return CGRectMake(
                          self.bounds.origin.x - leftEdge.floatValue,
                          self.bounds.origin.y - topEdge.floatValue,
                          self.bounds.size.width + leftEdge.floatValue + rightEdge.floatValue,
                          self.bounds.size.height + topEdge.floatValue + bottomEdge.floatValue
                          );
    } else {
        return self.bounds;
    }
}

- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    
    // 扩大后的响应区域
    CGRect rect = [self enlargedRect];
    
    // 如果扩大的响应区域 == 当前自身的响应区域，直接执行父类的事件处理
    if (CGRectEqualToRect(rect, self.bounds))
    {
        return [super hitTest:point withEvent:event];
    }
    
    // 扩大的响应区域 > 当前自身的响应区域
    return CGRectContainsPoint(rect, point) ? self : nil;
}

@end
```

## 位移枚举 + Mask掩码

这种枚举适用于一个统一的枚举类型来定义，具备:

- (1) 多种情况
- (2) 每一种情况，又分为其他的小情况

一个简单的demo


```objc
typedef NS_OPTIONS(NSInteger, PersonState) {
	
	// 第一种类型: 占用1~8位的二进制位，掩码是 1111,1111
    PersonStateMask                     = 0xFF,//1-8位的掩码（十六进制数，一个数代表4位，F:1111，0:0000）
    PersonStateUnknown                  = 0,
    PersonStateAlive                    = 1,
    PersonStateWork                     = 2,
    PersonStateDead                     = 3,

	// 第二种类型: 占用9~16位的二进制位，掩码是 1111,1111,0000,0000   
    HouseStateMask                      = 0xFF00,//9-16位，左移8位
    HouseStateNone                      = 1 << 8,
    HouseStateSmall                     = 1 << 9,
    HouseStateBig                       = 1 << 10,
    
	// 第三种类型: 占用17~24位的二进制位，掩码是 1111,1111,0000,0000,0000,0000   
    CarStateMask                        = 0xFF0000,//17-24位，左移16位
    CarStateNone                        = 1 << 16,
    CarStateSmall                       = 1 << 17,
    CarStateBig                         = 1 << 18,
};
```

如下就是分别获取得到 1~8位、9~16位、17~24位 这三个区段的所谓的Mask掩码

```c
0xFF		>>> 1111,1111 >>> 获取低8位值
0xFF00 		>>> 1111,1111,0000,0000 >>> 获取9-16位值
0xFF0000  	>>> 1111,1111,0000,0000,0000,0000 >>> 获取17-24位值
```

使用当前的枚举混合值，通过与Mask掩码，进行`按位与`获取Mask掩码对应长度的值:

```c
(1) & 上 `FF` 获取低8位的值
(2) & 上 `FF00` 获取第9位到16位的值
(3) & 上 `FF0000` 获取第17位到24位的值
```

示例代码

```c
//1.
PersonState state = PersonStateUnknown;

//2.
state = PersonStateDead;

//3.
state = state | HouseStateBig;
NSLog(@"state = %ld", state);
NSLog(@"person state = %ld", state & PersonStateMask);
NSLog(@"house state = %ld", state & HouseStateMask);

//4.
state = state | CarStateBig;

//5. 
NSLog(@"state = %ld", state);
NSLog(@"person state = %ld", state & PersonStateMask);
NSLog(@"house state = %ld", state & HouseStateMask);
NSLog(@"car state = %ld", state & CarStateMask);
```

## objc对象/c struct实例中，实例变量/成员变量的`内存布局` 规则:


```objc
@interface Cat : NSObject {

    @public
    NSString *_name;
    NSString *_type;
    
    @private
    NSString *_cid;
}
@end
@implementation Cat
@end
```

如上写法定义的实例变量的`地址布局`，在程序代码`编译期间`就已经确定了。那么如果运行时去修改这个布局，就会导致Ivar地址错乱，出现存取数据错误。


### 对象的所有实例变量的布局计算规律是，按照实例变量的定义`从上到下`的顺序

- 每一个实例变量的布局起始地址，排列在上一个实例变量地址的`末尾`
- 每一个实例变量占用的`内存长度`，就是是自己数据类型的占用总长度
- 每一个实例变量的`偏移量offset`，都是相对于`所属对象`所在内存块的`起始地址`
- OC对象实例变量的偏移量计算，和`struct结构体属性`的偏移量计算是非常相似的，也存在`字节对齐`的问题

### 现代计算机都是使用`字节`进行内存地址分配，也就存在`字节对齐`的问题

结构体中的元素布局（字节对齐），如下两个结构体，元素都是一样的，以32位系统为例.

```c
struct A {
    int a;      // 占4个字节
    char b;     // 占1个字节
    short c;    // 占2个字节
};
```

```c
struct B {
    char a;     // 占1个字节
    int b;      // 占4个字节
    short c;    // 占2个字节
};
```

看起来的话两个结构体的总长度都是7个字节，但是如下打印却是:

```c
int main() {
    
    printf("%lu\n", sizeof(struct A));
    printf("%lu\n", sizeof(struct B));
    
    return 0;
}
```

```
8
12
Program ended with exit code: 0
```

长度分别是8和12，那么为什么不是7了？原因是`字节对齐`的规则。

现代计算机内存单元，都是使用`字节（代替8个二进制位）`作为基本单位划分。理论上说可以从任何的内存地址来访问到变量的所在内存单元。但是实际上，为了提升效率，各种类型的数据地址是按照`一定的规律来有序排列`，那么所以对于某一些类型的数据访问，可以直接从特定的位置上开始访问，就不用从头开始遍历寻找。牺牲空间来换取时间，这就是字节对齐。

### 结构体默认的字节对齐一般满足三个标准:

- (1) 结构体变量的`首地址`能够被该结构体中`最大长度`的成员变量的长度所`整除`.
	- 首地址不好控制，一般程序操作不了的，直接由操作系统分配的

- (2) 结构体变量中每个成员变量相对于`结构体变量首地址`的偏移量（offset）都是成员变量自身长度的`整数倍`.
	- `成员变量的偏移量`不够时由编译器自动填充

- (3) 结构体变量的总长度为成员变量中最大长度的`整数倍`.
	- `结构体的总长度`不够时由编译器自动填充


### 编译器会自动进行字节对齐，使用空白无用的内存字节作为填充。

那么对于上面的`结构体A变量`的内存字节布局是这样的:

- 首先变量a是int类型，占用4个字节
	- 第一个变量直接放
- 然后变量b是char类型，占用1个字节
	- 相对于起始地址的偏移量 = 4，满足规则2
	- 所以直接从第五个字节开始存放b，并占用一个字节长度
- 最后变量c是short类型，占用2个字节
	- 此时相对于起始地址的偏移量 = 5，`不满足规则2`
	- 所以向后填充一个字节，这个填充字节什么事都不干，就是为了字节对齐
	- 那么此时变量c相对起始地址的偏移量 = 6，刚好是2的整数倍，c占用两个字节
- 所以最终长度 = 8个字节


### 那么对于`结构体变量B`的内存字节布局是这样的:

- 首先变量a是char类型，占用1个字节
	- 第一个变量直接放
- 然后变量b是int类型，占用4个字节
	- 此时相对于起始地址的偏移量 = 1，`不满足规则2`
	- 为了让b相对起始地址的偏移量是int长度（4）的整数倍
	- 所以修改为`偏移量 = 4`，就是说`空出三个`没用的字节，什么都不干，就是为了字节对齐
	- 从第5个字节开始排布变量b，并占用4个字节长度
	- 此时总长度为8个字节
- 最后变量c是short类型，占用2个字节
	- 此时相对于起始地址的偏移量 = 8，`满足规则2`，直接排布c
	- c此时占用2个字节长度
	- 那么此时结构体总长度 = 10，`不满足规则3`
- 为了让结构体总长度是成员最大长度的整数倍
	- 最大成员长度是b = 4
	- 所以最后扩展为长度 = 12，刚好大于10又是4的整数倍
- 所以最终的结构体B的长度是12

### 对于上面的两个结构体可以使用`保留字节`来避免编译器自动填充字节作为无用字节:

```c
struct A {
    int a;          	// 占4个字节
    char b;         	// 占1个字节
    char reserved;  	// 此处会由编译器填充1个字节，那么我们可以定义以后使用的保留变量来操作
    short c;        	// 占2个字节
};

struct B {
    char a;             // 占1个字节
    char reserved1[3];  // 此处会由编译器填充3个字节，那么我们可以定义以后使用的保留变量来操作
    int c;              // 占4个字节
    short b;            // 占2个字节
    char reserved2[2];  // 此处会由编译器填充2个字节，那么我们可以定义以后使用的保留变量来操作
};
```

对于上面使用reserved保留变量声明的内存字节，我们也可以去使用了。而如果我们不使用reserved保留变量去指向这些字节，那么这些字节就相当于放在那什么也不干就是为了字节对齐，什么用都没有。


### 对于结构体成员变量从上到下排布的原则:

- (1) 成员变量的排布按照自身的长度`从小到大`依次排列
- (2) 尽量的使用`保留字段`，来使用由编译器进行字节对齐时填充无用的字节

### 对上面两个结构体按照如上原则最后的修改版本为如下:

```c
struct C {
    char a;             // 占1个字节
    char reserved;      // 此处会由编译器填充1个字节，那么我们可以定义以后使用的保留变量来操作
    short b;            // 占2个字节
    int c;              // 占4个字节
};
```

### 再回头看看Cat对象的所有实例变量的内存地址布局如下所示:

| 属性相对Cat对象起始地址的偏移量offset | Cat对象的实例变量 | 
| :-------------: |:-------------:| 
| +0 字节 | `_firstName` | 
| +4 字节 | `_lastName` | 
| +8 字节 | `_pid` | 

上面所示的实例变量的内存布局在代码`编译期间`就已经确定了，如果是新增加一个实例变量或者减少一个实例变量，就必须要`重新编译`程序代码，让编译器重新计算所有实例变量的内存布局，否则就会出现地址错乱。

### 对于`Cat对象->_firstName`这句代码实际作用:

- (1) 首先找到当前`Cat对象`的所在内存的`起始地址`
- (2) 从某个全局记录表中，查询到`_firstName`这个Ivar对应的地址`偏移量offset`
- (3) 通过 `偏移量offset + Cat对象内存起始地址` 作为存取 实例变量 `_firstName` 的所在内存地址

### 如果在`程序运行`期间，通过`objc/runtime.h`中提供的运行时api来给Cat对象添加一个新的实例变量Ivar或者移除某一个已有的Ivar，会引出问题吗？

 肯定会的。

```objc
@interface Cat : NSObject {

    @public
    NSString *_runtimeAddIvar;//假设这个Ivar是在运行时添加
    
    NSString *_firstName;
    NSString *_lastName;
    
    @private
    NSString *_pid;
}

@end
```

如果运行时将Cat对象添加一个新的实例变量`_runtimeAddIvar`，那么此时Dog对象所有实例变量的内存布局应该是如下这样:

| 属性相对Cat对象起始地址的偏移量 | Cat对象的实例变量 | 
| :-------------: |:-------------:| 
| +0 | `_runtimeAddIvar` | 
| +4 | `_firstName ` | 
| +8 | `_lastName ` | 
| +12 | `_pid` | 


`如果系统没有更新保存实例变量对应的内存布局地址`，就会导致如下错误:

- (1) `Dog对象->_runtimeAddIvar`访问实例变量，实际上访问到了新添加的实例变量`_runtimeAddIvar`

- (2) `Dog对象->_firstName`访问实例变量，实际上访问到了新添加的实例变量`_lastName`

- (3) `Dog对象->_lastName`访问实例变量，实际上访问到了新添加的实例变量`_pid`

所以如果记录实例变量对应内存地址布局的全局配置中，没有得到及时的更新的话，就会造成实例变量的访问全部乱套。

### 系统会不会根据Class改变后自动进行调整布局吗？

OK，对如上的Cat类在运行时尝试添加Ivar看看。

```objc
- (void)demo1 {
    
    Cat *c = [Cat new];
    
    if(class_addIvar(objc_getClass("Cat"), "runtimeAddIvar", sizeof(NSString *), 0, "@")) {
        NSLog(@"add ivar success");
        
        c->_firstName = @"Li";
        c->_lastName = @"Ming";
        [c setValue:@"hello world" forKey:@"runtimeAddIvar"];
        
        NSLog(@"_firstName = %@", c->_firstName);
        NSLog(@"_lastName = %@", c->_lastName);
        NSLog(@"_runtimeAddIvar = %@", [c valueForKey:@"runtimeAddIvar"]);
        
    } else {
        NSLog(@"add ivar failed");
    }
}
```

我试了很久`class_addIvar()`方法返回值一直是`NO`，结果表示添加Ivar一直是失败的。我开始一直以为`class_addIvar()`方法中的参数有问题，但是尝试了很多次参数修改，就是无法添加成功。

我就很纳闷了，以前都可以添加成功的，为啥现在不行了？于是我打开我之前记录的添加Ivar成功的代码如下:

```objc
- (void)demo2 {
    
    // 运行时创建一个类
    Class MyClass = objc_allocateClassPair([NSObject class], "myclass", 0);
    
    // 添加一个NSString的实例变量，第四个参数是对其方式，第五个参数是参数类型编码
    if(class_addIvar(MyClass, "itest", sizeof(NSString *), 0, "@")) {
        NSLog(@"add ivar success");
    } else {
        NSLog(@"add ivar failed");
    }
    
    // 向运行时系统注册创建的类
    objc_registerClassPair(MyClass);
}
```

运行结果

```
2016-07-17 17:17:34.550 Demo[14993:239703] add ivar success
```

确实可以啊，为啥上面的demo1方法中就死添加不成功了，我对比两种添加的方式对比了好久好久，没有发现哪里不一样。但是突然看到了有一个很大的区别:

- (1) 前面的`Cat`类，是我在Xcode工程中添加的存在源文件的类

- (2) 后面的`MyClass`类，在代码编译期间是不存在的，只有程序运行期执行了`objc_allocateClassPair()`函数之后才会出现的类

我觉得这个区别就是导致`Cat`类不能添加Ivar的原因。于是，我在网上搜是否有这样的记录。于是在`http://www.cocoachina.com/ios/20141031/10105.html`中找到了答案:

```
- (1) Objective-C 不支持 往`已存在`的类中添加实例变量
	- 系统库中，已经存在的源码文件的类
	- 我们自己Xcode工程中，已经存在源码文件的类
	- 我理解`已存在`的意思是 >>> 已经存在源码文件定义的类

- (2) 只能向还没有存在（运行时动态创建并注册到系统的类）类，才能够使用`class_addIvar()`添加Ivar实例变量

- (3) 且必须注意 `class_addIvar()`函数出现位置必须满足:
	- (1) objc_allocateClassPair()之后 >>> 创建类之后
	- (2) objc_registerClassPair()之前 >>> 注册类之前

- (4) 不能够向`Meta元类`进行使用 >>> 因为实例变量只能够出现在OC对象
```

第三点，我觉得是最重要的，其实我们所编写出现在 `.h与.m 中的Objetive-C类`其实最终也是走上面出现的那一段注册类代码步骤:

```objc
// 1. 运行时创建一个类
Class MyClass = objc_allocateClassPair([NSObject class], "myclass", 0);
    
// 2. 添加一个NSString的实例变量，第四个参数是对其方式，第五个参数是参数类型编码
if(class_addIvar(MyClass, "itest", sizeof(NSString *), 0, "@")) {
    NSLog(@"add ivar success");
} else {
    NSLog(@"add ivar failed");
}
    
// 3. 向运行时系统注册创建的类
objc_registerClassPair(MyClass);
```

我觉得这几个步骤已经由运行时runtime库帮我们做了。runtime自动读取我们的.h与.m，并使用如下这些函数将我们编写的类注册到运行时系统环境，以便后续使用:

- (1) `objc_allocateClassPair()` 创建一个类
- (2) `class_addIvar()` 添加实例变量
- (3) `class_addMethod()` 添加实例方法
- (4) `objc_registerClassPair()` 将类注册到运行时环境，这一步可能就会进行Ivar的内存布局计算了，之后就无法再改变了，所以这就是为什么之前给Cat类添加Ivar不成功的原因

### 对于已经执行`objc_registerClassPair()`的类不能够使用`class_addIvar()`来添加实例变量，这是否就是对之前的可能会出现Ivar地址错乱问题的规避了？

问题: 如果运行时动态给对象添加一个实例变量时，如果不重新计算所有实例变量的内存地址布局，就会导致实例变量的访问错位。

我觉得这正是直接避免产生这样的问题的直接解决方案，直接不让开发者对已经存在的类去添加实例变量Ivar的操作，这不就是直接去避免了上面的问题吗？

上面这部分是我看书的时候突然想到的一个问题，然而不是书上主要描述的问题。那么书上建议对属性的操作原则:

- (1) 不要在外部直接操作对象的实例变量
- (2) 而是应该通过对象的暴露的存取方法来间接操作实例变量

如上两点正是引出了`@property`的使用，告诉编译器自动做如下事情:

- (1) 属性最终对应的实例变量 >>> `_name`
- (2) 实例变量的读取方法 >>> `name`实现
- (3) 实例变量的修改方法 >>> `setName:`实现
- (4) `NSString  *str = 对象.name;` 自动调用`name`方法实现
- (5) `对象.name = @"haha"` 自动调用`setName:`方法实现
- (6) 对于`对象.name = @"haha"`调用setter时
	- 遵守属性声明时给出的`对象内存管理策略`
	- 修改值之后，自动发出属性修改KVO通知
	- 如果通过`_name = @"haha"` 则不会触发上面两件事



## `-[NSObject class]`、`+[NSObject class]`、`objc_getClass(<#const char *name#>)`的区别

### `-[NSObject class]`源码实现

```c
- (Class) class
{
  return object_getClass(self);
}
```

### `+[NSObject class]`源码实现

```c
+ (Class) class
{
  return self;
}
```

### `objc_getClass(<#const char *name#>)`源码实现

```c
Class object_getClass(id obj)
{
    if (obj) return obj->getIsa();//读取的是isa指针，所指向的objc_class实例
    else return Nil;
}
```

## objc对象的内存管理修饰符

对应关系如下

| 属性的内存管理修饰符 | 对象所有权修饰符 | 
| :-------------: |:-------------:| 
| assign | `__unsafe_unretained` | 
| copy | `__strong`（首先是拷贝原始对象得到一个新的对象，然后再强引用新的对象） | 
| retain | `__strong` | 
| strong | `__strong` | 
| unsafe_unretained | `__unsafe_unretained` | 
| weak | `__weak` |

### 最基础的对象内存管理修饰

- `__weak`

不会也不能持有指向的对象，即不会让指向的对象的retainCount++
	
-  `__strong`

- (1) 释放已有的老对象，`[老对象 release]`，老对象retainCount--
- (2) 持有传入的新对象，`[新对象 retain]`，新对象retainCount++

- `__unsafe_unretaind`

直接使用地址去操作对象，不进行任何的内存管理操作。被修饰的指针变量，在对象被废弃掉时，不会被设置为nil。仍然强制使用`__unsafe_unretained`修饰的指针去操作废弃掉的对象，就会导致程序crash崩溃，报错 `EXC_BAD_ACCESS....异常`.

- `__autoreleasing`

就相当于如下代码，将对象注册到自动释放池。

```objc
NSAutoreleasePool * pool = [[NSAutoreleasePool alloc] init];
Dog *dog =[Dog new];
[dog release];//将Dog对象加入到最近的pool对象中
[pool release];
```

### 对于属性提供的额外几种
	
- copy 

其实和strong/retain很相似的，`只是多一步拷贝`的操作，对于copy修饰的属性setter方法实际上做了如下三件事:

```
- (1) id newObj = [oldObj copy];
- (2) [oldObj release];
- (3) [newObj retain];
```

- assign:
	- 一般使用一些基本数据类型的属性变量（int、float、bool...）
	- 类似weak，但是区别是，在指向的对象被废弃掉时，指针变量值不会自动赋值为nil
	- 而weak修饰的指针变量会自动赋值nil



## 对象内部进行属性变读取时，尽量使用 `_varibale` 直接操作实例变量

对于实例变量访问方式有如下两种：

- 通过属性变量 `_variable` 来读写
- 通过 `self.variable` 来读写


### 通过 `对象.属性` 进行读取
	
- 可以`懒加载`的属性
- **调用setter设置新值时，会触发KVO通知**
- 设置新值时，会根据属性定义的`内存管理`对应的修饰规则
	- assign、copy、weak、strong、`unsafe_unretained` ...
- 还可以重写 setter与getter 方法，完成断点调试

### 通过 `_属性名` 进行读取

- `绕过`了属性定义的`内存管理`修饰
- `绕过`了属性定义的`读写权限`修饰（可以使用KVC操作私有实例变量）
- 无法触发KVO通知

### 合理组合上面两种形式

- (1) 对象内部，大多数情况下都应该如下
	- `读`实例变量 >>> 使用 `_variable` 来读实例变量，避免每次进入消息发送进入class结构体中查询SEL对应的IMP
	- `写`实例变量 >>> 使用 `self.属性名 = 新值` 来写数据，按照属性定义的内存管理修饰，以及提供KVO通知

- (2) 对象内部，`initXxx`函数与`dealloc`函数中，总是应该是通过`_variable`进行`读和写`
	- 因为`子类`可能重写属性的setter与getter方法实现
	- 那么调用子类的setter时，就会导致父类方法中的某一些实例变量未能初始化，导致程序崩溃

- (3)对于使用`懒加载`的属性，总应该是使用 `self.属性名`来进行`读`


## 单例模板

```c
@implementation Tool

+ (instancetype)sharedInstance {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _instance = [[self alloc] init];
    });
    return _instance;
}

+ (instancetype)allocWithZone:(struct _NSZone *)zone {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _instance = [super allocWithZone:zone];
    });
    return _instance;
}

- (id)copyWithZone:(NSZone *)zone {
    return _instance;
}

@end
```

## Cache 内存缓存多线程环境的代码模板

```objc
@interface ClassMapper : NSObject
@end

@implementation ClassMapper
+ (instancetype)mapperWithClass:(Class)cls {
    if (Nil == cls) {return nil;}
    
    /**
     *  1. 单例模板控制缓存正确初始化、信号值为1的信号量初始化
     */
    static CFMutableDictionaryRef       _cache = NULL;
    static dispatch_semaphore_t         _semephore = NULL;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _cache = CFDictionaryCreateMutable(kCFAllocatorDefault, 32, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
        _semephore = dispatch_semaphore_create(1);
    });
    
    /**
     *  2. 先查询缓存
     */
    const void *clsName =  (__bridge const void *)(NSStringFromClass(cls));
    dispatch_semaphore_wait(_semephore, DISPATCH_TIME_FOREVER);
    ClassMapper *clsMapper = CFDictionaryGetValue(_cache, clsName);
    dispatch_semaphore_signal(_semephore);
    
    /**
     *  3. 如果有缓存就直接返回，如果没有缓存则创建新的对象并完成缓存
     */
    if (!clsMapper) {
        clsMapper = [ClassMapper new];
        
        dispatch_semaphore_wait(_semephore, DISPATCH_TIME_FOREVER);
        CFDictionarySetValue(_cache, clsName, (__bridge const void *)(clsMapper));
        dispatch_semaphore_signal(_semephore);

    }

    return clsMapper;;
}
@end
```


## Category不会`覆盖`原始方法实现，以及多个Category重写相同的方法实现的调用顺序

### 主要涉及的问题

```
1. Category是否会覆盖原始Class中的Method？
2. 多个Category添加相同的Method，调用顺序是什么？
```

### 一、Category中重写原始类中已经存在的方法实现时

- (1) Category中重写的Method，肯定会排在原始类的Method的`最前面`

- (2) 每当从编译路径中读取到Category重写的Method，就会将这个重写的Method采用`头插法`插入到原始类的`method_list`的`第一个`位置

也就是说，越在后面编译的Category中重写的Method，却会出现在`method_list`的第一个位置
	
### 二、多个Category中，都重写了相同的方法实现时

会按照Category在编译路径中的顺序，将Method依次`头插`到原始类的`method_list`单链表中。

所以，也就是Category出现的越晚，重写的Method就会出现在第一个位置。

具体测试，搜索`Category覆盖原始类中的方法实现存在的问题.md`。

## 触发CPU与GPU的离屏渲染的场景

### CPU触发离屏渲染

- (1) 使用`CoreGraphics`库函数进行绘制图像

- (2) 重写`-[UIView drawRect]`方法实现中写的任何绘制代码
    - 甚至是`空方法实现`也会触发

### GPU触发离屏渲染

- (1) CALayer对象设置 shouldRasterize（光栅化）

- (2) CALayer对象设置 masks（遮罩）

- (3) CALayer对象设置 shadows（阴影）

- (4) CALayer对象设置 group opacity（不透明）

- (5) 所有`文字`的绘制（UILabel、UITextView...），包括`CoreText`绘制文字、`TextKit`绘制文字


尽量避免GPU离屏渲染，但是为了能够异步进行绘制，也有可能操作CPU离屏渲染。

但是最后这两种离屏渲染，都不做。

## FMDatabaseQueue解决`dispatch_sync`可能导致多线程死锁

主要是如下两个相关函数的使用:

```c
dispatch_queue_set_specific(dispatch_queue_t queue, 
							const void *key,
							void *context, dispatch_function_t destructor);
```	

```c
void * dispatch_get_specific(const void *key);
```

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

通过取出当前线程中key对应的value，是否是之前指定的value:


- (1) 如果是，说明是`同一个`线程，则不能执行`dispatch_sync()`
- (2) 如果不是，说明是`不同`的线程，可以执行

下面是一个基于c语言的context的例子：

```c
//1.
typedef struct NetworkMetaDataDispatchQueueContext {
    char reversed;
}NetworkMetaDataDispatchQueueContext;

//2. 定义一个全局存在的结构体实例，用来判断是否一致
static NetworkMetaDataDispatchQueueContext _context;

//3. 唯一key
static const void * const kNetworkCacheMetaDispatchQueueSpecificKey = &kNetworkCacheMetaDispatchQueueSpecificKey;

//4. 给queue绑定全局静态实例context
dispatch_queue_t queue = queue = dispatch_queue_create("queue.xzhnrtwork.incomplet.data.write", attr);
dispatch_queue_set_specific(queue, kNetworkCacheMetaDispatchQueueSpecificKey, (void*)(&_context), NULL);

//5. 取出queue绑定的context，判断是否全全局context一致，避免处于同一线程，使用dispatch_sync()导致线程发生死锁
if ((void*)(&_context) == dispatch_get_specific(kNetworkCacheMetaDispatchQueueSpecificKey)) {
    return nil;
}
```



## 借助`NSProxy`实现动态代理，以及`消息转发阶段1`机制来模拟`多继承`

### 有三个抽象接口

```objc
@protocol SayChinese <NSObject>
- (void)sayChinese;
@end
```

```objc
@protocol SayEnglish <NSObject>
- (void)sayEnglish;
@end
```

```objc
@protocol SayFranch <NSObject>
- (void)sayFranch;
@end
```

### 内部拥有三个具体实现类，但是我希望不对外暴露实现，只是我内部知道具体实现

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

注意，这些实现类是私有的，不向外暴露的。

### 向外暴露`一个类`，来操作上面三个类所有方法实现，即多继承的效果

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

## type encodings 数据类型的系统存放的字符值

### `objc_property_attribute_t.name[0]` 的type encodings

```c
 static const char XZHPropertyAttributeBegin = 'T';//T@\"NSString\",C,N,V_name，作为属性编码的开始符，不作为属性权限修饰符
static const char XZHPropertyAttributeIvarName = 'V';//V_name，表示Ivar的名字，不作为属性权限修饰符
static const char XZHPropertyAttributeCopy = 'C';
static const char XZHPropertyAttributeCustomGetter = 'G';
static const char XZHPropertyAttributeCustomSetter = 'S';
static const char XZHPropertyAttributeDynamic = 'D';
static const char XZHPropertyAttributeEligibleForGarbageCollection = 'P';
static const char XZHPropertyAttributeNonAtomic = 'N';
static const char XZHPropertyAttributeOldStyle = 't';
static const char XZHPropertyAttributeReadOnly = 'R';
static const char XZHPropertyAttributeRetain = '&';
static const char XZHPropertyAttributeWeak = 'W';
```

### `objc_ivar` 的type encodings

```c
static const char XZHIvarTypeUnKnown = _C_UNDEF;//?
static const char XZHIvarTypeObject = _C_ID;//@
static const char XZHIvarTypeClass = _C_CLASS;//#
static const char XZHIvarTypeSEL = _C_SEL;//:
static const char XZHIvarTypeChar = _C_CHR;//c
static const char XZHIvarTypeUnsignedChar = _C_UCHR;//C
static const char XZHIvarTypeInt = _C_INT;//i
static const char XZHIvarTypeUnsignedInt = _C_UINT;//I
static const char XZHIvarTypeShort = _C_SHT;//s
static const char XZHIvarTypeUnsignedShort = _C_USHT;//S
static const char XZHIvarTypeLong = _C_LNG;//l
static const char XZHIvarTypeUnsignedLong = _C_ULNG;//L
static const char XZHIvarTypeLongLong = 'q';
static const char XZHIvarTypeUnsignedLongLong = 'Q';
static const char XZHIvarTypeFloat = _C_FLT;//f
static const char XZHIvarTypeDouble = _C_DBL;//d
static const char XZHIvarTypeLongDouble = 'D';
static const char XZHIvarTypeBOOL = 'B';
static const char XZHIvarTypeVoid = _C_VOID;//v
static const char XZHIvarTypeCPointer = _C_PTR;//^
static const char XZHIvarTypeCString = _C_CHARPTR;//*
static const char XZHIvarTypeCArray = _C_ARY_B;//[
static const char XZHIvarTypeCArrayEnd = _C_ARY_E;//]
static const char XZHIvarTypeCStruct = _C_STRUCT_B;//{
static const char XZHIvarTypeCStructEnd = _C_STRUCT_E;//}
static const char XZHIvarTypeCUnion = _C_UNION_B;//(
static const char XZHIvarTypeCUnionEnd = _C_UNION_E;//)
static const char XZHIvarTypeCBitFields = _C_BFLD;//b
```

### `runtime.h`中定义出的type encoding字符

```c
#define _C_ID       '@'
#define _C_CLASS    '#'
#define _C_SEL      ':'
#define _C_CHR      'c'
#define _C_UCHR     'C'
#define _C_SHT      's'
#define _C_USHT     'S'
#define _C_INT      'i'
#define _C_UINT     'I'
#define _C_LNG      'l'
#define _C_ULNG     'L'
#define _C_LNG_LNG  'q'
#define _C_ULNG_LNG 'Q'
#define _C_FLT      'f'
#define _C_DBL      'd'
#define _C_BFLD     'b'
#define _C_BOOL     'B'
#define _C_VOID     'v'
#define _C_UNDEF    '?'
#define _C_PTR      '^'
#define _C_CHARPTR  '*'
#define _C_ATOM     '%'
#define _C_ARY_B    '['
#define _C_ARY_E    ']'
#define _C_UNION_B  '('
#define _C_UNION_E  ')'
#define _C_STRUCT_B '{'
#define _C_STRUCT_E '}'
#define _C_VECTOR   '!'
#define _C_CONST    'r'
```

注意，`'A'`是一个字符，`"A"`才是一个字符串。

## objc对象的内存管理


### 错误纠正: 对象的释放与废弃，是两个`不同的阶段`。`释放`持有是`同步`完成，内存`废弃`是`异步`完成。

- (1) 当对象的`持有数==0（retainCount==0）`时
- (2) 表示这个对象，`即将`会被废弃，但是此时`并未废弃`
	- 只是标示这个对象，将要被废弃
- (3) 当某个空闲时间，系统彻底将对象所在内存数据全部擦除，然后与系统未分配内存进行合并，以待后续继续使用

所以说，执行了dealloc，并不是说对象所在内存就被废弃了。


```objc
- (void)testMRC {

    _mrc = [[MRCTest alloc] init];
    NSLog(@"[_mrc retainCount] = %lu", [_mrc retainCount]);
    
    MRCTest *tmp1 = [_mrc retain];
    NSLog(@"[_mrc retainCount] = %lu", [_mrc retainCount]);
    
    [_mrc release];
    NSLog(@"[_mrc retainCount] = %lu", [_mrc retainCount]);
    
    [tmp1 release];
    NSLog(@"[_mrc retainCount] = %lu", [_mrc retainCount]);
    
    //【重要】尝试多次输出retainCount
    for (NSInteger i = 0; i < 10; i++) {
        NSLog(@"[_mrc retainCount] = %lu", [_mrc retainCount]);//【重要】循环执行几次之后，崩溃到此行
    }
}
```

运行之后，结果崩溃到for循环中的第二次或第三次循环，`程序崩溃`报错如下:

```
thread 1:EXC_BAD_ACCESS .... 
```

释放掉对象之后，指向该对象的指针，仍然会保留在局部方法块的所在栈中，仍然是可以在短暂的时间内继续通过指针访问到对象。但是超过一定时间后，对象才会被彻底废弃掉，这个时候如果还去使用这个指针就会造成程序崩溃。

那这样是说最终对象的内存废弃过程，是一个`异步`执行的吗？或者说有一定的`延迟时间`吗？

是延迟的，因为最终对象内存会被擦除掉，并与系统内存合并到一起，所以这个过程确实是一个异步的。

### 小结使用 `__strong` 修饰对象的指针变量:

- (1) 不必再次写retain、release的消息代码
- (2) 完美的满足了引用计数器方式管理内存
	- 自己生成的对象，自己持有
	- 非自己生成的对象，自己也能持有
	- 不再需要自己持有的对象时进行释放
	- 非自己持有的对象无法释放


### 使用 `__strong、__weak、__autoreleasing` 这三种修饰指向的对象指针，会由ARC系统自动完成:

- 自动初始化为nil
- 当所指向的对象被废弃掉时，同样也会被设置为nil

相反，`__unsafe_unretained`在对象被废弃掉时，不会被设置为nil。

### `__weak`为了解决对象之间的循环引用而生

图示

![Markdown](http://i1.piimg.com/1949/88a1d99f16424bcf.jpg)


###  `__unsafe_unretained`

正如其名包含两个部分:

- (1) unsafe 不安全，这点是与`weak`不同点
- (2) unretained 不强引用持有，这是与`weak`的相同点

### `__autoreleasing`

分析下`+[NSMutableArray array]`返回值对象处理:

```objc
+ (id)array {

	//1. 生成一个数组对象
	id obj = objc_msgSend(NSMutableArray, @selector(alloc));
	
	//2. 执行对象的初始化init方法
	obj = objc_msgSend(obj, @selector(init));
	
	//3. 返回一个autorelease的返回值
	return objc_autoreleaseReturnValue(obj);
}
```

- (1) 默认`__strong`修饰的指针变量，持有住生成的NSMutableArray对象
- (2) 局部指针变量`obj`超出函数域，被自动设置为nil
- (3) 因为方法名并非上面的四种之一，所以返回值NSMutableArray对象，会被自动注册到autoreleasePool

在使用`__weak`修饰的指针变量时，会由编译器自动插入，将指针变量指向的对象注册到autoreleasPool的代码。

那么为什么要这样的了？因为`__weak`变量:

- (1) 支持有对象的一个`弱引用`
- (2) 不能保证访问该对象的整个过程中，对象一定不会被废弃

所以，通过将弱引用的对象，注册到autoreleasePool中，从而保证整个操作过程（autoreleasePool结束之前），弱引用对象都不会被废弃。


### 属性修饰符与对象所有权修饰符的关系

| 属性声明时的修饰符 | 对象所有权修饰符 | 
| :-------------: |:-------------:| 
| assign | `__unsafe_unretained` | 
| copy | `__strong`（首先是拷贝原始对象得到一个新的对象，然后再强引用新的对象，释放老的对象） | 
| retain | `__strong` | 
| strong | `__strong` | 
| `unsafe_unretained` | `__unsafe_unretained` | 
| weak | `__weak` | 

### 使用`__strong`强持有方法的返回值对象时，与`__weak`是有区别的

有一对重要的函数，用于返回值对象的内存管理优化

- (1) `objc_retainAutoreleasedReturnValue()`

- (2) `objc_autoreleaseReturnValue()`

对于`+[NSMutableArray array]`的实现代码

```objc
+ (id)array {

	//1. 生成一个数组对象
	id obj = objc_msgSend(NSMutableArray, @selector(alloc));
	
	//2. 执行对象的初始化init方法
	obj = objc_msgSend(obj, @selector(init));
	
	//3. 返回一个autorelease的返回值
	return objc_autoreleaseReturnValue(obj);
}
```

对于使用`__strong`持有该方法的返回值对象

```objc
id __strong obj = [NSMutableArray array];
```

会被编译成如下代码:

```c
//1. 
id obj = objc_msgSend(NSMutableArray, @selector(array));//取的别人生成的对象

//2. 【重点】 
objc_retainAutoreleasedReturnValue(obj);

//3.
objc_release(obj);
```

所以，`objc_autoreleaseReturnValue(obj)`与`objc_autoreleaseReturnValue(obj)`之间是有关系的，决定到底是如何处理方法的返回值

- (1) `objc_retainAutoreleasedReturnValue()` 与 `objc_autoreleaseReturnValue()` 是 `成对` 出现的

- (2) `objc_retainAutoreleasedReturnValue()`: 出现在调用方，获取返回值的代码中
	
- (3) `objc_autoreleaseReturnValue()`: 出现在被调函数代码中

- (4) `objc_retainAutoreleasedReturnValue()`的作用:
	- 首先是去持有一个方法返回的对象，但是分为`两种`持有方式
	- 持有方式一【weak】、将返回的对象注册到一个`autoreleasePool`去持有
	- 持有方式二【retain、strong】、直接通过`传递`的方式获取对象（不会进行释放池及注册）

- (5) `objc_autoreleaseReturnValue()`的作用:
	- 首先监测`函数调用方`，是否在取得了函数放回对象之后，紧接着使用了`objc_retainAutoreleasedReturnValue()`
		- **看获取返回值对象，使用的是weak还是retain/strong**
	- 如果使用了`objc_retainAutoreleasedReturnValue()`:
		- 使用`持有方式二`来直接传递返回对象
		
所以，最好不要使用`__weak`的方式持有方法的返回值，会有一个注册到autoreleasePool的多余过程。

### 由此可知，当使用`__weak`修饰的指针变量指向的对象时，会做很多的额外的处理代码，所以会消耗一定的cpu时间。所以，最好只在避免循环引用的时候，去使用`__weak`。

- (1) 情况一、使用`__weak`修饰的指针变量指向，刚刚创建的对象
	- 会将对象自动注册到autoreleasePool

- (2) 情况二、当被指向的对象被废弃时
	- (2.1) 从weak表中，获取`被废弃对象`对应的`__weak修饰的 指针变量`
    - (2.2) 将找到的`weak指针变量`赋值为nil
    - (2.3) 表一删除、从`weak表`中`删除`废弃对象地址对应的记录项
    - (2.4) 表二删除、从`引用计数器表`中`删除`废弃对象地址对应的记录项


## objc对象的等同性判断写法模板

重写NSObject的实例方法

- (1) isEqual:
- (2) hash


```objc
@implementation Person

- (instancetype)initWithPid:(NSString *)pid Age:(NSInteger)age Name:(NSString *)name {
    self = [super init];
    if (self) {
        _pid = [pid copy];
        _name = [name copy];
        _age = age;
    }
    return self;
}

- (BOOL)isEqualToPerson:(Person *)person {
    
    //1. 如果这是两个相同对象，直接就等价了
    if (self == person) return YES;
    
    //2. 依次比较对象内部的子数据项
    if (_age != person.age) return NO;
    if (![_name isEqualToString:person.name]) return NO;
    if (![_pid isEqualToString:person.pid]) return NO;
    
    //3. 默认返回YES，表示对象等价
    return YES;
}

- (BOOL)isEqual:(id)object {
    
    //1. 比较对象的类型是否一致
    if ([self class] == [object class]) {
        //2.2 其他对象指针比较、对象子数据项比较交给isEqualToPerson:
        return [self isEqualToPerson:(Person *)object];
    } else {
        //2.1 不是同一个类型的对象
        return [super isEqual:object];
    }
}

- (NSUInteger)hash {
    NSInteger ageHash = _age;
    NSUInteger nameHash = [_name hash];
    NSUInteger pidHash = [_pid hash];
    return ageHash ^ nameHash ^ pidHash;
}

@end
```

## 最好不要重写`drawRect: 或 drawInContext:`，使用`CoreGraphics`完成绘制图像。而是使用`专用图层`来专门完成绘制，还可以直接将`专用图层`作为View的`宿主图层`。为什么?

### 首先清楚，CPU与GPU的强项与弱势：

- (1) CPU、对数据的计算处理相当快，但是对于图像的渲染很差
- (2) GPU、有很多核心来同时做图像的渲染，所以很快。但是对于数据的计算处理，是很慢的


所以，一定要充分利用GPU与CPU的强项：

```
CPU >>> 大量进行数据计算，少进行图像渲染
GPU >>> 大量进行图像渲染，少进行数据计算
```

- (1) `OpenGL`绘制图像，会交给`GPU`完成渲染
- (2) `CoreGraphics`绘制图像，会交给`CPU`完成渲染

所以，最好让CPU只做一些`数据计算`，而CPU只会`图像渲染`，这样整体性能会提升很多。

### 为什么不要重写`drawRect: 或 drawInContext:`，使用`CoreGraphics`完成绘制图像？

- (1) 只要重写`drawRect: 或 drawInContext:`，就会给layer创建一个空的宿主图像，浪费内存

- (2) CoreGraphics的图像绘制，会触发`CPU的离屏渲染`，而CPU的`图像渲染`能力是很差的，会影响CPU的执行效率

- (3) `专用图层`把图像渲染代码使用OpenGL操作`GPU`来完成图像的渲染，内存优化

下面是使用专用图层来绘制自定义路径的代码模板，`代替`使用重写`drawRect:`

```objc
@implementation XingNengVC {
    UIView  *_bottomView;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    //1. 创建UIView容器
    _bottomView = [[UIView alloc] initWithFrame:CGRectMake(0, 0, 300, 200)];
    [self.view addSubview:_bottomView];
    
    //2. 创建具体绘制图像的专用图层
    CAShapeLayer *layer = [CAShapeLayer layer];
    layer.frame = _bottomView.bounds;
    
    //3. 设置要绘制的图像路径
    UIBezierPath *path = [[UIBezierPath alloc] init];
    [path moveToPoint:CGPointMake(175, 100)];
    [path addArcWithCenter:CGPointMake(150, 100) radius:25 startAngle:0 endAngle:2*M_PI clockwise:YES];
    [path moveToPoint:CGPointMake(150, 125)];
    [path addLineToPoint:CGPointMake(150, 175)];
    [path addLineToPoint:CGPointMake(125, 225)];
    [path moveToPoint:CGPointMake(150, 175)];
    [path addLineToPoint:CGPointMake(175, 225)];
    [path moveToPoint:CGPointMake(100, 150)];
    [path addLineToPoint:CGPointMake(200, 150)];
    
    //4. 将要绘制的路径设置给layer
    layer.path = path.CGPath;
    
    //3.
    [_bottomView.layer addSublayer:layer];
}

@end
```

## `BAD_ACCESS`异常崩溃模板代码

```objc
NSObject *obj1 = [NSObject new];

//1.
__unsafe_unretained NSObject *obj2 = obj1;

//2.
obj1 = nil;

//3.
[obj2 description];
```

## 几种获取Class的系统方法实现源码

### `-[NSObject class]`

```objc
- (Class) class
{
  return object_getClass(self);
}
```

### `+[NSObject class]`

```objc
+ (Class) class
{
  return self;
}
```

### `object_getClass(id obj)`

```c
Class object_getClass(id obj)
{
    if (obj) return obj->getIsa();
    else return Nil;
}
```

## NSAarray与NSMuatbleArray在alloc、init时的类型是不同的

> NSArray、NSMutableArray中的类簇应用.md

```objc
- (void)testArrayAllocInit {
    
    id obj1 = [NSArray alloc];
    id obj2 = [NSMutableArray alloc];
    
    id obj3 = [obj1 init];
    //id obj4 = [obj1 initWithCapacity:16];//崩溃，因为obj1不是 __NSArrayM
    id obj5 = [obj1 initWithObjects:@"1", nil];
    
    id obj6 = [obj2 init];
    id obj7 = [obj2 initWithCapacity:16];
    id obj8 = [obj2 initWithObjects:@"1", nil];
    
    NSLog(@"");
}
```

输出如下

```
(__NSPlaceholderArray *) obj1 = 0x00007ff79bc04b20
(__NSPlaceholderArray *) obj2 = 0x00007ff79bc06d00
(__NSArray0 *) obj3 = 0x00007ff79bc02eb0
(__NSArrayI *) obj5 = 0x00007ff79be0fb60 @"1 object"
(__NSArrayM *) obj6 = 0x00007ff79be1e350 @"0 objects"
(__NSArrayM *) obj7 = 0x00007ff79be1e380 @"0 objects"
(__NSArrayM *) obj8 = 0x00007ff79be22db0 @"1 object"
```

可以得到：

- (1) 最终使用的并不是NSArray、NSMuatbleArray，而是内部的一些带有`__`的私有类

- (2) `[NSArray alloc]`与`[NSMuatbleArray alloc]`得到都是`__NSPlaceholderArray`

- (3) `[[NSArray alloc] init]`得到的是`__NSArray0`

- (4) `[[NSArray alloc] initWithObjects:@"1", nil]`得到的是`__NSArrayI`

- (5) `NSMuatbleArray`任何的init函数，得到的都是`__NSArrayM`

可以肯定的是，这些内部的私有类，都是NSArray或NSMuatbleArray的`子类`。

```objc
- (void)testArrayAllocInit2 {
    id obj1 = [NSArray alloc];
    id obj2 = [NSMutableArray alloc];
    id obj3 = @[@"1"];
    id obj4 = [obj3 mutableCopy];
    id obj5 = @[];

    Class cls1 = class_getSuperclass([obj1 class]);
    Class cls2 = class_getSuperclass([obj2 class]);
    Class cls3 = class_getSuperclass([obj3 class]);
    Class cls4 = class_getSuperclass([obj4 class]);
    Class cls5 = class_getSuperclass([obj5 class]);
    
    NSLog(@"");
}
```

输出如下

```
(__NSPlaceholderArray *) obj1 = 0x00007fcf8b506cb0
(__NSPlaceholderArray *) obj2 = 0x00007fcf8b5030f0
(__NSArrayI *) obj3 = 0x00007fcf8b41a940 @"1 object"
(__NSArrayM *) obj4 = 0x00007fcf8b41a980 @"1 object"
(__NSArray0 *) obj5 = 0x00007fcf8b5045e0

(Class) cls1 = NSMutableArray
(Class) cls2 = NSMutableArray
(Class) cls3 = NSArray
(Class) cls4 = NSMutableArray
(Class) cls5 = NSArray
```

根据输出可以得到这几种Array的继承结构：

```
- NSArray
	- NSMutableArray
		- __NSPlaceholderArray
		- __NSArrayM
	- __NSArrayI
	- __NSArray0
```

所以类簇最终实现方式以`继承`的方式可能是比较好的一种设计方法。

###  那`-[__NSPlaceholderArray init]`是如何判断到底是生成NSArray还是NSMutableArray的子类对象了?

看了下`《iOS高级内存管理编程指南》`中关于类对象类簇的部分内容，得到如下的伪代码实现:

- (1) `__NSPlacehodlerArray`全局使用一个`单例对象 A`，来记录`[NSArray alloc]`的操作

```c
static __NSPlacehodlerArray *GetPlaceholderForNSArray() {
    static __NSPlacehodlerArray *instanceForNSArray;
    if (!instanceForNSArray) {
        instanceForNSArray = [__NSPlacehodlerArray alloc];
    }
    return instanceForNSArray;
}
```

- (2) `__NSPlacehodlerArray`全局使用一个`单例对象 B`，来记录`[NSMutableArray alloc]`的操作

```c
static __NSPlacehodlerArray *GetPlaceholderForNSMutableArray() {
    static __NSPlacehodlerArray *instanceForNSMutableArray;
    if (!instanceForNSMutableArray) {
        instanceForNSMutableArray = [__NSPlacehodlerArray alloc];
    }
    return instanceForNSMutableArray;
}
```

- (3) NSArray的alloc方法实现

```objc
+ (id)alloc
{
    if (self == [NSArray class]) {
        return GetPlaceholderForNSArray();//获取单例A
    }
}
```

- (4) NSMutableArray的alloc方法实现

```objc
+ (id)alloc
{
    if (self == [NSMutableArray class]) {
        return GetPlaceholderForNSMutableArray();//获取单例B
    }
}
```

- (5) `-[__NSPlacehodlerArray init]`方法实现

```objc
- (id)init
{	
    if (self == GetPlaceholderForNSArray()) {//单例A
    	/**
    	 *	重新创建一个__NSArrayI对象返回
    	 */
        self = [[__NSArrayI alloc] init];
    }
    else if (self == GetPlaceholderForNSMutableArray()) {//单例B
    	/**
    	 *	重新创建一个__NSArrayM对象返回
    	 */
        self = [[__NSArrayM alloc] init];
    }
    return self;
}
```


### 新增内部私有实现类的规则

- (1) 新增的实现类，必须继承自抽象父类
- (2) 子类必须实现所有必须的的方法，其他可以新增自己的方法实现

### 对于类簇类进行比较注意

- (1) 不能使用`[obj1 class] == [obj2 class]`，因为永远都不可能相等
- (2) 必须使用`[obj1 isKindOf:[obj2 class]]`，沿着父类一直比较

```objc
- (void)compare {
    id array = @[@"111", @"222", @"3333", @"4444"];
    
    // 错误比较
    if ([array class] == [NSArray class]) {
       NSLog(@"属于数组");// 其实永远都不会执行
    }
    
    // 正确比较
    if ([array isKindOfClass:[NSArray class]]) {
        NSLog(@"属于数组");
    }
}
```

但是如果是针对我们自己编写的一些基本类，并非是类簇的情况下，可以采用上述两种方法进行比较类型，可以使用如下这几种都可以

```objc
- (void)compare {

    Person *person = [Person new];
    
    /**
     *  因为一个NSObejct类在运行时对应的objc_class结构体实例是一个 全局存在的单例
     */
    if ([person class] == [Person class]) {
        NSLog(@"属于Person类");
    }
    
    if ([person isKindOfClass:[Person class]]) {
        NSLog(@"属于Person类");
    }
    
    if (object_getClass(person) == [Person class]) {
        NSLog(@"属于Person类");
    }
    
    if (object_getClass(person) == objc_getClass("Person")) {
        NSLog(@"属于Person类");
    }
}
```


## 使用`类簇`在不同的iOS系统版本下，对一些不同系统版本的系统函数进行适配

### 暴露给客户的统一入口类.h（类簇类）

```objc
#import <Foundation/Foundation.h>

@interface MyTool : NSObject

// 客户只需要关心调用这个方法即可
- (void)doWork;

@end
```

### 入口类.m中，管理各种iOS系统版本下的实现类，并判断当前系统版本，返回对应的实现类对象


```objc
#import "MyTool.h"

// 其他私有的不同系统版本的实现子类
#import "MyTool_iOS6.h"
#import "MyTool_iOS7.h"
#import "MyTool_iOS8.h"

@implementation MyTool

// 判断当前系统版本，选择对应系统版本下的实现类
+ (instancetype)alloc {
    if (self == [MyTool class]) {
        if (floor(NSFoundationVersionNumber) <= NSFoundationVersionNumber_iOS_6_1) {
            //iOS6
            return [MyTool_iOS6 alloc];
        } else if (floor(NSFoundationVersionNumber) > NSFoundationVersionNumber_iOS_6_1 && floor(NSFoundationVersionNumber) < NSFoundationVersionNumber_iOS_8_0) {
            //iOS7
            return [MyTool_iOS7 alloc];
        } else if (floor(NSFoundationVersionNumber) > NSFoundationVersionNumber_iOS_7_1){
            //iOS8及以上
            return [MyTool_iOS8 alloc];
        }
    }
    return [super alloc];
}

- (void)doWork { /* 空实现，由具体子类去实现 */ }

@end
```

### 入口类内部依赖的不同系统下的实现类

- iOS6下的实现类

```objc
#import "MyTool.h"

@interface MyTool_iOS6 : MyTool

@end
```

```objc
#import "MyTool_iOS6.h"

@implementation MyTool_iOS6

- (void)doWork {
    NSLog(@"iOS6 work");
}

@end
```

- iOS7的实现类

```objc
#import "MyTool.h"

@interface MyTool_iOS7 : MyTool

@end
```

```objc
#import "MyTool_iOS7.h"

@implementation MyTool_iOS7

- (void)doWork {
    NSLog(@"iOS7 work");
}

@end
```

- iOS8及以上的实现类

```objc
#import "MyTool.h"

@interface MyTool_iOS8 : MyTool

@end
```

```objc
#import "MyTool_iOS8.h"

@implementation MyTool_iOS8

- (void)doWork {
    NSLog(@"iOS8 work");
}

@end
```

### 最后是客户只需要找到入口类MyTool即可，而不需要知道如上的具体版本下的某一个实现类.

```objc
- (void)test3 {
	
	//1. 
    MyTool *tool = [[MyTool alloc] init];
    
    //2. 
    [tool doWork];
}
```

### 图示小结

<img src="./类簇类应用1.png" alt="" title="" width="700"/>