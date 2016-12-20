---
layout: post
title: "NSCharacterSet"
date: 2014-01-21 23:05:36 +0800
comments: true
categories: 
---


###NSCharacterSet和NSMutableCharacterSet是用来表示一组Unicode字符


***

###常见系统的 字符集组合CharSet

```objc
[NSCharacterSet alphanumericCharacterSet];          //所有数字和字母(大小写)  

[NSCharacterSet decimalDigitCharacterSet];          //0-9的数字  

[NSCharacterSet letterCharacterSet];                //所有字母  

[NSCharacterSet lowercaseLetterCharacterSet];       //小写字母  

[NSCharacterSet uppercaseLetterCharacterSet];       //大写字母  

[NSCharacterSet punctuationCharacterSet];           //标点符号  

[NSCharacterSet whitespaceAndNewlineCharacterSet];  //空格和换行符  

[NSCharacterSet whitespaceCharacterSet];            //空格  
```

还有很多iOS7之后新添加的

```objc
@interface NSCharacterSet (NSURLUtilities)

// Predefined character sets for the six URL components and subcomponents which allow percent encoding. These character sets are passed to -stringByAddingPercentEncodingWithAllowedCharacters:.

// Returns a character set containing the characters allowed in an URL's user subcomponent.
+ (NSCharacterSet *)URLUserAllowedCharacterSet NS_AVAILABLE(10_9, 7_0);

// Returns a character set containing the characters allowed in an URL's password subcomponent.
+ (NSCharacterSet *)URLPasswordAllowedCharacterSet NS_AVAILABLE(10_9, 7_0);

// Returns a character set containing the characters allowed in an URL's host subcomponent.
+ (NSCharacterSet *)URLHostAllowedCharacterSet NS_AVAILABLE(10_9, 7_0);

// Returns a character set containing the characters allowed in an URL's path component. ';' is a legal path character, but it is recommended that it be percent-encoded for best compatibility with NSURL (-stringByAddingPercentEncodingWithAllowedCharacters: will percent-encode any ';' characters if you pass the URLPathAllowedCharacterSet).
+ (NSCharacterSet *)URLPathAllowedCharacterSet NS_AVAILABLE(10_9, 7_0);

// Returns a character set containing the characters allowed in an URL's query component.
+ (NSCharacterSet *)URLQueryAllowedCharacterSet NS_AVAILABLE(10_9, 7_0);

// Returns a character set containing the characters allowed in an URL's fragment component.
+ (NSCharacterSet *)URLFragmentAllowedCharacterSet NS_AVAILABLE(10_9, 7_0);

@end
```

###查找字符串中的一个字符 '.'

```objc
NSCharacterSet *set = [NSCharacterSet characterSetWithRange:NSMakeRange('.', 1)];
    
NSString *str = @"dawd.awdawdwdawd.dawd";
    
NSRange range = [str rangeOfCharacterFromSet:set];
```


###去除一个NSArray中的重复元素

```objc
NSOrderedSet *set1 = [NSOrderedSet orderedSetWithArray:@[@3,@7,@3,@3,@0]];
NSLog(@"%@",set1);
```

```
输出结果:

{(
    3,
    7,
    0
)}
```

###截取一个NSArray的部分元素，同时去除重复元素

```objc
NSArray *array = @[@3,@7,@3,@3,@0];
NSOrderedSet *set2 = [NSOrderedSet orderedSetWithArray:array range:NSMakeRange(0, 4) copyItems:NO];
NSLog(@"%@",set2);
```

```
输出结果:

{(
    3,
    7
)}
```

###将多个NSObject对象组装成NSOrderSet

```objc
NSOrderedSet *set3 = [NSOrderedSet orderedSetWithObjects:@"aaaa",@4, nil];
NSLog(@"%@",set3);
```

```
输出结果:

{(
    aaaa,
    4
)}
```

###从NSOrderSet中查询某个元素

```objc
NSOrderedSet *set1 = [NSOrderedSet orderedSetWithArray:@[@3,@7,@3,@3,@0]];
NSLog(@"%@",set1);

if ([set1 containsObject:@3]) {
    NSLog(@"EXIST");
}
```

```
输出结果:

{(
    3,
    7,
    0
)}

EXIST
```

###NSOrderSet遍历，以及去除重复元素的规则

```objc
NSOrderedSet *set1 = [NSOrderedSet orderedSetWithArray:@[@3,@7,@3,@3,@0]];
    
[set1 enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
    NSLog(@"obj = %@， idx = %ld", obj, idx);

}];
```

```
输出结果:

obj = 3， idx = 0
obj = 7， idx = 1
obj = 0， idx = 2
```

###NSOrderSet的集合运算

```objc
NSOrderedSet *set1 = [NSOrderedSet orderedSetWithObjects:@4,@5,@6,@7,nil];

NSMutableOrderedSet *set2 = [NSMutableOrderedSet orderedSetWithObjects:@8,@9,@10, nil];

//判断 set1中是否存在至少一个元素，是否存在于set2中
if ([set1 intersectsOrderedSet:set2]) {
    NSLog(@"intersectsOrderedSet -- yes");
}

//判断 set1中所有元素，是否存在于set2中（是否是字串）
if ([set1 isSubsetOfOrderedSet:set2]) {
    NSLog(@"isSubsetOfOrderedSet -- yes");
}

//合并两个order set
//可变set主动合并
[set2 unionOrderedSet:set1];
```

```
输出结果:

{(
    8,
    9,
    10,
    4,
    5,
    6,
    7
)}
```

###NSOrderSet过滤元素

