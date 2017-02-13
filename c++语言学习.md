## c/c++ 基本数据类型占用的字节长度

```c
char	1 个字节	-128 到 127 或者 0 到 255
unsigned char	1 个字节	0 到 255
signed char	1 个字节	-128 到 127
int	4 个字节	-2147483648 到 2147483647
unsigned int	4 个字节	0 到 4294967295
signed int	4 个字节	-2147483648 到 2147483647
short int	2 个字节	-32768 到 32767
unsigned short int	2 个字节	0 到 65,535
signed short int	2 个字节	-32768 到 32767
long int	8 个字节	-9,223,372,036,854,775,808 到 9,223,372,036,854,775,807
signed long int	8 个字节	-9,223,372,036,854,775,808 到 9,223,372,036,854,775,807
unsigned long int	8 个字节	0 to 18,446,744,073,709,551,615
float	4 个字节	+/- 3.4e +/- 38 (~7 个数字)
double	8 个字节	+/- 1.7e +/- 308 (~15 个数字)
long double	8 个字节	+/- 1.7e +/- 308 (~15 个数字)
wchar_t	2 或 4 个字节	1 个宽字符
```

枚举定义格式

```c
enum color {
    red = 1,
    green,
    blue
};
```

## 常量

### 基本数值类型

```c
85         // 十进制
0213       // 八进制 
0x4b       // 十六进制 
30         // 整数 
30u        // 无符号整数 
30l        // 长整数 
30ul       // 无符号长整数
```

### 浮点常量

```
3.14159       // 合法的 
314159E-5L    // 合法的 
510E          // 非法的：不完整的指数
210f          // 非法的：没有小数或指数
.e55          // 非法的：缺少整数或分数
```

### 字符常量

```c
\\	\ 字符
\'	' 字符
\"	" 字符
\?	? 字符
\a	警报铃声
\b	退格键
\f	换页符
\n	换行符
\r	回车
\t	水平制表符
\v	垂直制表符
\ooo	一到三位的八进制数
\xhh . . .	一个或多个数字的十六进制数
```

## c++11中的Lambada表达式（匿名函数、函数指针、block ...）

### 一段最简单的使用

```c
void testLambada1() {
    
    //1. 定义一个内部函数闭包
    auto func = []() {
        cout << "hello, world" << endl;
    };
    
    //2. 调用函数闭包
    func();
}
```

输出

```
hello, world
```

### 带有参数和返回值的Lambada表达式使用

Lambada表达式定义格式:

```c
[capture](parameters) -> return-type {
	body
};
```

```c
void testLambada2() {
    
    //1. 定义一个内部函数闭包
    auto func = [](int x, int y) ->int {
        return (x > y) ? x : y;
    };
    
    //2. 调用函数闭包
    int max = func(1, 2);
    printf("max = %d\n", max);
}
```

输出

```
max = 2
```

### Lambada表达式，访问外部的所有变量

Lambada表达式定义格式:

```c
[=](parameters) -> return-type {
	body
};
```

例子

```c
void testLambada3() {
    
    //1.
    int x = 1;
    int y = 2;
    
    //2.
    auto func = [=]() {
        printf("x = %d, y = %d\n", x, y);
    };

    //3.
    func();
}
```

输出

```
x = 1, y = 2
```

### Lambada表达式，捕获外部变量，分为两种方式: (1)传值 (2)传引用

对于Lambada表达式，捕获外部变量设置由如下几种形式:

```c
[]      // 不对外部变量进行捕获，即表达式内部不能使用外部的任何变量。

[=]     // 使用【值传递】，捕获外部所有变了。

[&]     // 使用【引用传递】，捕获外部所有变了。

[this]  // 可以使用Lambda所在类中的成员变量。

[x, &y] // x以传值方式传入（默认），y以引用方式传入。

[&, x]  // x显式地以传值方式加以引用。其余变量以引用方式加以引用。

[=, &z] // z显式地以引用方式加以引用。其余变量以传值方式加以引用。
```

测试一、`[=]`值传递捕获外部变量的存在的问题

```c
void testLambada4() {
    
    //1.
    int x = 1;
    int y = 2;
    
    //2.
    auto func = [=]() {
        printf("x = %d, y = %d\n", x, y);
    };
    
    //3. 对x,y的值进行修改
    x = 3;
    y = 4;
    
    //4.
    func();
}
```

输出

```
x = 1, y = 2
```

测试二、`[&x, y]`，x传引用，y传值

```c
void testLambada5() {
    
    //1.
    int x = 1;
    int y = 2;
    
    //2.
    auto func = [&x, y]() {
        printf("x = %d, y = %d\n", x, y);
    };
    
    //3.
    x = 3;
    y = 4;
    
    //4.
    func();
}
```

输出

```
x = 3, y = 2
```

## c++中的一些数学运算的函数

```c
1	double cos(double);
该函数返回弧度角（double 型）的余弦。

2	double sin(double);
该函数返回弧度角（double 型）的正弦。

3	double tan(double);
该函数返回弧度角（double 型）的正切。

4	double log(double);
该函数返回参数的自然对数。

5	double pow(double, double);
假设第一个参数为 x，第二个参数为 y，则该函数返回 x 的 y 次方。

6	double hypot(double, double);
该函数返回两个参数的平方总和的平方根，也就是说，参数为一个直角三角形的两个直角边，函数会返回斜边的长度。

7	double sqrt(double);
该函数返回参数的平方根。

8	int abs(int);
该函数返回整数的绝对值。

9	double fabs(double);
该函数返回任意一个十进制数的绝对值。

10	double floor(double);
该函数返回一个小于或等于传入参数的最大整数。
```

