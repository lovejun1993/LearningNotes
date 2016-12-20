---
layout: post
title: "NSMutableAttributedString"
date: 2015-08-12 21:50:14 +0800
comments: true
categories: 
---

###NSAttributedString、NSMutableAttributedString是由`<CoreText/CoreText.h>`提供，可以让我们使一个字符串内部同时可以有很多不同颜色、不同大小、下划线、间距...等等丰富的显示样式设置。

***

###列举下可用的 富文本字符串设置属性名字

- 命名结构: `NS...AttributeName`

```
NSFontAttributeName                设置字体属性，默认值：字体：Helvetica(Neue) 字号：12
NSForegroundColorAttributeName      设置字体颜色，取值为 UIColor对象，默认值为黑色
NSBackgroundColorAttributeName     设置字体所在区域背景颜色，取值为 UIColor对象，默认值为nil, 透明色
NSLigatureAttributeName            设置连体属性，取值为NSNumber 对象(整数)，0 表示没有连体字符，1 表示使用默认的连体字符
NSKernAttributeName                设定字符间距，取值为 NSNumber 对象（整数），正值间距加宽，负值间距变窄
NSStrikethroughStyleAttributeName  设置删除线，取值为 NSNumber 对象（整数）
NSStrikethroughColorAttributeName  设置删除线颜色，取值为 UIColor 对象，默认值为黑色
NSUnderlineStyleAttributeName      设置下划线，取值为 NSNumber 对象（整数），枚举常量 NSUnderlineStyle中的值，与删除线类似
NSUnderlineColorAttributeName      设置下划线颜色，取值为 UIColor 对象，默认值为黑色
NSStrokeWidthAttributeName         设置笔画宽度，取值为 NSNumber 对象（整数），负值填充效果，正值中空效果
NSStrokeColorAttributeName         填充部分颜色，不是字体颜色，取值为 UIColor 对象
NSShadowAttributeName              设置阴影属性，取值为 NSShadow 对象
NSTextEffectAttributeName          设置文本特殊效果，取值为 NSString 对象，目前只有图版印刷效果可用：
NSBaselineOffsetAttributeName      设置基线偏移值，取值为 NSNumber （float）,正值上偏，负值下偏
NSObliquenessAttributeName         设置字形倾斜度，取值为 NSNumber （float）,正值右倾，负值左倾
NSExpansionAttributeName           设置文本横向拉伸属性，取值为 NSNumber （float）,正值横向拉伸文本，负值横向压缩文本
NSWritingDirectionAttributeName    设置文字书写方向，从左向右书写或者从右向左书写
NSVerticalGlyphFormAttributeName   设置文字排版方向，取值为 NSNumber 对象(整数)，0 表示横排文本，1 表示竖排文本
NSLinkAttributeName                设置链接属性，点击后调用浏览器打开指定URL地址
NSAttachmentAttributeName          设置文本附件,取值为NSTextAttachment对象,常用于文字图片混排
NSParagraphStyleAttributeName      设置文本段落排版格式，取值为 NSParagraphStyle 对象
```

***

###主要使用的步骤

- 1) 将原始字符串使用`NSMutableAttributedString`包装

```
NSString *targetStr = @"将要多样化显示的原始字符串";

NSMutableAttributedString * targetStr = [[NSMutableAttributedString alloc] initWithString: targetStr];
```

- 2) 开始编辑属性

```
[targetStr beginEditing];
```

- 3) 给富文本字符串添加样式

```
[attString addAttribute:属性名
                      value:属性值
                      range:范围];
```

- 4) 结束编辑属性

```
[targetStr endEditing];
```

- 5) 显示设置后的富文本字符串

```
直接显示给UI
label.attributedText = targetStr;
```

```
使用绘图绘制
[targetStr drawInRect:区域];
```

***

###如下有一个简单的使用例子

```
//原始数据字符串
NSString *string = @"我是好人我是好人我是个好人额鹅鹅鹅鹅鹅鹅饿哇哇哇哇哇哇哇";
    
//1. 创建可变属性化字符串
NSMutableAttributedString *attrString = [[NSMutableAttributedString alloc] initWithString:string];
    
//2.1 设置单个样式
UIFont *smallFont = [UIFont systemFontOfSize:30];
[attrString addAttribute:NSFontAttributeName
                   value:smallFont
                   range:NSMakeRange(5, 10)];
    
//2.2 设置单个样式
UIColor *rcolor = [UIColor redColor];
[attrString addAttribute:NSForegroundColorAttributeName
                   value:rcolor
                   range:[string rangeOfString:@"是"]];
    
//3. 将设置过属性的【富文本字符串】设置给label显示
label.attributedText = attrString;
```

> 小结`富文本字符串`属性三个东西:

- 要设置的富文本`属性名`. 
	- Attribute Key （如: NSForegroundColorAttributeName颜色 ... ）

- 要设置的富文本属性对应的`属性值`.
	- Attribute Value （如: [UIColor redColor] .. id类型）

