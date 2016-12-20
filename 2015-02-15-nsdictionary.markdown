---
layout: post
title: "NSDictionary"
date: 2015-02-15 13:39:54 +0800
comments: true
categories: 
---

###在YYModel中，看到 NSDictionary 使用 `数组` 作为key 进行存取

- -[NSObject hash]
- -[NSObject description]
- NSDictionary 是按照 `key的值`，而不是 `key的地址` 进行运算


###作为字典key的要求

```
NSMutableDictionary *mdict = [@{} mutableCopy];
```

```
[mdict setObject:(nonnull id) forKey:(nonnull id<NSCopying>)];
```

```
id value = [mdict objectForKey:(nonnull id)];
```

- 可以看到能够作为 `字典Key`实例，必须实现`NSCopying 协议`的类对象.

- 那么常见实现了`NSCopying`协议的类
	- NSString、NSMutableString
	- NSArray、NSMutableArray
	- NSDictionary、NSMutableDictionary
	- NSSet、NSMutableSet
	- **NSObject没有实现NSCopying协议的**

那么如上的数据结构类型的对象，都可以作为 `字典key`.

***

###一个字符串 的 `内存地址 是唯一` 的

```
NSString *STR1 = @"111";
NSString *STR2 = @"111";
NSString *STR3 = @"111";
NSString *STR4 = @"112";
```

断点后输出结果

```
(__NSCFConstantString *) STR1 = 0x0000000105d742e0 @"111"
(__NSCFConstantString *) STR2 = 0x0000000105d742e0 @"111"
(__NSCFConstantString *) STR3 = 0x0000000105d742e0 @"111"
(__NSCFConstantString *) STR4 = 0x0000000105d74300 @"112"
```


可以得到如下:

- `字符串 @"111"` 只开辟了 `一块内存块` 地址为: 0x0000000105d74`2e0`

- `字符串 @"112"` 开辟了 另一块不同的内存块 0x0000000105d74`300`

- 指针STR1、STR2、STR3，只是指向起始地址 `0x0000000105d742e0 ` 这一个内存块中的 内容 `@"111"`，并不是指向不同的内存块

###数组的内存地址是不一样的，尽管数组内元素长度、内容、顺序都一致

```
NSArray *array1 = @[@"111", @"122", @"133"];
NSArray *array2 = @[@"111", @"122", @"133"];
```

断点后输出结果

```
(__NSArrayI *) array1 = 0x00007ff442c276d0 @"3 objects"
(__NSArrayI *) array2 = 0x00007ff442c12310 @"3 objects"
```

###字典同数组

```
NSDictionary *dict1 = @{
                        @"name" : @"name"
                        };
    
NSDictionary *dict2 = @{
                        @"name" : @"name"
                        };
```

断点后输出结果

```
(__NSDictionaryI *) dict1 = 0x00007ff442c297c0 1 key/value pair
(__NSDictionaryI *) dict2 = 0x00007ff442c29800 1 key/value pair
```

###可以得到 数组、字典、字符串的区别

- NSArray和NSDictionary 不像 `NSString只要内容一致只会开辟一块内存`
- NSArray和NSDictionary 即使 `内部元素内容一致`，也会 开辟 `不同` 的内存块

- 复杂 类型都是如此（如: 自定义类的对象）

***

###看下使用`数组`作为字典key的代码示例

```
//数组 1
NSArray *array1 = @[@"111", @"222", @"333"];
    
//数组 2
NSArray *array2 = @[@"111", @"222", @"333"];
    
NSMutableDictionary *mdict = [@{} mutableCopy];
    
//使用 数组1 作为 key 存取
mdict[array1] = @"name";
    
//使用 数组2 作为key 取值
id value = mdict[array2];
```

输出结果

```
(__NSArrayI *) array1 = 0x00007f87a1c9a4f0 @"3 objects"
(__NSArrayI *) array2 = 0x00007f87a1c9a520 @"3 objects"
(__NSDictionaryM *) mdict = 0x00007f87a1c9a550 1 key/value pair
(__NSCFConstantString *) value = 0x000000010ff1f900 @"name"
```

可以看到 

- `array 1` 与 `array 2` 的内存地址 是不一样的
- 用`array 1`做为key存值
- 用`array 2`做为key也能够取到值.
- 所以猜测 字典key 是根据 `对象的 值内容`，而不是 `对象的 地址`
	- **因为array1与array2数组元素` 值（内容）、长度、顺序`都一致**

