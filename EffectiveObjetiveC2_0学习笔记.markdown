
## Objetive-C的runtime库，封装了大量的c数据结构

> http://www.gnustep.org/resources/downloads.php GNUStep源码下载地址

- (1) 一个NSObject类的c结构

```objc
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
```

这个`objc_class`实例可以表示两个版本的类: 类本身和元类。类本身存放类对象的实例变量、方法实现。元类存放类的方法实现。

- (2) 一个NSObject类的一个对象的c结构

```objc
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};
```

- (3) 类对象的实例变量的c结构

```objc
struct objc_ivar {
    char *ivar_name                                          OBJC2_UNAVAILABLE;
    char *ivar_type                                          OBJC2_UNAVAILABLE;
    int ivar_offset                                          OBJC2_UNAVAILABLE;
#ifdef __LP64__
    int space                                                OBJC2_UNAVAILABLE;
#endif
} 
```

- (4) NSObject类/NSObject类的一个对象，中的一个Objetive-C函数的c结构

```objc
struct objc_method {
    SEL method_name                                          OBJC2_UNAVAILABLE;
    char *method_types                                       OBJC2_UNAVAILABLE;
    IMP method_imp                                           OBJC2_UNAVAILABLE;
}                                                            OBJC2_UNAVAILABLE;
```

- (5) 一个Objetive-C函数的唯一标示的c结构

```objc
objc_selector 没有找到结构定义
```

- (6) 用来指向最终的OC函数对应的c函数实现的万能指针IMP

```objc
#if !OBJC_OLD_DISPATCH_PROTOTYPES
typedef void (*IMP)(void /* id, SEL, ... */ ); 
#else
typedef id (*IMP)(id, SEL, ...); 
#endif
```

可以看出来IMP就是一个指向 `id func(id, SEL, ...); ` 这样格式的c函数的一个指针类型。


还有很多的结构，常用的就这几个。也就是说我们所编写的所有的Objetive-C代码，在代码进行编译的时候会自动编译生成使用如上这些c结构体表示的c代码，也就是打包在ipa包里面的可执行的二进制文件中。

其实最终运行在iOS系统中的我们写的OC代码，都是这些上面的一个一个的c数据结构的具体实例。

## `NSObject类也是一个对象` 的说法


我们写的一个NSObject类，最终程序运行时，就是使用一个`objc_class`结构体实例所保存，但是又包含:

- `ivars` 记录所有的实例变量
- `methodLists` 记录所有的Objective-C函数
- `protocols` 记录实现的所有的Protocol协议
- `cache` 是为了消息查询Method做的一个内存缓存

所以确切的说，NSObject类应该是`objc_class`结构体的一个实例，而且是`单例`。

## Objective-C是建立在`Runtime system 运行时环境`基础上，而并非是编译器基础

上面使用到的所有的c数据结构，以及所有的runtime c函数，都是由这个Runtime库所提供。比如:

- (1) 内存管理时代中，对NSObejct对象进行内存管理的函数
	- release
	- retain
	- autorelease

- (2) 以及熟悉的运行时c函数
	- `object_getClass()`
	- `class_addIvar()`
	- `class_addProperty()`
	- `class_addMethod()`
	- `class_replaceMethod()`
	- `method_exchangeImplementations()`


Runtime运行时环境，其实就是一个动态链接库`dynamic library`，和开发者开发的App程序代码`粘连`在一起。

而使用动态链接库`dynamic library`与App程序代码进行`粘连`的好处是不再需要编译期确定与库代码的关系:

- (1) 静态库: 
	- 每次库工程代码更新后，都需要重新编译生成二进制文件
	- 让引用的App主工程`编译期`重新引入新的静态库二进制文件，并让App工程也`重新编译`生成新的ipa包

- (2) 动态库: 
	- 每次库工程代码更新后，同样需要重新编译生成二进制文件
	- 只是不再需要让App主工程在`编译期`去引入库代码
	- 而是推迟到App程序`运行时`动态导入库代码

## 所有的Objective-C对象，都必须使用`指针`来指向，因为对象的内存总是分配在`堆`上，而非`栈`上

- (1) 错误的对象引用

```objc
NSString string;
```

- (2) 正确的对象引用

```objc
NSString *string;
```

如果多个`栈上指针` 指向 同一个`堆上对象`:

```objc
- (void)test {
	NSString *str1 = @"Hello World!";
	NSString *str2 = str1;

}
```

运行代码断点输出

```
(__NSCFConstantString *) str1 = 0x00000001090181e0 @"Hello World!"
(__NSCFConstantString *) str2 = 0x00000001090181e0 @"Hello World!"
```

可以看到指针str1与指针str2，都是指向同一个内存地址，而这个内存地址中存放的数据就是`Hello World!`字符串。

在`栈帧 stack frame`里分配了两块内存，来分别存放指针str1与指针str2的值:

- (1) 内存块1、存放`Hello World!`字符串在堆区的地址`0x00000001090181e0 `
- (2) 内存块2、同样存放`Hello World!`字符串在堆区的地址`0x00000001090181e0 `


注意: 1) 在32位系统上，一个指针变量占用的长度是4字节。在64位系统上，一个指针变量占用的长度是8字节。 2) `指针变量`都是分配在`栈`上。

