
## Objectivec-C的是基于`消息结构`，并非`函数调用`

### c/c++（java都是类似） 普遍使用的`函数调用`

```c
void func1() {
    printf("func1");
}

void func2() {
    printf("func2");
}

void func3(int type) {
    if (type == 1) {
        func1();
    } else if (type == 2){
        func2();
    }
}
```


可以看到，在func3()中直接通过函数调用的方式调用func()1和func2()，那么编译期间会做这样的事情:

- (1) 将func1()、func2()、func3()和三个函数分别保存在不同的内存块中，分配该内存块的首地址

- (2) 然后在func3()所在内存，通过硬编码的形式将func1()、func2()所在内存的首地址写入，以便后续对func1()、func2()的函数调用

实际上就是在编译期确定了func3()与func1()、func2()之间的对应关系，后续是无法再修改了。虽然可能会对func1()或func2()其中一个函数不会使用，但是还是将func3()与func1()、func2()进行了绑定。

对上述代码使用`函数指针`进行修改之后，模拟动态绑定函数之间的关系:

```c
void (*funcP)(void);

void func1() {
    printf("func1");
}

void func2() {
    printf("func2");
}

void func3(int type) {
    if (type == 1) {
        funcP = func1;
    } else if (type == 2){
        funcP = func2;
    }
    
    funcP();
}
```

使用了函数指针来指向对应需要使用的函数实现之后，那么就不会向之前在`编译期间`对func3()强行绑定func1()与func2()的调用关系了。

只在程序`运行期间`时，函数指针动态的指向func1()或func2()所在内存块的地址，才能够调用对应的函数实现了。

简单描述下`[target sel]`查询最终c函数实现的过程:

- (1) 确定target是OC对象还是OC类
- (2) 然后去对应的`objc_class->method_list`根据`sel`查询得到一个`objc_method`
- (3) 然后根据找到的`objc_method->IMP`即可找到最终的c函数的地址了，继而完成方法调用

所以在Objective-C中，所谓的消息其实就是基于c语言中的`函数指针`来完成的。因为是在运行时确定函数指针最终指向的函数实现`IMP`，那么在运行时也就可以替换掉函数指针的指向，这也就是经常听到的`Method Swizzle`。

### Objectivc-C 中所谓的 `消息发送`

首先来一个Objective-C的类

```objc
@interface Person : NSObject
- (void)logInfo:(NSString *)info;
@end
@implementation Person
- (void)logInfo:(NSString *)info {
    NSLog(@"info = %@", info);
}
@end
```

然后使用上面的类生成对象并调用对象的方法实现

```objc
Person *obj = [Person new];
[obj logInfo:@"方法实现执行所需要的参数"];
```

上面的代码，我刚开始搞iOS的时候，机会也是认为就是OC的方法调用格式，就相当于说`[target 方法]`这样就是方法调用。

c/c++/java这些语言中，都是使用`func()`这样来表示方法调用。第一次接触Objective-C时，感觉好奇怪`[target sel]`这样也叫做方法调用。其实这是错误的理解，但是可以当做最简单的理解方式。

`[target sel]`这个并不是方法调用，而是所谓的消息发送。至于为什么这样叫，其原因是Objective-C强大的`运行时系统 runtime system`中封装了大量的数据结构，并用来完成方法调用以及类型。

### Objectivc-C 中用来描述 `消息` 的结构类型

本来我想看一下最终用来发送消息的`objc_msgSend()`的源码，但是在苹果runtime开源中没有找到，后来查了下说`objc_msgSend()`直接用`汇编`实现的.....so 即使找到开源代码我也看不懂。

但是可以参考`objc_msgSend()`的函数入参大致了解一个消息的的构成:

```c
objc_msgSend(id self, SEL op, ...)
```

- (1) id self
- (2) SEL op
- (3) arguments 多个参数

然后根据一些消息发送的一些术语，可以大致知道，一个`消息`主要包含: 

- (1) 消息发送给哪一个
- (2) 消息执行的函数名字是什么
- (3) 最终c函数执行所需要的全部参数

### `objc_msgSend(target, sel, arguments...)`使用demo:

```objc
@interface Person : NSObject
- (void)logInfo:(NSString *)info;
@end
@implementation Person
- (void)logInfo:(NSString *)info {
    NSLog(@"info = %@", info);
}
@end
```

使用`[target sel:args]`形式发送消息:

```objc
[[[Person alloc] init] logInfo:@"hello world!"];
```

使用`objc_msgSend()`形式发送消息:

```objc
//1. 
id person = ((id (*)(id, SEL)) (void *) objc_msgSend)([Person class], @selector(alloc));

//2.
person = ((id (*)(id, SEL)) (void *) objc_msgSend)(person, @selector(init));

//3.
((void (*)(id, SEL, NSString*)) (void *) objc_msgSend)(person, @selector(logInfo:), @"hello world!");
```

也就是说我们使用`[target sel:参数]` 最终会被编译器编译成`objc_msgSend()`函数的调用形式的c代码，其实我们写的所有的OC代码，最终都会被编译成c代码。

### Foundation提供了一个`objc_msgSend()`的Objective-C实现版本:

```objc
//1.
Person *person = [Person new];

//2.
NSMethodSignature *signature = [Person instanceMethodSignatureForSelector:@selector(logInfo:)];

//3.
NSInvocation *invoke = [NSInvocation invocationWithMethodSignature:signature];

//4.
NSString *arg = @"hello world!";

//5.
[invoke setArgument:&arg atIndex:2];

//6.
[invoke setSelector:@selector(logInfo:)];

//7.
[invoke setTarget:person];

//8.
[invoke invoke];
```

觉得但是最终发送消息都是使用`objc_msgSend()`来完成的。并没有从runtime库中头文件找到关于`objc_msgSend()`函数最终生成的消息的c数据结构定义。

### 可以参考`NSInvocation`的这个Objective-C类的结构简单知道消息的组成结构:

```objc
@interface NSInvocation : NSObject
+ (NSInvocation *)invocationWithMethodSignature:(NSMethodSignature *)sig;

@property (readonly, retain) NSMethodSignature *methodSignature;

- (void)retainArguments;
@property (readonly) BOOL argumentsRetained;

@property (nullable, assign) id target;
@property SEL selector;

- (void)getReturnValue:(void *)retLoc;
- (void)setReturnValue:(void *)retLoc;

- (void)getArgument:(void *)argumentLocation atIndex:(NSInteger)idx;
- (void)setArgument:(void *)argumentLocation atIndex:(NSInteger)idx;

- (void)invoke;
- (void)invokeWithTarget:(id)target;

@end
```

大致可以得到一个最终被编译器编译生成的消息大概组成:

- (1) methodSignature >>> `用于找到最终编译生成的c函数`
- (2) target >>> `消息发送给谁`
- (3) SEL	>>> 找到`objc_method`实例的唯一id
