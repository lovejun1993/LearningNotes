---
layout: post
title: "自定义快速枚举"
date: 2015-08-25 23:54:49 +0800
comments: true
categories: 
---

###在写缓存先关代码时，发现了一个从来没注意到的东西`NSFastEnumeration`，来记录下学习笔记.


***

####如果一个`集合`要使用`for-in`形式进行快速遍历，那么就要求这个`集合`需要实现某一些规定的方法 -- `NSFastEnumeration协议`.

```
forin demo代码:

for (Person *person in personArray) {
	//....
}
```

好像我们对`NSArray`、`NSDictionary`等等，经常使用过`for-in`的快速遍历方法，那么也就是说`NSArray`、`NSDictionary`已经是实现了`NSFastEnumeration协议`.

可以去看一下`NSArray`、`NSDictionary`头文件定义.

```
@interface NSArray : NSObject <NSCopying, NSMutableCopying, NSSecureCoding, NSFastEnumeration>

//...略
```

```
@interface NSDictionary : NSObject <NSCopying, NSMutableCopying, NSSecureCoding, NSFastEnumeration>

//...略
```	

如上可以看出，其实已经实现了`NSFastEnumeration协议了`，所以我们才能对他们使用`for-in`方式进行快速遍历。

***

###看一下与快速枚举遍历相关协议声明

```
@protocol NSFastEnumeration

- (NSUInteger)countByEnumeratingWithState:(NSFastEnumerationState *)state objects:(id __unsafe_unretained [])buffer count:(NSUInteger)len;

@end

@interface NSEnumerator : NSObject <NSFastEnumeration>

- (id)nextObject;

@end

@interface NSEnumerator (NSExtendedEnumerator)

@property (readonly, copy) NSArray *allObjects;

@end
```

如上看出:

1. 有一个协议叫做`NSFastEnumeration`.
2. 有一些协议实现类是`NSEnumerator`.

所以说，`NSEnumerator`是实现了`NSFastEnumeration`定义的相关功能函数接口，因此具备了快速遍历的功能。

***

###使用快速枚举遍历有何好处？

1. 比使用`for (int i = 0; i < count; i++) { ... }`效率高

2. 使用`for-in`语法简单清晰

3. 快速遍历使用了`修改监视器`，如果检测到在遍历过程中对`被遍历的集合的长度`进行修改，那么会由系统抛出异常，让系统程序进行崩溃.

4. 由于在枚举的过程中是不能对`集合长度`进行修改的，因此多个枚举是可以同时并发进行的

注意，`for-in`遍历时，不能对`集合的长度`进行修改，以一个例子说明.

```
// 创建一个Person实例的集合.

NSMutableArray *personArray = [NSMutableArray array];
for (int i = 0; i < 10; i++) {
    Person *p = [[Person alloc] initWithName:@"zhangsan" Age:i];
    [personArray addObject:p];
}
```

```
// 使用for-in快速遍历集合.

for (Person *person in personArray) {
    NSLog(@"age = %ld\n", person.age);
}

输出结果:
2015-10-02 00:24:25.246 AFNDemo[3684:55169] age = 0
2015-10-02 00:24:25.247 AFNDemo[3684:55169] age = 1
2015-10-02 00:24:25.247 AFNDemo[3684:55169] age = 2
2015-10-02 00:24:25.248 AFNDemo[3684:55169] age = 3
2015-10-02 00:24:25.248 AFNDemo[3684:55169] age = 4
2015-10-02 00:24:25.248 AFNDemo[3684:55169] age = 5
2015-10-02 00:24:25.248 AFNDemo[3684:55169] age = 6
2015-10-02 00:24:25.248 AFNDemo[3684:55169] age = 7
2015-10-02 00:24:25.248 AFNDemo[3684:55169] age = 8
2015-10-02 00:24:25.248 AFNDemo[3684:55169] age = 9
```

```
// 对被遍历集合中的某一个元素进行属性修改

for (Person *person in personArray) {
    person.age = 17;
}
```

上一步执行后，发现程序并没有崩溃，那么也就是说`使用for-in快速遍历集合时，可以对元素进行属性值修改`。

```
// 使用for-in快速遍历集合.

for (Person *person in personArray) {
    NSLog(@"age = %ld\n", person.age);
}

输出结果:
2015-10-02 00:24:25.248 AFNDemo[3684:55169] age = 17
2015-10-02 00:24:25.249 AFNDemo[3684:55169] age = 17
2015-10-02 00:24:25.249 AFNDemo[3684:55169] age = 17
2015-10-02 00:24:25.249 AFNDemo[3684:55169] age = 17
2015-10-02 00:24:25.249 AFNDemo[3684:55169] age = 17
2015-10-02 00:24:25.249 AFNDemo[3684:55169] age = 17
2015-10-02 00:24:25.249 AFNDemo[3684:55169] age = 17
2015-10-02 00:24:25.249 AFNDemo[3684:55169] age = 17
2015-10-02 00:24:25.249 AFNDemo[3684:55169] age = 17
2015-10-02 00:24:25.249 AFNDemo[3684:55169] age = 17
```

```
// 对集合的长度进行修改

for (Person *person in personArray) {
    [personArray removeObject:person];
}
```

如上代码就会崩溃。

> 小结:

使用`forin`快速遍历集合时，`不能修改集合的长度`，可以对元素的属性进行修改.

***

###先看一下`NSFastEnumeration`协议，就定义了一个实现函数.

```
@protocol NSFastEnumeration

- (NSUInteger)countByEnumeratingWithState:(NSFastEnumerationState *)state objects:(id __unsafe_unretained [])buffer count:(NSUInteger)len;

@end
```

1. 实现了`NSFastEnumeration`协议的任何集合类的实例，才能够使用`for-in`进行快速遍历.

2. 协议方法中的三个参数
	* state: 编译器给我们分配好的一个NSFastEnumerationState结构体变量地址。
	* buffer: 一个指针变量，指向消息接受者所分配的栈空间。
	* len: 单次枚举最大能放多少元素。


***

###再看下有一个结构体`NSFastEnumerationState`

```
typedef struct {
    unsigned long state;					// 表示当前状态，初始为0
    id __unsafe_unretained *itemsPtr;		// 指向当前所要枚举的集合首地址
    unsigned long *mutationsPtr;			// 用于检测，所要枚举的对象是否发生了改变
    unsigned long extra[5];					// 这里可以由实现者随意存放必要的额外数据
} NSFastEnumerationState;
```

对于这个结构体类型的实例，必须要设置`itemsPtr`与`mutationsPtr`这两个属性的值，并且`不能为空`。除非，此次枚举遍历直接返回0 。	

***

###有时间学习一下自定义枚举遍历.

等有时间看吧，该睡觉了...