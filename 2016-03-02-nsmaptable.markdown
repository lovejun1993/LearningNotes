---
layout: post
title: "NSMapTable"
date: 2016-03-02 11:44:37 +0800
comments: true
categories: 
---

###YYCache中看到使用了`NSMapTable`这个字典类，一直没用过这个类字典，真不知道有什么用。百度学习了下，记录...

***

###NSMapTable的头文件

```objc
NS_CLASS_AVAILABLE(10_5, 6_0)
@interface NSMapTable<KeyType, ObjectType> : NSObject <NSCopying, NSCoding, NSFastEnumeration>

- (instancetype)initWithKeyOptions:(NSPointerFunctionsOptions)keyOptions valueOptions:(NSPointerFunctionsOptions)valueOptions capacity:(NSUInteger)initialCapacity NS_DESIGNATED_INITIALIZER;
- (instancetype)initWithKeyPointerFunctions:(NSPointerFunctions *)keyFunctions valuePointerFunctions:(NSPointerFunctions *)valueFunctions capacity:(NSUInteger)initialCapacity NS_DESIGNATED_INITIALIZER;

+ (NSMapTable<KeyType, ObjectType> *)mapTableWithKeyOptions:(NSPointerFunctionsOptions)keyOptions valueOptions:(NSPointerFunctionsOptions)valueOptions;

#if (TARGET_OS_MAC && !(TARGET_OS_EMBEDDED || TARGET_OS_IPHONE)) || TARGET_OS_WIN32
+ (id)mapTableWithStrongToStrongObjects NS_DEPRECATED_MAC(10_5, 10_8);
+ (id)mapTableWithWeakToStrongObjects NS_DEPRECATED_MAC(10_5, 10_8);
+ (id)mapTableWithStrongToWeakObjects NS_DEPRECATED_MAC(10_5, 10_8);
+ (id)mapTableWithWeakToWeakObjects NS_DEPRECATED_MAC(10_5, 10_8);
#endif

+ (NSMapTable<KeyType, ObjectType> *)strongToStrongObjectsMapTable NS_AVAILABLE(10_8, 6_0);
+ (NSMapTable<KeyType, ObjectType> *)weakToStrongObjectsMapTable NS_AVAILABLE(10_8, 6_0); // entries are not necessarily purged right away when the weak key is reclaimed
+ (NSMapTable<KeyType, ObjectType> *)strongToWeakObjectsMapTable NS_AVAILABLE(10_8, 6_0);
+ (NSMapTable<KeyType, ObjectType> *)weakToWeakObjectsMapTable NS_AVAILABLE(10_8, 6_0); // entries are not necessarily purged right away when the weak key or object is reclaimed

/* return an NSPointerFunctions object reflecting the functions in use.  This is a new autoreleased object that can be subsequently modified and/or used directly in the creation of other pointer "collections". */
@property (readonly, copy) NSPointerFunctions *keyPointerFunctions;
@property (readonly, copy) NSPointerFunctions *valuePointerFunctions;

- (nullable ObjectType)objectForKey:(nullable KeyType)aKey;

- (void)removeObjectForKey:(nullable KeyType)aKey;
- (void)setObject:(nullable ObjectType)anObject forKey:(nullable KeyType)aKey;   // add/replace value (CFDictionarySetValue, NSMapInsert)

@property (readonly) NSUInteger count;

- (NSEnumerator<KeyType> *)keyEnumerator;
- (nullable NSEnumerator<ObjectType> *)objectEnumerator;

- (void)removeAllObjects;

- (NSDictionary<KeyType, ObjectType> *)dictionaryRepresentation;  // create a dictionary of contents
@end

```

- 该Api是iOS6之后使用的

- init初始化，传入指针引用类型

```
- (instancetype)initWithKeyOptions:(NSPointerFunctionsOptions)keyOptions valueOptions:(NSPointerFunctionsOptions)valueOptions capacity:(NSUInteger)initialCapacity NS_DESIGNATED_INITIALIZER;
- (instancetype)initWithKeyPointerFunctions:(NSPointerFunctions *)keyFunctions valuePointerFunctions:(NSPointerFunctions *)valueFunctions capacity:(NSUInteger)initialCapacity NS_DESIGNATED_INITIALIZER;
```

```
+ (NSMapTable<KeyType, ObjectType> *)mapTableWithKeyOptions:(NSPointerFunctionsOptions)keyOptions valueOptions:(NSPointerFunctionsOptions)valueOptions;
```

- 只能在Mac OS 使用的类方法初始化的Api

```
#if (TARGET_OS_MAC && !(TARGET_OS_EMBEDDED || TARGET_OS_IPHONE)) || TARGET_OS_WIN32
+ (id)mapTableWithStrongToStrongObjects NS_DEPRECATED_MAC(10_5, 10_8);
+ (id)mapTableWithWeakToStrongObjects NS_DEPRECATED_MAC(10_5, 10_8);
+ (id)mapTableWithStrongToWeakObjects NS_DEPRECATED_MAC(10_5, 10_8);
+ (id)mapTableWithWeakToWeakObjects NS_DEPRECATED_MAC(10_5, 10_8);
#endif
```

- iOS6 之后可以使用的类方法初始化Api，就不用使用前面的init方法去传入对应的指针引用类型

