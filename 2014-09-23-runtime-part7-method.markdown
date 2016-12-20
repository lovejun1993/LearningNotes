---
layout: post
title: "Runtime Part7 Method"
date: 2014-09-23 23:48:52 +0800
comments: true
categories: 
---

###`Method`: 抽象一个方法的结构体类型

- `Method`定义为`objc_method结构体`类型的一个`指针类型`
	
```
typedef struct objc_method *Method;
```

Method本身就是一个指针类型，所以不要使用如下代码

```
Method * method = ....
```

而是使用

```
Method method = ...
```

- `objc_method`对一个方法的结构定义

> 通过一个SEL值作为找到一个objc_method实例的唯一id值

```
typedef struct objc_method *Method;

typedef struct objc_ method {
	
	//1. 苹果是SEL这个数据类型，作为一个方法Method实例的标识
    SEL method_name;
		
	//2. 方法的@encode()系统编码字符串
    char *method_types;
	
	//3. 最终OC函数编译生成的c函数代码的地址
    IMP method_imp;
};
```

- 外界通过一个SEL值，从Class的`method_list`查询到一个objc_method实例
- 然后再从 `objc_method实例->method_imp` 获取到函数实现的内存地址
- 再简单的说，`一个SEL >>> objc_method实例 >>> IMP 或 encodingTypes`
- 那么所以，交换两个objc_method实例的`SEL的指向不同的Method`
	- 交换方法的具体实现代码，动态方法拦截代理实现

****

###查找类方法与对象方法

- 类方法 >>> Meta类 中查找

```objc
class_getClassMethod([类 class], @selector(func));
```

- 对象方法 >>> 类本身 中查找

```objc
class_getInstanceMethod([类 class], @selector(func));
```


****

###一个OC类中的一个方法，最终编译生成一个`objc_method`结构体实例的图示

