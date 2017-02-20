### AsyncDisplayKit中认为会消耗主线程耗时操作类型:

- (1) Layout 布局计算
	- 计算文本内容的 宽度计算、高度计算
	- UI对象的 frmae计算、frmae设置、frmae调整

- (2) Rendering 显示数据渲染
	- 文本内容的渲染
	- 图片的解码
	- 图形的绘制

- (3) UIKit Obejcts UI对象
	- UI对象的属性值调整
	- UI对象的创建
	- UI对象的销毁（废弃）

AsyncDisplayKit主要是讲 `(1) Layout`  和 `(2) Rendering` 进行了子线程异步等优化（这个是最基础的优化，还有很多复杂的优化），但是对于 `(3) UIKit Obejcts` 则必须仍然放在`主线程`上进行操作。

###UIView对象与CALayer之间的关系

通常情况下，我们直接使用的任何系统UI对象、自定义的UI对象，几乎都是继承自`UIView`这个类，包含了我们几乎所有的`界面数据显示`、`触摸事件`、`手势事件`等等各种事件的响应。

这正是苹果设计的非常好的一个地方，因为`UIView`这个类将底层真正负责`绘制界面显示`的那个类`CALayer`完完全全封装起来了，对于我们一般这样的开发者基本上一个App的开发过程中不需要过多的去写很复杂的`CALayer`的代码。

CALayer对象在最终传递到`GPU`时，由`GPU`取出`CALayer对象中`的保存的所有显示数据项进行渲染处理，当完成全部的渲染处理之后，将处理的最终能够显示的数据发送给`CPU`，由CPU负责执行最终绘制到屏幕上的事情。

如果CALayer对象包含图片数据，那么在提交给GPU之前，会在主线程上同步执行`图片的解码`操作。

CALayer是属于`QuartzCore`框架提供的，而这个框架是一个`跨平台`的`绘制`框架。这里的跨平台指的是在`iOS` 和 `Mac OS X`上均能使用，也就是说CALayer能在`iOS` 和 `Mac OS X`上面都能够绘制内容。但是这两个平台接收用户交互的方式完全不一样:

- iOS、是通过`触摸事件` UITouch对象 来实现
- Mac OS X、是通过`监听 鼠标和键盘`事件来实现

那么也就是说，对于 iOS 和 Mac OS X 两个平台来说，`界面绘制`都是采用同一种方式（CALayer）来完成，但是`事件响应`却是不相同的方式。

由于`界面上显示`的内容，又需要同时具备`事件响应`功能，所以苹果将 `界面绘制` 和 `事件响应` 都封装起来并放到一起。但由于两个平台的事件响应机制不同，所以两个平台是不同的api:

- iOS系统上，`界面绘制` 和 `事件响应` 结合体 >>> `UIResponder`
- Mac OS X系统上，`界面绘制` 和 `事件响应` 结合体 >>> `NSView`

总之，我们现在在iOS中接触的各种的View类因为都是继承自`UIView`，而UIView最终继承自`UIResponder`，所以UIView其实包含两部分:

- (1) `CALayer`: 负责界面显示数据的绘制
- (2) `UIResponder`: 负责各种UI事件响应

对于我们平时对UIView子类对象的操作如: 

- (1) 设置frame
- (2) 设置背景色
- (3) 设置透明度
- (4) 设置图像...等等操作

最终UIView对象都会原封不动的同步设置给`自己内部的CALayer图层`对象，而这个同步的过程系统UIKit也是同步执行，最终并转换成`CALayer`所认识的CoreGraphic库格式的数据（都是以 `CG...`开头的各种结构体实例，比如: CGColor....）。


####总结UIView与CALayer:

- (1) CALayer 与 UIView 的共同点:
	- 都只能在`主线程`上进行操作（创建、修改、释放）
	-  UIView 与 CALayer 两者都有`树状层级结构`
		- UIView: subviews
		- CALayer: sublayers

- (2) CALayer 做的事情:
	- 无法响应任何的UI事件
	- 只负责最终绘制到屏幕上进行显示的数据的一个数据载体
	- Layer 比 View 多了个 `AnchorPoint`
	
- (3) UIView 做的事情:	
	- 每个 UIView 内部都有一个 `Root CALayer` 在背后提供内容的绘制和显示
	- 负责处理用户交互的各种UI事件响应
	- UIView充当 `CALayer对象 CALayerDelegate对象`，渲染动画时产生的事件
	- 访问和设置`UIView`的属性，实际上访问和设置的是UIView内部的`CALayer`对应的属性

####图示UIView与CALayer二者之间的关系

