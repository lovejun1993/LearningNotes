---
layout: post
title: "UITableViewCell一些优化"
date: 2014-06-03 17:52:56 +0800
comments: true
categories: 
---

###编写一个自定义UITableViewCell的基本步骤

- 新建一个UITableViewCell子类

- 重写`initWithStyle:`方法
	- 创建所有的UI Subviews实例
	- 将所有的subviews实例添加到`UITableViewCell.contentView`
	- 对一些subview设置基础属性（字体大小、字体名、字体颜色、背景色..）

- 提供两个模型
	- 模型1 >>> `实体类`:
		- 存储Cell上所有subviews显示需要的数据，来决定某些subview是显示or隐藏
	- 模型2 >>> `Frame`:
		- 存储Cell显示的frame大小，主要是高度

- UITableViewDelegate的`高度`回调函数返回cell的高度时
	- 返回Cell位置对应的`Frame模型`记录的`高度`
	- 做了一个Cell的`高度缓存`

****

###先记录一个简单的Cell编写步骤

首先是先写一个Base基类Cell，提供一些给子类Cell重写的模板方法.

```objc
#import <UIKit/UIKit.h>

@interface BaseCell : UITableViewCell

/**
 *  传入TableView查询一个cell对象，该identifier由各自Cell类自己管理，不向外暴露
 */
+ (instancetype)dequeFromTableView:(UITableView *)tableView;

/**
 *  传入实体类对象，设置subviews的数据显示
 */
- (void)setupData:(id)data;

/**
 *  初始化subviews
 */
- (void)setupSubviews;

@end
```

```
#import "BaseCell.h"

@implementation BaseCell

//子类实现
+ (instancetype)dequeFromTableView:(UITableView *)tableView {return nil;}

//子类实现
- (void)setupData:(id)data {}

//子类实现
- (void)setupSubviews {}

- (instancetype)initWithStyle:(UITableViewCellStyle)style reuseIdentifier:(NSString *)reuseIdentifier {
    if (self = [super initWithStyle:style reuseIdentifier:reuseIdentifier]) {
        [self setupSubviews];
    }
    return self;
}

@end
```

然后看写一个子类Cell.

```objc
#import "BaseCell.h"

@interface NewsCell : BaseCell

@end
```

```objc
#import "NewsCell.h"

//导入cell依赖的实体类
#import "News.h"

@interface NewsCell ()

@property (nonatomic, strong) UILabel *titleLabel;
@property (nonatomic, strong) UILabel *contentLabel;

@end

@implementation NewsCell

+ (instancetype)dequeFromTableView:(UITableView *)tableView {
    NewsCell *cell = [tableView dequeueReusableCellWithIdentifier:@"mycell"];
    if (!cell) {
        cell = [[NewsCell alloc] initWithStyle:UITableViewCellStyleValue2 reuseIdentifier:@"mycell"];
    }
    return cell;
}

- (void)setupSubviews {
    _titleLabel = [UILabel new];
    _contentLabel = [UILabel new];
    
    _titleLabel.font = [UIFont systemFontOfSize:16.0f];
    _contentLabel.font = [UIFont systemFontOfSize:16.0f];
    
    [self.contentView addSubview:_titleLabel];
    [self.contentView addSubview:_contentLabel];
}

- (void)setupData:(id)data {
    //1.
    if ([data isEqual:[NSNull null]]) return;
    if (![data isKindOfClass:[News class]]) return;
    
    //2.
    News *news = (News *)data;
    
    //3.
    _titleLabel.text = news.title;
    _contentLabel.text = news.content;
    
    //4. 通知重新计算cell内部所有subview的frame
    [self setNeedsLayout];
}

#pragma mark - 重用时清除之前的数据

- (void)prepareForReuse {
    [super prepareForReuse];
    
    _titleLabel.text = @"";
    _contentLabel.text = @"";
}

#pragma mark - frame布局

- (void)layoutSubviews {
    [super layoutSubviews];
    
    /* 下面对所有的subviews进行frame布局 */
    
    //titleLabel的frame
    CGFloat x = 10;
    CGFloat y = 10;
    CGFloat w = CGRectGetWidth(self.frame);
    CGFloat h = CGRectGetHeight(self.frame) * 0.3;
    _titleLabel.frame = CGRectMake(x, y, w, h);

    //contentLabel的frame
    x = 10;
    y = CGRectGetMaxY(_titleLabel.frame) + 10;
    w = CGRectGetWidth(self.frame);
    h = CGRectGetHeight(self.frame) * (1 - 0.3) - 10 * 2;
    _contentLabel.frame = CGRectMake(x, y, w, h);
}

@end
```