![](http://i4.piimg.com/3d3a440667091342.png)

###这些 type encodings 字符串编码 做什么的？

```
1. return type encodings: 返回之类的编码
method_getArgumentType(<#Method m#>, <#unsigned int index#>, <#char *dst#>, <#size_t dst_len#>)

2. argument type encodings: 方法参数类型的编码
method_getReturnType(<#Method m#>, <#char *dst#>, <#size_t dst_len#>)

3. method type encodings: 方法本身的类型编码（应该包含如上）
method_getTypeEncoding(<#Method m#>)
```

- 每一个OC类的中我们编写的所有的`OC函数`，最后都会编译成一个个独立的 `c函数`

- 但是如何知道每一个独立的`c函数`，对应哪一个`OC函数`了？
	- 每一个OC函数，创建一个 `objc_method结构体实例` 来记录映射关系
	- SEL、记录 c函数 对应 哪一个OC方法SEL
	- IMP、记录 c函数具体实现保存在的哪一个内存块中（内存块首地址）


###比如如下OC方法与对应测C函数实现转换

OC中的函数

```
-(int)say:(NSString *)str {
	OC消息代码
}
```

编译生成的最终c函数实现

```
int say(id self, SEL _cmd, NSString *str) {
	C的函数调用代码
}
```

那么就需要建立OC函数与c函数之间的关系，就是通过method type encodings来做这个事情。

****

###简介Method常用runtime方法

- 基础操作，获取objc_method实例

```
//所属类（提供一个 method_list 存放所有objc_method实例的数组容器）
Class class = NSClassFromString(@"ViewController");

//要获取方法实例的SEL值
SEL selector = NSSelectorFromString(@"viewDidLoad");
    
//从类中获取如上SEL值对应的objc_method实例（对象方法）
Method method = class_getInstanceMethod(class, selector);

//从类中获取如上SEL值对应的objc_method实例（类方法）
Method method = class_getClassMethod(class, selector);
```

- 得到一个`objc_method`实例之后，继续可以从这个`objc_method`实例获得信息

```
//获取方法的SEL值
SEL method_getName ( Method m );
const char * mName= sel_getName(method_getName(method));

//获取方法的IMP
IMP method_getImplementation ( Method m );

//获取方法 描述方法参数 的@encode()编码字符串
const char * method_getTypeEncoding ( Method m );

//获取方法 方法返回值类型 的@encode()编码字符串
char * method_copyReturnType ( Method m );

//获取方法 形参列表所有参数的 的@encode()编码字符串
char *method_copyArgumentType(Method m, unsigned int index);

//获取方法的参数总个数
unsigned int method_getNumberOfArguments ( Method m );
```

- 设置方法实现、添加方法实现、替换方法实现

```
//给某个类添加一个方法实现
BOOL class_addMethod(Class cls, SEL name, IMP imp,  const char *types) 

//设置方法的实现
IMP method_setImplementation ( Method m, IMP imp );

//`修改`类的Method IMP
IMP class_replaceMethod(Class cls, SEL name, IMP imp, const char *types) 

//`交换`2个方法中的IMP
void method_exchangeImplementations ( Method m1, Method m2 );
```

###获取Class的 method_list 方法列表

```
//1. Method个数
unsigned int methodCount;
	
//2. Method结构体数组
Method *methodList = class_copyMethodList([self class], & methodCount);
	
//3. 依次遍历每一个Method
for (unsigned int i; i<count; i++) {
    
    //取出当前遍历额Method
	Method method = methodList[i];
	    
	//得到Method的方法名的SEL值
	SEL sel = method_getName(method)
	
	//根据SEL值得到方法名
	NSString *methodName = NSStringFromSelector(sel);
}
```

###`class_addMethod()`函数所有参数意义

```
//参数1: 某个类
//参数2: 方法的SEL值
//参数3: 要添加的 c方法的方法名 or 地址 （IMP）
//参数4: 要添加的 方法的编码字符串
BOOL class_addMethod ( Class cls, SEL name, IMP imp, const char *types ); 
```
	
```
注意:
和成员变量不同的是可以为类动态添加方法。
如果有同名会返回NO。
修改的话需要使用method_setImplementation()函数
```

举例使用`class_addMethod()`函数，直接给OC类添加一个c实现函数作为objc_method实例的IMP属性值

```objc
static const NSString* my_constMehtod(NSString * name) {
    return @"给OC类直接添加一个c函数实现";
}

@implementation ViewController

+ (void)load {
    [self testConstMethodImpl];
}

+ (void)testConstMethodImpl {
    IMP imp = my_constMehtod;
    
    class_addMethod([ViewController class],
                    @selector(constMehtod),
                    imp,
                    "@@:@");//c实现函数格式的编码
}

//测试调用创建的c函数实现
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	[self testConstMethod];
}

- (void)testConstMethod {
    
    //1. 完成方法调用，并获取返回值
    NSMethodSignature *sign = [NSMethodSignature signatureWithObjCTypes:"@@:@"];
    NSInvocation *invocate = [NSInvocation invocationWithMethodSignature:sign];
    [invocate setTarget:self];
    [invocate setSelector:@selector(constMehtod)];
    [invocate invoke];
    
    void *ret = NULL;
    [invocate getReturnValue:&ret];
    NSString *retString = (__bridge NSString *)(ret);
    NSLog(@"ret = %@", retString);
    
    //2. 输出所有的方法参数编码
    NSInteger numberOfArguments = [sign numberOfArguments];
    for (NSInteger i = 0; i < numberOfArguments; i++) {
        const char *argumentType = [sign getArgumentTypeAtIndex:i];
        NSLog(@"%s", argumentType);
    }
}

@end
```

运行App点击屏幕输出如下

```
2016-03-31 18:35:10.916 JSPatchDeme[77051:506442] 原始viewDidLoad方法实现
2016-03-31 18:35:11.536 JSPatchDeme[77051:506442] ret = 给OC类直接添加一个c函数实现
2016-03-31 18:35:11.536 JSPatchDeme[77051:506442] @
2016-03-31 18:35:11.536 JSPatchDeme[77051:506442] :
2016-03-31 18:35:11.537 JSPatchDeme[77051:506442] @
```

可以看到c函数实现中的所有参数的编码

- 第一个是@ >>>> 返回值类型。如果是void，那么就是v
- 第二个是: >>>> 
- 第三个是@ >>>>

###`class_replaceMethod()`函数所有参数意义

```
//参数1: 某个类
//参数2: 被替换的方法的SEL值
//参数3: 将要要替换成的新的 c方法的方法名 or 地址 （IMP）
//参数4: `被替换的原始` c函数的编码字符串
IMP class_replaceMethod(Class cls, SEL name, IMP imp, 
                                    const char *types) 
```

eg、

```
IMP originIMP = class_replaceMethod(cls , 
									overrideSEL, 
									method_getImplementation(新的method),
									method_getTypeEncoding(origMethod)
									);
```

该方法返回的IMP，是被替换的`原始c函数实现`的函数名 (函数指针).

***

###获取某个函数的实现指针IMP

- 当获取到一个Method实例时

```
IMP method_getImplementation(Method method)
```

- 只知道一个Method实例对应的SEL时

```
#import <Foundation/Foundation.h>

@interface Person : NSObject

@property (nonatomic, copy) NSString *name;

@end

@implementation Person

@end
```

```
Person *p = [Person new];
    
//使用函数指针
void (*func)(id, SEL, NSString*);
func = (void (*)(id, SEL, NSString*))[p methodForSelector:@selector(name)];

//直接使用IMP
IMP imp = [p methodForSelector:@selector(name)];
```

***

###交换两个SEL指向的最终编译生成的c函数（具体实现IMP、函数地址、函数名）


![](http://i4.piimg.com/aa17cd0c9ccfdcc6.png)


###一、直接替换ViewController中的SEL指向我们自己写的C函数实现

```
void my_viewWillAppear(id self, SEL _cmd, BOOL isAnimate) {

}

@implementation ViewController

+ (void)load {
    
    Method m = class_getInstanceMethod([ViewController class], @selector(viewWillAppear:));
    const char *types = method_getTypeEncoding(m);
    
    class_replaceMethod([ViewController class],
                        @selector(viewWillAppear:),
                        (IMP)my_viewWillAppear,
                        types);
}

@end
```

###二、替换ViewController中的SEL指向自己写的OC函数

```objc
@implementation ViewController

+ (void)load {
    
    Method m = class_getInstanceMethod([ViewController class], @selector(viewWillAppear:));
    const char *types = method_getTypeEncoding(m);
    
    class_replaceMethod([ViewController class],
                        @selector(viewWillAppear:),
                        class_getMethodImplementation([ViewController class], @selector(my_viewWillAppear:)),
                        types);
}

- (void)my_viewWillAppear:(BOOL)animated {
    
}

@end
```

注意: 

如上两个例子只是能够完成拦截，但是拦截之后并不能够回到原来的函数继续执行，仅做demo示例。

###三、hook之后继续让原始函数进行执行，并直接使用c函数作.

```objc
#import "ViewController.h"
#import <objc/message.h>
#import <objc/runtime.h>

//替换成的c函数的声明
static void container_viewDidAppear(id self, SEL sel, BOOL flag);

//替换成的c函数的实现
static void container_viewDidAppear(id self, SEL sel, BOOL flag) {
	//让被hook的原始函数实现继续执行
    ViewDidAppearIMP(self, sel, flag);
}

//指向替换成的c函数的指针
static void (*ViewDidAppearIMP)(id self, SEL sel, BOOL flag);

//保存被替换的原始c函数
typedef IMP *IMPPointer;

/**
 *  直接使用IMP的c函数实现的形式，交换SEL指向的IMP
 *
 *  @param class       交换实现的Class
 *  @param original    要交换的SEL
 *  @param replacement 交换成的c实现函数地址
 *  @param store       用来保存被交换的原始c函数实现IMP
 */
BOOL class_swizzleMethodAndStore(Class class, SEL original, IMP replacement, IMPPointer store) {

    IMP imp = NULL;
	
	// 获取传入Class中要替换实现的SEL对应的 objc_method实例    
    Method method = class_getInstanceMethod(class, original);
    
    if (method) {
    
    	// 方法实现的编码
        const char *type = method_getTypeEncoding(method);
        
        // 将原来OC函数的IMP替换为传入的新的IMP，并记录之前的OC函数的IMP
        imp = class_replaceMethod(class, original, replacement, type);
        
        // 替换IMP失败，获取原来的OC函数实现IMP
        if (!imp) {
            imp = method_getImplementation(method);
        }
    }
    
    // 保存被替换实现的OC函数对应的IMP
    if (imp && store) {
        *store = imp;
    }
    
    // imp不为NULL，表明交换成功
    return (imp != NULL);
}

@implementation ViewController

- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    //只交换一次
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        class_swizzleMethodAndStore([self class],
                                    @selector(viewDidAppear:),
                                    (IMP)container_viewDidAppear,
                                    (IMP *)&ViewDidAppearIMP);
    });
    
    [self.navigationController pushViewController:[UIViewController new] animated:YES];
}

@end
```

***


###交换两个OC函数实现的IMP的简单版本，复杂版本可参阅JRSwizzle

```objc
分情况进行 1)类方法交换实现 2)对象方法交换实现

/**
 *  交换一个类中的两个方法SEL指向的IMP
 *
 *  @param cls       类
 *  @param origSEL   原始方法的SEL
 *  @param newSEL    交换后新方法的SEL
 */
void Swizzle(Class cls, SEL origSEL, SEL newSEL)
{
    
    //1. 首先尝试获取对象方法的实现
    Method origMethod = class_getInstanceMethod(cls, origSEL);
    Method newMethod = nil;
    
    //2. 判断originMethod是`类`方法还是`对象`方法
    if (!origMethod) {
        
        //2.1 交换类方法实现
        
        // 获取类方法
        origMethod = class_getClassMethod(cls, origSEL);
        
        if (!origMethod) {
            return;
        }
        
        //说明是交换类方法，则查找新的方法(拦截的方法)
        newMethod = class_getClassMethod(cls, newSEL);
        
        //类中不存在新方法实现
        if (!newMethod) {
            return;
        }
        
    }else{
        
        //2.2 交换对象方法
        
        //获取新方法实现
        newMethod = class_getInstanceMethod(cls, newSEL);
        
        //未找到新方法实现
        if (!newMethod) {
            return;
        }
    }
    
    /**
		1. 这里有一个注意的问题 >>> 要交换的方法有两种情况:
        
        情况一、被替换实现的函数，没有出现在当前类/对象，而是出现在当前类的父类，也就是当前类中并未重写.
        情况二、被替换实现的函数，已经在当前类重写.
        情况三、是当前类/对象自己的函数，并不是继承过来的.
     */
    if(class_addMethod(cls, origSEL, method_getImplementation(newMethod), method_getTypeEncoding(newMethod)))
    {
        //情况一: 间接交换方法实现
        class_replaceMethod(cls, newSEL, method_getImplementation(origMethod), method_getTypeEncoding(origMethod));
    
    } else {
    
        //情况二、情况三: 直接交换方法实现
        method_exchangeImplementations(origMethod, newMethod);
    }
}
```

###注意上面注释的一个问题: 被替换实现的函数是否是继承自父类.

- （情况一）如果是被替换实现的函数，是继承自父类得到的

	- 那么直接使用exchange()函数去交换SEL对应的IMP
	- 就会造成`父类`的方法实现IMP被交换，继而造成 该父类的`所有子类`的这个方法实现，也同样会被替换
	- 那么就会导致，该父类的其他子类，也会走拦截的方法中的逻辑，造成一些错误
	- 显然只应该替换当前类的方法实现，而不应该影响其他子类
     
     
- （情况二、情况三）如果该函数不是通过继承、该函数虽然是继承但是已经在当前类重写
	- 直接可以通过exchange()函数去交换SEL对应的IMP
	- 只会对当前类自己的方法实现去替换，而不会影响其他子类

- 那么结合如上三种情况，最完善的逻辑如下:
        	
	- 向当前类的`objc_class`结构体中，新添加一个新的 `objc_method`实例
		- 使用要被替换实现的方法 `originSEL`
		- 指向的新的函数实现 `newIMP`
	
	- 如果添加成功 >>> `进入情况一`
			- 第一步、向当前类 使用 `class_addMethod()函数` 添加`新的`方法实现，使用newSEL
          - 第二步、使用 `class_replaceMethod()函数` 让 新的方法SEL 指向 被替换掉的原始方法实现IMP（为了拦截后回到原来的函数）
	
	- 如果添加失败 >>> `进入情况二、情况三`说明被替换的函数实现已经存在于当前类的`objc_class`中
		- 直接使用 `method_exchangeImplementations()函数` 交换 原始方法SEL 与 新方法SEL指向的IMP

###那么我觉得对于 `method_exchangeImplementations()` 与 `class_replaceMethod()` 这两个函数的区别是:

- exchange >>> 如果两个方法的实现早就存在于类中（代码编写阶段）
- replace >>> 如果一个方法实现是在运行时动态添加的（程序运行时阶段）

不知道这样理解是否有误，如果有误再来更正。

***

###`JRSwizzle`源码

- NSObject+JRSwizzle.h


```objc
#import <Foundation/Foundation.h>

@interface NSObject (JRSwizzle)

/**
 *	交换对象的方法实现
 */
+ (BOOL)jr_swizzleMethod:(SEL)origSel_ withMethod:(SEL)altSel_ error:(NSError**)error_;

/**
 *	交换类的方法实现
 */
+ (BOOL)jr_swizzleClassMethod:(SEL)origSel_ withClassMethod:(SEL)altSel_ error:(NSError**)error_;

@end
```

- NSObject+JRSwizzle.m

```objc
#import "NSObjct+JRSwizzle.h"

//根据iPhone与mac os区别导入runtime库
#if TARGET_OS_IPHONE
	#import <objc/runtime.h>
	#import <objc/message.h>
#else
	#import <objc/objc-class.h>
#endif

//快速组装一个NSError实例
#define SetNSErrorFor(FUNC, ERROR_VAR, FORMAT,...)	\
	if (ERROR_VAR) {	\
		NSString *errStr = [NSString stringWithFormat:@"%s: " FORMAT,FUNC,##__VA_ARGS__]; \
		*ERROR_VAR = [NSError errorWithDomain:@"NSCocoaErrorDomain" \
										 code:-1	\
									 userInfo:[NSDictionary dictionaryWithObject:errStr forKey:NSLocalizedDescriptionKey]]; \
	}
#define SetNSError(ERROR_VAR, FORMAT,...) SetNSErrorFor(__func__, ERROR_VAR, FORMAT, ##__VA_ARGS__)

//不同SDK获取对象的Class
#if OBJC_API_VERSION >= 2
#define GetClass(obj)	object_getClass(obj)
#else
#define GetClass(obj)	(obj ? obj->isa : Nil)
#endif


@implementation NSObject (JRSwizzle)

//先看交换类方法实现指针
+ (BOOL)jr_swizzleClassMethod:(SEL)origSel_ withClassMethod:(SEL)altSel_ error:(NSError**)error_ {

	//1. 获取当前使用swizzle的Class
	Class targetCls = GetClass((id)self);
	
	//2. 再用获得的Class调用swizzle方法
	return [targetCls jr_swizzleMethod:origSel_ withMethod:altSel_ error:error_];
}

//最终交换方法实现
+ (BOOL)jr_swizzleMethod:(SEL)origSel_ withMethod:(SEL)altSel_ error:(NSError**)error_ 
{

//首先判断当前运行的iOS SDK支持的 OBJC_API_VERSION 版本号，使用不同的Runtime Api进行交换方法交换

#if OBJC_API_VERSION >= 2

	//下面是当 `OBJC_API_VERSION >= 2`时使用苹果封装后的一些较高级方便使用的Runtime Api完成方法交换
	
	//1. 首先获取原始方法SEL对于的Method实例
	Method origMethod = class_getInstanceMethod(self, origSel_);
	
	//2. 如果没有找到方法实现Method，先创建NSError，然后返回NO，结束交换
	if (!origMethod) {
		
		//先创建NSError，并返回
#if TARGET_OS_IPHONE
		SetNSError(error_, @"original method %@ not found for class %@", NSStringFromSelector(origSel_), [self class]);
#else
		SetNSError(error_, @"original method %@ not found for class %@", NSStringFromSelector(origSel_), [self className]);
#endif
		
		然后返回NO，结束交换
		return NO;
	}
	
	//3. 再获取替换方法SEL的Method实例
	Method altMethod = class_getInstanceMethod(self, altSel_);
	
	//4. 同样判断有没有找到方法实现Method，如果没有就先创建NSError，然后返回NO，结束交换
	if (!altMethod) {
#if TARGET_OS_IPHONE
		SetNSError(error_, @"alternate method %@ not found for class %@", NSStringFromSelector(altSel_), [self class]);
#else
		SetNSError(error_, @"alternate method %@ not found for class %@", NSStringFromSelector(altSel_), [self className]);
#endif
		return NO;
	}
	
	//5. 给当前Class的在运行时，添加originMethod，并使用originSEL指向
	class_addMethod(self,
					origSel_,
					class_getMethodImplementation(self, origSel_),
					method_getTypeEncoding(origMethod));
					
	//6. 给当前Class的在运行时，添加altMethod，并使用altSEL指向
	class_addMethod(self,
					altSel_,
					class_getMethodImplementation(self, altSel_),
					method_getTypeEncoding(altMethod));
					
	//注意，如上5，6步，是防止传入的两个方法没有找到引起崩溃
	
	//7. 交换originSEL与altSEL指向的函数实现IMP
	method_exchangeImplementations(class_getInstanceMethod(self, origSel_), 
								   class_getInstanceMethod(self, altSel_));
								   
	//8. 返回交换成功
	return YES;
	
#else 
	
	//下面是当 `OBJC_API_VERSION < 2`时使用的`低级`的Runtime API进行方法交换
	
	//----------------- 1. 从当前Class的Method列表中，不是继承自父类的Method方法 中查找originMethod与altMethod-------------------
	
	Method directOriginalMethod = NULL;
	Method directAlternateMethod = NULL;
	
	void *iterator = NULL;
	
	//获取到当前Class的Method列表（数组）
	struct objc_method_list *mlist = class_nextMethodList(self, &iterator);
	
	//遍历Method数组，获取到originMethod与altMethod
	while (mlist) {
		
		//用来记录遍历的下标值
		int method_index = 0;
		for (; method_index < mlist->method_count; method_index++) {
			
			//使用originSEL判断，找到originMethod
			if (mlist->method_list[method_index].method_name == origSel_) {
				assert(!directOriginalMethod);

				//找到Method的地址
				directOriginalMethod = &mlist->method_list[method_index];
			}
			
			//使用altSEL判断，找到altMethod
			if (mlist->method_list[method_index].method_name == altSel_) {
				assert(!directAlternateMethod);
				
				//找到Method的地址
				directAlternateMethod = &mlist->method_list[method_index];
			}
		}
		
		//指针移到下一个Method
		mlist = class_nextMethodList(self, &iterator);
	}
	
	//----------------- 2. 如果从当前Class没有找到两个Method，就从`superClass`查找，并复制到当前Class -------------------

	//从SuperClass查找Method
	if (!directOriginalMethod || !directAlternateMethod) {
		
		Method inheritedOriginalMethod = NULL, inheritedAlternateMethod = NULL;
		//处理originMethod没找到
		if (!directOriginalMethod) {
		
			//会从superClass查找
			inheritedOriginalMethod = class_getInstanceMethod(self, origSel_);
			
			if (!inheritedOriginalMethod) {
				SetNSError(error_, @"original method %@ not found for class %@", NSStringFromSelector(origSel_), [self className]);
				return NO;
			}
		}
		
		//处理altMethod没找到
		if (!directAlternateMethod) {
		
			//会从superClass查找
			inheritedAlternateMethod = class_getInstanceMethod(self, altSel_);
			
			if (!inheritedAlternateMethod) {
				SetNSError(error_, @"alternate method %@ not found for class %@", NSStringFromSelector(altSel_), [self className]);
				return NO;
			}
		}
		
		//从当前Class中为查找到Method的个数
		int hoisted_method_count = !directOriginalMethod && !directAlternateMethod ? 2 : 1;
		
		//创建一个`struct objc_method_list`结构体类型的实例
		//总内存大小 = objc_method_list结构体初始大小 + method_list数组大小
		struct objc_method_list *hoisted_method_list = malloc(sizeof(struct objc_method_list) + (sizeof(struct objc_method)*(hoisted_method_count-1)));
		
		//给创建出来`struct objc_method_list`结构体类型的实例，进行赋值
		hoisted_method_list->obsolete = NULL;
		hoisted_method_list->method_count = hoisted_method_count;//记录从当前cLASS未找到Method的个数
		
		//使用一个指针指向`objc_method_list结构体实例`的方法数组属性`method_list`
		Method hoisted_method = hoisted_method_list->method_list;
		
		//如果在当前Class没有找到originMethod
		if (!directOriginalMethod) {
		
			//extern void bcopy(const void *src, void *dest, int n);
			//将从父类找到的Method复制到之前创建的Method列表中的Method数组的第一个地址位置
			bcopy(inheritedOriginalMethod, hoisted_method, sizeof(struct objc_method));
			
			//Method数组指针移到下一个位置
			directOriginalMethod = hoisted_method++;
		}
		
		//如果在当前Class没有找到altMethod
		if (!directAlternateMethod) {
			
			//如上
			bcopy(inheritedAlternateMethod, hoisted_method, sizeof(struct objc_method));
			directAlternateMethod = hoisted_method;
		}
		
		//将Method列表，添加到当前Class
		class_addMethods(self, hoisted_method_list);
	}
	
	//----------------- 3. 交换方法实现 -------------------
	IMP temp = directOriginalMethod->method_imp;
	directOriginalMethod->method_imp = directAlternateMethod->method_imp;
	directAlternateMethod->method_imp = temp;
	
	return YES;
	
#endif	
}

@end
```

- 小结如上源码:
	- objc2.0 与 objc1.0 使用Runtime Api的区别
	- `extern void bcopy(const void *src, void *dest, int n);`函数完成复制

***

###获取类对象的方法的签名

```
NSMethodSignature *methodSignature = [cls methodSignatureForSelector:selector];
```

随便输出得到的方法签名内容如下面所示:

```
<NSMethodSignature: 0x7fc22841c7a0>
    number of arguments = 2
    frame size = 224
    is special struct return? NO
    return value: -------- -------- -------- --------
        type encoding (@) '@'
        flags {isObject}
        modifiers {}
        frame {offset = 0, offset adjust = 0, size = 8, size adjust = 0}
        memory {offset = 0, size = 8}
    argument 0: -------- -------- -------- --------
        type encoding (@) '@'
        flags {isObject}
        modifiers {}
        frame {offset = 0, offset adjust = 0, size = 8, size adjust = 0}
        memory {offset = 0, size = 8}
    argument 1: -------- -------- -------- --------
        type encoding (:) ':'
        flags {}
        modifiers {}
        frame {offset = 8, offset adjust = 0, size = 8, size adjust = 0}
        memory {offset = 0, size = 8}
```

可以看到，可以很完整的描述一个方法的所有信息。那么在构造NSInvocation的时候，需要得到一个方法的签名设置给NSInvocation，让NSInvocation知道如何调用这个方法.

***

###如何判断method是否被swizzled

> 这个学习来源: https://segmentfault.com/a/1190000004542608、https://segmentfault.com/a/1190000003950284。

从上面可以学习到任何的Objective-C的方法实现都可以在运行时替换成另外一个Objective-C方法实现，那么有时候可能需要检测一个Objective-C方法实现是否已经被替换掉，那如何判断了？

使用如下代码进行判断:

```c
static inline bool isDiff(const char *func, SEL _cmd)
{
    char buff[256] = {'\0'};
    if (strlen(func) > 2) {
        char* s = strstr(func, " ") + 1;
        char* e = strstr(func, "]");
        memcpy(buff, s, sizeof(char) * (e - s) );
        return strcmp(buff, sel_getName(_cmd));
    }
    return false;
}
```

测试代码如下:

```objc
#import "SwizzleViewController.h"
#import <objc/runtime.h>

void Swizzle(Class cls, SEL origSEL, SEL newSEL)
{
    Method origMethod = class_getInstanceMethod(cls, origSEL);
    Method newMethod = nil;
    
    if (!origMethod) {
        origMethod = class_getClassMethod(cls, origSEL);

        if (!origMethod) {
            return;
        }
        
        newMethod = class_getClassMethod(cls, newSEL);
        if (!newMethod) {
            return;
        }
        
    }else{
        newMethod = class_getInstanceMethod(cls, newSEL);
        if (!newMethod) {
            return;
        }
    }
    
    if(class_addMethod(cls, origSEL, method_getImplementation(newMethod), method_getTypeEncoding(newMethod)))
    {
        class_replaceMethod(cls, newSEL, method_getImplementation(origMethod), method_getTypeEncoding(origMethod));
        
    } else {
        method_exchangeImplementations(origMethod, newMethod);
    }
}

/**
 *  就是比较 __PRETTY_FUNCTION__中的sel字符串 是否与 _cmd字符串 完全相对
 *
 *  @param func  __PRETTY_FUNCTION__，如:  -[SwizzleViewController testMethod]
 *  @param _cmd 当前方法的sel，如: _swizzle_testMethod
 *
 *  @return YES没有替换实现，NO表示替换了实现
 */
static inline bool isDiff(const char *func, SEL _cmd)
{
    char buff[256] = {'\0'};
    if (strlen(func) > 2) {
        char* s = strstr(func, " ") + 1;
        char* e = strstr(func, "]");
        memcpy(buff, s, sizeof(char) * (e - s) );
        return strcmp(buff, sel_getName(_cmd));
    }
    return false;
}


@implementation SwizzleViewController

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Swizzle([SwizzleViewController class], @selector(testMethod), @selector(_swizzle_testMethod));
        Swizzle([SwizzleViewController class], @selector(viewDidLoad), @selector(_swizzle_viewDidLoad));
    });
}

- (void)testMethod {

    NSLog(@"__PRETTY_FUNCTION__ = %@", [NSString stringWithUTF8String:__PRETTY_FUNCTION__]);
    NSLog(@"_cmd = %@", NSStringFromSelector(_cmd));
    
    // 判断函数实现是否已经被替换
    BOOL ret = isDiff(__PRETTY_FUNCTION__, _cmd);
    if (ret) {
        NSLog(@"实现已经被替换");
    } else {
        NSLog(@"实现未被替换");
    }
    
    NSLog(@"origin method implementations");
}

- (void)_swizzle_testMethod {
    NSLog(@"swizzle method implementations");
    [self _swizzle_testMethod];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self testMethod];
    
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    NSLog(@"__PRETTY_FUNCTION__ = %@", [NSString stringWithUTF8String:__PRETTY_FUNCTION__]);
    NSLog(@"_cmd = %@", NSStringFromSelector(_cmd));
    
    // 判断函数实现是否已经被替换
    BOOL ret = isDiff(__PRETTY_FUNCTION__, _cmd);
    if (ret) {
        NSLog(@"实现已经被替换");
    } else {
        NSLog(@"实现未被替换");
    }

}

- (void)_swizzle_viewDidLoad {
    [self _swizzle_viewDidLoad];
}

@end
```

运行结果

```
2016-08-03 19:21:25.225 Demos[38746:2541168] __PRETTY_FUNCTION__ = -[SwizzleViewController viewDidLoad]
2016-08-03 19:21:25.225 Demos[38746:2541168] _cmd = _swizzle_viewDidLoad
2016-08-03 19:21:25.225 Demos[38746:2541168] 实现已经被替换
2016-08-03 19:23:36.459 Demos[38746:2541168] swizzle method implementations
2016-08-03 19:23:36.459 Demos[38746:2541168] __PRETTY_FUNCTION__ = -[SwizzleViewController testMethod]
2016-08-03 19:23:36.459 Demos[38746:2541168] _cmd = _swizzle_testMethod
2016-08-03 19:23:36.459 Demos[38746:2541168] 实现已经被替换
2016-08-03 19:23:36.459 Demos[38746:2541168] origin method implementations
```

发现系统的方法实现也是可以判断的，不知道为啥原文作者说不可以。

有一个注意点，在被替换后执行的原始方法实现中，`_cmd`表示的是替换实现的方法sel。

****

学习资源

```
http://blog.csdn.net/yiyaaixuexi/article/details/9374411
http://www.v2ex.com/t/273893
https://segmentfault.com/a/1190000004542608
```