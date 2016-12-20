---
layout: post
title: "Kache、KHolder"
date: 2015-08-23 19:26:46 +0800
comments: true
categories: 
---

###KHolder提供最终的缓存功能

- 一个KHolder对象，就是一个单独的`内存块`
- 使用这个`内存块`来保存一些常用的内存对象
- 以及将超时的缓存项、超过长度的缓存项，写入磁盘文件


***

```objc
@interface KHolder ()

@property (nonatomic, strong) dispatch_queue_t syncThreadQueue;

// 正在进行归档的状态位
@property (assign, atomic)      BOOL                        archiving;

// 正在清理超时缓存的状态位
@property (assign, atomic)      BOOL                        cleaning;

@property (strong, nonatomic)   NSFileManager               *fileManager;

//缓存Key列表
@property (strong, atomic)      NSMutableArray              *keys;

// 缓存大小
@property (assign, nonatomic)   NSUInteger                  size;

// 缓存内容
@property (strong, nonatomic)   NSMutableDictionary         *objects;

// 默认磁盘缓存路径
@property (strong, nonatomic)   NSString                    *path;

// 同步锁
@property (strong, nonatomic)   NSRecursiveLock             *lock;

// 把数据写到磁盘
- (void)doArchive;
- (void)archiveData;
- (void)archiveAllData;

- (void)doClean;
- (void)cleanExpiredObjects;

@end
```

****

###KHolder对象初始化，传入一个唯一标识

```objc

//同步队列
- (dispatch_queue_t)syncThreadQueue {
    if (!_syncThreadQueue) {
        _syncThreadQueue = dispatch_queue_create("com.cn.xzh.queue", DISPATCH_QUEUE_SERIAL);
    }
    return _syncThreadQueue;
}


- (id)initWithToken:(NSString *)token
{
    self = [super init];
    
    if (self) {
        
        //使用dispatch_async + serial queue 同步多线程
        __weak __typeof(self)weakSelf = self;
        dispatch_async(self.syncThreadQueue, ^{
            __strong __typeof(weakSelf)strongSelf = weakSelf;
            
            //1. 缓存key -- 缓存value
            strongSelf.objects = [[NSMutableDictionary alloc] init];
            
            //2. 缓存key数组
            strongSelf.keys = [[NSMutableArray alloc] init];
            
            //3. 默认初始化长度为0
            strongSelf.size = 0;
            
            //4. 同步锁，同步进行内存操作
            self.lock = [[NSRecursiveLock alloc] init];
            
            //5. 磁盘缓存目录
            NSArray *paths = NSSearchPathForDirectoriesInDomains(NSLibraryDirectory, NSUserDomainMask, YES);
            NSString *libDirectory = [paths objectAtIndex:0];
            strongSelf.path = [libDirectory stringByAppendingPathComponent:[Kache_Objects_Disk_Path stringByAppendingPathExtension:token]];
            
            //6. 异步执行监控内存缓存开销，将多余的内存对象写入磁盘
            dispatch_async(dispatch_get_global_queue(0, 0), ^{
                [strongSelf doArchive];
            });
            
            //7. 异步执行清理超时缓存，写入磁盘
            dispatch_async(dispatch_get_global_queue(0, 0), ^{
                [strongSelf doClean];
            });

        });
    }

    return self;
}
```

###将多余的内存对象写入磁盘

- 首先在`当前线程`的runloop注册一个NSTimer

```
- (void)doArchive
{
    //1.
    NSTimer *archivingTimer = [NSTimer timerWithTimeInterval:10.f
                                                      target:self
                                                    selector:@selector(archiveData)
                                                    userInfo:nil
                                                     repeats:YES];
    //2.
    NSRunLoop *archivingRunloop = [NSRunLoop currentRunLoop];
    
    //3.
    [archivingRunloop addTimer:archivingTimer forMode:NSDefaultRunLoopMode];
    
    //4. 写一个while循环，来防止runloop退出
    //这里其实可以模仿AFNetworking，给runloop注册为一个NSMachPort来实现
    while (YES) {
        [archivingRunloop runMode:NSDefaultRunLoopMode
                       beforeDate:[NSDate dateWithTimeIntervalSinceNow:2.f]];
    }
}
```

这里可以使用一个单例子线程的runloop来完成读写优化.


