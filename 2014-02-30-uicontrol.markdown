---
layout: post
title: "UIControl"
date: 2014-02-30 00:59:13 +0800
comments: true
categories: 
---

- UIControl是继承自`UIView`，所以其实UIControl也是一个UI控件

- UIControl就是一个`最简单的UIView子类`，**只具备接收事件的功能**，其他显示的子控件什么的都没有

- UIControl适用于自定义一些`接收事件`、内部非常规的subviews的自定义UI控件

- 继承自`UIControl`的子控件
	- UISwitch开关
	- UIButton按钮
	- UISegmentedControl分段控件
	- UISlider滑块
	- UITextField文本输入框
	- UIPageControl分页控件

- 大多数UI控件是继承自`UIView`.

- UIView是继承自`UIResponder`

***

###UIControl接收事件，如下这些属性

####enabled属性

- 控件默认是启用的
- 要禁用控件，可以将enabled属性设置为NO，这将导致控件忽略任何触摸事件
- 被禁用后，控件还可以用不同的方式显示自己，比如变成灰色不可用

####selected属性

- 当用户选中控件时，UIControl类会将其selected属性设置为YES

####contentVerticalAlignment属性

- 主要是设置`内容`的`垂直方向`的对齐方式

- 这个属性值有如下4个枚举值

	- UIControlContentVerticalAlignmentCenter 	- UIControlContentVerticalAlignmentTop  
	- UIControlContentVerticalAlignmentBottom  
	- UIControlContentVerticalAlignmentFill

####contentHorizontalAlignment属性

- 主要是设置`内容`的`水平方向`的对齐方式

- 这个属性值有如下4个枚举值

	- UIControlContentHorizontalAlignmentCenter 
	- UIControlContentHorizontalAlignmentTop
	- UIControlContentHorizontalAlignmentBottom
	- UIControlContentHorizontalAlignmentFill

####事件通知

- UIControl类提供了一个标准机制，来进行`事件登记`和`事件移除`

- 事件登记

```
UIControl *control = nil;
    
[control addTarget:事件处理对象
            action:@selector(消息的名字，方法的SEL值)
  forControlEvents:事件名];
```

- 事件可以用逻辑OR合并在一起，因此可以再一次单独的addTarget调用中指定多个事件
-  `事件名可以自己定义`


- 事件移除

```
[myControl removeTarget:myDelegate   action:nil  forControlEvents:UIControlEventAllEvents];
```

- 获取UIControl登记的所有事件Target，返回一个NSSet

```
NSSet* myActions = [myConreol allTargets]; 
```

- 获取UIControl登记的`某一个事件`Target，所有的的处理方法的SEL

	- 如下例子是以`UIControlEventValueChanged`对应的Target为例

```
NSArray* myActions = [myControl actionForTarget:UIControlEventValueChanged];  
```

- 让UIControl发送一个事件通知，也就是让之前登记的回调方法执行

```
-[UIControl sendActionsForControlEvents:事件名]; 
```