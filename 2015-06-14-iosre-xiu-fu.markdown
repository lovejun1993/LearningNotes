---
layout: post
title: "iOS热修复"
date: 2015-06-14 10:16:21 +0800
comments: true
categories: 
---


###目前iOS中用于热更新界面显示的方法

- webview + Html5 （较原始）
- 基于开源框架cordova使用 Html5（较方便）
- 使用Facebook开源的ReactNative（较新意，需要熟悉他的js语法，比H5的方式更接近iOS原生）
- 动态下发可执行文件
	- 下发lua脚本文件 >>> WaxPatch（长期没有维护更新了）
	- 下发js脚本文件 >>> JSPatch（不推荐热更新，只适合bug、crash临时修复）
	- 下发framework、.a、等其他可执行文件是苹果禁止的

****

###iOS选择原生代码 or H5

- 使用H5的地方
	- 如果APP中出现大段文字（如新闻、攻略等），且格式比较丰富（如加粗，字体多样）
	- 如果APP用户常见页面频换，如（淘宝首页各种不同活动）
	- 如果预算有限（H5开发一套可跨平台覆盖安卓、ios，黑莓、塞班），不是很讲究用户体验，不在乎加载速度

- 使用原生的地方
	- 如果讲究APP反应速度（含页面切换流畅性），则选用原生开发（因为H5其本质是网页，换页时，基本要加载整个页面，就像是浏览器打开一个新页面一样，显得较慢）
	- 如果APP对有无网络、网络优劣敏感（譬如有离线操作，在线操作）、很强用户交互性
	- 如果APP需要频繁调用手机硬件

***

###选择原生代码实现后面临一个问题: 发出的一个版本出现闪退、逻辑性bug时，只能通过重新发版本才能修复

> 本文只叙述借助热更新的思路去临时解决原生代码bug、crash的问题，对于界面显示热更新、逻辑热更新不做叙述.

苹果并没有直接提供方法让开发者做到热更替原生代码，但是苹果提供了两种脚本语言来让开发者可以`操作原生代码`.

- lua 
- javascript

那么知道了可以间接操作原生代码的方法，也就可以相处进行热更替原生代码的思路了:

- App启动时，首选从我们的服务器同步下载脚本代码
- 下载到手机本地后，然后解析并执行脚本代码
- 而脚本代码可以去操作原生代码，
	- 实际就是使用iOS runtime特性替换掉原来产生bug、crash的代码实现
- 一切替换成功之后，再进入App主界面UI

这样一来就通过下载脚本的方式临时对发出版本完成了热修复，注意这只是iOS运行时的特性在App启动的时刻完成的临时修复，但实际产生bug的代码仍然还是在App中，只有通过下次发版本才能彻底修复.

***

###市面上基于lua与javascript封装出来的热修复框架:

- WaxPatch
	- 没有iOS系统版本限制
	- 使用类lua的的伪代码编写脚本
	- 引擎使用不太清楚

- javascript >>> JSPatch
	- 对iOS系统版本必须是iOS7及以上版本（依赖javascriptcore.framework）
	- 使用类JS的的伪代码编写脚本
	- 使用苹果自带的JavaScriptCore.framework作为脚本解析、运行引擎

目前来说，使用JSPatch的会更多，后续着重使用JSPatch做热修复的选择.

***

###使用JSPatch进行热修复原生objc代码的过程:

- 假设发出版本的App中的某一段代导致App程序crash

```objc
@implementation JPTableViewController
...
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath
{
 //如下这一句代码可能会超出数组范围导致crash
  NSString *content = self.dataSource[[indexPath row]];  
  
  JPViewController *ctrl = [[JPViewController alloc] initWithContent:content];
  [self.navigationController pushViewController:ctrl];
}
...
@end
```

- 通过JSPatch可以下发这样一段的JS脚本代码到手机本地

```objc
defineClass("JPTableViewController", {
  //instance method definitions
  tableView_didSelectRowAtIndexPath: function(tableView, indexPath) {
    var row = indexPath.row()
    if (self.dataSource().length > row) {  //加上判断越界的逻辑
      var content = self.dataArr()[row];
      var ctrl = JPViewController.alloc().initWithContent(content);
      self.navigationController().pushViewController(ctrl);
    }
  }
}, {})
```

