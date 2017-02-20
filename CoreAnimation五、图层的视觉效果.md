## 剪裁

```
masksToBounds
```

我们可以沿边界裁剪图形。

<img src="./masksToBounds.png" alt="" title="" width="700"/>

## 圆角

```
cornerRadius
masksToBounds
```

这个会触发GPU的离屏渲染，过多会导致性能降低，可以搜`高性能圆角处理`。

## 边框

```
borderWidth
borderColor
```

## 阴影

```
shadowOpacity
shadowColor
shadowOffset
shadowRadius
```

## shadowPath自定义阴影路径

```objc
@interface ViewController ()

@property (nonatomic, weak) IBOutlet UIView *layerView1;
@property (nonatomic, weak) IBOutlet UIView *layerView2;
@end

@implementation ViewController

- (void)viewDidLoad
{
  [super viewDidLoad];

  //enable layer shadows
  self.layerView1.layer.shadowOpacity = 0.5f;
  self.layerView2.layer.shadowOpacity = 0.5f;

  //create a square shadow
  CGMutablePathRef squarePath = CGPathCreateMutable();
  CGPathAddRect(squarePath, NULL, self.layerView1.bounds);
  self.layerView1.layer.shadowPath = squarePath; CGPathRelease(squarePath);

  ￼//create a circular shadow
  CGMutablePathRef circlePath = CGPathCreateMutable();
  CGPathAddEllipseInRect(circlePath, NULL, self.layerView2.bounds);
  self.layerView2.layer.shadowPath = circlePath; CGPathRelease(circlePath);
}
@end
```

<img src="./shadowPath.png" alt="" title="" width="700"/>

如果是一个矩形或者是圆，用CGPath会相当简单明了。但是如果是更加复杂一点的图形，UIBezierPath类会更合适，它是一个由UIKit提供的在CGPath基础上的Objective-C包装类。


## 图层蒙板

CALayer有一个mask属性，来指定一个CALayer对象:

```c
@property(nullable, strong) CALayer *mask;
```

这个mask图层定义了父图层中`可见`的区域。mask图层的Color属性是无关紧要的，真正重要的是图层的轮廓。mask属性就像是一个饼干切割机，mask图层实心的部分会被保留下来，其他的则会被抛弃。


<img src="./mask_layer.png" alt="" title="" width="700"/>

代码如下:

```objc
@interface ViewController ()

@property (nonatomic, weak) IBOutlet UIImageView *imageView;
@end

@implementation ViewController

- (void)viewDidLoad
{
  [super viewDidLoad];

  //create mask layer
  CALayer *maskLayer = [CALayer layer];
  maskLayer.frame = self.imageView.bounds;
  UIImage *maskImage = [UIImage imageNamed:@"Cone.png"];
  maskLayer.contents = (__bridge id)maskImage.CGImage;

  //apply mask to image layer￼
  self.imageView.layer.mask = maskLayer;
}
@end
```

运行图示:


<img src="./make_layer2.png" alt="" title="" width="700"/>


## 拉伸过滤

主要缩小和放大图片:

- (1) minification（缩小图片）
- (2) magnification（放大图片）

当使用UIView显示一个`图片`的时候，都应该正确地显示这个图片（意即：以正确的比例和正确的1：1像素显示在屏幕上）。原因如下：

- (1) 能够显示最好的画质，像素既没有被`压缩`也没有被`拉伸`（变形）

- (2) 能更好的使用内存，因为这就是所有你要存储的东西

- (3) **最好的性能表现，CPU不需要为此额外的计算**

不过有时候，显示一个非真实大小的图片确实是我们需要的效果。比如说一个头像或是图片的缩略图。

有一种叫做`拉伸过滤`的算法就起到作用了，它作用于`原图的像素`上并根据需要`生成新的像素`显示在屏幕上。CALayer为此提供了三种拉伸过滤方法，他们是：

```
- (1) kCAFilterLinear
- (2) kCAFilterNearest
- (3) kCAFilterTrilinear
```

使用deme:

```c
view.layer.magnificationFilter = kCAFilterNearest;
```

minification（缩小图片）和magnification（放大图片）默认的过滤器都是`kCAFilterLinear`，这个过滤器采用双线性滤波算法，它在大多数情况下都表现`良好`，但是当`放大倍数比较大`的时候图片就`模糊不清`了。

`kCAFilterTrilinear`和`kCAFilterLinear`非常相似，二者都看不出来有什么差别。

> 对于大图来说，双线性滤波和三线性滤波表现得更出色。


## 组透明

如果你给一个图层设置了opacity属性，那它的`子图层`都会受此影响。

iOS常见的做法是把一个空间的alpha值设置为0.5（50%）以使其看上去呈现为不可用状态。对于独立的视图来说还不错，但是当一个控件有子视图的时候就有点奇怪了，图4.20展示了一个内嵌了UILabel的自定义UIButton；左边是一个不透明的按钮，右边是50%透明度的相同按钮。我们可以注意到，里面的标签的轮廓跟按钮的背景很不搭调。

<img src="./group_opacity1.png" alt="" title="" width="700"/>

可以设置CALayer的一个叫做`shouldRasterize`属性（见清单4.7）来实现组透明的效果。

如果它被设置为YES，在应用透明度之前，图层及其子图层都会被整合成一个整体的图片，这样就没有透明度混合的问题了。

为了启用shouldRasterize属性，我们还要设置图层的`rasterizationScale`属性。默认情况下，所有图层拉伸都是1.0， 所以如果你使用了shouldRasterize属性，你就要确保你设置了rasterizationScale属性去匹配屏幕，以防止出现Retina屏幕像素化的问题。

代码实例:

```objc
@interface ViewController ()
@property (nonatomic, weak) IBOutlet UIView *containerView;
@end

@implementation ViewController

- (UIButton *)customButton
{
  //create button
  CGRect frame = CGRectMake(0, 0, 150, 50);
  UIButton *button = [[UIButton alloc] initWithFrame:frame];
  button.backgroundColor = [UIColor whiteColor];
  button.layer.cornerRadius = 10;

  //add label
  frame = CGRectMake(20, 10, 110, 30);
  UILabel *label = [[UILabel alloc] initWithFrame:frame];
  label.text = @"Hello World";
  label.textAlignment = NSTextAlignmentCenter;
  [button addSubview:label];
  return button;
}

- (void)viewDidLoad
{
  [super viewDidLoad];

  //create opaque button
  UIButton *button1 = [self customButton];
  button1.center = CGPointMake(50, 150);
  [self.containerView addSubview:button1];

  //create translucent button
  UIButton *button2 = [self customButton];
  ￼
  button2.center = CGPointMake(250, 150);
  button2.alpha = 0.5;
  [self.containerView addSubview:button2];

  //enable rasterization for the translucent button
  button2.layer.shouldRasterize = YES;
  button2.layer.rasterizationScale = [UIScreen mainScreen].scale;
}
@end
```

运行效果

<img src="./group_opacity2.png" alt="" title="" width="700"/>