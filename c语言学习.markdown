---
layout: post
title: "重新捡起C++"
date: 2015-06-16 23:00:37 +0800
comments: true
categories: 
---

c/c++重拾

##0 的四种情况

- 整形 0
	- int 类型，占用四个字节/32个二进制位
	- 仅表示 `数值0`

- 空指针 NULL
	- 一个空指针的常量
	- 其实和整形0区别不是太大，但是有一点区别
	- NULL是`地址0`，也可以是`数值0`

- 字符串结束标志 
	- 是一个`字符` >>>> `'\0'`
	- 仅仅占一个字节 >>> 8位
	- 二进制表示为 `0000 0000`

- 逻辑 false / true
	- FAULSE/TRUE 是`int`类型 >>> 32位
	- faulse/true 是`bool`类型 >>> 1位

所以使用0的时候需要小心如上四种情况.

##原码、补码、反码

- 规则

	- **`补码` 是二进制中表示 `负数` 的一种方法**
	- **`正数` 的 原码、反码、补码 都是自己**
	- **负数的`原码`: 最高位是`符号位`，即`1`表示负，`0`表示正，后面是数值位**
		- -15的原码 >>> 1000 1111
	- **负数的`反码` 表示为在`原码`基础上，`符号位不变`，其余各位`取反`**
	- **负数的`补码` 表示为`反码`在末位加1**

- 那看到一个补码，怎么知道它是`什么数值`的补码呢？
	- 如果是`带符号`的二进制数
	- 看到第一位是0，那它是个整数，而正数的原码，反码，补码都一样，直接运算就得值了
	- 看到第一位是1，说明它是个负数，那就三部曲
		- `第一位符号位不变`
		- 其余位按位取反
		- 然后末位加1

- 举例、1000 1111 这个数是谁的补码呢？
	- 假设是带符号位的类型
	- 保留第一位符号位，其余各位取反 >>>> 1111 0000
	- 然后 1111 0000 最低位末尾 + 1 >>>> 1111 0001
	- 1111 0001 >>>> 转换成十进制数
		- 第一位，说明 它是负数
		- 后面再算就会是64+32+16+1=113
		- 所以 1000 1111 是 -113 的补码

##判断比较，常量值放左边

```c
if (0 == x) {
	//....
}
```

##宏的使用总结

- 就是一段代码的复制替换、和一些语法编译不过去的语法表示

```objc
for (NSUInteger i = 2; i < numberOfArguments; i++) {
    const char *argumentType = [methodSignature getArgumentTypeAtIndex:i];
    switch(argumentType[0] == 'r' ? argumentType[1] : argumentType[0]) {
    
    	// 宏定义代替一段代码
        #define JP_FWD_ARG_CASE(_typeChar, _type) \
        case _typeChar: {   \
            _type arg;  \
            [invocation getArgument:&arg atIndex:i];    \
            [argList addObject:@(arg)]; \
            break;  \
        }
        
        // 宏使用
        JP_FWD_ARG_CASE('c', char)
        JP_FWD_ARG_CASE('C', unsigned char)
        JP_FWD_ARG_CASE('s', short)
        JP_FWD_ARG_CASE('S', unsigned short)
        JP_FWD_ARG_CASE('i', int)
        JP_FWD_ARG_CASE('I', unsigned int)
        JP_FWD_ARG_CASE('l', long)
        JP_FWD_ARG_CASE('L', unsigned long)
        JP_FWD_ARG_CASE('q', long long)
        JP_FWD_ARG_CASE('Q', unsigned long long)
        JP_FWD_ARG_CASE('f', float)
        JP_FWD_ARG_CASE('d', double)
        JP_FWD_ARG_CASE('B', BOOL)
        
        .....
	}
}
```

- `#标识`: 如果宏中的的某一个参数前面添加`#`,预处理器会把这个参数转换为一个`字符数组`

```c
#define ERROR_LOG(module)   fprintf(stderr,"error: "#module"\n")
```

如下两种宏使用

```c
ERROR_LOG("add");  >>>> fprintf(stderr,"error: "add"\n");
```

```c
ERROR_LOG(devied =0); >>> fprintf(stderr,"error: devied=0\n");
```

也就是说`#参数`统一作为字符串输出.

- `##标识##`强制连接

```c
#define TYPE1(type)   type var_##type##_type
```

宏使用

```c
TYPE1(int); >>> int var_int_type ;
```

- 空指针宏定义

```c
#define NULL_PTR ((void *) 0) >>> NULL
```

- 防止宏连续被声明、以及 `.h头文件被重复导入`

```c
#ifndef kCount
#define kCount
#endif
```

```c
#ifndef kCount

// 导入.h
#include "xxxx1.h"
#include "xxxx2.h"

#endif
```


##c/c++中的指针必须先初始化再使用

- 使用前先初始化

```c
int *p = NULL;

//....
```

- free()、release()系统释放等函数使用模板

```c
if (point) {
	CFRelease(point);
	point = NULL;
}
```

##防止重复包含头文件

- 解决方法一

```c
#ifndef __模块名_H__
#define __模块名_H__

//导入头文件
#include "a.h"
#include "b.h"
#include "c.h"

//方法定义
void test();

#endif 
```

这种方式是c/c++标准支持的方法，建议使用这种来解决。但是也容易出现`__模块名_H__`重复定义的情况，那么所以这个宏的命名格式最好是: `__项目名_路径_文件_H__` 这样的格式，那就能很好的解决重复定义了.

- 解决方法二

```c
#pragam once

//导入头文件
#include "a.h"
#include "b.h"
#include "c.h"

//方法定义
void test();
```

这种方法效率有提升，但是不被c/c++标准支持，所以少用.

- iOS/mac 采用`#import`解决

##结构体中的元素布局（字节对齐）

如下两个结构体，元素都是一样的，以32位系统为例.

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

