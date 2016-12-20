---
layout: post
title: "NSURLSession、Backgroud Download Task"
date: 2016-01-07 16:01:51 +0800
comments: true
categories: 
---


###将一些大数据下载Task在后台执行下载

***

###`后台`Task执行过程中，`NSURLSession实例` 会与 `AppDelegate实例` 进行交互，大致分为以下几种:

- 情况1、向Backgroud Session中添加了多个Task，`App一直处于 前台`，等待所有的Task下载结束

	- 这种情况下，所有的Task会按照 SessionConfiguration的配置，全部下载执行完毕
	- Session实例 `不会与` AppDelegate实例 进行交互


- 情况2、向Backgroud Session中添加了多个Task，`App 立刻运行到后台，且一直在后台运行`
	- Session实例 `会与` AppDelegate实例 进行交互
	
- 情况3、向Backgroud Session中添加了多个Task，`App先运行到后台，然后隔一段时间又恢复到前台，App程序 未 退出`


- 情况4、向Backgroud Session中添加了多个Task，`某个时刻App程序 被 退出`
	- 完成所有下载Task后，由iOS系统自动运行App程序，但是运行模式是`后台模式`
	- 然后Session实例 `会与` AppDelegate实例 进行交互


***


###对于 一个NSURLSession实例 对应一个唯一标识来区分 （苹果官方文档建议）

```objc
- (NSURLSession *)backgroundURLSession
{
    static NSURLSession *session = nil;
    static dispatch_once_t onceToken;
    
    dispatch_once(&onceToken, ^{
    
    	//1. 唯一标识创建NSURLSession配置对象
        NSString *identifier = @"后台session的唯一标识";
        
        //2. NSURLSession的配置对象（可以自定义NSURLSessionConfiguration子类化）
        NSURLSessionConfiguration* sessionConfig = [NSURLSessionConfiguration backgroundSessionConfiguration:identifier];
        
        //3. 使用NSURLSessioConfiguration对象创建NSURLSession对象
        //指定回调方法实现对象
        //指定回调方法在哪个线程队列执行
        session = [NSURLSession sessionWithConfiguration:sessionConfig 
                                                delegate:self 
                                           delegateQueue:[NSOperationQueue mainQueue]];
    });

    return session;
}
```

###demo1、可以先取消后恢复执行的下载Task


- 属性保存 可恢复的下载任务

```objc
@property (strong, nonatomic) NSURLSessionDownloadTask *resumableTask;   
```

- NSData属性保存 `当前为止，总共已经经下载的数据`

```objc
@property (strong, nonatomic) NSData *partialData;  
```

- 执行下载任务时，如果是`恢复下载`，那么就使用`downloadTaskWithResumeData:`方法，然后根据`partialData`resume恢复执行

```objc
- (void)resumeDownload {

	//1. 判断Task是否已经执行过
	
	if (!self.resumableTask) 
	{ 
		//2. Task已经存在，从session查找	
		if (self.partialData) 
		{ 
			// 如果是之前被`暂停`的任务，就使用已经保存的数据从session查询到之前被暂停的task
			self.resumableTask = [self.session downloadTaskWithResumeData:self.partialData];
			
		} else { 
		
			//3. Task不存在，即还没有开始下载过 ，直接创建一个新的Task，然后resume执行
			
			NSString *imageURLStr = @"http://farm3.staticflickr.com/2846/9823925914_78cd653ac9_b_d.jpg";  
			
            NSURLRequest *request = [NSURLRequest requestWithURL:[NSURL URLWithString:imageURLStr]];  
            
            self.resumableTask = [self.currentSession downloadTaskWithRequest:request];
		}
		
		//4. 统一执行resume方法，开始执行
		[self.resumableTask resume];
		
	} 
}
```

- 取消可恢复执行的下载任务，应该先将数据保存到partialData中，注意在这里不要调用cancel方法

```objc
- (void)cancelDownLoad {

	[self.resumableTask cancelByProducingResumeData:^(NSData *resumeData) {  
        
        //1. 保存当前下载的数据
        self.partialData = resumeData; 
        
        //2. 只是释放掉Task，不要cancel
        self.resumableTask = nil;  
    }]; 
}
```

- 如果是`恢复执行`Task，那么回调执行`NSURLSessionDownloadDelegate协议`的如下方法

```objc
#pragma mark - NSURLSessionDownloadDelegate

/* 从fileOffset位移处恢复下载任务 */  
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
 didResumeAtOffset:(int64_t)fileOffset
expectedTotalBytes:(int64_t)expectedTotalBytes
{
    //空实现即可，iOS8+ 不用事先此方法
}
```

该方法有个缺陷，就是当用户直接杀掉App程序进程，就不会走cancelDownLoad去调用downloadTask的cancelByProducingResumeData:方法，那么也就无法保存此时下载到的data.


###demo2、App运行在`前台`完成下载，并下文文件并显示进度 