### c++中的string的常用操作函数

```c
1	strcpy(s1, s2);
复制字符串 s2 到字符串 s1。

2	strcat(s1, s2);
连接字符串 s2 到字符串 s1 的末尾。

3	strlen(s1);
返回字符串 s1 的长度。

4	strcmp(s1, s2);
如果 s1 和 s2 是相同的，则返回 0；如果 s1<s2 则返回小于 0；如果 s1>s2 则返回大于 0。

5	strchr(s1, ch);
返回一个指针，指向字符串 s1 中字符 ch 的第一次出现的位置。

6	strstr(s1, s2);
返回一个指针，指向字符串 s1 中字符串 s2 的第一次出现的位置。
```

## c++中的日期&时间

### 获取当前系统的日期和时间，包括本地时间和协调世界时（UTC）


```c
#include <ctime>

void testTime1() {
    
    // 基于当前系统的当前日期/时间
    time_t now = time(0);
    
    // 把 now 转换为字符串形式
    char* dt = ctime(&now);
    
    cout << "本地日期和时间：" << dt << endl;
    
    // 把 now 转换为 tm 结构
    tm *gmtm = gmtime(&now);
    dt = asctime(gmtm);
    cout << "UTC 日期和时间："<< dt << endl;
}
```

输出信息

```
本地日期和时间：Sun Feb 12 15:16:12 2017

UTC 日期和时间：Sun Feb 12 07:16:12 2017
```

### 使用结构 tm 格式化时间

结构类型 tm 把日期和时间以 C 结构的形式保存，tm 结构的定义如下：

```c
struct tm {
  int tm_sec;   // 秒，正常范围从 0 到 59，但允许至 61
  int tm_min;   // 分，范围从 0 到 59
  int tm_hour;  // 小时，范围从 0 到 23
  int tm_mday;  // 一月中的第几天，范围从 1 到 31
  int tm_mon;   // 月，范围从 0 到 11
  int tm_year;  // 自 1900 年起的年数
  int tm_wday;  // 一周中的第几天，范围从 0 到 6，从星期日算起
  int tm_yday;  // 一年中的第几天，范围从 0 到 365，从 1 月 1 日算起
  int tm_isdst; // 夏令时
}
```

测试

```c
#include <ctime>

void testTime2() {
    
    //1. 基于当前系统的当前日期/时间
    time_t now = time(0);
    
    cout << "Number of sec since January 1,1970:" << now << endl;
    
    //2. 获取tm结构体实例
    tm *ltm = localtime(&now);
    
    //3. 输出 tm 结构的各个组成部分
    cout << "Year: "<< 1900 + ltm->tm_year << endl;
    cout << "Month: "<< 1 + ltm->tm_mon<< endl;
    cout << "Day: "<<  ltm->tm_mday << endl;
    cout << "Time: "<< 1 + ltm->tm_hour << ":";
    cout << 1 + ltm->tm_min << ":";
    cout << 1 + ltm->tm_sec << endl;
}
```

输出信息

```
Number of sec since January 1,1970:1486883972
Year: 2017
Month: 2
Day: 12
Time: 16:20:33
```

## c++中的运算符重载

```c
#include <iostream>
using namespace std;

class Box
{
   public:

      double getVolume(void)
      {
         return length * breadth * height;
      }
      void setLength( double len )
      {
          length = len;
      }

      void setBreadth( double bre )
      {
          breadth = bre;
      }

      void setHeight( double hei )
      {
          height = hei;
      }
      
      // 重载 + 运算符，用于把两个 Box 对象相加
      Box operator+(const Box& b)
      {
         Box box;
         box.length = this->length + b.length;
         box.breadth = this->breadth + b.breadth;
         box.height = this->height + b.height;
         return box;
      }
      
   private:
      double length;      // 长度
      double breadth;     // 宽度
      double height;      // 高度
};
```

测试

```c
int main( )
{
   Box Box1;                // 声明 Box1，类型为 Box
   Box Box2;                // 声明 Box2，类型为 Box
   Box Box3;                // 声明 Box3，类型为 Box
   double volume = 0.0;     // 把体积存储在该变量中
 
   // Box1 详述
   Box1.setLength(6.0); 
   Box1.setBreadth(7.0); 
   Box1.setHeight(5.0);
 
   // Box2 详述
   Box2.setLength(12.0); 
   Box2.setBreadth(13.0); 
   Box2.setHeight(10.0);
 
   // Box1 的体积
   volume = Box1.getVolume();
   cout << "Volume of Box1 : " << volume <<endl;
 
   // Box2 的体积
   volume = Box2.getVolume();
   cout << "Volume of Box2 : " << volume <<endl;

   //【重要】测试运算符重载：把两个对象相加，得到 Box3
   Box3 = Box1 + Box2;

   // Box3 的体积
   volume = Box3.getVolume();
   cout << "Volume of Box3 : " << volume <<endl;

   return 0;
}
```