现代计算机内存单元，都是使用`字节（代替8个二进制位）`作为基本单位划分。理论上说可以从任何的内存地址来访问到变量的所在内存单元。但是实际上，为了提升效率，各种类型的数据地址是按照`一定的规律来有序排列`，那么所以对于某一些类型的数据访问，可以直接从特定的位置上开始访问，就不用从头开始遍历寻找。牺牲空间来换取时间，这就是字节对齐。

> 结构体默认的字节对齐一般满足三个标准:

- 结构体变量的`首地址`能够被该结构体中`最大长度`的成员变量的长度所`整除`.
	- 首地址不好控制

- 结构体变量中每个成员变量相对于`结构体变量首地址`的偏移量（offset）都是成员变量自身长度的`整数倍`.
	- `成员变量的偏移量`不够时由编译器自动填充

- 结构体变量的总长度为成员变量中最大长度的`整数倍`.
	- `结构体的总长度`不够时由编译器自动填充


> 编译器会自动进行字节对齐，使用空白无用的内存字节作为填充。
	
感觉第一条规则我们写代码的控制不了吧，那么久最后两条规则我们可以代码控制:

- 成员变量相对于结构体变量首地址的`偏移量`长度
- 结构体的总长度

那么对于上面的`结构体A变量`的内存字节布局是这样的:

- 首先变量a是int类型，占用4个字节
	- 第一个变量直接放
- 然后变量b是char类型，占用1个字节
	- 相对于起始地址的偏移量 = 4，满足规则2
	- 所以直接从第五个字节开始存放b，并占用一个字节长度
- 最后变量c是short类型，占用2个字节
	- 此时相对于起始地址的偏移量 = 5，`不满足规则2`
	- 所以向后填充一个字节，这个填充字节什么事都不干，就是为了字节对齐
	- 那么此时变量c相对起始地址的偏移量 = 6，刚好是2的整数倍，c占用两个字节
- 所以最终长度 = 8个字节


那么对于`结构体变量B`的内存字节布局是这样的:

- 首先变量a是char类型，占用4个字节
	- 第一个变量直接放
- 然后变量b是int类型，占用4个字节
	- 此时相对于起始地址的偏移量 = 1，`不满足规则2`
	- 为了让b相对起始地址的偏移量是int长度（4）的整数倍
	- 所以偏移量修改 = 4，就是说空出了三个没用的字节，什么都不干，就是为了字节对齐
	- 从第5个字节开始排布变量b，并占用4个字节长度
	- 那么此时长度为8个字节
- 最后变量c是short类型，占用2个字节
	- 此时相对于起始地址的偏移量 = 8，`满足规则2`，直接排布c
	- c此时占用2个字节长度
	- 那么此时结构体总长度 = 10，`不满足规则3`
- 为了让结构体总长度是成员最大长度的整数倍
	- 最大成员长度是b = 4
	- 所以最后扩展为长度 = 12，刚好大于10又是4的整数倍
- 所以最终的结构体B的长度是12

> 对于上面的两个结构体可以使用`保留字节`来避免编译器自动填充字节作为无用字节:

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

对于上面使用reserved保留变量声明的内存字节，我们也可以去使用了。而如果我们不使用reserved保留变量去指向这些字节，那么这些字节就相当于放在那什么也不干就是为了字节对齐，什么用都没有。


> 对于结构体成员变量从上到下排布的原则:

- 成员变量的排布按照自身的长度`从小到大`依次排列
- 尽量的使用保留字段，来使用由编译器进行字节对齐时填充无用的字节

那么对上面两个结构体按照如上原则最后的修改版本为如下:

```c
struct C {
    char a;             // 占1个字节
    char reserved;      // 此处会由编译器填充1个字节，那么我们可以定义以后使用的保留变量来操作
    short b;            // 占2个字节
    int c;              // 占4个字节
};
```

##基本数据类型占用的字节（32/64）

0UL 或 1UL 是什么意思？

```
0UL >>> 无符号(unsigned) + 长整型(long) + 0
```

```
1UL >>> 无符号(unsigned) + 长整型(long) + 1
```

- 如果不写UL后缀，系统默认为：int
- 整型常数默认是signed int
- 在c语言代码层面中，整形数值的编码格式只有如下:
	- 十进制形式
	- 以`0`开头的八进制形式
	- 以`0x`开头的十六进制形式
	- 注意: 并没有 `二进制` 形式

> 数据类型

- 整形数值的类型

```
char / signed char / unsigned char

int / unsigned int

short int / unsigned short int

long int / unsigned long int

long long int (C99才支持) / unsigned long long int
```

- 主要分为 有符号（signed）与 无符号（unsigned）:

	- signed: 取值范围内有正负数之分
	- unsigned: 取值范围内全是正数

- int、short int、long int、long long int 系统分为的单元长度

```
short >= 2个字节/16位
int >= short，一般是4个字节/32位
long >= 4个字节/32 位，且 >= int 
long long >= 8个字节/64位，且 >= long a
```

如上的不确定，造成`int`类型的长度就不确定

```
造成int可以是: 16位、24位、32位、64位...
```

如下是iOS中对NSInteger根据当前操作系统32/64时，宏编译使用typedef使用对应的长度的类型

```c
#if __LP64__ || (TARGET_OS_EMBEDDED && !TARGET_OS_IPHONE) || TARGET_OS_WIN32 || NS_BUILD_32_LIKE_64
	typedef long NSInteger;
	typedef unsigned long NSUInteger;
#else
	typedef int NSInteger;
	typedef unsigned int NSUInteger;
#endif
```

- char , signed char , unsined char 之间的区别:

	- char >>> c语言会根据当前操作系统的位数，去选择使用signed char 或 unsined char
	- signed char >>> -128 ~ 127
	- unsined char >>> 0 ~ 255
	- 所以，如果确定了char值会`小于-128 或 大于127` 一定不要使用char

- `固定`长度的类型

```
int16_t 固定占用16位
int32_t 固定占用32位
int64_t 固定占用64位
```

- 满足`最少`要求长度的类型