***

###验证 `字典` 是根据 `key所指对象` 的 `值内容`，而不是 `地址`

- 对如下正常代码做三种情况的修改

```
//数组 1
NSArray *array1 = @[@"111", @"222", @"333"];
    
//数组 2
NSArray *array2 = @[@"111", @"222", @"333"];
    
NSMutableDictionary *mdict = [@{} mutableCopy];
    
//使用 数组1 作为 key 存取
mdict[array1] = @"name";
    
//使用 数组2 作为key 取值
id value = mdict[array2];
```

```
(__NSArrayI *) array1 = 0x00007f99a850dce0 @"3 objects"
(__NSArrayI *) array2 = 0x00007f99a85675d0 @"3 objects"
(__NSDictionaryM *) mdict = 0x00007f99a8568460 1 key/value pair
(__NSCFConstantString *) value = 0x0000000103d64900 @"name"
```

能取到值

- 是array1与array2数组 `长度不一致`

```
//数组 1
NSArray *array1 = @[@"111", @"222", @"333", @"444"];

//数组 2
NSArray *array2 = @[@"111", @"222", @"333"];

NSMutableDictionary *mdict = [@{} mutableCopy];

//使用 数组1 作为 key 存取
mdict[array1] = @"name";

//使用 数组2 作为key 取值
id value = mdict[array2];
```

```
(__NSArrayI *) array1 = 0x00007f92b84ac460 @"4 objects"
(__NSArrayI *) array2 = 0x00007f92b840a900 @"3 objects"
(__NSDictionaryM *) mdict = 0x00007f92b84bcb90 1 key/value pair
(id) value = nil
```

不能取到值

- 是array1与array2数组 `循序不一致`

```
//数组 1
NSArray *array1 = @[@"333", @"222", @"111"];

//数组 2
NSArray *array2 = @[@"111", @"222", @"333"];

NSMutableDictionary *mdict = [@{} mutableCopy];

//使用 数组1 作为 key 存取
mdict[array1] = @"name";

//使用 数组2 作为key 取值
id value = mdict[array2];
```

```
(__NSArrayI *) array1 = 0x00007ffc815acf70 @"3 objects"
(__NSArrayI *) array2 = 0x00007ffc815b0850 @"3 objects"
(__NSDictionaryM *) mdict = 0x00007ffc815aa220 1 key/value pair
(id) value = nil
```

不能取到值

- 是array1与array2数组 `内容不一致`

```
//数组 1
NSArray *array1 = @[@"222", @"333", @"444"];

//数组 2
NSArray *array2 = @[@"111", @"222", @"333"];

NSMutableDictionary *mdict = [@{} mutableCopy];

//使用 数组1 作为 key 存取
mdict[array1] = @"name";

//使用 数组2 作为key 取值
id value = mdict[array2];
```

```
(__NSArrayI *) array1 = 0x00007f9c50769e10 @"3 objects"
(__NSArrayI *) array2 = 0x00007f9c50769e40 @"3 objects"
(__NSDictionaryM *) mdict = 0x00007f9c50769e70 1 key/value pair
(id) value = nil
```

不能取到值


####所以说，字典key 是根据 `值内容` 来进行存取

***

###猜测一、上面说的`值内容` 是借助 `对象的hash值` 来表示

####hash值属性定义在 `协议NSObject`

```objc
@protocol NSObject

- (BOOL)isEqual:(id)object;

/**
	对象的hash值属性
*/
@property (readonly) NSUInteger hash;

@property (readonly) Class superclass;
- (Class)class OBJC_SWIFT_UNAVAILABLE("use 'anObject.dynamicType' instead");
- (instancetype)self;

- (id)performSelector:(SEL)aSelector;
- (id)performSelector:(SEL)aSelector withObject:(id)object;
- (id)performSelector:(SEL)aSelector withObject:(id)object1 withObject:(id)object2;

- (BOOL)isProxy;

- (BOOL)isKindOfClass:(Class)aClass;
- (BOOL)isMemberOfClass:(Class)aClass;
- (BOOL)conformsToProtocol:(Protocol *)aProtocol;

- (BOOL)respondsToSelector:(SEL)aSelector;

- (instancetype)retain OBJC_ARC_UNAVAILABLE;
- (oneway void)release OBJC_ARC_UNAVAILABLE;
- (instancetype)autorelease OBJC_ARC_UNAVAILABLE;
- (NSUInteger)retainCount OBJC_ARC_UNAVAILABLE;

- (struct _NSZone *)zone OBJC_ARC_UNAVAILABLE;

/**
	对象的描述值属性
 */
@property (readonly, copy) NSString *description;


@optional
@property (readonly, copy) NSString *debugDescription;

@end
```