```objc
NSOrderedSet *set = [NSOrderedSet orderedSetWithObjects:@3,@1,@8,@6,@12, nil];
    
NSIndexSet *indexs = [set indexesOfObjectsWithOptions:NSEnumerationConcurrent
                                          passingTest:^BOOL(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop)
{
    //返回YES，当前遍历元素会被添加到NSIndexSet
    //返回NO，被过滤掉
    
    if ([obj integerValue] >= 6) {
        return YES;
    } else {
        return NO;
    }
}];
    
NSArray *valueArray = [set objectsAtIndexes:indexs];
    
NSLog(@"%@",valueArray);
```

```
输出结果:

(
    8,
    6,
    12
)
```

###自定义创建NSCharacterSet

```objc
使用给定的字符串组成一个CharSet

[NSCharacterSet characterSetWithCharactersInString:@"Hello"];
```

***

###判断 `数字0 （字符编码为48）`是否存在于 `十进制数的字符集`中

```objc
//1. 十进制数字的字符集
[NSCharacterSet alphanumericCharacterSet]

//2. 判断数字0是否存在于字符集
BOOL isExist = [charSet characterIsMember:48];
```

###NSMutableCharacter的使用

```objc
NSMutableCharacterSet *set1 = [NSMutableCharacterSet characterSetWithCharactersInString:@"Hell"];  
NSMutableCharacterSet *set2 = [NSMutableCharacterSet characterSetWithCharactersInString:@"ello"];  

//去掉某些字符  
[set2 removeCharactersInString:@"e"]; //--->l, o  

//加上某些字符  
[set2 addCharactersInString:@"e"];    //--->e, l, o  

//set相加  
[set2 formUnionWithCharacterSet:set1]; //--->H,e,l,o  

//本身加上另外一个的set相交  
[set2 formIntersectionWithCharacterSet:set1]; //--->H,e,l   

//除以包含的以外的set  
[set2 invert]; 
```

###去除 `两端` 的空格

```objc
//1. 空格的字符集
NSCharacterSet *whiteSpaceSet = [NSCharacterSet whitespaceCharacterSet];

//2. 去除字符集中的字符
[@"  aaa   " stringByTrimmingCharactersInSet: whiteSpaceSet]; 
```

###通过数字把字符串变成数组

```objc
[@"a1aa2aaa3aaaa4aaaaa" componentsSeparatedByCharactersInSet:[NSCharacterSet decimalDigitCharacterSet]];
```

###过滤URL中的特殊字符

```objc
static NSString * AFPercentEscapedStringFromString(NSString *string) {
    
    //对URL中包含的特殊字符（:#[]@）替换成 %
    // does not include "?" or "/" due to RFC 3986 - Section 3.4
    static NSString * const kAFCharactersGeneralDelimitersToEncode = @":#[]@";
    
    static NSString * const kAFCharactersSubDelimitersToEncode = @"!$&'()*+,;=";
    
    //创建一个用于URL的字符集
    NSMutableCharacterSet * allowedCharacterSet = [[NSCharacterSet URLQueryAllowedCharacterSet] mutableCopy];
    
    //:#[]@ + !$&'()*+,;=
    NSString *temp = [kAFCharactersGeneralDelimitersToEncode stringByAppendingString:kAFCharactersSubDelimitersToEncode];
    
    //从字符集中移除上面的所有 特殊字符
    [allowedCharacterSet removeCharactersInString:temp];
    
    // FIXME: https://github.com/AFNetworking/AFNetworking/pull/3028
    // return [string stringByAddingPercentEncodingWithAllowedCharacters:allowedCharacterSet];
    
    //下面是从传入的string中
    static NSUInteger const batchSize = 50;
    
    NSUInteger index = 0;
    NSMutableString *escaped = @"".mutableCopy;
    
    while (index < string.length) {
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wgnu"
        NSUInteger length = MIN(string.length - index, batchSize);
#pragma GCC diagnostic pop
        NSRange range = NSMakeRange(index, length);
        
        // To avoid breaking up character sequences such as 👴🏻👮🏽
        range = [string rangeOfComposedCharacterSequencesForRange:range];
        
        NSString *substring = [string substringWithRange:range];
        
        //从substring中 替换掉 不存在于allowedCharacterSet字符集中的 其他字符
        NSString *encoded = [substring stringByAddingPercentEncodingWithAllowedCharacters:allowedCharacterSet];
        
        [escaped appendString:encoded];
        
        index += range.length;
    }
    
    return escaped;
}
```

###去除单词中的 `多余空格`

```
@"  My     name    is     Johnny!  "

变成

@"My name is Johnny!"
```

实现代码如下，主要使用NSCharSet + NSPredicate + 数组截取、过滤、拼接

```objc
NSString *exampleStr = @"   My    name    is    Johnny!     ";

//1. 去掉两端的空格
exampleStr = [exampleStr stringByTrimmingCharactersInSet:[NSCharacterSet whitespaceCharacterSet]];

//2. 然后以空格字符集截取成数组（很多空格也被截取成一个数组元素）
NSArray *exampleArr = [exampleStr componentsSeparatedByCharactersInSet:[NSCharacterSet whitespaceCharacterSet]];

//3. 创建一个过滤条件: 不等于空字符
NSPredicate *predicate = [NSPredicate predicateWithFormat:@"self <> ''"];

//4. 数组过滤掉空格字符的元素
exampleArr = [exampleArr filteredArrayUsingPredicate:predicate];

//5. 数组组装成字符串
exampleStr = [exampleArr componentsJoinedByString:@" "];
```