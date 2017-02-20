## 动画： 从旧值变化到新值的一个缓慢的变化过程

当你改变CALayer的一个可做动画的属性，它并不能立刻在屏幕上体现出来。相反，它是从`先前的值`平滑过渡到`新的值`。这一切都是默认的行为，你不需要做额外的操作。

而这个缓慢的、自动的变化过程形成了动画，这就是所谓的`隐式动画`。

## 随机改变图层颜色的代码

```objc
@interface ViewController ()

@property (nonatomic, weak) IBOutlet UIView *layerView;
@property (nonatomic, weak) IBOutlet CALayer *colorLayer;

@end

@implementation ViewController

- (void)viewDidLoad
{
    [super viewDidLoad];
    
    //1. create sublayer
    self.colorLayer = [CALayer layer];
    
    //2. 
    self.colorLayer.frame = CGRectMake(50.0f, 50.0f, 100.0f, 100.0f);
    
    //3.
    self.colorLayer.backgroundColor = [UIColor blueColor].CGColor;
    
    //4. add it to our view
    [self.layerView.layer addSublayer:self.colorLayer];
}

- (IBAction)changeColor
{
    //1. randomize the layer background color
    CGFloat red = arc4random() / (CGFloat)INT_MAX;
    CGFloat green = arc4random() / (CGFloat)INT_MAX;
    CGFloat blue = arc4random() / (CGFloat)INT_MAX;
    
    //2. 改变layer的backgroudColor属性值
    self.colorLayer.backgroundColor = [UIColor colorWithRed:red green:green blue:blue alpha:1.0].CGColor;                                                                                       ￼
}

@end
```

我们仅仅改变了CALayer的一个属性，然后Core Animation来决定如何并且何时去做动画。

## 事务: 就像将多个sql语句放在一起执行，CoreAnimation也是有事务的概念的

### CoreAnimation的事务（`CATransaction`）:

- (1) 包含一系列属性动画操作的集合
- (2) 其中被包含的某一个动画操作，并不会立刻让layer发生变化
- (3) 而是当事务被`提交`的时候，开始一个接一个的执行所有的动画


### CoreAnimation事务模板代码:

```objc
//1.
[CATransaction begin];
    
//2. 各种动画操作
//...................
    
//3.
[CATransaction commit];
```

eg、

```objc
//1. begin a new transaction
[CATransaction begin];

//2. set the animation duration to 1 second
[CATransaction setAnimationDuration:1.0];

//3. randomize the layer background color
CGFloat red = arc4random() / (CGFloat)INT_MAX;
CGFloat green = arc4random() / (CGFloat)INT_MAX;
CGFloat blue = arc4random() / (CGFloat)INT_MAX;

//4.
self.colorLayer.backgroundColor = [UIColor colorWithRed:red green:green blue:blue alpha:1.0].CGColor;

￼//5. commit the transaction
[CATransaction commit];
```

CATransaction这个类设计的很奇怪，CATransaction没有属性或者实例方法，并且也不能用`+alloc`和`-init`方法创建它。


> Core Animation在`每个runloop周期`中`自动开始一次新`的事务（run loop是iOS负责收集用户输入，处理定时器或者网络事件并且重新绘制屏幕的东西），即使你不显式的用[CATransaction begin]开始一次事务，任何在一次run loop循环中属性的改变都会被集中起来，然后做一次`0.25秒`的动画。


## 上面是直接对一个单独的CALayer修改值后触发的隐式动画，那么对UIView内部的layer进行值修改了？

试着直接对UIView关联的图层做动画而不是一个单独的图层:

```objc
@interface ViewController ()

@property (nonatomic, weak) IBOutlet UIView *layerView;

@end

@implementation ViewController

- (void)viewDidLoad
{
    [super viewDidLoad];
    
    //set the color of our layerView backing layer directly
    self.layerView.layer.backgroundColor = [UIColor blueColor].CGColor;
}

- (IBAction)changeColor
{
    //begin a new transaction
    [CATransaction begin];
    
    //set the animation duration to 1 second
    [CATransaction setAnimationDuration:1.0];
    
    //randomize the layer background color
    CGFloat red = arc4random() / (CGFloat)INT_MAX;
    CGFloat green = arc4random() / (CGFloat)INT_MAX;
    CGFloat blue = arc4random() / (CGFloat)INT_MAX;
    
    //【重要】直接对UIView关联的图层做动画而不是一个单独的图层
    self.layerView.layer.backgroundColor = [UIColor colorWithRed:red green:green blue:blue alpha:1.0].CGColor;

    //commit the transaction
    [CATransaction commit];
}
```

运行程序发现，颜色`瞬间`就切换到了新的值，而不是之前平滑过渡的动画。

为什么对UIView内部的CALayer做值修改，并不能触发隐式动画了？

