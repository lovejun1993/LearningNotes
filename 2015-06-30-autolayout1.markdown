---
layout: post
title: "AutoLayout1"
date: 2015-06-30 10:59:57 +0800
comments: true
categories: 
---


###AutoLayout与Frame系列函数的区别?

- AutoLayout设置约束中用到的`数字`和Frame中用到的`数字`是一样的东西，并没有什么不同

```
make.top.mas_equal(100);
```

等价于

```
CGRectMake(x,100,width,height);
```

- 使用AutoLayout布局与使用Frame布局的不同	

	- Frame是当View在创建时，我们就自己手动的计算View出现的x,y坐标、以及宽高是多少。这些规则，稍后iOS系统自动根这些规则计算得到显示的frame。

	- AutoLayout是当View在创建时，我们给View对象设置一些约束（frame计算的规则）
		- 设置与superView或者兄弟层级View之间的如下
		- 自己的宽度与高度的比例，与其他人的宽度或高度的比例
		- 上下左右间距
		- 居中X、居中Y、成比例...

- 注意在Autolayout中，最好不要再试图使用Frame，但是实在不行就写一下，最好不要写...
	- 对一个使用约束布局的View对象，在`init执行时`是拿不到正确的frame的
	- 只有在**viewDidLoad、viewDidAppear、layoutSubviews**这些方法调用的时候，才会得到正确的frame
	- 可以在`layoutSubviews`和`viewDidLayoutSubviews`系统重写函数中，完成frame的微调


***

###使用AutoLayout，完成九宫格布局

```
NSInteger Columns = 3;
CGFloat ItemHeight = 80;
CGFloat Gap = 35;

__block UIView *lastView = nil;
    for (int i = 0; i < self.shareList.count; i++) {
        ZSYInviteFriendsItemView *item = _shareList[i];
        [_shareView addSubview:item];
        
        [item mas_makeConstraints:^(MASConstraintMaker *make) {
            
            if (lastView) {
                make.width.equalTo(lastView.mas_width);
            } else {
                make.size.mas_equalTo(CGSizeMake((SCREEN_WIDTH - Gap * (Columns + 1))/Columns, ItemHeight));
            }
            
            if (i % Columns == 0) {
                make.left.equalTo(item.superview).offset(Gap);
            }
            else{
                make.left.equalTo(lastView.mas_right).offset(Gap);
            }
            if (i % Columns == (Columns -1)) {
                make.right.equalTo(item.superview).offset(-Gap);
            }
            int top = (i / Columns) * Gap + ( i / Columns * ItemHeight);
            make.top.equalTo(item.superview).offset(top);
            
            lastView = item;
        }];
    }
```

***

###使用Frame，完成九宫格布局

- 找了一个封装的比较好的九宫格frame计算的代码

```
/**
 *  按照给定的参数，返回当前index在九宫格布局中的frame
 *
 *  @param rect              九宫格布局的最大frame
 *  @param itemCount         总个数
 *  @param perRowItemCount   每一行个数
 *  @param perColumItemCount 每一列个数
 *  @param itemWidth         每一个具体项的宽度
 *  @param itemHeight        每一个具体项的高度
 *  @param paddingX          x轴间距
 *  @param paddingY          y轴间距
 *  @param index             所在下标
 *  @param page              所在页，可以有多个页
 */
- (CGRect)getFrameWithRegionRect:(CGRect)rect
                       ItemCount:(NSInteger)itemCount
                perRowItemCount:(NSInteger)perRowItemCount
              perColumItemCount:(NSInteger)perColumItemCount
                      itemWidth:(CGFloat)itemWidth
                     itemHeight:(NSInteger)itemHeight
                       paddingX:(CGFloat)paddingX
                       paddingY:(CGFloat)paddingY
                        atIndex:(NSInteger)index
                         onPage:(NSInteger)page
{
    //1. 计算得到总共几行
    NSUInteger rowCount = itemCount / perRowItemCount + (itemCount % perColumItemCount > 0 ? 1 : 0);
    
    //2. 得到顶部与底部的多余的高度
    CGFloat insetY = (CGRectGetHeight(rect) - (itemHeight + paddingY) * rowCount) / 2.0;
    
    //3. 获取起点x坐标
    CGFloat originX = (index % perRowItemCount) * (itemWidth + paddingX) + paddingX + (page * CGRectGetWidth(rect));
    
    //4. 获取起点y坐标
    CGFloat originY = ((index / perRowItemCount) - perColumItemCount * page) * (itemHeight + paddingY) + paddingY;
    
    //5. 构造当前index的frame
    CGRect itemFrame = CGRectMake(originX, originY + insetY, itemWidth, itemHeight);
    
    return itemFrame;
}

```

- 使用例子:


```
- (void)layoutButtonsWithRootViewFrame:(CGRect)rect {
    
    UIImage *image = [UIImage imageNamed:@"gesture_node_highlighted"];
    CGFloat itemH = image.size.height;
    CGFloat itemW = image.size.width;
    
    for (int i = 0; i < 9; i++) {
        GestureLockButton *btn = [[_gestureView subviews] objectAtIndex:i];
        
        CGRect frame = [self getFrameWithRegionRect:_gestureView.frame
                                          ItemCount:9
                                    perRowItemCount:3
                                  perColumItemCount:3
                                          itemWidth:itemW
                                         itemHeight:itemH
                                           paddingX:20
                                           paddingY:20
                                            atIndex:i
                                             onPage:0];
        
        btn.frame = frame;

    }
    
    
}
```

***

###使用Masonry动态更新view的高度宽度或者size.

第一步、使用一个变量保存变化的 高度或宽度或size

第二步、在要更新约束的View实例的updateContraints函数中设置view的约束

