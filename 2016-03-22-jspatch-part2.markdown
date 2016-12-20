---
layout: post
title: "JSPatch Part2"
date: 2016-03-22 13:14:54 +0800
comments: true
categories: 
---

###主要看下JSPatch.js

由于js只是会点皮毛，没有正儿八经的用过，还是从最简单的点记录...

***

###首先看下js中的内置对象，也就是在当前作用域直接可以使用的去全局对象

![](http://i4.tietuku.cn/964d162028a84ee5.png)

- 内置的 arguments对象

```
function f(a, b, c){
  debug("==============================")
  debug("参数总长度 = " + arguments.length); 
  debug("第一个参数 = " + arguments[0])  
  debug("第二个参数 = " + arguments[1])  
  debug("第三个参数 = " + arguments[2])
}

f(1)
f(1, 2)
f(1, 2, 3)
```

输出如下

```
--> ==============================
--> 参数总长度 = 1
--> 第一个参数 = 1
--> 第二个参数 = undefined
--> 第三个参数 = undefined
--> ==============================
--> 参数总长度 = 2
--> 第一个参数 = 1
--> 第二个参数 = 2
--> 第三个参数 = undefined
--> ==============================
--> 参数总长度 = 3
--> 第一个参数 = 1
--> 第二个参数 = 2
--> 第三个参数 = 3
```

可以看出来arguments对象，就是存在于某个方法内部的内置对象，保存所有传递过来的参数.

***

###然后js代码中通常为了解决命名冲突，而使用一个可以直接执行的匿名闭包来作为一个区域

```
var global = {};

//直接执行的匿名闭包1
(function(){
	//局部区域一
	global.A = {}
	global.A.name = "name1"

	debug(global.A.name);
})();

//直接执行的匿名闭包2
(function(){
	//局部区域二
	global.B = {}
	global.B.name = "name2"

	debug(global.B.name);
})();
```

```
var global = {};


(function(){

	global.A = {}
	global.A.name = "name1"

	global.A.test = function (){
		debug('test A');
	}
})();

(function(){

	global.B = {}
	global.B.name = "name1"

	global.B.test = function (){
		debug('test B');
	}
})();

global.A.test()
global.B.test()
```

直接运行js文件，输出如下

```
--> name1
--> name2
[Finished in 0.1s]
```

关于上面写法的分解

```
//只是定义闭包，没有执行
var block = (function(){
	debug('使用()，执行闭包')
});//注意: 这里没有()

//执行闭包
block();
```

```
//定义闭包+执行闭包
(function(){
	debug('使用()，执行闭包')
})();
```

****

###一个js文件中的this

```
this.name = "haha";
this.age = 19;
this.func = function() {
	debug('hahahah');
};

debug(this);
debug(this.name);
debug(this.age);
debug(this.func);
```

输出

```
--> haha
--> 19
--> function () {
	debug('hahahah');
```

- 说明this指向的是一个`全局对象`，在当前整个js文件的作用域内
- this还是一个类似Map的容器，使用key-value

***

###js中 Object对象的prototype属性

> 简而言之，prototype就是“一个给类的对象添加方法的方法”，使用prototype属性，可以给类动态地添加方法，以便在JavaScript中实现“继承”的效果.

eg1、给Number所有对象，添加一个add方法

```
Number.prototype.add = function(num){
								return(this+num);
							} 
```

eg2、给Array类对象增加push方法

```
Array.prototype.push = function(new_element){ 
                this[this.length]=new_element; 
                return this.length; 
              };

var arr = new Array();

arr.push('hahah1');
arr.push('hahah2');
arr.push('hahah3');

debug(arr)
```

输出

```
--> hahah1,hahah2,hahah3
```

***

###快速将一个Map对象转换成Array

```
var arr = Array.prototype.slice.call(Map对象);//该Map对象必须要有length属性值
```

eg1、 

```
var dic = {}

dic.length = 3;
dic[0] = "value1";
dic[1] = "value2";
dic[2] = "value3";

var args = Array.prototype.slice.call(dic);
debug(args)
```

输出

```
--> value1,value2,value3
```

eg2、

```
var dic = {
  length : 3,
  0 : "value4",
  1 : "value5",
  2 : "value6",
}

var args = Array.prototype.slice.call(dic);
debug(args)
```

输出


```
--> value4,value5,value6
```

- Array的splice()函数使用

```
var arr = new Array()
arr[0] = "1"
arr[1] = "2"
arr[2] = "3"
debug(arr)

arr.splice(1, 0,"4", "5", "6")//第二个参数写0，表示插入个数岁参数个数而定
debug(arr)
```

输出

```
--> 1,2,3
--> 1,4,5,6,2,3
```

在列举个带删除的写法

```
var lang = ["php","java","javascript"]; 
//删除 
var removed = lang.splice(1,1); 
alert(lang); //php,javascript 
alert(removed); //java ,返回删除的项 
//插入 
var insert = lang.splice(0,0,"asp"); //从第0个位置开始插入 
alert(insert); //返回空数组 
alert(lang); //asp,php,javascript 
//替换 
var replace = lang.splice(1,1,"c#","ruby"); //删除一项，插入两项 
alert(lang); //asp,c#,ruby 
```


***

###然后看下JSPatch.js的大体结构

eg、 var gloabl = {}; 时

```
//global指向的是一个新Map
var global = {};

(function(){

	global.test = function (){
		debug('test B');
	}

})();

//调用test方法时，也需要指针新的Map中的test方法
global.test()
```

eg、 var gloabl = this; 时

```
//global指向的是 全局Map对象
var global = this;

(function(){

	global.test = function (){
		debug('test B');
	}

})();

//调用test方法时，直接写test()即可，默认从全局Map查找
test()
```

JSPatch.js选用的是下面`var gloabl = this;`的情况，因为不存在多个js区域在一个js文件中的情况，直接使用this指向的`全局Map对象`

****

###了解完JSPatch.js文件结构之后，开始正式看其内部具体实现逻辑，一个一个函数来看吧...

> 学习来源: https://segmentfault.com/a/1190000003648832

```
js 文件开头使用global代替this，当前js全局作用域
var global = this;
```

***

###JSPatch.js闭包内部的`require函数`，在JS全局作用域上创建一个与类名相同的`变量`，变量指向一个对象，其中属性`__clsName`记录了对应的类名字符串

```
//全局Map对象的函数，直接可以使用require(clsName)调用

/**
 *	接收一个数组 >>> require('UIAlertView', 'UIButton', 'UITextfiled')
 */
global.require = function(clsNames) {
	
	//保存数组`最后`一个类名
	var lastRequire
	
	//依次取出每一个类名，保存到global map对象
	clsNames.split(',').forEach(function(clsName) {
		//对每一个类名交给 _require()函数处理
	   lastRequire = _require(clsName.trim())
	})
	
	//返回最后的一个类名
	return lastRequire
}
```

```
//将还不存在于global（this）中的类名，使用一个Map对象保存下来
//key: __clsName
//value: 传入的 clsName

var _require = function(clsName) {
    if (!global[clsName]) {
    
      //在当前全局Map对象中，保存一个类名的变量，直接可以在js全局代码区域使用这个变量来代替类名字符串
      global[clsName] = {
        					__clsName: clsName
      					}
    } 
    return global[clsName]
}
```

eg、UIAlertView

```
global['UIAlertView'] = {
							__clsName: 'UIAlertView',
						};
```

那么在js全局作用于就可以使用这个`全局变量UIAlertView`了.

```
//1. 在全局js作用域创建一个UIView变量，其属性 _clsName记录了类名字符串 UIView
require('UIView')

//2. 下面js区域，直接可以使用 全局变量UIView
var view = UIView.alloc().init()
view.setBackgroundColor(require('UIColor').grayColor())
view.setAlpha(0.5)
```

***

###完成js中代码中的`UIView.alloc()`方法调用

- OC中当调用一个`不存在`的方法SEL时，会进入消息转发过程，从而可以动态添加一个方法实现

- 但是JS没有消息转发机制，当执行一个`不存在`的方法时，直接抛出一个exception

```
Exception: ReferenceError: Can't find variable: noFunc
```

- 解决方式是: 简单版的消息转发 

	- 第一步、将js代码中的方法调用的代码，进行转成成能在js中运行的方式
	- 第二步、js最终通过javascriptcore.framework交给oc处理调用

- 将js代码中的方法调用的代码，进行转成成能在js中运行的方式

```
//用户编写的代码（非js能识别的代码，因为UIView是一个全局变量，并没有alloc()方法的）
UIView.alloc().init()

转换成->

//最终js能够识别的方法调用
UIView.__c('alloc')().__c('init')()
```

那么当JSPatch执行如下js代码时:

```
[JPEngine startEngine];

[JPEngine evaluateScript:@"\
 require('UIAlertView, UIButton, UITextfiled'); \
 var alertView = UIAlertView.alloc().init();\
 alertView.setTitle('Alert');\
 alertView.setMessage('AlertView from js'); \
 alertView.addButtonWithTitle('OK');\
 alertView.show(); \
 "];
```

首先 `[JPEngine evaluateScript:]`方法对上面`var alertView = UIAlertView.alloc().init();`这样在js对象中不存在方法的调用字符串进行修改

```objc
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
    
    //3. 使用正则式处理 传入 js代码 >>> 将 alloc()这样的函数调用 替换成 __c("alloc")()
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
    
    //6. 将处理后的js代码加载到JSContext
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

如上代码主要做两件事:

- 将`.alloc()` >>> `.__c('alloc')` 

```
UIAlertView.alloc().init() >>> var alertView = UIAlertView.__c("alloc")().__c("init")()
```

转换成如下对于 `__c()`函数的调用

```
alertView.setTitle('Alert') >>> alertView.__c("setTitle")('Alert')
```

- 然后对传入的原始js使用`try-catch`包裹

> 这个 `UIAlertView` js全局对象，本身是不存在`alloc()`方法的，最终为了让OC中的UIAlertView执行alloc方法的思路:

- 第一步、将`UIAlertView.alloc()`这个在字符串，进行字符串替换成使用`__c(参数)()`函数的调用的字符串

- 第二步、因为后面的js代码给全局所有的js对象，都添加了一个`__c`的property属性

> 所以最终是通过 `__c` 这个属性完成js方法调用

下面看是如何添加这个 __c属性的...

****

###完成对js对象任意方法调用的属性 `__c`

- js语法格式，将属性添加到对象，或修改现有属性的特性

```
object 必需。  要在其上添加或修改属性的对象。  这可能是一个本机 JavaScript 对象（即用户定义的对象或内置对象）或 DOM 对象。  
propertyname 必需。  一个包含属性名称的字符串。  
descriptor 必需。  属性描述符。  它可以针对数据属性或访问器属性。

Object.defineProperty(object, propertyname, descriptor)
```

eg、

```
Object.defineProperty(obj, "newDataProperty", {
	
	//属性的默认值
    value: 101,
    
    //下面三个数属性修饰	
    writable: true,
    enumerable: true,
    configurable: true
});
```

####简单版实现给全局所有js对象添加 __c属性，而__c属性又是一个方法

eg1、

```
//1. 给所有js对象添加一个属性方法: __(参数)
Object.defineProperty(Object.prototype, "__c", {value: function(methodName) {
	debug('methodName = ' + methodName)
}});

//一个js对象
var UIView = {}

//调用js对象添加的 __c(参数)方法
UIView.__c('alloc')
```

eg2、

```
//1. 给所有js对象添加一个属性方法: __(参数)
Object.defineProperty(Object.prototype, "__c", {value: function(methodName) {
	return function() {
		debug('Hello world~')	
	}
}});

//一个js对象
var UIView = {}

//调用js对象添加的 __c(参数)方法
UIView.__c('alloc')()
```

####bind()函数，多次执行方法，将前一次方法执行结果值作为第二次执行方法的参数

```
function add(arg1, arg2, arg3, arg4) { 
	return arg1 + ' ' + arg2 + ' ' + arg3 + ' ' + arg4; 
} 
var addMore = add.bind({}, 'a', 'b'); 
debug(addMore('c', 'd')); 
```

输出

```
a b c d
```

####给全局js对象添加 __c(methodName) 这个属性方法，获取在OC中要执行的方法SEL值

> js类方法调用 与 js对象方法调用 都由 js对象的 `__c()函数`处理

```
//全局js区域创建一个全局变量UIView
require('UIAlertView')

//类方法调用
var allociate = UIAlertView.alloc(); >>> UIAlertView.__c('alloc')()

//对象方法调用
var alert = allociate.init() >>> allociate.__c('init')()
alert.setTitle('Alert') >>> alert.__c('setTitle')('Alert')
```

```
Object.defineProperty(Object.prototype, "__c", {value: function(methodName) {

    //依次判断调用__c()函数的 当前对象 的数据类型

    //对象是bool类型
    //使用eg、
    //var obj = false;
    //debug(obj.__c('alloc')())
    if (this instanceof Boolean) {
      //返回的是一个function，需要()执行，否则得到的是方法代码的字符串
      return function() {
        return false
      }
    }
    
    //js对象至少应该具备右边两个属性中的一个: __obj 与 __clsName
    //默认情况下，由gloabl.require()函数，给全局js对象添加 __clsName属性，保存代表的类名
    //__obj属性，是后面返回值是OC对象时，由OC中加上去的，@{@"__obj": obj}
    if (!this.__obj && !this.__clsName) {
      if (!this[methodName]) {
        throw new Error(this + '.' + methodName + ' is undefined')
      }
      return this[methodName].bind(this);
    }

    //super方法执行（对象方法 or 类方法）
    var self = this
    if (methodName == 'super') {
      return function() {
      
      	//对象的super方法
        if (self.__obj) {
        
        	//取出 Map对象{__obj:OC对象}中的 OC对象，给其添加所属类声明
          self.__obj.__clsDeclaration = self.__clsDeclaration;
        }
        
        //对象 or 类的 super方法，返回一个Map对象
        return {
        				__obj: self.__obj, 
			  	        __clsName: self.__clsName, 					
			  	        __isSuper: 1
				}
      }
    }

    //在js中使用 performSelector 来执行方法
    if (methodName.indexOf('performSelector') > -1) {
      if (methodName == 'performSelector') {
        return function(){
          var args = Array.prototype.slice.call(arguments)
          return _methodFunc(self.__obj, self.__clsName, args[0], args.splice(1), self.__isSuper, true)
        }
      } else if (methodName == 'performSelectorInOC') {
        return function(){
          var args = Array.prototype.slice.call(arguments)
          return {__isPerformInOC:1, obj:self.__obj, clsName:self.__clsName, sel: args[0], args: args[1], cb: args[2]}
        }
      }
    }

	//处理其他类型的方法调用
	//[类 SEL] 或 [对象 SEL]
    return function(){
      var args = Array.prototype.slice.call(arguments)

      //调用 _methodFunc()函数完成除开supper、performSelector形式的方法调用
      return _methodFunc(self.__obj, self.__clsName, methodName, args, self.__isSuper)
    }
  }, 
  configurable:false, 
  enumerable: false
})
```

- 返回的都是一个function

```
js对象.__c('alloc')() 这样写才会得到正确的返回值
```

- 三种类型方法调用
	- super方法
		- 对象方法
		- 类方法
	- 对象方法
	- 类方法

- 两种发送消息方式
	- performSelector
	- [Target SEL]


- 最终通过 `_methodFunc()`js函数，完成调用OC block的工作.



####`_methodFunc()`这个js方法只在js文件内部使用，为了把相关信息传给OC，再让OC用 Runtime 接口调用相应方法.

```
/**
   *  instance: 对象
   *  clsName: 类名
   *  methodName: 方法名
   *  args: 参数列表
   *  isSuper: 是否调用super父类的方法
   *  isPerformSelector: 
	   */
var _methodFunc = function(instance, clsName, methodName, args, isSuper, isPerformSelector) {
	
	var selectorName = methodName
	
	if (!isPerformSelector) {
	
	  //不是 performSelector方式的方法调用流程
	  
	  //处理得到OC中的方法SEL值
	  methodName = methodName.replace(/__/g, "-")
	  selectorName = methodName.replace(/_/g, ":").replace(/-/g, "_")
	  var marchArr = selectorName.match(/:/g)
	  var numOfArgs = marchArr ? marchArr.length : 0
	
	  if (args.length > numOfArgs) {
	    selectorName += ":"
	  }
	}
	
	//得到调用OC方法后的返回值
	//如果是获取一个OC对象，那么这个ret = {} 首先是一个Map对象
	//然后里面的内容 ret = {"__obj" : OC对象}
	var ret = instance ? _OC_callI(instance, selectorName, args, isSuper):
	                     _OC_callC(clsName, selectorName, args)
	
	//获取OC方法执行完毕的返回值，并转化成JS对象
	return _formatOCToJS(ret)
}
```

注意如上调用OC方法得到ret，结构格式: `ret = {"__obj" : OC对象}`

####然后到了JPEngine实例化时注册给js代码使用的回调block

```
@implementation JPEngine

+ (void)startEngine
{
	//1. 如果说没有JSContext这个类 >>> iOS7以下
    if (![JSContext class] || _context) {
        return;
    }
    
    //2. 创建一个js运行环境
    JSContext *context = [[JSContext alloc] init];
    
    //3. 注册在JSPatch.js调用JPEngine中定义的c函数的oc block
    
    ....
    
    //3.3 js调用oc的对象方法
    context[@"_OC_callI"] = ^id(JSValue *obj, NSString *selectorName, JSValue *arguments, BOOL isSuper) {
        return callSelector(nil, selectorName, arguments, obj, isSuper);
    };
    
    //3.4 js调用oc的类方法
    context[@"_OC_callC"] = ^id(NSString *className, NSString *selectorName, JSValue *arguments) {
        return callSelector(className, selectorName, arguments, nil, NO);
    };
    
    ....
    
}   
```

`_OC_callI` 与 `_OC_callC` 统一由 `callSelector()`这个static c函数处理，接下来看这个callSelector()函数了，这个函数n多...


####JPEngine.m中的callSelector()函数

```
/**
 *  完成OC中的方法调用
 *
 *  @param className    类名
 *  @param selectorName 方法SEL值
 *  @param arguments    方法执行参数
 *  @param instance     对象（js对象中的变量，如: var UIAlertView = { __clsName : 'UIAlertView'}）
 *  @param isSuper      是否调用父类方法
 *
 *  @return 方法执行后的结果值，返回给js代码中
 */
static id callSelector(NSString *className,
                       NSString *selectorName,
                       JSValue *arguments,
                       JSValue *instance,
                       BOOL isSuper);
```

> 核心是创建NSInvocation手动完成消息发送

####NSInvocation使用模板

```objc
//1. 获取要执行方法的签名
NSMethodSignature  *signature = [类 instanceMethodSignatureForSelector:方法SEL];

//2. 未找到方法实现
if (!signature) {
	//...处理...
	return;
}

//2. 创建NSInvocation
NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:signature];

