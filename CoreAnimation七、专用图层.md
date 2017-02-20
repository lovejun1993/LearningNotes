## 专用图层

到此为止，学习了layer可以显示各种视觉效果、各种酷炫的动画、2d和3d的变换...

但都只是看到了最简单的CALayer类，其实还有很多不同的CALayer子类，各自都可以提供不同的功能，针对每一种场景充分利用，来提高UI的性能。

## 列举下所有的专用图层

- (1) CAShapeLayer
- (2) CATextLayer
- (3) CATransformLayer
- (4) CAGradientLayer
- (5) CAReplicatorLayer
- (6) CAScrollLayer
- (7) CATiledLayer
- (8) CAEmitterLayer
- (9) CAEAGLLayer
- (10) AVPlayerLayer

## CAShapeLayer、是一个通过`矢量图形`而不是`bitmap`来绘制的图层子类

### 在layer中进行绘制路径有两种方法:

- (1) 重写`drawRect: 或 drawInContext:`，用CoreGraphics绘制代码，直接向`原始的CALyer`的寄宿图中绘制一个路径

- (2) 使用CAShapeLayer这个专用图层，然后使用`CGPath`定义要绘制的路径

相比上面两种方法，`(2)`方法性能会更好。

### iOS系统关于图像渲染的层级结构图:

<img src="./layer_rendering.jpeg" alt="" title="" width="700"/>

可以看出：

- (1) `OpenGL`绘制图像，会交给`GPU`完成渲染
- (2) `CoreGraphics`绘制图像，会交给`CPU`完成渲染（**而CPU执行渲染时，是做的off-screen-render，非常影响性能**）

### CPU与GPU的强项与弱势：

- (1) CPU、对数据的计算处理相当快，但是对于图像的渲染很差
- (2) GPU、有很多核心来同时做图像的渲染，所以很快。但是对于数据的计算处理，是很慢的


所以，一定要充分利用GPU与CPU的强项：

```
CPU >>> 大量进行数据计算，少进行图像渲染
GPU >>> 大量进行图像渲染，少进行数据计算
```

所以，最好让CPU只做一些`数据计算`，而CPU只会`图像渲染`，这样整体性能会提升很多。

### 然后再看具体原因:

- (1) 渲染快速。

CAShapeLayer使用了`硬件加速（直接由GPU绘制）`，绘制同一图形会比用`CoreGraphics（由CPU绘制，进行CPU离屏渲染）`快很多。

CPU离屏渲染，会导致CPU做大量的`图像渲染`任务，而CPU本身并不是用来`图像渲染`的，而是用来`数据计算`的。

所以，将大量的图像渲染代码交给`专用图层`来完成，而`专用图层`使用OpenGL操作`GPU`来完成图像的渲染。

`GPU`的`图像渲染`能力，远远大于CPU，但是CPU的`数据计算`能力，又远远大于GPU。

- (2) 高效使用内存。

一个CAShapeLayer不需要像普通CALayer一样创建一个寄宿图形，所以无论有多大，都不会占用太多的内存。

- (3) 不会被图层边界剪裁掉。

一个CAShapeLayer可以在边界之外绘制。你的图层路径不会像在使用Core Graphics的普通CALayer一样被剪裁掉。

- (4) 不会出现像素化。

当你给CAShapeLayer做3D变换时，它不像一个有寄宿图的普通图层一样变得像素化。

### 用CAShapeLayer + UIBezirePath（对CGPathRef的高度封装）绘制一个火柴人