上面的例子比较简单的，是需要Cell自己内部计算所有subviews的frame。下面给出一种比较好的优化思路，将计算cell内部所有subviews的frame代码单独使用一个类对象来封装，这样做的好处:

- Cell自己不再计算内部subviews的frame
- 不用Cell自己每次调用setNeedLayout重新计算设置subviews的frame
- 由外部对象计算frame，并缓存起来，下一次直接传入对应实体对象的frame模型即可，避免多次重复计算subviews的frame 

***

###再看一个使用FrameModel来缓存Cell高度的例子

Cell依赖的实体类Statuses

```
#import <Foundation/Foundation.h>

//依赖User实体类
#import "User.h"

@interface Statuses : NSObject

@property (nonatomic, copy) NSString *idstr;

@property (nonatomic, strong) NSNumber *attitudes_count;
@property (nonatomic, strong) NSNumber *biz_feature;
@property (nonatomic, strong) NSNumber *comments_count;
@property (nonatomic, strong) NSNumber *favorited;
@property (nonatomic, strong) NSNumber *reposts_count;

@property (nonatomic, copy) NSString *created_at;
@property (nonatomic, copy) NSString *text;
@property (nonatomic, copy) NSString *source;

@property (nonatomic, copy) NSString *thumbnail_pic;
@property (nonatomic, copy) NSString *bmiddle_pic;
@property (nonatomic, copy) NSString *original_pic;

@property (nonatomic, strong) User *user;

//来自的转发微博
@property (nonatomic, strong) Statuses *retweeted_status;

//配图数组
@property (nonatomic, strong) NSArray *pic_urls;

/**
 *  拼接得到转发微博的内容
 */
- (NSString *)retweetContent;

@end
```

```
#import <Foundation/Foundation.h>

@interface User : NSObject

@property (nonatomic, copy) NSString *idstr;
@property (nonatomic, strong) NSNumber *mbtype;
@property (nonatomic, strong) NSNumber *mbrank;
@property (nonatomic, copy) NSString *screen_name;
@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *location;
@property (nonatomic, copy) NSString *url;
@property (nonatomic, copy) NSString *profile_image_url;
@property (nonatomic, copy) NSString *cover_image;
@property (nonatomic, copy) NSString *cover_image_phone;
@property (nonatomic, copy) NSString *profile_url;
@property (nonatomic, copy) NSString *domain;
@property (nonatomic, strong) NSNumber *followers_count;
@property (nonatomic, strong) NSNumber *friends_count;
@property (nonatomic, strong) NSNumber *favourites_count;
@property (nonatomic, copy) NSString *created_at;
@property (nonatomic, copy) NSString *avatar_large;
@property (nonatomic, copy) NSString *avatar_hd;
@property (nonatomic, copy) NSString *verified_reason;

//添加一个vip变量，并且提供一个get方法叫做isVip
@property (nonatomic, assign, getter = isVip) BOOL vip;

@end
```

创建一个FrameModel实体类专门负责计算一个`实体类Statuses的对象`显示在`一个Cell内部所有subviews`显示的frame，以及Cell的总高度.


```
#import <Foundation/Foundation.h>
#import <UIKit/UIKit.h>

#import "Statuses.h"
#define padding 10
#define NameFont  [UIFont systemFontOfSize:13.f]
#define TimeFont  [UIFont systemFontOfSize:12.f]
#define ContentFont  [UIFont systemFontOfSize:14.f]

/**
 *  这个模型类的职责:
 *
 *    1. 承载实体类模型Statuses实例
 *    2. 计算负责显示Statuses实例对应的TableViewCell需要的frame
 *          2.1 TableViewCell内部所有subviews显示的frame
 *          2.2 TableViewCell自己显示的总高度
 *    3. 所有的frame基于`[UIScreen mainScreen].bounds.size`计算
 *
 */
@interface StatusFrameModel : NSObject

//实体类模型
@property (nonatomic, strong) Statuses *status;

//对应subview显示数据的frame
@property (nonatomic, assign) CGRect iconImageViewFrame;
@property (nonatomic, assign) CGRect vipImageViewFrame;
@property (nonatomic, assign) CGRect photoImageViewFrame;
@property (nonatomic, assign) CGRect nameLabelFrame;
@property (nonatomic, assign) CGRect timeLabelFrame;
@property (nonatomic, assign) CGRect sourceLabelFrame;
@property (nonatomic, assign) CGRect contentLabelFrame;
@property (nonatomic, assign) CGRect contentContainerFrame;

@property (nonatomic, assign) CGRect retweetContainerFrame;
@property (nonatomic, assign) CGRect retweetContentLabelFrame;
@property (nonatomic, assign) CGRect retweetImageViewFrame;

@property (nonatomic, assign) CGRect bottomContainerFrame;

//最终cell的高度
@property (nonatomic, assign) CGFloat cellHeight;

@end
```