//3. 设置消息接收者
invocation.target = ....;

//4. 消息中执行方法的SEL
invocation.selector = 方法sel;

//5. 方法执行参数设置，需要考虑: 外界传递参数个数 != 方法的参数个数
//方法参数的个数，除去 self与_cmd两个系统参数
NSUInteger argsCount = signature.numberOfArguments - 2;

//外界传递的参数个数
NSUInteger arrCount = objects.count;

//取前面两个参数个数的最小值
NSUInteger count = MIN(argsCount, arrCount);

//对每个参数进行NSNull处理，并设置给NSInvocation
for (int i = 0; i < count; i++) {
    id obj = objects[i];
    if ([obj isKindOfClass:[NSNull class]]) {
        obj = nil;
    }
    
	[invocation setArgument:&obj atIndex:i + 2];//除开self、_cmd
}

//6. 返回值的处理
const char *returnType = [methodSignature methodReturnType];
id returnValue;
    
if (strncmp(returnType, "v", 1) != 0) {
    //方法返回值不是void
    
    if (strncmp(returnType, "@", 1) == 0) {
        //返回值类型是 Foundation类型（NSObject*）
        
        //从NSInvocation获取方法执行后的返回值
        void *result;
        [invocation getReturnValue:&result];
        
        //返回值方式一、将Core Foundation的对象转换为Objective-C对象，同时将内存管理权交给ARC
        returnValue = (__bridge_transfer id)result;
        
        //返回值方式二、只是类型转换成Objective-C对象
        returnValue = (__bridge id)result;
        
	} else {
		//基本数据类型
	    //SEL
	    //struct
	    //char* 与 ^其他类型
	    //Class
	    
	    //先从invocation获取的返回值，转换成对应的类型的值
	    //再分别按照如上类型，调用对应的方法
	    returnValue = .....;
	    
		return returnValue;
	}
}

