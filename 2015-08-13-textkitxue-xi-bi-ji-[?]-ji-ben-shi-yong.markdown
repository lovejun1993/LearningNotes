---
layout: post
title: "TextKit学习笔记一基本使用"
date: 2015-08-13 23:09:43 +0800
comments: true
categories: 
---



###一直都是使用别人封装的框架完成的图文混排，于是今天自己学习一下图文混排的简单实现原理，使用ios7的TextKit。

***

###首先看看，使用富文本属性设置多格式字符串.

```
//原始数据字符串
NSString *string = @"我是好人我是好人我是个好人额鹅鹅鹅鹅鹅鹅饿哇哇哇哇哇哇哇";
    
// 创建可变属性化字符串
NSMutableAttributedString *attrString = [[NSMutableAttributedString alloc] initWithString:string];
    
//改变字符串当中从第18位置向后的10位数的字体
UIFont *smallFont = [UIFont systemFontOfSize:30];
[attrString addAttribute:NSFontAttributeName
                   value:smallFont
                   range:NSMakeRange(5, 10)];
    
//改变字符串当中第一个“1”的颜色
UIColor *rcolor = [UIColor redColor];
[attrString addAttribute:NSForegroundColorAttributeName
                   value:rcolor
                   range:[string rangeOfString:@"是"]];
    
UILabel *label = [[UILabel alloc] initWithFrame:CGRectMake(20, 44, 320, 200)];
label.numberOfLines = 0;
[self.view addSubview:label];
label.attributedText = attrString;
```

小结富文本字符串属性三个东西:

1. 要设置的富文本属性名字. 【如: NSForegroundColorAttributeName颜色】

2. 要设置的富文本属性对应的值.【如: [UIColor redColor]】

3. 上如富文本设置对应的范围. [如:[string rangeOfString:@"是"]]

也就是:

```
1. attribute key
2. attribute value
3. range
```


***

###TextKit中主要用到的几个Api:

* **NSTextStorage**对象 ， 表示存储一段需要被处理的原始文本字符串和一些富文本属性设置项（NSAttributeString）
 
* **NSLayoutManager**对象 ，文字排版布局管理

* **NSTextContainer**对象 ，决定能够显示内容的区域

* **NSAttributeString**， **NSMUtableAttrinuteString**，**NSParagraphStyle** ....等等一些富文本基本设置key

***

###Demo1: 实现一个文字个性排版.



- 第一步:创建textView

```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.title = @"关于我们";
    
    [self createTextView];
}

```

- 第二步:创建TextView时，创建NSTextContainer


```
- (void)createTextView {
    
    
    _textView = [[UITextView alloc] initWithFrame:CGRectMake(20, CGRectGetMaxY(self.lebel.frame) + 30, SCREEN_WIDTH-40, CGRectGetMaxY(self.view.frame)-CGRectGetMaxY(_lebel.frame)) textContainer:self.textContainer];
    
    _textView.scrollEnabled                    = YES;
    _textView.editable                         = NO;
    _textView.selectable                       = NO;
    _textView.layer.masksToBounds              = YES;
    _textView.showsVerticalScrollIndicator     = NO;
//    _textView.delegate                         = self;
    
    [self.view addSubview:_textView];
}

```

- 第三步:创建NSTextContainer时，同时创建NSTextStorage、NSLayoutManager

