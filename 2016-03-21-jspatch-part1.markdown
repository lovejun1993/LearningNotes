---
layout: post
title: "JSPatch Part1"
date: 2016-03-21 11:27:47 +0800
comments: true
categories: 
---

###之前几次面试都问到过动态更新，我当时回到到的几种方法:

- WebView加载H5动态更新
- lua文件控制动态更新
	- 其实我知道，但是没说，因为几乎从来没接触过
- react native框架js动态更新
	- 这个只要关注过iOS动态的基本都知道，但是我从来没看过实现原理，所以...
- framework模块化后动态库式更新
	- 当时我说了，但是那个面试官说没有签名，不能在iOS系统运行
	- 这之前对Mach-O、代码签名确实从来没去看过，真的是不懂
	- 后来花了几天时间看懂了，于是我对那天面试官说`没有签名，不能在iOS系统运行`的说法来否定使用framework动态更新的方法，我表示疑惑
	- 既然是没有通过签名，那么我不能够在下载framework之前，已经给他签好名吗？呵呵
		- 学习博客地址:`http://devguo.com/blog/2015/06/16/iosdong-tai-geng-xin-frameworkshi-xian/`
		- **查阅苹果官方文档后，苹果支持的可下发可执行文件只有两种**	
			- lua脚本与js脚本
			- 所以.framewoek、.a等其他形式的可执行文件不允许下发的

- jspatch框架实现动态更新
	- 如其名字，js + patch
	- **使用javaScript代码调用原生Objetive-C方法**
	- 实现patch补丁包的形式增量更新
	- 之前面试很过都问到过，我也知道但是没说，因为没有看过他的实现原理也没具体使用过
	- 学习来源: https://github.com/bang590/JSPatch/

- 其实在jspatch之前，还有一个叫做`waxpatch`的框架
	- **使用Lua代码调用原生Objetive-C方法**

- jspatch与waxpatch的比较
	- jspatch使用`javascript`代码调用原生oc代码
	- waxpatch使用`lua`代码调用原生oc代码
	- javascript代码比lua代码更亲民...
	- 苹果规定动态更新代码，只允通过`JavaScriptCore.framework或WebKit执行的代码`
		- 所以jspatch比waxpatch更符合Apple规则
	- 使用系统内置的JavaScriptCore.framework，无需内嵌lua脚本引擎来解释运行lua代码
	- jspatch只能运行在`iOS7及更高版本`的系统上
		- JavaScriptCore.framework是在`iOS7`加入的

***

###学习JSPatch实现的原因

- 可以在项目中更好的使用热修复机制
- 弄清楚到底是如何热修，复原生oc类代码中产生crash方法
- 对runtime的一种应用学习
- JavaScriptCore.framework使用
- js <---> JavaScriptCore.framework <---> oc

***

###JSPatch开篇

下面这段话摘录自JSPtach Wiki

- JSPatch 是一个 iOS 动态更新框架
- 只需在项目中引入极小的`引擎` （JavaScriptCore.framework）
- 就可以使用 JavaScript 调用任何 Objective-C 原生接口
- 获得脚本语言的优势
	- 为项目动态添加`模块`
	- `替换`App程序代码中的原生Objective-C代码
		- eg、动态修复某一行产生bug、崩溃的原生代码

		

****

###JSPatch VS WebView加载H5网页

- 本地的js 与 网页服务端的js
	- JSPatch可以通过从服务端下载js代码包，存放到手机本地
		- 直接操作原生OC代码
	- 通常H5界面上的js代码是跑在浏览器上，存在于服务端
		- 如果需要操作原生OC代码，需要借助`WebView`
		- 实现WebView的delegate方法，在此方法中`拦截`与服务端上来好的`url`的请求

		```
		- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType		
		```
		
		- 其实也有通过下载到本地的H5，但仍然没有通过直接下载js来的方便、小巧

- App从服务端下载js代码需要注意
	
	- 不要在网络传输过程中下发`明文`的JS，一定需要加密，否则:
		- 可能会被中间人篡改JS脚本，达到执行任意方法攻击App
		- 可以对传输过程进行`加密`，或用直接使用`https网络链接`解决

	- 即使js下载到手机沙盒本地后，同样也需要`加密存储`
		- 在`越狱`后的手机，是可以随便访问沙盒目录的任何文件
		- 其实在`未越狱`的机器上用户也可以手动替换或篡改脚本
			- 因为iOS系统上其实可以进行调试、打开任何App
			- 可以通过加载lldb/gdb脚本副本
			- 具体参见iOS安全攻防之类的

***


###JSPatch的核心: 使用Objective-C的Runtime机制来替换OC方法的SEL所指向的方法实现IMP

