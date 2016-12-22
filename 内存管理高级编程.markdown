---
layout: post
title: "multithread and memory programing  for iOS"
date: 2016-06-20 23:29:43 +0800
comments: true
categories: 
---

###一直想看看这本书，之前看过一次翻了一下就没看了，觉得都是一些意淫的东西就放弃了，刚好同事桌上有一本这个书，还是拿过来看看觉得还是有一些好东西的。

***

###本书主要讲解的东西

- 自动引用计数器
- Blocks
- Grand Central Dispatch（GCD）

***

###第一部分、引用计数器（非ARC时代使用）


办公室电灯的管理场景

- 第一个人进入办公室时，`打开`点灯 >>> count = 1
- 第二个人进入的办公室的人，`使用`电灯照明 >>> count = 2
- 第三个人进入的办公室的人，`使用`电灯照明 >>> count = 3
- 第四个人进入的办公室的人，`使用`电灯照明 >>> count = 4
- 第五个人进入的办公室的人，`使用`电灯照明 >>> count = 5
- 第一个人下班离开的每一个人，`不再使用`电灯照明 >>> count = 4
- 第二个人下班离开的每一个人，`不再使用`电灯照明 >>> count = 3
- 第三个人下班离开的每一个人，`不再使用`电灯照明 >>> count = 2
- 第四个人下班离开的每一个人，`不再使用`电灯照明 >>> count = 1
- 第五个人离开办公室的人，`关闭`电灯 >>> count = 0

这样通过对`电灯`这个资源使用`计数器`的来记录当前使用`电灯`的总人数，来决定`电灯`这个资源是否仍然被使用。

####上面的结构转换成ARC引用计数器的表达是如下

| 对电灯的操作 | 对象OC对象的操作 | 
| :-------------: |:-------------:| 
| 开灯 | `创建`一个新的OC对象 | 
| 使用 | `持有`一个已有的OC对象 | 
| 不使用 | `释放（解除持有）`一个OC对象 | 
| 关灯 | `废除`一个OC对象 | 

####从`引用计数器`这几个字来看，我们应该把注意力放在`引用`而不是`计数器`

从`引用`这两个字引发的思考点:

- 我自己生成的对象，我自己持有 >>> `生成`
- 别人生成的对象，我也能持有 >>> `持有`
- 当我不再需要持有这个对象时，对这个对象进行释放 >>> `释放`
- 没有被我持有的对象，我无法对其释放 >>> `释放`
- 最终当一个对象没有任何的引用/持有时，就会彻底被系统回收表示废弃掉 >>> `废弃`

| 对OC对象的操作 | Objective-C对应的方法 | 
| :-------------: |:-------------:| 
| 生成并持有对象 | alloc / new / copy / mutableCopy |
| 持有对象 | retain |
| 释放对象 | release |
| 废弃对象 | dealloc |

上述的所有的OC方法都是封装在`NSObject类（还有一个协议叫NSObject）`中:

```objc
@protocol NSObject

- (instancetype)retain;
- (oneway void)release;
- (instancetype)autorelease;
- (NSUInteger)retainCount;

@end
```

```objc
@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}

- (id)copy;
- (id)mutableCopy;

+ (instancetype)new;
+ (instancetype)allocWithZone:(struct _NSZone *)zone;
+ (instancetype)alloc;

- (void)dealloc;

@end
```

####我自己生成的对象，我自己持有

主要针对如下几种开头的方法，表明该方法时自己生成的对象，并属于当前创建环境所持有

- alloc
- new
- copy
- mutableCopy

自己持有，自己释放。

####别人生成的对象，我也能持有

- 别人生成的对象

```objc
id obj = [NSArray array];
```

- 对别人的对象进行持有

```objc
[obj  retain];
```

将别人的对象retain之后，就表示被当前环境所持有了，可以随意的使用该持有的对象，一定不会被释放.

####不再需要别人生成的对象时，对其释放持有的关系

- 返回生成的一个对象方式一、让接收者去持有

> 方法名一定要按照 allocXxxx、newXxxx、copyXxx、mutableCopyXxx 命令，表示由接收者自己去持有对象.

```objc
+ (id)allocObject {
	NSObject *obj = [NSObject new];
	
	//...
	
	return obj;
}
```		

- 返回生成的一个对象方式二、接收者只是使用并非持有，对象持有由`创建者`或`自动释放池`来负责

> 方法名一定不能是 allocXxxx、newXxxx、copyXxx、mutableCopyXxx 命令.

```objc
+ (id)getInstance {
	NSObject *obj = [NSObject new];
	
	// 将对象加入自动释放池
	[obj autorelease];
	
	return obj;
}
```

[NSArray array]返回的对象，就是通过注册到自动释放池去持有的。被放入释放池持有的对象，当释放池释放的时候才会被释放。

- 可以将一个处于自动释放池持有的对象，变为自己来持有

```objc
//1. 获取来自释放池的对象
id obj = [self getInstance];

//2. 变为自己持有
[obj retain];

//3. 使用对象
//...

//4. 不再使用对象
[obj release];
```

####不要释放`非自己`持有的对象，而应该让真正的持有者自己去释放

- 自己持有的对象途径:
	- alloc/new/copy/mutableCopy
	- retain

- 那么对于上面途径之外获取的对象 >>>> **绝对不能对其进行释放操作**

> 错误情况一、对自己持有的对象，进行连续释放，导致程序崩溃 EXC BAD ACCESS


```objc
id obj = [[NSArary alloc] init];
[obj release];
[obj release];//导致程序崩溃
```

> 错误情况二、对非自己持有的对象，进行释放

```objc
NSArray *array = [NSArray array];
[array release];//错误的，因为这个array对象并不是当前环境所持有
```

****

###深入NSObject类的 alloc/retain/release/dealloc方法实现

- OSX/iOS大部分系统代码是开源的公开在Apple Open Source官网
- 但是OC平台的Foundation框架并没有开源，那么就无法直接了解NSObject类的具体实现
- OC平台的Foudnation库对应的c平台使用的CoreFoundaiton却是开源的，但是仍然无法了解NSObject类中如何去调用CoreFoundation完成工作的

虽然无法直接了解iOS中OC平台的Foundation NSObejct类实现，但是可以通过`GNUStep`的源码来对NSObject类实现了解到90%。

那么GNUStep是什么？现在的iOS的Cocoa框架就是在GNUStep的基础上演变过来的，其大部分都是相同的，所以了解了GNUStep就相当于明白了大部分的Cocoa底层实现。


GNUStep源码的下载地址:

```
http://www.gnustep.org/resources/downloads.php#core
```

打开下载完的base.xcodeproj工程可以看到所有的源码.m文件.

首先看到NSObejct.m 的alloc方法实现:

```objc
+ (id) alloc
{
  return [self allocWithZone: NSDefaultMallocZone()];
}
```

继续走到

```objc
+ (id) allocWithZone: (NSZone*)z
{
  return NSAllocateObject (self, 0, z);
}
```

继续走到

```c
struct obj_layout {

	// 1. padding
    char	padding[__BIGGEST_ALIGNMENT__ - ((UNP % __BIGGEST_ALIGNMENT__)
      ? (UNP % __BIGGEST_ALIGNMENT__) : __BIGGEST_ALIGNMENT__)];
      
    //2. 对象的引用计数器
    NSUInteger	retained;
};

typedef struct obj_layout *obj;
```

```c
inline id
NSAllocateObject (Class aClass, NSUInteger extraBytes, NSZone *zone)
{
	id	new;
    int	size;

    // 不能够对Meta类进行创建对象
    NSCAssert((!class_isMetaClass(aClass)), @"Bad class for new object");
    
    // 计算objc_class结构体实例在内存占用的总大小
    size = class_getInstanceSize(aClass) + extraBytes + sizeof(struct obj_layout);
    
    // NSZone块默认
    if (zone == 0)
    {
        zone = NSDefaultMallocZone();
    }
    
    // 创建objc_class的一个实例
    new = NSZoneMalloc(zone, size);
    
    
    if (new != nil)
    {
        // 使用memset函数初始化分配的内存为数值0
        memset (new, 0, size);
        
        // 得到对象内存的首地址
        new = (id)&((obj)new)[1];
        
        // 绑定objc_class结构体实例给对象指针
        object_setClass(new, aClass);
	}
	
	return new;
}
```

NSZone对应的c结构体定义:

```c
struct _NSZone
{
  /* Functions for zone. */
  void *(*malloc)(struct _NSZone *zone, size_t size);
  void *(*realloc)(struct _NSZone *zone, void *ptr, size_t size);
  void (*free)(struct _NSZone *zone, void *ptr);
  void (*recycle)(struct _NSZone *zone);
  BOOL (*check)(struct _NSZone *zone);
  BOOL (*lookup)(struct _NSZone *zone, void *ptr);
  struct NSZoneStats (*stats)(struct _NSZone *zone);
  
  // Zone granularity
  size_t gran; 
  
  // Name of zone (default is 'nil')
  __unsafe_unretained NSString *name; 
  
  // 下一个Zone
  NSZone *next;
};
```

Zone就是区域的意思，是苹果最早用来内存管理的方法，来防止内存`碎片化`。也就是说将整块内存分成若干个区域Zone，来存放不同的对象。

如果使用一个Zone时（不使用区域）来存放所有对象时，可能存在的内存碎片化问题:

![](http://i1.piimg.com/8311/cee8d11dd67deda5.png)


如上图示当去除小内存块数据时，可能造成出现一些不连续、不会再被使用的无用内存，`因为该内存块太小了，无法存放比较大的数据`，就可能造成这一小块内存空间永远也无法存放数据了。

那如果，按照一定的规律（大小）将整个内存进行切割成不同的区域，然后将对象按照规律分配到对应的区域中存放了，那么效果是下面这样。

- 将整个内存空间，按照一定规律的大小进行划分
- 将对象按照其大小，放到对应的内存块存放

![](http://i1.piimg.com/8311/a7350c8f762519bf.png)

图示将整个内存块分为两个区域，第一个区域是左侧较小的内存块，第二个是右侧较大的内存块。当较小的数据就分配到左侧区域存放，而较大的数据分配到右侧区域存放。很显然这可以一定程度减少无用内存的出现。


但是由于苹果的runtime系统对`内存管理`做的已经足够的牛逼，所以就不再使用上面的`区域Zone`的这种方法去管理内存问题了。也就是说NSZone这个东西，基本上被废弃掉了....

那么说当去掉了NSZone相关代码之后，NSAllocateObject函数简化成如下:

```c
inline id
NSAllocateObject (Class aClass, NSUInteger extraBytes, NSZone *zone)
{
	id	new;
	int	size;
	
	//1. 要分配的内存总字节大小 = 对象的长度字节 + 额外字节 + obj_layout实例长度字节
	size = class_getInstanceSize(aClass) + extraBytes + sizeof(struct obj_layout);
	
	//2. 分配上面计算出的size大小的内存空间
	new = NSZoneMalloc(zone, size);
	
	//3.
	if (new != nil)
    {
    	//3.1 内存数据全部初始化为0，包括obj_layout.retainCount也是0
		memset (new, 0, size);
		
		//3.2  因为new指向整个空间主要分为两部分:
		// new[0]: 存放`obj_layout结构体`实例，保存的是retainCount值
		// new[1 ~ size-1]:  存放的是`对象`数据
		//所以这里，alloc最终要返回对象数据的起始地址 &new[1]
      	new = (id)&((obj)new)[1];
      	
      	// 3.3 分配内存对应的objc_class实例
      	object_setClass(new, aClass);
    }
	
	retrun new;
}
```

由于本人的c语言水平很菜，对于`new = (id)&((obj)new)[1];`这一句代码，我有几个疑惑:

- 疑惑一、new[0]一定存放`obj_layout`实例，new[1]一定存放`objc_class`结构体实例吗？

- 疑惑二、为什么要使用`obj_layout`这个类型转换new地址了？并且最后返回 `&news[1]`的地址了？

为了解决如上几个问题，继续将上面的`NSAllocateObject()`函数简化成如下最简单的形式便于分析:

```c
//1. 总长度字节
size_t = size = objc_class实例字节长度 + obj_layout实例字节长度;

//2. 对象长度字节 + obj_layout实例字节
id new = calloc(1, size);

//3. 将分配的内存首地址，交给obj_layout类型的指针变量来指向
struct obj_layout *p = (struct obj_layout *)new;

//4. 返回指针变量p指向的，下一个obj_layout类型元素的首地址
return (id)(p+1);
```

然后从书上截了个图解释这些问题:

![](http://i2.piimg.com/8311/6dea845ba10fcdf8.png)

根据上面的图示继续思考，我想原因可能是这样的:

- 第一、对于`obj_layout`实例占用的字节长度，放在整个总长度空间的最前面

- 第二、当`obj_layout`长度存放完毕之后，后面跟着全部存放的是`objc_class`实例的数据

- 第三、当申请到 `sizeof(obj_layout) + sizeof(objc_class)` 这样总长度内存空间之后，使用 `struct obj_layout *` 的指针变量类型来对`总内存空间的首地址new`进行强制转换，得到的是`obj_layout 实例占用头部的整个空间`，继而此时`p`指向的是obj_layout实例占用的整个空间的起始地址

- 第四、那么此时使用 `p+1` 刚好就是 `objc_class实例` 所在占用空间的 `起始首地址`

小小的证明一把上述的猜测，写了一段简单的代码:

```c
struct Person {
    int pid;//四个字节
};

struct Man {
    int mid;//四个字节
    int sex;//四个字节
};

static struct Man* instance() {
    
    // 分配总长度的内存空间
    size_t p_s = sizeof(Person);
    size_t m_s = sizeof(Man);
    size_t size = p_s + m_s;
    printf("p_s = %ld, m_s = %ld, size = %ld\n", p_s, m_s, size);
    
    // 分配内存空间
    void *news = calloc(1, size);
    printf("整个长度内存空间起始地址: %p\n", news);
    
    // 将头部空间先转换成Person类型
    struct Person *p = (struct Person *)news;
    printf("Person地址: %p\n", p);
    p->pid = 1111;
    
    // 再将后面的空间转换成Man类型
    struct Man *m = (struct Man *)(p + 1);
    printf("Man地址: %p\n", m);
    m->mid = 2222;
    m->sex = 1;
    
    // 先对哪一种类型转换，就决定了谁先谁后存放.
    
    return m;
}


int main() {
    
    struct Man *m = instance();
    printf("mid = %d, sex = %d\n", m->mid, m->sex);
    
    return 0;
}
```

输出如下


```
p_s = 4, m_s = 8, size = 12
整个长度内存空间起始地址: 0x100400000
Person地址: 0x100400000
Man地址: 0x100400004
mid = 2222, sex = 1
```

可以看到，new指针指向的起始地址 == Person地址，而p+1之后指向的地址 == Man的起始地址。那么说，虽然p最开始指向的是Person实例的所在空间的首地址，但是通过p+1之后指向的就是下一个Man实例所在的新空间的首地址了。

OK，对于NSObject的alloc方法内部实现理解差不多了，然后跟着书往下看看`-[NSObject retainCount]`的实现:

```objc
- (NSUInteger) retainCount
{
#if	GS_WITH_GC
  return UINT_MAX;
#else
  return NSExtraRefCount(self) + 1;
#endif
}
```

前面宏的是说如果当前系统使用自动垃圾回收器GC时（Mac OS）返回的是一个最大的数，也就是说系统负责回收不需要手动进行操作。对于iOS系统系统来说，返回的是`NSExtraRefCount(self) + 1;`.

那么NSExtraRefCount()函数实现:

```c
inline NSUInteger
NSExtraRefCount(id anObject)
{
#ifdef __OBJC_GC__
  if (objc_collecting_enabled())
    {
      return UINT_MAX-1;
    }
#endif
#if	GS_WITH_GC
  return UINT_MAX - 1;
#else	/* GS_WITH_GC */
  return ((obj)anObject)[-1].retained;
#endif /* GS_WITH_GC */
}
```

后面贴的代码就直接过滤掉GC之类的宏代码简化如下:

```c
inline NSUInteger
NSExtraRefCount(id anObject)
{
	return ((obj)anObject)[-1].retained;
}
```

上面一句代码`((obj)anObject)[-1].retained;`的步骤:

```c
//1. 将当前对象空间的地址转换成 obj_layout指针类型
struct obj_layout *p = (struct obj_layout *)anObject;

//2. 由于obj_layout处于整个空间的最顶部，而当前p指向的是obj_layout下面的objc_class实例空间
// 所以p-1才指向obj_layout的实例空间
p = p - 1;

//3. 取出obj_layout实例的retained属性值
NSUInteger retained = p.retained
```

以书上给出的，通过对象空间首地址，查找obj_layout实例空间的图示:

![](http://i4.piimg.com/8311/0d2a282d22706860.png)

那么对于`-[NSObject retain]`方法实现我想也能想到了，就是将整个空间头部的obj_layout的retained属性+1.

```objc
- (id) retain {

	// 自增retained
	NSIncrementExtraRefCount(self);
	
	// 返回当前对象
	return self;
}
```

```c
inline void
NSIncrementExtraRefCount(id anObject) 
{
	if (allocationLock != 0)
    {
    	// 多线程环境
    	#if	defined(GSATOMICREAD)
	      /* I've seen comments saying that some platforms only support up to
	       * 24 bits in atomic locking, so raise an exception if we try to
	       * go beyond 0xfffffe.
	       */
	      if (GSAtomicIncrement((gsatomic_t)&(((obj)anObject)[-1].retained))
	        > 0xfffffe)
			{
		  		[NSException raise: NSInternalInconsistencyException
		    format: @"NSIncrementExtraRefCount() asked to increment too far"];
			}
		#else	/* GSATOMICREAD */
      		NSLock *theLock = GSAllocationLockForObject(anObject);

      		[theLock lock];
	      if (((obj)anObject)[-1].retained == UINT_MAX - 1)
			{
			  [theLock unlock];
		  	[NSException raise: NSInternalInconsistencyException
		    format: @"NSIncrementExtraRefCount() asked to increment too far"];
			}
	      ((obj)anObject)[-1].retained++;
	      [theLock unlock];
		#endif	/* GSATOMICREAD */
    }
   else {
   
   		// 非多线程环境
	   	if (((obj)anObject)[-1].retained == UINT_MAX - 1)
		{
		  [NSException raise: NSInternalInconsistencyException
		    format: @"NSIncrementExtraRefCount() asked to increment too far"];
		}
	      ((obj)anObject)[-1].retained++;
	    }
	}
}
```

上面代码对多线程环境进行了互斥写入retained属性值的处理:

- 如果是多线程环境，则有两种方式互斥:
	- 一、Atomic 原子操作
	- 二、NSLock简单互斥锁

- 那么不管是多线程环境or非多线程环境，最终的操作都是:

```c
((obj)anObject)[-1].retained++;
```

进行步骤分拆如下:

```c
//1. 将当前对象空间的地址转换成 obj_layout指针类型
struct obj_layout *p = (struct obj_layout *)anObject;

//2. 由于obj_layout处于整个空间的最顶部，而当前p指向的是obj_layout下面的objc_class实例空间
// 所以p-1才指向obj_layout的实例空间
p = p - 1;

//3. 取出obj_layout实例的retained属性值
p.retained = p.retained + 1;
```

那么对于`-[NSObject release]`就跟上面差不多了，就是`p.retained = p.retained - 1;`，就不贴代码了。

继续对于 `-[NSObject dealloc]`的函数实现:

```objc
- (void) dealloc
{
  NSDeallocateObject (self);
}
```

```c
inline void
NSDeallocateObject(id anObject)
{
	// 获取当前对象对应的 objc_class实例
	Class aClass = object_getClass(anObject);
	
	// 
	if ((anObject != nil) && !class_isMetaClass(aClass)) {
		
		// 取出对象上面的obj_layout实例
		obj	o = &((obj)anObject)[-1];
		
		// 根据obj_layout实例，获得当前对象存放在哪一个区域Zone
		NSZone	*z = NSZoneFromPointer(o);
		
		// 将当前对象的objc_class实例设置为: 0xdeadface
		object_setClass((id)anObject, (Class)(void*)0xdeadface);
		
		// 最后释放掉对象，将obj_layout实例空间的起始地址所在内存全部free()
		NSZoneFree(z, o);
	}
}
```

OK，如上就是NSObject对象的内存相关方法实现:

- alloc
- retain/release
- dealloc

从上几个方法实现，可以看出在整个对象所在的内存块，对于内存创建、被引用、被释放、被废弃整个过程中，最重要的结构体:

> obj_layout 这个结构体实例

- 首先，保存了retained属性值，标记该内存块是否应该被废弃掉还是继续使用

```c
struct obj_layout *p = 整一块的内存首地址;
```

```c
p = p + 1;//指向对象objc_class结构体实例所在空间
```

- 然后，所以的alloc/retain/release/dealloc方法，都是用过`对象(objc_class实例)`空间首地址，来找到`obj_layout实例`首地址，继而操作其retained属性值.


```c
struct obj_layout *p = (struct obj_layout *)anObject;//将当前对象空间的地址转换成 obj_layout指针类型
```

```c
p = p - 1;//p-1指向obj_layout的实例空间
```

如上是GNUStep的实现，但是苹果的实现有所不同。苹果的实现是通过一个`表 Map容器`来管理对象的引用计数器.

![](http://i4.piimg.com/8311/fa4b043abca00151.png)

使用内存块地址作为key值，内存块对应的引用计数作为value值，保存在表中。那么GNUStep如上实现基本上和苹果的一致，只是在引用计数器存放的问题上有所不同:

- GNUStep将引用计数器存放在`obj_layout实例`中，继而将`obj_layout实例`与`objc_class实例`存放在同一个内存块，并放在真个内存空间的`头部`

- 苹果将引用计数器放在了一个`表`来进行管理
	- 在使用工具进行内存泄露监测时，实际上就是从表中查看，所有的内存块是否存在持有者（引用计数器是否大于0）

***

####autorelease

autorelease借助NSAutoRelasePool来完成，完成的步骤:

- 生成并持有NSAutoRelasePool对象
- 然后接着创建其他的对象，并放入当前的NSAutoRelasePool对象中管理
- 废弃掉NSAutoRelasePool对象，里面的所有被管理对象也会被废弃

```objc
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];

id obj = [[NSObject alloc] init];

// 将对象放入池中管理
[obj autorelase];

// 不再使用时
[pool relase];
```

下面看看`-[NSObejct autorelease]`的GNUStep的实现

```c
- (id) autorelease {
	(*autorelease_imp)(autorelease_class, autorelease_sel, self);
	return self;
}
```

从上面代码似乎看不出什么具体的东西，但其实已经完成了，而且还做了`优化`。那如果没有做`优化`时，那么上面的代码应该是如下这样:

```c
- (id) autorelease {
	[NSAutoreleasePool addObject:self];
	return self;
}
```

那么不同的一句代码`(*autorelease_imp)(autorelease_class, autorelease_sel, self);`到底做了什么优化了？

点击去查看 `autorelease_imp、autorelease_class、autorelease_sel` 的类型定义:

```c
static id autorelease_class = nil; 
static SEL autorelease_sel;
static IMP autorelease_imp;
```

然后再找到这三个静态变量进行赋值的地方，在`+[NSObject initialize]`方法中

```c
+ (void) initialize {
	
	.....
	
	autorelease_class = [NSAutoreleasePool class];//获取NSObject类运行时对应的objc_class实例
	autorelease_sel = @selector(addObject:);//消息SEL
	autorelease_imp = [autorelease_class methodForSelector: autorelease_sel];//最终调用的c函数实现
	
	.....
}
```

再结合调用的代码

```c
(*autorelease_imp)(autorelease_class, autorelease_sel, self)
```

```
注意，所有的IMP函数格式，前两个参数target、sel是固定的:
returnType funcName(id target, SEL sel, id args1, id args2, ...);

target决定在哪一个objc_class实例的method_list查找传入的sel对应的objc_methid实例.
```

可以看出优化了什么了，实际上就是避免大量次数通过`-[NSObject autorelease]`都要`发送消息`继而还要`查询objc_class的method_list`来找到最终的autolrease这个sel对于的c函数实现IMP。那么直接获取到autolrease这个sel对于的`c函数实现IMP`（函数指针），直接通过函数指针调用最终的函数视，而不必走查询等过程。


当调用了`-[NSObject autorelase]`之后，接着调用了`+[NSAutorelasePool addObject:]`

```objc
+ (void) addObject: (id)anObj
{
    // 当前所在线程
    NSThread		*t = GSCurrentThread();
    
    // 如果当前线程不存在直接崩溃程序
    NSAssert(nil != t, @"Creating autorelease pool on nonexistent thread!");
    
    // 当前线程使用的释放池pool
    NSAutoreleasePool	*pool;
    pool = t->_autorelease_vars.current_pool;

    // 如果当前行线程使用的pool不存在、没有激活，重新创建一个pool并保存
    if (pool == nil && t->_active == NO)
    {
        // Don't leak while exiting thread.
        pool = t->_autorelease_vars.current_pool = [self new];
    }

    // 使用当前行线程的pool添加传入的对象
    if (pool != nil)
    {
        // +[NSAutoreleasePool addObject:]
        (*pool->_addImp)(pool, @selector(addObject:), anObj);
        
    } else {
		NSLog(报错信息.......);
	}	
}
```

那么说，当调用`-[NSObject autorelase]`时，实际上完成如下几步:

- 将当前对象作为参数，并发送消息 `+[NSAutorelasePool addObject:]`

- NSAutorelasePool的addObject:对应函数实现中，接着完成几步:
	- 获取当前程序所执行在的线程NSThread对象
	- 然后读取`NSThread对象->autorelease_thread_vars`结构体实例保存的NSAutoreleasePool对象
	- 接着判断线程绑定的Pool对象是否存在:
		- 存在，直接将传入的对象添加到Pool池中
		- 不存在或未激活，创建一个新的Pool对象，并保存到Thread对象
			- 再将传入的对象添加到新的Pool池中


> 所以说，对象并不是随便添加到一个Pool中，而是被放到`当前线程绑定的一个Pool`对象中。


最终NSAutoreleasePool对象，完成添加对象的方法`-[NSAutorelasePool addObejct:]`


```c
// 用来存放加入Pool管理的对象的一个容器结构.
typedef struct autorelease_array_list
{
  struct autorelease_array_list *next;
  unsigned size;
  unsigned count;
  __unsafe_unretained id objects[0];//数组保存加入Pool的对象
} array_list_struct;
```

```objc
@interface NSAutoreleasePool : NSObject 
{

	......
	struct autorelease_array_list *_released;
	struct autorelease_array_list *_released_head;
	unsigned _released_count;
	......
}
```

```objc
- (void) addObject: (id)anObj {
	
	....对当前所有的管理对象进行清理
	
	// 将传入的对象放入到 ob
   _released->objects[_released->count] = anObj;
   
   // 累加当前管理的总对象数
   (_released->count)++;
}
```

OK，那么对于 `-[NSObject autorelase]`的实现细节基本上了解的差不多了。

接下来，继续看关于NSAutoreleasePool对象的释放、清空、废弃的实现.

```objc
- (void) drain
{
  // Don't call -release, make both -release and -drain have the same cost in
  // non-GC mode.
  [self dealloc];
}
```

```objc
- (oneway void) release
{
  [self dealloc];
}
```

```objc
- (void) dealloc
{
	//1. 对管理的所有对象发送relase消息
	[self emptyPool];
	
	//2. 将当前Thread对象的autorelease_thread_vars实例从链表中移除
	struct autorelease_thread_vars *tv = ARP_THREAD_VARS;
	if (tv->current_pool == self)
    {
        tv->current_pool = _parent;
    }
    if (_parent != nil)
    {
        _parent->_child = nil;
        _parent = nil;
    }
	
	//3. 并没有立即释放掉autorelease_thread_vars实例，而是缓存起来，以便后面又要使用
	push_pool_to_cache (tv, self);
}
```

```objc
- (void) emptyPool {
	// 简化了代码的
	for(id obj in released->objects) {
		[obj release];
	}
}
```

苹果对autorealse的实现基本上和上面的GNUStep的实现一致的。

另外从书上记录一个用于在lldb进行调试，打印当前所有的Pool对象中的对象的`私有函数`.

```c
_objc_autorelasePoolPrint();
```

注意，如果执行`-[NSAutoreleasePool autorelase]`会发生异常

****

####ARC（Auto Reference Count）

在上面叙述的环境（非ARC）下，主要注意的四个问题:

- 自己生成的对象，被自己持有
- 非自己生成的对象，自己也能去持有
- 自己持有的对象，在不需要使用的情况下进行释放
- 非自己持有的对象，禁止对其进行释放

那么在ARC时代，如上四个问题其实依然存在，只是代码上写法发生变化了。

> 变化一、对象所有权的修饰符.

对象所有权的修饰符主要有如下这几种:

```objc
__strong
__weak
__unsafe_unretained
__autoreleasing
```

> __strong修饰符: 是对象指针和id指针的`默认`所有权修饰符。

```objc
id obj = [[NSObject alloc] init];

等价于

id __strong obj = [[NSObject alloc] init];
```

- 使用 __strong 修饰 `局部作用域` 生成并自己持有的对象.

```objc
- (void)arc {
    
    {
        Perosn *p = [[Perosn alloc] init];
        p.name = @"xzh1";
    }
    
    NSLog(@">>>>>>>>>>>>>>>>>>>");
    
    {
        Perosn * __strong p = [[Perosn alloc] init];
        p.name = @"xzh2";
    }
}
```

运行后的输出

```
dealloc >>>>> 线程 = <NSThread: 0x7fdbc3c04870>{number = 1, name = main}, 地址 = 0x7fdbc3c17950, name = xzh1
>>>>>>>>>>>>>>>>>>>
dealloc >>>>> 线程 = <NSThread: 0x7fdbc3c04870>{number = 1, name = main}, 地址 = 0x7fdbc3d09cd0, name = xzh2
```

当使用 __strong修饰的对象，超出了其作用域时，该对象就会失去强引用，继而被废弃掉。

- 使用 __strong 修饰 `局部作用域` 生成 `非自己` 持有的对象.

```objc
- (void)arc {
    {	
    	// 取得非自己生成的对象，但是通过使用 __strong修饰符来强制让自己来持有别人生成的对象
        NSMutableArray __strong *array = [NSMutableArray array];
    }
    
    // __strong修饰的指针超出作用域，对象失去了强引用的作用，继而被释放
}
```

上述代码在非ARC时代为如下这样的:

```objc
- (void)arc {
	{
		//1. 取得别人生成的对象
		NSMutableArray *array = [NSMutableArray array];
		
		//2. 让自由来持有这个对象
		[array retain];
		
		//3. 操作自己持有的对象
		//.....
		
		//4. 使用完毕然后释放持有的对象
		[array release];
	}	
	
	// 超出作用域，从别人获取到对象让别人废弃掉
}
```

继续看一段书上改写的一个使用 __strong 在多个指针指针的赋值:

```objc
- (void)arc {
    
    Perosn __strong *p1 = [[Perosn alloc] initWithName:@"person 1"];
    Perosn __strong *p2 = [[Perosn alloc] initWithName:@"person 2"];
    Perosn __strong *p3 = nil;
    
    // p1强引用了p2
    //1. 此时person 2对象，拥有两个强引用指针
    //2. 但是原来p1强引用的对象失去了强引用
    p1 = p2;//person 1 被废弃
    NSLog(@"p1 = p2");
    
    p3 = p1;
    NSLog(@"p3 = p1");
    
    // 让 person 2对象 减少一个强引用，但仍然不会被废弃掉
    p1 = nil;
    NSLog(@"p1 = nil");
    
    // 让 person 2对象 继续减少一个强引用，但仍然不会被废弃掉
    p3 = nil;
    NSLog(@"p3 = nil");
    
    // 最后让 person 2对象 最后的一个强引用取消
    p2 = nil;//person 2对象就会被废弃
    NSLog(@"p2 = nil");
}
```

输出结果

```
2016-06-26 23:36:41.478 Demos[24248:344608] dealloc >>>>> 线程 = <NSThread: 0x7fcc626051e0>{number = 1, name = main}, 地址 = 0x7fcc62713ef0, name = person 1
2016-06-26 23:36:41.479 Demos[24248:344608] p1 = p2
2016-06-26 23:36:41.479 Demos[24248:344608] p3 = p1
2016-06-26 23:36:41.479 Demos[24248:344608] p1 = nil
2016-06-26 23:36:41.479 Demos[24248:344608] p3 = nil
2016-06-26 23:36:41.480 Demos[24248:344608] dealloc >>>>> 线程 = <NSThread: 0x7fcc626051e0>{number = 1, name = main}, 地址 = 0x7fcc62703a30, name = person 2
2016-06-26 23:36:41.480 Demos[24248:344608] p2 = nil
```

- 使用 __strong 修饰 `对象成员变量`指向的对象.

```objc
@interface ViewController () {
    Perosn __strong *_person;
}
@end

@implementation ViewController

- (void)setPerson:(Perosn *)person {
    if (_person != person) {
		_person = person;
	}
}

- (void)arc2 {
    for (int i = 0; i <= 2; i++) {
        Perosn *p = [[Perosn alloc] initWithName:[NSString stringWithFormat:@"person%d", i]];
        [self setPerson:p];
    }
}

@end
```

输出如下

```
2016-06-26 23:46:35.833 Demos[24867:355795] dealloc >>>>> 线程 = <NSThread: 0x7fce80407670>{number = 1, name = main}, 地址 = 0x7fce80514420, name = person0
2016-06-26 23:46:35.833 Demos[24867:355795] dealloc >>>>> 线程 = <NSThread: 0x7fce80407670>{number = 1, name = main}, 地址 = 0x7fce805170f0, name = person1
```

可以看到，person0和person1都被废弃掉了，而最后传入的person2对象被保留下来。

如果是非ARC时代，那么setPerson:方法实现还得写成如下这种格式:

```objc
- (void)setPerson:(Perosn *)person {

	// 外层判断对象是否相同
	if (_person != person) {
	
		// 释放旧值
		[_person release];
		
		// 持有新值
		_person = [person retain];
	}
}
```

那么相比ARC时代使用 __strong修饰的成员变量的setter方法实现，就是下面这样简单:

```objc
- (void)setPerson:(Perosn *)person {
    if (_person != person) {
		_person = person;//直接赋值，ARC系统自动完成旧值释放，新值持有
	}
}
```

那么就是说在ARC下，使用 `__strong` 来修饰成员对象指针的对象所有权时，编译器会自动对成员属性的setter方法实现做两件事情:

- 对成员指针指向的`原始对象`做relase操作
- 对传入的`新的对象`做retain操作，并保存持有

> 再次贴上 非ARC 下考虑四件事情，在ARC下发生的改变:

- 自己生成的对象，被自己持有
- 非自己生成的对象，自己也能去持有
- 自己持有的对象，在不需要使用的情况下进行释放
- 非自己持有的对象，禁止对其进行释放

前面三个事情，都可以通过 __strong 修饰符完成，当失去强引用指针时由系统废弃。最后一项对非自己持有的对象禁止释放，在ARC下这是肯定满足的。因为Xcode现在的版本基本上不允许输入release/retain/autorelease的消息。而且在ARC下默认都是使用 __strong 来修饰对象的指向，只是增加了一个指向别人对象的`强指针`，但废弃对象的权利仍然没有。

另外，使用 `__strong、__weak、__autoreleasing` 这三种修饰指向的对象指针，会由ARC系统自动完成:

- 自动初始化为nil
- 当所指向的对象被废弃掉时，同样也会被设置为nil

相反，`__unsafe_unretained`在对象被废弃掉时，不会被设置为nil。那么此时仍然强制使用`__unsafe_unretained`修饰的指针去操作废弃掉的对象，就会导致程序crash崩溃，报错 `EXC_BAD_ACCESS....异常`.

> __weak 修饰符、为解决 循环引用 而生.

在使用block的现在经常会出现对象之间的循环引用，不过在以前使用delegate时一样也会出现对象之间的循环引用。

那么什么是对象的循环引用？

![](http://ww1.sinaimg.cn/large/0060lm7Tgw1f5h691rw89j30o60dedhf.jpg)

简单的说，就是`对象A强引用对象B`，而`对象B又强引用对象A`，`两个对象之间都是强引用对方`，这样一来就导致双方都不会被释放废弃。

如下就是对象之前强引用的错误代码:

```objc
#import <Foundation/Foundation.h>

@interface Dog : NSObject

- (void)setObj:(NSObject *)obj;

@end
```

```objc
#import "Dog.h"

@implementation Dog {
    NSObject __strong *_obj;
}

- (void)setObj:(NSObject *)obj {
    _obj = obj;
}

- (void)dealloc {
    NSLog(@"%@", self);
}

@end
```

测试强引用的代码

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	    
	// 都是局域内对象，正常来说出了方法块就会被废弃掉
	Dog *dog1 = [Dog new];
	Dog *dog2 = [Dog new];
	    
	// dog1 强引用 dog2
	[dog1 setObj:dog2];
	    
	// dog2 强引用 dog1
	[dog2 setObj:dog1];
}
```

运行结果

```
没有任何的dealloc输出信息
```

那么表示此时 dog1 与 dog2 这两个对象并没有被废掉，即发生了内存泄漏了。那么为什么会发生泄漏了，分析如上这一段使用Dog的代码:


```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	
	// dog1 强指向 Dog对象
	Dog *dog1 = [Dog new];
	
	// dog2 强指向 Dog对象
	Dog *dog2 = [Dog new];
	    
	// dog1 强指向的对象中的成员变量 _obj 也是 强指向了 dog2指向的对象
	[dog1 setObj:dog2];
	    
	// dog2 强指向的对象中的成员变量 _obj 也是 强指向了 dog1指向的对象
	[dog2 setObj:dog1];
	
	// 所以此时，两个dog对象的强引用指针如下:

	// 第一个Dog对象有如下强引用指针
	//	- dog1
	//	- dog2->_obj
	
	// 第二个Dog对象有如下强引用指针
	//	- dog2
	//	- dog1->_obj
}

// 出了方法块之后，两个对象的局域强引用指针dog1与dog2失去了强引用的作用，但是仍然都还有一个强引用指针，就是对方.
```

所以说，对象之间相互的强引用(retain cycle) >>>> 内存泄露的发生。

上面的情况是，对象1 与 对象2 之间的强引用。但是还有 `对象自己强引用自己`的情况.

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	Dog *dog = [Dog new];
	[dog setObj:dog];
}
```

上面会产生对象的循环引用，归根结底就是全部都使用`__strong 强引用`去指向一个对象造成的。

那么解决循环引用，ARC提供了另一个对象指向修饰符 `__weak 弱引用`。弱引用不会增加一个强指针指向数，而仅仅只是能够去调用指向的对象，是不能够去`持有`这个对象。

那么通过 `__weak`修饰成员指针，来打破对象之间的循环引用，对之前的例子修改如下:


```objc
#import <Foundation/Foundation.h>

@interface Dog : NSObject

- (void)setObj:(NSObject *)obj;

@end
```

```objc
#import "Dog.h"

@implementation Dog {
    NSObject __weak *_obj;//由之前的 __strong 改为 __weak
}

- (void)setObj:(NSObject *)obj {
    _obj = obj;
}

- (void)dealloc {
	NSLog(@"Dog对象%@被废弃", self);
}

@end
```

然后测试代码

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	    
	// 都是局域内对象，正常来说出了方法块就会被废弃掉
	Dog *dog1 = [Dog new];
	Dog *dog2 = [Dog new];
	    
	// dog1 弱引用 dog2
	[dog1 setObj:dog2];
	    
	// dog2 弱引用 dog1
	[dog2 setObj:dog1];
}
```

运行结果输出

```
2016-07-04 13:14:11.234 demo[2139:558253] Dog对象<Dog: 0x7fc482519090>被废弃
2016-07-04 13:14:11.235 demo[2139:558253] Dog对象<Dog: 0x7fc48251b440>被废弃
```

可以看到，Dog对象使用 `__weak`来修饰成员变量指针后，去指向对方Dog对象，之间不会再形成循环强引用了。那么不会再形成循环强引用，就不会出现内存泄漏了。

现在两个Dog对象之间的指向关系图:

![](http://i1.piimg.com/567571/cec57b02425559e3.jpg)

> 使用 `__weak`修饰的对象指针，在该对象被释放时，自动将指针设置为nil，ARC系统避免使用野指针造成程序崩溃。

Dog类

```objc
#import <Foundation/Foundation.h>

@interface Dog : NSObject

- (void)test;

@end
```

```objc
@implementation Dog

- (void)test {
    NSLog(@"test");
}

@end
```

ViewController测试

```objc
@implementation ViewController {
    id __weak _obj1;
    id __strong _obj2;
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    _obj2 = [Dog new];
    
    _obj1 = _obj2;
    
    _obj2 = nil;
    
    NSLog(@"开始调用weak指针指向的对象方法");
    
    [_obj1 test];
}

@end
```

运行结果

```
2016-07-04 13:28:59.367 demo[2461:633794] Dog对象<Dog: 0x7f82696107e0>被废弃
2016-07-04 13:28:59.368 demo[2461:633794] 开始调用weak指针指向的对象方法
```

可以看到Dog对象先被废弃掉了，再调用Dog对象的test方法，程序仍然能够正常运行结束，那说明指针已经被赋值为`nil`了。

但如果将成员变量指针 `_obj1`的修饰符改成 `__unsafe_unretained`再运行上面的代码，基本上都会程序崩溃。

```objc
@implementation ViewController {
//    id __weak _obj1;
    id __unsafe_unretained _obj1;
    id __strong _obj2;
}

...

@end
```

运行后程序走到 `-[Dog test]`后崩溃掉

```
2016-07-04 13:33:48.278 demo[2556:648807] Dog对象<Dog: 0x7f9d20c02ab0>被废弃
2016-07-04 13:33:48.279 demo[2556:648807] 开始调用weak指针指向的对象方法
(lldb)//程序崩溃了...
```

> 通过 __weak修饰的指针不会持有这个对象，且当该对象被释放时，会由ARC系统自动将指针赋值为nil。这一个特性，可以用来检测对象是否被废弃.

```objc
@implementation ViewController {
    id __weak _obj1;
    id __strong _obj2;
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    //1.
    _obj2 = [Dog new];
    
    //2. 使用一个弱引用指针指向创建的对象
    _obj1 = _obj2;
    
    //3. 释放创建的对象
    _obj2 = nil;
    
    //4. 使用弱引用指针判断对象是否被废弃
    if (nil == _obj1) {
        NSLog(@"对象被废弃掉了");
    } else {
        NSLog(@"对象没有被废弃，内存泄漏");
    }
}

@end
```

输出结果

```
2016-07-04 13:41:57.986 demo[2613:664209] Dog对象<Dog: 0x7fe33b5a3040>被废弃
2016-07-04 13:41:57.987 demo[2613:664209] 对象被废弃掉了
```

注意: `__weak`是在iOS5和OX X Lion推出的，之前只能使用 `__unsafe_unretained`修饰符.

> __weak 修饰的指针可以解除循环强引用，但是并 不能够持有 其指向的对象，所以也不能一味的使用 __weak 来修饰指向对象的指针.

不能够持有对象的意思如下:

```objc
@interface Dog : NSObject

- (void)test;

@end

@implementation Dog 

- (void)test {
    NSLog(@"Dog work");
}

- (void)dealloc {
    NSLog(@"Dog对象: %@ 被废弃", self);
}

@end
```

```objc
@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	    
	 NSLog(@"1");
    
    _obj = [Dog new];//弱引用指针持有创建的对象
    
    NSLog(@"2");
    
    [_obj test];//实际上不会发送这条消息，因为Dog对象被废弃，_obj = nil
    
    NSLog(@"3");
}

@end
```

运行后的结果

```
2016-07-06 00:54:25.272 Demos[4064:65360] 1
2016-07-06 00:54:25.273 Demos[4064:65360] Dog对象: <Dog: 0x7fbd79d06ea0> 被废弃
2016-07-06 00:54:25.273 Demos[4064:65360] 2
2016-07-06 00:54:25.273 Demos[4064:65360] 3
```

可以看到，创建出Dog对象，往下执行时立马就废弃掉了。所以使用 `__weak`修饰的指针，并不能够保证指向的对象不会被废弃，也就是做不到 `__strong`的对象所有权的作用。

> `__unsafe_unretained`对象指针修饰符.

正如其名包含两个部分:

- unsafe 不安全
- unretained 不强引用持有

> 注意: ARC环境，编译器会自动对 `__strong、__weak` 修饰的指针做内存管理的代码处理，但是不会对 `__unsafe_unretained` 修饰的指针做处理.

使用 `__unsafe_unretained` 修饰的指针与`__weak`的指针是有一点是一样的: 不会持有对象，一旦创建的对象出了作用域就会被废弃。

```objc
#import <Foundation/Foundation.h>

@interface Dog : NSObject

- (void)test;

@end

@implementation Dog 

- (void)test {
    NSLog(@"Dog work");
}

- (void)dealloc {
    NSLog(@"Dog对象: %@ 被废弃", self);
}

@end
```

```objc
@implementation ViewController {
    id __unsafe_unretained _obj;
}

- (void)newDog {
    _obj = [Dog new];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	[self newDog];
    NSLog(@"_obj = %@", _obj);
}

@end
```

运行结果

```
2016-07-04 23:30:07.665 Demos[1764:21915] Dog对象: <Dog: 0x7f8652ea4300> 被废弃
2016-07-04 23:30:07.666 Demos[1764:21915] _obj = <Dog: 0x7f8652ea4300>
```

咦奇怪了，出了newDog方法后Dog对象被废弃掉了，但是 `_obj`指针仍然指向者那个被废弃掉的对象.

而且我发现，如上代码不一定每次崩溃，而是有几率性的崩溃，将如上touchesBegan:方法最后再加上一句代码之后再次运行:

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	[self newDog];
    NSLog(@"_obj = %@", _obj);
    
    // 再次调用对象方法
    [self newDog];
}
```

发现同样是有几率性的崩溃，但是几率就比不加最后一句时大多了。那么，为什么了会发生崩溃了？

> 因为 `__unsafe_unretained` 修饰的指针，不会由ARC系统自动完成当被指向对象废弃掉时赋值为`nil`.

如果一旦被指向的对象已经被废弃掉了，但是指针并不会被赋值nil，继续使用一块被废弃掉的内存块，就`有可能`造成程序崩溃。

所以说，`__unsafe_unretained`不要对`对象型`数据指针修饰，应该使用`__weak`。

> `__autoreleasing` 修饰符

在ARC环境下，`-[NSObject autorelease]`已经不能被使用了，也不能够使用`NSAutoreleasePool`，但是有新的方式去使用。


在非ARC环境下时，将一个对象交给自动释放池去持有并管理后续的释放:

```objc
NSAutoreleasePool * pool = [[NSAutoreleasePool alloc] init];
Dog *dog =[Dog new];
[dog release];//将Dog对象加入到最近的pool对象中
[pool release];
```

在ARC下，通过另外一种方式来创建`NSAutoreleasePool 对象`，并将对象注册到自动释放池.

```objc
@autoreleasepool {    
    Dog * __autoreleasing dog =[Dog new];//通过使用__autoreleasing修饰的指针指向对象，将对象注册到pool中
}
```

也即是说，在非ARC与ARC下，将对象注册到释放池的操作区别:

|  ARC/非ARC | 创建自动释放池 | 将对象注册到释放池 |
| :-------------: |:-------------:| :----------:|
| 非ARC | [[NSAutoreleasePool alloc] init] 与 [pool release] 之间 | [对象 release] |
| ARC  | @autoreleasepool { //注册对象代码 } | Dog * __autoreleasing dog = 对象 （可以不用显示使用 __autoreleasing修饰指针，由编译器默认会加上）|

> 当获取 非自己创建的对象 但是 自己变成持有者 时，释放池的一些知识:

```objc
#import <Foundation/Foundation.h>

@interface Dog : NSObject

+ (Dog *)dog;

@end

@implementation Dog {

+ (Dog *)dog {
    return [Dog new];
}

@end
```

从dog方法实现中，只是创建了一个对象，既然是当前dog方法内创建的对象，自然这个Dog对象就是被dog方法这个环境所持有。然后当做`函数的返回值`返回给外部的调用者时，ARC系统其实是做了一些自动释放池处理的。

将上述的dog方式改写一下:

```objc
+ (Dog *)dog {

	//1. 当前方法块强引用Dog对象
    id __stong obj = [Dog new];
    
    //2. 将Dog对象当做返回值
    return obj;
}
```

上述代码如果在非ARC环境，应该写成如下

```objc
+ (Dog *)dog {

	//1. 
    id __stong obj = [Dog new];
    
    //2. 加入线程绑定的默认自动释放池
    return [obj autorelease];
}
```

再比较ARC下的dog方法实现，好像我们并没有操作任何的`释放池`，那么这个返回的对象不会被废弃掉吗？

在ARC环境下，编译器会对`方法返回值 是NSObject对象`时做下几件事:

- 首先、检测这个对象，是否是通过以 `new/alloc/copy/mutableCopy` 开始的方法名 所返回的对象

- 然后、如果不是，就说明是`非自己创建`的对象，那么就会将该对象加入到当前线程默认绑定的释放池中管理

所以，前面在ARC下的dog方法实现中，没有任何的操作释放池的原因就是这。经常使用`-[NSArray array]`创建的非我们自己持有的数组对象，ARC环境就是像如下这样放入释放池的:

```objc
- (void)test {

	// ARC系统自动找到一个合适的释放池
	@autoreleasepool {
	
		//1. 使用__autoreleasing修饰指针，将指向的对象放入释放池
		NSArray __autoreleasing *array = [NSMutableArray array];
  	
  	
  		//2. 后面只需要操作数组对象
  		//...
  		
  		//3. 至于数组的释放废弃不用接受者考虑
  		// 由释放池来管理对象废弃  	
	}
}
```

其实如上代码不同写`__autoreleasing`也能自动将对象注册到释放池.

> 那么就有一个对于 获取对象的方法 的命名规范: 如果一个方法返回的对象是`自己持有`，那么该方法一定要以 new/alloc/copy/mutableCopy 作为开头来命名方法，否则就不应该使用如上四中作为方法名的开头。


使用`__weak`修饰的指针指向的对象时，可能在使用的过程中，该对象就被废弃掉了。虽然`__weak`修饰的指针会被ARC系统自动设置为nil，程序是不会崩溃。但是对象方法调用消息不会被发送，也就是不起作用。那么，对一个`__weak`修饰的指针指向的对象时，需要保证他一定不释放做完一个操作，必须将该对象放入到一个自动释放池:

```objc
@implementation ViewController 

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    // 使用 weak指针 指向一个方法返回值对象
    Dog __weak *weakDog = [Dog dog];
    
    // 将weak指针 指向的对象，注册到释放池去延迟废弃
    @autoreleasepool {
        Dog __autoreleasing *autoDog = weakDog;
        [autoDog test];
    }
}

@end
```

上面是对 `一重指针` 的释放池注册操作，那么对于`二重指针`了？比如，如下

```objc
NSString *path = @".....";
NSError *error = nil;

[[NSFileManager defaultManager] removeItemAtPath:path error:&error];
```

作用就是传入一个NSError对象指针的指针，然后由NSFileManager得方法创建一个NSError对象，并让传入的指针指向这个NSError对象.

```objc
- (BOOL)removeItemAtPath:(NSString *)path error:(NSError **)errorP {
	if (出错) {

		//1. 
		NSError *error = [NSError errorWithDomain:@"错误域" code:@"错误码" userInfo:@{@"key" : @"value"}];
		
		//2.
		*errorP = error;
		
		return NO;
	} else {
		return YES;
	}	
}
```

在ARC下，其实编译器会将`removeItemAtPath:error:`的方法声明变成如下:

```objc
- (BOOL)removeItemAtPath:(NSString *)path error:(NSError * __autoreleasing *)error;
```

其中 `NSError * __autoreleasing *error` 表示将 `*error`指向的对象放入到自动释放池管理。

> 将对象指针取地址后赋值给二重指针时，必须保证两个指针的对象所有权修饰符必须一致，否则编译器报错.

```objc
Dog *dog = nil;
Dog **dogP = &dog;//编译器报错: Pointer to non-const 'Dog *' with no explicit ownership
```

改为如下就通过编译了


```objc
Dog *dog = nil;// __strong
Dog * __strong *dogP = &dog;
```

```objc
Dog __autoreleasing *dog = nil;
Dog * __autoreleasing *dogP = &dog;
```

```objc
Dog __weak *dog = nil;
Dog * __weak *dogP = &dog;
```

那就有一个疑问了，编译器会自动对二重指针加入`__autoreleasing`来修饰，那么调用方法时，我们传入的却是一个 `__strong`强引用的对象指针，那为什么编译器不报错了？

```objc
- (BOOL)removeItemAtPath:(NSString *)path error:(NSError * __autoreleasing *)error;
```

比如调用如上方法的代码如下:


```objc
NSError *error = nil;//显示写为 NSError __strong *error = nil;

[[NSFileManager defaultManager] removeItemAtPath:@"" error:&error];
```

但是居然是不报错的，其实编译器将如上的调用代码改成如下:

```objc
NSError *error = nil;

//1. 
NSError __autoreleasing *temp = error;

//2.
[[NSFileManager defaultManager] removeItemAtPath:@"" error:& temp];
```

那么可以将二重指针的对象所有权修改为`__strong`，那不就可以省去上面的多余的转换权限步骤吗？


```objc
- (BOOL)removeItemAtPath:(NSString *)path error:(NSError * __strong *)error;
```

- `*error` 指向的对象，不会被注册到自动释放池管理
- 因为最终传递的NSError对象，是如上方法内创建，也就是如上方法去持有
- 那如果调用这个方法的地方，最终获取到被填充的NSError对象，自己并不是这个NSError对象的`持有者`
	- 能够被持有的对象，只能通过`new/alloc/copy/mutableCopy`开头的方法得到的对象
- 如果不注册到释放池，就有可能使用的过程中突然被废弃掉


在ARC下，对一个返回值是对象的方法名和之前非ARC环境下的规则是一样的。如果一个方法的返回对象，就是被方法调用者所持有，那么该方法的名字必须是以如下四种其中一种作为前缀:

- new
- alloc
- copy
- mutableCopy

在ARC下，对`initXxx`函数的命名也是有规则的:

- 必须是对象的方法
- 必须返回一个对象
- 返回的对象不会被注册到autorelease pool
- 只会对返回的对象做一些初始化

####在ARC下，显示的转换 `id（Objc类型）` 与 `void*（c结构体类型）` 类型的指针:

```objc
NSObject *obj;
void *p;

p = obj;

obj = p;
```

如上代码在ARC下会报错，在非ARC下是可以通过编译的。那么在ARC下，必须要`显示`的进行类型转换:

```objc
id obj = [NSObject new];

// id ---> void*
void *p1 = (__bridge void *)(obj);

// void* ---> id
id p2 = (__bridge id)(p1);
```

`__bridge`只是简单的将`objc类型数据 <<--->> CF结构体类型数据`进行数据类型转换，不会涉及到`retain`或`release`，但是下面的两种就会涉及到:

- `__bridge_retained` >>> 将OC对象转换成CF对象，并对OC对象做一次retain操作

```objc
void *p = (__bridge_retained <CF type>)(<expression>);
```

- `__bridge_transfer` >>> 将CF对象转换成OC对象，并对CF对象做一次release操作

```objc
NSObject *obj = (__bridge_transfer <Objective-C type>)(<expression>);
```


比如，`__bridge_retained` 使用的例子


```objc
id obj = [NSObject new];
void *p1 = (__bridge_retained void *)(obj);
```

如上的代码，在非ACR下时

```objc
//1. 一个OC对象
id obj = [NSObject new];

//2. 将 OC对象 >>> CF对象
void *p = obj;

//2. 最后对OC对象做一次retain
[(id)p retain];
```

下面看两个代码看`__bridge_retained`与`__bridge`的区别:

```objc
void *p = NULL;

NSLog(@"1");

{
	NSLog(@"2");
	
	id obj = [Dog new];
	
	NSLog(@"3");
	
	//1. 普通转换
	p = (__bridge void *)(obj);
	NSLog(@"4");
}

NSLog(@"5");

NSLog(@"class = %@", [(__bridge id)(p) class]);//这一句可能会崩溃

NSLog(@"6");
```

运行结果

```
2016-07-07 00:03:50.559 Demos[5402:68737] 1
2016-07-07 00:03:50.560 Demos[5402:68737] 2
2016-07-07 00:03:50.560 Demos[5402:68737] 3
2016-07-07 00:03:50.560 Demos[5402:68737] 4
2016-07-07 00:03:50.560 Demos[5402:68737] Dog对象: <Dog: 0x7f87b3e47a30> 被废弃
2016-07-07 00:03:50.560 Demos[5402:68737] 5
2016-07-07 00:03:50.560 Demos[5402:68737] class = Dog
2016-07-07 00:03:50.561 Demos[5402:68737] 6
```

可以看到，普通转换的对象是不能被持有的，不能持有就无法控制其废弃。

那么改用`__bridge_retained`的代码:

```objc
void *p = NULL;

NSLog(@"1");

{
    NSLog(@"2");

    id obj = [Dog new];//retainCount = 1

    NSLog(@"3");

    //2. 转换后进行retain，从而持有这个对象
    p = (__bridge_retained void *)(obj);//retainCount == 2

    NSLog(@"4");
    
}//retainCount == 1

NSLog(@"5");

NSLog(@"class = %@", [(__bridge id)(p) class]);

NSLog(@"6");
```

输出

```
2016-07-07 00:08:59.111 Demos[5675:73305] 1
2016-07-07 00:08:59.111 Demos[5675:73305] 2
2016-07-07 00:08:59.112 Demos[5675:73305] 3
2016-07-07 00:08:59.112 Demos[5675:73305] 4
2016-07-07 00:08:59.112 Demos[5675:73305] 5
2016-07-07 00:08:59.112 Demos[5675:73305] class = Dog
2016-07-07 00:08:59.112 Demos[5675:73305] 6
```

可以看到局部创建的Dog对象就算出了作用域，也没有被废弃掉，因为最终Dog对象的retainCount == 1， 所以没有被废弃。注意，上面例子只是为了说明`__bridge_retained`的区别，但是不要像上面这样写，会导致对象不会被废弃的。


> 那么`__bridge_retained` 会将被转换的对象进行retain操作，那么相反就有另外一种将对象进行转换时进行release操作的 >>> `__bridge_transfer`。负责将CF对象转换成OC对象，并对CF对象做一次release操作。


```objc
- (void)demo6 {

    void *p = NULL;
    id obj = [Dog new];//retainCount = 1
    
    //p = (__bridge void *)(obj);如果简单的转换，对象的retainCount没有+1，retainCount = 1
    p = (__bridge_retained void *)(obj);//retainCount = 2
    
    NSLog(@"1");
    
    {
        NSLog(@"2");
        id temp = (__bridge_transfer id)(p);//retainCount = 1
        NSLog(@"3");
        [temp test];
        NSLog(@"4");
    }
    NSLog(@"5");
    
}//retainCount = 0
```

如上例子中，如果使用`p = (__bridge void *)(obj);`，那么重新运行上面代码，就会崩溃到最后函数执行完毕的时候。因为会执行两次dealloc方法，造成程序崩溃。

####Objective-C对象 与 CoreFoundation对象

- Foundation: Objective-C语言版本的对象.
	- 持有对象 >>> [OC对象 retain];
	- 释放对象 >>> [OC对象 release];

- CoreFoundation: c语言版本的结构体实例.
	- 持有对象 >>> CFRetain(CF实例);
	- 释放对象 >>> CFRelease(CF实例);

- 两个类型数据通过 `__bridge`、`__bridge_retained`、`__bridge_transfer`进行转换
	- `__bridge`: OC对象 与 CF实例 之间相互转换，只是简单的类型转换，不涉及retain/release
	- `__bridge_retained`:  OC对象 >>>> CF实例，并对象OC对象进行retain
	- `__bridge_transfer`:  CF实例 >>>> OC对象，并对CF实例进行release

除了使用上面的三种修饰符来进行OC对象与CF实例转换，还有CoreFoundation提供的一些c函数，c函数里面其实就是使用了三种修饰符:

- OC对象转CF对象，并进行retain

```c
CFTypeRef __nullable CFBridgingRetain(id __nullable X) {
    return (__bridge_retained CFTypeRef)X;
}
```

```c
id obj = [NSObject new];
CFTypeRef cfObj = CFBridgingRetain(obj);
```

```c
- (void)demo7 {
    
    CFMutableArrayRef cfArray = NULL;
    
    {
        id ocArray = [[NSMutableArray alloc] init];//retainCount = 1
        
        cfArray = (CFMutableArrayRef)CFBridgingRetain(ocArray);//retainCount = 2
        
        CFShow(cfArray);
        printf("局部块内 retain count = %ld\n", CFGetRetainCount(cfArray));
        
    }//retainCount = 1
    
    printf("局部块外 retain count = %ld\n", CFGetRetainCount(cfArray));
    
    CFRelease(cfArray);//retainCount = 0
	cfArray = NULL;//此时这个c指针成为危险指针，如果继续使用操作数组就会造成程序崩溃
}
```

运行结果

```
(
)
局部块内 retain count = 2
局部块外 retain count = 1
```

- CF实例转OC对象，并进行release

```objc
- (void)demo8 {

    CFMutableArrayRef cfArray = CFArrayCreateMutable(kCFAllocatorDefault, 0, NULL);//retainCount = 1
    
    printf("retain count = %ld\n", CFGetRetainCount(cfArray));
    
    id ocArray = CFBridgingRelease(cfArray);//retainCount = 1 （首先右侧-1，但是左侧强引用持有+1，所以还是1）
    //id __strong ocArray = CFBridgingRelease(cfArray);//编译器最后将上面编译成这样的，强引用持有转换后的对象
    
    printf("retain count = %ld\n", CFGetRetainCount(cfArray));
    
    NSLog(@"class = %@", [ocArray class]);
}
```

运行结果

```
retain count = 1
retain count = 1
2016-07-08 00:16:43.149 Demos[4551:68833] class = __NSCFArray
```

最后小结，建议使用CFBridgingRetain()与CFBridgingRelease()来进行OC对象与CF实例的互相转换，而避免使用`__bridge`简单的进行转换，可能造成对象retainCount != 0，从而不会被释放出现内存泄露。

####属性 @property(修饰符1, 修饰符2, ...) 类型 *obj

- `属性`的修饰符种类（除开 nonatomic、atomic）:
	- assign
	- copy
	- strong
	- retain
	- unsafe_unretained
	- weak

- 那么前面讲述的`对象所有权`修饰符种类:
	- `__strong`
	- `__weak`
	- `__unsafe_unretained`

- 实际上这两种修饰符是有关联的，可以说`属性`修饰种类借助`对象所有权`修饰:

| 属性的内存管理修饰符 | 对象所有权修饰符 | 
| :-------------: |:-------------:| 
| assign | `__unsafe_unretained` | 
| copy | `__strong`（首先是拷贝原始对象得到一个新的对象，然后再强引用新的对象） | 
| retain | `__strong` | 
| strong | `__strong` | 
| unsafe_unretained | `__unsafe_unretained` | 
| weak | `__weak` | 


在属性的修饰符中，除了`copy`是先执行`对象的拷贝（实现NSCopying协议）`在执行强引用。其他的属性修饰符，都是直接赋值。

- 对于属性的修饰符 与 对象成员变量的修饰符 必须保持一致，否则编译器会报错

```objc
#import <Foundation/Foundation.h>

@interface Person : NSObject

//weak修饰
@property (nonatomic, weak) id obj;

@end
```

```objc
#import "Person.h"

@implementation Person {
	
	// 如下两句都会报错
    id _obj;
    //默认就是 id __strong _obj;
}

@end
```

####ARC的实现，首先是借助于`编译器进行内存代码自动添加`，然后是`运行时`的一些额外处理

- 编译器: clang(LLVM编译器) 3.0 以上
- 运行时环境: Objective-C运行时 493.9 以上


由于直接使用苹果的开源代码比较多，就按照书上给的一些伪代码，明白其原理即可。


####使用 `__strong` 修饰的对象指针是如何工作的？

比如，如下objc代码对应的编译器生成的c代码

```objc
{
	NSObject __strong *obj = [[NSObject alloc] init];
}
```

```c
id obj = objc_msgSend(NSObject, @selector(alloc));
objc_msgSend(obj, @selector(init));
objc_release(obj);//由编译器自动插入的内存管理代码
```

再看如下OC代码的编译器生成的c伪代码:

```objc
{
	id __strong obj = [NSMutableArray array];
}
```

前面说，只有以`new/alloc/copy/mutableCopy`开头的方法返回的对象，才是当前环境自己持有。那么对于`[NSMutableArray array]`明显返回的并不是当前环境自己持有的对象，这种情况下的得到的对象，只能通过如下方式去持有:

- 将得到的对象注册到一个自动释放池pool中去持有
- 直接当做函数返回值


```c
id obj = objc_msgSend(NSMutableArray, @selector(array));//取的别人生成的对象
objc_retainAutoreleasedReturnValue(obj);//由编译器自动插入的内存管理代码
objc_release(obj);
```

那么如上使用的`objc_retainAutoreleasedReturnValue()`函数，是做设么的？

- 拆开来看 >>> retain  autoreleased  returnValue 
- 第一个单词 retian >>> 就是去`持有`一个对象
- 第二个单词 autoreleased >>> 被注册到自动释放池管理的对象
- 第三个单词 returnValue >>> 方法执行后的`返回值`


那么很明显了，该函数就是去持有一个方法返回的对象。但是后面的两个单词如何解释，具体作用如何还得看下面的的伪代码实现.

- NSMutableArray的类方法array的伪代码:

```objc
+ (id)array {

	//1. 生成一个数组对象
	id obj = objc_msgSend(NSMutableArray, @selector(alloc));
	
	//2. 执行对象的初始化init方法
	obj = objc_msgSend(obj, @selector(init));
	
	//3. 返回一个
	return objc_autoreleaseReturnValue(obj);//由编译器自动插入的内存管理代码
}
```

注意到最后的一行代码使用到的函数`objc_autoreleaseReturnValue()`，干什么的了，那么这个函数会做如下几件事:

- (1) 检测`函数调用方`在调用了`当前所在函数（如: array方法)`代码的后面，是否紧接着`objc_retainAutoreleasedReturnValue()`函数使用代码

- (2) 如果调用方在后面使用到了`objc_retainAutoreleasedReturnValue()`函数，那么在调用方取得返回的对象之后，`不会将返回对象注册到自动释放池`，而是直接传递到调用者所在方法中

OK，那么现在可以小结下`objc_retainAutoreleasedReturnValue()`的作用，那么就必须联系到上面的`objc_autoreleaseReturnValue()`函数:

- `objc_retainAutoreleasedReturnValue()` 与 `objc_autoreleaseReturnValue()` 是 `成对` 出现的
	- `objc_retainAutoreleasedReturnValue()`: 出现在调用方代码
	- `objc_autoreleaseReturnValue()`: 出现在被调函数中代码

- `objc_retainAutoreleasedReturnValue()`的作用:
	- 首先是去持有一个方法返回的对象，但是分为两种持有方式
	- 对象持有方式一、将返回的对象注册到一个`自动释放池`去持有
	- 对象持有方式二、直接通过`传递`的方式获取对象（不会进行释放池及注册）

- `objc_autoreleaseReturnValue()`的作用:
	- 首先监测`函数调用方`，是否在取得了函数放回对象之后，紧接着使用了`objc_retainAutoreleasedReturnValue()`
	- 如果使用了，就不会使用上面的`对象持有方式一`，而是使用`对象持有方式二`来直接传递返回对象

通过如上两个函数的配合使用，可以不用将返回的对象注册到autoreleasepool中去持有管理，而是直接通过`传递`的方式，达到了性能上的优化。

####使用 `__weak` 修饰的对象指针是如何工作的？

涉及到的两个点:

- (1) weak指针变量与被指向对象之间如何关联的
- (2) weak指针指向的对象会自动注册到释放池，防止使用时被废弃

先小结下之前的关于`__weak`对象所有权的作用:

- 当指向的对象被废弃掉时，指针自动设置为`nil`，避免程序野指针崩溃
- 指向的对象被使用时，自动注册到自动释放池

再看下一段`__weak`使用的OC代码翻译成的c伪代码:

```objc
{
	id __weak obj1 = obj;//假设obj在外面是通过 __strong修饰的指针指向
}
```

对应的c伪代码

```c
id obj1;
objc_initWeak(&obj1, obj);
objc_destroyWeak(&obj1);
```

上面看到两个关于weak的函数，`objc_initWeak()` 与 `objc_destroyWeak()` :

- `objc_initWeak()`函数:
	- 初始化使用了`__weak`修饰的指针变量
	- 作为变量存活作用域的`开始`

- `objc_destroyWeak()`函数:
	- 作为变量存活作用域的`结束`
	- 对变量指向的对象进行`释放`release（不是废弃）

	
- 如上两个函数内部又都使用到了另一个函数: `objc_storeWeak()`

比如，`objc_initWeak(&obj1, obj);`中使用`objc_storeWeak()`的伪代码


```c
obj1 = 0;//初始化地址0
objc_storeWeak(&obj1, obj);//传入的obj对象指针
```

而`objc_destroyWeak(&obj1);`中使用`objc_storeWeak()`的伪代码


```c
objc_storeWeak(&obj1, 0);//传入的是地址0
```

那么前面的如下代码，使用上面的`objc_storeWeak()`函数展开后代码:

```c
id obj1;
objc_initWeak(&obj1, obj);
objc_destroyWeak(&obj1);
```

展开后的伪代码

```c
id obj1;

//1. objc_initWeak()展开
obj1 = 0;
objc_storeWeak(&obj1, obj);

//2. objc_destroyWeak()展开
objc_storeWeak(&obj1, 0);
```

结合如上分析，之前的代码的逻辑如下:

```
id __strong obj = ....;//强引用对象

{
	id __weak weakObj = obj;//初始化weak指针变量，作用域开始
	
	//操作weak指针变量
	//....
	
	//不再对象weak指针变量进行操作
	//结束weak指针变量作用域，销毁weak指针变量，赋值为nil
}
```

稍微了解下`objc_storeWeak(参数一地址, 参数二变量)`函数的作用:

- 首先是有一个Weak表（类似引用计数表），用来记录`参数二变量` 与 `参数一地址`的对应关系
	- 可以是 1:n的关系
	- key: 参数二变量的`地址`
	- value: 参数一地址

- 如果第二个参数变量是`0`，则会将`参数二变量的地址`对应存放在Weak表中的`参数一地址`删除

- 通过Weak表，可以快速查询到`参数二变量（对象）` 对应的 `使用 __weak修饰的指针变量的地址`

当指向的对象被废弃掉时:

- (1) 从weak表中，获取被废弃对象的地址对应的weak指针变量
- (2) 将找到的weak指针变量赋值为nil
- (3) 然后从`weak表`中`删除`废弃对象地址对应的记录项
- (4) 最后从`引用计数器表`中`删除`废弃对象地址对应的记录项

那么可以看到，对`__weak`修饰的指针变量，ARC系统会做这么多的步骤，所以肯定会消耗一定的cpu资源。

> 所以，应该只在需要解决 `循环强引用` 的情况下，才去使用 __weak修饰指针变量.


而且最重要的是，`__weak`修饰的指针不能够持有一个对象，随时可能被废弃掉。

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    NSLog(@"1");
    
    {
        NSLog(@"2");
        Dog __weak *dog = [Dog new];//Dog对象创建之后，立马被废弃了
        NSLog(@"3, dog = %@", dog);
    }
    
    NSLog(@"4");
}
```

运行结果

```
2016-07-10 19:07:09.299 Demos[8134:88862] 1
2016-07-10 19:07:09.300 Demos[8134:88862] 2
2016-07-10 19:07:09.300 Demos[8134:88862] Dog对象: <Dog: 0x7fcc3070a990> 被废弃
2016-07-10 19:07:09.300 Demos[8134:88862] 3, dog = (null)
2016-07-10 19:07:09.300 Demos[8134:88862] 4
```

可看到`__weak`修饰的指针指向的对象，被创建之后立马又被废弃掉了，为什么了?

- 因为`__weak`指针无法对被指向的对象，形成一个有效的`强引用`指向，即无法让`retainCount++`

- 当执行完`Dog __weak *dog = [Dog new];`之后，被创建出来的Dog对象，没有一个`强引用指针`指向，所以被立即`废弃`掉了


那么可以小结`__weak`修饰的指针的缺点:

- `无法` 让被指向对象的 `retainCount + 1`，即不能持有一个对象

- weak指针变量，只有很短的生存作用域
	- 声明weak指针，并指向一个对象 >>> 作用域`开始`
	- 执行完所有的weak指针操作 >>> 作用域`结束`
	- 一旦weak指针的作用域结束，就会对weak指针指向的对象进行`释放`操作
	
注意: `__unsafe_unretained`同`__weak`一样的，不能够持有一个对象。

> 使用 __weak修饰的指针所指向的对象，会自动注册到自动释放池.

前面的例子代码

```objc
{
	id __weak obj1 = obj;//假设obj在外面是通过 __strong修饰的指针指向
}
```

可以看到其，最终使用`Weak表`来存放原始对象地址与weak指针变量的对应关系的这个功能。

但是还有一个功能，就是还有`__weak`修饰的指针所指向的对象，会自动注册到自动释放池。

如上的例子，`没有被注册到自动释放池（没有出现任何有关释放池的代码）`，因为没有操作`obj1`，那如果加上操作`obj1`之后的代码:

```objc
{
	id __weak obj1 = obj;//假设obj在外面是通过 __strong修饰的指针指向
	NSLog(@"obj1 = %@", obj1);//操作obj1指向的对象
}
```

然后对上面修改的代码进行clang反编译，可以得到如下的c伪代码:

```c
id obj1;

//1. 初始化weak指针变量
objc_initWeak(&obj1);

//2. 【重要】释放池相关代码1
id tmp = objc_loadWeakRetaind(&obj1);

//3.【重要】释放池相关代码2
objc_autorelease(tmp);

//4.
NSLog(@"obj1 = %@", obj1);

//5. 操作完weak指针变量之后，立马进行释放（与步骤2对应）
objc_destoryWeak(&obj1);
```

可以看到比之前多出了两行代码，而是一看就是与释放池相关的:

```c
id tmp = objc_loadWeakRetaind(&obj1);
objc_autorelease(tmp);
```

那么看看如上两个函数的作用:

- `objc_loadWeakRetaind(weak指针变量的地址)`
	- (1) 从`Weak表`查询到`__weak`修饰的`指针变量`所引用的`对象`
	- (2) 将找到的对象进行`retain`

- `objc_autorelease(对象指针变量)`
	- (1) 将传入的对象注册到一个自动释放池

	
由此可以得到，直接使用`__weak`指针指向的对象时，会自动将所指向的对象放入自动释放池去管理。所以，如果大量的直接使用`__weak`指针变量，就会将大量的对象都注册到释放池，从而增加cpu的消耗。

那么，再看如下两种写法。

写法一:

```objc
id __strong obj = ....;//强引用对象

{
	//1. 弱引用指向外界的对象 
	id __weak weakObj = obj;
	
	//2. 多次操作弱引用指针
	NSLog(@"obj1 = %@", weakObj);//操作weakObj指向的对象第一次
	NSLog(@"obj1 = %@", weakObj);//操作weakObj指向的对象第二次
	NSLog(@"obj1 = %@", weakObj);//操作weakObj指向的对象第三次
	NSLog(@"obj1 = %@", weakObj);//操作weakObj指向的对象第四次
	NSLog(@"obj1 = %@", weakObj);//操作weakObj指向的对象第五次
}
```

写法二:

```objc
id __strong obj = ....;//强引用对象

{
	//1. 弱引用指向外界的对象 
	id __weak weakObj = obj;
	
	//2. 再强引用弱引用指针指向的对象
	id __strong strongObj = weakObj;
	
	//3. 最后多次操作强引用指针，来减少对象注册到释放池的操作
	NSLog(@"obj1 = %@", strongObj);//操作strongObj指向的对象第一次
	NSLog(@"obj1 = %@", strongObj);//操作strongObj指向的对象第二次
	NSLog(@"obj1 = %@", strongObj);//操作strongObj指向的对象第三次
	NSLog(@"obj1 = %@", strongObj);//操作strongObj指向的对象第四次
	NSLog(@"obj1 = %@", strongObj);//操作strongObj指向的对象第五次
}
```

`写法二`只会`执行一次`将对象注册到自释放池，而`写法一会执行5次`。那么所以，我们经常在block内部使用外面weak，内部strong的写法如下，意思也正是如此:

```objc
__weak __typeof(self)weakSelf = self;
void (^block)(void) = ^() {
	__strong __typeof(weakSelf)strongSelf = weakSelf;
	
	//后面都是操作	strongSelf 这个强引用指针.
	//....
};
```

那么所以，block内部的`强引用指针`也正是为了只让所指向的对象注册到自动释放池`一次`。


####使用 `__autoreleasing` 修饰的对象指针是如何工作的？

将对象赋值给具有`__autoreleasing`修饰的指针变量时，等价于在非ARC时使用`-[NSObject autorelease]`方法。

通过如下的代码进行clang反编译:

```objc
@autoreleasepool {//开始一个释放池
    id __autoreleasing obj = [[NSObject alloc] init];//将obj对象注册到释放池中
}
```

反编译出来的c伪代码大概如下:


```c
//1. 使用一个释放池
id pool = objc_autoreleasePoolPush();

//2. 创建对象、以及初始化
id obj = objc_msgSend(NSObject, @selector(alloc));
objc_msgSend(obj, @selector(init));

//3. 将对象放入释放池
objc_autorelease(obj);

//4. 
objc_autoreleasePoolPop(pool);
```

再看看NSMutableArray的array方法返回对象也放入释放池的反编译伪代码:

```objc
@autoreleasepool {
    id __autoreleasing obj = [NSMutableArray array];
}
```

```c
//1. 使用一个释放池
id pool = objc_autoreleasePoolPush();

//2. 创建对象、以及初始化
id obj = objc_msgSend(NSObject, @selector(array));

//3. 声明直接传递返回值对象，而不是注册到释放池
objc_retainAutoreleasedReturnValue(obj);//比上面优化的地方

//4. 
objc_autoreleasePoolPop(pool);

//5. 
objc_autoreleasePoolPop(pool);
```
***

###终于开始第二部分了、Blocks

看到第二部分，看了大概两三周，天天下班回来看半个多小时...

> Block内部截获外部的变量值.

很久之前，去腾讯面试，当时出了一个关于block的题目，没有回答出来.

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    int x = 1;
    
    void (^block)() = ^() {
        NSLog(@"x = %d", x);
    };
    
    x = 2;
    
    block();
}
```

当时我直接回答的2，然后我又说可能释放了....哎，当时我是多么的菜逼啊。

上面代码运行后结果:

```
2016-07-10 23:13:34.174 Demos[13195:158793] x = 1
```

结果其实仍然还是1，并不是2。后来我才知道原因，不过看了这本书之后更加清楚了。

Block有几点要搞清楚:

- (1) Block对外部变量值会自动进行捕获
- (2) Block捕获的外部变量/对象的一份`拷贝`值，并不是原来的内存地址

如果，将上述代码稍加修改如下:

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    __block int x = 1;
    
    void (^block)() = ^() {
        NSLog(@"x = %d", x);
    };
    
    x = 2;
    
    block();
}
```

运行结果

```
2016-07-10 23:17:16.720 Demos[13396:162015] x = 2
```

通过`__block`修饰，表示传递的是原始数据的内存地址，而不是拷贝数据。

所以也就是说，Block默认情况下，截获变量在`Block表达式` 之前 的一份`拷贝`数据，并非数据本身，后面即使数据发生改变，也无法反应到Block内部。

在Objective-C的`运行时`框架中，我们平时经常写的NSObejct类，在App程序正在在iOS系统中运行时，都会生成一个与之对于的`objc_class`结构体的`单例`来记录我们编写的NSObject类的所有信息:

- (1) 对象成员变量 Ivar
- (2) 所有的属性 Property
- (3) 对象方法 Method
- (4) 实现的所有协议 Protocol
- (5) super_class父类...等等信息

> Block可以看做成一个对象.

通过书上，跟着他的步骤我也尝试使用clang反编出了Block的结构，其实Block也是有数据结构的:

```c
struct __main_block_impl_0 {

  void *isa;//指针，占用四个字节
  
  int Flags;//32位占用2字节、64位占用4字节
  
  int Reserved;//32位占用2字节、64位占用4字节
  
  void *FuncPtr;//函数指针，占用四个字节
};
```


可以看到，里面有一个`void *isa`指针，很类似`objc_object`、`objc_class`的结构:

```c
typedef struct objc_object {
    Class isa;//isa指向所属类 objc_class实例
} *id;
```

```c
typedef struct objc_class {  
    Class isa;//isa指向元类，是另外一个objc_class实例
    
    Class super_class;  
    const charchar *name;  
    long version;  
    long info;  
    long instance_size;  
    struct objc_ivar_list *ivars;  
    struct objc_method_list **methodLists;  
    struct objc_cache *cache;  
    struct objc_protocol_list *protocols;  
  
} *id;
```

那么看下来，可以发现`__main_block_impl_0`其实就和`objc_object`的结构很相似。

然后Block结构体的isa指针，反编译出的代码对其初始化代码:

```c
isa = &_NSConcreteStackBlock;
```

那么可以猜测，`_NSConcreteStackBlock`就是Block这个`对象`所属的`类`即（等同objc_class实例），那么Block作为`对象`的所有信息，都存放在`类 _NSConcreteStackBlock结构体实例`中。

> 使用 __block修饰的变量或指针，是如何在Block内部传递的？

可以对如下代码使用clang进行反编译

```objc
__block int val = 10;

void (^block)() = ^() {
	val = 1;
};
```


首先，如下这一行代码

```objc
__block int val = 10;
```

对应反编译出来的一些相关代码

```c
typedef struct __Block_byref_val_0 {
	void *_isa;
	__Block_byref_val_0 *__forwarding;
	int __flags;
	int __size;
	int val;
};
```

```c
__Block_byref_val_0 val = {
	0,
	&val,
	0,
	sizeof(__Block_byref_val_0),
	10
};
```

可以得到如下几点:

- 由编译器自动生成了一个c结构体类型 `__Block_byref_val_0` 
- 然后在 `栈上` 创建了一个`__Block_byref_val_0` 结构体实例
- 然后对该 `__Block_byref_val_0` 结构体实例 进行赋值

那么就是说，`__block修饰的变量` 被做成了一个`栈上`存在的`c结构体实例`，而该结构体实例同时也保存着变量的值`10`。这个过程就相当于是`自动变量`的截获，只是多了一个创建结构体实例的过程。

然后是Block内部对val变量赋值语句对应的反编译代码:

```objc
^() {
	val = 1;
};
```


```c
static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
	//1.
	__Block_byref_val_0 *val = __cself->val;
	
	//2.
	(val->__forwarding->val) = 1;
}
```

从上面函数代码可以得到:

- (1)、`__main_block_impl_0实例`的val属性（编译器添加的）指向之前的`__blcok变量对应的结构体（编译器生成）`实例

- (2)、`__Block_byref_val_0实例`--->`__forwarding`来指向的之前的`__block变量`结构体实例

那么上面的`val->__forwarding`这句代码，我是比较不太明白，这不是饶了一圈又回到`__Block_byref_val_0`实例了吗？

该书在本章节并未给出答案。

> Block的存储区域.

回顾下前面的分析中，可以看到Block对前面出现的`__block修饰的变量`进行捕获时，有如下几个关键点:

- (1) 由编译器生成一个与`__block修饰的变量`相匹配的c结构体类型
- (2) 在系统`栈区`创建一个实例，来保存这个`__block修饰的变量`的值数据
- (3) 然后Block对应的结构体`__main_block_impl_0 `实例，指向上面创建的结构体实例
- (4) 以后直接操作`Block结构体实例`中的`__block修饰的变量结构体实例`的属性值

可以对如上步骤小结下，Block实例 与`__block修饰的变量结构体实例` 的内存存放地点

| 名称 | 内存存放地点 |
| :-------------: |:-------------:|
| Block结构体实例 | 栈 |
| `__block修饰的变量`（被生成的对应结构体实例） | 栈 |


查看我们写的Block在运行时真正的类型:

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    void (^block)() = ^() {
        NSLog(@"");
    };
    
    或
    
//    void (^block)() = [^() {
//        NSLog(@"");
//    } copy];
    
    Class cls = [block class];
    NSLog(@"%@ - %p", cls, cls);
    
    cls = object_getClass(cls);
    NSLog(@"%@ - %p", cls, cls);
    
    while (!class_isMetaClass(cls)) {
        cls = object_getClass(cls);
        NSLog(@"%@ - %p", cls, cls);
    }
}
```

