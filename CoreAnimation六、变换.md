## 变换

前面的`《图层视觉效果》`中介绍了一些layer的显示效果，这一章学习layer的各种变换：

- CGAffineTransform >>> 2d变换
	- (1) 旋转
	- (2) 扭曲
	- (3) 移动
	- (4) 缩小、放大

- CATransform3D >>> 3d变换
	- 扁平物体转换成三维空间对象

## UIView的transform属性，只能做2d（二维）效果的的变换

UIView的transform属性是一个`CGAffineTransform`类型，用于在`二维空间`做`旋转`、`缩放`、`平移`。

`CGAffineTransform`是一个可以和二维空间向量（例如CGPoint）做乘法的3X2的矩阵

> CALayer同样也有一个transform属性，但它的类型是CATransform3D，而不是CGAffineTransform。

`CALayer`对应于UIView的transform属性叫做`affineTransform`。

## 使用affineTransform对图层做了45度顺时针旋转的例子

```objc
@interface ViewController ()

@property (nonatomic, weak) IBOutlet UIView *layerView;

@end

@implementation ViewController

- (void)viewDidLoad
{
    [super viewDidLoad];

    //1. rotate the layer 45 degrees
    CGAffineTransform transform = CGAffineTransformMakeRotation(M_PI_4);
    
    //2. add transform to layer
    self.layerView.layer.affineTransform = transform;
}

@end
```

<img src="./transform1.png" alt="" title="" width="700"/>

注意我们使用的旋转常量是`M_PI_4`，而不是你想象的`45`。这是因为iOS的变换函数参数，使用的是`弧度`而不是`角度`作为单位。


`弧度`用数学常量pi的倍数表示，一个pi代表180度，所以四分之一的pi就是45度。

### 弧度(radians) 与 角度(degree) 的换算

角度 >>>> 弧度:

```
#define DEGREES_TO_RADIANS(x) ((x)/180.0*M_PI)
```

弧度 >>>> 角度:

```
#define RADIANS_TO_DEGREES(x) ((x) / M_PI*180.0) 
```

## 混合变换

`CoreGraphics`提供了一系列的函数可以在一个`变换的基础上`做更深层次的变换，如果做一个`既要缩放又要旋转`的变换，这就会非常有用了。例如下面几个函数：

```c
//1. 在之前transform基础上，继续旋转某个角度
CGAffineTransform CGAffineTransformRotate(CGAffineTransform t, CGFloat angle)     

//2. 在之前transform基础上，x和y方向缩放倍数
CGAffineTransform CGAffineTransformScale(CGAffineTransform t, CGFloat sx, CGFloat sy)

//3. 在之前transform基础上，再加上一点平移
CGAffineTransform CGAffineTransformTranslate(CGAffineTransform t, CGFloat tx, CGFloat ty)
```

> 当操纵一个变换的时候，初始生成一个什么都不做的变换很重要。

也就是创建一个CGAffineTransform类型的空值，矩阵论中称作`单位矩阵`，Core Graphics同样也提供了一个方便的常量：

```c
CGAffineTransformIdentity
```

最后，如果需要`混合`两个已经存在的变换矩阵，就可以使用如下方法，在两个变换的基础上`创建一个新的`变换：

```c
CGAffineTransform CGAffineTransformConcat(CGAffineTransform t1,
  CGAffineTransform t2) 
```

## 们来用这些函数组合一个更加复杂的变换，先缩小50%，再旋转30度，最后向右移动200个像素的例子

```objc
- (void)viewDidLoad
{
    [super viewDidLoad]; 
    
    //1. create a new transform 
    CGAffineTransform transform = CGAffineTransformIdentity; 
    
    //2. scale by 50%
    transform = CGAffineTransformScale(transform, 0.5, 0.5); 
    
    //3. rotate by 30 degrees
    transform = CGAffineTransformRotate(transform, M_PI / 180.0 * 30.0); 
    
    //4. translate by 200 points
    transform = CGAffineTransformTranslate(transform, 200, 0); 

    //5. apply transform to layer
    self.layerView.layer.affineTransform = transform;
}
```

> 注意：变换的顺序会影响最终的结果，也就是说`旋转之后的平移`和`平移之后的旋转`结果可能不同。


## 剪切变换

### 实现一个斜切变换

<img src="./transform2.png" alt="" title="" width="700"/>

```objc
@implementation ViewController

CGAffineTransform CGAffineTransformMakeShear(CGFloat x, CGFloat y)
{
	//1. 
    CGAffineTransform transform = CGAffineTransformIdentity;
    
    //2.
    transform.c = -x;
    
    //3.
    transform.b = y;
    
    return transform;
}

- (void)viewDidLoad
{
    [super viewDidLoad];
    
    //shear the layer at a 45-degree angle
    self.layerView.layer.affineTransform = CGAffineTransformMakeShear(1, 0);
}

@end
```