> UIView关联图层，默认关闭了隐式动画功能。

## 测试单独修改CALayer属性时发送的事情

> 当CALayer属性值修改后，会自动回调调用 -[CALayer actionForKey:]

首先是，自定义一个CALayr子类，重写`-[CALayer actionForKey:]`:

```objc
@interface MyLayer : CALayer
@end
@implementation MyLayer

- (id<CAAction>)actionForKey:(NSString *)event {
    NSLog(@"-[MyLayer actionForKey:] >>>> event = %@", event);
    return [super actionForKey:event];
}

@end
```

然后是ViewController修改CALayer的属性值:

```objc
#import "MyLayer.h"

@implementation YinShiAnimateVC {
    MyLayer *_myLayer;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor whiteColor];
    
    //1. create sublayer
    _myLayer = [MyLayer layer];
    
    //2.
    _myLayer.frame = CGRectMake(50.0f, 100.0f, 100.0f, 100.0f);
    
    //3.
    _myLayer.backgroundColor = [UIColor blueColor].CGColor;
    
    //4. add it to our view
    [self.view.layer addSublayer:_myLayer];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    //1. randomize the layer background color
    CGFloat red = arc4random() / (CGFloat)INT_MAX;
    CGFloat green = arc4random() / (CGFloat)INT_MAX;
    CGFloat blue = arc4random() / (CGFloat)INT_MAX;
    
    //2. 改变layer的backgroudColor属性值
    _myLayer.backgroundColor = [UIColor colorWithRed:red green:green blue:blue alpha:1.0].CGColor;
}

@end
```

当程序一开始执行，就会输出如下打印

```
2017-02-19 12:39:32.421 CoreAnimationDemo[1806:25238] -[MyLayer actionForKey:] >>>> event = position
2017-02-19 12:39:32.421 CoreAnimationDemo[1806:25238] -[MyLayer actionForKey:] >>>> event = bounds
2017-02-19 12:39:32.421 CoreAnimationDemo[1806:25238] -[MyLayer actionForKey:] >>>> event = backgroundColor
2017-02-19 12:39:32.424 CoreAnimationDemo[1806:25238] -[MyLayer actionForKey:] >>>> event = onOrderIn
2017-02-19 12:39:32.426 CoreAnimationDemo[1806:25238] -[MyLayer actionForKey:] >>>> event = onLayout
2017-02-19 12:39:32.935 CoreAnimationDemo[1806:25238] -[MyLayer actionForKey:] >>>> event = onOrderOut
2017-02-19 12:39:32.935 CoreAnimationDemo[1806:25238] -[MyLayer actionForKey:] >>>> event = onOrderIn
```

然后点击屏幕就会不断的输出如下

```
2017-02-19 12:40:21.164 CoreAnimationDemo[1806:25238] -[MyLayer actionForKey:] >>>> event = backgroundColor
```

```
2017-02-19 12:40:32.801 CoreAnimationDemo[1806:25238] -[MyLayer actionForKey:] >>>> event = backgroundColor
```

改变属性时CALayer自动应用的动画称作行为，当CALayer的属性被修改时候，它会调用`-actionForKey:`方法，`传递属性的名称`。`-actionForKey:`方法实现做了如下几件事:

- (1) layer首先检测它是否有委托，并且是否实现`CALayerDelegate`协议指定的`-actionForLayer:forKey`方法。如果有，直接调用并返回结果。

- (2) 如果没有委托，或者委托没有实现`-actionForLayer:forKey`方法，layer接着检查包含属性名称对应行为映射的`actions字典`

- (3) 如果`actions字典`没有包含对应的属性，那么layer接着在它的`style字典`接着搜索属性名

- (4) 最后，如果在`style`里面也找不到对应的行为，那么layer将会直接调用定义了每个属性的标准行为的`-defaultActionForKey:`方法

所以一轮完整的搜索结束之后，

- (1) `-actionForKey:`要么返回空，这种情况下将不会有动画发生
- (2) `-actionForKey:`返回一个`CAAction协议`实现类的对象，最后CALayer拿这个结果去对先前和当前的值做动画

OK，测试下是不是这样的，改成如下:

```objc
#import "MyLayer.h"

@implementation YinShiAnimateVC {
    MyLayer *_myLayer;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor whiteColor];
    
    //1. create sublayer
    _myLayer = [MyLayer layer];
    
    //2.
    _myLayer.frame = CGRectMake(50.0f, 100.0f, 100.0f, 100.0f);
    
    //3.
    _myLayer.backgroundColor = [UIColor blueColor].CGColor;
    
    //4. add it to our view
    [self.view.layer addSublayer:_myLayer];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    //1.
    [CATransaction begin];
    
    //2.
    [CATransaction setAnimationDuration:2];
    
    //3. randomize the layer background color
    CGFloat red = arc4random() / (CGFloat)INT_MAX;
    CGFloat green = arc4random() / (CGFloat)INT_MAX;
    CGFloat blue = arc4random() / (CGFloat)INT_MAX;
    
    //4. 改变layer的backgroudColor属性值
    _myLayer.backgroundColor = [UIColor colorWithRed:red green:green blue:blue alpha:1.0].CGColor;
    
    //5.
    [CATransaction commit];
}

@end
```

