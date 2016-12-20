---
layout: post
title: "CFSocket底层网络请求"
date: 2015-01-08 11:40:16 +0800
comments: true
categories: 
---

###刚好今天回看自己之前的文章，突然不经意看到了CFSocket最终使用NSMachPort完成向用户子线程发送获取到的网络数据这个部分，所以觉得还是有必要仔细学习一下这个部分做的一些事情.

****

###网络请求delegate回调 使用到了 RunLoop + `Mach Port`完成不同线程之间的事件传递


####首先，我们使用NSURLConnection发起一个网络请求，然后断点回调Block块，查看左侧当前`App进程内 的 所有子线程`

![](http://i4.tietuku.com/1858ff78c2804c3f.jpg)

####从左侧线程栈发现，使用NSURLConnection进行网络请求时，多创建出来的`两个`子线程
	
- 子线程1: `com.apple.CFSocket.private`
- 子线程2: `com.apple.NSURLConnectionLoader`

![](http://i4.tietuku.com/a085adbf338087e9.png)

####iOS中的`网络请求分层`如下，从上至下抽象层次越来越高

- `CFSocket` 			-> 最底层的 网络数据传输 的api
- `CFNetwork`       -> ASIHttpRequest 基于此层封装
- `NSURLConnection` -> AFNetworking OperationManager 基于此层封装
- `NSURLSession`    -> AFNetworking SessionManager, Alamofire


####由NSURLConnection创建的子线程1: `com.apple.CFSocket.private`

- 此线程主要负责完成底层的所有`Socket`相关操作

- 完成客户端Socket与服务端Socket进行数据传输


####由NSURLConnection创建的子线程2: `com.apple.NSURLConnectionLoader`

- 在自己线程的 RunLoop 添加 `Socket事件源` 的监听

- 线程之间 通过 `Mach Port` 进行 `事件传递`

- 当前runloop运行在的 Runloop Mode内部的`_srouce0` 通知 delegate 执行回调


如上是在之前RunLoop文章中写过的，先直接copy过来，有点懒...

###然后开始下面重要的部分

***


###网络请求的过程

- 最终完成网络数据通信的是 CFSocket 做底层

- CFNetwork 是对 CFSocket 的封装，而 NSURLConnection 又是对 CFNetwork 的封装

- NSURLSession 其实内部 还是使用了 NSURLConnection，因为进行数据通信时，还是会创建 `com.apple.NSURLConnectionLoader`这个子线程

- 系统完成网络请求时创建了两个系统子线程
	- com.apple.NSURLConnectionLoader 子线程
	- com.apple.CFSocket.private 子线程

- com.apple.NSURLConnectionLoader 子线程 完成的事情
	- 开启runloop 接收 CFSocket子线程 事件 ，应该是基于 端口 的事件
	- 接收到基于 端口 的事件后，转换成 source0类事件
	- 通知delegate

- com.apple.CFSocket.private 子线程 完成的事情
	- 主要是完成CFSocket底层网络数据通信
	- 完成后像 NSURLConnectionLoader子线程 发送 端口消息