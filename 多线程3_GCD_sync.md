## dispatch sync queue 不会使用queue对应的thread执行block，而是在当前执行sync操作的线程上执行block


### 在主线程上 dispatch sync concurrent queue

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

- (1) block全部都是在`主线程`上执行，而`并非`是在一个`子线程`执行
- (2) block是同步等待执行完毕，才会走后面的代码
- (3) block任务全部按照`顺序`一个一个接着完成

### 在子线程上 dispatch sync concurrent queue

```objc
@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	
	dispatch_async(dispatch_get_global_queue(0, 0), ^() {
    
	    NSLog(@"*******************开始，所在线程: %@*******************", [NSThread currentThread]);
	    
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
2016-12-22 00:24:27.723 Demo[6748:88097] *******************开始，所在线程: <NSThread: 0x7f9fd8e1a630>{number = 2, name = (null)}*******************
2016-12-22 00:24:27.724 Demo[6748:88097] Block任务1, 所在Thread: <NSThread: 0x7f9fd8e1a630>{number = 2, name = (null)}
2016-12-22 00:24:27.724 Demo[6748:88097] *******************任务1执行结束*******************
2016-12-22 00:24:27.724 Demo[6748:88097] Block任务2, 所在Thread: <NSThread: 0x7f9fd8e1a630>{number = 2, name = (null)}
2016-12-22 00:24:27.724 Demo[6748:88097] *******************任务2执行结束*******************
2016-12-22 00:24:27.724 Demo[6748:88097] Block任务3, 所在Thread: <NSThread: 0x7f9fd8e1a630>{number = 2, name = (null)}
2016-12-22 00:24:27.725 Demo[6748:88097] *******************任务3执行结束*******************
2016-12-22 00:24:27.725 Demo[6748:88097] Block任务4, 所在Thread: <NSThread: 0x7f9fd8e1a630>{number = 2, name = (null)}
2016-12-22 00:24:27.725 Demo[6748:88097] *******************任务4执行结束*******************
2016-12-22 00:24:27.725 Demo[6748:88097] Block任务5, 所在Thread: <NSThread: 0x7f9fd8e1a630>{number = 2, name = (null)}
2016-12-22 00:24:27.725 Demo[6748:88097] *******************任务5执行结束*******************
```


- (1) **block这次全部都是在进入的global queue中的thread上执行**，与之前唯一的区别
- (2) block是同步等待执行完毕，才会走后面的代码
- (3) block任务全部按照`顺序`一个一个接着完成

### 可以看到使用`dispatch_sync(queue, block);`时，不管传入的queue是 `串行队列` or `无序队列`:

- (1) `不会创建新的子线程`来执行block
	- 就是不会使用传入的queue所对应的线程，去执行传入的block
	- 而是直接在当前线程执行block

- (2) 只是将block临时放到这个queue的`末尾`进行排队，执行的时候再从这个队列取出来

- (3) block执行线程是在执行`dispatch_sync(queue, block);`操作所在的当前线程上

- (4) 必须走完`dispatch_sync(queue, block);` 中的block代码，才会走后面的代码，常常也可以用于多线程同步

如果只是一些内存数据的同步访问，可以使用`dispatch_sync(queue, block);`。但如果对于一些大文件的读写代码，还是要避免使用`dispatch_sync(queue, block);`，否则会造成线程卡顿的，最好还是使用`dispatch_async(queue, block);`异步的形式。


## `dispatch_sync(queue, block);`造成死锁的代码

### 造成死锁最简单的代码形式一

```objc
@implementation ViewController

- (void)testSyncDeadLock1 {
    NSLog(@"1 所在Thread: %@", [NSThread currentThread]);
    dispatch_sync(dispatch_get_main_queue(), ^{
        NSLog(@"2 所在Thread: %@", [NSThread currentThread]);
    });
    NSLog(@"3 所在Thread: %@", [NSThread currentThread]);
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	[self testSyncDeadLock1];
}

@end
```

运行结果