![](http://i1.piimg.com/567571/07f8da055f783c8a.png)

- (1) View `持有（__strong）` Layer 用于显示，View 中大部分显示属性实际是从 Layer 映射而来

- (2) Layer 的 `delegate（__weak）` 在这里是 View，当Layer对象的`属性改变`、`动画产生`等时候，通知 View 进行处理

- (3) UIView 和 CALayer `不是` 线程安全的，并且`只能`在`主线程`创建、访问、销毁


下面使用一个简单的例子看下UIView与内部CALayer如何协作的

- (1) Layer

```objc
@interface TestLayer : CALayer
@end
@implementation TestLayer

- (void)setFrame:(CGRect)frame
{
    NSLog(@"%@ - %@", self, NSStringFromSelector(_cmd));
    [super setFrame:frame];
}

- (void)setPosition:(CGPoint)position
{
    NSLog(@"%@ - %@", self, NSStringFromSelector(_cmd));
    [super setPosition:position];
}

- (void)setBounds:(CGRect)bounds
{
    NSLog(@"%@ - %@", self, NSStringFromSelector(_cmd));
    [super setBounds:bounds];
}

- (CGPoint)position
{
    NSLog(@"%@ - %@", self, NSStringFromSelector(_cmd));
    return [super position];
}

@end
```

- UIView

```objc
@interface TestLayerView : UIView
@end
@implementation TestLayerView

- (instancetype)initWithFrame:(CGRect)frame
{
    self = [super initWithFrame:frame];
    if (self) {
        NSLog(@"%@ - %@", self, NSStringFromSelector(_cmd));
    }
    return self;
}

/**
 *  指定这个UIView被初始化出来之后，自动创建并持有的layer的类
 */
+ (Class)layerClass {
    return [TestLayer class];
}

- (CGPoint)center
{
    NSLog(@"%@ - %@", self, NSStringFromSelector(_cmd));
    return [super center];
}

- (void)setFrame:(CGRect)frame
{
    NSLog(@"%@ - %@", self, NSStringFromSelector(_cmd));
    [super setFrame:frame];
}

- (void)setCenter:(CGPoint)center
{
    NSLog(@"%@ - %@", self, NSStringFromSelector(_cmd));
    [super setCenter:center];
}

- (void)setBounds:(CGRect)bounds
{
    NSLog(@"%@ - %@", self, NSStringFromSelector(_cmd));
    [super setBounds:bounds];
}

@end
```

- ViewController

```objc
@implementation LayerViewController {
    TestLayerView *_view;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    _view = [[TestLayerView alloc] initWithFrame:CGRectMake(20, 100, 200, 200)];
//    [self.view addSubview:_view];
}

@end
```

运行后输出

```
2016-08-04 13:21:39.429 Demos[41645:3273685] <TestLayerView: 0x7f874a552080; frame = (-100 -100; 200 200); layer = <TestLayer: 0x7f874a557ea0>> - center
2016-08-04 13:21:39.429 Demos[41645:3273685] <TestLayer: 0x7f874a557ea0> - position
2016-08-04 13:21:39.429 Demos[41645:3273685] <TestLayer: 0x7f874a557ea0> - setFrame:
2016-08-04 13:21:39.429 Demos[41645:3273685] <TestLayer: 0x7f874a557ea0> - setPosition:
2016-08-04 13:21:39.429 Demos[41645:3273685] <TestLayer: 0x7f874a557ea0> - setBounds:
2016-08-04 13:21:39.429 Demos[41645:3273685] <TestLayerView: 0x7f874a552080; frame = (20 100; 200 200); layer = <TestLayer: 0x7f874a557ea0>> - center
2016-08-04 13:21:39.429 Demos[41645:3273685] <TestLayer: 0x7f874a557ea0> - position
```

不对调用顺序逻辑做分析，只是看我们仅仅针对UIView设置了frame，但是其实涉及到了内部CALayer对象的很多操作。如上只是列举了frame设置的方法实现，还有很多的设置方法实现就不贴了。


###ASDK对如上UIView与CALayer之间联系做了扩展，并对UIView和CALayer的使用进行了很多的性能优化

![](http://i1.piimg.com/567571/886a82937dfae9c2.png)

- (1) ASDK 提供了一些 `AsyncDisplayNode`类，包含常见的`UI对象属性`
	- frame
	- bounds
	- alpha
	- transform
	- backgroundColor
	- superNode
	- subNodes 
	- 等等

- (2) 然后 ASDK 模拟了 `UIView <--> CALayer` 的联系方式，创造了 `AsyncDisplayNode <--> UIView` 之前的联系

- (3) 与 UIView 和 CALayer 不同，`AsyncDisplayNode 是 线程安全`的，所有对于AsyncDisplayNode对象是可以在任何子线程进行`修改`（创建是不可以的，销毁暂时没尝试）

- (4) Node 刚`创建`时，并不会在内部新建 UIView 和 CALayer。直到`第一次 在主线程`访问 `view 或 layer 属性`时，它才会在内部生成对应的对象

- (5) 同样对于Node对象的属性值被`修改`时，也不会立刻同步给UIView与CALayer去执行修改，而是先将修改缓存起来，稍后再需要这些修改的时候，一次性同步给UIView与CALayer去执行修改

- (6) ASDK 可以让多个 CALayer图层预先进行合成
	- 当UIView对象的CALayer对象，又具备多个层次的其他的sub layers对象重叠时
	- **使用条件: 这些UI显示是不需要任何的响应时间、也不需要调整位置时可以室友ASDK提供的这个合成技术**
	- ASDK 会把 所有的 sub layers 合成渲染为一张图片，这样的好处是:
		- CPU 避免了创建 UIKit 对象的资源消耗
		- GPU 避免了多张 texture 合成和渲染的消耗

让开发者可以通过使用AsyncDisplayNode去间接使用UIKit中的各种UI控件对象，获得 ASDK 底层提供大量的性能优化。


###UIView 的继承结构图

![](http://i4.piimg.com/567571/a6fdb487c6a51508.png)


###ASyncDisplayNode 的继承结构图

![](http://i4.piimg.com/567571/79cfd7eea538dbfe.png)


可以看到UIView与ASyncDisplayNode的继承结构看起来`很相似`的，比如:

```
UIButton ----> UIControl -----> UIView
```

```
ASButtonNode ----> ASControlNode -----> ASyncDisplayNode
```

也就是说`ASyncDisplayNode`就相当于`UIView的位于UIKit`中的级别一样，可以通过使用`ASXxxNode`来代替`UIXxxView`来获取ASDK所提供的性能优化。


###ASyncDisplayNode 基础

ASDK提供的 一些核心Node包括:

- (1) AsDisplayNode:UIView子类对应使用，用于自定义view。
- (2) AsCellNode:UICollectionviewcell或UITableViewCell类对应使用，用于自定义cell。
- (3) AsControlNode:UIControl类对应使用，用于自定义按钮。
- (4) AsImageNode:UIImageView类对应使用，处理图形解码之类。
	- ASNetworkImageNode:AsImageNode，类似于`SDWebImage`提供的网络图片下载、解码、绘制
- (5) AsTextNode:UITextView、UILabel类对应使用，建造全功能的富文本支持库。

对于ASyncDisplayNode对象，可以直接访问ASyncDisplayNode对象持有的`UIView对象`和`CALayer对象`:

- (1) node.view
- (2) node.layer

###ASNetworkImageNode、Gif图片、以及各种其他格式网络图片加载

####在主线程上创建ASNetworkImageNode对象，并添加到UIView对象去

```objc
@implementation LayerViewController

- (void)viewDidLoad {
    [self testGifNode];
}

- (void)testGifNode {
    
    NSString *url = @"http://imgsrc.baidu.com/forum/w%3D580/sign=55073ac06059252da3171d0c0499032c/f09d60cf3bc79f3d74223ceabaa1cd11708b294d.jpg";
    
    CGRect rect = CGRectMake(10, 74, [UIScreen mainScreen].bounds.size.width - 20 , [UIScreen mainScreen].bounds.size.height - 84);
    
    ASNetworkImageNode *imageNode = [[ASNetworkImageNode alloc] init];
    imageNode.layer.borderWidth = 5;
    imageNode.URL = [NSURL URLWithString:url];
    imageNode.frame = rect;
    imageNode.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
    imageNode.contentMode = UIViewContentModeScaleAspectFit;
    [self.view addSubnode:imageNode];   
}

@end
```

运行结果

![](http://i2.piimg.com/567571/85e1b4937d9006a3.gif)

但是从代码上看，Node对象是在`主线程`上创建的并完成添加的，好像并没有牵涉任何的`异步子线程`处理的逻辑，那么到底所谓的性能提升在哪里了？

从官方文档说明中大概是这么说的: 使用这个`ASNetworkImageNode`对象去加载网络图片时，会将下载到的`压缩图片`后续的`图片解码`处理放在`子线程`上异步执行，或者当处于`多核CPU`时直接放在另外一个CPU上去执行处理。

为了让用户界面平滑并且随时可以相应，app程序需要`一秒内至少渲染60帧`（经常说的60FPS），换算下一秒内刷新是说系统处理`一帧`所占用时间`不能超过 1/60秒`。

`1/60秒`约等于`0.01666667秒`，一秒等于`1000毫秒`，那么`0.01666667秒`大概就是`0.01666667 * 1000 = 16毫秒`。

也就是说，处理某一帧显示时间最多为`16毫秒`，但实际上16毫秒内还包含有系统的一些额外处理时间。那么最理想的情况下，一帧处理的时间保持在`10毫秒`左右，这`10毫秒`时间要做的事情:


- (1) CPU >>>> 所有的UI对象创建、UI对象的frame布局计算、文本占用尺寸计算、图片的解压解码、最终的图像渲染

- (2) GPU >>>> 视图的合成、图片的合成、纹理渲染以及合成


上例代码中最主要的耗时代码就是对下载得到的网络图片进行`图片解码`的操作（一般情况下都是在`主线程上同步执行图片解码`的操作）和最终解码后的`图像绘制`到屏幕上，对这两大的优化:

- (1) 将图片解码的代码，放到`子线程上异步`执行，完成后回调完成下面的`图像绘制`

- (2) 由于最终绘制图像使用的是`CoreGraphics库`，而`CoreGraphics库`的函数都是`多线程安全`的，所以也可以将`绘制图像`的代码放到`子线程`上执行


所以，即使上面代码中没有看到任何异步子线程处理的代码，但其实内部实现中已经大量使用后台多线程进行优化。

当然这些目前只是参考了官方文档介绍和网络上一些文章的记录，并没有真正的去查看ASDK的源码实现，等熟练使用了ASDK之后再去看内部实现吧。

有一个测试FPS的开源代码:

```
https://github.com/RolandasRazma/RRFPSBar
```

####ASDK Node对象不能在子线程上进行创建，当直接将Node对象`添加到UIView对象`中时，会崩溃到ASDK的一些Assert断言处，告诉你必须处于`主线程`上执行


如下代码必崩，原因是如下代码是在`子线程`上执行将node添加到`UIView对象的操作`，但是`UIView对象`的所有操作都必须处于`主线程`上，这就是崩溃的原因。

解决: 在子线程中不要使用任何的`UIView对象`代码，而是转成使用各种Node对象、Node容器对象、Node之间的`addSubnode:`。

```objc
@implementation LayerViewController

- (void)viewDidLoad {
    [self testGifNode];
}
- (void)testGifNode {
    
    NSString *url = @"http://imgsrc.baidu.com/forum/w%3D580/sign=55073ac06059252da3171d0c0499032c/f09d60cf3bc79f3d74223ceabaa1cd11708b294d.jpg";
    
    CGRect rect = CGRectMake(10, 74, [UIScreen mainScreen].bounds.size.width - 20 , [UIScreen mainScreen].bounds.size.height - 84);
    
	dispatch_async(dispatch_get_global_queue(0, 0), ^{
        ASNetworkImageNode *imageNode = [[ASNetworkImageNode alloc] init];
        imageNode.layer.borderWidth = 5;
        imageNode.URL = [NSURL URLWithString:url];
        imageNode.frame = rect;
        imageNode.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
        imageNode.contentMode = UIViewContentModeScaleAspectFit;
    });
}

@end
```

####想要在子线程上创建Node对象，就必须使用`-[Node容器对象 addSubnode:subnode对象]`，也就是不应该在子线程中node之间的添加，而不应该涉及任何的UIView对象.


```objc
@implementation LayerViewController

- (void)viewDidLoad {

dispatch_async(dispatch_get_global_queue(0, 0), ^{
        
        /**
         * root node （root view）
         */
        ASDisplayNode *rootNode = [[ASDisplayNode alloc] init];
        rootNode.backgroundColor = [UIColor redColor];
        
        /**
         * ViewController 使用 root node
         */
        ASViewController *vc = [[ASViewController alloc] initWithNode:rootNode];
        
        // sub node 1
        NSString *url = @"http://imgsrc.baidu.com/forum/w%3D580/sign=55073ac06059252da3171d0c0499032c/f09d60cf3bc79f3d74223ceabaa1cd11708b294d.jpg";
        ASNetworkImageNode *node1 = [[ASNetworkImageNode alloc] init];
        node1.URL = [NSURL URLWithString:url];
        node1.frame = CGRectMake(10, 10, 300, 300);
        
        // sub node 2
        ASTextNode *node2 = [[ASTextNode alloc] init];
        node2.attributedText = [[NSMutableAttributedString alloc] initWithString:@"Hello" attributes:nil];
        node2.frame = CGRectMake(10, 320, 300, 100);
        
        // sub node 3
        ASTextNode *node3 = [[ASTextNode alloc] init];
        node3.attributedText = [[NSMutableAttributedString alloc] initWithString:@"world" attributes:nil];
        node3.frame = CGRectMake(10, 480, 300, 100);
        
        /**
         *  将subnode 添加到 root node，而不是UIView对象中，这样就可以在子线程上创建node了
         */
        [rootNode addSubnode:node1];
        [rootNode addSubnode:node2];
        [rootNode addSubnode:node3];
        
        [self presentViewController:vc animated:YES completion:nil];
    });
}

@end
```

####我又使用了SDWebImage的加载网络图片的代码试了一下

```objc
@implementation LayerViewController

- (void)viewDidLoad {
    [self testGifNode];
}

- (void)testGifNode {
    
    NSString *url = @"http://imgsrc.baidu.com/forum/w%3D580/sign=55073ac06059252da3171d0c0499032c/f09d60cf3bc79f3d74223ceabaa1cd11708b294d.jpg";
    
    CGRect rect = CGRectMake(10, 74, [UIScreen mainScreen].bounds.size.width - 20 , [UIScreen mainScreen].bounds.size.height - 84);
   
    UIImageView *imageView = [[UIImageView alloc] init];
    imageView.contentMode = UIViewContentModeScaleAspectFit;//图片比例拉伸
    imageView.layer.borderWidth = 5;
    NSURL *url_pic = [NSURL URLWithString:url];
    [imageView sd_setImageWithURL:url_pic completed:nil];
    imageView.frame = rect;
    imageView.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
    [self.view addSubview:imageView];
}

@end 
```

运行结果和之前的一样的，貌似还看不出什么特别大的性能问题。据说SDWebImage使用的是`NSOperationQueue` 并发后台多线程完成`图片解码`的。那可能在图片解码的问题上，两个框架都是是一致的。

还可以继续优化，将最终`图像绘制`的代码也干脆放`异步`到了`子线程上`去统统完成。因为绘制图像最终使用的是`CoreGraphics库`，这套库函数是纯c语言实现的，基本上所有的函数都是多线程安全操作的。

但是在多线程上执行`CoreGraphics`的一些函数绘制图像时，可能导致一个线程锁定无法解除，接着造成越来越来多的子线程创建，而这些创建的线程一样无法解除，导致程序一个时间内线程大量的创建，从而占用很高的CPU，作者是通过一个`多线程队列缓存池`的结构来处理这个问题的，可以参看作者开源的`YYDispatchQueuePool`。

***

###实现来`扩大事件响应区`，ASDK所有的Node对象都可以直接通过属性设置hitTest的响应区大小

UIView对象，必须通过重写`hitTest:withEvent:`

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

ASDK中所有的Node都有这个属性直接操作，如下即可让`Node最终的UIView对象的`响应区上左下右都增加20点

```objc
node.hitTestSlop = UIEdgeInsetsMake(-20, -20, 20, 20);
```

***

###上面代码直接将`ASNode对象`直接添加到了`UIView对象`中，ASDK官方在文档里并不推荐这么做

如果把一个Node节点直接添加到一个现有的view视图层次结构里。这样做会导致你的节点在渲染的时候会闪烁一下，可能是因为重复渲染所导致的，也可能进行一次渲染之后就不再做任何渲染了。

原因是，这个Node没有ASDK提供的`容器`管理这些Node对象，这些Node无法知道当前`界面的状态`，就无法知道到底现在要做什么事情。

应该把nodes节点，当做一个`sub node 对象`添加到一个`ASDK提供的容器类对象`里。这些容器类负责告诉所包含的节点，他们现在都是什么状态以便于尽可能有效的加载数据与渲染。

并且容器可以`预先通知`所有的子节点都可以异步的执行布局计算，获取数据，解码，渲染。

ASDK提的几个容器类:

- (1) ASViewController 类似于 UIViewController

- (2) ASCollectionNode 类似于 UICollectionView

- (3) ASTableNode 类似于 UITableView

- (4) ASPagerNode 类似于 UIPageViewController

***

###Node容器一、ASViewController

```objc
@interface ASViewController<__covariant DisplayNodeType : ASDisplayNode *> : UIViewController <ASVisibilityDepth>
```

ASViewController是继承自UIViewController的，所以完全可将UIViewController当做平时的UIViewController使用。

使用ASViewController的一大好处是节省内存，当ASViewController`离开屏幕`的时候会自动减少其子node的数据范围（前文提到）和显示范围（前文提到），这是大型app管理的关键。

UIViewController提供他自己的`root view`，ASViewController提供了他自己的node，是在`-initWithNode:`方法实现中创建`root node`。

ASDK并没有提供一个类似`UITableViewController`的东西，而是需要使用`ASViewController`初始化时，内部创建一个ASTableNode对象作为Root Node。

```objc
@interface XxxASViewController

- (instancetype)init
{
  // 创建ViewController对象的 root node对象，来代替 root view
  _tableNode = [[ASTableNode alloc] initWithStyle:UITableViewStylePlain];
  self = [super initWithNode:_tableNode];

  if (self) {
    _tableNode.dataSource = self;
    _tableNode.delegate = self;
  }

  return self;
}

@end
```

###Node容器二、ASTableNode

ASTableNode等效于UIKit的UITableview，ASTableNode的数据源获取CellNode的回调方法有两个:

```objc
- (ASCellNode *)tableView:(ASTableView *)tableView nodeForRowAtIndexPath:(NSIndexPath *)indexPath;
```
	
```objc
- (ASCellNodeBlock)tableView:(ASTableView *)tableView nodeBlockForRowAtIndexPath:(NSIndexPath *)indexPath;
```
	
- (1) 如上这两个方法，都`不用`像UITableView那样需要考虑`Cell重用`的问题

- (2) `nodeBlock`的版本返回一个创建CellNode的Block，以便这些节点可以同时进行准备和显示处理，所有的子节点都可以在`后台线程`进行初始化，但是一定要保持他们的代码是`线程安全`的
	- 在node block中操作外部数据的代码，必须`保持多线程安全`
	- 因为node block可能在`后台多个子线程`上同时调用

####关于ASTableNode的CellNode如何计算行高cell height

UITableView对象是通过delegate函数实现来完成:

```objc
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath;
```

一个node通过`-layoutSpecThatFits:`方法返回的布局规则确定了行高，所有的节点只要提供了约束大小，就有能力自己确定自己的尺寸。

如果你对一个ASCellNode调用了`setNeedsLayout`，他会自动的把布局传递，如果整体所需要的大小发生了变化，表会被告知要进行更新，和UIKit不同的时，你不需要调用reload，这样很节省了代码。

***

###使用ASDisplayNode将一些UIView对象包裹起来当做一个ASDisplayNode使用

直接创建UIView对象、添加UIView对象、设置UIView对象等等这些代码，都必须处于`主线程`上执行，且代码都是`同步顺序往下执行`。

但是可以通过使用一个ASDisplayNode对象来包裹起来，就可以`伪装`做到子线程上操作以及异步执行，但是`最终`还是在`主线程`上操作UIView对象和CALayer对象。

```objc
ASDisplayNode *node = [ASDisplayNode alloc] initWithViewBlock:^{
    SomeView *view = [[SomeView alloc] init];
    return view;
}];
```

后续通过使用`node对象`，来间接操作内部包裹的`SomeView对象`。

***

###Layer-Backing、使用`CALayer对象`代替`UIView对象`处理一些仅仅只是显示数据而不需要事件响应的UI

前面已经说了UIView是最终所有的显示效果都是由内部的CALayer对象来负责的。

然后有时候很多View只是用来显示数据，不会有任何的事件响应处理。那么此种情况的View，如果直接换成CALayer对象去处理，而不是通过UIView对象去处理，在性能上绝对能提高很多。

但是，直接操作CALayer对象比直接操作UIView对象难度还是高多了，这也就是为何大部分开发者都只操作UIView对象，而不去直接操作CALayer对象。

但是ASDK提供了可以操作CALayer的方法，可以让像操作UIView一样去操作CALayer对象。

ASDK中，如果需要直接使用CALayer去完成UIView的功能，直接可以:

```objc
node.layerBacked = YES;
```

这样ASDK就不会创建UIView对象，而只是创建CALayer对象去显示数据，但是没有任何的事件响应。

****

###待记录的其他ASDK学习点....

***

###https://github.com/WildDylan/AsyncDisplayKitDemo 有一个很好的ASDK使用demo

记录下一些关键的代码

- (1) ViewController对象中创建 ASTableView对象

```objc
@interface ViewController () <ASTableViewDelegate, ASTableViewDataSource, TimeLineNodeDelegate>

@property ( nonatomic, strong ) ASTableView * tableView;
@property ( nonatomic, strong ) NSMutableArray * displayLayouts;

@end

@implementation ViewController

- (void) viewDidLoad {
    [super viewDidLoad];
    
    .......
    
    [self.view addSubview:self.tableView];

    .......    
}

- (ASTableView *) tableView {
    
    if ( !_tableView ) {
        
        _tableView = [[ASTableView alloc] initWithFrame:self.view.bounds
                                                  style:UITableViewStylePlain
                                      asyncDataFetching:YES];//异步加载数据
        
        /**
         *  ASTableView提供的异步回调代理
         */
        _tableView.asyncDelegate = self;
        _tableView.asyncDataSource = self;
        
        _tableView.tableFooterView = [UIView new];
        _tableView.separatorStyle = UITableViewCellSeparatorStyleNone;
    }
    
    return _tableView;
}

- (NSInteger) numberOfSectionsInTableView: (UITableView *) tableView {
    return 1;
}

- (NSInteger) tableView: (UITableView *) tableView
  numberOfRowsInSection: (NSInteger) section {
    return self.displayLayouts.count;
}

- (ASCellNode *) tableView: (ASTableView *) tableView
     nodeForRowAtIndexPath: (NSIndexPath *) indexPath {
    
    // 直接返回Cell的新的对象，不必考虑重用的问题，ASDK会自动处理
    TimeLineNode * timeLineNode = [[TimeLineNode alloc]
                                   initWithDisplayLayout:_displayLayouts[indexPath.row]];
    timeLineNode.delegate = self;
    timeLineNode.selectionStyle = UITableViewCellSelectionStyleNone;
    
    return timeLineNode;
}

// 发现用这个node block创建cell的方式会崩溃，因为是在子线程执行返回的block，但是没看到操作UI对象，暂时不太清除为何
//- (ASCellNodeBlock)tableView:(ASTableView *)tableView nodeBlockForRowAtIndexPath:(NSIndexPath *)indexPath {
//    return ^() {
//        TimeLineNode * timeLineNode = [[TimeLineNode alloc]
//                                       initWithDisplayLayout:_displayLayouts[indexPath.row]];
//        timeLineNode.delegate = self;
//        timeLineNode.selectionStyle = UITableViewCellSelectionStyleNone;
//        return timeLineNode;
//    };
//}

- (void) tableView: (ASTableView *) tableView willBeginBatchFetchWithContext: (ASBatchContext *) context {
    // 暂时不知道这个方法有何用
}

@end
```

- (2) CellNode 

```objc
#import <AsyncDisplayKit/AsyncDisplayKit.h>

@class TimeLineDisplayModel, TimeLineNode, DLMutiImageNode, DLSharedNode, DLFavourAndComment;

@protocol TimeLineNodeDelegate <NSObject>
//声明cell点击回调处理
@end

@interface TimeLineNode : ASCellNode

@property ( nonatomic, strong ) TimeLineDisplayModel * displayModel;

@property ( nonatomic, assign ) id<TimeLineNodeDelegate> delegate;

- (instancetype)initWithDisplayLayout:(TimeLineDisplayModel *)displayModel;

@end
```

```objc
#import "TimeLineNode.h"
#import "TimeLineDisplayLayout.h"
#import "DLMutiImageNode.h"
#import "DLSharedNode.h"
#import "DLFavourAndComment.h"

@interface TimeLineNode ()

//--------------------------所有的的sub nodes------------------------------------
// 头部 sub nodes

@property ( nonatomic, strong ) ASNetworkImageNode * avatarNode;
@property ( nonatomic, strong ) ASTextNode         * nickNameNode;
@property ( nonatomic, strong ) ASTextNode         * contentNode;

//------------------------------------------------------------------------------
// 中间 sub nodes

@property ( nonatomic, strong ) DLMutiImageNode * mutiImageNode; // max count: 9
@property ( nonatomic, strong ) ASVideoNode     * videoNode; // Video
@property ( nonatomic, strong ) DLSharedNode    * sharedNode; // URL

//------------------------------------------------------------------------------
// location, referred, time, show delete button if is my post, option button<favor, comment>

@property ( nonatomic, strong ) ASTextNode  * locationNode;
@property ( nonatomic, strong ) ASTextNode  * referredNode;
@property ( nonatomic, strong ) ASTextNode  * timeNode;
@property ( nonatomic, strong ) ASTextNode  * deleteNode;
@property ( nonatomic, strong ) ASImageNode * optionNode;

//------------------------------------------------------------------------------
// 底部评论

@property ( nonatomic, strong ) DLFavourAndComment * fcNode;
@property ( nonatomic, strong ) ASDisplayNode * sepLine;

//------------------------------------------------------------------------------

@end

@implementation TimeLineNode

- (instancetype) initWithDisplayLayout: (TimeLineDisplayModel *) displayModel {
    
    self = [super init];
    if ( self ) {
    
        _displayModel = displayModel;
        
        // 初始化所有的 sub nodes
        [self initializeComponentWithLayout:displayModel];
    }
    
    return self;
}

#pragma mark - sub nodes initialized

/**
 *  根据传入的实体类对象，创建对应的sub node，并设置要显示的实体类对象
 *  不要进行frame布局等设置
 */
- (void) initializeComponentWithLayout: (TimeLineDisplayModel *) layout {
    
    // Load avatarNode
    _avatarNode = [[ASNetworkImageNode alloc] init];
    _avatarNode.placeholderColor = ASDisplayNodeDefaultPlaceholderColor();
    _avatarNode.URL = layout.user.avatarURL;
    [self addSubnode:_avatarNode];
    
    // Load nickNameNode
    _nickNameNode = [[ASTextNode alloc] init];
    _nickNameNode.attributedString = [[NSAttributedString alloc] initWithString:layout.user.nickName
                                                                     attributes:@{
                                                                                  
                                                                                  NSFontAttributeName : [UIFont fontForFontName:FontNameSTHeitiSCMedium size:16.],
                                                                                  NSForegroundColorAttributeName : ASDisplayNodeDefaultTintColor()
                                                                                  
                                                                                  }];
    [self addSubnode:_nickNameNode];
    
    // Load contentTextNode
    if ( layout.content ) {
        
        _contentNode = [[ASTextNode alloc] init];
        _contentNode.attributedString = [[NSAttributedString alloc] initWithString:layout.content
                                                                        attributes:@{
                                                                                     
                                                                                     NSFontAttributeName : [UIFont fontForFontName:FontNameSTHeitiSCLight size:14.],
                                                                                     NSForegroundColorAttributeName : RGB(101, 101, 101)
                                                                                     
                                                                                     }];
        [self addSubnode:_contentNode];
    }
    
    // Load content and load all
    switch (layout.contentType) {
            
        case DLContentType_MutiImage: {
            
            /**
             * 多图显示的node
             */
            _mutiImageNode = [[DLMutiImageNode alloc] initWithMutiImage:layout.images];
            [self addSubnode:_mutiImageNode];
            
            break;
        }
        case DLContentType_Video: {
            
            /**
             * video显示的node
             */
            _videoNode = [[ASVideoNode alloc] init];
            _videoNode.backgroundColor = ASDisplayNodeDefaultPlaceholderColor();
            _videoNode.asset = [AVAsset assetWithURL:layout.video.videoURL];
            [self addSubnode:_videoNode];
            break;
        }
        case DLContentType_Shared: {
            
            /**
             * 分享显示的node
             */
            _sharedNode = [[DLSharedNode alloc] initWithShared:layout.share];
            [self addSubnode:_sharedNode];
            break;
        }
        default:
            
            break;
    }
    
    // Load commons
    if ( layout.location ) {
        
        _locationNode = [[ASTextNode alloc] init];
        _locationNode.attributedString = [[NSAttributedString alloc] initWithString:layout.location.locationName
                                                                         attributes:@{
                                                                                      
                                                                                      NSFontAttributeName : [UIFont fontForFontName:FontNameSTHeitiSCLight size:11.],
                                                                                      NSForegroundColorAttributeName : ASDisplayNodeDefaultTintColor()
                                                                                      
                                                                                      }];
        [self addSubnode:_locationNode];
    }
    
    if ( layout.referred ) {
        
        _referredNode = [[ASTextNode alloc] init];
//        _referredNode.layer.borderWidth = 1;
        NSString * referredString = [NSString stringWithFormat:@"提到了: %@", [layout.referred componentsJoinedByString:@", "]];
        _referredNode.attributedString = [[NSAttributedString alloc] initWithString:referredString
                                                                         attributes:@{
                                                                                      
                                                                                      NSFontAttributeName : [UIFont fontForFontName:FontNameSTHeitiSCLight size:11.],
                                                                                      NSForegroundColorAttributeName : RGB(181, 181, 181)
                                                                                      
                                                                                      }];
        
        [self addSubnode:_referredNode];
    }
    
    _timeNode = [[ASTextNode alloc] init];
    _timeNode.layer.borderWidth = 1;
    NSString * timeStr =  [NSDate dateInformationDescriptionWithInformation:[NSDate dateWithTimeIntervalSince1970:layout.timestamp].dateInformation];
    _timeNode.attributedString = [[NSAttributedString alloc] initWithString:timeStr
                                                                     attributes:@{
                                                                                  
                                                                                  NSFontAttributeName : [UIFont fontForFontName:FontNameSTHeitiSCLight size:11.],
                                                                                  NSForegroundColorAttributeName : RGB(181, 181, 181)
                                                                                  
                                                                                  }];
    [self addSubnode:_timeNode];
    
    if ( [layout.user.uid isEqualToString:@"13088488288"] ) {
        
        _deleteNode = [[ASTextNode alloc] init];
        _deleteNode.attributedString = [[NSAttributedString alloc] initWithString:@"删除"
                                                                     attributes:@{
                                                                                  
                                                                                  NSFontAttributeName : [UIFont fontForFontName:FontNameSTHeitiSCLight size:11.],
                                                                                  NSForegroundColorAttributeName : ASDisplayNodeDefaultTintColor()
                                                                                  
                                                                                  }];
        [self addSubnode:_deleteNode];
    }
    
    _optionNode = [[ASImageNode alloc] init];
    _optionNode.layer.borderWidth = 1;
    _optionNode.image = [UIImage imageNamed:@"AlbumOperateMore"];
    [self addSubnode:_optionNode];
    
    // Favour and comments
    if ( layout.favour || layout.comments ) {
        
        _fcNode = [[DLFavourAndComment alloc] initWithFavour:layout.favour andComment:layout.comments];
        [self addSubnode:_fcNode];
    }
    
    _sepLine = [[ASDisplayNode alloc] init];
    _sepLine.backgroundColor = ASDisplayNodeDefaultPlaceholderColor();
    [self addSubnode:_sepLine];
}

#pragma mark - sub nodes layout

/**
 *  给所有subnode设置布局的规则，参考 flex布局使用
 */
- (ASLayoutSpec *)layoutSpecThatFits:(ASSizeRange) constrainedSize {
    
    /**
     *  头像node的size
     */
    _avatarNode.preferredFrameSize = CGSizeMake(40, 40);
    
    /**
     *  垂直布局
     */
    ASStackLayoutSpec * stackLayout = [ASStackLayoutSpec verticalStackLayoutSpec];
    stackLayout.spacing = 1;
    
    /**
     *  保存需要放入到上面`垂直布局`中的 sub node，创建时包含 _nickNameNode 昵称
     */
    NSMutableArray * layoutableNode = [NSMutableArray arrayWithObjects:_nickNameNode, nil];
    
    /**
     *  将 _contentNode 加入 布局数组
     */
    if ( _displayModel.content ) {
        [layoutableNode addObject:_contentNode];
    }
    
    /**
     *  按照实体对象类型，加入对应的 node 到布局数组
     */
    switch (_displayModel.contentType) {

        case DLContentType_MutiImage: {
            
            /**
             *  显示多张图片的node
             */
            [layoutableNode addObject:_mutiImageNode];
            break;
        }
        case DLContentType_Video: {
            
            /**
             *  播放video的node，并设置固定size
             */
            _videoNode.preferredFrameSize = CGSizeMake(SCREEN_WIDTH, 175);
            [layoutableNode addObject:_videoNode];
            break;
        }
        case DLContentType_Shared: {
            
            /**
             *  分享的node
             */
            [layoutableNode addObject:_sharedNode];
            break;
        }
        default:
            
            break;
    }
    
    /**
     *  地址定位node 加入 布局数组
     */
    if ( _displayModel.location ) {
        [layoutableNode addObject:_locationNode];
    }
    
    /**
     *  提到了...node 加入 布局数组
     */
    if ( _displayModel.referred ) {
        [layoutableNode addObject:_referredNode];
    }
    
    /**
     *  当 deleteNode 不存在，能够让 timeNode 和 optinImaheNode 各自顶到左右侧 的布局对象
     */
    ASLayoutSpec *spacer = [[ASLayoutSpec alloc] init];
    //宽度和高度自动伸缩
    spacer.flexGrow = YES;
    
    /**
     *  创建一个水平布局，主要布局 timeNode、deleteNode 、optinImaheNode 这三个node
     *  但是 deleteNode 不一定存在，如果不存在，就需要 timeNode 和 optinImaheNode 各自顶到左右侧
     */
    ASStackLayoutSpec * timeAndOptionLayout = [ASStackLayoutSpec horizontalStackLayoutSpec];
    timeAndOptionLayout.spacing = 5;
    timeAndOptionLayout.alignItems = ASStackLayoutAlignItemsCenter;
    
    /**
     *  justifyContent属性: 定义了node显示的内容，在主轴上的对齐方式，有点类似 NSTextAlignment
     *  - (1) 左对齐 ASStackLayoutJustifyContentStart
     *  - (2) 右对齐 ASStackLayoutJustifyContentEnd
     *  - (3) 居中对齐 ASStackLayoutJustifyContentCenter
     *  - (4) 两端对齐，项目之间的间隔都相等 ASStackLayoutJustifyContentSpaceBetween
     *  - (5) 每个项目两侧的间隔相等,所以，项目之间的间隔比项目与边框的间隔大一倍 ASStackLayoutJustifyContentSpaceAround
     */
    timeAndOptionLayout.justifyContent = ASStackLayoutJustifyContentStart;
    
    /**
     *  让 spacer 这个布局 自动伸缩占据没有node的宽度
     */
    if ( [_displayModel.user.uid isEqualToString:@"13088488288"] ) {
        [timeAndOptionLayout setChildren:@[_timeNode, _deleteNode, spacer, _optionNode]];
    } else {
        [timeAndOptionLayout setChildren:@[_timeNode, spacer, _optionNode]];
    }
    
    /**
     *  timeNode、deleteNode 、optinImaheNode 这三个node的水平布局对象，放入到最开始的垂直布局对象中
     */
    [layoutableNode addObject:timeAndOptionLayout];
    
    /**
     *  再将显示评论数据node加入到垂直布局中
     */
    if ( _displayModel.favour || _displayModel.comments ) {
        [layoutableNode addObject:_fcNode];
    }
    
    /**
     *  最后设置垂直布局对象
     */
    //因为垂直布局，设置布局中所有node宽度自动拉伸，除开设置了preferredFrameSize属性的node
    stackLayout.alignItems = ASStackLayoutAlignItemsStretch;
    
    /**
     *  flexShrink属性: 设置当父元素的宽度小于所有子元素的`宽度之和`时（即子元素会超出父元素），子元素如何缩小自己的宽度
     *  YES >>> 表示缩小subNode的宽度。具体可参考 `flex布局中的 flex-shrink`
     *  NO >>> 不作处理，任其subNode超过superNode
     */
    stackLayout.flexShrink = YES;
    [stackLayout setChildren:layoutableNode];
    
    /**
     *  最终左边的头像与右侧的垂直布局之间的水平布局
     */
    ASStackLayoutSpec * avatarAndContent = [ASStackLayoutSpec horizontalStackLayoutSpec];
    avatarAndContent.spacing = 5;
    [avatarAndContent setChildren:@[_avatarNode, stackLayout]];
    
    /**
     *  返回一个最终设置完所有subnode布局规则layout对象，这个layout对象切记不要缓存
     *  cell的内边距通过 UIEdgeInsetsMake(10, 10, 10, 10) 设置
     */
    ASInsetLayoutSpec * insetLayout = [ASInsetLayoutSpec
                                       insetLayoutSpecWithInsets:UIEdgeInsetsMake(10, 10, 10, 10)//cell的内边距
                                       child:avatarAndContent];
    
    return insetLayout;
}

/**
 *  类似UIView的 layoutSubviews实现，用来做最后的frame手动调整
 *  在此方法中，通过 self.calculatedSize 获取最终计算出来的node的尺寸size
 */
- (void)layout {
    [super layout];
    
    _avatarNode.layer.cornerRadius = 1;
    _avatarNode.layer.masksToBounds = YES;
    _avatarNode.layer.borderWidth = .7;
    _avatarNode.layer.borderColor = RGB(201, 201, 201).CGColor;
    
    /**
     *  调整cellNode 最下面的的 分割线node
     */
    CGFloat pixelHeight = 1.0f / [[UIScreen mainScreen] scale];
    _sepLine.frame = CGRectMake(0.0f, self.calculatedSize.height - pixelHeight, self.calculatedSize.width, pixelHeight);
}

@end
```

- (3) DLMutiImageNode 多图片显示的node
- (4) DLSharedNode 显示分享数据的node
- (5) DLFavourAndComment 显示评论数据的node

如上几个sub node就不贴代码了，可以参考其中创建node、添加node、布局node内部所有其他subNode的写法。

布局方式有点类似 flex布局，有一些不明白的属性可以参考 flex布局 中的解释。

其实使用`ASDK Node`代替`UIKit UIView`换个思路去写UI的层级结构组织、布局等等的代码。并且ASDK可以异步子线程创建Node，也需要注意一些对象的线程安全。

UIView的布局是`layoutSubviews`，而ASDK Node的布局有两个:

- (1) `layoutSpecThatFits:` >>> 主要是创建一个 `ASLayoutSpec`类型的一个布局对象，并包裹一些需要布局的Node对象

- (2) 常见的ASLayoutSpec布局:
	- ASStackLayoutSpec: 类似iOS9推出的 stack layout，即线性布局
		- vertical 垂直方向布局 sub nodes
		- horrizental 水平向下布局 sub nodes

- (3) `layout` >>> 布局规则对象会把`所有的子节点`都 `计算完毕`，这个时间点可以对一些node的布局进行微调，或者设置一些`不太好使用布局对象设置`的node的布局



看完ASDK基础使用之后，感觉还是基础太薄弱，直接去看ASDK的源码实现有点难度，还是需要从基础来。弄清楚iOS系统如何进行`图像渲染`的。