- 创建下载Task，并执行

```objc
- (void)downloadDemo {

	//1.创建url
	NSString *urlStr = @"http://192.168.1.208/FileDownload.aspx?file=1.jpg";
	urlStr = [urlStr stringByAddingPercentEscapesUsingEncoding:NSUTF8StringEncoding];
	NSURL *url = [NSURL URLWithString:urlStr];

	//2.创建请求
	NSMutableURLRequest *request=[NSMutableURLRequest requestWithURL:url];
	    
	//3.创建SessionConfiguration
	//请求的缓存、超时、证书..都使用Config来配置 NSURLSessionConfiguration*sessionConfig=[NSURLSessionConfiguration defaultSessionConfiguration];
	sessionConfig.timeoutIntervalForRequest = 5.0f;//请求超时时间
	sessionConfig.allowsCellularAccess = true;//是否允许蜂窝网络下载（2G/3G/4G）

	//4. 创建会话时，指定delegate、delegateQueue
	NSURLSession *session = [NSURLSession sessionWithConfiguration:sessionConfig delegate:self delegateQueue:nil];//指定配置和代理

	//5. 创建下载Task
	_downloadTask=[session downloadTaskWithRequest:request];

	//6. 执行Task
	[_downloadTask resume];
}
```

- NSURLSessionDownloadDelegate实现

```objc
#pragma mark - NSURLSessionDownloadDelegate



//回调多次接收每一次下载的部分数据
//bytesWritten参数: 每次写入的data字节数
//totalBytesWritten参数: 当前总共写入的data字节数
//totalBytesExpectedToWrite参数: 期望收到的所有data字节数

-(void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask didWriteData:(int64_t)bytesWritten totalBytesWritten:(int64_t)totalBytesWritten totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite{

	//1. 计算当前下载进度并更新视图  
    double downloadProgress = totalBytesWritten / (double)totalBytesExpectedToWrite;  
    
    //2. 格式化进度
    NSString *progressStr = [NSString stringWithFormat:@"%.1f", progress * 100];  
    progressStr = [progressStr stringByAppendingString:@"%"];
    
	//3. 主线程异步执行更新UI代码
    dispatch_async(dispatch_get_main_queue(), ^{  
        self.downloadingProgressView.progress = progress;  
        self.currentProgress_label.text = progressStr;  
    }); 
}

//每一个Task下载结都会执行
-(void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask didFinishDownloadingToURL:(NSURL *)location {

	//注意location是下载后的临时保存路径
	//需要将临时文件移动到需要保存的位置
	//当此回调方法执行完毕后，系统会将临时文件自动删除

	//1. 构造最终存放下载文件的路径
    NSError *error;
    NSString *cachePath=[NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES) lastObject];
    NSString *savePath=[cachePath stringByAppendingPathComponent:_textField.text];
    NSLog(@"%@",savePath);
    NSURL *saveUrl=[NSURL fileURLWithPath:savePath];
    
    //2. 将下载的临时文件，复制到最终存放的目录
    [[NSFileManager defaultManager] copyItemAtURL:location toURL:saveUrl error:&error];
    if (error) {
        NSLog(@"Error is:%@",error.localizedDescription);
    }
    
    //3. 异步主队列更新UI
	dispatch_async(dispatch_get_main_queue(), ^{
		//....
	}
}

//断点下载时回调使用
//一般不用做什么，只需要做一个空实现，iOS8以上不用实现
- (void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask
                                      didResumeAtOffset:(int64_t)fileOffset
                                     expectedTotalBytes:(int64_t)expectedTotalBytes
{
	//空实现...
}
```

- NSURLSessionDelegate实现

```objc
#pragma mark - NSURLSessionDelegate ，任务正常结束 or 错误结束

-(void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error{
	
	//区分有没有发生错误
	
    if (error) {
		//发生错误
		//...处理...
    } else {
    	//未发生错误
    	//...处理...
    }	
}


```

###demo3、App运行在`后台`，进行下载Task执行，除了实现上面的函数之外，还需要额外实现下面的这些函数和设置

- 首先需要工程设置使用`后台获取功能`


- 定义管理后台Task的BackgroudSession单例

```objc
//1. 给backgroud session指定一个`唯一的标识`
//（在有多个后台下载Task时，这个标识符就起作用了）
static NSString *const kBackgroundSessionID = @"myBackgroudDownLoadTaskId_1";

//2. 给backgroud session指定一个`描述`
static NSString *const kBackgroundSessionDes = @"描述信息...";
```

