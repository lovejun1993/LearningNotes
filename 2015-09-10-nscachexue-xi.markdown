---
layout: post
title: "NSCache学习"
date: 2015-09-10 23:31:00 +0800
comments: true
categories: 
---

###之前看了一下NSURLCache，然后发现还有一个叫做NSCache的，感觉有点一样，但是不太清楚是做什么的，来学习下吧。

***

###首先、NSCache的用法与NSDictionary很相似，提供key，value的存储。

###不一样的是NSCache在系统【内存吃紧】的时候会做【自动释放】.

***

先看一下NSCache.h，看看有哪些东西？

```
@class NSString;

//有一个定义的回调代理函数接口
@protocol NSCacheDelegate;
```

看一下NSCacheDelegate定义哪些方法.

```
缓存将要删除对象时调用 不能在此方法中修改缓存

@protocol NSCacheDelegate <NSObject>
@optional
- (void)cache:(NSCache *)cache willEvictObject:(id)obj;
@end
```

再看下NSCache的实例属性

```
@property (copy) NSString *name;
```

```
@property (assign) id<NSCacheDelegate> delegate;
```

```
设置最大的缓存数据总大小（默认值是0，表示无限制）

@property NSUInteger totalCostLimit;	
```

```
设置能够缓存键值对的最大个数（默认值是0，表示无限制）

@property NSUInteger countLimit;
```

注意:
NSCache从 缓存【数据总大小+数据总个数】来确定是否清理内存。


```
设置当接受到内存超过限制的时候，是否自动回收释放被删除数据。

@property BOOL evictsObjectsWithDiscardedContent;
```

再看下NSCache的实例方法（只有实例方法）

```
- (id)objectForKey:(id)key;
- (void)setObject:(id)obj forKey:(id)key; // 0 cost
- (void)setObject:(id)obj forKey:(id)key cost:(NSUInteger)g;
- (void)removeObjectForKey:(id)key;
```

(从如上Api可以看出，NSCache好像就是一个NSMutableDictionary字典形式的东西，按照key-value的形式存放键值对。)

最后一个清除所有NSCache实例保存的键值对的方法。

```
- (void)removeAllObjects;
```


小结NSCache的结构:

1. NSCache 是苹果官方提供的缓存类，用法与 NSMutableDictionary 的用法很相似。

2. NSCache 在系统内存很低时，会自动释放一些对象。

3. NSCache 是线程安全的，在多线程操作中，不需要对 Cache 加锁。

4. NSCache 的 Key 只是做强引用，不需要实现 NSCopying 协议。

***

###使用很简单

创建一个实例.

```
NSCache *cache = [[NSCache alloc] init];
```

设置NSCache实例最大存放10个键值对.

```
cache.countLimit = 10;
```

设置NSCache实例最大存储10M的大小数据.

```
cache. totalCostLimit = 10 * 1024 *1024;
```

设置NSCache实例的回调代理对象.

```
cache.delegate = self;
```

将键值对存放到NSCache实例.

```
for (int i = 0; i < 20; ++i) {

    NSString *str = [NSString stringWithFormat:@"hello - %04d", i];
 
    [self.cache setObject:str forKey:@(i)];
}
```

取出NSCache的键值对【NSCache 没有提供遍历的方法，只支持用 key 来取值】


```
for (int i = 0; i < 20; ++i) {
    NSLog(@"缓存中----->%@", [self.cache objectForKey:@(i)]);
}
```

实现代理方法

```
- (void)cache:(NSCache *)cache willEvictObject:(id)obj {
 	NSLog(@"将要被删除的对象: %@\n", obj);   
}
```

当超多最大成本限制的时候，会先清除缓存中的一条数据，再存入一条新的数据

***

####总结NSCache:

1. NSCache是使用内存来缓存键值对.

2. 解决在使用大量图片的app中，每次都从文件系统里面读取文件会造成卡顿现象。

```
第一步、先从NSCache里面查找数据图片.
第二步、如果从NSCache实例没有找到图片，就去文件系统查找图片.
第三步、如果文件系统也没有找到，就去读取网络URL图片.
第四步、当第三步读取到图片NSData之后，保存到本地文件系统.
第五步、继续使用NSCache实例缓存图片的NSData.
第六步、下一次直接从NSCache内存缓存读取图片.
```

```
内存 -> 文件系统  -> 网络图片
```