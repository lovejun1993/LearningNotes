---
layout: post
title: "layoutSubviews"
date: 2014-06-15 00:13:36 +0800
comments: true
categories: 
---

###layoutSubviews用来布局`subviews`的frame

- 主要处理`内部subview`的frame需要随着`父View`的frame变化而产生变化.

- 对 subviews（当前view内部的所有的子view） 进行frame位置大小的重新布局

- 想更新子视图的位置/大小的时候，可以通过调用layoutSubviews方法完成

- layoutSubviews默认是不做任何事情的，需要在重写这个方法完成对subviews的布局

###layoutSubviews以下情况会被调用

- [当前View对象 setNeedsLayout]
- [当前View对象 addSubview:subview]
- 当前View对象的frame发生改变的时候
- 当前View对象如果是UIScrollView，那么滑动UIScrollView的时候也会调用
- 旋转屏幕会触发`父UIView`的layoutSubviews函数
- 改变一个UIView大小的时候，会触发`父UIView`的layoutSubviews函数

注意:

- init初始化不会触发layoutSubviews
- 不允许直接调用直接调用setLayoutSubviews函数

***

###layoutSubviews方法中，`不能修改当前View自己的frame`，只能对其subviews的frame进行设置，但是可以通过暴露一个方法函数来计算内部subviews显示所需要的最终的整个size大小

比如，九宫格显示图片View，根据传入显示的图片的张数得到最终尺寸

- 规定1: 最多显示9张图片
- 规定2: 一行最多显示3张图片
- 规定3: 最多显示3行

```objc
#import <UIKit/UIKit.h>

//每一张图片的固定 宽度与高度
extern CGFloat PhotoWH;

/**
 *  显示图片九宫格布局的容器View
 */
@interface WeiboPhotoView : UIView

/**
 *  传入显示的图片数组
 */
@property (nonatomic, strong) NSArray *photos;

/**
 *  根据传入显示的图片的张数得到最终WeiboPhotoView显示所需要的尺寸大小
 *
 *  规定1: 最多显示9张图片
 *  规定2: 一行最多显示3张图片
 *  规定3: 最多显示3行
 */
+ (CGSize)photoViewSizeWithPhotoCount:(NSInteger)count;

@end
```

```objc
CGFloat PhotoWH = 70;
CGFloat PhotoPadding = 10;

@implementation WeiboPhotoView

/**
 *  重写setter方法，针对图片数组来决定内部ImageView:
 *  - 重新创建view
 *  - 直接使用缓存的view
 *  - 不需要的hiden隐藏
 */
- (void)setPhotos:(NSArray *)photos {
    _photos = photos;
    
    //添加与传入图片数相对的ImageView
    while (self.subviews.count < _photos.count) {
        WeiboPhotoImageView *subview = [[WeiboPhotoImageView alloc] init];
        [self addSubview:subview];
    }
    
    //遍历ImageView，显示图片
    for (NSInteger index = 0; index < _photos.count; index++)
    {
        UIView *subview = [self.subviews objectAtIndex:index];
        
        if (![subview isMemberOfClass:[WeiboPhotoImageView class]])
            return;

        WeiboPhotoImageView *imageView = (WeiboPhotoImageView *)subview;
        
        ThunbImage *item = [_photos objectAtIndex:index];
        
        if (index < _photos.count) {
            
            imageView.hidden = NO;
            imageView.image = nil;
            
            [imageView setImageURL:[item thumbnail_pic]];
            
        } else {
            
            imageView.hidden = YES;
            imageView.image = nil;
        }
    }
}

#pragma mark - subviews的frame布局

- (void)layoutSubviews {
    [super layoutSubviews];
    
    for (NSInteger i = 0; i < self.subviews.count; i++)
    {
        UIView *subview = [self.subviews objectAtIndex:i];
       
        if (![subview isMemberOfClass:[WeiboPhotoImageView class]])
            return;
            
        if (i < self.photos.count) {
            
            NSInteger row = i / [[self class] maxColCountWithTotolCount:self.photos.count];
            NSInteger col = i % [[self class] maxColCountWithTotolCount:self.photos.count];
            
            CGFloat x = (PhotoPadding + PhotoWH) * col + PhotoPadding;
            CGFloat y = (PhotoPadding + PhotoWH) * row + PhotoPadding;
            
            CGRect rect = CGRectMake(x, y, PhotoWH, PhotoWH);
            
            subview.frame = rect;
            
        } else {
            subview.hidden = YES;
            subview.frame = CGRectZero;
        }
            
    }
}

#pragma mark - 暴露给外部使用计算尺寸

+ (CGSize)photoViewSizeWithPhotoCount:(NSInteger)count {
    
    //1. 最大列数
    NSInteger maxColCount = [self maxColCountWithTotolCount:count];
    
    //2. 计算总列数
    NSInteger colCount = (count >= maxColCount) ? maxColCount : count;
    
    //3. 计算总行数
    NSUInteger rowCount = (count + maxColCount - 1) / maxColCount;
    
    //4. 总宽度
    CGFloat w = colCount * (PhotoPadding + PhotoWH) + PhotoPadding;
    CGFloat h = rowCount * (PhotoPadding + PhotoWH) + PhotoPadding;
    CGSize size = CGSizeMake(w, h);
    
    return size;
}

#pragma mark - Private

/**
 *  根据图片总数，返回最大列数
 */
+ (NSInteger)maxColCountWithTotolCount:(NSInteger)count {
    
    if (count == 4) {//四张就田子布局
        return 2;//返回两列
    } else {
        return 3;
    }
}

@end
```