- 在App启动的时刻从服务器同步下载上面的这一段脚本代码

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    // 初始化JSPatch
    [JPEngine startEngine];
    
    // 同步下载Patch补丁脚本文件
    [NSURLConnection sendSynchronousRequest:[NSURLRequest requestWithURL:[NSURL URLWithString:@"http://cnbang.net/test.js"]] queue:[NSOperationQueue mainQueue] completionHandler:^(NSURLResponse *response, NSData *data, NSError *connectionError) {
    
	    // 安装下载到的js脚本，可以缓存到手机本地，下次App启动直接读取加载
	    NSString *script = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
	    [JPEngine evaluateScript:script];
	}];
	
	// 安装脚本完成完之后，再进入App主界面，此时bug代码已经被替换
	[self enterApp];
 	
 	return YES;   
}

- (void)enterApp {
	self.window = [[UIWindow alloc] initWithFrame:[UIScreen mainScreen].bounds];
    JPViewController *rootViewController = [[JPViewController alloc] init];
    UINavigationController *navigationController = [[UINavigationController alloc] initWithRootViewController:rootViewController];
    self.window.rootViewController = navigationController;
    [self.window makeKeyAndVisible];
    [[UINavigationBar appearance] setBackgroundImage:nil forBarMetrics:UIBarMetricsCompact];
}

@end
```

对于使用JSPatch进行热修复大部分的bug代码，如上步骤已经够用了。

JSPatch还提供了其他热更新代码的用途，但是觉得还是不要使用，至于原因下面叙述，需要知道下JSPatch是如何使用js脚本文件去替换objc的实现.

***

###JSPatch内部对js补丁文件的使用过程

####首先是JSPatch初始化时做的事情

```objc
+ (void)startEngine
{
    if (![JSContext class] || _context) {
        return;
    }
    
    JSContext *context = [[JSContext alloc] init];
    
    // 声明类
    context[@"_OC_defineClass"] = ^(NSString *classDeclaration, JSValue *instanceMethods, JSValue *classMethods) {
        return defineClass(classDeclaration, instanceMethods, classMethods);
    };
    
    // 调用对象方法
    context[@"_OC_callI"] = ^id(JSValue *obj, NSString *selectorName, JSValue *arguments, BOOL isSuper) {
        return callSelector(nil, selectorName, arguments, obj, isSuper);
    };
    
    // 调用类方法
    context[@"_OC_callC"] = ^id(NSString *className, NSString *selectorName, JSValue *arguments) {
        return callSelector(className, selectorName, arguments, nil, NO);
    };
    
    ....略
    
    // 加载JSPatch自带的一个js文件 >>> JSPatch.js
    if ([_context respondsToSelector:@selector(evaluateScript:withSourceURL:)]) {
        [_context evaluateScript:jsCore withSourceURL:[NSURL URLWithString:@"JSPatch.js"]];
    } else {
        [_context evaluateScript:jsCore];
    }
}
```

向JSContext注册了一些js代码中可以调用的`c函数`，再`运行时`用来完成:

- 创建一个类
- 给类类实现一个协议
- 给类添加对象方法、类方法
- 以及对象方法、类方法的实现替换
- oc对象 转 js对象
- 操作oc中的线程队列
- ...等等

可以看做是一个对ios runtime api的小型封装.

还加载了一个自带的JSPatch.js文件，这个文件用来:

- 解析我们下发到App的类js的伪代码 >>> 符合规范的js代码
- 然后调用再JSContext注册好的objc函数

这样就让我们的js脚本代码与objc原生函数建立了关系了

####然后将下载到的js补丁文件扔给JSPatch处理

```objc
[JPEngine evaluateScriptWithPath:js脚本文件的路径];

或

