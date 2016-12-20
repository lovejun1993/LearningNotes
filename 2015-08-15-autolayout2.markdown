---
layout: post
title: "AutoLayout2"
date: 2015-08-15 10:59:59 +0800
comments: true
categories: 
---

###Masonry完成autolayout的cell


###No1. 定义一个所有cell必须实现的协议，协议方法就是让所有cell类实现一个将传入的实体类对象，设置给cell.contentView.subviews

```objc

//完成将传入的实体类对象，设置给cell，以便后续计算cell高度
@protocol ZSYAutoLayoutCellProtocol <NSObject>

@required

- (void)setupDataItem:(id)data;

@end


```

***

###No2. 抽一个BaseCell基类，封装计算cell高度的一些公共方法

> BaseCell.h

```objc

//回传一个cell对象，让调用者设置实体类对象显示数据之后，返回用于计算高度
typedef id (^setupCellBlock)(id<ZSYAutoLayoutCellProtocol> cellToSetup);

@interface ZSYBaseCell : UITableViewCell <ZSYAutoLayoutCellProtocol>

//是否动态更新cell高度
+ (BOOL)isDynamic;

//默认的cell大小
+ (CGSize)defaultCellSize;

//计算cell高度的方法
+ (CGSize)sizeForCellWithDefaultSize:(CGSize)defaultSize setupCellBlock:(setupCellBlock)block;

@end

```

> BaseCell.m封装计算`cell内部约束`得到cell的高度的逻辑代码

```

+ (CGSize)sizeForCellWithDefaultSize:(CGSize)defaultSize setupCellBlock:(setupCellBlock)block {

	
    __block ZSYBaseCell *cell = nil;
   
   //1. 首先查询是否已经有当前要计算高度的【cell类型】的一个对象 
    [cellArray enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
        if ([obj isKindOfClass:[[self class] class]]) {
            cell = obj;
            *stop = YES;
        }
    }];
    
    //2. 如果没有，创建一个【当前cell类型的实例】并保存起来，下次不用再创建，直接使用
    if (!cell) {
        cell = [[[self class] alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:@"XZHAutoLayoutCellIdentifier"];
        cell.frame = CGRectMake(0, 0, defaultSize.width, defaultSize.height);
        [cellArray addObject:cell];
    }
    
    //3. 获取到设置了属性值的cell对象【会执行一次cell的setupData方法】 --》 这里会造成 [cell setupData]会每次计算高度，都会执行一次
    cell = block((id<ZSYAutoLayoutCellProtocol>) cell);
    
    //4. 通知系统计算更新后的约束
    [cell setNeedsLayout];
    [cell layoutIfNeeded];
    
    //5. 使用iOS的api根据cell对象的约束，计算出高度
    CGSize size = [cell.contentView systemLayoutSizeFittingSize:UILayoutFittingCompressedSize];
    
    //6. 为了底线显示
    size.height += 1.0f;
    
    //7. 返回计算处的高度
    return size;
}

```

***

###No3. 使用Masonry完成一个AutoLayout的cell

第一步、init时，创建所有的subviews，并设置初始约束

```objc

- (void)initSubviews {
    
    self.userInteractionEnabled = YES;
    
    self.backgroundColor = [UIColor whiteColor];
    self.accessoryType = UITableViewCellAccessoryNone;
    self.accessoryView = nil;
    self.selectionStyle = UITableViewCellSelectionStyleNone;
    
    _titleLeftImageView = [[UIImageView alloc] init];
    _titleLeftImageView.image = [UIImage imageNamed:@"icon_title"];
    _titleLeftImageView.translatesAutoresizingMaskIntoConstraints = NO;
    _titleLeftImageView.contentMode = UIViewContentModeScaleAspectFill;
    
    _titleLabel = [[UILabel alloc] init];
    _titleLabel.textAlignment = NSTextAlignmentLeft;
    _titleLabel.numberOfLines = 0;
    _titleLabel.lineBreakMode = NSLineBreakByWordWrapping;
    _titleLabel.translatesAutoresizingMaskIntoConstraints = NO;
    _titleLabel.font = [UIFont fontWithName:@"HelveticaNeue" size:15];
    _titleLabel.textColor = [UIColor hex:@"#514647"];
    
    [self.contentView addSubview:_titleLeftImageView];
    [self.contentView addSubview:_titleLabel];
    
    @weakify(self);
    [self.contentView mas_makeConstraints:^(MASConstraintMaker *make) {
        @strongify(self);
        make.edges.equalTo(self);
        make.width.equalTo(self);
    }];
    
    //注意： Label如果高度自动变化多行显示，
    //		  一定要设置这个属性
    [self preferMaxWidthLabels];
    
}

```

