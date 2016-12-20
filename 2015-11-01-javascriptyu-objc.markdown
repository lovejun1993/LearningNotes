---
layout: post
title: "JavaScript与objc"
date: 2015-11-01 20:55:17 +0800
comments: true
categories: 
---

###前面其实我写过一些JS与OC交互的demo代码，于是今天去找了出来，可是自己都看不懂啥意思了...哎，看来还是要做笔记啊

****

###首先说下JavaScript与Object-c产生作用的桥梁 --> `UIWebView`

- OC调用JS: `-[UIWebView stringByEvaluatingJavaScriptFromString:]`直接加载js脚本字符串

- JS调用OC:`-[UIWebView shouldStartLoadWithRequest:navigationType:]`进行对JS发起的网络url进行`拦截`，判断url是否是回到本地OC代码，从而停止请求当前的网络url

- 所以，`JavaScript`与`Objective-c`之间必须通过`UIWebView`进行连接

***

###首先来完成一个使用网页html中的JS代码，调用本地OC代码的小例子

- 首次，添加一个`html文件`，并在其中写一些简单的JS代码
	- 注意，这个html可以`网络`上的，也可以是放在`工程本地`的
	- 但最终JS点击事件，都必须完成`window.location.href="某个url"`

```
<!doctype html>
<html>
    <head lang="en">
        <meta charset="UTF-8">
    </head>
    <body>
        <div>
            <button onclick="fn_camera();">拍照</button>
            <button onclick="fn_call();">电话</button>
        </div>
        
        <script>
            function fn_camera() {
                window.location.href = 'xzh://camera';
            }
        
            function fn_call() {
                window.location.href = 'xzh://call';
            }
        </script>
        
    </body>
</html>
```

如上，所以要求所有最终的JS代码，如果想要回到OC中，就必须使用`window.location.href = 'xzh://call';`，让oc中的webview的代理函数可以拦截这个网络请求，从而回到OC代码。

- `ViewController`中创建UIWebview的实例

```
- (UIWebView *)webView {
    if (!_webView) {
        _webView = [[UIWebView alloc] init];
        _webView.frame = self.view.frame;
        [self.view addSubview:_webView];
        
        //切记不要忘记写此句
        _webView.delegate = self;
    }
    return _webView;
}
```

- `ViewController`中加载`html文件`
	- `本地url`html文件
	- `网络url`html文件

```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self loadHtml];
}
```

```
- (void)loadHtml {
    
    //1. 加载网络url对应的html
//    NSURLRequest *request = [NSURLRequest requestWithURL:[NSURL URLWithString:@"http://dawdaw/test.html"]];
    
    //2. 加载本地url对应的html
    NSURL *url = [[NSBundle mainBundle] URLForResource:@"test2" withExtension:@"html"];
    
    //3. 封装成Request
    NSURLRequest *request = [NSURLRequest requestWithURL:url];
    
    //2. 使用webview加载这个request
    [self.webView loadRequest:request];
}
```

- 然后走webview的回调代理函数

```
@protocol UIWebViewDelegate <NSObject>

@optional

//拦截所有发起的网络请求
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType;

//开始请求
- (void)webViewDidStartLoad:(UIWebView *)webView;

//结束请求
- (void)webViewDidFinishLoad:(UIWebView *)webView;

//请求错误
- (void)webView:(UIWebView *)webView didFailLoadWithError:(NSError *)error;

@end
```

如下为当JS代码定向到一个url时，被拦截执行的回调函数，此方法用于从JS返回到本地OC程序，间接完成`JS调用OC`

```
/**
 *  webview每一次发起一个url的网络请求时，都会走这个拦截方法
 *
 *  返回YES: 允许，这个url的网络请求
 *  返回NO: 禁止，这个url的网络请求
 */
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType
{
    //1. 获取当前将要请求的scheme
    NSString *scheme = request.URL.scheme;
    
    //2. 获取当前将要请求的sceme后面的path
    NSRange range = [request.URL.absoluteString rangeOfString:scheme];
    NSString *path = [request.URL.absoluteString substringFromIndex:(range.length + 3)];//+3，去掉 "://"三个字符
    
    //2. 判断当前请求的scheme是否为自己定义的，如果是，就拦截掉进行调用OC的操作
    if ([scheme isEqualToString:@"xzh"]) {
        
        //判断是否执行拍照
        if ([path hasPrefix:@"camera"]) {
            [self fn_openCamera];
        }
        
        //判断是否执行打电话
        if ([path hasPrefix:@"call"]) {
            [self fn_call];
        }
        
        return NO;
    }
    
    //3. 其他发起网络请求
    return YES;
}
```

