---
layout: post
title: "NSProgress"
date: 2016-05-15 21:13:39 +0800
comments: true
categories: 
---

###iOS 7 and OS X 10.9新引入了这个 NSProgress类，目标是建立一个标准的机制用来报告长时间运行的任务的进度。

刚好在看AFNetworking的NSURLSessionManager实现中看到这个类，但是我一直没用过，看这个名字好像和进度相关，但还是抱着好奇心去仔细看看怎么用的。

这个类其实出来一两年了，汗 .... 现在才知道，弱爆了。于是网上搜关于这个的资料，尼玛一搜出来全部都是复制CocoaChina那篇文章的仿制品，卧槽 ... 还不如自己去苹果官网看看，尼玛家里这蛋疼的网速...我又草了 ...

***

###去苹果开发文档看一下关于这个类的介绍

https://developer.apple.com/library/ios/documentation/Foundation/Reference/NSProgress_Class/index.html

> The NSProgress class provides a self-contained mechanism for progress reporting. It makes it easy for code that does work to report the progress of that work, and for user interface code to observe that progress for presentation to the user. Specifically, it can be used to show the user a progress bar and explanatory text, both updated properly as progress is made. It also allows work to be cancelled or paused by the user.

大致意思是:

- NSProgress提供了一直`自我ratain`的机制来进行 `进度报告`
	-  self-contained >>> 没明白啥意思...

- 可以通过KVO观察NSProgress属性值改变来随时获取进度值

- NSProgress还可以被cancelled 和 paused、以及设置对应情况的回调block

***

###NSProgress实例化的方式

- 单例

```
+ (nullable NSProgress *)currentProgress;
```

- 多例

```
[[NSProgress alloc] initWithParent:nil userInfo:nil];
```

```
+ (NSProgress *)progressWithTotalUnitCount:(int64_t)unitCount;
+ (NSProgress *)discreteProgressWithTotalUnitCount:(int64_t)unitCount NS_AVAILABLE(10_11, 9_0);
+ (NSProgress *)progressWithTotalUnitCount:(int64_t)unitCount parent:(NSProgress *)parent pendingUnitCount:(int64_t)portionOfParentTotalUnitCount NS_AVAILABLE(10_11, 9_0);
```

****

###NSProgress主要使用的属性

- totalUnitCount

totalUnitCount property represents the total number of units of work that need to be performed

- completedUnitCount and fractionCompleted

	- both represents how much of that work has `already` been completed
	- `fractionCompleted` property is useful for updating progress indicators or textual descriptors
	- to check whether progress `is complete`, you should test that `completedUnitCount >= totalUnitCount` (assuming, of course, that totalUnitCount > 0).
	- **一般都是使用fractionCompleted来获取完成的进度小数值**

****

###NSProgress四个主要的设计目的：

- 松耦合

基本上管理进度的事情统统交给NSProgress，不要在我们的其他层次代码中自己计算进度、在不同类之间传递进度...等等。

- 组合性

往往一个任务，可能由其他几个子任务组成，那么子任务的进度应该反馈到整个父任务的进度中。


- 重用性

NSProgress几乎能被所有能够链接到Foundation框架的代码使用。

- 可用性（个人觉得这个比较重要）

	- 因为往往执行下载、耗时任务都是在其他的`子线程上`，那么其实对于子线程是最需要一个专门负责进度报告的东西
	- 所以苹果允许`给每一个线程绑定一个NSProgress对象`，用来专门报告当前线程所执行任务的进度值

> 每个线程可以绑定一个属于自己的进度报告NSProgress对象。

看下如何给线程绑定一个NSProgress对象

```objc
@interface ViewController ()
@property (nonatomic, strong) NSProgress *progress;
@end

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    //1. 创建一个新的Progress
    _progress = [NSProgress progressWithTotalUnitCount:1];
    
    //2. 打印当前线程的Progress
    NSLog(@"Current Thread Progress1 : %@", [NSProgress currentProgress]);
    
    //3. 给当前主线程绑定一个Progress
    [_progress becomeCurrentWithPendingUnitCount:1];
    
    //4. 打印当前线程的Progress
    NSLog(@"Current Thread Progress2 : %@", [NSProgress currentProgress]);
    
    //5. 解除当前主线程对Progress的绑定
    [_progress resignCurrent];
    
    //6. 打印当前线程的Progress
    NSLog(@"Current Thread Progress3 : %@", [NSProgress currentProgress]);
}

@end
```

输出如下

```
2016-05-15 23:16:39.933 NSProgressDemo[38577:492817] Current Thread Progress1 : (null)
2016-05-15 23:16:39.934 NSProgressDemo[38577:492817] Current Thread Progress2 : <NSProgress: 0x7fcba8515ab0> : Parent: 0x0 / Fraction completed: 0.0000 / Completed: 0 of 1  
2016-05-15 23:16:39.934 NSProgressDemo[38577:492817] Current Thread Progress3 : (null)
```

