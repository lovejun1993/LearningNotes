---
layout: post
title: "runtime part 13 TypeEncodings"
date: 2014-09-26 15:25:28 +0800
comments: true
categories: 
---

###类型编码: 苹果会对每一种数据类型，统一按照特定的格式进行编码，得到一个格式化字符串，作用是可以加快消息分发

- 对所有的开发者使用的数据类型，按照`特定的格式`进行编码

- 系统按照`特定的格式`进行解码，找到最终要执行的方法、是被方法参数，类型、返回值类型...等等

- 而如果不使用这种`特定的格式`进行编码和解码，就需要进行`全局的搜索`找到目标方法，这样效率很低下


###类型编码分为两种

- 数据类型 的编码

- 方法 的编码

其实可以理解成 方法的类型编码 是建立在 数据类型的编码 的基础之上。

***

###`数据类型` 编码的对照表

| 最终编码 | 开发时使用的对应数据类型 | 
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

***

###使用`@encode(数据类型)`获取结构体类型编码字符串

```c
typedef struct example {
    int* aPint;
    double  aDouble;
    char *aString;
    int  anInt;
    BOOL isMan;
    struct example *next;
} Example; 
```

```c
char *typeEncodings = @encode(Example);
```

结果输出

```
"{example=^id*iB^{example}}"
```

对上面的typeEncodings字符串每一个元素分析

- 首先以`{`开头，说明是`结构体`类型
- 然后跟着`example`，说明结构体的 类型
- 然后跟着`=`，右侧就是该 结构体 `所有属性` 的 `类型描述`
	- 属性描述按照结构体定义中，从上到下的顺序排列
- `^i` >>> int类型的 `指针`， `^`表示指针 
- `d` >>> double类型
- `*` >>> char* 类型
- `i` >>> int类型
- `B` >>> BOOL类型
- `^{example}` >>> 结构体example类型 的 指针

***

###再看个综合性的例子

```objc
- (void)testTypesEncodings {
    
    NSLog(@">>>>>>>Foundation 类对象>>>>>>>>>");
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(id)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(NSArray*)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(NSDictionary*)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(NSSet*)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(NSString*)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(NSData*)]);
    
    NSLog(@">>>>>>> char >>>>>>>>>");
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(char)]);                   
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(unsigned char)]);
    
    NSLog(@">>>>>>>bool >>>>>>>>>");
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(BOOL)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(Boolean)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(boolean_t)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(_Bool)]);
    
    NSLog(@">>>>>>> short >>>>>>>>>");
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(short)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(unsigned short)]);

    NSLog(@">>>>>>> int >>>>>>>>>");
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(int)]);//==int32_t
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(unsigned int)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(NSInteger)]);//==int64_t
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(NSUInteger)]);//==uint64_t
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(int8_t)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(uint8_t)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(int16_t)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(uint16_t)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(int32_t)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(uint32_t)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(int64_t)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(uint64_t)]);

    NSLog(@">>>>>>> float >>>>>>>>>");
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(float)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(CGFloat)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(double)]);

    NSLog(@">>>>>>> long >>>>>>>>>");
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(long)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(unsigned long)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(long long)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(unsigned long long)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(long double)]);
    
    NSLog(@">>>>>> c >>>>>>>>>>");
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(char *)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(int *)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(void *)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(CGRect)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(struct LinkMap1)]);
    NSLog(@"%@", [NSString stringWithUTF8String:@encode(struct LinkMap2)]);
    
}
```

输出结果