[JPEngine evaluateScript:js脚本文件内容字符串];
```

最终进行解释知悉js代码的函数

```objc
+ (JSValue *)_evaluateScript:(NSString *)script withSourceURL:(NSURL *)resourceURL
{
	// iOS系统版本判断
    if (!script || ![JSContext class]) {
        _exceptionBlock(@"script is nil");
        return nil;
    }
    
    // 所以外部不需要 [JPEngine startEngine]
    [self startEngine];
    
    // 使用正则表达式将我们的类js的脚本文件中的伪代码解析成符合规范的js代码
	// 比如:
	//	UIView.alloc().init()
	//	转换成如下的js代码
	//  UIView.__c('alloc')().__c('init')()
    if (!_regex) {
        _regex = [NSRegularExpression regularExpressionWithPattern:_regexStr options:0 error:nil];
    }
    NSString *formatedScript = [NSString stringWithFormat:@";(function(){try`{`%`@}catch(e){_OC_catch(e.message, e.stack)}})();", [_regex stringByReplacingMatchesInString:script options:0 range:NSMakeRange(0, script.length) withTemplate:_replaceStr]];
    
    // 将解析完毕的js代码，交给JSContext运行，并借助前面加载的JSPtahc.js中的代码
    @try {
        if ([_context respondsToSelector:@selector(evaluateScript:withSourceURL:)]) {
            return [_context evaluateScript:formatedScript withSourceURL:resourceURL];
        } else {
            return [_context evaluateScript:formatedScript];
        }
    }
    @catch (NSException *exception) {
        _exceptionBlock([NSString stringWithFormat:@"%@", exception]);
    }
    return nil;
}
```

####当JSPatch.js处理完毕之后，回调在JSContext中注册好的c函数

- 比如找到/创建一个类/修改类中的方法实现（对象方法or类方法）

```c
static NSDictionary *defineClass(NSString *classDeclaration, JSValue *instanceMethods, JSValue *classMethods)
{
	//1. 如果类不存在就创建
	//2. 获取类实现的所有的Protocol，并添加实现
	//3. 判断是替换 对象方法/类方法
	//（调用overrideMethod这个c函数完成方法实现的替换）
}
```

- 接着调用overrideMethod这个c函数完成函数实现的替换

```c
static void overrideMethod(Class cls, NSString *selectorName, JSValue *function, BOOL isClassMethod, const char *typeDescription)
{
	// 被替换实现方法的SEL标识
    SEL selector = NSSelectorFromString(selectorName);
    
    // 被替换掉的方法实现的编码
    if (!typeDescription) {
        Method method = class_getInstanceMethod(cls, selector);
        typeDescription = (char *)method_getTypeEncoding(method);
    }
    
    // 获取被替换掉的对象方法实现
    IMP originalImp = class_respondsToSelector(cls, selector) ? class_getMethodImplementation(cls, selector) : NULL;
    
    // 系统消息转发函数的实现
    IMP msgForwardIMP = _objc_msgForward;
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

	// 替换系统消息转发函数实现，为 JPForwardInvocation这个c函数
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wundeclared-selector"
    if (class_getMethodImplementation(cls, @selector(forwardInvocation:)) != (IMP)JPForwardInvocation) {
        IMP originalForwardImp = class_replaceMethod(cls, @selector(forwardInvocation:), (IMP)JPForwardInvocation, "v@:@");
        if (originalForwardImp) {
            class_addMethod(cls, @selector(ORIGforwardInvocation:), originalForwardImp, "v@:@");
        }
    }
#pragma clang diagnostic pop
	
	// 构造 `ORIG被替换函数名` 这个SEL回到被替换的函数实现
    if (class_respondsToSelector(cls, selector)) {
        NSString *originalSelectorName = [NSString stringWithFormat:@"ORIG%@", selectorName];
        SEL originalSelector = NSSelectorFromString(originalSelectorName);
        if(!class_respondsToSelector(cls, originalSelector)) {
            class_addMethod(cls, originalSelector, originalImp, typeDescription);
        }
    }
    
    // 构造 `JP被替换函数名`作为key，使用全局字典将要替换的方法实现JSValue对象保存取来
    NSString *JPSelectorName = [NSString stringWithFormat:@"_JP%@", selectorName];
    _initJPOverideMethods(cls);
    _JSOverideMethods[cls][JPSelectorName] = function;
    
	// 最后让被替换函数实现的SEL指向系统消息转发函数实现中
	// (也就是说指向[对象 SEL]时，会直接走到 forwardInvocation:这个SEL对应的函数实现中)
    class_replaceMethod(cls, selector, msgForwardIMP, typeDescription);
}
```

如上完成了两个函数实现的替换:

```
1. 让将要被替换实现的objc函数的SEL指向系统消息转发的objc函数实现forwardInvocation:
2. 在让forwardInvocation:这个SEL指向JPForwardInvocation这个c函数实现
```

- 然后走到JPForwardInvocation这个c函数，这个函数是拦截了系统的消息转发

```objc
static void JPForwardInvocation(__unsafe_unretained id assignSlf, SEL selector, NSInvocation *invocation)
{
	//1. 判断是否是JSPatch处理过的方法
	// （如果不是，直接放行让其回到被拦截的方法实现中去，停止往下执行）
	
	//2. 获取被替换函数执行的所有的参数列表，并转换成JS中使用的数据类型
	
	//3. 从上面_JSOverideMethods缓存字典中查询到被替换的方法实现JSValue对象（js脚本描述的新的方法实现）
	
	//4. 然后通过 `-[JSValue callWithArguments:转换后的JS参数数组]` 完成对js脚本中描述的新的方法实现的调用
	
	//5. 然后在对`-[JSValue callWithArguments:转换后的JS参数数组]`执行后的返回值，再按照typpe encodings进转换
}
```

OK，那么对于JSPatch读取脚本文件中的js代码，对App中的产生bug、crash的`函数`进行`运行时`的的修复（替换），归根结底就是利用了iOS的runtime特性对objc_method_t函数对于的IMP指针的指向进行替换。



***

###JSPatch并非只是一套代码，也提供了一个补丁管理平台

http://www.jspatch.com/Docs/intro

- JSPatch的代码是开源的，就是说我们可以随便查看、修改他的代码

- JSPatch提供了一个对下发补丁文件的后台管理系统，通过SDK的形式提供:
	- 下发脚本的安全问题
	- 脚本的灰度、条件下发
	- 后台管理系统有免费额度

	```
	每个用户每个月有一定额度的免费请求次数配额，目前免费额度为 50w	次请求/月。
	正常情况下这个免费配额可以支撑 1.7w日活 以下的 APP。
	当月超过 50w 次请求后，超出的部分会计费，价格为 1元/10w次请求
	```
	
####由于JSPatch的核心代码是开源的，所以可以考虑自己构建补丁的后台管理系统

- 图简单、方便，就直接使用JSPatch提供的SDK.
- 如果考虑日后自己维护补丁版本等，需要自己做一套这样的后台.

***

###如果说我们自己做补丁文件的后台管理系统

####需要解决的几个问题

- 下发脚本的安全性
- 补丁的增删改查逻辑
- 脚本的灰度下发、条件下发

####问题、下发脚本的安全性

- 1. 首选RSA加密算法（github搜openssl库）
- 2. 服务端计算出脚本文件的 MD5 值，作为这个文件的数字签名
- 3. 服务端通过`私钥`加密第 1 步算出的 MD5 值，得到一个加密后的 MD5 值
- 4. 把 `脚本文件` 和 `加密后的 MD5 值` 一起下发给 客户端
- 5. 客户端拿到加 `密后的 MD5 值`，通过保存在客户端的`公钥`解密
- 6. 客户端再对 下载到的 脚本文件 计算得到一个 MD5 值
- 7. 客户端对比第 5/6 步的两个 MD5 值（分别是客户端和服务端计算出来的 MD5 值），若相等则通过校验，不相等就说明脚本文件被篡改就不安装到本地直接删除

####问题、补丁的增删改查逻辑

```
增: 服务器返回的补丁,本地不存在时,会默认下载存储,并执行.
删: 服务器返回的补丁集中,不包含本地的某个补丁,则此补丁下次不会再被执行，并从本地删除补丁缓存.
改: 服务器返回的补丁,本地包含,但md5值变化,此时会重新下载此补丁.
查: 会默认在应用启动时,执行所有存在,且md5值匹配的补丁.补丁集的信息,会在每次App启动时联网时更新.
```

####问题、脚本的灰度下发、条件下发

这个暂时做不了，需要后台开发配合。