![](http://ww1.sinaimg.cn/large/74311666jw1f5w8ubpnlzj21a00fgmzc.jpg)

### 分配在`栈`上的内存会在退栈时自动清除，但是分配在`堆`上的内存必须手动释放。

所以需要保存指针变量，一旦出了方法栈就会被清除，导致`堆`上的内存块无法再找到访问的地址。

Objective-C语言中，将管理内存的c系统方法`malloc()/free()`等方法封装了，并引入了`引用计数器 retainCount`的概念来作为是否废弃对象的标准。

现在基本上都是使用`ARC (Automatic Reference Counting)`，已经屏蔽了之前`MRC`所有手动内存管理的函数api使用。改成了如下的内存管理修饰符:

- (1) `__unsafe_unretained` 类似 assign，`不会`修改修饰的指针变量指向的对象的retainCount

```
特别注意: 和assign一样，当所指向对象被释放之后，继续使用该指针变量，就会造成程序崩溃，因为不会自动设置为nil，也就是直接是一个野指针
```

- (2) `__strong` 类似 retain，会修改修饰的指针变量指向的对象的`retainCount++`

- (3) `__weak` 类似 weak，`不会`修改修饰的指针变量指向的对象的retainCount


- (4) `__autoreleasing` 类似 autorelease，自动完成`retainCount++和retainCount--`

MRC中对一个对象进行持有的代码:

```objc
Person *obj = [person retain];
```

ARC中对一个对象进行持有的代码:

```objc
@interface ViewController() {
	Person __strong *_obj;
}
```

```objc
_obj = person;
```

在ARC下，只要一个对象增加一个`__strong`修饰符修饰的指针变量指向，就意味着执行`[对象 retain]`，且ARC下默认不写任何的内存管理修饰符就默认为`__strong`修饰符。

MRC下对一个方法解除持有，对其释放的代码:

```objc
[_obj release];
```

ARC下对一个方法解除持有，对其释放的代码:

```objc
_obj = nil;
```

在ARC下，只要将指向的指针变量赋值nil，即不再指向那个对象，那么那个对象就会执行释放（release）操作，即相当于`[person release]`。

### 对于一些简单的数据结构（int、float、double、char等）封装，建议使用`c结构体`的方式，而非`NSObject子类`

因为创建OC类对象比创建结构体实例会更消耗资源（需要生产各种objc结构体实例记录、消息传递、消息转发、函数实现查询...等等一系列运行时计算）。

可以参看YYKit中一些组件，大量使用了c struct、c函数，因为这样可以减少一些不必要的Objective-C的消息传递过程，直接可以调用函数实现。

## `xxx.h`中尽量使用`@class 类`来告诉编译器需要这么一个类，而非`#import 'xxx.h'`直接导入其所有的具体实现代码

Person类需要使用Address类

```objc
#import <Foundation/Foundation.h>

//.h中向前导入
@class Address;

@interface Personn : NSObject

@property (nonatomic, strong) Address *addr;

@end
```

```objc
#import "Personn.h"

//.m导入具体依赖的.h
#import "Address.h"

@implementation Personn

@end
```

使用@class 类向前声明类的好处:

- (1) 延迟引入`Address.h`的时机，只在需要使用到他的内部实现的时候才导入

- (2) 防止导入过多没用的.h，从而导致增加整个工程的编译时间

- (3) 解决了两个类相互引用之后，编译器报错的问题
	- A.h 中 import B.h
	- B.h 中 import A.h


## 协议声明的位置

- (1) 对一类事物的抽象:

最好放在一个单独的.h文件中，避免导入过多不相关的代码。

- (2) 回调delegate的抽象:

最好定义在对应的View或者Manager的头文件.h中，就不要写成单独的.h中了。


```objc
/**
 *  回调事件处理
 */
@protocol FQLDropMenuViewDelegate <NSObject>
@optional
- (void)dropMenuViewItemDidClick:(NSInteger)index;
- (void)dropMenuViewDidScroll;
@end

/**
 *  回调获取显示的数据
 */
@protocol FQLDropMenuViewDataSource <NSObject>
@optional
/**
 *  返回 @[@"111", @"222", @"333"]
 */
- (NSArray *)dropMenuViewDataSources;
@end

@interface FQLDropMenuView : UIView

- (instancetype)initWithFrame:(CGRect)frame delegate:(id<FQLDropMenuViewDelegate>)delegate dataSource:(id<FQLDropMenuViewDataSource>)dataSource;

- (void)reloadData;

@end
```

## 多用字面量，少用与之等价的方法函数

- (1) NSString 

使用字面量

```objc
NSString *str = @"Hello World~~~";
```

如果使用函数

```objc
NSString *str = [NSString stringWithFormat:@"%@", @"Hello World~~~"];
```

```objc
NSString *str = [[NSString alloc] initWithString:@"Hello World~~~"];

//注意： Xcode编译会提示让你替换成使用字面量来初始化字符串
```

- (2) NSNumber

```objc
NSNumber *int_num = @(1);
NSNumber *float_num = @(2.5f);
NSNumber *double_num = @(2.444444);
NSNumber *bool_num = @(YES);
NSNumber *char_num = @('A');
```

- (3) NSArray

```objc
NSArray *array = @[@"1", @"2", @"3"];
```

```objc
NSMutableArray *mutablArray = [@[@"1", @"2", @"3"] mutableCopy];
```

但是注意，使用字面量创建数组对象时，如果其中`某个对象 == nil`，则会引起程序崩溃，但是使用NSArray函数创建对象则不会引起崩溃。

```objc
- (void)test {
    NSArray *array1 = @[@"1", @"2", @"3"];
    NSArray *array2 = nil;
    NSArray *array3 = nil;
    
    NSArray *array4 = [NSArray arrayWithObjects:array1, array2, array3, nil];
    NSLog(@"array4 complet");
    
    NSArray *array5 = @[array1, array2, array3];
    NSLog(@"array5 complet");
}
```

程序运行结果

```
2016-07-17 14:52:25.851 Demo[7303:111069] array4 complet
2016-07-17 14:52:25.884 Demo[7303:111069] *** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '*** -[__NSPlaceholderArray initWithObjects:count:]: attempt to insert nil object from objects[1]'
```

因为arrayWithObjects:函数会自动发现`第一个为nil`的对象，就会结束继续向数组添加对象，从而避免添加nil对象导致程序崩溃。

注意，此问题对于NSDictionary同样适用。

- (4) NSDictionary

```objc
NSString *key = ....;
NSString *value = ....;

NSDictionary *dtc = @{
                      (key ? key : @"") : (value ? value : @""),
                      };
```

注意，无法使用 `字面量` 的形式来创建一些其他的NSObject类的对象:

- (1) 自定义的NSObject类的对象
- (2) NSDate、NSData .... 

## 对于NSArray、NSDictionary、NSNumber这些类，其实都是`类簇`类，所以一般都不会去继承自他们覆写自己的子类。

我们通过NSArray、NSDictionary、NSNumber得到的对象，并非是我们看到的类型，而是内部私有子类的一个对象。

所以，即使我们重写NSArray、NSDictionary、NSNumber，也不会影响到内部的私有类。所以从来没看到谁重写过NSArray、NSDictionary、NSNumber....

而且就算是重写，也不一定能比苹果的实现写的好。

之前我测试的一些Foundation对象的真实类型:

- (1) NSArray、NSMutableArray类簇类所隐藏的内部私有类

```objc
__NSArrayI、__NSArray0、__NSArrayM、__NSPlaceHolderArray
```	

- (2) NSSet、NSMutableSet类簇类所隐藏的内部私有类

```objc
__NSSetI、__NSSingleObjectSetI、__NSSetM
```	

- (3) NSDictionary、NSMutableDictionary类簇类所隐藏的内部私有类

```objc
__NSDictionary0、__NSDictionaryI、__NSDictionaryM
```	

- (4) NSDate类簇类

```objc
__NSDate
```	

- (5) NSData、NSMutableData类簇类所隐藏的内部私有类

```objc
_NSZeroData、NSConcreteData、NSConcreteMutableData
```	

- (6) NSNumber类簇类所隐藏的内部私有类

```objc
__NSCFNumber、NSDecimalNumber
```	

- (7) NSString、NSMutableString类簇类所隐藏的内部私有类

```objc
__NSCFConstantString、NSTaggedPointerString、__NSCFString
```

- (8) NSValue类簇类

```objc
NSConcreteValue
```

- (9) NSBlock类簇类（注意: NSBlock本身也是私有api类）

```objc
__NSGlobalBlock__、__NSMallocBlock__、__NSStackBlock__
```

## 多用类型常量，而少用`#define`声明的宏常量

- (1) 使用宏声明常量

```objc
#define AGE 10
```

- (2) 使用类型常量(.h中暴露声明，.m中具体定义)

.h声明类型常量 >>> 告诉别人有这么一个常量可以使用

```objc
FOUNDATION_EXPORT NSString *const kNotificationKey;
```

.m具体定义类型常量 >>> 类型常量具体保存的什么值

```objc
NSString *const kNotificationKey = @"kNotificationKey";
```

如上两种写法的区别:

- 宏常量
	- 将当前整个源文件中，所有的`AGE`全部`替换`成10
	- 不做类型判断、不做语法检查，就是简单的全文替换

- 类型常量
	- 做类型检查、语法检查
	- `static`可以让该常量定义只在`当前文件中`有效，即使导入到其他文件中有一个同名的常量也可以
	- `const`修饰为不可被修改

	
对于一些字符串形式的常量定义，比如通知key等等最好使用类型常量，而不是要是#define宏

- (1) 两个不同的字符串类型常量，比较的时候直接比较`指针指向的地址`是否相同

- (2) 两个`#define`定义的字符串宏，比较则是`从头到尾`一个一个字符进行比较


如果不希望某个类型常量在`其他文件`中使用，那么类型常量应该只在`xxx.m` 中使用 `static` 定义:

```objc
static const NSTimeInterval kAnimationTime = 3.f;
```

## 使用枚举类表示 状态、选项、状态码，且switch不要写default分支处理

.....

##Objetive-C对象的属性 `@property`

###首先稍微记录下属性的基本东西

- (1)  `@property = ivar + getter + setter;`
	- 对于一个Objetive-C对象，最终都是使用实例成员变量`Ivar`来保存数据
	- 只不过@property帮忙生成了三个东西:
		- 对象的实例变量Ivar
		- Ivar的setter方法实现
		- Ivar的getter方法实现

- (2) 对于实例成员变量，通常要提供`getter/setter`方法来读取/写入，并非直接操作`Ivar`本身
	- 因为一个对象所有的实例变量的内存布局都是在编译的时刻全部计算写死，后续无法再进行更改
	- 可能会出现Iavr的内存布局的错乱，导致读取到错误的Ivar
	- 后续可以证明: 对于已经注册到runtime system的OC类，是无法通过运行时的函数修改Ivar，也正是为了规避这个问题

- (3) `objective-C 2.0`中，可以通过` @property`修饰符告诉编译器`自动生成`实例变量的`getter/setter方法的实现`
	- 之前的Xcode版本需要使用 `@property + @synthesize` 配合

- (4) objective-C 2.0中，还引入了点语法`对象.属性`的方法来起到调用getter/setter方法实现
	- `self.name = @"hahah";` >>> `[self setName:@"hahaha"];`
	- `self.name;` >>> `[self name];`

- (5) 如果重写了setter与getter，此时编译器就不会自动生成实例变量Ivar

![](http://i2.buimg.com/4851/4c20c65a6fc1d125.png)

此时就必须要自己去手动声明和使用Ivar

```objc
@interface PropertyViewController ()
@property (nonatomic, copy) NSString *name;
@end

@implementation PropertyViewController {
    NSString *_name;//必须手动添加
}

- (void)setName:(NSString *)name {
    _name = [name copy];
}

- (NSString *)name {
    return [_name copy];
}
@end
```

- (6) 子类继承父类的属性

通常情况下是这么写的

```objc
@interface Father : NSObject
@property (nonatomic, copy) NSString *name;
@end
@implementation Father
@end
```

```objc
@interface Son : Father
- (void)logName;
@end
@implementation Son
- (void)logName {
    NSLog(@"%@", self.name);//必须使用 self.name 才能访问
}
@end
```

可以看到子类Son对象，只能通过`self.name`才能去操作Ivar，因为这个Ivar（`_name`）是在父类Father中。也就是说Son对象可以使用Ivar，但是这个Ivar并非直接属于Son对象。所以只能通过setter/getter方法实现去访问这个Ivar。

那如果，希望Son对象也能够使用`_name`这样能直接访问Ivar了？使用`@synthesize`重新生成属性方法存取的Ivar。

```objc
@interface Son : Father
- (void)logName;
@end
@implementation Son
@synthesize name = _sonName;
- (void)logName {
    NSLog(@"%@", _sonName);
}
@end
```

但是好像一般没怎么见过这么搞的，因为这样改变了Ivar了，可能会造成一些意想不到的问题吧。

- (7) 属性的读写权限

```objc
@interface Father : NSObject
@property (nonatomic, copy, readonly) NSString *name;
@end
@implementation Father
@end
```

那么如下操作name属性的代码就会被编译器报错:

```objc
Father *f = [Father new];
f.name = @"haha";
```

![](http://i4.buimg.com/4851/ea0537f4aa9984cf.png)

那么一定要去访问了？

一、使用KVC:

```objc
Father *f = [Father new];
[f setValue:@"hahah" forKey:@"_name"];
```

二、objc_msgSend(): >>> `发现是不行的，虽然编译器不报错，但是运行时程序崩溃， -[Father setName:]: unrecognized selector sent to instance 0x7fb22a658ab0`

```objc
Father *f = [Father new];
((void (*)(id, SEL, NSString*)) (void *) objc_msgSend)(f, @selector(setName:), @"hahah");
```

三、分类修改Ivar的setter实现

```objc
@interface Father (ACCESS)
@property (nonatomic, copy, readwrite, setter=a_setName:) NSString *name;
@end
@implementation Father (Access)
- (void)a_setName:(NSString *)name {
    _name = name;
}
@end
```

- (8) 使用`@package`修饰Ivar，可以让当前.m中所有代码，都可以访问到该对象的Ivar

```objc
@interface Father : NSObject {
    @package
    NSString *_age;
}
@end
@implementation Father
@end
```

```objc
@implementation PropertyViewController
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    Father *f = [Father new];
    f->_age = @"aaaa";
}

@end
```

- (9) `@proeprty (copy) NSString *name;`默认使用`atomic`而不是`nonatomic`
	- atomic: 原子性，完成多线程同步
	- nonatomic: 锁/信号/队列，完成多线程同步


##@synthesize的使用

Xcode 4.5之前的版本，@property 与 @synthesize 必须配合使用:

- (1) @property 负责在.h中，声明实例变量的getter/setter
- (2) @synthesize 负载在.m中，告诉编译器在编译期生成:
	- 实例变量的定义
	- getter/setter的具体方法实现

那么在Xcode5之后的版本，使用一个@property即可完成如上所有的事情了。

- 使用 @synthesize 修改 @property默认生成的实例变量:

```objc
#import <Foundation/Foundation.h>

@interface Cat : NSObject

@property (nonatomic, copy) NSString *name;

- (void)log;

@end

@implementation Cat

@synthesize name = _catName;//修改默认的实例变量 _name

- (void)log {
    NSLog(@"_catName = %@", _catName);
}

@end
```

测试代码

```objc
- (void)demo3 {
    Cat *c = [Cat new];
    c.name = @"Cat 1111";
    [c log];
}
```

运行结果

```
2016-07-17 18:06:30.366 Demo[17589:276864] _catName = Cat 1111
```

可以看到@synthesize成功修改了实例变量。

##使用@dynamic告诉编译不要自动生成getter/setter实现，以及实例变量的定义

```objc
#import <Foundation/Foundation.h>

@interface Cat : NSObject

@property (nonatomic, copy) NSString *name;

- (void)log;

@end

@implementation Cat

@dynamic name;//告诉编译不要自动生成getter/setter实现，以及实例变量声明

- (void)log {
    NSLog(@"_name = %@", _name);//会报错，没有定义 _name
}

@end
```

![](http://7xrn7f.com1.z0.glb.clouddn.com/16-7-17/76549528.jpg)

那么就需要自定定义实例变量，以及实例变量的getter/setter实现

```objc
#import <Foundation/Foundation.h>

@interface Cat : NSObject

@property (nonatomic, copy) NSString *name;

- (void)log;

@end

@implementation Cat {
    NSString *_name;
}

@dynamic name;//告诉编译不要自动生成getter/setter实现，以及实例变量声明

- (void)log {
    NSLog(@"_name = %@", _name);
}

- (void)setName:(NSString *)name {
    _name = [name copy];
}

- (NSString *)name {
    return _name;
}

@end
```

如在CoreData中，使用@dynamic告诉编译不要自动生成NSManagedObject子类属性对应的实例变量以及实例变量的getter/setter方法实现:

- 通常是在写继承自NSManagedObject子类时，或者由CoreData根据表结构自动创建出来的NSManagedObject子类

- 会将属性设置为 `@dynamic` ，意味着不会生成实例变量，以及实例变量的setter与getter实现

CoreData中使用@dynamic阻止编译器生成属性的getter与setter方法的目的是因为:
	
- 当执行`[self performSelector:@selector(setName:) withObject:@"haha"];`时，因为没有对应的实现，就会走到`消息转发阶段1中的 resolveInstanceMethod:函数`中

- 然后通过`class_addMethod(cls, IMP)` 完成setter与getter方法实现的动态添加


如下简单模拟CoreData通过动态方法解析时，添加对应的属性方法实现的简单代码分析

> 核心: 利用执行一个不存在实现的SEL，进入消息转发阶段1中的`resolveInstanceMethod:`函数，然后使用`class_addMethod()`完成方法添加。

```
#import <Foundation/Foundation.h>
#import <CoreData/CoreData.h>

@interface Entity : NSManagedObject

@property (nonatomic, strong) NSNumber *uid;
@property (nonatomic, strong) NSString *name;

@end
```

```
#import "Entity.h"
#import <objc/runtime.h>

//////////////////////////////////////////////////////////////////////////
////如下几个c实现函数就是属性对的getter/setter 这两个OC消息最终对应的c函数实现
////并且是由CoreData自动通过编译器添加到源文件中

//uid的getter与setter
id autoUIDGetter(id self, SEL sel);
void autoUIDSetter(id self, SEL sel, id value);

//name的getter与setter
id autoNameGetter(id self, SEL sel);
void autoNameSetter(id self, SEL sel, id value);

//////////////////////////////////////////////////////////////////////////

@implementation Entity

//////////////////////不自动生成属性的getter/setter的方法实现//////////////////////
@dynamic uid;
@dynamic name;


//////////////////////  当执行属性方法实现没有找到时，进入消息转发阶段1 //////////////////////
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    NSString *selName = NSStringFromSelector(sel);
    
    if ([selName hasPrefix:@"set"]) {
        //所有的setter
        
        //判断setUid还是setName
        NSString *ivarName = [selName substringFromIndex:2];
        
        if ([ivarName isEqualToString:@"uid"]) {
            //setUid:
            class_addMethod([selName class],
                            sel,
                            (IMP)autoUIDSetter,
                            "v@:@");
            
        } else if ([ivarName isEqualToString:@"name"]) {
            //setName:
            class_addMethod([selName class],
                            sel,
                            (IMP)autoNameSetter,
                            "v@:@");
        }
        
    } else {
        //所有的getter
        
        if ([selName isEqualToString:@"uid"]) {
            //getUid
            class_addMethod([selName class],
                            sel,
                            (IMP)autoUIDGetter,
                            "@@:");
        } else if ([selName isEqualToString:@"name"]) {
            //getName
            class_addMethod([selName class],
                            sel,
                            (IMP)autoNameGetter,
                            "@@:");
        }
    }

    return [super resolveInstanceMethod:sel];
}

@end
```

##@property 中的几种类型的修饰符: 

### nonatomic / atomic

- 多线程环境下，对同时执行的多个线程进行互斥同步的方法
	- (1) Lock/Condition 锁 
	- (2) 原子性操作 (OSAtomic库)
	- (3) GCD....等不属于此时属性修饰能够指定的

- 属性默认使用 `atomic` 修饰符，由编译器完成`原子性加锁操作`来`同步`属性变量的多线程访问，会降低性能

```c
#import <libkern/OSAtomic.h>

int32_t i = 0;
OSAtomicIncrement32(&i);//自增
OSAtomicDecrement32(&i);//自减
OSAtomicAdd32(3, &i);//加上一个数
//还有很多函数...
```

对于一些简单的基本数据类型的变量，可以优先考虑使用OSAtomic.h提供的一些原子性操作的函数进行多线程同步，而且效率会更高。

- nonatomic 需要手动执行多线程的同步，不会由编译器提供原子性操作代码，是非线程安全的，但是性能高。常见的多线程同步方法:
	- 锁: NSLock、NSRecursiveLock、NSConditionLock
	- 信号: `dispatch_semaphore_t`
	- GCD队列: 

### readwrite / readonly 

属性的读写权限

###属性变量的内存管理策略: `weak / strong / retain / copy / assign / unsafe_unretained`

调用setter时，如何对传入的值进行内存管理

- weak 
	- 不会也不能持有指向的对象，即不会让指向的对象的retainCount++
	
- strong/retain 
	- 二者是一样的对象所有权，即持有指向的对象，会让指向的对象的retainCount++
		- (1) 释放已有的老对象，`[老对象 release]`，老对象retainCount--
		- (2) 持有传入的新对象，`[新对象 retain]`，新对象retainCount++
	
- copy 
	-  其实和strong/retain很相似的，`只是多一步拷贝`的操作
	-  将传入的对象首先进行拷贝（`[对象 copy]`）继而会生成一个`新的对象`
		- (1) `id newObj = [oldObj copy]`;
	-  然后使用 strong/retain强引用的方式来持有拷贝出来的新对象
		- (2) 释放已有的老对象，`[oldObj release]`，老对象retainCount--
		- (3) 持有传入的新对象，`[newObj retain]`，新对象retainCount++
	
- assign:
	- 一般使用一些基本数据类型的属性变量（int、float、bool...）
	- 类似weak，但是区别是，在指向的对象被废弃掉时，指针变量值不会自动赋值为nil
	- 而weak修饰的指针变量会自动赋值nil

- `unsafe_unretaind`
	- 类似assign，只不过可以对OC对象指针进行修饰
	- 同样和assign在指向的对象被废弃掉时，指针变量值不会自动复制为nil


如上都是属性内存管理的修饰符，那么对应iOS系统在ARC环境下的对象内存修饰符:

```
__assign
__strong
__weak
__unsafe_unretaind
```

其实没有copy这个在属性定义时使用的内存管理修饰类型，就是在`__strong`的基础上封装了一个`-[NSObject copy]`的操作。

所以，没有实现NSCopying协议的类型，千万不要是copy属性修饰符，会程序崩溃的。
	
### 设置属性变量的值时，遵循内存管理修饰类型

```objc
@interface Person : NSObject

@property (nonatomic, assign) int age;

@property (nonatomic, copy) NSString *name;

@property (nonatomic, strong) NSArray *childs;

@property (nonatomic, strong) User *user;

@end
```

```objc
@implementation Person

- initWithAge:(NSInteger)age
         Name:(NSString *)name
       Childs:(NSArray *)childs
         User:(User *)user
{
    self = [super init];
    if (self) {
        //assign 修饰的属性
        _age = age;
        
        //copy 修饰的属性
        _name = [name copy];
        
        //strong 修饰的属性
        //会调用 setXxx: 方法，内部会根据属性的内存管理修饰类型
        //进行计数器的具体处理
        _childs = childs;
        _user = user;
    }
    return self;
}

@end
```

## @property对外只读，对内读写，以及子类继属性

接着上面变量一般声明称私有的，那么在子类中就无法直接使用 `_name` 来访问这个私有成员变量。

- (1) 父类属性向外只读

```objc
#import <Foundation/Foundation.h>

@interface Father : NSObject

//向外暴露只读属性
@property (nonatomic, readonly, assign) BOOL isEnable;
@property (nonatomic, readonly, copy) NSString *name;

//@property (nonatomic, copy) NSString *name;

@end
```

```objc
#import "Father.h"

@interface Father ()

//对内读写属性
@property (nonatomic, readwrite, assign) BOOL isEnable;
@property (nonatomic, readwrite, copy) NSString *name;

@end

@implementation Father {
    NSString *_name;
}

//@synthesize name = _name;//注释这句话，就会马上编译报错

- (void)dealloc {
    _name = nil;
}

- (NSString *)name {
    if ([_name hasPrefix:@"haha"]) {
        return _name;
    } else {
        return @"nil";
    }
}

- (void)setName:(NSString *)name {
    if ([_name hasPrefix:@"haha"]) {
        _name = [name copy];
    }
}

@end
```

- (2) 子类.m实现中`修改`父类中`只读属性权限`为读写权限

```objc
#import "Father.h"

@interface Son : Father

- (void)logName;

@end

/* 声明父类的匿名分类，将父类只读属性设置为读写 */
@interface Father ()

@property (nonatomic, readwrite, assign) BOOL isEnable;
@property (nonatomic, readwrite, copy) NSString *name;

@end

@implementation Son

- (void)logName {
    
    //1. 访问父类私有变量
    NSLog(@"name = %@", self.name);
    
    //2. 读写修改权限后的属性
    self.isEnable = YES;
    self.name = @"Hello World!!!";
}

@end
```

## 对象内部进行属性变读取时，尽量使用 `_varibale` 直接操作实例变量

对于实例变量访问方式有如下两种：

- 通过属性变量 `_variable` 来读写
- 通过 `self.variable` 来读写


### 通过 `对象.属性` 进行读取
	
- 可以懒加载的属性
- **调用setter设置新值时，会触发KVO通知**
- 设置新值时，会根据属性定义的`内存管理`对应的修饰规则
	- assign、copy、weak、strong、unsafe_unretained ...
- 还可以重写 setter与getter 方法，完成断点调试

### 通过 `_属性名` 进行读取

- `绕过`了属性定义的`内存管理`修饰
- `绕过`了属性定义的`读写权限`修饰（可以使用KVC操作私有实例变量）
- 无法触发KVO通知

### 合理组合上面两种形式

- (1) 对象内部，大多数情况下都应该如下
	- `读`实例变量 >>> 使用 `_variable` 来读实例变量，避免每次进入消息发送进入class结构体中查询SEL对应的IMP
	- `写`实例变量 >>> 使用 `self.属性名 = 新值` 来写数据，按照属性定义的内存管理修饰，以及提供KVO通知

- (2) 对象内部，`initXxx`函数与`dealloc`函数中，总是应该是通过`_variable`进行`读和写`
	- 因为`子类`可能重写属性的setter与getter方法实现
	- 那么调用子类的setter时，就会导致父类方法中的某一些实例变量未能初始化，导致程序崩溃

- (3)对于使用`懒加载`的属性，总应该是使用 `self.属性名`来进行`读`


对于对象的懒加载，我看到很多人都是将所有的UI对象都统统作为懒加载，我觉得有点是不是设计过度了。我觉得分情况:

- (1) 如果这个对象必须要马上使用，那么就在容器对象init的时候直接创建出来并使用实例变量保存以便后续使用

- (2) 而如果只有满足某一个条件时刻才会去使用这个对象，且创建的成本本较大，那么这个对象才应该去做成懒加载

## 对象的`等同性`，即判断两个对象是否是等价的


判断两个对象的等同性，并不是直接通过`A对象 == B对象`比较，这样比较的是两个对象的地址，而并非两个对象内部的所有实例变量的值内容。

NSObject协议中，已经声明了用于判断对象等同的方法定义

```objc
@protocol NSObject

//是否等同
- (BOOL)isEqual:(id)object;

//当前对象的hash值
@property (readonly) NSUInteger hash;

....省略
```

如上贴出来的两个重要的东西:

- (1) 判断对象等同的方法
- (2) 获取当前对象的hash值的属性getter方法

那么对于如上两个东西他们之间是有如下关系的:

- 如果`[A对象 isEqual:B对象] == YES` >>>>> `[A对象 hash] == [B对象 hash]`
- 但是反过来，当A对象与B对象的的hash相等时，并不一定有`[A对象 hash] == [B对象 hash]`


有Person类如下

```objc
#import <Foundation/Foundation.h>

@interface Person : NSObject

@property (nonatomic, copy) NSString *pid;
@property (nonatomic, assign) NSInteger age;
@property (nonatomic, copy) NSString *name;

- (instancetype)initWithPid:(NSString *)pid Age:(NSInteger)age Name:(NSString *)name;

@end

@implementation Person

- (instancetype)initWithPid:(NSString *)pid Age:(NSInteger)age Name:(NSString *)name {
    self = [super init];
    if (self) {
        _pid = [pid copy];
        _name = [name copy];
        _age = age;
    }
    return self;
}

@end
```

测试下`-[NSObject isEqual:]`方法

```objc
- (void)test {
    Person *p1 = [[Person alloc] initWithPid:@"111" Age:19 Name:@"xzh"];
    Person *p2 = [[Person alloc] initWithPid:@"111" Age:19 Name:@"xzh"];
    
    BOOL ret = [p1 isEqual:p2];
//断点 
}
```

断点后输出

```
(Person *) p1 = 0x00007fe3fc827ac0
(Person *) p2 = 0x00007fe3fc827b30
(BOOL) ret = NO
```

虽然这是两个不同的Person对象，但是他们的每一个子数据却是一模一样的。但是看输出结果是NO，表示这两个对象并不是等价的。这很明显从数据内容上看，这两个对象应该是等价的。

那么对于我们自定义类型的对象，需要实现等价判断，就需要自己实现NSObejct对象的isEqual:方法实现

### 重写`-[NSObejct isEqual:]`实现，完成自己类对象的等同性比较


```objc
@implementation Person

- (instancetype)initWithPid:(NSString *)pid Age:(NSInteger)age Name:(NSString *)name {
    self = [super init];
    if (self) {
        _pid = [pid copy];
        _name = [name copy];
        _age = age;
    }
    return self;
}

- (BOOL)isEqual:(id)object {
    
    //1. 先比较对象的地址，及指针变量
    if (self == object) return YES;
    
    //2. 在比较这两个对象的所属类型（objc_class单例），如果类型都不一致，直接就不等价了
    if ([object class] != [self class]) return NO;
    
    //3. 将传入的id对象进行类型强转
    Person *person = (Person *)object;
    
    //4. 依次比较对象内部的子数据项
    if (_age != person.age) return NO;
    if (![_name isEqualToString:person.name]) return NO;
    if (![_pid isEqualToString:person.pid]) return NO;
    
    //5. 默认返回YES，表示对象等价
    return YES;
}

@end
```

再测试下`-[NSObject isEqual:]`方法

```objc
- (void)test {
    Person *p1 = [[Person alloc] initWithPid:@"111" Age:19 Name:@"xzh"];
    Person *p2 = [[Person alloc] initWithPid:@"111" Age:19 Name:@"xzh"];
    
    BOOL ret = [p1 isEqual:p2];
//断点 
}
```

断点后输出

```
(Person *) p1 = 0x00007ff66863bca0
(Person *) p2 = 0x00007ff66863bd10
(BOOL) ret = YES
```

OK，返回YES达到了需要的两个对象的等价判断。


### 如果两个对象等价，那么这两个对象的hash属性值就一定相等

- (1) hash属性的getter方法的实现方式一

```objc
#import <Foundation/Foundation.h>

@interface Person : NSObject

@property (nonatomic, copy) NSString *pid;
@property (nonatomic, assign) NSInteger age;
@property (nonatomic, copy) NSString *name;

- (instancetype)initWithPid:(NSString *)pid Age:(NSInteger)age Name:(NSString *)name;

@end

- (NSUInteger)hash {
    return 1;
}
```

在上面实现hash属性值getter方法后，使用NSMutableSet来测试一下:

```objc
- (void)test {
    Person *p1 = [[Person alloc] initWithPid:@"111" Age:19 Name:@"xzh"];
    Person *p2 = [[Person alloc] initWithPid:@"111" Age:19 Name:@"xzh"];
    Person *p3 = [[Person alloc] initWithPid:@"111" Age:19 Name:@"xzh"];
    
    NSMutableSet *set = [NSMutableSet set];
    [set addObject:p1];
    [set addObject:p2];
    [set addObject:p3];
//断点输出
}
```

输出信息

```
(Person *) p1 = 0x00007f8e5bdcb4b0
(Person *) p2 = 0x00007f8e5bdcb520
(Person *) p3 = 0x00007f8e5bdcb540
(__NSSetM *) set = 0x00007f8e5bdcb560 1 object
```



这样实现hash属性的getter方法确实也是可以的，但是这样的话不管Person类有多少个不同的对象，但是hash只都是一致的，就造成`NSSet`类的对象，只能加入一个唯一的Person类对象。


- (2) hash属性的getter方法的实现方式二、比较推荐的方式

```objc
- (NSUInteger)hash {
    NSInteger ageHash = _age;
    NSUInteger nameHash = [_name hash];
    NSUInteger pidHash = [_pid hash];
	
	//如果有自定义类对象属性，需要再实现那个类对象的hash值方法
	//NSUInteger dogHash = [_dog hash];
		    
    return ageHash ^ nameHash ^ pidHash;
    //return ageHash ^ nameHash ^ pidHash ^ dogHash;
}
```

在hash算法中，有一个名词: `碰撞 collision`。

###给`Person类`添加一个类似NSString的`isEqualToString:`这样的方法，来提示方法调用者是比较这一类所有的对象等同性

```objc
@implementation Person

- (instancetype)initWithPid:(NSString *)pid Age:(NSInteger)age Name:(NSString *)name {
    self = [super init];
    if (self) {
        _pid = [pid copy];
        _name = [name copy];
        _age = age;
    }
    return self;
}

- (BOOL)isEqualToPerson:(Person *)person {
    
    //1. 如果这是两个相同对象，直接就等价了
    if (self == person) return YES;
    
    //2. 依次比较对象内部的子数据项
    if (_age != person.age) return NO;
    if (![_name isEqualToString:person.name]) return NO;
    if (![_pid isEqualToString:person.pid]) return NO;
    
    //3. 默认返回YES，表示对象等价
    return YES;
}

- (BOOL)isEqual:(id)object {
    
    //1. 比较对象的类型是否一致
    if ([self class] == [object class]) {
        //2.2 其他对象指针比较、对象子数据项比较交给isEqualToPerson:
        return [self isEqualToPerson:(Person *)object];
    } else {
        //2.1 不是同一个类型的对象
        return [super isEqual:object];
    }
}

- (NSUInteger)hash {
    NSInteger ageHash = _age;
    NSUInteger nameHash = [_name hash];
    NSUInteger pidHash = [_pid hash];
    
    return ageHash ^ nameHash ^ pidHash;
}

@end
```

### 注意，如上通过`[self class] == [object class]`来判断两个对象的类型是否一致，这种绝对不能对`类簇类`的对象进行比较，因为类簇类对象的类型并不是当前类簇类的类型，而是被屏蔽的内部私有类型。

如下代码比较是错误的，而且永远也不可能成立:

```objc
- (void)test {
    NSArray *array = [NSArray arrayWithObjects:@"1", @"2", @"3"];
    Class cls1 = [array class];
    Class cls2 = [NSArray class];
    BOOL ret = cls1 == cls2;
    
}
```

断点输出如下

```
(__NSArrayI *) array = 0x00007ffde3c38910 @"3 objects"
(Class) cls1 = __NSArrayI
(Class) cls2 = NSArray
(BOOL) ret = NO
```

可以看到创建出来的Array对象真正的类型是`__NSArrayI`，并不是`NSArray`，所以这样对NSArray对象进行比较永远都是不相等的。

这其实就是类簇模式在作怪，可以参看之前的关于类簇模式文章记录，这里只是稍微注意下有这么个问题。

## 等同性判断的执行深度，针对两个Array对象进行等同性比较

比如，对于两个数组对象进行等同性判断的过程是如下:

- (1) 首先比较两个数组对象的长度是否一致
- (2) 如果长度一致，继续挨个对数组相同位置的对象分别进行等同性判断，如果全部位置的对象都是等同的，那就说明这两个数组对象是等同
- (3) 如果长度都不一致，直接就是两个不等同的数组对象

那么，从如上两个数组等同性判断过程来看，是分两个方面进行判断:

- (1) 首先根据对象的某几个字段来判断
- (2) 如果(1)是等价的，那么就转成判断其他所有的字段分别比较判断

如果说，如上的每一个Person对象都对应数据库表中的一条表记录，那就是说每一个Person对象都有一个唯一表示属性值identifier与数据库表中一条唯一记录主键对应，那么在此种情况下对于两个Person对象的比较就应该是如下这样:

- (1) 直接比较两个Person对象的identifier属性值是否一致
- (2) 如果一致，说明是同一条数据库表记录，也就是对象是等同的
- (3) 如果不一致，就是不一样的表记录，不是等同的

## 容器对象内中的可变对象的等同性判断

当一个可变对象被加入到容器对象后，就不应该再去修改这个可变对象的一些属性值了。一旦修改，就可能导致这个可变对象的hash值改变。对于Set这样的容器，是根据对象的hash值来判断，是否加入这个对象的。所以当hash值被修改时，Set就会产生一些错误的问题。

```objc
- (void)test {
    
    NSMutableSet *set = [NSMutableSet new];
    
    NSMutableArray *array1 = [@[@"1", @"2"] mutableCopy];
    [set addObject:array1];
    NSLog(@"set = %@", set);
    
    NSMutableArray *array2 = [@[@"1", @"2"] mutableCopy];
    [set addObject:array2];
    NSLog(@"set = %@", set);
    
    NSMutableArray *array3 = [@[@"1"] mutableCopy];
    [set addObject:array3];
    NSLog(@"set = %@", set);
    
    //【出现错误】此时如果修改Set容器中的可变对象，array3与array1此时等同了
    [array3 addObject:@"2"];
    NSLog(@"set = %@", set);
    
    //拷贝此时的Set对象
    NSSet *copySet = [set copy];
    NSLog(@"copySet = %@", copySet);
}
```

输出结果

```
2016-07-18 00:43:23.764 Demo[25539:403037] set = {(
        (
        1,
        2
    )
)}
2016-07-18 00:43:23.765 Demo[25539:403037] set = {(
        (
        1,
        2
    )
)}
2016-07-18 00:43:23.765 Demo[25539:403037] set = {(
        (
        1
    ),
        (
        1,
        2
    )
)}
2016-07-18 00:43:23.765 Demo[25539:403037] set = {(
        (
        1,
        2
    ),
        (
        1,
        2
    )
)}
2016-07-18 00:43:23.765 Demo[25539:403037] copySet = {(
        (
        1,
        2
    )
)}
```

看到第四个set输出中，存在两个等同的数组对象，这显然和Set对象只能包含一个等同对象的规则矛盾了。这就是因为将一个可变对象加入到Set之后，然后再修改这个可变对象的属性值，造成这个对象的hash值改变之后出现的错误。

第五个copy之后，让错误的Set对象又恢复了正常。

所以，对于集合使用`NSSet`对象时，切记不要对NSSet内部的对象属性值再进行修改，否则会产生错误导致程序逻辑也错误，然后根本都不知道什么原因引起的。

其关键是，对于NSSet容器对象，对于同一个hash值的对象，只能存放一个。

## 给已有的对象动态关联另一个对象

前面知道，对于一个已经存在源文件的Objective-C类，是无法向这个类的对象在运行时添加实例变量Ivar的。

但是可以通过objc AssociatedObject 使用一个key去关联另一个对象，间接的解决了无法给已经存在的类对象添加实例变量Ivar的问题。

### 关联对象的基本可以使用的api

- (1) 设置一个关联对象

```c
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
```

- (2) 获取被关联的对象

```c
id objc_getAssociatedObject(id object, const void *key)
```

- (3) 移除object对象关联的所有对象

```c
void objc_removeAssociatedObjects(id object)
```

可以根据不同的key，来给一个对象关联另一个对象，并且可以指定关联另一个对象时使用什么样的内存管理策略。


##Objective-C中之所以将`[target sel]`成为发送消息是因为最终会编译成这样`objc_msgSend(id target, SEL sel, args)`一段的c代码


对于c语言，是直接通过`函数调用`，那么其实在`编译期间`就决定了调用的是哪一个c函数。











对于，OC中初学者称做方法调用的如下代码:

```objc
id ret = [target test:params];
```

最终会被编译成一段类似如下的c语言代码:


```
id ret = objc_msgSend(target, @selector(test));
id ret = objc_msgSend(target, @selector(test:), params1);
id ret = objc_msgSend(target, @selector(test:and:), params1, params2);
id ret = objc_msgSend(target, @selector(test:andA:andB:), params1, params2, params3);

.....参数可以无限制
```

如其名字，`objc_msgSend()`函数就是专门负责消息发送，那么一个消息包含三部分:

- (1) target、消息发送给谁
- (2) SEL、要执行的方法名（其实会根据这个值查找对应的函数实现IMP）
- (3) params、最终实现函数执行需要的参数

###objc_msgSend(target, @selector(test))做的事情

```objc
@implementation ViewController

//- (NSInteger)haha1:(NSInteger)ha1 haha2:(NSInteger)ha2 haha3:(NSInteger)ha3 {
//    return 0;
//}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self performSelectorOnMainThread:@selector(haha1:haha2:haha3:) withObject:nil waitUntilDone:NO];
}

/**
 *  改方法实现的作用: 描述最终要被调用的c函数的长什么样子
 *  - (const char*)types: 描述c函数的格式（返回值类型、所有的参数类型）
 *  - 这个函数，和函数的名字并没有什么关系
 *  - 这个函数必须返回不为空的methodSignature对象，否则没等走到forwardInvocation:，就直接在这个方法实现中崩溃了
 *  - 所以可以通过返回一个已经存在实现的方法的types组装的NSMethodSignature对象即可，避免崩溃
 */
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    // 会崩溃
//    return nil;
    
    NSMethodSignature *sig = [[NSNull class] instanceMethodSignatureForSelector:aSelector];
    if(sig == nil) {
        sig = [NSMethodSignature signatureWithObjCTypes:"@0@0:8"];// -、+[NSObject self]方法实现的types
    }
    return sig;
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    NSLog(@"anInvocation.target = %@", anInvocation.target);
    NSLog(@"anInvocation.sel = %@", NSStringFromSelector(anInvocation.selector));
    NSLog(@"anInvocation.methodSignature.numberOfArguments = %lu", anInvocation.methodSignature.numberOfArguments);
    NSLog(@"anInvocation.methodSignature.methodReturnType = %s", anInvocation.methodSignature.methodReturnType);
    for (int i = 0; i < anInvocation.methodSignature.numberOfArguments; i++) {
        NSLog(@"Method参数【%d】的type = %s", i, [anInvocation.methodSignature getArgumentTypeAtIndex:i]);
    }
}
@end
```

输出如下

```
2016-12-16 14:27:59.894 FQLDemo[12344:600311] anInvocation.target = <ViewController: 0x7f8d634bcff0>
2016-12-16 14:27:59.894 FQLDemo[12344:600311] anInvocation.sel = haha1:haha2:haha3:
2016-12-16 14:27:59.894 FQLDemo[12344:600311] anInvocation.methodSignature.numberOfArguments = 2
2016-12-16 14:27:59.894 FQLDemo[12344:600311] anInvocation.methodSignature.methodReturnType = @
2016-12-16 14:27:59.894 FQLDemo[12344:600311] Method参数【0】的type = @
```

可以看到，当执行`[target sel:args]`之后，其实系统会创建出一个类似`NSInvocation的消息对象`，然后里面又包含一个`NSMethodSignature对象`用来描述最终被调用的c函数具体长什么样。

###NSNull Crash 避免

```objc
@implementation NSNull (nullExtention)

- (void)forwardInvocation:(NSInvocation *)invocation {
	// 判断是否实现了selector，如果没实现，就不让消息继续执行
	if ([self respondsToSelector:[invocation selector]]) {
		[invocation invokeWithTarget:self];
	}
}

- (NSMethodSignature *)methodSignatureForSelector:  (SEL)selector {
	NSMethodSignature *sig = [[NSNull class] instanceMethodSignatureForSelector:selector];
	if(sig == nil) {	
		// 返回存在系统默认实现的函数的types避免程序直接在这个程序崩溃
		sig = [NSMethodSignature signatureWithObjCTypes:"@^v^c"];	}
	return sig;
}
@end
```

###系统接收到消息后从target的Class中查询SEL对应Method过程

- (1) 将组装好的消息发送给target（对象或类）

- (2) 然后找到接收到消息的target对应的`objc_class`体实例，从中查询消息中SEL对应的`objc_method`实例
	- 对象方法，查询类本身的`objc_class`体实例
	- 类方法、查询Meta元类的`objc_class`体实例

- (3) 查询`objc_method`实例的过程又分为
	- (3.1) 先从`method_cache`中查询SEL是否存在对应的`objc_method`实例
		- (3.1.1) 如果有，直接使用找到的`objc_method`实例，流程进入(4)
		- (3.1.2) 如果没有，进入(3.2)
	- (3.2) 获取找到的`objc_class`结构体实例的`method_list`方法列表
	- (3.3) 然后遍历找到的`method_list`中的所有`objc_method`实例，对比其SEL值是否和消息中的SEL值一致
		- (3.3.1) 如果一致，表示找到了消息SEL对应个的函数实现`objc_method`实例，然后使用消息SEL作为key，找到的`objc_method`实例作为value缓存起来
		- (3.3.2) 如果都没有找到，进入父类的`objc_class`体实例中查找，流程走到(2)

- (4) 找到了消息SEL对应的`objc_method`实例了
	- 通过`objc_method实例->IMP`找到存放最终要调用的函数实现的寻访地址
	- 继而载入找到的函数地址，那么就执行了函数调用

###还有三个和`objc_msgSend()`类似的函数

- (1) `objc_msgSend_stret(id self, SEL op, ...)` 

函数名可以拆开理解记忆为 objc msgSend struct return 。

如果发送消息等待函数执行完毕返回的是一个`c结构体实例`情况时，就需要使用`objc_msgSend_stret()`来发送消息了。


- (2) `void objc_msgSend_fpret(id self, SEL op, ...)`

函数名可以拆开理解记忆为 objc msgSend floating pointer 。

也就是说，如果消息发送后等待函数执行完毕返回值是一个float类型的值，就需要使用这个函数进行消息发送。

- (3) `objc_msgSendSuper(struct objc_super *super, SEL op, ...)`

对于`objc_msgSend()`、`objc_msgSend_fpret()`、`objc_msgSend_fpret()`都是将消息发送给`当前对象/当前类`。

那么明显`objc_msgSendSuper()`是将消息发送给: 当前对象的父对象、当前类的父亲类。

同样也和子对象/子类消息发送一样，也有关于返回值为结构体实例和float值的区分。


###我们写的每一个objc函数:

-  1) 编译器: 编译生成一个具体的`C函数` ，并生成一个`objc_method`结构体实例，并指向`C函数`
-  2) 运行时: 创建一个`objc_method`实例在内存中，方便找到 1) 生成的c函数

我们经常写的OC函数格式如下，并且OC中分为对象方法和类方法

```
对象方法
- (返回值类型)helloWord:(NSString *)param {
	//......实现细节......
	return 返回值;
}

类方法
+ (返回值类型)helloWord:(NSString *)param {
	//......实现细节......
	return 返回值;
}
```

那么回到c语言中，好像并没有什么`对象方法/类方法`这说，因为压根就没`类/对象`这一说，只有`struct`结构体。

实际上我们编写在.h与.m中的每个NSObject类在`运行时`都对应生成一个`objc_class`实例保存，但是分为两个版本:

- (1) 对象方法，保存在当前NSObject对应的`objc_class`实例
- (2) 类方法，保存在当前NSObject的`Meta元类`对应的`objc_class`实例

简单的说，最终所有的OC函数都是一个`objc_method`实例，然后又全部保存到一个`objc_class`实例中，只是分为不同版本的`objc_class`实例。

```
objc_class
	- objc_method
		- 最终调用的c函数
```

那么说只是不同版本的`objc_class`实例，但是还是一样的`objc_method`实例，所以对于一个OC函数不管其是`对象方法`还是`类方法`，最终由编译器生成的c函数都是如下类似格式，其实就是一个普通的c函数:

```c
//真正的函数名可能不太一样，要看苹果具体编译生成c函数的规则了

<return_type>  Class_selector(id self, SEL _cmd, ...) {
	......
}
```

一个`OC函数`最终其实是通过`type encoding`（一串有规律的字符串，类型编码）与最终`编译生成的 C函数`对应起来的。

```
苹果api提供的每一种`数据类型`都有自己唯一的系统编码，称之为`type encoding`。
```

常见的类型编码如下:

| 系统认识的数据类型 | 开发者使用的数据类型 | 
|:-------------:|:-------------:| 
| c | char | 
| i | int | 
| s | short |
| l | long (注意: long is treated as a 32-bit quantity on 64-bit programs) |
| q | long long  |
| C | unsigned char |
| I | unsigned int |
| S | unsigned short |
| L | unsigned long |
| Q | unsigned long long |
| f | float |
| d | double |
| B | C++ bool or a C99 _Bool |
| v （小写） | void |
| * | character string (char *) |
| @ | 自定义类型的对象，NSObject * |
| # | class object (Class) |
| : | A method selector (SEL) |
| [array type] | array数组 |
| {name=type...} |  structure 结构体 |
| (name=type...) | union 共用体 |
| bnum | A bit field of num bits |
| ^type | A `pointer` to type |
| ? | An unknown type (among other things, this code is used for `function pointers 函数指针`) |
| ^f | float * |
| ^v | void * |
| ^@ | NSError ** |
| [5i] | int[] |
| [3f] | float[] |

注意，苹果是不支持`long double`类型的，即`@encode(long double)仍然返回的是 d`。

对于一个OC函数，苹果对其进行了类型编码，主要是如下几部分编码:

- (1) 返回值类型的编码
- (2) 方法参数类型的编码
- (3) 方法修饰类型的编码

所以也就是说其实`方法的类型编码`就是建立在`数据类型的编码`的基础之上。

那么对我们经常写一个Objective-C函数，它的结构是这样分配的:

![](http://i2.piimg.com/4851/362eca5744e117bc.png)

通过这样对数据类型编码，也正是为了给一个OC函数进行编码做的铺垫。那么给一个OC函数做编码的好处是，能够快速找到对应的c函数实现，而不必进行全局的遍历查找。

###NSMethodSignature 方法签名，正是用来打包上面的所有的types

用来描述一个`objc_method`最终对应的c函数的所有的信息，其类结构如下

```objc
@interface NSMethodSignature : NSObject {
@private
    void *_private;
    void *_reserved[6];
}

// 传入c函数最终的整体编码，构造一个方法签名
+ (nullable NSMethodSignature *)signatureWithObjCTypes:(const char *)types;

//1. 方法的形参总数
@property (readonly) NSUInteger numberOfArguments;

//2. 获取每一个形参的数据类型
- (const char *)getArgumentTypeAtIndex:(NSUInteger)idx NS_RETURNS_INNER_POINTER;

//3. 暂时不知道是干嘛的
@property (readonly) NSUInteger frameLength;

//4. 暂时不知道是干嘛的
- (BOOL)isOneway;

//5. 方法返回值的数据类型的type encoding字符串
@property (readonly) const char *methodReturnType NS_RETURNS_INNER_POINTER;

//6. 返回值的长度。如果大于0，表示非void类型，否则就是void类型
@property (readonly) NSUInteger methodReturnLength;

@end
```

那么可以看到NSMethodSignature包含如下信息

- (1) 方法有多少个参数、以及每一个参数的类型
- (2) 方法的返回值类型

完全可以描述一个最终编译生成的c函数。


##消息转发、当发送`对象`消息`objc_msgSend(对象,sel,params)`时，并且接受到`一个不存在实现`的消息时就会启动消息转发

###对于消息转发分为两个阶段的图示:

![](http://i11.tietuku.cn/826855e2817fe7c0.jpg)


###阶段一、resolve object/class method

询问接受消息者，是否继续往下处理这个在接收者对应的`objc_class->method_list`中查询不存在`objc_method`实现的`消息中指定的SEL`

- 返回YES，表明继续往下处理消息，并结束消息转发过程，但会`导致程序崩溃`
- 返回NO，进入 `消息转发阶段二`

分为 `对象/类` 两种方法类型的解析，注意这两个方法都是`类方法`

```objc
@protocol NSObject

// 解析类方法
+ (BOOL)resolveClassMethod:(SEL)sel __OSX_AVAILABLE_STARTING(__MAC_10_5, __IPHONE_2_0);

@end
```

```objc
@protocol NSObject

//解析对象方法
+ (BOOL)resolveInstanceMethod:(SEL)sel __OSX_AVAILABLE_STARTING(__MAC_10_5, __IPHONE_2_0);

@end
```

###阶段二、forward target/invocation

称为完整的消息转发阶段，又分为两个小阶段

- 小阶段一、forward to target
	- 首先询问消息接收者，返回一个能够解读当前消息SEL的Target对象
	- 返回 `非空对象`，重新将消息发送给返回的 非空对象 解读消息，结束消息转发过程
	- 返回 `空对象(nil)`，进入到 `小阶段二`

```objc
@protocol NSObject

- (id)forwardingTargetForSelector:(SEL)aSelector __OSX_AVAILABLE_STARTING(__MAC_10_5, __IPHONE_2_0);

@end
```

- 小阶段二、forward invocation
	- 自己手动构造`NSInvocation`OC消息对象了，指定:
		- (1) 消息接收者
		- (2) 消息SEL
		- (3) 目标函数实现需要的参数

完整的消息转发，需要手动构造NSInvocation消息

```objc
@protocol NSObject

//第一步、获取SEL对应的Method的签名（方法参数列表、返回值列表 ... ）
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector OBJC_SWIFT_UNAVAILABLE("");

//第二步、将封装好的消息NSInvocation对象，交给Target处理
- (void)forwardInvocation:(NSInvocation *)anInvocation OBJC_SWIFT_UNAVAILABLE("");

@end
```

###消息转发阶段一的一些基础使用

现在在一个ViewController对象发送一个不存在实现的sel消息


```objc
@implementation RuntimeController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self demo4];
}

- (void)demo4 {
    // 进入对象方法的消息转发
    [self performSelector:@selector(haha1)];
}

- (void)demo5 {
    // 进入类方法的消息转发
    [[self class] performSelector:@selector(haha2)];
}

@end
```

执行一个不存在实现的对象方法，然后流程会进入到`resolveInstanceMethod:`实现中，上述代码肯定是会程序崩溃的，因为最终都没有一个消息接收处理者。

如果只是简单的重写`resolveInstanceMethod: 返回YES或NO`，程序都会崩溃运行的，必须要告诉系统这个消息到底由谁来接收处理，而不用进入消息转发阶段二。

```objc
@implementation RuntimeController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self demo4];
}

- (void)demo4 {
    // 进入对象方法的消息转发
    [self performSelector:@selector(haha1)];
}

- (void)demo5 {
    // 进入类方法的消息转发
        [[self class] performSelector:@selector(haha2)];
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    return YES;//告诉系统消息可以处理，但实际上并没有做处理，所以崩溃
    return NO;//告诉系统进入消息转发阶段二，但是没有重写阶段二的转发消息函数，所以一样崩溃
}

@end
```

消息转发阶段一中如果要处理，就必须预先在写好要添加的函数实现，然后再运行时添加到`objc_class`实例中去

```objc
// 必须预先在写好要添加的函数实现
static void dynamicMethodIMP(id self, SEL _cmd)
{
    printf("SEL %s 不存在对应的实现 \n",sel_getName(_cmd));
}

@implementation RuntimeController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self demo4];
}

- (void)demo4 {
    // 进入对象方法的消息转发
    [self performSelector:@selector(haha1)];
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    
    // 针对一些特定的sel做防止崩溃处理，也可以做类似重定向
    if (sel == @selector(haha1)) {
        class_addMethod([self class], sel, (IMP)dynamicMethodIMP, "v@:");
        return YES;
    }
    
    return [super resolveInstanceMethod:sel];
}

@end
```

这样以后，对于SEL是`haha1`的消息处理就不会再崩溃了。

如果，对所有的不能够处理的消息都做处理就讲上面的`resolveInstanceMethod:`改成如下

```objc
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    
    class_addMethod([self class], sel, (IMP)dynamicMethodIMP, "v@:");
    return YES;
}
```

改成上面以后，我发现打出的log如下:

```
SEL _shouldApplyExclusiveTouch 不存在对应的实现 
SEL _isInExclusiveTouchSubviewTree 不存在对应的实现 
SEL haha1 不存在对应的实现 
```

处理自己的`haha1`之外，还有两个其他的，暂时不知道为什么没有实现，也不知道是什么时候发送的消息，后面再看吧。


还有一个消息转发阶段一的应用:

就是`NSManagedObject`的属性全局声明为`@dynamc`，不由编译器生成getter与setter，而是在运行时使用消息转发机制将自己实现的函数添加到`objc_class`实例中去。

###消息转发阶段二、其中的子阶段一的使用 >>> `forwardingTargetForSelector:`

简单的例子是使用一个预先写好的类

```objc
@interface FIXObject : NSObject
- (void)haha1;
+ (void)haha1;
@end

@implementation FIXObject
- (void)haha1{NSLog(@"对象方法haha1执行");}
+ (void)haha1{NSLog(@"类方法haha1执行");}
@end
```

然后在ViewCOntroller的消息转发函数中返回上面类的一个对象

```objc
@implementation RuntimeController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self demo4];
}

- (void)demo4 {
    // 进入对象方法的消息转发
    [self performSelector:@selector(haha1)];
}

- (id)forwardingTargetForSelector:(SEL)aSelector {
    if (aSelector == @selector(haha1)) {
        return [FIXObject new];
    }
    return [super forwardingTargetForSelector:aSelector];
}

@end
```

就是将对象`haha1`这个sel的消息重定向交给`FIXObject`的一个对象去处理。


###组合一些不同类型的对象，来实现一个类的`多继承`的效果

- (1) 首先是三个类型的抽象接口

```objc
@protocol SayChinese <NSObject>
- (void)sayChinese;
@end

@protocol SayEnglish <NSObject>
- (void)sayEnglish;
@end

@protocol SayFranch <NSObject>
- (void)sayFranch;
@end
```

- (2) 三个类型抽象接口的各自对应的实现

```objc
@interface SayChineseImpl : NSObject <SayChinese>
@end

@implementation SayChineseImpl
- (void)sayChinese {
    NSLog(@"说中国话");
}
@end

@interface SayEnglishImpl : NSObject <SayEnglish>
@end

@implementation SayEnglishImpl
- (void)sayEnglish {
    NSLog(@"说英语话");
}
@end


@interface SayFranchImpl : NSObject <SayFranch>
@end

@implementation SayFranchImpl
- (void)sayFranch {
    NSLog(@"说法国话");
}
@end
```

- (3) 只暴露一个类，来统一处理上面三种抽象类型接口的调用，模拟多继承的效果

```objc
/**
 *  模拟继承多种语言
 */
@interface SpeckManager : NSObject <SayChinese, SayEnglish, SayFranch>

@end

@implementation SpeckManager {
    id<SayChinese> _sayC;
    id<SayEnglish> _sayE;
    id<SayFranch> _sayF;
}

- (instancetype)init
{
    self = [super init];
    if (self) {
        _sayC = [SayChineseImpl new];
        _sayE = [SayEnglishImpl new];
        _sayF = [SayFranchImpl new];
    }
    return self;
}

- (id)forwardingTargetForSelector:(SEL)aSelector {
    
    if (aSelector == @selector(sayChinese)) {
        return _sayC;
    }
    
    if (aSelector == @selector(sayEnglish)) {
        return _sayE;
    }
    
    if (aSelector == @selector(sayFranch)) {
        return _sayF;
    }
    
    return [super forwardingTargetForSelector:aSelector];
}

@end
```

- (4) ViewController对象中测试

```objc
@implementation RuntimeController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self demo4];
}

- (void)demo4 {
    SpeckManager *speak = [SpeckManager new];
    [speak sayChinese];
    [speak sayEnglish];
    [speak sayFranch];
}

@end
```

类对象的组成结构:

```
- SpeckManager
	- SayChinese
	- SayEnglish
	- SayFranch
```

所有的消息统一发送给SpeckManager对象，然后SpeckManager对象内部并没有任何的函实现，从而进入消息转发阶段。然后将消息分别转发给内部持有的SayChinese对象、SayEnglish对象、SayFranch对象去处理，从而就模拟实现了多继承的效果。

###还记录过一种NSProxy然后再加上如上方法实现屏蔽内部版本实现类，其实大体上都差不多，主要是在消息转发阶段，将消息转发给其他能够处理消息的对象


- (1) 首先接口抽象


```objc
@protocol GetData  <NSObject>
- (NSString *)getData1;
- (NSString *)getData2;
- (NSString *)getData3;
@end
```

- (2) 不同版本的实现类

```objc
@interface __GetDataProxy_fromServerA : NSObject <GetData>
@end
@implementation __GetDataProxy_fromServerA

- (NSString *)getData1 {
    return @"使用server A 1 的数据";
}

- (NSString *)getData2 {
    return @"使用server A 2 的数据";
}

- (NSString *)getData3 {
    return @"使用server A 3 的数据";
}

@end
```

```objc
@interface __GetDataProxy_fromServerB : NSObject <GetData>
@end
@implementation __GetDataProxy_fromServerB

- (NSString *)getData1 {
    return @"使用server B 1 的数据";
}

- (NSString *)getData2 {
    return @"使用server B 2 的数据";
}

- (NSString *)getData3 {
    return @"使用server B 3 的数据";
}

@end
```

- (3) 使用一个plist配置工程中使用的实现类

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>ImplementationClass</key>
	<string>__GetDataProxy_fromServerB</string>
</dict>
</plist>
```

- (4) GetDataProxy获取单例时读取plist配置的实现类进行与协议protocol绑定

```objc
@implementation GetDataProxy {
    NSMutableDictionary *_handlerDic;
}

+ (instancetype)sharedInstance {
    static GetDataProxy *_proxy = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _proxy = [GetDataProxy new];
    });
    return _proxy;
}

/**
 *  因为是在dispatch_once()块中执行的，所以肯定是线程安全的
 *  但如果不是在dispatch_once()中执行，需要同步多线程进行操作 _handlerDic 缓存字典对象
 */
- (instancetype)init
{
    self = [super init];
    if (self) {
        _handlerDic = [NSMutableDictionary new];
        
        // 获取配置的实现类
        NSString *file = [[NSBundle mainBundle] pathForResource:@"GetSystemDataProxy" ofType:@"plist"];
        NSDictionary *configDic = [NSDictionary dictionaryWithContentsOfFile:file];
        
        // 实例化配置的实现类对象
        NSString *clsName = configDic[@"ImplementationClass"];
        Class cls = NSClassFromString(clsName);
        id<GetData> getData = [[cls alloc] init];
        
        // 绑定得到对象作为协议实现类对象
        [self _registerHandlerProtocol:@protocol(GetData) handler:getData];
    }
    return self;
}

/**
 *  绑定协议中所有定义方法对应的实现类对象
 *
 *  @param protocol 协议
 *  @param handler  协议中每一个方法发送的消息处理对象
 */
- (void)_registerHandlerProtocol:(Protocol *)protocol handler:(id)handler {
    
    //记录protocol中的方法个数
    unsigned int numberOfMethods = 0;
    
    //获取protocol中定义的方法描述
    struct objc_method_description *methods = protocol_copyMethodDescriptionList(protocol,
                                                                                 YES,//是否必须实现
                                                                                 YES,//是否对象方法
                                                                                 &numberOfMethods);
    
    //遍历所有的方法描述，设置其Target对象
    for (unsigned int i = 0; i < numberOfMethods; i++) {
        
        //方法描述结构体实例
        struct objc_method_description method = methods[i];
        
        //方法的名字
        NSString *methodNameStr = NSStringFromSelector(method.name);
        
        //保存所有方法名对应的Target对象，最终接收处理消息
        [_handlerDic setValue:handler forKey:methodNameStr];
    }
}

- (id)forwardingTargetForSelector:(SEL)aSelector {
    
    NSString *selString = NSStringFromSelector(aSelector);
    id<GetData> getData = [_handlerDic objectForKey:selString];
    if (getData) {
        return getData;
    }
    
    return [super forwardingTargetForSelector:aSelector];
}

@end
```

- (5) ViewController中测试

```objc
@implementation RuntimeController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self demo6];
}