运行结果

```
2016-07-11 15:42:12.507 demo[9184:1282723] __NSGlobalBlock__ - 0x10a6d5450
2016-07-11 15:42:12.507 demo[9184:1282723] __NSGlobalBlock__ - 0x10a6d54d0
```

可以看到，此时的Block作为对象时，他所属的类型是`__NSGlobalBlock__`，以双下划线开头的类名，基本上都是`私有类`。还有，一样和之前的类具有`类自己`和`Meta类`两个`objc_class`结构体实例。

但是书上通过clang反编译出的Block类型是这样的三种，其字面的意思也很明显:

- (1) _NSConcreteGlobalBlock >>> `全局`存在的Block对象，不会访问任何外部变量，这种block不存在于Heap或是Stack而是作为代码片段存在，类似于C函数

- (2) _NSConcreteStackBlock >>> `栈`上存在的Block对象，当函数返回时会被销毁

- (3) _NSConcreteMallocBlock >>> `堆`上存在的Block对象，只有当引用计数为0时会被销毁

但是貌似和我测试代码log出来的`__NSGlobalBlock__`一点都不一样。但是仔细一看，还是有详细点:

| 名称1 | 名称2 |
| :-------------: |:-------------:|
| `_NSConcreteGlobalBlock` | `__NSGlobalBlock__` |