根据输出可以得到:

- 将一个NSProgress对象绑定到一个线程上使用

```objc
[_progress becomeCurrentWithPendingUnitCount:1];
```

- 线程绑定完成之后，可以通过如下方法获取与线程绑定的NSProgress对象

```objc
[NSProgress currentProgress];//必须在绑定Progress操作的线程执行
```

- 最后解除线程绑定的NSProgress对象

```objc
[_progress resignCurrent];
```

####关于`becomeCurrentWithPendingUnitCount:` 与 `resignCurrent`两个方法的使用

```objc
static void logProgress(NSInteger i, NSProgress *progress) {
    NSLog(@"Current Thread Progress %ld : %@", i, progress);
}

@interface ViewController ()
@property (nonatomic, strong) NSProgress *progress;
@end

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    _progress = [NSProgress progressWithTotalUnitCount:10];
    logProgress(1, [NSProgress currentProgress]);
    
    [_progress becomeCurrentWithPendingUnitCount:5];
    logProgress(2, [NSProgress currentProgress]);

    [_progress resignCurrent];
    logProgress(3, [NSProgress currentProgress]);
    logProgress(4, _progress);
    
    [_progress becomeCurrentWithPendingUnitCount:5];
    logProgress(5, [NSProgress currentProgress]);
    
    [_progress resignCurrent];
    logProgress(6, [NSProgress currentProgress]);
    logProgress(7, _progress);
}

@end
```

输出如下

```
2016-05-15 23:52:26.012 NSProgressDemo[40495:524530] Current Thread Progress 1 : (null)
2016-05-15 23:52:26.013 NSProgressDemo[40495:524530] Current Thread Progress 2 : <NSProgress: 0x7fb09ad2d680> : Parent: 0x0 / Fraction completed: 0.0000 / Completed: 0 of 10  
2016-05-15 23:52:26.014 NSProgressDemo[40495:524530] Current Thread Progress 3 : (null)
2016-05-15 23:52:26.014 NSProgressDemo[40495:524530] Current Thread Progress 4 : <NSProgress: 0x7fb09ad2d680> : Parent: 0x0 / Fraction completed: 0.5000 / Completed: 5 of 10  
2016-05-15 23:52:26.014 NSProgressDemo[40495:524530] Current Thread Progress 5 : <NSProgress: 0x7fb09ad2d680> : Parent: 0x0 / Fraction completed: 0.5000 / Completed: 5 of 10  
2016-05-15 23:52:26.014 NSProgressDemo[40495:524530] Current Thread Progress 6 : (null)
2016-05-15 23:52:26.014 NSProgressDemo[40495:524530] Current Thread Progress 7 : <NSProgress: 0x7fb09ad2d680> : Parent: 0x0 / Fraction completed: 1.0000 / Completed: 10 of 10  
```

- 对于 1、3、6输出(null)是因为解除了线程的绑定
- 对于4、7输出，分别可以看到:
	- Fraction completed: 0.5000 / Completed: 5 of 10 
	- Fraction completed: 1.0000 / Completed: 10 of 10
	- 我想应该可以知道 `becomeCurrentWithPendingUnitCount:` 与 `resignCurrent`方法的作用了吧

- `becomeCurrentWithPendingUnitCount:`的作用:
	- 将NSProgress对象绑定到一个线程
	- 并制定当前线程/当前绑定操作，完成的工作量

- `resignCurrent`的作用:
	- 将NSProgress对象从一个线程上移除绑定
	- 并将移除的NSProgress对象的当前完成工作量（fractionCompleted）`累加` becomeCurrentWithPendingUnitCount:方法指定的工作量数

****

###摘录自苹果开发文档关于NSProgress使用范例一、单个简单任务进度

- demo示例

```objc
- (void)startTaskWithData:(NSData *)data {
    NSUInteger batchSize = ... use a suitable batch size
    
    // 第一步、确定NSProgress要完成的总工作量
    NSProgress *progress = [NSProgress progressWithTotalUnitCount:batchSize];
 
    for (NSUInteger index = 0; index < batchSize; index++) {
        // 第二步、每次都需要检查任务是否已经取消了
        // 任务取消  >>>  NSProgress也要取消掉
        if ([progress isCancelled]) {
             // Tidy up as necessary...
             break;
        }
 
        // Do something with this batch of data...
 
        // 第三步、修改NSProgress中已经完成的工作量
        [progress setCompletedUnitCount:(index + 1)];
    }
}
```

- 主要涉及的NSProgress的api

```
[NSProgress progressWithTotalUnitCount:设置当前NSProgress总共需要完成的工作量];
```

```
[progress setCompletedUnitCount:设置线程完成的工作量];
```

****

