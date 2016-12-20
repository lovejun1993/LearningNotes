---
layout: post
title: "AutoLayout3"
date: 2015-08-12 11:00:04 +0800
comments: true
categories: 
---

###Masonry使用记录一

***

###使用Masonry完成如下例子

- 例子1、UILabel高度动态变化、以及动态改变UIView的高度

- 例子2、动态的添加UIView

- 例子3、UIScrollView

- 例子4、键盘上方的ToolBar工具栏

- 例子5、Alert

****

###大概步骤:

- 第一步、创建subviews对象，添加到superView

- 第二步、设置一些不会变化的subviews的约束

- 第三步、声明属性变量或约束对象，来存放要改变的约束的值或直接保存一个约束对象

```
eg.1

@property (nonatomic, assign) CGFloat middleLabelHeight;
```

```
eg.2

@property (nonatomic, strong) MASConstraint *middleLabelConstraint
```

- 第四步、需要改变约束一定要在`-[UIView updateConstraints]`方法里面进行设置

	- 第一种，直接在 `-[UIView updateConstraints]`设置全部的约束
	- 第二种，在subviews创建时就设置约束，需要改变的约束使用属性保存起来，然后在`-[UIView updateConstraints]`修改


```
eg.1 将所有的约束都在回调函数设置

- (void)updateConstraints {
    
    [self.middleLabel mas_updateConstraints:^(MASConstraintMaker *make) {
        
        //1. 这些是不会变化值的约束make.left.mas_equalTo(self.mas_left).offset(10);
        make.right.mas_equalTo(self.mas_right).offset(-10);
        make.top.mas_equalTo(self.topLabel.mas_bottom).offset(10);
		
		//2. 如下这个约束的值是会变化的，直接将约束中的值使用变化后的变量值即可
        //表示约束优先级普通，一旦约束变化时，可能会影响其他View的布局
        //make.height.mas_equalTo(self.middleLabelHeight);
        //表示约束优先级较低，不会影响其他View的布局
        make.height.mas_equalTo(self.middleLabelHeight).priorityLow();
    }];
    
    //这句必学写
    [super updateConstraints];
}
```

```
eg.2 在回调函数设置只设置部分需要改变的约束

- (void)updateConstraints {
    
	//1. 先取消这个约束
    [self.middleLabelConstraint uninstall];
    
    //2. 再使用新的约束
    self.middleLabelConstraint.mas_equalTo(self.middleLabelHeight).priorityLow();
    
    //这句必学写
    [super updateConstraints];
}
    
```

- 第五步、修改约束的值，触发执行动画，会执行`-[UIView updateConstraints]`

```
eg. 

//1. 改变高度
self.middleLabelHeight = 50.f;

//2. 更新约束
[self setNeedsUpdateConstraints];
[self updateConstraintsIfNeeded];

//3. 执行动画
[UIView animateWithDuration:0.4 animations:^{
    [self layoutIfNeeded];
}];
```

****


###例子1、UILabel高度动态变化、以及动态改变UIView的高度

####效果图示
	
- 点击之前

