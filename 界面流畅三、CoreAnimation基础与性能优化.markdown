## CoreAnimation学习笔记

一个很好的学习资源，翻译的iOS-Core-Animation-Advanced-Techniques文章。

```
https://github.com/AttackOnDobby/iOS-Core-Animation-Advanced-Techniques
```

以及一些专有图层CALayer介绍与使用的的

```
http://blog.csdn.net/xiepanqi/article/details/50113093
http://www.jianshu.com/p/8c1c1697c0ce
```

图像渲染的一些文章

```
http://blog.csdn.net/sqc3375177/article/details/52758557
```

## No1. iOS图像渲染框架结构


<img src="./layer_rendering.jpeg" alt="图1.1" title="图1.1" width="700"/>

UIKit作为CoreAnimation的入口，分为存在iOS版本和MaxOS两个不同的版本。

而CoreAnimation负责调用更加底层的基于c语言的绘图框架:

- (1) OpenGL ES >>> 基于GPU渲染
- (2) Core Graphics >>> 基于CPU渲染

最终，由OpenGL ES或Core Graphics，去调用底层的`硬件`（GPU 或 CPU）完成数据渲染。

所以，CoreAnimation不仅仅只是用来做`动画`而已，而是进行`数据显示`的一个很重要的`入口`。

## No2. CoreAnimation 并不只是完成动画，而是让数据快速的显示到屏幕上的一个引擎或底层类库


### 图层（CALayer）、图层树（CALayer Tree）

- (1) 每一部分要显示的数据，被某一个单独的`图层`管理
- (2) 所有的`图层`存储在一个叫做`图层树`的体系之中


而CoreAnimation，就是负责将所有的图层进行`复合`处理成为可以显示在屏幕上的东西。

### 图示iOS系统关于显示的整个系统层级结构



## 图层（CALayer）与 视图（UIView）的关系


### 他们两的关系图示为如下:


<img src="./uiview_layer.png" alt="图1.1" title="图1.1" width="700"/>


### iOS中的视图（UIView）分类、以及继承层级

```c
- NSObject
	- UIResponder
		- UIView
			- UIControl
				- UIButton
				- UISwitch
				- UITextField
			- UIAlertView
			- UILabel
			- UIScrollView
				- UITextView
				- UITableView
				- UICollectionView
			- UIToolbar
			- UITabBar
```


### iOS中的图层（CALayer）分类、以及继承层级

```c
- NSObject
	- CALayer
		- CAShapeLayer、一个通过矢量图形而不是bitmap来绘制的图层子类。
		- CATextLayer、以图层的形式包含了UILabel几乎所有的绘制特性，并且额外提供了一些新的特性。CATextLayer也要比UILabel渲染得快得多。
		- CATransformLayer、做一些3D变换的操作。
		- CAGradientLayer、用来生成两种或更多颜色平滑渐变，真正好处在于绘制使用了【硬件加速】。
		- CAReplicatorLayer、为了高效生成许多【相似】的图层。它会绘制一个或多个图层的子图层，并在每个复制体上应用不同的变换
		- CAScrollLayer、类似于UIScrollView。
		- CATiledLayer、为载入大图造成的性能问题提供了一个解决方案：将大图分解成小片然后将他们单独按需载入。
		- CAEmitterLayer、一个高性能的粒子引擎，被用来创建实时例子动画如：烟雾，火，雨等等这些效果。
		- CAEAGLLayer、用来操作OpenGL来绘制图像。
		- AVPlayerLayer、AVPlayerLayer是用来在iOS上播放视频的。
```

### CALayer 与 UIView，最大的不同是CALayer不处理用户的事件交互

CALayer，虽然是提供了一个方法实现`containsPoint:`来判断触摸事件Point是否位于CALayer区域内部。

但是并没有实现如何参与`事件响应链`的代码，因为只有`UIView`实现了

```objc
//1. 查询是否能够响应触摸事件
- (nullable UIView *)hitTest:(CGPoint)point withEvent:(nullable UIEvent *)event;

//2. 触摸事件Point是否位于UIView内
- (BOOL)pointInside:(CGPoint)point withEvent:(nullable UIEvent *)event;
```
	