```
2016-12-22 00:30:04.079 Demo[7061:93049] 1 所在Thread: <NSThread: 0x7fadd9500e70>{number = 1, name = main}
```

可以看到，直走了1，后面的2，3都没走，这就是线程死锁。


### 造成死锁最简单的代码形式二


```objc
@implementation ViewController

- (void)testSyncDeadLock2 {
    
    //【串行队列】时需要注意
    static dispatch_queue_t serial_queue = NULL;
    serial_queue =  dispatch_queue_create("用户队列", NULL);
    
    NSLog(@"1 所在Thread: %@", [NSThread currentThread]);

    dispatch_async(serial_queue, ^{
        NSLog(@"2 所在Thread: %@", [NSThread currentThread]);
        
        dispatch_sync(serial_queue, ^{
            NSLog(@"3 所在Thread: %@", [NSThread currentThread]);
        });
        
        NSLog(@"4 所在Thread: %@", [NSThread currentThread]);
    });
    
    NSLog(@"5 所在Thread: %@", [NSThread currentThread]);
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	[self testSyncDeadLock2];
}

@end
```

输出结果

```
2016-12-22 00:34:25.577 Demo[7320:96967] 1 所在Thread: <NSThread: 0x7fcf3bf07e30>{number = 1, name = main}
2016-12-22 00:34:25.578 Demo[7320:96967] 5 所在Thread: <NSThread: 0x7fcf3bf07e30>{number = 1, name = main}
2016-12-22 00:34:25.578 Demo[7320:97052] 2 所在Thread: <NSThread: 0x7fcf3bea8ab0>{number = 2, name = (null)}
```

同样，3和4没有输出，说明也发生了线程死锁。

### 形式一与形式二的相同结构如下

```objc
{// 当前处于线程A
	dispatch_sync(线程A对应的queue, ^{
		.....
	});
}
```

这个是使用`dispatch_sync()`时出现线程死锁的模板。

### 但是如果是`dispatch_async()`是没有任何问题的，因为`async`不存在等待block执行完毕的问题，都是异步执行的

```objc
- (void)testSyncDeadLock1 {
    NSLog(@"1 所在Thread: %@", [NSThread currentThread]);
    dispatch_async(dispatch_get_main_queue(), ^{
        NSLog(@"2 所在Thread: %@", [NSThread currentThread]);
    });
    NSLog(@"3 所在Thread: %@", [NSThread currentThread]);
}
```

输出结果

```
2016-12-22 00:39:40.256 Demo[7600:101629] 1 所在Thread: <NSThread: 0x7f9de8d02260>{number = 1, name = main}
2016-12-22 00:39:40.256 Demo[7600:101629] 3 所在Thread: <NSThread: 0x7f9de8d02260>{number = 1, name = main}
2016-12-22 00:39:40.257 Demo[7600:101629] 2 所在Thread: <NSThread: 0x7f9de8d02260>{number = 1, name = main}
```

```objc
- (void)testSyncDeadLock2 {
    
    //【串行队列】时需要注意
    static dispatch_queue_t serial_queue = NULL;
    serial_queue =  dispatch_queue_create("用户队列", NULL);
    
    NSLog(@"1 所在Thread: %@", [NSThread currentThread]);

    dispatch_async(serial_queue, ^{
        NSLog(@"2 所在Thread: %@", [NSThread currentThread]);
        
        dispatch_async(serial_queue, ^{
            NSLog(@"3 所在Thread: %@", [NSThread currentThread]);
        });
        
        NSLog(@"4 所在Thread: %@", [NSThread currentThread]);
    });
    
    NSLog(@"5 所在Thread: %@", [NSThread currentThread]);
}
```

输出结果

