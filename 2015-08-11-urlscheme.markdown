---
layout: post
title: "urlscheme"
date: 2015-08-11 14:19:51 +0800
comments: true
categories: 
---

###以前使用过一些分享的SDK时，发现一个神奇的效果。点击分享后，会跳转到其他的App然后完成分享，又跳回到自己的App。一直在想这个过程是如何完成的....

###后来无意中在github看到了opensource开源代码后，于是clone下来，仔细看了一遍，终于知道整个过程是如何实现了。

***

###No1. Scheme Url是什么？有什么用？

> scheme我理解的是，就是一个打开某一个App的标示符.

> iOS7以前，App之间的跳转只能通过xcode给工程设置唯一的scheme来完.

**注：可以给App设置多个scheme，让其他App可以使用不同的标示打开我们的App**



###No2. 简单的使用Scheme设置来掉起对应App

* xcode配置scheme

![Xcode设置](http://i3.tietuku.com/33a33e3473404212.png)


* 分析腾讯分享SDK为我们做得事情：
	
	* 我们的App构造完分享的数据
	* scheme后面，还可以带各种不同的url，标示不同的业务操作
	* openURL: 腾讯QQ指定的scheme，跳转到QQ App处理业务
	* QQ处理完之后，使用 tx+Appid 作为回调我们App的scheme【需要按照SDK文档设置回调scheme】
	* 回调起我们的App


* 当一个App被掉起的时候，走AppDelegate里面的一个函数

```objc

//url: 当前传入的scheme url
//sourceApplication: 掉起我们App的程序进程
- (BOOL)application:(UIApplication *)application openURL:(NSURL *)url sourceApplication:(NSString *)sourceApplication annotation:(id)annotation
  {  
  		//此处写我们App，是否能够处理这个scheme url的代码
  		//1. 能够处理： return YES;
  		//2. 不能处理： return NO;
  }

```


***

###OK，尝试在不使用官方提供的SDK情况下，完成数据分享，那么需要做得步骤：

* 找到对应APP完成对应操作的 **scheme**  【很重要，否则没任何反应】
* 监听到scheme 后面带的一些url参数 --> 完整的url
* App进程之间可以通过 UIPastBoard存取公用数据 【所有App进程都可以存取的map】
* openURL打开App程序
* handleOpenURL处理回调


####No1. 获取到对应的scheme，可以使用swiizle监听得到

####No2. 我自己写了一个组装参数成为一个scheme url的方法:
> 注意：不同的App，需要的参数组装肯定是不同的，需要各自实现

以拼接完成QQ分享的参数拼接为例子：

```objc

//根据传入枚举，分别构造分享的参数url

- (NSString*)_generateShareUrl:(XZHMessage *)msg ForType:(XZHTencentPlatform)type {
    
    NSString *url = [[NSString alloc] initWithString:ShareSchema];
    
    NSString *boundleName = [XZHShareManager base64Encode:[XZHShareManager CFBundleDisplayName]];
    NSString *callback_name = [[self optionDict] objectForKey:@"callback_name"];
    
    NSMutableDictionary *params = [@{
                                    @"thirdAppDisplayName" : boundleName,
                                    @"version" : @"1",
                                    @"cflag" : [NSString stringWithFormat:@"%ld", type],
                                    @"callback_type" : @"scheme",
                                    @"generalpastboard" : @"1",
                                    @"callback_name" : callback_name,
                                    @"src_type" : @"app",
                                    @"shareType" : @"0",
                                    } mutableCopy];
    
    //分享消息是媒体类型
    if (msg.link && !msg.messageType) {
        msg.messageType = XZHMessageNews;
    }
    
    NSDictionary *subParams = nil;
    
    //分享消息是文本
    if ([self computeMessageType:msg] == XZHMessageText) {
        
        NSString *fileData = [XZHShareManager base64AndUrlEncode:msg.title];
        subParams = @{
                      @"file_type" : @"text",
                      @"file_data" : fileData
                      };
        
    } else if ([self computeMessageType:msg] == XZHMessageImage) {
    //分享消息是图像
        
        NSDictionary *data=@{
                             @"file_data":msg.imageData,
                             @"previewimagedata":msg.thumbImageData?:msg.imageData
                             };
        //将图像保存到剪贴板
        [[XZHShareManager manager] clipBoardSave:data
                                          ForKey:SaveObjectForQQPlatformKey
                                        Encoding:XZHClipBoardNSKeyedArchiver];
        
        NSString *title = [XZHShareManager base64AndUrlEncode:msg.title];
        NSString *desc = [XZHShareManager base64AndUrlEncode:msg.desc];
        subParams = @{
                      @"file_type" : @"img",
                      @"title" : title,
                      @"objectlocation" : @"pasteboard",
                      @"description" : desc,
                      };
        
    }else if ([self computeMessageType:msg] == XZHMessageNews) {
    //分享消息是新闻
        
        NSDictionary *data=@{@"previewimagedata":msg.imageData};
        
        //将图像保存到剪贴板
        [[XZHShareManager manager] clipBoardSave:data
                                          ForKey:SaveObjectForQQPlatformKey
                                        Encoding:XZHClipBoardNSKeyedArchiver];
        
        NSString *title = [XZHShareManager base64AndUrlEncode:msg.title];
        NSString *url = [XZHShareManager base64AndUrlEncode:msg.link];
        NSString *desc = [XZHShareManager base64AndUrlEncode:msg.desc];
        
        NSString *msgType=@"news";
        if (msg.messageType == XZHMessageNews) {
            msgType = @"news";
        } else if (msg.messageType == XZHMessageAudio) {
            msgType = @"audio";
        }
        
        subParams = @{
                      @"file_type" : msgType,
                      @"title" : title,
                      @"url" : url,
                      @"description" : desc,
                      @"objectlocation" : @"pasteboard",
                      };
    }
    
    [params addEntriesFromDictionary:subParams];
    
    //拼接完成最终要打开的scheme url
    url = [XZHShareManager urlStringWithOriginUrlString:url appendParameters:params];
    
    return url;
}


然后就是 [[UIApplication sharedApplication] openURL:我们构造出来的scheme url] 即可，完成与sdk一样的功能.

```


####No3. 完成处理之后，对应App会回调我们的App

```objc

//这里可以封装一个所有分享平台统一处理的入口
- (BOOL)application:(UIApplication *)application
      handleOpenURL:(NSURL *)url {
    
    //其他应用掉起当前应用时，传回的参数，如果传入的url能够处理
    if ([[XZHShareManager manager] handleOpenURL:url]) {
        return YES;
    }
    
    //不能处理的App调起
    return YES;
}

//这里轮训所有其他分享平台，看哪一个平台能够处理传回的scheme url
- (BOOL)handleOpenURL:(NSURL *)url {
    self.returnURL = url;
    
    //轮询所有分享平台，看哪个能处理
    for (id<XZHShareInterface> impl in self.platforms) {
        if ([impl handleOpenURL]) {
            return YES;
        }
    }
    return NO;
}

//假设QQ分享的处理
- (BOOL)handleOpenURL {
    XZHShareManager *manager = [XZHShareManager manager];
    
    //保存的是handleOpenUrl方法传入的scheme url
    NSURL *returnURL = manager.returnURL;
    
    //判断scheme是否是当前平台能够处理，然后执行对应回调Block
    if ([returnURL.scheme hasPrefix:ShareCallbackSchemePrix]) {
        NSDictionary *dict = [XZHShareManager parseUrl:[XZHShareManager manager].returnURL];
        if ([dict hasKey:@"error_description"]) {
            [dict setValue:[XZHShareManager base64Decode:dict[@"error_description"]] forKey:@"error_description"];
        }
        if ([dict hasKey:@"error"]) {
            NSInteger code = [dict[@"error"] integerValue];
            if (code != 0) {
                NSError *error = [NSError errorWithDomain:@"response_from_qq" code:code userInfo:dict];
                if (manager.onShareFail) {
                    manager.onShareFail(manager.shareMessage, error);
                }
            }else {
                if (manager.onShareSuccess) {
                    manager.onShareSuccess(manager.shareMessage);
                }
            }
        }
        
        [manager clearCompletions];
        return YES;
    } else if ([returnURL.scheme hasPrefix:OauthCallbackSchemePrix]) {
        
        [manager clearCompletions];
        return YES;
    } else {
        
        [manager clearCompletions];
        return NO;
    }
}

```

***

###总结使用scheme完成App之间的跳转，以及数据传输.

![](http://m1.yea.im/2Hr.png)

***

###所以可以知道这些App应用厂商提供的SDK要完成事情

- 1) 定义好让其他App打开他们自己App的`url scheme`
	- 并且是每一种不同的业务操作，定义不同的`url scheme`或者`url path`

- 2) 规定用户必须按照厂商的规则，在用户的工程中配置好自己的`url scheme`
	- 格式为 `SDK厂商公司名英文缩写://用户要调用的业务对应的路径Path`
	- 只有这样，当厂商的App打开后，并处理完操作之后，才能正确回到用户的App（同样使用url scheme 打开用户App，加上结果值作为参数）

- 3) 让用户使用SDK厂商公开的Api，构造对应要处理的业务的数据项
	- 第一是构造要处理业务所需要的一些参数拼接
	- 第二是最终打开厂商App