比如，在App程序启动时，修改某个自定义ViewController的实例方法viewDidLoad

> 如果是交换UIViewController的方法，不能像如下写，参看Runtime Method文章记录的.


- 在main.m添加修改SEL指向IMP的代码

```c
#import <UIKit/UIKit.h>
#import "AppDelegate.h"
#import <objc/runtime.h>

static void newViewDidLoad(id slf, SEL sel) {
    NSLog(@"交换后的viewDidLoad方法");
}

void my_swizlle() {
    
    //1.
    Class class = NSClassFromString(@"ViewController");
    
    //2.
    SEL selector = NSSelectorFromString(@"viewDidLoad");
    
    //3.
    Method method = class_getInstanceMethod(class, selector);
    
    //3.
    class_replaceMethod(class,
                        selector,
//                        method_getImplementation(method),
                        (IMP)newViewDidLoad,
                        method_getTypeEncoding(method)
                        );
}

int main(int argc, char * argv[]) {
    @autoreleasepool {
    	//1. 修改SEL指向的IMP
        my_swizlle();
        
		//2. 启动App程序
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

注意，`my_swizzle`方法修改的是`ViewController`而不是`UIViewController`，下面的代码才适合交换`从父类继承的方法`的情况:

```objc
/**
 *  交换一个类中的两个方法SEL指向的IMP
 *
 *  @param cls       类
 *  @param origSEL   原始方法的SEL
 *  @param newSEL    交换后新方法的SEL
 */
void Swizzle(Class cls, SEL origSEL, SEL newSEL)
{
    
    //1.
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
     *  这里有一个注意的问题 >>> 要交换的方法有两种情况:
        
            情况一、原始方法没有出现在当前类，而是出现在当前类的父类中继承过来的，并未在当前类重新实现
            情况二、原始方法已经在当前类重新实现
     
        如果是情况一，直接使用exchange交换SEL，那么可能造成`父类`的方法实现IMP被交换
     
        结合上面两种情况:
        - 使用原始方法SEL添加一个新的方法实现IMP
            - 添加成功 >>> 情况一，再修改新方法SEL指向原始方法实现IMP
            - 添加失败 >>> 情况二，直接交换原始方法与新方法的SEL指向的IMP
     
     */
    if(class_addMethod(cls, origSEL, method_getImplementation(newMethod), method_getTypeEncoding(newMethod)))
    {
        //情况一:
        class_replaceMethod(cls, newSEL, method_getImplementation(origMethod), method_getTypeEncoding(origMethod));
    }else{
        //情况二:
        method_exchangeImplementations(origMethod, newMethod);
    }
}
```

代码运行后，效果是只走了main.m中的newViewDidLoad方法，却没有走ViewController中的viewDidLoad方法，就形成了`替换`方法的实现的效果。道理很简单了，因为没有让newViewDidLoad方法SEL指向viewDidLoad方法IMP。

***

###iOS7之后引入了JavaScriptCore.framework，主要是为了完成:

> Objective-C代码 与 JavaScript代码 之间的互调

- 在Objective-C代码中直接执行JavaScript代码段

- 在JavaScript语言环境里调用Objective-C公开给JavaScript的方法

- 内存管理和线程封装

****

###JavsScriptCore规定了js中弱类型与Objective-C中强类型的映射

| Objective-C数据类型 | js数据类型 | 
| :------------- |:-------------:| 
| nil | undefined | 
| NSNull | null | 
| NSString | string |
| NSNumber | nsnumber、boolean |
| NSDictionary | Object |
| NSArray | Array |
| NSDate | Date |
| NSBlock | Function |
| id | Wrapper |
| Class | Constructor |

****

###JavsScriptCore主要使用到Api

```
JSContext，JS代码的运行环境
```

```
JSValue，OC强数据类型 转换 JS弱数据类型
```

```
JSManagedValue: 解决OC对象与JS对象不同的内存管理机制问题

OC的内存管理机制是基于`引用计数`
JavaScript则如同Java/C#那样用的是垃圾回收机制(GC, Garbage Collection)

主要解决OC对象与JS对象之间互掉时，内存释放的问题:
防止OC使用JS对象时，JS对象被释放.
防止JS使用OC对象时，OC对象被释放.

使用步骤:
- 将一个JSValue转成JSManagedValue
- 然后将转换的JSManagedValue添加到JSVirtualMachine中
- 这样在运行期间就可以保证在Objective-C和JavaScript两侧都可以正确访问对象而不会造成不必要内存释放问题
```

```
JSVirtualMachine:

用来管理OC对象与JS对象的声明周期.
```

```
JSExpor:

一个协议，如果JS对象想直接调用OC对象里面的方法和属性，那么这个OC对象只要实现这个JSExport协议就可以了
```

****

###看使用JavaScriptCore.framework完成Objective-C代码中调用js方法

####先来个简单的，直接通过JSContext运行js代码

例子一

```objc
//1. 创建一个js环境
JSContext *context = [[JSContext alloc] init];

//2. 在js环境中加载执行js代码
[context evaluateScript:@"function add(a, b) { return a + b; }"];

//3. 取出js环境中的js方法
JSValue *add = context[@"add"];
NSLog(@"Func: %@", add);

//4. 调用js方法，并接受其js方法的返回值
JSValue *sum = [add callWithArguments:@[@(7), @(21)]];
NSLog(@"Sum: %d",[sum toInt32]);
```

例子二

```
JSContext *context = [[JSContext alloc] init];
    
//在js运行时环境中创建一个js数组对象
[context evaluateScript:@"var arr = [21, 7 , 'iderzheng.com'];"];
    
//从js运行时环境取出js数组对象 >>> 取出的是JSValue对象
JSValue *jsArr = context[@"arr"];
NSLog(@"JS Array: %@;    Length: %@", jsArr, jsArr[@"length"]);
    
//获取数组元素
jsArr[1] = @"blog"; //JSValue此时是一个数组类型
jsArr[7] = @7;
    
NSLog(@"JS Array: %@;    Length: %d", jsArr, [jsArr[@"length"] toInt32]);
    
//使用JSValue将js数组转换成oc数组
NSArray *nsArr = [jsArr toArray];
NSLog(@"NSArray: %@", nsArr);
```

####再看个麻烦点的，调用一个`js文件`中的js代码

- 首先在工程创建一个本地js源码文件

![](http://ww2.sinaimg.cn/large/74311666jw1f24jxhqzjej215u0tm44n.jpg)

其代码是一个简单的函数

```
var funcDemo = function(n) {
    return n * 100;
}
```

- 然后OC代码中通过JavaScriptCore.framework调用写好的本地js代码

```objc
#import "ViewController.h"

//导入JS库
#import <JavaScriptCore/JavaScriptCore.h>

@interface ViewController ()

@end

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self callJavaScriptDemo1];
}

- (void)callJavaScriptDemo1 {
    
    //1. 读取js代码内容
    NSString *path = [[NSBundle mainBundle]pathForResource:@"test1"ofType:@"js"];
    NSString *testScript = [NSString stringWithContentsOfFile:path encoding:NSUTF8StringEncoding error:nil];
    
    //2. 创建运行js代码的环境
    JSContext *context = [[JSContext alloc] init];
    
    //3. 执行js脚本代码
    [context evaluateScript:testScript];
    
    //4. 从js运行环境读取键值funcDemo对应的函数内容
    JSValue *function = context[@"funcDemo"];
    
    //5. 调用js函数
    JSValue *result = [function callWithArguments:@[@10]];
    
    //6. 使用JSValue 将JS数据 转换成 OC数据
    NSLog(@"%d", [result toInt32]);
}

@end
```

###反过来JS代码中调用一个OC方法

> 因为此时JS代码不是运行在浏览器中，自然不会有window、document、console这些类了，也就不会有那些UI的方法实现.

####JS调用OC方法有两种途径

- 通过JSContext注册NSBlock对象
- 通过OC对象实现`JSExport协议`

####通过JSContext注册NSBlock对象

- 修改本地test1.js的代码

```
var CallOCLog = function(n) {
	//调用在JSContext注册的log标识对应的Block
    log('ider', [7, 21], { hello:'world', js:100 });
}
```

- 在OC本地代码中，向JSContext注册NSBlock对象

```
#import "ViewController.h"

//导入JS库
#import <JavaScriptCore/JavaScriptCore.h>

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
//    [self callJavaScriptDemo1];
//    [self callJavaScriptDemo2];
    [self callJavaScriptDemo3];
}