```
2016-09-01 22:39:39.514 XZHRuntimeDemo[7990:78552] >>>>>>>Foundation 类对象>>>>>>>>>
2016-09-01 22:39:39.515 XZHRuntimeDemo[7990:78552] @
2016-09-01 22:39:39.515 XZHRuntimeDemo[7990:78552] @
2016-09-01 22:39:39.515 XZHRuntimeDemo[7990:78552] @
2016-09-01 22:39:39.515 XZHRuntimeDemo[7990:78552] @
2016-09-01 22:39:39.515 XZHRuntimeDemo[7990:78552] @
2016-09-01 22:39:39.515 XZHRuntimeDemo[7990:78552] @
2016-09-01 22:39:39.515 XZHRuntimeDemo[7990:78552] >>>>>>> char >>>>>>>>>
2016-09-01 22:39:39.516 XZHRuntimeDemo[7990:78552] c
2016-09-01 22:39:39.516 XZHRuntimeDemo[7990:78552] C
2016-09-01 22:39:39.516 XZHRuntimeDemo[7990:78552] >>>>>>>bool >>>>>>>>>
2016-09-01 22:39:39.516 XZHRuntimeDemo[7990:78552] B
2016-09-01 22:39:39.516 XZHRuntimeDemo[7990:78552] C
2016-09-01 22:39:39.516 XZHRuntimeDemo[7990:78552] I
2016-09-01 22:39:39.516 XZHRuntimeDemo[7990:78552] B
2016-09-01 22:39:39.516 XZHRuntimeDemo[7990:78552] >>>>>>> short >>>>>>>>>
2016-09-01 22:39:39.516 XZHRuntimeDemo[7990:78552] s
2016-09-01 22:39:39.517 XZHRuntimeDemo[7990:78552] S
2016-09-01 22:39:39.517 XZHRuntimeDemo[7990:78552] >>>>>>> int >>>>>>>>>
2016-09-01 22:39:39.517 XZHRuntimeDemo[7990:78552] i
2016-09-01 22:39:39.517 XZHRuntimeDemo[7990:78552] I
2016-09-01 22:39:39.552 XZHRuntimeDemo[7990:78552] q
2016-09-01 22:39:39.552 XZHRuntimeDemo[7990:78552] Q
2016-09-01 22:39:39.552 XZHRuntimeDemo[7990:78552] c
2016-09-01 22:39:39.552 XZHRuntimeDemo[7990:78552] C
2016-09-01 22:39:39.552 XZHRuntimeDemo[7990:78552] s
2016-09-01 22:39:39.553 XZHRuntimeDemo[7990:78552] S
2016-09-01 22:39:39.553 XZHRuntimeDemo[7990:78552] i
2016-09-01 22:39:39.553 XZHRuntimeDemo[7990:78552] I
2016-09-01 22:39:39.553 XZHRuntimeDemo[7990:78552] q
2016-09-01 22:39:39.553 XZHRuntimeDemo[7990:78552] Q
2016-09-01 22:39:39.553 XZHRuntimeDemo[7990:78552] >>>>>>> float >>>>>>>>>
2016-09-01 22:39:39.553 XZHRuntimeDemo[7990:78552] f
2016-09-01 22:39:39.553 XZHRuntimeDemo[7990:78552] d
2016-09-01 22:39:39.553 XZHRuntimeDemo[7990:78552] d
2016-09-01 22:39:39.553 XZHRuntimeDemo[7990:78552] >>>>>>> long >>>>>>>>>
2016-09-01 22:39:39.553 XZHRuntimeDemo[7990:78552] q
2016-09-01 22:39:39.553 XZHRuntimeDemo[7990:78552] Q
2016-09-01 22:39:39.553 XZHRuntimeDemo[7990:78552] q
2016-09-01 22:39:39.553 XZHRuntimeDemo[7990:78552] Q
2016-09-01 22:39:39.554 XZHRuntimeDemo[7990:78552] D
2016-09-01 22:39:39.554 XZHRuntimeDemo[7990:78552] >>>>>> c >>>>>>>>>>
2016-09-01 22:39:39.554 XZHRuntimeDemo[7990:78552] *
2016-09-01 22:39:39.554 XZHRuntimeDemo[7990:78552] ^i
2016-09-01 22:39:39.554 XZHRuntimeDemo[7990:78552] ^v
2016-09-01 22:39:39.554 XZHRuntimeDemo[7990:78552] {CGRect={CGPoint=dd}{CGSize=dd}}
2016-09-01 22:39:39.555 XZHRuntimeDemo[7990:78552] {LinkMap1={LinkLeftNode=i^{LinkLeftNode}}{LinkRightNode=i^{LinkRightNode}}i}
2016-09-01 22:39:39.555 XZHRuntimeDemo[7990:78552] {LinkMap2=^{LinkLeftNode}^{LinkRightNode}i}
```

我想可以根据结果，来看明白这些数据类型的最终在os系统中的编码字符串了。即我们操作的这些数据类型，最终在os系统中，就是这一些特殊却又规律的字符来表示。


