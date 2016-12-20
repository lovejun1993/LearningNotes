---
layout: post
title: "DesignMode、Copy"
date: 2015-01-05 22:38:15 +0800
comments: true
categories: 
---

###刚好看书看到关于`复制模式`，觉得挺有意思就记录下来了。之前同事说OC里面传递的都是`指针`，所以外面的修改会影响方法内部的参数值。当时我觉得始终是有点不对...

***

###传递参数的三种方式

C语言中，使用称为`按值传递`的技术给函数提供参数。按值传递意味着参数是`隐式`赋值的，使得在函数内部做出的修改只会影响副本，而不会影响原始值。还有一种`指针传递`方式，在C++中还有一种`引用传递`.

***

###C语言中参数传递

- 基本数据类型传递

```objc
static void Test1(int a) {
    a = 3;
}

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    int x = 6;
    Test1(x);
//断点输出
}

@end
```

输出为

```
(int) x = 6
```

基本数据类型毋庸置疑，确实是值传递。


- 字符数组指针类型

```objc
static void Test2(char a[]) {
    a = "Hahahah";
}

static void Test3() {
    char name[] = "AAAAAAA";
    Test2(name);
//断点输出
}
```

```objc
static void Test2(char *a) {
    a = "Hahahah";
}

static void Test3() {
    char *name = "AAAAAAA";
    Test2(name);
//断点输出
}
```

说明如上两种都不是地址传递，传递的都是值（拷贝后）。但是有点别捏，传递的确实是一个指针啊...只能说明传递的指针并非原始指针，而是指向另一块被复制后的数据的新指针.

那么修改为如下之后

```objc
static void Test2(char **a) {
    *a = "Hahahah";//修改地址指向的对象
}

static void Test3() {
    char *name = "AAAAAAA";
    Test2(&name);//取地址
//断点输出
}
```

这样就是传递的原始指针了，就可以在调用方法内修改原始指针值了。那么说c语言中，对于字符串、字符数组、基本数据类型，都是传递的`拷贝`数据，并非原始数据.

- c结构体指针类型

```objc
typedef struct Man {
    int age;
}Man;

static void Test(Man *man) {
    man->age = 20;
}

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    Man *man = (Man *)malloc(sizeof(Man));
    man->age = 19;
    Test(man);
//断点输出
}

@end
```

输出

```
(int) man->age = 20
```

说明对于结构体类型指针，传递的就是原始指针.

***

###OC中的参数传递

- NSString对象

```objc
@implementation ViewController

- (void)Test1:(NSString *)str {
    str = @"2222";
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    NSString *str = @"1111";
    [self Test1:str];
//断点输出
}
@end 
```

输出

```
1111
```

可以看出NSString类对象传递的`另外一个拷贝后的NSString对象指针`，并不是原始指针，所以方法内部的修改不会影响到原来NSString对象.

- NSObject对象

```objc
@implementation ViewController

- (void)Test2:(Person *)person {
    person.name = @"2222";
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    Person *person = [Person new];
    person.name = @"111";
    [self Test2:person];
//断点输出
}

@end
```

输出

```
2222
```

传递的原始对象的指针.

- NSAarray容器对象

第一种、不能达到修改原始array

```objc
- (void)Test3:(NSArray *)array {
    array = @[@"444", @"555"];
}

- (void)viewDidLoad {
    [super viewDidLoad];
 	
 	NSArray *array = @[@"111", @"222"];
    [self Test3:array];  
//断点输出array
} 
```

输出

```
<__NSArrayI 0x7fb679f12190>(
111,
222
)
```

第二种、是可以修改原始array里面的子元素

```objc
- (void)Test4:(NSMutableArray *)array {
    [array addObject:@"333"];
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    NSMutableArray *array = [@[@"111", @"222"] mutableCopy];
    [self Test4:array];
//断点输出array
}
```

输出

```
<__NSArrayM 0x7ff3f8f2aee0>(
111,
222,
333
)
```

第三种、改变array对象指针，指向其他array（不起作用）

```objc
- (void)Test4:(NSMutableArray *)array {
//    [array addObject:@"333"];
    array = [@[@"444", @"555"] mutableCopy];
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    NSMutableArray *array = [@[@"111", @"222"] mutableCopy];
    [self Test4:array];   
//断点输出array
}
```

输出

```
<__NSArrayM 0x7fd1b3613d10>(
111,
222
)
```

那么应该可以总结`对于传递数组Array作为方法参数时`:

- 对指针本身的重新赋值操作，仅在方法块内起作用，但是不能影响到原始指针
- 但是能够通过该NSArray对象指针，修改指向的内部子对象的值

其实很简单:

> **在没有使用 & 来取地址时，一切的传递都是 值传递，那么只要是值传递，就不会影响原始指针指向的对象.**


***

###最后关于参数传递总结

- 不使用`&`来取地址传参数时，都是使用`值传递`，那么对其参数的所有修改，都不会影响到原始指针指向的对象.

- 作为`值传递`方式传入方法的参数，其自身的数据（值、地址）修改之后，不会影响传入方法之前的原始参数.

- 但是可以通过`值传递`方式传入修改`指针->内部子对象 = 新的值`来修改传入指针内部指向的其他子对象的值，且能够反映到原始对象中.