![](http://i5.tietuku.com/52e1d12e9832f24c.png)
	
- 点击之后

![](http://i5.tietuku.com/7e406fdefd2148f1.png)
	
- 点击之后，多出了一个View `请填写手机号码`


####代码实现

代码都在一个单独的UIView子类`RootView`完成

- `RootView.h`

```
#import "RootView.h"
#import <Masonry/Masonry.h>
#import <ReactiveCocoa/ReactiveCocoa.h>
#import <ReactiveCocoa/RACEXTScope.h>

@interface RootView : UIView

@end
```

- `RootView.m` 实现方式1

```
@interface RootView ()

//顶部文字描述的Label
@property (nonatomic, strong) UILabel *topLabel;

//中间的Label，高度需要动态改变的
@property (nonatomic, strong) UILabel *middleLabel;

//底部按钮
@property (nonatomic, strong) UIButton *bottmButton;

//记录改变高度Label的高度值
@property (nonatomic, assign) CGFloat middleLabelHeight;

//保存中间的Label，高度约束
@property (nonatomic, strong) MASConstraint *middleLabelConstraint;

@end
```

```
@implementation RootView

- (void)dealloc {
    //....
}

- (id)initWithCoder:(NSCoder *)aDecoder {
    self = [super initWithCoder:aDecoder];
    if (self) {
    
    	//1. 初始化变量值
        [self initVariables];
        
        //2. 创建Subviews
        [self initSubviews];
    }
    return self;
}

//初始化变量值
- (void)initVariables {

	//默认中间Label的高度，开始为0
    self.middleLabelHeight = 0;
}

- (void)initSubviews {
    
    //1. 创建TopLabel，并设置约束
    [self initTopLabel];
    
    //2. 创建MiddleLabel，默认高度为0
    [self initMiddleLabel];
    
    //3. 创建底部按钮
    [self initBottmButton];
}

- (void)initTopLabel {
    
    self.topLabel = [[UILabel alloc] init];
    self.topLabel.layer.borderWidth = 2;
    
    //必须设置1，Label内容的字体
    self.topLabel.font = [UIFont fontWithName:@"HelveticaNeue-Bold" size:17.0f];
    
    //必须设置2，Label内容自动换行
    self.topLabel.numberOfLines = 0;
    
    //必须设置3，Label内容最大行宽度，才能动态的改变Label高度
    self.topLabel.preferredMaxLayoutWidth = [UIScreen mainScreen].bounds.size.width - 2 * 10;
    
    [self addSubview:self.topLabel];
    
    self.topLabel.text = @"dawd不阿布速度爱上度噶预算的刮痧油的刮痧油冬瓜意思冬瓜是高端是孤独呀阿哥速度呀A股上移动古雅是够短时孤独感udawd不阿布速度爱上度噶预算的刮痧油的刮痧油冬瓜意思冬瓜是高端是孤独呀阿哥速度呀A股上移动古雅是够短时孤独感udawd不阿布速度爱上度噶预算的刮痧油的刮痧油冬瓜意思冬瓜是高端是孤独呀阿哥速度呀A股上移动古雅是够短时孤独感udawd不阿布速度爱上度噶预算的刮痧油的刮痧油冬瓜意思冬瓜是高端是孤独呀阿哥速度呀A股上移动古雅是够短时孤独感udawd不阿布速度爱上度噶预算的刮痧油的刮痧油冬瓜意思冬瓜是高端是孤独呀阿哥速度呀A股上移动古雅是够短时孤独感udawd不阿布速度爱上度噶预算的刮痧油的刮痧油冬瓜意思冬瓜是高端是孤独呀阿哥速度呀A股上移动古雅是够短时孤独感u";
    
    //设置约束
    [self.topLabel mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.mas_equalTo(self.mas_left).offset(10);
        make.right.mas_equalTo(self.mas_right).offset(-10);
        make.top.mas_equalTo(self.mas_top).offset(20);
        
        //注意: 不需要设置高度的约束，因为高度是动态变化的
    }];
}

- (void)initMiddleLabel {
    
    self.middleLabel = [[UILabel alloc] init];
    self.middleLabel.numberOfLines = 1;
    self.middleLabel.font = [UIFont fontWithName:@"HelveticaNeue-Bold" size:14.0f];
    [self addSubview:self.middleLabel];
 	
 	//middleLabel的约束的需要改变，不在此设置约束   
}

- (void)initBottmButton {
    
    self.bottmButton = [UIButton buttonWithType:UIButtonTypeCustom];
    [self.bottmButton setBackgroundColor:[UIColor lightGrayColor]];
    [self.bottmButton setTitle:@"点击改变中间Label的高度" forState:UIControlStateNormal];
    [self addSubview:self.bottmButton];
    
    [self.bottmButton mas_updateConstraints:^(MASConstraintMaker *make) {
        make.top.mas_equalTo(self.middleLabel.mas_bottom).offset(10);
        make.left.mas_equalTo(self.mas_left).offset(10);
        make.right.mas_equalTo(self.mas_right).offset(-10);
        make.height.mas_equalTo(40);
        make.bottom.lessThanOrEqualTo(self.mas_bottom).offset(-10);
    }];

	//----------------------------------------------------------------------------------
    //按钮点击之后，改变middleLabel的高度约束值
    @weakify(self);
    [[self.bottmButton rac_signalForControlEvents:UIControlEventTouchUpInside] subscribeNext:^(id x) {
        @strongify(self);
        
        //1. 改变高度
        self.middleLabelHeight = 50.f;

        //2. 设置Label显示内容
        self.middleLabel.text = @"请填写手机号码...";

        //3. 通知要更新约束，会执行 -[UIView updateConstraints]方法
        [self setNeedsUpdateConstraints];
        [self updateConstraintsIfNeeded];
        
        //4. 约束改变过程，执行动画
        [UIView animateWithDuration:0.4 animations:^{
            [self layoutIfNeeded];
        }];
    }];
}

#pragma mark - 更新约束计算Frame

/**
 * 改变约束的的View，其约束都需要在该方法中设置
 */

- (void)updateConstraints {
    
    //设置中间Label的约束
    [self.middleLabel mas_updateConstraints:^(MASConstraintMaker *make) {
        make.left.mas_equalTo(self.mas_left).offset(10);
        make.right.mas_equalTo(self.mas_right).offset(-10);
        make.top.mas_equalTo(self.topLabel.mas_bottom).offset(10);

        //表示约束优先级普通，一旦约束变化时，可能会影响其他View的布局
        //make.height.mas_equalTo(self.middleLabelHeight);
        
        //表示约束优先级较低，不会影响其他View的布局
        make.height.mas_equalTo(self.middleLabelHeight).priorityLow();
    }];
    
    //这句必学写
    [super updateConstraints];
}

@end
```

- `RootView.m` 实现方式2，主要对上述修改如下几个方法

```
- (void)initMiddleLabel {
    
    self.middleLabel = [[UILabel alloc] init];
    self.middleLabel.numberOfLines = 1;
    self.middleLabel.font = [UIFont fontWithName:@"HelveticaNeue-Bold" size:14.0f];
    [self addSubview:self.middleLabel];
    
    //保存需要改变的约束对象
    [self.middleLabel mas_updateConstraints:^(MASConstraintMaker *make) {
        make.left.mas_equalTo(self.mas_left).offset(10);
        make.right.mas_equalTo(self.mas_right).offset(-10);
        make.top.mas_equalTo(self.topLabel.mas_bottom).offset(10);
        
        //表示约束优先级普通，一旦约束变化时，可能会影响其他View的布局
        //make.height.mas_equalTo(self.middleLabelHeight);
        
        //表示约束优先级较低，不会影响其他View的布局
        self.middleLabelConstraint = make.height.mas_equalTo(self.middleLabelHeight).priorityLow();
    }];
}
```

```
//只需要修改需要改变的约束，而不需要改变所有的约束

- (void)updateConstraints {
    
    //设置中间Label的约束
    
	//1. 先取消这个约束
    [self.middleLabelConstraint uninstall];
    
    //2. 再设置新的约束self.middleLabelConstraint.mas_equalTo(self.middleLabelHeight).priorityLow();
    
    //这句必学写
    [super updateConstraints];
}
```

****

###例子2、有点像例子1让中间的Label慢慢的变出来，给别人错觉是动态添加进来的

####效果图

- 点击前，只有三个View

![](http://i5.tietuku.com/9d65ddc1a958898b.png)
	
- 点击之后，多出两个View的位置，并且压缩了紫色的高度

![](http://i5.tietuku.com/e2b8a812a5aeca81.png)
	
	
####代码实现

代码都在一个独立的UIView子类`DynamicRootView`中

- `DynamicRootView.h`

```
@interface DynamicRootView : UIView

@property (strong, nonatomic) UIView *firstView;
@property (strong, nonatomic) UIView *secondtView;
@property (strong, nonatomic) UIView *thirdView;
@property (strong, nonatomic) UIView *fourthView;
@property (strong, nonatomic) UIView *fiveView;

//记录改变的高度
@property (assign, nonatomic) NSInteger firstViewH;
@property (assign, nonatomic) NSInteger thirdViewH;
@property (assign, nonatomic) NSInteger fiveViewH;

@end
```

- `DynamicRootView.m`

```
#import "DynamicRootView.h"
#import <Masonry/Masonry.h>
#import <ReactiveCocoa/ReactiveCocoa.h>
#import <ReactiveCocoa/RACEXTScope.h>
```

```
@implementation DynamicRootView

- (void)dealloc {
    //....
}

- (id)initWithCoder:(NSCoder *)aDecoder {
    self = [super initWithCoder:aDecoder];
    if (self) {
        [self initSubviews];
    }
    return self;
}

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        [self initSubviews];
    }
    return self;
}

- (void)initSubviews {
    
    //添加一个点击手势，测试点击后改变约束
    [self addGestureRecognizer:[[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(update:)]];
    
    _firstView = [[UIView alloc] init];
    _secondtView = [[UIView alloc] init];
    _thirdView = [[UIView alloc] init];
    _fourthView = [[UIView alloc] init];
    _fiveView = [[UIView alloc] init];
    
    _firstView.backgroundColor = [UIColor redColor];
    _secondtView.backgroundColor = [UIColor blueColor];
    _thirdView.backgroundColor = [UIColor yellowColor];
    _fourthView.backgroundColor = [UIColor grayColor];
    _fiveView.backgroundColor = [UIColor purpleColor];
    
    [self addSubview:_firstView];
    [self addSubview:_secondtView];
    [self addSubview:_thirdView];
    [self addSubview:_fourthView];
    [self addSubview:_fiveView];
    
    //一开始高度设置为0，后面代码改变高度
    _firstViewH = 0;
    _thirdViewH = 0;
    _fiveViewH = 0;
}

#pragma mark - 这个函数必须写
+ (BOOL)requiresConstraintBasedLayout
{
    return YES;
}

//这个函数为了防止y起始点下移64
- (UIRectEdge)edgesForExtendedLayout {
    return UIRectEdgeNone;
}

#pragma mark - 手势点击回调
- (void)gestureDidClick:(UITapGestureRecognizer *) gesture{
    
    //1. 改动的约束
    self.firstViewH = 50;
    self.thirdViewH = 80;
    self.fiveViewH = 100;
    
    //2. 通常更新约束
    [self setNeedsUpdateConstraints];
    [self updateConstraintsIfNeeded];
    
    //3. 执行动画
    [UIView animateWithDuration:0.8 animations:^{
        [self layoutIfNeeded];
    }];
}

#pragma mark - 在更新约束回调方法中设置subviews的约束
- (void)updateConstraints {
    
    //view1 高度动态改变
    [self.firstView mas_updateConstraints:^(MASConstraintMaker *make) {
        make.left.mas_equalTo(self.mas_left).offset(10);
        make.right.mas_equalTo(self.mas_right).offset(-10);
        make.top.mas_equalTo(@5);
        make.height.mas_equalTo(self.firstViewH).priorityLow();//约束可以改变
    }];
    
    //view2 高度不变
    [self.secondtView mas_updateConstraints:^(MASConstraintMaker *make) {
        make.left.mas_equalTo(self.mas_left).offset(10);
        make.right.mas_equalTo(self.mas_right).offset(-10);
        make.bottom.mas_equalTo(self.thirdView.mas_top).offset(-10);
        make.top.mas_equalTo(self.firstView.mas_bottom).offset(10);
        make.height.mas_equalTo(@50).priorityHigh();//约束不能改变
    }];
    
    //view3 高度动态改变
    [self.thirdView mas_updateConstraints:^(MASConstraintMaker *make) {
        make.left.mas_equalTo(self.mas_left).offset(10);
        make.right.mas_equalTo(self.mas_right).offset(-10);
        make.bottom.mas_equalTo(self.fourthView.mas_top).offset(-10);
        make.top.mas_equalTo(self.secondtView.mas_bottom).offset(10);
        
        make.height.mas_equalTo(self.thirdViewH).priorityLow();//约束可以改变
    }];
    
    //view4 高度不变
    [self.fourthView mas_updateConstraints:^(MASConstraintMaker *make) {
        make.left.mas_equalTo(self.mas_left).offset(10);
        make.right.mas_equalTo(self.mas_right).offset(-10);
        make.bottom.mas_equalTo(self.fiveView.mas_top).offset(-10);
        make.top.mas_equalTo(self.thirdView.mas_bottom).offset(10);
        
        make.height.mas_equalTo(@100).priorityLow();//约束不能改变
    }];
    
    //view5 高度动态改变
    [self.fiveView mas_updateConstraints:^(MASConstraintMaker *make) {
        make.left.mas_equalTo(self.mas_left).offset(10);
        make.right.mas_equalTo(self.mas_right).offset(-10);
        make.bottom.mas_equalTo(self.mas_bottom).offset(-10).priorityLow();//约束可以改变
        make.top.mas_equalTo(self.fourthView.mas_bottom).offset(10);
    }];
    
    [super updateConstraints];
}

@end
```

***

###例子3、ScrollView

####图示效果

![](http://i5.tietuku.com/8b91d0a931999c80.png)

####代码实现

- 自定义UIScrollView `ScrollRootView.h`

```
#import <UIKit/UIKit.h>

@interface ScrollRootView : UIView

//传入一个数组初始化subviews
@property (strong, nonatomic) NSArray *dataList;

@end
```

- 自定义UIScrollView `ScrollRootView.m`

```
#import "ScrollRootView.h"
#import <Masonry/Masonry.h>
#import "UIColor+RandomColor.h"

@interface ScrollRootView ()

//里面有一个UIScrollView
@property (strong, nonatomic) UIScrollView *scrollView;

//添加到UIScrollView的管理subviews的容器的，最后使用这个contentView计算约束得到contentSize
@property (strong, nonatomic) UIView *contentView;

@end
```

```
@implementation ScrollRootView

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        [self initScrollView];
    }
    return self;
}

//创建UIScrollview实例
- (void)initScrollView {

	//1. 
    _scrollView = [[UIScrollView alloc] init];
    [self addSubview:_scrollView];
    
    //2. 给scrollView设置初始时的约束
    [_scrollView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.edges.equalTo(self);
    }];
}

//创建scrollView内部的contentView
- (void)initContentView {

	//1. 
    _contentView = UIView.new;
    [_scrollView addSubview:_contentView];
    
    //2. 给contentView设置初始时的约束
    [_contentView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.edges.equalTo(self.scrollView);
        make.width.equalTo(self.scrollView);//纵向滚动
//        make.height.equalTo(self.scrollView);//横向滚动
    }];
}

#pragama mark - 根据传入的数组个数，创建subviews，并添加到contentView

- (void)setDataList:(NSArray *)dataList {
    
    _dataList = dataList;
    
    //创建contentView实例
    [self initContentView];
    
    //保存最后一个被添加到contentView的subview
    __block UIView *lastViwe = nil;
    
    //间距
    CGFloat padding = 10;
    
    //依次创建subview，并添加到contentView
    for (NSString *value in _dataList) {
        
        //1.
        UIView *subView = UIView.new;
        subView.backgroundColor = [UIColor randomColor];
        
        //2.
        [_contentView addSubview:subView];
        
        //3. 添加手势点击
        UITapGestureRecognizer *singleTap = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(singleTap:)];
        [subView addGestureRecognizer:singleTap];
        
        //4. 设置subview的约束
        [subView mas_makeConstraints:^(MASConstraintMaker *make) {
        
        	//1. 顶部约束需要判断
            make.top.mas_equalTo(lastViwe ? lastViwe.mas_bottom : @0).offset(padding);
            
            //其他的都不变
            make.height.mas_equalTo(@80);
            make.left.mas_equalTo(_contentView.mas_left).offset(padding);
            make.right.mas_equalTo(_contentView.mas_right).offset(-padding);
        }];
        
        //5. 最后一定要保存为最后添加到subview
        lastViwe = subView;
    }
    
    //6. 根据最后一个被添加的subview的底部，来确定contentView的高度
    [_contentView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.bottom.mas_equalTo(lastViwe.mas_bottom);
    }];
}

//scrollview点击
- (void)singleTap:(UITapGestureRecognizer*)sender {

	//1. 设置被点击的view透明度变化
    [sender.view setAlpha:sender.view.alpha / 1.20];
    
    //2. 让scrollView滚动到被点击这个view的位置
    [self.scrollView scrollRectToVisible:sender.view.frame animated:YES];
};

@end
```

***

###例子4、键盘上的工具栏

- 键盘View

```
@interface KeyBoardView : UIView

@end

@implementation KeyBoardView

@end
```

- TooBar

```
@interface ToolBarView ()

@property (nonatomic, strong) KeyBoardView *keyBoardView;

@end
```

```
@implementation ToolBarView

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        [self initSubveiws];
    }
    return self;
}

- (void)initSubveiws {
    
    UITextField *txt = [[UITextField alloc] init];
    txt.placeholder = @"....";
    [self addSubview:txt];
    
    [txt mas_makeConstraints:^(MASConstraintMaker *make) {
        make.center.mas_equalTo(self);
        make.size.mas_equalTo(CGSizeMake(200, 45));
    }];
    
    _keyBoardView = [[KeyBoardView alloc] init];
    _keyBoardView.backgroundColor = [UIColor redColor];
    [self addSubview:_keyBoardView];
    
    [_keyBoardView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.centerX.mas_equalTo(self.mas_centerX);
        make.width.mas_equalTo(self.mas_width);
        make.height.mas_equalTo(44);
        make.bottom.mas_equalTo(self);
    }];
    
    //键盘通知
    NSNotificationCenter *center = [NSNotificationCenter defaultCenter];
    
    [center addObserver:self
               selector:@selector(keyboardEvent:)
                   name:UIKeyboardWillChangeFrameNotification
                 object:nil];
}

#pragma mark - 

- (void)keyboardEvent:(NSNotification *)notify {
    
    CGFloat duration = [notify.userInfo[UIKeyboardAnimationDurationUserInfoKey] floatValue];
    
    CGRect keyBoardFrame = [notify.userInfo[UIKeyboardFrameEndUserInfoKey] CGRectValue];
    
    if (self.frame.size.height == keyBoardFrame.origin.y) {
        //键盘未弹起，还原位置
        [UIView animateWithDuration:duration animations:^{
            _keyBoardView.transform = CGAffineTransformIdentity;
        }];
        
    } else {
        //键盘已经弹起，改变位置
        CGFloat h = keyBoardFrame.size.height;
        [UIView animateWithDuration:duration animations:^{
            _keyBoardView.transform = CGAffineTransformMakeTranslation(0, -h);
        }];
    }
}

@end
```

- 小结:

	- 并没有通过修改约束值，来改变位置
	- 而是通过Transform形变，直接改变frame来完成
		- 因为此时肯定是有frame
	- 所以不一定所有的都得用`改变约束`，也可以结合使用`Transform`来改变位置


***

###例子5、Alert

####效果图示

- 被点击之前

![](http://i5.tietuku.com/e9836ca12ceebb5f.png)

- 点击之后，弹出一个AlertView，显示输入框风格

![](http://i5.tietuku.com/ff9d7cd2fd143181.png)

- 点击之后，弹出一个AlertView，显示提示信息

![](http://i5.tietuku.com/587146ae51261274.png)

- 点击之后，弹出一个AlertView，显示提示信息，且当提示信息超过一定`高度`时，内部使用`UITextView`来显示

![](http://i5.tietuku.com/05fdcd0661d56497.jpg)


####代码实现

- 先看看封装后的使用Api

```
@implementation AlertDemoController


//展示显示信息的Alert
- (void)showInfo {
    
    //1.
    SFPayMessageAlertView *vc = [[SFPayMessageAlertView alloc] init];
    
    //2.
    [vc showInfoWithTile:@"标题" Content:@"n你好啊啊啊hidden安徽宿大户室大户室胡" DoneButton:@"我知道了" OnDoneButtonClick:^(SFPayAlertButton *button) {
        
    } CancelButton:@"取消" OnCancelButtonClick:^(SFPayAlertButton *button) {
        
    } duration:2 InViewController:self OnDissmiss:^{
        
    }];
}

//展示编辑框的Alert
- (void)showEdit {
    //1.
    SFPayMessageAlertView *vc = [[SFPayMessageAlertView alloc] init];
    
    //2.
    [vc addInputItemWithPlaceholder:@"请输入手机号码..." OnValueChanged:^(SFPayAlertTextfiled *clickItem) {

    }];

    [vc addInputItemWithPlaceholder:@"请输入用户名..." OnValueChanged:^(SFPayAlertTextfiled *clickItem) {

    }];

    [vc addInputItemWithPlaceholder:@"请输入密码..." OnValueChanged:^(SFPayAlertTextfiled *clickItem) {

    }];

    [vc addInputItemWithPlaceholder:@"请输入密码..." OnValueChanged:^(SFPayAlertTextfiled *clickItem) {

    }];
    
    [vc showEditWithTile:@"填写信息" Content:@"f安顿好杀敌阿红丝带阿是但还是大海" SubmitButton:@"提交" OnSubmitButtonClick:^(NSMutableArray *dataArray) {
        
    } CancelButton:@"取消" OnCancelButtonClick:^{
        
    } duration:2 InViewController:self

    OnDissmiss:^{
        
    }];

}


@end
```

- 实现1、Alert中的每一个按钮自定义类 `SFPayAlertButton`

```
#import <UIKit/UIKit.h>

@interface SFPayAlertButton : UIButton

@property (copy, nonatomic) void (^onButtonClick)(SFPayAlertButton *clickedButton);

@end
```

```
#import "SFPayAlertButton.h"

@implementation SFPayAlertButton

@end
```

主要就是保存一个Block，回传按钮自己

- 实现2、Alert中的每一个输入框自定义类 `SFPayAlertTextfiled`

```
#import <UIKit/UIKit.h>

@interface SFPayAlertTextfiled : UITextField

@property (copy, nonatomic)void (^onChanged)(SFPayAlertTextfiled *filed);

@end
```

```
#import "SFPayAlertTextfiled.h"

@implementation SFPayAlertTextfiled

@end
```

主要就是保存一个Block，回传输入框自己

- 实现3、AlertView

	- 其实是继承自一个UIViewController的控制器类

	
- `SFPayMessageAlertView.h`

```
#import <UIKit/UIKit.h>
#import <Foundation/Foundation.h>
#import "SFPayAlertButton.h"
#import "SFPayAlertTextfiled.h"

//定义Alert显示的样式
typedef NS_ENUM(NSInteger, XZHAlertViewType) {
    XZHAlertViewTypeInfo = 1,	//信息
    XZHAlertViewTypeEdit ,	//编辑
    XZHAlertViewTypeWaring , //警告
    XZHAlertViewTypeFail , //失败
};
```

```
@interface SFPayMessageAlertView : UIViewController

/* 在Alert上添加按钮 */
- (SFPayAlertButton *)addButtonItemWithTitle:(NSString *)title
                              OnClick:(void (^)(SFPayAlertButton *clickedButton))callback;

/* 在Alert上添加输入框 */
- (SFPayAlertTextfiled *)addInputItemWithPlaceholder:(NSString *)placeHolder
                            OnValueChanged:(void (^)(SFPayAlertTextfiled *clickItem)) callback;

/* 显示弹窗信息 */
- (void)showInfoWithTile:(NSString *)title
                 Content:(NSString *)content
              DoneButton:(NSString *)done
       OnDoneButtonClick:(void (^)(SFPayAlertButton *button)) onDoneClick
            CancelButton:(NSString *)cancel
     OnCancelButtonClick:(void (^)(SFPayAlertButton *button)) onCancelClick
                duration:(NSTimeInterval)duration
        InViewController:(UIViewController *)targetController
              OnDissmiss:(void (^)(void)) onDismiss;

/* 显示输入框 */
- (void)showEditWithTile:(NSString *)title
                 Content:(NSString *)content
            SubmitButton:(NSString *)submit
     OnSubmitButtonClick:(void (^)(NSMutableArray *dataArray))onDoneClick
            CancelButton:(NSString *)cancel
     OnCancelButtonClick:(void (^)(void))onCancelClick
                duration:(NSTimeInterval)duration
        InViewController:(UIViewController *)targetController
              OnDissmiss:(void (^)(void)) onDismiss;

@end
```

- `SFPayMessageAlertView.m`

```
#import "SFPayMessageAlertView.h"
#import <Masonry/Masonry.h>

//解决键盘遮挡
#import <IQKeyboardManager/IQKeyboardManager.h>
```

```
//iOS系统版本判断宏
#define SYSTEM_VERSION_EQUAL_TO(v)                  ([[[UIDevice currentDevice] systemVersion] compare:v options:NSNumericSearch] == NSOrderedSame)
#define SYSTEM_VERSION_GREATER_THAN(v)              ([[[UIDevice currentDevice] systemVersion] compare:v options:NSNumericSearch] == NSOrderedDescending)
#define SYSTEM_VERSION_GREATER_THAN_OR_EQUAL_TO(v)  ([[[UIDevice currentDevice] systemVersion] compare:v options:NSNumericSearch] != NSOrderedAscending)
#define SYSTEM_VERSION_LESS_THAN(v)                 ([[[UIDevice currentDevice] systemVersion] compare:v options:NSNumericSearch] == NSOrderedAscending)
#define SYSTEM_VERSION_LESS_THAN_OR_EQUAL_TO(v)     ([[[UIDevice currentDevice] systemVersion] compare:v options:NSNumericSearch] != NSOrderedDescending)
```

```
//定义输入框最小输入的长度
static NSInteger kTextfileLength = 6;
```

```
@interface SFPayMessageAlertView () <UITextFieldDelegate> {
	
	//AlertView的宽度
    NSInteger _AlertViewWidth;
    
	//AlertView的高度
    NSInteger _AlertViewHeight;
    
    //subviews之间的间距
    NSInteger _Padding;
    
    //标题Label的高度
    NSInteger _TitleLabelHeight;
    
    NSInteger _ButtonHeight;
    NSInteger _FiledHeight;
    
    //记录内容显示的最高高度
    //1.没有超过这个高度，就使用 _contentLabel显示
    //2.一旦超过这个高度，就使用 _textView来滚动显示
    NSInteger _MaxTextHeight;
    
    //记录所有的输入框
    NSMutableArray *_inputs;
    
    //记录所有的按钮
    NSMutableArray *_buttons;
    
    //标题Label
    UILabel *_titleLabel;
    
    //显示提示信息的Label，当提示信息没有超过规定长度时
    UILabel *_contentLabel;
    
    //显示提示信息的UItextView，当提示信息超过规定长度时，替换掉_contentLabel来显示信息
    UITextView *_textView;
    
    //标题Label的字体
    UIFont *_titleLabelFont;
    
    //提示信息的字体
    UIFont *_contentTextFont;
    
    //按钮的字体
    UIFont *_buttonTextFont;
    
    //保存添加到当前控制器View的最下方的一个View
    UIView *_lastView;
    
    //保存传入的控制器对象
    UIViewController *_targetController;
    
    //添加到控制器view上的AlertView
    UIView *_rootView;
    
    //当前Alert显示的样式
    XZHAlertViewType _type;
    
    //计时器
    NSTimer *_timer;
}

//当Alert移除时，执行的Block
@property (copy, nonatomic) void (^onDismiss)(void);

//当Alert提交按钮点击时，执行的Block，并回传输入框的数据
@property (copy, nonatomic) void (^onSubmit)(NSArray *datas);

@end
```

####接下来都是`@implementation SFPayMessageAlertView ... @end`的实现代码

- dealloc函数，释放内存，移除键盘通知

```
- (void)dealloc {

    _rootView = nil;
    _inputs = nil;
    _buttons = nil;
    _timer = nil;
    
    _titleLabelFont = nil;
    _contentTextFont = nil;
    _buttonTextFont = nil;
    
    _onDismiss = nil;
    _onSubmit = nil;
    
    [[NSNotificationCenter defaultCenter] removeObserver:self name:UIKeyboardWillShowNotification object:nil];
    [[NSNotificationCenter defaultCenter] removeObserver:self name:UIKeyboardWillHideNotification object:nil];
}
```

- init函数

```
- (instancetype)init {
    self = [super init];
    if (self) {
    
    	//1. 初始化变量值
        [self initVariables];
        
        //2. 创建subviews
        [self initSubviews];
    }
    return self;
}

```

- 计算比例，返回当前屏幕宽度与320的比例，后续的所有宽度和高度，都得乘上这个比例

```
- (NSInteger)screenWithRate {
    NSInteger rate = [UIScreen mainScreen].bounds.size.width / 320;
    return rate;
}
```

- initVariables函数，初始化所有的变量值

```
- (void)initVariables {
    
    //都得乘上这个比例

    _AlertViewWidth = 280 * [self screenWithRate];
	
    _AlertViewHeight = 0;
    
    _Padding = 10 * [self screenWithRate];
    
    
    _TitleLabelHeight = 30 * [self screenWithRate];
    
    
    _ButtonHeight = 44 * [self screenWithRate];
    
    
    _FiledHeight = 40 * [self screenWithRate];
    
    _MaxTextHeight = 300 * [self screenWithRate];
    
    _inputs = [NSMutableArray array];
    
    _buttons = [NSMutableArray array];
    
    //监听键盘事件
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(onKeyboardWillShow:) name:UIKeyboardWillShowNotification object:nil];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(onKeyboardWillHide:) name:UIKeyboardWillHideNotification object:nil];
    
    [[IQKeyboardManager sharedManager] setEnable:YES];
}
```

- 键盘事件回调处理

```
- (void)onKeyboardWillShow:(NSNotification *)sender {
	
	//没用
}

- (void)onKeyboardWillHide:(NSNotification *)sender {
    BOOL isValid = YES;
    
    NSMutableArray *array = [NSMutableArray array];
    
    //查看所有的输入框的值，是否都满足最小长度
    for (SFPayAlertTextfiled *filed in _inputs) {
    
        isValid = isValid && [self isLengthForTextfile:filed];
        [array addObject:filed.text];
        
        //如果没满足最小长度，就让输入框的layer color显示红色，如果符合还原颜色
        if ([self isLengthForTextfile:filed]) {
            filed.layer.borderColor = [UIColor blackColor].CGColor;
        }else{
            filed.layer.borderColor = [UIColor redColor].CGColor;
        }
    }
}
```

- initSubviews函数，创建所有的subviews，添加到当前控制器View上

```
- (void)initSubviews {
    
    // font
    _titleLabelFont = [UIFont fontWithName:@"HelveticaNeue-Bold" size:17.0f];
    _contentTextFont = [UIFont fontWithName:@"HelveticaNeue" size:14.0f];
    _buttonTextFont = [UIFont fontWithName:@"HelveticaNeue" size:14.0f];
    
    // rootView，也就是AlertView
    _rootView = [[UIView alloc] init];
    _rootView.backgroundColor = [UIColor whiteColor];
    _rootView.userInteractionEnabled = YES;
    _rootView.layer.cornerRadius = 10.0f;
    _rootView.layer.masksToBounds = YES;
    _rootView.layer.borderWidth = 3.0f;
    _rootView.layer.borderColor = [UIColor lightGrayColor].CGColor;
    [self.view addSubview:_rootView];
    
    // titleLabel
    _titleLabel = [[UILabel alloc] init];
    _titleLabel.numberOfLines = 1;
    _titleLabel.font = _titleLabelFont;
    _titleLabel.textAlignment = NSTextAlignmentCenter;
    [_rootView addSubview:_titleLabel];
    
    [_titleLabel mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.mas_equalTo(_rootView.mas_left);
        make.right.mas_equalTo(_rootView.mas_right);
        make.top.mas_equalTo(_rootView.mas_top).offset(_Padding);
        make.height.mas_equalTo(_TitleLabelHeight);
    }];
    
    //累计AlertView的高度
    _AlertViewHeight += _TitleLabelHeight;
    
    // contentLabel
    //先累加一个间距的高度
    _AlertViewHeight += _Padding;
    
    _contentLabel = [[UILabel alloc] init];
    [_rootView addSubview:_contentLabel];
    _contentLabel.textColor = [UIColor blackColor];
    _contentLabel.preferredMaxLayoutWidth = _AlertViewWidth - 2 * _Padding;//设置最大显示宽度
    _contentLabel.numberOfLines = 0;//自动换行
    _contentLabel.font = _contentTextFont;
    
    // textView
    _textView = [[UITextView alloc] init];
    [_rootView addSubview:_textView];
    _textView.editable = NO;
    _textView.font = _contentTextFont;
    _textView.textColor = [UIColor redColor];
    _textView.allowsEditingTextAttributes = YES;
    _textView.textAlignment = NSTextAlignmentCenter;
    
    //按系统版本，设置UITextView的内容的上、下、左、右、的缩进为0
    if (SYSTEM_VERSION_GREATER_THAN_OR_EQUAL_TO(@"7.0"))
    {
        _textView.textContainerInset = UIEdgeInsetsZero;
        _textView.textContainer.lineFragmentPadding = 0;
    }
    
    //累加间距高度
    _AlertViewHeight += _Padding;
}
```

- 小结`initSubviews`函数做的事情:

	- 1) `_rootView`也就是作为AlertView
	- 2) 首先先把所有的subviews都创建，并且添加到`_rootView`，不需要的就先`hiden`
	- 3) 给`_titleLabel`设置了约束，因为他的frame是不会变化的
	- 4) 此时只是累计了`padding` + `_titleLabel的定高度` + `padding`总高度

- 显示提示信息入口

```
- (void)showInfoWithTile:(NSString *)title
                 Content:(NSString *)content
              DoneButton:(NSString *)done
       OnDoneButtonClick:(void (^)(SFPayAlertButton *button)) onDoneClick
            CancelButton:(NSString *)cancel
     OnCancelButtonClick:(void (^)(SFPayAlertButton *button)) onCancelClick
                duration:(NSTimeInterval)duration
        InViewController:(UIViewController *)targetController
              OnDissmiss:(void (^)(void)) onDismiss
{	

	//1. 添加subveiws
    //如果需要添加Done按钮（OK按钮）
    if (done) {
    	[self addButtonItemWithTitle:done OnClick:[onDoneClick copy]];
    }
    
	//如果需要添加取消按钮
    if (cancel) {
		[self addButtonItemWithTitle:cancel OnClick:[onDoneClick copy]];
    }
    
    //2. 调用创建输入框和按钮的统一入口函数
    //完成添加完成的所有subviews的约束设置
    [self showAlertViewWithTitle:title
                         Content:content
                            Type:XZHAlertViewTypeInfo
                        duration:3
                InViewController:targetController
                      OnDissmiss:onDismiss];
}
```

- 显示编辑框的入口

```
- (void)showEditWithTile:(NSString *)title
                 Content:(NSString *)content
            SubmitButton:(NSString *)submit
     OnSubmitButtonClick:(void (^)(NSMutableArray *dataArray))onSubmit
            CancelButton:(NSString *)cancel
     OnCancelButtonClick:(void (^)(void))onCancelClick
                duration:(NSTimeInterval)duration
        InViewController:(UIViewController *)targetController
              OnDissmiss:(void (^)(void)) onDismiss
{
	//保存提交按钮点击后的回调执行Block
    self.onSubmit = [onSubmit copy];
    
    __weak __typeof(self)weakSelf = self;
    
    //1. 添加subveiws
    //1.1 添加一个按钮提交按钮，并给按钮设置好回调Block，就是检查所有输入框
    if (submit) {

        [self addButtonItemWithTitle:submit
                             OnClick:^(SFPayAlertButton *clickedButton)
        {
            __strong __typeof(weakSelf)strongSelf = weakSelf;
            
            //验证所有输入框的输入值长度是否合法
            BOOL isValid = YES;
            NSMutableArray *array = [NSMutableArray array];
            for (SFPayAlertTextfiled *filed in strongSelf->_inputs) {
                isValid = isValid && [strongSelf isLengthForTextfile:filed];
                [array addObject:filed.text];
                
                if ([strongSelf isLengthForTextfile:filed]) {
                    filed.layer.borderColor = [UIColor blackColor].CGColor;
                }else{
                    filed.layer.borderColor = [UIColor redColor].CGColor;
                }
            }
            
            if (isValid) {
                
                //如果长度都合法，执行提交按钮点击的回调Block
                if (strongSelf.onSubmit) {
                    strongSelf.onSubmit(array);
                }
                
                //释放block
                strongSelf.onSubmit = nil;
                
                //执行Alert移除Block
                [strongSelf onAlertViewDismiss:nil];
            }
        }];
    }
    
    //1.2 添加一个按钮取消按钮，设置回调Block
    if (cancel) {
        
        [self addButtonItemWithTitle:cancel OnClick:^(SFPayAlertButton *clickedButton) {
            __strong __typeof(weakSelf)strongSelf = weakSelf;
            
            //清除所有输入框的值
            for (SFPayAlertTextfiled *filed in strongSelf->_inputs) {
                filed.text = @"";
            }
            
            //执行取消按钮被点击的回调Block
            if (onCancelClick) onCancelClick();
        }];
    }
    
    //2. 调用创建输入框和按钮的统一入口函数
	//完成添加完成的所有subviews的约束设置
    [self showAlertViewWithTitle:title
                         Content:content
                            Type:XZHAlertViewTypeEdit
                        duration:duration
                InViewController:targetController
                      OnDissmiss:onDismiss];
}

```


- 统一在Alert`_rootView`上完成最终的操作
	
	- 添加所有的subviews
	- 设置所有subviews的约束
	- 移除不要的subview
	- 输入框校验
	- 按钮点击事件，按钮的Block设置


```
- (void)showAlertViewWithTitle:(NSString *)title
                       Content:(NSString *)content
                          Type:(XZHAlertViewType) type
                      duration:(NSTimeInterval)duration
              InViewController:(UIViewController *)targetController
                    OnDissmiss:(void (^)(void)) onDismiss
{	
	//copy block
    _onDismiss = [onDismiss copy];
    
    //保存要显示的Alert样式
    _type = type;
    
    //设置控制器View的背景色，带有透明度
    self.view.backgroundColor = [[UIColor lightGrayColor] colorWithAlphaComponent:0.5];
    
    //保存要显示Alert的目标控制器
    _targetController = targetController;
    
    //将我们自己的AlertController.view 添加到 目标控制器.view 
    [targetController.view addSubview:self.view];
    
    //让我们AlertController引用目标控制器
    [targetController addChildViewController:self];
    
    //添加点击手势，点击之后，执行移除Alert
    UITapGestureRecognizer *tap = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(onAlertViewDismiss:)];
    [self.view addGestureRecognizer:tap];
    
    //设置 _titleLable显示的内容
    [_titleLabel setText:title];
    
    //根据显示的内容长度，判断使用 _contentLabel 还是 _textView
    [_contentLabel setText:content];
    
    //计算 _contentLabel设置内容后的一个size
    __block CGSize size = [_contentLabel sizeThatFits:CGSizeMake(_contentLabel.preferredMaxLayoutWidth, MAXFLOAT)];
    
    //如果高度大于规定高度，使用 _textView来代替显示内容
    if (size.height > _MaxTextHeight) {//超过规定高度
        
        //将 _contentLabel移除
        [_contentLabel removeFromSuperview];
        _contentLabel = nil;
        
        //使用 _textView来显示内容
        _textView.text = content;
        
        //设置 _textView 约束
        [_textView mas_makeConstraints:^(MASConstraintMaker *make) {
            make.top.mas_equalTo(_titleLabel.mas_bottom).offset(_Padding);
            
            //textView的高度为 _MaxTextHeight 
            make.height.mas_equalTo(_MaxTextHeight);
            
            make.left.mas_equalTo(_rootView).offset(_Padding);
            make.width.mas_equalTo(_AlertViewWidth - 2 * _Padding);
        }];
        
        //记录 _textView 为最后加入的subview
        _lastView = _textView;
        
        //累计 _textView的高度
        _AlertViewHeight += _MaxTextHeight;
        
        //累计一个间距的高度
        _AlertViewHeight += _Padding;
        
    } else {
        
        //内容没有超过规定高度，
        [_textView removeFromSuperview];
        
        //记录 _contentLabel 为最后加入的subview
        _lastView = _contentLabel;
        
        //这里不用记录 _contentLabel 的高度
        //因为此种情况时，可以通过 _lastView 来设置下一个subview的top约束
    }
    
    //添加所有的输入框
    for (UITextField *filed in _inputs) {
        
        //设置约束
        [filed mas_makeConstraints:^(MASConstraintMaker *make) {
            make.top.mas_equalTo(_lastView.mas_bottom).offset(_Padding);
            make.left.mas_equalTo(_rootView).offset(_Padding);
            make.width.mas_equalTo(_AlertViewWidth - 2 * _Padding);
            make.height.mas_equalTo(_FiledHeight);
        }];
        
        //累计一个间距的高度
        _AlertViewHeight += _Padding;
        
        //累计一个输入框的高度
        _AlertViewHeight += _FiledHeight;
        
        //记录 filed 为最后加入的subview
        _lastView = filed;
    }
    
    // 内容显示下面的一条分割线
    UIView *line = [[UIView alloc] init];
    [_rootView addSubview:line];
    line.backgroundColor = [UIColor lightGrayColor];
    [line mas_makeConstraints:^(MASConstraintMaker *make) {
        make.top.mas_equalTo(_lastView.mas_bottom).offset(_Padding);
        make.height.mas_equalTo(1);
        make.left.mas_equalTo(_rootView.mas_left);
        make.width.mas_equalTo(_rootView);
    }];
    
    //记录 line 为最后加入的subview
    _lastView = line;
    
    //累计一个间距的高度
    _AlertViewHeight += _Padding;
    
    //所有按钮的创建，分两种情况
    //1. 只有一个按钮
    //2. 两个按钮
    if ([_buttons count] == 1) {
    
    	//设置约束
        SFPayAlertButton *btn = [_buttons objectAtIndex:0];
        [btn mas_makeConstraints:^(MASConstraintMaker *make) {
            make.left.mas_equalTo(_rootView.mas_left).offset(_Padding);
            make.width.mas_equalTo(_AlertViewWidth - 2 * _Padding);
            make.top.mas_equalTo(_lastView.mas_bottom).offset(_Padding);
            make.height.mas_equalTo(_ButtonHeight);
        }];
        
        //累计一个按钮的高度
        _AlertViewHeight += _ButtonHeight;
        
        //累计一个间距的高度
        _AlertViewHeight += _Padding;
        
        //记录 btn 为最后加入的subview
        _lastView = btn;
        
    } else if ([_buttons count] == 2) {
        
        //计算得到每一个按钮的宽度
        NSInteger width = (_AlertViewWidth - 3 * _Padding) / 2;
        
        UIView *lastTmpView = nil;
        
        for (SFPayAlertButton *btn in _buttons) {
            [btn mas_makeConstraints:^(MASConstraintMaker *make) {
                make.left.mas_equalTo((lastTmpView?lastTmpView.mas_right:@0)).offset(_Padding);
                make.width.mas_equalTo(width);
                make.top.mas_equalTo(_lastView.mas_bottom).offset(_Padding);
                make.height.mas_equalTo(_ButtonHeight);
            }];
            
            //记录最后添加的subview
            lastTmpView = btn;
        }
        
        //累计按钮高度
        _AlertViewHeight += _ButtonHeight;
        
        //累计一个间距的高度
        _AlertViewHeight += _Padding;
        
    } else {
        //按钮数大于 > 2，那么纵向排列按钮
        //待实现...
    }
}
```

- 小结

	- 根据计算出`_contentLabel`设置内容后的size.height，如果超过规定高度，将其溢出，换成`UITextView`来显示内容

	- `_AlertViewHeight`记录了当前subviews在`_rootView`中的`总高度`

- 从目标控制器.view 移除 AlertController.view

```
- (void)onAlertViewDismiss:(id)sender {

    __weak __typeof(self)weakSelf = self;
    dispatch_async(dispatch_get_main_queue(), ^{
        __strong __typeof(weakSelf)strongSelf = weakSelf;
        
        //移除
        [strongSelf.view removeFromSuperview];
        [strongSelf removeFromParentViewController];
        
        //执行Block回调
        if (strongSelf.onDismiss) {
            strongSelf.onDismiss();
        }
    });
}
```

- 添加按钮

```
#pragma mark  添加按钮
- (SFPayAlertButton *)addButtonItemWithTitle:(NSString *)title
                              OnClick:(void (^)(SFPayAlertButton *clickedButton))callback
{
    SFPayAlertButton *button = [SFPayAlertButton buttonWithType:UIButtonTypeCustom];
    [_rootView addSubview:button];
    
    //复制传入的按钮的Block
    button.onButtonClick = callback;
    
    button.titleLabel.font = _buttonTextFont;
    [button setTitleColor:[UIColor blackColor] forState:UIControlStateNormal];
    [button setTitle:title forState:UIControlStateNormal];
    
    //按钮的点击事件，统一处理
    [button addTarget:self action:@selector(buttonDidClick:) forControlEvents:UIControlEventTouchUpInside];
	
	//保存到数组
    [_buttons addObject:button];

    return button;
}

- (void)buttonDidClick:(SFPayAlertButton *)clickedBtn
{
    __weak __typeof(self)weakSelf = self;
    dispatch_async(dispatch_get_main_queue(), ^{
        __strong __typeof(weakSelf)strongSelf = weakSelf;
        
        if (clickedBtn.onButtonClick) {
            clickedBtn.onButtonClick(clickedBtn);
        }
        
        //如果Alert只是显示提示信息，点击按钮后，直接移除Alert
        if (strongSelf->_type == XZHAlertViewTypeInfo) {
            [strongSelf onAlertViewDismiss:nil];
        }
    });
}
```

-  添加输入框

```
- (SFPayAlertTextfiled *)addInputItemWithPlaceholder:(NSString *)placeHolder
                            OnValueChanged:(void (^)(SFPayAlertTextfiled *clickItem)) callback
{
    SFPayAlertTextfiled *filed = [[SFPayAlertTextfiled alloc] init];
    filed.delegate = self;
    filed.layer.borderWidth = 1;
    filed.placeholder = placeHolder;
    filed.onChanged = callback;
    [_rootView addSubview:filed];
    
    [_inputs addObject:filed];
    
    return filed;
}
```

- UITextfiledDelegate方法，检验输入框的值是否合法

```
- (BOOL)textFieldShouldReturn:(SFPayAlertTextfiled *)textField {
    if (textField == [_inputs lastObject]) {
        if ([self isLengthForTextfile:textField]) {
            [textField resignFirstResponder];
            textField.layer.borderColor = [UIColor blackColor].CGColor;
        } else {
            textField.layer.borderColor = [UIColor redColor].CGColor;
        }
    } else {
        if ([self isLengthForTextfile:textField]) {
            NSInteger currentIdx = [_inputs indexOfObject:textField];
            [[_inputs objectAtIndex:(currentIdx + 1)] becomeFirstResponder];
            textField.layer.borderColor = [UIColor blackColor].CGColor;
        } else {
            textField.layer.borderColor = [UIColor redColor].CGColor;
        }
    }
    return NO;
}

- (BOOL)isLengthForTextfile:(SFPayAlertTextfiled *)filed {
    if (filed.text.length >= kTextfileLength) {
        return YES;
    }else{
        return NO;
    }
}
```

- 如下方法必须重写

```
+ (BOOL)requiresConstraintBasedLayout {
    return YES;
}
```

- 重写系统更新约束的函数，设置 `_rootView`的最终约束

```
- (void)updateViewConstraints {
    
    //如果显示提示提示，使用的 _contentLabel
    if (_contentLabel) {
        
        //1. 得到size
        __block CGSize size = [_contentLabel sizeThatFits:CGSizeMake(_contentLabel.preferredMaxLayoutWidth, MAXFLOAT)];
        
        //2. 设置 _contentLabel的约束
        [_contentLabel mas_makeConstraints:^(MASConstraintMaker *make) {
            make.left.mas_equalTo(_rootView.mas_left).offset(_Padding);
            make.width.mas_equalTo(_AlertViewWidth - 2 * _Padding);
            make.top.mas_equalTo(_titleLabel.mas_bottom).offset(_Padding);
            
            //主要是设置高度约束
            make.height.mas_equalTo(size.height);
        }];
        
        //3. 累计高度
        _AlertViewHeight += size.height;
        _AlertViewHeight += _Padding;
    }
    
    //最终设置 _rootView（Alert）的约束
    [_rootView mas_makeConstraints:^(MASConstraintMaker *make) {
    
		//设置中心约束
        make.center.mas_equalTo(self.view);
        make.width.mas_equalTo(_AlertViewWidth);
        
        //主要是将最后累计得到的高度
        //设置给 _rootView
        make.height.mas_equalTo(_AlertViewHeight);
    }];
    
    [super updateViewConstraints];
}
```

- 小结更新约束

	- 1) 提示信息Alert时，不能有`输入框`
	- 2) `_AlertViewHeight`变量最终累计的高度，作为`_rootView`的高度

- iOS7

```
- (UIRectEdge)edgesForExtendedLayout {
    return UIRectEdgeNone;
}
```

- 小结:
	
	- 主要是通过 `[targetController.view addSubview:alertControlelr.view];`

	- 累计总高度，作为 Alert的高度

	- 先添加完所有的subviews，然后进行约束的设置

- `https://git.coding.net/xiongzenghui/AutoLayout.git`