输出信息

```
Volume of Box1 : 210
Volume of Box2 : 1560
Volume of Box3 : 5400
```

不可以被重载的运算符:

```
- (1) ::
- (2) .*
- (3) .
- (4) ?:
```

对象的`++`，运算符重载例子:（对象++）

```c
Time operator++ ()  
{
 ++minutes;          // 对象加 1
 if(minutes >= 60)  
 {
    ++hours;
    minutes -= 60;
 }
 return Time(hours, minutes);
}
```

对象的`=`，赋值运算符重载例子: (对象1 = 对象2)

```c
void operator=(const Distance &D )
{ 
 feet = D.feet;
 inches = D.inches;
}
```

数组的`[i]`，赋值运算符重载例子: 

```c
int& operator[](int i)
{
  if( i > SIZE )
  {
      cout << "索引超过最大值" <<endl; 
      // 返回第一个元素
      return arr[0];
  }
  return arr[i];
}
```


## 传指针变量的地址 VS 传指针变量的引用


### c++中的`基本类型`变量，传递引用

```c
void func1(int &x) {
    x++;
}

void test1() {
    int x = 1;
    func1(x);
    printf("x = %d\n", x);
}
```

输出结果

```
x = 2
```

### c++中的`指针`类型变量，传递引用

```c
typedef struct Person {
    int     age;
}Person;

void func2(Person *&p) {
    p = (Person *)malloc(sizeof(Person));
    p->age = 20;
}

void test2() {
    
    Person *p = (Person *)malloc(sizeof(Person));
    p->age = 19;
    printf("%p - %d\n", p, p->age);
    
    func2(p);
    printf("%p - %d\n", p, p->age);
}
```

`test1()`方法输出结果

```
0x1001031a0 - 19
0x1001003b0 - 20
```

p指向的是两个不同的内存地址。


> 在c语言中，没有引用。只能够通过传递地址的方式。


```c
typedef struct Person {
    int     age;
}Person;

void func2(Person *&p) {
    p = (Person *)malloc(sizeof(Person));
    p->age = 20;
}

void func3(Person **p) {
    *p = (Person *)malloc(sizeof(Person));
    (*p)->age = 21;
}

void test2() {
    
    Person *p = (Person *)malloc(sizeof(Person));
    p->age = 19;
    printf("%p - %d\n", p, p->age);
    
    func2(p);//传递指针变量的【引用】
    printf("%p - %d\n", p, p->age);
    
    func3(&p);//传递指针变量的【地址】
    printf("%p - %d\n", p, p->age);
}
```

输出信息

```
0x1002017c0 - 19
0x1002017d0 - 20
0x1002017e0 - 21
```

都不是一个地址，但可以看到c++的写法比c的写法简洁一些。

下面的写法是错误的:

```c
typedef struct Person {
    int     age;
}Person;

void func4(Person *p) {
    p = (Person *)malloc(sizeof(Person));
    p->age = 21;
}

void test2() {
    
    Person *p = (Person *)malloc(sizeof(Person));
    p->age = 19;
    printf("%p - %d\n", p, p->age);
    
        func4(p);
    printf("%p - %d\n", p, p->age);
}
```

输出信息

```
0x1004005a0 - 19
0x1004005a0 - 19
```

可以看到，func4()实际上没有修改外部的Person实例的地址。

### 所以，如果需要在函数内修改外部传入的变量值，必须:

- (1) c语言、传递变量的【地址】
- (2) c语言、传递变量的【引用】

而如果不需要修改，只需要读取，那么直接传递【变量本身】即可。

### 数组作为方法形参时的写法

> 数组名就是数组的起始地址，实际上就是传递的【地址】。所以对于数组变量来说，没有【引用】与【非引用】之分。

一唯数组作为形参的标准写法:

```c
void f(int arr[], int n) 
{
	....
}
```

其中arr表示数组起始地址，n表示对arr数组中要进行处理的元素总个数，并不是数组总长度。

二唯数组作为形参的标准写法:

```c
void f(int x[][二维长度], int n) 
{

}
```

## c++中的函数重载

> c语言中，是没有的。

```c
void func5(int x)
{
    
}

void func5(int x, int y)
{
    
}

// 这个重载是错误的，编译器会报错，必须参数列表也不同
int func5(int x, int y)
{
    
}

int func5(int x, int y, int z)
{
    return 0;
}
```

不要写太多函数重载，影响代码阅读。

## c/c++，定义struct、初始化struct模板

```c
#include <iostream>

typedef struct Man {
    std::string name;
    std::string uid;
    char sex;
}Man;
```

```c
void test3() {

    //1. 方法块所在栈上，进行创建实例
    Man m1 = {0};
    Man m2 = {"名字", "001", 'm'};
    
    //2. 在堆区，进行创建实例
    Man *m3 = (Man *)malloc(sizeof(Man));
    free(m3);//堆区的，必须最后手动废弃
}
```

## c/c++，定义struct数组的模板

```c
#include <iostream>

typedef struct Man {
    std::string name;
    std::string uid;
    char sex;
}Man;
```