```
#import "StatusFrameModel.h"
#import "NSString+Extension.h"

@implementation StatusFrameModel

/**
 *  重写setter实体类对象方法，计算实体类对象在cell上显示的所有frame以及总高度
 */
- (void)setStatus:(Statuses *)status {

    _status = status;
    
    /** 
     * 计算所有数据项显示在cell上时对应的frame 
     *
     *  规则1. 从上到下，累计计算
     *  规则2. 从左到右，累计计算
     */
     
     User *user = status.user;
    
    //头像frame
    CGFloat iconWH = 50;
    CGFloat iconX = padding;
    CGFloat iconY = padding;
    _iconImageViewFrame = CGRectMake(iconX, iconY, iconWH, iconWH);
    
    //昵称
    CGFloat nameX = CGRectGetMaxX(_iconImageViewFrame) + padding;
    CGFloat nameY = iconY;
    CGSize nameSize = [user.name sizeWithfont:NameFont];
    _nameLabelFrame = CGRectMake(nameX, nameY, nameSize.width, nameSize.height);

    
    //Vip图像
    if ([user isVip]) {//如果是会员才设置Vip图像
        CGFloat vipX = CGRectGetMaxX(_nameLabelFrame) + padding;
        CGFloat vipY = nameY;
        CGFloat vipW = 14;
        CGFloat vipH = nameSize.height;
        _vipImageViewFrame = CGRectMake(vipX, vipY , vipW, vipH);
    }
    
    //时间
    CGFloat timeX = nameX;
    CGFloat timeY = CGRectGetMaxY(_nameLabelFrame) + padding;
    CGSize timeSize = [status.created_at sizeWithfont:TimeFont];
    _timeLabelFrame = CGRectMake(timeX, timeY, timeSize.width, timeSize.height);
    
    //来源
    CGFloat sourX = CGRectGetMaxX(_timeLabelFrame) + padding;
    CGFloat sourY = timeY;
    CGSize sourSize = [status.created_at sizeWithfont:TimeFont];
    _sourceLabelFrame = CGRectMake(sourX, sourY, sourSize.width, sourSize.height);
    
    //正文
    CGFloat contentX = iconX;
    CGFloat contentY = MAX(CGRectGetMaxY(_timeLabelFrame), CGRectGetMaxY(_iconImageViewFrame)) + padding;
    CGFloat contentW = [UIScreen mainScreen].bounds.size.width - 2 * contentX;
    CGFloat contentH = [status.text sizeWithfont:ContentFont MaxWidth:contentW].height;
    _contentLabelFrame = CGRectMake(contentX, contentY, contentW, contentH);
    
    //配图
    BOOL isHasPhotos = status.pic_urls.count > 0;
    
    if (isHasPhotos) {
        CGFloat photoX = iconX;
        CGFloat photoY = CGRectGetMaxY(_contentLabelFrame) + padding;
        CGSize photoSize = [WeiboPhotoView photoViewSizeWithPhotoCount:status.pic_urls.count];
        
        _photoImageViewFrame = CGRectMake(photoX, photoY, photoSize.width, photoSize.height);
        
    } else {
        
        //如果实体类数据没有这个subview显示的数据项
        _photoImageViewFrame = CGRectZero;
    }
    
    //顶部和中间的容器
    CGFloat containerMiddleX = 0;
    CGFloat containerMiddleY = 0;
    CGFloat containerMiddleW = [UIScreen mainScreen].bounds.size.width;
    CGFloat containerMiddleH;
    
    if (!isHasPhotos) {
        containerMiddleH = CGRectGetMaxY(_contentLabelFrame) + padding;
    } else {
        containerMiddleH = CGRectGetMaxY(_photoImageViewFrame) + padding;
    }
    
    _contentContainerFrame = CGRectMake(containerMiddleX, containerMiddleY, containerMiddleW, containerMiddleH);
   
    //转发微博
    BOOL isHasRetweet = (status.retweeted_status != nil);
    
    if (isHasRetweet) {//存在转发微博
        
        //转发微博正文
        CGFloat retweetContentX = padding;
        CGFloat retweetContentY = padding;
        CGFloat retweetContentW = [UIScreen mainScreen].bounds.size.width - 2 * padding;
        CGFloat retweetContentH = [[status retweetContent] sizeWithfont:ContentFont MaxWidth:retweetContentW].height;
        
        _retweetContentLabelFrame = CGRectMake(retweetContentX, retweetContentY, retweetContentW, retweetContentH);
        
        //转发微博图片
        BOOL isHasRetweetImage = (status.retweeted_status.pic_urls.count > 0);
        
        if (isHasRetweetImage) {//有微博配图
            CGFloat retweetImageX = retweetContentX;
            CGFloat retweetImageY = CGRectGetMaxY(_retweetContentLabelFrame) + padding;
            CGSize retweetImageSize = [WeiboPhotoView photoViewSizeWithPhotoCount:status.retweeted_status.pic_urls.count];
            
            _retweetImageViewFrame = CGRectMake(retweetImageX, retweetImageY, retweetImageSize.width, retweetImageSize.height);
        } else {

            //如果实体类数据没有这个subview显示的数据项
            _retweetImageViewFrame = CGRectZero;
        }
        
        //转发微博容器
        CGFloat retweetContainerX = 0;
        CGFloat retweetContainerW = [UIScreen mainScreen].bounds.size.width;
        CGFloat retweetContainerY = CGRectGetMaxY(_contentContainerFrame);
        CGFloat retweetContainerH;
        
        if (isHasRetweetImage) {
            retweetContainerH = CGRectGetMaxY(_retweetImageViewFrame) + padding;
        } else {
            retweetContainerH = CGRectGetMaxY(_retweetContentLabelFrame) + padding;
        }
        
        _retweetContainerFrame = CGRectMake(retweetContainerX, retweetContainerY, retweetContainerW, retweetContainerH);
    } else {
        
        //如果实体类数据没有这个subview显示的数据项
        _retweetContainerFrame = CGRectZero;
    }
    
    //底部按钮容器
    CGFloat bootmContainerX = 0;
    CGFloat bootmContainerY;
    CGFloat bootmContainerW = [UIScreen mainScreen].bounds.size.width;;
    CGFloat bootmContainerH = 44;
    
    if (isHasRetweet) {//有转发微博
        bootmContainerY = CGRectGetMaxY(_retweetContainerFrame) + padding;
    } else {//没有转发微博
        bootmContainerY = CGRectGetMaxY(_contentContainerFrame) + padding;
    }
    
    _bottomContainerFrame = CGRectMake(bootmContainerX, bootmContainerY, bootmContainerW, bootmContainerH);
    
    //最终Cell高度
    _cellHeight = CGRectGetMaxY(_bottomContainerFrame) + padding;
}

@end
```