```
int_least16_t 可以得到一个当前平台所支持的至少有 16 位宽的最短整数类型
```

```
int_fast32_t 可以得到当前平台下得到处理速度最快的至少为 32 位的整数类型
```

- 对如上类型使用L、LL、LLU、f、U等符号修饰的意义

```
8L，表示long型 
8LL，表示long long型 
8LLu或8uLL，表示 无符号 的long long型 
56.0 >>> double类型 
56.0f 或 56.f >>> float型，但56f是错误的。 
56.0L >>> long double类型
```

所以也就是说

```
L >>> long
U >>> 无符号
f >>> float
5.5 >>> double
```

##查看int、float、long、double在`当前操作系统运行环境`下的最大值、最小值

```
#include <climits>
```

- 获取int在当前操作系统运行时长度

```
INT_MAX
SHRT_MAX
L0NG_MAX
LLONG MAX
```

- 获取float在当前操作系统运行时长度

```
FLT_MIN, 最小长度
FLT_DIG, 一定能够保证的有效位数
FLT_MAX, 最大长度
```

- 获取double在当前操作系统运行时长度

```
DBL_MIN
DBL_DIG
DBL_MAX
```

## `void*`指针类型（无类型指针），指向一块内存对象的地址，但不能获取该内存对象的其他定义，也就是说需要类型强制转换。类似iOS中的`id`万能指针类型

- void* 就类似Objective-C中的 id万能指针

通常在c语言中，使用malloc()创建一个结构体实例的代码中会对malloc()创建返回的对象地址进行类型强转.

```c
LNode *node = (LNode *)malloc(sizeof(LNode));
```

- c中定义的内存管理的库函数都是使用 void* 指针类型

```c
void* malloc(size_t);
```

```c
void	 free(void *);//接收任意类型的指针变量
```

- void指针可以指向任意类型的数据，也可用任意数据类型的指针对void指针赋值

```c
int *p1 = 6;
void* p2;
    
p2 = p1; 
p1 = (int *)p2;
```

- 在ANSI C标准中，不允许对void指针进行算术运算如`pvoid++或pvoid+=1`等，而在GNU中则允许

```
void*pvoid;
pvoid++;//ANSI：错误
pvoid+=1;//ANSI：错误
```

- 如果函数的参数可以是任意类型指针，那么应声明其参数为`void*`

```c
void	 free(void *);//接收任意类型的指针变量
```

##按位与 >>> 用于位移枚举，用于枚举值可以相互组合


> 核心是与1进行按位与会保留值，与0按位与会清零，即可以得到指定位的数值.

- 一个枚举类型包含多种操作类型时，Mask掩码 >>>> `按位与`

首先是多种类型的枚举类型定义，摘录自YYModel中的type encoding的枚举定义.

```c
typedef enum TypeEncoding {
    
    TypeEncodingUnkonwn                     =   0,
    
    //类型一、占1~8位的掩码
    TypeEncodingIvarMask                    =   0xFF, //低8位数值为正数255 >>> 8个二进制位全1
    TypeEncodingIvarInt                     =   1,
    TypeEncodingIvarBool                    =   2,
    TypeEncodingIvarObject                  =   3,
    //还可以添加(255 - 3)个

    //类型二、占9位~16位
    TypeEncodingQualifierMask               =   0xFF00,//9位到16位数值为255 >>> 同样8个二进制位全1
    TypeEncodingQualifierConst              =   1 << 8,//左移8位
    TypeEncodingQualifierInOut              =   1 << 9,//左移9位
    //还可以添加(8 - 2)个
    
    //类型三、占17位~24位
    TypeEncodingPropertyMask                =   0xFF0000,//17位到24位数值为255 >>> 同样8个二进制位全1
    TypeEncodingPropertyReadonly            =   1 << 16,
    TypeEncodingPropertyCopy                =   1 << 17,
    TypeEncodingPropertyStrong              =   1 << 18,
    //还可以添加(8 - 3)个
    
}TypeEncoding;
```

测试代码

```c
void testMask() {
    
    printf("%d\n", 0xFF);
    printf("%d\n", 0xFF00);
    printf("%d\n", 0xFF0000);
    
    printf("%ld\n", sizeof(TypeEncoding));
    
    TypeEncoding coding = TypeEncodingUnkonwn;
    
    coding |= TypeEncodingIvarInt;
    printf("%d\n", coding);
    
    coding |= TypeEncodingQualifierMask;
    printf("%d\n", coding);
    printf("%d\n", coding & TypeEncodingIvarMask);
    printf("%d\n", coding & TypeEncodingQualifierMask);
    
    coding |= TypeEncodingPropertyReadonly;
    printf("%d\n", coding);
    printf("%d\n", coding & TypeEncodingIvarMask);
    printf("%d\n", coding & TypeEncodingQualifierMask);
    printf("%d\n", coding & TypeEncodingPropertyMask);
}
```

通过对应的类型的Mask掩码来获取对应长度位的数值:

- 0xFF 掩码 >>>	获取低8位的数值
- 0xFF00 掩码 >>>		获取9位到16位的数值	
- 0xFF0000 掩码 >>>		获取17位到24位的数值

如上例子如果不太明白，下面还有一个简单的例子:

```c
typedef enum ActionType{
    ActionTypeUp    = 1 << 0, // 1
    ActionTypeDown  = 1 << 1, // 左移一位，10 >>> 2
    ActionTypeRight = 1 << 2, // 左移二位，100 >>> 4
    ActionTypeLeft  = 1 << 3, // 左移三位，1000 >>> 8
}ActionType;
```

```c
void testWithActionType(ActionType type) {
    if (type == 0)
    {
        return;
    }
    
    if ((type & ActionTypeUp) == ActionTypeUp)
    {
        printf("上\n");
    }
    
    if ((type & ActionTypeDown) == ActionTypeDown)
    {
        printf("下\n");
    }
    
    if ((type & ActionTypeLeft) == ActionTypeLeft)
    {
        printf("左\n");
    }
    
    if ((type & ActionTypeRight) == ActionTypeRight)
    {
        printf("右\n");
    }
}
```

