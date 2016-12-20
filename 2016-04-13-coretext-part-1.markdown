---
layout: post
title: "CoreText Part 1"
date: 2016-04-13 22:15:56 +0800
comments: true
categories: 
---

###由于实现txt阅读那么就不可避免的需要与文字排列、效果修饰、点击处理...等等文字之类的东西打交道。

- iOS7之前，只有一个基本的文本布局和渲染框架 >>>> CoreText
	- 是一套纯c语言的api，很接近底层，还需要自己注意释放内存

- 但是在iOS7之后，苹果推出 >>> TextKit
	- 基于CoreText封装
	- 并且提供了几组api类:
		- NSTextContainer
		- NSTextStorage
		- NSLayoutManager

- 下图是截取自YYText介绍自己的Architeture时的TextKit结构图:

![](http://i4.piimg.com/f78913b528cb4e6b.png)

所以仍然还是可以通过CoreText完成各种复杂的文字排版以及效果，但是通过TextKit可能就会变得简单一些。


****

###文字排版中涉及到的一些基础概念

- 字体(Font)
	- 是一系列字号、样式和磅值相同的`字符`
	- 当我们为文字设置粗体，斜体时其实是使用了另外一种字体(下划线不算)

- 字体集(Font family)
	- 表示一个字体簇
	- 每一个字体簇，包含多种不同的字体Font

- 磅值(Weight)
	- 于描述字体的`粗`度
	- 如: 有极细、细、book、中等、半粗、粗、较粗、极粗

- 样式(Style)
	- `字形`的样式，有三种样式:
	- roman type >>> 直体
	- oblique type >>> 斜体
	- utakuc type >>> 斜体兼曲线

- x高度(X height)
	- 指 `小写字母` 的 平均高度，以`字母x`为基准值

- Cap高度(Cap height)
	- 指 `大写字母` 的 平均高度，因 `字母C`为基准值

- 字符(Character)和字形(Glyphs)
	- 排版过程中一个重要的步骤就是从字符到字形的转换
	- 字符表示`信息`本身，字符一般就是指某种`编码`，如Unicode编码 >>>> 到底是什么字（数据）
		- 我们输入的一个具体的字，但是在计算机中使用编码存储
	- 字形是它的图形`表现形式`，字形则是这些编码对应的`图片` >>> 长什么样（图片）
		- 最终被渲染在iOS系统界面上的图片
	- 那么一个字就可以使用不同的样式表现出来，所以字符与字形并不是一一对应

- 字形描述集(Glyphs Metris)
	- `字形` 的 所有的 参数

![](http://i3.piimg.com/0320fcf09b73c0b2.png)

- 边框(Bounding Box)
	- 一个假想的边框，尽可能地容纳整个字形

- 基线(Baseline)
	- 一条假想的参照线，以此为基础进行字形的渲染
	- 一般来说是一条横线

![](http://i3.piimg.com/0320fcf09b73c0b2.png)

- 基础原点(Origin)
	- 首先是位于`基线`上
	- 处于基线`最左侧`的位置

- 行间距(Leading)
	- 行与行 之间的间距

- 字间距(Kerning)
	- 字与字 之间的距离
	- 为了排版的美观，`并不是` 所有的字形之间的距离都是一致的

- 上行高度(Ascent)和下行高度(Decent)
	- 上行高度(Ascent) >>> 字形的最高点 ~ 基线的距离 >>> 正数
	- 下行高度(Decent) >>> 字形的最低点 ~ 基线的距离 >>> 负数

![](http://i3.piimg.com/273551fdd62b4c1d.png)

再看下 `行高`、 `上行高度` 、`下行高度`、 `行间距`  他们之间的关系图:

![](http://i3.piimg.com/c27113eb86e75316.png)

- 行高 >>> 整个红色框的高度
- 上行高度 >>> 红色框顶部线 ~ 绿色基线 的距离
- 下行高度 >>> 绿色基线 ~ 黄色框顶部线 的距离
- 行间距 >>> 整个黄色框的高度

那么说，行高其实是包含一个行间距的。由此可以得出：lineHeight(行高) = Ascent（上行高度） + |Decent（下行高度）| + Leading（行间距）

- 段落样式（Paragragh Style）

主要是控制段落文字显示的风格样式

```objc
@interface NSMutableParagraphStyle : NSParagraphStyle

@property(NS_NONATOMIC_IOSONLY) CGFloat lineSpacing;
@property(NS_NONATOMIC_IOSONLY) CGFloat paragraphSpacing;
@property(NS_NONATOMIC_IOSONLY) NSTextAlignment alignment;
@property(NS_NONATOMIC_IOSONLY) CGFloat firstLineHeadIndent;
@property(NS_NONATOMIC_IOSONLY) CGFloat headIndent;
@property(NS_NONATOMIC_IOSONLY) CGFloat tailIndent;
@property(NS_NONATOMIC_IOSONLY) NSLineBreakMode lineBreakMode;
@property(NS_NONATOMIC_IOSONLY) CGFloat minimumLineHeight;
@property(NS_NONATOMIC_IOSONLY) CGFloat maximumLineHeight;
@property(NS_NONATOMIC_IOSONLY) NSWritingDirection baseWritingDirection;
@property(NS_NONATOMIC_IOSONLY) CGFloat lineHeightMultiple;
@property(NS_NONATOMIC_IOSONLY) CGFloat paragraphSpacingBefore;
@property(NS_NONATOMIC_IOSONLY) float hyphenationFactor;
@property(null_resettable, copy, NS_NONATOMIC_IOSONLY) NSArray<NSTextTab *> *tabStops NS_AVAILABLE(10_0, 7_0);
@property(NS_NONATOMIC_IOSONLY) CGFloat defaultTabInterval NS_AVAILABLE(10_0, 7_0);
@property(NS_NONATOMIC_IOSONLY) BOOL allowsDefaultTighteningForTruncation NS_AVAILABLE(10_11, 9_0);

- (void)addTabStop:(NSTextTab *)anObject NS_AVAILABLE(10_0, 9_0);
- (void)removeTabStop:(NSTextTab *)anObject NS_AVAILABLE(10_0, 9_0);

- (void)setParagraphStyle:(NSParagraphStyle *)obj NS_AVAILABLE(10_0, 9_0);

@end
```

属性的意思如其名，大概意思就那样的了...

- 描边(Stroke)
	- 组成字符的 `线或曲线`，并且可以加粗或改变字符形状

***

###TextKit相比CoreText提供的功能:

- 字距调整（Kerning）

![](http://i3.piimg.com/250918c7a645f5ee.png)

- 连写

字母之间的连写

- 图像附件（ NSTextAttachment ）
	- 可以在普通文本内容里面嵌入图片文件


- 断字
	- TextKit设置 hyphenationFactor 属性就可以启用断字


- 更多的富文本属性
	- 可以设置不同的`下划线样式`（双线、粗线、虚线、点线，或者它们的组合）
	- 渲染的文本绘制`背景`颜色

- 序列化
	- TextKit提供了将带属性的富文本字符串直接写入磁盘或从磁盘读取
	- NSAttributedString 实现了 NSSecureCoding协议，既可以完成磁盘文件归档与解档

****

###再看下使用TextKit的结构图:

![](http://i4.piimg.com/0214e1068b152ff9.png)

- NSTextStorage: 主要用来存储被处理的文本字符串

- NSTextContainer: 定义了一个用来 绘制文本 的 `区域`
	- 在简单的情况下，这是一个垂直的`无限大`的矩形区域
	- 文本被填充到这个区域，并且文本视图允许用户`滚动`它

- NSLayoutManager: 布局管理器是中心组件
	- 监听NSTextStorage中存储的文本或属性改变的通知，一旦接收到通知就触发布局进程
	- 从文本存储提供的文本开始，它将所有的字符翻译为`字形`（Glyph）
	- 一旦`字形`全部生成，这个管理器向它的`文本容器 NSTextStorage`（们）`查询`文本可用来 `绘制的区域`
	- 然后这些区域被`行`逐步填充，而`行`又被`字形`逐步填充
		- 一旦一行填充完毕，`下一行`开始填充
	- 对于每一行，布局管理器必须考虑`断行`行为
		- 放不下的单词必须移到下一行
		- 连字符
		- 内联的图像附件
	- 所有工作完成后，布局管理器将前面几步排版好的文本设给`文本视图 UITextView`

在真个TextKit的体系中，没有直接包含CoreText，但是最终是被TextKit调用，所以最终的文字排版工作还是由CoreText完成。


我一直觉得上面绘制字形的部分，总觉得和CoreGraphics有一点类似，后来找到了一张TextKit的架构图，终于解决了之前的疑惑:

![](http://i2.piimg.com/a8f7f4e16204986d.png)

- TextKit调用CoreText完成文字排列、富文本修饰 >>> 字符数据到字形图片的翻译工作，也就是说找到要显示的图片了.

- 而最终负责将找到的字形图片渲染到iOS系统屏幕上时，是通过CoreGraphics库完成.

****


###先看个使用CoreText在一个Label中实现简单的文字排版

```objc
#import <UIKit/UIKit.h>

@interface CoreTextLabel : UILabel

//字间距
@property(nonatomic,assign) CGFloat characterSpacing;

//行间距
@property(nonatomic,assign)long linesSpacing;

@end
```

```objc
#import "CoreTextLabel.h"

//导入系统文字处理库
#import <CoreText/CoreText.h>

@implementation CoreTextLabel

-(id) initWithFrame:(CGRect)frame
{
    if(self =[super initWithFrame:frame])
    {
        //初始化字间距、行间距
        self.characterSpacing = 2.0f;
        self.linesSpacing = 5.0f;
    }
    
    return self;
}

//外部调用设置字间距
-(void)setCharacterSpacing:(CGFloat)characterSpacing {
    
    //1. 设置值
    _characterSpacing = characterSpacing;
    
    //2. 重新绘制文字
    [self setNeedsDisplay];
}

//外部调用设置行间距
-(void)setLinesSpacing:(long)linesSpacing {
    
    //1. 设置值
    _linesSpacing = linesSpacing;
    
    //2. 重新绘制文字
    [self setNeedsDisplay];
}

#pragma mark - 系统绘制方法

- (void)drawTextInRect:(CGRect)rect {
    
//////////////////////////////去掉文字的空行//////////////////////////////
    NSString *labelString = self.text;
    NSString *myString = [labelString stringByReplacingOccurrencesOfString:@"\r\n" withString:@"\n"];

//////////////////////////////创建富文本属性字符串//////////////////////////////
    NSMutableAttributedString *attStr = [[NSMutableAttributedString alloc] initWithString:myString];
    
//////////////////////////////添加 CoreText 字体//////////////////////////////
    CTFontRef helveticaBold = CTFontCreateWithName((CFStringRef)self.font.fontName,
                                                   self.font.pointSize,
                                                   NULL);
    
    [attStr addAttribute:(id)kCTFontAttributeName
                   value:(__bridge id)helveticaBold
                   range:NSMakeRange(0,[attStr length])];
    

//////////////////////////////设置字间距//////////////////////////////
    if(self.characterSpacing) {
        
        long number = self.characterSpacing;
        
        CFNumberRef num = CFNumberCreate(kCFAllocatorDefault, kCFNumberSInt8Type, &number);
        
        [attStr addAttribute:(id)kCTKernAttributeName
                       value:(__bridge id)num
                       range:NSMakeRange(0,[attStr length])];
        
        CFRelease(num);
    }
    
//////////////////////////////设置字体颜色//////////////////////////////
    [attStr addAttribute:(id)kCTForegroundColorAttributeName
                   value:(id)(self.textColor.CGColor)
                   range:NSMakeRange(0,[attStr length])];
    
//////////////////////////////段落样式//////////////////////////////
    
    // 文本对齐方式
    CTTextAlignment alignment = kCTLeftTextAlignment;
    
    if(self.textAlignment == NSTextAlignmentCenter)
    {
        alignment = kCTCenterTextAlignment;
    }
    
    if(self.textAlignment == NSTextAlignmentRight)
    {
        alignment = kCTRightTextAlignment;
    }
    
    CTParagraphStyleSetting alignmentStyle;
    
    alignmentStyle.spec = kCTParagraphStyleSpecifierAlignment;
    
    alignmentStyle.valueSize = sizeof(alignment);
    
    alignmentStyle.value = &alignment;
    
    // 文本 行间距
    CGFloat lineSpace = self.linesSpacing;
    
    CTParagraphStyleSetting lineSpaceStyle;
    
    lineSpaceStyle.spec = kCTParagraphStyleSpecifierLineSpacingAdjustment;
    
    lineSpaceStyle.valueSize = sizeof(lineSpace);
    
    lineSpaceStyle.value =&lineSpace;
    
    // 文本 段落间距
    CGFloat paragraphSpacing = 5.0;
    
    CTParagraphStyleSetting paragraphSpaceStyle;
    
    paragraphSpaceStyle.spec = kCTParagraphStyleSpecifierAlignment;
    
    paragraphSpaceStyle.valueSize = sizeof(CGFloat);
    
    paragraphSpaceStyle.value = &paragraphSpacing;
    
    // 创建设置数组 （1.对其style  2.行间距style 3.段落间距style）
    CTParagraphStyleSetting settings[] = { alignmentStyle, lineSpaceStyle, paragraphSpaceStyle};
    
    // 创建最终的富文本样式
    CTParagraphStyleRef style = CTParagraphStyleCreate(settings , sizeof(settings));
    
    // 段落样式添加到 富文本属性
    [attStr addAttribute:(id)kCTParagraphStyleAttributeName value:(id)style range:NSMakeRange(0 , [attStr length])];
    
//////////////////////////////确定富文本属性显示的区域//////////////////////////////
    
    //1. 使用 CTFramesetterRef 包装 NSMutableAttributedString
    CTFramesetterRef framesetter = CTFramesetterCreateWithAttributedString((CFAttributedStringRef)attStr);
    
    //2. 创建一个rect区域的路径path
    CGMutablePathRef path = CGPathCreateMutable();
    CGPathAddRect(path,
                  NULL ,
                  CGRectMake(0 , 0 , self.bounds.size.width , self.bounds.size.height));
    
    //3. 根据封闭路径得到frame
    CTFrameRef leftFrame = CTFramesetterCreateFrame(framesetter,
                                                    CFRangeMake(0, attStr.length),
                                                    path ,
                                                    NULL);
    
//////////////////////////////翻转坐标系统//////////////////////////////
    //因为CoreText文本原来是倒的 所以要翻转下
    
    CGContextRef context = UIGraphicsGetCurrentContext();
    
    CGContextSetTextMatrix(context , CGAffineTransformIdentity);
    
    CGContextTranslateCTM(context , 0 , self.bounds.size.height);
    
    CGContextScaleCTM(context, 1.0 , -1.0);
    
    
//////////////////////////////调用CoreGraphics完成最后的绘制//////////////////////////////
    //画出文本
    CTFrameDraw(leftFrame,context);

//////////////////////////////释放系统内存//////////////////////////////
    CGPathRelease(path);
    CFRelease(framesetter);
    CFRelease(helveticaBold);
    
//////////////////////////////提交对画布的绘制修改//////////////////////////////
    
    UIGraphicsPushContext(context);
}

@end
```

ViewController测试

```objc
@implementation CoreTextViewController1

- (void)viewDidLoad {
    [super viewDidLoad];
    
    CoreTextLabel *label = [[CoreTextLabel alloc] init];
    label.text = @"哈师大i大会谁都会啊适度阿哈师大i大会谁都会啊适度阿哈师大i大会谁都会啊适度阿哈师大i大会谁都会啊适度阿哈师大i大会谁都会啊适度阿哈师大i大会谁都会啊适度阿哈师大i大会谁都会啊适度阿辉地阿苏丢阿什顿i";
    label.linesSpacing = 5;
    label.characterSpacing = 3;
    label.frame = CGRectMake(20, 100, 280, 450);
    label.font = [UIFont systemFontOfSize:15.f];
    [self.view addSubview:label];
}

@end
```

最终效果图

![](http://i2.piimg.com/74420b3e7b1d1cfc.png)

####注意点1: 

core text显示出来的字是颠倒的，使用时要翻转下
 
```
CGContextRef context = UIGraphicsGetCurrentContext();
 
CGContextSetTextMatrix(context,CGAffineTransformIdentity);
 
CGContextTranslateCTM(context,0,self.bounds.size.height);
 
CGContextScaleCTM(context,1.0,-1.0);
```
 
####注意点2: 

最后一点要注意的是Mac下的回车和Windows的是不一样的，Windows下的回车是由\r \n组成的而Mac下只有一个\n，所以如果没有去掉的话在每一段的最后都会多出一个空行来，去掉的方法如下：
 
```
NSString *myString = [labelString stringByReplacingOccurrencesOfString:"\r\n" withString:"\n"];
```



####注意点3:

iOS6之后提供的富文本属性Key >>>>  `NS......AttributeName`

****

###最后对比使用TextKit完成上面的文字排版

```objc
@interface BasicViewController () {
    
    //显示的文本
    NSString *_Str;
}

// 最终显示数据的UI
@property (strong, nonatomic) UITextView *txtView;

// 控制文本的显示区域
@property (strong, nonatomic) NSTextContainer *textContainer;

@end

@implementation BasicViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    //初始化数据
    [self setupData];
    
    //创建subviews
    [self initSubviews];
    
}

- (void)setupData {
    _Str = @"属性字符串（NSAttributedString）在CoreText中使用广泛，接下来你可能会经常遇到他。他用于管理字符串和相关的属性集（例如：字体、字间距），这些属性可以被用于单个字符也可以是一段连续的字符串。 他使用一个NSDictionary来管理唯一标示的属性名称，你可以为任一范围的字符指定想要的字符属性。 在iOS6及以后，你可以通过attributedText属性为UILabel、UITextView、UITextField指定想要展示的字符串。通过这种方式我们就可以展示一些带格式的文本了。但是也许你会问如果要兼容iOS5或者更早的版本，要怎么做呢？这个我们在后面的Core Text部分会有解答属性字符串（NSAttributedString）在Core Text中使用广泛，接下来你可能会经常遇到他。";
}

/**
 *  NSTextContainer: 定义了文本可以排版的区域
 *
 *  NSLayoutManager: 该类负责对文字进行编辑排版处理
 *
 *  NSTextStorage: 存储将要被排版的内容
 *
 *  NSAttributedString/NSMutableAttributedString: 支持渲染不同风格的文本
 */

- (void)initSubviews {
    
    //1. 设置排版区域 NSTextContainer
    _textContainer = [[NSTextContainer alloc] initWithSize:CGSizeMake(300, 300)];
    
    //2. 排版布局 NSLayoutManager
    NSLayoutManager *layout = [[NSLayoutManager alloc] init];
    [layout addTextContainer:_textContainer];
    
    //3. 设置排版的内容 NSTextStorage
    NSTextStorage *storage = [[NSTextStorage alloc] initWithString:_Str
                                                        attributes:nil];
    [storage addLayoutManager:layout];
    
    //4. 创建TextView
    CGRect textViewRect = CGRectInset(self.view.bounds, 10.0, 20.0);
    _txtView = [[UITextView alloc] initWithFrame:textViewRect
                                   textContainer:_textContainer];
    
    _txtView.layer.borderWidth = 1;
    _txtView.scrollEnabled = NO;
    _txtView.editable = NO;
    [self.view addSubview:_txtView];
    
    //5. 通知 NSTextStorage 开始排版
    [storage beginEditing];
    
    //6. 将设置文字效果的富文本丢给 NSTextStorage
    NSDictionary *attrsDic = @{NSTextEffectAttributeName: NSTextEffectLetterpressStyle};
    
    NSMutableAttributedString *attrStr = [[NSMutableAttributedString alloc] initWithString:_txtView.text attributes:attrsDic];
    
    [storage setAttributedString:attrStr];
    
    //7. 查找特定的单词，单独设置效果
    [self markWord:@"iOS6" inTextStorage:storage];
    [self markWord:@"CoreText" inTextStorage:storage];
    
    //8. 结束排版，调用CoreText进行最终的渲染字形图片
    [storage endEditing];
}

- (void) markWord:(NSString*)word inTextStorage:(NSTextStorage*)textStorage
{
    //1. regx
    NSRegularExpression *regex = [NSRegularExpression regularExpressionWithPattern:word
                                                                           options:0
                                                                             error:nil];
    
    //2. 查找到单词的位置
    NSArray *matches = [regex matchesInString:_txtView.text
                                      options:0
                                        range:NSMakeRange(0, [_txtView.text length])];
    
    //3. 设置文字效果
    for (NSTextCheckingResult *match in matches) {
        
        NSRange matchRange = [match range];
        
        //storage添加属性字符串设置
        [textStorage addAttribute:NSForegroundColorAttributeName
                            value:[UIColor redColor]
                            range:matchRange];
    }
}

@end
```

效果图如下

![](http://i3.piimg.com/d5b13adfc7675161.png)

可以看出来，比直接使用CoreText处理文字排列、显示效果真的还是方便的很多了...

****


###学习资料来源:

```
http://www.cocoachina.com/industry/20131126/7417.html.
http://www.tuicool.com/articles/jEBrq2B
http://www.tuicool.com/articles/jEBrq2B
```