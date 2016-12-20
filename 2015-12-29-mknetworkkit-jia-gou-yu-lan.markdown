---
layout: post
title: "MKNetworkKit 架构预览"
date: 2015-12-29 22:55:32 +0800
comments: true
categories: 
---

###之前看过了AFNetworking源码，感觉挺不错的。但是以前也用过MKNetworkKit，又看到它更新支持iOS9了，所以也静下心来阅读MKNetworkKit的源码.

###MKNetworkKit 2.0，基于NSURLSession与NSURLConfiguration，注意只可以在`iOS7+`以上的SDK上运行.

- iOS7一下，使用 NSOperation + NSURLConnection
- iOS7以上，使用 NSURLSession

****

###MKNetworkKit 2.0 结构:

- Extensions 主要是一些`分类扩展`

- MKNetworkHost 管理所有的`Request`相关、以及服务器Host相关参数

	- 服务器Host相关
		- hostName
		- path
		- portNumber
		- defaultHeaders

	- 管理Request
		- 创建Request
		- 开始一个Request 普通数据请求
		- 开始一个Request 上传
		- 开始一个Request 下载

- MKNetworkRequest 对一个网络`请求`实体类的抽象
	- 对一次网络请求需要的 `参数、请求路径、请求方法、回调Block`实体类抽象
	- 响应元数据描述response、以及response data 
	- 并提供是否使用 缓存response data
	- 包含了一个NSURLSessionTask，方便后续取消执行

- MKCache 提供了一套通用的缓存框架
	- 对象缓存
	- 图片缓存

- NSHTTPURLResponse+MKNKAdditions 提供了基于HTTP 1.1 Cache Control
	- 借助MKCache缓存逻辑

- MKObject 主要提供 实体类对象映射
	- XML
	- JSON

****