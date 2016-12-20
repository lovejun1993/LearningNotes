---
layout: post
title: "HitTestWithEvent"
date: 2014-09-04 22:52:52 +0800
comments: true
categories: 
---


##UI事件传递、`hitTest:withEven:`方法、`pointInside:withEvent:`方法的使用

###一个UIView不接收`触摸事件`的几种情况

- (1) view是`不接收用户交互`

```
view.userInteractionEnabled == NO
```

- (2) view是`隐藏`

```
view.hidden == YES
```

- (3) view是`透明`

```
view.alpha == 0.0 ~ 0.01
```

- (4) `superView.frame` 或 `view自己.frame` 为 `CGRectZero`

```objc
superView.frame = CGRectZero或没有设置
UIView *subView = ...
[superView addSubview:subView];

superView无法hitTest:，就造成内部的subView也无法hitTest:，即最终无法响应UI事件
```

****

###当UIView不满足如上四个条件时，都会接受用户的手势事件、触摸事件，首先会调用如下UIView对象的方法实现

```objc
@interface GrayView : UIView
@end

@implementation GrayView

- (instancetype)initWithFrame:(CGRect)frame
{
    self = [super initWithFrame:frame];
    if (self) {
        self.backgroundColor = [UIColor grayColor];
//        self.userInteractionEnabled = YES; 不使用也会接受触摸事件回调如下2个函数
    }
    return self;
}

- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    NSLog(@"%@ - hitTest:withEvent: 接收到触摸事件", self);
    return [super hitTest:point withEvent:event];
}

- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event {
    NSLog(@"%@ - pointInside:withEvent: 接收到触摸事件", self);
    return [super pointInside:point withEvent:event];
}

@end
```

```objc
@implementation HitTestViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    _gray = [[GrayView alloc] initWithFrame:CGRectMake(0, 0, kScreenW, kScreenH)];
    [self.view addSubview:_gray];
    
}

@end
```

程序运行后点击屏幕，得到如下log

```
2016-09-04 23:14:22.876 UICodes[23948:388135] <GrayView: 0x7ff91af0b350; frame = (0 0; 375 667); layer = <CALayer: 0x7ff91d055aa0>> - hitTest:withEvent: 接收到触摸事件
2016-09-04 23:14:22.877 UICodes[23948:388135] <GrayView: 0x7ff91af0b350; frame = (0 0; 375 667); layer = <CALayer: 0x7ff91d055aa0>> - pointInside:withEvent: 接收到触摸事件
2016-09-04 23:14:22.877 UICodes[23948:388135] <GrayView: 0x7ff91af0b350; frame = (0 0; 375 667); layer = <CALayer: 0x7ff91d055aa0>> - hitTest:withEvent: 接收到触摸事件
2016-09-04 23:14:22.878 UICodes[23948:388135] <GrayView: 0x7ff91af0b350; frame = (0 0; 375 667); layer = <CALayer: 0x7ff91d055aa0>> - pointInside:withEvent: 接收到触摸事件
```

先执行`hitTest:withEvent:`，然后在执行`pointInside:withEvent:`，而且还会执行两次，不明白为啥...

所以一个UIView发生触摸事件时，一定会调用两个方法实现:

- (1) `hitTest:withEvent:`
- (2) `pointInside:withEvent:`

那么这两个函数实现是干什么的了？？？从字面意思来看:

- (1) `hitTest:withEvent:` >>> 测试能否处理这个UIEvent对象，也就是找到最适合处理事件的UIView对象

- (2) `pointInside:withEvent:` >>> 测试触摸事件.point是否在当前view区域内

****

####从网上搜到一份关于`-[UIView hitTest:withEvent:]`的源码大致实现

```objc
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event
{
    // 1. 判断自己是否能够接收触摸事件（是否打开事件交互、是否隐藏、是否透明）
    if (self.userInteractionEnabled == NO || self.hidden == YES || self.alpha <= 0.01) return nil;
    
    // 2. 判断触摸点在不在自己范围内（frame）
    if (![self pointInside:point withEvent:event]) return nil;
    
    // 3. 从`上到下`（最上面开始）遍历自己的所有子控件，看是否有子控件更适合响应此事件
    int count = self.subviews.count;
    
    for (int i = count - 1; i >= 0; i--) {
        UIView *childView = self.subviews[i];
        
        // 将产生事件的坐标，转换成当前相对subview自己坐标原点的坐标
        CGPoint childPoint = [self convertPoint:point toView:childView];
        
        // 又继续交给每一个subview去hitTest
        UIView *fitView = [childView hitTest:childPoint withEvent:event];
        
        // 如果childView的subviews存在能够处理事件的，就返回当前遍历的childView对象作为事件处理对象
        if (fitView) {
            return fitView;
        }
    }
    
    //4. 没有找到比自己更合适的view
    return self;
}
```

可以看到这个`hitTest:withEvent:`函数实现，主要就是测试这个UIView对象，到底能不能够处理这个UI触摸事件。

###使用一个例子代码看`hitTest:withEvent:`与`pointInside:withEvent:`怎么使用

首先写一个BaseView，用来拦截UIView的如上两个方法执行，添加一些log

