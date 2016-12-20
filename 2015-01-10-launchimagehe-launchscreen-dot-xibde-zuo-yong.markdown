---
layout: post
title: "LaunchImage和LaunchScreen.xib的作用"
date: 2015-01-10 20:08:45 +0800
comments: true
categories: 
---

###这两个东西都都是用来设置`App的启动图`。

![](http://i12.tietuku.com/7c3aaa9fcdf937b0.png)

***

###在iOS7与iOS8以上，LaunchImage和LaunchScreen.xib设置的问题

- ####二者存在的问题

	- 问题1) 在iOS8中，如果同时设置LaunchImage和LaunchScreen.xib的启动图，那么只调用`LaunchScreen.xib`，而不会调用`LaunchImage`。

	- 问题2) 在iOS8中，如果只设置`LaunchImage`，而不设置`LaunchScreen.xib`，那么在`App Store上展示app时不会有 "已针对iPhone 6、iPhone 6 Plus 优化"`.

	- 问题3) 在iOS7中，启动图只支持`LaunchImage`

- ####所以鉴于如上问题，想`同时支持iOS7与iOS8以上`的系统版本
	- 就必须同时设置LaunchImage和LaunchScreen.xib的对应的尺寸图片

***

- `LaunchImage` 的图片设置

	- 未过滤不同设备、不同方向时的设置项

		![](http://i12.tietuku.com/e4ba442dbb0db940.png)
		
	- 手动过滤掉iphone或ipad、横向或竖向、iOS系统版本

		![](http://i13.tietuku.com/8338dc811ae3ac60.png)

- 针对`竖屏`启动模式下

	- iPhone竖屏需要的尺寸图
	
		```	
		1242*2208 (iPhone6 plus 启用高分辨率模式)
		750*1334  (iPhone6 启用高分辨率模式)
		640*1136
		640*960
		320*480
		```
		
		```
		iPhone Portrait iOS 8-Retina HD 5.5 （1242×2208） @3x
		iPhone Portrait iOS 8-Retina HD 4.7 （750×1334） @2x

		iPhone Portrait iOS 7,8-2x （640×960） @2x
		iPhone Portrait iOS 7,8-Retina 4 （640×1136） @2x

		iPhone Portrait iOS 5,6-1x （320×480） @1x
		iPhone Portrait iOS 5,6-2x （640×960） @2x
		iPhone Portrait iOS 5,6-Retina4 （640×1136） @2x
		```

	- ipad竖屏需要的尺寸图:

		```
		768*1004
		768*1024
		1536*2008
		1536*2048
		```

- 针对`横屏`启动模式下

	- iPhone横屏需要的尺寸图

		```
		2208*1242  (iPhone6 plus)
		```

	- ipad横横屏需要的尺寸图
	
		```
		1024*748
		1024*768
		2048*1496
		2048*1536
		```

***

- `LaunchScreen.xib` 的图片设置

	- 只需要提供一张可以拉伸的图片

	- 在`LaunchScreen.xib`中，拖入一个`UIImageView`，然后设置图片

	- 最后给拖入的`UIImageView`，设置上下左右贴到superView的约束即可
	
	- 并且还可以在`LaunchScreen.xib`中拖入更多的UI控件
	
	- 所以，使用`LaunchScreen.xib`来设置`App启动图`更加方便、更加灵活
	
***

###小结:

- iOS7之前，使用`LaunchImage`提供不同尺寸的png图像，来设置不同尺寸设备的启动图.
	- 需要提供不同尺寸的启动图

- iOS8+之后，使用`LaunchScreen.xib`可以自己在里面添加一个`UIImageView`，然后再设置一张可以拉伸的图片即可，完成各个不同尺寸设备的启动图.
	- 只需要提供一张启动图
	- 并且还可以在xib上添加更多的UIView