都是代表`Global`，全局的意思。

然后我网上查了下，原来`全局Block`产生的条件: 

- (1) 该Block必须不能对外界的变量进行捕捉（引用）

但是可以去操作一个c函数，但是不能够去对变量进行捕获:

```objc
static NSString * test() {
    return @"haha";
}

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

    void (^block)() = ^() {
    
    	//1. 操作c函数这是可以的
        NSString *ret = test();
        NSLog(@"ret = %@", ret);
        
        //2. 如果只选OC对象函数，就会成为堆Block
        //[self doWork];
    };
    
    Class cls = [block class];
    NSLog(@"%@ - %p", cls, cls);
    
    cls = object_getClass(cls);
    NSLog(@"%@ - %p", cls, cls);
    
    while (!class_isMetaClass(cls)) {
        cls = object_getClass(cls);
        NSLog(@"%@ - %p", cls, cls);
    }
}

@end
```

OK，那么将上面的log代码改成对外界变量引用再试试:

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    int x = 0;
    void (^block)() = ^() {
        NSLog(@"x = %d", x);
    };
    
    Class cls = [block class];
    NSLog(@"%@ - %p", cls, cls);
    
    cls = object_getClass(cls);
    NSLog(@"%@ - %p", cls, cls);
    
    while (!class_isMetaClass(cls)) {
        cls = object_getClass(cls);
        NSLog(@"%@ - %p", cls, cls);
    }
}
```

这次运行结果

```
2016-07-11 15:52:31.989 demo[9302:1327669] __NSMallocBlock__ - 0x1117d2150
2016-07-11 15:52:31.990 demo[9302:1327669] __NSMallocBlock__ - 0x1117d21d0
```

这次的Block对象的类型就是`__NSMallocBlock__`了，其字面意思就很明显告诉说这个Block对象是分配在`堆`上的。


再看看如果Block操作的是`__block修饰的变量`:

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    __block int x = 0;
    void (^block)() = ^() {
        NSLog(@"x = %d", x);
        x = 2;
    };
    
    Class cls = [block class];
    NSLog(@"%@ - %p", cls, cls);
    
    cls = object_getClass(cls);
    NSLog(@"%@ - %p", cls, cls);
    
    while (!class_isMetaClass(cls)) {
        cls = object_getClass(cls);
        NSLog(@"%@ - %p", cls, cls);
    }
}
```