- (void)callJavaScriptDemo3 {
    
    //1. js运行环境
    JSContext *context = [[JSContext alloc] init];
    
    //2. 给运行环境注册一个block，使用log为标识
    context[@"log"] = ^() {
        NSLog(@"+++++++Begin Log+++++++");
        
        //当前js运行环境中所有的参数值
        NSArray *args = [JSContext currentArguments];
        
        for (JSValue *jsVal in args) {
            NSLog(@"%@", jsVal);
        }
        
        JSValue *this = [JSContext currentThis];
        NSLog(@"this: %@",this);
        NSLog(@"-------End Log-------");
    };
    
    //3.1 直接从js运行环境获取log block之后执行
//    [context evaluateScript:@"log('ider', [7, 21], { hello:'world', js:100 });"];
    
    //3.2 在js文件中调用log block
    NSString *path = [[NSBundle mainBundle]pathForResource:@"test1"ofType:@"js"];
    NSString *testScript = [NSString stringWithContentsOfFile:path encoding:NSUTF8StringEncoding error:nil];
    
    //解释js代码
    [context evaluateScript:testScript];
    
    //获取js中的函数
    JSValue *function = context[@"CallOCLog"];
    
    //执行js函数
    [function callWithArguments:nil];
}

@end
```

上面是log block是一个变参，那么列举一个固定参数的例子

```objc
self.context = [[JSContext alloc] init];

self.context[@"add"] = ^(NSInteger a, NSInteger b) {
    NSLog(@"---%@", @(a + b));
};

[self.context evaluateScript:@"add(2,3)"];
```

- 如上向JSContext注册的是一个不接受回传值的block类型，那么就无法接受JSContext执行出现的异常。那么需要给JSContext实例设置异常回调Block来处理执行出现异常的情况:

```objc
JSContext *context = [[JSContext alloc] init];
    
context.exceptionHandler = ^(JSContext *con, JSValue *exception) {
    NSLog(@"%@", exception);
    con.exception = exception;
    
    //异常处理
    //.....
};
```

####通过OC对象实现`JSExport协议`

- 第一步，声明一个JSExport子协议，定义要暴露给JS代码的OC方法接口

```objc
#import "ViewController.h"

//导入JS库
#import <JavaScriptCore/JavaScriptCore.h>

//定义一个JSExport子协议，暴露OC方法定义
@protocol JSExportViewController <JSExport>

- (void)log:(id)value;

@end
```

- 要设置给JSContext的对象类实现JSExport子协议定义

```
@interface ViewController () <JSExportViewController>

@end

@implementation ViewController

- (void)callJavaScriptDemo4 {
    JSContext *context = [[JSContext alloc] init];
    
    //将实现了上面定义的协议的对象设置给JSContext
    context[@"ViewController"] = self;
    
    //执行在JSContext中的JS代码，既可以执行传入的对象的JSExport协议中定义的方法
    [context evaluateScript:@"ViewController.log('Hello JavaScript')"];
}

#pragma mark - JSExportViewController

- (void)log:(id)value {
    NSLog(@"value = %@", value);
}

@end
```

接收多个参数的例子

```
@protocol JSExportViewController <JSExport>

- (void)log:(id)value;

- (void)addX:(int)x withY:(int)y;

@end
```

```
- (void)callJavaScriptDemo4 {
    JSContext *context = [[JSContext alloc] init];
    
    context[@"ViewController"] = self;
    
    [context evaluateScript:@"ViewController.log('Hello JavaScript');"];
    
    [context evaluateScript:@"ViewController.addXWithY(1, 2);"];
}

#pragma mark - JSExportViewController

- (void)log:(id)value {
    NSLog(@"value = %@", value);
}

- (void)addX:(int)x withY:(int)y {
    NSLog(@"x + y = %d", x + y);
}
```

****

###当需要把block外部的JSValue传入block内时，一定要使用JS方法+形参接收

```


@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];

    //1. 创建context
    self.context = [[JSContext alloc] init];
    
    //2. 设置异常处理
    self.context.exceptionHandler = ^(JSContext *context, JSValue *exception) {
        [JSContext currentContext].exception = exception;
        NSLog(@"exception:%@",exception);
    };
   
   //3. 加载JS代码到context中
   [self.context evaluateScript:@" \
      function callback (){}; \
                              \
      function setObj(obj) { \
        this.obj = obj;      \
        obj.jsValue=callback;\
      }"
    ];

   //4. 调用JS方法，将
   [self.context[@"setObj"] callWithArguments:@[self.obj]];  
}

@end
```

###正式开始学习JSPatch源码，从一个最简单的demo代码开始一步一步断点...

```objc
#import "ViewController.h"
#import <JSPatch/JPEngine.h>

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

	//1. 实例化Engine 
	[JPEngine startEngine];
    
    //2. 直接执行js代码
    [JPEngine evaluateScript:@"\
     var alertView = require('UIAlertView').alloc().init();\
     alertView.setTitle('Alert');\
     alertView.setMessage('AlertView from js'); \
     alertView.addButtonWithTitle('OK');\
     alertView.show(); \
     "];
}

