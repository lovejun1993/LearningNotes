## 分析`VVeboLabel `如何使用CoreText绘制复杂的UI布局

-------


## 使用了两个`UIImageView`作为最终文字渲染生成的Image显示的UI

```objc
@implementation VVeboLabel {
    UIImageView *labelImageView;
    UIImageView *highlightImageView;
    ....
}

```

## `initWithFrame:`添加subviews

```objc
- (id)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
    
		    //1.
		    labelImageView = [[UIImageView alloc] initWithFrame:CGRectMake(0, -5, frame.size.width, frame.size.height+10)];
		    labelImageView.contentMode = UIViewContentModeScaleAspectFit;
		    labelImageView.tag = NSIntegerMin;
		    labelImageView.clipsToBounds = YES;
		    [self addSubview:labelImageView];
		    
		    //2.
		    highlightImageView = [[UIImageView alloc] initWithFrame:CGRectMake(0, -5, frame.size.width, frame.size.height+10)];
		    highlightImageView.contentMode = UIViewContentModeScaleAspectFit;
		    highlightImageView.tag = NSIntegerMin;
		    highlightImageView.clipsToBounds = YES;
		    highlightImageView.backgroundColor = [UIColor clearColor];
		    [self addSubview:highlightImageView];
	    
		    ....
		}
		return self;
}
```


两个ImageView都占据满VVeboLabel的所有区域，并且还大一点。也可以使用lazy。


## 重写`setFrame:`在当前VVeboLabel在某个Cell被用进行使用时，调整内部两个ImageView的frame

```objc
- (void)setFrame:(CGRect)frame{
    
    //1. 如果当前VVeboLabel.szie 不再等于 内部两个ImageView.image.size，
    // 说明此时，需要显示不同的文本渲染得到的Image了
    if (!CGSizeEqualToSize(labelImageView.image.size, frame.size)) {
        
        // 清空之前显示的Image
        labelImageView.image = nil;
        highlightImageView.image = nil;
    }
    
    //2. 重新调整VVeboLabel内部两个ImageView的frame
    labelImageView.frame = CGRectMake(0, -5, frame.size.width, frame.size.height+10);
    highlightImageView.frame = CGRectMake(0, -5, frame.size.width, frame.size.height+10);
    
    //3.
    [super setFrame:frame];
}
```

### 为什么了需要这么做：

- (1) VVeboLabel是添加到Cell上显示的，而Cell是会不断的重用
- (2) 重用到不同NSInexPath时，会显示不同的文本数据，那么渲染得到的Image尺寸不同
- (3) 如果不调整内部的两个ImageView的frame，就会出现显示区域的不正确
- (4) 所以当前VVeboLabel的frame变化后，一定要接着调整内部两个ImageView的frame

总之，就是必须在父亲VVeboLabel自己的frame变化的时候，同步更新内部的所有儿子View的frame。那其实也可以在`layoutSubviews`中做处理:

```objc
- (void)layoutSubviews {
    [super layoutSubviews];
    
    CGRect frame = self.frame;
    
    //1. 如果当前VVeboLabel.szie 不再等于 内部两个ImageView.image.size，
    // 说明此时，需要显示不同的文本渲染得到的Image了
    if (!CGSizeEqualToSize(labelImageView.image.size, frame.size)) {
        
        // 清空之前显示的Image
        labelImageView.image = nil;
        highlightImageView.image = nil;
    }
    
    //2. 重新调整VVeboLabel内部两个ImageView的frame
    labelImageView.frame = CGRectMake(0, -5, frame.size.width, frame.size.height+10);
    highlightImageView.frame = CGRectMake(0, -5, frame.size.width, frame.size.height+10);
}
```

### 当前VVeboLabel在某个Cell被用进行使用时，调整内部两个ImageView的frame，以及清空之前的显示的数据，可以在如下两个时机进行处理

- (1) `setFrame:`
- (2) `layoutSubviews`

## 将文本、图形、图像 的绘制到一个`Context`，然后从`Context`获取渲染完毕的图像Image

### 一、UIKit提供的绘制

- (1) NSString 文本内容

```objc
@interface NSAttributedString(NSStringDrawing)
- (CGSize)size NS_AVAILABLE(10_0, 6_0);
- (void)drawAtPoint:(CGPoint)point NS_AVAILABLE(10_0, 6_0);
- (void)drawInRect:(CGRect)rect NS_AVAILABLE(10_0, 6_0);
@end
```

- (2) UIImage 图像内容

```objc
@interface UIImage : NSObject <NSSecureCoding>

- (void)drawAtPoint:(CGPoint)point;                                                        // mode = kCGBlendModeNormal, alpha = 1.0
- (void)drawAtPoint:(CGPoint)point blendMode:(CGBlendMode)blendMode alpha:(CGFloat)alpha;
- (void)drawInRect:(CGRect)rect;                                                           // mode = kCGBlendModeNormal, alpha = 1.0
- (void)drawInRect:(CGRect)rect blendMode:(CGBlendMode)blendMode alpha:(CGFloat)alpha;

- (void)drawAsPatternInRect:(CGRect)rect; // draws the image as a CGPattern

@end
```