第二步、传入实体类对象时，讲数据设置给subviews

```objc

- (void)setupDataItem:(id)data {
    
    if ([data isKindOfClass:[WMModelProduct class]]) {
        
        //实体类对象
        _itemData = (WMModelProduct *)data;
        
        //设置给subviews
        _titleLabel.text = _itemData.name;
        //还有其他subviews...
        
        //让cell更新约束
        [self setNeedsUpdateConstraints];
        [self updateConstraintsIfNeeded];
    }
}

```

***

###哈哈，简单的思路就是这样，如果写复杂的自适应cell，那么也就是如下几点:

* cell内部的约束更复杂
* 为计算cell高度可以做一个缓存逻辑，计算一次高度后就以indexPath保存起来


****

###举个小例子

* 自定义一个Cell，并在init函数创建subviews

```
@implementation MyCell

- (instancetype)initWithStyle:(UITableViewCellStyle)style reuseIdentifier:(NSString *)reuseIdentifier {
    self = [super initWithStyle:style reuseIdentifier:reuseIdentifier];
    if (self) {
        [self createSubviews];
    }
    return self;
}

- (void)createSubviews {
	//1. 添加所有subviews到contentView
	//2. 设置contentView的最开始状态约束
	//3. 陆续设置所有subviews的约束
}

//后续写

@end
```

* createSubviews方法具体代码