@end
```

代码执行结果，就是点击屏幕，弹出一个UIAlertView.

****

###[JPEngine startEngine]做的事情

```objc
+ (void)startEngine
{
    //1. 如果说没有JSContext这个类 >>> iOS7以下
    if (![JSContext class] || _context) {
        return;
    }
    
    //2. 创建一个js运行环境
    JSContext *context = [[JSContext alloc] init];
    
    //3. 注册在JSPatch.js调用JPEngine中定义的c函数的oc block
    
    //3.1 ，完成创建类
    context[@"_OC_defineClass"] = ^(NSString *classDeclaration, JSValue *instanceMethods, JSValue *classMethods) {
        id ret = defineClass(classDeclaration, instanceMethods, classMethods);
        return ret;
    };

    //3.2 完成给类类实现一个协议
    context[@"_OC_defineProtocol"] = ^(NSString *protocolDeclaration, JSValue *instProtocol, JSValue *clsProtocol) {
        return defineProtocol(protocolDeclaration, instProtocol,clsProtocol);
    };
    
    //3.3 js调用oc的对象方法
    context[@"_OC_callI"] = ^id(JSValue *obj, NSString *selectorName, JSValue *arguments, BOOL isSuper) {
        return callSelector(nil, selectorName, arguments, obj, isSuper);
    };
    
    //3.4 js调用oc的类方法
    context[@"_OC_callC"] = ^id(NSString *className, NSString *selectorName, JSValue *arguments) {
        return callSelector(className, selectorName, arguments, nil, NO);
    };
    
    //3.5 js对象 转 oc对象
    context[@"_OC_formatJSToOC"] = ^id(JSValue *obj) {
        return formatJSToOC(obj);
    };

    //3.5 oc对象 转 js对象
    context[@"_OC_formatOCToJS"] = ^id(JSValue *obj) {
        return formatOCToJS([obj toObject]);
    };
    
    //3.6 对js对象 weak
    context[@"__weak"] = ^id(JSValue *jsval) {
        id obj = formatJSToOC(jsval);
        return [[JSContext currentContext][@"_formatOCToJS"] callWithArguments:@[formatOCToJS([JPBoxing boxWeakObj:obj])]];
    };

    //3.7 对js对象 strong
    context[@"__strong"] = ^id(JSValue *jsval) {
        id obj = formatJSToOC(jsval);
        return [[JSContext currentContext][@"_formatOCToJS"] callWithArguments:@[formatOCToJS(obj)]];
    };

    __weak JSContext *weakCtx = context;
    context[@"dispatch_after"] = ^(double time, JSValue *func) {
        id currSelf = formatJSToOC(weakCtx[@"self"]);
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(time * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            JSValue *prevSelf = weakCtx[@"self"];
            weakCtx[@"self"] = formatOCToJS([JPBoxing boxWeakObj:currSelf]);
            [func callWithArguments:nil];
            weakCtx[@"self"] = prevSelf;
        });
    };
    
    //3.8 让js方法在 main queue dispatch async 执行
    context[@"dispatch_async_main"] = ^(JSValue *func) {
        id currSelf = formatJSToOC(weakCtx[@"self"]);
        dispatch_async(dispatch_get_main_queue(), ^{
            JSValue *prevSelf = weakCtx[@"self"];
            weakCtx[@"self"] = formatOCToJS([JPBoxing boxWeakObj:currSelf]);
            [func callWithArguments:nil];
            weakCtx[@"self"] = prevSelf;
        });
    };
    
    //3.9 让js方法在 main queue dispatch sync 执行
    context[@"dispatch_sync_main"] = ^(JSValue *func) {
        if ([NSThread currentThread].isMainThread) {
            [func callWithArguments:nil];
        } else {
            dispatch_sync(dispatch_get_main_queue(), ^{
                [func callWithArguments:nil];
            });
        }
    };
    
    //3.10 让js方法在 global queue dispatch async 执行
    context[@"dispatch_async_global_queue"] = ^(JSValue *func) {
        id currSelf = formatJSToOC(weakCtx[@"self"]);
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            JSValue *prevSelf = weakCtx[@"self"];
            weakCtx[@"self"] = formatOCToJS([JPBoxing boxWeakObj:currSelf]);
            [func callWithArguments:nil];
            weakCtx[@"self"] = prevSelf;
        });
    };
    
    
    //3.11 释放js创建的oc对象
    context[@"releaseTmpObj"] = ^void(JSValue *jsVal) {
        if ([[jsVal toObject] isKindOfClass:[NSDictionary class]]) {
            void *pointer =  [(JPBoxing *)([jsVal toObject][@"__obj"]) unboxPointer];
            id obj = *((__unsafe_unretained id *)pointer);
            @synchronized(_TMPMemoryPool) {
                [_TMPMemoryPool removeObjectForKey:[NSNumber numberWithInteger:[(NSObject*)obj hash]]];
            }
        }
    };

    //3.12 js调用oc方法进行打印
    context[@"_OC_log"] = ^() {
        NSArray *args = [JSContext currentArguments];
        for (JSValue *jsVal in args) {
            id obj = formatJSToOC(jsVal);
            NSLog(@"JSPatch.log: %@", obj == _nilObj ? nil : (obj == _nullObj ? [NSNull null]: obj));
        }
    };
    
    //3.13 将js捕捉到的异常交给oc方法处理
    context[@"_OC_catch"] = ^(JSValue *msg, JSValue *stack) {
        NSAssert(NO, @"js exception, \nmsg: %@, \nstack: \n %@", [msg toObject], [stack toObject]);
    };
    
    //4. 注册JSContext执行出现异常时的回调
    context.exceptionHandler = ^(JSContext *con, JSValue *exception) {
        NSLog(@"%@", exception);
        NSAssert(NO, @"js exception: %@", exception);
    };
    
    //5. 创建一个 OC中的null对象，转换成js的null对象，并设置到JSContext实例让js代码可以获取
    _nullObj = [[NSObject alloc] init];
    context[@"_OC_null"] = formatOCToJS(_nullObj);
    
    //6. 使用static便利保存JSContext实例
    _context = context;
    
    //7. OC中的 nil对象
    _nilObj = [[NSObject alloc] init];
    
    //8. 同步锁
    _JSMethodSignatureLock = [[NSLock alloc] init];
    _JSMethodForwardCallLock = [[NSRecursiveLock alloc] init];
    
    //9. 暂时不知道做什么的...
    _registeredStruct = [[NSMutableDictionary alloc] init];
    
    //10. 注册结束到内存警告通知