```objc
#import "DrawingView.h"
#import <QuartzCore/QuartzCore.h>

@interface ViewController ()

@property (nonatomic, weak) IBOutlet UIView *containerView;

@end

@implementation ViewController

- (void)viewDidLoad
{
  [super viewDidLoad];

  //1. create path
  UIBezierPath *path = [[UIBezierPath alloc] init];
  
  //2. start point
  [path moveToPoint:CGPointMake(175, 100)];
  ￼
  //3. add toute（路径）
  [path addArcWithCenter:CGPointMake(150, 100) radius:25 startAngle:0 endAngle:2*M_PI clockwise:YES];
  [path moveToPoint:CGPointMake(150, 125)];
  [path addLineToPoint:CGPointMake(150, 175)];
  [path addLineToPoint:CGPointMake(125, 225)];
  [path moveToPoint:CGPointMake(150, 175)];
  [path addLineToPoint:CGPointMake(175, 225)];
  [path moveToPoint:CGPointMake(100, 150)];
  [path addLineToPoint:CGPointMake(200, 150)];

  //4. create shape layer
  CAShapeLayer *shapeLayer = [CAShapeLayer layer];
  
  //5. 给layer设置各种显示的效果
  shapeLayer.strokeColor = [UIColor redColor].CGColor;
  shapeLayer.fillColor = [UIColor clearColor].CGColor;
  shapeLayer.lineWidth = 5;
  shapeLayer.lineJoin = kCALineJoinRound;
  shapeLayer.lineCap = kCALineCapRound;
  
  //6. 将弄好的路径设置给layer
  shapeLayer.path = path.CGPath;
  
  //7. add it to our view
  [self.containerView.layer addSublayer:shapeLayer];
}
@end
```

<img src="./专用图层1.png" alt="" title="" width="700"/>

### CAShapeLayer的圆角、优势就是可以单独指定每个角

我们创建圆角矩形其实就是人工绘制单独的直线和弧度，但是事实上UIBezierPath有自动绘制圆角矩形的构造方法。

下面这段代码绘制了一个有`三个圆角`+`一个直角`的矩形：

```objc
//1. define path parameters
CGRect rect = CGRectMake(50, 50, 100, 100);

//2.
CGSize radii = CGSizeMake(20, 20);

//3. 只需要画三个圆角，剩下的一个为直角
UIRectCorner corners = UIRectCornerTopRight | UIRectCornerBottomRight | UIRectCornerBottomLeft;

//4. create path
UIBezierPath *path = [UIBezierPath bezierPathWithRoundedRect:rect byRoundingCorners:corners cornerRadii:radii];
```

如上只是在父亲layer中，`添加`绘制出一个路径，但是可以被其他的sublayer绘制的路径所擦除。

如果我们想依照此layer来`剪裁`内容，我们可以把CAShapeLayer作为视图的`宿主图层`，而不是添加一个子视图:

```objc
@interface MyView : UIView
@end
@implementation MyView
- (CALayer *)layer {
    return CAShapeLayer对象;
}
@end
```

## CATextLayer、以图层的形式包含了UILabel几乎所有的绘制特性，并且额外提供了一些新的特性

### CATextLayer的优点:

要比UILabel渲染文字内容得`快得多`，因为CATextLayer使用了`CoreText`进行文件渲染。

iOS6及之前，UILabel使用的是WebKit进行文字渲染，显然会慢很多。而CoreText是一专门用来文字渲染成图像的高性能框架。

### 用CATextLayer来实现一个UILabel

```objc
@interface ViewController ()
@property (nonatomic, weak) IBOutlet UIView *labelView;
@end

@implementation ViewController
- (void)viewDidLoad
{
  [super viewDidLoad];

  //1. create a text layer
  CATextLayer *textLayer = [CATextLayer layer];
  
  //2.
  textLayer.frame = self.labelView.bounds;
  
  //3.
  [self.labelView.layer addSublayer:textLayer];

  //4. set text attributes
  textLayer.foregroundColor = [UIColor blackColor].CGColor;
  textLayer.alignmentMode = kCAAlignmentJustified;
  textLayer.wrapped = YES;

  //5. choose a font
  UIFont *font = [UIFont systemFontOfSize:15];

  //6. set layer font（UIFont转换成CGFontRef）
  CFStringRef fontName = (__bridge CFStringRef)font.fontName;
  CGFontRef fontRef = CGFontCreateWithFontName(fontName);
  textLayer.font = fontRef;
  textLayer.fontSize = font.pointSize;
  CGFontRelease(fontRef);

  //7. choose some text
  NSString *text = @"Lorem ipsum dolor sit amet, consectetur adipiscing \ elit. Quisque massa arcu, eleifend vel varius in, facilisis pulvinar \ leo. Nunc quis nunc at mauris pharetra condimentum ut ac neque. Nunc elementum, libero ut porttitor dictum, diam odio congue lacus, vel \ fringilla sapien diam at purus. Etiam suscipit pretium nunc sit amet \ lobortis";

  //8. 设置显示普通文字内容
  textLayer.string = text;
}
@end
```