比较有用的两个属性

- hash
- description

可以用来 `区分两个不同对象` 的 唯一标识符


####NSObject类 实现了 NSObject协议，自然就实现了属性的 `setter方法与getter方法`

- NSString 继承自 NSObject
- NSArray 继承自 NSObject
- NSDictionary 继承自 NSObject
- ...等等其他常用类

所以基本我们日常用到的所有类，都有获取如下值的方法

- hash值
- description值



####以NSArray的hash值为例

```objc
//数组 1
NSArray *array1 = @[@"111", @"222", @"333"];

//数组 2
NSArray *array2 = @[@"111", @"222", @"333"];
    
//数组1 的hash值
NSUInteger hash1 = [array1 hash];

//数组2 的hash值
NSUInteger hash2 = [array2 hash];
```

输出结果

```
(objc_property_t) property = 0x0000000103bc72f8
(__NSArrayI *) array1 = 0x00007fe431d22440 @"3 objects"
(__NSArrayI *) array2 = 0x00007fe431d2b4f0 @"3 objects"
(NSUInteger) hash1 = 3
(NSUInteger) hash2 = 3
```

然后让两个数组稍加修改再看看


```objc
//数组 1
NSArray *array1 = @[@"dawdwd", @"dawdawd", @"dawdawd"];

//数组 2
NSArray *array2 = @[@"111", @"222", @"333"];
    
//数组1 的hash值
NSUInteger hash1 = [array1 hash];

//数组2 的hash值
NSUInteger hash2 = [array2 hash];
```

输出结果

```
(__NSArrayI *) array1 = 0x00007fb78a416150 @"3 objects"
(__NSArrayI *) array2 = 0x00007fb78a404f90 @"3 objects"
(NSUInteger) hash1 = 3
(NSUInteger) hash2 = 3
```

```objc
//数组 1
NSArray *array1 = @[@"dawdwd", @"dawdawd", @"dawdawd", @"dawdawd"];

//数组 2
NSArray *array2 = @[@"111", @"222", @"333"];
    
//数组1 的hash值
NSUInteger hash1 = [array1 hash];

//数组2 的hash值
NSUInteger hash2 = [array2 hash];
```

```
(__NSArrayI *) array1 = 0x00007fb95b6089c0 @"4 objects"
(__NSArrayI *) array2 = 0x00007fb95b66f310 @"3 objects"
(NSUInteger) hash1 = 4
(NSUInteger) hash2 = 3
```

可以得到

- NSArray的 `hash属性的getter方法` 只是简单的
	- 统计 数组的长度
	- 将 长度数字 转换为 数值
	- 然后返回

- 但是两个 `长度一致，但是 内容不一致 数组` 的 `hash值` 确是一样的
	- 所以肯定不可能按照 数组的hash值 `来区分数组 内容 是否相同`

***

###猜测二、上面说的`值内容` 是借助 `descriptions值` 来表示

- eg1. 

```
//数组 1
NSArray *array1 = @[@"dawdwd", @"dawdawd", @"dawdawd", @"dawdawd"];

//数组 2
NSArray *array2 = @[@"111", @"222", @"333"];
    
NSString *des1 = [array1 description];
NSString *des2 = [array2 description];
    
BOOL res = [des1 isEqualToString:des2];
```

输出

```
(__NSArrayI *) array1 = 0x00007fe4cad04df0 @"4 objects"
(__NSArrayI *) array2 = 0x00007fe4cad7a370 @"3 objects"
(__NSCFString *) des1 = 0x00007fe4cad0d3d0 @"(\n    dawdwd,\n    dawdawd,\n    dawdawd,\n    dawdawd\n)"
(__NSCFString *) des2 = 0x00007fe4cad016d0 @"(\n    111,\n    222,\n    333\n)"
(BOOL) res = NO
```

- eg2.