可以看到，直接通过UIKit可以进行`文本`与`图像`的绘制。

但是对于`自定义路径、自定义图形`就无能为力了，这时只能通过`CoreGraphics`来完成了。

### 二、CoreText

能够绘制的任务类型:

- (1) 纯文本
- (2) 图片
- (3) 图文混排

主要涉及到的绘制数据:

- (1) `NSMutableAttributedString`
- (2) `NSAttributedString`
- (3) `NSString`
- (4) `UIImage`

CoreText对上面的原始数据，进行解析得到的内部数据:

- (1) `CTFramesetterRef`
- (2) `CTFrameRef` （必须时，需要进行内存缓存，避免重复对同一段文本进行解析）
- (3) `CTLineRef`
- (4) `CTRunRef`
- (5) `CTFontRef`
- (6) `CTGlyphInfoRef`
- (7) `CFStringRef`

绘制`CTFrameRef`步骤中常用的CoreText函数:

- (1) `CGPath + CTFramesetterRef + NSAttributedString + rect` 生成 `CTFrameRef`

```objc
//1. 根据要绘制区域rect，构建CGPath
CGMutablePathRef path = CGPathCreateMutable();
CGPathAddRect(path, NULL, rect);

//2. 使用CGPath + CTFramesetterRef + NSAttributedString 获取 CTFrameRef
CTFrameRef frame = CTFramesetterCreateFrame(framesetter, textRange, path, NULL);
```

- (2) 获取`所有行`

```objc
CFArrayRef lines = CTFrameGetLines(frame);
NSInteger numberOfLines = CFArrayGetCount(lines);
```

- (3) 获取每一行的开始`range.location`

```objc
CGPoint lineOrigins[numberOfLines];
CTFrameGetLineOrigins(frame, CFRangeMake(0, numberOfLines), lineOrigins);
```

- (4) 设置每一行的开始绘制文本的位置

```objc
CGContextSetTextPosition(ctx, lineOrigin.x, lineOrigin.y);
```

- (5) 获取具体某一行

```objc
CTLineRef line = CFArrayGetValueAtIndex(lines, lineIndex);
```

- (6) 获取某一行的bounds

```objc
CGFloat descent = 0.0f;
CGFloat ascent = 0.0f;
CGFloat lineLeading;
CTLineGetTypographicBounds((CTLineRef)line, &ascent, &descent, &lineLeading);
```

- (7) 获取某一行的range

```objc
CTLineRef line = CFArrayGetValueAtIndex(lines, lineIndex);
CFRange range = CTLineGetStringRange(line);
```

- (8) 使用富文本属性字符串，创建一个CTLineRef实例

```objc
CTLineRef truncationLine = CTLineCreateWithAttributedString((__bridge CFAttributedStringRef)truncationString);
```

- (9) 将某一个CTLineRef使用`...`省略号显示

```objc
NSAttributedString *truncatedString = [[NSAttributedString alloc]initWithString:@"\u2026"];  
CTLineRef token = CTLineCreateWithAttributedString((__bridge CFAttributedStringRef)truncatedString);  
CTLineTruncationType ltt = kCTLineTruncationStart;//kCTLineTruncationEnd;  
CTLineRef newline = CTLineCreateTruncatedLine(line, self.bounds.size.width-200, ltt, token);
```

- (10) 获取CTRunRef的range

```objc
CFRange range = CTRunGetStringRange(run);
```

- (11) 获取CTRunRef的bounds、offset

```objc
CGFloat runAscent;
CGFloat runDescent;
CTRunRef run = CFArrayGetValueAtIndex(runs, j);

CFRange range = CTRunGetStringRange(run);
CGRect runRect;
runRect.size.width = CTRunGetTypographicBounds(run, CFRangeMake(0,0), &runAscent, &runDescent, NULL);
float offset = CTLineGetOffsetForStringIndex(line, range.location, NULL);
float height = runAscent;
runRect=CGRectMake(lineOrigin.x + offset, (self.height+5)-y-height+runDescent/2, runRect.size.width, height);
```

- (12) 绘制

```objc
CTLineDraw(line);
CTFrameDraw(ctframe);
```

- (13) 最后不再使用CoreText实例时，需要释放废弃

```objc
CFRelease(xxx);
```

- (14) 如果需要对文本进行用户交互处理，可以从如下入手:
	- (一) `UIResponder`提供的`touch触摸`回调函数
	- (二) `pointInside:withEvent:`
	- (三) `hitTest:withEvent:`

### 三、CoreGraphics

能够绘制的任务类型:

- (1) 自定义路径
- (2) 自定义图形

主要涉及到的绘制数据类型:

- (1) `CGContext`、`CGPDFContext`、`CGBitmapContext` 画布
- (2) `CGPath` 路径
- (2) `CGColor` 颜色
- (3) `CGFont` 字体
- (4) `CGImage` 图像
- (5) `CGLayer` 图层

### 四、最终Context获取渲染得到的数据

```c
UIImage* UIGraphicsGetImageFromCurrentImageContext(void);
```

复杂时，需要对不同CALayer渲染得到的Image，进行Image的合成。