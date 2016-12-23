## GCD Dispatch Queue 与 thread线程 的关系

### 组织结构

![](http://i1.piimg.com/4851/42558425db6fbe0d.png)

可以看到分为三层:

- (1) 最外层，是我们代码中自己创建的queue实例
	- serial queue
	- concurrent queue
	- 其优先级都是从系统的queue进行复制
		- serial queue: 从main queue复制
		- concurrent queue: 从global queue复制

- (2) 中间层，系统dispatch queue实例
	- main queue 串行有序
	- global queue 并发无序，其实分为4个具体的queue实例:
		- (1) `hign` priority queue
		- (2) `default` priority queue
		- (3) `low` priority queue
		- (4) `backgroud` priority queue

- (3) 最底层，就是最终系统直接使用的thread线程
	- serial queue:
		- main queue >>> main thread 主线程
		- 我们创建的serial queue >>> 新的子线程
	- concurrent queue:
		- 将会从 `GCD Thread Pool` 缓存池中随机取出一个子线程

## dispatch queue 换算为多少个 thread ？

![](http://i1.piimg.com/4851/2427128a9d1662ac.png)

### `一个`serial dispatch queue 实例 == `一个`thread实例

- 从队列中依次按照顺序取出头部的`dispatch_block_t`实例(block任务)

- 会一直`重复使用一个唯一`的线程去执行任务，不再创建第二个子线程

- 并且只有当一个任务在线程上执行完毕之后，才去开始执行下一个任务

所以就跟无限制创建线程对象一样，也需要避免无限制创建`serial dispatch queue`实例。


比如、YYDispatchQueuePool实现了一个`serial dispatch queue`实例的缓存池，来管理所有的`serial dispatch queue`实例:

- 框架实例化时，根据cpu核心数创建一定数量的`serial dispatch queue`实例并缓存起来
- 从缓存中`随机`取出一个`serial dispatch queue`实例来调度block任务
- 适用于`并发无序`的异步线程任务代码

### `一个`concurrent dispatch queue 实例 == `多个`thread实例

- 可以指定一个最大并发数

- 从队列中`随机`取出几个（`dispatch_block_t`实例)，执行顺序没有先后之分
	
- 每一个`dispatch_block_t`，可能分配到`不同的`线程上去执行


一般其实很少自己去创建`concurrent dispatch queue`，用的最多的可能还是`serial dispatch queue`，比如:

- (1) dispatch main queue 操作UI对象
- (2) 自己创建一个`单例 serial dispatch queue`来完成一些大任务又需要多线程同步

## dispatch async queue block

如下代码都是在ViewController中的方法实现调用。

### async serial queue

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

输出结果

```
2016-12-21 23:53:53.097 Demo[5017:61336] 1
2016-12-21 23:53:53.098 Demo[5017:61336] 2
2016-12-21 23:53:53.098 Demo[5017:61593] Block任务1, 所在Thread: <NSThread: 0x7fb48243e5f0>{number = 2, name = (null)}
2016-12-21 23:53:53.099 Demo[5017:61336] 3
2016-12-21 23:53:53.099 Demo[5017:61593] Block任务2, 所在Thread: <NSThread: 0x7fb48243e5f0>{number = 2, name = (null)}
2016-12-21 23:53:53.100 Demo[5017:61336] 4
2016-12-21 23:53:53.100 Demo[5017:61593] Block任务3, 所在Thread: <NSThread: 0x7fb48243e5f0>{number = 2, name = (null)}
2016-12-21 23:53:53.101 Demo[5017:61593] Block任务4, 所在Thread: <NSThread: 0x7fb48243e5f0>{number = 2, name = (null)}
2016-12-21 23:53:53.102 Demo[5017:61593] Block任务5, 所在Thread: <NSThread: 0x7fb48243e5f0>{number = 2, name = (null)}
2016-12-21 23:53:53.105 Demo[5017:61593] Block任务6, 所在Thread: <NSThread: 0x7fb48243e5f0>{number = 2, name = (null)}
2016-12-21 23:53:53.107 Demo[5017:61593] Block任务7, 所在Thread: <NSThread: 0x7fb48243e5f0>{number = 2, name = (null)}
2016-12-21 23:53:53.107 Demo[5017:61593] Block任务8, 所在Thread: <NSThread: 0x7fb48243e5f0>{number = 2, name = (null)}
```

- (1) block都是在一个`相同`的子线程上执行
- (2) block是按照`顺序`一个一个接着完成
- (3) block是`异步`执行

### async concurrent queue

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
```

输出结果

```
2016-12-21 23:57:33.848 Demo[5259:65743] 1
2016-12-21 23:57:33.849 Demo[5259:65743] 2
2016-12-21 23:57:33.849 Demo[5259:65743] 3
2016-12-21 23:57:33.849 Demo[5259:65743] 4
2016-12-21 23:57:33.849 Demo[5259:65830] Block任务1, 所在Thread: <NSThread: 0x7fe271547d80>{number = 2, name = (null)}
2016-12-21 23:57:33.849 Demo[5259:65831] Block任务2, 所在Thread: <NSThread: 0x7fe27155ac80>{number = 3, name = (null)}
2016-12-21 23:57:33.850 Demo[5259:65895] Block任务4, 所在Thread: <NSThread: 0x7fe271407f70>{number = 5, name = (null)}
2016-12-21 23:57:33.850 Demo[5259:65830] Block任务5, 所在Thread: <NSThread: 0x7fe271547d80>{number = 2, name = (null)}
2016-12-21 23:57:33.850 Demo[5259:65831] Block任务6, 所在Thread: <NSThread: 0x7fe27155ac80>{number = 3, name = (null)}
2016-12-21 23:57:33.850 Demo[5259:65833] Block任务3, 所在Thread: <NSThread: 0x7fe27150bbe0>{number = 4, name = (null)}
```

- (1) block是在`多个不同`的子线程上执行
- (2) block是按照`无序`执行，随机的
- (3) block是`异步`执行
- (4) 底层使用的thread是经过`重用`处理的


## dispatch sync queue block

### sync serial queue

```objc
dispatch_queue_t queue = dispatch_queue_create("用户队列", NULL);

NSLog(@"1");

dispatch_sync(queue, ^{
    NSLog(@"Block任务1, 所在Thread: %@\n", [NSThread currentThread]);
});

NSLog(@"2");

dispatch_sync(queue, ^{
    NSLog(@"Block任务2, 所在Thread: %@\n", [NSThread currentThread]);
});

NSLog(@"3");

dispatch_sync(queue, ^{
    NSLog(@"Block任务3, 所在Thread: %@\n",[NSThread currentThread]);
});

NSLog(@"4");

dispatch_sync(queue, ^{
    NSLog(@"Block任务4, 所在Thread: %@\n",[NSThread currentThread]);
});
dispatch_sync(queue, ^{
    NSLog(@"Block任务5, 所在Thread: %@\n",[NSThread currentThread]);
});
dispatch_sync(queue, ^{
    NSLog(@"Block任务6, 所在Thread: %@\n",[NSThread currentThread]);
});
```

输出结果

```
2016-12-22 00:01:07.852 Demo[5471:69346] 1
2016-12-22 00:01:07.853 Demo[5471:69346] Block任务1, 所在Thread: <NSThread: 0x7fd229f00ec0>{number = 1, name = main}
2016-12-22 00:01:07.853 Demo[5471:69346] 2
2016-12-22 00:01:07.853 Demo[5471:69346] Block任务2, 所在Thread: <NSThread: 0x7fd229f00ec0>{number = 1, name = main}
2016-12-22 00:01:07.853 Demo[5471:69346] 3
2016-12-22 00:01:07.853 Demo[5471:69346] Block任务3, 所在Thread: <NSThread: 0x7fd229f00ec0>{number = 1, name = main}
2016-12-22 00:01:07.854 Demo[5471:69346] 4
2016-12-22 00:01:07.854 Demo[5471:69346] Block任务4, 所在Thread: <NSThread: 0x7fd229f00ec0>{number = 1, name = main}
2016-12-22 00:01:07.854 Demo[5471:69346] Block任务5, 所在Thread: <NSThread: 0x7fd229f00ec0>{number = 1, name = main}
2016-12-22 00:01:07.854 Demo[5471:69346] Block任务6, 所在Thread: <NSThread: 0x7fd229f00ec0>{number = 1, name = main}
```

- (1) block全部都是在`主线程`上执行，而`并非`是在一个`子线程`执行
- (2) block是同步等待执行完毕，才会走后面的代码
- (3) block任务全部按照`顺序`一个一个接着完成

### sync concurrent queue

```objc
dispatch_queue_t queue = dispatch_queue_create("用户队列", DISPATCH_QUEUE_CONCURRENT);
    
NSLog(@"1");
    
dispatch_sync(queue, ^{
    NSLog(@"Block任务1, 所在Thread: %@\n", [NSThread currentThread]);
});
    
NSLog(@"2");
    
dispatch_sync(queue, ^{
    NSLog(@"Block任务2, 所在Thread: %@\n", [NSThread currentThread]);
});
    
NSLog(@"3");
    
dispatch_sync(queue, ^{
    NSLog(@"Block任务3, 所在Thread: %@\n",[NSThread currentThread]);
});
    
NSLog(@"4");
    
dispatch_sync(queue, ^{
    NSLog(@"Block任务4, 所在Thread: %@\n",[NSThread currentThread]);
});
dispatch_sync(queue, ^{
    NSLog(@"Block任务5, 所在Thread: %@\n",[NSThread currentThread]);
});
dispatch_sync(queue, ^{
    NSLog(@"Block任务6, 所在Thread: %@\n",[NSThread currentThread]);
});
```

输出如下

```
2016-12-22 00:11:54.519 Demo[6073:77156] 1
2016-12-22 00:11:54.520 Demo[6073:77156] Block任务1, 所在Thread: <NSThread: 0x7fcbf9e03550>{number = 1, name = main}
2016-12-22 00:11:54.520 Demo[6073:77156] 2
2016-12-22 00:11:54.521 Demo[6073:77156] Block任务2, 所在Thread: <NSThread: 0x7fcbf9e03550>{number = 1, name = main}
2016-12-22 00:11:54.521 Demo[6073:77156] 3
2016-12-22 00:11:54.521 Demo[6073:77156] Block任务3, 所在Thread: <NSThread: 0x7fcbf9e03550>{number = 1, name = main}
2016-12-22 00:11:54.522 Demo[6073:77156] 4
2016-12-22 00:11:54.522 Demo[6073:77156] Block任务4, 所在Thread: <NSThread: 0x7fcbf9e03550>{number = 1, name = main}
2016-12-22 00:11:54.523 Demo[6073:77156] Block任务5, 所在Thread: <NSThread: 0x7fcbf9e03550>{number = 1, name = main}
2016-12-22 00:11:54.523 Demo[6073:77156] Block任务6, 所在Thread: <NSThread: 0x7fcbf9e03550>{number = 1, name = main}
```

- (1) block全部都是在`主线程`上执行，而`并非`是在一个`子线程`执行
- (2) block是同步等待执行完毕，才会走后面的代码
- (3) block任务全部按照`顺序`一个一个接着完成

可以得到，sync调度block时，不管queue是`串行`还是`并行`都是一样的效果。


![](http://a2.qpic.cn/psb?/V11ePBui3l2qGa/mfvtHS8pFneveCrP28kf30LnTHHACBf.FI0Q37nPRXw!/b/dHgBAAAAAAAA&bo=sQKAAhsD4gIFCOI!&rf=viewer_4)


## dispatch once来让一段代码只执行一次

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

大概知道了让代码执行一次的道理了:

- 代码没有执行之前，将标志位置为 0
- 代码第一次执行之后，就将标志位改为 -1
- 每次执行到`dispatch_once()`代码块时，判断标志位如果是`0`就执行Block，如果是`-1`就不执行Block
- 而改变这个`onceToken`值的代码，肯定是做了多线程同步处理的

## `dispatch_after` 在给定的线程队列中，延迟执行block

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


## iOS8之后提交到gcd队列中的block也可`取消`了

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


## `dispatch_group_t`，将一个队列上的`多个block任务`组合起来，可以方便设置所有任务完成时候的回调.


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

下面两张方式是等价的.

```objc
dispatch_group_async(group, queue, ^{ 
	// 。。。 
}); 
```

```objc
dispatch_group_enter (group);

dispatch_async(queue, ^{
	// 。 。。。
});

dispatch_group_leave (group);

});
```

### `dispatch_group_notify`组任务完成之后一定会执行的代码块

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


###  `dispatch_group_wait` 等待该group中所有的线程任务全部执行完毕


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


## `dispatch_source_t` 创建事件源，用于注册到RunLoop事件监听

### 所有的source分类

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

### 主要的使用步骤:

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


## `dispatch_queue_set_specific()` 、`dispatch_get_specific()`

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


## `dispatch_set_target_queue(子队列1, 集合队列)`完成两件事: 1)将多个子队列组合到一个整体队列 2)复制队列的优先级

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


## `dispatch_barrier_async`来同步等待`concurrent queue`中的一个队列中的block执行

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


## `dispatch_async_f` 完成对`C方法`的异步调用


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

## `dispatch_suspend/dispatch_resume` 挂起/恢复一个队列进行block调度

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


注意:  对于`全局并发队列`来执行 `dispatch_suspend(), dispatch_resume(), dispatch_set_context()` 这三种操作是没有任何效果的。