```c
void testMask() {

	ActionType type = ActionTypeUp | ActionTypeLeft | ActionTypeRight | ActionTypeDown;
	
	testWithActionType(type);
}
```

输出结果

```
上
下
左
右
```

##复合字面值（C99标准提出的） 

> 格式: (类型){初始化数据列表}

数组直接初始化

```
float *arr = (float []){0.1, 0.2, 0.3, 0.4};
```

```
int *arr2 = (int []){1, 2, 3, 4, 5, 6};
```

结构体实例初始化

```
struct Student {
    int pid;
    char name[10];
    int age;
};
```

```
struct Student s = {.pid = 18888, .name = "XiaoMing", .age = 19};
```

还有一种在开始地方初始化:

```c
struct __Man {
    int pid;
};

typedef struct __Man Man;

static Man man = {
    111,
};
```

定义并初始化一个字符串常量（const修饰），如果不是在一个函数内定义，而是在头文件开始的全局内定义的，那么就会变成一个`静态的字符数组`

```
const char arr[] = (const char []){"我是一个常量"};
```

##数组作为方法参数的两种写法

```
void work1(int arr[]) {
    printf("%d\n", arr[1]);
}

void work2(int *arr2) {
	printf("%d\n", arr2[1]);
}
```

##main()函数的参数意义

随便写一个Main.c代码如下

```c
#include <stdio.h>

int main(int args, char *argv[]) {
    
    if (args == 0) {
        printf("执行main()时没有带额外参数\n");
    } else {
        printf("执行main()时带有%d个额外参数\n", args);
        for (int i = 0; i < args; i++) {
            printf("args[%d] = %s\n", i, argv[i]);
        }
    }
    
    return 1;
}
```

然后到该Main.c目录下，使用gcc编译生成二进制可执行文件
	
- 首先生成 Main.c >>> Main.o
- 然后 Main.o >>> 二进制可执行文件Main	

```
gcc Main.c -o Main
```

打开终端执行如下shell命令调用Main可执行文件

```
./Main One Two "Three" 4
```

终端输出如下

```
执行main()时带有5个额外参数
args[0] = ./Main
args[1] = One
args[2] = Two
args[3] = Three
args[4] = 4
```

哈哈，懂main()中参数就是执行最终生成的二进制文件时，外界传入的参数.

##参数长度不固定 >>> va_list

```
#include <stdarg.h>

int sum(int count, ...)
{
    int sum = 0;
    int i;
    va_list ap;
    va_start(ap, count);
    for (i = 0; i < count; i++)
    {
        sum += va_arg(ap, int);
    }
    va_end(ap);
    return sum;
}
```

main()测试代码

```
printf("sum = %d\n", sum(5, 4, 5, 6, 10, 1));
```

- va_arg(ap , type)中的type绝对不能为以下类型:

```
——char、signed char、unsigned char
——short、unsigned short
——signed short、short int、signed short int、unsigned short int
——float
```

##指针的指针 (类型 **p;) 通常用于回传一个内部创建的实例

通常用来动态创建值，并填充值返回出去（类似NSError **error）

```c
typedef struct Error {
    int errorCode;
    char *errorDomain;
}Error;
```

```c
void writeToFile(char *fileName, Error **errP) {
    printf("fileName = %s\n",fileName);
    
    //取二重指针的值（地址）
    *errP = (Error *)malloc(sizeof(Error));
    (*errP)->errorCode = 10001;
    (*errP)->errorDomain = "Time Out ....";
}
```

下面是main()测试代码

```c
//1. 一重指针
Error *err;

//2. 一重指针取地址
writeToFile(fileName, &err);

//3. 使用一重指针
printf("error code = %d, error domain = %s\n", err->errorCode, err->errorDomain);
```

##常量、指针常量、常量指针、

常量

```
const int MAX_WIDTH = 100;
int const MAX_HEIGHT = 200;
```

指针常量 >>> 固定值

```
const char arr[] = (const char []){"我是一个常量"};
```

```
char *const kLaunchFinishedKey = "kLaunchFinishedKey";
```


指向常量的指针 >>> 固定指针

指针所保存的地址可以改变，然而指针所指向的值却不可以改变

```c
typedef struct Error {
    int errorCode;
    char *errorDomain;
}Error;
```

```c
Error *error1 = (Error *)malloc(sizeof(Error));
Error *error2 = (Error *)malloc(sizeof(Error));

//指针所保存的地址可以改变
const Error *errP = error1;
errP = error2;

//指针所指向的值却不可以改变
errP->errorCode = 111111; //这句话会报错
```

##函数指针

```c
typedef int (*IMP)(int x, char *s);
```

```c
int work(int x, char *s) {
    return 1;
}
```

```c
IMP imp = work;
```

```c
int ret = imp(19, "Hello World!");
```

##OC中的`objc_msgSend()`支持多种类型参数重载

> c语言不支持函数重载，只有c++才支持

下面是c++中函数重载写法，模拟OC中的objc_msgSend()函数的重载效果

```c
#include "Chapeter2.hpp"
#include <iostream>

using namespace std;
using  std::wstring;
using  std::string;

void work(void *self, char *sel) {
    
}

int work(void *self, char *sel ,int x) {
    return 1;
}

bool work(void *self, char *sel ,int x, int y) {
    return true;
}

string work(void *self, char *sel ,int x, int y, int z) {
    return string("Hello world~!!");
}
```

如上函数指针格式

```
void    (*funcP1)(void *self, char *sel);
int     (*funcP2)(void *self, char *sel ,int x);
bool    (*funcP3)(void *self, char *sel ,int x, int y);
string  (*funcP4)(void *self, char *sel ,int x, int y, int z);
```

函数指针转换

