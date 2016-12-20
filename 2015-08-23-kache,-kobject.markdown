---
layout: post
title: "Kache、KObject"
date: 2015-08-23 19:26:51 +0800
comments: true
categories: 
---

###KObject管理被缓存的内存对象，以及缓存超时控制.

```objc
#import <Foundation/Foundation.h>

@interface KObject : NSObject

/**
 *  内存对象 --> 缓存项
 *
 *  @param data     内存对象
 *  @param duration 缓存存活时间
 */
- (KObject *)initWithData:(id<NSCoding>)data
          andLifeDuration:(NSInteger)duration;

/**
 *  缓存项 --> 内存对象
 *
 *  @param data 解档字典之后的NSData
 */
- (KObject *)initWithData:(NSData *)data;

/**
 *  获取KObject对象的归档字典对应的NSData
 */
- (NSData *)data;

/**
 *  获取KObject对象的归档字典中的 被缓存的内存对象
 */
- (id)value;

/**
 *  返回KObject缓存项的预期超时时间
 */
- (NSInteger)expiredTimestamp;

/**
 *  KObject缓存项是否已经超时
 */
- (BOOL)expired;

/**
 *  更新KObject缓存项的预期超时时间
 */
- (void)updateLifeDuration:(NSInteger)duration;

/**
 *  获取缓存项的大小
 */
- (NSUInteger)size;

@end
```

###实现文件中的匿名类

```objc
@interface KObject ()

/**
 *  保存被归档的缓存数据字典
 */
@property (strong, nonatomic)   NSData                  *data;

/**
 *  被解档后的数据字典，包含:
 *  1. 缓存对象 id<NSCoding> 类型
 *  2. 超时时间 NSString* 类型
 */
@property (strong, nonatomic)   NSMutableDictionary     *object;

@end
```

###内存对象 ---> 缓存项

```objc
@implementation KObject

- (KObject *)initWithData:(id<NSCoding>)aData andLifeDuration:(NSInteger)aDuration
{
    self = [super init];

    if (self) {
        
        //1. 确定缓存项的存活时间
        aDuration = (0 >= aDuration) ? KACHE_DEFAULT_LIFE_DURATION : aDuration;
        
        //2. 当前时间 + 存活时间 = 超时预期时间
        NSInteger expirateTime = [KUtil expiredTimestampForLife:aDuration];
        NSString * expirateTimeStr = [NSString stringWithFormat:@"%ld", expirateTime];
        
        //3. 组装数据成待归档字典
        //组装参数一、传入的要缓存的内存对象
        //组装参数二、缓存超时预期时间
        NSMutableDictionary *archiveDict = [[NSMutableDictionary alloc] initWithObjectsAndKeys:
                                            aData, @"data",
                                            expirateTimeStr, @"life",
                                            nil];
        
        //3. 将如上字典归档成NSData，稍后写入磁盘文件
        self.data = [NSKeyedArchiver archivedDataWithRootObject:archiveDict];
        
    }
    
    return self;
}

@end
```

###缓存项（在磁盘文件） ---> 内存对象

```objc
- (KObject *)initWithData:(NSData *)data
{
    self = [super init];
    
    if (self) {
        
        self.data = data;
        
        return self;
    }
    
    return nil;
}
```

- 需要先从磁盘文件读取成NSData
- 然后调用这个初始化方法


***

###获取KObject缓存项中的被缓存的内存对象

```objc
- (id)value
{
    //1. 先填充字典保存解档后的缓存数据字典
    if (nil == self.object) {
        
        //解档
        self.object = [NSKeyedUnarchiver unarchiveObjectWithData:self.data];
    }
    
    if ([[self.object allKeys] containsObject:KCH_OBJ_DATA]
        && [self.object objectForKey:KCH_OBJ_DATA])
    {
        //从数据字典获取缓存key对应的object
        return [self.object objectForKey:KCH_OBJ_DATA];
    }

    return nil;
}
```

###获取KObject缓存项的超时时间

```objc
- (NSInteger)expiredTimestamp
{
    if (nil == self.object) {
        self.object = [NSKeyedUnarchiver unarchiveObjectWithData:self.data];
    }
    
    return [[self.object objectForKey:KCH_OBJ_LIFE] intValue];
}
```

###获取KObject缓存项是否超时

```objc
- (BOOL)expired
{
    if (nil == self.object) {
        self.object = [NSKeyedUnarchiver unarchiveObjectWithData:self.data];
    }

    if ([KUtil nowTimestamp] < [self expiredTimestamp]) {
        return NO;
    }

    return YES;
}
```

###修改KObject缓存项的预期超时时间

```objc
- (void)updateLifeDuration:(NSInteger)aDuration
{
    //1. 填充数据字典
    if (nil == self.object) {
        self.object = [NSKeyedUnarchiver unarchiveObjectWithData:self.data];
    }

    //2. 修正要设置的超时间隔
    aDuration = (0 >= aDuration) ? KACHE_DEFAULT_LIFE_DURATION : aDuration;

    //3. 得到新的预期超时时间
    NSString *expirateTime = [NSString stringWithFormat:@"%ld", [KUtil expiredTimestampForLife:aDuration]];
    
    //4. 给数据字典，设置新的超时预期时间
    [self.object setValue:expirateTime forKey:KCH_OBJ_LIFE];
}
```

###获取KObject缓存项的缓存数据NSData，以及缓存数据NSData长度

```objc
- (NSData *)data
{
    return _data;
}

- (NSUInteger)size
{
    return _data.length;
}
```

###启发:可以不用实现NSCoding来实现归档，也可以将部分数据使用一个字典来组装，最后归档成NSData，再写入磁盘文件.