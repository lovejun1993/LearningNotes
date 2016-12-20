---
layout: post
title: "Kache、KQueue"
date: 2015-08-22 19:26:41 +0800
comments: true
categories: 
---

###KQueue对象

- 提供队列机制来缓存传入的对象
- 最终借助KHolder对象完成对象的缓存与读取

***

###KQueue.h 暴露的api

```objc
#import <Foundation/Foundation.h>

@class KHolder;

@interface KQueue : NSObject

/**
 *  队列的名字
 */
@property (assign, nonatomic)   NSString          *name;

/**
 *  队列最大缓存长度
 */
@property (assign, nonatomic)   NSInteger         size;


/**
 *  传入KHolder对象，完成对象的存储
 */
- (KQueue *)initWithHolder:(KHolder *)holder;

/**
 *  使用队列结构缓存对象，传入的对象类需要实现NSCoding协议具备归档
 */
- (void)push:(id<NSCoding>)obj;

/**
 *  出队头key对应的缓存对象
 */
- (id<NSCoding>)pop;

/**
 *  将保存的数据归档为字典返回出去
 */
- (NSDictionary *)serialize;

/**
 *  从传入的字典恢复数据
 */
- (void)unserializeFrom:(NSDictionary *)dict;

@end
```

****

###接下来开始看 KQueue.m ....

***

####匿名分类声明属性与方法


```objc
@interface KQueue ()

@property (strong, nonatomic)   KHolder               *holder;

/**
 *	当前缓存key的id值，这个offset是累加值
 */
@property (assign, nonatomic)   NSInteger             offset;

/**
 *	使用一个数组模拟队列结构
 */
@property (strong, atomic)      NSMutableArray        *queue;

/** 清理缓存 */
- (void)cleanExpiredObjects;

@end
```

####KQueue的初始化init函数

```objc
- (KQueue *)initWithHolder:(KHolder *)aHolder
{
    self = [super init];
    
    if (self)
    {
        //1. 保存Kache对象的持有的KHolder对象（内存缓存块）
        self.holder = aHolder;
        
        //2. 创建一个数组，模拟队列结构
        self.queue  = [[NSMutableArray alloc] init];
        
        //3. 默认的队列最大长度
        self.size   = KACHE_DEFAULT_QUEUE_SIZE;
        
        //4. 队列存放元素的id偏移值（递增）
        self.offset = 0;
    }
    
    return self;
}
```

####KQueue队列入队 缓存key

```objc
- (void)push:(id<NSCoding>)obj
{
    //1. 给传入的key生成一个id = QUEUE_当前Queue名字_offset值
    NSString *identifier = [NSString stringWithFormat:@"QUEUE_%@_%ld",
                      self.name,
                      self.offset];
    
    //2. 累加offset
    self.offset ++;
    
    //3. 使用identifier作为key，传入的对象为value
    [self.holder setValue:obj
                   forKey:identifier
             expiredAfter:0];

    //4. 清理超时的缓存项
    [self cleanExpiredObjects];
    
    //5. 判断当前内存缓存长度是否超过
    if ([self.queue count] >= self.size)
    {
        //如下处理缓存长度超过 self.size
        
        //5.1 移除队头第一个元素的缓存
        [self.holder removeObjectForKey:[self.queue objectAtIndex:0]];
        
        //5.2 从队列移除队头第一个元素
        [self.queue removeObjectAtIndex:0];
    }

    //6. 缓存key添加到queue数组
    [self.queue addObject:key];
}
```

- 对传入的缓存key生成一个唯一标识identidier
- KQueue内部数组管理identidier
- 使用KHolder内存缓存块存储 `identidier : 缓存key`

####清理超时的内存缓存

```objc
- (void)cleanExpiredObjects
{
    if (self.queue && 0 < [self.queue count])
    {
        //左 --> 右，遍历queue数组，遍历出所有的 identidier
        for (int i = 0; i < [self.queue count] - 1; i ++)
        {
            //1. identidier
            NSString *tmpKey = [self.queue objectAtIndex:i];

            //2. identidier 对应存储在KHolder中的 缓存key
            KObject *leftObject = [self.holder objectForKey:tmpKey];
            
            //3. 判断缓存项目是否已经超时
            if ([leftObject expiredTimestamp] < [KUtil nowTimestamp])
            {
                //缓存超时，只从KQueue维护的数组中移除
                //并没有从KHolder中移除缓存
                [self.queue removeObject:tmpKey];
                
            } else {
                
                break;
            }
        }
    }
}
```

####KQueue退出队头保存的key对应的缓存value（缓存对象）

```objc
- (id<NSCoding>)pop
{
    //判断queue数组是否存在identifier
    if (0 < [self.queue count])
    {
        //1. 取出queue数组保存的第一个 identifier
        NSString *key = [self.queue objectAtIndex:0];
        
        //2. 从queue数组移除
        [self.queue removeObjectAtIndex:0];
        
        //3. 从KHolder缓存快查询identifier对应的缓存项
        KObject *object = [self.holder objectForKey:key];
        
        
        //4. 从KHolder缓存快移除缓存项
        [self.holder removeObjectForKey:key];
        
        //5. 返回缓存项KObject保存的缓存key
        return [object value];
        
    } else {
        
        return nil;
    }
}
```

###组装KQueue对象数据成字典

```objc
- (NSDictionary *)serialize
{
    return @{
             @"size" : [NSString stringWithFormat:@"%ld", self.size],
             @"name" : self.name,
             @"queue" : self.queue,
             @"offset" : [NSString stringWithFormat:@"%ld", self.offset]
            };
}
```

###从传入的字典恢复KQueue对象数据

```objc
- (void)unserializeFrom:(NSDictionary *)dict
{
    if ([[dict allKeys] containsObject:@"size"]
        && [[dict allKeys] containsObject:@"name"]
        && [[dict allKeys] containsObject:@"queue"]
        && [[dict allKeys] containsObject:@"offset"])
    {
        self.size   = [[dict objectForKey:@"size"] intValue];
        self.name   = [NSString stringWithFormat:@"%@", [dict objectForKey:@"name"]];
        self.queue  = [[dict objectForKey:@"queue"] mutableCopy];
        self.offset = [[dict objectForKey:@"offset"] intValue];
    }
}
```

###KQueue使用demo一、通过使用Kache对象注册KQueue使用


```objc
//1.
Kache *kache = [[Kache alloc] initWithFiletoken:@"kache名字"];
    
//2.
[kache newQueueWithName:@"queue名字" size:10];
    
//3.
Person *person = [[Person alloc] init];
[kache pushValue:person toQueue:@"在Kache对象中注册的queue的名字"];
    
//4.
[kache popFromQueue:@"在Kache对象中注册的queue的名字"];
```

###KQueue使用demo二、直接使用KQueue对象

```objc
//1. 需要自己创建一个KHolder对象，作为最终的所有内存对象的缓存内存块
KHolder *holder = [[KHolder alloc] initWithToken:@"名字"];

//2.
KQueue *queue = [[KQueue alloc] initWithHolder:holder];

//3.
Person *p = [[Person alloc] init];
[queue push:p];

//4.
p = [queue pop];
```

两种用法的区别是，需不需要自己创建KHolder对象.

- 第一种，KQueue使用Kache的Kholder对象
- 第二种，需要自己创建一个KHolder对象作为缓存内存块