//返回值类型void，返回nil
return nil;
```

- callSelector()函数比较长，涉及的东西也比较多，需要一定是见看懂

- js中通过 `_methodFunc` 方法， 获取到 OC中返回给js方法的 OC对象返回值.

****

####图示总结下上面叙述的，JSPatch通过js字符串调用到JPEngine中的oc代码的过程

![](http://ww3.sinaimg.cn/large/74311666jw1f2dv4cqh8oj21k80x8wl5.jpg)

***

###callSelector函数的实现细节一、overrideMethod函数，使用JSValue中将要替换的方法实现，来替换objc类中的方法实现.

源码注释

```objc
static void overrideMethod(Class cls,                   //被替换的类
                           NSString *selectorName,      //被替换实现的SEL
                           JSValue *function,           //在js代码中定义的将要替换成的新的实现
                           BOOL isClassMethod,          //是否类方法，就需要找到MetaClass
                           const char *typeDescription) //被替换的实现方法的编码
{
    //1. 要重写的方法的 SEL
    SEL selector = NSSelectorFromString(selectorName);
    
    //2. 获取重写方法的 具体实现函数的 格式编码
    if (!typeDescription) {
        Method method = class_getInstanceMethod(cls, selector);
        typeDescription = (char *)method_getTypeEncoding(method);
    }
    
    //3. 获取到 Class 中 被重写SEL对应的 原始c函数实现IMP
    IMP originalImp = class_respondsToSelector(cls, selector) ? class_getMethodImplementation(cls, selector) : NULL;

    //4. 预先准备进入消息转发处理的系统函数实现IMP
    //注意: _objc_msgForward函数，当cpu架构不是 arm64时，处理返回值非特殊struct类型可能crash
    IMP msgForwardIMP = _objc_msgForward;
    
    //5. 针对 `非arm64` 架构时，消息转发使用 _objc_msgForward_stret系统函数
    #if !defined(__arm64__)
        if (typeDescription[0] == '{') {
            //In some cases that returns struct, we should use the '_stret' API:
            //http://sealiesoftware.com/blog/archive/2008/10/30/objc_explain_objc_msgSend_stret.html
            //NSMethodSignature knows the detail but has no API to return, we can only get the info from debugDescription.
            NSMethodSignature *methodSignature = [NSMethodSignature signatureWithObjCTypes:typeDescription];
            if ([methodSignature.debugDescription rangeOfString:@"is special struct return? YES"].location != NSNotFound) {
                msgForwardIMP = (IMP)_objc_msgForward_stret;
            }
        }
    #endif

    //6. 让被替换的方法调用时，直接进入`系统消息转发`系统函数（_objc_msgForward函数 或 _objc_msgForward_stret函数）
    //>>> eg、[ViewController对象 viewDidLoad] >>>> [ViewController对象 forwardInvocation:]
    class_replaceMethod(cls,                //当前Class
                        selector,           //要替换方法实现的方法SEL
                        msgForwardIMP,      //替换成消息转发的系统函数实现IMP
                        typeDescription);   //原始实现函数的编码描述

    //7. 继续替换掉系统的forwardInvocation:的实现IMP 成 JPForwardInvocation这个c函数
    //>>> 就可以得到NSInvocation对象，继而获得所有的方法调用参数
    //>>> 获取完参数后，让系统的消息转发函数继续执行
    //>>> eg、[ViewController对象 forwardInvocation:] >>>> JPForwardInvocation这个c函数
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wundeclared-selector"

    if (class_getMethodImplementation(cls, @selector(forwardInvocation:)) != (IMP)JPForwardInvocation) {
        
        //将Class中原来forwardInvocation:实现，替换成 JPForwardInvocation函数实现
        //class_replaceMethod()返回的IMP，是替换前的IMP
        IMP originalForwardImp = class_replaceMethod(cls,
                                                     @selector(forwardInvocation:),
                                                     (IMP)JPForwardInvocation,
                                                     "v@:@");
        
        //给Class添加一个`新的SEL`，并指向原始的forwardInvocation:实现IMP
        //为了进行hook后，继续让原来的函数实现执行
        //>>>> 当执行[ViewController对象 ORIGforwardInvocation:]就会回到系统的 forwardInvocation:实现中
        class_addMethod(cls,
                        @selector(ORIGforwardInvocation:),
                        originalForwardImp,
                        "v@:@");
    }
#pragma clang diagnostic pop

    //8. 给Class添加一个ORIGvidDidLoad方法，用于指向 viewDidLoad 原始的实现IMP
    if (class_respondsToSelector(cls, selector)) {
        
        //ORIGviewDidLoad
        NSString *originalSelectorName = [NSString stringWithFormat:@"ORIG%@", selectorName];
        SEL originalSelector = NSSelectorFromString(originalSelectorName);
        
        //给Class添加一个objc_method
        //>>> SEL : ORIGviewDidLoad
        //>>> IMP : 被重写方法的原始实现IMP
        if(!class_respondsToSelector(cls, originalSelector)) {//没有实现ORIGviewDidLoad才去添加
            class_addMethod(cls,
                            originalSelector,       //ORIGviewDidLoad
                            originalImp,            //ORIGviewDidLoad这个SEL，指向了被替换的原始方法SEL的具体实现IMP
                            typeDescription);       //viewDidLoad实现函数的编码
        }
    }
    
    //9. 构造替换后新实现的SEL >>>  _JPvidDidLoad
    NSString *JPSelectorName = [NSString stringWithFormat:@"_JP%@", selectorName];
    SEL JPSelector = NSSelectorFromString(JPSelectorName);
    
    //10. 单例字典记录 _JPvidDidLoad SEL 对应的 js传过来的要被替换目标方法的实现
    _initJPOverideMethods(cls);
    _JSOverideMethods[cls][JPSelectorName] = function;
    
    //11. 给Class添加一个_JPvidDidLoad SEL >>> 指向系统消息转发函数IMP
    //TODO: 这里不明白为什么添加 _JPviewDidLoad 这个sel对应的函数实现types是原来viewDidLoad实现的types
    class_addMethod(cls,
                    JPSelector,
                    msgForwardIMP,
                    typeDescription);
}
```

####写一个替换ViewController的方法成forwardInvocation:的例子，来说明可以直接获取NSInvocation消息

```objc
@implementation ViewController