```objc
@implementation MyLayer

//- (id<CAAction>)actionForKey:(NSString *)event {
//    NSLog(@"-[MyLayer actionForKey:] >>>> event = %@", event);
//    return [super actionForKey:event];
//}

- (id<CAAction>)actionForKey:(NSString *)event {
    
    //1.
    NSLog(@"event = %@", event);

    //2.
    id<CAAction> action = [super actionForKey:event];
    NSLog(@"action = %@", action);
    
    return action;
}

@end
```

运行后点击屏幕输出

```
2017-02-19 12:52:26.370 CoreAnimationDemo[1880:32146] event = backgroundColor
2017-02-19 12:52:26.370 CoreAnimationDemo[1880:32146] action = <CABasicAnimation: 0x7fe1295ca120>
```

看到运行的效果，就是背景颜色有一个2秒的变化过程，因为找到并返回了一个`id<CAAction>`对象。

<img src="./隐式动画1.gif" alt="" title="" width="300"/>


那么，尝试让`-[MyLayer actionForKey:]`返回nil了？

```objc
- (id<CAAction>)actionForKey:(NSString *)event {
    
    //1.
    NSLog(@"event = %@", event);

    //2.
    id<CAAction> action = [super actionForKey:event];
    NSLog(@"action = %@", action);
   
    //3. 返回nil
    return nil;
}
```

<img src="./隐式动画2.gif" alt="" title="" width="300"/>

可以看到，一下子就颜色变了，没有一个缓慢的变化过程。


## 于是这就解释了UIKit是如何禁用隐式动画的：

- (1) 每个UIView对它关联的图层都扮演了一个委托，并且提供了`-actionForLayer:forKey`的实现方法

- (2) 当直接修改`UIView/UIView.layer`的属性值时，`-[UIView actionForLayer:forKey]` 返回`nil`

- (3) 当使用`-[UIView animateWithDuration:animations:]`时，`-[UIView actionForLayer:forKey]` 返回一个`id<CAAction>对象`

返回nil并不是禁用隐式动画唯一的办法，CATransaction有个方法叫做`+setDisableActions:`，可以用来对所有属性打开或者关闭隐式动画。

```
[CATransaction setDisableActions:YES];
```

当改变一个layer的属性:

- (1) 属性`值`的确是立刻更新的（如果你读取它的数据，你会发现它的值在你设置它的那一刻就已经生效了）
- (2) 但是`屏幕上`并没有马上发生改变
- (3) 这是因为你设置的属性值，并没有直接调整图层的外观。相反，他只是定义了图层动画`结束之后`将要变化的外观。


## 实际上，iOS动画体系遵循了MVC的设计模式:

- (1) CALayer作为用户操作的界面`View`
- (2) CALayer又充当了存储最终在屏幕上，如何显示和动画的数据模型`Model`
- (3) CoreAniamtion扮演了控制器`Controller`

(1)与(2)好像CALayer既充当了View如何显示，又充当了存储各种显示数据的Model。

确实是这样的，但其实是因为CALayer内部还有两个子layer:

- (1) 呈现图层/呈现图层树

```c
CALayer presentationLayer = -[CALayer presentationLayer];
```

- (2) 模型图层/模型图层树

```c
CALayer modelLayer = -[CALayer modelLayer];
```

这两个layer干嘛的了？

## presentationLayer 与 modelLayer 

首先要明白的是，其实我们操作的CALayer并不是最重要的，其实只是操作内部两个layer的入口。

> 最终屏幕上如何显示layer、以及显示什么颜色、大小、位置...都是presentationLayer和modelLayer记录的。


#### presentationLayer:

- (1) 我们在屏幕上看到的一切效果，都是`presentationLayer`的数据
- (2)`presentationLayer`的数据，在每一次进行屏幕绘制的时候才会去使用

#### modelLayer:

- (1) 我们我们对`CALayer`的各种绘图属性进行赋值和访问，实际上都是访问的`modelLayer`的属性（比如bounds、backgroundColor、position等）
- (2) 对这些属性进行赋值，不会影响`presentationLayer`，也就是不会影响绘制内容

## iOS系统每次进行屏幕重新绘制的时间间隔（FPS，Frames Per Second，每一秒刷新的帧数）

- (1) 在iOS中，屏幕`每一秒`钟重绘屏幕`60次 或 60帧数`
- (2) 意味着每一次屏幕绘制的时间间隔只有`16毫秒（1/60 == 0.016..）`
- (3) 但是出去系统大概`5毫秒+`的开销
- (4) 实际上每一次进行界面绘制的时间，大概只有`10毫秒`