###使用`NSValue`包装`c类型实例（cString、cArray、cStrcut、cUnion）`

```objc
Example exa;
exa.aString = "ssssssssss";
exa.anInt = 1;
exa.isMan = true;
exa.next = &exa;

NSValue *value = [NSValue value:&exa withObjCType:@encode(Example)];

const char *typeEncodings = value.objCType;
```

###使用NSValue对象包装 指针、c字符串后，其原来的指针类型都会变成`void *`类型

```
char x[10] = "dwadaw";
NSValue *value = [NSValue valueWithPointer:x];
const char *type = [value objCType];
```

输出type如下

```
type = "^v"
```

"^v" 就是 void* 的类型编码.

***

###`方法`码的对照表，用来声明`无法用 @encode()`得到的系统编码

| 最终苹果的编码格式 | iOS代码中对应的数据类型 | 
|:-------------:|:-------------:| 
| r | const | 
| n | in | 
| N | inout | 
| o | out | 
| O | bycopy | 
| R | byref | 
| V | oneway |


****

###@property声明的属性的 `特征值objc_property_attribute_t`保存其属性的编码


###`objc_property_attribute_t` 属性的编码格式

```objc
typedef struct { 
    const char *name;           // 特性名，如: T、C、N、V、&、R、W、G、S
    const char *value;          // 特性值，如: @"NSString"、_name、q、B、@"NSArray"、@"NSDictionary" 
} objc_property_attribute_t; 
```

###假设有两个实体类Person、User

```objc
//
//  Person.h
//  XZHRequest
//

#import <Foundation/Foundation.h>
#import "User.h"

@interface Person : NSObject

@property (nonatomic, copy) NSString *name;

@property (nonatomic, assign) NSInteger age;

@property (nonatomic, assign) float height;

@property (nonatomic, assign) BOOL isMan;

@property (nonatomic, strong) NSArray *users;

@property (nonatomic, strong) User *user;

@property (nonatomic, strong) NSDictionary *propertyArrayMap;

@end
```

```objc
//
//  Person.m
//  XZHRequest
//

#import "Person.h"

@implementation Person

@end
```

```objc
//
//  User.h
//  XZHRequest
//

#import <Foundation/Foundation.h>

@interface User : NSObject

@property (nonatomic, copy) NSString *name;

@end
```

```objc
//
//  User.m
//  XZHRequest
//

#import "User.h"

@implementation User

@end
```

###执行如下代码1

```objc
//1. 获取Person类的所有属性
unsigned int outCount;
objc_property_t *properties = class_copyPropertyList([Person class], &outCount);
    
//2. 获取每一个属性的编码字符串
for (int i = 0; i < outCount; i++) {
    
    //2.1
    objc_property_t property = properties[i];
    
    //2.2 
    const char* propertyName = property_getName(property);
    
    //2.3
    const char* propertyAttributeString = property_getAttributes(property);
    
    fprintf(stdout, "属性名 = %s , 属性编码字符串 = %s\n", propertyName, propertyAttributeString);
}

//3. 使用完copy、create、retain出来指针一定要释放
if (!properties) {
	free(properties);
	properties = NULL;
}
```

输出内容

```
属性名字 = name , 属性编码字符串 = T@"NSString",C,N,V_name
属性名字 = age , 属性编码字符串 = Tq,N,V_age
属性名字 = height , 属性编码字符串 = Tf,N,V_height
属性名字 = isMan , 属性编码字符串 = TB,N,V_isMan
属性名字 = users , 属性编码字符串 = T@"NSArray",&,N,V_users
属性名字 = propertyArrayMap , 属性编码字符串 = T@"NSDictionary",&,N,V_propertyArrayMap
```

如上可以看出 

- 属性的attributes编码字符串(Type Encodings)，都是以 `T` 开头

	- `T@`复杂类型
		- `T@"NSString"` 字符串
		- `T@"NSArray"` 数组
		- `T@"NSDictionary"` 字典
	- `Tq`整形
	- `Tf`浮点型
	- `TB`布尔类型

- `C` 表示属性使用 `copy`

- `N` 表示属性使用 `nonatomic`

- `V` 表示 `对象的变量`