+ (void)load {
    
    Method m = class_getInstanceMethod([ViewController class], @selector(viewWillAppear:));
    const char *types = method_getTypeEncoding(m);
    
    class_replaceMethod([ViewController class],
                        @selector(viewWillAppear:),
                        (IMP)_objc_msgForward,
                        types);
}

- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    
    NSLog(@"%s", __func__);
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    NSLog(@"%s, anInvocation = %@", __func__, [anInvocation description]);
}

@end
```

输出如下

```
2016-03-31 14:04:00.127 JSPatchDeme[33696:185546] -[ViewController forwardInvocation:], anInvocation = <NSInvocation: 0x7fbe3044e6f0, {
    methodSignature = "<NSMethodSignature: 0x7fbe30415de0>";
    sel = "viewWillAppear:";
    target = "<ViewController: 0x7fbe30448180>";
}>
```

可以看到


####小结overrideMethod函数做的事情:

- 将要替换实现的SEL对应的IMP，替换成 `forwardInvocation:` 实现IMP

```
eg、 viewDidLoad SEL >>>> forwardInvocation: 实现IMP
```

- 继续将 `forwardInvocation:` 实现IMP，替换成 JPEngine.m中的编写的c函数 `JPForwardInvocation`

```
eg、forwardInvocation: SEL >>>> JPForwardInvocation这个c函数实现
```


替换成的JSPatch提供的消息转发c函数原型定义如下

```
static void JPForwardInvocation(__unsafe_unretained id assignSlf, SEL selector, NSInvocation *invocation);
```

来到该函数中，主要目的是为了，获取目标消息SEL执行时，传递进来的所有的`参数值`.

- 因为替换了系统的 `forwardInvocation:` 实现IMP，所以就造成所有的消息转发都会走到 JPEngine提供的c函数`JPForwardInvocation `中而不会进行消息转发。所以还需要使用`一个新的SEL`指向系统`forwardInvocation:` 实现IMP，让其他进入消息转发情况继续执行。

```
eg、ORIGforwardInvocation: >>>> 系统的 forwardInvocation:实现IMP
```

- 一个NSInvocation对象包含:
	- Receiver
	- SEL
	- SEL对应的实现函数的编码
	- 所有的方法执行参数值

- 系统消息转发函数有两种，区分非arm64与arm64下，消息执行后返回值是一些特殊的struct时

在 `非arm64`架构下 下，消息执行后返回值是一些特殊的struct时，会产生crash

```
id _objc_msgForward(id receiver, SEL sel, ...) 
```

解决上述问题替换另外一个系统进行消息转发的c实现函数

```
void _objc_msgForward_stret(id receiver, SEL sel, ...) 
```

代码片段

```objc
//arm64架构下，消息转发c函数
IMP msgForwardIMP = _objc_msgForward;