- (void)demo6 {
    NSLog(@"data 1 = %@", [[GetDataProxy sharedInstance] getData1]);
    NSLog(@"data 2 = %@", [[GetDataProxy sharedInstance] getData2]);
    NSLog(@"data 3 = %@", [[GetDataProxy sharedInstance] getData3]);
}

@end
```

输出log

```
2016-07-23 23:46:25.396 Demos[18839:229499] data 1 = 使用server B 1 的数据
2016-07-23 23:46:25.396 Demos[18839:229499] data 2 = 使用server B 2 的数据
2016-07-23 23:46:25.396 Demos[18839:229499] data 3 = 使用server B 3 的数据
```

- (6) 修改plist配置的实现类，再次运行程序

```objc
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>ImplementationClass</key>
	<string>__GetDataProxy_fromServerA</string>
</dict>
</plist>
```

修改完plist配置文件之后直接运行程序即可，无序修改Proxy内任何代码和ViewController外部调用代码

```
2016-07-23 23:48:15.427 Demos[19121:236373] data 1 = 使用server A 1 的数据
2016-07-23 23:48:15.427 Demos[19121:236373] data 2 = 使用server A 2 的数据
2016-07-23 23:48:15.427 Demos[19121:236373] data 3 = 使用server A 3 的数据
```

###消息转发阶段二、其中的子阶段二的使用 >>> `forwardInvocation:` + `methodSignatureForSelector:`

仍然沿用上面的plist配置实现类的例子，改为在消息转发阶段二中的子阶段二进行消息重新转发，只是为了记录学习使用。但是没必要到这个阶段才进行消息转发，其实在上一个子阶段会更好。

如果是需要拿到目标方法的一些执行参数，就可以延迟到这个阶段，通过传入的NSInvocation对象，获取到所有的参数之后再进行消息转发。

> 那么在此阶段，主要是操作该方法中由系统传入的NSInvocation消息对象:

- (1) 改变消息接收者对象
- (2) 修改目标函数实现执行的参数
- (3) 获取目标函数实现执行完毕后的函数返回值，可以再次进行修改

那么针对上面的例子，稍加修改为如下:

```objc
@implementation GetDataProxy {
    NSMutableDictionary *_handlerDic;
}

+ (instancetype)sharedInstance {
    static GetDataProxy *_proxy = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _proxy = [GetDataProxy new];
    });
    return _proxy;
}

/**
 *  因为是在dispatch_once()块中执行的，所以肯定是线程安全的
 *  但如果不是在dispatch_once()中执行，需要同步多线程进行操作 _handlerDic 缓存字典对象
 */
- (instancetype)init
{
    self = [super init];
    if (self) {
        _handlerDic = [NSMutableDictionary new];
        
        // 获取配置的实现类
        NSString *file = [[NSBundle mainBundle] pathForResource:@"GetSystemDataProxy" ofType:@"plist"];
        NSDictionary *configDic = [NSDictionary dictionaryWithContentsOfFile:file];
        
        // 实例化配置的实现类对象
        NSString *clsName = configDic[@"ImplementationClass"];
        Class cls = NSClassFromString(clsName);
        id<GetData> getData = [[cls alloc] init];
        
        // 绑定得到对象作为协议实现类对象
        [self _registerHandlerProtocol:@protocol(GetData) handler:getData];
    }
    return self;
}

/**
 *  绑定协议中所有定义方法对应的实现类对象
 *
 *  @param protocol 协议
 *  @param handler  协议中每一个方法发送的消息处理对象
 */
- (void)_registerHandlerProtocol:(Protocol *)protocol handler:(id)handler {
    
    //记录protocol中的方法个数
    unsigned int numberOfMethods = 0;
    
    //获取protocol中定义的方法描述
    struct objc_method_description *methods = protocol_copyMethodDescriptionList(protocol,
                                                                                 YES,//是否必须实现
                                                                                 YES,//是否对象方法
                                                                                 &numberOfMethods);
    
    //遍历所有的方法描述，设置其Target对象
    for (unsigned int i = 0; i < numberOfMethods; i++) {
        
        //方法描述结构体实例
        struct objc_method_description method = methods[i];
        
        //方法的名字
        NSString *methodNameStr = NSStringFromSelector(method.name);
        
        //保存所有方法名对应的Target对象，最终接收处理消息
        [_handlerDic setValue:handler forKey:methodNameStr];
    }
}