下面是Cell类

```
#import "StatusFrameModel.h"

/**
 *  微博Cell
 */
@interface WBTableViewCell : BaseTableViewCell

@end
```

```
@interface WBTableViewCell ()

//-----------------------原创微博-----------------------
@property (nonatomic, strong) UIView *contentContainer;
@property (nonatomic, strong) UIImageView *iconImageView;//头像
@property (nonatomic, strong) UIImageView *vipImageView;//VIP图像
@property (nonatomic, strong) UIImageView *photoImageView;//配图
@property (nonatomic, strong) UILabel *nameLabel;//昵称
@property (nonatomic, strong) UILabel *timeLabel;//时间
@property (nonatomic, strong) UILabel *sourceLabel;//来源
@property (nonatomic, strong) UILabel *contentLabel;//正文

//-----------------------转发微博-----------------------
@property (nonatomic, strong) UIView *retweetContainer;
@property (nonatomic, strong) UILabel *retweetContentLabel;
@property (nonatomic, strong) UIImageView *retweetImageView;

//-----------------------底部按钮-----------------------
@property (nonatomic, strong) UIView *bottomContainer;
//@property (nonatomic, strong) UIView *bottomContainer;
//@property (nonatomic, strong) UIView *bottomContainer;
@end
```

```
@implementation WBTableViewCell

+ (NSString *)identifier {
    return NSStringFromClass([WBTableViewCell class]);
}

+ (instancetype)instanceWithTableView:(UITableView *)tv {
    WBTableViewCell *cell = [tv dequeueReusableCellWithIdentifier:[self identifier]];
    if (!cell) {
        cell = [[WBTableViewCell alloc] initWithStyle:UITableViewCellStyleSubtitle reuseIdentifier:[self identifier]];
    }
    return cell;
}

- (void)dealloc {
    
    _iconImageView = nil;
    _vipImageView = nil;
    _photoImageView = nil;
    
    _nameLabel = nil;
    _timeLabel = nil;
    _sourceLabel = nil;
    _contentLabel = nil;
}


- (instancetype)initWithStyle:(UITableViewCellStyle)style
              reuseIdentifier:(NSString *)reuseIdentifier
{
    self = [super initWithStyle:style reuseIdentifier:reuseIdentifier];
    if (self) {
        [self initSubviews];
    }
    return self;
}

#pragma mark  添加所有subviews、以及设置

- (void)initSubviews {
    
    //1. 让cell背景色透明，显示tableView的背景颜色
    self.backgroundColor = [UIColor clearColor];
    
    //2. 初始化原创微博subviews
    [self initOriginWeiboSubviews];
    
    //3. 初始化转发微博subviews
    [self initRetweetWeiboSubviews];
    
    //4. 底部按钮
    [self initBottomWeiboSubviews];
}

- (void)initOriginWeiboSubviews {
    
    //1. 初始化subviews
    _contentContainer = [[UIView alloc] init];
    _contentContainer.backgroundColor = [UIColor whiteColor];
    
    _iconImageView = [[UIImageView alloc] init];
    _vipImageView = [[UIImageView alloc] init];
    _photoImageView = [[UIImageView alloc] init];
    
    _nameLabel = [[UILabel alloc] init];
    _timeLabel = [[UILabel alloc] init];
    _contentLabel = [[UILabel alloc] init];
    _sourceLabel = [[UILabel alloc] init];
    
    
    //2. subview基础设置
    _iconImageView.contentMode = UIViewContentModeCenter;
    _vipImageView.contentMode = UIViewContentModeCenter;
    _photoImageView.contentMode = UIViewContentModeCenter;
    
    _nameLabel.font = NameFont;
    _timeLabel.font = TimeFont;
    _sourceLabel.font = TimeFont;
    
    _contentLabel.font = ContentFont;
    //自动换行Label
    _contentLabel.numberOfLines = 0;
    //最大显示宽大Label
    _contentLabel.preferredMaxLayoutWidth = [UIScreen mainScreen].bounds.size.width - 2 * padding;
    
    
    //3. 添加subviews
    
    [_contentContainer addSubview:_iconImageView];
    [_contentContainer addSubview:_vipImageView];
    [_contentContainer addSubview:_photoImageView];
    [_contentContainer addSubview:_nameLabel];
    [_contentContainer addSubview:_timeLabel];
    [_contentContainer addSubview:_contentLabel];
    [_contentContainer addSubview:_sourceLabel];
    
    [self.contentView addSubview:_contentContainer];
}

- (void)initRetweetWeiboSubviews {
    
    _retweetContainer = [[UIView alloc] init];
    _retweetContainer.backgroundColor = [UIColor orangeColor];
    [self.contentView addSubview:_retweetContainer];
    
    _retweetContentLabel = [[UILabel alloc] init];
    _retweetContentLabel.font = ContentFont;
    _retweetContentLabel.numberOfLines = 0;
    _retweetContentLabel.preferredMaxLayoutWidth = [UIScreen mainScreen].bounds.size.width - 2 * padding;
    [_retweetContainer addSubview:_retweetContentLabel];
    
    _retweetImageView = [[UIImageView alloc] init];
    [_retweetContainer addSubview:_retweetImageView];
}

- (void)initBottomWeiboSubviews {
    
    _bottomContainer = [[UIView alloc] init];
    [self.contentView addSubview:_bottomContainer];
    _bottomContainer.backgroundColor = [UIColor blueColor];
}

#pragma mark  实体对象赋值协议方法实现

- (void)setupData:(id)data
{
    //传入的是FrameModel
    if ([data isMemberOfClass:[StatusFrameModel class]])
    {
        
        //1.转换成 FrameModel
        StatusFrameModel *frameModel = (StatusFrameModel *)data;
        
        //2. 完成subviews显示的实体对象数据
        //如果对应数据项没有，就隐藏对应的subviews
        [self setupSubviewsDatas:frameModel.status];
        
        //3. 传入cell内部所有subviews显示的frame
        //不显示的subview设置frame为CGRectZero
        [self setupSubviewsFrame:frameModel];
    }
    
}

/**
 * 主要完成给subviews设置显示的数据，
 * 全部subview都设置frame，不区分有没有数据
 * 因为FrameModel中将没有数据项的subview的frame设置为CGRectZero，没有显示的大小
 */
- (void)setupSubviewsDatas:(Statuses *)status {
    
    //1. 设置头像数据
    [_iconImageView sd_setImageWithURL:[NSURL URLWithString:status.user.profile_image_url] placeholderImage:[UIImage imageNamed:@"avatar_default"] options:SDWebImageProgressiveDownload];
    
    //3. 昵称显示
    _nameLabel.text = status.user.name;
    
    //4. 会员等级显示
    if (status.user.isVip) {
        
        _vipImageView.image = [UIImage imageNamed:@"common_icon_membership"];
        
        //显示会员等级
        _vipImageView.hidden = NO;
        
        //修改名字颜色
        _nameLabel.textColor = [UIColor orangeColor];
        
    } else {
        
        //隐藏会员等级
        _vipImageView.hidden = YES;
        
        //修改名字颜色
        _nameLabel.textColor = [UIColor blackColor];
    }
    
    //5. 配图
    _photoImageView.backgroundColor = [UIColor redColor];
    
    if (status.pic_urls.count > 0)
    {
        _photoImageView.hidden = NO;
        
    } else {
        
        _photoImageView.hidden = YES;
    }
    
    //6. 时间
    _timeLabel.text = status.created_at;
    
    //7.正文
    _contentLabel.text = status.text;
    
    //8. 来源
    _sourceLabel.text = status.source;
    _sourceLabel.textColor = [UIColor lightGrayColor];
    
    //9. 转发微博
    if (status.retweeted_status) {//有转发微博
        
        //9.1 转发微博容器
        _retweetContainer.hidden = NO;
        
        //9.2 转发内容
        _retweetContentLabel.text = [status retweetContent];
        
        //9.3 转发配图
        if (status.retweeted_status.pic_urls) {//转发微博有配图
            _retweetImageView.hidden = NO;
            _retweetImageView.backgroundColor = [UIColor yellowColor];
            
        } else {//转发微博无配图
            _retweetImageView.hidden = YES;
        }
        
    } else {//无转发微博
        _retweetContainer.hidden = YES;
    }
}

/**
 *  主要完成给subview设置显示的frame
 */
- (void)setupSubviewsFrame:(StatusFrameModel *)frameModel
{
    
    //1.
    _contentContainer.frame = frameModel.contentContainerFrame;
    
    //2.
    _iconImageView.frame = frameModel.iconImageViewFrame;
    
    //3.
    _nameLabel.frame = frameModel.nameLabelFrame;
    
    //4.
    _vipImageView.frame = frameModel.vipImageViewFrame;
    
    //5.
    _photoImageView.frame = frameModel.photoImageViewFrame;
    
    //6.
    _timeLabel.frame = frameModel.timeLabelFrame;
    
    //7.
    _contentLabel.frame = frameModel.contentLabelFrame;
    
    //8.
    _sourceLabel.frame = frameModel.sourceLabelFrame;
    
    //9.
    _retweetContainer.frame = frameModel.retweetContainerFrame;
    _retweetContentLabel.frame = frameModel.retweetContentLabelFrame;
    _retweetImageView.frame = frameModel.retweetImageViewFrame;
    
    //10.
    _bottomContainer.frame = frameModel.bottomContainerFrame;
}


@end
```