```c
void test4() {
    
    //1. 栈实例
    Man m1 = {"名字", "001", 'm'};
    
    //2. 堆实例
    Man *m2 = (Man *)malloc(sizeof(Man));
    
    //3. 存储栈实例的static数组
    static Man arr1[3] = {0};
    arr1[0] = m1;
    arr1[1] = m1;
    arr1[2] = m1;
    
    //4. 存储堆实例的static数组
    static Man *arr2[3] = {0};
    arr2[0] = m2;
    arr2[1] = m2;
    arr2[2] = m2;
    
    //5. 存储堆实例的堆区数组
    Man **arr3 = (Man **)malloc(sizeof(Man) * 3);
    arr3[0] = m2;
    arr3[1] = m2;
    arr3[2] = m2;
}
```

## c/c++，2个不同的struct占用同一个内存空间，可以互相转换

```c
typedef struct Dog {
    int pid;//四个字节
}Dog;

typedef struct Cat {
    int cid;//四个字节
    int sex;//四个字节
}Cat;
```

下面这个c函数，实际上是创建了Dog实例和Cat实例，但是返回的是两个实例占用的存储空间的起始地址，也就是Dog实例的地址。


```c
Dog* getDogInstance() {
    
    //1. 计算占用内存空间的总长度
    size_t len1 = sizeof(Dog);
    size_t len2 = sizeof(Cat);
    size_t size = len1 + len2;
    printf("【create】len1 = %ld, len2 = %ld, size = %ld\n", len1, len2, size);
    
    //2. 分配内存空间
    void *ins = calloc(1, size);
    printf("【create】整个长度内存空间起始地址: %p\n", ins);
    
    //3. 将内存空间的，前半部分（1~k），作为Dog类型的存储空间
    Dog *dog = (Dog *)ins;
    dog->pid = 111;
    printf("【create】dog = %p, pid = %d\n", dog, dog->pid);
    
    //4. 将内存空间的，剩下的部分（k+1~ins），作为Cat类型的存储空间
    Cat *cat = (Cat *)(dog + 1);
    cat->cid = 222;
    cat->sex = 'M';
    printf("【create】cat = %p, cat.cid = %d, cat.sex = %c\n", cat, cat->cid, cat->sex);
    
    //5. 注意: 先对哪一种类型转换，就决定了谁先谁后存放
    
    return dog;
}
```

测试函数，先读取Dog实例，然后通过`（Dog实例地址+1）`获取Cat实例:

```c
void test5() {
    
    //1. 先按照返回的Dog类型取值
    Dog *dog = getDogInstance();
    printf("【test】dog = %p, dog.pid = %d\n", dog, dog->pid);
    
    //2. 继续读取后半部分的Cat实例数据
    Cat *cat = (Cat *)(dog + 1);
    printf("【test】cat = %p, cat.cid = %d, cat.sex = %c\n", cat, cat->cid, cat->sex);
}
```

输出结果

```
【create】len1 = 4, len2 = 8, size = 12
【create】整个长度内存空间起始地址: 0x100102360

【create】dog = 0x100102360, pid = 111
【create】cat = 0x100102364, cat.cid = 222, cat.sex = M

【test】dog = 0x100102360, dog.pid = 111
【test】cat = 0x100102364, cat.cid = 222, cat.sex = M
```

## c++与c中的struct功能有所不同

### C++中的struct对C中的struct进行了扩充:

- (1) struct能包含`成员函数`
- (2) struct能`继承`
- (3) struct能实现`多态`

实际上就与C++中的Class就很相似了的，但还是有区别的。

## c++中的struct与Class的不同与相同之处

Class:

- (1) 成员变量、成员函数，都可以分配`访问权限`属性
	- private、public、proteced....

- (2) 类可以完成: 继承、多态


struct

- (1) 成员变量、成员函数的访问权限`都是public`，不支持其他访问权限

- (2) struct 同样可以完成: 继承、多态

也就是说`C++中的struct`基本上能完成`class`能完成的所有事情，那么最根本的区别是，对于成员变量、成员函数的权限访问控制:

- (1) struct、只能都是public `一种`
- (2) class、可以支持如下`几种`
	- protected
	- private
	- public


## class类可以继承自struct结构体

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

## struct结构体也可以继承自class类

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

### 最后小结`c++（c语言不支持的）`中的struct与class的区别:

-  1) struct更适合看成是一个`数据结构`的实现体
-  2) class更适合看成是一个`对象`的实现体
-  3) 二者的访问权限不同

## c++中的【构造】函数

### 构造函数，与其他功能函数的区别

- (1) 构造函数的名字，必须与类名一致
- (2) 系统在创建一个Class对象时，会第一时间调用构造函数
- (3) 构造函数，不能写返回值


### 不同类型的构造函数写法

- (1) 如果含有 `const类型的成员变量` 时的初始化函数写法.

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

- (2) 带有默认值的构造函数写法

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