###摘录自苹果开发文档关于NSProgress使用范例二、多个任务一起完成

- demo示例

```objc
- (void)startLongOperation {

	// 1. 父级NSProgress进度，来管理两个子任务NSProgress的完成进度，总工作量是100，那么每一个子任务完成后就累加50个工作量
    self.overallProgress = [NSProgress progressWithTotalUnitCount:100];
 
 	//2. 完成第一个任务，并跟踪其进度
    [self.overallProgress becomeCurrentWithPendingUnitCount:50];//让当前线程完成50个工作量
    [self work1];//完成子任务1
    [self.overallProgress resignCurrent];//释放Progress与线程绑定，此时self.overallProgress.fractionCompleted == 50;
 
	//3. 完成第二个任务，并跟踪其进度
    [self.overallProgress becomeCurrentWithPendingUnitCount:50];
    [self work2];
    [self.overallProgress resignCurrent];//此时self.overallProgress.fractionCompleted == 100;
}
 
- (void)work1 {

	// 子任务又划分为10个总工作量
    NSProgress *firstTaskProgress = [NSProgress progressWithTotalUnitCount:10];
	
	//其他代码
}
 
- (void)work2 {

	// 子任务又划分为10个总工作量
    NSProgress *secondTaskProgress = [NSProgress progressWithTotalUnitCount:10];

	//其他代码
}
```

如果多个任务是`异步`完成的，那么需要在异步的回调中执行`-[NSProgress resignCurrent]`

- 主要涉及的NSProgress的api

```
-[NSProgress becomeCurrentWithPendingUnitCount:给某一个线程分配要完成的工作量]
```

```
-[NSProgress resignCurrent];

1. 接触当前线程对NSProgress对象的绑定
2. 解除帮个后让NSProgress对象的当前完成工作量累加上面方法指定的工作量数
```

***

###摘录自苹果开发文档关于NSProgress使用范例三、多个任务一起完成且具有父子关系

```objc
- (void)startLongOperation {

	//1. 确定所有下载任务的总工作量
    self.overallProgress = [NSProgress progressWithTotalUnitCount:10];
 
 	//2. 添加一个子任务progress
    [self.overallProgress addChild:download.progress withPendingUnitCount:8];
    // Do the download…
	
 	//3. 添加一个子任务progress 
    [self.overallProgress addChild:filter.progress withPendingUnitCount:2];
    // Perform the filter…
}
```

- 一定要保证总下载任务的总大小工作量正确
- 以及每一个子任务要完成的工作量大小确定

感觉例子2和例子3区别不是太大，反正都是用来一个大任务来包含多个小任务，然后小任务有自己的进度，大任务也有一个总的进度:

- 例子1、需要在每一个小任务完成之后执行`-[NSProgress resignCurrent]`，如果是异步子任务，就需要在回调中异步执行

- 例子2、首先就获取到每一个小任务的实际工作量、然后加起来就是整个大任务的工作量

****

###下面看一个比上面稍微复杂的例子、单个网络下载文件显示进度