//- (id)forwardingTargetForSelector:(SEL)aSelector {
//    
//    NSString *selString = NSStringFromSelector(aSelector);
//    id<GetData> getData = [_handlerDic objectForKey:selString];
//    if (getData) {
//        return getData;
//    }
//    
//    return [super forwardingTargetForSelector:aSelector];
//}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    
    NSString *methodNameStr = NSStringFromSelector(aSelector);
    
    id target = [_handlerDic objectForKey:methodNameStr];
    
    if (target && [target respondsToSelector:aSelector]) {
        return [target methodSignatureForSelector:aSelector];
    } else {
        return [super methodSignatureForSelector:aSelector];
    }
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {

    SEL sel = anInvocation.selector;
    NSString *methodNameStr = NSStringFromSelector(sel);
    
    id target = [_handlerDic objectForKey:methodNameStr];
    
    if (target && [target respondsToSelector:sel]) {
        [anInvocation invokeWithTarget:target];
    } else {
        [super forwardInvocation:anInvocation];
    }

}

@end
```

进入子阶段二进行消息转发时，此时就需要同时实现两个函数:

- (1) `methodSignatureForSelector:`实现，告诉系统稍后消息sel对应的具体实现函数的信息
	- 方法的返回值类型
	- 方法参数列表个数、每一个参数的类型
	- 等等信息

- (2) `forwardInvocation:`实现，告诉系统消息将会由哪一个新的对象来接收并处理


###最后还有一个函数实现`doesNotRecognizeSelector:`干什么的了？

上面可以看到在重写`forwardingTargetForSelector:`、`methodSignatureForSelector:`、`forwardInvocation:`这三个方法实现时，分支都是分为两部分:

- (1) 如果sel命中我们缓存，那么就我们自己来处理消息，并结束消息转发过程

- (2) 如果sel没有命中缓存，也就是不是我们代码定义好的sel，那么此时应该交给`super父类方法的实现`去做处理


然而对于一个`非Root`类，如果无法处理消息，就往`super父类`的转发逻辑中走，一直走到`Root类 NSObject类`来处理消息，而如果最终`NSObject类`也无法处理这个消息，那么:

- (1) 调用 `-[NSObject doesNotRecognizeSelector:]`函数实现

- (2) 而在`-[NSObject doesNotRecognizeSelector:]`默认实现中做两件事:
	- `结束`掉消息转发过程
	- 抛出`NSInvalidArgumentException异常`导致程序崩溃退出

在一般情况下，`doesNotRecognizeSelector:`实现都是由运行时系统去调用，开发者不要去调用这个函数，因为这个函数实现中会抛出异常，导致程序崩溃退出。

但是，有一些情况恰好就可以利用这个特性。比如，提醒子类必须要重写父类的某一些函数，否则让程序崩溃退出执行，提醒开发者要实现完整:

- (1) 抽象父类声明抽象函数（C++中的虚函数）

```objc
#import <Foundation/Foundation.h>

@interface Cat : NSObject

/**
 *  类型抽象虚函数，需要子类完成具体实现
 */
- (void)abstractMethod;

@end

@implementation Cat

- (void)abstractMethod {
  
    //方法一、使用NSObject基类提供的停止消息转发并崩溃
    [self doesNotRecognizeSelector:_cmd];
    
    //方法二、直接抛出一个异常
//    @throw [NSException exceptionWithName:@"not implement method" reason:@"必须要重写这个函数" userInfo:nil];
}

@end
```

- (2) 子类如果没有重新实现`abstractMethod`函数就会在运行时崩溃退出

```objc
#import <Foundation/Foundation.h>
#import "Cat.h"

@interface CatChild : Cat

@end

@implementation CatChild
@end
```

测试程序

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self demo7];
}

- (void)demo7 {
    CatChild *c = [CatChild new];
    [c abstractMethod];
}
```

运行后会崩溃


##拦截/修改一个Objective-C函数实现

###法一、创建一个要拦截方法实现目标类的一个`子类`，然后重写要拦截实现的函数

想要拦截父类的run函数实现

```objc
#import <Foundation/Foundation.h>

@interface Cat : NSObject
- (void)run;
@end

@implementation Cat

- (void)run {
    NSLog(@"Cat run");
}

@end
```

创建一个Cat子类，重写run函数实现完成拦截

```objc
#import <Foundation/Foundation.h>
#import "Cat.h"

@interface CatChild : Cat
@end

@implementation CatChild

- (void)run {
    
    //1.
    NSLog(@"cat run previous");
    
    //2.
    [super run];
    
    //3.
    NSLog(@"cat run after");
}

@end
```


###法二、使用`class_addMethod()、class_replaceMethod()、method_exchangeImplementations()` 交换两个`objc_method`实例的SEL指向

在没有交换SEL指向的情况，一个`objc_class`实例（NSObject类）所有的方法实现都对应一个`objc_method`实例，并由原始的SEL指向