那么外部使用WeiboPhotoView的代码：

```objc
CGFloat photoX = iconX;
CGFloat photoY = CGRectGetMaxY(_contentLabelFrame) + padding;

// 根据WeiboPhotoView暴露的计算size函数计算得到WeiboPhotoView自己需要的尺寸
CGSize photoSize = [WeiboPhotoView photoViewSizeWithPhotoCount:status.pic_urls.count];
    
_photoImageViewFrame = CGRectMake(photoX, photoY, photoSize.width, photoSize.height);
```

###关于layoutSubviews方法小结:

- 首先ViewA必须有一个确定的frame
- 然后ViewA内部的所有的subviewsN的布局
	- initWithFrame: 方法中 >>> 不会变的布局
	- layoutSubviews: 方法中 >>> 经常会变的布局

***

###重写`setFrame:`方法实现来`固定`当前View的frame，让外界修改不了

```objc
//外界设置frame的入口
- (void)setFrame:(CGRect)frame {
    
    //1. 对外界设置的frame进行内部调整
    //（对宽度调整增加100）
    CGRect rect = frame;
    rect.size.width += 100;
    
    //2. 最后让系统设置我们修改过的frame
    [super setFrame:rect];
}
```

***

###如果现在对于subviewB的x值往右增加，那么联动的让subviewA的宽度跟着往右移动，这个怎么完成了？

> 那么就是 不变的布局条件 与 经常变化的布局条件 并存.

####不变的布局条件

```objc
- (instancetype)initWithFrame:(CGRect)frame;
```

####变化的布局条件

```objc
- (void)layoutSubviews;
```

####结合不变与变化的代码如下

```objc
#import <UIKit/UIKit.h>

@interface DemoView : UIView

@end
```

```objc
#import "DemoView.h"

@interface DemoView ()
@property (nonatomic, strong) UIView *view1;
@property (nonatomic, strong) UIView *view2;
@end

@implementation DemoView

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        _view1 = [[UIView alloc] init];
        _view2 = [[UIView alloc] init];
        [self addSubview:_view1];
        [self addSubview:_view2];
        
        _view1.backgroundColor = [UIColor redColor];
        _view2.backgroundColor = [UIColor blueColor];
        
        // 对 _view1、_view2初始化时设置的布局，一些不会变化的条件
        CGFloat x = 10;
        CGFloat y = 10;
        CGFloat w = 120;
        CGFloat h = 200;
        _view1.frame = CGRectMake(x, y, w, h);
        
        x = 140;
        _view2.frame = CGRectMake(x, y, w, h);
    }
    return self;
}

- (void)layoutSubviews {
    [super layoutSubviews];
    
    // 通过 _view2的最新x值，来计算view1的最新宽度依次+10
    CGFloat w = _view2.frame.origin.x - 2 * 10;
    
    // 将新的宽度设置给 _view1
    CGRect rect = _view1.frame;
    rect.size.width = w;
    _view1.frame = rect;
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    // 每次点击之后修改view2的x值，往右移动10
    CGRect rect = _view2.frame;
    rect.origin.x += 10;
    _view2.frame = rect;
}

@end
```

ViewController测试

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 给DemoView一个确定的frame
    demo = [[DemoView alloc] initWithFrame:CGRectMake(10, 100, 300, 300)];
    [self.view addSubview:demo];
}
```

##自定义View不直接依赖实体类，因为实体类的字段会经常变化。那么自定义View只依赖自己需要显示的数据的抽象结构。外界将实体类对象转换成这个自定义View内部依赖的类型。

.h 

```objc
#import <UIKit/UIKit.h>

/////////////////////////////////数据源/////////////////////////////////
@interface ShopCartViewDataSource : NSObject
@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *price;
@property (nonatomic, copy) NSString *num;
//....
@end

@protocol ShopCartViewDataSourceDelegate <NSObject>
@optional
- (ShopCartViewDataSource *)shopCartViewDataSource;
@end

//////////////////////////////////回调//////////////////////////////////
@protocol ShopCartViewCallbackDelegate <NSObject>
@optional
- (void)shopViewDidClick;
@end

///////////////////////////////// UI ///////////////////////////////////
@interface ShopCartView : UIView
- (instancetype)initWithFrame:(CGRect)frame
                   dataSource:(id<ShopCartViewDataSourceDelegate>)datasource
                     delegate:(id<ShopCartViewCallbackDelegate>)delegate;
@end
```

.m

```objc
#import "ShopCartView.h"

@implementation ShopCartViewDataSource
@end

@interface ShopCartViewManager : NSObject
// 提供业务处理代码方法
//.................
@end
@implementation ShopCartViewManager
@end

@implementation ShopCartView {
    id<ShopCartViewDataSourceDelegate>          __weak _dataSource;
    id<ShopCartViewCallbackDelegate>            __weak _delegate;
    ShopCartViewManager                         *_manager;
}
- (instancetype)initWithFrame:(CGRect)frame dataSource:(id<ShopCartViewDataSourceDelegate>)datasource delegate:(id<ShopCartViewCallbackDelegate>)delegate {
    if (self = [super initWithFrame:frame]) {
        _dataSource = datasource;
        _delegate = delegate;
        _manager = [[ShopCartViewManager alloc] init];
        [self initSubviews];
    }
    return self;
}

- (void)initSubviews {
    //....
}

- (void)layoutSubviews {
    [super layoutSubviews];
    //....
}

//.....

@end
```

