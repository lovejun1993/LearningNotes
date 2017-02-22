## 在一个线程上的某一个执行函数中，使用autoreleasepool

```objc
//1. 创建一个释放池
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];

//2. 将我们自己的对象放入到释放池
id obj = [[NSObject alloc] init];
[obj autorelase];

//3. 不在使用释放池时，将释放池中所有的对象全部清除，而最终释放池随着线程的废弃而废弃
[pool relase];
```

## 给我们自己创建的子线程，添加autoreleasepool

```objc
@interface NSThread (XZHAddtions)

+ (void)xzh_addAutoreleasePool;

@end
```

当创建了一个子线程thread时，调用该分类的方法，完成给该子线程添加autoreleasepool。

因为我们自己创建的子线程thread，默认是没有autoreleasepool的，需要我们自己手动创建添加。

## `xzh_addAutoreleasePool`分类方法实现

```objc
@implementation NSThread (XZHAddtions)

+ (void)xzh_addAutoreleasePool {

    //1. 主线程thread，会自己创建释放池
    if ([NSThread isMainThread]) {return;}
    
    //2. 取出线程的字典对象
    NSMutableDictionary *threadDic = [NSThread currentThread].threadDictionary;
    
    //3. 从线程的字典对象中，取出标记位的值，判断是否已经创建了释放池
    if ([threadDic objectForKey:kXZHNSThreadAutoleasePoolKey]) {return;}
    
    //4. 走到这里，说明还没有添加释放池，就给当前线程创建释放池
    AutoreleasePoolSetup();
    
    //5. 标记当前线程已经创建了释放池
    [threadDic setObject:kXZHNSThreadAutoleasePoolKey forKey:kXZHNSThreadAutoleasePoolKey];
}

@end
```

所以，对于`threadDictionary`线程对象自身维护的一个dic缓存对象，可以在任何的地方，从对应的线程取出出来。


给线程存储一些标记位，来做一些额外的逻辑处理。

## RunLoop所有的状态

```c
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {

    // 状态一、即将进入Loop
    kCFRunLoopEntry = (1UL << 0),
    
    // 状态二、 即将处理 Timer
    kCFRunLoopBeforeTimers = (1UL << 1),
    
    // 状态三、 即将处理 Source
    kCFRunLoopBeforeSources = (1UL << 2),
    
    // 状态四、 即将进入休眠
    kCFRunLoopBeforeWaiting = (1UL << 5),
    
    // 状态五、 刚从休眠中唤醒
    kCFRunLoopAfterWaiting = (1UL << 6),
    
    // 状态六、 即将退出Loop
    kCFRunLoopExit = (1UL << 7),
    
    // Mask掩码
    kCFRunLoopAllActivities = 0x0FFFFFFFU
};
```

## `AutoreleasePoolSetup()`实现

```c
static void AutoreleasePoolSetup() {

    //1. 给runloop添加第一个observer
    AddRunLoopObserverForAutoreleasePoolPush();
    
    //2. 给runloop添加第二个observer
    AddRunLoopObserverForAutoreleasePoolPop();
}
```

依次完成两个runloop observer的添加。

## `AddRunLoopObserverForAutoreleasePoolPush()`的实现