cell不用每次对传入的实体对象再自己重新计算frame了，直接使用外部传入的FraneModel对象保存的所有的subviews对应的frame.

> 有一个开源的Cell高度缓存框架，有时间去看看...

***

###使用AutoLayout形式的cell高度计算

首先定义一个用于计算Cell内部约束得到最后高度的协议， ZSYAutoLayoutCellProtocol.h

```objc
#import <Foundation/Foundation.h>

@protocol ZSYAutoLayoutCellProtocol;


static CGFloat ZSYDynamicTableViewCellAccessoryWidth = 33;

static NSMutableArray *cellArray;

typedef id (^setupCellBlock)(id<ZSYAutoLayoutCellProtocol> cellToSetup);

@protocol ZSYAutoLayoutCellProtocol <NSObject>

- (void)setupDataItem:(id)data;

+ (CGSize)sizeForCellWithDefaultSize:(CGSize)defaultSize setupCellBlock:(setupCellBlock)block;

@end
```

然后BaseCell实现上面的协议方法，用于计算约束最后的cell高度

```objc
#import <UIKit/UIKit.h>

@interface BaseCell : UITableViewCell

/**
 *  传入TableView查询一个cell对象，该identifier由各自Cell类自己管理，不向外暴露
 */
+ (instancetype)dequeFromTableView:(UITableView *)tableView;

/**
 *  传入实体类对象，设置subviews的数据显示
 */
- (void)setupData:(id)data;

/**
 *  初始化subviews
 */
- (void)setupSubviews;

@end
```