其运行结果

```
2016-07-11 15:56:35.314 demo[9340:1343541] __NSMallocBlock__ - 0x107e37150
2016-07-11 15:56:35.314 demo[9340:1343541] __NSMallocBlock__ - 0x107e371d0
```

> 结果一样是`堆`上分配的Block对象，但是我觉得这有点不太对劲啊。明明这个block我是在`方法内`部创建的一个`局部`的，为什么就成了`Malloc堆`上的Block对象了？

这时候突然想起Effective OC 2.0书上说过，在非ARC环境下时，Block对象却是是分为上面的三种:

- (1) _NSConcreteGlobalBlock
- (2) _NSConcreteStackBlock
- (3) _NSConcreteMallocBlock 

但是在ARC环境下时，间接屏蔽了`栈Block`，且对应的类型修改成如下:

- (1) `__NSGlobalBlock__`
- (2) `__NSMallocBlock__`

实际上是因为ARC系统，会自动生成`对栈Block对象进行copy拷贝到堆区`的内存管理代码。就是为了Block可能会被释放的问题。

但是我就有个问题了，既然局部创建的Block对象，会自动拷贝到堆的Block对象，那么Blcok内部引用的对象，会造成强引用而内存泄漏吗？

```objc
@interface Dog : NSObject

@property (nonatomic, copy) NSString *name;

- (void)test;

@end

@implementation Dog

- (void)dealloc {
    NSLog(@"Dog%@ - %p 被废弃", _name, self);
}

- (void)test {
    NSLog(@"name = %@", _name);
}

@end
```

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    //1.
    Dog *dog1 = [Dog new];
    dog1.name = @"1111";
    void (^block)() = ^() {
        [dog1 test];
    };
    block();
    
    //2.
    Dog *dog2 = [Dog new];
    dog2.name = @"2222";
    dispatch_async(dispatch_get_main_queue(), ^{
        [dog2 test];
    });
}
```

运行结果

```
2016-07-11 16:33:26.375 demo[9854:1517718] name = 1111
2016-07-11 16:33:26.376 demo[9854:1517718] Dog1111 - 0x7ffb017020e0 被废弃
2016-07-11 16:33:26.376 demo[9854:1517718] name = 2222
2016-07-11 16:33:26.376 demo[9854:1517718] Dog2222 - 0x7ffb01616d70 被废弃
```

可以看到，即使是堆Block对象，再执行完毕之后，内部引用的对象一样可以释放。根本原因就是，在堆Block对象执行完毕之后，会自动被ARC系统进行一次释放release，当其retainCount == 0时，就会被废弃掉。继而内部引用的对象，也会进行一次释放release。


再对如下的代码分下retainCount变化过程:

```objc
- (void)work {//1.

	//2.
	Dog *dog1 = [Dog new];
    dog1.name = @"1111";
   
   //3.
    void (^block)() = ^() {
    	
    	//4. 
        [dog1 test];
    };
    
    //5.
    block();
    
}//6. 
```

- (1) 最外面的OC函数work，运行时创建一个`objc_method`实例
- (2) 内部的创建的Dog对象，其retainCount == 1
- (3) 创建一个栈Block，然后被copy到堆区，这个Block对象的retainCount == 1
- (4、5) Block内部持有了Dog对象，此时Dog对象的retainCount == 2
- (6) 指向Dog对象的指针，超出作用域失效为nil，那么此时Dog对象的retainCount == 1
- (7) Block对象执行完毕之后，自动由ARC系统从堆上释放release，此时Block对象的retainCount == 0
- (8) 紧接着Dog对象又失去了强引用指向，Dog对象也会执行一次释放release，此时Dog对象的retainCount == 0

那么，如果执行栈Block拷贝到堆Block得代码:

```objc
- (void)work {
    
    Dog *dog1 = [Dog new];

    void (^block)() = ^() {
        NSLog(@"%p", dog1);
        [dog1 test];
    };
    
    // 如果把这一句作为耗时的异步回调时执行
    [Tools asyncTask:^() {
        block();
    }];
}
```

很有可能asyncTask方法传入的block，在work函数执行完毕的时候，才会被执行。那么此时如果该Block对象如果没有拷贝到堆区，那么该Block对象肯定已经被释放了。那么当asyncTask方法再去执行这个Block对象时，肯定是因为对象已经释放而造成程序崩溃退出。


栈Block拷贝到堆Block涉及的:

| 栈区 | 堆区 |
| :-------------: |:-------------:|
| 栈Block | 堆Block |
| 栈上的`__block`变量 | 堆上的`__block`变量 |

书上明确指出，当StackBlock对象被拷贝到HeapBlock对象后，该Block对象的isa指向会被替换:

- 如果是StackBlock

```objc
__main_block_impl_0实例->isa = __NSStackBlock__
```

- 如果被拷贝成HeapBlock

```objc
__main_block_impl_0实例->isa = __NSMallocBlock__
```

那么现在就有一个问题了，如果没有超出上面的`work函数实现`的范围时，此时我们使用的Block对象，既存在于栈内又存在于堆上。那么此时，该决定到底使用堆上还是栈上的？

- 对于`__block`修饰的变量的访问
	- `__block`修饰的变量，会由编译器生成一个对应的c结构体类型
	- 其创建的实例默认是在`栈`上
	- 其isa指针，指向所属的Block对象
	- 其`__forwarding`指针，指向的是被`拷贝到堆上` 的 `__block`修饰的变量对应的`新的结构体实例`
	- 如果操作`栈`上的`__block`变量 >>> 栈Block变量结构体实例.val = 10;
	- 如果操作`堆`上的`__block`变量 >>> 栈Block变量结构体实例-> `__forwarding`->val = 10;
	- 所以说，不管是在栈上or堆上，都是能够正确访问到`__block`变量的结构体实例来进行操作


- 使用`__block`修饰的变量的对应的结构体实例中的`__forwarding`的指向随着Block对象拷贝到堆上的变化图示

	- (1) 当还是StackBlock对象时，`__forwarding`指向自己
	- (2) 当将StackBlock对象拷贝出得到一个新的HeapBlock对象后
	- (3) `StackBlock对象->__forwarding = HeapBlock对象;`

![](http://i2.piimg.com/567571/df485992535bcfa7.png)


- 对于使用StackBlock对象 or HeapBlock对象，书上好像并未提及

```
等找到答案，再回头补
```

####触发将StackBlock对象拷贝成HeapBlock的场景:

- (1) 主动调用Block对象的copy方法
- (2) 将Block对象作为函数返回值时
- (3) 将Block对象赋值给使用`__strong`修饰的id指针变量或Block类型指针变量时
- (4) 调用系统提供的方法传入Block对象时
	- 包含`usingBlock`的Cocoa框架api
	- Grand Central Dispatch(GCD)的api

> Block造成的循环强引用.

对于下面的这样的Block直接强引用对象，是不会造成循环引用的，因为最后Block对象执行完毕，会自动被ARC系统进行release释放.

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    //1.
    Dog *dog1 = [Dog new];
    dog1.name = @"1111";
    
    void (^block)() = ^() {
        [dog1 test];
    };
    
    block();
    
    //2.
    Dog *dog2 = [Dog new];
    dog2.name = @"2222";
    
    dispatch_async(dispatch_get_main_queue(), ^{
        [dog2 test];
    });
}
```

