## 绘图和动画有两种处理的方式：CPU（中央处理器）和GPU（图形处理器）

### CPU与GPU所处的层面

- (1) CPU所做的工作都在`软件`层面
- (2) GPU在`硬件`层面


### CPU的优缺点

- (1) CPU可以干任何事情：数据处理、`图像绘制`
- (2) 但是对于`图像绘制`，基于`硬件的GPU`图像绘制，会更快

### GPU的优缺点

- (1) GPU使用图像对高度并行浮点运算做了优化，所以对处理图像是很快的
- (2) 但是GPU并没有`无限制处理`性能，当遇到大量的任务时，性能就会开始下降了


### 将CPU与GPU结合起来，各自发挥自己的优势，才能提高性能

- (1) CPU >>> 我们作死的让他进行数据处理
	- 对象创建/释放/废弃
	- frame计算、size计算
	- 图像文件的读取、解压、解码..

- (2) GPU >>> 尽量的值让他绘图
	- 尽量减少layer的 透明度、阴影、重叠...
	- 也可以偶尔让CPU承担一部分绘图任务，在异步子线程使用CoreGraphics完成

## 额外的渲染服务进程

- (1) 动画和屏幕上组合的图层实际上被一个单独的进程管理，而不是我们的应用程序所在进程，这个进程就是所谓的`渲染服务`

- (2) 这个渲染服务进程，同时管理着iOS系统手机的`主屏`

- (3) 在iOS5和之前的版本是`SpringBoard`进程，在iOS6之后的版本中叫做`BackBoard`

## 当运行一段动画时，这个过程会被四个分离的阶段被打破

### 如下四个阶段

- (1) 布局 
- (2) 显示
- (3) 准备
- (4) 提交

### 布局

这是准备你的视图/图层的层级关系，以及设置图层属性（位置，背景色，边框等等）的阶段

- (1) 创建一个UIView对象
- (2) addSubview
- (3) view.frme、backgroudColor、borderWith ....

### 显示

是图层的`寄宿图片`被绘制的阶段，绘制有可能涉及UIView/CALayer的如下函数被调用:

- (1) `-drawRect:`
- (2) `-drawLayer:inContext:`

### 准备

这是Core Animation准备发送`动画数据`到`渲染服务`的阶段。

### 提交

这是最后的阶段，Core Animation打包所有图层和动画属性，然后通过IPC（内部处理通信）发送到渲染服务进行显示。

## GPU相关的操作

### 宽泛的说，大多数CALayer的属性都是用GPU来绘制，不需要软件层面做任何绘制

- (1) layer的背景或者边框的颜色
- (2) layer.contents属性设置一张图片，然后裁剪它

### 如下这些事情会降低GPU的绘制性能

- (1) 太多的几何结构

即CALayer/UIView的个数太多、结构复杂.

- (2) 过多的半透明图层重叠时

GPU对重叠的区域，用相同的像素填充多次，但是目前GPU都会应付这种情况。

- (3) 离屏绘制

GPU的离屏渲染，会新开一个进程，导致分配额外的时间与空间给另外的进程，并且进程之间的切换的消耗，都会降低GPU的性能。

CPU的离屏渲染，不需要新开一个进程，而是在进程内完成，但同样会降低CPU的计算性能。

- (4) 过大的图片

如果视图绘制超出GPU支持的`2048x2048`或者`4096x4096`尺寸的纹理，就必须要用`CPU`在图层每次显示之前对图片`预处理`，同样也会降低性能。

可以考虑使用`CATiledLayer`，进行大图的`切割分批次`进行渲染。

## CPU相关的操作

### 常规要做的事情

- (1) 布局计算

复杂的frame计算是消耗CPU时间的一个主要原因，比如autolayout会更大。

- (2) 视图惰性加载

懒加载视图，会让内存得到节约使用。但是`大量`的视图懒加载，会导致在对顶层视图进行数据渲染时，一个时刻执行大量的工作: 

创建UI对象、调整frame、读取压缩图片、图片的解压 ...

这些工作，都压在一个时间点，我觉得可能还没有提前将一些`肯定`存在的UI对象创建完毕，再执行顶层UI对象的数据渲染更快。

- (3) 使用CoreGraphics 进行图像的绘制

如果对视图实现了`-[UIView -drawRect:]`方法，或者`CALayerDelegate的-drawLayer:inContext:`方法，那么在绘制任何东西之前都会产生一个巨大的性能开销。

因为会立刻创建一个中等大小的寄宿图，传递给该函数内，让开发者来任意的进行图像绘制。

然后一旦绘制结束之后，必须把`图片数据`通过IPC传到渲染服务器。在此基础上，CoreGraphics绘制就会变得十分缓慢，所以在一个对性能十分挑剔的场景下这样做十分不好。

- (4) 解压图片

PNG或者JPEG压缩之后的图片文件会比同质量的位图小得多。

但是在`图片绘制到屏幕`上之前，必须对其进行图片解压缩，其尺寸通常等同于: `图片宽 x 图片长 x 4个字节`。

为了节省内存，iOS通常`直到真正绘制`的时候才去`解码`图片。

根据你加载图片的方式，第一次对图层内容赋值的时候（直接或者间接使用UIImageView）或者把它绘制到CoreGraphics中，都需要对它解压。

这样的话，对于一个较大的图片，都会占用一定的时间。

## 保持屏幕每一秒刷新次数，最低为60次