![](http://i4.piimg.com/4851/b2c7f36ae9647316.png)

那么一旦修改了一个SEL指向原来的`objc_method`实例变成下面这样

![](http://i4.piimg.com/4851/7cf5d277435df4d2.png)

###法三、强制进入系统的消息转发函数实现`-[NSObject forwardInvocation:]`（JSPatch就是这么做的）

举个简单的例子，比如使用JSPatch来交换一个ViewController对象的viewDidLoad方法实现的过程:

- (1) 首先是三个`objc_method`实例和一个`JPForwardInvocation()`c函数实现

```c
1. -[ViewController viewDidLoad]的实现
```

```c
2. -[ViewController forwardInvocation:]的实现
```


```c
3. -[ViewController _JPviewDidLoad:]的实现，这个是使用main.js描述的将要替换成为的Objc方法实现
```

```c
4. static void JPForwardInvocation(__unsafe_unretained id assignSlf, SEL selector, NSInvocation *invocation){ .... } c函数实现
```

- (2) 然后让`@selector(viewDidLoad)`指向`@selector(forwardInvocation:)`的实现IMP

```
1. @selector(viewDidLoad) >>>>> -[NSObject forwardInvocation:]的实现IMP
```

- (3) 然后构造回到`-[ViewController viewDidLoad]`实现的SEL

```c
ORIG + viewDidLoad >>>> ORIGviewDidLoad >>>>> -[ViewController viewDidLoad]实现
```

- (3) 然后让`@selector(forwardInvocation:)`指向`JPForwardInvocation()`c函数实现

```
2. @selector(forwardInvocation:) >>>>> static void JPForwardInvocation(__unsafe_unretained id assignSlf, SEL selector, NSInvocation *invocation){ 
	.... 
}
```

- (4) 最后流程走到`JPForwardInvocation()`c函数实现，然后手动构造NSInvocation并发送消息

```c
>>>SEL:		 	_JPviewDidLoad
>>>Target:		ViewController对象
>>>Arguments:		使用之前走入-[NSObject forwardInvocation:]实现获得invocation，并从中获取得到参数
```

![](http://a2.qpic.cn/psb?/V11ePBui3l2qGa/b5iyvyG6B54djRukpPZNZtqNkNOOTj6w12BQwnfc5Pg!/b/dHEBAAAAAAAA&ek=1&kp=1&pt=0&bo=NQSAAkAGtwMFCGo!&sce=0-12-12&rf=viewer_311)

- (5) 记录下进入`-[ NSObject forwardInvocation:]`实现之后，获取到当前被拦截执行实现函数的所有的参数个数、参数类型

```objc
- (void)forwardInvocation:(NSInvocation *)anInvocation {
	//1. 实现函数的方法签名
	NSMethodSignature *ms = [anInvocation methodSignature];
	
	//2. 获取实现函数参数个数的总数
	NSUInteger numberOfArguments = [ms numberOfArguments];
	
	//3. 获取实现函数返回值类型的编码
	const char *methodReturnType = [ms methodReturnType];
	
	//4. 依次获取每一个参数的类型的编码
	for (int i = 0; i < numberOfArguments; i++) {
		const char *argumentTypeAtIndex = [ms getArgumentTypeAtIndex:i];
		
		// 然后根据每一个type encoding字符串比较得到对于的数据类型
		#define _C_ID       '@'
		#define _C_CLASS    '#'
		#define _C_SEL      ':'
		#define _C_CHR      'c'
		#define _C_UCHR     'C'
		#define _C_SHT      's'
		#define _C_USHT     'S'
		#define _C_INT      'i'
		#define _C_UINT     'I'
		#define _C_LNG      'l'
		#define _C_ULNG     'L'
		#define _C_FLT      'f'
		#define _C_DBL      'd'
		#define _C_BFLD     'b'
		#define _C_VOID     'v'
		#define _C_UNDEF    '?'
		#define _C_PTR      '^'
		#define _C_CHARPTR  '*'
		#define _C_ARY_B    '['
		#define _C_ARY_E    ']'
		#define _C_UNION_B  '('
		#define _C_UNION_E  ')'
		#define _C_STRUCT_B '{'
		#define _C_STRUCT_E '}'
	}
}
```

##一个Objective-C类、一个Objective-C类的对象

Objective-C语言规定一个Objective-C类的对象只能创建在`堆上`，而不能够创建在`栈上`，所以不会看到如下这种创建对象的语法:

```objc
Person person = [Person new]; //错误的
```

而应该是这样

```objc
Person *person = [Person new];
```

也可以使用Objective-C中的`万能类型指针`id

```objc
id person = [Person new];
```

乍一看起来，好像并没有`*`来表示指针，但其实`id`本身就是一个`指针类型`

```objc
// 一个OC类对象 最终运行时的内存载体
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};
```

```objc
typedef struct objc_object *id;
```

所以`id`就是指向`objc_object实例`的指针类型，所以使用`id`声明的变量其实就是一个`指针变量`，就不能再使用`*`了。如果一旦使用`*`，其实就是`**`二重指针的效果了。

还有一个`Class`和上面的`id 指针类型`类似的

```objc
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
```

```objc
typedef struct objc_class *Class;
```

Class其实也是一个`指针类型`，指向的是`objc_class`这一类结构体的实例。

###上述引出了两个c结构体，这两个结构体是very重要的.		

- (1) `objc_object` 

```objc
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};
```

这个c结构体的一个实例，用来在运行时描述一个Objective-C类的`一个对象`的数据结构。

而此结构体实例只有一个成员变量 `Class *isa` 指针变量即`objc_class`的一个实例。

而这个被指向的`objc_class`实例，就是这个`Objective-C对象`所属的`类`，严格来说是所属`Objective-C类对应的 objc_class实例`。

这个结构体类型的实例，统一使用id指针类型变量指向。

- (2) `objc_class`

```objc
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
```

上面既然有`Objective-C类 的一个对象`的数据结构表示，那么一个`Objective-C类`自然也有其对应的数据结构表示，而正是上面的`objc_class`结构体的一个实例。

可以看到这个`objc_class`结构体实例的成员变量包含几个与我们编写的Objetive-C类的共同点:

```
- (1) 类的名字	
- (2) 所有的实例变量 
- (3) 所有的方法实现
- (4) 实现的所有协议
- (5) 父类是谁
```

这个结构体类型的实例，统一使用`Class`指针类型变量指向。

###那么既然有Objetive-C类，然后我们可以通过Objetive-C类创建出对象，那么为何还需要上面这两个结构体了？

- (1) `iOS/MacOSX`平台最底层都是基于`c语言`实现的

- (2) 而我们编程时使用的`Objetive-C`语言，实际上是对`c`的一个封装，并且只能应用于`iOS/MacOSX`平台

- (3) 我们平时都是使用`Objetive-C`语言编写代码，但是最终由Xcode编译器编译生成`c语言`代码

- (4) 而在`c语言`中并没有`类`这一说，只有`结构体/共用体`这样的数据结构来模拟`类`

- (5) 所以结果就是Xcode编译器在`编译期间`，会将我们所编写的所有基于`Objetive-C`语言的代码进行转换:
	- 将编写在一个`Objetive-C`类中的所有`函数实现`、`实例变量`、`实现的协议`统统转成`c语言`的代码

- (6) 最终在`程序运行期间`，会创建出对应`c结构体实例`表达:
	- 一个`Objetive-C类`，分别使用两个`objc_class`实例来表达对象方法、类方法两个版本
	- 一个`Objetive-C类`new出来的对象，使用`objc_object`实例来表达
	- 一个具体的`对象方法实现/类方法实现`，使用`objc_method`实例来表达
	- 一个`实例变量`，使用`objc_ivar`实例来表达

- (7) 而最终编译生成的一个独立的c函数，到底属于对象还是类，且到底属于哪一个对象或哪一个类，正是将上述所有的结构体组织起来即可确定

这个结构体的实例也有一个`Class *isa`指针变量，来指向另外一个`objc_class`结构体实例，那么这个另外的`objc_class`数据结构用来描述谁了？

###用来描述当前Objective-C类的另外一半，MetaClass 元类。

实际上我们通过Xcode去创建一个Objective-C类文件时，只出现我们编写代码这个Objective-C类文件，但其实还有另外一个与之配合的类，就叫做`Meta Class 元类`，那么它能做什么了?

- (1) 所有的`类方法`对应的实现都存放在 meta objective-c class 对应的 `objc_class`实例中

###一个Objective-C类的一个对象、一个Objective-C类、Objective-C类对应的Meta Class 元类 ，这三者之间的isa指向关系图:

![](http://i11.tietuku.cn/e14be286f89eef95.png)

- (1) 一个Objective-C类的`对象`的 isa指针，指向`所属类`

- (2) `Objective-C类`的isa指针，指向其`Meta Class 元类`

###所以我们编写的一个`Objective-C类`实际上包含`两部分`:

- (1) 类本身 
	- 在代码中可以使用的类
	- 用来存放`对象`的数据
		- 所有方法的实现
		- 所有的实例变量

- (2) Meta Class 元类
	-  并非我们直接可以操作的类
	-  存放`类`的所有的`函数实现` 


###小结下在 `objc_class` 结构体中两个非常重要的指针:

- (1) `Class *isa`
	-  object->isa >>>> Class
	-  Class->isa >>> MeatClass

- (2) `Class *super_class` >>> `super_class`指针，决定当前对象继承自哪一个类型

###`+[NSObject class]`返回的`objc_class`实例是单例

- (1) [类本身 class]
- (2) [Meta类 class]

```objc
- (void)test3 {
    
    Person *person1 = [Person new];
    Person *person2 = [Person new];
    Person *person3 = [Person new];
    
    // 通过OC类的对象，获取所属类的objc_class实例
    Class cls1 = [person1 class];
    Class cls2 = [person2 class];
    Class cls3 = [person3 class];
    
    // 等价于上面的代码
    Class cls4 = object_getClass(person1);
    Class cls5 = object_getClass(person1);
    Class cls6 = object_getClass(person1);
    
    // 通过OC类，获取当前类的objc_class实例
    Class cls7 = [Person class];
    
    // 通过OC类，获取当前类对应的Meta元类的objc_class实例
    Class cls8 = object_getClass(cls5);

    NSLog(@"cls1 = %p", cls1);
    NSLog(@"cls2 = %p", cls2);
    NSLog(@"cls3 = %p", cls3);
    NSLog(@"cls4 = %p", cls4);
    NSLog(@"cls5 = %p", cls5);
    NSLog(@"cls6 = %p", cls6);
    NSLog(@"cls7 = %p", cls7);
    NSLog(@"cls8 = %p", cls8);
}
```

输出信息

```
2016-07-24 21:16:36.994 Demos[5407:77553] cls1 = 0x10b0e70e8
2016-07-24 21:16:36.995 Demos[5407:77553] cls2 = 0x10b0e70e8
2016-07-24 21:16:36.995 Demos[5407:77553] cls3 = 0x10b0e70e8
2016-07-24 21:16:36.996 Demos[5407:77553] cls4 = 0x10b0e70e8
2016-07-24 21:16:36.996 Demos[5407:77553] cls5 = 0x10b0e70e8
2016-07-24 21:16:36.996 Demos[5407:77553] cls6 = 0x10b0e70e8
2016-07-24 21:16:36.996 Demos[5407:77553] cls7 = 0x10b0e70e8
2016-07-24 21:16:36.996 Demos[5407:77553] cls8 = 0x10b0e7110
```

可以看到，一个Objetive-C类确实在运行时是有一个与之对应的`objc_class`实例，而且与之对应另外一个`objc_class`实例就是Meta Class 元类。

注意，`<objc/runtime.h>`中并没有提供函数直接获取Meta Class 元类的`objc_class`实例，只有如下关于MetaClass元类的函数:

```objc
BOOL class_isMetaClass(Class cls) 
```

那么获取一个Meta Class 元类的`objc_class`实例的步骤:

- (1) 首先获取到Objetive-C类本身的`objc_class`实例
- (2) 然后通过`objc_class实例->isa`找到所属的Meta Class 元类

比如，判断一个Objetive-C类对象是否实现了某一个方法实现:

```objc
@interface Person : NSObject

- (void)logI;
+ (void)logC;

@end

@implementation Person

- (void)logI {
    NSLog(@"对象方法实现");
}

+ (void)logC {
    NSLog(@"类方法实现");
}

@end
```

对应的查找对象方法实现与类方法实现的测试代码

```objc
- (void)test3 {
    
    Person *person = [Person new];

    // 对象方法实现判断
    BOOL isImpl1 = [person respondsToSelector:@selector(logI)];//实际上是调用如下c函数完成
    BOOL isImpl2 = class_respondsToMethod([person class], @selector(logI));
    
    // 错误判断类方法实现
    BOOL isImpl3 = [person respondsToSelector:@selector(logC)];
    BOOL isImpl4 = class_respondsToMethod([person class], @selector(logC));
    
    // 先获取到MetaClass
    Class metaClass = object_getClass([person class]);
    BOOL isImpl5 = [metaClass respondsToSelector:@selector(logC)];// 错误判断类方法实现
    BOOL isImpl6 = class_respondsToMethod(metaClass, @selector(logC));// 正确判断类方法实现
    
    // 直接通过之前的MetaClass的api获取类方法实现IMP
    IMP imp1 = class_getMethodImplementation([person class], @selector(logC));//错误查找实现
    IMP imp2 = class_getMethodImplementation([Person class], @selector(logC));//错误查找实现
    IMP imp3 = class_getMethodImplementation(metaClass, @selector(logC));//正确查找实现
    
}
```

输出log如下

```
(Person *) person = 0x00007f9b596072e0
(BOOL) isImpl1 = YES
(BOOL) isImpl2 = YES
(BOOL) isImpl3 = NO
(BOOL) isImpl4 = NO
(Class) metaClass = 0x000000010756e0b0
(BOOL) isImpl5 = NO
(BOOL) isImpl6 = YES
(IMP) imp1 = 0x0000000107a8d280 (libobjc.A.dylib`_objc_msgForward)
(IMP) imp2 = 0x0000000107a8d280 (libobjc.A.dylib`_objc_msgForward)
(IMP) imp3 = 0x000000010755ee00 (Demos`+[Person logC] at Person.m:61)
```

看到最后三个关于IMP获取到的log，前两个获取到了`_objc_msgForward()`函数实现，而这个函数实现明显不是我们在Person类中的类方法`logC`的实现，实际上这个函数实现是`系统的消息转发函数的实现`，因为Person类对象并不存在`logC`的实现，所以获取得到就是`系统消息转发的实现`。

看到最后一个IMP获取，就是我们Person类中的类方法`logC`的实现了。

其实看log信息已经可以完全明白怎么使用了，小结下吧:

- (1) `对象`方法实现，查找OC类本身的`objc_class`实例
- (2) `类`方法实现，查找OC类本身的`objc_class`实例的isa指针指向的另外一个`objc_class`实例

在判断`对象是否属于某个类型` 或 判断`两个对象的类型是否一致` 时，用得最多的是`isKindOf:`


```objc
@interface Person : NSObject

- (void)logI;
+ (void)logC;

@end

@interface Man : Person
@end

@implementation Person

- (void)logI {
    NSLog(@"对象方法实现");
}

+ (void)logC {
    NSLog(@"类方法实现");
}

@end

@implementation Man

@end
```

一般判断两个对象类型是否一致使用最多的方式，`isKindOf:` 会沿着传入的Class一直往上查询`super_class`，一直遍历到Root类才会停止（NSObject、NSProxy）.

```objc
- (void)test3 {
    
    Person *person1 = [Person new];
    Person *person2 = [Person new];

    if ([person1 isKindOfClass:[person2 class]]) {
        NSLog(@"person1 === person2");
    }
}
```

如果是判断判断两个有继承关系的对象

```objc
- (void)test4 {
    Person *person = [Person new];
    Man *man = [Man new];
    
    // 会沿着super_class一直查询到NSObject、NSProxy类
    // 因为Man继承自Person，而Perosn继承自NSObject，所以二者有共同的父类，所以类型是一致的
    if ([man isKindOfClass:[person class]]) {
        NSLog(@"1");
    }
    
    // 只会判断对象的isa指向的类型查询一次
    if ([man isMemberOfClass:[person class]]) {
        NSLog(@"2");
    }
}
```

运行结果

```
2016-07-24 21:51:03.586 Demos[7674:113910] 1
```

既然一个Objective-C类对应的`objc_class`是一个`单例`，那么比较两个对象的类型是否一致也可以像如下

```objc
- (void)test5 {
    
    Person *person = [Person new];
    Man *man = [Man new];
    
    if ([man class] == [Man class]) {
        NSLog(@"1");
    }
    
    if ([person class] == [Person class]) {
        NSLog(@"2");
    }
    
    if ([man class] == [person class]) {
        NSLog(@"3");
    }
}
```

输出结果

```
2016-07-24 21:55:58.694 Demos[7953:118690] 1
2016-07-24 21:55:58.695 Demos[7953:118690] 2
```

> 但是如果比较的对象所属类，如果是`类簇`时，就不能这么比较了，永远都不相等.

```objc
- (void)test5 {
    
    NSArray *array = [NSArray arrayWithObjects:@"1", @"2", nil];
    
    // 错误
    if ([array class] == [NSArray class]) {
        NSLog(@"1");
    }
    
    // 错误
    if ([array isMemberOfClass:[NSArray class]]) {
        NSLog(@"2");
    }
    
    // 正确
    if ([array isKindOfClass:[NSArray class]]) {
        NSLog(@"3");
    }
}
```

输出结果

```
2016-07-24 21:59:20.478 Demos[8152:122039] 3
```

至于原因，如果不懂类簇可以去详细看下类簇这个设计模式就懂了。

##当一个类具有`很多类型`的init初始化方法时，需要提供一个`全能`的最大init函数入口，来确保所有的默认的实例变量都能被初始化

- 其他的小init函数，最终调用`全能init`函数完成最终初始化

- 其实类似iOS最新提出的两种init函数
	- **便利构造器**，横向 调用其他的小init函数实现
	- **指定构造器**，纵向 调用父类的各种init函数实现

- 需要的情况下，可以存在多个`全能`init函数

- 如果父类某一个init函数，需要子类强制去自己重写，可以在父类init函数实现中
	- `[self doesNotRecognizeSelector:_cmd]`
	- `@throw [NSException exceptionWithName:<#(nonnull NSString *)#> reason:<#(nullable NSString *)#> userInfo:<#(nullable NSDictionary *)#>]`

如果子类没有重新实现此函数，就会在运行时崩溃退出，从而看到崩溃的方法处，然后去重新实现。

##对象的description方法实现，用来描述一个对象所有数据，和用来在lldb调试看到这个对象的所有数据

description的建议写法模板，返回一个NSString字符串对象

```objc
- (NSString *)description {

    return [NSString stringWithFormat:@"<%@: %p, %@>",
            [self class],
            self,
            
            //使用一个字典组装对象的所有属性值
            @{
              @"name" : _name,
              @"age" : @(_age)
              }];
}
```

lldb调试时，使用po输出为如下格式，可以很清楚看到这个对象所有属性值

```
<Person: 0x7fce1ae165c0, {
    age = 25;
    name = xiongzenghui;
}>
```

还有一个`debugDescription`方法，其实和description差不多，只不过是只有在lldb debug调试时候才有用。

最好的办法是结合这两个方法，如下这么写:

```objc
- (NSString *)description {
    
    return [NSString stringWithFormat:@"<%@: %p, %@>",
            [self class],
            self,
            
            //使用一个字典组装对象的所有属性值
            @{
              @"name" : _name,
              @"age" : @(_age)
              }];
}

- (NSString *)debugDescription {
    
    //1. 可以直接调用description
    return [self description];
    
    //2. 还可以组装description
    NSString *des = [self description];
    return [NSString stringWithFormat:@"ver: %d, des = %@", 1 , des];
}
```

可以参考YYModel中使用runime相关api解析出当前类的所有实例变量类型，然后全部自动化实现当前类的description对象方法。


##尽量向`外部`暴露`不可变`的对象版本，而对`内`使用`可变`对象版本。即对外只读，对内读写，来保证数据的安全性。

- 头文件.h中属性设置为readonly

```objc
@interface ViewController : UIViewController

@property (nonatomic, copy, readonly) NSString *identifier;

@end
```

- 实现文件.m中设置为readwrite

```objc
@interface ViewController ()

@property (nonatomic, copy, readwrite) NSString *identifier;

@end

@implementation ViewController

....

@end
```

可以防止`对象.identifier = 新值`会直接由编译器报错。

但是仍然可以通过KVC方式`[对象 setValue:新值 forKey:@"identifier"]`绕过属性读写权限的限制，直接完成变量的赋值修改。

但是总比直接暴露可写属性给外部调用者好一些吧。

###向外暴露`不可变`版本的集合对象，但是提供接口api函数来操作内部`可变`版本的集合对象

```objc
#import <Foundation/Foundation.h>

@interface Dog : NSObject

// 对外不可变集合对象
- (NSArray *)objects;

// 但是提供接口api函数操作内部可变集合对象
- (void)addObject:(id)obj;
- (void)removeObject:(id)obj;

@end

@implementation Dog {
    NSMutableArray __strong *_objects;
}

- (instancetype)init
{
    self = [super init];
    if (self) {
        _objects = [NSMutableArray new];
    }
    return self;
}

- (NSArray *)objects {
    
    //返回可变集合对象的一个`浅拷贝`后的不可变版本集合对象
    return [_objects copy];
}

- (void)addObject:(id)obj {
    // 做一些对obj的预先处理逻辑
    //...
    
    [_objects addObject:obj];
}

- (void)removeObject:(id)obj {
    // 做一些对obj的预先处理逻辑
    //...
    
    [_objects removeObject:obj];
}

@end
```

返回给外部调用者一个`浅拷贝`之后的`不可变`数组对象，让外界调用者只能读取数组对象，而不能够对数组对象执行修改。

但是可以对数组中的`item子对象`进行属性值修改，只是对`数组对象`不能修改。

##代码逻辑出错之后，反馈给调用者的形式

> 当引发异常后，那么本该在作用域末尾将要自动释放掉的对象，就不会自动被释放。


```objc
@try {
    // 逻辑代码....
}
@catch (NSException *exception) {
    // 捕捉到上述代码发生异常后执行的代码块
}
@finally {
    // 最后一定会被执行的代码块
    // 比如，释放之前创建的一些对象
}
```

> 抛出一个异常，只适用于严重的错误，因为会让程序崩溃退出。



那么当`没有出现那么严重`的问题时，对于api函数返回给客户端发生错误的写法模板:

- (1) 方法的返回值，返回nil/0、YES/NO

- (2) 二重指针，指针传递一个注册到自动释放池的NSError对象
	- Error Domain >>> 错误域，如网络错误是NSURLErrorDomain
	- Error Code >>> 错误码
	- User Info >>> 传递参数的字典对象
		- 可以是包含一段错误描述信息
		- 也可以包含另一个NSError对象，形成`错误链`

二重指针返回一个NSError对象的形式


```objc
- (void)doSomething:(NSError *__autoreleasing*)error_p;
```

即使不写`__autoreleasing`，编译器也会自动加上的。表示将指向的这个指针最终指向的NSError对象放入到一个自动释放池去管理，由自动释放池来管理这个NSError对象的释放。


```objc
NSError *error = nil;

[service doSomething:&error];
    
if (error) {
    //执行出错
} else {
    //执行成功
}
```

对上述doSomething方法进行改进，添加`BOOL值`，让客户端可以在`不关心NSError`的情况下，也能够知道执行成功还是失败.

```objc
- (BOOL)doSomething:(NSError *__autoreleasing*)error_p;
```

```
BOOL res = [service doSomething:nil];
    
if (!res) {
    //执行出错
} else {
    //执行成功
}
```

####使用枚举定义好所有可能的错误业务Code

```objc
/* 登陆相关错误 */
typedef  NS_ENUM(NSInteger, XZHLoginError) {
    XZHLoginErrorUnKnow             = 0,
    XZHLoginErrorUsernameInvalid    = 10001,
    XZHLoginErrorPasswordInvalid    = 10002,
    XZHLoginErrorAlreadyLogined     = 10004,
};

/* 支付相关错误 */
typedef  NS_ENUM(NSInteger, XZHPayError) {
    XZHPayErrorUnKnow               = 0,
    XZHPayErrorBankInvalid          = 10009,
};
```

##delegate协议回调与dataSource协议回调优化点

通常将delegate方法与dataSource方法，全都使用 `@optional`，那么就会大量出现如下判断代码

```objc
if (_delegate respondsToSelector:@selector(<#selector1#>)) {
    [_delegate selelector1]
}

if (_delegate respondsToSelector:@selector(<#selector2#>)) {
    [_delegate selelector2]
}

if (_delegate respondsToSelector:@selector(<#selector3#>)) {
    [_delegate selelector3]
}

...

```

- @protocol中有多个可选方法，就得写多少次
- 并且还可能重复性写一个方法的判断实现

###使用`位段 结构体`存储是否实现进行缓存优化

```objc
#import <Foundation/Foundation.h>

@class Dog;

@protocol XZHDogLifeConnvertDelegate <NSObject>

@optional

- (void)xzh_dogLife:(Dog *)dog didStart:(NSData *)data;

- (void)xzh_dogLife:(Dog *)dog didFailWithError:(NSError *)error;

- (void)xzh_dogLife:(Dog *)dog didEnd:(NSData *)data;

@end

@interface Dog : NSObject <NSCopying>

@property (nonatomic, weak) id<XZHDogLifeConnvertDelegate> dogLifeDelegate;

@end
```

```objc
#import "Dog.h"

/* 位段（bitfiled）结构体定义 */
struct DogLifeDelegateFlags{
    unsigned int didStart               : 1;//分配一个二进制位，存放bool值，0或1
    unsigned int didFailWithError       : 1;
    unsigned int didEnd                 : 1;
};

@implementation Dog {
    
    /* 存储delegate方法实现 */
    struct DogLifeDelegateFlags _dogLifeDelegateFlags;
}

- (void)setDogLifeDelegate:(id<XZHDogLifeConnvertDelegate>)dogLifeDelegate {
    
    //1.
    _dogLifeDelegate = dogLifeDelegate;
    
    //2. delegate方法实现判断进行缓存优化
    _dogLifeDelegateFlags.didStart = [dogLifeDelegate respondsToSelector:@selector(xzh_dogLife:didStart:)];
    _dogLifeDelegateFlags.didFailWithError = [dogLifeDelegate respondsToSelector:@selector(xzh_dogLife:didFailWithError:)];
    _dogLifeDelegateFlags.didEnd = [dogLifeDelegate respondsToSelector:@selector(xzh_dogLife:didEnd:)];
    
}

- (void)testDelegate {
    if (_dogLifeDelegateFlags.didStart) {
        [_dogLifeDelegate xzh_dogLife:self didStart:[NSData new]];
    }
    
    if (_dogLifeDelegateFlags.didFailWithError) {
        [_dogLifeDelegate xzh_dogLife:self didFailWithError:[NSError new]];
    }
    
    if (_dogLifeDelegateFlags.didEnd) {
        [_dogLifeDelegate xzh_dogLife:self didEnd:[NSData new]];
    }
}

@end
```

还有一种写法

```objc
#import "Person.h"

@protocol PersonProtocol <NSObject>

- (void)test1;

- (void)test2;

@end

@interface Person () {
    
    //位段结构体
    struct {
        unsigned int test1 : 1;
        unsigned int test2 : 1;
    }_delegateFlags;
}

@end

@implementation Person

- (void)doMethodWith:(NSString *)args {
    
    if (_delegateFlags.test1) {
        //delegate实现了test1
    } else if (_delegateFlags.test2){
        //delegate实现了test2
    }
}

@end
```

###如上用到了一个 `位段结构体` 定义，其格式如下

```c
struct 位结构名{
	unsigned int filed1 : 占用多少个二进制位(0~15);
	unsigned int filed2 : 1;//占用一个二进制位存放数值，如:BOOL、0、1
	....
};
```

- 上面的代码示例中，使用1个二进制位存放BOOL值0或1
- 0，表示`没有实现`该位段代表的delegate方法
- 1，表示`已经实现`该位段代表的delegate方法

所以表示YES（1）和 NO（0）使用 `一个` 二进制位就够了.

###为什么做这样的缓存优化？

- 一个实现类Delegate协议的具体类的对象，不太可能突然无法响应某一个SEL

- 所以可以在第一次设置delegate的时候，全部计算完毕并缓存起来


###关于回调的情况

- 回调执行一段代码，用于当前对象与回调业务的分离

- 回调获取数据，用于UI与数据的分离

- 使用delegate or block？
	
	- delegate方便调试，但是需要走消息查询，没有Block直接指向函数实现指向速度快
	- block会造成不易调试、产生retain cycle等

##将类的多个功能点，分散到单独的分类Category中

- 未做单独分离时的代码

```objc
#import <Foundation/Foundation.h>

@class Person;

@interface XZHPersonViewModel : NSObject

/* 处理一些统一的业务 */
- (void)doCommonTask;

- (void)addFriend:(Person *)person;

- (void)gotoWork;

- (void)payMoney:(Float32)money;

@end

#import "XZHPersonViewModel.h"

@implementation XZHPersonViewModel

- (void)doCommonTask {
    //....
}

- (void)addFriend:(Person *)person {
    //...
}

- (void)gotoWork {
    //...
}

- (void)payMoney:(Float32)money {
    //...
}

@end
```

```objc
#import "XZHPersonViewModel.h"

@implementation XZHPersonViewModel

- (void)doCommonTask {
    //....
}

- (void)addFriend:(Person *)person {
    //...
}

- (void)gotoWork {
    //...
}

- (void)payMoney:(Float32)money {
    //...
}

@end
```

- 单独分离到不同的Category后的代码

```objc

#import <Foundation/Foundation.h>

@class Person;

@interface XZHPersonViewModel : NSObject

/* 处理一些统一的业务 */
- (void)doCommonTask;

//- (void)addFriend:(Person *)person;
//
//- (void)gotoWork;
//
//- (void)payMoney:(Float32)money;

@end

#import "XZHPersonViewModel.h"

@implementation XZHPersonViewModel

- (void)doCommonTask {
    //....
}

@end
```

```objc
#import "XZHPersonViewModel.h"

@class Person;

@interface XZHPersonViewModel (FriendShip)

- (void)person_addFriend:(Person *)person;

@end

#import "XZHPersonViewModel+FriendShip.h"

@implementation XZHPersonViewModel (FriendShip)

- (void)person_addFriend:(Person *)person {
    //....
}

@end
```

```objc
#import "XZHPersonViewModel.h"

@interface XZHPersonViewModel (Work)

- (void)person_gotoWork;

@end

#import "XZHPersonViewModel+Work.h"

@implementation XZHPersonViewModel (Work)

- (void)person_gotoWork {
    //...
}

@end
```

还可以提供一个`私有 Category`单独封装一些不向外暴露的方法

```objc
#import "XZHPersonViewModel.h"

@interface XZHPersonViewModel (Private)

- (void)person_payMoney:(Float32)money;

@end

#import "XZHPersonViewModel+Private.h"

@implementation XZHPersonViewModel (Private)

- (void)person_payMoney:(Float32)money {
    //...
}

@end
```

注意如上分类中的方法，都是带有`前缀 person_xxx`的，来与其他分类中同名方法进行区分

###给实体类提供分类来封装一些简单的实体类数据的转换以及逻辑

- 实体类

```objc
#import <Foundation/Foundation.h>

@class Account;

@interface Person : NSObject

@property (nonatomic, copy) NSString *pid;
@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *sex;

//尽量向外只提供不可变数组，
@property (nonatomic, strong) NSMutableArray<Account *> *accounts;

@end
```

```objc
#import "Person.h"

@implementation Person

@end
```

```objc
#import <Foundation/Foundation.h>

@interface Account : NSObject

@property (nonatomic, copy) NSString *aid;
@property (nonatomic, copy) NSString *name;

@end
```

```objc
#import "Account.h"

@implementation Account

@end
```

- Person实体类的分类封装一些简单的逻辑

```objc
#import "Person.h"

typedef NS_ENUM(NSInteger, PersonSex) {
    PersonSexMan            = 1,
    PersonSexWomen          = 2,
};

@class Account;

@interface Person (Logic)

//解析属性对应的枚举类型
- (PersonSex)p_sex;

//对Account数组操作入口
- (void)p_addAccount:(Account *)account;
- (void)p_remooveAccount:(Account *)account;
- (NSArray *)p_findAccounts;
- (Account *)p_findAccountByAccountId:(NSString *)accountId;

@end
```

```objc
#import "Person+Logic.h"
#import "Account.h"

@implementation Person (Logic)

- (PersonSex)p_sex {
    if ([self.sex isEqualToString:@"man"]) {
        return PersonSexMan;
    } else {
        return PersonSexWomen;
    }
}

- (void)p_addAccount:(Account *)account {
    /*
     当Person提供的是不可变Account数组时，使用mutableCopy拷贝一个可变数组
     
     NSMutableArray *mArr = [self.accounts mutableCopy];
     
     //对传入的account做一些条件判断
     //看是否能够加入到数组
     
    [mArr addObject:account];
    self.accounts = mArr;
     */
    
    //
    
    if (account.aid) {
        [self.accounts addObject:account];
    }
}

- (void)p_remooveAccount:(Account *)account {
    if (account.aid) {
        [self.accounts removeObject:account];
    }
}

- (NSArray *)p_findAccounts {
    return [self.accounts copy];
}
- (Account *)p_findAccountByAccountId:(NSString *)accountId {
    Account *account = nil;
    for (Account *tmp in self.accounts) {
        if ([tmp.aid isEqualToString:accountId]) {
            
            //看是否需要修改，如果不需要尽量返回copy后的拷贝数据
            account = [tmp copy];
            break;
        }
    }
    return account;
}

@end
```

##`MRC`下对象的释放机制是根据对象的`引用计数器 retainCount`来决定对象是否释放。而`ARC`下是根据是否存在`__strong`类型的指针指向来决定对象是否释放。但实际上ARC还是使用的`引用计数器retainCount`的原理。

###MRC/ARC下，都有如下4个关键的核心概念

- (1) 自己`创建`的对象，由自己持有

```
属于自己创建的对象，所调用的OC方法命名的规则是，以如下开头的方法

- (1) new....
- (2) alloc....
- (3) copy....
- (4) mutableCopy....
```

- (2) `别人`创建的对象，我也能够去持有


MRC下，去持有别人创建的对象: 

```c
id obj = [别人创建的对象 retain];
```


ARC下，去持有别人创建的对象: 

```c
id __strong pointer = 别人创建的对象;
或
id pointer = 别人创建的对象;//默认缺省就是使用 __strong修饰符
```

- (3) 自己持有的对象，在不需要使用时候对其`释放`，让其`retainCount--`


MRC下:

```c
//1. 需要的时候持有别人创建的对象
id obj = [别人创建的对象 retain];

//2. 不需要时候，对持有的对象进行释放
[obj release];
```

ARC下:

```c
//1. 需要的时候持有别人创建的对象
id __strong *pointer = 别人创建的对象;

//2. 不需要时候，对持有的对象进行释放
pointer = nil;
```

- (4) `非自己`持有的象，不要去进行释放

千万不要对这类型的对象进行释放，否则造成这个对象多进行一次释放或提前释放，导致程序崩溃。而应该是让这个对象的持有者他自己去完成最后的释放操作，我们只管使用。

###ACR与MRC下，对象的持有、释放、废弃

- (1) ARC下
	- 通过 `__strong`类型的指针个数来判定是否废弃，底层仍然根据`retainCount`
	- 持有，增加一个strong指针
	- 释放，减少一个strong指针，即 `strong指针变量 = nil`;

- (2) MRC下，
	- 直接通过`retainCount`值来判定是否废弃这个对象
	- 持有，通过 NSObject提供的函数
		- 持有一，自己创建并持有，调用 `alloc/new/copy/mutableCopy`开头的方法返回的对象
		- 持有二，`id obj = [别人创建的对象 retain];`
	- 释放，`[obj release];`


###`释放`对象 和 `废弃`对象，这是两个不同的概念。

- (1) 释放，只是让对象的`retainCount - 1`，但对象可能仍然存活在内存中

- (2) 废弃，彻底将对象所在内存块抹掉，标记为`可重用`。即成为初始化的内存块工其他对象来使用，这个时候如果继续操作这个内存块就会崩溃程序

###ARC下对象的内存管理策略，对实例变量指针提供了如下几种对象指针修饰符:

- (1) strong强引用类型，会让其retainCount + 1

```objc
id __strong obj = [NSObject new];
```

- (2) weak弱引用类型，不会让 retainCount + 1，所以就不能持有一个对象

```objc
id __weak obj = [NSObject new];//创建之后立马就被废弃了
```

- (3) 看下 strong与weak指针修饰符的区别:

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    NSObject  *obj1 = [NSObject new];
    NSObject __strong *obj2 = [NSObject new];
    NSObject __weak *obj3 = [NSObject new];
    
}
```

fr v 输出结果

```
(NSObject *) obj1 = 0x00007fb263f0c3e0
(NSObject *) obj2 = 0x00007fb263f09a10
(NSObject *) obj3 = nil
```

可以看到最后一个使用`__weak`修饰的指针变量并没有持有创建出来的对象。

- (4) 自动释放池管理对象

```objc
id __autoreleasing obj = [NSObject new];//就类型[对象 autorelase];的效果
```

- (5)` __unsafe_unretained` 类似 `__weak`，但是不同的是当对象废弃时，修饰的指针变量不会自动赋值为nil，而` __weak`会

```objc
id __unsafe_unretained obj = [NSObject new];//创建之后立马就被废弃了
```

###下面一段代码说明MRC下是根据retainCount废弃对象，而ARC下是根据有多少个strong类型强引用指针指向来废弃对象

```objc
#import <Foundation/Foundation.h>

@interface Car : NSObject 

@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *cid;

- (instancetype)initWithName:(NSString *)name cid:(NSString *)cid;

@end

@implementation Car

- (instancetype)initWithName:(NSString *)name cid:(NSString *)cid {
    if (self = [super init]) {
        _name = [name copy];
        _cid = [cid copy];
    }
    return self;
}

- (void)dealloc {
    NSLog(@"<%@ - %p> name:%@ cid:%@ dealloc", self, self, _name, _cid);
}

@end
```


```objc
@interface ViewController () {
	Car *_car1;//默认就是 Car __strong *_car; 强引用指针
    Car *_car2;
    Car *_car3;
}	

- (void)test8 {

    // 临时指针
    Car *tmpCar = [[Car alloc] initWithName:@"c1" cid:@"c1"];
    
    _car1 = tmpCar;
    _car2 = tmpCar;
    _car3 = tmpCar;
    
    _car1 = nil;
    NSLog(@"execute _car1 = nil");
    
    _car2 = nil;
    NSLog(@"execute _car2 = nil");
    
    _car3 = nil;
    NSLog(@"execute _car3 = nil");
    
    tmpCar = nil;
    NSLog(@"execute tmpCar = nil");
}

@end
```

log输出结果

```
2016-07-26 23:09:15.831 Demos[6158:46961] execute _car1 = nil
2016-07-26 23:09:15.832 Demos[6158:46961] execute _car2 = nil
2016-07-26 23:09:15.832 Demos[6158:46961] execute _car3 = nil
2016-07-26 23:09:15.832 Demos[6158:46961] <<Car: 0x7ffcbb474ff0> - 0x7ffcbb474ff0> name:c1 cid:c1 dealloc
2016-07-26 23:09:15.832 Demos[6158:46961] execute tmpCar = nil
```

从输出的结果信息可以看出来，执行完`tmpCar = nil;`这一句代码之后，局部创建的Car对象就已经被废弃掉了，继而立刻在被废弃前执行了dealloc方法实现。然后在执行`NSLog(@"execute tmpCar = nil");`这一句打印代码。

那么分析下上面test8函数中，Car对象的retainCount变化过程:


```objc
- (void)test8 {

    Car *tmpCar = [[Car alloc] initWithName:@"c1" cid:@"c1"];// retainCount == 1

    _car1 = tmpCar;// retainCount == 2
    
    _car2 = tmpCar;// retainCount == 3
    
    _car3 = tmpCar;// retainCount == 4
    
    _car1 = nil;// retainCount == 3
    NSLog(@"execute _car1 = nil");

    _car2 = nil;// retainCount == 2
    NSLog(@"execute _car2 = nil");
    
    _car3 = nil;// retainCount == 1
    NSLog(@"execute _car3 = nil");
    
    tmpCar = nil;// retainCount == 0，此时立马去执行废弃对象，而后再执行后续的NSLog打印代码
    NSLog(@"execute tmpCar = nil");
}
```

###从输出结果和retainCount变化过程分析，一旦对象的retainCount（或者对象的strong引用指针个数）变为0，就会立马被废弃掉，而废弃这个对象的过程是`同步`执行的，并非`异步`后台执行。

ARC下是根据统计`strong指针的个数`，只要`strong指针个数 > 0`，那么被指向的对象就不会被释放。这其实就很类似对象的 `retain count > 0`不会被释放。

ARC环境下，所有的指针只要没有显示的使用 `__weak`、`__unsafe_unretained`进行修饰，那么都默认是强引用类型 `__stong`的指针，也就会默认让对象的`retainCount + 1` 。

所以，在局部函数块内使用的一个临时指针，默认情况下其实也是使用`__stong`修饰的强引用类型指针


```objc
@implementation DemosTableViewController

- (void)test9 {

    Car *car1 = [[Car alloc] initWithName:@"c1" cid:@"c1"];// retainCount == 1
    
    {
        Car *car2 = [[Car alloc] initWithName:@"c2" cid:@"c2"];// retainCount == 1

    } //car2.retainCount == 0
    
}//car1.retainCount == 0

@end
```

输出如下

```
2016-07-26 23:10:03.471 Demos[6224:48285] <<Car: 0x7fd240f00660> - 0x7fd240f00660> name:c2 cid:c2 dealloc
2016-07-26 23:10:03.472 Demos[6224:48285] <<Car: 0x7fd240f07e70> - 0x7fd240f07e70> name:c1 cid:c1 dealloc
```

可以看到car2指向对象先被废弃，而后才会废弃掉car1指向的对象。

展示一段非ARC的关于内存管理的写法

```objc
NSMutableArray *array = [[NSMutableArray alloc] init];
    
//data retainCount = 1
NSData *data = [NSData new];
    
//data retainCount = 2
[array addObject:data];
    
//在某个时刻，数组不再需要这个data，从数组移除后，data retainCount = 1
[array removeObject:data];
    
//不再需要这个data时执行如下
[data release];
data = nil;//防止野指针
    
//最后数组release，让内部所有对象执行release
[array release];
```

展示NSObject类型属性的setter方法的内存管理

```
@property (nonatomic, strong) User *user;
```

```
- (void)setUser:(User *)user {
    if (_user != user) {
        //1.
        [user retain];
        
        //2.
        [_user release];
        
        //3.
        _user = user;
    }
}
```

ARC如何清理对象，实际上就是在dealloc方法中，添加如下release代码

```
- (void)dealloc {
	[对象1 release];
	[对象2 release]
	[对象3 release]
	
	[super dealloc];//ARC下不用写，也不能写，ARC会自动加上此句代码
}
```

回调执行dealloc方法，是借助了C++的`析构函数`。

如果是`非Objective-C`对象，如CoreFoundation对象、c结构体实例等，那么就需要手动使用如下函数进行内存释放或对象释放

```objc
- (void)dealloc {

	//对于CoreFoundation对象进行释放
	CFRelease(CoreFoundation对象);
	
	//对于一些c结构体实例，直接进行废弃
	free(c结构体实例);
}
```

##ARC下常见的几种产生内存泄露的情况


###第一种、delegate指针变量必须使用`weak`来修饰指针，如果使用`strong`修饰指针会造成循环强引用，从而导致内存泄露

```objc
#import <UIKit/UIKit.h>

@class MyView;

@protocol MyViewDelegate

- (void)myViewDidWork:(MyView *)myView;

@end

@interface MyView : UIView

- (void)startWithDelegate:(id<MyViewDelegate>)delegate;

- (void)doWork;

@end


@interface MyViewDelegateImpl : NSObject <MyViewDelegate>
@property (nonatomic, strong) MyView *view;
@end

@implementation MyView {
    
    // 如果使用强引用delegate对象，造成两个对象之间的循环强引用
    id<MyViewDelegate> __strong _delegate;
}

- (void)startWithDelegate:(id<MyViewDelegate>)delegate {
    _delegate = delegate;
}

- (void)doWork {
    [_delegate myViewDidWork:self];
}

- (void)dealloc {
    NSLog(@"MyView dealloc");
}

@end


@implementation MyViewDelegateImpl

- (void)myViewDidWork:(MyView *)myView {
    NSLog(@"delegate 协议实现");
}

- (void)dealloc {
    NSLog(@"MyViewDelegateImpl dealloc");
}

@end
```

测试代码

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    //1. 局部对象
    MyView *view = [[MyView alloc] init];
    
    //2. 局部delegate实现类对象
    MyViewDelegateImpl *impl = [MyViewDelegateImpl new];
    
    //3. 让两个局部对象互相循环强引用
    impl.view = view;
    [view startWithDelegate:impl];
}
```

运行后没有任何的dealloc打印，说明这两个局部对象都没有被废弃，而没有被废弃的原因就是各自的strong指针不等于0，即retainCount不等于0。而终其原因就是retain了之后，并没有执行对应的release。

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    // MyView对象.retaiCount == 1
    MyView *view = [[MyView alloc] init];
    
    // MyViewDelegateImpl对象.retaiCount == 1
    MyViewDelegateImpl *impl = [MyViewDelegateImpl new];
    
    // MyView对象.retaiCount == 2
    impl.view = view;
    
    // MyViewDelegateImpl对象.retaiCount == 2
    [view startWithDelegate:impl];

}//MyView对象.retaiCount == 1，MyViewDelegateImpl对象.retaiCount == 1
```

那么也就说两个不同的对象A与B，产生了如下指向关系:

- (1) A对象 >>>>>>`strong`>>>>>>> B对象
- (2) B对象 >>>>>>`strong`>>>>>>> A对象

这样一来，A对象与B对象，一直都存在一个指向自己的strong类型指针，所以就都不会被废弃掉。

###解决方法一，使用weak修饰符来修饰指针变量，避免互相strong引用，修改各自的retainCount

```objc
@implementation MyView {
    
    // 如果使用强引用delegate对象，造成两个对象之间的循环强引用
//    id<MyViewDelegate> __strong _delegate;
    id<MyViewDelegate> __weak _delegate;
}
```

此时两个不同的对象A与B在一开始，就产生了如下指向关系:

- (1) A对象 >>>>>>`strong`>>>>>>> B对象
- (2) B对象 >>>>>>`weak`>>>>>>> A对象


这中情况下，A对象与B对象之间根本就不存在互相strong引用，也就根本不会产生循环引用。

###解决方法二、在不需要持有delegate时，主动让释放delegate对象。delegate对象一旦释放，就不会再对MyView对象形成strong引用，即就会执行`[MyView对象 release]`


在MyView对象不再需要持有delegate对象时:

```objc
delegate = nil;
```

这种情况，在没有主动执行`delegate = nil;`时，A与B在产生了如下指向关系:

- (1) A对象 >>>>>>`strong`>>>>>>> B对象
- (2) B对象 >>>>>>`strong`>>>>>>> A对象

只有在某个时刻，执行了`delegate = nil;`之后，才会产生如下指向关系:

- (1) A对象 >>>>>>`strong`>>>>>>> B对象
- (2) B对象 >>>>>>`weak`>>>>>>> A对象

所以，只有当MyView对象执行了`delegate = nil;`之后，才会解除循环strong引用的问题。

###NSURLSession对象接收回调对象的delegate属性声明时，就存在循环引用的问题

```objc
@interface NSURLSession : NSObject

....

@property (nullable, readonly, retain) id <NSURLSessionDelegate> delegate;

....

@end
```

可以看到delegate属性使用的是`retain`修饰符，和`strong`是一样的，都会对传入的回调对象造成`强`引用，即让传入对象的`retainCount+1`。

此时就需要考虑，我们代码中去持有NSURLSession对象的对象是不是`单例`，如果是单例就基本上不用去考虑释放不释放的问题了。而如果不是一个单例的话，就得按照之前的思路做了:

```objc
//1. 
_session = [[NSURLSession alloc] initWithDelegate:self .....];

//2. 使用session发起网络请求
.....

//3. 某个时候不再需要使用session
[_session invalidateAndCancel];
_session = nil;
```

###第二种、Block是某一个对象的成员变量，此时Block对象不管是引用自己还是别人，都会造成循环强引用，从而导致内存泄露

- ViewModel业务类对象

```objc
@interface ViewModel : NSObject

/**
 *  提供给UI层对象的接口
 */
- (void)apiCall:(void (^)(id entity))block;

@end
```

```objc
#import "Request.h"

@implementation ViewModel {
    
    // 成员属性强引用
    Request __strong *_req;
}

- (void)apiCall:(void (^)(id entity))block {
   
    _req = [Request new];
    
    _req.block = ^() {
        id entity = [self parseJSON];
        if (block) block(entity);
    };
    
    [_req start];
}

- (void)dealloc {
    NSLog(@"ViewModel Dealloc %@ - %p", self, self);
}

/**
 *  解析json成为实体类对象
 */
- (id)parseJSON {
    return [NSObject new];
}

@end
```


- 网络请求Request类对象

```objc
@interface Request : NSObject

@property (nonatomic, copy) void (^block)(void);

- (void)start;

@end
```

```objc
@implementation Request

- (void)dealloc {
    NSLog(@"当前废弃的Request对象: %p", self);
}

- (void)start {
    NSLog(@"开始网络请求");
    
    if (_block) {
        _block();
    }
}

@end
```


- ViewController测试类

```objc
@interface ViewController () 
@property (nonatomic, strong) id entity;
@end

@implementation ViewController  
- (void)release8 {
    
    //1. 局部ViewModel对象
    ViewModel *vm = [ViewModel new];
    
    //2. 调用ViewModel对象的api
    [vm apiCall:^(id entity) {
        self.entity = entity;
        _entity = entity;
    }];
}
@end
```

运行后log输出

```
2016-10-26 22:55:09.242 Demos[3134:32672] 开始网络请求
```

但是并没有看到任何的关于`局部创建的ViewModel对象`的 dealloc信息 和 `Request对象` dealloc信息。那么说明，此时发生了内存泄漏。

分析下ViewModel对象与Request对象之间的持有关系:

```
1. ViewModel对象->_req ----> Request对象
2. Request对象->_block -----> ViewModel对象
```

分析一下ViewModel的`apiCall:`对应的方法中各自的retainCount变化过程:

```objc
- (void)apiCall:(void (^)(id entity))block {

    _req = [Request new];
    // ViewModel.retainCount = 1, Request.retainCount = 1
    
    _req.block = ^() {
        id entity = [self parseJSON];//持有了ViewModel对象
        if (block) block(entity);
    };
    // ViewModel.retainCount = 2, Request.retainCount = 1
    
    [_req start];
    
}// ViewModel.retainCount = 1 >>>> Request.retainCount = 1
```

最终出了方法块之后，由于`ViewModel.retainCount = 1`导致ViewModel无法废弃，直接影响Reuqest也存在一个强指针，也就导致Request对象无法废弃。这就是ViewModel对象与Request对象之间互相进行持有。谁都释放不了....


###不嫌麻烦，最简单的写法就是block外面使用weak修饰，block内部使用strong修饰

- block外面使用weak修饰

主要是为了block内部使用这个weak指针时，不会让对象执行retain消息

- block内部使用strong修饰

可以去看下OC的那本内存高级编程的书，主要是避免每一次使用weak指针时，都会添加注册到一个autoreleasepool中去。

因为weak修饰的指针使用时，runtime system 为了防止过早的释放废弃，所以会默认添加注册到一个autoreleasepool中去。

而使用strong修饰指针后，好像是只会添加一次到autoreleasepool中去。

具体的还得去看看那本书，有点忘记了。。。。

###其他解决方法、类似上面delegate的思路，让其中一方`主动`释放对另外一方的strong持有关系

让Request暴露一个方法实现，用于外界主动解除block对外部对象的strong引用

```objc
@interface Request : NSObject

@property (nonatomic, copy) void (^block)(void);

/**
 *  主动解除Block对象对传入的对象的strong强引用的关系
 */
- (void)clearBlocks;

- (void)start;

@end
```

```objc
@implementation Request

- (void)dealloc {
    NSLog(@"当前废弃的Request对象: %p", self);
}

- (void)start {
    NSLog(@"开始网络请求");
    
    if (_block) {
        _block();
    }
}

- (void)clearBlocks {
    // 释放掉Block对象，从而让Block对象释放所有持有的对象
    _block = nil;
}

@end
```

然后在ViewModel对象中使用Request对象的地方修改，在最终执行完毕不再需要Request对象时，主动调用上面的方法实现解除对外部对象的持有

```objc
@implementation ViewModel {
    
    // 成员属性强引用
    Request __strong *_req;
}

- (void)apiCall:(void (^)(id entity))block {
    
    _req = [Request new];
    
    _req.block = ^() {
        id entity = [self parseJSON];
        if (block) block(entity);
    };
    
    [_req start];
    
    //TODO: (这里应该是异步完成后执行) 主动释放Request对象的block对象
    [_req clearBlocks];

}

- (void)dealloc {
    NSLog(@"ViewModel Dealloc %@ - %p", self, self);
}

/**
 *  解析json成为实体类对象
 */
- (id)parseJSON {
    return [NSObject new];
}

@end
```

再次运行上面的ViewController可以得到输出

```
2016-10-26 23:30:49.002 Demos[4976:62489] 开始网络请求
2016-10-26 23:30:49.003 Demos[4976:62489] ViewModel Dealloc <ViewModel: 0x7fd03bc06460> - 0x7fd03bc06460
2016-10-26 23:30:49.003 Demos[4976:62489] 当前废弃的Request对象: 0x7fd03bc53540
```

那么这样现在ViewModel对象与Request对象都废弃掉了。

###还有一种就是Request对象通过Block实例变量引用Request对象`自己`也会造成循环引用。造成的原因和上述产生原因是一致的，解决的办发也是一样的。

```objc
@implementation ViewController  
- (void)release9 {
    
    //1. 
    Request *req = [Request new];
    
    //2. 自己持有自己
    req.block = ^() {
        [req description];
        [req clearBlocks];
    };
    
    //3. 
    [req start];
    
    //4. 
    req = nil;
}
@end
```

运行之后，Request对象没有被dealloc。如上代码一样会产生循环强引用，而不同的是Request对象自己引用自己。


```objc
@implementation ViewController  
- (void)release9 {
    
    Request *req = [Request new];
    
    //自己持有自己
    req.block = ^() {
        [req description];
        
        // 强制释放block对象
        [req clearBlocks];
    };
    
    [req start];
    
    req = nil;
}
@end
```

输出如下

```
2016-10-26 23:49:07.421 Demos[6112:81833] 开始网络请求
2016-10-26 23:49:07.422 Demos[6112:81833] 当前废弃的Request对象: 0x7fb5905be710
```

现在能够释放废弃了。

###参考自YTKNetwork中，YTKRequest对象的successBlock与faileBlock对外界对象持有解除的做法

- (1) YTKNetworkAgant单例，负责所有的AFNetworking网络请求的回调
- (2) YTKNetworkAgant单例，找到当前回调的AFHttpRequstOperation对象，所对应的YTKRequest对象
- (3) 取出YTKRequest对象的successBlock与faileBlock这两个block实例变量，然后分别按情况执行
- (4) 执行完成YTKRequest对象block之后，YTKNetworkAgant单例继续执行`-[YTKRequest clearCompletionBlocks]`释放block对象

###MRC下分 全局、栈、堆三种Block对象，在ARC其实也分三种，但是默认会对 `栈Block` 进行拷贝生成 堆Block。

而对于一些 堆Block 执行完就会自动进行释放时，其实直接传入`self`也不会形成循环引用:

Block对象虽然持有当前对象，但是当前对象并不持有Block对象，就不会形成循环强引用:

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

    Car *car = [Car new];
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"car:%p name = %@", car, car.name);
    });
}
```

依然是会输出Car对象的废弃时执行dealloc函数实现。像一些GCD队列调度Block对象，在执行完时会被ARC运行时系统自动释放。

##dealloc方法能做的事情

当一个对象即将废弃掉时，会由系统自动回调dealloc方法实现，且在这个对象声明周期中`只会调用一次`。

###那么建议在dealloc方法实现中做的一些事情:

- (1) 释放strong强引用类型指针（OC对象、CoreFoundation实例、c结构体实例）
- (2) 移除关注的通知
- (3) 移除KVO实例变量值改变的观察

如果是持有文件资源、网络连接资源等耗内存的资源，就不应该放在dealloc方法中去完成释放。而应该自己提供一个方法来完成释放，并在适当的实际自己去调用。

###不要再dealloc方法内完成的事情

- (1) 调度异步子线程的任务，而等待回调时又继续操作当前对象

因为等到回调的时候，很可能这个对象已经被废弃了，如果此时继续执行回调操作这个废弃的对象，就会造成程序崩溃。

- (2) 不要在dealloc方法执行的所在线程上，去执行需要在其他线程（主线程）执行的操作

- (3) 不要在dealloc中调用属性的setter/getter

因为很可能当前对象监听了实例变量的KVO通知，而操作setter就会发出KVO通知，而此时观察的回调对象已经被废弃，所以也会造成程序崩溃。

##善用`@try-@catch-@finally`捕获引发异常的代码段，并在`@catch{} or @finally{} 代码块中，对异常代码段中创建的内存对象进行释放

> 当引发异常后，那么本该在作用域末尾将要自动释放掉的对象，就不会自动被释放。

```objc
- (void)test10 {

    @try {
    	//1. 
        Car *car = [[Car alloc] initWithName:@"car1" cid:@"car1"];
        
        //2. 
        [car setValue:@"hello world" forKey:@"price"];//不会造成程序崩溃
        
        //3. 
        [car.name integerValue];//不会造成程序崩溃
        
        //4. 
        [car noImple];//执行一个没有实现的消息，会造成崩溃
        
        //5.
        car = nil;
        
        NSLog(@"");
    }
    @catch (NSException *exception) {
        NSLog(@"代码执行异常");
    }
}
```

运行之后的输出

```
2016-07-27 23:23:30.238 Demos[3848:34367] -[Car noImple]: unrecognized selector sent to instance 0x7fc313d2f0a0
2016-07-27 23:23:34.203 Demos[3848:34367] 代码执行异常
```

但是并没有看到执行任何的Car对象dealloc废弃的日志，那就意味着上面发生异常并捕获时，造成了没有对Car对象进行释放，导致Car对象内存泄露了。

因为当执行到`4.`时立马抛出一个异常，然后流程立马走到`@cache(){}`块中，所以`5.`根本就不会被执行。

> 解决发生异常并捕获到异常时，释放掉创建的对象。

```objc
- (void)test11 {
    
    Car *car = nil;
    
    @try {
        car = [[Car alloc] initWithName:@"car1" cid:@"car1"];
        [car setValue:@"hello world" forKey:@"price"];//不会造成程序崩溃
        [car.name integerValue];//不会造成程序崩溃
        [car noImple];//执行一个没有实现的消息，会造成崩溃
        NSLog(@"");
    }
    @catch (NSException *exception) {
        NSLog(@"代码执行异常");
    }
    @finally {
        //在最终执行块执行释放对象
        car = nil;
    }
}
```

运行后的输出

```
2016-07-27 23:28:37.508 Demos[4141:39536] -[Car noImple]: unrecognized selector sent to instance 0x7fc801e10a50
2016-07-27 23:28:37.509 Demos[4141:39536] 代码执行异常
2016-07-27 23:28:37.510 Demos[4141:39536] <<Car: 0x7fc801e10a50> - 0x7fc801e10a50> name:car1 cid:car1 dealloc
```

可以看到Car对象执行dealloc，也就是捕获到异常后，最终被废弃了。


####小结异常处理

- 在`@try{}`中发生到异常捕获之后，必须要将`@try{}`代码块中创建的所有对象都要清理干净

- 默认情况下，ARC不会生成捕获到异常之后，清理创建出来的资源的代码
	- 但是可以通过打开 `-fobjc-arc-exceptions` 设置打开，让ARC生成
	- 但是不建议这么做，这样会造成程序的大小变大，运行效率降低

- Objective-C 默认只有当发生异常情况导致`程序即将要终止`的时候，才会抛出异常。所以程序中，我们应该尽可能`少`的自己抛出自己的一些异常。而应该是`ErrorCode + BOOL + NSError`的形式。

- 对于`NSError`的形式来代替抛出异常形式
	- Error Domain 错误域、错误描述
	- Error Code	错误码
	- User Info字典 错误的参数

****

###在`不需要持有`一个对象的情况下，尽量使用`weak`引用。注意不要使用`assign`、`unsafe_unretained`来修饰`Objective-C类对象`的指向，会形成野指针导致程序崩溃。


- strong 
	- 会retain一次传入的新值，表示一个拥有对象的关系

- weak 
	- 不会retain传入的新值
	- 仅仅只是指向传入的对象，想使用对象的方法而已，并不会持有一个对象
	- 如果使用`id __weak obj = [NSObject new];`执行完时对象立马会被废弃，因为无法持有对象
	- 当所指对象被释放时，weak指向的指针会由ARC`赋值nil`，不会造成程序崩溃

- `unsafe_unretained`
	- 基本上类似 weak
	- 但是当所指对象释放时，`不会`由ARC将指针`赋值nil`

- assign
	- 绝大多数情况下，不应该使用这种修饰指针指向`对象`
	- 同 `unsafe_unretained ` 一样，在所指对象废弃时，`不会`由ARC将指针`赋值nil`

- copy
	- 首先对传入的对象进行拷贝，一般都是`浅拷贝`，生成一个`新的对象
`
	- 然后使用`__strong`修饰符修饰指针来管理该`新对象`
	- 并对象之前的老对象进行`释放release`
	- 其实很类似`__strong`，只不过会`拷贝出一个新的对象`，这就有点问题了

- autorelasing
	- 将对象放入到一个pool对象去持有
	- 当pool对象废弃掉时，将pool内所有的对象发送release消息
	- 通常是为了延迟对一些对象进行释放release消息


所以如果是`不需要长期拥有`一个对象，最好使用`__weak`来修饰的指针去指向一个对象，不会让其retainCount++。

##使用autoreleasepool降低内存峰值

- 内存峰值: 内存会`忽高忽低`，主要是针对一些临时对象的创建与废弃

```objc
- (void)test12 {

    Car *car = [Car new];
    NSString *str = @"Hello World";
    NSInteger age = 19;
    //...
    
}   
```

如上这些只在局部内使用的变量或对象，都成为临时的对象。

- 当出现`内存峰值`的时候，可以考虑使用autorelasepool来包裹，用于即时释放掉一些不必要的`临时对象`

- 对于App程序进程中的`主线程`
	- (1) `主线程`创建的时候，就已经自动创建了pool对象
	- (2) 所以对于`主线程`是不需要再创建pool对象
	- (3) 并且在主线程开始执行下一个事件循环（runloop循环）时，清空上一次的pool中的所有对象

- 如下格式的代码就会造成内存峰值

```objc
- (void)test12 {
    for (NSInteger i = 0; i < 10000; i++) {
        Car *car = [[Car alloc] init];
    }
}
```

就是不断的创建的新的对象，而在创建这个对象的过程中，可能会借助一些其他的类对象，那么就不断的也会创建其他类的对象，也即是说会创建很多次最终不使用的`临时`对象。

而这些`临时`对象其实是根本不太重要的，早应该被废弃掉的。但是这些对象都被加入到`主线程的pool中`，而一个线程（主线程、子线程都一样的）只有在进入`下一个RunLoop事件循环`时，才会将上一次pool中所有的对象。

那么就是说只有执行完这所有的创建对象之后，才会去清空pool对象。就会造成开始时不断的创建对象，内存就不断的上涨。最后一次性清空pool中的所有临时对象，而一下子内存又爆降。这就是内存峰值出现的场景。

那么解决的办法就是，使用`局部pool对象`来包裹所有的临时对象，而不使用`主线程自己的pool对象`。因为`局部pool对象`只要出了作用域就会自动被释放废弃掉。

```objc
- (void)test12 {
    @autoreleasepool {
        for (NSInteger i = 0; i < 10000; i++) {
            Car *car = [[Car alloc] init];
        }
    }
}
```

MRC下，还有一种更为`重量级`的释放池。并不会在执行完上1W次循环后，最后一次性去释放pool内的对象，而是会经常的去尝试释放pool内没有其他地方使用的临时对象

```objc
NSAutoreelasePool *pool = [NSAutoreelasePool new];

for (NSInteger i = 0; i < 10000; i++) {
    Car *car = [[Car alloc] init];
}

[pool drain];
```

在ARC下，可以直接使用`@autoreleasepool{}`来完成上面的释放池

```objc
for (NSInteger i = 0; i < 10000; i++) {
	@autoreleasepool {
		Car *car = [[Car alloc] init];
	}
}
```


##永远不要直接使用 `[对象 retainCount]`

###虽然现在已经是ARC的时代，但还是有必要理解下

```
当一个对象被创建出来之后，其 retainCount = 1
```

其实如上的retainCount = 1，这是错误的，正确描述应该是`retainCount > 0`

###不能根据retainCount来判断对象是否回收

- [对象 retainCount] 返回的只是调用该方法`这个时间点`上的值
- 但并不能确定稍后如果是释放掉自动释放池时，可能造成对象释放

###错误使用retainCount的例子

```objc
while([object retainCount]) {
	[object release];
}
```

- 其代码愿意是，一直让retainCount-1，直到为0，然后被释放.

- 但其实如上代码有如下几点错误:

	- 错误一、没考虑到对象可能是**处于一个自动释放池中**，会多进行一次release，导致程序崩溃

	- 错误二、**[对象 retainCount] 可能永远都不会返回 0**
		- 很可能retainCount=1的对象此刻也会被释放掉
		- 而此时再执行[object release]时，就会导致程序崩溃

###[对象 retainCount] 可能永远都不会返回 0

- iOS系统当内存紧张的时候，会对内存做出优化
- 而优化就是将之前retainCount=0才释放的标准`提高`
- 也就是此时此刻程序中的`retainCount=1 或 retainCount=2`的对象也会被释放掉

###单例对象的retainCount可能是一个很大的数字（如: 2^64-1、2^63-1）

- 系统会尽可能的将NSString对象，作为一个`单例`对象处理
- 而如果字符串是类似 @"hello world!" 这样的NSString对象时，会处理为`编译器常量 compile-time constant`
	- 此种情况下的字符串，编译器会把该`字符串内容`放到应用程序的`二进制可执行库`中
	- 这样在运行期的时候，就不用创建`NSString对象`，而是直接从二进制文件中获取字符串内容
	- **使用的是一种 标签指针（tagged pointer）概念**

### 对`单例`对象进行retain、release都是没有任何作用的

- **单例对象的retainCount不会做出修改**
- 即就是 空操作（no-op）

### 在任何时候都不要通过 `[对象 retainCount]` 完成任何逻辑性代码

##苹果异步子线程完成各种耗时代码

- 第一块、代码块（block），就是一个NSBlock类簇类的对象，同样具备内存管理

- 第二块、Grand Central Dispatch (GCD)  线程调度中心
	- (1) 两种 Queue 线程队列:
		- Concurrent Queue 无序并发线程队列
		- Serial Queue 顺序多线程同步队列
	- (2) 两种将Block入队方式:
		- async 异步执行
		- sync 立刻等待执行

使用GCD来屏蔽直接使用NSThread、-[NSObject performSelectorXxx:]等操作线程的api，开发者完全不用接触任何的线程函数api，只需要使用Queue+Block即可。

###blcok（块）基础知识

- block也是具有`数据类型`的一种值

如下代码是摘录自YYModel中获取一个block的数据类型的代码

```c
static force_inline Class YYNSBlockClass() {
    static Class cls;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
    
    	//1. 定义一个block，其实得到一个Block对象
        void (^block)(void) = ^{};
        
        //2. block值转换成NSObject对象，获取其所属类
        cls = ((NSObject *)block).class;
        
        //3. 根据其isa指针，一直查找，找到其真实类型
        while (class_getSuperclass(cls) != [NSObject class]) {
            cls = class_getSuperclass(cls);
        }
    });
    return cls; 
}
```

运行输出结果

```
2016-07-28 00:19:10.498 Demos[6980:83290] NSBlock
```

可以看到Block的类是NSBlock。但其实这是个`类簇类`，并不是最终Block对象的真实类型。但总而言之，我们经常写的Block其实他就是一个`NSObject`子类的对象。那么也就具备OC对象内存管理的一切规则。

也正因为如此，前面说的`Block对象`会去持有传入的指针所指向的对象的原因。因为Block自己正是一个对象，所以也能去持有其他的对象。

###查看Block的真实类型

```objc

- (void)test12 {

    void (^block1)() = ^(){};
    
    NSInteger i = 0;
    void (^block2)() = ^(){
        NSLog(@"i = %ld", i);
    };
//断点 
    NSLog(@">>>>>>>>>>>>>>");
    
    block1();
    block2();
//断点    
    NSLog(@">>>>>>>>>>>>>>");
    
}
```

断点lldb输出结果

```
(lldb) po block1
<__NSGlobalBlock__: 0x1097dc3c0>

(lldb) po block2
<__NSMallocBlock__: 0x7f9069e2aad0>

(lldb) c
2016-07-28 00:24:56.393 Demos[7301:89497] >>>>>>>>>>>>>>
2016-07-28 00:24:56.394 Demos[7301:89497] i = 0
Process 7301 resuming
(lldb) po block1
<__NSGlobalBlock__: 0x1097dc3c0>

(lldb) po block2
<__NSMallocBlock__: 0x7f9069e2aad0>

(lldb) c
2016-07-28 00:25:08.221 Demos[7301:89497] >>>>>>>>>>>>>>
Process 7301 resuming
```

可以得到两个真实的Block类型: `__NSGlobalBlock__` 与 `__NSMallocBlock__`。

那么应该还有`栈Block`，估计应该是叫做`__NSStackBlock__`，但是整不出来。而且block2就是一个局部Block，按理说就是一个栈Block。

其实本来是真的是栈Block，只不过默认被ARC进行`拷贝并在堆上生成了一个新`的Block对象，这么就成为了一个`堆Block对象`了。

block内部可以访问（捕获）外部的变量，但是`不能`对`外部变量进行修改`。

###Block捕获的是 `对象类型` 的指针，就会自动对指针指向的对象 `发送retain消息` ，也就是会去`持有`指针指向的对象，让这个对象的retainCount++ 。

- 当Block如果是GCD调度执行的Block，那么就会在最终执行完毕的时候，对Block进行释放，从而让Block对象废弃，继而让Block对象持有的对象也执行释放

- 但是当Block对象是属于`某个类对象的 实例变量`时，如果`类对象不被废弃` 或 `类对象不主动去释放Block对象`，那么就会让Block对象持有传入内部指针指向对象无法释放，从而导致Block持有的对象内存泄露

###对于Block对象中传入对象指针时，能让传入对象保持retainCount平衡的关键点:

- (1) 对象指针传入到Block对象时，Block对象会对传入指针指向的对象 `retainCount++`

- (2) 那么必须在Block对象执行完毕或不再使用时，将Block对象废弃掉，从而让Block对象持有的对象 `retainCount--`

只有在最后不再使用Block对象时，将`Block对象废弃掉`，这样才能让传入Block的对象的retainCount恢复平衡。

- 如果需要 `修改` 外部变量，需要加上 `__block` 修饰外部变量 .

- block内部可以捕获`当前范围`内的所有变量，也包括`self指针`，但是不同的是block总是可以直接修改self指向对象的实例变量


###一个Block对象的内存结构

一个block也是一个NSObject子类对象，所以每一个对象都是使用不同的内存块、都有不同的大小，也拥有不同的数据结构。

如下图示一个block对象的内存中存储的结构

![](http://i4.piimg.com/567571/785e457281da0291.png)

对于Block对象结构中有如下几个重要的东西:

- (1) isa指针，指向block对象的所属类，这个同之前NSObject对象的isa指针是一样的

- (2) invoke`函数指针`，指向block块内的实现代码

```c
函数返回值类型 (*函数指针变量名)(void *, 参数1, 参数2, 参数3 ..., 参数n);
```
第一个参数是一个 `void *`类型的指针，这个是第一个必须传递的参数。而这个指针变量参数指向的就是要执行的Block对象。

为何要传递Block对象指针进来了？因为执行Blcok对象时，需要从内存中找到Block执行时所需要的一切对象/变量的所在地址，从而去操作这些变量/对象。

虽然只是拷贝的原始对象的指针，但是Block对象会持有这个拷贝指针，也就对原始对象形成了一个strong引用，在MRC下其实就相当于对原始对象做了一次retain。


##多使用`GCD队列`进行多线程同步，而少用`锁`进行多线程同步

####确定这`一个变量、一个对象、一段代码`，到底需不需要进行多线程的同步处理

如果一个变量、一个对象、一段代码，始终都只会在`唯一的一个线程上执行读写`，那么就不需要进行多线程同步处理。比如`UI对象`，规定UI对象只能在`主线程`这唯一一个线程上执行。

但如果一个变量、一个对象、一段代码，将有可能在`二个及二个以上的不同线程`上执行读写，那么此时就`必须`进行多线程同步处理。如果不进行处理，一般会出现数据的不统一，严重的会出现野指针导致程序崩溃。

比如，下面这一段多个线程同时读写一个懒加载的缓存对象时，基本上都会崩溃:

```objc
@interface MultithreadViewController ()
@property (nonatomic, strong) NSMutableDictionary *cache;
@end

@implementation MultithreadViewController

- (NSMutableDictionary *)cache {
    if (!_cache) {
        _cache = [NSMutableDictionary new];
    }
    return _cache;
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

	dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 1000; i++) {
            [self.cache setObject:@"111" forKey:@"111"];
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 1000; i++) {
            [self.cache setObject:@"222" forKey:@"2222"];
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 1000; i++) {
            [self.cache setObject:@"333" forKey:@"333"];
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 1000; i++) {
            [self.cache setObject:@"4444" forKey:@"5555"];
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 1000; i++) {
            [self.cache setObject:@"6666" forKey:@"6666"];
        }
    });
}

@end
```

也不一定是必崩，但是很大的几率会崩溃，最后抛出的异常时`EXC_BAD_ACCESS`，基本上就是因为使用了`已经被废弃的对象`造成崩溃，也就是使用了`野指针`。

那么上述会造成野指针的原因:

- (1) `第一个线程`进入执行`self.cache`，执行getter方法，发现此时`_cache`为nil，流程走到创建`NSMutableDictionary对象`，但此时突然被CPU切换到`第二个线程`进行执行

- (2) `第二个线程`进入执行`self.cache`，执行getter方法，发现此时`_cache`也是nil。可能此时`第二个线程`开始创建`NSMutableDictionary对象`，并使用`_cache`指向，但此时又突然被CPU切换到第一个线程进行执行

- (3) CPU再次切换回`第一个线程`，让第一个线程接着之前被暂停的地方继续执行。也就是继续执行创建`NSMutableDictionary对象`，并使用`_cache`指向。
	- 这一步就出问题了，之前第二个线程创建的`NSMutableDictionary对象`，因为失去了`_cache`指向，然后就被废弃了

- (4) 很有可能`NSMutableDictionary对象`废弃到`_cache`指向新的`NSMutableDictionary对象`这个过程中，CPU再次切换回第二个线程执行，`[self.cache setObject:@"222" forKey:@"2222"];`，但是此时`_cache`指向之前已经被废弃的`NSMutableDictionary对象`，所以程序崩溃了


这个过程中，最主要的原因就是: `CPU随意的切换其他的线程执行任务，导致某一个线程没有完整的执行玩一段逻辑`。

> 解决，让对 `_cache`指向的字典对象的读写处于一个`完整的逻辑`中，告诉CPU必须执行完之后才能切换其他线程执行。

```objc
@interface MultithreadViewController ()
@property (nonatomic, strong) NSMutableDictionary *cache;
@property (nonatomic, strong) dispatch_queue_t serialQueue;
@end

@implementation MultithreadViewController

- (NSMutableDictionary *)cache {
    if (!_cache) {
        _cache = [NSMutableDictionary new];
    }
    return _cache;
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    // 在主线程上，创建一个信号值
//    semaphore = dispatch_semaphore_create(1);
    
    // 在主线程创建一个串行队列
    _serialQueue = dispatch_queue_create("serial", DISPATCH_QUEUE_SERIAL);
    
    // 后续线程争夺这一个信号值
//    dispatch_async(dispatch_get_global_queue(0, 0), ^{
    dispatch_async(_serialQueue, ^{
        for (int i = 0; i < 1000; i++) {
            [self.cache setObject:@"111" forKey:@"111"];
        }
    });
    
//    dispatch_async(dispatch_get_global_queue(0, 0), ^{
    dispatch_async(_serialQueue, ^{
        for (int i = 0; i < 1000; i++) {
            [self.cache setObject:@"222" forKey:@"2222"];
        }
    });
    
//    dispatch_async(dispatch_get_global_queue(0, 0), ^{
    dispatch_async(_serialQueue, ^{
        for (int i = 0; i < 1000; i++) {
            [self.cache setObject:@"333" forKey:@"333"];
        }
    });
    
//    dispatch_async(dispatch_get_global_queue(0, 0), ^{
    dispatch_async(_serialQueue, ^{
        for (int i = 0; i < 1000; i++) {
            [self.cache setObject:@"4444" forKey:@"5555"];
        }
    });
    
//    dispatch_async(dispatch_get_global_queue(0, 0), ^{
    dispatch_async(_serialQueue, ^{
        for (int i = 0; i < 1000; i++) {
            [self.cache setObject:@"6666" forKey:@"6666"];
        }
    });
}

@end
```

上面是通过GCD的串行队列（先进先出，排队等待，顺序执行），让所有的对`_cache`指向的对象的线程读取任务，统一排队按照顺序执行。

这样就通过队列的机制完成了并发无序的控制，继而达到了多线程同步的效果。

###在没有GCD之前，多线程同步只能通过 `同步代码块`、 `锁`、 `信号量`

```objc
//1. 第一种
@synchronized(对象){
	操作对象的代码
};
```

```objc
//2. 第二种
NSLock
NSRecursiveLock
NSCondition
```
	
###使用@synchronized(对象){操作对象的代码}的优缺点:

- (1) 基本原理
	- 可以直接让括号里面对指定的对象操作的代码，在多线程上顺序执行
	- 当进入`@synchronized(对象){ ... `时，默认会创建一个`互斥锁`，然后进行锁定，让其他线程在外面等待
	- 当出`}`时解除锁，让其他线程进入

- (1) 优点
	- 快速、简单，对一个对象的操作的代码进行多线程同步
	- 不需要太多的代码操作

- (2) 缺点
	- 全局所有的同步代码块，都使用`同一个`互斥锁，那么所有的同步代码块都必须按照顺序一个一个执行完毕，就可能造成效率低下
	- 当某一个`@synchronized(){}`块的锁`无法解除`时，那么其他线程都无法获得锁，也就无法进去执行同步代码块中的代码了，这就是线程死锁的一种


如果说一个实体类对象会在多线程下进行操作，那么如果使用同步代码块的话类似如下:

```objc
@interface Car : NSObject 

@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *cid;
@property (nonatomic, assign) NSInteger price;

@end

@implementation Car {
    NSString *_name;
    NSString *_cid;
    NSInteger _price;
}

- (void)setName:(NSString *)name {
    @synchronized(self) {
        _name = [name copy];
    }
}

- (NSString *)name {
    @synchronized(self) {
        return _name;
    }
}

- (void)setCid:(NSString *)cid {
    @synchronized(self) {
        _cid = [cid copy];
    }
}

- (NSString *)cid {
    @synchronized(self) {
        return _cid;
    }
}

- (void)setPrice:(NSInteger)price {
    @synchronized(self) {
        _price = price;
    }
}

- (NSInteger)price {
    @synchronized(self) {
        return _price;
    }
}

...

@end
```

如果是这样的话，那么在多线程下操作上面的`_name`、`_cid`、`_price`这三个对象的实例变量时，就得全部按照顺序一个一个执行读写。

在多线程环境所有的`读`也需要按照顺序进行排队秩序，必须等待前面的所有`读`操作执行完毕，但很可能前面都是`读`，而并没有任何的`写`。

那么此时的读，其实应该优化成并发一起执行读操作，这样才会提高效率，但是对同步代码块是做不到的。

###改为使用 NSLock简单互斥锁、NSRecursiveLock递归锁，来完成上面的例子了？

```objc
static NSLock * Lock() {
    static NSLock *lock = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        lock = [[NSLock alloc] init];
    });
    return lock;
}

@implementation Car {
    NSString *_name;
    NSString *_cid;
    NSInteger _price;
}

- (void)setName:(NSString *)name {
//    @synchronized(self) {
//        _name = [name copy];
//    }
    
    [Lock() lock];
    _name = [name copy];
    [Lock() unlock];
}

- (NSString *)name {
//    @synchronized(self) {
//        return _name;
//    }
    
    [Lock() lock];
    NSString *name = _name;
    [Lock() unlock];
    return name;
}

- (void)setCid:(NSString *)cid {
//    @synchronized(self) {
//        _cid = [cid copy];
//    }
    
    [Lock() lock];
    _cid = [cid copy];
    [Lock() unlock];
}

- (NSString *)cid {
//    @synchronized(self) {
//        return _cid;
//    }
    
    [Lock() lock];
    NSString *cid = _cid;
    [Lock() unlock];
    return cid;
}

- (void)setPrice:(NSInteger)price {
//    @synchronized(self) {
//        _price = price;
//    }
    
    [Lock() lock];
    _price = price;
    [Lock() unlock];
}

- (NSInteger)price {
//    @synchronized(self) {
//        return _price;
//    }

    [Lock() lock];
    NSInteger price = _price;
    [Lock() unlock];
    return price;
}

...

@end
```

注意，比之前使用`@synchronized(self){}`同步代码块基础上，需要自己使用一个`静态单例NSLock对象`来进行实例变量的读写顺序控制。

而如果不使用一个`静态单例NSLock对象`，就可能造成这个`锁对象本身`在多个线程上`重复`执行`创建`出现野指针问题，所以必须`预先`创建出来一个锁对象，且只能被`创建一次`的锁对象。

在ViewController中测试上面的对象属性值在多线程环境下并发读写:

```objc
@implementation MultithreadViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

    Car *car = [Car new];
    car.name = @"car1";
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        car.name = @"car2";
    });
    
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        car.name = @"car3";
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"car.name = %@", car.name);
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        car.name = @"car4";
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"car.name = %@", car.name);
    });
}

@end
```

第一次运行结果

```
2016-07-31 00:45:39.683 Demos[5679:84228] car.name = car4
2016-07-31 00:45:39.683 Demos[5679:84227] car.name = car3
2016-07-31 00:45:39.684 Demos[5679:84227] <<Car: 0x7ff983629d40> - 0x7ff983629d40> name:car4 cid:(null) dealloc
```

第二次运行结果

```
2016-07-31 00:45:40.322 Demos[5679:84228] car.name = car3
2016-07-31 00:45:40.322 Demos[5679:84227] car.name = car4
2016-07-31 00:45:40.322 Demos[5679:84227] <<Car: 0x7ff9834b3760> - 0x7ff9834b3760> name:car4 cid:(null) dealloc
```

可以看到两次结果还是不一样的，而理想正确顺序应该是不管运行多少次都是如下:

```
car.name = car3
car.name = car4
```

但是上面代码时而正确时而错误，那么肯定是有出问题的。而问题就是虽然使用了NSLock对于实例变量的setter/getter方法在`某一次执行时`进行了顺序控制，但关键是在`多次的setter/getter方法调用任务之间`并没有做顺序控制。

- (1) 控制: 当进入到具体的setter/getter实现中，确实是必须等待完整执行完毕之后，才会让其他线程来执行

- (2) 未控制: 到底是 `第一个`线程的setter/getter调用 还是 `第二个`线程的setter/getter调用，哪一个先执行

并且`@synchronized(self){}`与NSLock这一类锁完成同步时，都有同样的缺点:

- (1) 所有的同步代码块，都是`共用同一个锁`，造成效率低
- (2) 一旦同步锁无法解除，导致其他排队等待的线程无法继续进入同步代码块执行
- (3) 可以控制`某一次调用的 完整性`，但无法控制`多次操作之间的 顺序`


###将上面所有的 读写操作 统统放入到一个 GCD 串行队列来保持顺序执行，解决上面的问题.


- (1) GCD第一种同步法、dispatch `async` serailQueue block >>> `读/写`都可以使用`async`，但如果需要`立刻`获取到结果值就使用`sync`来分配`读操作`

```objc
static dispatch_queue_t SerialQueue() {
    static dispatch_queue_t _queue = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _queue = dispatch_queue_create("com.Car.queue.serial", DISPATCH_QUEUE_SERIAL);
    });
    return _queue;
}

@interface MultithreadViewController ()

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    dispatch_async(SerialQueue(), ^{
        NSLog(@"A - Thread = %@", [NSThread currentThread]);
    });
    
    dispatch_async(SerialQueue(), ^{
        NSLog(@"B - Thread = %@", [NSThread currentThread]);
    });

    
    dispatch_async(SerialQueue(), ^{
        NSLog(@"C - Thread = %@", [NSThread currentThread]);
    });

    
    dispatch_async(SerialQueue(), ^{
        NSLog(@"D - Thread = %@", [NSThread currentThread]);
    });

    
    dispatch_async(SerialQueue(), ^{
        NSLog(@"E - Thread = %@", [NSThread currentThread]);
    });

}

@end
```

运行结果

```
2016-07-30 23:56:14.568 Demos[2837:33957] A - Thread = <NSThread: 0x7fad286368f0>{number = 2, name = (null)}
2016-07-30 23:56:14.568 Demos[2837:33957] B - Thread = <NSThread: 0x7fad286368f0>{number = 2, name = (null)}
2016-07-30 23:56:14.568 Demos[2837:33957] C - Thread = <NSThread: 0x7fad286368f0>{number = 2, name = (null)}
2016-07-30 23:56:14.569 Demos[2837:33957] D - Thread = <NSThread: 0x7fad286368f0>{number = 2, name = (null)}
2016-07-30 23:56:14.569 Demos[2837:33957] E - Thread = <NSThread: 0x7fad286368f0>{number = 2, name = (null)}
```

可以看到很轻松的就可以实现之前使用NSLock、@sychronized{}完成多线程顺序执行的效果，并注意`dispatch_async + serail_queue`只会创建`唯一一个线程`来一次按照顺序执行队列中调度的所有的Block。

使用此种方法其实很容易实现上面所有的问题，在外部使用一个串行队列来完成所有的同步问题，去掉Car对象内部所有的同步处理:

```objc
@interface Car : NSObject 

@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *cid;
@property (nonatomic, assign) NSInteger price;
@end

@implementation Car

- (void)dealloc {
    NSLog(@"<%@ - %p> name:%@ cid:%@ dealloc", self, self, _name, _cid);
}

@end
```

```objc
static dispatch_queue_t SerialQueue() {
    static dispatch_queue_t _queue = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _queue = dispatch_queue_create("com.Car.queue.serial", DISPATCH_QUEUE_SERIAL);
    });
    return _queue;
}

@implementation MultithreadViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

    Car *car = [Car new];
    car.name = @"car1";
    
    dispatch_async(SerialQueue(), ^{
        car.name = @"car2";
    });
    
    
    dispatch_async(SerialQueue(), ^{
        car.name = @"car3";
    });
    
    dispatch_async(SerialQueue(), ^{
        NSLog(@"car.name = %@", car.name);
    });
    
    dispatch_async(SerialQueue(), ^{
        car.name = @"car4";
    });
    
    dispatch_async(SerialQueue(), ^{
        NSLog(@"car.name = %@", car.name);
    });
}

@end
```

运行结果，不管点击n次都是如下结果

```
2016-07-31 01:01:46.333 Demos[6450:98095] car.name = car3
2016-07-31 01:01:46.333 Demos[6450:98095] car.name = car4
2016-07-31 01:01:46.333 Demos[6450:98095] <<Car: 0x7fb150c7f520> - 0x7fb150c7f520> name:car4 cid:(null) dealloc
```

> 注意，因为async是异步执行的，那么如果需要得到上一次的写操作结果数据，下一次任务也需要async到串行队列才能获取到数据。

但是，有时候可能修改Car对象的值，并不是处于同一个线程上。比如上面的测试代码中所有的async serail queue 都是在主线程上，那么如果是每一个不同的子线程去修改Car对象属性值了？

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

    Car *car = [Car new];
    car.name = @"car1";
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
       
        // 模拟在其他子线程计算得到修改的值
        NSString *tmp = @"car2";
        
        // 将操作Car对象逻辑，统一分配掉串行同步队列
        dispatch_async(SerialQueue(), ^{
            car.name = [tmp copy];
            NSLog(@"car.name = %@", car.name);
        });
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSString *tmp = @"car3";
        
        dispatch_async(SerialQueue(), ^{
            car.name = [tmp copy];
            NSLog(@"car.name = %@", car.name);
        });
    });

    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSString *tmp = @"car4";
        
        dispatch_async(SerialQueue(), ^{
            car.name = [tmp copy];
            NSLog(@"car.name = %@", car.name);
        });
    });
}
```

此时的运行结果就有很多种情况了:

```
2016-07-31 01:11:28.633 Demos[7233:108798] car.name = car2
2016-07-31 01:11:28.634 Demos[7233:108798] car.name = car3
2016-07-31 01:11:28.634 Demos[7233:108798] car.name = car4
```

```
2016-07-31 01:11:35.082 Demos[7233:108799] car.name = car2
2016-07-31 01:11:35.083 Demos[7233:108799] car.name = car4
2016-07-31 01:11:35.083 Demos[7233:108799] car.name = car3
```

```
2016-07-31 01:11:35.402 Demos[7233:108800] car.name = car3
2016-07-31 01:11:35.402 Demos[7233:108800] car.name = car4
2016-07-31 01:11:35.402 Demos[7233:108800] car.name = car2
```

这都是正确的结果，因为最外面分配了三个子线程来执行Car对象属性值修改的任务。而对于这外面的三个子线程，因为是并发无序执行，所以他们的执行顺序是无法控制的。

但是必须要控制三个子线程对Car对象属性值读写的顺序，如果出现下面的类似结果就是错误的:

```
car.name = car2
car.name = car2
car.name = car4
```

即出现两个一样的Car对象属性值，这就是错误的情况，造成原因就是属性值读写操作没有同步处理正确。

小结下，其实对于实体类对象内部是不需要任何的同步处理代码的，只需要在外部操作Car对象逻辑代码加上GCD串行队列控制就可以完成多线程的同步。

- (2) GCD第二种同步法、dispatch `sync` serailQueue block >>> 读写对象都使用`sync`调度

稍微修改上面的例子中使用`dispatch_sync`.

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    dispatch_sync(SerialQueue(), ^{
        NSLog(@"A - Thread = %@", [NSThread currentThread]);
    });
    
    dispatch_sync(SerialQueue(), ^{
        NSLog(@"B - Thread = %@", [NSThread currentThread]);
    });

    
    dispatch_sync(SerialQueue(), ^{
        NSLog(@"C - Thread = %@", [NSThread currentThread]);
    });

    
    dispatch_sync(SerialQueue(), ^{
        NSLog(@"D - Thread = %@", [NSThread currentThread]);
    });

    
    dispatch_sync(SerialQueue(), ^{
        NSLog(@"E - Thread = %@", [NSThread currentThread]);
    });

}
```