- (3) 限制一个类通过new调用构造函数来创建对象

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
```

```c
int main() {
    Person *p = new Person();//编译报错
    
    Person *p = new Person();//编译正确
    
    return 0;
}
```

### 构造函数的继承

```c
class Base1          // 基类Base1，只有默认构造函数
{
public:
    Base1()         { cout<<"Base1 construct"<<endl; }
};
```

```c
class Base2          // 基类Base2，只有带参数的构造函数
{
public:
    Base2(int x)    { cout<<"Base2 construct "<<x<<endl; }
};
```

```c
class Base3          // 基类Base3，只有带参数的构造函数
{
public:
    Base3(int y)    { cout<<"Base3 construct "<<y<<endl; }
};
```

```c
class Child : public Base2, public Base1, public Base3   // 派生类Child
{
public:
    Child(int i,int j,int k,int m):Base2(i),b3(j),b2(k),Base3(m)    { }
private:             // 派生类的内嵌对象成员
    Base1 b1;
    Base2 b2;
    Base3 b3;
};
```

测试

```c
Child child(3,4,5,6);
```

输出

```
Base2 construct 3
Base1 construct
Base3 construct 6
Base1 construct
Base2 construct 5
Base3 construct 4
```

## c++中的【析构】函数


```c
class Book {
    ~Book(){/* 释放占用的各种资源 */};
};
```

不能有任何的形参列表，前面加一个`~`符号。

## c++中的访问控制

| 级别 | 允许谁来访问 |
| :------------- |:-------------:| 
| public | 任何代码 |
| protected | 这个类本身、`这个类的所有子类` |
| private | 只有这个类本身 |


## c++中的继承时，权限控制

继承时，使用权限的格式为如下:

```c
class 派生类名:［继承方式］ 基类名{
    派生类新增加的成员
};
```

包括 public（公有的）、private（私有的）和 protected（受保护的）。此项是可选项，如果不写，默认为 `private`（成员变量和成员函数默认也是 private）。


### public继承方式（一般使用此种继承权限）

- 基类中所有 public 成员在派生类中为 public 属性；
- 基类中所有 protected 成员在派生类中为 protected 属性；
- 基类中所有 private 成员在派生类中不能使用。

### protected继承方式

- 基类中的所有 public 成员在派生类中为 protected 属性；
- 基类中的所有 protected 成员在派生类中为 protected 属性；
- 基类中的所有 private 成员在派生类中不能使用。

### private继承方式

- 基类中的所有 public 成员在派生类中均为 private 属性；
- 基类中的所有 protected 成员在派生类中均为 private 属性；
- 基类中的所有 private 成员在派生类中不能使用。

### 图示总结

| 继承方式/基类成员	 | public成员 | protected成员 | private成员 |
| :-------------: |:-------------:| :----------:| :----------:|
| public继承 | public | protected | 不可见 |
| protected继承 | `protected` | protected  | 不可见 |
| private继承 | `private`  |  `private` | 不可见 |


类成员的访问权限由高到低依次为 public --> protected --> private。

## c++中的继承、子类覆盖父类的成员函数，需要手动调用父类的函数。但是对于构造器函数即使重写，会自动调用父类的构造函数。

父类

```c
class Human {
private:
    int mid;
    
public:
    
    Human() {
        std::cout << "Human 构造函数..." << std::endl;
    }
    
    void running() {
        std::cout << "Human running..." << std::endl;
    }
};
```

子类

```c
class XiaoMing : Human {
public:
    
    // 重写父类的方法实现
    void running() {
        
        //1. 先调用父类的方法
        Human::running();

        //2. 再写子类的附加的代码逻辑
        std::cout << "XiaoMing running..." << std::endl;
    }
    
    // 重写构造函数，也是一样继承父类
    XiaoMing() {//就相当于下面的代码
//    XiaoMing() : Human() {
        std::cout << "XiaoMing 构造函数..." << std::endl;
    }
};
```

测试

```c
XiaoMing *xiaoming = new XiaoMing;
xiaoming->running();
```

输出信息

```
Human 构造函数...
XiaoMing 构造函数...
Human running...
XiaoMing running...
```

子类重写并不是`覆盖`，只是如果`没执行父类的方法`，就只在当前子类查找方法，就不再去父类查找方法了。

## 友元、访问某个类的私有成员变量、私有成员函数

可以把一个函数指定为类的友元，也可以把整个类指定为另一个类的友元。

### 将一个非成员函数 reset( ) 声明为类 example 的友元函数，使得该非成员函数可以访问类 example 的私有成员.

```c
//1. 
class example; // 这里必须对类 example 做前向声明，否则下面的函数声明将报错

//2. 
void reset(example &e);  

//3.
class example  
{  
public:  

	//4. 在class内部，定义友元函数
    friend void reset(class example &e);  
private:  
    int pid;  
};  

//5. 该函数定义必须放在类 example 的后面，否则无法解析变量pid 
void reset(example &e)  
{  
    e.pid = 111;  
}  
```

### 将类man声明为类woman的友元类，使得可以通过类man对象访问类woman的私有成员

```c
//1. 
class woman; // 前向声明  

//2.
class man  
{  
public:  
    void disp(woman &w);  
    void reset(woman &w);  
};  
  
//3.
class woman  
{  
public:  

	//4. 将man设为woman的友元类，这样man对象的任何成员函数都可以访问woman的私有成员
    friend class man;   
    
private:  
    string name;  
};  

//5. man对象的任何成员函数都可以访问woman的私有成员
void man::disp(woman &w)  
{  
    cout << w.name << endl;  
}  
  
void man::reset(woman &w)  
{  
    w.name.clear();  
}  
```

### 将一个类man的成员函数disp()声明为类woman的友元函数，使得可以通过disp成员函数访问类woman的私有成员（上面是完全暴露，这个是选择性暴露）

```c
//1. 
class woman; // 前向声明  
  