所以每一次我们编写的界面呈现代码，完成时间需要限定在`10毫秒`左右，否则会引起`丢失帧数`，造成`界面卡顿`的效果。

也就是说明，在`设置要显示的数据`和`屏幕进行数据绘制`之间是有一个`16毫秒`的时间间隔的。

只是这个`16毫秒`太小，人的肉眼是无法感知出来的，就好像`设置数据`和`绘制数据`是在同一个时间，但其实是有一个`16毫秒`间隔的。

## presentationLayer 与 modelLayer 协作完成数据显示

- (1) 假设`t0`时刻，我们对CALayer进行了属性值修改，`t1`时刻接收到了一个`屏幕绘制`的指令

- (2) 然后CoreAnimation首先从`modelLayer`，取出修改的属性值 >>> 要呈现在屏幕上的效果

- (3) 如果动画时间超过`1/60秒`，可能会切割成多个帧进行完成，那么CoreAnimation还会将(2)取出的数据，减去从`presentationLayer`取出的当前显示数据，计算当前帧需要显示成的样子

- (4) 然后当前帧数据已经绘制到屏幕上了，CoreAnimation立刻将当前显示后的数据写入到`presentationLayer`中保存


可以这样简单的理解:

- (1) presentationLayer: 当前屏幕上显示的样子
- (2) modelLayer: 下一帧或将要显示的样子，暂存的数据

## 我们不断的对modelLayer（或对CALayer）进行写操作时，并不会立刻触发屏幕的绘制。因为`屏幕的绘制信号`是由iOS系统来发送的，我们是无法来控制的。

考虑如下场景:

- (1) 当前屏幕正在完成绘制`第一针`，即正在读取`第一帧`对应的modelLayer中的数据

- (2) 又对modelLayer进行操作，从`点1--->点2--->点3`

- (3) 完成`第一针`的绘制后，此时又开始完成`第二针`

- (4) 直接将CALayer绘制到`点3`的位置。而不是先绘制`点1`，再绘制`点2，最后绘制`点3`。


可以看到这种方式极大的提高了绘制效率:

- (1) 我们不断的对modelLayer写入新的绘制数据
- (2) 但是并不会立刻就会读取modelLayer的数据，进行屏幕绘制
- (3) 而是等到`接收到系统的屏幕绘制信号`的时候，才会从modelLayer中，读取当前帧要显示的数据进行绘制
- (4) 并且读取的是这一帧，CALayer`最终`要显示的样子，对之前的`同类型`设置的数据直接忽略了

## CAAnimation对presentationLayer的控制

如果在CALayer中使用`addAnimation:forKey:`添加了一个CAAnimation之后，那么就会`打破`之前`presentationLayer与modelLayer`之间的协作关系:

- (1) 此时，当前CALayer要显示的数据，全部来自于CAAnimation计算后的数据

- (2) modelLayer被无情的暂时抛弃，等待CAAnimation执行完毕

- (3) CAAnimation执行完毕，CALayer要显示的数据，又由modelLayer开始控制了

- (4) 而此时modelLayer还是处于最开始的状态，那么CALayer会绘制到最开始的状态，即回到动画之前的位置

```
可以通过设置，来让modelLayer改变为动画之后的数据
A.fillMode = kCAFillModeForwards
```

- (5) presentationLayer的数据，存储为当前modelLayer的数据


## 小结

- (1) CALayer内部存在另外两个layer
	- presentationLayer 存储当前屏幕上显示的layer数据（`已经`绘制的）
	- modelLayer 存储即将要绘制到屏幕上的layer数据（`还没有`绘制的）

- (2) 我们经常操作的UIView或CALayer的bounds、frame、position..等，其实都是操作CALayer内部的`modelLayer`

- (3) 我们不断的对`modelLayer`进行写时，并不会每次都立刻出发屏幕的绘制

- (4) 而是等到接收到屏幕绘制命令时，会从`modelLayer`中取出当前帧要显示的数据，并过滤掉中间的一些变化过程，直接读取最后变化成的样子数据进行绘制

- (5) 当给CALayer添加CAAnimation后，会暂时代替`modelLayer`的位置。等待CAAnimation执行完毕，`modelLayer`才会恢复执行

- (6) FPS，屏幕每一秒刷新60次。每一次刷新完成时间为`1/60秒 == 16毫秒`，但是出去系统的大概5s消耗，对于我们开发者真正时间为`10毫秒`

- (7) 如果我们写的代码在每一帧消耗时间，最好保持在`10毫秒`上下，否则就会引起`当前帧`处理事件过多，而耽误`下一帧的数据处理`，从而造成丢失帧数，引起界面卡顿的效果