```
void test() {
    
    void *ptr = NULL;
    char sel[] = "sel";
    
    //1.
    ((void (*)(void *, char *))work)(ptr, sel);
    
    //2.
    int ret1 = ((int (*)(void *, char *, int))work)(ptr, sel, 19);
    
    //3.
    bool ret2 = ((bool (*)(void *, char *, int, int))work)(ptr, sel, 19, 20);
    
    //4.
    string ret3 = ((string (*)(void *, char *, int, int, int))work)(ptr, sel, 19, 20, 21);
    
}
```

##struct结构体之、基础使用

测试struct

```c
typedef struct Stu{
    char    *name;
    int     num;
    int     age;
    char    group;
    float   score;
}Stu;
```

栈实例

```c
//1. 
int *arr_int = (int []){1, 2, 3, 4, 5, 6};

//2. 乱序初始化成员变量
Stu stu1 = {.name = "haha1", .num = 19, .age = 19, .group = 'M', .score = 99.1};

//3. 顺序初始化成员变量
Stu stu2 = {"haha1", 19, 19, 'M', 99.1};

//3. 初始化一个栈结构体数组
Stu arr1[] = {
    {"haha1", 19, 19, 'M', 99.1},
    {"haha2", 20, 19, 'M', 99.1},
    {"haha3", 21, 19, 'M', 99.1},
    {"haha4", 22, 19, 'M', 99.1},
    {"haha5", 23, 19, 'M', 99.1},
};

//4. 初始化一个栈结构体数组
Stu *arr2 = (Stu[]){
    {"haha1", 19, 19, 'M', 99.1},
    {"haha2", 20, 19, 'M', 99.1},
    {"haha3", 21, 19, 'M', 99.1},
    {"haha4", 22, 19, 'M', 99.1},
    {"haha5", 23, 19, 'M', 99.1},
};
  
//5. 静态数组元素是struct栈实例  >>> item是 `值`
static Stu arr3[] = {
    {"haha1", 19, 19, 'M', 99.1},
    {"haha2", 20, 19, 'M', 99.1},
    {"haha3", 21, 19, 'M', 99.1},
    {"haha4", 22, 19, 'M', 99.1},
    {"haha5", 23, 19, 'M', 99.1},
};

static Stu *arr4 = NULL;
arr4 = (Stu[]){
    {"haha1", 19, 19, 'M', 99.1},
    {"haha2", 20, 19, 'M', 99.1},
    {"haha3", 21, 19, 'M', 99.1},
    {"haha4", 22, 19, 'M', 99.1},
    {"haha5", 23, 19, 'M', 99.1},
};
  
//6. 静态数组元素是堆实例的指针 >>> item是 `指针`
Stu *s1 = malloc(sizeof(Stu));
s1->name = "hahah6";
s1->num = 24;
static Stu *arr5[5] = {0};
arr5[0] = s1;
arr5[1] = s1;
arr5[2] = s1;
arr5[3] = s1;
arr5[4] = s1;
```    

堆实例

```c
//1. 
struct Student **arr5 = malloc(sizeof(struct Student) * 5);
arr5[0] = s3;
arr5[1] = s3;
arr5[2] = s3;
arr5[3] = s3;
arr5[4] = s3;

//2.
Stu *arr7[] = {0};
arr7[0] = s1;
arr7[1] = s1;
arr7[2] = s1;
arr7[3] = s1;
arr7[4] = s1;

//3. 
Stu **arr8 = calloc(5, sizeof(Stu));
arr8[0] = s1;
arr8[1] = s1;
arr8[2] = s1;
arr8[3] = s1;
arr8[4] = s1;
```


##struct结构体之、Context类型的作用

- 第一种、方法的参数可能很多，不太好在c方法中全部写下
- 第二种、方法需要返回很多个返回值，一个一个使用指针显的不太方便

以回传多个返回值为例，先声明返回多个返回值的struct.

```c
/**
 *  CTRun参数打包
 */
typedef struct XJ_CTRunContext {
    CFIndex runIndex;
    CTRunRef run;
    CGRect runBounds;
}XJ_CTRunContext;
```

然后c函数实现

```c
void XJ_CTRunContextWithUIKitPoint(CTFrameRef ctFrame, CGPoint point, CGRect graphicsRect, XJ_CTRunContext *runContext)
{
	//1. 操作前面的几个方法需要的参数:
	//ctFrame
	//point
	//graphicsRect
	
	//2. 计算得到结构体中定义的三个返回值
	runIndex = ...;
	run = ....;
	runBounds = ....;
	
	//3. 将上面的返回值塞给结构体实例返回给c方法调用者
	if (!runContext) {
        runContext = (XJ_CTRunContext *)malloc(sizeof(XJ_CTRunContext));
    }
    runContext->runIndex = runIndex;
    runContext->run = run;
	runContext->runBounds = temp;
}
```

##struct结构体之、位段结构体

> 位段结构体常用于OC中判断是否实现了声明的Delegate中的协议方法时

定义格式

```c
struct __touchDelegate {
    
    //格式:
    变量类型 成员名 : 分配的长度;
    
    //例子:
    unsigned int touchBegin : 1;
};
```

OC中delegate的定义

```objc
@protocol TouchDelegate : NSObject

- (void)touchBegin;
- (void)touchMoved;
- (void)touchEnd;

@end
```

对于上面Protocol对于的位段结构体定义，因为只需要判断TargetClass是否实现了如上的协议方法，所以只需要两种值: 

- 1)true/1 2)false/0
- 归根结底就是 0/1 >>> 只需要`一个二进制位`即可表示
- so，位段结构体成员长度全部分配为1

```c
struct __touchDelegate {
    unsigned int touchBegin : 1;
    unsigned int touchMoved : 1;
    unsigned int touchEnd   : 1;
};
```

然后在接收一个id<TouchDelegate>的setter方法内判断delegate目标类/对象是否已经实现了协议方法，后面直接根据位段结构体实体来判断是否已经实现.（因为一般不会出现之前实现了协议方法，而后面又没有实现的情况，除非是runtime时动态的移除了IMP）