//非arm64架构下，消息转发c函数
#if !defined(__arm64__)
if (typeDescription[0] == '{') {
    //In some cases that returns struct, we should use the '_stret' API:
    //http://sealiesoftware.com/blog/archive/2008/10/30/objc_explain_objc_msgSend_stret.html
    //NSMethodSignature knows the detail but has no API to return, we can only get the info from debugDescription.
    NSMethodSignature *methodSignature = [NSMethodSignature signatureWithObjCTypes:typeDescription];
    if ([methodSignature.debugDescription rangeOfString:@"is special struct return? YES"].location != NSNotFound) {
        msgForwardIMP = (IMP)_objc_msgForward_stret;
    }
}
#endif
```

- 记录被替换实现的SEL的原始实现IMP（如: viewDidLoad的原来实现IMP)

```
eg、ORIGvidDidLoad 指向 viewDidLoad的实现IMP
```

- 获取完被替换方法实现的所有参数，现在就是要替换原始方法的实现IMP了，但是并没有直接指向viewDidLoad实现IMP，而是指向`系统消息转发函数的IMP`


```
eg、_JPviewDidLoad SEL >>>> _objc_msgForward函数 or _objc_msgForward_stret函数
```

***

###callSelector函数的实现细节二、JPForwardInvocation函数，负责拦截系统消息转发函数传入的NSInvocation并从中获取到所有的方法执行参数值


****

###JPBoxing包装OC的某一些对象，防止JavaScriptCore.framework转换类型.

> NSMutableArray / NSMutableDictionary / NSMutableString，JavaScriptCore 会把它们转成了 JS 的 Array / Object / String

如上这句话是摘录自JSPatch作者博客...

- test.js

```