而CALayer虽然也实现了上面两个类似的函数，但是并没有用来接收`UIEvent *`类型对象的参数，所以是无法参与事件传递的

```objc
//1.
- (nullable CALayer *)hitTest:(CGPoint)p;

//2.
- (BOOL)containsPoint:(CGPoint)p;
```

### CALayer 与 UIView，是平行的层级关系

CALayer类在概念上和UIView类似，同样也是一些被层级关系树管理的矩形块，同样也可以包含一些内容（像图片，文本或者背景色），管理子图层的位置。它们都有一些方法和属性用来做`动画 Animaiton`和`变换 Transform`。

每一个视图（UIView）都有一个图层（CALayer）实例，也就是所谓的`backing layer`。

视图（UIView）的职责：

- (1) 创建并管理这个图层（CALayer）
- (2) 确保当子视图（subviews）在层级关系中，添加或者被移除的时候，他们关联的图层（CALayer），也同样对应在层级关系树当中有相同的操作


实际上这些背后关联的 `图层（CALayer）` 才是真正用来在屏幕上`显示`和`做动画`的，`视图（UIView）`仅仅是对它的一个`封装`，提供了如下:

- (1) 用户与屏幕，触摸事件响应处理
- (2) frame、bounds、backgroudColor、text、font...赋值给CALayer实例
- (4) 添加/删除UIView层级，同时添加/删除CALayer层级
- (5) 操作CoreAnimation库的高级接口


### 图层（CALayer）除开不能接收事件响应之后，提供了视图（UIView）所不具备的能力

- (1) 阴影、圆角、带颜色的边框
- (2) 3D变换
- (3) 非矩形 范围
- (4) 透明遮罩
- (5) 多级 非线性 动画

### 使用图层（CALayer）的简单代码、在一个UIView对象内直接添加一个CALayer图层来显示

```objc
#import "ViewController.h"
#import <QuartzCore/QuartzCore.h>

@interface ViewController ()
@property (nonatomic, weak) IBOutlet UIView *view;
@end

@implementation ViewController
- (void)viewDidLoad
{
    [super viewDidLoad];
	
	//1. 创建要显示数据的layer
    CALayer *blueLayer = [CALayer layer];
    blueLayer.frame = CGRectMake(50.0f, 50.0f, 100.0f, 100.0f);
    blueLayer.backgroundColor = [UIColor blueColor].CGColor;

	//2. 将layer添加到UIView内部的layer层级中
    [self.view.layer addSublayer:blueLayer];
}
@end
```

一个视图只有一个相关联的图层（自动创建），同时它也可以支持添加无数多个子图层。

通常情况下，并不会单独使用CALayer，然后去`addSublayer:`、`removeFromSuperlayer`...。

而是UIView关联，好处在于你能在使用所有CALayer底层特性的同时，也可以使用UIView的高级API（比如自动排版，布局和事件处理）。

### 使用CALayer而不是UIView的场景

- (1) 开发同时可以在Mac OS上运行的`跨平台`应用
- (2) 使用多种CALayer的子类，并且不想创建额外的UIView去包封装他们
- (3) 做一些对性能特别挑剔的工作，比如:
	- CALayer相比UIView轻量级的多，占用资源也更少
	- 专有的CALayer，能够做很多性能优化（直接OpenGL绘图、文字处理、大图切割...）
	- CALayer异步子线程完成绘图

	
## No3. 寄宿图

CALayer类能够包含一张你喜欢的`图片`，这一章节我们将来探索CALayer的`寄宿图`（CALayer图层中包含的`图`）。

### CALayer对象的contents属性

- (1) contents属性的类型被定义为`id`，意味着它可以是`任何类型`的对象

```c
@property(nullable, strong) id contents;
```

- (2) 可以给contents属性赋任何值，你的app仍然能够编译通过

- (3) 在实践中，如果你给contents赋的不是 `CGImage`类型的实例，那么你得到的图层将是空白的

contents属性的类型，之所以被定义为id类型，是因为在`Mac OS`系统上，这个属性对`CGImage`和`NSImage`类型的值都起作用。

如果你试图在`iOS`平台上将`UIImage`的值赋给它，只能得到一个`空白`的图层。