```objc
- (void)setDelegate:(id<TouchDelegate>)delegate {
    _delegate = delegate;
    
    if ([delegate respondsToSelector:@selector(touchBegin)]) {
        __touchDelegate.touchBegin = 1;
    }
    
    if ([delegate respondsToSelector:@selector(touchMoved)]) {
        __touchDelegate.touchMoved = 1;
    }
    
    if ([delegate respondsToSelector:@selector(touchEnd)]) {
        __touchDelegate.touchEnd = 1;
    }
}
```

而后面调用delegate对应的方法时，根据上面的位段结构体实例的对应数据项进行回调协议方法.

##动态内存管理

malloc()、calloc() 分配内存

```c
//1. 一个实例
Error *error =  (Error *)malloc(sizeof(Error));
```


```c
//2. 数组
Error **array = calloc(10, sizeof(Error));
```

realloc()调整内存大小

> 安全起见，小心使用C语言realloc()函数.

```
void	*realloc(void *ptr, size_t size);
```

第一点、realloc() 会将 ptr 所指向的内存块的大小修改为 size，并将新的内存指针返回，如果size < 之前指针内存的size，就会产生截断某一部分.

第二点、分配的新的内存地址，可以和之前的内存地址完全相同，并开始于同一个起始地址.


```c
Error **array = calloc(10, sizeof(Error));
printf("%p\n", array);

realloc(array, 10 * sizeof(error));
printf("%p\n", array);
```

```c
void *error =NULL;
void *newError = realloc(error, sizeof(Error));
if (!newError) {
    printf("realloc()失败\n");
}
error = newError;
```


- free()释放内存

```
Error *error =  (Error *)malloc(sizeof(Error));

//标准两步
free(error);
error = NULL;
```

###信号

- 什么是信号？信号是干什么的？

	- 信号是由操作系统产生
	- 信号由`操作系统`发送给某一些`进程`
	- 进程接收到`特定的信号`，做出特定的处理，来知道系统当前发生了什么事情

- signal.h中定义了很多系统信号，常见的信号如下:

| 信号        | 用处           |
| ------------- |:-------------:|
| SIGABRT  | 异常终止 |
| SIGFPE | 在算术运算中发生错误 |
| SIGILL | 无效指令 |
| SIGINT | 中断  |
| SIGSEGV | 无效存储访问 |
| SIGTERM | 终止请求 |

- 信号的函数简单使用

```c
void handler(int sig)
{
    printf("signal = %d\n", sig);
}

void testChapeter3() {

	// 发出信号并指定信号的回调处理函数
    signal(SIGALRM, handler);
    signal(SIGVTALRM, handler);
}
```

##C++中的命名空间

```c
namespace Li{
    int flag;
    
    void test() {
        
    }
}
namespace Han{
    bool flag;
    
    void test() {
        
    }
}
```

```c
int main() {
    
    Li::flag = 1;
    Han::flag = false;
    
    Li::test();
    Han::test();
    
    return 0;
}
```

##c/c++ 只有传地址才能在方法内部改变实例

```c
struct Node {
    int value;
    BOOL isMan;
    char sex;
};
```

测试1

```c
void test1(struct Node *node) {
    node = malloc(sizeof(struct Node));
    node->isMan = YES;
}
```

测试2

```c
void test2(struct Node **node) {
    *node = malloc(sizeof(struct Node));
    (*node)->isMan = YES;
}
```

测试代码

```c
struct Node *node1 = NULL;
test1(node1);
    
struct Node *node2 = NULL;
test2(&node2);
```

输出

```
(lldb) po node1
<nil>

(lldb) po node2
0x00007f94daf360e0
```

可以看到第二种方式才回传了内部创建的实例。

##c/c++ 直接传递一个指针，默认都是采用`浅拷贝`，即此时指针指向的是一份拷贝实例，并非原始实例。但是`浅拷贝`是让实例内部实例变量直接是指针指向，所以对实例内部实例变量的修改是生效的。

```c
void test3(struct Node *node) {
    node->isMan = NO;
}
```

测试

```c
struct Node *node3 = malloc(sizeof(struct Node));
node2->isMan = YES;
test3(node3);
```

输出

```
(lldb) po node3->isMan
NO
```

确实是可以修改内部实例变量的指针所指向的数据。那如果对传入的指针重新赋值另外一个实例了？

测试代码

```c
void test4(struct Node *node) {
    node = malloc(sizeof(struct Node));
}
```

测试

```objc
struct Node *node3 = malloc(sizeof(struct Node));
NSLog(@"%p", node3);
test4(node3);
NSLog(@"%p", node3);
```

输出

```
2016-10-20 00:34:44.082 XZHRuntimeDemo[9684:100568] 0x7fdea17757b0
2016-10-20 00:34:44.082 XZHRuntimeDemo[9684:100568] 0x7fdea17757b0
```

可以看到即使内部改变传入指针指向的实例，但是外部依然没有改变。


所以，对传入指针的修改是无效的。但是对指针指向的实例内部的实例变量的指针进行修改是生效的。根本原因就是方法传递的值默认都是采用`浅拷贝`，而浅拷贝主要有两点核心:

- (1) 传递的对象是一份新的拷贝数据，即使修改也不会反应到原始实例
- (2) 但是拷贝数据内部的实例变量确是直接指向原始的对应数据，是可以直接修改的

这和Objective-C中方法参数传递的道理也是一样的。

##C++引用（Reference）

一般情况除传递指针和引用时，其他情况都是默认使用`拷贝`的方式传递，方法内得到的只是原始值的一份拷贝数据，而非原始数据本身。

但是默认情况都是使用`浅拷贝`方式，所以对于传递的数据本身修改无效，但是对数据内部的`子数据`进行修改仍然是有效果的。

- 传引用的例子