```
2016-12-22 00:40:36.884 Demo[7670:102934] 1 所在Thread: <NSThread: 0x7fe3c17000f0>{number = 1, name = main}
2016-12-22 00:40:36.885 Demo[7670:102934] 5 所在Thread: <NSThread: 0x7fe3c17000f0>{number = 1, name = main}
2016-12-22 00:40:36.885 Demo[7670:103019] 2 所在Thread: <NSThread: 0x7fe3c1614350>{number = 2, name = (null)}
2016-12-22 00:40:36.885 Demo[7670:103019] 4 所在Thread: <NSThread: 0x7fe3c1614350>{number = 2, name = (null)}
2016-12-22 00:40:36.886 Demo[7670:103019] 3 所在Thread: <NSThread: 0x7fe3c1614350>{number = 2, name = (null)}
```

### 造成死锁变形的代码形式三

```objc
static dispatch_queue_t TaskQueue() {
    static dispatch_queue_t queue = NULL;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        //【串行队列】
        queue = dispatch_queue_create("myqueue", NULL);
    });
    return queue;
}

@implementation ViewController

- (void)testSyncDeadLock3 {
    NSLog(@"1 所在Thread: %@", [NSThread currentThread]);
    
    //【重要】不管是async还是sync，都会造成死锁
    dispatch_async(TaskQueue(), ^{
        NSLog(@"2 ret = %ld 所在Thread: %@", [self doWork], [NSThread currentThread]);
    });
    
    NSLog(@"3 所在Thread: %@", [NSThread currentThread]);
}

- (NSInteger)doWork {
    __block NSInteger ret = 0;
    
    //【重要】
    dispatch_sync(TaskQueue(), ^{
        ret = 10;
    });
    
    return ret;
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	    [self testSyncDeadLock3];
}

@end
```

输出结果

```
2016-12-22 00:44:44.010 Demo[7905:107228] 1 所在Thread: <NSThread: 0x7fed40600f20>{number = 1, name = main}
2016-12-22 00:44:44.011 Demo[7905:107228] 3 所在Thread: <NSThread: 0x7fed40600f20>{number = 1, name = main}
```

没有输出2，依然发生线程死锁，这种其实是前面2种复杂化后的情况。

### sync会等待block执行完毕，才会往下执行代码。async只是将block丢到queue的队尾后，立马就往下执行代码。

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

因为block代码执行是异步的，还没等待执行block，就已经往下执行了2的NSLog().

### 那如果上面代码一定要拿到self.name的修改值了？

- （1）第一种、修改与读写，全部放到async到serial队列


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

- (2) 第二种、使用`dispatch_sync()`让修改值的代码同步等待执行

```objc
static dispatch_queue_t TaskQueue() {
    static dispatch_queue_t queue = NULL;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        //【串行队列】
        queue = dispatch_queue_create("myqueue", NULL);
    });
    return queue;
}


@interface ViewController ()
@property (nonatomic, copy) NSString *name;
@end

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	
	//1. 
	dispatch_sync(dispatch_get_main_queue(), ^{
        self.name = @"hahahah";
    });
        
	//2. 
	NSLog(@"self.name = %@", self.name);
}

@end
```

输出结果

```
2016-12-22 00:52:02.241 Demo[8323:113762] self.name = hahahah
```

前面说过，其实对于sync来说，串行队列与并行队列效果都是一样的。

```objc
static dispatch_queue_t TaskQueue() {
    static dispatch_queue_t queue = NULL;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        // 替换成【并行队列】对于sync，效果是一样的
        queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_CONCURRENT);
    });
    return queue;
}


@interface ViewController ()
@property (nonatomic, copy) NSString *name;
@end

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	
	//1. 
	dispatch_sync(dispatch_get_main_queue(), ^{
        self.name = @"hahahah";
    });
        
	//2. 
	NSLog(@"self.name = %@", self.name);
}

@end
```

输出结果

```
2016-12-22 00:54:25.412 Demo[8453:115629] self.name = hahahah
```

## dispatch sync 还可以用来在多线程访问同一块内存时进行顺序控制，保证数据读写安全

### 假设有一个cache类的代码如下

