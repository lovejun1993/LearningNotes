## Category不会覆盖原始方法实现，以及多个Category重写相同的方法实现的调用顺序

主要涉及的问题

```
1. Category是否会覆盖原始Class中的Method？
2. 多个Category添加相同的Method，调用顺序是什么？
```

Person原始类

```objc
@interface Person : NSObject
- (void)log1;
- (void)log2;
- (void)log3;
@end
@implementation Person
- (void)log1 {
    NSLog(@"Person");
}
- (void)log2 {
    NSLog(@"Person");
}
- (void)log3 {
    NSLog(@"Person");
}
@end
```

ViewController测试类

```objc
#import "Person.h"
#import <objc/runtime.h>

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    Class currentClass = [Person class];
    Person *per = [[Person alloc] init];
    [per log1];
    
    unsigned int methodCount;
    Method *methodList = class_copyMethodList(currentClass, &methodCount);
    NSLog(@"methodCount = %d", methodCount);
    
    for (NSInteger i = 0; i < methodCount; i++) {
        Method method = methodList[i];
        struct  objc_method_description *ms = method_getDescription(method);
        IMP imp = method_getImplementation(method);
        SEL sel1 = method_getName(method);
        SEL sel2 = ms->name;
        const char *types = ms->types;
        NSLog(@"imp = %p, sel1 = %@, sel2 = %@, types = %s", imp, NSStringFromSelector(sel1), NSStringFromSelector(sel2), types);
    }
}

@end
```

输出如下

```
2016-12-14 22:56:10.688 Demos[2249:20243] Person
2016-12-14 22:56:10.688 Demos[2249:20243] methodCount = 3
2016-12-14 22:56:10.688 Demos[2249:20243] imp = 0x10ee85d30, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 22:56:10.689 Demos[2249:20243] imp = 0x10ee85d60, sel1 = log2, sel2 = log2, types = v16@0:8
2016-12-14 22:56:10.689 Demos[2249:20243] imp = 0x10ee85d90, sel1 = log3, sel2 = log3, types = v16@0:8
```

无论点击多少次重新执行，都是同样的排列顺序输出。

```
log1 >>> log2 >>> log3
```

应该是按照Method在类中定义从上到下的顺序添加到Class的`method_list`中。

###创建Person分类1，覆盖掉Person类对象的log1实现，并添加log4、log5新方法实现

```objc
@interface Person (Additions)
- (void)log1;
- (void)log4;
- (void)log5;
@end
@implementation Person (Additions)
- (void)log1 {//这个函数Xcode会报警告 >>> Category is implementing a method which will also be implemented by its primary class.
    NSLog(@"Additions 1");
}
- (void)log4 {
    NSLog(@"Additions 1");
}
- (void)log5 {
    NSLog(@"Additions 1");
}
@end
```

ViewController测试类

```objc
#import "Person.h"
#import "Person+Additions.h"
#import <objc/runtime.h>

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    Class currentClass = [Person class];
    Person *per = [[Person alloc] init];
    [per log1];
    
    unsigned int methodCount;
    Method *methodList = class_copyMethodList(currentClass, &methodCount);
    NSLog(@"methodCount = %d", methodCount);
    
    for (NSInteger i = 0; i < methodCount; i++) {
        Method method = methodList[i];
        struct  objc_method_description *ms = method_getDescription(method);
        IMP imp = method_getImplementation(method);
        SEL sel1 = method_getName(method);
        SEL sel2 = ms->name;
        const char *types = ms->types;
        NSLog(@"imp = %p, sel1 = %@, sel2 = %@, types = %s", imp, NSStringFromSelector(sel1), NSStringFromSelector(sel2), types);
    }
}

@end
```

编译路径中的顺序:

```
1. Person.m
2. Person+Additions.m
```

输出结果

```
2016-12-14 23:09:50.655 Demos[2989:32823] Additions 1
2016-12-14 23:09:50.655 Demos[2989:32823] methodCount = 6
2016-12-14 23:09:50.655 Demos[2989:32823] imp = 0x10dd24d00, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:09:50.655 Demos[2989:32823] imp = 0x10dd24c70, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:09:50.655 Demos[2989:32823] imp = 0x10dd24ca0, sel1 = log2, sel2 = log2, types = v16@0:8
2016-12-14 23:09:50.656 Demos[2989:32823] imp = 0x10dd24cd0, sel1 = log3, sel2 = log3, types = v16@0:8
2016-12-14 23:09:50.656 Demos[2989:32823] imp = 0x10dd24d30, sel1 = log4, sel2 = log4, types = v16@0:8
2016-12-14 23:09:50.656 Demos[2989:32823] imp = 0x10dd24d60, sel1 = log5, sel2 = log5, types = v16@0:8
```