```c
void swap(int &a, int &b)
{
    int temp = a;
    a = b;
    b = temp;
}

int main()
{
    int num1 = 10;
    int num2 = 20;
    cout<<num1<<" "<<num2<<endl;
    swap(num1, num2);
    cout<<num1<<" "<<num2<<endl;
    return 0;
}
```

运行结果：

```
10 20
20 10
```

- 函数的返回值是一个引用

普通的返回值类型

```c
int valplus1(int &a)
{
    a = a + 5;
    return a;
}
```

引用的返回值类型

```c
int & valplus2(int &a)
{
    a = a + 5;
    return a;
}
```

从最终的效果来看，如上两种得到的结果都是一样的

```c
int main() {
   int num1 = 10;
    int num2 = 0;
    num2 = valplus1(num1);
    cout<<num1<<" "<<num2<<endl;
    
    num1 = 10;
    num2 = 0;
    num2 = valplus1(num1);
    cout<<num1<<" "<<num2<<endl;
    
    return 0;
}
```

```
15 15
15 15
```

但是还是有区别的，相同点是返回值都是拷贝的

- valplus1()使用到普通返回值方式:
	- 先将`要返回的结果值` 拷贝 到一个`临时存储空间`
	- 然后再从临时存储空间拷贝给num2变量

- valplus2()使用到`引用`返回值方式:
	- 直接将`要返回的结果值` 拷贝给 `num2`的，中间没有经过拷贝给临时空间

	
- 可能获取不到返回值的例子
	- 变量b的作用域仅在这个valplus3()函数体`内部`
	- 当函数调用完成，b变量就会被销毁
	- 此时若将b变量的值通过`引用`的方式，返回并将要拷贝给变量num2时
	- 有可能会出现在拷贝之前b变量已经被销毁
	- 从而导致num2变量获取不到返回值

```c
int & valplus3(int a)
{
    int b = a+5;
    return b;
}
```

##C++ 强制类型转换

- C语言中的野蛮类型转换

```c
int a = 10;
int b = 3;
double result = (double)a / (double)b;
```

- C++中使用 `static_cast<<强转到的类型>>(变量)` 进行强制转换

```c
int a = 10;
int b = 3;
double result = static_cast<double>(a) / static_cast<double>(b);
```

- 使用const_cast对const修饰的类型去掉const修饰

```c
// 常量整形a
const int a = 10;

// 常量指针指向a
const int * p = &a;
    
//编译报错，不允许改变常量指针指向的值
*p = 20;       

//编译报错，不允许将常量进行去掉const修饰
int b = const_cast<int>(a);  
```

虽然不允许直接对`常量`进行去掉const修饰，但是可以对`指向的常量指针`进行去掉const修饰，从而修改指向的值

```c
// 常量整形a
const int a = 10;

// 常量指针指向a
const int * p = &a;

// 对`指向的常量指针`进行去掉const修饰
int *b = const_cast<int *>(p);

// 修改指向的值
*b = 20;
```

再整合起来看如上代码的一个小问题

```c
const int a = 10;
const int *p = &a;
int *q;
q = const_cast<int *>(p);
*q = 20;
cout <<a<<" "<<*p<<" "<<*q<<endl;
cout <<&a<<" "<<p<<" "<<q<<endl;
```

输出结果

```
10 20 20
0x7fff5fbff760 0x7fff5fbff760 0x7fff5fbff760
```

很奇怪，三个地址都是一样的，但是a的值还是10，p和q指向的是20，那么原因: 变量a一开始就被声明为一个常量变量，不管后面的程序怎么处理，它就是一个常量，就是不会变化的.

> 少用const_cast<int>(a)去掉指针或引用的常量性并且去修改原始变量的数值.

- reinterpret_cast主要用途
	- 改变指针或引用的类型
	- 将指针或引用转换为一个`足够长度`的`整形`
	- 将`整型`转换为指针或引用类型

```c
// 整形变量指针
int *a = new int;

// 转换成双精度变量指针
double *d = reinterpret_cast<double *>(a);
```

- dynamic_cast用于类的继承层次之间的强制类型转换

##内联函数

- 内联函数的特点:
	- 关键字 inline 修饰的函数
	- 在程序`编译`时，编译器会将内联函数调用处用函数体替换，这一点类似于C语言中的宏扩展
	- 但是区别于宏的是，会做`类型检查`语法检查，而宏没有类型检查只是`纯粹的代码替换`

	
```c
inline void swap(int &a, int &b)
{
    int temp = a;
    a = b;
    b = temp;
}
```

内联函数通常只应该做一些简单的逻辑代码。

##C++ new和delete操作符

在C语言中，动态分配和释放内存的函数是malloc、calloc和free.

在C++语言中，使用new、new[]、delete和delete[]来代替.

```c
int *p = new int;
```

```c
int *A = new int[10];
```

```c
delete p;
```

```c
delete[] p;
```

##异常处理

主要分为: 抛出异常 和 捕捉异常.

- 捕捉异常代码块

```c
try
{
    //可能抛出异常的语句
}
catch (异常类型1)
{
    //异常类型1的处理程序
}
catch (异常类型2)
{
    //异常类型2的处理程序
}
// ……
catch (异常类型n)
{
    //异常类型n的处理程序
}

//注意: C++中没有finally块
```

- 结合抛出异常和捕捉异常的例子

```c
#include<iostream>
using namespace std;

// 异常的错误码
enum index{underflow, overflow};

// 函数内部抛出可能发生的异常
int array_index(int *A, int n, int index)
{
    if(index < 0) throw underflow;
    if(index > n-1) throw overflow;
    return A[index];
}

// main函数捕捉异常
int main()
{
    int *A = new int[10];
    for(int i=0; i<10; i++)
        A[i] = i;
    try
    {
        cout<<array_index(A,10,5)<<endl;
        cout<<array_index(A,10,-1)<<endl;
        cout<<array_index(A,10,15)<<endl;
    }
    catch(index e)
    {
        if(e == underflow)
        {
            cout<<"index underflow!"<<endl;
            exit(-1);
        }
        if(e == overflow)
        {
            cout<<"index overflow!"<<endl;
            exit(-1);
        }
    }
    return 0;
}
```