运行结果

```
2016-07-11 19:01:19.249 demo[11152:1938553] name = 1111
2016-07-11 19:01:19.249 demo[11152:1938553] Dog1111 - 0x7ffee2f26670 被废弃
2016-07-11 19:01:19.249 demo[11152:1938553] name = 2222
2016-07-11 19:01:19.250 demo[11152:1938553] Dog2222 - 0x7ffee2c0c470 被废弃
```

可以看到，上面的Block强引用对象，并不会造成循环引用的。

但是下面的例子就会引起循环引用了。


```objc
@interface Dog : NSObject <NSCopying>

@property (nonatomic, copy) NSString *name;

@property (nonatomic, copy) void (^block)(void);

- (void)log;

@end

@implementation Dog

- (void)dealloc {
    NSLog(@"Dog%@ - %p 被废弃", _name, self);
}

- (void)log {
    NSLog(@"name = %@", _name);
}

- (id)copyWithZone:(NSZone *)zone {
    Dog *newDog = [Dog new];
    newDog.name = [_name copy];
    return newDog;
}

@end
```

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

	//1. 
    Dog *dog = [Dog new];
    
    //2. 
    dog.block = ^() {
        [dog log];//造成循环引用
    };
}
```

运行后不管点击多少次，都没有看到Dog对象的dealloc方法输出的信息，那么说明Dog对象没有被废弃掉，所以就是内存泄漏了。

那么来分析上上面那一段产生循环引用的代码中Dog对象的retainCount的变化过程:

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

	Dog *dog = [Dog new];//retainCount == 1
	
	dog.block = ^() {
	    [dog log];//Block对象又引用或了Dog对象，retainCount == 2
	};
	
}//超出作用域后，指针进行释放release后，retainCount == 1，Dog对象仍然不会被释放
```