```objc
#import <UIKit/UIKit.h>

@interface BaseView : UIView

@end

@implementation BaseView

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSLog(@"<%@> touchBegan ", [self class]);
    [super touchesBegan:touches withEvent:event];
}

- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    NSLog(@"<%@> hitTest:withEvent: previous", [self class]);
    UIView *view = [super hitTest:point withEvent:event];
    NSLog(@"<%@> hitTest:withEvent after >>>> retView =%@ ", [self class], view);
    return view;
}

- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event {
    NSLog(@"<%@> pointInside:withEvent: previous", [self class]);
    BOOL ret = [super pointInside:point withEvent:event];
    NSLog(@"<%@> pointInside:withEvent: after >>>> retBOOL = %u ", [self class], ret);
    return ret;
}

@end
```

然后有如下层级结构的自定义View对象:

![](http://i2.buimg.com/4851/025b0e4cb726000b.png)

```
- 控制器.view
	- 灰色View
		- 红色View
		- 绿色View
		- 黄色View
			- 白色View
```


然后上面每一种颜色View都是一个自定义UIView子类，内部不需要重写任何的方法，仅仅是建立一个简单的自定义View即可。

###第一种点击情况、WhiteView中间，得到如下输出:

```
2016-09-04 23:53:52.044 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - hitTest:withEvent: 接收到触摸事件
2016-09-04 23:53:52.044 UICodes[26224:436600] <GrayView> hitTest:withEvent: previous
2016-09-04 23:53:52.045 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - pointInside:withEvent: 接收到触摸事件
2016-09-04 23:53:52.045 UICodes[26224:436600] <GrayView> pointInside:withEvent: previous
2016-09-04 23:53:52.045 UICodes[26224:436600] <GrayView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-04 23:53:52.045 UICodes[26224:436600] <YellowView> hitTest:withEvent: previous
2016-09-04 23:53:52.045 UICodes[26224:436600] <YellowView> pointInside:withEvent: previous
2016-09-04 23:53:52.045 UICodes[26224:436600] <YellowView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-04 23:53:52.045 UICodes[26224:436600] <WhiteView> hitTest:withEvent: previous
2016-09-04 23:53:52.046 UICodes[26224:436600] <WhiteView> pointInside:withEvent: previous
2016-09-04 23:53:52.046 UICodes[26224:436600] <WhiteView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-04 23:53:52.046 UICodes[26224:436600] <WhiteView> hitTest:withEvent after >>>> retView =<WhiteView: 0x7fb690d18550; frame = (70 -30; 235 360); layer = <CALayer: 0x7fb690d58240>> 
2016-09-04 23:53:52.046 UICodes[26224:436600] <YellowView> hitTest:withEvent after >>>> retView =<WhiteView: 0x7fb690d18550; frame = (70 -30; 235 360); layer = <CALayer: 0x7fb690d58240>> 
2016-09-04 23:53:52.047 UICodes[26224:436600] <GrayView> hitTest:withEvent after >>>> retView =<WhiteView: 0x7fb690d18550; frame = (70 -30; 235 360); layer = <CALayer: 0x7fb690d58240>> 
2016-09-04 23:53:52.047 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - hitTest:withEvent: 接收到触摸事件
2016-09-04 23:53:52.047 UICodes[26224:436600] <GrayView> hitTest:withEvent: previous
2016-09-04 23:53:52.047 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - pointInside:withEvent: 接收到触摸事件
2016-09-04 23:53:52.047 UICodes[26224:436600] <GrayView> pointInside:withEvent: previous
2016-09-04 23:53:52.047 UICodes[26224:436600] <GrayView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-04 23:53:52.048 UICodes[26224:436600] <YellowView> hitTest:withEvent: previous
2016-09-04 23:53:52.048 UICodes[26224:436600] <YellowView> pointInside:withEvent: previous
2016-09-04 23:53:52.048 UICodes[26224:436600] <YellowView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-04 23:53:52.048 UICodes[26224:436600] <WhiteView> hitTest:withEvent: previous
2016-09-04 23:53:52.048 UICodes[26224:436600] <WhiteView> pointInside:withEvent: previous
2016-09-04 23:53:52.048 UICodes[26224:436600] <WhiteView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-04 23:53:52.048 UICodes[26224:436600] <WhiteView> hitTest:withEvent after >>>> retView =<WhiteView: 0x7fb690d18550; frame = (70 -30; 235 360); layer = <CALayer: 0x7fb690d58240>> 
2016-09-04 23:53:52.048 UICodes[26224:436600] <YellowView> hitTest:withEvent after >>>> retView =<WhiteView: 0x7fb690d18550; frame = (70 -30; 235 360); layer = <CALayer: 0x7fb690d58240>> 
2016-09-04 23:53:52.049 UICodes[26224:436600] <GrayView> hitTest:withEvent after >>>> retView =<WhiteView: 0x7fb690d18550; frame = (70 -30; 235 360); layer = <CALayer: 0x7fb690d58240>> 
2016-09-04 23:53:52.050 UICodes[26224:436600] <WhiteView> touchBegan 
2016-09-04 23:53:52.050 UICodes[26224:436600] <YellowView> touchBegan 
2016-09-04 23:53:52.050 UICodes[26224:436600] <GrayView> touchBegan 
```

hitTest和pointInside执行过程:

- (1) GrayView
- (2) YellowView
- (3) WhiteView

WhiteView作为找到的处理者。

后续的输出和前面的是一样的，难道是走了两次事件传递？不太清楚，后面再看吧。

###第二种点击情况、点击WhiteView超出YellowView的区域，得到如下输出:

```
2016-09-04 23:55:38.279 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - hitTest:withEvent: 接收到触摸事件
2016-09-04 23:55:38.279 UICodes[26224:436600] <GrayView> hitTest:withEvent: previous
2016-09-04 23:55:38.279 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - pointInside:withEvent: 接收到触摸事件
2016-09-04 23:55:38.279 UICodes[26224:436600] <GrayView> pointInside:withEvent: previous
2016-09-04 23:55:38.280 UICodes[26224:436600] <GrayView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-04 23:55:38.280 UICodes[26224:436600] <YellowView> hitTest:withEvent: previous
2016-09-04 23:55:38.280 UICodes[26224:436600] <YellowView> pointInside:withEvent: previous
2016-09-04 23:55:38.280 UICodes[26224:436600] <YellowView> pointInside:withEvent: after >>>> retBOOL = 0 
2016-09-04 23:55:38.280 UICodes[26224:436600] <YellowView> hitTest:withEvent after >>>> retView =(null) 
2016-09-04 23:55:38.280 UICodes[26224:436600] <GreenView> hitTest:withEvent: previous
2016-09-04 23:55:38.280 UICodes[26224:436600] <GreenView> pointInside:withEvent: previous
2016-09-04 23:55:38.280 UICodes[26224:436600] <GreenView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-04 23:55:38.281 UICodes[26224:436600] <GreenView> hitTest:withEvent after >>>> retView =<GreenView: 0x7fb690d70b80; frame = (50 50; 275 503); layer = <CALayer: 0x7fb690d8e240>> 
2016-09-04 23:55:38.281 UICodes[26224:436600] <GrayView> hitTest:withEvent after >>>> retView =<GreenView: 0x7fb690d70b80; frame = (50 50; 275 503); layer = <CALayer: 0x7fb690d8e240>> 
2016-09-04 23:55:38.281 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - hitTest:withEvent: 接收到触摸事件
2016-09-04 23:55:38.281 UICodes[26224:436600] <GrayView> hitTest:withEvent: previous
2016-09-04 23:55:38.282 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - pointInside:withEvent: 接收到触摸事件
2016-09-04 23:55:38.282 UICodes[26224:436600] <GrayView> pointInside:withEvent: previous
2016-09-04 23:55:38.282 UICodes[26224:436600] <GrayView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-04 23:55:38.282 UICodes[26224:436600] <YellowView> hitTest:withEvent: previous
2016-09-04 23:55:38.282 UICodes[26224:436600] <YellowView> pointInside:withEvent: previous
2016-09-04 23:55:38.282 UICodes[26224:436600] <YellowView> pointInside:withEvent: after >>>> retBOOL = 0 
2016-09-04 23:55:38.282 UICodes[26224:436600] <YellowView> hitTest:withEvent after >>>> retView =(null) 
2016-09-04 23:55:38.282 UICodes[26224:436600] <GreenView> hitTest:withEvent: previous
2016-09-04 23:55:38.282 UICodes[26224:436600] <GreenView> pointInside:withEvent: previous
2016-09-04 23:55:38.282 UICodes[26224:436600] <GreenView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-04 23:55:38.283 UICodes[26224:436600] <GreenView> hitTest:withEvent after >>>> retView =<GreenView: 0x7fb690d70b80; frame = (50 50; 275 503); layer = <CALayer: 0x7fb690d8e240>> 
2016-09-04 23:55:38.283 UICodes[26224:436600] <GrayView> hitTest:withEvent after >>>> retView =<GreenView: 0x7fb690d70b80; frame = (50 50; 275 503); layer = <CALayer: 0x7fb690d8e240>> 
2016-09-04 23:55:38.284 UICodes[26224:436600] <GreenView> touchBegan 
2016-09-04 23:55:38.284 UICodes[26224:436600] <GrayView> touchBegan 
```

hitTest和pointInside执行过程:

- (1) GrayView
- (2) YellowView
- (3) GreenView

GreenView作为找到的处理者。

###第三种点击情况、点击YellowView中超出WhiteView的区域

```
2016-09-04 23:58:08.414 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - hitTest:withEvent: 接收到触摸事件
2016-09-04 23:58:08.414 UICodes[26224:436600] <GrayView> hitTest:withEvent: previous
2016-09-04 23:58:08.415 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - pointInside:withEvent: 接收到触摸事件
2016-09-04 23:58:08.415 UICodes[26224:436600] <GrayView> pointInside:withEvent: previous
2016-09-04 23:58:08.415 UICodes[26224:436600] <GrayView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-04 23:58:08.415 UICodes[26224:436600] <YellowView> hitTest:withEvent: previous
2016-09-04 23:58:08.415 UICodes[26224:436600] <YellowView> pointInside:withEvent: previous
2016-09-04 23:58:08.416 UICodes[26224:436600] <YellowView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-04 23:58:08.416 UICodes[26224:436600] <WhiteView> hitTest:withEvent: previous
2016-09-04 23:58:08.416 UICodes[26224:436600] <WhiteView> pointInside:withEvent: previous
2016-09-04 23:58:08.416 UICodes[26224:436600] <WhiteView> pointInside:withEvent: after >>>> retBOOL = 0 
2016-09-04 23:58:08.416 UICodes[26224:436600] <WhiteView> hitTest:withEvent after >>>> retView =(null) 
2016-09-04 23:58:08.416 UICodes[26224:436600] <YellowView> hitTest:withEvent after >>>> retView =<YellowView: 0x7fb690d197a0; frame = (0 200; 375 300); layer = <CALayer: 0x7fb690d19910>> 
2016-09-04 23:58:08.417 UICodes[26224:436600] <GrayView> hitTest:withEvent after >>>> retView =<YellowView: 0x7fb690d197a0; frame = (0 200; 375 300); layer = <CALayer: 0x7fb690d19910>> 
2016-09-04 23:58:08.417 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - hitTest:withEvent: 接收到触摸事件
2016-09-04 23:58:08.417 UICodes[26224:436600] <GrayView> hitTest:withEvent: previous
2016-09-04 23:58:08.417 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - pointInside:withEvent: 接收到触摸事件
2016-09-04 23:58:08.417 UICodes[26224:436600] <GrayView> pointInside:withEvent: previous
2016-09-04 23:58:08.417 UICodes[26224:436600] <GrayView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-04 23:58:08.418 UICodes[26224:436600] <YellowView> hitTest:withEvent: previous
2016-09-04 23:58:08.418 UICodes[26224:436600] <YellowView> pointInside:withEvent: previous
2016-09-04 23:58:08.418 UICodes[26224:436600] <YellowView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-04 23:58:08.418 UICodes[26224:436600] <WhiteView> hitTest:withEvent: previous
2016-09-04 23:58:08.418 UICodes[26224:436600] <WhiteView> pointInside:withEvent: previous
2016-09-04 23:58:08.418 UICodes[26224:436600] <WhiteView> pointInside:withEvent: after >>>> retBOOL = 0 
2016-09-04 23:58:08.418 UICodes[26224:436600] <WhiteView> hitTest:withEvent after >>>> retView =(null) 
2016-09-04 23:58:08.419 UICodes[26224:436600] <YellowView> hitTest:withEvent after >>>> retView =<YellowView: 0x7fb690d197a0; frame = (0 200; 375 300); layer = <CALayer: 0x7fb690d19910>> 
2016-09-04 23:58:08.419 UICodes[26224:436600] <GrayView> hitTest:withEvent after >>>> retView =<YellowView: 0x7fb690d197a0; frame = (0 200; 375 300); layer = <CALayer: 0x7fb690d19910>> 
2016-09-04 23:58:08.420 UICodes[26224:436600] <YellowView> touchBegan 
2016-09-04 23:58:08.420 UICodes[26224:436600] <GrayView> touchBegan 
```

hitTest和pointInside执行过程:

- (1) GrayView
- (2) YellowView
- (3) WhiteView
- (4) GreenView

YellowView作为找到的处理者。

###第四种点击情况、点击GreenView

```
2016-09-05 00:00:32.265 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - hitTest:withEvent: 接收到触摸事件
2016-09-05 00:00:32.266 UICodes[26224:436600] <GrayView> hitTest:withEvent: previous
2016-09-05 00:00:32.266 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - pointInside:withEvent: 接收到触摸事件
2016-09-05 00:00:32.266 UICodes[26224:436600] <GrayView> pointInside:withEvent: previous
2016-09-05 00:00:32.266 UICodes[26224:436600] <GrayView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-05 00:00:32.266 UICodes[26224:436600] <YellowView> hitTest:withEvent: previous
2016-09-05 00:00:32.266 UICodes[26224:436600] <YellowView> pointInside:withEvent: previous
2016-09-05 00:00:32.266 UICodes[26224:436600] <YellowView> pointInside:withEvent: after >>>> retBOOL = 0 
2016-09-05 00:00:32.267 UICodes[26224:436600] <YellowView> hitTest:withEvent after >>>> retView =(null) 
2016-09-05 00:00:32.267 UICodes[26224:436600] <GreenView> hitTest:withEvent: previous
2016-09-05 00:00:32.267 UICodes[26224:436600] <GreenView> pointInside:withEvent: previous
2016-09-05 00:00:32.267 UICodes[26224:436600] <GreenView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-05 00:00:32.267 UICodes[26224:436600] <GreenView> hitTest:withEvent after >>>> retView =<GreenView: 0x7fb690d70b80; frame = (50 50; 275 503); layer = <CALayer: 0x7fb690d8e240>> 
2016-09-05 00:00:32.267 UICodes[26224:436600] <GrayView> hitTest:withEvent after >>>> retView =<GreenView: 0x7fb690d70b80; frame = (50 50; 275 503); layer = <CALayer: 0x7fb690d8e240>> 
2016-09-05 00:00:32.268 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - hitTest:withEvent: 接收到触摸事件
2016-09-05 00:00:32.268 UICodes[26224:436600] <GrayView> hitTest:withEvent: previous
2016-09-05 00:00:32.268 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - pointInside:withEvent: 接收到触摸事件
2016-09-05 00:00:32.268 UICodes[26224:436600] <GrayView> pointInside:withEvent: previous
2016-09-05 00:00:32.268 UICodes[26224:436600] <GrayView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-05 00:00:32.268 UICodes[26224:436600] <YellowView> hitTest:withEvent: previous
2016-09-05 00:00:32.268 UICodes[26224:436600] <YellowView> pointInside:withEvent: previous
2016-09-05 00:00:32.269 UICodes[26224:436600] <YellowView> pointInside:withEvent: after >>>> retBOOL = 0 
2016-09-05 00:00:32.269 UICodes[26224:436600] <YellowView> hitTest:withEvent after >>>> retView =(null) 
2016-09-05 00:00:32.269 UICodes[26224:436600] <GreenView> hitTest:withEvent: previous
2016-09-05 00:00:32.269 UICodes[26224:436600] <GreenView> pointInside:withEvent: previous
2016-09-05 00:00:32.269 UICodes[26224:436600] <GreenView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-05 00:00:32.269 UICodes[26224:436600] <GreenView> hitTest:withEvent after >>>> retView =<GreenView: 0x7fb690d70b80; frame = (50 50; 275 503); layer = <CALayer: 0x7fb690d8e240>> 
2016-09-05 00:00:32.269 UICodes[26224:436600] <GrayView> hitTest:withEvent after >>>> retView =<GreenView: 0x7fb690d70b80; frame = (50 50; 275 503); layer = <CALayer: 0x7fb690d8e240>> 
2016-09-05 00:00:32.270 UICodes[26224:436600] <GreenView> touchBegan 
2016-09-05 00:00:32.270 UICodes[26224:436600] <GrayView> touchBegan 
```

hitTest和pointInside执行过程:

- (1) GrayView
- (2) YellowView
- (3) GreenView
- (4) GreenView

GreenView作为找到的处理者。

###第五种点击情况、点击RedView区域

```
2016-09-05 00:01:46.251 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - hitTest:withEvent: 接收到触摸事件
2016-09-05 00:01:46.251 UICodes[26224:436600] <GrayView> hitTest:withEvent: previous
2016-09-05 00:01:46.252 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - pointInside:withEvent: 接收到触摸事件
2016-09-05 00:01:46.252 UICodes[26224:436600] <GrayView> pointInside:withEvent: previous
2016-09-05 00:01:46.252 UICodes[26224:436600] <GrayView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-05 00:01:46.252 UICodes[26224:436600] <YellowView> hitTest:withEvent: previous
2016-09-05 00:01:46.253 UICodes[26224:436600] <YellowView> pointInside:withEvent: previous
2016-09-05 00:01:46.253 UICodes[26224:436600] <YellowView> pointInside:withEvent: after >>>> retBOOL = 0 
2016-09-05 00:01:46.253 UICodes[26224:436600] <YellowView> hitTest:withEvent after >>>> retView =(null) 
2016-09-05 00:01:46.253 UICodes[26224:436600] <GreenView> hitTest:withEvent: previous
2016-09-05 00:01:46.253 UICodes[26224:436600] <GreenView> pointInside:withEvent: previous
2016-09-05 00:01:46.253 UICodes[26224:436600] <GreenView> pointInside:withEvent: after >>>> retBOOL = 0 
2016-09-05 00:01:46.253 UICodes[26224:436600] <GreenView> hitTest:withEvent after >>>> retView =(null) 
2016-09-05 00:01:46.254 UICodes[26224:436600] <RedView> hitTest:withEvent: previous
2016-09-05 00:01:46.254 UICodes[26224:436600] <RedView> pointInside:withEvent: previous
2016-09-05 00:01:46.254 UICodes[26224:436600] <RedView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-05 00:01:46.254 UICodes[26224:436600] <RedView> hitTest:withEvent after >>>> retView =<RedView: 0x7fb690d6e2b0; frame = (30 30; 315 543); layer = <CALayer: 0x7fb690d76570>> 
2016-09-05 00:01:46.254 UICodes[26224:436600] <GrayView> hitTest:withEvent after >>>> retView =<RedView: 0x7fb690d6e2b0; frame = (30 30; 315 543); layer = <CALayer: 0x7fb690d76570>> 
2016-09-05 00:01:46.255 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - hitTest:withEvent: 接收到触摸事件
2016-09-05 00:01:46.255 UICodes[26224:436600] <GrayView> hitTest:withEvent: previous
2016-09-05 00:01:46.255 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - pointInside:withEvent: 接收到触摸事件
2016-09-05 00:01:46.255 UICodes[26224:436600] <GrayView> pointInside:withEvent: previous
2016-09-05 00:01:46.255 UICodes[26224:436600] <GrayView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-05 00:01:46.255 UICodes[26224:436600] <YellowView> hitTest:withEvent: previous
2016-09-05 00:01:46.256 UICodes[26224:436600] <YellowView> pointInside:withEvent: previous
2016-09-05 00:01:46.256 UICodes[26224:436600] <YellowView> pointInside:withEvent: after >>>> retBOOL = 0 
2016-09-05 00:01:46.256 UICodes[26224:436600] <YellowView> hitTest:withEvent after >>>> retView =(null) 
2016-09-05 00:01:46.256 UICodes[26224:436600] <GreenView> hitTest:withEvent: previous
2016-09-05 00:01:46.256 UICodes[26224:436600] <GreenView> pointInside:withEvent: previous
2016-09-05 00:01:46.256 UICodes[26224:436600] <GreenView> pointInside:withEvent: after >>>> retBOOL = 0 
2016-09-05 00:01:46.256 UICodes[26224:436600] <GreenView> hitTest:withEvent after >>>> retView =(null) 
2016-09-05 00:01:46.256 UICodes[26224:436600] <RedView> hitTest:withEvent: previous
2016-09-05 00:01:46.257 UICodes[26224:436600] <RedView> pointInside:withEvent: previous
2016-09-05 00:01:46.257 UICodes[26224:436600] <RedView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-05 00:01:46.257 UICodes[26224:436600] <RedView> hitTest:withEvent after >>>> retView =<RedView: 0x7fb690d6e2b0; frame = (30 30; 315 543); layer = <CALayer: 0x7fb690d76570>> 
2016-09-05 00:01:46.257 UICodes[26224:436600] <GrayView> hitTest:withEvent after >>>> retView =<RedView: 0x7fb690d6e2b0; frame = (30 30; 315 543); layer = <CALayer: 0x7fb690d76570>> 
2016-09-05 00:01:46.258 UICodes[26224:436600] <RedView> touchBegan 
2016-09-05 00:01:46.258 UICodes[26224:436600] <GrayView> touchBegan 
```

hitTest和pointInside执行过程:

- (1) GrayView
- (2) YellowView
- (3) GreenView
- (4) RedView

RedView作为找到的处理者。


###第六种点击情况、点击GrayView

```
2016-09-05 00:02:52.549 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - hitTest:withEvent: 接收到触摸事件
2016-09-05 00:02:52.549 UICodes[26224:436600] <GrayView> hitTest:withEvent: previous
2016-09-05 00:02:52.549 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - pointInside:withEvent: 接收到触摸事件
2016-09-05 00:02:52.549 UICodes[26224:436600] <GrayView> pointInside:withEvent: previous
2016-09-05 00:02:52.550 UICodes[26224:436600] <GrayView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-05 00:02:52.550 UICodes[26224:436600] <YellowView> hitTest:withEvent: previous
2016-09-05 00:02:52.550 UICodes[26224:436600] <YellowView> pointInside:withEvent: previous
2016-09-05 00:02:52.550 UICodes[26224:436600] <YellowView> pointInside:withEvent: after >>>> retBOOL = 0 
2016-09-05 00:02:52.551 UICodes[26224:436600] <YellowView> hitTest:withEvent after >>>> retView =(null) 
2016-09-05 00:02:52.551 UICodes[26224:436600] <GreenView> hitTest:withEvent: previous
2016-09-05 00:02:52.551 UICodes[26224:436600] <GreenView> pointInside:withEvent: previous
2016-09-05 00:02:52.551 UICodes[26224:436600] <GreenView> pointInside:withEvent: after >>>> retBOOL = 0 
2016-09-05 00:02:52.551 UICodes[26224:436600] <GreenView> hitTest:withEvent after >>>> retView =(null) 
2016-09-05 00:02:52.551 UICodes[26224:436600] <RedView> hitTest:withEvent: previous
2016-09-05 00:02:52.551 UICodes[26224:436600] <RedView> pointInside:withEvent: previous
2016-09-05 00:02:52.552 UICodes[26224:436600] <RedView> pointInside:withEvent: after >>>> retBOOL = 0 
2016-09-05 00:02:52.552 UICodes[26224:436600] <RedView> hitTest:withEvent after >>>> retView =(null) 
2016-09-05 00:02:52.552 UICodes[26224:436600] <GrayView> hitTest:withEvent after >>>> retView =<GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> 
2016-09-05 00:02:52.552 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - hitTest:withEvent: 接收到触摸事件
2016-09-05 00:02:52.553 UICodes[26224:436600] <GrayView> hitTest:withEvent: previous
2016-09-05 00:02:52.553 UICodes[26224:436600] <GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> - pointInside:withEvent: 接收到触摸事件
2016-09-05 00:02:52.553 UICodes[26224:436600] <GrayView> pointInside:withEvent: previous
2016-09-05 00:02:52.553 UICodes[26224:436600] <GrayView> pointInside:withEvent: after >>>> retBOOL = 1 
2016-09-05 00:02:52.553 UICodes[26224:436600] <YellowView> hitTest:withEvent: previous
2016-09-05 00:02:52.554 UICodes[26224:436600] <YellowView> pointInside:withEvent: previous
2016-09-05 00:02:52.554 UICodes[26224:436600] <YellowView> pointInside:withEvent: after >>>> retBOOL = 0 
2016-09-05 00:02:52.554 UICodes[26224:436600] <YellowView> hitTest:withEvent after >>>> retView =(null) 
2016-09-05 00:02:52.554 UICodes[26224:436600] <GreenView> hitTest:withEvent: previous
2016-09-05 00:02:52.554 UICodes[26224:436600] <GreenView> pointInside:withEvent: previous
2016-09-05 00:02:52.554 UICodes[26224:436600] <GreenView> pointInside:withEvent: after >>>> retBOOL = 0 
2016-09-05 00:02:52.554 UICodes[26224:436600] <GreenView> hitTest:withEvent after >>>> retView =(null) 
2016-09-05 00:02:52.554 UICodes[26224:436600] <RedView> hitTest:withEvent: previous
2016-09-05 00:02:52.555 UICodes[26224:436600] <RedView> pointInside:withEvent: previous
2016-09-05 00:02:52.555 UICodes[26224:436600] <RedView> pointInside:withEvent: after >>>> retBOOL = 0 
2016-09-05 00:02:52.555 UICodes[26224:436600] <RedView> hitTest:withEvent after >>>> retView =(null) 
2016-09-05 00:02:52.555 UICodes[26224:436600] <GrayView> hitTest:withEvent after >>>> retView =<GrayView: 0x7fb690d20d50; frame = (0 64; 375 667); layer = <CALayer: 0x7fb690d70920>> 
2016-09-05 00:02:52.556 UICodes[26224:436600] <GrayView> touchBegan 
```

hitTest和pointInside执行过程:

- (1) GrayView
- (2) YellowView
- (3) GreenView
- (4) RedView

GrayView作为找到的处理者。


###下从上述6个触摸事件日志中，小结事件传递的过程:

- (1) 都是先从 `super view` 开始，然后传递给所有的`subviews`尝试响应

- (2) 一个View调用`hitTest`开始尝试事件响应，而调用`pointInside`看自己是否能够处理这个坐标的事件

- (3) 如果能够处理，就判断还有没有subviews（`[View对象 pointInside:withEvent:] == YES`）
	- 如果没有，将`自己作为事件响应者`返回
	- 如果有，继续交给`subviews`去尝试响应

- (4) 最终如果没有找到事件响应者，那么将`root view`比如上面的`GaryView`作为最终的事件响应者

###事件传递的查找结构就类似一个树形结构:

- (1) 从`最底层`的`RootView`开始发送`hitTest:withEvent:`消息，依次询问是否能成功响应者

- (2) 但是对于某个View的所有`subviews`的`hitTest:withEvent:`消息询问，却是从`最上面`的到`最下面`的subview依次询问

- (3) 每个能执行`hitTest:withEvent:`方法的view都属于事件传递的一部分，但是只有`pointInside:withEvent:`返回`YES`的view才属于响应者链条

###除了事件传递之外，还有 `事件响应、事件响应链`

- (1) 事件响应者: 继承UIResponder的对象称之为响应者对象，能够处理touchesBegan等触摸事件。

- (2) 响应者链: 由很多`事件响应者`链接在一起`组合`起来的一个链条称之为响应者链。但是其中有一个View是第一之间响应者，其他的都是备用。

####事件传递是`自底向上`依次询问View对象，那么当找到了事件第一响应者之后，在构成的响应者链条中去处理事件时，却是`自上而下`依次去响应事件

```
找到的第一响应者View -> super view -> root view -> 控制器view -> 控制器 -> window
```

- 让一个触摸事件让`处于响应者链条中的多个响应者同时处理该事件`，先重写BaseView的touchXxx等方法实现，然后所有的View继承自BaseView

```objc
@implementation BaseView

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSLog(@"<%@> touchBegan ", [self class]);
    [super touchesBegan:touches withEvent:event];
}

- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    NSLog(@"<%@> hitTest:withEvent: previous", [self class]);
    UIView *view = [super hitTest:point withEvent:event];
    NSLog(@"<%@> hitTest:withEvent after >>>> retView =%@ ", [self class], view);
    return view;
}

- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event {
    NSLog(@"<%@> pointInside:withEvent: previous", [self class]);
    BOOL ret = [super pointInside:point withEvent:event];
    NSLog(@"<%@> pointInside:withEvent: after >>>> retBOOL = %u ", [self class], ret);
    return ret;
}

@end
```

- (2) `屏蔽`某个View的成为事件响应者，比如: 屏蔽上面例子中的黑色View的事件影响

```objc
#import <UIKit/UIKit.h>
#import "BaseView.h"

@interface BlackView : BaseView

@end

@implementation BlackView


- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event {
    return NO;//表示不参与事件传递，即直接让自己不成为响应者
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSLog(@"触摸事件拦截处理.....");
}

@end
```

再运行上面的demo工程，然后任意点击屏幕的上的任何view的任何位置，都不会执行`touchesBegan`方法实现。

但是如果RootView都无法响应事件，也就更不会到黑色View的其他subviews了。

注意:

而在一般情况下，不建议将view的`pointInside:withEvent:`直接返回YES，会造成一些多余的事件传递过程。

- (3) 事件穿透。将上面例子修改成: 点击覆盖在黄色区域的白色区域，点击事件由黄色View处理，点击超出黄色区域的白色区域时由白色View处理。


再贴下之前的所有View的结构图

![](http://i1.piimg.com/4851/947fe4a50eb50d85.png)

```
- 控制器.view
	- 黑色View
		- 红色View
		- 绿色View
		- 黄色View
			- 白色View
```


首先，让白色View内的事件交给黄色View处理

```objc
@interface WhiteView()

// 直接找到黄色View
@property (nonatomic, weak) IBOutlet UIView *yellowView;

@end

@implementation WhiteView

- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {

	// 如果白色View内的触摸点，仍然处于黄色View区域内，叫给黄色View处理事件
    CGPoint yellowPoint = [self convertPoint:point toView:_yellowView];
    if ([_yellowView pointInside:yellowPoint withEvent:event]) {
        return _yellowView;
    }
    
    return [super hitTest:point withEvent:event];
}

@end
```

还需要让绿色View不响应白色View超过黄色View露在绿色View中一部分白色View的事件，来让白色View处理事件

```objc
#import "GreenView.h"

@interface GreenView()

// 直接找到白色View
@property (nonatomic, weak) IBOutlet UIView *whiteView;
@end

@implementation GreenView

- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    CGPoint whitePoint = [self convertPoint:point toView:_whiteView];
    if ([_whiteView pointInside:whitePoint withEvent:event]) {
        return _whiteView;
    }
    
    return [super hitTest:point withEvent:event];
}

@end
```

上面是 白色View属于黄色View的儿子的关系，但如果二者是`兄弟关系`的话，要实现穿透就简单多了

- (1) 因为是兄弟关系，而白色View又是在最上面，所以先从白色View开始询问

- (2) 让白色View的`hitTest:withEvent:`或者`pointInside: withEvent:`方法中，交给黄色View去处理即可

```objc
#import "WhiteView.h"

@interface WhiteView()

@property (nonatomic, weak) IBOutlet UIView *yellowView;

@end

@implementation WhiteView

- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    CGPoint yellowPoint = [self convertPoint:point toView:_yellowView];
    if ([_yellowView pointInside:yellowPoint withEvent:event]) {
        return _yellowView;
    }
    
    return [super hitTest:point withEvent:event];
}

//和如上方法二选一即可
//- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event {
//    CGPoint yellowPoint =[_yellowView convertPoint:point fromView:self];
//    if ([_yellowView pointInside:yellowPoint withEvent:event]) return NO;
//    
//    return [super pointInside:point withEvent:event];
//}

@end
```

- (3) 那么交给黄色View处理之后，黄色View是能够处理这个坐标的时间的，而已经没有其他的subviews了，所以`[黄色View hitTest:withEvent:]`返回的就是自己作为响应者

- (4) 而作为红色、绿色、黄色的父亲View >>> 黑色View，此时一旦找到黄色View是第一响应者，就直接停止询问了，直接将黄色View作为事件的第一响应者

####关于hitTest和pointInside方法的使用建议

- (1) 如果是仅仅控制一个View不参与事件传递，及不想成为事件响应者和内部的subviews也不想成为事件响应者。这种情况，只需要重写`-[UIView pointInside:withEvent]`实现返回`NO`即可

- (2) 但是如果需要在某个View中根据情况将事件`传递给另外的View对象`去处理，那么就需要重写`-[UIView hitTest:withEvent:]`方法实现了

####当事件传递结束找到了第一响应者之后，那么事件的响应是自上而下

```
找到的第一响应者View -> super view -> root view -> 控制器view -> 控制器 -> key window
```

- (1) 对于一个普通View，那么他的上一个响应者就是super view
- (2) 对于控制器.view，那么他的上一个响应者就是 ViewController
- (3) 事件响应是从上面的View开始依次询问是否响应


之前在Cell上贴一个Button，然后Button事件拦截了Cell事件，这下就很好解决了。

重写Button的`hitTest:withEvent`返回Cell对象（Button使用weak指针引用cell对象）即可完成。

之前是直接`[tableView didSelectAtIndexPath:]`完成的。

***

###使用Category Associate 扩大UI的事件响应区域

```objc
#import <UIKit/UIKit.h>

@interface UIButton (EnlargeTouchArea)

/**
 *  设置按钮上下左右的扩展响应区域
 */
- (void)setEnlargeEdgeWithTop:(CGFloat)top
                        right:(CGFloat)right
                       bottom:(CGFloat)bottom
                         left:(CGFloat)left;

@end
```

```
#import "UIButton+EnlargeTouchArea.h"
#import <objc/runtime.h>

static void *kButtonUpKey = &kButtonUpKey;
static void *kButtonLeftKey = &kButtonLeftKey;
static void *kButtonDownKey = &kButtonDownKey;
static void *kButtonRightKey = &kButtonRightKey;

@implementation UIButton (EnlargeTouchArea)

- (void)setEnlargeEdgeWithTop:(CGFloat)top right:(CGFloat)right bottom:(CGFloat)bottom left:(CGFloat)left
{
    objc_setAssociatedObject(self, kButtonUpKey, @(top), OBJC_ASSOCIATION_ASSIGN);
    objc_setAssociatedObject(self, kButtonLeftKey, @(left), OBJC_ASSOCIATION_ASSIGN);
    objc_setAssociatedObject(self, kButtonDownKey, @(bottom), OBJC_ASSOCIATION_ASSIGN);
    objc_setAssociatedObject(self, kButtonRightKey, @(right), OBJC_ASSOCIATION_ASSIGN);
}

- (CGRect) enlargedRect
{
    NSNumber* topEdge = objc_getAssociatedObject(self, &kButtonUpKey);
    NSNumber* rightEdge = objc_getAssociatedObject(self, &kButtonRightKey);
    NSNumber* bottomEdge = objc_getAssociatedObject(self, &kButtonDownKey);
    NSNumber* leftEdge = objc_getAssociatedObject(self, &kButtonLeftKey);
    
    if (topEdge && rightEdge && bottomEdge && leftEdge)
    {
        // 上下左右分别扩大响应区域
        return CGRectMake(
                          self.bounds.origin.x - leftEdge.floatValue,
                          self.bounds.origin.y - topEdge.floatValue,
                          self.bounds.size.width + leftEdge.floatValue + rightEdge.floatValue,
                          self.bounds.size.height + topEdge.floatValue + bottomEdge.floatValue
                          );
    } else {
        return self.bounds;
    }
}

- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    
    // 扩大后的响应区域
    CGRect rect = [self enlargedRect];
    
    // 如果扩大的响应区域 == 当前自身的响应区域，直接执行父类的事件处理
    if (CGRectEqualToRect(rect, self.bounds))
    {
        return [super hitTest:point withEvent:event];
    }
    
    // 扩大的响应区域 > 当前自身的响应区域
    return CGRectContainsPoint(rect, point) ? self : nil;
}

@end
```