##类的定义模板

```c
class Student {
    int sid;
    char name[100];
    
public:
    
    void test1();
    void test2();
};

void Student::test1() {
    cout<<"test method"<<endl;
}

void Student::test2() {
    cout<<"test method"<<endl;
}
```

##class类 和 struct结构体 的区别

###Class类

- 成员变量、成员函数可以分配`访问权限`属性
	- private、public、proteced

- 类可以完成: 继承、多态

###Struct结构体

- 成员变量、成员函数的访问权限`都是public`

- C++中的struct对C中的struct进行了扩充:
	- struct能包含`成员函数
	- struct能`继承`
	- struct能实现`多态`

	
那么也就是说C++中的struct基本上能完成class能完成的所有事情，那么最根本的区别是 >>> `默认的访问控制: struct是public的，class是private的`.


###所以Class与Struct最大的区别就是实例变量的访问权限:

- struct作为`数据结构`的实现体，它默认的数据访问控制是`public`的.
- class作为`对象`的实现体，它默认的成员变量访问控制是`private`的

> Class类可以继承自Struct结构体

- 结构体

```c
typedef struct Person {
    // 属性和方法访问权限都是public
    int pid;
    char name[20];
    
    void run() {
        printf("Person struct method\n");
    }
}Person;
```

- 类继承自结构体

```c
class Man : Person {
private:
    int pid;
    
public:
    void run() {
        
        // 访问继承自结构体的属性
        cout<<"继承自结构体的属性name = "<<this->Person::pid<<endl;

        // 访问继承自结构体的方法
        cout<<"继承自结构体的属性name = "<<this->name<<endl;
        this->Person::run();
        
        // 访问自己的属性和方法
        cout<<"类自己的属性pid = "<<this->pid<<endl;
        this->run();
    }
};
```

- struct结构体也可以继承自Class类

类

```c
class Person {

private:
    int pid;
    
public:
    string *name;
    
    void person_run() {
        //.....
    }
    
    Person() {
        
    }
    
    ~Person() {
        
    }
};
```

结构体

```c
typedef struct Man : Person {
    char addrs[100];
    
    void man_run() {
        // 访问结构体的属性和方法
        this->Person::pid;//这句会报错，因为类Person中的pid是private，只能内部对象使用
        this->Person::name;
        this->Person::person_run();
    }
}Man;
```


最后小结二者的区别:

-  1) struct更适合看成是一个`数据结构`的实现体
-  2) class更适合看成是一个`对象`的实现体
-  3) 二者的访问权限不同

##C++ 构造函数

- 如果含有const类型的成员变量时的初始化函数写法.

```c
class Array
{
public:

	// 这么写是报错的
    Array()
    {
        length = 0;
    };
private:
    const int length;
};
```

上面的写法会报错的，应该是下面的写法

```c
class Array
{
public:
    Array(): length(0)
    {
        //其他变量初始化
    };
    
    Array(int x): length(0)
    {
        //其他变量初始化
    };
    
    Array(int x, int y): length(0)
    {
        //其他变量初始化
    };
    
    Array(int x, int y, int z): length(0)
    {
        //其他变量初始化
    };
private:
    const int length;
};
```

- 带有默认值的构造函数写法

```c
class book
{
public:
    book(){}
    book(char* a, double p = 5.0);//默认参数值
    void display();
private:
    double price;
    char * title;
};
```

多数情况下，不应该使用带有默认值的构造参数.

- 限制一个类通过new调用构造函数来创建对象

```c
class Person {
private:
    int pid;
    
    // 限制使用默认的构造函数
    Person() {
        
    }
    
public:
    void test() {
        
    }
    
    // 必须使用指定的构造函数
    Person(int pid) {
        this->pid = pid;
    }

};



int main() {
    Person *p = new Person();//编译报错
    
    Person *p = new Person();//编译正确
    
    return 0;
}
```

##与static结合的成员变量和成员函数不再属于对象的了，而是属于`类`

> 静态成员变量属于类而不属于任何一个对象，如此一来可以实现多个对象之间的数据共享功能.

```c
#include<iostream>
using namespace std;
class test
{
public:
    static int num;
};
int test::num = 1;
int main()
{
    test one;
    test two;
    test three;
    cout<<test::num<<" "<<one.num<<" "<<two.num<<" "<<three.num<<endl;
    test::num = 5;
    cout<<test::num<<" "<<one.num<<" "<<two.num<<" "<<three.num<<endl;
    one.num = 8;
    cout<<test::num<<" "<<one.num<<" "<<two.num<<" "<<three.num<<endl;
    two.num = 4;
    cout<<test::num<<" "<<one.num<<" "<<two.num<<" "<<three.num<<endl;
    three.num = 2;
    cout<<test::num<<" "<<one.num<<" "<<two.num<<" "<<three.num<<endl;
    return 0;
}
```

运行结果

```
1 1 1 1
5 5 5 5
8 8 8 8
4 4 4 4
2 2 2 2
```

##继承、子类覆盖父类的成员函数时（构造函数继承）

```c
class Man {
private:
    int mid;
    
public:
    
    Man() {
        cout<<"Man 构造函数..."<<endl;
    }
    
    void test() {
        cout<<"Man test函数..."<<endl;
    }
};
```

```c
class Person : Man {
private:
    int pid;
    
public:
    void test() {
        //1.
        cout<<"Person test函数..."<<endl;
        
        //2.
        this->Man::test();
    }
    
    Person() : Man() {
        cout<<"Person 构造函数..."<<endl;
    }
};
```

```c
int main() {

    Person *person = new Person();
    person->test();

    return 0;
}
```

结果

```
Man 构造函数...
Person 构造函数...
Person test函数...
Man test函数...
```

所以并不是`覆盖`，只是如果没执行`super`的方法，就只在当前子类查找方法，就不再去父类查找方法了。

