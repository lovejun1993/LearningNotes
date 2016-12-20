---
layout: post
title: "CoreAnimation learning notes"
date: 2015-08-11 12:45:27 +0800
comments: true
categories: 
---

###其实并不是想学习动画之类，只是想学习一些关于iOS系统图像如何渲染到最终显示的过程细节，但是找了好久也没有找到相关的点，只有CoreAnimation有一些点可以进行切入

> 此篇文字几乎所有文字全部来源于 `iOS-Core-Animation-Advanced-Techniques` 。

只是做一个读书笔记而已。

***

### `Core Animation` 其实是一个令人误解的命名

可能认为`Core Animation`只能是用来做`动画效果`的，但实际上`Core Animation`这个称呼是从一个叫`Layer Kit`这么一个不怎么和动画有关的名字`演变`而来，所以做`动画`这只是`Core Animation`特性的冰山一角。

`Core Animation`是一个`复合`引擎，它的职责就是尽可能快地`组合`屏幕上不同的可视内容:

- (1) 每一种内容可能是被独立到某一个`图层`上
- (2) 所有的`图层`存储在一个叫做`图层树`的体系之中
- (3) CoreAnimation就是负责如何`复合`这些不同的图层上的显示数据

我们通过创建的各种各样的UIView对象、以及设置给UIView对象的背景颜色、背景图片、文字、阴影、遮罩...最终都是通过`图层树`保存在其中的不同的`图层`中，最终通过CoreAnimation将这些不同的图层复合组织之后，呈现在屏幕上。（具体没有这么简单，可以先这么理解）


关于 UIView 与 CALayer 的关系:

![](http://i2.buimg.com/4851/4b19463722c9aaed.png)

- (1) CALayer布局被事件响应
	- 是提供了一个方法实现`containsPoint:`来判断触摸事件Point是否位于CALayer区域内部

	- 但是并没有实现如何参与`事件响应链`的代码，因为只有`UIView`实现了

	```objc
	// 查询是否能够响应触摸事件
	- (nullable UIView *)hitTest:(CGPoint)point withEvent:(nullable UIEvent *)event;
	```
	
	```objc
	// 触摸事件Point是否位于UIView内
	- (BOOL)pointInside:(CGPoint)point withEvent:(nullable UIEvent *)event;
	```
	
	- 而CALayer虽然也实现了上面两个类似的函数，但是并没有用来接收`UIEvent *`类型对象的参数，所以是无法参与事件传递的

	
- (2) UIView 与 CALayer 处于`平行`的级别

	- 每一个`UIview对象`默认都有一个`CALayer实对象`的属性，也就是所谓的`backing layer`（个人觉得也可以叫 root layer）

	- 当UIView对象的层级关系改变时，会触发对CALayer对象层级关系改变，保持两个层级的统一性

- (3) 实际上这些`背后关联的图层`才是真正用来在屏幕上显示和做动画的，UIView只是对图层api的一个封装，使得可以使用更简单的api操作CALayer的api

- (4) 为什么要提供一个UIKit高层封装入口来操作底层CALayer了？
	- 原因一、CALayer无法传递事件
	- 原因二、最终图层合并依赖`CoreAnimation库`，而`CoreAnimation库`是一套跨平台的api
	- 原因三、但是直接让开发者使用`CoreAnimation库`的api又太复杂、太麻烦
	- 原因四、针对iOS系统/MaxOS系统，各自开发了一套高层api来作为`CoreAnimation库`的入口
		- UIKit（UIView）、iOS的入口
		- ApplicationKit（NSView）、Mac OS的入口

		
- (5) UIView能够满足大部分界面绘制需求，为什么还需要操作CALayer了？ 因为`UIView没有暴露出来的CALayer`的如下这些函数api:

	- 阴影、圆角、边框
	- 3D变换
	- 非矩形范围
	- 透明遮罩
	- 多级非线性动画（特别复杂的动画）

	
一个简单的使用CALayer的代码，在一个UIView对象内直接添加一个CALayer图层来显示:

```objc
#import "ViewController.h"
#import <QuartzCore/QuartzCore.h>
@interface ViewController ()
@property (nonatomic, weak) IBOutlet UIView *view;
￼
@end
@implementation ViewController
- (void)viewDidLoad
{
    [super viewDidLoad];
	
	//1. 
    CALayer *blueLayer = [CALayer layer];
    blueLayer.frame = CGRectMake(50.0f, 50.0f, 100.0f, 100.0f);
    blueLayer.backgroundColor = [UIColor blueColor].CGColor;

	//2. 
    [self.view.layer addSublayer:blueLayer];
}
@end
```

这是比较简单的CALayer使用，复杂情况下远远比这复杂的多、麻烦的多。在大部分UI代码中仍然使用UIKit中的各种UIView，但是可能更需要使用`CALayer`的情况:

- (1) 开发同时可以在`Mac OS/iOS`上运行的`跨平台`应用
- (2) 使用多种`自定义CALayer的子类`，并且不想创建额外的UIView去包封装它们所有
- (3) 直接使用CALayer肯定比使用UIView轻量级一些，在一些对性能要求较高的地方
	- 如果性能要求特别高，就可能继续往底层，直接使用`OpenGL`来绘制图形了，连CALayer都可不用，但这是很复杂的

	
YYKit作者也在界面流畅度优化中提到过，在一些不具备事件响应的UI代码编写时，完全可以通过使用CALayer代替UIView，来获取一定的性能提升。


****

###寄宿图: CALayer对象.contens属性值

CALayer 有一个属性叫做`contents`，这个属性的类型被定义为id，意味着它可以是任何类型的对象。在这种情况下，你可以给contents属性赋任何值，你的app仍然能够编译通过。但是，在实践中，如果你给contents赋的不是CGImage，那么你得到的图层将是空白的。

而被设置到这个`contents`属性中的`CGImage 实例`就称为`寄宿图`。

CALayer主要使用的一些属性:

- (1) contents

	- 这个属性的类型被定义为id，意味着它可以是任何类型的对象
	- 在这种情况下，你可以给contents属性赋任何值，你的app仍然能够编译通过
	- 但是，在实践中，如果你给contents赋的不是`CGImage`，那么你得到的图层将是空白的

设置layer的寄宿图

```objc
@implementation ViewController

- (void)viewDidLoad
{
  [super viewDidLoad]; //load an image
  UIImage *image = [UIImage imageNamed:@"Snowman.png"];

  //add it directly to our view's layer
  self.layerView.layer.contents = (__bridge id)image.CGImage;
}
@end
```


- (2) contentGravity，就是UIView中的contentMode属性对应的

```objc
view.contentMode = UIViewContentModeScaleAspectFit;
```

而对应CALayer的设置

```objc
self.layerView.layer.contentsGravity = kCAGravityResizeAspect;
```

- (3) contentsScale，如果contentsScale设置为1.0，将会以每个点1个像素绘制图片，如果设置为2.0，则会以每个点2个像素绘制图片，这就是我们熟知的Retina屏幕

当用代码的方式来处理寄宿图的时候，一定要记住要手动的设置图层的contentsScale属性，否则，你的图片在Retina设备上就显示得不正确啦

```objc
layer.contentsScale = [UIScreen mainScreen].scale;
```

- (4) maskToBounds，是否绘制超出layer范围的多余部分

```
UIView有一个叫做 clipsToBounds 的属性可以用来决定是否显示超出边界的内容
```

- (5) contentsRect，允许我们在layer内部区域内，显示寄宿图的一个`子域`

	- 和bounds，frame不同，contentsRect不是按`点`来计算的，它使用了`单位坐标`，单位坐标指定在0到1之间，是一个相对值（像素和点就是绝对值）

	- iOS使用了以下的`坐标系统`:
		- 点
			- 在iOS和Mac OS中最常见的坐标体系
			- 点就像是`虚拟的像素`，也被称作逻辑像素
			- 在标准设备上，一个点就是一个像素（非retina屏幕）
			- 但是在Retina设备上:
				- 6plus之前，`一个点` == `2*2` 4个像素
				- 6plus之后，`一个点` == `3*3` 9个像素
			- iOS用点作为屏幕的坐标测算体系就是为了在Retina设备和普通设备上能有一致的视觉效果

		- 像素
			- `物理像素`坐标并`不会`用来`屏幕布局`，但是仍然与图片有相对关系
			- `UIImage` 是一个屏幕分辨率解决方案，所以使用`点`来度量大小，而不是像素
			- 但是一些`底层的图片`（如`CGImage`）绘制时就会使用`像素`
			- 就是说如果使用像素，在任何屏幕都是这么多的像素。但如果使用点，那么就会根据屏幕的scale变化像素
			
		- 单位
			- 个人觉得就是一个大小的`比例值` 
			- 只要比例值固定，那么当大小改变的时候，也不需要再次调整比例值
			- 单位坐标在`OpenGL`这种`纹理坐标系统`中用得很多，`CoreAnimation`中也用到了单位坐标


默认的 `layer.contentsRect == {0, 0, 1, 1}`，这意味着整个`寄宿图`默认都是可见的，如果我们指定一个小一点的矩形，`layer.contentsRect == {0, 0, 0.5, 0.5}`，图片就会被裁剪:

![](http://i1.piimg.com/567571/4f4664c1c3f9126f.png)

事实上给contentsRect设置一个负数的原点或是大于{1, 1}的尺寸也是可以的。这种情况下，最外面的像素会被拉伸以填充剩下的区域。

contentsRect在app中最有趣的地方在于一个叫做image sprites（图片拼合）的用法。图片拼合后可以打包整合到一张大图上一次性载入。相比多次载入不同的图片，这样做能够带来很多方面的好处：内存使用，载入时间，渲染性能等等。比如2D游戏引擎入Cocos2D使用了拼合技术，它使用OpenGL来显示图片。

假设有如下一张大的合成图:

![](http://i4.piimg.com/567571/d1b9f133b2c246ff.png)

然后程序中加载这个大图，然后截取出其中的4个小图进行显示，通过layer.contentsRect属性截取:

```objc
@interface ViewController ()
@property (nonatomic, weak) IBOutlet UIView *coneView;
@property (nonatomic, weak) IBOutlet UIView *shipView;
@property (nonatomic, weak) IBOutlet UIView *iglooView;
@property (nonatomic, weak) IBOutlet UIView *anchorView;
@end

@implementation ViewController

- (void)addSpriteImage:(UIImage *)image withContentRect:(CGRect)rect ￼toLayer:(CALayer *)layer //set image
{
  layer.contents = (__bridge id)image.CGImage;

  //scale contents to fit
  layer.contentsGravity = kCAGravityResizeAspect;

  //set contentsRect
  layer.contentsRect = rect;
}

- (void)viewDidLoad {
	[super viewDidLoad]; 
  
	// 加载大图片
  	UIImage *image = [UIImage imageNamed:@"Sprites.png"];
  
	// layer1 截取大图的 左上 1/4 显示
  	[self addSpriteImage:image withContentRect:CGRectMake(0, 0, 0.5, 0.5) toLayer:self.iglooView.layer];
  
	// layer2 截取大图的 右上 1/4 显示
	[self addSpriteImage:image withContentRect:CGRectMake(0.5, 0, 0.5, 0.5) toLayer:self.coneView.layer];

	// layer3 截取大图的 左下 1/4 显示
  [self addSpriteImage:image withContentRect:CGRectMake(0, 0.5, 0.5, 0.5) toLayer:self.anchorView.layer];

	// layer4 截取大图的 右下 1/4 显示
	[self addSpriteImage:image withContentRect:CGRectMake(0.5, 0.5, 0.5, 0.5) toLayer:self.shipView.layer];
}
@end
```

最后运行效果图

![](http://i4.piimg.com/567571/08852a42a490e3c9.png)

如上就是一个大图，然后1/4尺寸就是一个单独的小图。每个layer首先都是处理整个大图，只是通过`contentsRect`比例值，来截图`contens`寄宿图中某一部分进行渲染显示。

Mac上有一些商业软件可以为你自动拼合图片，这些工具自动生成一个包含拼合后的`坐标`的XML或者plist文件，拼合图片的使用大大简化。这个文件可以和图片一同载入，并给每个拼合的图层设置contentsRect，这样开发者就不用手动写代码来摆放位置了。

有一个iOS开源库叫做`LayerSprites`可以读取拼合图，然后使用CoreAnimation显示。

- (6) contentsCenter，是一个`CGRect`而并不是一个中间点，它定义了一个固定的边框和一个在图层上`可拉伸的区域`。


默认情况下，contentsCenter是 `{0, 0, 1, 1}`，这意味着如果大小（由conttensGravity决定）改变了,那么寄宿图将会均匀地拉伸开。但是如果我们增加原点的值并减小尺寸。

效果和`-[UIImage resizableImageWithCapInsets:]` 方法效果非常类似，只是它可以运用到任何寄宿图，甚至包括在Core Graphics运行时绘制的图形。

####使用Core Graphics绘制layer.contents，而不是通过设置一个CGImage实例

- (1) 首先，通过继承UIView一个子类，重写`-[UIView drawRect:]`方法来自定义绘制图形

- (2) `-drawRect:` 方法`没有`默认的实现

- (3) 如果检测到UIView的`-drawRect:` 方法被重写了，就会为UIView对象分配一个`寄宿图`，这个寄宿图的`像素尺寸 == UIView的大小乘以 contentsScale`

- (4) 如果不需要寄宿图，那就`不要重写`这个方法了。这会造成CPU资源和内存的浪费，这也是为什么苹果建议:
	- 如果没有自定义绘制的任务，就不要在UIView子类中写一个`空的 -drawRect:`方法实现

- (5) 当View对象在屏幕上出现的时候， `-drawRect:`方法实现 就会被自动调用

- (6) `-drawRect:`方法实现中，使用`Core Graphics`去绘制一个`寄宿图`，然后内容就会被`缓存`起来，一直到它需要被更新然后进行`重新绘制`

- (7) 通常情况下回触发更新重新绘制的情况:	
	- 开发者调用了`-[UIView setNeedsDisplay]`方法实现
	- 一些视图类型会被自动重绘，如`bounds`属性

- (8) 底层都是由`CALayer`安排了`重绘`工作和`保存`了因此产生的`图片`

####CALayerDelegate，通过一个NSObject分类提供

```objc
@interface NSObject (CALayerDelegate)

/* If defined, called by the default implementation of the -display
 * method, in which case it should implement the entire display
 * process (typically by setting the `contents' property). */

- (void)displayLayer:(CALayer *)layer;

/* If defined, called by the default implementation of -drawInContext: */

- (void)drawLayer:(CALayer *)layer inContext:(CGContextRef)ctx;

/* Called by the default -layoutSublayers implementation before the layout
 * manager is checked. Note that if the delegate method is invoked, the
 * layout manager will be ignored. */

- (void)layoutSublayersOfLayer:(CALayer *)layer;

/* If defined, called by the default implementation of the
 * -actionForKey: method. Should return an object implementating the
 * CAAction protocol. May return 'nil' if the delegate doesn't specify
 * a behavior for the current event. Returning the null object (i.e.
 * '[NSNull null]') explicitly forces no further search. (I.e. the
 * +defaultActionForKey: method will not be called.) */

- (nullable id<CAAction>)actionForLayer:(CALayer *)layer forKey:(NSString *)event;

@end
```

 当需要被重绘时，CALayer会请求它的代理给他一个寄宿图来显示。它通过调用下面这个方法做到的:

```objc
- (void)displayLayer:(CALayer *)layer;
```

比如:

```objc
@implementation XXXView: UIView 

- (void)displayLayer:(CALayer *)layer {
	layer.contens = 一个CGImage实例;
}

@end
```

如果代理没有实现 `-displayLayer:`方法，CALayer就会转而尝试调用下面这个方法：

```objc
- (void)drawLayer:(CALayer *)layer inContext:(CGContextRef)ctx;
```

在调用这个方法之前系统自动做了两件事:

- (1) CALayer创建了一个合适尺寸的`空寄宿图`（尺寸由bounds和contentsScale决定）
- (2) 传入一个Core Graphics的绘制上下文环境，为绘制寄宿图做准备，他作为ctx参数传入

下面是一个简单的CALayer通知delegate（UIView对象）进行绘制一个寄宿图（CALayer》contens属性值填充）

```objc
@implementation ViewController

- (void)viewDidLoad
{
  [super viewDidLoad];
  ￼
  //create sublayer
  CALayer *blueLayer = [CALayer layer];
  blueLayer.frame = CGRectMake(50.0f, 50.0f, 100.0f, 100.0f);
  blueLayer.backgroundColor = [UIColor blueColor].CGColor;

  //set controller as layer delegate
  blueLayer.delegate = self;

  //ensure that layer backing image uses correct scale
  blueLayer.contentsScale = [UIScreen mainScreen].scale; //add layer to our view
  [self.layerView.layer addSublayer:blueLayer];

  //force layer to redraw
  [blueLayer display];
}

- (void)drawLayer:(CALayer *)layer inContext:(CGContextRef)ctx
{
  //draw a thick red circle
  CGContextSetLineWidth(ctx, 10.0f); 
  CGContextSetStrokeColorWithColor(ctx, [UIColor redColor].CGColor);
  CGContextStrokeEllipseInRect(ctx, layer.bounds);
}
@end
```

可以看到在`viewDidLoad`方法实现中最后一句代码`[blueLayer display];`，即对blueLayer上显式地调用了`display`方法实现。为什么要这么做了，好像我们对UIView的显示时并没有主动调用过display什么的函数.

- (1) CALayer不同于UIView，当CALayer显示在屏幕上时，CALayer`不会自动重绘`它的内容

- (2) 必须开发者手动的调用`display`方法实现，告诉系统开始绘制CALayer的寄宿图


尽管我们没有用CALayer对象的`masksToBounds`属性，但是绘制的那个圆仍然沿边界被`裁剪`了。这是因为当你使用`CALayerDelegate`回到方法实现中，进行绘制寄宿图的时候，并没有`对超出边界外的内容`提供绘制支持。

通过`layer.contens = 图像;`方式设置寄宿图，会由系统提供`对超出边界外的内容`提供绘制支持。

而UIView对象创建时会自动完成:

- (1) 会自动创建内部的CALayer主图层对象
- (2) 然后自动地把CALayer对象的delegate设置为它自己
- (3) 并提供了一个`-displayLayer:`的默认实现

通常情况下，我们需要完成一个路径不规则、较复杂形状路线的图形绘制时，通常做法是实现`-[UIView drawRect:]`方法实现即可，并使用CoreGraphics进行绘制。至于CALayerDelegate的那些方法，我们都不用去管，UIView已经做了处理了。

****

###CALayer图层几何学

在UIView布局中，有三个重要的属性:

- (1) frame
- (2) bounds
- (3) center

而在CALayer中，与上面三个同样，只是center有点区别:

- (1) frame
- (2) bounds
- (3) `position`

虽然CALayer用了`position`，UIView用了`center`，但是他们都代表一样的作用。


- frame

frame代表了图层/视图的`外部`的坐标（也就是在父图层上占据的空间）

- bounds

bounds是`内部`坐标。比如，	`{0, 0}通常是图层的左上角.

- center、position

center和position都代表了 `相对于 父图层`的`anchorPoint`所在的位置。anchorPoint可以把它想成`图层的中心点`就好了。


UIView的frame，bounds和center属性仅仅只是操作UIView内部的CALayer的frame的入口。

对于视图或者图层来说，frame并不是一个非常清晰的属性，它其实是一个`虚拟`属性，是根据`bounds、position、transform`计算而来。所以当其中任何一个值发生改变，frame都会变化。相反，改变frame的值同样会影响到他们当中的值。

当对图层做变换的时候，比如旋转或者缩放，frame实际上代表了覆盖在图层旋转之后的整个轴对齐的矩形区域，也就是说frame的宽高可能和bounds的宽高不再一致了（会变大，也可能变小）

- (1) 原始视图的frame布局

![](http://i4.piimg.com/567571/dd39db9a0513049a.png)

- (2) 对原始视图进行旋转之后的布局

![](http://i4.piimg.com/567571/1ae6bd5d39fe0959.png)

可以看到，此时相对原始视图的frame，旋转后的frame会变的大一些了。

####anchorPoint 锚点 >>> 可以认为anchorPoint是用来`移动图层`的把柄

- (1) 默认来说，anchorPoint位于CALayer的`中心点`，所以CALayer的将会以这个点为中心放置
	- `layer.anchorPoint == CGPointMake(0.5, 0.5);`
	
- (2) anchorPoint属性并没有被UIView接口暴露出来

比如，如下例子将`layer.anchorPoint = CGPointMake(0, 0);`之后的变化:

![](http://wiki.jikexueyuan.com/project/ios-core-animation/images/3.3.jpeg)
 
可以看到:

- (1) 宽度和高度不会发生变化，只是x和y会发生变化
- (2) 寄宿图的单位坐标(0,0)出现在了layer的中心点(0.5, 0.5)的位置

总而言之，当改变了`anchorPoint`之后，`position`属性保持固定的值并没有发生改变，但是`frame`却移动了。

CALayer并不关心任何响应链事件，所以不能直接处理触摸事件或者手势。但是它有一系列的方法帮你处理事件：`-containsPoint:`和`-hitTest:`。

当使用视图的时候，可以充分利用UIView类接口暴露出来的UIViewAutoresizingMask和NSLayoutConstraintAPI，但如果想随意控制CALayer的布局，就需要手工操作。最简单的方法就是使用CALayerDelegate如下函数：

```objc
- (void)layoutSublayersOfLayer:(CALayer *)layer;
```

当图层的`bounds`发生改变，或者图层的`-setNeedsLayout`方法被调用的时候，这个函数将会被执行，可以手动地重新摆放或者重新调整子图层的大小。

***

###专用图层、Core Animation图层不仅仅能作用于图片和颜色而已，提供了一些针对性使用的CALayer子类


####通过CAShapeLayer、快速高效的绘制指定路径的图像

前面基本上都是通过`位图bitmap`来绘制layer中的图形路径，而`CAShapeLayer`是一个通过`矢量图形`绘制寄宿图的CALayer子类。

常见的在layer中绘制图形的方法:

- (1) 使用`Core Graphics`直接向`原始的CALyer`的内容中绘制一个`路径`，然后填充、描边....

- (2) 用`CGPath`来定义想要绘制的图形，最后`CAShapeLayer`就自动渲染出来了

相比以上两种在layer进行绘制图形的方法，使用CAShapeLayer有以下一些优点:

- (1) 渲染快速

CAShapeLayer使用了硬件加速，绘制同一图形会比用Core Graphics快很多

- (2) 高效使用内存

一个CAShapeLayer不需要像普通CALayer一样创建一个寄宿图形，所以无论有多大，都不会占用太多的内存

- (3) 不会被图层边界剪裁掉

一个CAShapeLayer可以在`边界之外`绘制。`图层路径`不会像在使用`Core Graphics的普通CALayer`绘制时发生被剪裁掉的情况

- (4) 不会出现像素化

当你给CAShapeLayer做`3D变换`时，它不像一个`有寄宿图的普通CALayer`一样变得`像素化`

在使用CAShapeLayer绘图时，以前要使用`CGPath`计算绘图路径，因为是纯c的函数api。有一个OC版本的api叫做`UIBezierPath`，一样的效果，操作起来更简单。

比如，如下一段简单的先使用UIBezierPath计算得到一个图形路径（不不一定是封闭的路径），然后再使用CAShapeLayer渲染这个路径的示例代码:

```objc
@implementation ViewController

- (void)viewDidLoad
{
	[super viewDidLoad];

	//1. 创建一个图形路径
	UIBezierPath *path = [[UIBezierPath alloc] init];
	
	//2. 移动到路径起始点
	[path moveToPoint:CGPointMake(175, 100)];
	￼
	//3. 添加移动路径
	[path addArcWithCenter:CGPointMake(150, 100) radius:25 startAngle:0 endAngle:2*M_PI clockwise:YES];
	[path moveToPoint:CGPointMake(150, 125)];
	[path addLineToPoint:CGPointMake(150, 175)];
	[path addLineToPoint:CGPointMake(125, 225)];
	[path moveToPoint:CGPointMake(150, 175)];
	[path addLineToPoint:CGPointMake(175, 225)];
	[path moveToPoint:CGPointMake(100, 150)];
	[path addLineToPoint:CGPointMake(200, 150)];
	
	//4. 创建一个CAShapeLayer对象
	CAShapeLayer *shapeLayer = [CAShapeLayer layer];
	
	//5. 设置CAShapeLayer对象的绘制图形时需要的属性值
	shapeLayer.strokeColor = [UIColor redColor].CGColor;
	shapeLayer.fillColor = [UIColor clearColor].CGColor;
	shapeLayer.lineWidth = 5;
	shapeLayer.lineJoin = kCALineJoinRound;
	shapeLayer.lineCap = kCALineCapRound;
	
	//6. 设置CAShapeLayer对象要渲染的图形路径
	shapeLayer.path = path.CGPath;

	//7. 将创建的CAShapeLayer对象，注册给一个View对象，作为View对象的寄宿图层（root layer），对layer进行渲染绘制图形
	[self.containerView.layer addSublayer:shapeLayer];
}
@end
```

####CATextLayer、底层调用CoreText引擎进行渲染文字，渲染速度非常快

Core Animation提供了一个CALayer的子类CATextLayer，它以`图层`的形式包含了UILabel几乎所有的绘制特性，并且额外提供了一些新的特性。

`CATextLayer也要比UILabel渲染得快得多`:

- (1) `iOS 6及之前`的版本，`UILabel`其实是通过`WebKit来实现绘制`的，这样就造成了当有很多文字的时候就会有极大的性能压力

- (2) `CATextLayer`是使用了`CoreText`作为排版引擎，所以渲染得非常快

使用CATextLayer来显示一些文字的简单示例代码:

```objc
@implementation ViewController

- (void)viewDidLoad {
  [super viewDidLoad];

	//1. 创建一个CATextLayer对象，并添加到root layer中
	CATextLayer *textLayer = [CATextLayer layer];
	textLayer.frame = self.labelView.bounds;
	[self.labelView.layer addSublayer:textLayer];
	
	//2. 设置CATextLayer对象的绘制文字时的一些属性值
	textLayer.foregroundColor = [UIColor blackColor].CGColor;
	textLayer.alignmentMode = kCAAlignmentJustified;
	textLayer.wrapped = YES;
	
	//3. 创建一个UIKit字体，并转成CF版本字体，设置给CATextLayer对象
	UIFont *font = [UIFont systemFontOfSize:15];
	CFStringRef fontName = (__bridge CFStringRef)font.fontName;
	CGFontRef fontRef = CGFontCreateWithFontName(fontName);
	textLayer.font = fontRef;
	textLayer.fontSize = font.pointSize;
	CGFontRelease(fontRef);
	
	//4. 要绘制的文字
	NSString *text = @"Lorem ipsum dolor sit amet, consectetur adipiscing \ elit. Quisque massa arcu, eleifend vel varius in, facilisis pulvinar \ leo. Nunc quis nunc at mauris pharetra condimentum ut ac neque. Nunc elementum, libero ut porttitor dictum, diam odio congue lacus, vel \ fringilla sapien diam at purus. Etiam suscipit pretium nunc sit amet \ lobortis";
	
	//5. 将绘制的文字设置给CATextLayer对象
	textLayer.string = text;
}
@end
```

上面代码会有一个问题: 绘制出来的文字最终显示的图片会`像素化`，即在非retina屏显示是正常，但是在retina屏上显示就会出现边缘模糊的效果。

这是因为并没有`以Retina`的方式渲染，可以通多`layer.contentScale`属性，用来决定`图层内容`应该`以怎样的分辨率`来渲染。contentsScale并不关心屏幕的拉伸因素而总是默认为1.0。如果我们想以Retina的质量来显示文字，我们就得手动地设置CATextLayer的contentsScale属性，如下：

```objc
textLayer.contentsScale = [UIScreen mainScreen].scale;
```

CATextLayer的string属性并不是你想象的NSString类型，而是id类型。这样你既可以用`NSString（普文文本内容）`也可以用`NSAttributedString（带有各种富文本属性的文本内容）`来指定文本了（注意，NSAttributedString并不是NSString的子类）。

通过`NSAttributedString（带有各种富文本属性的文本内容）`来指定富文本内容，可以避免直接使用`Core Text`的一些纯c语言的函数api进行富文本属性设置。

下面是一段CoreFoundation版本操作富文本属性字符串的示例代码:

```objc
@implementation ViewController

- (void)viewDidLoad
{
  [super viewDidLoad];

  //create a text layer
  CATextLayer *textLayer = [CATextLayer layer];
  textLayer.frame = self.labelView.bounds;
  textLayer.contentsScale = [UIScreen mainScreen].scale;
  [self.labelView.layer addSublayer:textLayer];

  //set text attributes
  textLayer.alignmentMode = kCAAlignmentJustified;
  textLayer.wrapped = YES;

  //choose a font
  UIFont *font = [UIFont systemFontOfSize:15];

  //choose some text
  NSString *text = @"Lorem ipsum dolor sit amet, consectetur adipiscing \ elit. Quisque massa arcu, eleifend vel varius in, facilisis pulvinar \ leo. Nunc quis nunc at mauris pharetra condimentum ut ac neque. Nunc \ elementum, libero ut porttitor dictum, diam odio congue lacus, vel \ fringilla sapien diam at purus. Etiam suscipit pretium nunc sit amet \ lobortis";
  ￼
  //create attributed string
  NSMutableAttributedString *string = nil;
  string = [[NSMutableAttributedString alloc] initWithString:text];

  //convert UIFont to a CTFont
  CFStringRef fontName = (__bridge CFStringRef)font.fontName;
  CGFloat fontSize = font.pointSize;
  CTFontRef fontRef = CTFontCreateWithName(fontName, fontSize, NULL);

  //set text attributes
  NSDictionary *attribs = @{
    (__bridge id)kCTForegroundColorAttributeName:(__bridge id)[UIColor blackColor].CGColor,
    (__bridge id)kCTFontAttributeName: (__bridge id)fontRef
  };

  [string setAttributes:attribs range:NSMakeRange(0, [text length])];
  attribs = @{
    (__bridge id)kCTForegroundColorAttributeName: (__bridge id)[UIColor redColor].CGColor,
    (__bridge id)kCTUnderlineStyleAttributeName: @(kCTUnderlineStyleSingle),
    (__bridge id)kCTFontAttributeName: (__bridge id)fontRef
  };
  [string setAttributes:attribs range:NSMakeRange(6, 5)];

  //release the CTFont we created earlier
  CFRelease(fontRef);

  //set layer text
  textLayer.string = string;
}
@end
```

每一个UIView都是寄宿在一个CALayer的示例上。这个图层是由视图自动创建和管理的，那我们可以用别的图层类型替代它么？一旦被创建，我们就无法代替这个图层了。但是如果我们继承了UIView，那我们就可以重写+layerClass方法使得在创建的时候能返回一个不同的图层子类。UIView会在初始化的时候调用+layerClass方法，然后用它的返回类型来创建宿主图层。

继承UILabel一个子类，然后内部使用CATextLayer实现文本内容的绘制:

```objc
#import <QuartzCore/QuartzCore.h>

@implementation LayerLabel
+ (Class)layerClass
{
  //this makes our label create a CATextLayer //instead of a regular CALayer for its backing layer
  return [CATextLayer class];
}

- (CATextLayer *)textLayer
{
  return (CATextLayer *)self.layer;
}

- (void)setUp
{
  //set defaults from UILabel settings
  self.text = self.text;
  self.textColor = self.textColor;
  self.font = self.font;

  //we should really derive these from the UILabel settings too
  //but that's complicated, so for now we'll just hard-code them
  [self textLayer].alignmentMode = kCAAlignmentJustified;
  ￼
  [self textLayer].wrapped = YES;
  [self.layer display];
}

- (id)initWithFrame:(CGRect)frame
{
  //called when creating label programmatically
  if (self = [super initWithFrame:frame]) {
    [self setUp];
  }
  return self;
}

- (void)awakeFromNib
{
  //called when creating label using Interface Builder
  [self setUp];
}

- (void)setText:(NSString *)text
{
  super.text = text;
  //set layer text
  [self textLayer].string = text;
}

- (void)setTextColor:(UIColor *)textColor
{
  super.textColor = textColor;
  //set layer text color
  [self textLayer].foregroundColor = textColor.CGColor;
}

- (void)setFont:(UIFont *)font
{
  super.font = font;
  //set layer font
  CFStringRef fontName = (__bridge CFStringRef)font.fontName;
  CGFontRef fontRef = CGFontCreateWithFontName(fontName);
  [self textLayer].font = fontRef;
  [self textLayer].fontSize = font.pointSize;
  ￼
  CGFontRelease(fontRef);
}
@end
```

如果你运行代码，你会发现文本并没有像素化，而我们也没有设置contentsScale属性。把`CATextLayer`作为`宿主图层`（root layer）的另一好处就是`自动设置了contentsScale属性`。


####CATransformLayer、用于快速构造一个层级的3D结构

```objc
@interface ViewController ()

@property (nonatomic, weak) IBOutlet UIView *containerView;

@end

@implementation ViewController

/**
 *  构建立方体中的某一个面
 */
- (CALayer *)faceWithTransform:(CATransform3D)transform
{
    //create cube face layer
    CALayer *face = [CALayer layer];
    face.frame = CGRectMake(-50, -50, 100, 100);
    
    //apply a random color
    CGFloat red = (rand() / (double)INT_MAX);
    CGFloat green = (rand() / (double)INT_MAX);
    CGFloat blue = (rand() / (double)INT_MAX);
    face.backgroundColor = [UIColor colorWithRed:red green:green blue:blue alpha:1.0].CGColor;
    
    ￼//apply the transform and return
    face.transform = transform;
    return face;
}

- (CALayer *)cubeWithTransform:(CATransform3D)transform
{
  // 创建一个3D形变结合体layer
  CATransformLayer *cube = [CATransformLayer layer];

  // 立方体第一个面
  CATransform3D ct = CATransform3DMakeTranslation(0, 0, 50);
  [cube addSublayer:[self faceWithTransform:ct]];

  //add cube face 2
  ct = CATransform3DMakeTranslation(50, 0, 0);
  ct = CATransform3DRotate(ct, M_PI_2, 0, 1, 0);
  [cube addSublayer:[self faceWithTransform:ct]];

  //add cube face 3
  ct = CATransform3DMakeTranslation(0, -50, 0);
  ct = CATransform3DRotate(ct, M_PI_2, 1, 0, 0);
  [cube addSublayer:[self faceWithTransform:ct]];

  //add cube face 4
  ct = CATransform3DMakeTranslation(0, 50, 0);
  ct = CATransform3DRotate(ct, -M_PI_2, 1, 0, 0);
  [cube addSublayer:[self faceWithTransform:ct]];

  //add cube face 5
  ct = CATransform3DMakeTranslation(-50, 0, 0);
  ct = CATransform3DRotate(ct, -M_PI_2, 0, 1, 0);
  [cube addSublayer:[self faceWithTransform:ct]];

  //add cube face 6
  ct = CATransform3DMakeTranslation(0, 0, -50);
  ct = CATransform3DRotate(ct, M_PI, 0, 1, 0);
  [cube addSublayer:[self faceWithTransform:ct]];

  //center the cube layer within the container
  CGSize containerSize = self.containerView.bounds.size;
  cube.position = CGPointMake(containerSize.width / 2.0, containerSize.height / 2.0);

  //apply the transform and return
  cube.transform = transform;
  return cube;
}

- (void)viewDidLoad
{￼
  [super viewDidLoad];

  //set up the perspective transform
  CATransform3D pt = CATransform3DIdentity;
  pt.m34 = -1.0 / 500.0;
  self.containerView.layer.sublayerTransform = pt;

  // 第一个立方体，只有2个面
  CATransform3D c1t = CATransform3DIdentity;
  c1t = CATransform3DTranslate(c1t, -100, 0, 0);// 
  CALayer *cube1 = [self cubeWithTransform:c1t];
  [self.containerView.layer addSublayer:cube1];

  // 第二个立方体，有三个面
  CATransform3D c2t = CATransform3DIdentity;
  c2t = CATransform3DTranslate(c2t, 100, 0, 0);
  c2t = CATransform3DRotate(c2t, -M_PI_4, 1, 0, 0);
  c2t = CATransform3DRotate(c2t, -M_PI_4, 0, 1, 0);
  CALayer *cube2 = [self cubeWithTransform:c2t];
  [self.containerView.layer addSublayer:cube2];
}
@end
```

####CAGradientLayer、快速绘制颜色平滑渐变

```objc
@implementation ViewController

- (void)viewDidLoad
{
	[super viewDidLoad];
	
	//1. 
	CAGradientLayer *gradientLayer = [CAGradientLayer layer];
	gradientLayer.frame = self.containerView.bounds;
	[self.containerView.layer addSublayer:gradientLayer];
	  
	//2. 
	gradientLayer.colors = @[(__bridge id)[UIColor redColor].CGColor, (__bridge id)[UIColor blueColor].CGColor];
	  
	//3. 绘制整个layer区域，颜色都渐变
	gradientLayer.startPoint = CGPointMake(0, 0);
	gradientLayer.endPoint = CGPointMake(1, 1);
}
@end
```

上面是对于一种颜色进行渐变，还有同时对多种颜色进行渐变，即`多重渐变`:

```objc
- (void)viewDidLoad {
    [super viewDidLoad];

    //create gradient layer and add it to our container view
    CAGradientLayer *gradientLayer = [CAGradientLayer layer];
    gradientLayer.frame = self.containerView.bounds;
    [self.containerView.layer addSublayer:gradientLayer];

    //set gradient colors
    gradientLayer.colors = @[(__bridge id)[UIColor redColor].CGColor, (__bridge id) [UIColor yellowColor].CGColor, (__bridge id)[UIColor greenColor].CGColor];

    //set locations
    gradientLayer.locations = @[@0.0, @0.25, @0.5];

    //set gradient start and end points
    gradientLayer.startPoint = CGPointMake(0, 0);
    gradientLayer.endPoint = CGPointMake(1, 1);
}
```

####还有一些系统封装好的layer

- (1) CAReplicatorLayer、高效生成许多相似的图层
- (2) CAScrollLayer、可以简单的代替UIScrollView
- (3) CAEmitterLayer、高性能的粒子引擎，被用来创建实时例子动画如：烟雾，火，雨等等这些效果

#### CATiledLayer、当绘制一个超过了OpenGL对顶的单个图最大的纹理尺寸（`通常是2048*2048，或4096*4096`)超大图时，会将大图分解成小片然后将他们单独按需载入

- (1) 首先将大图预先裁剪成小图片

虽然可以通过`代码`在`程序运行时`来完成这件事情，但是如果在`运行时`读入整个图片并裁切，那CATiledLayer这些所有的性能优点就损失殆尽了。所以必须是，程序启动之前就要把大图片进行裁剪小图片完毕。

可以参考很多开源的使用`CATiledLayer`进行大图裁剪的库代码。

比如，将一个大图裁剪成如下这样一些小图:

```
Snowman_00_00.jpg
Snowman_00_01.jpg
Snowman_00_02.jpg
...
Snowman_07_07.jpg
```

- (2) 程序代码加载运行`按照`需求来加载对应的`小图`

结合使用UIScrollView

```objc
#import "ViewController.h"
#import <QuartzCore/QuartzCore.h>

@interface ViewController ()

@property (nonatomic, weak) IBOutlet UIScrollView *scrollView;

@end

@implementation ViewController

- (void)viewDidLoad
{
    [super viewDidLoad];
    
    //add the tiled layer
    CATiledLayer *tileLayer = [CATiledLayer layer];￼
    tileLayer.frame = CGRectMake(0, 0, 2048, 2048);
    
    // 设置layer的代理
    tileLayer.delegate = self; 
    
    // 将layer添加到scrollView的图层中
    [self.scrollView.layer addSublayer:tileLayer];

    //configure the scroll view
    self.scrollView.contentSize = tileLayer.frame.size;

    //draw layer
    [tileLayer setNeedsDisplay];
}

- (void)drawLayer:(CATiledLayer *)layer inContext:(CGContextRef)ctx
{
    //determine tile coordinate
    CGRect bounds = CGContextGetClipBoundingBox(ctx);
	CGFloat scale = [UIScreen mainScreen].scale;
	NSInteger x = floor(bounds.origin.x / layer.tileSize.width * scale);
	NSInteger y = floor(bounds.origin.y / layer.tileSize.height * scale);

    //load tile image
    NSString *imageName = [NSString stringWithFormat: @"Snowman_%02i_%02i", x, y];
    NSString *imagePath = [[NSBundle mainBundle] pathForResource:imageName ofType:@"jpg"];
    UIImage *tileImage = [UIImage imageWithContentsOfFile:imagePath];

    //draw tile
    UIGraphicsPushContext(ctx);
    [tileImage drawInRect:bounds];
    UIGraphicsPopContext();
}
@end
```

CATiledLayer（不同于大部分的UIKit和Core Animation方法）支持`多线程绘制`，`-drawLayer:inContext:`方法可以在`多个线程`中同时地并发调用，所以请小心谨慎地确保你在这个方法中实现的绘制代码是线程安全的，通过使用CoreGraphics库代码绘制即可保证线程安全。

####CAEAGLLayer、当iOS要处理高性能图形绘制，必要时就是OpenGL。相比Core Animation和UIkit框架，它不可思议地复杂。

关于使用的代码就不贴了，网络上很多，代码略显麻烦。

注意的是，在一个真正的`OpenGL应用`中:

- (1) 我们可能会用`NSTimer`或`CADisplayLink`周期性地`每秒钟`调用`-drawRrame`方法执行`60次`

- (2) 也就是说，一秒内，进行60次图像`重新绘制`
	- 每秒显示60帧图形
	- `Frames Per Second` >>> FPS 衡量界面是否卡顿的标准
	
- (3) 同时会将几何的`图形生成`和`绘制`分开，以便不会每次都重新生成三角形的顶点（这样也可以让我们绘制其他的一些东西而不是一个三角形而已）

####AVPlayerLayer、iOS上播放视频

```objc
#import "ViewController.h"
#import <QuartzCore/QuartzCore.h>
#import <AVFoundation/AVFoundation.h>

@interface ViewController ()

@property (nonatomic, weak) IBOutlet UIView *containerView; @end

@implementation ViewController

- (void)viewDidLoad
{
    [super viewDidLoad];
    //get video URL
    NSURL *URL = [[NSBundle mainBundle] URLForResource:@"Ship" withExtension:@"mp4"];

    //create player and player layer
    AVPlayer *player = [AVPlayer playerWithURL:URL];
    AVPlayerLayer *playerLayer = [AVPlayerLayer playerLayerWithPlayer:player];

    //set player layer frame and attach it to our view
    playerLayer.frame = self.containerView.bounds;
    [self.containerView.layer addSublayer:playerLayer];

    //play the video
    [player play];
}
@end
```

***

###性能调优

####CoreAnimation是构建在OpenGL和CoreGraphics的基础之上，也就意味着图像处理时: GPU VS CPU

最终绘制图形有两种类型的硬件:

- (1) GPU（中央处理器）、OpenGL库
- (2) CPU（图形处理器）、CoreGraphics库

由于历史原因，我们可以说:

- (1) CPU所做的工作都在`软件`层面
- (2) GPU在`硬件`层面

我们可以用软件（`使用CPU`）做任何事情，包括`图像处理`。但是对于`图像处理`，通常用`硬`件会更快，因为`GPU`特别针对`浮点运算`做了优化，而图像的大多处理都是基于`浮点运算`。

但也不是全部将图像处理统统交给GPU硬件处理，也可以把一小部分图像处理交给CPU处理。大多数动画性能`优化`都是关于`智能利用 GPU和CPU`，使得它们都不会`超出负荷`。

那么首先需要知道`Core Animation`是如何在这两个处理器之间分配工作的。

- (1) `动画`和屏幕上`组合的layers`实际上被一个`单独的进程`管理，而不是我们编写的App应用程序

- (2) 这个单独的进程叫做`渲染服务`
	- iOS5: `SpringBoard`进程（同时管理着iOS的主屏）
	- iOS6之后: `BackBoard`进程

当`运行一段动画`时候，这个过程会被四个分离的阶段被打破:

- (1) 布局
	- 这是准备你的视图/图层的层级关系，以及设置图层属性（位置，背景色，边框等等）的阶段

- (2) 显示 
	- 这是图层的`寄宿图片`被绘制的阶段
	- 主要是重写`-drawRect:`或`-drawLayer:inContext:`方法实现，告诉系统要绘制什么图像

- (3) 准备 
	- 这是Core Animation准备发送动画数据到渲染服务的阶段
	- 这同时也是Core Animation将要执行一些别的事务例如解码动画过程中将要显示的图片的时间点

- (4) 提交 
	- 这是最后的阶段，Core Animation打包所有图层和动画属性
	- 然后通过`IPC（内部处理通信）`发送到渲染服务进行显示


但是如上四个步骤仅仅`发生在你的应用程序所在进程`之内，在动画在屏幕上显示之前仍然有更多的工作。一旦打包的图层和动画到达渲染服务进程，他们会被反序列化来形成另一个叫做`渲染树的图层树`。使用这个树状结构，渲染服务对动画的每一帧做出如下工作:

- (5) 对所有的图层属性计算中间值，设置OpenGL几何形状（纹理化的三角形）来执行渲染
- (6) 在屏幕上渲染可见的三角形

所以对于一段动画的执行，应该一共有`六个`阶段。

- `最后两个`阶段在动画过程中不停地`重复`执行
- `前五个`阶段都在软件层面处理（通过`CPU`）
- 只有最后一个被`GPU`硬件执行
- 而且我们真正只能控制前两个阶段:
	- 布局
	- 显示
- Core Animation框架在内部处理剩下的事务，也控制不了它

####在布局和显示阶段，可以决定哪些由CPU执行，哪些交给GPU去做。那么改如何判断呢？


- GPU相关的操作
	- (1) `采集图片`和`形状`（三角形），运行`变换`，应用`纹理`和`混合`然后把它们输送到屏幕上
	- (2) 但是Core Animation并没有暴露出直接的接口去操作GPU
	- (3) 只能是通过OpenGL这个库去直接操作GPU
	- (4) 宽泛的说，大多数`CALayer的属性值`都是用`GPU`来绘制
	- (5) 但是有一些事情会降低（基于GPU）图层绘制
		- 结构层次深、数量多的图层，会引起CPU的瓶颈
		- 图像的重绘
		- GPU的离屏绘制（CPU也有离屏绘制）
		- 绘制超过`2048x2048或者4096x4096尺寸`的纹理图片

- CPU相关的操作，主要是注意一些导致`动画延迟`的`CPU`操作
	- (1) 复杂的布局计算
		- 视图层级过于复杂
		- 视图呈现或者修改的时候，计算图层帧率就会消耗一部分时间
		- 特别是使用iOS6的`自动布局`机制尤为明显
	- (2) 视图懒加载
		- 虽然对`内存使用`和`程序启动时间`很有好处
		- 但是当呈现到屏幕上之前，按下按钮导致的许多工作都会`不能被及时响应`
		- 所以对于一些必须的UI，都直接创建，不必使用懒加载
	- (3) `Core Graphics`图像绘制代码
		- 如果对视图实现了`-drawRect:`方法，或者CALayerDelegate的`-drawLayer:inContext:`方法，那么在`绘制`任何东西`之前`都会产生一个巨大的性能开销
		- 为了让能够在图层内进行`任意`的绘制，Core Animation必须创建一个`内存中等大小的寄宿图片`以备后面的图像绘制
		- 一旦绘制结束之后，必须把图片数据通过`IPC`的方式传到`渲染进程`
		- `Core Graphics`绘制就会变得十分缓慢，所以在一个对性能十分挑剔的场景下这样做十分不好
	- (4) 解压图片 
		- `PNG`或者`JPEG`这些`压缩`之后的图片文件会比同质量的`位图`体积小得多
		- 但是在`图片绘制`到屏幕上`之前`，必须把`PNG/JPEG压缩图片`扩展成`完整的未解压`的尺寸
			- 通常等同于图片宽 x 长 x 4个字节
		- 为了节省内存，iOS通常直到`真正绘制`的时候才去`解码`图片
		- 会触发解压图片的场景:
			- 第一次对图层内容赋值的时候（直接或者间接使用UIImageView）
			- 把图片即将绘制到`Core Graphics Context`中时
		- 这样的话，对于一个`较大`的图片，都会占用一定的`图片解码`时间

- 当图层被成功打包，发送到渲染服务器之后，`CPU`仍然要做如下工作:
	- (1) 为了显示屏幕上所有的图层layer，`Core Animation`必须对`渲染树`中的`每个可见图层layer`通过`OpenGL`循环转换成`纹理三角板`
	- (2) 因为`GPU`并不知晓`Core Animation CALayer`的数据结构，所以必须要由`CPU`做这些事情
	- (3) 这里`CPU`的处理时间，就与`图层layer个数`成`正比`

####使用`真机`测试动画代码的执行性能，而不是`模拟器`

- (1) 模拟器上的测试出的性能会高度失真
- (2) 性能测试一定要用`发布 release`配置，而不是`调试 debug`模式
	- 因为当用发布环境打包的时候，编译器会引入一系列提高性能的优化，例如去掉调试符号或者移除并重新组织代码
- (3) 最好在你支持的设备中`性能最差`的设备上测试

####使用Instruments中的`CoreAnimation监测工具`的几个选项作用

> Color Blended Layers

Instruments可以在物理机上显示出被混合的图层Blended Layer(用`红色`标注)，Blended Layer是因为这些Layer是`透明`的(Transparent)，系统在渲染这些view时需要将该view和下层view`混合(Blend)`后才能 计算出该像素点的实际颜色，如果这种blended layer很多，那么在滚动列表时就甭想有流畅的效果。

这个选项基于渲染程度对屏幕中的混合区域进行`绿到红`的高亮（也就是多个半透明图层的叠加）。由于`重绘`的原因，`对所有layer透明的区域进行混合`对GPU性能会有影响，同时也是滑动或者动画帧率下降的罪魁祸首之一。

解决`blended layer`问题也很简单，检查`红色区域`view/layer的如下属性:

- (1) `opaque`属性，记得设置成`YES`；
- (2) `backgroundColor`属性是不是`[UIColor clearColor]`，

> Color Hits Greenand MissesRed

当使用`shouldRasterizep`属性的时候，耗时的图层绘制会被`缓存`，然后当做一个简单的`扁平图片`呈现。

也就是说系统会将这些`Layer`缓存成`Bitmap位图`供渲染使用，那么下次判断这个layer是否有对应的bitmap位图，如果有直接对这个bitmap进行渲染，而不需要对这个layer进行核查渲染等处理，如果失效时便丢弃这些Bitmap重新生成。

使用这个选项后时，如果Rasterized的Layer`失效`，便会标注为`红色`，如果`有效`标注为`绿色`。当测试的应用频繁闪现出`红色`标注图层时，表明对图层做的`Rasterization作用不大`。

> Color Copied Images

有时候`寄宿图片`的生成意味着Core Animation被`强制生成一些图片`，然后发送到渲染服务器，而不是简单的`指向原始指针`。复制图片对内存和CPU使用来说都是一项非常昂贵的操作，所以应该尽可能的避免。

这个选项把这些使用复制图片的图层渲染成`蓝色`。

> Color Immediately

 通常Core Animation Instruments 以`每毫秒10次`的频率更`新图层`的`调试颜色`。对某些效果来说，这显然太慢了。这个选项就可以用来设置每帧都更新（可能会影响到渲染性能，而且会导致帧率测量不准，所以不要一直都设置它）。

> Color Misaligned Images

这里会`高亮`那些被`缩放`或者`拉伸`以及`没有正确对齐`到像素边界的`图片`（也就是`非整型`坐标）。

这些中的大多数通常都会导致图片的不正常缩放，如果把一张大图当缩略图显示，或者不正确地模糊图像，那么这个选项将会帮你识别出问题所在。

> Color Offscreen-Rendered Yellow

这里会把那些需要`离屏渲染`的图层高亮成`黄色`。这些图层很可能需要用`shadowPath`或者`shouldRasterize`来优化。

下列情况会导致视图的Offscreen- Rendering:

- (1) Core Graphics (any class prefixed with CG*)
- (2) The drawRect() method, even with an empty implementation.
- (3) CALayers with a shouldRasterize property set to YES.
- (4) CALayers using masks (setMasksToBounds) and dynamic shadows (setShadow*).
- (5) Any text displayed on screen, including Core Text.
- (6) Group opacity (UIViewGroupOpacity).

> Color OpenGL Fast Path Blue

这个选项会对任何`直接使用OpenGL绘制的图层`进行高亮。如果仅仅使用`UIKit或者Core Animation`的API，那么不会有任何效果。

如果使用`GLKView或者CAEAGLLayer`，那如果不显示`蓝色块`的话就意味着你正在`强制CPU渲染额外的纹理`，而不是绘制到屏幕。

> Flash Updated Regions

这个选项会对`重绘`的内容高亮成`黄色`（也就是任何在软件层面使用`Core Graphics绘制的图层`），这种绘图的`速度很慢`。

如果频繁发生这种情况的话，这意味着有一个隐藏的bug或者说通过`增加缓存`或者使用替代方案会有提升性能的空间。

####使用Instruments中的`OpenGL ES`和`GPU Driver`侧栏的邮编是一系列有用的属性值

- (1) `Renderer Utilization` - 如果这个值`超过50%`，就意味着你的动画可能对帧率有所限制，很可能因为`离屏渲染`或者是`重绘`导致的过度混合

- (2) `Tiler Utilization` - 如果这个值`超过50%`，就意味着你的动画可能限制于几何结构方面，也就是在屏幕上有`太多的图层`

	
####对于一些结构都差不多的UITbleviewCell，可以使用`shouldRasterize`来缓存图层中一些需要离屏渲染的图像内容（阴影、圆角、透明...）

可以使用shouldRasterize来缓存图层内容。这将会让图层离屏之后渲染一次然后把结果保存起来，直到下次利用的时候去更新

```objc
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
￼{
    //dequeue cell
    UITableViewCell *cell = [self.tableView dequeueReusableCellWithIdentifier:@"Cell"
                                                                 forIndexPath:indexPath];
    ...
    //set text shadow
    cell.textLabel.backgroundColor = [UIColor clearColor];
    cell.textLabel.layer.shadowOffset = CGSizeMake(0, 2);
    cell.textLabel.layer.shadowOpacity = 0.5;
    
    //rasterize
    cell.layer.shouldRasterize = YES;
    cell.layer.rasterizationScale = [UIScreen mainScreen].scale;
    return cell;
}
```

我们仍然离屏绘制图层内容，但是由于显式地禁用了栅格化，Core Animation就对绘图缓存了结果，于是对提高了性能。

那么对于比较好的性能检测: 大部分都是`绿色`，只有当滑动到屏幕上的时候会闪烁成`红色`标示复杂图层合成或者离屏渲染。

***

###高性能绘图

####iOS中系统绘图的方式

- (1) `CoreAnimation图层`完成绘图

- (2) （硬件绘图）通过`OpenGL`代码，最终交给`GPU`渲染

- (3)（软件绘图）通过`CoreGraphics`代码，编写好要绘制的图形，交给`CPU`进行渲染

三者进行绘图的性能排序:

`OpenGL` > `CoreAnimation图层` > `CoreGraphics`


####软件绘图不仅`效率低`，还会`占用内存`也很多

`CALayer`只需要一些与自己相关的内存，只有它的`寄宿图`会消耗一定的内存空间。即使直接赋给`layer.contents`属性`一张图片`，也不需要增加额外的照片存储大小。如果相同的`一张图片`被`多个图层layer`作为`contents`属性值引用，那么他们将会`共用同一块内存`，而`不是复制`内存块。


一旦你实现如下两种绘图方法实现:

- (1) CALayerDelegate协议中的`-drawLayer:inContext:`方法
- (2) UIView中的`-drawRect:`方法（其实就是上面的包装方法）

图层就创建了一个绘制上下文，这个上下文需要的大小的内存可从这个算式得出:

```
绘图上下文占用的内存 = 图层宽（像素） * 图层高（像素）* 4 个字节
```

比如，对于一个在Retina iPad上的`全屏`图层，绘图时占用的内存:

```
绘图上下文占用的内存 = 2048 * 1526 * 4 个字节 = 12M
```

可以看到，对于此时绘制一个全屏图像时，需要消耗掉12M得内存。并且当重新绘制这个全屏图像时，会将之前的12M内存`抹掉`，然后`重新分配`出12M的内存用于绘图。

`软件`绘图的代价昂贵，除非绝对必要。但是有时有必须使用，此时只能应该尽量的避免`重新绘制`图像产生很大的性能消耗。

####通常使用CoreGraphics软件绘图的原因: 有一些可能`路径不规则`或`效果复杂`的图像，无法使用简单的图片或layer图层绘制出来

有一些可能`路径不规则`或`效果复杂`的图像:

- (1) 任意多边形（不仅仅是一个矩形）
- (2) 斜线或曲线
- (3) 文本富文本效果
- (4) 颜色渐变

比如，做一个在屏幕上随着用户触摸的路线进行路线绘制，这个就无法简单的使用图片、或者layer完成。就只能使用软件绘图的方式自定义出要绘制的图像已经路径。下面是使用`UIBezierPath`完成的路径绘制（也可以使用CoreGraphics提供的`CGPath`绘制路径，但是稍微麻烦点）:

```objc
@interface DrawingView ()

@property (nonatomic, strong) UIBezierPath *path;

@end

@implementation DrawingView

- (void)awakeFromNib
{
    // 创建出一个要绘制的路径
    self.path = [[UIBezierPath alloc] init];
    
    // 设置路径的线条样式
    self.path.lineJoinStyle = kCGLineJoinRound;
    self.path.lineCapStyle = kCGLineCapRound;
    ￼
    // 设置路径的线条宽度
    self.path.lineWidth = 5;
}

- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
    // 获取开始的触摸点
    CGPoint point = [[touches anyObject] locationInView:self];

	// 设置路径的起始点
    [self.path moveToPoint:point];
}

- (void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event
{
	// 不断的获取当前触摸点
    CGPoint point = [[touches anyObject] locationInView:self];

	// 不断的将当前触摸点加到路径中
    [self.path addLineToPoint:point];

	// 不断的进行路径的重新绘制
    [self setNeedsDisplay];
}

- (void)drawRect:(CGRect)rect
{
	// 将路径绘制出来
    [[UIColor clearColor] setFill];
    [[UIColor redColor] setStroke];
    [self.path stroke];
}

@end
```

用`Core Graphics`这样实现的问题在于:

- (1) 路径画得越多越长越复杂，程序就会越慢
- (2) 因为每次移动手指的时候都会`重绘 整个 贝塞尔路径`（UIBezierPath）
- (3) 所以随着路径越来越复杂，每次重绘的工作就会增加，直接导致了`每一秒的刷新帧数(FPS)`的下降
- (4) 因为`每一帧`的处理时间明显变长了，理想情况下`每一帧`的处理时间控制在`10ms左右`

####使用Core Animation为这些图形类型的绘制提供了专门的layer类来进行优化。对于上面使用普通`CALayer + CoreGraphics绘制图像`有专门的优化layer >>> `CAShapeLayer`.

CAShapeLayer可以绘制多边形，直线和曲线。再比如如果高性能实现`UILabel`文字显示可以使用`CATextLayer`。再比如如果想实现颜色渐变可以使用`CAGradientLayer`。

总之，比直接使用 `普通 CALayer + CoreGrapihics` 进行图像绘制效率还是会高一些的，因为如上的这些特定的Layer会避免创造一个`空的寄宿图`。

那么使用`CAShapeLayer`对上述直接使用`普通 CALayer + CoreGrapihics` 进行图像绘制的代码优化如下:

```objc
#import "DrawingView.h"
#import <QuartzCore/QuartzCore.h>

@interface DrawingView ()
@property (nonatomic, strong) UIBezierPath *path;
@end

@implementation DrawingView

/// 告诉系统使用CAShapeLayer对象作为View的寄宿主图层
+ (Class)layerClass
{
    //this makes our view create a CAShapeLayer
    //instead of a CALayer for its backing layer
    return [CAShapeLayer class];
}

- (void)awakeFromNib
{
    //create a mutable path
    self.path = [[UIBezierPath alloc] init];

    // 创建CAShapeLayer对象
    CAShapeLayer *shapeLayer = (CAShapeLayer *)self.layer;
    shapeLayer.strokeColor = [UIColor redColor].CGColor;
    shapeLayer.fillColor = [UIColor clearColor].CGColor;
    shapeLayer.lineJoin = kCALineJoinRound;
    shapeLayer.lineCap = kCALineCapRound;
    shapeLayer.lineWidth = 5;
}

- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
    //get the starting point
    CGPoint point = [[touches anyObject] locationInView:self];

    //move the path drawing cursor to the starting point
    [self.path moveToPoint:point];
}

- (void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event
{
    //get the current point
    CGPoint point = [[touches anyObject] locationInView:self];

    //add a new line segment to our path
    [self.path addLineToPoint:point];

    // 将要绘制的路径交给CAShapeLayer对象，然后由View对象完成后续的所有事情即可
    ((CAShapeLayer *)self.layer).path = self.path.CGPath;
    
}

@end
```

但是此时情况下，CAShapeLayer对象只能绘制出一些`矢量基本图形、矢量路径`。但如果想要显示一些`图片`就只能使用`CoreGraphics`来绘制。

```objc
- (void)drawRect:(CGRect)rect
{
	// 图片绘制到画布上
	[[UIImage imageNamed:@"Chalk.png"] drawInRect:brushRect];
}
```

###尽管使用CAShapeLayer代替CoreGraphics来绘制一些矢量图形、矢量路径之后，性能会一定提升。但是仍然没有解决每一次重新绘制路径时，随着路径越来越长、越来越复杂，导致每次重新绘制的成本越来越大，最终导致每一秒的刷新帧数（FPS）下降

为了减少`不必要的绘制`，Mac OS和iOS设备将会把屏幕区分为:

- (1) 需要重绘的区域，称为`脏区域`
- (2) 不需要重绘的区域

在实际应用中，鉴于`非矩形`区域边界裁剪和混合的复杂性，通常会区分出包含指定视图的`矩形`位置，而这个位置就是`脏矩形`。 

当一个`视图`被`改动`过了，那么这个视图可能需要`重绘`。但是很多情况下，只是这个`视图的一部分`被改变了，所以`重绘`整个寄宿图就太`浪费`了。

但是`Core Animation`无法区分出我们自定义的图像绘图代码中，哪些需要进行重新绘制，哪些不需要重新绘制。即不知道`脏区域`。

那么优化的方案，就是告诉`Core Animation`我只需要重新绘制那一块区域 

- (1) `-[UIView setNeedsDisplayInRect:]` 
- (2) `-[CALayer setNeedsDisplayInRect:]` 

对上面两个例子，使用局部刷新进行优化的代码:

```objc
#import "DrawingView.h"
#import <QuartzCore/QuartzCore.h>
#define BRUSH_SIZE 32

@interface DrawingView ()

@property (nonatomic, strong) NSMutableArray *strokes;

@end

@implementation DrawingView

- (void)awakeFromNib
{
    //create array
    self.strokes = [NSMutableArray array];
}

- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
    //get the starting point
    CGPoint point = [[touches anyObject] locationInView:self];

    //add brush stroke
    [self addBrushStrokeAtPoint:point];
}

- (void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event
{
    //get the touch point
    CGPoint point = [[touches anyObject] locationInView:self];

    //add brush stroke
    [self addBrushStrokeAtPoint:point];
}

- (void)addBrushStrokeAtPoint:(CGPoint)point
{
    //add brush stroke to array
    [self.strokes addObject:[NSValue valueWithCGPoint:point]];

    //设置要局部刷新的区域，即告诉CoreGraphics重新绘制的区域，脏区域
    [self setNeedsDisplayInRect:[self brushRectForPoint:point]];
}

- (CGRect)brushRectForPoint:(CGPoint)point
{
	// 返回当前触摸点一个小的矩形区域
    return CGRectMake(point.x - BRUSH_SIZE/2, point.y - BRUSH_SIZE/2, BRUSH_SIZE, BRUSH_SIZE);
}

- (void)drawRect:(CGRect)rect
{
    //redraw strokes
    for (NSValue *value in self.strokes) {
    
        //get point
        CGPoint point = [value CGPointValue];

        // 取出每个触摸点的矩形区域
        CGRect brushRect = [self brushRectForPoint:point];
        ￼
        // 只对当前触摸点的矩形区域内进行图像重绘
        if (CGRectIntersectsRect(rect, brushRect)) {
            [[UIImage imageNamed:@"Chalk.png"] drawInRect:brushRect];
        }
    }
}

@end
```

这样改之后，就不会对整个路径每次都重新绘制，而只是绘制当前触摸点附近的区域内的路径和图像。

####绘图继续优化之、异步绘制

因为UIKit特点:

- (1) 只能在`单线程`上进行创建、修改、释放、废弃
- (2) 这个单线程，只能是唯一的`主线程`

这样的话，可能因为一些复杂的界面图像进行绘制时需要大量的时间，从而导致这段时间内界面无法响应任何的用户事件。

那么可以考虑将图像绘制渲染放到子线程上执行，但是最终仍然需要回到`主线程`:

- (1) 将图像绘制、图像渲染先`异步`放到`子线程`上完，最终得到渲染完毕的图片

- (2) 然后回到`主线程`，将渲染出来的图片设置给 `layer.contens` 进行显示

####异步绘制、CATiledLayer

除了将图层再次分割成独立更新的小块（类似于脏矩形自动更新的概念），CATiledLayer还有一个有趣的特性:

- (1) 在多个线程中为每个小块同时调用`-drawLayer:inContext:`方法
- (2) 这就避免了阻塞用户交互而且能够利用多核心新片来更快地绘制
- (3) CATiledLayer是实现异步更新图片视图的简单方法

####后台子线程绘制、iOS6之后`layer.drawsAsynchronously`完成异步绘制

- (1) iOS 6中，苹果为CALayer引入了这个令人好奇的属性
- (2) `drawsAsynchronously `属性对传入`-drawLayer:inContext:的CGContext`进行改动，允许CGContext延缓绘制命令的执行以至于不阻塞用户交互
- (3) `drawsAsynchronously`与CATiledLayer使用的`异步绘制`并不相同
- (4) `CATiledLayer`自己的`-drawLayer:inContext:`方法只会在主线程调用，但是CGContext并不等待每个绘制命令的结束
	- 最终还是在`主线程`绘制图形

- (5) `drawsAsynchronously`会将绘制命令加入`队列`，当方法返回时，在`后台线程`逐个执行真正的绘制
	- 真正的在`后台子线程`完成图形绘制

- (6) 根据苹果的说法。这个特性在需要`频繁重绘`的视图上效果最好（比如我们的绘图应用，或者诸如UITableViewCell之类的），对那些只`绘制一次或很少重绘`的图层内容来说没什么太大的帮助

***

###高性能图片IO处理

对于图片文件的读取加载也是需要考虑的性能优化点，既不可能永远从磁盘文件读入图片，也不可能永远无限制的将图片缓存在内存，需要在应用运行的时候周期性地`加载`和`卸载`图片。

对于一个图片的显示过程:

- (1) 从磁盘文件或内存中读取加载
- (2) 进行图片的解压缩（大部分图片PNG、JPEG格式都是压缩文件，需要进行解压缩才能使用）
- (3) 图片的绘制、渲染、合成...


####对于图片读取的优化点:

- (1) 在一些用户`视觉`场景时做一些后台读取图片
	- 启动图播放时
	- 播放某一些动画效果时

- (2) 将一些大图片的IO读取`异步`放入到`子线程`上执行
	- GCD dispatch queue
	- NSOperationQueue

	
```objc
- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView
                  cellForItemAtIndexPath:(NSIndexPath *)indexPath
￼{
    //dequeue cell
    UICollectionViewCell *cell = [collectionView dequeueReusableCellWithReuseIdentifier:@"Cell" forIndexPath:indexPath];
    ...
    //switch to background thread
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0), ^{
    
        //1. 读取图片文件
        NSInteger index = indexPath.row;
        NSString *imagePath = self.imagePaths[index];
        UIImage *image = [UIImage imageWithContentsOfFile:imagePath];
        
        //2. 使用上下文绘制图片，触发图片在子线程上进行解压缩
        UIGraphicsBeginImageContextWithOptions(imageView.bounds.size, YES, 0);
        [image drawInRect:imageView.bounds];
        image = UIGraphicsGetImageFromCurrentImageContext();
        UIGraphicsEndImageContext();
        
        //3. 回调主线程，将解压缩后的图片设置个UI或layer
        dispatch_async(dispatch_get_main_queue(), ^{
            if (index == cell.tag) {
                imageView.image = image;
            }
        });
    });
    return cell;
}
```

- (3) 延迟图片文件的`解压`（PNG/JPEG..）
	- 一旦图片文件被`加载`就必须要进行解码
	- PNG格式图片压缩后的`体积最小`
	- `PNG`解压花费时间比`JPEG`格式文件会`更短`，因为PNG格式文件压缩算法比较简单
	- 但是`PNG`图片的读取加载时间 远远大于 `JPEG`图片
	- 可以像ASDK那样延迟解压、异步子线程完成解压，最终回调主线程呈上将解压后的图片设置给layer.contens

- (4) 但是过多的延迟解压，会响应`准备绘制图片`的时候影响性能，因为需要在绘制之前进行解压。那么可以`立即`让系统`执行图片解压`的方法:
	- `-[UIImage imageNamed:]`，只对从应用资源束中的图片有效，所以对用户生成的图片内容或者是下载的图片就没法使用了
	- `UIImageView.image = 图片`，但是必须保证这句代码是在`主线程`上
	- 使用`ImageIO`函数api，强制图片立刻解压

- (5) 因为图片必须要在`绘制`之前解压，所以就强制了`解压`的`及时性`。而由于绘图如果使用`CoreGraphics`函数进行的话，可以将使用`CoreGraphics`绘图代码放入到`子线程`上执行。那么也就可以通过在`子线程绘图`的方式，间接的让`图片`在`子线程`完成`解压`，主要有两种思路:

	- 方法一、只将`整张图片`中的某一个像素，绘制到一个像素大小的`CGContext绘图上下文`中
		- 这样仍然会触发`整张图片`的解压缩
		- 但是只是绘制图片中的某一个像素，所以在绘图上的消耗可以忽略不计
		- 那么随时可以进行一像素的绘制，触发图片解压缩，用完后就可以扔掉解压出来的图片
		
	- 方法二、将`整张图片`绘制到`CGContext`中，丢弃原始的图片，然后用一个从上下文内容中获取的新图片来代替
		- 不会像上面方法解压多次，只会解压缩一次，并且将解压缩之后的图片要缓存起来



****

###图层性能优化点小结

#### `隐式绘制`图层的寄宿图 与 `显示绘制`图层的寄宿图

- `显示`绘制图层的寄宿图的方式

	- (1) 通过`Core Graphics`直接绘制
	- (2) 载入一个图片文件，并赋值给`layer.contents`属性
	- (3) 事先绘制一个`屏幕之外的CGContext`上下文（离屏渲染）

- `隐式`绘制图层的寄宿图
	
	- (1) 使用特定的`图层属性`
	- (2) 使用特定的`视图`
	- (3) 使用特定的`图层子类`

####对于文本

- (1) 对于经常变动、需要重新绘制的内容，放到一个子图层/子视图中，然后使用`setNeedsDisplayInRect:`进行局部的重绘

- (2) 如果追求文本的渲染性能，可以在UILabel子类中封装一个`CATextLayer`来完成文本的渲染
	- CATextLayer 用的是`Core Text`渲染更迅速
	- UILabel 用`WebKit的HTML渲染引擎`来绘制文本
	- 但是这两种方法都用了`软件`的方式绘制，因此他们实际上要比`硬件加速合成`方式要慢

####图层的光栅化、即`CALayer的shouldRasterize属性`

- (1) 它可以解决`重叠透明图层`的混合失灵问题

- (2) 启用shouldRasterize属性会将图层绘制到一个`屏幕之外`的图像，即会触发GPU的离屏渲染

- (3) 虽然GPU的离屏渲染是很消耗性能，但是使用了shouldRasterize属性后，只会触发`一次`离屏渲染，然后会将渲染得到的图片进行`缓存`

- (4) 但是一定不要在`内容不断变动`的图层上使用，否则它缓存方面的好处就会消失，而且会让性能变的更糟

- (5) 为了检测你是否正确地使用了光栅化方式，用Instrument查看一下`Color Hits Green和Misses Red`项目，是否已光栅化图像被频繁地刷新（这样就说明图层并不是光栅化的好选择，或则你无意间触发了不必要的改变导致了重绘行为）。


####离屏渲染

- (1) 当图层属性的`混合体`被指定为`在未预合成`之前`不能直接`在屏幕中`绘制`时，屏幕外渲染就被唤起了

- (2) 离屏渲染会让图层在被`显示`之前`在一个屏幕外上下文中被渲染（不论CPU还是GPU）

- (3) 常见的触发离屏渲染的代码

```
圆角（当和maskToBounds一起使用时）
图层蒙板
阴影
```

####圆角的高性能处理

- (1) UIBezirePath


```objc
- (UIImage *)makeCircularImageWithSize:(CGSize)size
{
  // make a CGRect with the image's size
  CGRect circleRect = (CGRect) {CGPointZero, size};
  
  // begin the image context since we're not in a drawRect:
  UIGraphicsBeginImageContextWithOptions(circleRect.size, NO, 0);
  
  // create a UIBezierPath circle
  UIBezierPath *circle = [UIBezierPath bezierPathWithRoundedRect:circleRect cornerRadius:circleRect.size.width/2];
  
  // clip to the circle
  [circle addClip];
  
  // draw the image in the circleRect *AFTER* the context is clipped
  [self drawInRect:circleRect];
  
  // create a border (for white background pictures)
  circle.lineWidth = 1;
  [[UIColor darkGrayColor] set];
  [circle stroke];
  
  // get an image from the image context
  UIImage *roundedImage = UIGraphicsGetImageFromCurrentImageContext();
  
  // end the image context since we're not in a drawRect:
  UIGraphicsEndImageContext();
  
  return roundedImage;
}
```

- (2) CAShapeLayer

```objc
@interface ViewController ()

@property (nonatomic, weak) IBOutlet UIView *layerView;

@end

@implementation ViewController

- (void)viewDidLoad
{
    [super viewDidLoad];

    //create shape layer
    CAShapeLayer *blueLayer = [CAShapeLayer layer];
    blueLayer.frame = CGRectMake(50, 50, 100, 100);
    blueLayer.fillColor = [UIColor blueColor].CGColor;
    blueLayer.path = [UIBezierPath bezierPathWithRoundedRect:
    CGRectMake(0, 0, 100, 100) cornerRadius:20].CGPath;
    ￼
    //add it to our view
    [self.layerView.layer addSublayer:blueLayer];
}
@end
```

####可伸缩图片

```objc
@implementation ViewController

- (void)viewDidLoad
{
    [super viewDidLoad];

    //create layer
    CALayer *blueLayer = [CALayer layer];
    blueLayer.frame = CGRectMake(50, 50, 100, 100);
    blueLayer.contentsCenter = CGRectMake(0.5, 0.5, 0.0, 0.0);
    blueLayer.contentsScale = [UIScreen mainScreen].scale;
    blueLayer.contents = (__bridge id)[UIImage imageNamed:@"Circle.png"].CGImage;
    //add it to our view
    [self.layerView.layer addSublayer:blueLayer];
}
@end
```

使用可伸缩图片的优势在于，它可以绘制成`任意边框效果`而不需要额外的性能消耗。举个例子，可伸缩图片甚至还可以显示出`矩形阴影`的效果。


####混合和过度绘制

- (1) GPU每一帧可以绘制的像素有一个最大限制（`2048*2048，或4096*4096`），这个情况下可以轻易地绘制整个屏幕的所有像素

- (2) 但是如果由于重叠图层的关系，需要不停地重绘同一区域的话，掉帧就可能发生了

- (3) GPU会放弃绘制那些`完全被其他图层遮挡`的像素，但是要计算出一个图层是否被遮挡也是相当复杂并且会消耗处理器资源

- (4) 合并不同图层的`透明 重叠 像素（即混合）`消耗的资源也是相当客观的

- (5) 所以为了加速处理进程，不到必须时刻不要使用`透明图层`，任何情况下，应该这么做:
	- 给视图的`backgroundColor`属性设置一个`固定、不透明`的颜色
	- 设置`opaque = YES`

这样做减少了`混合`行为（因为编译器知道在图层之后的东西都不会对最终的像素颜色产生影响）并且计算得到了加速，避免了过度绘制行为因为Core Animation可以舍弃所有被完全遮盖住的图层，而不用每个像素都去计算一遍。

如果用到了图像，尽量避免`透明`除非非常必要。如果图像要显示在一个`固定的背景颜色`或是`固定的背景图`之前，你没必要相对前景移动，你只需要`预填充背景图片`就可以避免运行时混色了。

如果是`文本`的话，一个`白色`背景的UILabel（或者其他颜色）会比透明背景要更高效。

最后，明智地使用`shouldRasterize`属性，可以将一个`固定的图层体系折叠成单张图片`，这样就不需要每一帧`重新合成`了，也就不会有因为子图层之间的混合和过度绘制的性能问题了。

#### 减少图层数量

对于一个图层主要经过的处理过程:

- (1) 初始化图层
- (2) 合成渲染图层
- (3) 打包通过IPC发给渲染引擎
- (4) 由GPU转化成OpenGL几何图形

这些是一个图层的大致资源开销，事实上，一次性能够在屏幕上显示的最大图层数量也是有限的。


###裁切

在对图层做任何优化之前，你需要确定你不是在创建一些`不可见的图层`，图层在以下几种情况下会是`不可见`的：

- (1) 图层在`屏幕边界之外`，或是在父图层边界之外
- (2) 完全在一个不透明图层之后（被遮挡了）
- (3) 完全透明

比如，下面例子、排除可视区域之外的图层

```objc
#import "ViewController.h"
#import <QuartzCore/QuartzCore.h>

#define WIDTH 100
#define HEIGHT 100
#define DEPTH 10
#define SIZE 100
#define SPACING 150
#define CAMERA_DISTANCE 500
#define PERSPECTIVE(z) (float)CAMERA_DISTANCE/(z + CAMERA_DISTANCE)

@interface ViewController () <UIScrollViewDelegate>

@property (nonatomic, weak) IBOutlet UIScrollView *scrollView;

@end

@implementation ViewController

- (void)viewDidLoad
{
    [super viewDidLoad];
    //set content size
    self.scrollView.contentSize = CGSizeMake((WIDTH - 1)*SPACING, (HEIGHT - 1)*SPACING);
    //set up perspective transform
    CATransform3D transform = CATransform3DIdentity;
    transform.m34 = -1.0 / CAMERA_DISTANCE;
    self.scrollView.layer.sublayerTransform = transform;
}
￼
- (void)viewDidLayoutSubviews
{
    [self updateLayers];
}

- (void)scrollViewDidScroll:(UIScrollView *)scrollView
{
    [self updateLayers];
}

- (void)updateLayers
{
    //calculate clipping bounds
    CGRect bounds = self.scrollView.bounds;
    bounds.origin = self.scrollView.contentOffset;
    bounds = CGRectInset(bounds, -SIZE/2, -SIZE/2);

    //create layers
    NSMutableArray *visibleLayers = [NSMutableArray array];
    for (int z = DEPTH - 1; z >= 0; z--)
    {
        //increase bounds size to compensate for perspective
       CGRect adjusted = bounds;
        adjusted.size.width /= PERSPECTIVE(z*SPACING);
        adjusted.size.height /= PERSPECTIVE(z*SPACING);
        adjusted.origin.x -= (adjusted.size.width - bounds.size.width) / 2;
        adjusted.origin.y -= (adjusted.size.height - bounds.size.height) / 2;
        
        for (int y = 0; y < HEIGHT; y++) {
        	
        	//check if vertically outside visible rect
            if (y*SPACING < adjusted.origin.y || y*SPACING >= adjusted.origin.y + adjusted.size.height)
            {
                continue;
            }
            
            for (int x = 0; x < WIDTH; x++) {
                //check if horizontally outside visible rect
                if (x*SPACING < adjusted.origin.x ||x*SPACING >= adjusted.origin.x + adjusted.size.width)
                {
                    continue;
                }
                ￼
                //create layer
                CALayer *layer = [CALayer layer];
                layer.frame = CGRectMake(0, 0, SIZE, SIZE);
                layer.position = CGPointMake(x*SPACING, y*SPACING);
                layer.zPosition = -z*SPACING;
                //set background color
                layer.backgroundColor = [UIColor colorWithWhite:1-z*(1.0/DEPTH) alpha:1].CGColor;
                                //attach to scroll view
                [visibleLayers addObject:layer];
            }
        }
    }
    
    //update layers
    self.scrollView.layer.sublayers = visibleLayers;
    
    //log
    NSLog(@"displayed: %i/%i", [visibleLayers count], DEPTH*HEIGHT*WIDTH);
}

@end
```

只对一些可见区域的图层进行重绘。

####对象回收

- (1) 对象的缓存池Pool
- (2) 使用子线程进行异步释放对象，避免在主线程上同步执行

####Core Graphics绘制图像至少比添加更多的图层好一些

- (1) 因为要尽量的减少图层数，那么可以使用Core Graphics绘制来绘制一些简单的图像
- (2) 但是Core Graphics绘制属于CPU软件绘制，比GPU硬件绘制会慢得多
- (3) 但毕竟软件绘制，起码能减少很多图层，避免了图层分配和操作问题
- (4) 而且Core Graphics绘制的代码天生就是多线程安全的

****

###随意重写`drawRect:`进行绘图，可能导致大量消耗内存