编译路径中的顺序:

```
1. Person+Additions.m
2. Person.m
```

输出结果

```
2016-12-14 23:11:52.297 Demos[3091:35206] Additions 1
2016-12-14 23:11:52.297 Demos[3091:35206] methodCount = 6
2016-12-14 23:11:52.297 Demos[3091:35206] imp = 0x101159c70, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:11:52.298 Demos[3091:35206] imp = 0x101159d00, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:11:52.298 Demos[3091:35206] imp = 0x101159ca0, sel1 = log4, sel2 = log4, types = v16@0:8
2016-12-14 23:11:52.298 Demos[3091:35206] imp = 0x101159cd0, sel1 = log5, sel2 = log5, types = v16@0:8
2016-12-14 23:11:52.298 Demos[3091:35206] imp = 0x101159d30, sel1 = log2, sel2 = log2, types = v16@0:8
2016-12-14 23:11:52.298 Demos[3091:35206] imp = 0x101159d60, sel1 = log3, sel2 = log3, types = v16@0:8
```

可以根据`Person.m`与`Person+Additions.m`在编译路径中前后的顺序不同的输出得到如下:

- (1) 都只执行了`Person+Additions分类`的log1方法实现

- (2) 并没有继续执行`Person类`的log1方法实现，因为已经在`method_list`第一个找到了log1的Method，就不再往后面查找Method了

- (3) 分类中的log1方法实现，并`不会替换`掉Person类Class中的log1原始方法实现。并且不管运行多少次，都是按照固定的顺序排列Method

```
1. log1 >>> `Person+Additions分类`的log1方法实现
2. log1 >>> `Person类`的log1方法实现
```

- (4) 对于分类中并不是覆盖重写的Method，会根据编译路径顺序而不同排列Method

```
1. log2
2. log3
3. log4
4. log5
```

```
1. log4
2. log5
3. log2
4. log3
```

- (5) 会将分类中的log1方法实现同样`添加`到Person类Class中。并且会将分类中的log1方法实现Method，采用`头插法`插入到Person类Class的`method_list`的`最前面`。

- (6) 所以只要在分类中`覆盖or重写`原始类的方法实现时，不管是分类与原始类的编译路径顺序如何
	- 都`只会`执行`分类`中的方法实现
	- 而`不会`再执行`原始类`中的方法实现

###创建Person分类2，覆盖掉Person类对象的log1实现，并添加log6、log7新方法实现

```objc
@interface Person (Addtions2)
- (void)log1;
- (void)log6;
- (void)log7;
@end
@implementation Person (Addtions2)
- (void)log1 {
    NSLog(@"Additions 2");
}
- (void)log6 {
    NSLog(@"Additions 2");
}
- (void)log7 {
    NSLog(@"Additions 3");
}
@end
```

- 编译路径中的顺序1:

```
1. Person+Addtions2.m
2. Person+Additions.m
3. Person.m
```

输出结果

```
2016-12-14 23:20:45.118 Demos[3581:42164] Additions 1
2016-12-14 23:20:45.118 Demos[3581:42164] methodCount = 9
2016-12-14 23:20:45.118 Demos[3581:42164] imp = 0x10e6e1c30, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:20:45.118 Demos[3581:42164] imp = 0x10e6e1930, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:20:45.118 Demos[3581:42164] imp = 0x10e6e1cc0, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:20:45.119 Demos[3581:42164] imp = 0x10e6e1960, sel1 = log6, sel2 = log6, types = v16@0:8
2016-12-14 23:20:45.119 Demos[3581:42164] imp = 0x10e6e1990, sel1 = log7, sel2 = log7, types = v16@0:8
2016-12-14 23:20:45.119 Demos[3581:42164] imp = 0x10e6e1c60, sel1 = log4, sel2 = log4, types = v16@0:8
2016-12-14 23:20:45.119 Demos[3581:42164] imp = 0x10e6e1c90, sel1 = log5, sel2 = log5, types = v16@0:8
2016-12-14 23:20:45.119 Demos[3581:42164] imp = 0x10e6e1cf0, sel1 = log2, sel2 = log2, types = v16@0:8
2016-12-14 23:20:45.119 Demos[3581:42164] imp = 0x10e6e1d20, sel1 = log3, sel2 = log3, types = v16@0:8
```