```c
static void AddRunLoopObserverForAutoreleasePoolPush() {
    
    /**
     *  注意: 使用CoreFoundation版本的RunLoop的api，因为是线程安全的
     */

    //1. 获取/创建 当前线程的runloop
    CFRunLoopRef runLoop = CFRunLoopGetCurrent();
    
    //2. 创建 runloop observer，监听 `runloop进入`的状态
    /*
    创建runloop observer函数的参数意义:
    CFRunLoopObserverCreate(CFAllocatorRef allocator,
                            CFOptionFlags activities,//监听runloop的哪一个状态
                            Boolean repeats,//重复性不断监听
                            CFIndex order,//RunLoopObserver的优先级，当在Runloop同一运行阶段中有多个CFRunLoopObserver时，根据这个来先后调用，CFRunLoopObserver，默认值是0
                            CFRunLoopObserverCallBack callout,//observer回调c函数
                            CFRunLoopObserverContext *context);//用于给observer回调c函数中，传递参数的context
     */
    
    /**
     *  observer的优先级最大:
     *  0x7FFFFFFF是一个十六进制数
     *  转成二进制数==>0111,1111,1111,1111,1111,1111,1111,1111==>32位二进制数，第一位0，表示正数
     *  而int类型变量占用32个二进制位，第一位是符号位（表示正负数），后面31位是数值位
     *  所以，0x7FFFFFFF表示最大的int类型正数，也可以使用 INT_MAX 代替
     */
    CFIndex order = 0x7FFFFFFF;
    CFRunLoopObserverRef pushObserver = NULL;
    pushObserver = CFRunLoopObserverCreate(CFAllocatorGetDefault(),
                                           kCFRunLoopEntry,
                                           true,
                                           order,
                                           XZHCFRunLoopObserverCallBack,
                                           NULL);
    
    //3. observer注册给runloop，注意runloop mode >>> commom modes
    CFRunLoopAddObserver(runLoop, pushObserver, kCFRunLoopCommonModes);
    
    //4.
    CFRelease(pushObserver);
}
```

监听runloop的`kCFRunLoopEntry`状态，即在`即将进入runloop的时候`，然后回调执行`XZHCFRunLoopObserverCallBack`这个c函数。

## `AddRunLoopObserverForAutoreleasePoolPop()`的实现

```c
static void AddRunLoopObserverForAutoreleasePoolPop() {

    //1.
    CFRunLoopRef runLoop = CFRunLoopGetCurrent();
    
    //2. runloop observer 优先级最低:
    CFIndex order = -0x7FFFFFFF;
    
    //3. 监听runloop状态: 休眠、退出执行
    CFRunLoopObserverRef popObserver = NULL;
    popObserver = CFRunLoopObserverCreate(CFAllocatorGetDefault(),
                                          kCFRunLoopBeforeWaiting | kCFRunLoopExit,
                                          true,
                                          order,
                                          XZHCFRunLoopObserverCallBack,
                                          NULL);
    
    //4.
    CFRunLoopAddObserver(runLoop, popObserver, kCFRunLoopCommonModes);
    
    //5.
    CFRelease(popObserver);
}
```

监听runloop的`kCFRunLoopBeforeWaiting 与 kCFRunLoopExit`状态，即在`runloop即将睡眠`和`runloop即将退出`，然后回调执行`XZHCFRunLoopObserverCallBack`这个c函数。

## `XZHCFRunLoopObserverCallBack()`的实现

该c函数，负责处理如上两个observer的回调。

```c
static void XZHCFRunLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *context)
{
	// 根据当前runloop的状态进行不同的操作
    switch (activity) {
            
        //状态一、runloop即将进入
        case kCFRunLoopEntry: {
            XZHAutoreleasePoolPush();// 直接创建新的释放池
        }
            break;
            
        //状态二、runloop即将休眠
        case kCFRunLoopBeforeWaiting: {
            XZHAutoreleasePoolPop();// 先废弃老的释放池
            XZHAutoreleasePoolPush();// 再创建新的释放池
        }
            break;
            
        //状态三、runloop即将退出
        case kCFRunLoopExit: {
            XZHAutoreleasePoolPop();// 直接释放老的释放池
        }
            break;
            
        default:
            break;
    }
}
```

从上面的逻辑可以看出，主要是三种状态：

- (1) runloop即将进入，也就是runloop的第一次初始化的时候
- (2) runloop即将睡眠，就是处理完当前所有的source了，已经没有其他事情了
- (3) runloop即将退出

## `XZHAutoreleasePoolPush()`负责创建新的释放池