```
//数组 1
NSArray *array1 = @[@"dawdwd", @"dawdawd", @"dawdawd"];

//数组 2
NSArray *array2 = @[@"111", @"222", @"333"];
    
NSString *des1 = [array1 description];
NSString *des2 = [array2 description];
    
BOOL res = [des1 isEqualToString:des2];
``` 

```
(__NSArrayI *) array1 = 0x00007ff90bf196e0 @"3 objects"
(__NSArrayI *) array2 = 0x00007ff90bf18950 @"3 objects"
(__NSCFString *) des1 = 0x00007ff90bf1f5d0 @"(\n    dawdwd,\n    dawdawd,\n    dawdawd\n)"
(__NSCFString *) des2 = 0x00007ff90bf1f610 @"(\n    111,\n    222,\n    333\n)"
(BOOL) res = NO
```


- eg3. 

```
//数组 1
NSArray *array1 = @[@"111", @"222", @"dawdawd"];

//数组 2
NSArray *array2 = @[@"111", @"222", @"333"];
    
NSString *des1 = [array1 description];
NSString *des2 = [array2 description];
    
BOOL res = [des1 isEqualToString:des2];
```

```
(__NSArrayI *) array1 = 0x00007f838251dd90 @"3 objects"
(__NSArrayI *) array2 = 0x00007f838250f000 @"3 objects"
(__NSCFString *) des1 = 0x00007f838250f060 @"(\n    111,\n    222,\n    dawdawd\n)"
(__NSCFString *) des2 = 0x00007f838258f090 @"(\n    111,\n    222,\n    333\n)"
(BOOL) res = NO
```

- eg4. 

```
//数组 1
NSArray *array1 = @[@"111", @"222", @"333"];

//数组 2
NSArray *array2 = @[@"111", @"222", @"333"];
    
NSString *des1 = [array1 description];
NSString *des2 = [array2 description];
    
BOOL res = [des1 isEqualToString:des2];
```

```
(__NSArrayI *) array1 = 0x00007ff8e9714940 @"3 objects"
(__NSArrayI *) array2 = 0x00007ff8e97100a0 @"3 objects"
(__NSCFString *) des1 = 0x00007ff8e973c460 @"(\n    111,\n    222,\n    333\n)"
(__NSCFString *) des2 = 0x00007ff8e973c4a0 @"(\n    111,\n    222,\n    333\n)"
(BOOL) res = YES
```

- description值，最终返回的是一个字符串，且能够识别 数据一致内存地址不一致的两个对象
- 所以猜测，字典可能是根据 `NSArray的description值`，进行存取数据

####当数组包含复杂对象类型是否还是能通过description值来区分

- Person类

```
#import <Foundation/Foundation.h>
#import "User.h"

@interface Person : NSObject

@property (nonatomic, copy) NSString *name;

@property (nonatomic, strong) NSArray *users;

@property (nonatomic, strong) User *user;


@end
```

```
#import "Person.h"

@implementation Person

@end
```

```
#import <Foundation/Foundation.h>

@interface User : NSObject

@property (nonatomic, copy) NSString *name;

@property (nonatomic, copy) NSString *address;

@end
```

- User类

```
#import "User.h"

@implementation User

@end
```

- 全部内容一致的情况

```
Person *p1 = [Person new];
Person *p2 = [Person new];
Person *p3 = [Person new];

NSArray *array1 = @[p1, p2, p3];
NSArray *array2 = @[p1, p2, p3];

NSString *des1 = [array1 description];
NSString *des2 = [array2 description];

BOOL res = [des1 isEqualToString:des2];
```

```
(Person *) p1 = 0x00007faaa3fe57e0
(Person *) p2 = 0x00007faaa3fe5850
(Person *) p3 = 0x00007faaa3fe5870
(__NSArrayI *) array1 = 0x00007faaa3fe5890 @"3 objects"
(__NSArrayI *) array2 = 0x00007faaa3fe58c0 @"3 objects"
(__NSCFString *) des1 = 0x00007faaa3fe5a50 @"(\n    \"<Person: 0x7faaa3fe57e0>\",\n    \"<Person: 0x7faaa3fe5850>\",\n    \"<Person: 0x7faaa3fe5870>\"\n)"
(__NSCFString *) des2 = 0x00007faaa3fe5c60 @"(\n    \"<Person: 0x7faaa3fe57e0>\",\n    \"<Person: 0x7faaa3fe5850>\",\n    \"<Person: 0x7faaa3fe5870>\"\n)"
(BOOL) res = YES
```

