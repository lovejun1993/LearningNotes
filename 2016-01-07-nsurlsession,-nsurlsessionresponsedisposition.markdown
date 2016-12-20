---
layout: post
title: "NSURLSession、NSURLSessionResponseDisposition"
date: 2016-01-07 17:40:54 +0800
comments: true
categories: 
---

###当一个`Data Task`的响应中获知数据很大时，可以讲转成 `Download Task`

***

###是否可以转换的枚举值定义

```objc
typedef NS_ENUM(NSInteger, NSURLSessionResponseDisposition) {
    NSURLSessionResponseCancel = 0,                                      /* Cancel the load, this is the same as -[task cancel] */
    NSURLSessionResponseAllow = 1,                                       /* Allow the load to continue */
    NSURLSessionResponseBecomeDownload = 2,                              /* Turn this request into a download */
    NSURLSessionResponseBecomeStream NS_ENUM_AVAILABLE(10_11, 9_0) = 3,  /* Turn this task into a stream task */
} NS_ENUM_AVAILABLE(NSURLSESSION_AVAILABLE, 7_0);
```

就是说可以告诉NSURLSession如何处理当前的Task.

***

###在接收到响应描述的回调函数中，告诉NSURLSession如何处理Task转换

```objc
- (void)URLSession:(NSURLSession *)session 

          dataTask:(NSURLSessionDataTask *)dataTask

didReceiveResponse:(NSURLResponse *)response

 completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler
 {
	if ([response.MIMEType rangeOfString:@"pdf"].length > 0) 
	{
		// 如果是pdf的话,文件比较大那么转为下载
		completionHandler(NSURLSessionResponseBecomeDownload);
	} else {
		//让Task继续执行
		completionHandler(NSURLSessionResponseAllow);
	}
}
```

###DataTask已经转为DownloadTask，会回调执行NSURLSessionDataDelegate协议中的如下函数

```objc
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
                              didBecomeDownloadTask:(NSURLSessionDownloadTask *)downloadTask
{
	
}
```