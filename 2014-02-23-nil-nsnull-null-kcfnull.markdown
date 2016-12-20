---
layout: post
title: "nil NSNull NULL kCFNull"
date: 2014-02-23 16:33:08 +0800
comments: true
categories: 
---

- NULL

c语法中的指针

```
char *p = NULL;
```

- nil

`ObjC 对象`的字面`空值`，对应 `id` 类型的对象

```
NSString *someString = nil;
NSURL *someURL = nil;
id someObject = nil;
```

- Nil

`ObjC 类`类型的书面`空值`，对应 `Class` 类型对象

```
Class someClass = Nil;
Class anotherClass = [NSString class];
```

- [NSNull null]

	- 用于表示`集合 NSArray/NSSet/NSDictionary`中值为`空值对象`
	- 因为 `nil` 被用来用为`集合结束`的标志，所以 nil 不能存储在 Foundation 集合里

		```
		NSArray *array = [NSArray arrayWithObjects:@"one", @"two", nil];
		```
	- nil 不能出现在 集合对象中

		```
		// 错误的使用
	　　NSMutableDictionary *dict = [NSMutableDictionary dictionary];
	　　[dict setObject:nil forKey:@"someKey"];
	　　
	　　// 正确的使用
	　　NSMutableDictionary *dict = [NSMutableDictionary dictionary];
	　　[dict setObject:[NSNull null] forKey:@"someKey"];
		```
		
		也就是说，集合对象如果要设置一个空值对象，必须要设置`[NSNull null]`得到的单例对象.
		
- kCFNull

是一个宏定义，其实就是NSNull的单例

```
const CFNullRef kCFNull;	// the singleton null instance
```

测试下

```
NSNull *null1 = (id)kCFNull;
NSNull *null2 = [NSNull null];
```

输出信息

```
(NSNull *) null1 = 0x0000000107b10af0
(NSNull *) null2 = 0x0000000107b10af0
```

哈哈，其实 `kCFNull这个宏` === `[NSNull null]`这一句代码

***

小结下 nil 与 [NSNull null] 的区别:

- nil:
	- Objc对象不存在，既`内存地址 都没有`

- [NSNull null]:
	- 得到是一个`单例对象`，是`有内存地址`的
	- 可以设置到集合对象中，而`nil`则不能够设置到集合对象中