<img src="./专用图层2.png" alt="" title="" width="700"/>

> 仔细看这个文本，你会发现一个奇怪的地方：这些文本有一些像素化了。

因为并没有以Retina的方式渲染，要使用layer的`contentScale`属性，用来决定图层内容应该以怎样的分辨率来渲染:

```c
textLayer.contentsScale = [UIScreen mainScreen].scale;
```

加上上面那句代码之后，就不会出现边缘像素化了。

<img src="./专用图层3.png" alt="" title="" width="700"/>

### CATextLayer还是可以设置`NSAttributedString`富文本文字内容，避免去直接使用CoreText的复杂函数

```objc
#import "DrawingView.h"
#import <QuartzCore/QuartzCore.h>
#import <CoreText/CoreText.h>

@interface ViewController ()

@property (nonatomic, weak) IBOutlet UIView *labelView;

@end

@implementation ViewController

- (void)viewDidLoad
{
  [super viewDidLoad];

  //1. create a text layer
  CATextLayer *textLayer = [CATextLayer layer];
  
  //2.
  textLayer.frame = self.labelView.bounds;
  
  //3.【重要】避免渲染时出现像素化
  textLayer.contentsScale = [UIScreen mainScreen].scale;
  
  //4.
  [self.labelView.layer addSublayer:textLayer];

  //5. set text attributes
  textLayer.alignmentMode = kCAAlignmentJustified;
  textLayer.wrapped = YES;

  //6. choose a font
  UIFont *font = [UIFont systemFontOfSize:15];

  //7. choose some text
  NSString *text = @"Lorem ipsum dolor sit amet, consectetur adipiscing \ elit. Quisque massa arcu, eleifend vel varius in, facilisis pulvinar \ leo. Nunc quis nunc at mauris pharetra condimentum ut ac neque. Nunc \ elementum, libero ut porttitor dictum, diam odio congue lacus, vel \ fringilla sapien diam at purus. Etiam suscipit pretium nunc sit amet \ lobortis";
  ￼
  //8. create attributed string
  NSMutableAttributedString *string = nil;
  string = [[NSMutableAttributedString alloc] initWithString:text];

  //9. convert UIFont to a CTFont
  CFStringRef fontName = (__bridge CFStringRef)font.fontName;
  CGFloat fontSize = font.pointSize;
  CTFontRef fontRef = CTFontCreateWithName(fontName, fontSize, NULL);

  //10. set text attributes
  NSDictionary *attribs = @{
    (__bridge id)kCTForegroundColorAttributeName:(__bridge id)[UIColor blackColor].CGColor,
    (__bridge id)kCTFontAttributeName: (__bridge id)fontRef
  };

  //11. 
  [string setAttributes:attribs range:NSMakeRange(0, [text length])];
  attribs = @{
    (__bridge id)kCTForegroundColorAttributeName: (__bridge id)[UIColor redColor].CGColor,
    (__bridge id)kCTUnderlineStyleAttributeName: @(kCTUnderlineStyleSingle),
    (__bridge id)kCTFontAttributeName: (__bridge id)fontRef
  };
  
  //12.
  [string setAttributes:attribs range:NSMakeRange(6, 5)];

  //13. release the CTFont we created earlier
  CFRelease(fontRef);

  //14. set layer text
  textLayer.string = string;
}
@end
```

<img src="./专用图层4.png" alt="" title="" width="700"/>

> 注意：由于绘制的实现机制不同（CoreText和WebKit），用CATextLayer渲染和用UILabel渲染出的文本行距和字距也不是不尽相同的。

### 单独的使用CATextLayer去替代UILabel？

CALayer不支持自动缩放和自动布局，所以每次view大小被更改，我们不得不手动更新子layer的边。

我们真正想要的是一个用CATextLayer作为宿主图层的UILabel子类，这样就可以随着视图自动调整大小而且也没有冗余的寄宿图啦。