```objc
@interface XZHCache : NSObject

- (void)setObject:(id)object forKey:(NSString *)aKey;
- (id)objectForKey:(NSString *)aKey;

@end
@implementation XZHCache {
    NSMutableDictionary *_cache;
}

- (NSMutableDictionary *)cache {
    if (!_cache) {
        _cache = [NSMutableDictionary new];
    }
    return _cache;
}

- (void)setObject:(id)object forKey:(NSString *)aKey {
    [self.cache setObject:object forKey:aKey];
}

- (id)objectForKey:(NSString *)aKey {
    return [self.cache objectForKey:aKey];
}

@end
```

### 没有做多线程安全处理的代码如下，当多个线程并发访问时，会造成崩溃

```objc
@implementation ViewController {
    XZHCache *_cache;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    _cache = [XZHCache new];
}

- (void)testSyncMultiThread1 {
    // 模拟1W个子线程同时访问cache对象，进行写操作
    for (NSInteger i = 0; i < 10000; i++) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
    //            NSLog(@"thread: %@", [NSThread currentThread]);
            NSString *key = [NSString stringWithFormat:@"key%ld", i+1];
            NSString *value = [NSString stringWithFormat:@"value%ld", i+1];
            [_cache setObject:value forKey:key];
        });
    }
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	    [self testSyncMultiThread1];
}

@end
```

运行后不是必崩，但是几率很大，崩溃到的代码为`XZHCache`中的如下代码:

```objc
- (void)setObject:(id)object forKey:(NSString *)aKey {
    [self.cache setObject:object forKey:aKey];//【崩溃】代码行
}
```

控制台的报错提示

```
Demo(5399,0x700000117000) malloc: *** error for object 0x7fe563f23a00: pointer being freed was not allocated
*** set a breakpoint in malloc_error_break to debug
(lldb) 
```


因为创建的缓存dic对象这块内存，已经被废弃掉了，而此时的`_cache`这个指针变量就是一个`野指针`。


### 那么上述会造成野指针的原因:

- (1) `第一个线程`进入执行`self.cache`，执行getter方法，发现此时`_cache`为nil，流程走到创建`NSMutableDictionary对象`，但此时突然被CPU切换到`第二个线程`进行执行

- (2) `第二个线程`进入执行`self.cache`，执行getter方法，发现此时`_cache`也是nil。可能此时`第二个线程`开始创建`NSMutableDictionary对象`，并使用`_cache`指向，但此时又突然被CPU切换到第一个线程进行执行

- (3) CPU再次切换回`第一个线程`，让第一个线程接着之前被暂停的地方继续执行。也就是继续执行创建`NSMutableDictionary对象`，并使用`_cache`指向。
	- 这一步就出问题了，之前第二个线程创建的`NSMutableDictionary对象`，因为失去了`_cache`指向，然后就被废弃了

- (4) 很有可能`NSMutableDictionary对象`废弃到`_cache`指向新的`NSMutableDictionary对象`这个过程中，CPU再次切换回第二个线程执行，`[self.cache setObject:@"222" forKey:@"2222"];`，但是此时`_cache`指向之前已经被废弃的`NSMutableDictionary对象`，所以程序崩溃了


这个过程中，最主要的原因就是: `CPU随意的切换其他的线程执行任务，导致某一个线程没有完整的执行玩一段逻辑`。

所以，解决办法就是，让多个线程排队按照顺序，一个接着一个的，对dic对象进行读写。

### 使用 sync + `串行队列`，尝试对多线程访问的同一块内存进行顺序控制

创建一个串行GCD队列


```objc
static dispatch_queue_t TaskQueue() {
    static dispatch_queue_t queue = NULL;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
//        queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_CONCURRENT);
        queue = dispatch_queue_create("myqueue", NULL);
    });
    return queue;
}
```

对XZHCache对象方法使用`dispatch_sync()`进行多线程同步处理

