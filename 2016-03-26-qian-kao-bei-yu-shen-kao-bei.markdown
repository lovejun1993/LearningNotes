---
layout: post
title: "浅拷贝与深拷贝"
date: 2016-03-26 23:24:25 +0800
comments: true
categories: 
---

###同事遇到了一个bug，问我这个bug是不是由于这个自定义对象深拷贝引起。我第一反应是《Effective Objective-C 2.0》上说的Objtive-C没有直接提供深拷贝的方法，我当时真的想建议一下他去看下这本书。

当时我也不太特别肯定，也可能我自己也理解的不正确，回去我自己好好了解了下。

但是有一个概念是需要及早弄清楚的:

> 浅拷贝与深拷贝、可变与不可变，这是两回事。

如果不太明白，可以自己查看《Effective Objective-C 2.0》第22条，理解NSCopying协议。

****

###前面第一个问题，Objective-C中有提供直接深拷贝对象的方法吗？

首先从《Effective Objective-C 2.0》书上很清楚的说道:

![Snip20160520_2.png](http://imgchr.com/images/Snip20160520_2.png)

有时候面试的人，也总会问如何让一个类实现深拷贝，他其实就是想让你回答在NSCopying协议的copyWithZone:方法实现内，对内部子对象也执行copy。

我觉得这样是可以的，但是Foundation类基本上都是使用的浅拷贝方式来实现NSCopying协议与NSMutableCopy协议。所以我觉得就不要去打破苹果的这种约定，我们可以通过额外协议来规定使用`深拷贝`的方式来实现NSCopying协议与NSMutableCopy协议，来完成可变与不可变版本的深拷贝。

***

###不管是浅拷贝与深拷贝、可变与不可变，首先需要实现拷贝对象的功能 >>> 实体类必须实现NSCopying协议中对象方法，来完成该类的对象拷贝。

####iOS提供了两种拷贝对象的协议:

[对象 copy] >>> 得到该类的 `不可变`版本的对象（不关心浅拷贝or深拷贝）

```
@protocol NSCopying

- (id)copyWithZone:(nullable NSZone *)zone;

@end
```

[对象 mutableCopy] >>> 得到该类的 `可变`版本的对象（不关心浅拷贝or深拷贝）

```
@protocol NSMutableCopying

- (id)mutableCopyWithZone:(nullable NSZone *)zone;

@end
```

####我听很多人说:

- （1）第一个协议是浅拷贝实现，第二个协议是深拷贝实现。
- （2）或者有的说自己实现如上方法来分别做浅拷贝or深拷贝实现。

第一种说法，从《Effective Objective-C 2.0》所示错的很离谱。而第二种说法一定程度上是正确的，但是不建议这么做，后面有更好的做法。

***

####如何去决定实现哪一种协议完成拷贝了？首先可以先告诉答案，来自《Effective Objective-C 2.0》书上，并且觉得也确实很有道理。

- 如果当前类的对象分为: 可变版本 与 不可变版本
	- 分别实现NSCopying协议、NSMutableCopying协议
	- NSCopying协议: 提供一个`不可变`版本的拷贝(new)对象
	- NSMutableCopying协议: 提供一个`可变`版本的拷贝(new)对象
	
- 如果当前类的对象就只存在: 一种不可变版本 或 一种可变版本
	- 只实现NSCopying协议 或 NSMutableCopying协议 其中的一种

****

###那么就来看下对象经过`浅拷贝`和`深拷贝`之后得到的对象的区别:

####从书上截取的浅拷贝与深拷贝的区别图示:

![Snip20160520_1.png](http://imgchr.com/images/Snip20160520_1.png)

- 浅拷贝与深拷贝的相同点:

	- 都对`外部对象本身`进行拷贝

- 浅拷贝与深拷贝的不同点:

	- `浅拷贝`
		- 拷贝出来的容器对象直接`指向`原始容器对象的`内部其他子对象`
		- 也即是`不会`创建新的子对象

	- `深拷贝`
		-  会对原始对象内部的子对象继续执行`拷贝`操作
		-  也即是`会`创建新的子对象，并从原始对象中的子对象复制数据（就是执行[子对象 copy]）

####按照书上的话，小结下深拷贝:

> 在拷贝对象自身时，将该对象内部子数据也一并复制。Foundation框架中的 collection类在默认情况下都执行`浅拷贝`，也就是说，`只拷贝对象本身`，而不对其内部子对象进行拷贝。

####说这么多，只想说明前面说的关于 `浅拷贝与深拷贝、可变与不可变，这是两个不同的话题`这句话的含义。

- copy与mutableCopy并不是就是指，浅拷贝 or 深拷贝
- copy与mutableCopy只是用于一个类分为`可变`与`不可变`两个版本时的`拷贝对象`之间的切换
- 而深拷贝 or 浅拷贝，意味着`对原始对象内部子对象`是否`继续执行拷贝`还是`直接指向`


####但是，`copy、mutableCopy` 与 `浅拷贝、深拷贝`之间又是有关系的。

***

###先从copyWithZone:传入的zone对象干什么的开始

- Zone: 一个单独的区

- 以前的iOS系统会将一个App程序内存，分为很多个单独的Zone

- 而现在的统一使用一个公用的Zone，所以不必关系Zone这个参数值

****

###Effective Objective-C 2.0 上指明的NSCopying协议与NSMutableCopying协议的区别:

> mutableCopy这个 “辅助方法”（helper）与 copy相似，也是用默认的 zone 参数来调用 “mutableCopyWithZone:”。如果你的类分为 可变版本（mutable variant） 与 不可变版本（immutable variant），那么就应该实现NSMutableCopying协议。若采用此模式，则在 可变类 中覆写 “copyWithZone:” 时，不要返回 可变的拷贝，而应该返回一份不可变的版本。无论当前实例是否可变，若需要获取可变版本的拷贝，均应调用 mutableCopy 方法。同理若需要不可变对象，则总应该调用 copy 方法来获取。

所以很多人，将copy作为浅拷贝方法，而将mutableCopy作为深拷贝，这种说法是`错误`的。

正确说法应该是copy返回`不可变`版本的对象，而mutableCopy返回的是`可变`版本的对象。

> 而关于 浅拷贝还是深拷贝， 却是另外一个问题。但是可变与不可变 和 深拷贝与浅拷贝 之前有是有联系的。

####个人总结的关于: 可不可变 与 对象浅拷贝or深拷贝 这两个话题之间的关系

![](http://i12.tietuku.cn/da787550ed3d3889.png)

那么说一个对象拷贝的工作，那么有四种情况:

```
1. 不可变 + 浅拷贝
2. 不可变 + 深拷贝
3. 可变 + 浅拷贝
4. 可变 + 深拷贝
```

而对于Foundation类，默认只是使用 1与3 两种对象拷贝的方式.

####所以我觉得在完成一个对象拷贝的操作时，你需要考虑两个问题

- 问题一、将要获取 原始对象的 `可变版本` 还是 `不可变版本` ？
	- 可变 >>> 实现`NSCoyping`协议方法，最终返回一个`不可变`对象
	- 不可变 >>> 实现`NSMutableCopying`协议方法，最终返回一个`可变`对象	

- 问题二、使用 `浅拷贝` 还是 `深拷贝` 哪一种`方式` 来完成对象拷贝的工作
	- 浅拷贝 >>> 只对容器对象本身进行拷贝，不对内部子对象拷贝
	- 深拷贝 >>> 除了对容器对象本身进行拷贝，仍然还要对内部子对象拷贝

****

###一个类的对象分为 可变 or 不可变 两个版本

就是能不能改变一个对象的各种属性值，例如: NSArray与NSMutableArray、NSURLRequest与NSNSMutableURLRequest.

因为分为可变版本与不可变版本，所以同时实现NSCopying, NSMutableCopying协议，用于copy与mutableCopy在可变对象与不可变对象之间的切换。

- 不可变版本 NSURLRequest

```objc
//因为分为可变版本与不可变版本，所以同时实现NSCopying, NSMutableCopying协议
@interface NSURLRequest : NSObject <NSSecureCoding, NSCopying, NSMutableCopying>
{
    @private
    NSURLRequestInternal *_internal;
}

+ (instancetype)requestWithURL:(NSURL *)URL;
+ (BOOL)supportsSecureCoding;
+ (instancetype)requestWithURL:(NSURL *)URL cachePolicy:(NSURLRequestCachePolicy)cachePolicy timeoutInterval:(NSTimeInterval)timeoutInterval;

- (instancetype)initWithURL:(NSURL *)URL;
- (instancetype)initWithURL:(NSURL *)URL cachePolicy:(NSURLRequestCachePolicy)cachePolicy timeoutInterval:(NSTimeInterval)timeoutInterval NS_DESIGNATED_INITIALIZER;

//属性全部修饰为只读
@property (nullable, readonly, copy) NSURL *URL;
@property (readonly) NSURLRequestCachePolicy cachePolicy;
@property (readonly) NSTimeInterval timeoutInterval;
@property (nullable, readonly, copy) NSURL *mainDocumentURL;
@property (readonly) NSURLRequestNetworkServiceType networkServiceType NS_AVAILABLE(10_7, 4_0);
@property (readonly) BOOL allowsCellularAccess  NS_AVAILABLE(10_8, 6_0);

@end
```

- 可变版本 NSNSMutableURLRequest（继承自不可变版本）
	- 继承自不可变版本的基础功能方法
	- 将`父类只读`属性修改为`读写`属性
	- 并添加一些操作`只读属性`的一些api方法

```objc
@interface NSMutableURLRequest : NSURLRequest

//将父类修饰为只读属性，全部改为readwrite（默认就是）
@property (nullable, copy) NSURL *URL;
@property NSURLRequestCachePolicy cachePolicy;
@property NSTimeInterval timeoutInterval;
@property (nullable, copy) NSURL *mainDocumentURL;
@property NSURLRequestNetworkServiceType networkServiceType NS_AVAILABLE(10_7, 4_0);
@property BOOL allowsCellularAccess NS_AVAILABLE(10_8, 6_0);

@end
```

****

###看下NSArray与NSMutableArray的copy与mutableCopy的作用

> 针对容器对象（Array、Set、Dictionary）.

####NSArray的copy与mutableCopy

```objc
- (void)testFoundation1 {
    
    User *user1 = [User new];
    User *user2 = [User new];
    User *user3 = [User new];
    User *user4 = [User new];
    
    //1. 原始数组
    NSArray *array = @[user1, user2, user3];
    
    //2. copy出来的数组
    id copy1 = [array copy];
    //    [copy1 addObject:user4];//会崩溃
    
    //3. mutableCopy出来的数组
    id copy2 = [array mutableCopy];
    //    [copy2 addObject:user4];//不会崩溃，说明已经是一个可变数组对象了    
}
```

- 控制台输出如上三个数组对象:

	- [NSArray对象 `copy`]出来的是同一个类型`__NSArrayI`，并且就是原来的__NSArrayI类型的对象.（同一个数组对象）
	- [NSArray对象 `mutableCopy`]出来的真实类型是 `__NSArrayM`，那么显然是两个不同的对象.（不同的数组对象）
	- **所以对于-[NSArray copy]和原来的NSArray对象一样的，没什么区别**

```
(__NSArrayI *) array = 0x00007fbdf058b440 @"4 objects"
(__NSArrayI *) copy1 = 0x00007fbdf058b440 @"4 objects"
(__NSArrayI *) copy2 = 0x00007fbdf058b440 @"4 objects"
(__NSArrayI *) copy3 = 0x00007fbdf058b440 @"4 objects"
(__NSArrayM *) copy4 = 0x00007fbdf058b470 @"4 objects"
(__NSArrayM *) copy5 = 0x00007fbdf058b4a0 @"4 objects"
(__NSArrayM *) copy6 = 0x00007fbdf0508a80 @"4 objects"
```

- 再看看数组`内部对象`:

```
(lldb) po copy1[0]
0x00007fcdf9496790

(lldb) po copy2[0]
0x00007fcdf9496790

(lldb) po copy3[0]
0x00007fcdf9496790

(lldb) po copy4[0]
0x00007fcdf9496790

(lldb) po copy5[0]
0x00007fcdf9496790

(lldb) po copy6[0]
0x00007fcdf9496790
```

这里只对数组[0]进行输出，还可以对数组[1]，数组[2]进行输出，但最终的效果都是一样的子对象。

所以说对于`NSArray对象`不管是`copy`还是`mutableCopy`出来的数组，内部包含的子元素对象地址仍然是一样的。

所以也就是说NSArray内部的数组类对于NSCoying协议与NSMutableCopying协议都是使用的`浅拷贝`方式来完成拷贝的。

###NSMutableArray的copy与mutableCopy

```objc
- (void)testFoundation1 {
    
    User *user1 = [User new];
    User *user2 = [User new];
    User *user3 = [User new];
    User *user4 = [User new];
    
    //1. 原始数组
    NSArray *array = @[user1, user2, user3];
    
    //2. copy出来的数组
    id copy1 = [array copy];
    id copy2 = [array copy];
    id copy3 = [array copy];
    //    [copy1 addObject:user4];//会崩溃
    
    //3. mutableCopy出来的数组
    id copy4 = [array mutableCopy];
    id copy5 = [array mutableCopy];
    id copy6 = [array mutableCopy];
    //    [copy4 addObject:user4];//不会崩溃，说明已经是一个可变数组对象了
}
```

- 控制台输出如上数组对象:
	- [NSMutableArray对象 `copy`]出来的是`__NSArrayI`类型的对象，都是`不同`的对象.（不同的数组对象）
	- [NSMutableArray对象 `mutableCopy`]出来的是 `__NSArrayM`类型对象，但是确实`不同`的对象.（不同的数组对象）

```
(__NSArrayM *) mutaArray = 0x00007ffb335a81e0 @"4 objects"
(__NSArrayI *) copy1 = 0x00007ffb335a8210 @"4 objects"
(__NSArrayI *) copy2 = 0x00007ffb335a8240 @"4 objects"
(__NSArrayI *) copy3 = 0x00007ffb335a8270 @"4 objects"
(__NSArrayM *) copy4 = 0x00007ffb335a82a0 @"4 objects"
(__NSArrayM *) copy5 = 0x00007ffb335a82f0 @"4 objects"
(__NSArrayM *) copy6 = 0x00007ffb335a8340 @"4 objects"
```

三个数组对象本身，可以看到是`不同`的对象`类型与地址`

```
(__NSArrayM *) mutaArray = 0x00007f94b0706b90 @"3 objects"
(__NSArrayI *) copy1 = 0x00007f94b0707e70 @"3 objects"
(__NSArrayM *) copy2 = 0x00007f94b0426790 @"3 objects"
```

再看看三个数组内部对象，内部对象仍然是一模一样的


```
(lldb) po copy1[0]
0x00007fbdf0586df0

(lldb) po copy2[0]
0x00007fbdf0586df0

(lldb) po copy3[0]
0x00007fbdf0586df0

(lldb) po copy4[0]
0x00007fbdf0586df0

(lldb) po copy5[0]
0x00007fbdf0586df0

(lldb) po copy6[0]
0x00007fbdf0586df0
```

所以说对于`NSMutableArray对象`不管是`copy`还是`mutableCopy`出来的数组，内部包含的子元素对象地址仍然是一样的，但是拷贝出来的数组对象都是不相同的。

所以也就是说NSMutableArray内部的数组类对于NSCoying协议与NSMutableCopying协议都是使用的`浅拷贝`方式来完成拷贝的。

####NSArray、NSMutableArray，不管是copy还是mutableCopy拷贝出来的数组对象`内部的子元素对象`，都是直接指向`原始数组内部所有的对象`.

> 所以Foundation类在实现NSCopying协议和NSMutableCopying协议时，都是采用的`浅拷贝`对象的方式来实现。

Foundation 框架中的所有 集合类（Set、Array、Dic）在默认情况下都执行的是`浅拷贝`。也就是说只拷贝容器对象本身，而不继续对容器内部子元素对象进行拷贝，而是直接指向。

####这样做的主要原因:

- 因为并不知道容器内`子元素对象`是否能够继续执行copy拷贝（是否实现了NSCopying协议、NSMutableCopying协议）.
- 外部容器对象，也不一定需要拷贝得到所有`新的`的子元素对象.

基于这些原因，Foundation类中基本上都是采用`浅拷贝`来实现NSCopying协议和NSMutableCopying协议完成对象的拷贝。

但是在有些情况下，的确需要对 `Foundation容器对象` 或 `我们自己的某个实体类对象`使用`深拷贝`的形式来完成拷贝的工作 >>> 不想对拷贝对象内部对象的修改去影响到原来对象内部的子对象。

****

###NSArray、NSDictionary、NSSet提供了`深拷贝`对象的方法api >>> initWithXxx:copyItems:，第二个copyItem:这个参数传一个BOOL值.

如果YES，表示继续对容器内子对象进行copy。
如果NO，表示直接指向容器内子对象，不进行copy。

> 注意、只是对象的深拷贝，并没有涉及可变与不可变。深拷贝得到的类型，与当前调用拷贝方法的类型一致。

####通过 NSArray 提供的 深拷贝/浅拷贝 方法

```objc
User *user1 = [User new];
User *user2 = [User new];
User *user3 = [User new];
User *user4 = [User new];
    
//1. 原始数组
NSArray *array = [[NSArray alloc] initWithObjects:user1, user2, user3, user4, nil];

//2. 深拷贝
id copy7 = [[NSArray alloc] initWithArray:array copyItems:YES];

//3. 浅拷贝
id copy8 = [[NSArray alloc] initWithArray:array copyItems:NO];
```

输出如下，可以看到数组类型都是原来的`__NSArrayI *`，但是NSArray对象都是`不同`的

```objc
(User *) user1 = 0x00007ffa8271a190
(User *) user2 = 0x00007ffa82722240
(User *) user3 = 0x00007ffa82722170
(User *) user4 = 0x00007ffa827221a0

(__NSArrayI *) array = 0x00007ffa8271c4b0 @"4 objects"

(__NSArrayI *) copy7 = 0x00007ffa8271f480 @"4 objects"

(__NSArrayI *) copy8 = 0x00007ffa8271f4b0 @"4 objects"
```

再看copy7（深拷贝）出来的数组内部的子对象，全部都是`新`的子对象.（创建新的子对象）

```objc
(lldb) po copy7[0]
0x00007ffa8270da90

(lldb) po copy7[1]
0x00007ffa8270dac0

(lldb) po copy7[2]
0x00007ffa82721360

(lldb) po copy7[3]
0x00007ffa82721390
```

再看copy8（浅拷贝）出来的数组内部的子对象，全部都是原来数组array内部的子对象.（指向原来数组内部子对象）

```objc
(lldb) po copy8[0]
0x00007ffa8271a190

(lldb) po copy8[1]
0x00007ffa82722240

(lldb) po copy8[2]
0x00007ffa82722170

(lldb) po copy8[3]
0x00007ffa827221a0
```

####通过 NSMutableArray 提供的 深拷贝/浅拷贝 方法

```objc    
User *user1 = [User new];
User *user2 = [User new];
User *user3 = [User new];
User *user4 = [User new];
    
//1. 原始数组
NSMutableArray *mutaArray = [[NSMutableArray alloc] initWithObjects:user1, user2, user3, user4, nil];
    
//2. 深拷贝
id copy7 = [[NSMutableArray alloc] initWithArray:mutaArray copyItems:YES];

//3. 浅拷贝
id copy8 = [[NSMutableArray alloc] initWithArray:mutaArray copyItems:NO];
```

输出如下，可以看到数组类型都是原来的`__NSArrayM *`，同样对象都是`不同`的

```
(User *) user1 = 0x00007f9122f0db70
(User *) user2 = 0x00007f9122f0e7e0
(User *) user3 = 0x00007f9122f0e810
(User *) user4 = 0x00007f9122f0e5f0

(__NSArrayM *) mutaArray = 0x00007f9122f0e620 @"4 objects"

(__NSArrayM *) copy7 = 0x00007f9122f0ea30 @"4 objects"

(__NSArrayM *) copy8 = 0x00007f9122f0e260 @"4 objects"
```

再看copy7（深拷贝）出来的数组内部的子对象，全部都是`新`的子对象.（创建新的子对象）

```
(lldb) po copy7[0]
0x00007f9122f0e970

(lldb) po copy7[1]
0x00007f9122f0e9a0

(lldb) po copy7[2]
0x00007f9122f0e9d0

(lldb) po copy7[3]
0x00007f9122f0ea00
```

再看copy8（浅拷贝）出来的数组内部的子对象，全部都是原来数组array内部的子对象.（指向原来数组内部子对象）

```
(lldb) po copy8[0]
0x00007f9122f0db70

(lldb) po copy8[1]
0x00007f9122f0e7e0

(lldb) po copy8[2]
0x00007f9122f0e810

(lldb) po copy8[3]
0x00007f9122f0e5f0
```

所以对于NSArray/NSMutableArray提供了直接进行`浅拷贝 or 深拷贝`的init函数:

- 创建出来的数组对象类型与调用init方法的类型保持一致
- 创建出来的数组对象是绝对不相同的
- copyItems:YES，表示新数组对象，对原来容器对象内部的子对象，继续执行copy拷贝得到新的子对象
- copyItems:NO，表示新数组对象，直接指向原来数组容器内部的子对象


那么对于NSDictionary/NSMutableDictionary，NSSet/NSMutableSet，同样有如上的方法来完成浅拷贝or深拷贝的操作:

```objc
initWithDictionary:copyItems:
initWithSet:copyItems:
```

注意仅仅只是选择`浅拷贝 or 深拷贝`，并没有选择可变不可变，对象的类型与之前类型保持一致。

****

###前面是NSArray/NSDictionry/NSSet容器类已经提供了`深拷贝`的api，但是给一个普通的NSObject类的对象提供`深拷贝`的功能了？

####有如下两种方法

- 第一种、对一个类按照 `深拷贝` 的方式去实现`NSCopying协议` 或 `NSMutableCopying协议`

- 第二种、自定义一个用于`深拷贝的协议`，去让那些需要提供深拷贝对象的类自己去实现这个协议，来具备深拷贝的功能

- 对于第一种方法 >>> 不推荐
	- 大多数Foundation类实现NSCopying协议/NSMutableCopy协议，都是按照`浅拷贝`的方式来完成对象拷贝
	- 所以如果我们对某个对象直接在实现NSCopying协议方法NSMutableCopy协议中，去使用`深拷贝`的方法去完成对象拷贝，那么就与iOS系统规范不太一样了
	
- 对于第二种方法 >>> 推荐
	- 首先自定义一个具备深拷贝功能的抽象协议
	- 然后需要实现深拷贝的类中实现这个协议
	- 并且在协议方法中，使用`深拷贝`的方式来完成对象拷贝

- 对于我们自定义的对象深拷贝协议同样也需要分为两个版本:
	- 深拷贝下的 不可变对象
	- 深拷贝下的 可变对象

- 那么可以写成如下，个人觉得....

```objc
#import <Foundation/Foundation.h>

/**
 *  深拷贝 + 不可变
 */
@protocol XZHDeepCopying <NSObject>

- (instancetype)xzh_copyWithDeep;

@end

/**
 *  深拷贝 + 可变
 */
@protocol XZHDeepMutableCopying <NSObject>

- (instancetype)xzh_mutableCopyWithDeep;

@end
```

具体类自己根据情况是否可变，来觉得去实现哪一个协议，并使用深拷贝的方式来完成最终的对象拷贝.

****

###看一个摘录自书上使用 `浅拷贝` + `不可变` 来实现对象拷贝

```objc
#import <Foundation/Foundation.h>
#import "Account.h"

@interface User : NSObject <NSCopying>//不可变对象拷贝

@property (nonatomic, assign) NSInteger uid;
@property (nonatomic, copy) NSString *uname;
@property (nonatomic, copy) NSString *upassword;

- (instancetype)initWithUid:(NSInteger)uid
                      uName:(NSString *)name
                  uPassword:(NSString *)password;

- (void)addAccount:(Account *)account;
- (void)removeAccount:(Account *)account;
- (NSArray *)accounts;

@end
```

```objc
#import "User.h"

@implementation User {
    NSMutableArray *_mutaAccounts;
}

- (instancetype)initWithUid:(NSInteger)uid
                      uName:(NSString *)name
                  uPassword:(NSString *)password
{
    self = [super init];
    if (self) {
        
        //基本数据类型
        _uid = uid;
        
        //copy
        _uname =
        _upassword = [password copy];
        
        //内部Account数组初始化
        _mutaAccounts = [NSMutableArray new];
    }
    return self;
}

- (void)addAccount:(Account *)account {
    [_mutaAccounts addObject:account];
}

- (void)removeAccount:(Account *)account {
    [_mutaAccounts removeObject:account];
}

- (NSArray *)accounts {
    return [_mutaAccounts copy];
}

#pragma mark - 浅拷贝 + 不可变 进行对象拷贝

- (id)copyWithZone:(NSZone *)zone {
    
    //1. 创建一个新的User，并直接传入当前对象的属性值
    User *newUser = [[User allocWithZone:zone] initWithUid:_uid
                                                     uName:_uname
                                                 uPassword:_upassword];
    
    //2. 内部数组子对象拷贝
    
    // 方式一、这样会导致修改内部子对象，会影响原来的数组内部子对象
	//newUser->_mutaAccounts = self->_mutaAccounts;

	// 方式二、建议对数组浅拷贝一下，这样就互相不会影响
    newUser->_mutaAccounts = [self->_mutaAccounts mutableCopy];
    
    // 方式三、与方式二等同
    newUser->_mutaAccounts = [[NSMutableArray alloc] initWithArray:newUser->_mutaAccounts copyItems:NO];
    
    //3. 返回新User容器对象
    return newUser;
}

@end
```

****

###对上面例子改用使用 `深拷贝` + `不可变` 来实现对象拷贝

####首先定义一个提供`深拷贝`方式的协议

> 不要在原来的NSCopying协议方法实现中修改成深拷贝的方式，而是在新协议的方法中实现深拷贝方式。注意，与之前一样，在深拷贝方式下，也分为可变版本与不可变版本的拷贝对象。

####EOCPerson类实现深拷贝协议方法

```
#import <Foundation/Foundation.h>
#import "Account.h"

//深拷贝协议
#import "XZHDeepCopying.h"

/**
 *  添加对 深拷贝协议的实现，由于当前User类就只有一个不可变版本，所以实现XZHDeepCopying协议
 */
@interface User : NSObject <NSCopying, XZHDeepCopying>

@property (nonatomic, assign) NSInteger uid;
@property (nonatomic, copy) NSString *uname;
@property (nonatomic, copy) NSString *upassword;

- (instancetype)initWithUid:(NSInteger)uid
                      uName:(NSString *)name
                  uPassword:(NSString *)password;

- (void)addAccount:(Account *)account;
- (void)removeAccount:(Account *)account;
- (NSArray *)accounts;

@end
```

```objc
#import "User.h"

@implementation User {
    NSMutableArray *_mutaAccounts;
}

- (instancetype)initWithUid:(NSInteger)uid
                      uName:(NSString *)name
                  uPassword:(NSString *)password
{
    self = [super init];
    if (self) {
        
        //基本数据类型
        _uid = uid;
        
        //copy
        _uname =
        _upassword = [password copy];
        
        //内部Account数组初始化
        _mutaAccounts = [NSMutableArray new];
    }
    return self;
}

- (void)addAccount:(Account *)account {
    [_mutaAccounts addObject:account];
}

- (void)removeAccount:(Account *)account {
    [_mutaAccounts removeObject:account];
}

- (NSArray *)accounts {
    return [_mutaAccounts copy];
}

#pragma mark - 浅拷贝 + 不可变

- (id)copyWithZone:(NSZone *)zone {
    
    //1. 创建一个新的User，并直接传入当前对象的属性值
    User *newUser = [[User allocWithZone:zone] initWithUid:_uid
                                                     uName:_uname
                                                 uPassword:_upassword];
    
    //2. 内部数组子对象同样直接指向
//    newUser->_mutaAccounts = self->_mutaAccounts;//这样会导致修改内部子对象，会影响原来的数组内部子对象
    newUser->_mutaAccounts = [self->_mutaAccounts mutableCopy];//建议对数组浅拷贝一下，这样就互相不会影响
    newUser->_mutaAccounts = [[NSMutableArray alloc] initWithArray:newUser->_mutaAccounts copyItems:NO];//与上一句效果一致，都是浅拷贝一下容器对象本身
    
    //3. 如果是还有单个1:1的对象
    //newUser.car = _car;
    
    //4. 返回新User容器对象
    return newUser;
}

#pragma mark - 深拷贝 + 不可变

- (instancetype)xzh_copyWithDeep {
    
    //1. 创建一个新的User，并传入copy后的属性值（Foundation类直接copy即可）
    User *newUser = [[User alloc] initWithUid:_uid
                                        uName:[_uname copy]
                                    uPassword:[_upassword copy]];
    
    //4. 对内部数组对象分别进行copy操作，得到新的子对象
	// 对数组执行深拷贝，得到一个新的数组对象，并且新数组内部子对象也同样是新的
    newUser->_mutaAccounts = [[NSMutableArray alloc] initWithArray:self->_mutaAccounts copyItems:YES];
    
    //5. 如果是还有单个1:1的对象，继续子对象的深拷贝
    //newUser.car = [_car xzh_copyWithDeep];
    
    //6.
    return newUser;
}

@end
```

严格来说对于此类User，NSCopying协议并不是来规定一个不可变版本的对象，因为对于User类本来就没有可变与不可变之分，此处的NSCopying协议就是对象的拷贝，但是按照`浅拷贝`方式完成对象拷贝。

> 而对于`深拷贝`方式完成拷贝，则是通过`实现另一个自定义协议`来完成，不要让`浅拷贝`方法 与 `深拷贝`方式 混在一起。

如果自定义类分为`可变 与 不可变`，又要使用`深拷贝`的方式去完成对象拷贝，那么此时就分情况选择两种深拷贝协议.

- 不可变对象的 深拷贝，实现

```
/**
 *  深拷贝 + 不可变
 */
@protocol XZHDeepCopying <NSObject>

- (instancetype)xzh_copyWithDeep;

@end
```

- 可变对象的 深拷贝，实现

```objc
/**
 *  深拷贝 + 可变
 */
@protocol XZHDeepMutableCopying <NSObject>

- (instancetype)xzh_mutableCopyWithDeep;

@end
```

在如上对应的协议方法中，使用深拷贝的方式进行对象拷贝即可.

****

###什么时候实现NSCopying协议？什么时候实现NSMutableCopying协议？什么时候实现自定义的深拷贝协议？

####一、不强制要求使用`深拷贝`的方式进行对象拷贝

- 当一个类不区分 可变版本与不可变版本，就是一种不可变情况

	- 实现NSCopying协议
	- 使用`浅拷贝`方式完成拷贝
	- 如果一定要求使用`深拷贝`，那么实现自定义深拷贝协议

- 当一个类分为 可变版本与不可变版本 
	- 同时实现 NSCopying协议 与 NSMutableCopying协议
	- 实现NSCopying协议:
		- 返回一个`不可变`版本的对象（外部容器对象）
		- 使用`浅拷贝`方式完成拷贝
	- 实现NSMutableCopying协议:
		- 返回一个`可变`版本的对象（外部容器对象）
		- 使用`浅拷贝`方式完成拷贝
	- 如果一定要求使用`深拷贝`，那么实现自定义深拷贝协议

####二、强制要求使用`深拷贝`的方式进行对象拷贝

- 实现自定义深拷贝协议方法，使用使用`深拷贝`方式完成对象拷贝

****

###书上对拷贝的总结

- 若想让自己编写的类的对象具备拷贝功能，则需要实现NSCopying协议

- 如果自定义的类对象分为`可变版本与不可变版本`，那么就需要同时实现NSCopying协议与NSMutableCopying协议

- 复制对象时需决定使用浅拷贝还是深拷贝，一般情况下应该尽量执行`浅拷贝`

- 如果你写的对象需要执行`深拷贝`，那么可以考虑新增一个专门描述深拷贝的协议方法


****

###第二个问题，改变容器内子对象属性值好吗？

####首先明白`对象等同性`的概念 >>> 属于同一个类的对象，而且所有的属性值全部相对.

看看NSString给出的对象等同性的方法，注意有两种方法

```objc
- (void)equal {
    
    NSString *str1 = @"Hello World!";
    NSString *str2 = @"Hello World!";
    
    //一、使用NSObject实现的 isEqual:
    NSLog(@"isEqual: %d", [str1 isEqual:str2]);
    
    //二、使用NSString自己实现的 isEqualToString:
    NSLog(@"isEqualToString: %d", [str1 isEqualToString:str2]);
}
```

输出结果是1，说明两个对象是等同的

```
2016-03-27 22:45:12.633 Copy[16032:223480] isEqual: 1
2016-03-27 22:45:12.633 Copy[16032:223480] isEqualToString: 1

```

如上有两种方法，一个是“isEqual:” , 一个是“isEqualToString:”，有什么区别了？

####NSObject协议声明的的 “isEqual:”

```objc
@protocol NSObject

- (BOOL)isEqual:(id)object;

@property (readonly) NSUInteger hash;

....其他省略
@end
```

可以看到isEqual: 是不区分传入的对象的类型.

####NSString自己实现的 “isEqualToString:”

```objc
//分类提供的
@interface NSString (NSStringExtensionMethods)

- (BOOL)isEqualToString:(NSString *)aString;

...其他省略
@end
```

限制了传入的对象类型，必须是NSString.

####那么应该优先调用单独提供的“isEqualToString:”来判断对象等同性，就已经去掉了对象所属类型判断。

- NSObject协议实现类NSObject类，实现了“isEqual:”对象等同判断.
- 但是对于我们自定义的NSObject子类，需要单独提供类似NSString提供的“isEqualToString:”限制传入的对象类型
	- 如果类型不一致，编译器会自动显示黄色警告

####从上面的isEqual:声明出，看到NSObject协议中还声明了一个只读的hash属性

- 如果两个NSObject对象，使用 “isEqul:” 比较返回YES，那么两个对象的hash属性值就`一定相等`.

- 但是反过来，两个对象的hash属性值相等时，`不一定`使用 “isEqul:” 比较返回YES.

####实现一个类的对象等同性判断的二个步骤

- 假定类属性声明这样的

```objc
@interface Person : NSObject

@property (nonatomic, copy) NSString *name;

@property (nonatomic, assign) NSInteger age;

@end
```

- 实现部分一、单独提供进行Person这一类的对象的等同性判断的方法入口

```objc
- (BOOL)isEqualToPerson:(Person *)person {
    
    //1. 先比较对象指针
    if (self == person) return YES;
    
    //2. 再比较两个对象的所有属性值
    if (_age != person.age) return NO;
    if (![_name isEqualToString:person.name]) return NO;

    //3.
    return YES;
}
```

- 实现部分二、重写继承自NSObject的 isEqual:

```objc
- (BOOL)isEqual:(id)object {
    
    // 只做类型判断，然后交给 isEqualToPerson: 处理
    if ([self class] == [object class]) {
        return [self isEqualToPerson:object];
    } else {
        return [super isEqual:object];
    }
}
```


####对于重写一个类的hash属性方法的算法

```objc
@interface Person : NSObject

@property (nonatomic, copy) NSString *name;

@property (nonatomic, assign) NSInteger age;

@end
```

```
@implementation Person

- (NSUInteger)hash {
    
    //1.
    NSUInteger ageHash = _age;
    
    //2.
    NSUInteger nameHash = [_name hash];
    
    //3. 自定义类的hash（需要单独实现）
    NSUInteger userHash = [_user hash];
    
    //4. 计算得到当前对象的hash
    return ageHash ^ nameHash ^ userHash;
}

@end
```

可以看到，对象的hash属性值，是通过该对象的所有其他属性值的hash值通过异或运算计算得到。那么也就是说，`修改了对象的其他数据属性值`，那么就意味着`该对象的hash属性值也跟着改变了`。

####为什么要说对象的hash属性值了？是有原因的，当一个Person对象加入到NSArray、NSSet、NSDictionary等容器对象中时，对象的hash属性值很重要的.

分别针对Array/Set/Dictionary看看各有什么区别.

###Array

- 对象加入是有顺序的，挨个往后面空位置放入新对象
- 如果在修改内部某个位置的对象属性值，造成该位置上对象的hash值改变
- 仍然是通过固定的位置来读取这个对象，好像是没有什么问题

```objc
- (void)testHash1 {
    
    User *user1 = [User new];
    user1.uid = 1;
    user1.uname = @"hahaha";
    user1.upassword = @"jjjjjjj";
    
    User *user2 = [User new];
    user2.uid = 1;
    user2.uname = @"hahaha";
    user2.upassword = @"jjjjjjj";
    
    User *user3 = [User new];
    user3.uid = 1;
    user3.uname = @"hahaha";
    user3.upassword = @"jjjjjjj";
    
    User *user4 = [User new];
    user4.uid = 1;
    user4.uname = @"hahaha";
    user4.upassword = @"jjjjjjj";
    
    NSUInteger hash1 = [user1 hash];
    NSUInteger hash2 = [user1 hash];
    NSUInteger hash3 = [user1 hash];
    NSUInteger hash4 = [user1 hash];
    
    //1. NSArray
    NSMutableArray *array = [@[user1, user2, user3, user4] mutableCopy];
    
    //2. 修改数组内部子对象的值
    user3.uid = 2;
    user3.uname = @"uuuuuu";
    user3.upassword = @"uuuuuu";
    
    //3. 在获取第三个位置的子对象
    User *temp = array[2];
    
    //4. 打印第三个位置上的user对象，是否正确的找到
    
}
```

输出修改之后位置的temp对象

```
<User: 0x7f9bbad0aad0, {
    uid = 2;
    uname = uuuuuu;
    upassword = uuuuuu;
}>
```

看出来对于Array因为是有固定下标指定一个对象，所以修改内部对象属性值是没有关系的.

****

###Set

- set是`无顺序`存放对象

- set中不能够存储重复的数据
	- 确切的说是两个`等同`的对象
	- [obj1 isEqual:obj2] == YES时，就只会存储obj1与obj2其中的一个

####下面看个例子一、没有重写User类的`isEqual:`判断对象等同的时候

```objc
- (void)testHash2 {
    
    User *user1 = [User new];
    user1.uid = 1;
    user1.uname = @"hahaha";
    user1.upassword = @"jjjjjjj";
    
    User *user2 = [User new];
    user2.uid = 1;
    user2.uname = @"hahaha";
    user2.upassword = @"jjjjjjj";
    
    User *user3 = [User new];
    user3.uid = 1;
    user3.uname = @"hahaha";
    user3.upassword = @"jjjjjjj";
    
    User *user4 = [User new];
    user4.uid = 1;
    user4.uname = @"hahaha";
    user4.upassword = @"jjjjjjj";
    
    BOOL flag1 = [user1 isEqual:user2];
    BOOL flag2 = [user2 isEqual:user3];
    BOOL flag3 = [user3 isEqual:user4];
    
    NSUInteger hash1 = [user1 hash];
    NSUInteger hash2 = [user1 hash];
    NSUInteger hash3 = [user1 hash];
    NSUInteger hash4 = [user1 hash];
    
    NSSet *set = [NSSet setWithObjects:user1, user2, user3, user4, nil];
    
}
```

输出如下

```
(User *) user1 = 0x00007fc4f0507710
(User *) user2 = 0x00007fc4f0511510
(User *) user3 = 0x00007fc4f050d0a0
(User *) user4 = 0x00007fc4f05193f0

(BOOL) flag1 = NO
(BOOL) flag2 = NO
(BOOL) flag3 = NO

(NSUInteger) hash1 = 140483822122768
(NSUInteger) hash2 = 140483822122768
(NSUInteger) hash3 = 140483822122768
(NSUInteger) hash4 = 140483822122768

(__NSSetI *) set = 0x00007fc4f05247b0 4 objects
```

- NSObject实现的isEqual:方法返回上面的对象都不是等同的，但是属性值确实都是相等的（后面自己修改下模拟NSString的实现）
- 但是如上的几个User对象其实属性值都相等，如果此时让NSSet只会保存一个User对象，就需要让 -[User isEqual:] 返回YES.

####下面看个例子二、自己重写User类的`isEqual:`判断对象等同的时候

- 首先对User类进行改写

```objc
@interface User : NSObject <NSCopying, XZHDeepCopying>

//....省略

// 判断对象等同
- (BOOL)isEqualToUser:(User *)user;

@end
```

```objc
@implementation User

//其他省略....

- (NSUInteger)hash {
    NSUInteger uidhash = _uid;
    NSUInteger unamehash = [_uname hash];
    NSUInteger upasshash = [_upassword hash];
    return uidhash ^ unamehash ^ upasshash;
}

- (BOOL)isEqual:(id)object {
    if ([object class] == [self class]) {
        return [self isEqualToUser:object];
    } else {
        return [super isEqual:object];
    }
}

- (BOOL)isEqualToUser:(User *)user {
    
    // 先看地址是否相同
    if (user == self) return YES;
    
    // 再看属性值是否相同
    if (_uid != user.uid) return NO;
    if (![_uname isEqualToString:user.uname]) return NO;
    if (![_upassword isEqualToString:user.upassword]) return NO;
    
    return YES;
}

@end
```

- 在执行前面的testHash2方法之后，输出如下

```
(User *) user1 = 0x00007fa0b0f1a660
(User *) user2 = 0x00007fa0b0f14890
(User *) user3 = 0x00007fa0b0f19f30
(User *) user4 = 0x00007fa0b0f19f60

(BOOL) flag1 = YES
(BOOL) flag2 = YES
(BOOL) flag3 = YES

(NSUInteger) hash1 = 4999407706194899149
(NSUInteger) hash2 = 4999407706194899149
(NSUInteger) hash3 = 4999407706194899149
(NSUInteger) hash4 = 4999407706194899149

(__NSSetI *) set = 0x00007fa0b0f19f90 1 object
```

OK，这样就达到目标了，主要就是重写isEqual:自己完成对象的等同判断的逻辑.

****

###Dictionary

本身就是按照key-value形式存储，随时可以对key指向的value进行修改.


