---
layout: post
title: "CoreText Part2"
date: 2016-04-14 11:13:56 +0800
comments: true
categories: 
---

###本文主要记录一些CoreText的具体使用demo代码，经过两三天的不停的折腾最终终于做成了下面这些效果，也学到了很多以前不知道的关于CoreText的东西.

- 效果一、针对富文本中的特殊内容的显示样式、以及点击的回调处理

![coreText2.gif](http://imgchr.com/images/coreText2.gif)

- 效果二、长按事件时对文字绘制下划线

![coreText.gif](http://imgchr.com/images/coreText.gif)


- 最终代码优化之后的效果

![coreText370db.gif](http://imgchr.com/images/coreText370db.gif)


> 这些都只是试水的一些demo小例子，大神直接可以路过，待把CoreText基本上搞通了，也会尝试自己封装下，向YYText看齐.

***

###首先是要搞清楚CoreText坐标系与UIKit坐标系的区别，主要是y坐标不一样:

> 前面例子中使用CoreText绘制文字时，需要对文字翻转.

而需要翻转的原因是，CoreText的坐标系与UIKit的坐标系不相同.

![](http://i2.piimg.com/6f2c29ac48621726.png)

- CoreText的坐标系原点 >>> 左下角
- UIKit的坐标系原点 >>> 左上角

解决方法 >>> 将当前UIKit所处的绘图上下文context坐标系进行翻转

```c
- (void)drawRect:(CGRect)rect {

	//1. 获取当前上下文
	CGContextRef context = UIGraphicsGetCurrentContext();
	
	//2. 设置为当前文本绘制的矩阵
    CGContextSetTextMatrix(context, CGAffineTransformIdentity);
    
    //3. 文本沿y轴移动
    CGContextTranslateCTM(context, 0, self.bounds.size.height);
    
    //4. 翻转当前context
    CGContextScaleCTM(context, 1, -1);
}
```

****

###CoreText 框架中最常用的几个类:

```
CTFont
CTFontCollection
CTFontDescriptor
CTFrame
CTFramesetter
CTGlyphInfo
CTLine
CTParagraphStyle
CTRun
CTTextTab
CTTypesetter
```

对于上述的几个类的关系，可以看下图所示:

![](http://i2.piimg.com/4232f7dc7f0c54f6.png)

CTFrame、CTLine、CTRun三者之间的关系:

- CTFrame: 就好比一篇文章，一篇文章会包含多个显示的`行`

- CTLine: 就是上面所说的文章中的`每一行`，而每一行又包含多个`块`

- CTRun: 就是一行中的很多的块，而块是指 `一组共享 相同属性 的字体 的 集合`

图示如上三者之间的关系:

![](http://i3.piimg.com/d7bff94e871a7bc7.png)


CTFramesetter:

- 依赖一个富文本属性字符串
- 根据这个富文本属性字符串，计算得到一个CTFrame
- 可以看做是CTFrame的生产工厂

其他类用到再回来记录吧...


***

###使用CoreText进行文字排版的步骤:

- 首先确定将要进行绘制文字的`总大小区域`，使用 CG(Mutable)PathRef确定

- 处理好一个富文本属性内容 ，其对应的类为 NS(Mutable)AttributedString

- 将处理好的NS(Mutable)AttributedString实例，设置给CTFramesetter工厂
	- 告诉要进行计算CTFrame的 `文本数据`

- 将处理好的CG(Mutable)PathRef实例，设置给CTFramesetter工厂
	- 告诉要进行计算CTFrame的 `区域边界`

- 最后通过CTFramesetterRef 得到 CTFrame，就是当前显示文本的画布所有的信息


下面是对上述步骤的一个简单例子:

```objc
- (void)drawTextInRect:(CGRect)rect {

//////////////////////////////去掉文字的空行//////////////////////////////
    NSString *labelString = self.text;
    NSString *myString = [labelString stringByReplacingOccurrencesOfString:@"\r\n" withString:@"\n"];

//////////////////////////////创建富文本属性字符串//////////////////////////////
    NSMutableAttributedString *attStr = [[NSMutableAttributedString alloc] initWithString:myString];
    
//////////////////////////////添加各种富文本属性设置//////////////////////////////

- 字体、颜色、样式、下划线了...
- 设置字间距、行间距...
- 段落样式...
- 文本对齐方式...

//////////////////////////////确定富文本属性显示的区域//////////////////////////////

 	//1. 使用 CTFramesetterRef 包装 NSMutableAttributedString富文本
    CTFramesetterRef framesetter = CTFramesetterCreateWithAttributedString((CFAttributedStringRef)attStr);
    
    //2. 创建一个封闭区域路径
    CGMutablePathRef path = CGPathCreateMutable();
    CGPathAddRect(path,
                  NULL ,
                  CGRectMake(0 , 0 , self.bounds.size.width , self.bounds.size.height));
    
    //3. 将文本绘制的区域告诉CTFramesetterRef，得到文本显示的画布数据
    CTFrameRef leftFrame = CTFramesetterCreateFrame(framesetter,
                                                    CFRangeMake(0, attStr.length),
                                                    path ,
                                                    NULL);
                                                    
//////////////////////////////翻转坐标系统//////////////////////////////
.....省略

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
```

***

###CoreText/TextKit都是需要一个富文本（NSAttributedString/NSMutableAttributedString）来显示，你得告诉他显示什么东西

> NSAttributedString比NSString多了Attribute的概念。它可以包含很多属性，粗体，斜体，下划线，颜色，背景色等等，每个属性都有其对应的字符区域。富文本属性key的规律: NS....AttributeName/kCT...AttributeName

从上面的例子也可以看出，整个进行绘制的最开始，就是从一个NSAttributedString/NSMutableAttributedString属性字符串开始.

![](http://i2.piimg.com/11136588beb8faee.png)

富文本区域与普通文本字符串NSString，可以用下面这段代码看出来:

```objc
NSString *content = @"Haha I love you ~~~";

NSMutableAttributedString *attString = [[NSMutableAttributedString alloc] initWithString:content];
[attString addAttribute:NSForegroundColorAttributeName value:[UIColor redColor] range:NSMakeRange(1, 3)];
[attString addAttribute:NSFontAttributeName value:[UIFont systemFontOfSize:18] range:NSMakeRange(2, 5)];

NSLog(@"%@", content);
NSLog(@"%@", attString);
```

输出如下

```
Haha I love you ~~~
H{
}a{
    NSColor = "UIDeviceRGBColorSpace 1 0 0 1";
}ha{
    NSColor = "UIDeviceRGBColorSpace 1 0 0 1";
    NSFont = "<UICTFont: 0x7ff071598b80> font-family: \".SFUIText-Regular\"; font-weight: normal; font-style: normal; font-size: 18.00pt";
} I {
    NSFont = "<UICTFont: 0x7ff071598b80> font-family: \".SFUIText-Regular\"; font-weight: normal; font-style: normal; font-size: 18.00pt";
}love you ~~~{
}
```

可以得到规律，将普通字符串分隔开来，使用`{ 富文本属性设置 }` 来存放富文本属性.

****

###CTRunDelegateRef: 自定义图文混排时需要用到

![](http://i4.piimg.com/3fc13bda51c39018.png)

> 需要告诉CoreText哪个区域（CTRun）绘制文字，哪个区域（CTRun）绘制图片.

- 创建CTRunDelegateRef的api函数

```c
CTRunDelegateRef __nullable CTRunDelegateCreate(
	const CTRunDelegateCallbacks* callbacks,
	void * __nullable refCon )
```

需要传入一个CTRunDelegateCallbacks指针.

- CTRunDelegateCallbacks

```c
typedef struct
{
	CFIndex							version;
	CTRunDelegateDeallocateCallback	dealloc;		//内存释放回调
	CTRunDelegateGetAscentCallback	getAscent;		//设置CTLine上行高度
	CTRunDelegateGetDescentCallback	getDescent;		//设置CTLine下行高度
	CTRunDelegateGetWidthCallback	getWidth;		//设置CTLine最大显示宽度
} CTRunDelegateCallbacks;
```

- 对CTRunDelegateCallbacks结构体中需要的回调函数进行定义

```c
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
```

- 准备好回调的c函数之后，创建CTRunDelegateCallbacks 结构体实例

```c
CTRunDelegateCallbacks runCallbacks;
runCallbacks.version = kCTRunDelegateVersion1;
runCallbacks.dealloc = TYTextRunDelegateDeallocCallback;
runCallbacks.getAscent = TYTextRunDelegateGetAscentCallback;
runCallbacks.getDescent = TYTextRunDelegateGetDescentCallback;
runCallbacks.getWidth = TYTextRunDelegateGetWidthCallback;
```

- 最后创建CTRunDelegateRef实例

```c
CTRunDelegateRef runDelegate = CTRunDelegateCreate(&runCallbacks, (__bridge void *)(self));
```

- 最后将CTRunDelegateRef添加到富文本NSAttributedString中
	- 属性key >>> kCTRunDelegateAttributeName
	- 属性value >>> CTRunDelegateRef实例
	- 处理范围 >>> range（针对CTLine中哪一块）

```c
[attributedString addAttribute:(__bridge_transfer NSString *)kCTRunDelegateAttributeName value:(__bridge id)runDelegate range:range];

CFRelease(runDelegate);
```

- 将上述代码结合起来

```objc
// 添加文本属性和runDelegate
- (void)addRunDelegateWithAttributedString:(NSMutableAttributedString *)attributedString range:(NSRange)range
{
    // 添加文本属性和runDelegate
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
```

> 经过上面这么多步骤，其实只是告诉Core Text 有一个地方需要占多大的位置，这样系统就会在指定的地方把空间腾出来，不绘制文字上去。

***

###获取一个CTFrame中某一行CTLine的上行高度、下行高度、行距

获取一个CTLine的如上数据的写法:

```objc
CGFloat lineAscent;
CGFloat lineDescent;
CGFloat lineLeading;


//返回一个run的宽度
CGFloat lineWidth  = CTLineGetTypographicBounds(line, &lineAscent, &lineDescent, &lineLeading);
```
如上需要注意: 

- 上面获取到的lineAscent、lineDescent等，都只是 参照`当前CTLine `的属性值，并不是 `真正绘图` 需要的是 `相对最终CoreText坐标系原点` 的坐标值

- 就需要在循环CTLine和CTRun的时候，要记录下line和run的origin，并累加起来才是真正相对于坐标原点的偏移量

类似还有获取一个CTRun的类似数据的函数

```c
double CTRunGetTypographicBounds(
    CTRunRef run,
    CFRange range,
    CGFloat * __nullable ascent,
    CGFloat * __nullable descent,
    CGFloat * __nullable leading )
```

后面例子会使用到.

****

###下面是一段简单的CoreText图文排布的demo示例代码

```objc
#import <UIKit/UIKit.h>

@interface CoreTextEventLabel : UILabel

@end
```

```objc
#import "CoreTextEventLabel.h"

#import <CoreText/CoreText.h>

#pragma mark - CTRunDelegateCallbacks

void RunDelegateDeallocCallback( void* refCon ){
    
}

CGFloat RunDelegateGetAscentCallback( void *refCon ){
    NSString *imageName = (__bridge NSString *)refCon;
    return [UIImage imageNamed:imageName].size.height;
}

CGFloat RunDelegateGetDescentCallback(void *refCon){
    return 0;
}

CGFloat RunDelegateGetWidthCallback(void *refCon){
    NSString *imageName = (__bridge NSString *)refCon;
    return [UIImage imageNamed:imageName].size.width;
}

@implementation CoreTextEventLabel {
    NSMutableAttributedString   *_attString;
    CTFrameRef                  _frame;
}

- (void)buildAttributeString {

    //测试图片
    NSString *imageName = @"1.jpg";
    
    //1. 包装原始文本
    _attString = [[NSMutableAttributedString alloc] initWithString: self.text];
    
    //3. 创建CTRunDelegate回调结构体，指定回调函数
    CTRunDelegateCallbacks imageCallbacks;
    imageCallbacks.version = kCTRunDelegateVersion1;
    imageCallbacks.dealloc = RunDelegateDeallocCallback;
    imageCallbacks.getAscent = RunDelegateGetAscentCallback;
    imageCallbacks.getDescent = RunDelegateGetDescentCallback;
    imageCallbacks.getWidth = RunDelegateGetWidthCallback;
    
    //4. 创建CTRun的回调delegte，获取被替换的CTRun的数据: 上行高度、下行高度、块宽度
    CTRunDelegateRef runDelegate = CTRunDelegateCreate(&imageCallbacks, (__bridge void * _Nullable)(imageName));
    
    //5. 假定对富文本中的空白字符进行替换成图片
    NSMutableAttributedString *imageAttributedString = [[NSMutableAttributedString alloc] initWithString:@" "];
    
    //6. 给空白字符对应的range
     [imageAttributedString addAttribute:(NSString *)kCTRunDelegateAttributeName value:(__bridge id _Nonnull)(runDelegate) range:NSMakeRange(0, 1)];
    CFRelease(runDelegate);
    
    //7. 给富文本某一个range添加键值对，这个range会被作为一个CTRun块
    [imageAttributedString addAttribute:@"imageName" value:imageName range:NSMakeRange(0, 1)];
    
    //8. 拼接到外层富文本
    [_attString appendAttributedString:imageAttributedString];
    
    //9. 创建多个段落样式
    
    //断行模式
    CTParagraphStyleSetting lineBreakMode;
    CTLineBreakMode lineBreak = kCTLineBreakByCharWrapping;
    lineBreakMode.spec = kCTParagraphStyleSpecifierLineBreakMode;//spec
    lineBreakMode.value = &lineBreak;//value
    lineBreakMode.valueSize = sizeof(CTLineBreakMode);//valueSize
    
    //文本对齐属性
    CTParagraphStyleSetting alignmentStyle;
    CTTextAlignment alignment = kCTJustifiedTextAlignment;
    alignmentStyle.spec=kCTParagraphStyleSpecifierAlignment;
    alignmentStyle.valueSize=sizeof(alignment);
    alignmentStyle.value=&alignment;
    
    //首行缩进
    CTParagraphStyleSetting fristline;
    CGFloat fristlineindent = 24.0f;
    fristline.spec = kCTParagraphStyleSpecifierFirstLineHeadIndent;
    fristline.value = &fristlineindent;
    fristline.valueSize = sizeof(float);
    
    //10. 将上述所有的样式组装成数组容器
    CTParagraphStyleSetting settings[] = {
//        lineBreakMode, alignmentStyle, fristline
        lineBreakMode
    };
    
    //11. 创建总段落样式
    CTParagraphStyleRef style = CTParagraphStyleCreate(settings, 1);
    
    //12. 段落样式字典
    NSMutableDictionary *attributes = [@{
                                         (id)kCTParagraphStyleAttributeName : (id)style,
                                         } mutableCopy];
    
    //13. 给富文本使用段落样式字典
    [_attString addAttributes:attributes range:NSMakeRange(0, _attString.length)];
    
    //14. 点击事件的字符heightlight效果
//     [_attString addAttribute:(id)kCTForegroundColorAttributeName
//                        value:(id)[[UIColor blueColor] CGColor]
//                        range:NSMakeRange(0, 10)];
}

- (void)drawTextInRect:(CGRect)rect {
    
    //1. 构建富文本
    [self buildAttributeString];
    
    //2. 开启一个绘图上下文
    CGContextRef context = UIGraphicsGetCurrentContext();
    
    //3. 翻转UIKit绘图上下文
    CGContextSaveGState(context);
    CGContextSetTextMatrix(context, CGAffineTransformIdentity);
    CGContextTranslateCTM(context, 0, rect.size.height);
    CGContextScaleCTM(context, 1.0, -1.0);
    
    //4. 计算得到文本显示的CTFrame
    CTFramesetterRef framesetter =  CTFramesetterCreateWithAttributedString((CFAttributedStringRef) _attString);
    CGMutablePathRef path = CGPathCreateMutable();
    CGPathAddRect(path, NULL, CGRectMake(0, 0, rect.size.width, rect.size.height));
    _frame = CTFramesetterCreateFrame(framesetter, CFRangeMake(0, _attString.length), path, NULL);
    
    //5. 绘制文本
    CTFrameDraw(_frame, context);
    
    //6. 获取画出来的内容的行CTLine
    CFArrayRef lines = CTFrameGetLines(_frame);
    
    //7. 获取每行的坐标原点
    CGPoint lineOrigins[CFArrayGetCount(lines)];//坐标原点数组
    CTFrameGetLineOrigins(_frame, CFRangeMake(0, 0), lineOrigins);
    NSLog(@"line count = %ld",CFArrayGetCount(lines));
    
    //8. 遍历所有的CTLine中的每一个CTRun小文本块（字体相同的文本块）
    for (int i = 0; i < CFArrayGetCount(lines); i++)
    {
        //8.1 遍历的当前CTLine
        CTLineRef line = CFArrayGetValueAtIndex(lines, i);
        
        //8.2 每一个CTLine中的所有CTRun的 上行高度、下行高度、行距
        CGFloat lineAscent;
        CGFloat lineDescent;
        CGFloat lineLeading;
        
        //8.3 获取每行的宽度和高度
        CTLineGetTypographicBounds(line, &lineAscent, &lineDescent, &lineLeading);
        NSLog(@"ascent = %f,descent = %f,leading = %f",lineAscent,lineDescent,lineLeading);
        
        //8.4 获取某一个CTLine中的所有的CTRun
        CFArrayRef runs = CTLineGetGlyphRuns(line);
        NSLog(@"run count = %ld",CFArrayGetCount(runs));
        
        //8.5 遍历一个CTLine内部的所有的CTRun
        for (int j = 0; j < CFArrayGetCount(runs); j++) {
            
            //8.5.1 记录一个CTRun的上行高度、下行高度
            CGFloat runAscent;
            CGFloat runDescent;
            
            //8.5.2 获取当前CTLine的坐标原点
            CGPoint lineOrigin = lineOrigins[i];
            
            //8.5.3 获取当前CTLine中的每个CTRun
            CTRunRef run = CFArrayGetValueAtIndex(runs, j);
            
            //8.5.4 取出每一个CTRun的属性字典
            NSDictionary* attributes = (NSDictionary*)CTRunGetAttributes(run);
            NSLog(@"CTLine = %@, CTRun = %@, attributes = %@", line, run, attributes);
            
            //8.5.5 获取CTRun的参数值
            double runWidth = CTRunGetTypographicBounds(run, CFRangeMake(0,0), &runAscent, &runDescent, NULL);
            
            //8.5.6 修正CTRun的frame
            CGFloat runX = lineOrigin.x + CTLineGetOffsetForStringIndex(line, CTRunGetStringRange(run).location, NULL);
            CGFloat runY = lineOrigin.y - runDescent;
            CGFloat runW = runWidth;
            CGFloat runH = runAscent + runDescent;
            NSLog(@"runX = %f, runY = %f, runW = %f, runH = %f", runX, runY, runW, runH);
            CGRect runRect = CGRectMake(runX, runY, runW, runH);
            
            //8.5.7 判断当前CTRun是否是应该显示图片，把需要被图片替换的字符位置画上图片
            NSString *imageName = [attributes objectForKey:@"imageName"];
            
            //8.5.8 给当前CTRun所处区域绘制图片
            if (imageName) {
                UIImage *image = [UIImage imageNamed:imageName];
                
                if (image) {
                    
                    // CTRun的绘图区域
                    CGRect imageDrawRect;
                    imageDrawRect.size = CGSizeMake(image.size.width, image.size.height);
                    imageDrawRect.origin.x = runRect.origin.x + lineOrigin.x;
                    imageDrawRect.origin.y = lineOrigin.y;
                    
                    // 绘制图片
                    CGContextDrawImage(context, imageDrawRect, image.CGImage);
                }
            }
        }
    }
    
    //9. 提交上下文的绘图修改
    CGContextRestoreGState(context);
    
}
```

ViewController中测试

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    CoreTextEventLabel *label = [[CoreTextEventLabel alloc] init];
    label.layer.borderWidth = 1;
    label.userInteractionEnabled = YES;
    label.text = @"哈师大i大会谁都哈师大i大会谁都哈师大i[image.png]大会谁都哈师大i大会谁都哈师大i大会谁都哈师大i大会谁都Ha";
    label.frame = CGRectMake(20, 100, 280, 450);
    label.font = [UIFont systemFontOfSize:15.f];
    [self.view addSubview:label];
}
```

####运行结果就是一段文字和一个图片很简单就不贴图了，然后小结一下如上代码:

- 将文本最后的一个空格字符Range(length-1, 1)添加了一个CTRunDelegate回调处理
	- 简单的将最后一个字符使用空格字符，作为图片显示的range
	- 其实不该使用空格符，而应该使用占位符`\uFFFC`

- CoreText当执行到解释那个`空格字符`到字形时，发现注册了CTRunDelegate

- 然后调用CTRunDelegate中的几个c回调函数，分别获取:
	- CTRun的上行高度 >>> 得到图片显示的高度
	- CTRun的下行高度	>>> 0
	- CTRun的宽度 >>> 得到图片显示的宽度
	
- 其实上一步就是告诉CoreText，在这个CTFrame中腾出一块额外的CTRun区域
	- 至于我们是显示图片or文字，CoreText其实根本不知道
	- 如果是绘制图片，需要使用ImageDrawContext完成绘制图片

- 最后需要我们自己受到完成显示图片的逻辑
	- 遍历当前CTFrame中的每一行CTLine
	- 每一行CTLine中的每一块CTRun
	- 再获取每一个CTRun属性字典（CFDictionaryRef CTRunGetAttributes(CTRunRef run)函数）
	- 从CTRun属性字典中取出要显示的图片名
		- 本例使用本地图片简单
		- 复杂的可能是通过异步获取网络URL图片
	- 最终使用`CGContextDrawImage()`完成图片绘制

***

###另外一个简单使用CTRunDelegate完成稍微复杂一点的例子，完成图文混排同时可以点击文字做出一些响应

图文混排的样子

![](http://i4.piimg.com/40fd8d9aac5af22e.png)

连接 被点击的Alert

![](http://i4.piimg.com/6c4abd46ae7e1312.png)

@xiaoming 被点击的Alert

![](http://i4.piimg.com/13240eb054dcf89b.png)

`#话题#`被点击的Alert

![](http://i4.piimg.com/b69638a24cea78b2.png)


下面小结一下实现的关键点:

####第一个、富文本属性字符串NSAttributedString的构建

- 对文本中的 `图片 range` 处理
- 对文本中的 `链接 range` 处理
- 对文本中的 `@小明 range` 处理
- 对文本中的 `#话题# range` 处理
- 其他的普通的文本字符串就不需要额外处理

可以使用一个NSMutableAttributedString分类来完成富文本属性的构建，包含如上所有情况的解析.

```objc
#import <Foundation/Foundation.h>

//占位符，用于说明某个range是绘制图片
#define CTAttachmentChar "\uFFFC"
#define CTAttachmentCharacter @"\uFFFC"

//CTRun的类型（emoji、链接、@、#话题#）
extern NSString *const kCustomGlyphAttributeType;

//CTRun富文本的range
extern NSString *const kCustomGlyphAttributeRange;

//CTRun显示图片的名字
extern NSString *const kCustomGlyphAttributeImageName;

//CTRun块类型
typedef enum CustomGlyphAttributeType {
    CustomGlyphAttributeURL = 0,        //URL
    CustomGlyphAttributeAt,             //
    CustomGlyphAttributeTopic,          //
    CustomGlyphAttributeImage,          //图片
    CustomGlyphAttributeInfoImage,      // 预留，给带相应信息的图片（如点击图片获取相关属性）
} CustomGlyphAttributeType;

//CTRun回调时获取数据结构体
typedef struct CustomGlyphMetrics {
    CGFloat ascent;         //上行高度
    CGFloat descent;        //下行高度
    CGFloat width;          //宽度
} CustomGlyphMetrics, *CustomGlyphMetricsRef;

@interface NSMutableAttributedString (Weibo)

/**
 *  对传入的带有富文本设置的NSString处理成富文本NSMutableAttributedString
 */
+ (NSMutableAttributedString *)weiboAttributedStringWithString:(NSString *)string;

@end
```

```objc

#import <CoreText/CoreText.h>
#import <UIKit/UIKit.h>

#import "NSMutableAttributedString+Weibo.h"

NSString *const kCustomGlyphAttributeType = @"CustomGlyphAttributeType";
NSString *const kCustomGlyphAttributeRange = @"CustomGlyphAttributeRange";
NSString *const kCustomGlyphAttributeImageName = @"CustomGlyphAttributeImageName";

//emoji字符串过滤的正则式，[熊猫]
NSString *const kRegexEmoji = @"\\[[a-zA-Z0-9\\u4e00-\\u9fa5]+\\]";

//url链接过滤的正则式
NSString *const kRegexShortLink = @"http://t.cn/[a-zA-Z0-9]+";

//@小明的过滤正则式
NSString *const kRegexCallBody = @"@[\\u4e00-\\u9fa5\\w\\-]+";

//#话题#的过滤正则式
NSString *const kRegexTopic = @"#([^\\#|.]+)#";

const CGFloat kLineSpacing = 4.0;
const CGFloat kContentTextSize = 13.0;
const CGFloat kAscentDescentScale = 0.25;

///////////////////////CFRunDelegate回调/////////////////////////////
static void deallocCallback(void *refCon) {
	free(refCon), refCon = NULL;
}

static CGFloat ascentCallback(void *refCon) {
	CustomGlyphMetricsRef metrics = (CustomGlyphMetricsRef)refCon;
	return metrics->ascent;
}

static CGFloat descentCallback(void *refCon) {
	CustomGlyphMetricsRef metrics = (CustomGlyphMetricsRef)refCon;
	return metrics->descent;
}

static CGFloat widthCallback(void *refCon) {
	CustomGlyphMetricsRef metrics = (CustomGlyphMetricsRef)refCon;
	return metrics->width;
}
//////////////////////////////////////////////////////////////////////

@implementation NSMutableAttributedString (Weibo)

//读取emoji的plist数据
+ (NSDictionary *)weiboEmojiDictionary {
    static NSDictionary *emojiDictionary = nil;
    static dispatch_once_t onceToken;
	dispatch_once(&onceToken, ^{
	    NSString *emojiFilePath = [[[NSBundle mainBundle] resourcePath] stringByAppendingPathComponent:@"emotionImage.plist"];
	    emojiDictionary = [[NSDictionary alloc] initWithContentsOfFile:emojiFilePath];
	});
	return emojiDictionary;
}

#pragma mark - emoji、链接、@小明、#话题#、这四种特殊内容的range处理

+ (NSMutableAttributedString *)weiboAttributedStringWithString:(NSString *)string {
    
    // 保存最终处理好的富文本
    NSMutableAttributedString *newStr = [[NSMutableAttributedString alloc] init];

////////////////////////////////////Emoji图片显示////////////////////////////////////
    
    /**
        先对出现emoji字符串的range单独处理，如: [熊猫] 这样的字符串
        其他range范围的字符串直接添加到 new富文本中
        对出现emoji的range不再添加到 new富文本中，使用 占位符\uFFFC 来代替，并追加到 new富文本中最后的位置
        等处理完emoji之后再次进行处理
            - Link
            - @..
            - #...#
     */
    
    // 匹配emoji [熊猫]
	NSRegularExpression *exp_emoji =
    [[NSRegularExpression alloc] initWithPattern:kRegexEmoji
                                         options:NSRegularExpressionCaseInsensitive | NSRegularExpressionDotMatchesLineSeparators
                                           error:nil];
    
    // 使用传入的原始NSString匹配emoji正则式
	NSArray *emojis = [exp_emoji matchesInString:string
	                                     options:NSRegularExpressionCaseInsensitive | NSRegularExpressionDotMatchesLineSeparators
	                                       range:NSMakeRange(0, [string length])];

    // 记录前一次找到的emoji范围的起始位置
    NSUInteger location = 0;
    
	for (NSTextCheckingResult *result in emojis) {
		
        // emoji出现的范围
        NSRange range = result.range;
        
        // 两次emoji之间的字串
		NSString *subStr = [string substringWithRange:NSMakeRange(location, range.location - location)];
		NSMutableAttributedString *attSubStr = [[NSMutableAttributedString alloc] initWithString:subStr];
		[newStr appendAttributedString:attSubStr];
        
        // 记录最新一次emoji的起始位置（包含了emoji的长度）
		location = range.location + range.length;
        
        // 取出emoji图片名
		NSString *emojiKey = [string substringWithRange:range];
		NSString *imageName = [[self weiboEmojiDictionary] objectForKey:emojiKey];
        
        // 是否存在对应的emoji图片名
		if (imageName) {
            
			// 这里不用空格，空格有个问题就是连续空格的时候只显示在一行
            //使用 "\uFFFC" 作为插入图片时的占位符
			NSMutableAttributedString *replaceStr = [[NSMutableAttributedString alloc] initWithString:CTAttachmentCharacter];
            
            // 将占位符插入到当前富文本传的最后一个位置
			NSRange __range = NSMakeRange([newStr length], 1);
			[newStr appendAttributedString:replaceStr];
            
			// 定义CTRunDelegate回调函数
			CTRunDelegateCallbacks callbacks;
			callbacks.version = kCTRunDelegateCurrentVersion;
			callbacks.getAscent = ascentCallback;
			callbacks.getDescent = descentCallback;
			callbacks.getWidth = widthCallback;
			callbacks.dealloc = deallocCallback;
            
			// 这里设置下需要绘制的图片的大小，这里我自定义了一个结构体以便于存储数据
			CustomGlyphMetricsRef metrics = malloc(sizeof(CustomGlyphMetrics));
			metrics->ascent = 12;//上行高度12
			metrics->descent = 12 * kAscentDescentScale;//下行高度12*0.25 = 3
            //所以图片高度 = 12 + 3 = 15
			metrics->width = 16;//图片宽度
            
            // 创建CTRunDelegate，并传入上面构造的CTRun参数结构体实例
			CTRunDelegateRef delegate = CTRunDelegateCreate(&callbacks, metrics);
            
            // 给当前占位符出现在富文本的range添加CTRunDelegate
			[newStr addAttribute:(NSString *)kCTRunDelegateAttributeName
			               value:(__bridge id)delegate
			               range:__range];
			CFRelease(delegate);
            
			// 给当前占位符出现在富文本的range添加自定义属性一，CTRun的类型
			[newStr addAttribute:kCustomGlyphAttributeType
			               value:[NSNumber numberWithInt:CustomGlyphAttributeImage]
			               range:__range];
            
			// 给当前占位符出现在富文本的range添加自定义属性二，CTRun如果是图片类型，设置对应的图片名
			[newStr addAttribute:kCustomGlyphAttributeImageName
			               value:imageName
			               range:__range];
		} else {
            
            //[haha]没有找到对应的emoji配置项，则直接将 [haha] 作为文本字符串追加到富文本
			NSString *rSubStr = [string substringWithRange:range];
			NSMutableAttributedString *originalStr = [[NSMutableAttributedString alloc] initWithString:rSubStr];
			[newStr appendAttributedString:originalStr];
		}
	}

////////////////////////////////////第一次emoji字符串range处理完毕之后////////////////////////////////////
    
    // 处理完emoji后的location 小于 传入的带有富文本字符串的NSString >>> 说明NSString中还有  非Emoji字符串 需要处理
    if (location < [string length]) {
        
        // 构建 NSString中 非Emoji字符串 的range范围
        NSRange range = NSMakeRange(location, [string length] - location);
        
        // 取出 NSString中 非Emoji字符串
        NSString *subStr = [string substringWithRange:range];
        
        // 添加到 new富文本末尾，后续等待处理
        NSMutableAttributedString *attSubStr = [[NSMutableAttributedString alloc] initWithString:subStr];
        [newStr appendAttributedString:attSubStr];
    }
        
////////////////////////////////////Link链接处理////////////////////////////////////
    
	// 使用富文本对应的NSString 来匹配 正则式
	NSString *__newStr = [newStr string];
    
	NSRegularExpression *exp_http =
    [[NSRegularExpression alloc] initWithPattern:kRegexShortLink
                                         options:NSRegularExpressionCaseInsensitive | NSRegularExpressionDotMatchesLineSeparators
                                           error:nil];
	NSArray *https = [exp_http matchesInString:__newStr
	                                   options:NSRegularExpressionCaseInsensitive | NSRegularExpressionDotMatchesLineSeparators
	                                     range:NSMakeRange(0, [__newStr length])];
    
    //处理找到的url链接range
	for (NSTextCheckingResult *result in https) {
		NSRange _range = [result range];
        
        /**
         *  给url链接对应的富文本中的range，设置自定义属性
         */
        
        //一、该range的类型是 >>> url链接
		[newStr addAttribute:kCustomGlyphAttributeType
		               value:[NSNumber numberWithInt:CustomGlyphAttributeURL]
		               range:_range];
        
        //二、该range的参数值 >>> 该url链接出现在富文本的range
		[newStr addAttribute:kCustomGlyphAttributeRange
		               value:[NSValue valueWithRange:_range]
		               range:_range];
	}
    
////////////////////////////////////匹配@ 处理////////////////////////////////////
    
    NSRegularExpression *exp_at =
    [[NSRegularExpression alloc] initWithPattern:kRegexCallBody
                                         options:(NSRegularExpressionCaseInsensitive | NSRegularExpressionDotMatchesLineSeparators)
                                           error:nil];
	// 使用富文本对应的NSString 来匹配 正则式
    NSArray *ats =
    [exp_at matchesInString:__newStr
                    options:NSRegularExpressionCaseInsensitive | NSRegularExpressionDotMatchesLineSeparators
                      range:NSMakeRange(0, [__newStr length])];
    
    
    for (NSTextCheckingResult *result in ats) {
        
        NSRange _range = [result range];
        
        // 设置自定义属性，绘制的时候需要用到
        [newStr addAttribute:kCustomGlyphAttributeType
                       value:[NSNumber numberWithInt:CustomGlyphAttributeAt]
                       range:_range];
        
        [newStr addAttribute:kCustomGlyphAttributeRange
                       value:[NSValue valueWithRange:_range]
                       range:_range];
        
        [newStr addAttribute:NSForegroundColorAttributeName
                       value:[UIColor purpleColor]
                       range:_range];
    }
    
////////////////////////////////////#话题# 处理////////////////////////////////////
    NSRegularExpression *exp_topic =
    [[NSRegularExpression alloc] initWithPattern:kRegexTopic
                                         options:NSRegularExpressionCaseInsensitive | NSRegularExpressionDotMatchesLineSeparators
                                           error:nil];
    NSArray *topics =
    [exp_topic matchesInString:__newStr
                       options:NSRegularExpressionCaseInsensitive | NSRegularExpressionDotMatchesLineSeparators
                         range:NSMakeRange(0, [__newStr length])];
    for (NSTextCheckingResult *result in topics) {
        NSRange _range = [result range];
        // 设置自定义属性，绘制的时候需要用到
        [newStr addAttribute:kCustomGlyphAttributeType
                       value:[NSNumber numberWithInt:CustomGlyphAttributeTopic]
                       range:_range];
        [newStr addAttribute:kCustomGlyphAttributeRange
                       value:[NSValue valueWithRange:_range]
                       range:_range];
        [newStr addAttribute:NSForegroundColorAttributeName
                       value:[UIColor purpleColor]
                       range:_range];
    }
    
////////////////////////////////////统一处理最终的富文本属性字符串////////////////////////////////////
    
    //1. 全局range添加统一的字体
    NSRange allTextRange = NSMakeRange(0, [newStr.string length]);
    [newStr addAttribute:NSFontAttributeName value:[UIFont systemFontOfSize:kContentTextSize] range:allTextRange];
    
    // 2. 段落样式
    NSMutableParagraphStyle *paragraphStyle = [[NSMutableParagraphStyle alloc] init];
    paragraphStyle.alignment = NSTextAlignmentLeft;
    paragraphStyle.lineSpacing = kLineSpacing;
    [newStr addAttribute:NSParagraphStyleAttributeName value:paragraphStyle range:allTextRange];
    
	return newStr;
}

@end
```

小结如上代码

- 对一个富文本字符串处理时，其实最重要的就是使用`正则式`将内容对应的`range`找到，然后做一些额外的处理:
	- 添加文字效果修饰...
	- 保存一个额外的参数值，方面在drawRect:绘制富文本字符串时，取出额外的参数值，进行区分处理富文本显示
		- 图片名
		- 出现在富文本中的range，用来点击文字处理

- 使用CTRunDelegate回调告诉CoreText，给当前CTRun空出一块多大的区域来绘制图片

###接下来就是自定义一个UIView，重写其drawRect:方法，内部是CoreText完成emoji图片、链接、@小明、#话题#的绘制，以及结合Touch事件完成文字点击处理

```objc
#import <UIKit/UIKit.h>

//两个CFRange是否有交集
NS_INLINE Boolean CFRangesIntersect(CFRange range1, CFRange range2) {
    CFIndex max_location = MAX(range1.location, range2.location);
    CFIndex min_tail = MIN(range1.location + range1.length, range2.location + range2.length);
    return (min_tail - max_location > 0) ? TRUE : FALSE;
}

//NSRange >>> CFRange
NS_INLINE CFRange CFRangeFromNSRange(NSRange source) {
    return CFRangeMake(source.location, source.length);
}

//location是否包含在Range内
NS_INLINE Boolean CFLocationInRange(CFIndex loc, CFRange range) {
    return (!(loc < range.location) && (loc - range.location) < range.length) ? TRUE : FALSE;
}

/**
 *  文字对应类型点击处理
 */
@protocol CoretextViewDelegate <NSObject>

@optional

// 链接点击
- (void)touchedURLWithURLStr:(NSString *)urlStr;

// @xiong点击
- (void)touchedURLWithAtStr:(NSString *)atStr;

// #话题#点击
- (void)touchedURLWithTopicStr:(NSString *)topicStr;

@end

/**
 *  封装绘制图像、文字的UIView
 */
@interface CoretextView : UIView

@property (nonatomic, weak) id<CoretextViewDelegate> c_delegate;

@property (nonatomic, strong) NSMutableAttributedString *attributedString;

/**
 *  给定一个最大显示宽度，计算富文本最终需要显示的大小
 */
+ (CGSize)adjustSizeWithAttributedString:(NSAttributedString *)attributedString maxWidth:(CGFloat)width;

@end
```

```objc
#import "CoretextView.h"

#import <CoreText/CoreText.h>

//记录点击的第几个ctrun
const CFIndex kNoTouchIndex = -1;

//默认的错误点击point，相对于CoreText坐标系
const CGPoint kErrorPoint = {.x = CGFLOAT_MAX, .y = CGFLOAT_MAX};

//将UIKit坐标转成CoreText坐标（x值是一样的，只是y值不同，UIKit左上角0，CoreText左下角0）
NS_INLINE CGPoint CGPointFlipped(CGPoint point, CGRect bounds) {
    return CGPointMake(point.x, CGRectGetMaxY(bounds) - point.y);
}

/**
 
 touch_range: 点击区域落在链接区域内
 run_range: CTRun所处的range

 判断两个事情:
 
 第一个:
    index是否处于touch_range范围内
 
 第二个:
    touch_range是否与run_range有交集
 
 */
static Boolean isTouchRange(CFIndex index, CFRange touch_range, CFRange run_range) {
    
    // 触摸的CTRun的range要包含index
    if ((touch_range.location < index) && (touch_range.location + touch_range.length >= index)) {
        
        // 返回touch_range 与 range 是否有交集
        return CFRangesIntersect(touch_range, run_range);
    } else {
        return FALSE;
    }
}


#import "NSMutableAttributedString+Weibo.h"

@implementation CoretextView {
    
    // 通过phase可以查看当前触摸事件在一个周期中所处的状态，包含如下几种状态:
    /**
         UITouchPhaseBegan（触摸开始） >>> 默认值
         UITouchPhaseMoved（接触点移动）
         UITouchPhaseStationary（接触点无移动）
         UITouchPhaseEnded（触摸结束）
         UITouchPhaseCancelled（触摸取消）
     */
    UITouchPhase touchPhase;
    
    // 触摸开始Point
    CGPoint beginPoint;
    
    // 触摸开始Point
    CGPoint endPoint;
    
    //触摸开始CTRun的偏移量
    CFIndex beginIndex;
    
    //触摸结束CTRun的偏移量
    CFIndex endIndex;
    
    struct {
        unsigned int touchURL : 1;
        unsigned int touchCall : 1;
        unsigned int touchTopic : 1;
    }_delegateFlags;
}

+ (CGSize)adjustSizeWithAttributedString:(NSAttributedString *)attributedString maxWidth:(CGFloat)width {
    CTFramesetterRef framesetter =
    CTFramesetterCreateWithAttributedString((__bridge CFMutableAttributedStringRef)attributedString);
    
    CGSize maxSize = CGSizeMake(width, CGFLOAT_MAX);
    CGSize size = CTFramesetterSuggestFrameSizeWithConstraints(framesetter, CFRangeMake(0, 0), NULL, maxSize, NULL);
    
    CFRelease(framesetter);
    
    return CGSizeMake(floor(size.width) + 1, floor(size.height) + 1);
}

- (void)setAttributedString:(NSMutableAttributedString *)attributedString {
    if (_attributedString != attributedString) {
        _attributedString = attributedString;
    }
    
    [self setNeedsDisplay];
}

- (void)setC_delegate:(id<CoretextViewDelegate>)c_delegate {
    _c_delegate = c_delegate;
    
    if ([c_delegate respondsToSelector:@selector(touchedURLWithURLStr:)]) {
        _delegateFlags.touchURL = true;
    }
    
    if ([c_delegate respondsToSelector:@selector(touchedURLWithAtStr:)]) {
        _delegateFlags.touchCall = true;
    }
    
    if ([c_delegate respondsToSelector:@selector(touchedURLWithTopicStr:)]) {
        _delegateFlags.touchTopic = true;
    }
}

/**
 根据不同的类型进行不同的配置绘制:
    - 图片
    - url链接
    - @...
    - #话题#
 
 在不同的状态下重新绘制:
    - content字符串被修改时
    - 触摸事件发生时
 */
- (void)drawRect:(CGRect)rect {
    
    /**
     *  该方法如果不走，可能当前UI没有大小
     */
 
    CTFramesetterRef framesetter =
    CTFramesetterCreateWithAttributedString((__bridge CFMutableAttributedStringRef)_attributedString);
    CGPathRef path = CGPathCreateWithRect(rect, &CGAffineTransformIdentity);
    CTFrameRef textFrame = CTFramesetterCreateFrame(framesetter, CFRangeMake(0, 0), path, NULL);
    
    // 开始绘图
    CGContextRef context = UIGraphicsGetCurrentContext();
    
    // 翻转绘图上下
    CGAffineTransform flipVertical = CGAffineTransformMake(1, 0, 0, -1, 0, rect.size.height);
    CGContextConcatCTM(context, flipVertical);
    CGContextSetTextDrawingMode(context, kCGTextFill);
    
    // 获取CTFrame中的所有的CTLine
    CFArrayRef lines = CTFrameGetLines(textFrame);
    
    // 每一行CTLine的坐标原点
    CGPoint origins[CFArrayGetCount(lines)];
    CTFrameGetLineOrigins(textFrame, CFRangeMake(0, 0), origins);
    
    // 找到开始触摸点与结束触摸点
    [self findBeginOrEndTouchPointWithCTLines:lines CTLineOrigins:origins ContextRect:rect];
    
    // 绘制Emoji图片、连接、@小敏、#话题#
    [self drawAllKindsOfDataWithCTLines:lines CTLineOrigins:origins Context:context ContextRect:rect];
    
    // 释放内存
    CFRelease(framesetter);
    CGPathRelease(path);
    CFRelease(textFrame);
}

/**
 *  根据当前触摸事件的状态，找到:
    - 开始触摸点Point 和 beginIndex（记录所处line的第几个run，是开始run）
    - 结束触摸点Point 和 endIndex（记录所处line的第几个run，是结束run）
 */
- (void)findBeginOrEndTouchPointWithCTLines:(CFArrayRef)lines CTLineOrigins:(CGPoint *)origins ContextRect:(CGRect)rect{
    
    for (CFIndex i = 0; i < CFArrayGetCount(lines); ++i)
    {
        //一行line
        CTLineRef line = CFArrayGetValueAtIndex(lines, i);
        
        //该line所有的run块
        CFArrayRef runs = CTLineGetGlyphRuns(line);
        
        for (CFIndex j = 0; j < CFArrayGetCount(runs); ++j)
        {
            //每一个run
            CTRunRef run = CFArrayGetValueAtIndex(runs, j);
            
            //run的上行高度
            CGFloat ascent;
            
            //run的下行高度
            CGFloat descent;
        
            //获取run的宽度
            CGFloat width = CTRunGetTypographicBounds(run, CFRangeMake(0, 0), &ascent, &descent, NULL);
            
            //获取run的高度
            CGFloat height = ascent + descent;
            
            //当前run在当前line中的x偏移量
            CGFloat xOffset = CTLineGetOffsetForStringIndex(line, CTRunGetStringIndicesPtr(run)[0], NULL);
            
            //当前run的x值 = 当前line.x + 当前run在当前line的x偏移值
            CGFloat x = origins[i].x + xOffset;
            
            //当前run的y值
            CGFloat y = origins[i].y - descent;
            
            //得到最终相对CoreText坐标系原点（左下角为0，0）的frame（eg、{0, 300, 15, 20}）
            CGRect runBounds = CGRectMake(x, y, width, height);
            
            //区分触摸开始 与 触摸结束
            if (touchPhase == UITouchPhaseBegan) {
                
                //开始触摸点 UIKit坐标系Point >>> CoreText坐标系Point
                CGPoint mirrorPoint = CGPointFlipped(beginPoint, rect);
                
                //开始触摸点 是否在run的区域内
                if (CGRectContainsPoint(runBounds, mirrorPoint)) {
                    
                    //获取被点击的run是所在line的从左至右第几个
                    beginIndex = CTLineGetStringIndexForPosition(line, mirrorPoint);
                }
            } else if (touchPhase == UITouchPhaseEnded) {
                
                //结束触摸点 UIKit坐标系Point >>> CoreText坐标系Point
                CGPoint mirrorPoint = CGPointFlipped(endPoint, rect);
                
                //结束触摸点 是否在run的区域内
                if (CGRectContainsPoint(runBounds, mirrorPoint)) {
                    
                    //获取被点击的run是所在line的从左至右第几个
                    endIndex = CTLineGetStringIndexForPosition(line, mirrorPoint);
                }
            }
        }
    }

}

/**
 *  根据上面方法完找到的触摸开始、触摸结束数据，完成具体的图片、文字等绘制
 
 */
- (void)drawAllKindsOfDataWithCTLines:(CFArrayRef)lines CTLineOrigins:(CGPoint *)origins Context:(CGContextRef)context ContextRect:(CGRect)rect
{
    //遍历每一行CTLine中的每一个CTRun进行图片绘制
    for (CFIndex i = 0; i < CFArrayGetCount(lines); ++i) {
        
        // 获取CTLine中的CTRun
        CTLineRef line = CFArrayGetValueAtIndex(lines, i);
        
        // 获取CTLine的 上线高度、下行高度
        //【重要】之前在attributeString中对应的range添加CTRunDelegate告诉CoreText显示当前CTRun需要多宽、多高
        CGFloat lineAscent;
        CGFloat lineDescent;
        CTLineGetTypographicBounds(line, &lineAscent, &lineDescent, NULL);
        
        // 获取CTLine所有的CTRun
        CFArrayRef runs = CTLineGetGlyphRuns(line);
        
        // 遍历当前CTLine每一个CTRun，分别按类型处理:
        //一、Emoji 图片
        //二、URL链接
        //三、@hahahahaha
        //四、#话题#
        for (CFIndex j = 0; j < CFArrayGetCount(runs); ++j) {
            
            // 遍历每一个CTRun
            CTRunRef run = CFArrayGetValueAtIndex(runs, j);
            
            // 获取当前CTRun对应的NSRange范围
            CFRange range = CTRunGetStringRange(run);
            
            // 当前line的坐标原点
            CGPoint linePoint = origins[i];
            
            //【重要】告诉context中要绘制文本的（x, y）坐标
            CGContextSetTextPosition(context, linePoint.x, linePoint.y);
            
            // 获取CTRun的属性字典（包含了run所在range内，给attributeString设置的各种key、value）
            NSDictionary *attDic = (__bridge NSDictionary *)CTRunGetAttributes(run);
            
            // 获取该range的CTRun之前设置的类型
            NSNumber *num = [attDic objectForKey:kCustomGlyphAttributeType];
            
            // 判断是否是上面四种我们规定的类型
            if (num) {
                
                // 不管是绘制链接还是表情，我们都需要知道绘制区域的大小，所以我们需要计算下
                
                //【重要】 如果该range配置了CTRunDelegate，就会调用回调函数获取需要显示的宽度、
                CGFloat width = CTRunGetTypographicBounds(run, CFRangeMake(0, 0), NULL, NULL, NULL);
                
                //【重要】 获取CTLine的上下行高度之和就是当前CTRun的高度
                CGFloat height = lineAscent + lineDescent;
                
                // 得到当前CTRun相对CoreText坐标系的（x，y）坐标
                CGFloat xOffset = CTLineGetOffsetForStringIndex(line, CTRunGetStringIndicesPtr(run)[0], NULL);
                CGFloat x = origins[i].x + xOffset;
                CGFloat y = origins[i].y - lineDescent;
                
                // 由上得到当前CTRun最终相对CoreText坐标系的frame
                CGRect runBounds = CGRectMake(x, y, width, height);
                
                // 当前CTRun对应我们定义的类型
                int type = [num intValue];
                
                // 如果是绘制: 链接 、@.. 、#话题#， 这三种情况，因为都是属于文字类型
                if (CustomGlyphAttributeURL <= type && type <= CustomGlyphAttributeTopic) {
                    
                    // 取出构建attributeString时设置的range
                    NSValue *value = [attDic valueForKey:kCustomGlyphAttributeRange];
                    NSRange _range = [value rangeValue];
                    
                    // NSRange >>> CFRange
                    CFRange linkRange = CFRangeFromNSRange(_range);
                    
                    // 我们先绘制背景，不然文字会被背景覆盖
                    if (touchPhase == UITouchPhaseBegan && isTouchRange(beginIndex, linkRange, range)) {
                        
                        //触摸开始....
                        
                        // 填充当前CTRun所处区域的背景颜色
                        CGContextSetFillColorWithColor(context, [UIColor lightGrayColor].CGColor);
                        CGContextFillRect(context, runBounds);
                        
                    } else {
                        
                        //触摸结束....
                        
                        // 记录 触摸开始点与触摸结束点，是否处于同一个CTRun对应的range内
                        BOOL isSameRange = NO;
                        
                        // 判断 触摸开始点 与 触摸结束点，是否处于同一个CTRun对应的range内
                        if (isTouchRange(beginIndex, linkRange, range)) {
                            // 先判断触摸开始点 是否落在 链接range区域内
                            
                            //清除CTRun的背景选中颜色
                            CGContextSetFillColorWithColor(context, [UIColor clearColor].CGColor);
                            CGContextFillRect(context, runBounds);
                            
                            
                            //计算 beginIndex & endIndex 是否都处于linkRange内
                            isSameRange = isTouchRange(endIndex, linkRange, range);
                        }
                        
                        // 将UIKit坐标系的触摸结束点Point相对当前画布Rect转换成CoreText坐标系的Point
                        CGPoint mirrorPoint = CGPointFlipped(endPoint, rect);
                        
                        // 触摸结束完成回调时需具备如下三个条件:
                        //一、触摸结束
                        BOOL isTouchEnd = touchPhase == UITouchPhaseEnded;
                        //二、触摸的UIkit坐标转成成CoreText坐标后，出现在CTRun的rect内
                        BOOL isContainTouchPoint = CGRectContainsPoint(runBounds, mirrorPoint);
                        //三、触摸开始于触摸结束，都是落在当前CTRun的range内
                        if (isTouchEnd && isContainTouchPoint && isSameRange)
                        {
                            
                            if (type == CustomGlyphAttributeURL) {
                                //URL点击
                                if (_delegateFlags.touchURL) {
                                    [_c_delegate touchedURLWithURLStr:[_attributedString.string substringWithRange:_range]];
                                }
                            } else if (type == CustomGlyphAttributeAt) {
                                //@...点击
                                if (_delegateFlags.touchCall) {
                                    [_c_delegate touchedURLWithAtStr:[_attributedString.string substringWithRange:_range]];
                                }
                            } else if (type == CustomGlyphAttributeTopic) {
                                //#话题#点击
                                if (_delegateFlags.touchTopic) {
                                    [_c_delegate touchedURLWithTopicStr:[_attributedString.string substringWithRange:_range]];
                                }
                            } else {
                                NSAssert(NO, @"no this type");
                            }
                        }
                    }
                    
                    // 这里需要绘制下划线，记住CTRun是不会自动绘制下滑线的
                    // 即使你设置了这个属性也不行
                    // CTRun.h中已经做出了相应的说明
                    // 所以这里的下滑线我们需要自己手动绘制
                    CGContextSetStrokeColorWithColor(context, [UIColor blueColor].CGColor);
                    CGContextSetLineWidth(context, 0.5);
                    CGContextMoveToPoint(context, runBounds.origin.x, runBounds.origin.y);
                    CGContextAddLineToPoint(context, runBounds.origin.x + runBounds.size.width, runBounds.origin.y);
                    CGContextStrokePath(context);
                    
                    // 绘制文字
                    CTRunDraw(run, context, CFRangeMake(0, 0));
                    
                } else if (type == CustomGlyphAttributeImage) {
                    // 如果是绘制emoji图片
                    
                    // 得到要显示的emoji图片
                    NSString *imageName = [attDic objectForKey:kCustomGlyphAttributeImageName];
                    UIImage *image = [UIImage imageNamed:imageName];
                    
                    // 直接使用绘制图片的context绘制图片
                    CGContextDrawImage(context, runBounds, image.CGImage);
                }
            } else {
                
                // 没有特殊处理的时候我们只进行文字的绘制
                CTRunDraw(run, context, CFRangeMake(0, 0));
            }
        }
    }

}

#pragma mark - 触摸事件

// 触摸开始
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event {
    //1. 转换点击的Point
    UITouch *touch = [touches anyObject];
    CGPoint point = [touch locationInView:self];
    
    //2. 记录触摸开始点
    beginPoint = point;
    
    //3. 记录当前触摸状态
    touchPhase = touch.phase;
    
    //4. 标记当前触摸的的CTRun处于line的坐标是-1，即没有
    beginIndex = kNoTouchIndex;
    
    //5. 绘图
    [self setNeedsDisplay];
}

// 触摸移动
- (void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event {
    
    //1.
    UITouch *touch = [touches anyObject];
    CGPoint point = [touch locationInView:self];
    
    //2.
    touchPhase = touch.phase;
    

    if (!CGRectContainsPoint(self.bounds, point)) {
        [self touchesCancelled:touches withEvent:event];
    }
}

// 触摸结束
- (void)touchesEnded:(NSSet *)touches withEvent:(UIEvent *)event {
    UITouch *touch = [touches anyObject];
    endPoint = [touch locationInView:self];
    touchPhase = touch.phase;
    endIndex = kNoTouchIndex;
    
    [self setNeedsDisplay];
}

// 触摸取消
- (void)touchesCancelled:(NSSet *)touches withEvent:(UIEvent *)event {
    UITouch *touch = [touches anyObject];
    touchPhase = touch.phase;
    endPoint = kErrorPoint;
    endIndex = kNoTouchIndex;
    
    [self setNeedsDisplay];
}

#pragma mark - UIView事件

// 被添加到UIView 或 UI被隐藏时
- (void)willMoveToWindow:(UIWindow *)newWindow {
    [super willMoveToWindow:newWindow];
    
    //传入window为nil，就是说被隐藏或移除的时候
    if (!newWindow) {
        touchPhase = UITouchPhaseCancelled;
        endPoint = kErrorPoint;
        endIndex = kNoTouchIndex;
        
        [self setNeedsDisplay];
    }
}

@end
```

最后是ViewController测试代码

```objc
@implementation CoreTextViewController2

- (void)viewDidLoad {
    [super viewDidLoad];
    
NSString *text = @"AAhttp://t.cn/123QHz http://t.cn/1er6Hz [兔子][熊猫][给力][浮云][熊猫]   http://t.cn/1er6Hz   \
    [熊猫][熊猫][熊猫][熊猫] Hello World 你好世界[熊猫][熊猫]熊@猫熊猫[熊猫] #iOS# http://t.cn/123QHz http://t.cn/1er6Hz [兔子][熊猫][给力][浮云][熊猫]   http://t.cn/1er6Hz   \
    [熊猫][熊猫][熊猫][熊猫] Hello World 你好世界[熊猫][熊猫]熊@猫熊猫[熊猫] #iOS#http://t.cn/123QHz http://t.cn/1er6Hz [兔子][熊猫][给力][浮云][熊猫]   http://t.cn/1er6Hz   \
    [熊猫][熊猫][熊猫][熊猫] Hello World 你好世界[熊猫][熊猫]熊@猫熊猫[熊猫] #iOS#http://t.cn/123QHz http://t.cn/1er6Hz [兔子][熊猫][给力][浮云][熊猫]   http://t.cn/1er6Hz   \
    [熊猫][熊猫][熊猫][熊猫] Hello World 你好世界[熊猫][熊猫]熊@猫熊猫[熊猫] ";
    
    NSMutableAttributedString *newText = [NSMutableAttributedString weiboAttributedStringWithString:text];
    
    self.textView = [[CoretextView alloc] init];
    self.textView.layer.borderWidth = 1;
    self.textView.c_delegate = self;
    self.textView.backgroundColor = [UIColor whiteColor];
    [self.view addSubview:self.textView];
    
    self.textView.attributedString = newText;
    
    //给定最大显示宽度200，得到计算后准确Size
    CGSize size = [CoretextView adjustSizeWithAttributedString:newText maxWidth:200];
    CGRect frame = CGRectMake(50, 70, size.width, size.height);
    
    self.textView.frame = frame;
}

@end
```

小结对拍本的富文本字符串的点击事件处理:

- 触摸Touch事件回调中，做的事情:
	- 记录触摸开始的Point >>> 转换成CoreText坐标系Point
	- 更新当前触摸事件的状态
	- 【最重要】通知当前 [UIView对象 drawRect:]重新绘制图像

- drawRect:回调中，做的事情:
	- 每一次进入绘图，都重新计算富文本显示的CTFrame区域
	- 开启一个UIKit绘图上下文，并且翻转绘图上下文
	- 获取CTFrame中的所有的CTLine，以及该line中的所有的CTRun
		- 比较当前触摸点击开始Point，处于哪一个CTRun的range内
		- 且必须满足触摸结束Point与触摸开始Point处于同一个CTRun的range内
		- 如果满足如上条件，那么就取出选中的CTRun的range对应NSAttributedString的subStringWithRange:部分字符串，通过delegate回传给外部
	- **绘制Emoji图片、连接、@小敏、#话题#**

***

###对上述代码稍加修改，支持图片点击

- 修改处一、构建富文本NSAttributedString时保存图片的range

```objc
[newStr addAttribute:kCustomGlyphAttributeRange
               value:[NSValue valueWithRange:__range]
               range:__range];
```

- 修改处二、增加图片点击的回调方法

```objc
@protocol CoretextViewDelegate <NSObject>

@optional

...

// 图片点击
- (void)touchedURLWithImage:(NSString *)imageName;

@end
```

- 修改处三、drawRect:绘制图像文字方法中，增加对图片的触摸结束时的处理，顺便把前面的代码逻辑整理了一下改为如下

```objc
- (void)drawAllKindsOfDataWithCTLines:(CFArrayRef)lines CTLineOrigins:(CGPoint *)origins Context:(CGContextRef)context ContextRect:(CGRect)rect
{
    //遍历每一行CTLine中的每一个CTRun进行图片绘制
    for (CFIndex i = 0; i < CFArrayGetCount(lines); ++i)
    {
        // 获取CTLine中的CTRun
        CTLineRef line = CFArrayGetValueAtIndex(lines, i);
        
        // 获取CTLine的 上线高度、下行高度
        //【重要】之前在attributeString中对应的range添加CTRunDelegate告诉CoreText显示当前CTRun需要多宽、多高
        CGFloat lineAscent;
        CGFloat lineDescent;
        CTLineGetTypographicBounds(line, &lineAscent, &lineDescent, NULL);
        
        // 获取CTLine所有的CTRun
        CFArrayRef runs = CTLineGetGlyphRuns(line);
        
        // 遍历当前CTLine每一个CTRun，分别按类型处理:
        //一、Emoji 图片
        //二、URL链接
        //三、@hahahahaha
        //四、#话题#
        for (CFIndex j = 0; j < CFArrayGetCount(runs); ++j)
        {
////////////////////////////////////获取每一个CTRun的数据项///////////////////////////////////////////
            
            // 遍历每一个CTRun
            CTRunRef run = CFArrayGetValueAtIndex(runs, j);
            
            // 获取当前CTRun对应的NSRange范围
            CFRange ctrunRange = CTRunGetStringRange(run);
            
            // 当前line的坐标原点
            CGPoint linePoint = origins[i];
            
            //【重要】告诉context中要绘制文本的（x, y）坐标
            CGContextSetTextPosition(context, linePoint.x, linePoint.y);
            
            // 获取CTRun的属性字典（包含了run所在range内，给attributeString设置的各种key、value）
            NSDictionary *attDic = (__bridge NSDictionary *)CTRunGetAttributes(run);
            
            // 获取该range的CTRun之前设置的类型
            NSNumber *num = [attDic objectForKey:kCustomGlyphAttributeType];
            
            // 判断是否是上面四种我们规定的类型
            if (num) {
                
                /* 不管是绘制链接还是表情，我们都需要知道绘制区域的大小，所以我们需要计算下 */
                
                //【重要】 如果该range配置了CTRunDelegate，就会调用回调函数获取需要显示的宽度、
                CGFloat width = CTRunGetTypographicBounds(run, CFRangeMake(0, 0), NULL, NULL, NULL);
                
                //【重要】 获取CTLine的上下行高度之和就是当前CTRun的高度
                CGFloat height = lineAscent + lineDescent;
                
                // 得到当前CTRun相对CoreText坐标系的（x，y）坐标
                CGFloat xOffset = CTLineGetOffsetForStringIndex(line, CTRunGetStringIndicesPtr(run)[0], NULL);
                CGFloat x = origins[i].x + xOffset;
                CGFloat y = origins[i].y - lineDescent;
                
                // 由上得到当前CTRun最终相对CoreText坐标系的frame
                CGRect runBounds = CGRectMake(x, y, width, height);

///////////////////////////根据触摸开始与触摸结束，并按照CTRun的不同类型不同事件处理//////////////////////////////
                
                // 当前CTRun对应我们定义的类型
                int runType = [num intValue];
                
                // 取出构建attributeString时设置的range
                NSValue *value = [attDic valueForKey:kCustomGlyphAttributeRange];
                NSRange _range = [value rangeValue];
                CFRange richTextRange = CFRangeFromNSRange(_range);
                
                if (touchPhase == UITouchPhaseBegan && isTouchRange(beginIndex, richTextRange, ctrunRange)) {
                    //触摸开始
                    
                    if (CustomGlyphAttributeURL <= runType && runType <= CustomGlyphAttributeTopic) {
                        //链接 、@.. 、#话题# 这三种ctrun的处理
                        
                        // 【重要】先绘制背景，不然文字会被背景覆盖
                        CGContextSetFillColorWithColor(context, [UIColor lightGrayColor].CGColor);
                        CGContextFillRect(context, runBounds);
                        
                        
                        
                    } else {
                        //图片的ctrun处理
                        //暂时不做处理....
                    }
                    
                } else {
                    //除开触摸开始状态的其余状态
                    
                    // 找到触摸开始点与触摸结束点，是否处于同一个CTRun对应的range内的标志位
                    BOOL isSameRange = NO;
                    if (isTouchRange(beginIndex, richTextRange, ctrunRange)) {
                        
                        if (isTouchRange(beginIndex, richTextRange, ctrunRange)) {

                            // 如果run是显示文字，就清除CTRun的背景选中颜色
                             if (CustomGlyphAttributeURL <= runType && runType <= CustomGlyphAttributeTopic) {
                                 CGContextSetFillColorWithColor(context, [UIColor clearColor].CGColor);
                                 CGContextFillRect(context, runBounds);
                             }

                            //计算 beginIndex & endIndex 是否都处于linkRange内
                            isSameRange = isTouchRange(endIndex, richTextRange, ctrunRange);
                        }
                    }
                    
                    //
                    
                    // 将UIKit坐标系的触摸结束点Point相对当前画布Rect转换成CoreText坐标系的Point
                    CGPoint mirrorPoint = CGPointFlipped(endPoint, rect);
                    
                    // 触摸结束完成回调时需具备如下三个条件:
                    //一、触摸结束
                    BOOL isTouchEnd = touchPhase == UITouchPhaseEnded;
                    //二、触摸的UIkit坐标转成成CoreText坐标后，出现在CTRun的rect内
                    BOOL isContainTouchPoint = CGRectContainsPoint(runBounds, mirrorPoint);
                    //三、触摸开始于触摸结束，都是落在当前CTRun的range内
                    
                    if (isTouchEnd && isContainTouchPoint && isSameRange) {
                        if (runType == CustomGlyphAttributeURL) {
                            //URL点击
                            if (_delegateFlags.touchURL) {
                                [_c_delegate touchedURLWithURLStr:[_attributedString.string substringWithRange:_range]];
                            }
                        } else if (runType == CustomGlyphAttributeAt) {
                            //@...点击
                            if (_delegateFlags.touchCall) {
                                [_c_delegate touchedURLWithAtStr:[_attributedString.string substringWithRange:_range]];
                            }
                        } else if (runType == CustomGlyphAttributeTopic) {
                            //#话题#点击
                            if (_delegateFlags.touchTopic) {
                                [_c_delegate touchedURLWithTopicStr:[_attributedString.string substringWithRange:_range]];
                            }
                        } else if (runType == CustomGlyphAttributeImage) {
                            //图片点击
                            if (_delegateFlags.touchImage) {
                                NSString *imageName = [attDic valueForKey:kCustomGlyphAttributeImageName];
                                [_c_delegate touchedURLWithImage:imageName];
                            }
                        } else {
                            //TODO: 暂不支持其他类型
                        }
                    }
                }
                
///////////////////////////管管触摸事件状态每次都进行绘制//////////////////////////////
                
                if (CustomGlyphAttributeURL <= runType && runType <= CustomGlyphAttributeTopic) {
                    // 这里需要绘制下划线，记住CTRun是不会自动绘制下滑线的
                    // 即使你设置了这个属性也不行
                    // CTRun.h中已经做出了相应的说明
                    // 所以这里的下滑线我们需要自己手动绘制
                    CGContextSetStrokeColorWithColor(context, [UIColor blueColor].CGColor);
                    CGContextSetLineWidth(context, 0.5);
                    CGContextMoveToPoint(context, runBounds.origin.x, runBounds.origin.y);
                    CGContextAddLineToPoint(context, runBounds.origin.x + runBounds.size.width, runBounds.origin.y);
                    CGContextStrokePath(context);
                    
                    // 绘制文字（链接、@、#话题#）
                    CTRunDraw(run, context, CFRangeMake(0, 0));
                    
                } else {
                    // 绘制empji图片
                    
                    NSString *imageName = [attDic objectForKey:kCustomGlyphAttributeImageName];
                    UIImage *image = [UIImage imageNamed:imageName];
                    CGContextDrawImage(context, runBounds, image.CGImage);
                }
                
            } else {
                
                // 没有特殊处理的时候我们只进行文字的绘制
                CTRunDraw(run, context, CFRangeMake(0, 0));
                
            }//end-if
        }//end-for
    }//end-for
}
```

效果图如下:

![](http://i4.piimg.com/f48ab8ca8089d3cc.jpg)


****

###再加一个小功能，触摸文字给文字底部画下划线

我看了下掌阅App对文字画下划线之后关掉App，下次打开App这个电子书的这个章节的这一页时仍然显示之前画的下划线，那说明是将对于画下划线的富文本NSMutableAttributedString/NSAttributedString保存到了磁盘本地，可以使用`文件名/文件属性`来保存如下几个参数:

- 哪一个小说
- 小说哪一个章节
- 章节的哪一个页
- 处理的哪一个range的内容

NSAttributedString已经实现了`NSSecureCoding协议`，所以直接可以被归档到磁盘文件.

- 归档

```objc
NSString *text = @"AAhttp://t.cn/123QHz http://t.cn/1er6Hz [兔子][熊猫][给力][浮云][熊猫]   http://t.cn/1er6Hz   \
    [熊猫][熊猫][熊猫][熊猫] Hello World 你好世界[熊猫][熊猫]熊@猫熊猫[熊猫] #iOS# http://t.cn/123QHz http://t.cn/1er6Hz [兔子][熊猫][给力][浮云][熊猫]   http://t.cn/1er6Hz   \
    [熊猫][熊猫][熊猫][熊猫] Hello World 你好世界[熊猫][熊猫]熊@猫熊猫[熊猫] #iOS#http://t.cn/123QHz http://t.cn/1er6Hz [兔子][熊猫][给力][浮云][熊猫]   http://t.cn/1er6Hz   \
    [熊猫][熊猫][熊猫][熊猫] Hello World 你好世界[熊猫][熊猫]熊@猫熊猫[熊猫] #iOS#http://t.cn/123QHz http://t.cn/1er6Hz [兔子][熊猫][给力][浮云][熊猫]   http://t.cn/1er6Hz   \
    [熊猫][熊猫][熊猫][熊猫] Hello World 你好世界[熊猫][熊猫]熊@猫熊猫[熊猫] ";
    
NSString *path = [NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES) objectAtIndex:0];
path = [path stringByAppendingPathComponent:@"haha"];
    

NSMutableAttributedString *attr = [[NSMutableAttributedString alloc] initWithString:text];
NSData *data = [NSKeyedArchiver archivedDataWithRootObject:attr];
[data writeToFile:path atomically:YES];
```

- 解档

```objc
NSMutableAttributedString *attr = [NSKeyedUnarchiver unarchiveObjectWithFile:path];
NSLog(@"attr = %@", attr);
```

***

###接下来就是如何实现长按手势时对文字进行画下划线

对上面代码稍作修改

- 首先记录触摸移动的两个CFRunIndex，一个触摸开始，一个触摸结束或移动

```objc
@interface CoretextView ()

@property (nonatomic) NSInteger selectionStartPosition;
@property (nonatomic) NSInteger selectionEndPosition;

@end
```

CoreTextView在init时，添加长按手势

```objc
- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        
        UIGestureRecognizer *longPressRecognizer = [[UILongPressGestureRecognizer alloc] initWithTarget:self
                                                                                                 action:@selector(userLongPressedGuestureDetected:)];
        [self addGestureRecognizer:longPressRecognizer];
    }
    return self;
}
```

- 长按手势回调函数

```objc
- (void)userLongPressedGuestureDetected:(UILongPressGestureRecognizer *)recognizer {
    CGPoint point = [recognizer locationInView:self];
    
    //【重要】根据当前触摸的UIKit坐标系Point，找到对应的CFRunIndex
    CFIndex curRunIndex = [self getCTRunIndexWithPoint:point];
    
    //CFRunIndex当CTLine往下时，是累加的
    //CFRunIndex当CTLine往上时，是递减的
    if (!(curRunIndex > -1 && curRunIndex <= _attributedString.length)) return;
    
    //
    if (recognizer.state == UIGestureRecognizerStateBegan)
    {
        //开始
        _selectionStartPosition = [self getCTRunIndexWithPoint:point];
        
    } else if (recognizer.state == UIGestureRecognizerStateChanged) {
        //移动
        _selectionEndPosition = curRunIndex;
    }
    
    //使用两个CFRunIndex创建一个CFRange
    CFRange range = CFRangeMake(_selectionStartPosition, _selectionEndPosition - _selectionStartPosition);
    
    //CFRange >>> NSRange
    NSRange __range = NSRangeFromCFRange(range);
    
    //给富文本属性字符串添加下划线属性
    [_attributedString addAttribute:NSUnderlineStyleAttributeName value:[NSNumber numberWithInteger:NSUnderlineStyleSingle] range:__range];
    
    //不断的重新绘图
    [self setNeedsDisplay];
}
```

- 上面代码有几个问题:

	- 只考虑CTLine不断往下递增，没有考虑CTLine往上递减的情况
		- CTLine往下递增，CTRun的index是递增的
		- CTLine往上递减，CTRun的index是递减的
	
	- 借助了一个函数`getCTRunIndexWithPoint:`完成
		- 根据UIKit坐标系的一个触摸Point，找到对应的CTRun的index


- 根据UIKit坐标系的一个触摸Point，找到对应的CTRun的index

```objc
- (CFIndex)getCTRunIndexWithPoint:(CGPoint)point {
    
    CTFrameRef aTextFrame = textFrame;
    CFArrayRef lines = CTFrameGetLines(aTextFrame);
    if (!lines) {
        return -1;
    }
    CFIndex count = CFArrayGetCount(lines);
    
    // 获得每一行的origin坐标
    CGPoint origins[count];
    CTFrameGetLineOrigins(aTextFrame, CFRangeMake(0,0), origins);
    
    // 翻转坐标系的形变
    CGAffineTransform transform =  CGAffineTransformMakeTranslation(0, self.bounds.size.height);
    transform = CGAffineTransformScale(transform, 1.f, -1.f);
    
    CFIndex idx = -1;
    for (int i = 0; i < count; i++) {
        
        // 获取CTFrame中的每一行CTLine、及其左边原点
        CGPoint linePoint = origins[i];
        CTLineRef line = CFArrayGetValueAtIndex(lines, i);
        
        // 获得每一行的CGRect信息
        CGRect flippedRect = [self getCTLineBounds:line point:linePoint];
        
        // 将得到的CTLine的基于CoreText坐标系的rect，翻转UIkit的绘图的坐标系的rect
        CGRect rect = CGRectApplyAffineTransform(flippedRect, transform);
        
        // 再判断是否包括UIKit下的触摸点Point
        if (CGRectContainsPoint(rect, point)) {
        
            // 将点击的坐标转换成相对于当前行的坐标
            CGPoint relativePoint = CGPointMake(point.x-CGRectGetMinX(rect),
                                                point.y-CGRectGetMinY(rect));
            
            // 获得当前UIKit下的触摸点Point，对应整个CTFrame中的CTRun的index
            //CTLine往下，CTRun index 递增
            //CTLine往上，CTRun index 递减
            idx = CTLineGetStringIndexForPosition(line, relativePoint);
        }
    }
    
    return idx;
}
```

- 最后一个是根据UIKit触摸Point，返回一个对应的CTLine的CGRect的函数

```objc
- (CGRect)getCTLineBounds:(CTLineRef)line point:(CGPoint)point {
    
    // 获取CTRun的CGRect，但是是基于CoreText坐标系的
    CGFloat ascent = 0.0f;
    CGFloat descent = 0.0f;
    CGFloat leading = 0.0f;
    
    CGFloat width = (CGFloat)CTLineGetTypographicBounds(line, &ascent, &descent, &leading);
    
    CGFloat height = ascent + descent;
    
    return CGRectMake(point.x, point.y - descent, width, height);
}
```

上面的效果只能往下，下图是继续优化后的效果，大家可以自己去整整....

![coreText.gif](http://imgchr.com/images/coreText.gif)

****

###看完后感觉CoreText图文混排主要就是如下几点:

- 使用`正则式`过滤富文本字符串中的对应内容的range，然后做一些额外的处理

- 使用CFRunDelegate告诉CoreText腾出CTRun显示图片的区域

- UIView的Touch事件的结合
	- 记录一些触摸状态、开始or结束Point、等状态数据
	- [UIView对象 setNeedsDisplay]不断的重新绘制图像

- 然后就是一些CoreText的c语言的api使用
	- CTFrameRef、CTFrameSetter
	- CTLineRef
	- CTRunRef
	- 然后就是一些文字排版样式设置
		- CTFontRef
		- CTParagraphStyle
			- 行距
			- 字距
			- 段落缩进
			- 等等一些

- CoreText库 与 CoreGraphics库 的分工
	- 前者，只是负责某一个具体字（字符编码）到对应的渲染出来的字形（图片）的解释过程
	- 后者，负责具体将图片文字渲染到iOS系统的window上

***

###学习资料来源

```
http://www.tuicool.com/articles/jEBrq2B
http://blog.csdn.net/mangosnow/article/details/37700553
http://www.cocoachina.com/industry/20131028/7250.html
http://blog.guitapro.com/ios-coretext-render-system/
```