一般情况下，没有涉及动画时，最少要保持每秒刷新屏幕60次。 但如果使用了NSTimer或CADisplayLink的复杂动画，每秒刷新屏幕次数可以降低到30次。

## 优化一、文件的IO操作

子线程异步读取文件，然后内存缓存起来，减少读取文件的次数。

## 优化二、对象的操作，放到子线程异步完成

- (1) 对象的创建/废弃（除非UI对象）
- (2) frame计算
- (3) 图片的解压缩
- (4) CoreGraphics图形绘制

## 优化三、使用`专用图层`来绘制自定义路径的代码模板，`代替`使用重写`drawRect:`

> 专用图层会使用OprnGL硬件加速绘制。


```objc
@implementation XingNengVC {
    UIView  *_bottomView;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    //1. 创建UIView容器
    _bottomView = [[UIView alloc] initWithFrame:CGRectMake(0, 0, 300, 200)];
    [self.view addSubview:_bottomView];
    
    //2. 创建具体绘制图像的专用图层
    CAShapeLayer *layer = [CAShapeLayer layer];
    layer.frame = _bottomView.bounds;
    
    //3. 设置要绘制的图像路径
    UIBezierPath *path = [[UIBezierPath alloc] init];
    [path moveToPoint:CGPointMake(175, 100)];
    [path addArcWithCenter:CGPointMake(150, 100) radius:25 startAngle:0 endAngle:2*M_PI clockwise:YES];
    [path moveToPoint:CGPointMake(150, 125)];
    [path addLineToPoint:CGPointMake(150, 175)];
    [path addLineToPoint:CGPointMake(125, 225)];
    [path moveToPoint:CGPointMake(150, 175)];
    [path addLineToPoint:CGPointMake(175, 225)];
    [path moveToPoint:CGPointMake(100, 150)];
    [path addLineToPoint:CGPointMake(200, 150)];
    
    //4. 将要绘制的路径设置给layer
    layer.path = path.CGPath;
    
    //3.
    [_bottomView.layer addSublayer:layer];
}

@end
```

## 优化四、避免大量的重新绘制

> 软件绘图的代价昂贵，除非绝对必要，你应该避免重绘你的视图。提高绘制性能的秘诀就在于尽量避免去绘制。


一次性对整个区域进行全部的擦除之后，再进行新的绘制，会导致有很多的之前一样的区域又重新的画一遍，肯定会性能低。

所以，最好是手动的确认，只需要刷新某一个变化的区域，进行重新绘制：

```
-setNeedsDisplayInRect:
```

用-setNeedsDisplayInRect:来减少不必要的绘制代码示例:

```objc
- (void)addBrushStrokeAtPoint:(CGPoint)point
{
    //add brush stroke to array
    [self.strokes addObject:[NSValue valueWithCGPoint:point]];

    //set dirty rect
    [self setNeedsDisplayInRect:[self brushRectForPoint:point]];
}

- (CGRect)brushRectForPoint:(CGPoint)point
{
    return CGRectMake(point.x - BRUSH_SIZE/2, point.y - BRUSH_SIZE/2, BRUSH_SIZE, BRUSH_SIZE);
}

- (void)drawRect:(CGRect)rect
{
    //redraw strokes
    for (NSValue *value in self.strokes) {
    
        //get point
        CGPoint point = [value CGPointValue];

        //get brush rect
        CGRect brushRect = [self brushRectForPoint:point];
        ￼
        //【重要】only draw brush stroke if it intersects dirty rect
        if (CGRectIntersectsRect(rect, brushRect)) {
            //draw brush stroke
            [[UIImage imageNamed:@"Chalk.png"] drawInRect:brushRect];
        }
    }
}
```

## 优化五、尽量避免改变，那些包含文本的视图的frame，因为这样做的话文本就需要重绘

...

## 优化六、减少离屏渲染，不管是CPU还是GPU

应该使用专用图层进行代替。

> 注意： CAShapeLayer，cornerRadius和maskToBounds俩结合在一起，就触发了屏幕外渲染。

## 优化七、当UI结构层级很复杂时，可以考虑使用CoreGraphics基于CPU的软件绘制来代替

这个提议看上去并不合理，因为大家都知道软件绘制行为要比GPU合成要慢，而且还需要更多的内存空间。

但是在因为图层数量而使得性能受限的情况下，软件绘制很可能提高性能呢，因为它避免了图层分配和操作问题。

用Core Graphics去绘制一个静态布局有时候，会比用层级的UIView实例来得快，但是使用UIView实例要简单得多而且比用手写代码写出相同效果要可靠得多。

## 优化八、`-[CALayer renderInContext:]` 方法使用

使用CALayer的-renderInContext:方法，你可以将图层及其子图层快照进一个Core Graphics上下文然后得到一个图片，它可以直接显示在UIImageView中，或者作为另一个图层的contents。

eg、把整个屏幕转化为图片

```objc
//1. 
UIImageView* imageV = [[UIImageView alloc]initWithFrame:CGRectMake(0, 0, self.view.frame.size.width, self.view.frame.size.height)];

//2.
UIGraphicsBeginImageContextWithOptions(imageV.frame.size, NO, 0);

//3.
CGContextRef context = UIGraphicsGetCurrentContext();

//4. 把当前的整个画面导入到context中，然后通过context输出UIImage，这样就可以把整个屏幕转化为图片
[self.view.layer renderInContext:context];

//5.
UIImage* image = UIGraphicsGetImageFromCurrentImageContext();

//6.
imageV.image = image;

//7. 
UIGraphicsEndImageContext();
```