可以看到执行的是`Person+Additions.m`中的log1方法实现。

- 编译路径中的顺序2:

```
1. Person+Additions.m
2. Person+Addtions2.m
3. Person.m
```

输出结果

```
2016-12-14 23:22:34.416 Demos[3674:44009] Additions 2
2016-12-14 23:22:34.417 Demos[3674:44009] methodCount = 9
2016-12-14 23:22:34.417 Demos[3674:44009] imp = 0x106bda9c0, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:22:34.417 Demos[3674:44009] imp = 0x106bda930, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:22:34.417 Demos[3674:44009] imp = 0x106bdaa50, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:22:34.417 Demos[3674:44009] imp = 0x106bda960, sel1 = log4, sel2 = log4, types = v16@0:8
2016-12-14 23:22:34.418 Demos[3674:44009] imp = 0x106bda990, sel1 = log5, sel2 = log5, types = v16@0:8
2016-12-14 23:22:34.418 Demos[3674:44009] imp = 0x106bda9f0, sel1 = log6, sel2 = log6, types = v16@0:8
2016-12-14 23:22:34.418 Demos[3674:44009] imp = 0x106bdaa20, sel1 = log7, sel2 = log7, types = v16@0:8
2016-12-14 23:22:34.418 Demos[3674:44009] imp = 0x106bdaa80, sel1 = log2, sel2 = log2, types = v16@0:8
2016-12-14 23:22:34.418 Demos[3674:44009] imp = 0x106bdaab0, sel1 = log3, sel2 = log3, types = v16@0:8
```

可以看到执行的是`Person+Addtions2.m`中的log1方法实现。


- 编译路径中的顺序3:

```
1. Person.m
2. Person+Additions.m
3. Person+Addtions2.m
```

输出结果

```
2016-12-14 23:29:48.763 Demos[4059:50331] Additions 2
2016-12-14 23:29:48.764 Demos[4059:50331] methodCount = 9
2016-12-14 23:29:48.764 Demos[4059:50331] imp = 0x1019a5a50, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:29:48.765 Demos[4059:50331] imp = 0x1019a59c0, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:29:48.766 Demos[4059:50331] imp = 0x1019a5930, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:29:48.766 Demos[4059:50331] imp = 0x1019a5960, sel1 = log2, sel2 = log2, types = v16@0:8
2016-12-14 23:29:48.767 Demos[4059:50331] imp = 0x1019a5990, sel1 = log3, sel2 = log3, types = v16@0:8
2016-12-14 23:29:48.767 Demos[4059:50331] imp = 0x1019a59f0, sel1 = log4, sel2 = log4, types = v16@0:8
2016-12-14 23:29:48.768 Demos[4059:50331] imp = 0x1019a5a20, sel1 = log5, sel2 = log5, types = v16@0:8
2016-12-14 23:29:48.768 Demos[4059:50331] imp = 0x1019a5a80, sel1 = log6, sel2 = log6, types = v16@0:8
2016-12-14 23:29:48.768 Demos[4059:50331] imp = 0x1019a5ab0, sel1 = log7, sel2 = log7, types = v16@0:8
```

可以看到执行的是`Person+Addtions2.m`中的log1方法实现。

- 编译路径中的顺序4:

```
1. Person.m
2. Person+Addtions2.m
3. Person+Additions.m
```

输出结果

```
2016-12-14 23:30:58.404 Demos[4140:52164] Additions 1
2016-12-14 23:30:58.404 Demos[4140:52164] methodCount = 9
2016-12-14 23:30:58.404 Demos[4140:52164] imp = 0x109a07a50, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:30:58.405 Demos[4140:52164] imp = 0x109a079c0, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:30:58.405 Demos[4140:52164] imp = 0x109a07930, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:30:58.405 Demos[4140:52164] imp = 0x109a07960, sel1 = log2, sel2 = log2, types = v16@0:8
2016-12-14 23:30:58.405 Demos[4140:52164] imp = 0x109a07990, sel1 = log3, sel2 = log3, types = v16@0:8
2016-12-14 23:30:58.405 Demos[4140:52164] imp = 0x109a079f0, sel1 = log6, sel2 = log6, types = v16@0:8
2016-12-14 23:30:58.405 Demos[4140:52164] imp = 0x109a07a20, sel1 = log7, sel2 = log7, types = v16@0:8
2016-12-14 23:30:58.406 Demos[4140:52164] imp = 0x109a07a80, sel1 = log4, sel2 = log4, types = v16@0:8
2016-12-14 23:30:58.406 Demos[4140:52164] imp = 0x109a07ab0, sel1 = log5, sel2 = log5, types = v16@0:8
```