运行结果

```
2016-07-31 00:03:43.764 Demos[3266:41108] A - Thread = <NSThread: 0x7feef9c05440>{number = 1, name = main}
2016-07-31 00:03:43.764 Demos[3266:41108] B - Thread = <NSThread: 0x7feef9c05440>{number = 1, name = main}
2016-07-31 00:03:43.765 Demos[3266:41108] C - Thread = <NSThread: 0x7feef9c05440>{number = 1, name = main}
2016-07-31 00:03:43.765 Demos[3266:41108] D - Thread = <NSThread: 0x7feef9c05440>{number = 1, name = main}
2016-07-31 00:03:43.765 Demos[3266:41108] E - Thread = <NSThread: 0x7feef9c05440>{number = 1, name = main}
```

效果仍然是可以顺序执行多个线程，但是有一个很大区别。区别就是可以看到所有的Blcok都是在`主线程`上执行的，而并`不是在子线程`上。

那其实sync到任何一个队列效果都是一样，最终Block都是在`当前执行sync操作的线程上`完成，比如继续将上面ViewController中的touchBegin方法实现中改为全局并发队列进行sync:

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    dispatch_sync(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"A - Thread = %@", [NSThread currentThread]);
    });
    
    dispatch_sync(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"B - Thread = %@", [NSThread currentThread]);
    });

    
    dispatch_sync(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"C - Thread = %@", [NSThread currentThread]);
    });

    
    dispatch_sync(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"D - Thread = %@", [NSThread currentThread]);
    });

    
    dispatch_sync(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"E - Thread = %@", [NSThread currentThread]);
    });

}
```

运行结果

```
2016-07-31 00:12:32.366 Demos[3773:48806] A - Thread = <NSThread: 0x7feccb5029e0>{number = 1, name = main}
2016-07-31 00:12:32.367 Demos[3773:48806] B - Thread = <NSThread: 0x7feccb5029e0>{number = 1, name = main}
2016-07-31 00:12:32.367 Demos[3773:48806] C - Thread = <NSThread: 0x7feccb5029e0>{number = 1, name = main}
2016-07-31 00:12:32.367 Demos[3773:48806] D - Thread = <NSThread: 0x7feccb5029e0>{number = 1, name = main}
2016-07-31 00:12:32.367 Demos[3773:48806] E - Thread = <NSThread: 0x7feccb5029e0>{number = 1, name = main}
```

可以看到仍然是可以控制多线程按照顺序执行的，并仍然处于主线程完成所有的Block。

再尝试将所有的sync操作，从主线程转移到另一个子线程上去完成:

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        
        dispatch_sync(dispatch_get_global_queue(0, 0), ^{
            NSLog(@"A - Thread = %@", [NSThread currentThread]);
        });
        
        dispatch_sync(dispatch_get_global_queue(0, 0), ^{
            NSLog(@"B - Thread = %@", [NSThread currentThread]);
        });
        
        
        dispatch_sync(dispatch_get_global_queue(0, 0), ^{
            NSLog(@"C - Thread = %@", [NSThread currentThread]);
        });
        
        
        dispatch_sync(dispatch_get_global_queue(0, 0), ^{
            NSLog(@"D - Thread = %@", [NSThread currentThread]);
        });
        
        
        dispatch_sync(dispatch_get_global_queue(0, 0), ^{
            NSLog(@"E - Thread = %@", [NSThread currentThread]);
        });
    });
}
```