[![NSProgress142ba3.gif](http://imgchr.com/images/NSProgress142ba3.gif)](http://imgchr.com/image/PiI)

- ViewController.m

```objc
#import "Demo1ViewController.h"

#import "ItemDownloader.h"

// KVO属性使用的Context
static void *DownloadProgressContext = &DownloadProgressContext;

// KVO NSProgress对象的 keyPath
static NSString * const FractionCompletedKeyPath = @"fractionCompleted";

// 完成下载任务的URL
static NSString * const DownloadURLString = @"http://m4.pc6.com/cjh3/mpegstreamclip.dmg";

@interface Demo1ViewController ()

@property (nonatomic, strong) UIProgressView *progressBar;
@property (nonatomic, strong) ItemDownloader *downloader;
@property (nonatomic, strong) NSProgress *progress;
@property (nonatomic, strong) UIButton *demo1;

@end

@implementation Demo1ViewController


- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor whiteColor];
    
    self.progressBar = [[UIProgressView alloc] initWithProgressViewStyle:UIProgressViewStyleBar];
    self.progressBar.progressTintColor = [UIColor greenColor];
    self.progressBar.backgroundColor = [UIColor grayColor];
    self.progressBar.frame = CGRectMake(10, 64, 290, 5);
    self.progressBar.progress = 0.f;
    [self.view addSubview:self.progressBar];
    
    _demo1 = [UIButton buttonWithType:UIButtonTypeCustom];
    _demo1.frame = CGRectMake(10, 300, 200, 66);
    [_demo1 setTitle:@"开始下载" forState:UIControlStateNormal];
    [_demo1 setBackgroundColor:[UIColor redColor]];
    [self.view addSubview:_demo1];
    
    [_demo1 addTarget:self action:@selector(startDownloadTask) forControlEvents:UIControlEventTouchUpInside];

}

- (void)startDownloadTask
{
    if (!self.downloader) {
        self.downloader = [[ItemDownloader alloc] init];
    }
    
    // 创建NSProgress，指定工作量为`一个`下载任务
    self.progress = [NSProgress progressWithTotalUnitCount:1];
    
    // 关注NSProgress的fractionCompleted属性值改变
    [self.progress addObserver:self forKeyPath:FractionCompletedKeyPath options:NSKeyValueObservingOptionInitial context:DownloadProgressContext];
    
    // 将创建的NSProgress绑定给当前主线程，并指定完成后工作量+1
    [self.progress becomeCurrentWithPendingUnitCount:1];
    
    // 完成具体的网络下载任务
    [self.downloader downloadItemAtURL:[NSURL URLWithString:DownloadURLString] completionHandler:^(NSData *downloadedData) {
        
        // 移除NSProgress观察者
        [self.progress removeObserver:self forKeyPath:FractionCompletedKeyPath context:DownloadProgressContext];
        
        // 更新UI
        [[NSOperationQueue mainQueue] addOperationWithBlock:^{
            [[[UIAlertView alloc] initWithTitle:@"下载完毕" message:@"" delegate:nil cancelButtonTitle:nil otherButtonTitles:@"知道了", nil] show];
        }];
        
        // 移除Progress与主线程的绑定
        [self.progress resignCurrent];
    }];
    
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    // 接收NSProgress对象的fractionCompleted属性值改变
    if ((context == DownloadProgressContext) && ([keyPath isEqualToString:FractionCompletedKeyPath])) {
      
        // 主线程进行UIProgressView的值修改
        [[NSOperationQueue mainQueue] addOperationWithBlock:^{
//            NSLog(@"progress.fractionCompleted: %f", self.progress.fractionCompleted);
            self.progressBar.progress = self.progress.fractionCompleted;
        }];
        
    } else {
        [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
    }
}

@end
```

- ItemDownloader 使用NSURLSession进行简单的`单任务`下载

```objc
#import <Foundation/Foundation.h>

@interface ItemDownloader : NSObject

- (void)downloadItemAtURL:(NSURL *)url completionHandler:(void (^)(NSData *downloadedData))handler;

@end
```

```objc
#import "ItemDownloader.h"

@interface ItemDownloader () <NSURLSessionDataDelegate>

@property (nonatomic, strong) NSURLSession *session;

@property (nonatomic, strong) NSURLSessionDataTask *dataTask;
@property (nonatomic, strong) NSMutableData *dataDownloaded;

@property (nonatomic, strong) NSProgress *progress;//必须要引用修改的NSProgress对象

@property (nonatomic, copy) void (^handler)(NSData *downloadedData);

@end

@implementation ItemDownloader

- (instancetype)init
{
    self = [super init];
    if (self) {
        NSURLSessionConfiguration *sessionConfiguration = [NSURLSessionConfiguration ephemeralSessionConfiguration];
        _session = [NSURLSession sessionWithConfiguration:sessionConfiguration delegate:self delegateQueue:nil];
    }
    return self;
}

- (void)downloadItemAtURL:(NSURL *)url completionHandler:(void (^)(NSData *downloadedData))handler;
{
    NSParameterAssert(url);
    
    if (self.dataTask) {
        // Only allow one download at a time.
        return;
    }
    
    self.handler = handler;
    
    self.dataDownloaded = [[NSMutableData alloc] init];
    self.dataTask = [self.session dataTaskWithURL:url];
    
    /**
     *  获取主线程绑定的Progress
     */
    self.progress = [NSProgress currentProgress];
    
    self.progress.cancellable = NO;
    self.progress.pausable = NO;
    
    [self.dataTask resume];
}

#pragma mark - NSURLSessionDataDelegate


- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data;
{
    // 总下载文件大小
    int64_t totalExpected = dataTask.countOfBytesExpectedToReceive;
    
    // 设置NSProgress的总工作量
    self.progress.totalUnitCount = totalExpected;
    
    // 当前总下载大小
    int64_t totalRecieved = dataTask.countOfBytesReceived;
    
    // 设置NSProgress当前完成的工作量
    self.progress.completedUnitCount = totalRecieved;

    // 拼接下载的数据
    [self.dataDownloaded appendData:data];
}

#pragma mark - NSURLSessionTaskDelegate
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error;
{
    if (self.handler) {
        
        NSData *data = [NSData dataWithData:self.dataDownloaded];
        
        [[NSOperationQueue mainQueue] addOperationWithBlock:^{
            self.handler(data);
        }];
        
        self.dataDownloaded = nil;
        self.dataTask = nil;
    }
}

@end
```

下载地址随便网上弄一个就是啦....

****

###当同时执行多个下载任务

> 下面看一个使用NSOperation完成网络任务时候有嵌套任务时使用NSProgress完成进度报告.

[![NSProgress28243a.gif](http://imgchr.com/images/NSProgress28243a.gif)](http://imgchr.com/image/Pim)

恩，其实就是在上面的例子稍加修改，使用一个并发NSOperQueue同时执行多个NSOperation，每一个NSOperation就是一个下载任务即可。

- ViewController.m

```objc
#import "Demo2ViewController.h"
#import "UIView+FrameLayout.h"

#import "DownloadTaskOperation.h"

static void *DownloadProgressContext = &DownloadProgressContext;

// KVO NSProgress对象的 keyPath
static NSString * const FractionCompletedKeyPath = @"fractionCompleted";

// 完成下载任务的URL
static NSString * const DownloadURLString = @"http://m4.pc6.com/cjh3/WavePad.dmg";

@interface Demo2ViewController () <NSURLSessionDataDelegate>

@property (nonatomic, strong) UIProgressView *progressBarAllTask;
@property (nonatomic, strong) UIProgressView *progressBarTask1;
@property (nonatomic, strong) UIProgressView *progressBarTask2;

@property (nonatomic, strong) UIButton *demo1;

@property (nonatomic, strong) NSOperationQueue *workQueue;
@property (nonatomic, strong) DownloadTaskOperation *task1;
@property (nonatomic, strong) DownloadTaskOperation *task2;

@property (nonatomic, strong) NSURLSession *session;

@property (nonatomic, strong) NSProgress *totalProgress;

@end

@implementation Demo2ViewController

- (NSOperationQueue *)workQueue {
    if (!_workQueue) {
        _workQueue = [[NSOperationQueue alloc] init];
        _workQueue.maxConcurrentOperationCount = 4;//最大并发数4
    }
    return _workQueue;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor whiteColor];
    
    UILabel *label1 = [UILabel new];
    label1.font = [UIFont systemFontOfSize:12.f];
    label1.text = @"总下载进度";
    [self.view addSubview:label1];
    label1.frame = CGRectMake(5, 50, 80, 35);
    
    _progressBarAllTask = [[UIProgressView alloc] initWithProgressViewStyle:UIProgressViewStyleBar];
    _progressBarAllTask.progressTintColor = [UIColor greenColor];
    _progressBarAllTask.backgroundColor = [UIColor grayColor];
    _progressBarAllTask.frame = CGRectMake(label1.right + 5, 65, 200, 5);
    _progressBarAllTask.progress = 0.f;
    [self.view addSubview:_progressBarAllTask];
    
    UILabel *label2 = [UILabel new];
    label2.font = [UIFont systemFontOfSize:12.f];
    label2.text = @"下载进度1";
    [self.view addSubview:label2];
    label2.frame = CGRectMake(5, 80, 80, 35);
    
    _progressBarTask1 = [[UIProgressView alloc] initWithProgressViewStyle:UIProgressViewStyleBar];
    _progressBarTask1.progressTintColor = [UIColor greenColor];
    _progressBarTask1.backgroundColor = [UIColor grayColor];
    _progressBarTask1.frame = CGRectMake(label2.right + 5, _progressBarAllTask.bottom + 30, 200, 5);
    _progressBarTask1.progress = 0.f;
    [self.view addSubview:_progressBarTask1];
    
    UILabel *label3 = [UILabel new];
    label3.font = [UIFont systemFontOfSize:12.f];
    label3.text = @"下载进度2";
    [self.view addSubview:label3];
    label3.frame = CGRectMake(5, 110, 80, 35);

    _progressBarTask2 = [[UIProgressView alloc] initWithProgressViewStyle:UIProgressViewStyleBar];
    _progressBarTask2.progressTintColor = [UIColor greenColor];
    _progressBarTask2.backgroundColor = [UIColor grayColor];
    _progressBarTask2.frame = CGRectMake(label3.right + 5, _progressBarTask1.bottom + 30, 200, 5);
    _progressBarTask2.progress = 0.f;
    [self.view addSubview:_progressBarTask2];
    
    _demo1 = [UIButton buttonWithType:UIButtonTypeCustom];
    _demo1.frame = CGRectMake(10, 300, 200, 66);
    [_demo1 setTitle:@"开始下载" forState:UIControlStateNormal];
    [_demo1 setBackgroundColor:[UIColor redColor]];
    [self.view addSubview:_demo1];
    
    [_demo1 addTarget:self action:@selector(startDownloadTask) forControlEvents:UIControlEventTouchUpInside];
}

- (void)startDownloadTask
{
    NSURLSessionConfiguration *sessionConfiguration = [NSURLSessionConfiguration ephemeralSessionConfiguration];
    _session = [NSURLSession sessionWithConfiguration:sessionConfiguration delegate:self delegateQueue:nil];
     NSURLSessionDataTask *task = [_session dataTaskWithURL:[NSURL URLWithString:DownloadURLString]];
    [task resume];
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    // 接收NSProgress对象的fractionCompleted属性值改变
    if ((context == DownloadProgressContext) && ([keyPath isEqualToString:FractionCompletedKeyPath])) {
        
        // 主线程进行UIProgressView的值修改
        [[NSOperationQueue mainQueue] addOperationWithBlock:^{
            _progressBarAllTask.progress = _totalProgress.fractionCompleted;
        }];
        
    } else {
        [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
    }
}

#pragma mark - NSURLSessionDataDelegate

- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveResponse:(NSURLResponse *)response completionHandler:(void (^)(NSURLSessionResponseDisposition))completionHandler
{
    // 获取到下载文件的总大小
    int64_t totalSize = [response expectedContentLength];
    
    // 总任务数为2 >>> 2个下载任务
    _totalProgress = [NSProgress progressWithTotalUnitCount:(totalSize * 2)];
    
    //
    [_totalProgress addObserver:self forKeyPath:FractionCompletedKeyPath options:NSKeyValueObservingOptionInitial context:DownloadProgressContext];
    
    // 开始第一个下载任务
    _task1 = [[DownloadTaskOperation alloc] initWithDownloadURL:DownloadURLString completion:^{
        self.progressBarTask1.progress = _task1.progress.fractionCompleted;
    }];
    [_totalProgress addChild:_task1.progress withPendingUnitCount:totalSize];
    [self.workQueue addOperation:_task1];
    
    //开始第二个下载任务
    _task2 = [[DownloadTaskOperation alloc] initWithDownloadURL:DownloadURLString completion:^{
        self.progressBarTask2.progress = _task2.progress.fractionCompleted;
    }];
    [_totalProgress addChild:_task2.progress withPendingUnitCount:totalSize];
    [self.workQueue addOperation:_task2];
}

@end
```

- DownloadTaskOperation:NSOperation封装每一次的网络下载任务

```objc
#import <Foundation/Foundation.h>

@interface DownloadTaskOperation : NSOperation

@property (nonatomic, strong, readonly) NSProgress* progress;

@property (nonatomic, copy, readonly) NSString *url;

- (instancetype)initWithDownloadURL:(NSString *)url completion:(void (^)(void))block;

@end
```

```objc
#import "DownloadTaskOperation.h"
#import "AppDelegate.h"

static NSString* const FinishedKey = @"isFinished";
static NSString* const ExecutingKey = @"isExecuting";

@interface DownloadTaskOperation () <NSURLSessionDataDelegate>

@property (nonatomic, strong) NSURLSession *session;
@property (nonatomic, strong) NSURLSessionDataTask *dataTask;

// 进度报告
@property (nonatomic, strong, readwrite) NSProgress* progress;
@property (nonatomic, copy) void (^complet)(void);

// NSOperation需要实现
@property (readwrite, getter=isFinished) BOOL finished;
@property (readwrite, getter=isExecuting) BOOL executing;

@end

@implementation DownloadTaskOperation

@synthesize finished = _finished;
@synthesize executing = _executing;


- (NSProgress *)progress {
    if (!_progress) {
        _progress = [NSProgress progressWithTotalUnitCount:1];
        _progress.totalUnitCount = NSURLSessionTransferSizeUnknown;
    }
    return _progress;
}

- (instancetype)initWithDownloadURL:(NSString *)url completion:(void (^)(void))block
{
    self = [super init];
    if (self) {
        
        NSURLSessionConfiguration *sessionConfiguration = [NSURLSessionConfiguration ephemeralSessionConfiguration];
        _session = [NSURLSession sessionWithConfiguration:sessionConfiguration delegate:self delegateQueue:nil];
        
        _url = [url copy];
        _complet = [block copy];
    }
    return self;
}

- (void)start {
    
    NSURL *url = [NSURL URLWithString:_url];
    
    [[UIApplication sharedApplication] setNetworkActivityIndicatorVisible:YES];
    
    self.executing = YES;
    
    
    if (_dataTask) {
        return;
    }
    
    _dataTask = [_session dataTaskWithURL:url];
    
    // 监听task的属性值改变
    [_dataTask addObserver:self
                forKeyPath:NSStringFromSelector(@selector(countOfBytesReceived))
                   options:NSKeyValueObservingOptionNew
                   context:NULL];
    
    [_dataTask addObserver:self
                forKeyPath:NSStringFromSelector(@selector(countOfBytesExpectedToReceive))
                   options:NSKeyValueObservingOptionNew
                   context:NULL];
    
    //上传时使用的
//    [task addObserver:self
//           forKeyPath:NSStringFromSelector(@selector(countOfBytesSent))
//              options:NSKeyValueObservingOptionNew
//              context:NULL];
//    [task addObserver:self
//           forKeyPath:NSStringFromSelector(@selector(countOfBytesExpectedToSend))
//              options:NSKeyValueObservingOptionNew
//              context:NULL];
    
    [self.progress addObserver:self
                    forKeyPath:NSStringFromSelector(@selector(fractionCompleted))
                       options:NSKeyValueObservingOptionNew
                       context:NULL];
    
    self.progress.cancellable = NO;
    self.progress.pausable = NO;
    
    [self.dataTask resume];
}

#pragma mark - <NSURLSessionDataDelegate>

- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveData:(NSData *)data
{
    // 总下载文件大小
    int64_t totalExpected = dataTask.countOfBytesExpectedToReceive;
    
    // 设置NSProgress的总工作量
    _progress.totalUnitCount = totalExpected;
    
    // 当前总下载大小
    int64_t totalRecieved = dataTask.countOfBytesReceived;
    
    // 设置NSProgress当前完成的工作量
    _progress.completedUnitCount = totalRecieved;
}

- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error;
{
    self.executing = NO;
    
    if (_complet) {
        [[NSOperationQueue mainQueue] addOperationWithBlock:^{
            _complet();
        }];
    }
    
    _finished = YES;
    
    [[UIApplication sharedApplication] setNetworkActivityIndicatorVisible:NO];
}

#pragma mark - KVO

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSString *,id> *)change context:(void *)context
{
    if ([object isKindOfClass:[NSURLSessionTask class]] || [object isKindOfClass:[NSURLSessionDownloadTask class]])
    {
        if ([keyPath isEqualToString:NSStringFromSelector(@selector(countOfBytesReceived))]) {
            // 总工作量
            self.progress.completedUnitCount = [change[NSKeyValueChangeNewKey] longLongValue];
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(countOfBytesExpectedToReceive))]) {
            // 当前完成的工作量
            self.progress.totalUnitCount = [change[NSKeyValueChangeNewKey] longLongValue];
        }
    } else if ([object isEqual:self.progress]) {
        
        // NSProgress对象属性值被修改后执行回调进度值
        if (_complet) {
            [[NSOperationQueue mainQueue] addOperationWithBlock:^{
                _complet();
            }];
        }
    }
}

#pragma mark - NSOperation KVO通知

- (void)setExecuting:(BOOL)executing {
    [self willChangeValueForKey:ExecutingKey];
    _executing = executing;
    [self didChangeValueForKey:ExecutingKey];
}

- (void)setFinished:(BOOL)finished {
    [self willChangeValueForKey:FinishedKey];
    _finished = finished;
    [self didChangeValueForKey:FinishedKey];
}

- (BOOL)isExecuting {
    return _executing;
}

- (BOOL)isFinished {
    return _finished;
}

- (BOOL)isAsynchronous {
    return YES;
}

@end
```

OK，到此为止... 看苹果文档真的很蛋疼，尤其是我这种英语又不过关的人...

> 经网上一个好友留言后，对上面的多下载任务时NSProgress有更好的方案，仔细看了下确实比我之前的要好很多，还指出了一些循环引用的错误。非常感谢，代码只是写demo，就没注意那么多，不过非常感谢~~~

优化后的NSOprogress组成结构:

```
- 两个下载任务 >>> _mainProgress = [NSProgress progressWithTotalUnitCount:2];//总任务数2个
	- 下载任务一 >>> [_mainProgress addChild:_task1.progress withPendingUnitCount:1];//占用总任务数的一半
	- 下载任务二 >>> [_mainProgress addChild:_task2.progress withPendingUnitCount:1];//占用总任务数的一半
```

就是总进度分为两个子任务，每一个子任务完成时让总进度+1，表示完成一半，那么对ViewController的代码优化如下(一些循环引用就懒得改了，仅仅是demo示例):


```objc
#import "Demo2ViewController.h"
#import "UIView+FrameLayout.h"

#import "DownloadTaskOperation.h"

static void *DownloadProgressContext = &DownloadProgressContext;

// KVO NSProgress对象的 keyPath
static NSString * const FractionCompletedKeyPath = @"fractionCompleted";

// 完成下载任务的URL
static NSString * const DownloadURLString = @"http://m4.pc6.com/cjh3/WavePad.dmg";

@interface Demo2ViewController () <NSURLSessionDataDelegate>

@property (nonatomic, strong) UIProgressView *progressBarAllTask;
@property (nonatomic, strong) UIProgressView *progressBarTask1;
@property (nonatomic, strong) UIProgressView *progressBarTask2;

@property (nonatomic, strong) UIButton *demo1;

@property (nonatomic, strong) NSOperationQueue *workQueue;
@property (nonatomic, strong) DownloadTaskOperation *task1;
@property (nonatomic, strong) DownloadTaskOperation *task2;

@property (nonatomic, strong) NSURLSession *session;

@property (nonatomic, strong) NSProgress *totalProgress;

@property (nonatomic, strong) NSProgress *mainProgress;

@end

@implementation Demo2ViewController

- (NSOperationQueue *)workQueue {
    if (!_workQueue) {
        _workQueue = [[NSOperationQueue alloc] init];
        _workQueue.maxConcurrentOperationCount = 4;//最大并发数4
    }
    return _workQueue;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor whiteColor];
    
    UILabel *label1 = [UILabel new];
    label1.font = [UIFont systemFontOfSize:12.f];
    label1.text = @"总下载进度";
    [self.view addSubview:label1];
    label1.frame = CGRectMake(5, 50, 80, 35);
    
    _progressBarAllTask = [[UIProgressView alloc] initWithProgressViewStyle:UIProgressViewStyleBar];
    _progressBarAllTask.progressTintColor = [UIColor greenColor];
    _progressBarAllTask.backgroundColor = [UIColor grayColor];
    _progressBarAllTask.frame = CGRectMake(label1.right + 5, 65, 200, 5);
    _progressBarAllTask.progress = 0.f;
    [self.view addSubview:_progressBarAllTask];
    
    UILabel *label2 = [UILabel new];
    label2.font = [UIFont systemFontOfSize:12.f];
    label2.text = @"下载进度1";
    [self.view addSubview:label2];
    label2.frame = CGRectMake(5, 80, 80, 35);
    
    _progressBarTask1 = [[UIProgressView alloc] initWithProgressViewStyle:UIProgressViewStyleBar];
    _progressBarTask1.progressTintColor = [UIColor greenColor];
    _progressBarTask1.backgroundColor = [UIColor grayColor];
    _progressBarTask1.frame = CGRectMake(label2.right + 5, _progressBarAllTask.bottom + 30, 200, 5);
    _progressBarTask1.progress = 0.f;
    [self.view addSubview:_progressBarTask1];
    
    UILabel *label3 = [UILabel new];
    label3.font = [UIFont systemFontOfSize:12.f];
    label3.text = @"下载进度2";
    [self.view addSubview:label3];
    label3.frame = CGRectMake(5, 110, 80, 35);

    _progressBarTask2 = [[UIProgressView alloc] initWithProgressViewStyle:UIProgressViewStyleBar];
    _progressBarTask2.progressTintColor = [UIColor greenColor];
    _progressBarTask2.backgroundColor = [UIColor grayColor];
    _progressBarTask2.frame = CGRectMake(label3.right + 5, _progressBarTask1.bottom + 30, 200, 5);
    _progressBarTask2.progress = 0.f;
    [self.view addSubview:_progressBarTask2];
    
    _demo1 = [UIButton buttonWithType:UIButtonTypeCustom];
    _demo1.frame = CGRectMake(10, 300, 200, 66);
    [_demo1 setTitle:@"开始下载" forState:UIControlStateNormal];
    [_demo1 setBackgroundColor:[UIColor redColor]];
    [self.view addSubview:_demo1];
    
    [_demo1 addTarget:self action:@selector(startDownloadTask) forControlEvents:UIControlEventTouchUpInside];
}

- (void)startDownloadTask
{    
    // 开始两个下载任务进度报告
    _mainProgress = [NSProgress progressWithTotalUnitCount:2];//总任务数2个
    
    // 监听progress属性值变化
    [_mainProgress addObserver:self forKeyPath:FractionCompletedKeyPath options:NSKeyValueObservingOptionInitial context:DownloadProgressContext];
    
    // 下载任务一
    _task1 = [[DownloadTaskOperation alloc] initWithDownloadURL:DownloadURLString completion:^{
        self.progressBarTask1.progress = self.task1.progress.fractionCompleted;
    }];
    [self.workQueue addOperation:_task1];
    [_mainProgress addChild:_task1.progress withPendingUnitCount:1];//占用总任务数的一半
    
    // 下载任务二
    _task2 = [[DownloadTaskOperation alloc] initWithDownloadURL:DownloadURLString completion:^{
        self.progressBarTask2.progress = self.task2.progress.fractionCompleted;
    }];
    [self.workQueue addOperation:_task2];
    [_mainProgress addChild:_task2.progress withPendingUnitCount:1];//占用总任务数的一半
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    if ((context == DownloadProgressContext) && ([keyPath isEqualToString:FractionCompletedKeyPath])) {
        [[NSOperationQueue mainQueue] addOperationWithBlock:^{
            self.progressBarAllTask.progress = self.mainProgress.fractionCompleted;
        }];
        
    } else {
        [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
    }
}

@end
```


***


###学习资料

```
http://www.cocoachina.com/industry/20140522/8518.html
https://developer.apple.com/library/ios/documentation/Foundation/Reference/NSProgress_Class/index.html
以及github上一些demo
```