- 上如富文本设置对应的`范围`. 
	- Attribute Range (如:[string rangeOfString:@"是"]..）

> 小结`NSMutableAttributedString`的常用Api

- 为某一范围内文字设置`多个属性`（做第一次的全部默认的样式设置）

```
- (void)setAttributes:(NSDictionary *)attrs range:(NSRange)range;
```

- 为某一范围内文字添加`单个属性`

```
- (void)addAttribute:(NSString *)name value:(id)value range:(NSRange)range;
```

- 为某一范围内文字添加`多个属性`

```
- (void)addAttributes:(NSDictionary *)attrs range:(NSRange)range;
```

- 移除某范围内的`单个属性`

```
- (void)removeAttribute:(NSString *)name range:(NSRange)range;
```

****

###以下看看有哪一些常用`富文本属性`可以设置

字体颜色、字体大小就不说了...

- 给部分内容设置`下划线`

```
[attString addAttribute:NSUnderlineStyleAttributeName
                      value:[NSNumber numberWithInteger:NSUnderlineStyleSingle]
                      range:[myString rangeOfString:@"learn"]];
```

- 给部分内容设置`删除线`

```
//1. 删除线样式
[attString addAttribute:NSStrikethroughStyleAttributeName
                      value:@(NSUnderlineStyleSingle)
                      range:[myString rangeOfString:@"learn"]];


//2. 删除线的颜色    
[attString addAttribute:NSStrikethroughColorAttributeName
                  value:[UIColor greenColor]
                  range:[myString rangeOfString:@"learn"]];
```

- 给部分内容设置`背景颜色`

```
[attString addAttribute:NSBackgroundColorAttributeName
                      value:[UIColor redColor]
                      range:[myString rangeOfString:@"learn"]];
```

- 给部分内容设置`空心字`

```
//1. 设置空心字的字体颜色
[attString addAttribute:NSStrokeColorAttributeName
                      value:[UIColor greenColor]
                      range:[myString rangeOfString:@"Fast"]];

//2. 设置空心字的字体宽度
[attString addAttribute:NSStrokeWidthAttributeName
                  value:@(5)
                  range:[myString rangeOfString:@"Fast"]];
```

****

###使用NSMutableAttributedString、NSTextAttachment、NSAttributedString完成UITextView内部显示文字与图片简单混排.

```
//2. 判断是Emoji还是其图片表情
if (model.code) {
   //显示Emoji表情: 插入的是文字
    
    //2.1 先对emoji表情十六进制数解码成字符串文字
    NSString *value = [model.code emoji];
    
    //2.2 再insert到textView显示
    [self.contentTextView insertText:value];
    
} else {
    //显示非Emoji表情: 插入的是图片
    
    //2.1 创建一个可变的富文本字符串
    NSMutableAttributedString *mutableAttrStr = [[NSMutableAttributedString alloc] init];
    
    //2.2 拼接之前的内容字符串（带字体的字符串、图片）
    [mutableAttrStr appendAttributedString:self.contentTextView.attributedText];
    
    //2.3 得到要插入的表情图片
    UIImage *image = [UIImage imageNamed:[model png]];
    
    //2.4 使用富文本附件包装显示的Image
    NSTextAttachment *textAttach = [[NSTextAttachment alloc] init];
    textAttach.image = image;
    
    //2.5 设置NSTextAttachment实例显示的宽度和高度
    CGFloat textLineH = self.contentTextView.font.lineHeight;//字体高度
    textAttach.bounds = CGRectMake(0, 0, textLineH, textLineH);
    
    //2.6 使用NSTextAttachment实例，创建富本文属性字符串
    NSAttributedString *textAttachAttString = [NSAttributedString attributedStringWithAttachment:textAttach];
    
    //2.7 可变富文本，拼接带有图片的子富文本
    //[mutableAttrStr appendAttributedString:textAttachAttString];这样会造成全部都是追加在最后面，不能够在中间插入内容
    
    //2.7 获取当前输入光标所在的位置
    NSUInteger location = self.contentTextView.selectedRange.location;
    
    //2.8 将带有图片的子富文本，插入到可变字符串中，输入光标所在的位置
    [mutableAttrStr insertAttributedString:textAttachAttString atIndex:location];
    
    //2.9 给可变富文本设置字体
    [mutableAttrStr addAttribute:NSFontAttributeName
                           value:self.contentTextView.font
                           range:NSMakeRange(0, mutableAttrStr.length)];
    
    //2.10 将最终的富文本字符串设置给TextView显示
    //这一句执行后，会将输入光标移动到末尾
    self.contentTextView.attributedText = mutableAttrStr;
    
    //2.11 调整输入光标到插入内容的后面以为
    self.contentTextView.selectedRange = NSMakeRange(location + 1, 0);
    
}
```

小结:

- 可变字符串的`insertAttributedString:atIndex:`插入其他子富文本字符串

- 当执行`UITextView对象.attributedText = mutableAttrStr;`后，输入光标会默认移动`末尾`

- 获取当前输入光标的位置

```
NSUInteger location = UITextView对象.selectedRange.location;
```

- 修改当前输入光标的位置

```
//1. 计算得到一个起始位置
NSUInteger location = ...;

//2. 设置给TextView
UITextView对象.selectedRange = NSMakeRange(location, 0);//长度是0，如果大于0，就会选中后面的一段文字
```