```
- (NSTextContainer *)textContainer {
    
    NSString *text = @"我加金融是深圳中顺易金融服务有限公司（以下简称“中顺易”）打造的互联网信托平台，致力于为客户提供投资理财、消费信托、社区金融等多元化金融产品和创新消费服务。\n中顺易公司由由中信信托有限责任公司、顺丰速运(集团)有限公司、杭州网易投资有限公司联合发起设立，注册资本10亿元。\n基于互联网+、发展普惠金融、扩大消费等国家战略，秉承三家股东公司在信托金融服务、现代物流服务和互联网技术创新方面的优势，中顺易通过互联网信息技术，有效整合资金流、信息流和物流，为用户提供稳健收益、品质消费的互联网金融产品和服务。";
    
    _textStorage = [[NSTextStorage alloc] initWithString:text attributes:self.paragraphyDict];
    
    _layoutManager = [[NSLayoutManager alloc] init];
    [_textStorage addLayoutManager:_layoutManager];
    
    _textContainer = [[NSTextContainer alloc] init];
    CGFloat width  = SCREEN_WIDTH - 20 * 2;
    CGSize size = CGSizeMake(width, MAXFLOAT);
    _textContainer.size = size;
    
    [_layoutManager addTextContainer:_textContainer];
    
    return _textContainer;
}
```

- 第四步，创建告诉TextKit如何排版TextView内容的`富文本属性字典`.

```
- (NSDictionary *)paragraphyDict {
    
    
    NSMutableDictionary *attributes = [NSMutableDictionary dictionary];
    
    UIColor *textColor = [UIColor hexString:@"#666666"];
    [attributes setValue:textColor forKey:NSForegroundColorAttributeName];
    
    UIFont *textFont = [UIFont systemFontOfSize:13.f];
    [attributes setValue:textFont forKey:NSFontAttributeName];
    
    //创建段落样式
    [attributes setValue:[self style] forKey:NSParagraphStyleAttributeName];
    
    return attributes;
}

```

- 第五步、由上调用得到一个`段落样式`

```
- (NSMutableParagraphStyle *)style {
	
	NSMutableParagraphStyle *style = [[NSMutableParagraphStyle alloc] init];
    
    style.lineSpacing = 8.5f;
    style.paragraphSpacing = 25.f;
    style.firstLineHeadIndent = 20.f;
    
    return style;
}
```

- 最终效果图示