var JPBoxingDemo = function(obj) {
    return obj
}
```

- OC代码

```objc
- (void)callJavaScriptDemo5 {
    
    //1. 读取js代码内容
    NSString *path = [[NSBundle mainBundle]pathForResource:@"test1"ofType:@"js"];
    NSString *testScript = [NSString stringWithContentsOfFile:path encoding:NSUTF8StringEncoding error:nil];
    
    //2. 创建运行js代码的环境
//    JSContext *context = [[JSContext alloc] init];
    [JPEngine startEngine];
    JSContext *context = [JPEngine context];
    
    //3. 执行js脚本代码
//    [context evaluateScript:testScript withSourceURL:[NSURL URLWithString:@"main.js"]];
    [JPEngine evaluateScript:testScript];
    
    //4. 从js运行环境读取键值funcDemo对应的函数内容
    JSValue *function = context[@"JPBoxingDemo"];
    
    //5. 测试NSMutableArray
    NSMutableArray *array = [NSMutableArray new];
    [array addObject:@"1111"];
    JSValue *result = [function callWithArguments:@[array]];
    NSLog(@"%@", [[result toObject] class]);
    id arrayObj = [result toObject];
    [arrayObj addObject:@"2222"];
    
    //6. 测试NSMutableDictionary
    NSMutableDictionary *dic = [NSMutableDictionary new];
    [dic setObject:@"1111" forKey:@"key"];
    result = [function callWithArguments:@[dic]];
    NSLog(@"%@", [[result toObject] class]);
    id dicObj = [result toObject];
    [dicObj setObject:@"1111" forKey:@"key"];

    //7. 测试N
    NSMutableString *string = [NSMutableString new];
    [string appendFormat:@"hahah"];
    result = [function callWithArguments:@[string]];
    NSLog(@"%@", [[result toObject] class]);
    id stringObj = [result toObject];
//    [stringObj appendFormat:@"hahah2"];这句话会崩溃，说明已经不是NSMutableString
    
    JSValue *function2 = context[@"funcDemo"];
    result = [function2 callWithArguments:@[@(1)]];
    
}
```

输出从js返回的数据类型

```
2016-03-31 17:43:59.818 JSPatchDeme[73615:445342] __NSArrayM
2016-03-31 17:43:59.818 JSPatchDeme[73615:445342] __NSDictionaryM
2016-03-31 17:43:59.818 JSPatchDeme[73615:445342] NSTaggedPointerString
```

看到只有NSMutableString类型变了，但是前面两个类型没有变化。不知道为什么，有待观察...

****

###区分类方法调用 or 对象方法调用

```
UIAlertView.alloc().init();