```
------- 实例化subviews ------- 

_titleLabel = [[UILabel alloc] init];
_titleLabel.numberOfLines = 1;
_titleLabel.text = @"投资理财";
    
_titleLine = [[UIView alloc] init];
_titleLine.backgroundColor = kZSYVCViewBackgroundColor;
    
_leftUpLabel = [[UILabel alloc] init];
_leftUpLabel.numberOfLines = 1;
    
_leftLowLabel = [[UILabel alloc] init];
_leftLowLabel.numberOfLines = 1;
_leftLowLabel.text = @"预期年化";
    
_rightTitleLabel = [[UILabel alloc] init];
_rightTitleLabel.numberOfLines = 1;
    
_rightDaysLabel = [[UILabel alloc] init];
_rightDaysLabel.numberOfLines = 1;
    
_rightMoneyLabel = [[UILabel alloc] init];
_rightMoneyLabel.numberOfLines = 1;
    
_arrow = [[UIImageView alloc] init];
_arrow.image = [UIImage imageNamed:@"icon箭头"];
    
[self.contentView addSubview:_titleLabel];
[self.contentView addSubview:_titleLine];
[self.contentView addSubview:_leftUpLabel];
[self.contentView addSubview:_leftLowLabel];
[self.contentView addSubview:_rightTitleLabel];
[self.contentView addSubview:_rightDaysLabel];
[self.contentView addSubview:_rightMoneyLabel];
[self.contentView addSubview:_arrow];

------- 开始设置约束 ------- 

@weakify(self);
    
//1. 先设置cell.contentView初始状态约束    

[self.contentView mas_makeConstraints:^(MASConstraintMaker *make) {
    @strongify(self);
    make.width.equalTo(self);
}];

//2. 再开始设置其他subviews的约束

[_titleLabel mas_makeConstraints:^(MASConstraintMaker *make) {
        @strongify(self);
        make.centerX.mas_equalTo(self.contentView.mas_centerX);
        make.top.mas_equalTo(self.contentView.mas_top).offset(18);
    }];
    
[_titleLine mas_makeConstraints:^(MASConstraintMaker *make) {
    @strongify(self);
    make.top.mas_equalTo(self.titleLabel.mas_bottom).offset(18);
    make.centerX.mas_equalTo(self.contentView.mas_centerX);
    make.width.mas_equalTo(self.contentView.mas_width);
    make.height.mas_equalTo(1);
}];

[_leftUpLabel mas_makeConstraints:^(MASConstraintMaker *make) {
    @strongify(self);
    make.left.mas_equalTo(self.contentView.mas_left).offset(30);
    make.top.mas_equalTo(self.titleLabel.mas_bottom).offset(43);
}];

[_leftLowLabel mas_makeConstraints:^(MASConstraintMaker *make) {
    @strongify(self);
    make.left.mas_equalTo(self.leftUpLabel.mas_left);
    make.width.mas_equalTo(self.leftUpLabel.mas_width);
    make.top.mas_equalTo(self.leftUpLabel.mas_bottom).offset(10);
}];
    
__block UIView *line = [[UIView alloc] init];
line.backgroundColor = kZSYVCViewBackgroundColor;
[self.contentView addSubview:line];
    
[line mas_makeConstraints:^(MASConstraintMaker *make) {
    @strongify(self);
    make.top.mas_equalTo(self.leftUpLabel.mas_top);
    make.bottom.mas_equalTo(self.leftLowLabel.mas_bottom);
    make.width.mas_equalTo(0.5);
    make.left.mas_equalTo(self.leftUpLabel.mas_right).offset(18);
}];

[_rightTitleLabel mas_makeConstraints:^(MASConstraintMaker *make) {
    @strongify(self);
    make.top.mas_equalTo(self.leftUpLabel.mas_top);
    make.left.mas_equalTo(line.mas_right).offset(18);
}];
    
[_rightDaysLabel mas_makeConstraints:^(MASConstraintMaker *make) {
    @strongify(self);
    make.top.mas_equalTo(self.rightTitleLabel.mas_bottom).offset(5);
    make.left.mas_equalTo(line.mas_right).offset(18);
}];
    
[_rightMoneyLabel mas_makeConstraints:^(MASConstraintMaker *make) {
    @strongify(self);
    make.top.mas_equalTo(self.rightDaysLabel.mas_bottom).offset(23);
    make.bottom.mas_equalTo(self.leftLowLabel.mas_bottom);
    make.left.mas_equalTo(line.mas_right).offset(18);
}];
    
[_arrow mas_makeConstraints:^(MASConstraintMaker *make) {
    @strongify(self);
    make.right.mas_equalTo(self.contentView.mas_right).offset(-16);
    make.centerY.mas_equalTo(line.mas_centerY);
}];

```

* 设置完约束之后，最后修改cell.conetentView的`底部约束`确定cell的`总高度`

```
- (void)updateConstraints {
    
    @weakify(self);
    [self.contentView mas_makeConstraints:^(MASConstraintMaker *make) {
        @strongify(self);
        
        //将subvsiews中，布局在最下方的的bottom赋值给cell的bottom，来确定cell的高度
        make.bottom.mas_equalTo(self.rightMoneyLabel.mas_bottom).offset(25);
    }];
    
    [super updateConstraints];
}
```

* 图示效果

![](http://i13.tietuku.com/dd68dbbda39d6954.png)

* 小结代码模板

```
@implementation ZSYHomepageCellForXiaofei

- (instancetype)initWithStyle:(UITableViewCellStyle)style reuseIdentifier:(NSString *)reuseIdentifier {
    self = [super initWithStyle:style reuseIdentifier:reuseIdentifier];
    if (self) {
        [self createSubviews];
    }
    return self;
}

- (void)createSubviews {
    //1. 添加所有subviews到contentView
    //...
    
    //2. 设置contentView的最开始状态约束
    [self.contentView mas_makeConstraints:^(MASConstraintMaker *make) {
        @strongify(self);
        make.width.equalTo(self);
    }];
    
    //3. 陆续设置所有subviews的约束
    //...
}

- (void)updateConstraints {
    
    @weakify(self);
    [self.contentView mas_makeConstraints:^(MASConstraintMaker *make) {
        @strongify(self);
        make.bottom.mas_equalTo(self.rightMoneyLabel.mas_bottom).offset(25);
    }];
    
    [super updateConstraints];
}

- (void)setupDataItem:(id)data {
    //将实体对象的属性值赋值给cell.contentView.sunbviews的属性
}

@end
```