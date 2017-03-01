## 全局block、栈block、堆block。声明一个block时，其实就是创建了objc对象

```objc
- (void)test1 {
    
    // 全局Block，不使用任何的外部变量
    void (^block1)() = ^(){};
    
    // 栈block，但是被默认拷贝到了堆区，所以在ARC下，相当于没有栈block了
    NSInteger i = 0;
    void (^block2)() = ^(){
        NSLog(@"i = %ld", i);
    };
    
    NSLog(@"block1: %@", block1);
    NSLog(@"block2: %@", block2);
}
```

输出如下

```
2017-03-01 23:23:32.269 Demos[1786:62019] block1: <__NSGlobalBlock__: 0x10309d9d0>
2017-03-01 23:23:32.269 Demos[1786:62019] block2: <__NSMallocBlock__: 0x7fcf10d0f870>
```

由于栈block默认被拷贝到堆区，所以实际上在ARC下，我们是看不到栈block了。

### 所以，根据输出，推测三种block类型如下:

- (1) `__NSGlobalBlock__`
- (2) `__NSMallocBlock__`
- (3) `__NSStackBlock__`(推测)

也可以使用如下代码进行测试`栈block`:

```objc
Class cls = NSClassFromString(@"__NSStackBlock__");
NSLog(@"block3: %@ - %p", cls, cls);
```

输出

```
2017-03-01 23:26:25.664 Demos[1826:64600] block3: __NSStackBlock__ - 0x105d5f050
```

确实是这样的。

## 如上三种是block具体的内部私有类型，同样也有类似NSArray这个类簇类

```objc
static Class GetNSBlock() {
    static Class _cls = Nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        
        //1. 随便创建一个block对象
        id block = ^() {};
        
        //2.
        _cls = [block class];
        
        //2. 一直向上查询super_class
        while (_cls && (class_getSuperclass(_cls) != [NSObject class])) {
            _cls = class_getSuperclass(_cls);
        }
    });
    return _cls;
}
```

```objc
@implementation BlockViewController

- (void)test2 {
    Class cls = GetNSBlock();
    NSLog(@"%@ - %p", cls, cls);
}

@end
```

输出如下

```
2017-03-01 23:32:55.644 Demos[1897:68289] NSBlock - 0x109d27d28
```

可以看到三种内部私有block类型的`类簇类`就叫做`NSBlock`。

## Block内不能修改基本类型变量

```objc
@implementation BlockViewController

- (void)test3 {
    int i = 5;
    
    void (^block)() = ^(){
        NSLog(@"i = %d", i);
//        i++; 这句会报错
    };
    
    block();
}

@end
```

如上注释的代码打开就会报错。

## Block捕获基本类型变量，只是`值传递`，传递的只是之前的一份`值拷贝`数据


```objc
@implementation BlockViewController

- (void)test4 {
    
    int i = 5;
    
    void (^block)() = ^(){
        NSLog(@"i = %d", i);
    };
    
    i = 6;
    
    block();
}

@end
```

输出如下

```
2017-03-01 23:36:29.024 Demos[1955:70973] i = 5
```

以前我也很奇怪，为什么不是6，而是5，因为这就是`值传递`，而并不是`传地址`。


```objc
@implementation BlockViewController

- (void)test4 {
    
    __block int i = 5;
    
    void (^block)() = ^(){
        NSLog(@"i = %d", i);
    };
    
    i = 6;
    
    block();
}

@end
```

输出如下

```
2017-03-01 23:37:38.957 Demos[1981:71973] i = 6
```

对基本类型变量前面加上`__block`修饰后，表示使用`地址传递`，而不再是`值传递`。

## Block捕获NSObject对象类型变量，默认就是`地址传递`

```objc
@interface Car : NSObject 
@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *cid;
@property (nonatomic, assign) NSInteger price;
- (instancetype)initWithName:(NSString *)name cid:(NSString *)cid;
@end
@implementation Car
@end
```

```objc
@implementation BlockViewController

- (void)test5 {
    
    //1.
    Car *car = [[Car alloc] initWithName:@"car1" cid:@"car1"];
    NSLog(@"car: %p", car);
    
    //2. block捕获objc对象
    void (^block)() = ^() {
        car.name = @"car3";
        car.cid = @"cid3";
        NSLog(@"car:%p, name:%@, pid:%@", car, car.name, car.cid);
    };
    
    //3. objc对象外面进行修改
    car.name = @"car2";
    car.cid = @"cid2";
    
    //4.
    block();
    
    //5.
    NSLog(@"car:%p, name:%@, pid:%@", car, car.name, car.cid);
}

@end
```

输出如下

```
2017-03-01 23:40:55.192 Demos[2015:74029] car: 0x7ffb584703f0
2017-03-01 23:40:55.193 Demos[2015:74029] car:0x7ffb584703f0, name:car3, pid:cid3
2017-03-01 23:40:55.193 Demos[2015:74029] car:0x7ffb584703f0, name:car3, pid:cid3
2017-03-01 23:40:55.193 Demos[2015:74029] <<Car: 0x7ffb584703f0> - 0x7ffb584703f0> name:car3 cid:cid3 dealloc
```

可以看到，一直都是操作的`0x7ffb584703f0`这个内存地址，所以对于objc对象的指针变量，默认就是`地址传递`。