#if TARGET_OS_IPHONE
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(handleMemoryWarning)
                                                 name:UIApplicationDidReceiveMemoryWarningNotification
                                               object:nil];
#endif
    
    //11. 读取JSPatch.js，方便传入的js代码中使用JSPatch.js提供的require()函数
    NSString *path = [[NSBundle bundleForClass:[self class]] pathForResource:@"JSPatch" ofType:@"js"];
    NSAssert(path, @"can't find JSPatch.js");
    NSString *jsCore = [[NSString alloc] initWithData:[[NSFileManager defaultManager] contentsAtPath:path] encoding:NSUTF8StringEncoding];
    
    //12. 加载读取到的js文件数据到JSContext
    if ([_context respondsToSelector:@selector(evaluateScript:withSourceURL:)]) {
        [_context evaluateScript:jsCore withSourceURL:[NSURL URLWithString:@"JSPatch.js"]];
    } else {
        [_context evaluateScript:jsCore];
    }
}
```

- 最后读取了JSPatch.js文件中的所有js代码到JSContext运行环境，使得JSPatch.js中定义的一些函数，可以让给JSContext的js代码直接使用.

- 首先给JSContext运行换注册了很多js调用oc的block，而block内部最终调用static的c函数完整最终处理

```
js >>> JSContext >>> oc block >>> c函数
```

- 稍后一个个去看这些c函数

```
static NSDictionary *defineClass(NSString *classDeclaration, JSValue *instanceMethods, JSValue *classMethods);
```

```
static void defineProtocol(NSString *protocolDeclaration, JSValue *instProtocol, JSValue *clsProtocol)
```

```
static id callSelector(NSString *className, NSString *selectorName, JSValue *arguments, JSValue *instance, BOOL isSuper)
```

```
static id callSelector(NSString *className, NSString *selectorName, JSValue *arguments, JSValue *instance, BOOL isSuper)
```

```
static id formatJSToOC(JSValue *jsval)
```

```
static id formatOCToJS(id obj)
```

方法干什么的，如其名字就知道了，基本上使用大量的runtime api函数

```
#import <objc/runtime.h>
#import <objc/message.h>
```

这些函数，先知道干什么的，其具体实现后面再看吧.

****

###再看看 [JPEngine evaluateScript:]做的事情

```
[JPEngine evaluateScript:@"\
     var alertView = require('UIAlertView').alloc().init();\
     alertView.setTitle('Alert');\
     alertView.setMessage('AlertView from js'); \
     alertView.addButtonWithTitle('OK');\
     alertView.show(); \
     "];