### 让CATextLayer子类作为UILabel的寄宿图层

```objc
#import "LayerLabel.h"
#import <QuartzCore/QuartzCore.h>

@implementation LayerLabel

//1. 
+ (Class)layerClass
{
  //this makes our label create a CATextLayer //instead of a regular CALayer for its backing layer
  return [CATextLayer class];
}

//2.
- (CATextLayer *)textLayer
{
	// 直接将系统创建的self.lay强转
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

运行代码，你会发现文本并没有像素化，而我们也没有设置`contentsScale`属性。把CATextLayer作为宿主图层的另一好处就是视图自动设置了`contentsScale`属性。

## CATransformLayer、不同于普通的CALayer，因为它不能显示它自己的内容。只有当存在了一个能作用于子图层的变换它才真正存在，用于构造一个层级的3D结构

```objc
@interface ViewController ()

@property (nonatomic, weak) IBOutlet UIView *containerView;

@end

@implementation ViewController

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
  //create cube layer
  CATransformLayer *cube = [CATransformLayer layer];

  //add cube face 1
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

  //set up the transform for cube 1 and add it
  CATransform3D c1t = CATransform3DIdentity;
  c1t = CATransform3DTranslate(c1t, -100, 0, 0);
  CALayer *cube1 = [self cubeWithTransform:c1t];
  [self.containerView.layer addSublayer:cube1];

  //set up the transform for cube 2 and add it
  CATransform3D c2t = CATransform3DIdentity;
  c2t = CATransform3DTranslate(c2t, 100, 0, 0);
  c2t = CATransform3DRotate(c2t, -M_PI_4, 1, 0, 0);
  c2t = CATransform3DRotate(c2t, -M_PI_4, 0, 1, 0);
  CALayer *cube2 = [self cubeWithTransform:c2t];
  [self.containerView.layer addSublayer:cube2];
}
@end
```

<img src="./专用图层5.png" alt="" title="" width="700"/>

可以看到，在同一视角下，存在2个不同变换的立方体，这就是CATransformLayer的好处。


## CAGradientLayer、生成两种或更多颜色平滑渐变

同样也可以用Core Graphics复制一个CAGradientLayer并将内容绘制到一个普通图层的寄宿图也是有可能的，但是CAGradientLayer的真正好处在于绘制使用了硬件加速。

前面已经说了，CoreGraphics绘制图像会触发`CPU的离屏渲染`，影响CPU的执行效率。

CAGradientLayer也有startPoint和endPoint属性，他们决定了渐变的方向。

### 简单的两种颜色的对角线渐变

```objc
@interface ViewController ()

@property (nonatomic, weak) IBOutlet UIView *containerView;

@end

@implementation ViewController

- (void)viewDidLoad
{
  [super viewDidLoad];
  //create gradient layer and add it to our container view
  CAGradientLayer *gradientLayer = [CAGradientLayer layer];
  gradientLayer.frame = self.containerView.bounds;
  [self.containerView.layer addSublayer:gradientLayer];

  //set gradient colors
  gradientLayer.colors = @[(__bridge id)[UIColor redColor].CGColor, (__bridge id)[UIColor blueColor].CGColor];

  //set gradient start and end points
  gradientLayer.startPoint = CGPointMake(0, 0);
  gradientLayer.endPoint = CGPointMake(1, 1);
}
@end
```

<img src="./专用图层6.png" alt="" title="" width="700"/>

### 多重渐变、用locations构造偏移至左上角的三色渐变

colors属性可以包含很多颜色，所以创建一个彩虹一样的多重渐变也是很简单的。默认情况下，这些颜色在空间上均匀地被渲染，但是我们可以用locations属性来调整空间。locations属性是一个浮点数值的数组（以NSNumber包装）。这些浮点数定义了colors属性中每个不同颜色的位置，同样的，也是以单位坐标系进行标定。0.0代表着渐变的开始，1.0代表着结束。

locations数组并不是强制要求的，但是如果你给它赋值了就一定要确保locations的数组大小和colors数组大小一定要相同，否则你将会得到一个空白的渐变。

清单6.7展示了一个基于清单6.6的对角线渐变的代码改造。现在变成了从红到黄最后到绿色的渐变。locations数组指定了0.0，0.25和0.5三个数值，这样这三个渐变就有点像挤在了左上角。

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

<img src="./专用图层7.png" alt="" title="" width="700"/>

## CAReplicatorLayer、高效生成许多相似的图层

使用CAReplicatorLayer绘制出一个样式的layer后，可以继续用这个layer重复的绘制多个一样样式的layer出来，并且都是在`前一个`layer的transform基础上做出额外的变换。

`instanceCount`属性指定了图层需要重复多少次。`instanceTransform`指定了一个CATransform3D3D变换。

变换是`逐步增加`的，每个实例都是`相对于前一实例`布局。这就是为什么这些复制体最终不会出现在同意位置上。

### 用CAReplicatorLayer绘制重复图层

```objc
@interface ViewController ()

