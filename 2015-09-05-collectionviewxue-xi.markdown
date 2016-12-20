---
layout: post
title: "CollectionView学习"
date: 2015-09-05 13:40:51 +0800
comments: true
categories: 
---

###TableView用的挺多，CollectionView一直没怎么用过，还是学习一下...

***

###首先CollectionView基本设置dataSource和delegate和TableView差不多的。

###不同的地方是，CollectionView最重要的一个东西就是UICollectionViewLayout。

###CollectionView完成布局内部的CollectionViewCell，就是通过读取这个Layout之后，才知道让没一个CollectionViewCell具体显示多大、怎么滚动、什么位置...

***

###使用iOS自带的`UICollectionViewLayout`类.

- 使用`scrollDirection`属性，设置滚动方向
	- 横向滚动:UICollectionViewScrollDirectionHorizontal
	- 竖向滚动:UICollectionViewScrollDirectionVertical


- 使用`itemSize`属性，设置collectionViewCell的尺寸
	- eg、layout.itemSize = [UIScreen mainScreen].bounds.size;

- 使用`minimumLineSpacing`属性，设置每一行或每一列之间的间距 

	- 当collectionView`纵向`滚动时，设置`行`间距

	![](http://i5.tietuku.com/cf7c478e0a53d769.png)
		
	- 当collectionView`横向`滚动时，设置`列`间距

	![](http://i5.tietuku.com/c533d1df40fcb0cf.png)

- 使用`minimumInteritemSpacing`属性，设置cell之间的间距
	
***

###例子、完成一个Segment点击后，切换CollectionView对象的colletionViewLayout，改变显示cell的布局

####完成如下效果图的例子

- 点击后，显示小cell布局

![](http://i5.tietuku.com/6f2f9361a9a38124.png)

- 点击后，显示大cell布局

![](http://i5.tietuku.com/232fa221ecab3a95.png)

###首先在`AFViewController`完成如下

- 匿名分类

```
@interface AFViewController ()

/** 显示小cell布局的layout */
@property (nonatomic, strong) AFCollectionViewFlowLargeLayout *largeLayout;

/** 显示大cell布局的layout */
@property (nonatomic, strong) AFCollectionViewFlowSmallLayout *smallLayout;


@property (nonatomic, strong) NSArray *images;

@end

static NSString *ItemIdentifier = @"ItemIdentifier";
```

- loadView方法，创建UICollectionView实例

```
-(void)loadView
{
	
	//1. 实例化小cell布局的自定义layout
	self.smallLayout = [[AFCollectionViewFlowSmallLayout alloc] init];
	
	//2. 实例化大cell布局的自定义layout
	self.largeLayout = [[AFCollectionViewFlowLargeLayout alloc] init];
	    
	//3. 创建UICollectionView实例，初始化使用 self.smallLayout布局cell
	self.collectionView = [[UICollectionView alloc] initWithFrame:CGRectZero collectionViewLayout:self.smallLayout];
	
	//4. 给UICollectionView实例，注册CollectionViewCell类型
	[self.collectionView registerClass:[AFCollectionViewCell class] forCellWithReuseIdentifier:ItemIdentifier];
	
	//5. 设置UICollectionView实例的数据源
	self.collectionView.delegate = self;
	
	//6. UICollectionView实例的回调代理实现对象
	self.collectionView.dataSource = self;
	
	//7. 设置滚动不显示滚动条
	self.collectionView.indicatorStyle = UIScrollViewIndicatorStyleWhite;
}

```

- viewDidLoad时，创建UISegmentedControl，完成点击时切换layout来布局UICollectionViewCell

```
- (void)viewDidLoad
{
    [super viewDidLoad];
    
    //1. 
    UISegmentedControl *segmentedControl = [[UISegmentedControl alloc] initWithItems:@[@"Small", @"Large"]];
    
    //2. 
    segmentedControl.segmentedControlStyle = UISegmentedControlStyleBar;
    
    //3.
    segmentedControl.selectedSegmentIndex = 0;
    
    //4. 设置点击事件处理函数，切换layout
    [segmentedControl addTarget:self action:@selector(segmentedControlValueDidChange:) forControlEvents:UIControlEventValueChanged];
    
    //5. 
    self.navigationItem.titleView = segmentedControl;
}

```

- segment点击处理，完成UICollectionView实例切换layout

```
-(void)segmentedControlValueDidChange:(id)sender
{
	if (self.collectionView.collectionViewLayout == self.smallLayout)
	{	
		//1. 先停止 self.largeLayout布局
		[self.largeLayout invalidateLayout];
		
		//2. 再使用新的布局 self.largeLayout
		[self.collectionView setCollectionViewLayout:self.largeLayout animated:YES];
	}
	else
	{
		//1. 先停止 self.smallLayout布局
		[self.smallLayout invalidateLayout];
		
		//2. 再使用新的布局 self.smallLayout 
		[self.collectionView setCollectionViewLayout:self.smallLayout animated:YES];
	}
}
```

- UICollectionView DataSource & Delegate methods

```
-(NSInteger)collectionView:(UICollectionView *)collectionView numberOfItemsInSection:(NSInteger)section
{
    return 100;
}

-(UICollectionViewCell *)collectionView:(UICollectionView *)collectionView cellForItemAtIndexPath:(NSIndexPath *)indexPath
{
    AFCollectionViewCell *cell = (AFCollectionViewCell *)[collectionView dequeueReusableCellWithReuseIdentifier:ItemIdentifier forIndexPath:indexPath];
    
    [cell setImage:[UIImage imageNamed:[NSString stringWithFormat:@"%d.jpg", indexPath.item % 4]]];
    
    return cell;
}
```

- 再看下`AFCollectionViewCell`

	- 注意`prepareForReuse`方法的使用，重用时会被执行

```
@interface AFCollectionViewCell : UICollectionViewCell

-(void)setImage:(UIImage *)image;

@end
```

```
@interface AFCollectionViewCell ()

@property (nonatomic, strong) UIImageView *imageView;

@end

@implementation AFCollectionViewCell

- (id)initWithFrame:(CGRect)frame
{
    if (!(self = [super initWithFrame:frame])) return nil;
    
    self.imageView = [[UIImageView alloc] initWithFrame:CGRectInset(CGRectMake(0, 0, CGRectGetWidth(frame), CGRectGetHeight(frame)), 5, 5)];
    self.imageView.autoresizingMask = UIViewAutoresizingFlexibleHeight | UIViewAutoresizingFlexibleWidth;
    [self.contentView addSubview:self.imageView];
    
    self.backgroundColor = [UIColor whiteColor];
    
    return self;
}

-(void)prepareForReuse
{	
	//重用时，情况之前的图像
    [self setImage:nil];
}

-(void)setImage:(UIImage *)image
{
    self.imageView.image = image;
}

@end
```

- 再看下实现显示小cell布局的layout类，`AFCollectionViewFlowSmallLayout`

```
@interface AFCollectionViewFlowSmallLayout : UICollectionViewFlowLayout

@end
```

```
@implementation AFCollectionViewFlowSmallLayout

-(id)init
{
	if (!(self = [super init]))
		return nil;
	    
	//cell的大小
	self.itemSize = CGSizeMake(30, 30);
	
	//section的间距
	self.sectionInset = UIEdgeInsetsMake(10, 10, 10, 10);
	
	//cell之间的间距
	self.minimumInteritemSpacing = 10.0f;
	
	//每一行的间距
	self.minimumLineSpacing = 10.0f;
	    
	return self;
}

@end
```

- 最后看下实现大cell布局的layout类，`AFCollectionViewFlowLargeLayout`

```
@interface AFCollectionViewFlowLargeLayout : UICollectionViewFlowLayout

@end
```

```
@implementation AFCollectionViewFlowLargeLayout

-(id)init
{
	if (!(self = [super init])) 
		return nil;
	    
	 //如下做的事情同上，只是修改了 itemSize 属性，改变了cell的大小
	    
	self.itemSize = CGSizeMake(130, 130);
	self.sectionInset = UIEdgeInsetsMake(10, 10, 10, 10);
	self.minimumInteritemSpacing = 10.0f;
	self.minimumLineSpacing = 10.0f;
	    
return self;
}

@end
```

- UITableViewCell和UICollectionViewCell都有一个子类重写的方法	

	- 在一个cell将被重用的时候，被执行

```
-(void)prepareForReuse {
	//做一些请清除cell.subviews的显示的数据.
}
```

- 小结`UICollectionViewLayout`的作用
	- 将一些UICollectionViewCell布局的细节单独封装
	- 当UICollectionViewLayout种类多了以后，也可以随时切换到另一种UICollectionViewLayout

***

###找了一个简单实现瀑布流的源码，跟着源码学习了一下。

效果如下:

![](http://i3.tietuku.com/2a7a8915bc393a6c.png)


```
核心:

	1. 继承自 UICollectionViewLayout 写一个子类。
	2. Layout子类告诉UICollectionView，如何排列不规则高度的UICollectionViewCell。
```

主要代码结构:

![](http://i3.tietuku.com/e6f2ded9e573d8bd.png)

####首先从ViewController看起.

```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    //添加collectionView
    [self.view addSubview:self.collectionView];
}
```

```
- (UICollectionView *)collectionView {
    if (!_collectionView) {
        
        //1. Water Layout
        WaterFlowCollectionViewLayout *layout = [[WaterFlowCollectionViewLayout alloc] init];
        
        //设置Layout不同样式
        layout.sectionInset = UIEdgeInsetsMake(10, 10, 10, 10);
        layout.headerHeight = 15;
        layout.footerHeight = 10;
        layout.minimumColumnSpacing = 20;
        layout.minimumInteritemSpacing = 30;
        
        //2. CollectionView
        //2.1
        _collectionView = [[UICollectionView alloc] initWithFrame:self.view.bounds
                                             collectionViewLayout:layout];
        
        //2.2
        _collectionView.autoresizingMask = UIViewAutoresizingFlexibleHeight | UIViewAutoresizingFlexibleWidth;
        
        //2.3
        _collectionView.dataSource = self;
        _collectionView.delegate = self;
        _collectionView.backgroundColor = [UIColor whiteColor];
        
        
        //2.4 注册Cell
        [_collectionView registerClass:[CollectionViewWaterfallCell class]
            forCellWithReuseIdentifier:CELL_IDENTIFIER];
        
        //2.5 注册Header
        [_collectionView registerClass:[CollectionViewWaterfallHeader class]
            forSupplementaryViewOfKind:CHTCollectionElementKindSectionHeader
                   withReuseIdentifier:HEADER_IDENTIFIER];
        
        //2.6 注册Footer
        [_collectionView registerClass:[CollectionViewWaterfallFooter class]
            forSupplementaryViewOfKind:CHTCollectionElementKindSectionFooter
                   withReuseIdentifier:FOOTER_IDENTIFIER];
        
    }
    
    return _collectionView;
}

```


```
#pragma mark - UICollectionViewDataSource
//2个区，每个区都是30个

- (NSInteger)collectionView:(UICollectionView *)collectionView numberOfItemsInSection:(NSInteger)section {
    return CELL_COUNT;
}

- (NSInteger)numberOfSectionsInCollectionView:(UICollectionView *)collectionView {
    return 2;
}

- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView
                  cellForItemAtIndexPath:(NSIndexPath *)indexPath
{
    //查询cell
    CollectionViewWaterfallCell *cell =
    (CollectionViewWaterfallCell *)[collectionView dequeueReusableCellWithReuseIdentifier:CELL_IDENTIFIER forIndexPath:indexPath];
    
    //设置cell显示额图片
    cell.imageView.image = [UIImage imageNamed:self.cats[indexPath.item % 4]];
    
    return cell;
}

//查询头和尾
- (UICollectionReusableView *)collectionView:(UICollectionView *)collectionView viewForSupplementaryElementOfKind:(NSString *)kind atIndexPath:(NSIndexPath *)indexPath {
    UICollectionReusableView *reusableView = nil;
    
    if ([kind isEqualToString:CHTCollectionElementKindSectionHeader]) {
        reusableView = [collectionView dequeueReusableSupplementaryViewOfKind:kind
                                                          withReuseIdentifier:HEADER_IDENTIFIER
                                                                 forIndexPath:indexPath];
    } else if ([kind isEqualToString:CHTCollectionElementKindSectionFooter]) {
        reusableView = [collectionView dequeueReusableSupplementaryViewOfKind:kind
                                                          withReuseIdentifier:FOOTER_IDENTIFIER
                                                                 forIndexPath:indexPath];
    }
    
    return reusableView;
}

```

```
#pragma mark - WaterFlowCollectionViewLayoutDelegate（不是UICollectionViewDelegate）

//这个方法是Layout子类定义的协议方法，Layout调用这个方法获取Cell要显示的大小.

- (CGSize)collectionView:(UICollectionView *)collectionView
                  layout:(UICollectionViewLayout *)collectionViewLayout
  sizeForItemAtIndexPath:(NSIndexPath *)indexPath
{
    //返回每一个cell随机的不同的宽高
    CGSize size = [self.cellSizes[indexPath.item % 4] CGSizeValue];
    return size;
}
```

####再来看看Header、Footer、Cell的代码

```
@implementation CollectionViewWaterfallCell

- (UIImageView *)imageView {
    if (!_imageView) {
        
        //1.
        _imageView = [[UIImageView alloc] initWithFrame:self.contentView.bounds];
        
        //2. 宽高拉伸
        _imageView.autoresizingMask = UIViewAutoresizingFlexibleHeight | UIViewAutoresizingFlexibleWidth;
        
        //3. 防止图片变形
        _imageView.contentMode = UIViewContentModeScaleAspectFit;
    }
    return _imageView;
}

- (id)initWithFrame:(CGRect)frame {
    if (self = [super initWithFrame:frame]) {
        [self.contentView addSubview:self.imageView];
    }
    return self;
}


@end
```

```
@implementation CollectionViewWaterfallHeader

- (id)initWithFrame:(CGRect)frame {
    if (self = [super initWithFrame:frame]) {
        self.backgroundColor = [UIColor redColor];
    }
    return self;
}

@end
```

```
- (id)initWithFrame:(CGRect)frame {
    if (self = [super initWithFrame:frame]) {
        self.backgroundColor = [UIColor blueColor];
    }
    return self;
}
```

如上三部分的代码都很简单，估计都看的懂...

####接下来主要看看Layout子类写的一些什么代码，也是最难的。看下WaterFlowCollectionViewLayout.h向外暴露的属性。

```
@interface WaterFlowCollectionViewLayout : UICollectionViewLayout

//默认显示的列数
@property (nonatomic, assign) NSInteger columnCount;

//同FlowLayout
@property (nonatomic, assign) CGFloat minimumColumnSpacing;

//同FlowLayout
@property (nonatomic, assign) CGFloat minimumInteritemSpacing;

//头部View的高度
@property (nonatomic, assign) CGFloat headerHeight;

//尾部View的高度
@property (nonatomic, assign) CGFloat footerHeight;

//头部View的缩进
@property (nonatomic, assign) UIEdgeInsets headerInset;

//尾部View的缩进
@property (nonatomic, assign) UIEdgeInsets footerInset;

//section的缩进
@property (nonatomic, assign) UIEdgeInsets sectionInset;

//cell从哪个方向开始排列（默认ShortestColumn最小的列高度）
@property (nonatomic, assign)CHTCollectionViewWaterfallLayoutItemRenderDirection itemRenderDirection;

//CollectionView的最小contentSize
@property (nonatomic, assign) CGFloat minimumContentHeight;

//cell的宽度
- (CGFloat)itemWidthInSectionAtIndex:(NSInteger)section;

@end
```

####WaterFlowCollectionViewLayout.h定义了一个枚举，控制CollectionView排列内部Cell的样式.

```
typedef NS_ENUM (NSUInteger, CHTCollectionViewWaterfallLayoutItemRenderDirection) {
    
    //以最小高度排列Cell
    CHTCollectionViewWaterfallLayoutItemRenderDirectionShortestFirst,
    
    //从左到右
    CHTCollectionViewWaterfallLayoutItemRenderDirectionLeftToRight,
    
    //从右到左
    CHTCollectionViewWaterfallLayoutItemRenderDirectionRightToLeft
};
```

####WaterFlowCollectionViewLayout.h还定义了一个protocol，用于获取Cell的大小、以及其他数据.

```
@protocol WaterFlowCollectionViewLayoutDelegate <UICollectionViewDelegate>
@required

- (CGSize)collectionView:(UICollectionView *)collectionView
                  layout:(UICollectionViewLayout *)collectionViewLayout
  sizeForItemAtIndexPath:(NSIndexPath *)indexPath;

@optional

- (NSInteger)collectionView:(UICollectionView *)collectionView
                     layout:(UICollectionViewLayout *)collectionViewLayout
      columnCountForSection:(NSInteger)section;

- (CGFloat)collectionView:(UICollectionView *)collectionView
                   layout:(UICollectionViewLayout *)collectionViewLayout
 heightForHeaderInSection:(NSInteger)section;

- (CGFloat)collectionView:(UICollectionView *)collectionView
                   layout:(UICollectionViewLayout *)collectionViewLayout
 heightForFooterInSection:(NSInteger)section;

- (UIEdgeInsets)collectionView:(UICollectionView *)collectionView
                        layout:(UICollectionViewLayout *)collectionViewLayout
        insetForSectionAtIndex:(NSInteger)section;


- (UIEdgeInsets)collectionView:(UICollectionView *)collectionView
                        layout:(UICollectionViewLayout *)collectionViewLayout
       insetForHeaderInSection:(NSInteger)section;


- (UIEdgeInsets)collectionView:(UICollectionView *)collectionView
                        layout:(UICollectionViewLayout *)collectionViewLayout
       insetForFooterInSection:(NSInteger)section;

- (CGFloat)collectionView:(UICollectionView *)collectionView
                   layout:(UICollectionViewLayout *)collectionViewLayout
minimumInteritemSpacingForSectionAtIndex:(NSInteger)section;

- (CGFloat)collectionView:(UICollectionView *)collectionView
                   layout:(UICollectionViewLayout *)collectionViewLayout
minimumColumnSpacingForSectionAtIndex:(NSInteger)section;

@end

```

####好，再来看看WaterFlowCollectionViewLayout.m

```
@interface WaterFlowCollectionViewLayout ()

//指向的是layout对象.collectionView.delegate（所以collectionView.delegate需要实现之前定义的协议方法）
@property (nonatomic, weak) id<WaterFlowCollectionViewLayoutDelegate> delegate;

//保存每一个section中，每一列的当前总高度
@property (nonatomic, strong) NSMutableArray *columnHeights;

//保存每一个section的LayoutAttribute对象
@property (nonatomic, strong) NSMutableArray *sectionItemAttributes;

//保存所有的LayoutAttribute对象
@property (nonatomic, strong) NSMutableArray *allItemAttributes;

//保存每一个section的Header LayoutAttribute对象 
@property (nonatomic, strong) NSMutableDictionary *headersAttribute;

//保存每一个section的Footer LayoutAttribute对象
@property (nonatomic, strong) NSMutableDictionary *footersAttribute;

//collectionView所有的可见区域的并集
@property (nonatomic, strong) NSMutableArray *unionRects;

@end
```

####然后就是alloc出WaterFlowCollectionViewLayout对象之后执行的`init`方法.

```
- (id)init {
    if (self = [super init]) {
        [self commonInit];
    }
    return self;
}

- (id)initWithCoder:(NSCoder *)aDecoder {
    if (self = [super initWithCoder:aDecoder]) {
        [self commonInit];
    }
    return self;
}
```

```
初始化一些变量值

- (void)commonInit {
	
	//默认每个section都是两列
    _columnCount = 2;
    
    _minimumColumnSpacing = 10;
    _minimumInteritemSpacing = 10;
    _headerHeight = 0;
    _footerHeight = 0;
    _sectionInset = UIEdgeInsetsZero;
    _headerInset  = UIEdgeInsetsZero;
    _footerInset  = UIEdgeInsetsZero;
    
    //默认排列最小高度（瀑布流布局）
    _itemRenderDirection = CHTCollectionViewWaterfallLayoutItemRenderDirectionShortestFirst;
}
```

####然后就是.h文件中的属性的一些setter方法重写

```
- (void)setColumnCount:(NSInteger)columnCount {
    if (_columnCount != columnCount) {
        _columnCount = columnCount;
        [self invalidateLayout];
    }
}

- (void)setMinimumColumnSpacing:(CGFloat)minimumColumnSpacing {
    if (_minimumColumnSpacing != minimumColumnSpacing) {
        _minimumColumnSpacing = minimumColumnSpacing;
        [self invalidateLayout];
    }
}

- (void)setMinimumInteritemSpacing:(CGFloat)minimumInteritemSpacing {
    if (_minimumInteritemSpacing != minimumInteritemSpacing) {
        _minimumInteritemSpacing = minimumInteritemSpacing;
        [self invalidateLayout];
    }
}

- (void)setHeaderHeight:(CGFloat)headerHeight {
    if (_headerHeight != headerHeight) {
        _headerHeight = headerHeight;
        [self invalidateLayout];
    }
}

- (void)setFooterHeight:(CGFloat)footerHeight {
    if (_footerHeight != footerHeight) {
        _footerHeight = footerHeight;
        [self invalidateLayout];
    }
}

- (void)setHeaderInset:(UIEdgeInsets)headerInset {
    if (!UIEdgeInsetsEqualToEdgeInsets(_headerInset, headerInset)) {
        _headerInset = headerInset;
        [self invalidateLayout];
    }
}

- (void)setFooterInset:(UIEdgeInsets)footerInset {
    if (!UIEdgeInsetsEqualToEdgeInsets(_footerInset, footerInset)) {
        _footerInset = footerInset;
        [self invalidateLayout];
    }
}

- (void)setSectionInset:(UIEdgeInsets)sectionInset {
    if (!UIEdgeInsetsEqualToEdgeInsets(_sectionInset, sectionInset)) {
        _sectionInset = sectionInset;
        [self invalidateLayout];
    }
}

- (void)setItemRenderDirection:(CHTCollectionViewWaterfallLayoutItemRenderDirection)itemRenderDirection {
    if (_itemRenderDirection != itemRenderDirection) {
        _itemRenderDirection = itemRenderDirection;
        [self invalidateLayout];
    }
}


注意:

1. 如上所有setter方法，属性值改变后，调用如下方法:
2. -[UICollectionViewLayout invalidateLayout]
3. 该消息告诉当前layout对象的CollectionView重新布局.
```

####然后还有一些getter方法

```
//这个getter，获取某个section中，将要排成几列.（默认都是2列）

//这个值，可以通过 1)实现代理方法. 2)所有section都是同一个值.

- (NSInteger)columnCountForSection:(NSInteger)section {

    SEL sel = @selector(collectionView:layout:columnCountForSection:);
    
    if ([self.delegate respondsToSelector:sel]) 
    {
		//1. Layout定义的协议的代理实现对象实现了代理方法
        return [self.delegate collectionView:self.collectionView
                                      layout:self
                       columnCountForSection:section];
    } else 
    {	
    	//2. 所有section返回的是通过属性设置的个数，默认为2
        return self.columnCount;
    }
}
```

####然后是私有分类`@interface Layout () .. @end`中添加的属性的getter方法进行属性值懒加载

```
- (NSMutableDictionary *)headersAttribute {
    if (!_headersAttribute) {
        _headersAttribute = [NSMutableDictionary dictionary];
    }
    return _headersAttribute;
}

- (NSMutableDictionary *)footersAttribute {
    if (!_footersAttribute) {
        _footersAttribute = [NSMutableDictionary dictionary];
    }
    return _footersAttribute;
}

- (NSMutableArray *)unionRects {
    if (!_unionRects) {
        _unionRects = [NSMutableArray array];
    }
    return _unionRects;
}

- (NSMutableArray *)columnHeights {
    if (!_columnHeights) {
        _columnHeights = [NSMutableArray array];
    }
    return _columnHeights;
}

- (NSMutableArray *)allItemAttributes {
    if (!_allItemAttributes) {
        _allItemAttributes = [NSMutableArray array];
    }
    return _allItemAttributes;
}

- (NSMutableArray *)sectionItemAttributes {
    if (!_sectionItemAttributes) {
        _sectionItemAttributes = [NSMutableArray array];
    }
    return _sectionItemAttributes;
}

```

####然后就是重写UIViewCollectionViewLayout的几个系统方法，让CollectionView调用完成cell的布局.

#####重写方法1: -[UICollectionViewLayout prepareLayout]

- 当CollectionView开始布局的时候，调用这个方法，在这个方法中主要做一些cell布局前的数据统计。（如：每一列的总高度）

```
- (void)prepareLayout {
    [super prepareLayout];
    
    //1. 清除数据
    [self clearDatas];
    
    //2. 获取当前CollectionView显示需要几个Section组
    NSInteger numberOfSections = [self.collectionView numberOfSections];
    if (numberOfSections == 0) {
        return;
    }
    
    //3. 断言实现协议方法
    [self assertCondition];
    
    //4. 初始化每一个section的每一列的累计高度数组
    [self initAllSectionsTotalHeightForPerColumn:numberOfSections];
    
    //5. 初始化用来统计高度的数组，传入Section个数
    [self compute:numberOfSections];
    
}
```

```
- (void)clearDatas {
    [self.headersAttribute removeAllObjects];
    [self.footersAttribute removeAllObjects];
    [self.unionRects removeAllObjects];
    [self.columnHeights removeAllObjects];
    [self.allItemAttributes removeAllObjects];
    [self.sectionItemAttributes removeAllObjects];
}
```

```
- (void)assertCondition {
    
    //必须实现delegate方法
    NSAssert([self.delegate conformsToProtocol:@protocol(WaterFlowCollectionViewLayoutDelegate)],@"必须返回每一个cell的大小");
    
    //设置layout的属性，或者实现delegate方法
    NSAssert(self.columnCount > 0 || [self.delegate respondsToSelector:@selector(collectionView:layout:columnCountForSection:)],@"必须告诉cell的个数，且大于0");
}
```

如下方法，完成所有section的每一列的高度数组的初始化.

```
- (void)initAllSectionsTotalHeightForPerColumn:(NSInteger)numberOfSections {
    
    for (NSInteger section = 0; section < numberOfSections; section++) {
        
        // 获取每一个section有多少列
        NSInteger columnCount = [self columnCountForSection:section];
        
        //记录每一个section中，每一列的累计高度（一行有3列，就数组有三个元素，分别记录每一一列的当前总高度）
        NSMutableArray *sectionColumnHeights = [NSMutableArray arrayWithCapacity:columnCount];
        
        //依次数组每个元素赋值0
        for (NSInteger idx = 0; idx < columnCount; idx++) {
            [sectionColumnHeights addObject:@(0)];
        }
        
        //每一个section的列数组，添加到section数组保存.
        [self.columnHeights addObject:sectionColumnHeights];
    }
}
```

如下方法比较长，主要是完成CollectionView每一个section内，从`header --> 所有cells -->footer`的高度统计.

- 每一个高度（header、cell、footer）放入到高度数组累加保存.

- 每一个header、cell、footer的`布局设置`，都使用一个`UICollectionViewLayoutAttributes`对象保存.

- 最后还需要计算CollectionView内部所有区域的rect的并集.

```
- (void)compute:(NSInteger)numberOfSections {
    
    CGFloat top = 0.f;
    
    UICollectionViewLayoutAttributes *attributes;
    
    //遍历所有的Section，依次统计每一个Section的总高度
    for (NSInteger section = 0; section < numberOfSections; ++section) {
        
        //如下属性，通过2中方式设置: 1)属性 2)实现协议方法
        
        //1. 每行之间的cell距离
        CGFloat minimumInteritemSpacing;
        
        if ([self.delegate respondsToSelector:@selector(collectionView:layout:minimumInteritemSpacingForSectionAtIndex:)])
        {
            minimumInteritemSpacing = [self.delegate collectionView:self.collectionView layout:self minimumInteritemSpacingForSectionAtIndex:section];
        } else {
            minimumInteritemSpacing = self.minimumInteritemSpacing;
        }
        
        //2. 某一列中cell的距离
        CGFloat columnSpacing = self.minimumColumnSpacing;
        
        if ([self.delegate respondsToSelector:@selector(collectionView:layout:minimumColumnSpacingForSectionAtIndex:)])
        {
            columnSpacing = [self.delegate collectionView:self.collectionView layout:self minimumColumnSpacingForSectionAtIndex:section];
        }
        
        //3. cell所在section的缩进距离
        UIEdgeInsets sectionInset;
        if ([self.delegate respondsToSelector:@selector(collectionView:layout:insetForSectionAtIndex:)]) {
            sectionInset = [self.delegate collectionView:self.collectionView layout:self insetForSectionAtIndex:section];
        } else {
            sectionInset = self.sectionInset;
        }
        
        //4. 计算CollectionView内部显示内容的总宽度
        //减去左右两侧的contentInset缩进距离
        CGFloat width = self.collectionView.bounds.size.width - sectionInset.left - sectionInset.right;
        
        //5. 当前section需要几列显示
        NSInteger columnCount = [self columnCountForSection:section];

		//6. 第四步得到的CollectionView总宽度除以总列数，得到每一个CollectionViewCell的宽度
        CGFloat itemWidth = CHTFloorCGFloat((width - (columnCount - 1) * columnSpacing) / columnCount);
        
        //7. 获取 Section header
        CGFloat headerHeight;
        if ([self.delegate respondsToSelector:@selector(collectionView:layout:heightForHeaderInSection:)]) {
            headerHeight = [self.delegate collectionView:self.collectionView layout:self heightForHeaderInSection:section];
        } else {
            headerHeight = self.headerHeight;
        }
        
        //8. 获取 Section header Inset 缩进距离
        UIEdgeInsets headerInset;
        if ([self.delegate respondsToSelector:@selector(collectionView:layout:insetForHeaderInSection:)]) {
            headerInset = [self.delegate collectionView:self.collectionView layout:self insetForHeaderInSection:section];
        } else {
            headerInset = self.headerInset;
        }
        
        //9. 先保存顶部向下缩进的距离
        top += headerInset.top;
        
        //10. 再记录header的 高度
        if (headerHeight > 0) {
            
            //10.1 创建一个与当前header处于同一个NSIndexPath的UICollectionViewLayoutAttributes对象
            attributes = [UICollectionViewLayoutAttributes layoutAttributesForSupplementaryViewOfKind:CHTCollectionElementKindSectionHeader withIndexPath:[NSIndexPath indexPathForItem:0 inSection:section]];
            
            //10.2 设置LayoutAttributes对象的frame
            attributes.frame = CGRectMake(headerInset.left,
                                          top,
                                          self.collectionView.bounds.size.width - (headerInset.left + headerInset.right),
                                          headerHeight);
            
            //10.3 保存 section header 对应的 LayoutAttributes对象
            self.headersAttribute[@(section)] = attributes;//字典保存
            [self.allItemAttributes addObject:attributes];//数组保存
            
            //10.4 累计header的高度
            top = CGRectGetMaxY(attributes.frame) + headerInset.bottom;
        }
        
        
        //11. 因为添加了header的高度，所以让当前section的保存高度的数组的每一列高度，都累加一个headaer的高度
        for (NSInteger idx = 0; idx < columnCount; idx++) {
            self.columnHeights[section][idx] = @(top);
        }
        
        //12. 记录每一列不断添加Cell之后的总高度
        
        //12.1 获取当前section的中Cell的总个数
        NSInteger itemCount = [self.collectionView numberOfItemsInSection:section];
        
        //12.2 创建当前Section，用来保存所有Cell的LayoutAttributes对象的数组
        NSMutableArray *itemAttributes = [NSMutableArray arrayWithCapacity:itemCount];
        
        //12.3.1 依次统计当前Section中所有Cell的高度
        //12.3.2 并且创建与Cell相关的LayoutAttributes对象，保存Cell的frame
        for (NSInteger idx = 0; idx < itemCount; idx++) {
            
            // 获取Cell坐在的Indexpath
            NSIndexPath *indexPath = [NSIndexPath indexPathForItem:idx inSection:section];
            
            // 找到要添加当前cell的，出现的第几列位置
            //需要遍历高度数组，找到一个最小高度的列位置来添加当前Cell
            NSUInteger columnIndex = [self nextColumnIndexForItem:idx inSection:section];
            
            // Cell出现位置的 X坐标
            CGFloat xOffset = sectionInset.left + (itemWidth + columnSpacing) * columnIndex;
            
            // Cell出现位置的 Y坐标
            CGFloat yOffset = [self.columnHeights[section][columnIndex] floatValue];
            
            // Cell的Size大小尺寸
            CGSize itemSize = [self.delegate collectionView:self.collectionView layout:self sizeForItemAtIndexPath:indexPath];
            
            // 按照之前算出的itemWidth，得到比例的cell高度
            CGFloat itemHeight = 0;
            if (itemSize.height > 0 && itemSize.width > 0) {
                itemHeight = CHTFloorCGFloat(itemSize.height * itemWidth / itemSize.width);
            }
            
            // 创建每个Cell 相关的 LayoutAttribute对象
            attributes = [UICollectionViewLayoutAttributes layoutAttributesForCellWithIndexPath:indexPath];
            attributes.frame = CGRectMake(xOffset, yOffset, itemWidth, itemHeight);
            
            // 将 LayoutAttribute对象 保存到数组 
            //保存到当前Section的LayoutAttribute数组
            [itemAttributes addObject:attributes];
            
            // 将 LayoutAttribute对象 保存到数组
			  //保存到所有Cell的LayoutAttribute数组
            [self.allItemAttributes addObject:attributes];
            
            // 每加入一个cell = cell最大y值 + 纵向距离
            // 注意: 这里使用CGRectGetMaxY(attributes.frame): 可以看到UICollectionViewLayoutAttributes的作用，可以对其使用CGRect相关的API，方便的操作frame等
            self.columnHeights[section][columnIndex] = @(CGRectGetMaxY(attributes.frame) + minimumInteritemSpacing);
        }
        
        //14. 保存当前Section的LayoutAttributes数组
        [self.sectionItemAttributes addObject:itemAttributes];
        
        
        //15. footer
        CGFloat footerHeight;
        NSUInteger columnIndex = [self longestColumnIndexInSection:section];
        
        //最后section footer，不需要纵向距离，加上footer顶部距离，最终得到footer的y起始坐标
        top = [self.columnHeights[section][columnIndex] floatValue] - minimumInteritemSpacing + sectionInset.bottom;
        
        if ([self.delegate respondsToSelector:@selector(collectionView:layout:heightForFooterInSection:)]) {
            footerHeight = [self.delegate collectionView:self.collectionView layout:self heightForFooterInSection:section];
        } else {
            footerHeight = self.footerHeight;
        }
        
        UIEdgeInsets footerInset;
        if ([self.delegate respondsToSelector:@selector(collectionView:layout:insetForFooterInSection:)]) {
            footerInset = [self.delegate collectionView:self.collectionView layout:self insetForFooterInSection:section];
        } else {
            footerInset = self.footerInset;
        }
        
        top += footerInset.top;
        
        if (footerHeight > 0) {
            attributes = [UICollectionViewLayoutAttributes layoutAttributesForSupplementaryViewOfKind:CHTCollectionElementKindSectionFooter withIndexPath:[NSIndexPath indexPathForItem:0 inSection:section]];
            attributes.frame = CGRectMake(footerInset.left,
                                          top,
                                          self.collectionView.bounds.size.width - (footerInset.left + footerInset.right),
                                          footerHeight);
            
            self.footersAttribute[@(section)] = attributes;
            [self.allItemAttributes addObject:attributes];
            
            top = CGRectGetMaxY(attributes.frame) + footerInset.bottom;
        }
        
        for (NSInteger idx = 0; idx < columnCount; idx++) {
            self.columnHeights[section][idx] = @(top);
        }
    }
    
    //16.
    NSInteger idx = 0;
    NSInteger itemCounts = [self.allItemAttributes count];
    
    while (idx < itemCounts) {
        
        CGRect unionRect = ((UICollectionViewLayoutAttributes *)self.allItemAttributes[idx]).frame;
        
        //0->20,20->40,40->60,60->最大数
        NSInteger rectEndIndex = MIN(idx + unionSize, itemCounts);
        
        for (NSInteger i = idx + 1; i < rectEndIndex; i++) {
            
            //取相邻两个元素的rect的合集
            unionRect = CGRectUnion(unionRect, ((UICollectionViewLayoutAttributes *)self.allItemAttributes[i]).frame);
        }
        
        idx = rectEndIndex;
        
        [self.unionRects addObject:[NSValue valueWithCGRect:unionRect]];
    }
    
    //self.unionRects保存了四块的union rect:
    //1块: 0->20（从header开始，一直到19个cell）
    //2块: 20->40 cell
    //3块: 40->60 cell
    //4块: 60->64: 三个cell+一个footer
    
}
```

####重写 -[UICollectionViewLayout collectionViewContentSize]

指定当前CollectionView的contentSize大小

```
- (CGSize)collectionViewContentSize {
    
    NSInteger numberOfSections = [self.collectionView numberOfSections];
    if (numberOfSections == 0) {
        return CGSizeZero;
    }
    
    CGSize contentSize = self.collectionView.bounds.size;
    
    //获取当前最大内容的高度
    contentSize.height = [[[self.columnHeights lastObject] firstObject] floatValue];
    
    if (contentSize.height < self.minimumContentHeight) {
        contentSize.height = self.minimumContentHeight;
    }
    
    return contentSize;
}

```

####重写 -[UICollectionViewLayout layoutAttributesForItemAtIndexPath:]

返回item对应cell的LayoutAttribute对象（获取frame）


```
- (UICollectionViewLayoutAttributes *)layoutAttributesForItemAtIndexPath:(NSIndexPath *)path {
    
    //section超过
    if (path.section >= [self.sectionItemAttributes count]) {
        return nil;
    }
    
    //section.item超过
    if (path.item >= [self.sectionItemAttributes[path.section] count]) {
        return nil;
    }
    
    return (self.sectionItemAttributes[path.section])[path.item];
}

```

####重写 -[UICollectionViewLayout layoutAttributesForSupplementaryViewOfKind: atIndexPath:]

返回header或footer的LayoutAttribute


```
- (UICollectionViewLayoutAttributes *)layoutAttributesForSupplementaryViewOfKind:(NSString *)kind atIndexPath:(NSIndexPath *)indexPath
{
    UICollectionViewLayoutAttributes *attribute = nil;
    
    if ([kind isEqualToString:CHTCollectionElementKindSectionHeader]) {
        attribute = self.headersAttribute[@(indexPath.section)];
    } else if ([kind isEqualToString:CHTCollectionElementKindSectionFooter]) {
        attribute = self.footersAttribute[@(indexPath.section)];
    }
    
    return attribute;
}
```

####重写 -[UICollectionViewLayout layoutAttributesForElementsInRect:]

返回一个当前CollectionView可见区域rect内，所有cell的LayoutAttribute数组

```
- (NSArray *)layoutAttributesForElementsInRect:(CGRect)rect {
    NSInteger i;
    NSInteger begin = 0, end = self.unionRects.count;
    NSMutableArray *attrs = [NSMutableArray array];
    
    //1. 获取开始的位置
    for (i = 0; i < self.unionRects.count; i++) {
        //开始发生有交集的rect
        if (CGRectIntersectsRect(rect, [self.unionRects[i] CGRectValue])) {
            begin = i * unionSize;// i * 20
            break;
        }
    }
    
    //2. 获取结束的位置
    for (i = self.unionRects.count - 1; i >= 0; i--) {
        //结束发生有交集的rect
        if (CGRectIntersectsRect(rect, [self.unionRects[i] CGRectValue])) {
            end = MIN((i + 1) * unionSize, self.allItemAttributes.count);
            break;
        }
    }
    
    //3. 返回从开始位置到结束位置之间，在屏幕上可见部分对应cell的LayoutAttribute数组
    for (i = begin; i < end; i++) {
        
        //获取每一个LayoutAttribute对象
        UICollectionViewLayoutAttributes *attr = self.allItemAttributes[i];
        
        //如果包含，添加到数组
        if (CGRectIntersectsRect(rect, attr.frame)) {
            [attrs addObject:attr];
        }
    }
    
    //返回LayoutAttribute数组
    return [NSArray arrayWithArray:attrs];
}

注意:

	CGRectIntersectsRect(rect1, rect2): 查看rect1与rect2是否有交集.

```

####重写 -[UICollectionViewLayout shouldInvalidateLayoutForBoundsChange]

告诉UICollectionView什么时候需要重新布局

```
- (BOOL)shouldInvalidateLayoutForBoundsChange:(CGRect)newBounds {
    CGRect oldBounds = self.collectionView.bounds;
    
    //屏幕宽度发生改变（屏幕旋转）时，告诉collectionView重新布局
    if (CGRectGetWidth(newBounds) != CGRectGetWidth(oldBounds)) {
        return YES;
    }
    
    return NO;
}
```

总结一下:

- 所有section的每一列的高度统计.
- 每一个元素（Header、Cell、Footer）创建对应的 LayoutAttributes对象并保存.
- 计算rect并集.
- 重写UICollectionViewLayout的系统方法，以便CollectionView调用，知道如何布局.

###总结主要重写UICollectionViewLayout的方法:

```
//1. 
- (void)prepareLayout;

//2. 
- (CGSize)collectionViewContentSize;

//3.
- (UICollectionViewLayoutAttributes *)layoutAttributesForItemAtIndexPath:(NSIndexPath *)path;

//4. 
- (UICollectionViewLayoutAttributes *)layoutAttributesForSupplementaryViewOfKind:(NSString *)kind 
			atIndexPath:(NSIndexPath *)indexPath;
			
//5.
- (NSArray *)layoutAttributesForElementsInRect:(CGRect)rect;

//6.
- (BOOL)shouldInvalidateLayoutForBoundsChange:(CGRect)newBounds;

```

***

###完成类似ScrollView来滚动图片，循环利用

- 方法1、使用一个UIScrollView + 两个UIImageView，循环调换UIImageView

- 方法2、使用UICollectionView，已经完成了类似TableViewCell的缓存读取

***