##struct实例 持有 Foundation对象，在struct实例废弃时同样让Foundation对象在子线程上异步释放废弃

Foundation 类

```objc
@interface Dog : NSObject
@property (nonatomic, copy) NSString *name;
@end
@implementation Dog
- (void)dealloc {
    NSLog(@"dealloc_dog.name = %@ on thread = %@", _name, [NSThread currentThread]);
}
@end
```

struct 使用 `void*` 万能指针类型持有 Foundation对象

```c
typedef struct DogsContext {
    void    *dogs;
}DogsContext;
```

Foundation 对象测试代码

```objc
static DogsContext *_dogsCtx = NULL;

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    _ctx = [[Context alloc] init];
    
    _dogsCtx = malloc(sizeof(DogsContext));
    _dogsCtx->dogs = NULL;
    
    NSMutableArray *dogs = [NSMutableArray new];
    for (int i = 0; i < 10; i++) {
        Dog *dog = [Dog new];
        dog.name = [NSString stringWithFormat:@"name_%d", (i + 1)];
        [dogs addObject:dog];
    }
    _dogsCtx->dogs = (__bridge_retained void*)dogs;//让c struct实例 持有 NSFoundation对象
}

- (void)testStructARC {

    //1. 先取出c struct实例持有的 NSFoundation对象 使用
    NSMutableArray *temp1 = (__bridge NSMutableArray*)_dogsCtx->dogs;
    for (Dog *dog in temp1) {
        NSLog(@"dog.name = %@", dog.name);
    }
    
    //2. 释放struct实例持有的NSMutableArray数组，继而释放掉了NSMutableArray数组持有的所有的Dogs对象
    NSMutableArray *temp2 = (__bridge_transfer NSMutableArray*)_dogsCtx->dogs;
    _dogsCtx->dogs = NULL;
    free(_dogsCtx);
    _dogsCtx = NULL;
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        [temp2 class];
    });   
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	    [self testStructARC];
}

@end
```

输出信息

```
2016-10-22 00:18:02.393 demo[5284:67959] dog.name = name_1
2016-10-22 00:18:02.393 demo[5284:67959] dog.name = name_2
2016-10-22 00:18:02.394 demo[5284:67959] dog.name = name_3
2016-10-22 00:18:02.394 demo[5284:67959] dog.name = name_4
2016-10-22 00:18:02.394 demo[5284:67959] dog.name = name_5
2016-10-22 00:18:02.394 demo[5284:67959] dog.name = name_6
2016-10-22 00:18:02.395 demo[5284:67959] dog.name = name_7
2016-10-22 00:18:02.395 demo[5284:67959] dog.name = name_8
2016-10-22 00:18:02.396 demo[5284:67959] dog.name = name_9
2016-10-22 00:18:02.396 demo[5284:67959] dog.name = name_10
2016-10-22 00:18:02.396 demo[5284:68142] dealloc_dog.name = name_1 on thread = <NSThread: 0x7ff999f86ce0>{number = 2, name = (null)}
2016-10-22 00:18:02.397 demo[5284:68142] dealloc_dog.name = name_2 on thread = <NSThread: 0x7ff999f86ce0>{number = 2, name = (null)}
2016-10-22 00:18:02.398 demo[5284:68142] dealloc_dog.name = name_3 on thread = <NSThread: 0x7ff999f86ce0>{number = 2, name = (null)}
2016-10-22 00:18:02.398 demo[5284:68142] dealloc_dog.name = name_4 on thread = <NSThread: 0x7ff999f86ce0>{number = 2, name = (null)}
2016-10-22 00:18:02.398 demo[5284:68142] dealloc_dog.name = name_5 on thread = <NSThread: 0x7ff999f86ce0>{number = 2, name = (null)}
2016-10-22 00:18:02.399 demo[5284:68142] dealloc_dog.name = name_6 on thread = <NSThread: 0x7ff999f86ce0>{number = 2, name = (null)}
2016-10-22 00:18:02.465 demo[5284:68142] dealloc_dog.name = name_7 on thread = <NSThread: 0x7ff999f86ce0>{number = 2, name = (null)}
2016-10-22 00:18:02.465 demo[5284:68142] dealloc_dog.name = name_8 on thread = <NSThread: 0x7ff999f86ce0>{number = 2, name = (null)}
2016-10-22 00:18:02.465 demo[5284:68142] dealloc_dog.name = name_9 on thread = <NSThread: 0x7ff999f86ce0>{number = 2, name = (null)}
2016-10-22 00:18:02.466 demo[5284:68142] dealloc_dog.name = name_10 on thread = <NSThread: 0x7ff999f86ce0>{number = 2, name = (null)}
```

参考自YYDispatchQueuePool中Context结构体持有`dispatch_queue_t`数组实例。

## `(__bridge_retained CoreFoundation实例)Foundation对象`

- (1) Foundation对象 >>> CoreFoundation实例
- (2) [Foundation对象 retain]

## `(__bridge_transfer Foundation对象)CoreFoundation实例`

- (1) CoreFoundation实例 >>> Foundation对象 
- (2) [转换后的Foundation对象 release]

## `__bridge` 

- (1) Foundation对象 >>> CoreFoundation实例
- (2) CoreFoundation实例 >>> Foundation对象 
- (3) 不会执行任何的`retain/release`效果，仅仅只是类型的转换