所以最终超出方法范围之后，Dog对象的retainCount仍然为1，所以Dog对象无法废弃。就是 因为Dog对象的`block属性`指向的Block对象对Dog对象产生了一次强引用指向，让其retainCount+1。但是随着最后Dog对象的retainCount==1，那么Dog对象本身无法废弃，也就造成`block属性`指向的Block对象也同时无法废弃。这就是循环强引用的产生场景。

- Dog对象强引用Block对象
- Block对象也强引用Dog对象

尝试解决方法一、使用`__weak指针`修饰Block属性

```objc
@interface Dog : NSObject <NSCopying>

@property (nonatomic, copy) NSString *name;

// 属性修改为weak弱引用
@property (nonatomic, weak) void (^block)(void);

- (void)log;

@end
```

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    Dog *dog = [Dog new];
    dog.name = @"你是狗";
    
    dog.block = ^() {
        [dog log];//造成循环引用
    };
    
    dog.block();
}
```

运行结果

```
2016-07-11 19:20:23.351 demo[11403:1992061] 1
2016-07-11 19:20:23.352 demo[11403:1992061] 2
2016-07-11 19:20:23.352 demo[11403:1992061] 5
2016-07-11 19:20:23.352 demo[11403:1992061] 3
2016-07-11 19:20:23.352 demo[11403:1992061] name = 你是狗
2016-07-11 19:20:23.352 demo[11403:1992061] 4
2016-07-11 19:20:23.352 demo[11403:1992061] 6
2016-07-11 19:20:23.352 demo[11403:1992061] Dog你是狗 - 0x7fba71e0cd10 被废弃
```

貌似看起来是正常运行，但是Xcode编译器会出现黄色警告。前面说了，使用`__weak`修饰的指针，是无法去`持有`一个对象，也就是说无法让其保证不被释放。那么说，这种方法不是他还好。


尝试解决方法二、传递`__weak`修饰的指针给Block对象去持有

```objc
@interface Dog : NSObject

@property (nonatomic, copy) NSString *name;

// block对象属性仍然使用copy拷贝
@property (nonatomic, copy) void (^block)(void);

- (void)log;

@end
```

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    Dog *dog = [Dog new];
    dog.name = @"你是狗";
    
    //使用weak指针指向上面的Dog对象
    Dog __weak *weakDog1 = dog;
    //__weak __typeof(dog)weakDog2 = dog; 和上面的一样的意思
    
    dog.block = ^() {
        [weakDog1 log];
    };
    
    dog.block();
}
```

这样修改后的代码，Xcode不再有任何的黄色警告了，看来是编译器比较认同的一种解决Block循环引用的写法了。

> 使用`__weak`修饰的指针指向的对象时，会由编译器自动加上注册到自动释放池的代码。而且每一次使用`__weak`指针都会被注册一次。

那么，最好是在Block外面weak，Block里面strong，让Block内部使用weak指针时，只需要将对象注册到释放池一次。

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    Dog *dog = [Dog new];
    dog.name = @"你是狗";
    
    //使用weak指针指向上面的Dog对象
    Dog __weak *weakDog1 = dog;
    //__weak __typeof(dog)weakDog2 = dog; 和上面的一样的意思
    
    dog.block = ^() {
        
        // 会将weak指针指向的对象，加入到自动释放池
        Dog __strong *strongDog = weakDog1;
        
        // 使用storng引用后的weak指针
        [strongDog log];
    };
    
    dog.block();
}
```

运行结果

```
2016-07-11 23:08:55.390 Demos[1423:17307] Dog name = 你是狗
2016-07-11 23:08:55.391 Demos[1423:17307] Dog 你是狗 - 0x7f8b6ac1b550 被废弃
```

运行一样是正常的，但效率绝对比之前高很多。

尝试解决方法三、推翻前面两种使用weak指针的方法，Dog对象提供一个函数来清除循环引用

```objc
@interface Dog : NSObject

@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) void (^block)(void);

- (void)log;

/**
 *  提供函数来废弃Block对象
 */
- (void)clearBlocks;

@end

@implementation Dog

- (void)log {
    NSLog(@"Dog name = %@", _name);
}

- (void)dealloc {
    NSLog(@"Dog %@ - %p 被废弃", _name, self);
}

- (void)clearBlocks {
    _block = nil;//将属性指向的Block对象释放废弃掉
}

@end
```

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    Dog *dog = [Dog new];
    dog.name = @"你是狗";
    
    dog.block = ^() {
        [dog log];//但是这里Xcode仍然会有一个黄色警告，引起循环引用
    };
    
    dog.block();
    
    // 主动释放掉属性引用的Block对象，解除循环引用
    [dog clearBlocks];
}
```

虽然Xcode仍然会有一个黄色警告，引起循环引用，但实际上已经解决循环引用的问题了。运行结果:


```
2016-07-11 23:13:56.312 Demos[1607:20636] Dog name = 你是狗
2016-07-11 23:13:56.313 Demos[1607:20636] Dog 你是狗 - 0x7fdcf0427e60 被废弃
```

尝试解决方法四、使用`__block`来修饰对象指针

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    __block Dog *dog = [Dog new];
    dog.name = @"你是狗";
    
    dog.block = ^() {
        [dog log];
        
        // 对Block对象指向的对象释放一次
        dog = nil;
    };
    
    dog.block();
}
```

粗略的终于看完Blocks这一部分了...

***

###第三部分、Grand Central Dispatch（GCD）

####iOS平台提供的多线程操作方法

- (1) pthread
- (2) NSThread
- (3) 线程队列
	- GCD dispatch queue + Block
	- NSOperationQueue + NSOperation

对于后面的`线程队列`方式，屏蔽了直接使用线程Api，而是操作`队列`的形式。

####多线程中遇到的问题

- (1) 多个线程同时修改/读取一个数据，导致数据不一致
- (2) 多个线程之间的互相等待，造成线程死锁
- (3) 创建太多的线程，消耗大量的系统内存，以及频繁切换线程占用过多的cpu

![](http://i1.piimg.com/567571/f27d5b954f885435.png)

####dispatch queue 线程调度队列

- (1) 数据结构遵循 队列 >>> 先进 先出
- (2) 依次取出位于 队列头部 的Block任务
- (3) 从线程池获取一个线程，去执行	Block任务

```objc
dispatch_async(queue, ^{
    NSLog(@"任务1");
});

dispatch_async(queue, ^{
    NSLog(@"任务2");
});

dispatch_async(queue, ^{
    NSLog(@"任务3");
});

dispatch_async(queue, ^{
    NSLog(@"任务4");
});
```

如上的Block任务加入到线程队列的图示:

![](http://i2.piimg.com/567571/748e2694f381347d.png)

注意: 扔到 dispatch queue 的Block任务，是放到队列的`末尾`排队等待调度。那也就是说会比当前正在执行的代码`慢执行`（异步执行）。

####线程队列分为两种: 1)串行队列 2)并行队列

| Dispatch Queue种类 | 说明 | 
| :-------------: |:-------------:| 
| Serial Dispatch Queue | 等待当前正在调度执行的任务执行完毕，才会调度后面的任务 | 
| Concurrent Dispatch Queue | 不会等待当前调度任务执行完毕，`并发 没有顺序`的调度任务 |

![](http://i2.piimg.com/567571/55c35fa7a8698f52.png)

- (1) 串行队列、当前调度任务必须等待前一个调度任务执行完毕（一个SerialQueue只会创建一个线程）

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    dispatch_queue_t queue = dispatch_queue_create("serial", DISPATCH_QUEUE_SERIAL);
    
    dispatch_async(queue, ^{
        NSLog(@"任务1 - 所在线程:%@", [NSThread currentThread]);
    });

    dispatch_async(queue, ^{
        NSLog(@"任务2 - 所在线程:%@", [NSThread currentThread]);
    });

    dispatch_async(queue, ^{
        NSLog(@"任务3 - 所在线程:%@", [NSThread currentThread]);
    });

    dispatch_async(queue, ^{
        NSLog(@"任务4 - 所在线程:%@", [NSThread currentThread]);
    });
}
```

运行结果

```
2016-07-12 13:17:43.044 demo[17974:2725166] 任务1 - 所在线程:<NSThread: 0x7ff22a51c0c0>{number = 4, name = (null)}
2016-07-12 13:17:43.044 demo[17974:2725166] 任务2 - 所在线程:<NSThread: 0x7ff22a51c0c0>{number = 4, name = (null)}
2016-07-12 13:17:43.044 demo[17974:2725166] 任务3 - 所在线程:<NSThread: 0x7ff22a51c0c0>{number = 4, name = (null)}
2016-07-12 13:17:43.045 demo[17974:2725166] 任务4 - 所在线程:<NSThread: 0x7ff22a51c0c0>{number = 4, name = (null)}
```