可以看到执行的是`Person+Addtions.m`中的log1方法实现。

- 编译路径中的顺序5:

```
1. Person+Addtions2.m
2. Person.m
3. Person+Additions.m
```

输出结果

```
2016-12-14 23:31:30.437 Demos[4175:53206] Additions 1
2016-12-14 23:31:30.438 Demos[4175:53206] methodCount = 9
2016-12-14 23:31:30.438 Demos[4175:53206] imp = 0x108d81a50, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:31:30.438 Demos[4175:53206] imp = 0x108d81930, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:31:30.439 Demos[4175:53206] imp = 0x108d819c0, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:31:30.439 Demos[4175:53206] imp = 0x108d81960, sel1 = log6, sel2 = log6, types = v16@0:8
2016-12-14 23:31:30.439 Demos[4175:53206] imp = 0x108d81990, sel1 = log7, sel2 = log7, types = v16@0:8
2016-12-14 23:31:30.439 Demos[4175:53206] imp = 0x108d819f0, sel1 = log2, sel2 = log2, types = v16@0:8
2016-12-14 23:31:30.439 Demos[4175:53206] imp = 0x108d81a20, sel1 = log3, sel2 = log3, types = v16@0:8
2016-12-14 23:31:30.439 Demos[4175:53206] imp = 0x108d81a80, sel1 = log4, sel2 = log4, types = v16@0:8
2016-12-14 23:31:30.439 Demos[4175:53206] imp = 0x108d81ab0, sel1 = log5, sel2 = log5, types = v16@0:8
```

可以看到执行的是`Person+Addtions.m`中的log1方法实现。

- 编译路径中的顺序6:

```
1. Person+Additions.m
2. Person.m
3. Person+Addtions2.m
```

输出结果

```
2016-12-14 23:33:22.445 Demos[4277:55509] Additions 2
2016-12-14 23:33:22.445 Demos[4277:55509] methodCount = 9
2016-12-14 23:33:22.445 Demos[4277:55509] imp = 0x1062a7a50, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:33:22.446 Demos[4277:55509] imp = 0x1062a7930, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:33:22.446 Demos[4277:55509] imp = 0x1062a79c0, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:33:22.446 Demos[4277:55509] imp = 0x1062a7960, sel1 = log4, sel2 = log4, types = v16@0:8
2016-12-14 23:33:22.446 Demos[4277:55509] imp = 0x1062a7990, sel1 = log5, sel2 = log5, types = v16@0:8
2016-12-14 23:33:22.446 Demos[4277:55509] imp = 0x1062a79f0, sel1 = log2, sel2 = log2, types = v16@0:8
2016-12-14 23:33:22.446 Demos[4277:55509] imp = 0x1062a7a20, sel1 = log3, sel2 = log3, types = v16@0:8
2016-12-14 23:33:22.446 Demos[4277:55509] imp = 0x1062a7a80, sel1 = log6, sel2 = log6, types = v16@0:8
2016-12-14 23:33:22.446 Demos[4277:55509] imp = 0x1062a7ab0, sel1 = log7, sel2 = log7, types = v16@0:8
```

可以看到执行的是`Person+Addtions2.m`中的log1方法实现。

###再看下不同Category中添加同一个SEL的方法实现Method

```objc
@interface Person (Additions)
- (void)log1;
- (void)log4;
- (void)log5;
- (void)logC;
@end
@implementation Person (Additions)
- (void)log1 {
    NSLog(@"Additions 1");
}
- (void)log4 {
    NSLog(@"Additions 1");
}
- (void)log5 {
    NSLog(@"Additions 1");
}
- (void)logC {
    NSLog(@"Additions 1");
}
@end
```

```objc
@interface Person (Addtions2)
- (void)log1;
- (void)log6;
- (void)log7;
- (void)logC;
@end
@implementation Person (Addtions2)
- (void)log1 {
    NSLog(@"Additions 2");
}
- (void)log6 {
    NSLog(@"Additions 2");
}
- (void)log7 {
    NSLog(@"Additions 2");
}
- (void)logC {
    NSLog(@"Additions 2");
}
@end
```

