---
layout: post
title: "TYAttributedLabel源码阅读"
date: 2016-04-25 16:45:21 +0800
comments: true
categories: 
---

由于自己完成一个TXT小说阅读的App过程中，使用到了这个框架的文字排版功能，但这个框架中有一些代码写的比较蛋疼，但是这个框架主体结构还是很不错的，所以决定看看这个框架代码自己再改改，如下图是最后修改框架并加上了一些UI的效果.

![CoreTextTxt.gif](http://imgchr.com/images/CoreTextTxt.gif)

[![CoreText.gif](http://imgchr.com/images/CoreText.gif)](http://imgchr.com/image/ADF)

所以我想达到的目标是:

- 仍然使用他的TextContainer、以及各种Storage完成TXT小说字符串的解析工作
- 但是TYAttributedLabel显然不太适合我的逻辑，他只对图片、Link有点击操作，对普通文字没有，so需要改进TYAttributedLabel的代码

过程中，因为从来没接触过CoreText，刚开始真的不知道从何入手修改，花了几天的时间看懂CoreText基础后，然后通读了一遍其源码终于可以修改了....

###TYAttributedLabel封装了下CoreText的一些api的常用操作，其结构组成:

- TYAttributedLabel（继承自UIView） >>> 顶层提供给外界使用的

- TYTextContainer >>> 具体调用CoreText的相关api完成文字排版处理，其文字偶排版的参数来自于如下几个Storage（数据存储器）

- 多种Storage >>> 负责存储要处理的文本数据
	- 处理什么文字、什么图片、什么链接...
	- 哪一个range、多大 ...

如上的结构有一点类似iOS7提出的TextKit的结构.


[![TextKitd2b08.png](http://imgchr.com/images/TextKitd2b08.png)]

[![TextKit.png](http://imgchr.com/images/TextKit.png)](http://imgchr.com/image/AQb)

[![TextKitMVC.png](http://imgchr.com/images/TextKitMVC.png)](http://imgchr.com/image/AQ9)

TextKit提供的Api类也是遵循MVC的结构，而TYAttributedLabel也是模仿了TextKit的体系.

***

###TYTextContainer的大致结构

- 毕竟不是苹果官方的东西，所以与TextKit体系肯定有差距的

- TYTextContainer依赖多个Storage，来告诉TYTextContainer
	- 要解析什么东西
	- 按照什么参数解析

- 要解析什么东西 And 按照什么参数解析 >>> 使用对应的Storage类对象来保存
	- TextStorage 
	- LinkStorage
	- DrawStorage
	- ImageStorage
	- ViewStorage

***

###接下来具体看看TYTextContainer.h告诉外部能够做什么

第一部分该类本身的声明，包含两部分:

- 解析富文本时的一些默认的排版参数
- 解析完后获取文本的一些结果参数值

```objc
@interface RDTextContainer : NSObject

//普通文本字符串
@property (nonatomic, strong)   NSString                        *text;

//带有属性的字符串
@property (nonatomic, strong)   NSAttributedString              *attributedText;

//行数
@property (nonatomic, assign)   NSInteger                       numberOfLines;

//文字颜色
@property (nonatomic, strong)   UIColor                         *textColor;

//链接颜色
@property (nonatomic, strong)   UIColor                         *linkColor;

//文字大小
@property (nonatomic, strong)   UIFont                          *font;

//字距
@property (nonatomic, assign)   unichar                         characterSpacing;

//行距
@property (nonatomic, assign)   CGFloat                         linesSpacing;

//文本对齐方式 kCTTextAlignmentLeft
@property (nonatomic, assign)   CTTextAlignment                 textAlignment;

//换行模式 kCTLineBreakByCharWrapping
@property (nonatomic, assign)   CTLineBreakMode                 lineBreakMode;

//调用 createTextContainerWithContentSize方法之后, have value
@property (nonatomic, assign, readonly) CGFloat                 textHeight;

//调用 createTextContainerWithContentSize方法之后, have value
@property (nonatomic, strong, readonly) NSArray                 *textStorages;

/**
 *  给定宽度，计算CTFrame
 */
- (instancetype)createTextContainerWithTextWidth:(CGFloat)textWidth;

/**
 *  给定大小，计算CTFrame
 */
- (instancetype)createTextContainerWithContentSize:(CGSize)contentSize;

/**
 *  解析传入的文本字符串成属性字符串
 */
- (NSAttributedString *)createAttributedString;

/**
 *  解析玩富文本字符产之后，再获取文本高度
 */
- (int)getHeightWithFramesetter:(CTFramesetterRef)framesetter width:(CGFloat)width;

@end
```

第二部分、是一个扩展分类，提供快捷方法，向当前TextContainer`添加`各种Storage对象

```objc
@interface RDTextContainer (AddStorage)

/**
 *  添加单个Storage对象
 */
- (void)addTextStorage:(id<RDTextStorageProtocol>)textStorage;

/**
 *  添加多个Storage对象
 */
- (void)addTextStorageArray:(NSArray <id<RDTextStorageProtocol>>*)textStorageArray;

@end
```

第三部分、是一个扩展分类，提供快捷方法，向当天当前TextContainer`追加`NSString、NSAttributedString、以及使用TextStorage对象存储的NSString、NSAttributedString.

```objc
@interface RDTextContainer (AppendText)

/**
 *  追加(添加到最后) 普通文本
 */
- (void)appendText:(NSString *)text;

/**
 *  追加(添加到最后) 属性文本
 */
- (void)appendTextAttributedString: (NSAttributedString *)attributedText;

/**
 *  追加单个textStorage到末尾
 */
- (void)appendTextStorage:(id<RDAppendTextStorageProtocol>)textStorage;

/**
 *  追加多个textStorage 数组到末尾
 */
- (void)appendTextStorageArray:(NSArray <id<RDAppendTextStorageProtocol>>*)textStorageArray;

@end
```

第四部分、是一个扩展分类，提供快捷方法，向当天当前TextContainer`添加`下划线

```objc
/**
 *  包含两种: 
    第一种，直接添加下划线的内容字符串
    第二种，添加带有下划线内容字符串的Storage
 */
@interface RDTextContainer (AddLink)

/**
 *  添加 带有链接的内容，其中linkData为链接link对应的数据，点击link时候获取使用
 *  eg、[container appendLinkWithText:@"百度一下" linkFont:[UIFont systemFontOfSize:15+arc4random()%4] linkData:@"http://www.baidu.com"];
 */
- (void)appendLinkWithText:(NSString *)linkText
                  linkFont:(UIFont *)linkFont
                  linkData:(id)linkData;

/**
 *  添加 带有链接的内容
 */
- (void)appendLinkWithText:(NSString *)linkText
                  linkFont:(UIFont *)linkFont
                 linkColor:(UIColor *)linkColor
                  linkData:(id)linkData;

/**
 *  添加 带有链接的内容
 *
 *  @param linkText         链接文本
 *  @param linkData         链接携带的数据
 *  @param underLineStyle   下划线样式（无，单 双) 默认单
 */
- (void)appendLinkWithText:(NSString *)linkText
                  linkFont:(UIFont *)linkFont
                 linkColor:(UIColor *)linkColor
            underLineStyle:(CTUnderlineStyle)underLineStyle
                  linkData:(id)linkData;


/**
 *  添加 带有链接内容的 LinkTextStorage
 */
- (void)addLinkWithLinkStorage:(id<RDLinkStorageProtocol>)storage
                         range:(NSRange )range;

/**
 *  添加 带有链接内容的 LinkTextStorage，带下划线颜色
 */
- (void)addLinkWithLinkStorage:(id<RDLinkStorageProtocol>)storage
                     linkColor:(UIColor *)linkColor
                         range:(NSRange )range;

/**
 *  添加 带有链接内容的 LinkTextStorage，指定下划线样式
 *
 *  @param linkData         链接携带的数据
 *  @param linkColor        链接颜色
 *  @param underLineStyle   下划线样式（无，单 双) 默认单
 *  @param range            范围
 */
- (void)addLinkWithLinkStorage:(id<RDLinkStorageProtocol>)storage
                     linkColor:(UIColor *)linkColor
                underLineStyle:(CTUnderlineStyle)underLineStyle
                         range:(NSRange )range;

@end
```

第五部分、是一个扩展分类，提供快捷方法，向当天当前TextContainer`添加`图片

```objc
@interface RDTextContainer (AddUIImage)

/**
 *  添加 image
 */
- (void)addImage:(UIImage *)image
           range:(NSRange)range;

/**
 *  添加 image，指定渲染的size大小
 */
- (void)addImage:(UIImage *)image
           range:(NSRange)range
            size:(CGSize)size;

/**
 *  添加 imageStorage image数据，指定渲染大小，指定图片对齐方式
 */
- (void)addImage:(UIImage *)image
           range:(NSRange)range
            size:(CGSize)size
       alignment: (RDDrawAlignment)alignment;

/**
 *  添加图片以图片名的形式
 */
- (void)addImageWithName:(NSString *)imageName
                   range:(NSRange)range;

/**
 *  添加图片以图片名的形式
 */
- (void)addImageWithName:(NSString *)imageName
                   range:(NSRange)range
                    size:(CGSize)size;

/**
 *  添加图片以图片名的形式
 */
- (void)addImageWithName:(NSString *)imageName
                   range:(NSRange)range
                    size:(CGSize)size
               alignment:(RDDrawAlignment)alignment;

@end
```

第六部分、是一个扩展分类，提供快捷方法，向当天当前TextContainer`追加`图片，是我从上面的分类中剥离出来的，因为不是属于添加，而是追加

```objc
@interface RDTextContainer (AppendImage)

/**
 *  追加 image
 */
- (void)appendImage:(UIImage *)image;

/**
 *  追加 image
 */
- (void)appendImage:(UIImage *)image
               size:(CGSize)size;

/**
 *  追加 image，指定对齐方式
 */
- (void)appendImage:(UIImage *)image
               size:(CGSize)size
          alignment:(RDDrawAlignment)alignment;

/**
 *  追加 image 以图片名的形式
 */
- (void)appendImageWithName:(NSString *)imageName;

/**
 *  追加 image 以图片名的形式
 */
- (void)appendImageWithName:(NSString *)imageName
                       size:(CGSize)size;

/**
 * 追加 image 以图片名的形式，指定对齐方式
 */
- (void)appendImageWithName:(NSString *)imageName
                       size:(CGSize)size
                  alignment:(RDDrawAlignment)alignment;

@end
```

最后一部分，是添加或追加UIView到TextContainer的

```objc
@interface RDTextContainer (AddUIView)

/**
 *  添加 viewStorage (添加 UI控件 需要设置frame)
 */
- (void)addView:(UIView *)view range:(NSRange)range;

/**
 *  添加 viewStorage (添加 UI控件 需要设置frame)
 */
- (void)addView:(UIView *)view
          range:(NSRange)range
      alignment:(RDDrawAlignment)alignment;

@end
```

```objc
@interface RDTextContainer (AppendUIView)

/**
 *  追加 viewStorage (添加 UI控件 需要设置frame)
 */
- (void)appendView:(UIView *)view;

/**
 *  追加 viewStorage (添加 UI控件 需要设置frame)
 */
- (void)appendView:(UIView *)view alignment:(RDDrawAlignment)alignment;

@end
```

###下面是看看TYTextContainer.m的实现代码

以下面最简单的代码断点进入里面看代码实现

```objc
NSString *text = @"大话王段位地海文化低奥委会的下午好i大话王段位地海文化低奥委会的下午好i大话王段位地海文化低奥委会的下午好i大话王段位地海文化低奥委会的下午好i大话王段位地海文化低奥委会的下午好i大话王段位地海文化低奥委会的下午好i大话王段位地海文化低奥委会的下午好i大话王段位地海文化低奥委会的下午好i大话王段位地海文化低奥委会的下午好i大话王段位地海文化低奥委会的下午好i大话王段位地海文化低奥委会的下午好i大话王段位地海文化低奥委会的下午好i大话王段位地海文化低奥委会的下午好i大话王段位地海文化低奥委会的下午好i大话王段位地海文化低奥委会的下午好i大话王段位地海文化低奥委会的下午好i大话王段位地海文化低奥委会的下午好i";
    
RDTextContainer *container = [RDTextContainer new];
container.font = [UIFont systemFontOfSize:15.f];
container.text = [text copy];
NSLog(@"完成解析后的富文本 = %@", container.attributedText);
    
NSAttributedString *attributeString = [container createAttributedString];
NSLog(@"%@", attributeString);
```

TYTextContainer的init主要是做了一些初始化参数

```objc
- (instancetype)init
{
    if (self = [super init]) {
        [self setupProperty];
    }
    return self;
}
```

```objc
- (void)setupProperty
{
    _font = [UIFont systemFontOfSize:15];
    _characterSpacing = 1;
    _linesSpacing = 2;
    _textAlignment = kCTLeftTextAlignment;
    _lineBreakMode = kCTLineBreakByCharWrapping;
    _textColor = kTextColor;
    _linkColor = kLinkColor;
    _replaceStringNum = 0;
}
```

一些属性的setter重写

```objc
- (void)setText:(NSString *)text
{
	//1. 完成传入的NSString解析成NSMutableAttributedString属性字符串
    _attString = [self createTextAttibuteStringWithText:text];
    
    //2. 重置前一次保存的显示文字的各种数据
    [self resetAllAttributed];
    [self resetFrameRef];
}
```

```objc
- (void)setAttributedText:(NSAttributedString *)attributedText
{
	// 1. 对传入的属性文本进行预处理
    if (attributedText == nil) {
    	//空处理
        _attString = [[NSMutableAttributedString alloc] init];
    }else if ([attributedText isKindOfClass:[NSMutableAttributedString class]]) {
    	//类型转换
        _attString = (NSMutableAttributedString *)attributedText;
    }else {
		//类型转换
        _attString = [[NSMutableAttributedString alloc]initWithAttributedString:attributedText];
    }
    
	//2. 重置前一次保存的显示文字的各种数据
    [self resetAllAttributed];
    [self resetFrameRef];
}
```

```objc
- (void)setTextColor:(UIColor *)textColor
{
	//做了一次重复颜色设置过滤
    if (textColor && _textColor != textColor){
        _textColor = textColor;
        
        //针对整个文本全部长度设置颜色
        [_attString addAttributeTextColor:textColor];
        
        //重新计算CTFrame
        [self resetFrameRef];
    }
}
```

```objc
- (void)setFont:(UIFont *)font
{
    if (font && _font != font){
        _font = font;
        
        //针对整个文本全部长度设置Font
        [_attString addAttributeFont:font];
   	
	   	//重新计算CTFrame     
        [self resetFrameRef];
    }
}
```

```objc
- (void)setCharacterSpacing:(unichar)characterSpacing
{
    if (characterSpacing >= 0 && _characterSpacing != characterSpacing) {
        _characterSpacing = characterSpacing;
        
        //针对整个文本全部长度
        [_attString addAttributeCharacterSpacing:characterSpacing];
        
        [self resetFrameRef];
    }
}

- (void)setLinesSpacing:(CGFloat)linesSpacing
{
    if (_linesSpacing != linesSpacing) {
        _linesSpacing = linesSpacing;
        
        //针对整个文本全部长度
        [_attString addAttributeAlignmentStyle:_textAlignment lineSpaceStyle:linesSpacing lineBreakStyle:_lineBreakMode];
        
        [self resetFrameRef];
    }
}

- (void)setTextAlignment:(CTTextAlignment)textAlignment
{
    if (_textAlignment != textAlignment) {
        _textAlignment = textAlignment;
        
        //针对整个文本全部长度
        [_attString addAttributeAlignmentStyle:textAlignment lineSpaceStyle:_linesSpacing lineBreakStyle:_lineBreakMode];
        
        [self resetFrameRef];
    }
}

- (void)setLineBreakMode:(CTLineBreakMode)lineBreakMode
{
    if (_lineBreakMode != lineBreakMode) {
        _lineBreakMode = lineBreakMode;
        if (_lineBreakMode == kCTLineBreakByTruncatingTail)
        {
            lineBreakMode = _numberOfLines == 1 ? kCTLineBreakByCharWrapping : kCTLineBreakByWordWrapping;
        }
        
        //针对整个文本全部长度
        [_attString addAttributeAlignmentStyle:_textAlignment lineSpaceStyle:_linesSpacing lineBreakStyle:lineBreakMode];
        
        [self resetFrameRef];
        
    }
}
```

然后到`[container createAttributedString]`完成是否有注册的Storage进行解析

```objc
- (NSAttributedString *)createAttributedString
{
    //1. 使用TextContainer内部注册的Storage处理富文本
    [self addTextStoragesWithAtrributedString:_attString];
    
    //2. 空处理
    if (_attString == nil) {
        _attString = [[NSMutableAttributedString alloc] init];
    }
    
    //3. 返回一个浅拷贝
    return [_attString copy];
}
```

OK，最简单的解析情况就涉及这么多代码，那么下面来个稍微复杂点的额.

****

###从TYAttributedLabel中抠了一些比较核心的代码完成如下的效果

[![TYAttributedLabel2b8bf.md.png](http://imgchr.com/images/TYAttributedLabel2b8bf.md.png)](http://imgchr.com/image/AQa)

使用TYAttributedLabel代码做出如上效果的ViewController代码

```objc
@implementation ViewController 

- (void)viewDidLoad {
    [super viewDidLoad];
    
    RDTestCoreTextView *view = [RDTestCoreTextView new];
    view.backgroundColor = [UIColor lightGrayColor];
    view.x = 10;
    view.y = 70;
    view.width = self.view.width - 20;
    view.height = self.view.height - 120;
    [self.view addSubview:view];
}

@end
```

如下代码是从TYAttributedLabel中扣出来的代码


```objc
#import "RDTestCoreTextView.h"

#import "RDTextContainer.h"
#import "RDTextStorage.h"
#import "RDImageStorage.h"
#import "RDLinkTextStorage.h"

#import "RegexKitLite.h"

static NSString *const kXEllipsesCharacter = @"\u2026";

@interface RDTextContainer ()

//遍历所有的RDDrawStorageProtocol
- (void)enumerateDrawRectDictionaryUsingBlock:(void (^)(id<RDDrawStorageProtocol> drawStorage, CGRect rect))block;

//遍历所有的RDTextStorageProtocol
- (BOOL)enumerateRunRectContainPoint:(CGPoint)point
                          viewHeight:(CGFloat)viewHeight
                        successBlock:(void (^)(id<RDTextStorageProtocol> textStorage))successBlock;

//遍历所有的RDLinkStorageProtocol
- (BOOL)enumerateLinkRectContainPoint:(CGPoint)point
                           viewHeight:(CGFloat)viewHeight
                         successBlock:(void (^)(id<RDLinkStorageProtocol> linkStorage))successBlock;

@end

@implementation RDTestCoreTextView {
    RDTextContainer *_container;
}

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        _container = [RDTextContainer new];
    }
    return self;
}

- (void)drawRect:(CGRect)rect {
    
    CGContextRef context = UIGraphicsGetCurrentContext();
    CGContextSetTextMatrix(context, CGAffineTransformIdentity);
    CGContextTranslateCTM(context, 0, self.bounds.size.height);
    CGContextScaleCTM(context, 1, -1);
    
    NSString *text = @"标题\n \
    Start位地海文[test,60,80]化低奥委会的下午好i大话王段位[test,100,80]地海文化低奥委会的下午文化低奥委会的下午好i大[test,120,60]段位地海化低奥委会的下午好i大话王段位化低奥委会的下午好i大话王段位化低奥委会的下午好i大话王段位化低奥委会的下午好i大话王段位化低奥委会的下午好i大话王段位化低奥委会的下午好i大话王段位化低奥委会的下午好i大话王段位化低奥委会的[test,40,40]下午好End";
    
    _container.font = [UIFont systemFontOfSize:15.f];
    _container.text = [text copy];
    
    __block NSMutableArray *storages = [NSMutableArray array];
    
    RDTextStorage *titleTextStorage = [[RDTextStorage alloc] init];
    titleTextStorage.range = NSMakeRange([text rangeOfString:@"标题"].location , 2);
    titleTextStorage.font = [UIFont fontWithName:@"Courier" size:17.f];
    titleTextStorage.textColor = [UIColor redColor];
    titleTextStorage.underLineStyle = kCTUnderlineStyleSingle;
    titleTextStorage.modifier = kCTUnderlinePatternSolid;
    [storages addObject:titleTextStorage];
    
    // 图片处理Storage，使用正则式匹配出字符串中的[图片名, 宽度, 高度]这样的range
    [text enumerateStringsMatchedByRegex:@"\\[(\\w+?),(\\d+?),(\\d+?)\\]" usingBlock:^(NSInteger captureCount, NSString *const __unsafe_unretained *capturedStrings, const NSRange *capturedRanges, volatile BOOL *const stop)
     {
         //一个图片信息由三部分组成 , [图片名, 宽度, 高度]
         if (captureCount > 3) {
             
             // 图片信息储存
             RDImageStorage *imageStorage = [[RDImageStorage alloc]init];
             
             // 图片名
             imageStorage.imageName = capturedStrings[1];
             
             // 图片渲染的尺寸
             imageStorage.size = CGSizeMake([capturedStrings[2]intValue], [capturedStrings[3]intValue]);
             
             // 记录图片字符串出现的位置
             imageStorage.range = capturedRanges[0];
             
             // 添加到container
             [storages addObject:imageStorage];
         }
     }];
    
    // 将所有的包含待处理文本的storage添加到TextContainer
    [_container addTextStorageArray:storages];
    
    // TextContainer根据内部的所有的storage解析程序属性富文本
    NSAttributedString *attributeString = [_container createAttributedString];
    
    // TextContainer 根据传入的size，将内部属性富文本计算出 CTFrameRef
    [_container createTextContainerWithContentSize:self.bounds.size];
    
    // 遍历绘制CTFrame中的每一个CTRunRef，完成基本的绘制
    [self drawText:_container.attributedText
             frame:_container.frameRef
              rect:rect
           context:context];
    
    // 通知处理比如ImageStorage绘制图片
    [self drawTextStorage];
}

- (void)drawText:(NSAttributedString *)attributedString
                frame:(CTFrameRef)frame
                 rect: (CGRect)rect
              context: (CGContextRef)context
{
    if (_container.numberOfLines > 0)
    {
        CFArrayRef lines = CTFrameGetLines(frame);
        NSInteger numberOfLines = MIN(_container.numberOfLines, CFArrayGetCount(lines));
        
        CGPoint lineOrigins[numberOfLines];
        CTFrameGetLineOrigins(frame, CFRangeMake(0, numberOfLines), lineOrigins);
        
        BOOL truncateLastLine = (_container.lineBreakMode == kCTLineBreakByTruncatingTail);
        
        for (CFIndex lineIndex = 0; lineIndex < numberOfLines; lineIndex++)
        {
            CGPoint lineOrigin = lineOrigins[lineIndex];
            CGContextSetTextPosition(context, lineOrigin.x, lineOrigin.y);
            CTLineRef line = CFArrayGetValueAtIndex(lines, lineIndex);
            
            BOOL shouldDrawLine = YES;
            if (lineIndex == numberOfLines - 1 && truncateLastLine)
            {
                // Does the last line need truncation?
                CFRange lastLineRange = CTLineGetStringRange(line);
                if (lastLineRange.location + lastLineRange.length < attributedString.length)
                {
                    CTLineTruncationType truncationType = kCTLineTruncationEnd;
                    NSUInteger truncationAttributePosition = lastLineRange.location + lastLineRange.length - 1;
                    
                    NSDictionary *tokenAttributes = [attributedString attributesAtIndex:truncationAttributePosition effectiveRange:NULL];
                    NSAttributedString *tokenString = [[NSAttributedString alloc] initWithString:kXEllipsesCharacter attributes:tokenAttributes];
                    CTLineRef truncationToken = CTLineCreateWithAttributedString((CFAttributedStringRef)tokenString);
                    
                    NSMutableAttributedString *truncationString = [[attributedString attributedSubstringFromRange:NSMakeRange(lastLineRange.location, lastLineRange.length)] mutableCopy];
                    
                    if (lastLineRange.length > 0)
                    {
                        // Remove last token
                        [truncationString deleteCharactersInRange:NSMakeRange(lastLineRange.length - 1, 1)];
                    }
                    [truncationString appendAttributedString:tokenString];
                    
                    
                    CTLineRef truncationLine = CTLineCreateWithAttributedString((CFAttributedStringRef)truncationString);
                    CTLineRef truncatedLine = CTLineCreateTruncatedLine(truncationLine, rect.size.width, truncationType, truncationToken);
                    if (!truncatedLine)
                    {
                        // If the line is not as wide as the truncationToken, truncatedLine is NULL
                        truncatedLine = CFRetain(truncationToken);
                    }
                    CFRelease(truncationLine);
                    CFRelease(truncationToken);
                    CTLineDraw(truncatedLine, context);
                    CFRelease(truncatedLine);
                    
                    shouldDrawLine = NO;
                }
            }
            if(shouldDrawLine)
            {
                CTLineDraw(line, context);
            }
        }
    }
    else
    {
        CTFrameDraw(frame,context);
    }
}

- (void)drawTextStorage
{

    [_container enumerateDrawRectDictionaryUsingBlock:^(id<RDDrawStorageProtocol> drawStorage, CGRect rect) {
        
        if ([drawStorage conformsToProtocol:@protocol(RDViewStorageProtocol) ]) {
            [(id<RDViewStorageProtocol>)drawStorage setOwnerView:self];
        }
        
        rect = UIEdgeInsetsInsetRect(rect,drawStorage.margin);
        
        [drawStorage drawStorageWithRect:rect];
    }];
    
//    if ([_textContainer existRunRectDictionary]) {
//        if (_delegateFlags.textStorageClickedAtPoint) {
//            [self addSingleTapGesture];
//        }else {
//            [self removeSingleTapGesture];
//        }
//        if (_delegateFlags.textStorageLongPressedOnStateAtPoint) {
//            [self addLongPressGesture];
//        }else {
//            [self removeLongPressGesture];
//        }
//    }else {
//        [self removeSingleTapGesture];
//        [self removeLongPressGesture];
//    }
}

- (void)addSingleTapGesture
{
//    if (_singleTapGuesture == nil) {
//        // 单指单击
//        _singleTapGuesture = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(singleTap:)];
//        
//        //不会走单击回调，会走UIGestureRecognizerDelegate回调函数
//        //        _singleTapGuesture.delegate = self;
//        
//        // 增加事件者响应者
//        [self addGestureRecognizer:_singleTapGuesture];
//    }
}

- (void)removeSingleTapGesture
{
//    if (_singleTapGuesture) {
//        [self removeGestureRecognizer:_singleTapGuesture];
//        _singleTapGuesture = nil;
//    }
}

- (void)addLongPressGesture
{
//    if (_longPressGuesture == nil) {
//        // 长按
//        _longPressGuesture = [[UILongPressGestureRecognizer alloc]initWithTarget:self action:@selector(longPress:)];
//        [self addGestureRecognizer:_longPressGuesture];
//    }
}

- (void)removeLongPressGesture
{
//    if (_longPressGuesture) {
//        [self removeGestureRecognizer:_longPressGuesture];
//        _longPressGuesture = nil;
//    }
}


@end
```

OK，可以顺着如上的代码来看看TYTextContainer、以及所有的Storage的具体实现了。so 这是一个完整看清楚一个开源框架实现细节的技巧，一定要整出一个走遍其内部实现的demo，然后不断的断点断点断点...

可以看到TextContainer基本上是用来注册其他各种类型的Storage对象，然后通过这些Storage对象持有数据进行富文本解析:

- range 
- font
- textColor
- underline style
- link 
- image


所以，先来看看这些各种的Storage.

****

###首先是Storage的协议抽象

TextStorage的抽象

```objc
@protocol RDTextStorageProtocol <NSObject>
@required

/* 范围（如果是appendStorage,range只针对追加的文本） */
@property (nonatomic,assign) NSRange range;

/* 文本中实际位置,因为某些文本被替换，会导致位置偏移 */
@property (nonatomic,assign) NSRange realRange;

/* 给传入的attributedString添加富文本属性: 字体、颜色、下划线 */
- (void)addTextStorageWithAttributedString:(NSMutableAttributedString *)attributedString;

@end
```

Append TextStorage 功能的抽象 继承自 上面的协议

```objc
@protocol RDAppendTextStorageProtocol <RDTextStorageProtocol>

@required

/* 不断的追加NSAttributedString属性到TextStorage保存的属性字符串中 */
- (NSAttributedString *)appendTextStorageAttributedString;

@end
```

Link Storage的抽象

```objc
@protocol RDLinkStorageProtocol <RDAppendTextStorageProtocol>

// 文本颜色
@property (nonatomic, strong) UIColor   *textColor;

@end
```

绘制图像的Storage抽象


```objc
@protocol RDDrawStorageProtocol <RDAppendTextStorageProtocol>

// 四周间距
@property (nonatomic, assign)   UIEdgeInsets    margin;

/* 添加View 或 绘画 到该区域 */
- (void)drawStorageWithRect:(CGRect)rect;

/* 设置字体高度 上下行高度 */
- (void)setTextfontAscent:(CGFloat)ascent descent:(CGFloat)descent;

/* 设置当前替换字符数 */
- (void)setCurrentReplacedStringNum:(NSInteger)replacedStringNum;

@end
```

添加UIView的Storage抽象


```objc
@protocol RDViewStorageProtocol <NSObject>

/* 设置所属的view */
- (void)setOwnerView:(UIView *)ownerView;

/* 设置不会绘画出来的View */
- (void)didNotDrawRun;

@end
```

小结如上协议的继承结构:

- RDTextStorageProtocol
	- RDAppendTextStorageProtocol
		- RDLinkStorageProtocol
		- RDDrawStorageProtocol
- RDViewStorageProtocol

***

###TextStorage

声明文件、TextStorage实现的是 `RDAppendTextStorageProtocol`协议(继承自RDTextStorageProtocol协议)

```objc
@interface RDTextStorage : NSObject <RDAppendTextStorageProtocol>

///////////////////////////////RDTextStorageProtocol/////////////////////////////
@property (nonatomic, assign)   NSRange     range;          // 记录对原始字符串处理的范围
@property (nonatomic, assign)   NSRange     realRange;      // label文本中实际位置,因为某些文本被替换，会导致位置偏移

///////////////////////////////RDTextStorage自己的属性/////////////////////////////

@property (nonatomic, assign)   NSInteger   tag;            // 标识
@property (nonatomic, strong)   NSString    *text;          // 要处理的原始文字
@property (nonatomic, strong)   UIColor     *textColor;     // 文本颜色
@property (nonatomic, strong)   UIFont      *font;          // 字体

@property (nonatomic, assign)   CTUnderlineStyle            underLineStyle;     //链接下划线样式，默认没有
@property (nonatomic, assign)   CTUnderlineStyleModifiers   modifier;           // 下划线样式 （点 线）（默认线）

@end
```

TextStorage实现部分，主要是实现RDAppendTextStorageProtocol协议与RDTextStorageProtocol协议

```objc
@implementation RDTextStorage

#pragma mark - RDTextStorageProtocol协议实现

- (void)addTextStorageWithAttributedString:(NSMutableAttributedString *)attributedString {
    
    /**
     *  将当前TextStorage保存的字体、字体颜色、下划线样式，设置给传入的attributedString
     */
         
    // 颜色
    if (_textColor) {
        [attributedString addAttributeTextColor:_textColor range:_range];
    }
    // 字体
    if (_font) {
        [attributedString addAttributeFont:_font range:_range];
    }
    
    // 下划线
    if (_underLineStyle) {
        [attributedString addAttributeUnderlineStyle:_underLineStyle modifier:_modifier range:_range];
    }
}

#pragma mark - RDAppendTextStorageProtocol协议实现

- (NSAttributedString *)appendTextStorageAttributedString {
    
    /** 
        当修改了当前TextStorage的 range属性值、text属性值、textColor属性值、font属性值等等时，
        调用此方法完成富文本属性的继续添加
     */
    
    //1. 原始字符串 >>> 富文本字符串
    NSMutableAttributedString *attributedString = [[NSMutableAttributedString alloc] initWithString:_text];
    
    //2. 修正 _range保存为当前
    if (NSEqualRanges(_range, NSMakeRange(0, 0))) {
        _range = NSMakeRange(0, attributedString.length);
    }
    
    //3. 调用RDTextStorageProtocol协议方法，完成富文本属性添加
    [(id<RDTextStorageProtocol>)self addTextStorageWithAttributedString:attributedString];
    
    return [attributedString copy];
}

@end
```

可以看出来，这个TextStorage主要是针对如下富文本属性:

- 字体大小、字体名、字体颜色
- 文字下划线
- 当修改了range、text、font、textColor时，重新设置如上富文本属性

***

###LinkTextStorage

- 继承自 TextStorage
- 实现RDLinkStorageProtocol协议

```objc
@interface RDLinkTextStorage : RDTextStorage <RDLinkStorageProtocol>

///////////////////////////////RDTextStorage自己的属性/////////////////////////////

// 链接携带的数据（如:点击文字之后跳转到 http://www.baidu.com 网页）
@property (nonatomic, strong) id                linkData;

@end
```

```objc
#import "RDLinkTextStorage.h"
#import "RDRichTextHeader.h"

@implementation RDLinkTextStorage

- (instancetype)init
{
    if (self = [super init]) {
        
        // 初始化下划线样式
        self.underLineStyle = kCTUnderlineStyleSingle;
        self.modifier = kCTUnderlinePatternSolid;
    }
    return self;
}

#pragma mark - 重写 RDTextStorageProtocol协议方法，完成链接Link文本内容的属性添加

- (void)addTextStorageWithAttributedString:(NSMutableAttributedString *)attributedString
{
    //1. 调用TextStorage父类方法，完成富文本属性添加
    [super addTextStorageWithAttributedString:attributedString];
    
    //2. 将当前Storage对象 保存到 富文本字符串对range，使用kRDTextRunAttributedName为key
    //（当点击点击的时候，取出self.linkData完成点击操作）
    [attributedString addAttribute:kRDTextRunAttributedName
                             value:self
                             range:self.range];
    
    //3. 保存原始链接Text文本
    self.text = [attributedString.string substringWithRange:self.range];
    
}


@end
```

代码也很少，主要是针对链接文本数据的处理，比如:

```
你滴爱我的爱我到位到我家
大师短时我东海湾嗲我
go to webSite 
```

点击如上的`go to webSite`之后跳转 `www.baidu.com`，对应代码实现如下

```objc
- (void)test {
    
    RDLinkTextStorage *storage = [RDLinkTextStorage new];
    
    // TextStorage参数
    storage.font = [UIFont systemFontOfSize:17.f];
    storage.textColor = [UIColor redColor];
    
    // LinkTextStorage参数
    storage.linkData = @"www.baidu.com";
    
    NSString *text = @"你滴爱我的爱我到位到我家\n  \
                    大师短时我东海湾嗲我\n \
                    go to webSite ";
    
    NSMutableAttributedString *attr = [[NSMutableAttributedString alloc] initWithString:text];
    
    [storage addTextStorageWithAttributedString:attr];
}
```

***

###DrawStorage

- 继承自NSObject
- 实现RDDrawStorageProtocol协议
	- RDAppendTextStorageProtocol
		- RDTextStorageProtocol

```objc
@interface RDDrawStorage : NSObject <RDDrawStorageProtocol>

///////////////////////////////RDTextStorageProtocol/////////////////////////////
@property (nonatomic, assign)   NSRange         range;          // 原始字符串文本范围
@property (nonatomic, assign)   NSRange         realRange;      // 插入富文本属性之后的字符串范围

///////////////////////////////RDDrawStorageProtocol/////////////////////////////
@property (nonatomic, assign)   UIEdgeInsets    margin;         // 图片四周间距

//////////////////////////////////////自己的参数///////////////////////////////////
@property (nonatomic, assign)   NSInteger       tag;            // 标识
@property (nonatomic, assign)   CGSize          size;           // 绘画物大小
@property (nonatomic, assign)   RDDrawAlignment drawAlignment;  // 对齐方式

/* 获取绘画区域`上行`高度(默认实现) */
- (CGFloat)getDrawRunAscentHeight;

/* 获取绘画区域`下行`高度 默认实现为0（一般不需要改写） */
- (CGFloat)getDrawRunDescentHeight;

/* 获取绘画区域宽度（默认实现） */
- (CGFloat)getDrawRunWidth;

@end
```

```objc
/**
 *  替换掉富文本中某些位置使用的 占位符
 */
NSString *p_spaceReplaceString()
{
    unichar objectReplacementChar           = 0xFFFC;
    NSString *objectReplacementString       = [NSString stringWithCharacters:&objectReplacementChar length:1];
    return objectReplacementString;
}

/**
 *  两种带有图片占位符类型range进行修正
 */
NSRange p_fixRange(NSRange range ,NSInteger replaceStringNum)
{
    NSRange fixRange = range;
    if (range.length <= 1 || replaceStringNum < 0)
        return fixRange;
    
    NSInteger location = range.location - replaceStringNum;
    NSInteger length = range.length - replaceStringNum;
    
    if (location < 0 && length > 0) {
        fixRange = NSMakeRange(range.location, length);
    }else if (location < 0 && length <= 0){
        fixRange = NSMakeRange(NSNotFound, 0);
    }else {
        fixRange = NSMakeRange(range.location - replaceStringNum, range.length);
    }
    return fixRange;
}

@interface RDDrawStorage () {
    CGFloat         _fontAscent;       
    CGFloat         _fontDescent;
    NSRange         _fixRange;
}

@end

@implementation RDDrawStorage

#pragma mark - RDTextStorageProtocol 协议实现

- (void)addTextStorageWithAttributedString:(NSMutableAttributedString *)attributedString
{
    // 要修正的范围
    NSRange range = _fixRange;
    
    // 对需要修正的range进行处理
    if (range.location == NSNotFound) {
        return;
    }else {

        // 将要绘制图像的range内容，使用 占位符0xFFFC 替换
        [attributedString replaceCharactersInRange:range withString:p_spaceReplaceString()];
        
        // 修正range，第一个被替换成0xFFFC的位置，长度为1
        range = NSMakeRange(range.location, 1);
        
        // 保存修正的range
        _realRange = range;
    }
    
    // 设置合适的对齐
    [self p_setAppropriateAlignment];
    
    // 给富文本添加runDelegate
    [self p_addRunDelegateWithAttributedString:attributedString range:range];
}

#pragma mark - RDAppendTextStorageProtocol 协议实现

- (NSAttributedString *)appendTextStorageAttributedString
{
    // 创建空字符属性文本
    NSMutableAttributedString *attributedString = [[NSMutableAttributedString alloc]initWithString:p_spaceReplaceString()];
    // 修正range
    _range = NSMakeRange(0, 1);
    
    // 设置合适的对齐
    [self p_setAppropriateAlignment];
    
    // 添加文本属性和runDelegate
    [self p_addRunDelegateWithAttributedString:attributedString range:_range];
    return attributedString;
}

#pragma mark - RDDrawStorageProtocol 协议实现

- (void)setCurrentReplacedStringNum:(NSInteger)replacedStringNum
{
    _fixRange = p_fixRange(_range, replacedStringNum);
}

- (void)setTextfontAscent:(CGFloat)ascent descent:(CGFloat)descent;
{
    _fontAscent = ascent;
    _fontDescent = -descent;
}

- (void)drawStorageWithRect:(CGRect)rect {
    //空实现
}


#pragma mark - public暴露方法

- (CGFloat)getDrawRunAscentHeight
{
    CGFloat ascent = 0;
    CGFloat height = self.size.height+_margin.bottom+_margin.top;
    
    switch (_drawAlignment)
    {
        case RDDrawAlignmentTop:
            ascent = height - _fontDescent;
            break;
        case RDDrawAlignmentCenter:
        {
            CGFloat baseLine = (_fontAscent + _fontDescent) / 2 - _fontDescent;
            ascent = height / 2 + baseLine;
            break;
        }
        case RDDrawAlignmentBottom:
            ascent = _fontAscent;
            break;
        default:
            break;
    }
    
    return ascent;
}

- (CGFloat)getDrawRunWidth
{
    return self.size.width + _margin.left + _margin.right;
}

- (CGFloat)getDrawRunDescentHeight
{
    CGFloat descent = 0;
    CGFloat height = self.size.height+_margin.bottom+_margin.top;
    switch (_drawAlignment)
    {
        case RDDrawAlignmentTop:
            descent = _fontDescent;
            break;
        case RDDrawAlignmentCenter:
        {
            CGFloat baseLine = (_fontAscent + _fontDescent) / 2 - _fontDescent;
            descent = height / 2 - baseLine;
            break;
        }
        case RDDrawAlignmentBottom:
            descent = height - _fontAscent;
            break;
        default:
            break;
    }
    
    return descent;
}

- (void)DrawRunDealloc {
    //空实现
}

#pragma mark - private内部私有方法

- (void)p_setAppropriateAlignment
{
    // 判断size 大小 小于 _fontAscent 把对齐设为中心 更美观
    if (_size.height <= _fontAscent + _fontDescent) {
        _drawAlignment = RDDrawAlignmentCenter;
    }
}

#pragma mark - CTRunDelegate Callbacks 获取图片显示的宽度、高度

// 添加文本属性和runDelegate
- (void)p_addRunDelegateWithAttributedString:(NSMutableAttributedString *)attributedString range:(NSRange)range
{
    // 现将当前Storage对象保存到 富文本中
    [attributedString addAttribute:kRDTextRunAttributedName value:self range:range];
    
    //为图片设置CTRunDelegate,delegate决定留给显示内容的空间大小
    CTRunDelegateCallbacks runCallbacks;
    runCallbacks.version = kCTRunDelegateVersion1;
    runCallbacks.dealloc = TYTextRunDelegateDeallocCallback;
    runCallbacks.getAscent = TYTextRunDelegateGetAscentCallback;
    runCallbacks.getDescent = TYTextRunDelegateGetDescentCallback;
    runCallbacks.getWidth = TYTextRunDelegateGetWidthCallback;
    
    CTRunDelegateRef runDelegate = CTRunDelegateCreate(&runCallbacks, (__bridge void *)(self));
    [attributedString addAttribute:(__bridge_transfer NSString *)kCTRunDelegateAttributeName value:(__bridge id)runDelegate range:range];
    CFRelease(runDelegate);
}

//CTRun的回调，销毁内存的回调
void TYTextRunDelegateDeallocCallback( void* refCon ){
    //TYDrawRun *textRun = (__bridge TYDrawRun *)refCon;
    //[textRun DrawRunDealloc];
}

//CTRun的回调，获取CTLine的上行高度
CGFloat TYTextRunDelegateGetAscentCallback( void *refCon ){
    
    RDDrawStorage *drawStorage = (__bridge RDDrawStorage *)refCon;
    return [drawStorage getDrawRunAscentHeight];
}

//CTRun的回调，获取CTLine的下行高度
CGFloat TYTextRunDelegateGetDescentCallback(void *refCon){
    RDDrawStorage *drawStorage = (__bridge RDDrawStorage *)refCon;
    return [drawStorage getDrawRunDescentHeight];
}

//CTRun的回调，获取CTLine的最大显示宽度
CGFloat TYTextRunDelegateGetWidthCallback(void *refCon){
    
    RDDrawStorage *drawStorage = (__bridge RDDrawStorage *)refCon;
    return [drawStorage getDrawRunWidth];
}

@end
```


- 将源码中的一些不必要的oc函数，直接写成c函数
- 暂时没有搞明白`setCurrentReplacedStringNum:`函数的作用

这个Storage主要是对待处理的文本text中，某一些range对应的text绘制成图像，如: [haha, 100, 200]

****

###ImageStorage

- 继承自DrawStorage进行绘制UIImage
- 实现RDViewStorageProtocol协议

```objc
typedef enum : NSUInteger {
    RDImageAlignmentCenter,  // 图片居中
    RDImageAlignmentLeft,    // 图片左对齐
    RDImageAlignmentRight,   // 图片右对齐
} RDImageAlignment;

@interface RDImageStorage : RDDrawStorage <RDViewStorageProtocol>

@property (nonatomic, strong) UIImage               *image;
@property (nonatomic, strong) NSString              *imageName;
@property (nonatomic, strong) NSURL                 *imageURL;
@property (nonatomic, strong) NSString              *placeholdImageName;
@property (nonatomic, assign) RDImageAlignment      imageAlignment; // default center
@property (nonatomic, assign) BOOL                  cacheImageOnMemory; // default NO ,if YES can improve performance，but increase memory

@end
```

```objc
@interface RDImageStorage ()
@property (nonatomic, weak) UIView *ownerView;
@property (nonatomic, assign) BOOL isNeedUpdateFrame;
@end

@implementation RDImageStorage

- (instancetype)init
{
    if (self = [super init]) {
        _cacheImageOnMemory = NO;
    }
    return self;
}

#pragma mark - RDViewStorageProtocol

- (void)setOwnerView:(UIView *)ownerView
{
    _ownerView = ownerView;
    
    __weak typeof(self)weakSelf = self;
    
    if (_imageURL && [_imageURL isKindOfClass:[NSURL class]]) {
        
        //使用SDWebImage下载网络图片
        [[SDWebImageDownloader sharedDownloader] downloadImageWithURL:_imageURL options:SDWebImageDownloaderIgnoreCachedResponse progress:^(NSInteger receivedSize, NSInteger expectedSize) {
            //progress
        } completed:^(UIImage *image, NSData *data, NSError *error, BOOL finished) {
            
            // 使用SDImageCache缓存下载的图片
            [[SDImageCache sharedImageCache] storeImage:image forKey:[_imageURL absoluteString]];
            
            // 调用drawRect:绘制图像
            if (weakSelf.isNeedUpdateFrame) {
                if (ownerView) {
                    [ownerView setNeedsDisplay];//drawRect:
                }
                _isNeedUpdateFrame = NO;
            }
        }];
    }
}

- (void)didNotDrawRun {}

#pragma mark - RDDrawStorageProtocol

/**
 *  完成在ownerView上绘制图像
 */
- (void)drawStorageWithRect:(CGRect)rect {
    UIImage *image = nil;
    
    //根据图片类型获取对应图像
    if (_image) {
        // 已经载入本地图片
        image = _image;
    }else if (_imageName){
        // 读取本地图片
        image = [UIImage imageNamed:_imageName];
        if (_cacheImageOnMemory) {
            _image = image;
        }
    } else if (_imageURL){
        // 网络图片
        image = [[SDImageCache sharedImageCache] imageFromDiskCacheForKey:[_imageURL absoluteString]];
        if (image) {
            if (_cacheImageOnMemory) {
                _image = image;
            }
        } else {
            image = _placeholdImageName ? [UIImage imageNamed:_placeholdImageName] : nil;
            _isNeedUpdateFrame = YES;
        }
    }
    
    //绘制图片
    if (image) {
        CGRect fitRect = [self rectFitOriginSize:image.size byRect:rect];
        CGContextRef context = UIGraphicsGetCurrentContext();
        CGContextDrawImage(context, fitRect, image.CGImage);
    }
}

#pragma mark - private

- (CGRect)rectFitOriginSize:(CGSize)size byRect:(CGRect)byRect{
    
    CGRect scaleRect = byRect;
    CGFloat targetWidth = byRect.size.width;
    CGFloat targetHeight = byRect.size.height;
    CGFloat widthFactor = targetWidth / size.width;
    CGFloat heightFactor = targetHeight / size.height;
    CGFloat scaleFactor = MIN(widthFactor, heightFactor);
    CGFloat scaledWidth  = size.width * scaleFactor;
    CGFloat scaledHeight = size.height * scaleFactor;
    scaleRect.size = CGSizeMake(scaledWidth, scaledHeight);
    
    // center the image
    if (widthFactor < heightFactor) {
        scaleRect.origin.y += (targetHeight - scaledHeight) * 0.5;
    } else if (widthFactor > heightFactor) {
        switch (_imageAlignment) {
            case RDImageAlignmentCenter:
                scaleRect.origin.x += (targetWidth - scaledWidth) * 0.5;
                break;
            case RDImageAlignmentRight:
                scaleRect.origin.x += (targetWidth - scaledWidth);
            default:
                break;
        }
    }
    
    return scaleRect;
}


@end
```

ImageStorage主要完成:

- 图片的获取
	- 本地图片 >>> ImageName
	- 网络图片 >>> ImageURL

- 图片的缓存
	- 将作者自己写的很挫的缓存，替换成SDWebImageCache

- 最终将图像Image绘制到OwnerView上

***

###ViewStorage

- 继承自DrawStorage
- 实现RDViewStorageProtocol协议

```objc
@interface RDViewStorage : RDDrawStorage <RDViewStorageProtocol>

@property (nonatomic, strong)   UIView              *view;       // 添加view

@end
```

```objc
@interface RDViewStorage ()

//传入View的superView
@property (nonatomic, weak) UIView *superView;

@end

@implementation RDViewStorage

#pragma mark - life

- (void)dealloc{
    // 需要去掉supview 的 强引用 否则内存泄露
    if (_view.superview) {
        [_view removeFromSuperview];
    }
}

//setter重写
- (void)setView:(UIView *)view
{
    _view = view;
    
    // 默认使用传入view的size尺寸
    if (CGSizeEqualToSize(self.size, CGSizeZero)) {
        self.size = view.frame.size;
    }
}

#pragma mark - RDViewStorageProtocol

- (void)setOwnerView:(UIView *)ownerView
{
    if (_view.superview) {
        [_view removeFromSuperview];
    }
    
    if (ownerView) {
        _superView = ownerView;
    }
}

- (void)didNotDrawRun
{
    [_view removeFromSuperview];
}

- (void)drawStorageWithRect:(CGRect)rect
{
    if (_view == nil || _superView == nil) return;
    
    //对当前绘制的rect翻转成CoreText坐标系的rect
    CGAffineTransform transform =  CGAffineTransformScale(CGAffineTransformMakeTranslation(0, _superView.bounds.size.height), 1.f, -1.f);
    rect = CGRectApplyAffineTransform(rect, transform);
    
    [_view setFrame:rect];
    [_superView addSubview:_view];
}

@end
```

主要是完成一个view添加到另一个superView上

***

###OK，看完了所有的Storage之后，继续回到TextContainer实现来


代码有点多，就不往下贴了，小结下主要做的事情:

- 管理各种Storage
- 并使用这些Storage的range、text来构建最终的富文本属性字符串，给对应的range添加Storage对象
	- 其中的DrawStorage完成图片的显示，封装调用CTRunDelegate Callbacks的相关代码来获取图片的高度（上行高度+下行高度）、图片的宽度
- 创建富文本属性字符串CTFrameRef
- 然后解析CTFrameRef中的每一个CTRunRef，并取出对应的Storage对象，然后使用NSDictionary缓存起来


真的整个代码都看完了，花了大概两个下午的时间，把代码看明白了才好去改他的代码，代码在对`普通文字`长按的时候不执行回调，只是针对Link和Image点击和长按回调。于是给作者发了issue，作者回答是不支持...好吧，那就给他改改吧。刚好准备跳槽了多看点面试也方便...

回来补充下解决的办法:

- 第一种，直接修改TextContainer里面的代码，给TextStorage也添加缓存字典保存，在AttributedLabel回调中回传TextStorage

- 第二种，直接使用宏编译修改AttributedLabel的代码，进行自己的绘制代码逻辑