//2.
class man  
{  
public:  
    void disp(woman &w);  
    void reset(woman &w);  
};  
  
//3.
class woman  
{  
public:  

	//4. 将man的其中一个成员函数disp()设为woman的友元函数，就可以使用该函数访问woman对象的私有成员了
    friend void man::disp(woman &w);   
  
private:  
    string name;  
};  

//5. man使用woman中的私有成员
void man::disp(woman &w)  
{  
    cout << w.name << endl;  
}  
  
//6. man的reset()成员函数不是woman类的友元函数，因此不能访问其私有成员  
/* 
void man::reset(woman &w) 
{ 
    w.name.clear(); 
} 
*/  
```

不管是友元类、友元成员函数还是友元非成员函数，都必须是`public`访问权限的，否则无法在类外被调用。

最后就是，少用友元，会破坏面向对象的结构。

## c++中的，静态成员变量、静态成员函数

### c++中的静态成员与c中的静态是不同的

和c语言里面的static属性是不一样的，c语言中的static既不属于类，也不属于对象，就是作为一个公共的内存数据块存在。

在C++中，静态成员是属于整个类的而不是某个对象，静态成员变量只存储一份供所有对象共用，静态成员可以通过双冒号来使用即 `<类名>::<静态成员名>`，比如`Dog::name或Dog::func()`。

比如，如下的例子:

```c
class HashiQiDog {
public:
    
    // 使用static声明的方法，属于类的方法
    static int getUID();

    // 使用static声明的方法，属于类的方法
    static void setUID(int uid);
    
    // 对象的函数
    void cry();
    
private:
    
    // 使用static声明的属性，属于类的属性
    static int _uid;
};

int HashiQiDog::getUID() {
    return _uid;
}

void HashiQiDog::setUID(int uid) {
    _uid = uid;
}

void HashiQiDog::cry() {
    printf("uid = %d\n", HashiQiDog::getUID());
}

//【重要】static成员变量，必须要进行初始化，否则编译器报错
int HashiQiDog::_uid = 0;

int main(int argc, const char * argv[]) {
    
    //1.
    HashiQiDog::setUID(100);
        
    //2.
    //HashiQiDog::cry();//报错的，cry()是对象函数
    
    //3.
    HashiQiDog dog;
    dog.cry();
    
    return 0;
}
```

输出

```
uid = 100
```

## c++中的 虚方法 （virtual method）

就相当于抽象协议中定义的抽象方法，让具体类去实现自己的具体代码。

```c
class Father {
public:
    virtual void run() {
        printf("father run ... \n");
    }
};
```

```c
class Son1 : public Father {
public:
    virtual void run() {
        printf("son1 run ... \n");
    }

};
```

```c
class Son2 : public Father {
public:
    virtual void run() {
        
        //1.
        Father::run();
        
        //2.
        printf("son2 run ... \n");
    }

};
```

```c
void testVirtual() {
    
    //1.
    Son1 s1;
    Son2 s2;
    
    //2.
    s1.run();
    s2.run();
}
```

```
son1 run ... 
father run ... 
son2 run ... 
```

## c++中的 抽象类

### 如果类中至少有一个函数被声明为`纯虚函数`，则这个类就是抽象类

纯虚函数是通过在声明中使用 "= 0" 来指定的，如下所示:

```c
class Box
{
   public:
      
      // 纯虚函数声明与定义:
      virtual double getVolume() = 0;
      
   private:
      double length;      // 长度
      double breadth;     // 宽度
      double height;      // 高度
};
```

### 抽象类类似抽象协议，然后具体的类去继承实现所有的函数

基类

```c
class Shape 
{
public:

   // 纯虚函数声明与定义
   virtual int getArea() = 0;
   
   void setWidth(int w)
   {
      width = w;
   }
   void setHeight(int h)
   {
      height = h;
   }
protected:
   int width;
   int height;
};
```

子类1

```c
class Rectangle: public Shape
{
public:
   int getArea()
   { 
      return (width * height); 
   }
};
```

子类2

```c
class Triangle: public Shape
{
public:
   int getArea()
   { 
      return (width * height)/2; 
   }
};
```

可以看到，子类分别实现抽象父类中定义的纯虚函数即可。


## c++中的多态

父类

```c
class Shape {
   protected:
      int width, height;
   public:
      Shape( int a=0, int b=0)
      {
         width = a;
         height = b;
      }
      int area()
      {
         cout << "Parent class area :" <<endl;
         return 0;
      }
};
```

子类1

```c
class Rectangle: public Shape{
   public:
      Rectangle( int a=0, int b=0):Shape(a, b) { }
      int area ()
      { 
         cout << "Rectangle class area :" <<endl;
         return (width * height); 
      }
};
```

子类2

```c
class Triangle: public Shape{
   public:
      Triangle( int a=0, int b=0):Shape(a, b) { }
      int area ()
      { 
         cout << "Triangle class area :" <<endl;
         return (width * height / 2); 
      }
};
```

多态使用

```c
int main( )
{
	//1. 
   Shape *shape;
   
   //2.
   Rectangle rec(10,7);
   Triangle  tri(10,5);
   
   //3.
   shape = &rec;
   
   //4.
   shape = &tri;
   
   //5.
   .....
}
```

## C++ 异常处理

`try-cache`模板写法

```c
try
{
   // 保护代码
}catch( ExceptionName e1 )
{
   // catch 块
}catch( ExceptionName e2 )
{
   // catch 块
}catch( ExceptionName eN )
{
   // catch 块
}
```

抛出异常

```c
double division(int a, int b)
{
   if( b == 0 )
   {
   	  // 抛出异常
      throw "Division by zero condition!";
   }
   return (a/b);
}
```

## 定义自己的异常

```c
#include <iostream>
#include <exception>
using namespace std;