```objc
- (void)archiveData
{
    //1. 标志正在归档操作
    //这个属性使用atomic原子操作，不用使用同步锁控制
    self.archiving = YES;
    
    //2. 判断不能进行clean操作
    if (self.cleaning) {
        return;
    }
    
    //3. 一直归档到指定限制大小的内存开销
    if (0 < ARCHIVING_THRESHOLD
       && ARCHIVING_THRESHOLD < self.size)
    {
        
        BOOL isDirectory = NO;
        if (! [[NSFileManager defaultManager] fileExistsAtPath:self.path isDirectory:&isDirectory]) {
           [self.fileManager createDirectoryAtPath:self.self.path
                       withIntermediateDirectories:YES
                                        attributes:nil
                                             error:nil];
        }
        
        //锁一下，阻止其他内存操作
        [self.lock lock];
        
        //拷贝一个新的key数组，因为后面要修改key数组
        NSMutableArray *copiedKeys = [self.keys mutableCopy];
        
        while (0 < [copiedKeys count])
        {
            // 归档至阈值一半的数据
            if ((ARCHIVING_THRESHOLD / 2) >= self.size) {
                break;
            }
            
            //取出最后一个key
            NSString *key = [copiedKeys lastObject];
            
            NSString *filePath = [self.path stringByAppendingPathComponent:key];
           
            NSData *data = [self.objects objectForKey:key];
            
            [data writeToFile:filePath atomically:YES];
            
            self.size -= data.length;
            
            [copiedKeys removeLastObject];
            
            [self.objects removeObjectForKey:key];
        }
        
        //不再使用释放掉拷贝数组
        copiedKeys = nil;
        
        //释放锁，可以进行其他内存操作
        [self.lock unlock];
    }
    
    //4.
    self.archiving = NO;
}
```

###清理超时缓存对象

```objc
- (void)cleanExpiredObjects
{
    //1. 标记正在clean操作
    self.cleaning = YES;
    
    //2. 不能够进行achive操作
    if (self.archiving) {
        return;
    }
    
    //3. 同步线程
    [self.lock lock];
    
    //4. 从左往右遍历
    if (self.keys && 0 < [self.keys count]) {
        
        for (int i = 0; i < [self.keys count] - 1; i ++) {
            
            NSString *tmpKey = [self.keys objectAtIndex:i];
            
            KObject *leftObject = [self objectForKey:tmpKey];
            
            if ([leftObject expiredTimestamp] < [KUtil nowTimestamp]) {
                [self removeObjectForKey:tmpKey];
            } else {
                break;
            }
        }
    }
    
    //解除锁，可以进行其他内存操作
    [self.lock unlock];
    
    self.cleaning = NO;
}
```

###缓存传入的内存对象

```objc
/**
 *  @param value    内存对象
 *  @param key      缓存key
 *  @param duration 超时间隔
 */
- (void)setValue:(id)value
          forKey:(NSString *)key
    expiredAfter:(NSInteger)duration
{
    //1. 将原始内存对象，包装成KObject对象
    KObject *object = [[KObject alloc] initWithData:value
                                    andLifeDuration:duration];
    
    //2. 判断是否在进行内存归档 或 清理
    if (self.archiving || self.cleaning) {
        
        //2.1 如果正在进行，直接将缓存写入磁盘
        NSString *filePath = [self.path stringByAppendingPathComponent:key];
        [object.data writeToFile:filePath atomically:YES];
        
    } else {
        
        //2.2 没有进行，将缓存保存到内存
        
        //先字典保存
        [self.objects setValue:object.data forKey:key];
        
        //内存大小累加
        self.size += [object size];
    }
    
    //3. 首先移除如果保存了当前的缓存key
    //稍后进行按照超时时间排序，重新确定缓存key的位置
    [self.keys removeObject:key];
    
    //4. 同步线程
    [self.lock lock];
    
    //5. key数组越后面的缓存项，超时时间越晚
    for (NSInteger i = [self.keys count] - 1; i >= 0; i --) {
        
        NSString *tmpKey = [self.keys objectAtIndex:i];
        
        KObject *leftObject = [self objectForKey:tmpKey];
        
        // 被遍历的缓存项超时时间 与 当前即将被缓存的项超时时间 大小
        if ([leftObject expiredTimestamp] <= [object expiredTimestamp]) {
            
            if (([self.keys count] - 1) == i) {
                
                //i == 最后一个位置
                [self.keys addObject:key];
                
            } else {
                
                //i == 中间的位置
                [self.keys insertObject:key atIndex:(i + 1)];
            }
            break;
        }
    }
    
    //6.
    [self.lock unlock];
    
    //7. 未找到位置，直接添加到第一个位置
    if (! [self.keys containsObject:key]) {
        [self.keys insertObject:key atIndex:0];
    }
    
    //8. 超过阈值，归档
    //此方法可能处于其他线程上，所以archiveData方法内部需要使用同步
    if ((! self.archiving)
        && 0 < ARCHIVING_THRESHOLD
        && ARCHIVING_THRESHOLD < self.size)
    {
        [self archiveData];
    }
}
```