从输出可以看到，任务是一个一个调度的，且让一个任务执行完毕之后才会调度下一个任务。而且所有的Block任务，只会在`一个新的子线程`上去执行。

- (2) 并行队列、不等待，同时执行多个，没有顺序（创建多个线程，会重复利用线程，并非无限制创建线程）

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    dispatch_queue_t queue = dispatch_queue_create("concurrent", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_async(queue, ^{
        NSLog(@"任务1 - 所在线程:%@", [NSThread currentThread]);
    });

    dispatch_async(queue, ^{
        NSLog(@"任务2 - 所在线程:%@", [NSThread currentThread]);
    });

    dispatch_async(queue, ^{
        NSLog(@"任务3 - 所在线程:%@", [NSThread currentThread]);
    });

    dispatch_async(queue, ^{
        NSLog(@"任务4 - 所在线程:%@", [NSThread currentThread]);
    });
}
```

运行结果

```
2016-07-12 13:20:00.659 demo[18046:2733995] 任务2 - 所在线程:<NSThread: 0x7f91cbfa1100>{number = 8, name = (null)}
2016-07-12 13:20:00.659 demo[18046:2733996] 任务1 - 所在线程:<NSThread: 0x7f91cbc1e620>{number = 7, name = (null)}
2016-07-12 13:20:00.659 demo[18046:2733999] 任务4 - 所在线程:<NSThread: 0x7f91cbc12b50>{number = 9, name = (null)}
2016-07-12 13:20:00.659 demo[18046:2733994] 任务3 - 所在线程:<NSThread: 0x7f91cbc104b0>{number = 6, name = (null)}
```

可以看到，Block任务是`无序`调度执行的。且会创建多个新的子线程去调度执行这些Block任务。


注意，决定创建多少个线程来处理这个并行队列中的所有Block任务，是由XNU内核决定的，他会重复利用以及完成任务的线程（线程池机制）。

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    dispatch_queue_t queue = dispatch_queue_create("concurrent", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_async(queue, ^{
        NSLog(@"任务1 - 所在线程:%@", [NSThread currentThread]);
    });

    dispatch_async(queue, ^{
        NSLog(@"任务2 - 所在线程:%@", [NSThread currentThread]);
    });

    dispatch_async(queue, ^{
        NSLog(@"任务3 - 所在线程:%@", [NSThread currentThread]);
    });

    dispatch_async(queue, ^{
        NSLog(@"任务4 - 所在线程:%@", [NSThread currentThread]);
    });
    
    dispatch_async(queue, ^{
        NSLog(@"任务5 - 所在线程:%@", [NSThread currentThread]);
    });
    
    dispatch_async(queue, ^{
        NSLog(@"任务6 - 所在线程:%@", [NSThread currentThread]);
    });
    
    dispatch_async(queue, ^{
        NSLog(@"任务7 - 所在线程:%@", [NSThread currentThread]);
    });
    
    dispatch_async(queue, ^{
        NSLog(@"任务8 - 所在线程:%@", [NSThread currentThread]);
    });
}
```

运行结果

```
2016-07-12 13:26:30.318 demo[18146:2760101] 任务1 - 所在线程:<NSThread: 0x7fbaf04a4c80>{number = 17, name = (null)}
2016-07-12 13:26:30.318 demo[18146:2760099] 任务2 - 所在线程:<NSThread: 0x7fbaf0735400>{number = 15, name = (null)}
2016-07-12 13:26:30.318 demo[18146:2760103] 任务3 - 所在线程:<NSThread: 0x7fbaf0512a10>{number = 16, name = (null)}
2016-07-12 13:26:30.318 demo[18146:2761069] 任务4 - 所在线程:<NSThread: 0x7fbaf04acc30>{number = 19, name = (null)}
2016-07-12 13:26:30.318 demo[18146:2760101] 任务5 - 所在线程:<NSThread: 0x7fbaf04a4c80>{number = 17, name = (null)}
2016-07-12 13:26:30.318 demo[18146:2760104] 任务6 - 所在线程:<NSThread: 0x7fbaf04a62c0>{number = 18, name = (null)}
2016-07-12 13:26:30.318 demo[18146:2760099] 任务7 - 所在线程:<NSThread: 0x7fbaf0735400>{number = 15, name = (null)}
2016-07-12 13:26:30.318 demo[18146:2760103] 任务8 - 所在线程:<NSThread: 0x7fbaf0512a10>{number = 16, name = (null)}
```

可以看到重复利用了17号、16号线程。

- 图示上面两种类型的队列调度Block任务的区别

![](http://i4.piimg.com/567571/afed448e7ab2b20a.png)

- SerialQueu 与 ConcurrentQueue 创建线程的区别

(1) 一个SerialQueu，只会创建一个线程

![](http://i4.piimg.com/567571/83cff0eaabba121c.png)

(2) 一个ConcurrentQueue，会创建多个线程，但是会重复利用，不会无限制的创建线程

![](http://i4.piimg.com/567571/29467040fdb72e71.png)

####使用Serial Queue 解决多线程并发无序执行读取/修改数据时，造成的数据不统一

![](http://i4.piimg.com/567571/538e3cb945361274.png)

比如，如下对一个对象的成员变量在多线程环境下进行修改/读取时进行多线程同步:

```objc
@interface ViewController ()

@property (nonatomic, strong) dispatch_queue_t serialQueue;

@end

@implementation ViewController

- (dispatch_queue_t)serialQueue {
    if (!_serialQueue) {
        _serialQueue = dispatch_queue_create("com.xzh.serail.queue", DISPATCH_QUEUE_SERIAL);
    }
    return _serialQueue;
}

//setter
- (void)modifyUserName:(NSString *)name {

  //异步派发到串行队列
    dispatch_async(self.serialQueue, ^{
        self.user.name = [name copy];
    });
}

//getter
- (NSString *)getUserName {

    //使用同步执行方法派发到串行队列
    __block returnName = nil;
    dispatch_sync(self.serialQueue, ^{
        returnName = self.user.name;
    });

    return returnName;
}

@end
```

在getter中，使用到了`dispatch_sync`告诉队列，`立马执行`放入队列的这个Block任务。

####书上说需要自己管理dispatch queue的内存释放，但是现在的编译器好像是不用自己管理的

```objc
dispatch_release(queue);
dispatch_retain(queue);
```

如上这两个与dispatch queue内存管理相关的函数，在现在的Xcode编译器下是不能够写的，直接报错。

那么在现在的ARC环境下，只需要在不使用的时候赋值nil即可。

####main dispatch queue

- (1) 在App程序进程创建之后，第一个创建的线程就是`主线程`，负责UI事件等等
- (2) 主线程对应的线程队列 >>> `dispatch_get_main_queue()`
- (3) 主线程必须保证串行的 >>> `dispatch_get_main_queue()`是 Serial Dispatch Queue


####global dispatch queue

- (1) 由系统自己创建的一个线程队列，任何App程序都能够去使用 >>> `dispatch_get_global_queue(<#long identifier#>, <#unsigned long flags#>)`

- (2) global dispatch queue 有四个调度任务等级:
	- 高优先级
	- 默认级
	- 低级
	- 后台级

```
#define DISPATCH_QUEUE_PRIORITY_HIGH 2
#define DISPATCH_QUEUE_PRIORITY_DEFAULT 0
#define DISPATCH_QUEUE_PRIORITY_LOW (-2)
#define DISPATCH_QUEUE_PRIORITY_BACKGROUND INT16_MIN
```

我们创建的SerailQueue与ConcurrentQueue以及GloabalQueue，在调度执行Block任务时，都是使用的`默认级`作为线程的优先级。这些优先级其实并不一定准确，只是起一个参考作用。

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
//    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);
//    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
//    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);
//    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);
//    dispatch_queue_t queue = dispatch_get_main_queue();
//    dispatch_queue_t queue = dispatch_queue_create("serail", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t queue = dispatch_queue_create("serail", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_async(queue, ^{
        NSLog(@"任务1 - 所在线程:%@ - 线程优先级:%lf", [NSThread currentThread], [NSThread threadPriority]);
    });

    dispatch_async(queue, ^{
        NSLog(@"任务2 - 所在线程:%@ - 线程优先级:%lf", [NSThread currentThread], [NSThread threadPriority]);
    });

    dispatch_async(queue, ^{
        NSLog(@"任务3 - 所在线程:%@ - 线程优先级:%lf", [NSThread currentThread], [NSThread threadPriority]);
    });
}

@end
```

如上所有类型的对象运行后的线程优先级都是`0.5`，不是说有线程优先级区分的吗？


####dispatch set target queue 组合多个 dispatch queue 按照顺序依次执行

如下三个串行队列，依次调度Block任务

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    dispatch_queue_t queue1 = dispatch_queue_create("serail", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t queue2 = dispatch_queue_create("serail", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t queue3 = dispatch_queue_create("serail", DISPATCH_QUEUE_SERIAL);
    
    dispatch_async(queue1, ^{
        NSLog(@"任务1 - 所在线程:%@ - 线程优先级:%lf", [NSThread currentThread], [NSThread threadPriority]);
    });

    dispatch_async(queue2, ^{
        NSLog(@"任务2 - 所在线程:%@ - 线程优先级:%lf", [NSThread currentThread], [NSThread threadPriority]);
    });

    dispatch_async(queue3, ^{
        NSLog(@"任务3 - 所在线程:%@ - 线程优先级:%lf", [NSThread currentThread], [NSThread threadPriority]);
    });
}
```

运行结果

```
2016-07-12 14:30:15.441 demo[19105:2978620] 任务2 - 所在线程:<NSThread: 0x7ff50bf16bc0>{number = 6, name = (null)} - 线程优先级:0.500000
2016-07-12 14:30:15.441 demo[19105:2978622] 任务3 - 所在线程:<NSThread: 0x7ff50be0e7b0>{number = 7, name = (null)} - 线程优先级:0.500000
2016-07-12 14:30:15.441 demo[19105:2978619] 任务1 - 所在线程:<NSThread: 0x7ff50be13110>{number = 5, name = (null)} - 线程优先级:0.500000
```

调度到三个Serial Queue中的任务，是并发无序执行的，处于不同的子线程中执行。

但是现在，如果需要这三个处于不同的Serial Queue中的Block任务，仍然需要依次按照顺序等待执行了？


```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    dispatch_queue_t queue1 = dispatch_queue_create("test.1", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t queue2 = dispatch_queue_create("test.2", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t targetQueue = dispatch_queue_create("test.3", DISPATCH_QUEUE_SERIAL);
    
    dispatch_set_target_queue(queue1, targetQueue);
    dispatch_set_target_queue(queue2, targetQueue);

    dispatch_async(queue1, ^{
        NSLog(@"1 in");
        NSLog(@"任务1 - 所在线程:%@ - 线程优先级:%lf", [NSThread currentThread], [NSThread threadPriority]);
        NSLog(@"1 out");
    });
    
    dispatch_async(queue1, ^{
        NSLog(@"2 in");
        NSLog(@"任务2 - 所在线程:%@ - 线程优先级:%lf", [NSThread currentThread], [NSThread threadPriority]);
        NSLog(@"2 out");
    });
    
    dispatch_async(queue2, ^{
        NSLog(@"3 in");
        NSLog(@"任务3 - 所在线程:%@ - 线程优先级:%lf", [NSThread currentThread], [NSThread threadPriority]);
        NSLog(@"3 out");
    });
    
    dispatch_async(queue2, ^{
        NSLog(@"4 in");
        NSLog(@"任务4 - 所在线程:%@ - 线程优先级:%lf", [NSThread currentThread], [NSThread threadPriority]);
        NSLog(@"4 out");
    });
}
```

运行结果

```
2016-07-12 16:26:58.284 demo[29903:3600206] 1 in
2016-07-12 16:26:58.284 demo[29903:3600206] 任务1 - 所在线程:<NSThread: 0x7fcffad11de0>{number = 4, name = (null)} - 线程优先级:0.500000
2016-07-12 16:26:58.284 demo[29903:3600206] 1 out
2016-07-12 16:26:58.284 demo[29903:3600206] 2 in
2016-07-12 16:26:58.285 demo[29903:3600206] 任务2 - 所在线程:<NSThread: 0x7fcffad11de0>{number = 4, name = (null)} - 线程优先级:0.500000
2016-07-12 16:26:58.285 demo[29903:3600206] 2 out
2016-07-12 16:26:58.285 demo[29903:3600206] 3 in
2016-07-12 16:26:58.285 demo[29903:3600206] 任务3 - 所在线程:<NSThread: 0x7fcffad11de0>{number = 4, name = (null)} - 线程优先级:0.500000
2016-07-12 16:26:58.285 demo[29903:3600206] 3 out
2016-07-12 16:26:58.285 demo[29903:3600206] 4 in
2016-07-12 16:26:58.285 demo[29903:3600206] 任务4 - 所在线程:<NSThread: 0x7fcffad11de0>{number = 4, name = (null)} - 线程优先级:0.500000
2016-07-12 16:26:58.286 demo[29903:3600206] 4 out
```

####dispatch after 

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    //1.
    dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3 * NSEC_PER_SEC));

    //2.
    dispatch_after(time, dispatch_get_main_queue(), ^{
        NSLog(@"延迟执行的代码");
    });
}
```

对上面代码最终效果就是延迟三秒执行代码，但是很多人的个误区:

> 并不是三秒后执行Block任务，而是三秒后将Block任务扔到dispatch queue队列中进行排队调度。

####dispatch group、dispatch barrier async、dispatch sync

在之前的GCD文章已经有记录.

####dispatch suspend/resume queue

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    // 自动拷贝堆区
    dispatch_queue_t queue = dispatch_queue_create("test", DISPATCH_QUEUE_CONCURRENT);

    dispatch_async(queue, ^{
        NSLog(@"1 in");
        NSLog(@"任务1 - 所在线程:%@ - 线程优先级:%lf", [NSThread currentThread], [NSThread threadPriority]);
        NSLog(@"1 out");
    });
    
    dispatch_async(queue, ^{
        NSLog(@"2 in");
        NSLog(@"任务2 - 所在线程:%@ - 线程优先级:%lf", [NSThread currentThread], [NSThread threadPriority]);
        NSLog(@"2 out");
    });
    
    // 暂停队列调度
    dispatch_suspend(queue);
    
    // 2庙后恢复队列调度
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        dispatch_resume(queue);//Block执行完毕后会释放，不会对queue形成不释放
    });
    
    dispatch_async(queue, ^{
        NSLog(@"3 in");
        NSLog(@"任务3 - 所在线程:%@ - 线程优先级:%lf", [NSThread currentThread], [NSThread threadPriority]);
        NSLog(@"3 out");
    });
    
    dispatch_async(queue, ^{
        NSLog(@"4 in");
        NSLog(@"任务4 - 所在线程:%@ - 线程优先级:%lf", [NSThread currentThread], [NSThread threadPriority]);
        NSLog(@"4 out");
    });
}
```

####dispatch semaphore 与 serial queue 同步多线程

- 信号量、可以保证代码的同步执行逻辑
- 队列、就可能出现代码的异步执行

Effective OC 2.0书上是推荐是队列来同步。

####最后dispatch source/io等等之前都记录过

终于看完了，做个笔记经常看。