struct MyException : public exception
{
  const char * what () const throw ()
  {
    return "C++ Exception";
  }
};
```

使用自定义的异常

```c
int main()
{
  try
  {
    throw MyException();
  }
  catch(MyException& e)
  {
    std::cout << "MyException caught" << std::endl;
    std::cout << e.what() << std::endl;
  }
  catch(std::exception& e)
  {
    //其他的错误
  }
}
```

输出

```
MyException caught
C++ Exception
```

## c++中的动态内存

### new 和 delete

一个是创建对象，另一个废弃对象。

`malloc()` 函数在 C 语言中就出现了，在 C++ 中仍然存在，但建议尽量不要使用 `malloc()` 函数。new 与 `malloc()` 函数相比，其主要的优点是，new 不只是分配了内存，它还创建了对象。

### 基本数据类型数组操作

创建数组

```c
char* pvalue  = NULL;   // 初始化为 null 的指针
pvalue  = new char[20]; // 为变量请求内存
```

删除数组

```c
delete [] pvalue;        // 删除 pvalue 所指向的数组
```

对基本数据类型的二维数组操作

```c
//1. 2行、3列的二维数组
int ROW = 2;
int COL = 3;

//2. 为行分配内存
double **pvalue  = new double* [ROW]; 

//3. 为列分配内存
for(int i = 0; i < COL; i++) {
    pvalue[i] = new double[COL];
}

//4. 释放多维数组内存
for(int i = 0; i < COL; i++) {//先释放列
    delete[] pvalue[i];
}
delete [] pvalue;//再释放行
```


### 自定义class对象数组的操作

```c
class Box
{
   public:
      Box() { 
         cout << "调用构造函数！" <<endl; 
      }
      ~Box() { 
         cout << "调用析构函数！" <<endl; 
      }
};
```

```c
int main( )
{
	//1.
   Box* myBoxArray = new Box[4];

	//2.
   delete [] myBoxArray; // Delete array

   return 0;
}
```

## 命名空间

### 定义自己的命名控制，防止方法名重名，也可以是为已有的命名空间增加新的元素

```c
namespace namespace_name {
   // 代码声明
}
```

调用带有命名空间的函数或变量

```c
name::code;  // code 可以是变量或函数
```

### 嵌套的命名空间

```c
namespace namespace_name1 {
   // 代码声明
   namespace namespace_name2 {
      // 代码声明
   }
}
```

使用

```c
// 访问 namespace_name2 中的成员
using namespace namespace_name1::namespace_name2;

// 访问 namespace:name1 中的成员
using namespace namespace_name1;
```

## c++中的模板

### 模板是什么

模板是泛型编程的基础，泛型编程即以一种独立于任何特定类型的方式编写代码。

比如 vector <int> 或 vector <string>。


### 基本数据类型的模板

下面是函数模板的使用例子，返回两个数种的最大值

```c
#include <string>

template <typename T>
inline T const& Max (T const& a, T const& b)
{
    return a < b ? b:a;
}
```

使用模板函数

```c
void testTemplate1() {
    
    //1. int类型
    int i = 39;
    int j = 20;
    cout << "Max(i, j): " << Max(i, j) << endl;
    
    //2. double类型
    double f1 = 13.5;
    double f2 = 20.7;
    cout << "Max(f1, f2): " << Max(f1, f2) << endl;
    
    //3. string类型
    string s1 = "Hello";
    string s2 = "World";
    cout << "Max(s1, s2): " << Max(s1, s2) << endl;
}
```

### class自定义类的模板

```c
#include <vector>

template <class T>
class Stack { 
  private: 
    vector<T> elems;     // 元素 

  public: 
    void push(T const&);  // 入栈
    void pop();               // 出栈
    T top() const;            // 返回栈顶元素
    bool empty() const{       // 如果为空则返回真。
        return elems.empty(); 
    } 
}; 

template <class T>
void Stack<T>::push (T const& elem) 
{ 
    // 追加传入元素的副本
    elems.push_back(elem);    
} 

template <class T>
void Stack<T>::pop () 
{ 
    if (elems.empty()) { 
        throw out_of_range("Stack<>::pop(): empty stack"); 
    }
	// 删除最后一个元素
    elems.pop_back();         
} 

template <class T>
T Stack<T>::top () const 
{ 
    if (elems.empty()) { 
        throw out_of_range("Stack<>::top(): empty stack"); 
    }
	// 返回最后一个元素的副本 
    return elems.back();      
} 
```

使用代码

```c
//1. 操作 int 类型的栈 
Stack<int>         	intStack; 
intStack.push(7); 
cout << intStack.top() <<endl; 