通过如下bridge转换方式，将UIImage实例设置给layer.contens属性:

```c
layer.contents = (__bridge id)image.CGImage;
```

下面一个例子，直接把UIView的`宿主图层（root layer、backing layer）`的contents属性设置成图片:

```objc
@implementation ViewController

- (void)viewDidLoad
{
  [super viewDidLoad]; 
  
  //1. load an image
  UIImage *image = [UIImage imageNamed:@"Snowman.png"];

  //2. add it directly to our view's layer
  self.layerView.layer.contents = (__bridge id)image.CGImage;
}
@end
```

我们并没有使用`UIImageView`，而是一个普通的`CALayer`，就能够显示出一个图片。

### CALayer对象的contentGravity属性

在`UIImageView`中，有一个叫做`contentMode`属性，来让要显示的图片适应`UIImageView`的尺寸、位置

```objc
view.contentMode = UIViewContentModeScaleAspectFit;
```

其实这个UIView对象的contentMode属性，实际上是对内部的CALayer对象进行操作的。

然而CALayer对象并没有contentMode属性，而是使用另外一个叫做`contentsGravity`属性:

```objc
@property(copy) NSString *contentsGravity;
```

是一个NSString类型，可选的常量值有以下一些:

```c
kCAGravityCenter
kCAGravityTop
kCAGravityBottom
kCAGravityLeft
kCAGravityRight
kCAGravityTopLeft
kCAGravityTopRight
kCAGravityBottomLeft
kCAGravityBottomRight
kCAGravityResize
kCAGravityResizeAspect
kCAGravityResizeAspectFill
```

分别与`UIViewContentMode`这个枚举类型的选项值，一一对应。

与contentMode属性一样，contentsGravity的目的是为了决定`内容`在`图层`的`边界中`怎么`对齐`。

> contentsGravity不同的设置，将会影响`寄宿图（CALayer.contents）`的大小变化。

### CALayer对象的contentsScale属性

contentsScale属性定义了`寄宿图`的像素尺寸和`视图`大小的比例，默认情况下它是一个值为1.0的浮点数。

```c
@property CGFloat contentsScale;
```

如果你尝试对我们的例子设置不同的值，你就会发现根本没任何影响。因为contents由于设置了`contentsGravity`属性，所以它已经被拉伸以适应图层的边界。

如果你只是单纯地想`放大`图层CALayer内部的`contents`图片，你可以通过使用图层的`transform`和`affineTransform`属性来达到这个目的，这(指放大)也不是contentsScale的目的所在.

contentsScale属性其实属于支持`高分辨率`（又称Hi-DPI或Retina）屏幕机制的一部分。

contentsScale属性，用来判断在绘制图像的时候，应该为`寄宿图（图形、图像）`创建的`空间大小`，和需要显示的`图片的拉伸度`（假设并没有设置contentsGravity属性）。UIView有一个类似功能但是非常少用到的contentScaleFactor属性。

如果contentsScale设置为`1.0`，将会以`每个点 == 1个像素`来绘制图片，如果设置为`2.0`，则会以`每个点 == 2个像素`来绘制图片，这就是我们熟知的Retina屏幕。

> 当使用UIImage读取图片文件时，会根据是否是Retina屏幕，来读取Retina版本的高清图片文件。

而如果一旦读取了，Retina版本的高清图片文件之后，将UIImage对象转换成CGImage实例后，图像会失去`拉伸`的处理，会变得很大、边缘有锯齿。

因为，CGImage没有`拉伸`的概念，就根本不会记录图片任何的`拉伸`的信息，就不知道去拉伸图片，就直接把图片展示出来了。

不过我们可以通过手动设置contentsScale来修复这个问题:

```objc
@implementation ViewController

- (void)viewDidLoad
{
  [super viewDidLoad]; 
  
  //1. load an image
  UIImage *image = [UIImage imageNamed:@"Snowman.png"]; 
  
  //2. add it directly to our view's layer
  self.layerView.layer.contents = (__bridge id)image.CGImage; 
  
  //3. center the image
  self.layerView.layer.contentsGravity = kCAGravityCenter;

  //4.【重要】修正CGImage丢失图片拉伸的信息
  self.layerView.layer.contentsScale = image.scale;
}

@end
```