![效果图](http://m1.yea.im/2Iz.png)

- 小结下关于`NSMutableParagraphStyle段落样式`的所有属性

```
// 段落的风格（设置首行，行间距，对齐方式什么的）看自己需要什么属性，写什么  

NSMutableParagraphStyle *paragraphStyle = [[NSMutableParagraphStyle alloc] init];  

// 字体的行间距
paragraphStyle.lineSpacing = 10;  

//首行缩进  
paragraphStyle.firstLineHeadIndent = 20.0f;

//段头缩进(首行除外) 
paragraphStyle.headIndent = 20;

//段尾缩进
paragraphStyle.tailIndent = 20;

//（两端对齐的）文本对齐方式：（左，中，右，两端对齐，自然）  
paragraphStyle.alignment = NSTextAlignmentJustified;

//结尾（还有头部、中间）部分的内容以……方式省略 ( "...wxyz" ,"abcd..." ,"ab...yz") 
paragraphStyle.lineBreakMode = NSLineBreakByTruncatingTail;

//最低行高
paragraphStyle.minimumLineHeight = 10;  

//最大行高
paragraphStyle.maximumLineHeight = 20;  

//段与段之间的间距  
paragraphStyle.paragraphSpacing = 15;

//段首行空白空间
paragraphStyle.paragraphSpacingBefore = 22.0f;

//从左到右的书写方向（还有其他方向）
paragraphStyle.baseWritingDirection = NSWritingDirectionLeftToRight;  


paragraphStyle.lineHeightMultiple = 15;
  
//连字属性 在iOS，唯一支持的值分别为0和1 
paragraphStyle.hyphenationFactor = 1; 
```

****

###Demo2: 简单实现一个图文混排的demo，效果图如下:

![](http://i3.tietuku.com/07926cd5b5f028b1.png)

 ***
 
 
### No1. ViewController中，创建BookView（封装图文混排的view）
 
 ```objc
 
 - (void)setupBookView {
    
    //1. 创建BookView
    _bookView = [[BookView alloc] initWithFrame:CGRectMake(10, 10, Width-20, self.view.frame.size.height - 10)];
    [self.view addSubview:_bookView];
    
    //2. 设置BookView要显示的文字
    _bookView.text = _text;
    
    //3. 设置BookView段落样式字典
    _bookView.paregaphyAttributeArgument = [ParagraphAttributes paragraphStyle1];
    
    //4. 设置BookView富文本属性
    _bookView.attributeStringArray = @[

										//字体颜色                            
										[StringAttributeItem foregroundColor:[UIColor blackColor] range:NSMakeRange(0, 9)],
										
										//字体大小、字体样式                                      
                                        [StringAttributeItem font:[UIFont systemFontOfSize:22.0f] range:NSMakeRange(0, 9)]
                                       ];
    
    //5. 设置BookView不让文字排版的区域
    //每一个排除的区域，使用一个 ExclusionView 表示
    ExclusionView *excludeView = [[ExclusionView alloc] initWithFrame:CGRectMake(50, 150, 200, 150)];
    self.bookView.exclusionViews = @[excludeView];
    
    UIImageView *imageView       = [[UIImageView alloc] initWithFrame:excludeView.bounds];
    imageView.image              = [UIImage imageNamed:@"image"];
    [excludeView addSubview:imageView];
    
    //6. 创建BookView内部的TextView，并设置排版
    [self.bookView createSubviews];
    
    //..
    [self.view addSubview:_bookView];
    
}
 
 ```
 
 
###No2. BookView内部封装一个TextView实现图文混排

BookView.h

```objc
@interface BookView : UIView

//显示的文字
@property (copy, nonatomic) NSString *text;

//段落设置
@property (strong, nonatomic) NSDictionary *paregaphyAttributeArgument;

//NSAttrribute数组
@property (strong, nonatomic) NSArray *attributeStringArray;

// 存储的图文混排的views
@property (nonatomic, strong)   NSArray        *exclusionViews;

//创建内部TextView，开始图文混排
- (void)createSubviews;

@end

```

BookView.m

```objc

//创建UITextView
- (void)createSubviews {
    
    //1.
    [self textStorage];
    
    //2.
    [self configAllAttributeStringOptions];
    
    //3.
    [self layoutManager];
    
    //4.
    [self textContainer];
    
    //5.
    _textView = [self textView];
    
    //6.
    [self markWord:@"汉子" inTextStorage:self.textStorage];
    
}


// 创建文本容器 NSTextStorage， 并设置段落样式
- (NSTextStorage *)textStorage {
    if (!_textStorage) {
        _textStorage = [[NSTextStorage alloc] initWithString:self.text
                                                  attributes:self.paregaphyAttributeArgument];
    }
    
    return _textStorage;
}

// 给textStorage设置富文本属性
- (void)configAllAttributeStringOptions {
    for (StringAttributeItem *item in self.attributeStringArray) {
        [self.textStorage addAttribute:[item attributeKey]
                            value:[item value]
                            range:[item range]];
    }

}

- (NSLayoutManager *)layoutManager {
    if (!_layoutManager) {
        
        _layoutManager = [[NSLayoutManager alloc] init];
        [self.textStorage addLayoutManager:self.layoutManager];
    }
    
    return _layoutManager;
}

//设置可以显示的区域
- (NSTextContainer *)textContainer {
    if (!_textContainer) {
        
        _textContainer = [[NSTextContainer alloc] init];
        
        CGFloat width  = self.frame.size.width;
        CGSize size = CGSizeMake(width, MAXFLOAT);//TextView无限制向下显示
        _textContainer.size = size;
        
        [self.layoutManager addTextContainer:_textContainer];
        
        return _textContainer;
    }
    
    return _textContainer;
}

// 给TextView添加带有内容和布局的容器
- (UITextView *)textView {
    if (!_textView) {
        
        //使用 NSTextContainer对象，创建出UITextView， 控制显示区域
        _textView = [[UITextView alloc] initWithFrame:self.frame
                                        textContainer:self.textContainer];
        
        _textView.scrollEnabled                    = YES;
        _textView.editable                         = NO;
        _textView.selectable                       = NO;
        _textView.layer.masksToBounds              = YES;
        _textView.showsVerticalScrollIndicator     = NO;
        _textView.delegate                         = self;
        
        //重要: 设置不让文字排版的view所在区域
        [self configExcludeViews:_textView];
        
        [self addSubview:_textView];
    }
    
    return _textView;
}

//将不让文字排版区域的path都保存下来，设置给TextContainer
- (void)configExcludeViews:(UITextView *)textView {
    
    if ((!self.exclusionViews) || (self.exclusionViews.count == 0)) {
        return;
    }
    
    NSMutableArray *excludePaths = [NSMutableArray arrayWithCapacity:self.exclusionViews.count];
    
    for (ExclusionView *excludeView in self.exclusionViews) {
        
        //将排除不显示文字的view依次加入到 _textView
        [textView addSubview:excludeView];
        
        UIBezierPath *bezierPath = [excludeView createExcludeBezierPathWithSelf];
        [excludePaths addObject:bezierPath];
    }
    
    //将不显示文字的区域，设置给 textView.textContainer
    self.textContainer.exclusionPaths = excludePaths;
}


```

###No3. ExclusionView作为排除文字排版的区域view的容器

//.h

```objc
@interface ExclusionView : UIView

//根据自身的frame构造处UIBezierPath边界
- (UIBezierPath *)createExcludeBezierPathWithSelf;

@end

```

//.m

```objc 
@implementation ExclusionView

- (UIBezierPath *)createExcludeBezierPathWithSelf {
    return [UIBezierPath bezierPathWithRect:self.frame];
}

@end

```

###No4. ParagraphAttributes封装段落文字的富文本属性设置项的实体类

.h

```objc

@interface ParagraphAttributes : NSObject

//实体属性
@property (strong, nonatomic) UIColor *textColor;
@property (strong, nonatomic) UIFont *textFont;
@property (assign, nonatomic) CGFloat textPadding;//文字之间的间距
@property (strong, nonatomic) NSNumber *perLinePadding;//行之间的间距
@property (strong, nonatomic) NSNumber *paragraphPadding;//段落之间的间距
@property (strong, nonatomic) NSNumber *firstLineInParagraphPadding;//每一个段落第一行缩进距离

//将实体属性，封装成富文本属性的NSDictionary
- (NSDictionary *)createAttributes;

@end

//一个工厂方法获取对象的分类
@interface ParagraphAttributes (Factory)

+ (NSDictionary *)paragraphStyle1;

@end

```

.m

```objc
@implementation ParagraphAttributes

- (NSDictionary *)createAttributes {
    
    NSMutableDictionary *attributes = [NSMutableDictionary dictionary];
    
    //字体颜色
    UIColor *textColor = self.textColor ? self.textColor : [ParagraphConfig default_textColor];
    [attributes setValue:textColor forKey:NSForegroundColorAttributeName];
    
    //字体样式
    UIFont *textFont = self.textFont ? self.textFont : [ParagraphConfig default_textFont];
    [attributes setValue:textFont forKey:NSFontAttributeName];
    
    //段落样式构造，使用一个单独的函数写
    [attributes setValue:[self paragraphStyle] forKey:NSParagraphStyleAttributeName];
    
    //每个字的间距
    NSNumber *kern = (self.textPadding <=0)?@([ParagraphConfig default_textPadding]):@(self.textPadding);
    [attributes setValue:kern forKey:NSKernAttributeName];
    
    return attributes;
}

- (NSMutableParagraphStyle *)paragraphStyle {
    NSMutableParagraphStyle *style = [[NSMutableParagraphStyle alloc] init];
    
    //行间距
    style.lineSpacing = self.perLinePadding?self.perLinePadding.floatValue:[ParagraphConfig default_linePadding];
    
    //段落间距
    style.paragraphSpacing = self.paragraphPadding?self.paragraphPadding.floatValue:[ParagraphConfig default_paragraphPadding];
    
    //段落首航缩进
    style.firstLineHeadIndent = self.firstLineInParagraphPadding?self.firstLineInParagraphPadding.floatValue:[ParagraphConfig default_FirLinePadding];
    
    return style;
}

@end

//分类
@implementation ParagraphAttributes (Factory)

//提供一个快速得到一个对象的工厂方法
+ (NSDictionary *)paragraphStyle1 {
    
    //1. 创建实体类对象
    ParagraphAttributes *style = [[ParagraphAttributes alloc] init];
    
    style.textColor = READ_WORD_COLOR;
    style.textFont = [UIFont fontWithName:@"FZQKBYSJW--GB1-0" size:22.f];
    style.textPadding = 1;
    
    style.perLinePadding = @(10);
    style.paragraphPadding = @(40);
    style.firstLineInParagraphPadding = @(0);
    
    //2. 打包成富文本属性字典，返回出去
    id styleDict = [style createAttributes];
    
    return styleDict;
}

@end

```

###No4. StringAttributeItem封装一个具体的富文本设置项

.h

```objc
@interface StringAttributeItem : NSObject

@property (copy, nonatomic) NSString *attributeKey;
@property (copy, nonatomic) id value;
@property (assign, nonatomic) NSRange range;

//通用配置
+ (instancetype)attributeItemWithKey:(NSString *)key
                               Value:(id)value
                               Range:(NSRange)range;


// 配置字体
+ (instancetype)font:(UIFont *)font
               range:(NSRange)range;

// 配置字体颜色
+ (instancetype)foregroundColor:(UIColor *)color
                          range:(NSRange)range;

// 配置字体背景颜色
+ (instancetype)backgroundColor:(UIColor *)color
                          range:(NSRange)range;

// 字体描边颜色
+ (instancetype)strokeColor:(UIColor *)color
                      range:(NSRange)range;

// 字体描边宽度
+ (instancetype)strokeWidth:(float)number
                      range:(NSRange)range;

// 配置字体阴影
+ (instancetype)shadow:(NSShadow *)shadow
                 range:(NSRange)range;

// 配置文字的中划线
+ (instancetype)strikethroughStyle:(NSInteger)number
                             range:(NSRange)range;

// 配置文字的下划线
+ (instancetype)underlineStyle:(NSInteger)number
                         range:(NSRange)range;

// 字间距
+ (instancetype)kern:(float)number
               range:(NSRange)range;

// 段落样式(需要将UILabel中的numberOfLines设置成0才有用)
+ (instancetype)paragraphStyle:(NSMutableParagraphStyle *)style
                         range:(NSRange)range;

@end

```

.m 

```objc

//举一个方法
+ (instancetype)strokeColor:(UIColor *)color range:(NSRange)range {
    StringAttributeItem *config = [self new];
 	
 	//1. key   
    config.attributeKey = NSStrokeColorAttributeName;
    
    //2. 对应的配置对象
    config.value = color;
    
    //3. 区域
    config.range = range;
    
    return config;
}

```

###No5. 最后一个搜索关键词，并设置富文本属性改变样式

```objc

- (void) markWord:(NSString*)word inTextStorage:(NSTextStorage*)textStorage
{
    //1. regx
    NSRegularExpression *regex = [NSRegularExpression regularExpressionWithPattern:word
                                                                           options:0
                                                                             error:nil];
    
    //2. 查找到单词的位置
    NSArray *matches = [regex matchesInString:self.textView.text
                                      options:0
                                        range:NSMakeRange(0, [self.textView.text length])];
    
    //3. 设置文字效果
    for (NSTextCheckingResult *match in matches) {
        
        NSRange matchRange = [match range];
        
        //storage添加属性字符串设置
        [textStorage addAttribute:NSForegroundColorAttributeName
                            value:[UIColor redColor]
                            range:matchRange];
        
        [textStorage addAttribute:NSFontAttributeName
                            value:[UIFont systemFontOfSize:22.0f]
                            range:matchRange];
    }
}


```