@property (nonatomic, weak) IBOutlet UIView *containerView;

@end

@implementation ViewController
- (void)viewDidLoad
{
    [super viewDidLoad];
    
    //1. create a replicator layer and add it to our view
    CAReplicatorLayer *replicator = [CAReplicatorLayer layer];
    replicator.frame = self.containerView.bounds;
    [self.containerView.layer addSublayer:replicator];

    //2. 设置重复绘制10个layer
    replicator.instanceCount = 10;

    //3. 设置每一个layer相对前一个layer的transform变换
    CATransform3D transform = CATransform3DIdentity;
    transform = CATransform3DTranslate(transform, 0, 200, 0);
    transform = CATransform3DRotate(transform, M_PI / 5.0, 0, 0, 1);
    transform = CATransform3DTranslate(transform, 0, -200, 0);
    replicator.instanceTransform = transform;

    //4. 设置每一个layer相对前一个layer的，蓝色和绿色通道递减值
    replicator.instanceBlueOffset = -0.1;
    replicator.instanceGreenOffset = -0.1;

    //5. create a sublayer and place it inside the replicator
    CALayer *layer = [CALayer layer];
    layer.frame = CGRectMake(100.0f, 100.0f, 100.0f, 100.0f);
    layer.backgroundColor = [UIColor whiteColor].CGColor;
    [replicator addSublayer:layer];
}

@end
```

注意到当图层在重复的时候，他们的颜色也在变化：这是用instanceBlueOffset和instanceGreenOffset属性实现的。通过逐步减少蓝色和绿色通道，我们逐渐将图层颜色转换成了红色。

实际场景为，一个子弹、导弹、飞机的`运行轨迹`。

<img src="./专用图层8.png" alt="" title="" width="700"/>


### 用CAReplicatorLayer 绘制一个`负比例`变换于一个复制图层（例如：水平面的倒影）

```objc
#import "ReflectionView.h"
#import <QuartzCore/QuartzCore.h>

@implementation ReflectionView

// 【重要】返回 CAReplicatorLayer
+ (Class)layerClass
{
    return [CAReplicatorLayer class];
}

- (void)setUp
{
    //1. configure replicator
    CAReplicatorLayer *layer = (CAReplicatorLayer *)self.layer;
    
    //2. 绘制2个重复的图层
    layer.instanceCount = 2;

    //3. 第二个相对第一个进行负比例的变换
    CATransform3D transform = CATransform3DIdentity;
    CGFloat verticalOffset = self.bounds.size.height + 2;
    transform = CATransform3DTranslate(transform, 0, verticalOffset, 0);
    transform = CATransform3DScale(transform, 1, -1, 0);
    layer.instanceTransform = transform;

    //4. 透明度值得递减
    layer.instanceAlphaOffset = -0.6;
}
￼
- (id)initWithFrame:(CGRect)frame
{
    //this is called when view is created in code
    if ((self = [super initWithFrame:frame])) {
        [self setUp];
    }
    return self;
}

