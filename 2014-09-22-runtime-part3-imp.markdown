---
layout: post
title: "Runtime Part3 IMP"
date: 2014-09-22 18:56:12 +0800
comments: true
categories: 
---

###什么是IMP？

- IMP 就是一个指向任意一个 c函数的 `函数指针`

IMP的原型

```
#if !OBJC_OLD_DISPATCH_PROTOTYPES
typedef void (*IMP)(void /* id, SEL, ... */ ); 
#else
typedef id (*IMP)(id, SEL, ...); 
#endif
```

使用IMP指向一个函数

```
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    //1. 声明一个IMP函数指针变量，并指向下面的c函数 test
    IMP imp = test;
    
    //2. 执行被指向的函数
    imp();
}

static void test() {
    
}

@end
```

就是一个函数的指针而已....

- SEL值查找到的方法实现内存地址，就是 IMP


| 方法的名字 | 方法的Id | 方法的内存地址 |
---------| ---------- | ----------
| func1 | 100001 | 0x2001 |
| func2 | 100002 | 0x2002 |
| func2: | 100003 | 0x2003 |


通过方法SEL值（方法名字的字符串）找到了类似 `0x2001`这样的 `内存地址`，那么就是IMP函数指针.


***

###运行时系统对类的所有方法的内存分配

- 所有的函数都是一个`单独的内存块`保存

- 函数所在内存块的首地址，就是作为调用这个函数的入口 即 IMP指针指向

- 所以说一个函数被调用或执行，其实就是这个函数所在内存块的首地址被系统开始读取

***

###SEL 与 IMP 的关系

- 通过SEL值，然后查映射表，找到函数实现的IMP
- 再通过IMP找到函数的所在地址，就可以执行函数了
- 就不用大面积遍历搜索方法地址，浪费时间.

###找到了一段苹果开源的，使用SEL查询Method的代码

```
static Method look_up_method(Class cls, SEL sel, BOOL withCache, BOOL withResolver)
{
	
	//保存找到的Method
    Method meth = NULL;
	
	//是否使用方法缓存查找Method
    if (withCache) {
    
    	//先从缓存中查找Method
        meth = _cache_getMethod(cls, sel, &_objc_msgForward_internal);
        
        
        if (meth == (Method)) {
            // Cache contains forward:: . Stop searching.
            return NULL;
        }
    }
	
	//不使用方法缓存，直接从Class或MetaClass查找SEL对应的Method
    if (!meth) meth = _class_getMethod(cls, sel);
		
	//未找到方法Method，回调指向类方法进行拦截 或 消息转发
    if (!meth  &&  withResolver) meth = _class_resolveMethod(cls, sel);

    return meth;
}
```

- 上述代码的逻辑
	
	- 首先去该类的方法 cache 中查找，如果找到了就返回它；

	- 如果没有找到，就去该类的方法列表中查找。如果在该类的方法列表中找到了，则将 Method 返回，并将它加入cache中缓存起来。根据最近使用原则，这个方法再次调用的可能性很大，缓存起来可以节省下次调用再次查找的开销。

	- 如果在该类的方法列表中没找到对应的 Method，在通过该类结构中的 super_class指针在其父类结构的方法列表中去查找，直到在某个父类的方法列表中找到对应的IMP，返回它，并加入cache中；

	- 如果在自身以及所有父类的方法列表中都没有找到对应的 Method，则看是不是可以进行动态方法决议

	- 如果动态方法决议没能解决问题，进入下面要讲的消息转发流程。

- 总之通过SEL值找到了`Method`，但是并没有出现IMP?

- 那么可以看看`Method`的定义，可以发现Method只是结构体类型的一个`指针`

```
typedef struct objc_method *Method;
```

- 那么真正的结构体`objc_method`定义

```
struct objc_method {
	
	//1. 方法的SEL值
    SEL method_name                 OBJC2_UNAVAILABLE;  
    
    //2. 方法的类型
    char *method_types                  OBJC2_UNAVAILABLE;
    
    //3. 函数的指针IMP
    IMP method_imp                      OBJC2_UNAVAILABLE;  
}
```

- 最终通过`objc_method`结构体实例的`method_imp`成员属性值，就是IMP函数指针

- 所以大致顺序如下:

```
SEL >> 查表或缓存 >> objc_method实例 >> IMP
```

***

###`objc_method`实例记录 `方法的SEL` 与 `方法实现IMP` 的关系

- 实际上相当于在`SEL和IMP`之间作了一个映射

- 首先可以快速使用SEL查找到对于的objc_method结构体变量的一个对象

- 继而取出变量的IMP属性，完成了方法实现的查找

****

###应用: 替换掉`SEL`对应的`IMP`实现.

***

###替换方法实现的用处？

- 拦截原始方法的执行

- 在原始方法`执行前`与`执行后`附带的做一些事情

- 最终让原始方法执行，必须发送`当前执行的拦截方法SEL`的消息

***

###简单的替换SEL对应的IMP指针

```
静态方法就交换静态方法，实例方法就交换实例方法

void Swizzle(Class c, SEL origSEL, SEL newSEL)
{
	//1. 要拦截的原始方法（查询传入Class是否存在这个方法）
    Method origMethod = class_getInstanceMethod(c, origSEL);
    
    //2. 
    Method newMethod = nil;
    
    //3. 判断originMethod是类方法还是对象方法
	if (!origMethod) {
			
		//3.1 如果Class存在要拦截的原始方法 --> 类方法
		origMethod = class_getClassMethod(c, origSEL);
		
        if (!origMethod) {
            return;
        }
        
        //说明是交换类方法，则查找新的方法(拦截的方法)
        newMethod = class_getClassMethod(c, newSEL);
        
        if (!newMethod) {
            return;
        }
        
    }else{
    
    	//3.2 如果Class存在要拦截的原始方法 --> 对象方法
        newMethod = class_getInstanceMethod(c, newSEL);
        
        if (!newMethod) {
            return;
        }
    }
    
    //判断Class是否有新的方法实现.（以原始方法的SEL值添加新方法的IMP实现.）
    if(class_addMethod(c, origSEL, method_getImplementation(newMethod), method_getTypeEncoding(newMethod)))
    {
    	//先给类添加新方法实现
    	//然后让新方法的SEL值指向原始方法.
        class_replaceMethod(c, newSEL, method_getImplementation(origMethod), method_getTypeEncoding(origMethod));
    }else{
    	//说明类已经有新方法实现，无需再添加
    	//那么就直接交换SEL对应的IMP指针
        method_exchangeImplementations(origMethod, newMethod);
	}
}
```

***

###例如拦截ViewController的 `viewDidAppear:` 方法



```
- (void)customViewDidAppear:(BOOL)animated{
    
    //1. 目标方法执行前做的事情
    //...
    
    //2. 这一句一定要写，否则被拦截的系统函数不会执行，导致程序崩溃.
    //并且就是发送当前拦截方法的SEL消息
    [self customViewDidAppear:animated];
    
    //3. 目标方法执行后做的事情
    //...
}
```

- 一开始接触runtime时，我对 `2.`继续执行当前方法表示不解，我觉得这样不会导致`死循环`吗？

- 因为 SEL值 `customViewDidAppear:` 指向的正是`系统viewDidLoad方法实现的IMP`

- 所以，要让系统viewDidLoad:方法执行，就必须发送`customViewDidAppear:`这个SEL的消息