//1.
UIAlertView.alloc()调用OC返回一个UIAlertView对象，js代码中使用一个js变量接收
//2.
UIAlertView对象（一个js变量）调用init()方法
```

JSPatch.js中给所有js对象添加 __c()函数中调用 _methodFunc()函数。_methodFunc()函数根据传入的参数值，进行判断是调用类方法or对象方法

```
Object.defineProperty(Object.prototype, "__c", {value: function(methodName) {
	//....
	
	return function(){
      var args = Array.prototype.slice.call(arguments)
      
      //1. self.__obj 有值，且是一个OC对象 >>> 调用对象方法
      //2. self.__obj不存在，self.__clsName存在 >>> 调用类方法
      return _methodFunc(self.__obj, self.__clsName, methodName, args, self.__isSuper)
    }
}
```

OC代码中对创建完毕后的OC对象后，返回的是一个 Map对象，并不是直接OC对象

```objc
static id formatOCToJS(id obj)
{
    if ([obj isKindOfClass:[NSString class]] || \
        [obj isKindOfClass:[NSDictionary class]] || \
        [obj isKindOfClass:[NSArray class]] || \
        [obj isKindOfClass:[NSDate class]])
    {
        return _wrapObj([JPBoxing boxObj:obj]);
    }
    
    if ([obj isKindOfClass:[NSNumber class]] || \
        [obj isKindOfClass:NSClassFromString(@"NSBlock")] || \
        [obj isKindOfClass:[JSValue class]])
    {
        return obj;
    }
    
    //其他类对象
    return _wrapObj(obj);
}
```

```objc
static NSDictionary *_wrapObj(id obj)
{
    if (!obj || obj == _nilObj) {
        return @{@"__isNil": @(YES)};
    }
    
    //返回的是一个Map
    //key : __obj，在js代码中用来判断，是否执行对象方法调用
    //value: 创建出来的OC对象
    return @{@"__obj": obj};
}

```

****

###JSPatch.js闭包内部的 `_formatOCToJS函数`

```
var _formatOCToJS = function(obj) {
    
    if (obj === undefined || obj === null) return false
  	
    //对象类型（Object、Array、null）
    if (typeof obj == "object") 
    {
       //判断传入的obj是否已经存在 __obj属性 或 __isNil属性
       //注意上面两个属性，并不是js对象自带的属性，而是后面JSPatch.js附加上去的
  	   if (obj.__obj) return obj
  	   if (obj.__isNil) return false
	  }

    //数组类型
    if (obj instanceof Array) {

       //依次递归解析数组元素对象，添加到ret数组
	     var ret = []
	     obj.forEach(function(o) {

        //递归调用
	       ret.push(_formatOCToJS(o))
	     })
	     return ret
	  }


   //方法类型
   if (obj instanceof Function) {

       return function() {

          //将内置arguments的Map形式转换成Array形式
    	    var args = Array.prototype.slice.call(arguments)

          //将内置arguments里面存放的所有OC类型参数 转换成 JS类型参数
    	    var formatedArgs = _OC_formatJSToOC(args)
    	    
          //依次判断数组元素类型，如果是js或oc不规则类型，就替换成js或oc统一处理的类型
          for (var i = 0; i < args.length; i++) {
    	        if (args[i] === null || args[i] === undefined || args[i] === false) {
                  //js数据类型不符合，替换成undefined
        	        formatedArgs.splice(i, 1, undefined)
        	    } else if (args[i] == nsnull) {
                  //oc数据类型不符合，替换成成null（注意: 这个null是调用OC方法处理的）
        	        formatedArgs.splice(i, 1, null)
        	    }
    	    }
    	    
          //让方法执行，并传入参数
          var rets = obj.apply(obj, formatedArgs);

          //将OC方法执行的返回值oc对象转换成js对象
          return _OC_formatOCToJS(rets);
    	 }
  	}

    //对象类型
  	if (obj instanceof Object) {
  	   var ret = {}
  	   for (var key in obj) {
         //将OC对象所有属性转换成js对象属性
  	     ret[key] = _formatOCToJS(obj[key])
  	   }
  	   return ret
  	}

    //不是Array、Function、Object
  	return obj
}
```

有些暂时还没搞懂啥意思，还有待具体看代码...