```

```
+ (JSValue *)evaluateScript:(NSString *)script
{
    return [self evaluateScript:script withSourceURL:[NSURL URLWithString:@"main.js"]];
}
```

很好奇，这里会构造一个`main.js`，暂时不知道为什么，先放在这里吧先看后面.

后来看JSPatch使用文档时明白是做什么的了，这个main.js就是热修复oc原生代码时对应的js文件，该main.js文件描述了如何让JSPatch通过js规则映射到oc代码中，继而使用oc的tuntime替换掉之前的oc方法实现.

> http://jspatch.com/Docs/start

如下代码实例摘录自JSPatch使用文档，如上连接进去:

```
线上App程序中，引发crash的某各类的某个方法

@implementation XRTableViewController

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath
{
  NSString *content = self.dataSource[[indexPath row]]; //可能会超出数组范围导致crash
  XRViewController *controller = [[JPViewController alloc] initWithContent:content];
  [self.navigationController pushViewController:controller];
}

@end
```

通过给JSPatch上传一个名字叫main.js的js文件，告诉JSPatch如何改变上述原生oc方法的实现 >>> 对整个方法实现一起改变的，并不是针对某一行代码

```
//main.js
defineClass("XRTableViewController", {
  tableView_didSelectRowAtIndexPath: function(tableView, indexPath) {
    var row = indexPath.row()
    if (self.dataSource().length > row) {  //加上判断越界的逻辑
      var content = self.dataArr()[row];
      var controller = XRViewController.alloc().initWithContent(content);
      self.navigationController().pushViewController(controller);
    }
  }
})
```

> JSPatch平台注释: 注意在 JSPatch 平台的规范里，JS脚本的文件名必须是 main.js。接下来就看如何把这个 JS 脚本下发给所有用户。

那么JSPatch拿到了用户上传的main.js这个文件，然后下发到对应的App程序中，最终使用JavaScriptCore.framework执行main.js，而js又通过JavaScriptCore.framework反调oc，最终通过oc的runtime机制替换掉之前的oc方法实现.

> 并不是替换某一行代码，而是将整个方法通过main.js描述出来，全部替换掉.

然后对如上继续调用的 `evaluateScript: withSourceURL:` 方法源码分解如下下，便于理解

```
+ (JSValue *)evaluateScript:(NSString *)script withSourceURL:(NSURL *)resourceURL
{
    //1. 只支持iOS7+
    if (!script || ![JSContext class]) {
        NSAssert(script, @"script is nil");
        return nil;
    }
    
    //2. 正则式构建 >>> (?<!\\)\.\s*(\w+)\s*\(
    if (!_regex) {
        _regex = [NSRegularExpression regularExpressionWithPattern:_regexStr options:0 error:nil];
    }
    
    //3. 使用正则式处理 传入的 js代码 >>> 将 alloc()这样的函数调用 替换成 __c("alloc")()
    NSString *regexedScript = [_regex stringByReplacingMatchesInString:script
                                                               options:0
                                                                 range:NSMakeRange(0, script.length)
                                                          withTemplate:_replaceStr];
    
    //4. 将传入的要执行的js代码 ，使用try-cache包裹，捕捉到js异常后，调用注册给JSContext的 `_OC_catch` 对应的oc方法
    NSString *tryCatchExecutedScript = @"try{   \
    %@  \
    }catch(e) { \
    _OC_catch(e.message, e.stack)   \
    }";

    //5. 将正则式处理后的js代码，使用try-cache包裹，并附加oc的异常处理方法
    NSString *formatedScript = [NSString stringWithFormat:tryCatchExecutedScript, regexedScript];
    
    //6. 将处理后的js代码加载到JSContext执行
    @try {
        
        if ([_context respondsToSelector:@selector(evaluateScript:withSourceURL:)]) {
            return [_context evaluateScript:formatedScript withSourceURL:resourceURL];//iOS8.0之后引入的api
        } else {
            return [_context evaluateScript:formatedScript];
        }
    }
    @catch (NSException *exception) {
        NSAssert(NO, @"%@", exception);
    }
    
    return nil;
}
```

上面首先是使用正则表达式 `(?<!\\)\.\s*(\w+)\s*\(` 处理传入的js代码
	
```
require('UIAlertView').alloc().init();    >>>> require('UIAlertView').__c("alloc")().__c("init")()
```

```
alertView.setTitle('Alert');  >>>> alertView.__c("setTitle")('Alert');
```

其转换格式为:

```
对象.方法名(参数) >>> 对象.__c("方法名")(参数)
```

####就是将所有传入的js代码中使用 `js对象.方法(参数)`调用方法的代码，统一转换成`js对象.__c("方法名")(参数)`来完成最终js方法调用.

> 如下这段话摘录自JSPatch作者的wiki，解释这个 `__c()`函数的来源与作用:


- 主要为了解决的问题

	- 对于用户传入的js代码中，如`UIView().alloc().init()`这样的代码，js其实根本没办法进行处理
	- 第一、js文件中，不可能拿到Objective-C的数据类型UIView
	- 第二、即使可以模拟，但是仍然没有alloc()、init()这样的js方法
		- JS 对于调用没定义的属性/变量，只会马上抛出异常
	

- 解决如上问题的方法: 

	- 实现所有js类继承机制
	- 给js类的`根类`添加一个`__c(方法名)元函数`，来处理传入的js代码中的alloc()、init()这样的方法调用
	- 最终 `__c(方法名)元函数`调用OC中的方法，将`类名字符串`与`方法名字符串`使用runtime反射执行

引用作者的话:

> 当时继续苦苦寻找解决方案，若按 JS 语法，这是唯一的方法，但若不按 JS 语法呢？突然脑洞开了下，CoffieScript/JSX 都可以用 JS 实现一个解释器实现自己的语法，我也可以通过类似的方式做到，再进一步想到其实我想要的效果很简单，就是调用一个不存在方法时，能转发到一个指定函数去执行，就能解决一切问题了，这其实可以用简单的字符串替换，把 JS 脚本里的方法调用都替换掉。最后的解决方案是，在 OC 执行 JS 脚本前，通过正则把所有方法调用都改成调用 __c() 函数，再执行这个 JS 脚本，做到了类似 OC/Lua/Ruby 等的消息转发机制：

```
UIView.alloc().init()
->
UIView.__c('alloc')().__c('init')()
```

给 JS 对象基类 Object 的 prototype 加上 __c 成员，这样所有对象都可以调用到 __c，根据当前对象类型判断进行不同操作：

```
Object.prototype.__c = function(methodName) {
  if (!this.__obj && !this.__clsName) return this[methodName].bind(this);
  var self = this
  return function(){
    var args = Array.prototype.slice.call(arguments)
    return _methodFunc(self.__obj, self.__clsName, methodName, args, self.__isSuper)
  }
}
```

_methodFunc() 就是把相关信息传给OC，OC用 Runtime 接口调用相应方法，返回结果值，这个调用就结束了。

这样做不用去 OC 遍历对象方法，不用在 JS 对象保存这些方法，内存消耗直降 99%，这一步是做这个项目最爽的时候，用一个非常简单的方法解决了严重的问题，替换之前又复杂效果又差的实现。

OK，到此可以看到一个传入的js代码被执行.

***


###大概看下来对JSPatch的感觉

- 通过`JavaScriptCore.framework`来让 OC代码 与 JS代码 之间的互相调用
- 然后通过解析JS代码中的固定语法格式，截取出用到的`类名、方法名、参数值`
- 然后再通过JavaScriptCore交给OC方法来处理
	- 根据类名 NSClassFromString() 反射出类名
	- 根据方法字符串 NSSelectorFromString() 得到方法SEL值
	- 再交换OC类的方法实现IMP，达到改变OC类方法实现的效果
	- 继而完成在线修复bug的效果

- 个人理解JSPatch的功能结构图

![](http://i13.tietuku.cn/a617bf2621e04574.png)

- 再文字小结下JSPatch的工作原理（之前的图示是没有弄懂main.js的作用时候画的）

	- 当需要热修复App中原生oc某一个方法时
	- 编写一个名字叫main.js的文件
	- 该main.js文件，使用js语法描述，就是要告诉JSPatch如何执行替换
		- 替换哪一个类 `objc_class`
		- 类中哪一个方法 `objc_method`
		- 以及方法将要替换成的实现 `objc_method->IMP`
	- 这些描述规则，必须通过`JSPatch规定的js语法描述规则`来完成
	- 编写后，上传到JSPatch，由JSPatch下发到App，保存到沙盒下
	- 当App程序重启时，由JSPatch执行沙盒下的main.js
	- JSPatch内部调用`JavaScriptCore.framework`来执行如下两个js文件
		- 用户上传的main.js（替换原生oc方法的规则）
		- 自带的JSPatch.js（提供了很多基础js代码）
	- 最终JSPatch.js解析出js描述规则后，又调用`JavaScriptCore.framework`，调用JSPatch提供的一系列c函数完整最终将目标oc方法替换的工作

	- 所以这一切都是通过`运行时`改变了方法实现IMP，App中的代码仍然还是错误代码，如果需要彻底修复还是需要重新发版.

> iOS7引入的 JavsScriptCore.framework起了很大的作用，可以让OC与JS很方便的互相调用，一切的热修复都是基于此完成.

接下来该看看JSPatch.js的代码了，如何实现`__c()元函数` ... 