- JS间接调用的本地OC代码

```
- (void)fn_openCamera {
    //...   
}

- (void)fn_call {
	//...    
}

```

- 小结: `本地JS/网络JS --> 本地OC`
	- 使用WebView加载包含JS代码的`html`文件
	- JS代码需要回到本地OC时，使用`window.location.href=我们自己定义的url`
	- WebView拦截到这个网络url请求，并拦截掉进行请求，回到OC代码，并调用OC对应的代码

***

###使用`WebViewJavascriptBridge`开源库完成`JavaScript`与`Objective-c`互相调用

- 1) 导入库文件

```
#import <WebKit/WebKit.h>
#import <WebViewJavascriptBridge/WebViewJavascriptBridge.h>
```

- 2) 在ViewController中实例化`UIWebView`实例

```
@interface ViewController ()

@property (strong, nonatomic) UIWebView *webView;
@property (strong, nonatomic) WebViewJavascriptBridge *bridge;

@end
```

```
- (UIWebView *)webView {
    if (!_webView) {
        _webView = [[UIWebView alloc] init];
        _webView.frame = self.view.frame;
        [self.view addSubview:_webView];
        
        //注意，使用WebViewJavascriptBridge时，可以不设置delegate
        //_webView.delegate = self;
    }
    return _webView;
}
```

- 3) 在ViewController中实例化`WebViewJavascriptBridge`实例，`完成 WebView与javaScript 建立联系`

```
- (WebViewJavascriptBridge *)bridge {
	
	if (!_bridge) {
		
		//1. 开启日志log
		[WebViewJavascriptBridge enableLogging];
		
		//2. 设置JS向OC 发送的数据后的回调处理Block
		_bridge = [WebViewJavascriptBridge bridgeForWebView:_webView webViewDelegate:self handler:^(id data, WVJBResponseCallback responseCallback)
		           {
		               //2.1 对JS发来的数据进行处理
							NSLog(@"收到来自JS端的数据=%@\n", data);
		               
		               //2.2 反馈给JS的数据
		               responseCallback(@"来自OC端的响应数据");
		           }];
		    
		//3. 注册当html中的`JavaScript Handler`被调用后的回调代码块
		[_bridge registerHandler:@"handler1"
                         handler:^(id data, WVJBResponseCallback responseCallback)
		 {
		     //3.1 对JS发来的数据进行处理
		     //...
		     
		     //3.2 返回JS Handler结果值
		     responseCallback(@"OC端的响应数据");
		 }];
		 
		 [_bridge registerHandler:@"handler2"
                         handler:^(id data, WVJBResponseCallback responseCallback)
		 {
		     //3.1 对JS发来的数据进行处理
		     //...
		     
		     //3.2 返回JS Handler结果值
		     responseCallback(@"OC端的响应数据");
		 }];
		 
		 [_bridge registerHandler:@"handler3"
                         handler:^(id data, WVJBResponseCallback responseCallback)
		 {
		     //3.1 对JS发来的数据进行处理
		     //...
		     
		     //3.2 返回JS Handler结果值
		     responseCallback(@"OC端的响应数据");
		 }];
		 
		 //...其他JavaSript Handler注册
		 
		 //4. 加载包含JS脚本的html文件（本地 or 网络）
        [self loadHtml];
	}
	
	return _bridge;
}
```

- 5) 加载html文件

```
- (void)loadHtml {
    
    NSURL *url = [[NSBundle mainBundle] URLForResource:@"test3" withExtension:@"html"];
    
    NSURLRequest *request = [NSURLRequest requestWithURL:url];
    
    [self.webView loadRequest:request];
}
```

- 6) 看下在html中的`JavaScript Handler`如何按照规定格式写