## Block既然就是NSObject子类对象，那么就能够持有其他NSObject类对象

在Car对象`即将`废弃的时候log下.

```objc
@implementation Car
- (void)dealloc {
    NSLog(@"%@ - %p", self, self);
}
@end
```

### 第一种情况，Block不持有局部Car对象:

```objc
- (void)test6 {
    
    //1.
    Car *car = [[Car alloc] initWithName:@"car1" cid:@"car1"];
    NSLog(@"car: %p", car);
    
    //2.
    void (^block)() = ^() {
//        NSLog(@"car:%p, name:%@, pid:%@", car, car.name, car.cid);
    };
    
    //3.
    NSLog(@"pre car = nil;");
    car = nil;
    NSLog(@"after car = nil;");
    
    //4.
    NSLog(@"pre block = nil;");
    block = nil;
    NSLog(@"after block = nil;");
}
```

输出如下

```
2017-03-01 23:50:22.457 Demos[2187:81440] car: 0x7fb318532450
2017-03-01 23:50:22.457 Demos[2187:81440] pre car = nil;
2017-03-01 23:50:22.458 Demos[2187:81440] <Car: 0x7fb318532450> - 0x7fb318532450
2017-03-01 23:50:22.458 Demos[2187:81440] after car = nil;
2017-03-01 23:50:22.458 Demos[2187:81440] pre block = nil;
2017-03-01 23:50:22.458 Demos[2187:81440] after block = nil;
```

执行完`car = nil;`car对象就被系统标记为废弃的内存了，**注意:只是被标记为被废弃的内存，但并没有立刻被系统废弃，是有一定的延迟时间的**。


### 第二种情况，Block持有局部Car对象:

```objc
- (void)test6 {
    
    //1.
    Car *car = [[Car alloc] initWithName:@"car1" cid:@"car1"];
    NSLog(@"car: %p", car);
    
    //2.
    void (^block)() = ^() {
        NSLog(@"car:%p, name:%@, pid:%@", car, car.name, car.cid);
    };
    
    //3.
    NSLog(@"pre car = nil;");
    car = nil;
    NSLog(@"after car = nil;");
    
    //4.
    NSLog(@"pre block = nil;");
    block = nil;
    NSLog(@"after block = nil;");
}
```

输出如下

```
2017-03-01 23:53:13.488 Demos[2218:83014] car: 0x7f8439ccf910
2017-03-01 23:53:13.488 Demos[2218:83014] pre car = nil;
2017-03-01 23:53:13.488 Demos[2218:83014] after car = nil;
2017-03-01 23:53:13.488 Demos[2218:83014] pre block = nil;
2017-03-01 23:53:13.489 Demos[2218:83014] after block = nil;
2017-03-01 23:53:13.489 Demos[2218:83014] <Car: 0x7f8439ccf910> - 0x7f8439ccf910
```

Car对象被系统标记为废弃的操作，延迟到了执行完`block = nil;`，说明Car对象被block持有了，必须等block标记废弃，才会随之废弃。


## Block与NSObject类对象之间的循环引用的最基本的情况

Block与其他的NSObject类对象，之间互相持有

```objc
@interface Car : NSObject 
@property (nonatomic, strong) id block;
@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *cid;
@property (nonatomic, assign) NSInteger price;
@end
@implementation Car
- (void)dealloc {
    NSLog(@"%@ - %p", self, self);
}
@end
```

```objc
@implementation BlockViewController 

- (void)test8 {
    
    //1. 局部对象
    Car *car = [[Car alloc] initWithName:@"car1" cid:@"car1"];
    
    //2. 局部对象
    void (^block)() = ^() {
        [car name];
    };
    
    //3. `循环强`引用产生
    car.block = block;
}

@end
```

运行后，没有任何的dealloc打印，说明Car对象没有被废弃。**block捕获外部的Car对象时，会执行`[car retain]`的操作**，但最终并没有执行`[car release]`，所以没有废弃。

这种情况下，必须是有`一方主动的解除对对方的持有关系`，才能解除循环引用，当然是`__weak`就不说了。

```objc
- (void)test8 {
    
    //1. 局部对象
    Car *car = [[Car alloc] initWithName:@"car1" cid:@"car1"];
    
    //2. 局部对象
    void (^block)() = ^() {
        [car name];
    };
    
    //3. `循环强`引用产生
    car.block = block;
    
    //4. 其中一方主动解除对另外一方的强引用
    car.block = nil;
}
```

输出结果

```
2017-03-02 00:12:48.432 Demos[2603:98372] <Car: 0x7fa611d43a80> - 0x7fa611d43a80
```

但是如下，只是对block释放是不行的

```objc
- (void)test8 {
    
    //1. 局部对象
    Car *car = [[Car alloc] initWithName:@"car1" cid:@"car1"];
    
    //2. 局部对象
    void (^block)() = ^() {
        [car name];
    };
    
    //3. `循环强`引用产生
    car.block = block;
    
    //4. 其中一方主动解除对另外一方的强引用
    block = nil;
}
```

运行后，没有任何输出，说明Car对象还是没有废弃。因为ARC下，block会默认拷贝到`堆区`，只是对栈内的block释放是没用的。

那么，还是得需要从强引用block的objc对象入手，进行主动的废弃持有的block。

注意，对于系统GCD、系统函数接收的block，最终会自动进行废弃，不会一直强引用block。