```objc
@interface XZHCache : NSObject

- (void)setObject:(id)object forKey:(NSString *)aKey;
- (id)objectForKey:(NSString *)aKey;

@end

@implementation XZHCache {
    NSMutableDictionary *_cache;
}

- (NSMutableDictionary *)cache {
    if (!_cache) {
        _cache = [NSMutableDictionary new];
    }
    return _cache;
}

- (void)setObject:(id)object forKey:(NSString *)aKey {
    dispatch_sync(TaskQueue(), ^{
        [self.cache setObject:object forKey:aKey];
    });
}

- (id)objectForKey:(NSString *)aKey {
    __block id object = nil;
    dispatch_sync(TaskQueue(), ^{
        object = [self.cache objectForKey:aKey];
    });
    return object;
}

@end
```

再跑前面的ViewController测试代码，正确完成。


### 将上面的GCD串行队列，替换为`并发无序队列`也能够正确同步多线程吗？

替换为GCD并发队列

```c
static dispatch_queue_t TaskQueue() {
    static dispatch_queue_t queue = NULL;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
		queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_CONCURRENT);
//		queue = dispatch_queue_create("myqueue", NULL);
    });
    return queue;
}
```

然后再跑一次ViewController测试代码，发现也是没有崩溃的，说明是没有问题的吗？

尝试其他的代码来测试一下:

```objc
@implementation ViewController

- (NSMutableDictionary *)cache {
    if (!_cache) {
        _cache = [NSMutableDictionary new];
    }
    return _cache;
}

- (void)testSyncMultiThread2 {

    // 模拟1W个子线程同时访问cache对象，进行写操作
    for (NSInteger i = 0; i < 10000; i++) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
//            NSLog(@"thread: %@", [NSThread currentThread]);
			
			// 使用【sync】完成多线程同步
            dispatch_sync(TaskQueue(), ^{
                NSString *key = [NSString stringWithFormat:@"key%ld", i+1];
                NSString *value = [NSString stringWithFormat:@"value%ld", i+1];
                [self.cache setObject:value forKey:key];
                NSLog(@"complet cache.count = %ld", self.cache.count);
            });
        });
    }
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	    [self testSyncMultiThread2];
}

@end
```

程序运行后，有两点错误:

- (1) 出现了`重复`插入的key

```
2016-12-22 13:11:57.678 Demo[5945:92267] complet cache.count = 2
2016-12-22 13:11:57.678 Demo[5945:92264] complet cache.count = 1
2016-12-22 13:11:57.678 Demo[5945:92265] complet cache.count = 1
......略
```

- (2) 还有可能崩溃到`[self.cache setObject:value forKey:key];`这句代码

```
Demo(5945,0x700000cdc000) malloc: *** error for object 0x113cca000: entry for pointer being freed from death-row vanished
*** set a breakpoint in malloc_error_break to debug
.....略
```

所以，当使用 `Concurrent GCD Queue` 时，不能简单的只使用`dispatch_sync()`来完成多线程的同步。


### 当使用 `Concurrent GCD Queue` 时，因为block任务是并发无序的随机执行，很可能并不是一个一个执行的。此时必须使用栅栏（barrier）完成多线程的同步。

必须使用如下的两个函数

```c
// sync版
void dispatch_barrier_sync(dispatch_queue_t queue, dispatch_block_t block);
```

```c
// async版
void dispatch_barrier_async(dispatch_queue_t queue, dispatch_block_t block);
```

当使用如上两种调度方式的block块代码会`单独执行`，并且只有等到`执行完毕`之后才开始调度后面的block块代码。

注意，`barrier`只针对`并发队列`有意义，因为`串行队列`总是一个完成之后才回去调度下一个block块代码。

### 并发队列，处理barrier块代码的逻辑

- (1) 发现当前调度的block块代码是通过`barrier`形式进入队列
- (2) 等待之前所有的block块代码执行`完毕`
- (3) 再`单独`的去执行这个`barrier`块代码
- (4) 等待`barrier`块代码执行完毕之后，才会开始执行后面的block块代码