运行结果

```
2016-07-31 00:24:19.385 Demos[4425:59380] A - Thread = <NSThread: 0x7fb9a9c1dd90>{number = 2, name = (null)}
2016-07-31 00:24:19.385 Demos[4425:59380] B - Thread = <NSThread: 0x7fb9a9c1dd90>{number = 2, name = (null)}
2016-07-31 00:24:19.385 Demos[4425:59380] C - Thread = <NSThread: 0x7fb9a9c1dd90>{number = 2, name = (null)}
2016-07-31 00:24:19.386 Demos[4425:59380] D - Thread = <NSThread: 0x7fb9a9c1dd90>{number = 2, name = (null)}
2016-07-31 00:24:19.386 Demos[4425:59380] E - Thread = <NSThread: 0x7fb9a9c1dd90>{number = 2, name = (null)}
```

可以得到`dispatch_sync(){}`如下几点:

- (1) `dispatch_sync()`只有当一个block任务执行完毕之后，才会去执行另外一个block任务
- (2) 所有的Block都是在当前子线程上执行，即不会新开子线程

那么sync完成多线程同步的优点:

- (1) 立刻能够得到Block执行完之后的`返回值`
	- async因为是异步所有立马获取不到返回值
- (2) 不会对Block进行拷贝
	- async会对Block对象拷贝到堆去

