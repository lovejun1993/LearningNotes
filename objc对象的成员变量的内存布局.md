## objc对象/c struct实例中，实例变量/成员变量的`内存布局` 规则:


```objc
@interface Cat : NSObject {

    @public
    NSString *_name;
    NSString *_type;
    
    @private
    NSString *_cid;
}
@end
@implementation Cat
@end
```

如上写法定义的实例变量的`地址布局`，在程序代码`编译期间`就已经确定了。那么如果运行时去修改这个布局，就会导致Ivar地址错乱，出现存取数据错误。


## 每个 objc 对象都有相同的结构

| Objective-C 对象的结构图 |
| :-------------: 
| `isa指针变量` | 
| `Root类` 所有的实例变量的排布 |
| `倒数第二层父类` 所有的实例变量的排布 |
| `倒数第三层父类` 所有的实例变量的排布 |
| .... |
| `直接父类` 所有的实例变量的排布 |
| `当前类` 所有的实例变量的排布 |

`从上到下`，从Root根类的实例变量开始排布。

## objc对象中所有的实例变量的内存布局规则:

- (1) 如上写法定义的实例变量的`地址布局`，在程序代码`编译期间`就已经确定了。

- (2) 如果`运行时`去修改这个布局，就会导致Ivar地址错乱，出现存取数据错误。

- (3) 对象的所有实例变量的布局计算规律是，按照实例变量的定义`从上到下`的顺序
	- 每一个实例变量的布局起始地址，排列在上一个实例变量地址的`末尾`
	- 每一个实例变量占用的`内存长度`，就是是自己数据类型的占用总长度
	- 每一个实例变量的`偏移量offset`，都是相对于`所属对象`所在内存块的`起始地址`
	- `objc类对象`和`struct实例`的实例变量的偏移量计算，是非常相似的，也存在`字节对齐`的问题

## 现代计算机都是使用`字节`进行内存地址分配，也就存在`字节对齐`的问题

结构体中的元素布局（字节对齐），如下两个结构体，元素都是一样的，以32位系统为例.

```c
struct A {
    int a;      // 占4个字节
    char b;     // 占1个字节
    short c;    // 占2个字节
};
```

```c
struct B {
    char a;     // 占1个字节
    int b;      // 占4个字节
    short c;    // 占2个字节
};
```

看起来的话两个结构体的总长度都是7个字节，但是如下打印却是:

```c
int main() {
    
    printf("%lu\n", sizeof(struct A));
    printf("%lu\n", sizeof(struct B));
    
    return 0;
}
```

```
8
12
Program ended with exit code: 0
```

长度分别是8和12，那么为什么不是7了？原因是`字节对齐`的规则。

现代计算机内存单元，都是使用`字节（代替8个二进制位）`作为基本单位划分。理论上说可以从任何的内存地址来访问到变量的所在内存单元。

但是实际上，为了提升效率，各种类型的数据地址是按照`一定的规律来有序排列`，那么所以对于某一些类型的数据访问，可以直接从特定的位置上开始访问，就不用从头开始遍历寻找。

牺牲空间来换取时间，这就是字节对齐。

## struct的字节对齐的规则

- (1) 结构体变量的`首地址`能够被该结构体中`最大长度`的成员变量的长度所`整除`.
	- 我们写的程序是控制不了内存分配到哪一个地址的，所以这一条pass

- (2) 结构体变量中每个成员变量相对于`struct实例空间的首地址`的偏移量（offset）都是成员变量自身长度的`整数倍`.
	- 成员变量的偏移量，不够时由编译器自动填充

- (3) 结构体变量的总长度为成员变量中`最大长度`的`整数倍`.
	- 结构体的总长度，不够时由编译器自动填充

## 那么对于上面的`结构体A变量`的内存字节布局是这样的:

- 首先变量a是int类型，占用4个字节
	- 第一个变量直接放
- 然后变量b是char类型，占用1个字节
	- 相对于起始地址的偏移量 = 4，满足规则2
	- 所以直接从第五个字节开始存放b，并占用一个字节长度