- 数据内容 长度不一致

```
Person *p1 = [Person new];
Person *p2 = [Person new];
Person *p3 = [Person new];

NSArray *array1 = @[p1, p2];
NSArray *array2 = @[p1, p2, p3];

NSString *des1 = [array1 description];
NSString *des2 = [array2 description];

BOOL res = [des1 isEqualToString:des2];
```

```
(__NSArrayI *) array1 = 0x00007fe9e1f26660 @"2 objects"
(__NSArrayI *) array2 = 0x00007fe9e1f24200 @"3 objects"
(__NSCFString *) des1 = 0x00007fe9e1f22e60 @"(\n    \"<Person: 0x7fe9e1f03130>\",\n    \"<Person: 0x7fe9e1f26a80>\"\n)"
(__NSCFString *) des2 = 0x00007fe9e1f01d40 @"(\n    \"<Person: 0x7fe9e1f03130>\",\n    \"<Person: 0x7fe9e1f26a80>\",\n    \"<Person: 0x7fe9e1f2dc40>\"\n)"
(BOOL) res = NO
```

- 长度一致，顺序不一致

```
Person *p1 = [Person new];
Person *p2 = [Person new];
Person *p3 = [Person new];

NSArray *array1 = @[p1, p3, p2];
NSArray *array2 = @[p1, p2, p3];

NSString *des1 = [array1 description];
NSString *des2 = [array2 description];

BOOL res = [des1 isEqualToString:des2];
```

```
(__NSArrayI *) array1 = 0x00007fdcc94203f0 @"3 objects"
(__NSArrayI *) array2 = 0x00007fdcc9481a40 @"3 objects"
(__NSCFString *) des1 = 0x00007fdcc941b990 @"(\n    \"<Person: 0x7fdcc942e4c0>\",\n    \"<Person: 0x7fdcc942e190>\",\n    \"<Person: 0x7fdcc9431650>\"\n)"
(__NSCFString *) des2 = 0x00007fdcc94804c0 @"(\n    \"<Person: 0x7fdcc942e4c0>\",\n    \"<Person: 0x7fdcc9431650>\",\n    \"<Person: 0x7fdcc942e190>\"\n)"
(BOOL) res = NO
```

- 观察 `数组对象的description值` 组织形式:
	- 首先是一个字符串
	- 由数组中 三个 `Person对象的description值` 拼接而成

```
@"(\n    \"<Person: 0x7fdcc942e4c0>\",\n    \"<Person: 0x7fdcc942e190>\",\n    \"<Person: 0x7fdcc9431650>\"\n)"
```

- 单独输出一个Person对象desceiption值

```
<Person: 0x7fad80c10020>
```

就是这样两边尖角的字符串，右侧的对象地址是唯一的.


####总之，数组Array 能通过 description值 来区分，`不同数组` 只需要

- 数组长度一致
- 数组中的每一个index对应的对象都 `是同一个对象`
	- index对应的对象的description值，才能对上号

那么此时 不同数组对象的description值 就会相同，也就能根据`数组的值内容`来区分是否一样.

****

###但通常我们自定义的类型中，成员变量类型不仅仅只有字符串，希望使用 `hash值` 或 `description值`，来更好的区分地址不同但是内容数据一样的两个对象.

####先看看字符串的hash值

- 两个不同字符串的hash值

```
NSString *str1 = @"1111";
NSString *str2 = @"dawdawdhawjdaw8d78dawddawdawdhawjdaw8d";
    
NSUInteger hash1 = [str1 hash];
NSUInteger hash2 = [str2 hash];
```

```
(__NSCFConstantString *) str1 = 0x000000010d1772e0 @"1111"
(__NSCFConstantString *) str2 = 0x000000010d177300 @"dawdawdhawjdaw8d78dawddawdawdhawjdaw8d78dawddawdawdhawjdaw8d78dawddawdawdhawjdaw8d78dawd"
(NSUInteger) hash1 = 18785280840
(NSUInteger) hash2 = 1191690051463814668
```

- 两个相同字符串的hash值

```
NSString *str1 = @"1111";
NSString *str2 = @"1111";
    
NSUInteger hash1 = [str1 hash];
NSUInteger hash2 = [str2 hash];
```

```
(__NSCFConstantString *) str1 = 0x000000010ff9d2e0 @"1111"
(__NSCFConstantString *) str2 = 0x000000010ff9d2e0 @"1111"
(NSUInteger) hash1 = 18785280840
(NSUInteger) hash2 = 18785280840
```