> 当用代码的方式来处理`寄宿图（contents属性值）`的时候，一定要记住要手动的设置图层的contentsScale属性，否则，你的图片在Retina设备上就显示得不正确啦。代码如下：

```objc
layer.contentsScale = [UIScreen mainScreen].scale;
```


### CALayer对象的maskToBounds属性

UIView有一个叫做`clipsToBounds`的属性可以用来决定是否显示超出边界的内容。

CALayer对应的属性叫做`masksToBounds`，把它设置为YES即可。


### CALayer对象的contentsRect属性

contentsRect属性，允许我们在CALayer边框里，显示`寄宿图`的一个`子域`。也就是说，让显示CALayer.contents图像中的哪一个小部分。

注意，并不是对CALayer本身划分区域，而只是针对`CALayer.contents`指向的要显示的图像进行划分区域。

和 `bounds`，`frame`不同，contentsRect不是按`点`来计算的。使用了`单位坐标`，单位坐标指定在`0到1`之间，是一个`相对值`（像素和点就是绝对值），所以它们是相对与寄宿图的尺寸的。

iOS使用了以下的坐标系统：

- (1) 点

```
1. 虚拟的像素，也被称作逻辑像素
2. 在标准设备上，一个点就是一个像素
3. 在Retina设备上，一个点等于2*2个像素
4. iOS用点作为屏幕的坐标测算体系就是为了在Retina设备和普通设备上能有一致的视觉效果
```

- (2) 像素 

```
1. 物理像素坐标并不会用来屏幕布局，但是仍然与图片有相对关系
2. UIImage是一个屏幕分辨率解决方案，所以指定`点`来度量大小
3. 但是一些底层的图片表示如CGImage就会使用像素
```

- (3) 单位

```
1. 对于与图片大小或是图层`边界相关`的显示，单位坐标是一个方便的度量方式
2. 就是一个比例的关系，当大小改变的时候，也不需要再次调整
3. 单位坐标在OpenGL这种纹理坐标系统中用得很多，Core Animation中也用到了单位坐标
```

默认的contentsRect是`{0, 0, 1, 1}`，这意味着整个寄宿图默认都是可见的。

contentsRect在app中最有趣的地方在于一个叫做`image sprites`（图片拼合）的用法。

可以将很多的小图片，拼合到一张大图片，然后一次性的载入到内存。相比多次载入不同的图片，这样做能够带来很多方面的好处：内存使用，载入时间，渲染性能等等。

下面是多个小图片，拼合成一张大图的简单例子代码:

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

- (void)viewDidLoad 
{
  [super viewDidLoad]; 
  
  //1. 读取大图
  UIImage *image = [UIImage imageNamed:@"Sprites.png"];

  //2. 按照比例，取出大图上的某一小部分
  [self addSpriteImage:image withContentRect:CGRectMake(0, 0, 0.5, 0.5) toLayer:self.iglooView.layer];
  
  //3. 按照比例，取出大图上的某一小部分
  [self addSpriteImage:image withContentRect:CGRectMake(0.5, 0, 0.5, 0.5) toLayer:self.coneView.layer];
  
  //4. 按照比例，取出大图上的某一小部分
  [self addSpriteImage:image withContentRect:CGRectMake(0, 0.5, 0.5, 0.5) toLayer:self.anchorView.layer];
 
  //5. 按照比例，取出大图上的某一小部分
  [self addSpriteImage:image withContentRect:CGRectMake(0.5, 0.5, 0.5, 0.5) toLayer:self.shipView.layer];
}
@end
```

Sprites.png拼合了多张小图

<img src="./Sprites.png" alt="" title="" width="700"/>

程序运行的效果

<img src="./Sprites_result.png" alt="" title="" width="700"/>

一个叫做LayerSprites的开源库（https://github.com/nicklockwood/LayerSprites)，它能够读取Cocos2D格式中的拼合图并在普通的Core Animation层中显示出来。

### CALayer对象的contentsCenter属性

contentsCenter属性是一个CGRect，但是使用`单位坐标系（0~1）`，它定义了一个固定的边框和一个在图层上可`拉伸的区域`。

改变contentsCenter的值并不会影响到寄宿图的显示，除非这个图层的大小改变了，你才看得到效果。

默认情况下，contentsCenter是`{0, 0, 1, 1}`，这意味着如果大小（由conttensGravity决定）改变了,那么寄宿图将会均匀地拉伸开。

图示contentsCenter设置为`{0.25, 0.25, 0.5, 0.5}`的效果:

<img src="./contentsCenter1.png" alt="" title="" width="700"/>

可以从右边的带颜色图看到:

- (1) 蓝色区域，水平拉伸
- (2) 红色区域，垂直拉伸
- (3) 绿色区域，既水平拉伸、又垂直拉伸
- (4) 黄色区域，不做任何的拉伸

这意味着我们可以随意重设尺寸，边框仍然会是连续的。

contentsCenter工作起来的效果和`-[UIImage resizableImageWithCapInsets:]` 方法效果非常类似，只是它可以运用到任何寄宿图，甚至包括在Core Graphics运行时绘制的图形。


代码中，使用contentsCenter对图像进行拉伸:

```objc
@interface ViewController ()