```
<!doctype html>
<html>
    <head lang="en">
        <meta charset="UTF-8">
    </head>
    <body>
        <div>
            <button onclick="fn_camera();">拍照</button>
            <button onclick="fn_call();">电话</button>
        </div>
        
        <script>
        
            function fn_camera() {
                window.location.href = 'xzh://camera';
            }
        
            function fn_call() {
                window.location.href = 'xzh://call';
            }
        
        
            //------------- WebViewJavascriptBridge 模板开始 -------------------
            function connectWebViewJavascriptBridge(callback) {
                if (window.WebViewJavascriptBridge) {
                    callback(WebViewJavascriptBridge)
                } else {
                    document.addEventListener('WebViewJavascriptBridgeReady', function() {
                                              callback(WebViewJavascriptBridge)
                                              }, false)
                }
            }
            
            connectWebViewJavascriptBridge(function(bridge) {
                                           
               //-------------  开始写自己的JavaScript代码---------------
               var uniqueId = 1
               
               //Log函数
               function log(message, data)
               {
	               var log = document.getElementById('log')
	               var el = document.createElement('div')
	               el.className = 'logLine'
	               el.innerHTML = uniqueId++ + '. ' + message + ':<br/>' + JSON.stringify(data)
	               if (log.children.length) {
	               log.insertBefore(el, log.children[0])
	               
	               } else {
	               		log.appendChild(el)
	               }
               }
               
               //初始化函数
               bridge.init(function(message, responseCallback) {
                           log('JS got a message', message)
                           var data = { 'Javascript Responds':'Wee!' }
                           log('JS responding with', data)
                           responseCallback(data)
                           })
               
               //注册OC可以使用的Handler1
               bridge.registerHandler('handler1',function(data, responseCallback) {
               							
               							  //接收到来自OC发送的数据
	                                      log('ObjC called testHandler with data:', data)
	                                      
	                                      //执行Block返回OC结果值
	                                      var responseData = { 'Javascript Says':'testHandler called!' }   
	                                      responseCallback(responseData)
                                      })
               
               //注册OC可以使用的Handler2
               bridge.registerHandler('handler2',function(data, responseCallback) {
               							
               							  //接收到来自OC发送的数据
	                                      log('ObjC called testHandler with data:', data)
	                                      
	                                      //执行Block返回OC结果值
	                                      var responseData = { 'Javascript Says':'testHandler called!' }
	                                      responseCallback(responseData)
                                      })
               
               //注册OC可以使用的Handler3
               bridge.registerHandler('handler3',function(data, responseCallback) {
               							
               							  //接收到来自OC发送的数据
	                                      log('ObjC called testHandler with data:', data)
	                                      
	                                      //执行Block返回OC结果值
	                                      var responseData = { 'Javascript Says':'testHandler called!' }
	                                      
       responseCallback(responseData)
                                      })
               
               
               /**
                *	如下JS代码是在某个时刻，需要向OC发送数据data时使用的，并接收OC返回的结果值responseData
                *
                */
               bridge.send(data, function(responseData) {
                           log('JS got response', responseData)
                           })
               }
               
               
               /**
                *   如下JS代码是在某个时刻，JS需要调用OC的代码。
                *   那么通过调用之前注册的Handler，达到间接触发OC的Handler回调Block块，完成调用OC的功能
                *
                */
               bridge.callHandler('Handler1', {'foo': 'bar'}, function(response) {
                                  log('JS got response', response)
                                  })
               }
               
               bridge.callHandler('Handler2', {'foo': 'bar'}, function(response) {
                                  log('JS got response', response)
                                  })
               }
               
               bridge.callHandler('Handler3', {'foo': 'bar'}, function(response) {
                                  log('JS got response', response)
                                  })
               }
                                           
            })
            
        </script>
        
    </body>
</html>

```

- 7) ViewController中，使用`_bridge`完成OC向JS发送数据

```
[self.bridge send:@"Well hello there"];
```

```
[self.bridge send:[NSDictionary dictionaryWithObject:@"Foo" forKey:@"Bar"]];
```

```
[self.bridge send:@"Give me a response, will you?" responseCallback:^(id responseData) {

	//发送后，带一个接受JS响应结果的Block
    NSLog(@"ObjC got its response! %@", responseData);
    
}];
```

- 8) ViewController中，OC调用JS代码中写好并注册了的Handler，完成调用JS代码

```
//1. 调用在JS代码中定义好的Handler名字
NSString *jsHandlerName = nil;
    
//2. 构造发送给JS的数据
id data = @{ @"greetingFromObjC": @"Hi there, JS!" };
    
//3. OC调用Handler
[_bridge callHandler:jsHandlerName data:data responseCallback:^(id response) {
    NSLog(@"调用JS Handler后，获取返回的数据: %@", response);
}];
```

OK，可以看出使用这个开源框架，可以不用我们自己去实现`WebView拦截函数`去按照`不同的url`进行拦截处理，达到JS调用OC的效果。

而是简单的完成如下操作

- JS与OC互相调用，通过`Handler`
	- JS中按照规定格式，`注册Handler`，完成被OC调用时的回调代码处理
	- OC中按照给定Api，同样`注册Handler`，完成被JS调用时的回调代码处理

- JS与OC完成相互发送数据
	- JS中使用send数据
	- OC中也是使用send数据