- 最后变量c是short类型，占用2个字节
	- 此时相对于起始地址的偏移量 = 5，`不满足规则2`
	- 所以向后填充一个字节，这个填充字节什么事都不干，就是为了字节对齐
- 所以最终长度 = 8个字节


## 那么对于`结构体变量B`的内存字节布局是这样的:

- 首先变量a是char类型，占用1个字节
	- 第一个变量直接放
- 然后变量b是int类型，占用4个字节
	- 此时相对于起始地址的偏移量 = 1，`不满足规则2`
	- 为了让b相对起始地址的偏移量是int长度（4）的整数倍
	- 所以修改为`偏移量 = 4`，就是说`空出三个`没用的字节，什么都不干，就是为了字节对齐
	- 从第5个字节开始排布变量b，并占用4个字节长度
	- 此时总长度为8个字节
- 最后变量c是short类型，占用2个字节
	- 此时相对于起始地址的偏移量 = 8，`满足规则2`，直接排布c
	- c此时占用2个字节长度
	- 那么此时结构体总长度 = 10，`不满足规则3`
- 为了让结构体总长度是成员最大长度的整数倍
	- 最大成员长度是b = 4
	- 所以最后扩展为长度 = 12，刚好大于10又是4的整数倍
- 所以最终的结构体B的长度是12

## 对于上面的两个结构体可以使用`保留字节`来避免编译器自动填充字节作为无用字节:

```c
struct A {
    int a;          	// 占4个字节
    char b;         	// 占1个字节
    char reserved;  	// 此处会由编译器填充1个字节，那么我们可以定义以后使用的保留变量来操作
    short c;        	// 占2个字节
};

struct B {
    char a;             // 占1个字节
    char reserved1[3];  // 此处会由编译器填充3个字节，那么我们可以定义以后使用的保留变量来操作
    int c;              // 占4个字节
    short b;            // 占2个字节
    char reserved2[2];  // 此处会由编译器填充2个字节，那么我们可以定义以后使用的保留变量来操作
};
```

对于上面使用reserved保留变量声明的内存字节，我们也可以去使用了。

而如果我们不使用reserved保留变量去指向这些字节，那么这些字节就相当于放在那什么也不干就是为了字节对齐，什么用都没有。

## 对于struct成员变量从上到下排布的原则:

- (1) 成员变量的排布按照自身的长度`从小到大`依次排列
- (2) 尽量的使用`保留字段`，来使用由编译器进行字节对齐时填充无用的字节

那么对上面两个结构体按照如上原则最后的修改版本为如下:

```c
struct C {
    char a;             // 占1个字节
    char reserved;      // 此处会由编译器填充1个字节，那么我们可以定义以后使用的保留变量来操作
    short b;            // 占2个字节
    int c;              // 占4个字节
};
```

那么，其实也是适用于objc对象的实例变量的排布规则的。

## 再回头看看Cat对象的所有实例变量的内存地址布局如下所示:

| 属性相对Cat对象起始地址的偏移量offset | Cat对象的实例变量 | 
| :-------------: |:-------------:| 
| +0 字节 | `_firstName` | 
| +4 字节 | `_lastName` | 
| +8 字节 | `_pid` | 

上面所示的实例变量的内存布局在代码`编译期间`就已经确定了。

如果是新增加一个实例变量或者减少一个实例变量，就必须要`重新编译`程序代码，让编译器重新计算所有实例变量的内存布局，否则就会出现地址错乱。

## 对于`Cat对象->_firstName`这句代码实际作用:

- (1) 首先找到当前`Cat对象`的所在内存的`起始地址`
- (2) 从某个全局记录表中，查询到`_firstName`这个Ivar对应的地址`偏移量offset`
- (3) 通过 `偏移量offset + Cat对象内存起始地址` 作为存取 实例变量 `_firstName` 的所在内存地址

那么如果在`程序运行`期间，通过`objc/runtime.h`中提供的运行时api来给Cat对象添加一个新的实例变量Ivar或者移除某一个已有的Ivar，会引出问题吗？ 肯定会的。