####举例，在网络请求类库中，需要将一个`Request url` 对应的 `responseJSON`缓存到`磁盘文件`

- 就必须要保证 磁盘文件的名字必须是唯一的
- 且需要多次重复计算，得到是同样的标识（如: MD5算法）
- 那么就可以重写NSObject实例的 hash方法，来获得一个YTKRequest对象的唯一hash值，作为文件名

####YTKRequest来生成请求对应的磁盘缓存文件名的方法

```
- (NSString *)cacheFileName {

	//1.
    NSString *requestUrl = [self requestUrl];
    
    //2.
    NSString *baseUrl = [YTKNetworkConfig sharedInstance].baseUrl;
    
    //3.
    id argument = [self cacheFileNameFilterForRequestArgument:[self requestArgument]];
    
    //4.
    NSString *requestInfo = [NSString stringWithFormat:@"Method:%ld Host:%@ Url:%@ Argument:%@ AppVersion:%@ Sensitive:%@",
                                                        (long)[self requestMethod], baseUrl, requestUrl,
                                                        argument, [YTKNetworkPrivate appVersionString], [self cacheSensitiveData]];
                                                        
	//5. 对字符串使用MD5算法，得到文件名
    NSString *cacheFileName = [YTKNetworkPrivate md5StringFromString:requestInfo];
    return cacheFileName;
}
```

####如果使用重写YTKRequest实例的hash方法，来生成请求对应的磁盘缓存文件名


- 第一步、重写hash值属性getter方法

```objc
//获取对象的hash值
- (NSUInteger)hash {
    
    //1. 请求路径path
    NSString *requestUrl = [self requestUrl];
    
    //2. 请求Host
    NSString *baseUrl = [YTKNetworkConfig sharedInstance].baseUrl;
    
    //3. 请求参数
    id argument = [self cacheFileNameFilterForRequestArgument:[self requestArgument]];
    
    //4. 根据前面三个数据得到一个字符串内容
    NSString *requestInfo = [NSString stringWithFormat:@"Method:%ld Host:%@ Url:%@ Argument:%@ AppVersion:%@ Sensitive:%@",
                             (long)[self requestMethod], baseUrl, requestUrl,
                             argument, [YTKNetworkPrivate appVersionString], [self cacheSensitiveData]];
    
    //6. 对字符串进行hash运算
    NSUInteger hashValue = [requestInfo hash];
    
    return hashValue;
}
```

- 第二步、使用hash值得到缓存文件名

```
- (NSString *)cacheFileName {
    
    //1. 得到当前对象的hash值（唯一的）
    NSUInteger hashValue = [self hash];
    
    //2. 将hash整数值转换为字符串值，就是文件名
    NSString *cacheFileName = [NSString stringWithFormat:@"%lu", hashValue];
    
    return cacheFileName;
}
```

- 相同字符串的hash值一定相同，且永远固定一致
- 不同字符串的hash值，一定不相同

***

###再看一个嵌套类型类，重写description方法的小例子

- Person类

```
#import <Foundation/Foundation.h>
#import "User.h"

@interface Person : NSObject
@property (nonatomic, copy) NSString *name;

@property (nonatomic, assign) NSInteger age;

@property (nonatomic, strong) User *user;

@end
```

```
#import "Person.h"

@implementation Person

- (NSString *)description {
    NSString *msg = [NSString stringWithFormat:@"%@%@", @"基础属性",[self.user description]];
    return msg;
}

@end
```

- User类

```
#import <Foundation/Foundation.h>

@interface User : NSObject

@property (nonatomic, copy) NSString *name;

@property (nonatomic, copy) NSString *address;

@end
```

```
#import "User.h"

@implementation User

- (NSString *)description {
    NSString *msg = [NSString stringWithFormat:@"%@.%@", self.name, self.address];
    return msg;
}

@end
```

***

###字典崩溃

```
NSMutableDictionary *dict = [NSMutableDictionary new];

//会崩溃一
NSString *key = nil;
[dict setObject:[NSObject new] forKey:key];

//会崩溃二
[dict setObject:nil forKey:@"key"];

//会崩溃三
[dict setObject:NULL forKey:@"key"];
```

```
//不会崩溃
[dict setObject:(id)kCFNull forKey:@"dawd"];
```