- 编译路径1

```
1. Person+Additions.m
2. Person+Addtions2.m
3. Person.m
```

输出结果

```
2016-12-14 23:43:42.979 Demos[5235:66425] Additions 2
2016-12-14 23:43:42.979 Demos[5235:66425] methodCount = 11
2016-12-14 23:43:42.979 Demos[5235:66425] imp = 0x10056ba20, sel1 = logC, sel2 = logC, types = v16@0:8
2016-12-14 23:43:42.979 Demos[5235:66425] imp = 0x10056b960, sel1 = logC, sel2 = logC, types = v16@0:8
2016-12-14 23:43:42.979 Demos[5235:66425] imp = 0x10056b990, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:43:42.980 Demos[5235:66425] imp = 0x10056b8d0, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:43:42.980 Demos[5235:66425] imp = 0x10056ba50, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:43:42.980 Demos[5235:66425] imp = 0x10056b900, sel1 = log4, sel2 = log4, types = v16@0:8
2016-12-14 23:43:42.980 Demos[5235:66425] imp = 0x10056b930, sel1 = log5, sel2 = log5, types = v16@0:8
2016-12-14 23:43:42.980 Demos[5235:66425] imp = 0x10056b9c0, sel1 = log6, sel2 = log6, types = v16@0:8
2016-12-14 23:43:42.980 Demos[5235:66425] imp = 0x10056b9f0, sel1 = log7, sel2 = log7, types = v16@0:8
2016-12-14 23:43:42.980 Demos[5235:66425] imp = 0x10056ba80, sel1 = log2, sel2 = log2, types = v16@0:8
2016-12-14 23:43:42.980 Demos[5235:66425] imp = 0x10056bab0, sel1 = log3, sel2 = log3, types = v16@0:8
```

- 编译路径2

```
1. Person+Addtions2.m
2. Person+Additions.m
3. Person.m
```

输出结果

```
2016-12-14 23:44:11.791 Demos[5263:67215] Additions 1
2016-12-14 23:44:11.792 Demos[5263:67215] methodCount = 11
2016-12-14 23:44:11.793 Demos[5263:67215] imp = 0x1009dea20, sel1 = logC, sel2 = logC, types = v16@0:8
2016-12-14 23:44:11.793 Demos[5263:67215] imp = 0x1009de960, sel1 = logC, sel2 = logC, types = v16@0:8
2016-12-14 23:44:11.793 Demos[5263:67215] imp = 0x1009de990, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:44:11.793 Demos[5263:67215] imp = 0x1009de8d0, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:44:11.794 Demos[5263:67215] imp = 0x1009dea50, sel1 = log1, sel2 = log1, types = v16@0:8
2016-12-14 23:44:11.794 Demos[5263:67215] imp = 0x1009de900, sel1 = log6, sel2 = log6, types = v16@0:8
2016-12-14 23:44:11.794 Demos[5263:67215] imp = 0x1009de930, sel1 = log7, sel2 = log7, types = v16@0:8
2016-12-14 23:44:11.794 Demos[5263:67215] imp = 0x1009de9c0, sel1 = log4, sel2 = log4, types = v16@0:8
2016-12-14 23:44:11.794 Demos[5263:67215] imp = 0x1009de9f0, sel1 = log5, sel2 = log5, types = v16@0:8
2016-12-14 23:44:11.794 Demos[5263:67215] imp = 0x1009dea80, sel1 = log2, sel2 = log2, types = v16@0:8
2016-12-14 23:44:11.797 Demos[5263:67215] imp = 0x1009deab0, sel1 = log3, sel2 = log3, types = v16@0:8
```

同样是Category在编译路径出现的越后面，其Method就会排在`method_list`的第一个。

### 小结:

- (1) Category中重写原始类中已经存在的方法实现时
	- Category中重写Method肯定会排在原始类的Method的`前面`
	- 每当从编译路径中读取到Category重写Method，就会将这个重写的Method采用`头插法`插入到原始类的`method_list`的第一个位置
	- 也就是说，越在后面编译的Category中重写的Method，却会出现在`method_list`的第一个位置
	
- (2) 多个Category中，都重写了相同的方法实现时
	- 会按照Category在编译路径中的顺序，将Method依次【头插】到原始类的`method_list`