```objc
@interface Cat : NSObject {

    @public
    NSString *_runtimeAddIvar;//假设这个Ivar是在运行时添加
    
    NSString *_firstName;
    NSString *_lastName;
    
    @private
    NSString *_pid;
}

@end
```

如果运行时将Cat对象添加一个新的实例变量`_runtimeAddIvar`，那么此时Dog对象所有实例变量的内存布局应该是如下这样:

| 属性相对Cat对象起始地址的偏移量 | Cat对象的实例变量 | 
| :-------------: |:-------------:| 
| +0 | `_runtimeAddIvar` | 
| +4 | `_firstName ` | 
| +8 | `_lastName ` | 
| +12 | `_pid` | 


`如果系统没有更新保存实例变量对应的内存布局地址`，就会导致如下错误:

- (1) `Dog对象->_runtimeAddIvar`访问实例变量，实际上访问到了新添加的实例变量`_runtimeAddIvar`

- (2) `Dog对象->_firstName`访问实例变量，实际上访问到了新添加的实例变量`_lastName`

- (3) `Dog对象->_lastName`访问实例变量，实际上访问到了新添加的实例变量`_pid`

所以如果记录实例变量对应内存地址布局的全局配置中，没有得到及时的更新的话，就会造成实例变量的访问全部乱套。

## 系统会不会根据Class改变后自动进行调整布局吗？

OK，对如上的Cat类在运行时尝试添加Ivar看看。

```objc
- (void)demo1 {
    
    Cat *c = [Cat new];
    
    if(class_addIvar(objc_getClass("Cat"), "runtimeAddIvar", sizeof(NSString *), 0, "@")) {
        NSLog(@"add ivar success");
        
        c->_firstName = @"Li";
        c->_lastName = @"Ming";
        [c setValue:@"hello world" forKey:@"runtimeAddIvar"];
        
        NSLog(@"_firstName = %@", c->_firstName);
        NSLog(@"_lastName = %@", c->_lastName);
        NSLog(@"_runtimeAddIvar = %@", [c valueForKey:@"runtimeAddIvar"]);
        
    } else {
        NSLog(@"add ivar failed");
    }
}
```

我试了很久`class_addIvar()`方法返回值一直是`NO`，结果表示添加Ivar一直是失败的。

我开始一直以为`class_addIvar()`方法中的参数有问题，但是尝试了很多次参数修改，就是无法添加成功。

我就很纳闷了，以前都可以添加成功的，为啥现在不行了？于是我打开我之前记录的添加Ivar成功的代码如下:

```objc
- (void)demo2 {
    
    // 运行时创建一个类
    Class MyClass = objc_allocateClassPair([NSObject class], "myclass", 0);
    
    // 添加一个NSString的实例变量，第四个参数是对其方式，第五个参数是参数类型编码
    if(class_addIvar(MyClass, "itest", sizeof(NSString *), 0, "@")) {
        NSLog(@"add ivar success");
    } else {
        NSLog(@"add ivar failed");
    }
    
    // 向运行时系统注册创建的类
    objc_registerClassPair(MyClass);
}
```

运行结果

```
2016-07-17 17:17:34.550 Demo[14993:239703] add ivar success
```

确实可以啊，为啥上面的demo1方法中就死添加不成功了，我对比两种添加的方式对比了好久好久，没有发现哪里不一样。但是突然看到了有一个很大的区别:

- (1) 前面的`Cat`类，是我在Xcode工程中添加的存在源文件的类

- (2) 后面的`MyClass`类，在代码编译期间是不存在的，只有程序运行期执行了`objc_allocateClassPair()`函数之后才会出现的类

我觉得这个区别就是导致`Cat`类不能添加Ivar的原因。

于是，我在网上搜是否有这样的记录。于是在`http://www.cocoachina.com/ios/20141031/10105.html`中找到了答案:

```
- (1) Objective-C 不支持 往`已存在`的类中添加实例变量
	- 系统库中，已经存在的源码文件的类
	- 我们自己Xcode工程中，已经存在源码文件的类
	- 我理解`已存在`的意思是 >>> 已经存在源码文件定义的类

- (2) 只能向还没有存在（运行时动态创建并注册到系统的类）类，才能够使用`class_addIvar()`添加Ivar实例变量

- (3) 且必须注意 `class_addIvar()`函数出现位置必须满足:
	- (1) objc_allocateClassPair()之后 >>> 创建类之后
	- (2) objc_registerClassPair()之前 >>> 注册类之前
	- (3) 不能够向`Meta元类`进行使用 >>> 因为实例变量只能够出现在objc类的对象
```

第三点，我觉得是最重要的，其实我们所编写的一个个继承自NSObject的子类，其实最终也是走上面出现的那一段注册类代码步骤:

```objc
// 1. 运行时创建一个类
Class MyClass = objc_allocateClassPair([NSObject class], "myclass", 0);
    
// 2. 添加一个NSString的实例变量，第四个参数是对其方式，第五个参数是参数类型编码
if(class_addIvar(MyClass, "itest", sizeof(NSString *), 0, "@")) {
    NSLog(@"add ivar success");
} else {
    NSLog(@"add ivar failed");
}
    
// 3. 向运行时系统注册创建的类
objc_registerClassPair(MyClass);
```

这几个步骤已经由运行时runtime库帮我们做了。runtime自动读取我们的`xxx.h与xxx.m`，并识别出所有的`@interface ... @end`与`@implementation ...@end`。

最后使用如下这些函数将我们编写的类注册到运行时系统环境:

```c
//1. 创建一个运行时识别的Class
objc_allocateClassPair(Class superclass, 
					   const char *name, 
					   size_t extraBytes);

//2. 添加实例变量
BOOL class_addIvar(Class cls, 
				   const char *name, 
				   size_t size, 
				   uint8_t alignment,
				    const char *types);

//3. 添加实例方法
BOOL class_addMethod(Class cls, 
					 SEL name, 
					 IMP imp, 
					 const char *types);
					 
//4. 添加实现的协议
BOOL class_addProtocol(Class cls, Protocol *protocol);

//5. 将处理完毕的Class注册到运行时系统，之后就无法再对其修改Ivar
void objc_registerClassPair(Class cls);
```

当执行完最后一步`objc_registerClassPair(Class cls)`，就会进行Ivar的内存布局计算了，之后就无法再改变了。

所以这就是为什么之前给Cat类添加Ivar不成功的原因

### 对于已经执行`objc_registerClassPair()`的类不能够使用`class_addIvar()`来添加实例变量，这是否就是对之前的可能会出现Ivar地址错乱问题的规避了？

问题: 如果运行时动态给对象添加一个实例变量时，如果不重新计算所有实例变量的内存地址布局，就会导致实例变量的访问错位。

我觉得这正是直接避免产生这样的问题的直接解决方案，直接不让开发者对已经存在的类去添加实例变量Ivar的操作，这不就是直接去避免了上面的问题吗？

上面这部分是我看书的时候突然想到的一个问题，然而不是书上主要描述的问题。那么书上建议对属性的操作原则:

- (1) 不要在外部直接操作对象的实例变量
- (2) 而是应该通过对象的暴露的存取方法来间接操作实例变量

如上两点正是引出了`@property`的使用，告诉编译器自动做如下事情:

- (1) 属性最终对应的实例变量 >>> `_name`
- (2) 实例变量的读取方法 >>> `name`实现
- (3) 实例变量的修改方法 >>> `setName:`实现
- (4) `NSString  *str = 对象.name;` 自动调用`name`方法实现
- (5) `对象.name = @"haha"` 自动调用`setName:`方法实现
- (6) 对于`对象.name = @"haha"`调用setter时
	- 遵守属性声明时给出的`对象内存管理策略`
	- 修改值之后，自动发出属性修改KVO通知
	- 如果通过`_name = @"haha"` 则不会触发上面两件事