- (void)awakeFromNib
{
    //this is called when view is created from a nib
    [self setUp];
}
@end
```

<img src="./专用图层9.png" alt="" title="" width="700"/>

## CAScrollLayer

类似UIScrollView的效果。

## CATiledLayer、优化载入一个大图

所有显示在屏幕上的`图片`最终都会被转化为`OpenGL纹理`，同时OpenGL有一个最大的纹理尺寸（通常是`2048*2048`，或`4096*4096`，这个取决于设备型号）。
 
### CATiledLayer为载入`大图`造成的性能问题提供了一个解决方案：: 将大图分解成小片然后将他们单独按需载入。让我们用实验来证明一下。

第一步、写一段MacOS代码，将一个大图切割成多个小图，并存储到不同的文件中:

```objc
#import <AppKit/AppKit.h>

int main(int argc, const char * argv[])
{
    @autoreleasepool{
        ￼//handle incorrect arguments
        if (argc < 2) {
            NSLog(@"TileCutter arguments: inputfile");
            return 0;
        }

        //input file
        NSString *inputFile = [NSString stringWithCString:argv[1] encoding:NSUTF8StringEncoding];

        //tile size
        CGFloat tileSize = 256; //output path
        NSString *outputPath = [inputFile stringByDeletingPathExtension];

        //load image
        NSImage *image = [[NSImage alloc] initWithContentsOfFile:inputFile];
        NSSize size = [image size];
        NSArray *representations = [image representations];
        if ([representations count]){
            NSBitmapImageRep *representation = representations[0];
            size.width = [representation pixelsWide];
            size.height = [representation pixelsHigh];
        }
        NSRect rect = NSMakeRect(0.0, 0.0, size.width, size.height);
        CGImageRef imageRef = [image CGImageForProposedRect:&rect context:NULL hints:nil];

        //calculate rows and columns
        NSInteger rows = ceil(size.height / tileSize);
        NSInteger cols = ceil(size.width / tileSize);

        //generate tiles
        for (int y = 0; y < rows; ++y) {
            for (int x = 0; x < cols; ++x) {
            //extract tile image
            CGRect tileRect = CGRectMake(x*tileSize, y*tileSize, tileSize, tileSize);
            CGImageRef tileImage = CGImageCreateWithImageInRect(imageRef, tileRect);

            //convert to jpeg data
            NSBitmapImageRep *imageRep = [[NSBitmapImageRep alloc] initWithCGImage:tileImage];
            NSData *data = [imageRep representationUsingType: NSJPEGFileType properties:nil];
            CGImageRelease(tileImage);

            //save file
            NSString *path = [outputPath stringByAppendingFormat: @"_%02i_%02i.jpg", x, y];
            [data writeToFile:path atomically:NO];
            }
        }
    }
    return 0;
}
```

- (1) 这个程序将`2048*2048`分辨率的雪人图案裁剪成了`64`个不同的`256*256`的小图

- (2) `256*256`是CATiledLayer的`默认小图大小`，默认大小可以通过`tileSize`属性更改

运行后，终端输入:

```
path/to/TileCutterApp path/to/Snowman.jpg
```


运行结果是64个新图的序列，如下面命名：

```
Snowman_00_00.jpg
Snowman_00_01.jpg
Snowman_00_02.jpg
...
Snowman_07_07.jpg
```

第二步、CATiledLayer结合UIScrollView，不断地显示某一小块图片

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
    
    
    //1. create tiled layer
    CATiledLayer *tileLayer = [CATiledLayer layer];￼
    
    //2. 设置tiled layer的大小，2048*2048，超出到屏幕之外了
    tileLayer.frame = CGRectMake(0, 0, 2048, 2048);
    
    //3. 回调 drawLayer:inContext:，用来绘制小图片
    tileLayer.delegate = self; 
    
    //4. 将tiled layer添加到 UIScrollView中
    [self.scrollView.layer addSublayer:tileLayer];

    //5. 设置scrollView的大小==tiled layer的大小
    self.scrollView.contentSize = tileLayer.frame.size;

    //6. draw layer
    [tileLayer setNeedsDisplay];
}

- (void)drawLayer:(CATiledLayer *)layer inContext:(CGContextRef)ctx
{
    //1. determine tile coordinate
    CGRect bounds = CGContextGetClipBoundingBox(ctx);
    
    //2. 计算x,y
    NSInteger x = floor(bounds.origin.x / layer.tileSize.width);
    NSInteger y = floor(bounds.origin.y / layer.tileSize.height);

    //3. 根据x,y去加载对应的小图片
    NSString *imageName = [NSString stringWithFormat: @"Snowman_%02i_%02i", x, y];
    
    //4. 
    NSString *imagePath = [[NSBundle mainBundle] pathForResource:imageName ofType:@"jpg"];
    
    //5. 
    UIImage *tileImage = [UIImage imageWithContentsOfFile:imagePath];

    //6. draw tile
    UIGraphicsPushContext(ctx);
    [tileImage drawInRect:bounds];
    UIGraphicsPopContext();
}
@end
```