@property (nonatomic, weak) IBOutlet UIView *button1;
@property (nonatomic, weak) IBOutlet UIView *button2;

@end

@implementation ViewController

- (void)addStretchableImage:(UIImage *)image withContentCenter:(CGRect)rect toLayer:(CALayer *)layer
{  
  //1. set image
  layer.contents = (__bridge id)image.CGImage;

  //2. set contentsCenter
  layer.contentsCenter = rect;
}

- (void)viewDidLoad
{
  [super viewDidLoad];
  
  //1. load button image
  UIImage *image = [UIImage imageNamed:@"Button.png"];

  //2. set button 1
  [self addStretchableImage:image withContentCenter:CGRectMake(0.25, 0.25, 0.5, 0.5) toLayer:self.button1.layer];

  //3. set button 2
  [self addStretchableImage:image withContentCenter:CGRectMake(0.25, 0.25, 0.5, 0.5) toLayer:self.button2.layer];
}

@end
```

contentsCenter的另一个很酷的特性就是，它可以在Interface Builder里面配置，根本不用写代码:


<img src="./contentsCenter2.png" alt="" title="" width="700"/>


### Custome Drawing

给contents赋CGImage的值不是唯一的设置寄宿图的方法。

可以通过继承自UIView，重写`drawRect:`方法实现，进行自定义绘制。

> 但是关于`drawRect:`的注意点:

- (1) `-drawRect:` 方法没有默认的实现。因为对UIView来说，寄宿图并不是必须的，它不在意那到底是单调的颜色还是有一个图片的实例。

- (2) 如果UIView检测到`-drawRect:` 方法被调用了，它就会为视图分配一个`寄宿图`，这个寄宿图的像素尺寸 = `视图大小 * contentsScale`。

- (3) 当视图在屏幕上出现的时候 `-drawRect:`方法就会被自动调用。

所以，如果你不需要寄宿图，那就不要创建这个方法了，会造成CPU资源和内存的浪费。


在`-drawRect:`方法实现中，可以利用Core Graphics去绘制一个寄宿图，然后内容就会被`缓存`起来直到它需要被更新（通常是因为开发者调用了-setNeedsDisplay方法，尽管影响到表现效果的属性值被更改时，一些视图类型会被自动重绘，如bounds属性）。


> 虽然 `-drawRect:`方法是一个UIView方法，事实上都是底层的CALayer安排了重绘工作和保存此时产生的图像。


CALayer有一个可选的delegate属性，实现了CALayerDelegate协议。

- (1) delegate1、当需要被重绘时，CALayer会请求它的代理给它一个寄宿图来显示

```objc
(void)displayLayer:(CALayer *)layer;
```

这个时候，代理可以自己创建一个寄宿图，然后通过`contents`属性进行设置给CALayer对象，不然没有别的方法可以调用了。

- (1) delegate2、如果代理不实现-displayLayer:方法，CALayer就会转而尝试调用下面这个方法：

```objc
- (void)drawLayer:(CALayer *)layer inContext:(CGContextRef)ctx;
```

在调用这个方法之前，CALayer已经创建了一个合适尺寸的`空寄宿图`（尺寸由bounds和contentsScale决定）和一个`Core Graphics的绘制上下文环境`，为绘制寄宿图做准备，它作为ctx参数传入。


使用CALayerDelegate并做一些绘图工作：

```objc
@implementation ViewController
- (void)viewDidLoad
{
  [super viewDidLoad];
  ￼
  //1. create sublayer
  CALayer *blueLayer = [CALayer layer];
  blueLayer.frame = CGRectMake(50.0f, 50.0f, 100.0f, 100.0f);
  blueLayer.backgroundColor = [UIColor blueColor].CGColor;

  //2. set controller as layer delegate
  blueLayer.delegate = self;

  //3. ensure that layer backing image uses correct scale
  blueLayer.contentsScale = [UIScreen mainScreen].scale; 
  
  //4. add layer to our view
  [self.layerView.layer addSublayer:blueLayer];

  //5. force layer to redraw
  [blueLayer display];//间接调用 displayLayer: 或 drawLayer: inContext:
}