```objc
/* 创建一个后台session单例 */  
- (NSURLSession *)backgroundSession {  

    static NSURLSession *backgroundSess = nil;  
    static dispatch_once_t onceToken;  
    dispatch_once(&onceToken, ^{  
    	
    	//1. 创建一个后台SessionConfiguration
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration backgroundSessionConfigurationWithIdentifier:kBackgroundSessionID];  
        //2. 使用后台Configuration创建NSURLSession
        //2.1 设置Session的回调代理对象
        //2.2 设置回调执行所在线程队列
        backgroundSess = [NSURLSession sessionWithConfiguration:config delegate:self delegateQueue:[NSOperationQueue mainQueue]];  
        
        //3. 设置session的描述
        backgroundSess.sessionDescription = kBackgroundSessionDes;  
        
    });  
      
    return backgroundSess;  
}  
```

- 使用后台session创建Task

```objc
self.backgroundTask = [self.backgroundSession downloadTaskWithRequest:request]; 
```

- 然后resume执行Task

```objc
[self.backgroundTask resume];
```

- App处于后台时，系统通过调用AppDelegate实现函数`application:(UIApplication *)application 
  handleEventsForBackgroundURLSession:(NSString *)identifier completionHandler:(void (^)())completionHandler`来唤醒App去执行一些操作，完成所有的后台Task任务时调用

```objc
#import <UIKit/UIKit.h>

@interface AppDelegate : UIResponder <UIApplicationDelegate>

@property (strong, nonatomic) UIWindow *window;

//声明一个void返回值类型Block，用来保存后面的Block
@property (copy) void (^backgroundURLSessionCompletionHandler)();

//最好使用一个字典来区分不同后台session保存不同的后台Task回调Block
@property(nonatomic, strong) NSMutableDictionary * completionHandlerDictionary;

@end
```

```objc
#import "AppDelegate.h"

@interface AppDelegate ()

@end

@implementation AppDelegate

//...略...

//系统调用通知App程序，所有下载任务完毕
//由系统传入当前session 的唯一标识 identifier
//此函数会被执行多次
- (void)application:(UIApplication *)application handleEventsForBackgroundURLSession:(NSString *)identifier completionHandler:(void (^)())completionHandler 
{	
	//1. 最好使用 单例的BackgroudSession实例
	//否则，因为App进入后台，Session的delgate不会再被通知调用
	NSURLSession *backgroundSession = [self backgroundURLSession];

	//2. 按照不同的sessionid来保存block
	[self.completionHandlerDictionary setObject:handler forKey:identifier];
	
	//3. 更新UI的操作
	//...
	
	//4. 告诉iOS系统已经处理完了
	completionHandler();
}
```

- 最后由系统执行`NSURLSessionDelegate 协议`中的如下方法，向delegate通知`后台Session完成所有的后台Task执行`

```objc
- (void)URLSessionDidFinishEventsForBackgroundURLSession:(NSURLSession *)session
{
    NSLog(@"Background session %@ finished events.\n", session);

    if (session.configuration.identifier)
     {
        // 调用在 -application:handleEventsForBackgroundURLSession: 中保存的 handler
        
        //1. 取出Block
    CompletionHandlerType handler = [self.completionHandlerDictionary objectForKey: identifier];

		//2. 执行Block，然后移除掉
	    if (handler) {
	        [self.completionHandlerDictionary removeObjectForKey: identifier];
	        NSLog(@"Calling completion handler for session %@", identifier);
		
	        handler();
	    }
    }
}
```


###demo4、App没有运行 or 已经退出时，完成后台下载，只需要对面代码进行如下修改

- 如果所有后台Task下载完毕时，App程序没有运行
- 那么iOS系统会自动运行App程序，但是是将App运行在`后台`
- 由iOS系统首先调用 `application:didFinishLaunchingWithOptions:`

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    
    if (UIApplicationStateBackground == application.applicationState) {
        
		
		//当前程序状态 >>> UIApplicationStateBackground
        //由后台下载任务下载完毕后，由iOS系统启动App程序
        //后台运行
        
    } else {
        //当前程序状态 >>> UIApplicationStateActive
        //App正常启动流程
        //前台运行
    }
    
    return YES;
}
```

- 然后再继续执行Session的delegate如下方法

```objc
- (void)URLSessionDidFinishEventsForBackgroundURLSession:(NSURLSession *)session
{
    NSLog(@"Background session %@ finished events.\n", session);

    if (session.configuration.identifier)
     {
        // 调用在 -application:handleEventsForBackgroundURLSession: 中保存的 handler
        
        //1. 取出Block
    CompletionHandlerType handler = [self.completionHandlerDictionary objectForKey: identifier];

		//2. 执行Block，然后移除掉
	    if (handler) {
	        [self.completionHandlerDictionary removeObjectForKey: identifier];
	        NSLog(@"Calling completion handler for session %@", identifier);
		
	        handler();
	    }
    }
}
```

****

###Backgroud Session 不支持创建`Data Task`，只支持`Download Task`与`Upload Task`.


****

###可以通过DataTask完成断点下载，设置请求头的range参数

```objc
NSMutableURLRequest *req = nil;
    
//设置从100字节之后往后下载
[req setValue:@"bytes=100-" forHTTPHeaderField:@"range"];
```