```
#import "BaseCell.h"

@implementation BaseCell

//子类实现
+ (instancetype)dequeFromTableView:(UITableView *)tableView {return nil;}

//子类实现
- (void)setupData:(id)data {}

//子类实现
- (void)setupSubviews {}

- (instancetype)initWithStyle:(UITableViewCellStyle)style reuseIdentifier:(NSString *)reuseIdentifier {
    if (self = [super initWithStyle:style reuseIdentifier:reuseIdentifier]) {
        [self setupSubviews];
    }
    return self;
}

//根据Cell内部设置的约束，计算得到cell的总高度
+ (CGSize)sizeForCellWithDefaultSize:(CGSize)defaultSize setupCellBlock:(setupCellBlock)block {
    __block ZSYBaseCell *cell = nil;
    
    //使用一个static数组，给每一类型的Cell保存一个对象，用来计算高度的
    [cellArray enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
        if ([obj isKindOfClass:[[self class] class]]) {
            cell = obj;
            *stop = YES;
        }
    }];
    
    //如果数组还不存在，就创建一个cell，并保存到数组中，以后直接使用，不会再创建
    if (!cell) {
        cell = [[[self class] alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:@"XZHAutoLayoutCellIdentifier"];
        cell.frame = CGRectMake(0, 0, defaultSize.width, defaultSize.height);
        [cellArray addObject:cell];
    }
    
    //回传用于计算约束的cell对象，返回给TableView，让TableView给这个回传的cell对象填充所有的显示数据
    cell = block((id<ZSYAutoLayoutCellProtocol>) cell);
    
    //通知系统重新计算frame或约束
    [cell setNeedsLayout];
    [cell setNeedsUpdateConstraints];
    [cell layoutIfNeeded];
    
    //通过苹果提供的api直接计算约束，得到cell的高度
    //注意，必须保证cell内部的所有subviews的约束设置正确
    CGSize size = [cell.contentView systemLayoutSizeFittingSize:UILayoutFittingCompressedSize];
    size.height += 1.0f;
    
    return size;
}

@end
```

如上可以看到，设置实体对象的数据之后，需要通知系统重新计算约束

```
[cell setNeedsLayout];
[cell setNeedsUpdateConstraints];
[cell layoutIfNeeded];
```

所以，每一次数据改变，需要重新计算一遍cell内部所有约束对应的frame，相比前面直接使用FrameModel可能性能稍微差一点，但是还是简单多了。所以如果对性能要求不高，要求简单的还是使用AutoLayout吧.

***

###Cell之间做一些间隙

- 第一步，设置TableView的backgroudColor 或 backgroudView

- 第二步、重写自定义cell的setFrame:方法，缩小cell的frame

```
- (void)setFrame:(CGRect)frame {
	
	//1. 对frame做出修改
	frame.x += 10;
	frame.w -= 10 * 2;
	
	//2. 再将修改的frame交给父类执行
	[super setFrame:frame]
}
```