//2. 操作 string 类型的栈 
Stack<string> 		stringStack;    
stringStack.push("hello"); 
cout << stringStack.top() << std::endl; 
stringStack.pop(); 
stringStack.pop();
```

## 预处理

### `#define` 来定义一个带有参数的宏

```c
#define MIN(a,b) (((a)<(b)) ? a : b)
```

### 条件编译

```c
#ifdef DEBUG
   cerr <<"Variable x = " << x << endl;
#endif
```

### `#` 和 `##` 运算符

```c
#define MKSTR( x ) #x
```

```c
int main ()
{
    cout << MKSTR(HELLO My Name is hahaa ~~~~) << endl;
    return 0;
}
```

输出

```
HELLO My Name is hahaa ~~~~
```

就是拼接成一个完整的字符串。

```c
#define concat(a, b) a ## b
```

```c
int main ()
{
    cout << concat(10, 10) << endl;
    return 0;
}
```

输出

```
1010
```

同样也是会被连接起来。

### C++ 中的预定义宏

```c
__LINE__	这会在程序编译时包含当前行号。
__FILE__	这会在程序编译时包含当前文件名。
__DATE__	这会包含一个形式为 month/day/year 的字符串，它表示把源文件转换为目标代码的日期。
__TIME__	这会包含一个形式为 hour:minute:second 的字符串，它表示程序被编译的时间。
```

## C++ 信号处理

信号是由操作系统传给进程的中断，会提早终止一个程序。

有些信号不能被程序捕获，但是下表所列信号可以在程序中捕获，并可以基于信号采取适当的动作。这些信号是定义在 C++ 头文件 <csignal> 中。

```c
SIGABRT	程序的异常终止，如调用 abort。
SIGFPE	错误的算术运算，比如除以零或导致溢出的操作。
SIGILL	检测非法指令。
SIGINT	接收到交互注意信号。
SIGSEGV	非法访问内存。
SIGTERM	发送到程序的终止请求。
```

### `signal()` 函数、用来捕获突发事件

```c
void (*signal (int sig, void (*func)(int)))(int); 
```

这个函数接收两个参数：第一个参数是一个整数，代表了信号的编号；第二个参数是一个指向信号处理函数的指针。

编写一个简单的 C++ 程序，使用 signal() 函数捕获 SIGINT 信号。

```c
#include <iostream>
#include <csignal>

using namespace std;

void signalHandler( int signum )
{
    cout << "Interrupt signal (" << signum << ") received.\n";

    // 清理并关闭
    // 终止程序  

   exit(signum);  

}

int main ()
{
    // 注册信号 SIGINT 和信号处理程序
    signal(SIGINT, signalHandler);  

    while(1){
       cout << "Going to sleep...." << endl;
       sleep(1);
    }

    return 0;
}
```

当上面的代码被编译和执行时，它会产生下列结果：

```
Going to sleep....
Going to sleep....
Going to sleep....
```

按 Ctrl+C 来中断程序，您会看到程序捕获信号，程序打印如下内容并退出：

```
Going to sleep....
Going to sleep....
Going to sleep....
Interrupt signal (2) received.
```

### `raise()` 函数、生成信号

```c
int raise (signal sig);
```

sig 是要发送的信号的编号，这些信号包括：SIGINT、SIGABRT、SIGFPE、SIGILL、SIGSEGV、SIGTERM、SIGHUP。


以下是我们使用 raise() 函数内部生成信号的实例：

```c
#include <iostream>
#include <csignal>

using namespace std;

void signalHandler( int signum )
{
    cout << "Interrupt signal (" << signum << ") received.\n";

    // 清理并关闭
    // 终止程序 

   exit(signum);  

}

int main ()
{
    int i = 0;
    
    //1. 注册信号 SIGINT 和信号处理程序
    signal(SIGINT, signalHandler);  


    while(++i){
       cout << "Going to sleep...." << endl;
       
       
       if( i == 3 ){
          raise( SIGINT);//2. 发送信号
       }
       
       sleep(1);
    }

    return 0;
}
```

## C++ 多线程

就是使用`pthread.h`中的POSIX线程。


## C++ Web 编程

### 什么是 CGI？

```
1. 公共网关接口（CGI），是一套标准，定义了信息是如何在 Web 服务器和客户端脚本之间进行交换的。

2. CGI 规范目前是由 NCSA 维护的，NCSA 定义 CGI 如下：

3. 公共网关接口（CGI），是一种用于外部网关程序与信息服务器（如 HTTP 服务器）对接的接口标准。

4. 目前的版本是 CGI/1.1，CGI/1.2 版本正在推进中。
```

### HTTP 头信息

```c
Content-type:	MIME 字符串，定义返回的文件格式。例如 Content-type:text/html。
Expires: Date	信息变成无效的日期。浏览器使用它来判断一个页面何时需要刷新。一个有效的日期字符串的格式应为 01 Jan 1998 12:00:00 GMT。
Location: URL	这个 URL 是指应该返回的 URL，而不是请求的 URL。你可以使用它来重定向一个请求到任意的文件。
Last-modified: Date	资源的最后修改日期。
Content-length: N	要返回的数据的长度，以字节为单位。浏览器使用这个值来表示一个文件的预计下载时间。
Set-Cookie: String	通过 string 设置 cookie。
```