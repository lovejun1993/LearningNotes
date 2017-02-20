	
## 寄宿图

CALayer类能够包含一张你喜欢的`图片`，而寄宿图正是CALayer中要显示的`图像`。

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