而sync完成多线程同步相对async的缺点:

- (1) Block是在当前sync操作现场执行的，如果对于一些耗时cpu的代码就不太适合

针对上面使用async完成效果，此处使用sync来完成:

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    Car *car = [Car new];
    
    // 为了不让在当前主线程执行
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        
        // 立刻同步执行修改Car对象数据
        dispatch_sync(dispatch_get_global_queue(0, 0), ^{
            car.name = @"car1";
        });
        
        // 然后立刻执行读取Car对象修改后的数据
        NSLog(@"car.name = %@, thread = %@", car.name, [NSThread currentThread]);
        
        dispatch_sync(dispatch_get_global_queue(0, 0), ^{
            car.name = @"car2";
        });
        
        NSLog(@"car.name = %@, thread = %@", car.name, [NSThread currentThread]);
        
        dispatch_sync(dispatch_get_global_queue(0, 0), ^{
            car.name = @"car3";
        });
        
        NSLog(@"car.name = %@, thread = %@", car.name, [NSThread currentThread]);
    });
}
```

不论点击多少次都是如下输出结果

```
2016-07-31 01:27:17.878 Demos[8010:122723] car.name = car1, thread = <NSThread: 0x7fa6e3e347c0>{number = 3, name = (null)}
2016-07-31 01:27:17.878 Demos[8010:122723] car.name = car2, thread = <NSThread: 0x7fa6e3e347c0>{number = 3, name = (null)}
2016-07-31 01:27:17.878 Demos[8010:122723] car.name = car3, thread = <NSThread: 0x7fa6e3e347c0>{number = 3, name = (null)}
2016-07-31 01:27:17.878 Demos[8010:122723] <<Car: 0x7fa6e3c62670> - 0x7fa6e3c62670> name:car3 cid:(null) dealloc
```

使用sync的话，立刻就可以得到修改后的数据。

- (3) 使用 async（异步派发block） + concurrnt queue（并发队列）+ barrier栅栏 >>> `写`操作使用 `dispatch_barrier_async`调度，`读`操作`dispatch_sync`调度。

barrier 栅栏 在并发队列派发执行block任务的作用

- 首先等待栅栏之前的所有并发block任务执行完毕
- 单独执行栅栏block块
- 等待栅栏block块执行完毕之后，才会继续之后后面的并发block任务

![](http://i13.tietuku.cn/8e4ba1744d51a4ab.png)

```objc
static dispatch_queue_t ConcurrentQueue() {
    static dispatch_queue_t _queue = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _queue = dispatch_queue_create("com.Car.queue.serial", DISPATCH_QUEUE_CONCURRENT);
    });
    return _queue;
}

@implementation MultithreadViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

    Car *car = [Car new];
    car.name = @"car1";

    // 异步子线程任务1
    dispatch_async(dispatch_get_global_queue(0, 0), ^{

        // 模拟在其他子线程计算得到修改的值
        NSString *tmp = @"car2";

        // 将操作Car对象逻辑，统一分配掉串行同步队列
        dispatch_barrier_async(ConcurrentQueue(), ^{
            car.name = [tmp copy];
        });
    });
    
    dispatch_sync(ConcurrentQueue(), ^{
        NSLog(@"car.name = %@", car.name);
    });

    // 异步子线程任务2
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSString *tmp = @"car3";

        dispatch_barrier_async(ConcurrentQueue(), ^{
            car.name = [tmp copy];
        });
    });
    
    dispatch_sync(ConcurrentQueue(), ^{
        NSLog(@"car.name = %@", car.name);
    });
}

@end
```

运行多少次都是如下结果

```
2016-07-31 02:13:53.218 Demos[10298:154039] car.name = car1
2016-07-31 02:13:53.219 Demos[10298:154039] car.name = car2
2016-07-31 02:13:53.219 Demos[10298:154077] <<Car: 0x7fc563417dd0> - 0x7fc563417dd0> name:car3 cid:(null) dealloc
```

用此种方式完成同步书上说效率会更高，因为他可以让很多个读操作并发执行，但是对于某一个写操作确实单独独立完成。

然后最后使用一个简单的缓存类进行再多线程环境下同步处理模板:

- (1) 首先看没有使用多线程同步的代码下，运行结果会导致程序崩溃

```objc
@interface Context : NSObject

- (void)setObject:(id)value forKey:(NSString *)key;
- (id)objectFirKey:(NSString *)key;

@end

@implementation Context {
    NSMutableDictionary *_dic;
}

- (void)setObject:(id)value forKey:(NSString *)key {
    if (!_dic) _dic = [NSMutableDictionary new];
    [_dic setObject:value forKey:key];
    NSLog(@"_dic = %@", _dic);
}

- (id)objectFirKey:(NSString *)key {
    if (!_dic) _dic = [NSMutableDictionary new];
    return [_dic objectForKey:key];
}

@end
```

```objc
@interface MultithreadViewController () {
    Context *_ctx;
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    _ctx = [Context new];
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (NSInteger i = 0; i < 1000; i++) {
            [_ctx setObject:@"1111" forKey:@"1111"];
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (NSInteger i = 0; i < 1000; i++) {
            [_ctx setObject:@"2222" forKey:@"2222"];
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (NSInteger i = 0; i < 1000; i++) {
            [_ctx setObject:@"3333" forKey:@"3333"];
        }
    });
}

@end
```

如上程序时而崩溃，时而不蹦，`就看CPU切换线程是否等待线程执行完`，但是存在很大的线程危险。


对上述Context类加上多线程同步处理的代码:

```objc
@interface Car : NSObject
@property (nonatomic, copy) NSString *name;
@end
@implementation Car
@end
```

```objc
static dispatch_queue_t ConcurrentQueue() {
    static dispatch_queue_t _queue = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _queue = dispatch_queue_create("com.Car.queue", DISPATCH_QUEUE_CONCURRENT);
    });
    return _queue;
}
```

```objc
@interface Context : NSObject

- (void)setObject:(id)value forKey:(NSString *)key;
- (id)objectForKey:(NSString *)key;

@end

@implementation Context {
    NSMutableDictionary *_dic;
}

- (void)setObject:(id)value forKey:(NSString *)key {
    dispatch_barrier_async(ConcurrentQueue(), ^{
        if (!_dic) _dic = [NSMutableDictionary new];
        [_dic setValue:value forKey:key];
//        NSLog(@"_dic = %@", _dic);
    });
}

/**
 *  同步读取key对应的value
 */
- (id)objectForKey:(NSString *)key {
    __block id ret = nil;
    dispatch_sync(ConcurrentQueue(), ^{//在执行objectFirKey:的线程上执行完使用 _dic的代码，并阻塞后面所有的线程执行
        if (!_dic) _dic = [NSMutableDictionary new];
        ret = [_dic objectForKey:key];
    });
    return ret;
}

/**
 *  异步读取key对应的value
 */
- (void)objectForKey:(NSString *)key compltionBlock:(void (^)(id value))block {
    dispatch_async(ConcurrentQueue(), ^{
        if (!_dic) _dic = [NSMutableDictionary new];
        id ret = [_dic objectForKey:key];
        if (block) {block(ret);}
    });
}

@end
```

测试代码

```objc
@interface ViewController () {
    Context *_ctx;
}

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    _ctx = [[Context alloc] init];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    // 同步写入
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (NSInteger i = 0; i < 1000; i++) {
            [_ctx setObject:@"1111" forKey:@"1111"];
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (NSInteger i = 0; i < 1000; i++) {
            [_ctx setObject:@"2222" forKey:@"2222"];
        }
    });
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (NSInteger i = 0; i < 1000; i++) {
            [_ctx setObject:@"3333" forKey:@"3333"];
        }
    });
    
    // 同步读取
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (NSInteger i = 0; i < 100; i++) {
            NSLog(@"%@ - %@", @"1111", [_ctx objectForKey:@"1111"]);
            NSLog(@"%@ - %@", @"2222", [_ctx objectForKey:@"2222"]);
            NSLog(@"%@ - %@", @"3333", [_ctx objectForKey:@"3333"]);
        };
    });
    
    // 异步读取
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (NSInteger i = 0; i < 100; i++) {
            [_ctx objectForKey:@"1111" compltionBlock:^(id value) {
                NSLog(@"1111 - %@", value);
            }];
            [_ctx objectForKey:@"2222" compltionBlock:^(id value) {
                NSLog(@"2222 - %@", value);
            }];
            [_ctx objectForKey:@"3333" compltionBlock:^(id value) {
                NSLog(@"33333 - %@", value);
            }];
        };
    });
}

@end
```

无论点击多少次运行，都是正常运行，结果也正确输出。

###最后对上面GCD队列同步多线程使用情况小结:

- (1) `dispatch_async` + `serial queue` + `读操作/写操作`
- (2) `dispatch_sync` + `serial/concurrent queue` + `读操作/写操作`
- (3) `dispatch_barrier_async 写操作` + `dispatch_sync 读操作`/`dispatch_async 读操作` + `concurrent queue`

建议使用第三种，因为第三种可以`并发`执行`多个读操作`，而只有 `同步顺序`执行`一个写操作`。


##构建缓存时选用`NSCache`而不是NSDictionary

- 当iOS系统资源紧张时，会自动删除NSCache的所有缓存资源
- NSCache会自动使用`LRU（最近最久未被使用）`缓存项淘汰算法
- NSCache对key不是copy，而是保留retain
	- 因为很多时候，传入的key，可能`没有`实现NSCopying协议
- NSCache是线程安全的，使用NSCache只需关心set与get，不用自己去写任何加锁同步代码
- NSCache提供两个维度管理缓存:
	- 一、缓存最大占用空间 `Cost` Limit
	- 二、缓存最大占用长度 `Count` Limit
- 可以考虑使用NSMutableData的子类`NSPurgeableData`来代替，存入到NSCache对象中
	- 因为当一个`NSPurgeableData对象`被废弃时，NSCache会自动清除对应的缓存

但是看了很多的对象缓存框架，TMCache、YYCache都是没有直接使用NSCache，而是自己实现的Cache方式，所以可能NSCache性能并不是太好且定制线不够高。

##initialize方法与load方法之间的区别

> load方法、`+[NSObject load]`: 最好不要使用load方法

- (1) 执行该方法时，运行时系统处于`脆弱的状态`

- (2) **应用程序启动时，首先将工程中`所有类`的load方法`都`执行完毕，才会继续做下面的事情**
	- load方法中，不应该做过多耗时的事情，会影响类的加载速度

- (3) 先执行`父类的load`方法实现，然后执行`当前类`的load方法

- (4) 在当前类的load方法中，调用`其他类的函数实现`是不安全的:
	- 因为搞不清楚到底`哪个类的load方法实现`会被`先`执行
	- 如果顺序出现问题，就会导致某个类的`一些辅助变量未被初始化`导致程序崩溃

- (5) load方法不遵守`继承规则`。也就是说，一个类写了load实现就调用，如没写load实现就不调用，即不会查找父类中load实现.

- (6) 当`类`也写了load方法，`分类`也写了load方法，那么有调用顺序:
	- 先调用`类`中的load方法
	- 再调用`分类`中的load方法


> initialize方法 `+[NSObject initialize]`: 可以完成一些`全局状态的变量`初始化，但是不到万不得已仍然不要使用此方法完成一些任务

- (1) 在`第一次使用该类`的时候执行initialize，且`只会执行一次`。在运行时由系统调用，不能人为的手动调用。

- (2) `只有这个类被使用（加载）的时刻`才会去执行initialize，否则永远都不会被执行

- (3) initialize方法执行时，运行期系统处于`正常状态`
	- 也就是说可以随意的调用`其他类`的方法实现

- (4) initialize方法是绝对处于`线程安全`的环境下执行
	- 在当前initialize方法中，操作的`其他类`或`对象`都是处于线程阻塞下执行，不会出现多个线程同时执行initialize方法实现
	- 一直等待initialize方法方法对这些类或对象操作完毕，才会放行其他的initialize方法实现

- (5) 当前类如果没有写initialize方法，就会去`父类`查找执行initialize方法实现，也即是说`具备继承规则`

- (6) initialize方法中，最好不要调用其他类的方法，甚至当前类自己的方法

##看到一个实现单例的最佳写法，来源于苹果官方文档建议写法，保证copy拷贝出来的仍然是单例对象.

单例类

```objc
#import <Foundation/Foundation.h>

@interface Tool : NSObject <NSCopying>

+ (instancetype)sharedInstance;

@end
```

```objc
#import "Tool.h"

static id _instance;

@implementation Tool

+ (instancetype)sharedInstance {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _instance = [[self alloc] init];
    });
    return _instance;
}

+ (instancetype)allocWithZone:(struct _NSZone *)zone {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _instance = [super allocWithZone:zone];
    });
    return _instance;
}

- (id)copyWithZone:(NSZone *)zone {
    return _instance;
}

@end
```

- alloc 会调用 allocWithZone:
- copy拷贝直接返回 static静态单例对象

测试一下

```objc
@implementation ViewController

- (void)testSharedInstance {
    
    /**
     如果三种创建对象，最终都是走 allocWithZone: 消息
     */
    id ins1 = [[Tool alloc] init];     // >>> 回调用 allocWithZone:
    id ins2 = [[Tool allocWithZone:NULL] init]; // allocWithZone:可以不用关心zone参数
    id ins3 = [Tool sharedInstance];    // [self alloc] >>> [self allocWithZone:]

    id ins4 = [ins1 copy];
    id ins5 = [ins2 copy];
    id ins6 = [ins3 copy];
    
    NSLog(@"ins1: %p", ins1);
    NSLog(@"ins2: %p", ins2);
    NSLog(@"ins3: %p", ins3);
    NSLog(@"ins4: %p", ins4);
    NSLog(@"ins5: %p", ins5);
    NSLog(@"ins6: %p", ins6);
}

@end
```


输出如下

```
ins1: 0x7fd949e30120
ins2: 0x7fd949e30120
ins3: 0x7fd949e30120
ins4: 0x7fd949e30120
ins5: 0x7fd949e30120
ins6: 0x7fd949e30120
```

可以看到全是同一个对象.

####可以将上述单例代码写成通用的宏定义，来在需要写成单例类中使用宏定义即可完成上面Tool类中的那些单例代码.

首先是，在单列类.h中用来声明sharedXxxIntsance方法的宏所在文件SingletonInterface.h

```c
#ifndef SingletonInterface_h
#define SingletonInterface_h

#define singleton_interface(class) \
+ (instancetype)shared##class##Instance;

#endif /* SingletonInterface_h */
```

然后是在单例类.m中实现单例的三个函数（allocWithZone:、copyWithZone:、sharedXxxInstance）的宏代码替换，SingletonImplement.h

```c
#ifndef SingletonImplement_h
#define SingletonImplement_h

#define singleton_implementation(class) \
static id _GlobalInstance; \
+ (instancetype)shared##class##Instance { \
    static dispatch_once_t onceToken; \
    dispatch_once(&onceToken, ^{ \
    _GlobalInstance = [[self alloc] init];  \
    });\
    return _GlobalInstance; \
}\
+ (instancetype)allocWithZone:(struct _NSZone *)zone { \
    static dispatch_once_t onceToken;   \
    dispatch_once(&onceToken, ^{    \
        _GlobalInstance = [super allocWithZone:zone]; \
    }); \
    return _GlobalInstance; \
}\
- (id)copyWithZone:(NSZone *)zone { \
    return _GlobalInstance;   \
}\

#endif /* SingletonImplement_h */
```

测试Dog类

```objc
#import <Foundation/Foundation.h>

//单例方法声明
#import "SingletonInterface.h"

@interface Dog : NSObject

//声明方法
singleton_interface(Dog)

@end
```

```objc
#import "Dog.h"

//单例方法实现
#import "SingletonImplement.h"

@implementation Dog

//粘贴实现单例的三个方法的实现
singleton_implementation(Dog);

@end
```

##NSTimer对象会retain目标对象，并且不会自动对目标对象执行release，就会造成目标对象内存泄露

- NSTimer事件回调处理对象

```objc
@interface TimerCallBackObject : NSObject

- (void)callback;

@end

@implementation TimerCallBackObject

- (void)callback {
    NSLog(@"NSTimer 回调处理对象");
}

- (void)dealloc {
    NSLog(@"TimerCallBackObject %p is dealloc", self);
}

@end
```

- (2) ViewController对象中测试NSTimer持有上面的局部对象

```objc
@implementation RetainCycleViewController {
    NSTimer *_timer;
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    //1.
    TimerCallBackObject *callback = [TimerCallBackObject new];
    
    //2.
    _timer = [NSTimer timerWithTimeInterval:2
                                     target:callback
                                   selector:@selector(callback)
                                   userInfo:nil
                                    repeats:YES];
    
    //3. 注册Timer到一个线程的runloop
    NSRunLoop *runloop = [NSRunLoop currentRunLoop];
    [runloop addTimer:_timer forMode:NSRunLoopCommonModes];
}

@end
```

程序运行之后，就是在不断的每隔两秒执行`TimerCallBackObject`对象的`callback`方法实现。

但是有没有发现，在`touchesBegan:withEvent:`方法实现中，创建的是一个`局部的TimerCallBackObject对象`，并没有使用实例变量去持有，但是一直没有看到输出该对象的dealloc打印信息。

那么说明，这个局部的TimerCallBackObject对象一直被NSTimer对象strong持有。因为NSTimer事件一直没有停止，所以一直需要回调这个TimerCallBackObject对象，所以要一直strong持有这个局部的TimerCallBackObject对象。

OK，那么模拟在某个时刻不再需要NSTiemr时，禁止NSTimer运行，看能否让NStimer持有的对象进行释放:

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    //1.
    TimerCallBackObject *callback = [TimerCallBackObject new];
    
    //2.
    _timer = [NSTimer timerWithTimeInterval:2
                                     target:callback
                                   selector:@selector(callback)
                                   userInfo:nil
                                    repeats:YES];
    
    //3. 注册Timer到一个线程的runloop
    NSRunLoop *runloop = [NSRunLoop currentRunLoop];
    [runloop addTimer:_timer forMode:NSRunLoopCommonModes];
    
    //4. 在10秒后停止NSTimer
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(10 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        [_timer invalidate];
        _timer = nil;
    });
}
```

运行结果

```
2016-07-31 15:58:57.196 Demos[16232:236183] NSTimer 回调处理对象
2016-07-31 15:58:59.191 Demos[16232:236183] NSTimer 回调处理对象
2016-07-31 15:59:01.196 Demos[16232:236183] NSTimer 回调处理对象
2016-07-31 15:59:03.196 Demos[16232:236183] NSTimer 回调处理对象
2016-07-31 15:59:05.191 Demos[16232:236183] NSTimer 回调处理对象
2016-07-31 15:59:05.192 Demos[16232:236183] TimerCallBackObject 0x7fc87bf4e380 is dealloc
```

可以看到被NSTimer对象strong持有的局部对象此时可以正常废弃了，所以一定要对NSTimer对象执行`invalidate`操作。


##使用offsetof访问struct结构体中成员的偏移量

```c
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)

struct student
{
    char gender;//offset = 0
    int id;//offset = 4
    int age;//offset = 8
    char name[20];//offset= 12
};

int main() {
//    LinkListDemo();
    int gender_offset, id_offset, age_offset, name_offset;
    gender_offset = offsetof(struct student, gender);
    id_offset = offsetof(struct student, id);
    age_offset = offsetof(struct student, age);
    name_offset = offsetof(struct student, name);
    printf("gender_offset = %d\n", gender_offset);
    printf("id_offset = %d\n", id_offset);
    printf("age_offset = %d\n", age_offset);
    printf("name_offset = %d\n", name_offset);
    return 1;
}
```

运行结果

```
gender_offset = 0
id_offset = 4
age_offset = 8
name_offset = 12
Program ended with exit code: 1
```


获得结构体(TYPE)的变量成员(MEMBER)在此结构体中的偏移量。

```
(01)  ( (TYPE *)0 )   将零转型为TYPE类型指针，即TYPE类型的指针的地址是0。
(02)  ((TYPE *)0)->MEMBER     访问结构中的数据成员。
(03)  &( ( (TYPE *)0 )->MEMBER )     取出数据成员的地址。由于TYPE的地址是0，这里获取到的地址就是相对MEMBER在TYPE中的偏移。
(04)  (size_t)(&(((TYPE*)0)->MEMBER))     结果转换类型。对于32位系统而言，size_t是unsigned int类型；对于64位系统而言，size_t是unsigned long类型。
```

## respondsToSelector 既可以判断类是否实现方法也可以判断对象是否实现方法

```objc
@implementation ViewController

+ (void)classMethod {
    
}

- (void)instanceMethod {
    
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

	BOOL flag1 = [self respondsToSelector:@selector(instanceMethod)];//YES
    BOOL flag2 = [[self class] respondsToSelector:@selector(instanceMethod)];//NO
    BOOL flag3 = [self respondsToSelector:@selector(classMethod)];//NO
    BOOL flag4 = [[self class] respondsToSelector:@selector(classMethod)];//YES
}

@end
```

输出

```
(BOOL) flag1 = YES
(BOOL) flag2 = NO
(BOOL) flag3 = NO
(BOOL) flag4 = YES
```

结果是正确的。

当`[self respondsToSelector:]`时，是向`self->isa`指向的`objc_class`实例中查询IMP。

而当`[[self class] respondsToSelector:]`时，是向`[self class]->isa`指向的`MetaClass`对应的`objc_class`实例中查询IMP。

所以既可以判断类是否实现方法也可以判断对象是否实现方法。