```
key指针引用类型 to value指针引用类型

+ (NSMapTable<KeyType, ObjectType> *)strongToStrongObjectsMapTable NS_AVAILABLE(10_8, 6_0);
+ (NSMapTable<KeyType, ObjectType> *)weakToStrongObjectsMapTable NS_AVAILABLE(10_8, 6_0); // entries are not necessarily purged right away when the weak key is reclaimed
+ (NSMapTable<KeyType, ObjectType> *)strongToWeakObjectsMapTable NS_AVAILABLE(10_8, 6_0);
+ (NSMapTable<KeyType, ObjectType> *)weakToWeakObjectsMapTable NS_AVAILABLE(10_8, 6_0); // entries are not necessarily purged right away when the weak key or object is reclaimed
```

- 暂时还没看有什么用，有时间看看

```
@property (readonly, copy) NSPointerFunctions *keyPointerFunctions;
@property (readonly, copy) NSPointerFunctions *valuePointerFunctions;
```

- 最后就是类型NSDictionary的Api方法

```
- (nullable ObjectType)objectForKey:(nullable KeyType)aKey;

- (void)removeObjectForKey:(nullable KeyType)aKey;
- (void)setObject:(nullable ObjectType)anObject forKey:(nullable KeyType)aKey;   // add/replace value (CFDictionarySetValue, NSMapInsert)

@property (readonly) NSUInteger count;

- (NSEnumerator<KeyType> *)keyEnumerator;
- (nullable NSEnumerator<ObjectType> *)objectEnumerator;

- (void)removeAllObjects;

- (NSDictionary<KeyType, ObjectType> *)dictionaryRepresentation;  // create a dictionary of contents
@end
```

###常用的指针引用类型枚举 NSPointerFunctionsOptions

- NSPointerFunctionsStrongMemory

创建了一个`retain/release`对象的集合，非常像常规的NSSet或NSArray

- NSPointerFunctionsWeakMemory

使用等价的`__weak`来存储对象并自动移除被销毁的对象

- NSPointerFunctionsCopyIn

对传入的Key/value进行copy

- NSPointerFunctionsObjectPersonality（默认）

对于isEqual:和hash比较的是 `-[对象 description]` 字符串值

- NSPointerFunctionsObjectPointerPersonality

对于isEqual:和hash比较的是 `指针` 地址

****

###先看下NSMapTable与NSDictionary/NSMutableDictionary的区别:

- NSMapTable
	- 没有可变/不可变之分，就是一个`可变`的
	- 对传入的key与value可以`自定义指定`指针引用方式
		- copy
		- strong
		- `weak` 这点很有用
			- 当key或者value被释放的时候，此entry会自动从NSMapTable中移除
	- 对传入的key与value`没有`类型限制
	- 既可以 `key-object 也可以 object-object`

- NSDictionary/NSMutableDictionary
	- 区分 不可变/可变 两种字典
	- 对传入的key与value默认做如下操作
		- `copy` 传入的key
		- `retain/strong` 强引用传入的value
	- 对传入的key与value有类型限制
		- key 必须是实现了 NSCopying协议的类对象
		- value 必须是Foundation 类对象
	- 只能够 `key-object`

****

###demo示例

```objc
//1. 初始化方式1: 指定对应的key为强引用，value为弱引用
NSMapTable *mapTable1 = [NSMapTable strongToWeakObjectsMapTable];
    
//2. 初始化方式2: 指定对应的key为强引用，value为弱引用
NSMapTable *mapTable2 = [[NSMapTable alloc] initWithKeyOptions:NSPointerFunctionsStrongMemory
                                                  valueOptions:NSPointerFunctionsWeakMemory
                                                      capacity:64];
    
//3. key - object
User *user1 = [User new];
user1.name = @"user_name_1111";
user1.address = @"user_address_1111";
    
[mapTable1 setObject:user1 forKey:@"object1"];
    
//4. object - object
User *user2 = [User new];
user2.name = @"user_name_1111";
user2.address = @"user_address_1111";
[mapTable2 setObject:user1 forKey:user2];
    
//5. 根据 key获取object（默认根据key的description值查询）
User *find1 = [mapTable1 objectForKey:@"object1"];
    
//6. 根据 object获取object（默认根据object的description值查询）
User *find2 = [mapTable2 objectForKey:user2];

//7. NSMapTable内部key-value转换成一个NSDictionary
//注意: 如果是object-object，那么作为key的object类，需要实现NSCopying协议
NSDictionary *dic1 = [mapTable1 dictionaryRepresentation];
NSDictionary *dic2 = [mapTable2 dictionaryRepresentation];
```

User实现NSCopying协议

```
- (id)copyWithZone:(NSZone *)zone {
    id obj = [[[self class] allocWithZone:zone] init];
    [obj setName:self.name];
    [obj setAddress:self.address];
    return obj;
}
```

断点fr v输出如下

```
(NSConcreteMapTable *) mapTable1 = 0x00007fbbf071ed30
(NSConcreteMapTable *) mapTable2 = 0x00007fbbf071ee60

(User *) user1 = 0x00007fbbf0721040
(User *) user2 = 0x00007fbbf0720920

(User *) find1 = 0x00007fbbf0721040
(User *) find2 = 0x00007fbbf0721040

(__NSDictionaryM *) dic1 = 0x00007fbbf0715330 1 key/value pair
(__NSDictionaryM *) dic2 = 0x00007fbbf071d880 1 key/value pair
```

po [dic1 description] 仔细看下dic1 

```
{
    object1 = "<User: 0x7fbbf0721040>";
}
```

po [dic2 description] 仔细看下dic2

```
{
    "<User: 0x7fbbf071f440>" = "<User: 0x7fbbf0721040>";
}
```