<img src="./专用图层10.png" alt="" title="" width="700"/>

> 注意：CATiledLayer（不同于大部分的UIKit和Core Animation方法）支持`多线程`绘制，`-drawLayer:inContext:`方法可以在`多个线程`中同时地`并发`调用。

所以请小心谨慎地确保你在这个方法中实现的绘制代码是线程安全的，可以使用CoreGraphics代码进行绘制，因为`Core.....`的类库基本上线程安全的。


也许已经注意到了这些小图并不是以Retina的分辨率显示的。为了以屏幕的原生分辨率来渲染CATiledLayer，我们需要设置图层的contentsScale来匹配UIScreen的scale属性：

```
tileLayer.contentsScale = [UIScreen mainScreen].scale;
```

但可以发现是没有任何效果，依然还是比较模糊。其实，`tileSize是以像素为单位`，而不是`点`。

所以增大了contentsScale就自动有了默认的小图尺寸（现在它是`128*128`的点而不是`256*256`）。所以，我们不需要手工更新小图的尺寸或是在Retina分辨率下指定一个不同的小图。我们需要做的是适应小图渲染代码以对应安排scale的变化

```objc
//1. 
CGRect bounds = CGContextGetClipBoundingBox(ctx);

//2. 获取点转换像素的比例
CGFloat scale = [UIScreen mainScreen].scale;

//3. 按照比例计算像素
NSInteger x = floor(bounds.origin.x / layer.tileSize.width * scale);
NSInteger y = floor(bounds.origin.y / layer.tileSize.height * scale);
```

通过这个方法纠正scale也意味着我们的雪人图将以一半的大小渲染在Retina设备上（总尺寸是`1024*1024`，而不是`2048*2048`）。这个通常都不会影响到用CATiledLayer正常显示的图片类型（比如照片和地图，他们在设计上就是要支持放大缩小，能够在不同的缩放条件下显示），但是也需要在心里明白。

## CAEmitterLayer、高性能的粒子引擎，被用来创建实时例子动画如：烟雾，火，雨等等这些效果

暂时用不到...

## CAEAGLLayer、直接使用OpenGL底层库，进行图形的绘制，其性能比使用UIKit、CoreAnimation高很多，那是也麻烦更多


### 用CAEAGLLayer绘制一个三角形

第一步、将`GLKit`和`OpenGLES`框架加入到你的项目中

第二步、使用了OpenGL ES 2.0 的绘图上下文，并渲染了一个有色三角