- `R` 表示属性使用 `readonly`

- `&` 表示属性使用 `retain`

- `D` 表示属性使用 `dynamic`

- `W` 表示属性使用 `weak`

- `G` 表示属性使用 `Custom Getter 方法`

- `S` 表示属性使用 `Custom Setter 方法`

****

###获取一个property的 编码字符串、方法一

```objc
//1. 
objc_property_t property = class_getProperty([Person class], "user");

//2. property_copyAttributeList()方法，先读取到属性编码字符串中的`每一个特征值`结构体实例
unsigned int num;
objc_property_attribute_t *attrs = property_copyAttributeList(property, &num);
  
//3. 得到每一个编码特征值的 name值与value之值
for (unsigned int i = 0; i < num; i++) {
    objc_property_attribute_t attr = attrs[i];
    fprintf(stdout, "name = %s, value = %s \n", attr.name, attr.value);
}
```
输出内容

```
//将下面的一个整体编码字符串，分隔成很多个小项
name = T, value = @"User" 
name = &, value =  
name = N, value =  
name = V, value = _user 
```

###获取一个property的 编码字符串、方法二

```objc
//1. 
objc_property_t property = class_getProperty([Person class], "user");

//2. property_getAttributes()方法直接获取属性的编码字符串
const char *chars = property_getAttributes(property);
fprintf(stdout, "%s\n", chars);
```

输出内容

```
//将上面的输出的每一项特征值串联在一起
T@"User",&,N,V_user
```

###@property的编码对照表

| 最终苹果的编码格式 | iOS代码中对应的数据类型 | 
|:-------------:|:-------------:| 
| R | readonly | 
| C | copy | 
| & | retain | 
| N | nonatomic | 
| G | getter=(name) | 
| S | setter=(name) | 
| D | @dynamic |
| W | weak |
| P | 	用于垃圾回收机制 |

注意，与属性编码字符串相关的其实还有两个字符，只不过严格来说并不是对某一种类型、修饰符进行编码的字符，而只是起到`标示`的作用:

- (1) `T`: 表示开始的是一个属性的编码字符串

- (2) `V`: 标示后面跟着的字符串就是实力变量的名字

***

###根据YYModel源码以及苹果官方的type encodings文档，总结出如下在编写json转实体框架时的类型编码结构

- (1) property encodings
	- `T` >>> 表示属性生成的实例变量的类型编码
		- `c` >>> A char
		- `i` >>> An int
		- `s` >>> A short
		- `l` >>> A long（l is treated as a 32-bit quantity on 64-bit programs.）
		- `q` >>> A long long
		- `C` >>> An unsigned char
		- `I` >>> An unsigned int
		- `S` >>> An unsigned short
		- `L` >>> An unsigned long
		- `Q` >>> An unsigned long long
		- `f` >>> A float	
		- `d` >>> A double
		- `B` >>> A C++ bool or a C99 _Bool
		- `v` >>> A void
		- `*` >>> A character string (char *)
		- `@` >>> An object (whether statically typed or typed id)
		- `#` >>> A class object (Class)
		- `:` >>> A method selector (SEL)
		- `[array type]` >>> An array
		- `{name=type...}` >>> A structure
		- `(name=type...)` >>>  A union
		- `bnum` >>> A bit field of num bits
		- `^type` >>> A pointer to type
		- `?` >>> An unknown type (among other things, this code is used for function pointers)
	- `V` >>> 标示实例变量的名字
	- `R` >>> The property is read-only (readonly).
	- `C` >>> The property is a copy of the value last assigned (copy).
	- `&` >>> The property is a reference to the value last assigned (retain).
	- `N` >>> The property is non-atomic (nonatomic).
	- `G` >>> The property defines a custom getter sel
	- `S` >>> The property defines a custom setter sel
	- `D` >>> The property is dynamic (@dynamic)
	- `W` >>> The property is a weak reference (__weak) 
	- `P` >>> The property is eligible for garbage collection
	- `t` >>> Specifies the type using old-style encoding
- (2) method encodings
	- `r` >>> const
	- `n` >>> in
	- `N` >>> inout
	- `o` >>> out
	- `O` >>> bycopy
	- `R` >>> byref
	- `V` >>> oneway