- (void)drawLayer:(CALayer *)layer inContext:(CGContextRef)ctx
{
  // 绘制一个红色的圆圈图形
  CGContextSetLineWidth(ctx, 10.0f); 
  CGContextSetStrokeColorWithColor(ctx, [UIColor redColor].CGColor);
  CGContextStrokeEllipseInRect(ctx, layer.bounds);
}
@end
```

注意一下一些有趣的事情：

- (1) 在blueLayer上显式地调用了`display`。不同于`UIView`，当UIView显示在屏幕上时，`CALayer不会自动重绘它的内容`。它把重绘的决定权交给了`开发者`，所以需要手动通知CALayer执行绘制。

- (2) 尽管我们没有用`masksToBounds`属性，绘制的那个圆仍然沿边界被`裁剪`了。这是因为当使用`CALayerDelegate`绘制`寄宿图`的时候，并`没有`对`超出边界外的内容`提供绘制支持。

只有创建了一个单独的图层，你几乎没有机会用到CALayerDelegate协议。因为当UIView创建了它的backing layer时，它就会自动地把CALayer的delegate设置为它自己，并提供了一个`-displayLayer:`的实现，那所有的问题就都没了。

## 图层几何学

### 布局

UIView有三个比较重要的布局属性：

- (1) frame
- (2) bounds
- (3) center

CALayer对应的三个属性叫做：

- (1) frame
- (2) bounds
- (3) position

为了能清楚区分，CALayer用了`position`，UIView用了`center`，但是他们都代表同样的值。

### frame

代表了图层CALayer在父图层（Super CALayer）上占据的位置和大小。


### bounds

`{0, 0, super.width, super.height}`。x，y默认都是0。

### UIView.center 和 CALayer.position

center和position都代表了当前UIView或CALayer内部区域的`中心点 {width/2.0, height/2.0}`，出现在`super UIView/CALayer`内部的坐标点。

UIView对象的frmae、bounds、center 只是操作CALayer对象属性的 getter/setter。内部其实调用的CALayer的frame、bounds、position属性。

### frame 与 bounds、position、transform 这三个属性值有关系

> 当对CALayer图层做transform变换时，frame实际上代表了覆盖在图层旋转之后的整个轴对齐的矩形区域。

如下图是UIView/CALayer，`没有`做变换之前的frame区域。


<img src="./transform1.jpeg" alt="未做变换时" title="图1.1" width="700"/>

而下图是UIView/CALayer，`做了`变换之前的frame区域。

<img src="./transform2.jpeg" alt="未做变换时" title="图1.1" width="700"/>

可以看到，做了变换后UIView、CALayer占用的frame区域，比之前大一些了，而且x、y、width、height都与之前不一样了。


### 锚点（anchorPoint）

AnchorPoint是一个单位坐标点:

<img src="./anchor_point.png" alt="图1.1" title="图1.1" width="700"/>

关于AnchorPoint的几点:

- (1) 默认来说，anchorPoint位于CALayer的`中心点`，所以CALayer将会以这个点为中心放置

- (2) anchorPoint属性并没有被UIView接口暴露出来，这也是UIView的position属性被叫做`center`的原因

- (3) CALayer的anchorPoint坐标，是可以移动的。可以认为anchorPoint是用来移动图层的把柄。

AnchorPoint使用的例子1、

```objc
@implementation AnchorPointVC