```c
static inline void XZHAutoreleasePoolPush() {
    
    //1. 读取线程对象的字典
    NSMutableDictionary *dic =  [NSThread currentThread].threadDictionary;

    //2. 获取线程对象存储的pool数组对象
    CFMutableArrayRef autoreleasePools = (__bridge CFMutableArrayRef)([dic objectForKey:kXZHNSThreadAutoleasePoolStackKey]);
    
    //3. 如果pool数组不存在，则先创建数组对象，并存入到线程字典对象中
    if (!autoreleasePools) {
    
    	//3.1 创建一个数组对象
        autoreleasePools = CFArrayCreateMutable(kCFAllocatorDefault, 1, NULL);
        
        //3.2 将数组对象存入到线程字典对象中
        [dic setObject:(__bridge id)(autoreleasePools) forKey:kXZHNSThreadAutoleasePoolStackKey];
        
        //3.3 释放掉局部的数组对象指针变量
        CFRelease(autoreleasePools);
    }
    
    //4. 创建新的pool对象
    NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
    
    //5. 将pool对象，加入到数组最后一个位置
    CFArrayAppendValue(autoreleasePools, (__bridge void*)(pool));
}
```

由于使用了`NSAutoreleasePool`，必须让该文件支持`MRC`，因为在ARC下不能使用`NSAutoreleasePool`。

下面这段宏，提醒开发者必须将该源文件使用MRC，不能使用ARC，否则编译器报错:

```
#if __has_feature(objc_arc)
#error This file must be compiled without ARC. Specify the -fno-objc-arc flag to this file.
#endif
```

## `XZHAutoreleasePoolPop()` 负责废弃老的释放池

```c
static inline void XZHAutoreleasePoolPop() {

    //1. 获取线程对象存储的pool数组对象
    CFMutableArrayRef autoreleasePools = (__bridge CFMutableArrayRef)([[NSThread currentThread].threadDictionary objectForKey:kXZHNSThreadAutoleasePoolStackKey]);
    
    //2. 获取最后的一个pool
    NSAutoreleasePool *lastPool = CFArrayGetValueAtIndex(autoreleasePools, CFArrayGetCount(autoreleasePools) - 1);
    
    //3. 清除pool中的所有的对象
    [lastPool drain];
    
    //4. 移除pool
    CFArrayRemoveValueAtIndex(autoreleasePools, CFArrayGetCount(autoreleasePools) - 1);
}
```

也就是说，`autoreleasePools`这个数组对象，永远都只会保存一个`NSAutoreleasePool`对象。

## 那么上面的代码使用模板

```objc
- (void)testAutoreleasePool {
    
    //1.
    static NSThread *thread;
    thread = [[NSThread alloc] initWithTarget:self
                                     selector:@selector(threadEntry:)
                                       object:nil];
    
    //2.
    [thread start];
}

- (void)threadEntry:(id)sender {
    
    //1. 添加port事件源
    [[NSRunLoop currentRunLoop] addPort:[NSMachPort port] forMode:NSRunLoopCommonModes];
    
    //2. 开启runloop
    [[NSRunLoop currentRunLoop] run];
    
    //3. 调用thread分类方法创建释放池
    [NSThread xzh_addAutoreleasePool];
    
    //4. 线程的逻辑处理代码，将对象放入到释放池，不用再创建释放池了...
    NSObject *entity = [NSObject new];
    [entity autorelease];
}
```

注意，内部代码会默认创建子线程的runloop，并添加observer，但是好像并没有`run`线程的runloop。

## 总结上面所有的代码，就是为了给线程一直持有一个`最新鲜`autoreleasepool对象

- (1) 当runloop进入时，创建一个autoreleasepool
- (2) 当runloop即将休息时
	- (2.1) 先`废弃`掉之前存在的一个`老的`autoreleasepool
	- (2.2) 再`重新创建`一个`新的`autoreleasepool
- (3) 当runloop即将退出时，直接废弃线程持有的autoreleasepool