![](http://p1.bpimg.com/567571/f6640190b47210e7.jpg)

### 使用`barrier`对上面的测试有问题的代码进行改造

```objc
@implementation ViewController

- (NSMutableDictionary *)cache {
    if (!_cache) {
        _cache = [NSMutableDictionary new];
    }
    return _cache;
}

- (void)testSyncMultiThread2 {

    // 模拟1W个子线程同时访问cache对象，进行写操作
    for (NSInteger i = 0; i < 10000; i++) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
//            NSLog(@"thread: %@", [NSThread currentThread]);
			
			// 使用【dispatch_barrier_sync】完成多线程同步
            dispatch_barrier_sync(TaskQueue(), ^{
                NSString *key = [NSString stringWithFormat:@"key%ld", i+1];
                NSString *value = [NSString stringWithFormat:@"value%ld", i+1];
                [self.cache setObject:value forKey:key];
                NSLog(@"complet cache.count = %ld", self.cache.count);
            });
        });
    }
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	    [self testSyncMultiThread2];
}

@end
```

程序正确执行完毕，打印如下

```
2016-12-22 13:29:14.887 Demo[6832:108971] complet cache.count = 1
2016-12-22 13:29:14.888 Demo[6832:108968] complet cache.count = 2
2016-12-22 13:29:14.888 Demo[6832:108967] complet cache.count = 3

............略

2016-12-22 13:29:21.897 Demo[6832:109065] complet cache.count = 10000
```

### 使用`barrier`对XZHCache代码进行改造

```objc
@interface XZHCache : NSObject

- (void)setObject:(id)object forKey:(NSString *)aKey;
- (id)objectForKey:(NSString *)aKey;

@end

@implementation XZHCache {
    NSMutableDictionary *_cache;
}

- (NSMutableDictionary *)cache {
    if (!_cache) {
        _cache = [NSMutableDictionary new];
    }
    return _cache;
}

- (void)setObject:(id)object forKey:(NSString *)aKey {
    dispatch_barrier_sync(TaskQueue(), ^{
        [self.cache setObject:object forKey:aKey];
    });
}

- (id)objectForKey:(NSString *)aKey {
    __block id object = nil;
    dispatch_barrier_sync(TaskQueue(), ^{
        object = [self.cache objectForKey:aKey];
    });
    return object;
}

@end
```

测试代码

```objc
static dispatch_queue_t TaskQueue() {
    static dispatch_queue_t queue = NULL;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_CONCURRENT);
//        queue = dispatch_queue_create("myqueue", NULL);
    });
    return queue;
}

@implementation ViewController {
    XZHCache *_cache;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    _cache = [XZHCache new];
}

- (void)testSyncMultiThread1 {
    // 模拟1W个子线程同时访问cache对象，进行写操作
    for (NSInteger i = 0; i < 10000; i++) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
    //            NSLog(@"thread: %@", [NSThread currentThread]);
            NSString *key = [NSString stringWithFormat:@"key%ld", i+1];
            NSString *value = [NSString stringWithFormat:@"value%ld", i+1];
            [_cache setObject:value forKey:key];
        });
    }
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self testSyncMultiThread1];
}

@end
```

程序正常执行。


### 上面改造的XZHCache虽然正常执行，但是还是有优化的地方的

- (1) 写操作，必须得`按照顺序`执行:
	- 可以将`同步等待`执行，替换为`异步`执行
	- 但是会增加block的拷贝

- (2) 读操作，其实是可以并发执行。可以分别提供两种接口:
	- `同步等待`获取返回值
	- `异步回调`获取返回值


最终XZHCache改造如下

```c
static dispatch_queue_t TaskQueue() {
    static dispatch_queue_t queue = NULL;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_CONCURRENT);
//        queue = dispatch_queue_create("myqueue", NULL);
    });
    return queue;
}
```

```objc
@interface XZHCache : NSObject

- (void)setObject:(id)object forKey:(NSString *)aKey;
- (id)objectForKey:(NSString *)aKey;
- (void)objectForKey:(NSString *)aKey competionBlock:(void (^)(id))block;

@end

@implementation XZHCache {
    NSMutableDictionary *_cache;
}

- (NSMutableDictionary *)cache {
    if (!_cache) {
        _cache = [NSMutableDictionary new];
    }
    return _cache;
}

- (void)setObject:(id)object forKey:(NSString *)aKey {
    // dispatch_barrier_sync >>>>> dispatch_barrier_async
    dispatch_barrier_async(TaskQueue(), ^{
        NSLog(@"写入缓存: key = %@, value = %@", aKey, object);
        [self.cache setObject:object forKey:aKey];
    });
}

- (id)objectForKey:(NSString *)aKey {
    __block id object = nil;
    dispatch_sync(TaskQueue(), ^{
        object = [self.cache objectForKey:aKey];
    });
    return object;
}

- (void)objectForKey:(NSString *)aKey competionBlock:(void (^)(id))block {
    dispatch_async(TaskQueue(), ^{
        block([self.cache objectForKey:aKey]);
    });
}

@end
```

测试代码

```objc
@implementation ViewController {
    XZHCache *_cache;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    _cache = [XZHCache new];
}

- (void)testSyncMultiThread3 {
    // 模拟1W个子线程同时访问cache对象，进行写操作
    for (NSInteger i = 0; i < 5; i++) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            NSString *key = [NSString stringWithFormat:@"key%ld", i+1];
            NSString *value = [NSString stringWithFormat:@"value%ld", i+1];
            [_cache setObject:value forKey:key];
        });
    }
    
    // 模拟1W个子线程同时访问cache对象，进行读操作
    for (NSInteger i = 0; i < 5; i++) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            NSString *key = [NSString stringWithFormat:@"key%ld", i+1];
            
            // 等待获取返回值
            NSLog(@"sync object = %@", [_cache objectForKey:key]);
            
            // 异步回调获取返回值
            [_cache objectForKey:key competionBlock:^(id object) {
                NSLog(@"async object = %@", object);
            }];
        });
    }
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self testSyncMultiThread3];
}

@end
```

为了看清楚读操作与写操作是否正确，把子线程数降低到5，输出结果

```
2016-12-22 14:00:36.825 Demo[9232:142361] 写入缓存: key = key1, value = value1
2016-12-22 14:00:36.825 Demo[9232:142361] 写入缓存: key = key2, value = value2
2016-12-22 14:00:36.825 Demo[9232:142361] 写入缓存: key = key4, value = value4
2016-12-22 14:00:36.826 Demo[9232:142361] 写入缓存: key = key5, value = value5
2016-12-22 14:00:36.826 Demo[9232:142350] sync object = value1
2016-12-22 14:00:36.826 Demo[9232:142361] 写入缓存: key = key3, value = value3
2016-12-22 14:00:36.827 Demo[9232:142359] sync object = value2
2016-12-22 14:00:36.827 Demo[9232:142349] sync object = value3
2016-12-22 14:00:36.827 Demo[9232:142361] async object = value1
2016-12-22 14:00:36.827 Demo[9232:142360] sync object = value5
2016-12-22 14:00:36.827 Demo[9232:142353] sync object = value4
2016-12-22 14:00:36.827 Demo[9232:142359] async object = value2
2016-12-22 14:00:36.827 Demo[9232:142349] async object = value3
2016-12-22 14:00:36.827 Demo[9232:142360] async object = value5
2016-12-22 14:00:36.827 Demo[9232:142353] async object = value4
```

可以看到结果是正确的。小结上面的结构:

```
- 并行GCD队列
- 写操作， dispatch_barrier_async
- 读操作1，dispatch_sync
- 读操作2，dispatch_async
```

这样的话，将同步操作与异步操作结合起来，使用队列的机制来代替加锁完成多线程同步，效率会更高。

### 对于XZHCache的setObject:forKey其实也可以使用`dispatch_barrier_sync`实现同步

这样会减少block的拷贝操作，所以如果block包裹的代码不是特别多、耗时不长的话，也还是可以考虑使用`dispatch_barrier_sync`来实现的。