```
- (void)updateViewConstraints {
    @weakify(self);
    
    //在这里写更改变量的值（改变view的约束值）
    
    //次句代码必须写
    [super updateViewConstraints];
}
```

第三步、某个时刻改变view的属性

```
// 做改变属性的代码
//....

// 后面2句代码，表示马上更新约束
[self.view layoutIfNeeded];
[self.view setNeedsUpdateConstraints];
```

***

###使用Masonry设置View的Size.

```
[thumbImageView mas_makeConstraints:^(MASConstraintMaker *make) {
    
    make.size.equls(CGSizeMake(200,300));
    
}];
```

***

###使用Masonry设置一个View的宽与高成比例.

```
[thumbImageView mas_makeConstraints:^(MASConstraintMaker *make) {
    make.top.equalTo(@0);
    make.left.equalTo(@0);
    
    //比例约束代码
    
    //1. 先设置宽度或者高度
    make.width.equalTo(superView);
	
	//2. 在设置另一个是之前的比例.
	make.height.equalTo(thumbImageView.mas_width).multipliedBy(0.8);
}];
```

***

###使用Masonry实现textaView高度动态改变

```
_labelPwdNotify = [[UILabel alloc] init];
_labelPwdNotify.font = [UIFont flatFontOfSize:13];
_labelPwdNotify.text = ZSYLocalization(@"tip_loginpassword_rule");
_labelPwdNotify.preferredMaxLayoutWidth = SCREEN_WIDTH - 17.5 * 2;
_labelPwdNotify.numberOfLines = 0;
[self.view addSubview:_labelPwdNotify];
    
[_labelPwdNotify mas_makeConstraints:^(MASConstraintMaker *make) {
    @strongify(self);
    make.top.mas_equalTo(self.loginPasswordView.mas_bottom).offset(16);
    make.left.mas_equalTo(self.view.mas_left).offset(17.5);
    make.right.mas_equalTo(self.view.mas_right).offset(-17.5);
}];

```


###使用AutoLayout完成位置变化、大小变化..在变化过程中的放慢的动画.

```
第一步: 修改约束的值
self.heightConstraint.constant += 100; //举个例子

第二步: 执行动画，让使用更改后的约束值重新计算frame
[UIView animateWithDuration:1.0 animations:^{
		[被修改约束的View实例 layoytIfNeed];
}];

```

如上第二步中，表示重新计算

1. 被修改约束的View本身
2. 以及这个View的所有subviews的

这些UIView实例的frame，并在一个时间内完成变化过程.

***

###使用NSLayoutConstraint设置两个同一级别View实例的相对位置约束.

```
constraint = [  
              NSLayoutConstraint  
              constraintWithItem:button1  
              attribute:NSLayoutAttributeBottom  
              relatedBy:NSLayoutRelationEqual  
              toItem:self.view  
              attribute:NSLayoutAttributeBottom  
              multiplier:1.0f  
              constant:-20.0f  
              ];  
  
```

如上的意思是:

```
button1.底部y值 = self.view.底部y值 * 1 - 20;
```

顺便总结下使用VFL使用autolayout的步骤:

1. 从superview的上部到底部.
2. 从superview的左边到右边.
3. 再设置相邻View之间的约束.

****

###使用Masonry简单做一个键盘的遮挡UITextfiled

```
#import <Masonry.h>

@interface ViewController () <UITextFieldDelegate>

@property (nonatomic, strong) UITextField *txtfiled;

//保存设置过的约束，方便后面修改
@property (nonatomic, assign) MASConstraint *constraint;

@end

@implementation ViewController

- (UITextField *)txtfiled {
    if (!_txtfiled) {
        _txtfiled = [[UITextField alloc] init];
        _txtfiled.layer.borderWidth = 1;
        _txtfiled.text = @"请输入....";
    }
    return _txtfiled;
}

- (void)viewDidLoad {
    [super viewDidLoad];

	//注册键盘弹起和隐藏的通知
   [self addNotifications];
    
    //初始化UITextfiled
    self.txtfiled.delegate = self;
    [self.view addSubview:self.txtfiled];
    
    //设置约束
    [self.txtfiled mas_makeConstraints:^(MASConstraintMaker *make) {
        make.centerX.mas_equalTo(self.view);
        make.width.mas_equalTo(self.view.mas_width);
        make.height.mas_equalTo(44);
        
        //保存底部约束的引用
        self.constraint = make.bottom.mas_equalTo(self.view.mas_bottom);
    }];
}

- (void)addNotifications {

    NSNotificationCenter *center  = [NSNotificationCenter defaultCenter];
    
	//1. 
    [center addObserver:self selector:@selector(handleKeyBoardWillShow:) name:UIKeyboardWillShowNotification object:nil];
    
    //2. 
    [center addObserver:self selector:@selector(handleKeyBoardWillHide:) name:UIKeyboardWillHideNotification object:nil];
}

//处理键盘弹起
- (void)handleKeyBoardWillShow:(NSNotification *)notify {
    
    //1. 获取弹出的键盘View的高度
    CGFloat h = [notify.userInfo[UIKeyboardFrameEndUserInfoKey] CGRectValue].size.height;
    
    //2. 改变底部View的底部距离约束
    self.constraint.offset(-h);
}

//处理键盘弹回
- (void)handleKeyBoardWillHide:(NSNotification *)notify {
    
    // 改变底部View的底部距离约束
    self.constraint.offset(0);
    
}

#pragma mark - text filed delegate

//点击return 按钮 去掉
-(BOOL)textFieldShouldReturn:(UITextField *)textField
{
	//取消输入框选中，让键盘弹回
    [self.txtfiled resignFirstResponder];
    
    return YES;
}

@end 
```