```objc
#import "ViewController.h"
#import <QuartzCore/QuartzCore.h>
#import <GLKit/GLKit.h>

@interface ViewController ()

@property (nonatomic, weak) IBOutlet UIView *glView;
@property (nonatomic, strong) EAGLContext *glContext;
@property (nonatomic, strong) CAEAGLLayer *glLayer;
@property (nonatomic, assign) GLuint framebuffer;
@property (nonatomic, assign) GLuint colorRenderbuffer;
@property (nonatomic, assign) GLint framebufferWidth;
@property (nonatomic, assign) GLint framebufferHeight;
@property (nonatomic, strong) GLKBaseEffect *effect;
￼
@end

@implementation ViewController

// 创建绘图时使用的缓冲数据区域
- (void)setUpBuffers
{
    //set up frame buffer
    glGenFramebuffers(1, &_framebuffer);
    glBindFramebuffer(GL_FRAMEBUFFER, _framebuffer);

    //set up color render buffer
    glGenRenderbuffers(1, &_colorRenderbuffer);
    glBindRenderbuffer(GL_RENDERBUFFER, _colorRenderbuffer);
    glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_RENDERBUFFER, _colorRenderbuffer);
    [self.glContext renderbufferStorage:GL_RENDERBUFFER fromDrawable:self.glLayer];
    glGetRenderbufferParameteriv(GL_RENDERBUFFER, GL_RENDERBUFFER_WIDTH, &_framebufferWidth);
    glGetRenderbufferParameteriv(GL_RENDERBUFFER, GL_RENDERBUFFER_HEIGHT, &_framebufferHeight);

    //check success
    if (glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE) {
        NSLog(@"Failed to make complete framebuffer object: %i", glCheckFramebufferStatus(GL_FRAMEBUFFER));
    }
}

// 删除绘图时使用的缓冲数据区域
- (void)tearDownBuffers
{
    if (_framebuffer) {
        //delete framebuffer
        glDeleteFramebuffers(1, &_framebuffer);
        _framebuffer = 0;
    }

    if (_colorRenderbuffer) {
        //delete color render buffer
        glDeleteRenderbuffers(1, &_colorRenderbuffer);
        _colorRenderbuffer = 0;
    }
}

// 绘制图形
- (void)drawFrame {
    //bind framebuffer & set viewport
    glBindFramebuffer(GL_FRAMEBUFFER, _framebuffer);
    glViewport(0, 0, _framebufferWidth, _framebufferHeight);

    //bind shader program
    [self.effect prepareToDraw];

    //clear the screen
    glClear(GL_COLOR_BUFFER_BIT); glClearColor(0.0, 0.0, 0.0, 1.0);

    //set up vertices
    GLfloat vertices[] = {
        -0.5f, -0.5f, -1.0f, 0.0f, 0.5f, -1.0f, 0.5f, -0.5f, -1.0f,
    };

    //set up colors
    GLfloat colors[] = {
        0.0f, 0.0f, 1.0f, 1.0f, 0.0f, 1.0f, 0.0f, 1.0f, 1.0f, 0.0f, 0.0f, 1.0f,
    };

    //draw triangle
    glEnableVertexAttribArray(GLKVertexAttribPosition);
    glEnableVertexAttribArray(GLKVertexAttribColor);
    glVertexAttribPointer(GLKVertexAttribPosition, 3, GL_FLOAT, GL_FALSE, 0, vertices);
    glVertexAttribPointer(GLKVertexAttribColor,4, GL_FLOAT, GL_FALSE, 0, colors);
    glDrawArrays(GL_TRIANGLES, 0, 3);

    //present render buffer
    glBindRenderbuffer(GL_RENDERBUFFER, _colorRenderbuffer);
    [self.glContext presentRenderbuffer:GL_RENDERBUFFER];
}

- (void)viewDidLoad
{
    [super viewDidLoad];

    //1. set up context
    self.glContext = [[EAGLContext alloc] initWithAPI: kEAGLRenderingAPIOpenGLES2];
    [EAGLContext setCurrentContext:self.glContext];

    //2. set up layer
    self.glLayer = [CAEAGLLayer layer];
    
    //3. 
    self.glLayer.frame = self.glView.bounds;
    
    //4.
    [self.glView.layer addSublayer:self.glLayer];
    
    //5. 
    self.glLayer.drawableProperties = @{kEAGLDrawablePropertyRetainedBacking:@NO, kEAGLDrawablePropertyColorFormat: kEAGLColorFormatRGBA8};

    //6. set up base effect
    self.effect = [[GLKBaseEffect alloc] init];

    //7. set up buffers
    [self setUpBuffers];

    //8. draw frame
    [self drawFrame];
}

- (void)viewDidUnload
{
    [self tearDownBuffers];
    [super viewDidUnload];
}

- (void)dealloc
{
	// 停止绘制
    [self tearDownBuffers];
    
    // 释放context
    [EAGLContext setCurrentContext:nil];
}
@end
```

> 在一个真正的OpenGL应用中，我们可能会用`NSTimer`或`CADisplayLink`周期性地`每秒`钟调用`-drawRrame`方法`60`次（`FPS=60`），同时会将几何图形生成和绘制分开以便不会每次都重新生成三角形的顶点。


<img src="./专用图层11.png" alt="" title="" width="700"/>