- (void)test1 {
    
    // 1. 灰色
    UIView *view1 = [[UIView alloc] initWithFrame:CGRectMake(80, 200, 100, 100)];
    view1.backgroundColor = [UIColor grayColor];
    view1.layer.anchorPoint = (CGPoint){0.5, 0.5};
    [self.view addSubview:view1];
    NSLog(@"view1.layer.position = %@", NSStringFromCGPoint(view1.layer.position));
    
    // 2. 蓝色
    UIView *view2 = [[UIView alloc] initWithFrame:CGRectMake(80, 200, 100, 100)];
    view2.backgroundColor = [UIColor blueColor];
    view2.layer.anchorPoint = (CGPoint){0, 0};
    [self.view addSubview:view2];
    NSLog(@"view2.layer.position = %@", NSStringFromCGPoint(view2.layer.position));
    
    // 3. 黄色
    UIView *view3 = [[UIView alloc] initWithFrame:CGRectMake(80, 200, 100, 100)];
    view3.backgroundColor = [UIColor yellowColor];
    view3.layer.anchorPoint = (CGPoint){1, 1};
    [self.view addSubview:view3];
    NSLog(@"view3.layer.position = %@", NSStringFromCGPoint(view3.layer.position));
    
    // 4. 绿色
    UIView *view4 = [[UIView alloc] initWithFrame:CGRectMake(80, 200, 100, 100)];
    view4.backgroundColor = [UIColor greenColor];
    view4.layer.position = (CGPoint){CGRectGetMaxX(view1.frame), CGRectGetMinY(view1.frame)};
    view4.layer.anchorPoint = (CGPoint){0, 1};
    [self.view addSubview:view4];
    NSLog(@"view4.layer.position = %@", NSStringFromCGPoint(view4.layer.position));
}

- (void)viewDidLoad {
    [super viewDidLoad];
    self.title = @"AnchorPoint";
    self.view.backgroundColor = [UIColor whiteColor];
    
    [self test1];
}

- (UIRectEdge)edgesForExtendedLayout {
    return UIRectEdgeNone;
}

@end
```

输出如下

```
2017-02-13 22:36:39.649 CoreAnimationDemo[1109:49338] view1.layer.position = {130, 250}
2017-02-13 22:36:39.650 CoreAnimationDemo[1109:49338] view2.layer.position = {130, 250}
2017-02-13 22:36:39.650 CoreAnimationDemo[1109:49338] view3.layer.position = {130, 250}
2017-02-13 22:36:39.651 CoreAnimationDemo[1109:49338] view4.layer.position = {180, 200}
```

注：CALayer的position属性，就等价于UIView的center属性。

从输出上可以看到，前面三个CALayer的position属性值，都是等于{130, 250}。

而只有最后一个由于执行了`view4.layer.position = (CGPoint){CGRectGetMaxX(view1.frame), CGRectGetMinY(view1.frame)};`，导致最后一个CALayer的position变为了{180, 200}。

那说明，前面3个CALayer的position都没变化，而只是按照anchorPoint指向的那一个点，将整个CALayer移动到CALayer的position为{130, 250}这一个坐标点上。

所以，也就是根据anchorPoint指示的CALayer内部的某一个单位点，将CALayer`平移`到CALayer的`position`所示的`frame坐标系点`的位置。

如上代码运行的效果图:

<img src="./transform3.png" alt="图1.1" title="图1.1" width="700"/>

所以，当改变anchorPoint值的时候，position值并没有发生改变，但是`frame却移动`了。


也就是说，我们可以通过修改anchorPoint值，来达到随意的改变CALayer的位置。下面创建一个模拟闹钟的项目：


## 基于CALayer的动画知识框架体系图

<img src="./动画知识框架.png" alt="图1.1" title="图1.1" width="700"/>