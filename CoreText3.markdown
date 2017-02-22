---
layout: post
title: "CoreText Part 3"
date: 2016-04-26 23:49:55 +0800
comments: true
categories: 
---

记录一些在使用CoreText文字拍本完成TXT阅读App过程中的遇到的问题.

***

###问题、点击CTFrameRef内`没有被排版的区域`（没有文字、没有图片、没有符号的空白区域）所在的CTRun Index 数值为 `-1`

[![CoreText2d01f.gif](http://imgchr.com/images/CoreText2d01f.gif)](http://imgchr.com/image/ADw)

我说一直蛋疼为什么长按手势代码不会执行End结束了，原来原因在这里，如下是我的长按手势回调的代码:

```objc
- (void)longPress:(UILongPressGestureRecognizer *)sender
{
	CGPoint point = [sender locationInView:self];
 	
 	// 获取当前UIKit坐标系下点击point 对应的 CTRun Index   
 	CFIndex curRunIndex = [self tool_getCTRunIndexWithUIKitTouchPoint:point];
 	
 	// 判断CTRun Index 是否正常 【问题就出在这里】
 	if (!(curRunIndex > -1 && curRunIndex < _tempAttributedString.string.length))
        return;
        
    // 后面是绘制文字下划线、以及手势结束后的回调处理
    //...   
    
    // 最后重新绘制图像文字
    [self setNeedsDisplay];
}
```

让我有两个感叹:

- 不要轻易的使用`return`结束语句执行，起码也得搞一句打印调试信息，要不然有时候真一时半会不知道问题出在哪里

- 空白区域的 CTRun Index == -1

OK，知道问题原因就好处理了

####有两种情况点击空白区域:

- 第一种、起始点击的时候就是空白区域
	- 子情况一、在一个段落`外部`区域
		- 找到段首、段尾的CTRunIndex
		- 断尾
	- 子情况二、在一个段落`内部`区域
		- 找到段首、段尾的CTRunIndex
		
- 第二种、长按移动过程中经过空白区域

也是搞了好久才解决如上这些CTRun Index = -1的情况，搞完后其实很简单就是NSRange计算比较麻烦

- 找到 Start CTRunRef Index
	- 找到第一个CTRunIndex != -1 或 第一个存在的CTLineRef的range.location

- 然后找到第一个 Moving CTRunRef Index
	- 找到 CTRunIndex != -1 
	- 注意当遇到 CTRunIndex == -1 时，需要进行修正
	- 修正就是使用一个辅助变量保存前一次正常的CTRunIndex即可

代码就不贴了，主要涉及的一些NSRange计算、CTRunIndex修正
查找、计算的

![CoreText7.gif](http://imgchr.com/images/CoreText7.gif)

****

###问题、UITextView.attributedText崩溃错误: -[__NSCFType textBlocks]: unrecognized selector sent to instance

设置了一些UITextView在执行自己的drawRect:时无法解析的富文本属性造成崩溃.

***

###问题、需要将NSAttributedString存储到本地文件

- 一开始直接将NSAttributedString使用归档到磁盘文件，发现不能够，控制台会报错，但是不会崩溃程序.

- 后来我去github找了些NSAttributedString封装的Model完成归档

- 其实最主要的就是记录那一夜的哪一些富文本属性range、以及range对应的属性key、属性value
	- XxxPageInfo >>> 某一页内容的模型
		- XxxPageAttributeInfo >>> 某一页n个富文本属性设置模型
			- attribute key
			- attribute value
			- attribute range >>> 我使用NSStringFromRange()转换NSString后存储
		- XxxPageAttributeInfo
		- XxxPageAttributeInfo
		- XxxPageAttributeInfo

最主要就是每一页的缓存数据对应起来，发现其实挺简单的.

[![CoreText896eb.gif](http://imgchr.com/images/CoreText896eb.gif)](http://imgchr.com/image/Aob)

***

###问题、UITapGestureRecognizer与UIPageViewController点击翻页事件之间的冲突

突然发现添加了UITapGestureRecognizer事件之后`不能翻页了`，但是很奇怪的是:

- 翻页效果不执行，就是不会翻页
- 但是翻页的UIPageViewController回调delegate函数却仍然执行
	- 真他妈屌，一直导致之前的页面富文本设置缓存对不上号
	- 因为点击右侧，虽然没有执行翻页效果，但是实际上已经执行翻页的操作了

解决就是 >>> UITapGestureRecognizer的delegate函数中，禁用UITapGestureRecognizer事件执行，在禁用前可以做我们自己随便做的事情

- 添加点击事件，注意要设置手势的delegate

```
- (void)addSingleTapGesture
{
    if (_singleTapGuesture == nil) {

        _singleTapGuesture = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(singleTap:)];
	
		// 注意:        
        _singleTapGuesture.delegate = self;
        
        [self addGestureRecognizer:_singleTapGuesture];
    }
}
```


- 手势的delegate回调函数

```
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldReceiveTouch:(UITouch *)touch
{
	// 禁止掉UITapGestureRecognizer事件点击，决定是否弹出编辑框
    if ([gestureRecognizer isKindOfClass:[UITapGestureRecognizer class]]) {
        CGPoint point = [touch locationInView:self];
        
        // 控制距离左侧50内，右侧50内，不弹出编辑框，直接进行翻页
        BOOL isLessLeft = point.x - 0 < kDefaultGestureEnableMargin;
        BOOL isLessRight = self.width - point.x < kDefaultGestureEnableMargin;
        if (isLessLeft || isLessRight) {
            return NO;
        }
	
		//////////////////////////////////////////////
		////	    我们自己的文字、UI处理代码         ///
		//////////////////////////////////////////////
	
		// 操作完成后返回NO，禁用点击事件，让UIPageViewController事件能够执行
	    return NO;
    } else {
        return YES;
    }
}
```

![CoreText8.gif](http://imgchr.com/images/CoreText8.gif)