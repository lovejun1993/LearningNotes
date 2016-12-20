---
layout: post
title: "CoreFoundation、CFDictionaryApplyFunction"
date: 2016-02-24 17:31:02 +0800
comments: true
categories: 
---

###遍历一个字典的key和value，不断的调用一个c函数，并传入一个Context实例

****

###Context

```
typedef struct {
    void* object;
    void* value;
}MyConext;
```

###回调c函数

```
static void MyContextCllbackFunc(const void *_key, const void *_value, void *context) {
    NSLog(@"%@", _key);
    NSLog(@"%@", _value);
    NSLog(@"%p", context);
}
```

###示例

```objc
NSDictionary *dict = @{
                           @"key1" : @"value",
                           @"key2" : @"value",
                           @"key3" : @"value",
                           };
    
User *user = [User new];
    
MyConext context = {0};
context.object = (__bridge void *)(user);
context.value = @"value";
    
CFDictionaryApplyFunction((CFDictionaryRef)dict, MyContextCllbackFunc, &context);
```

输出结果

```
2016-02-24 17:30:30.032 XZHRequest[34633:315777] key1
2016-02-24 17:30:30.032 XZHRequest[34633:315777] value
2016-02-24 17:30:30.032 XZHRequest[34633:315777] 0x7fff54de49e0
2016-02-24 17:30:30.033 XZHRequest[34633:315777] key3
2016-02-24 17:30:30.033 XZHRequest[34633:315777] value
2016-02-24 17:30:30.033 XZHRequest[34633:315777] 0x7fff54de49e0
2016-02-24 17:30:30.033 XZHRequest[34633:315777] key2
2016-02-24 17:30:30.033 XZHRequest[34633:315777] value
2016-02-24 17:30:30.033 XZHRequest[34633:315777] 0x7fff54de49e0
```

***

###类似还有CFArrayApplyFunction使用

```
NSMutableArray *array = [NSMutableArray new];
for (int i = 0; i < 3; i++) {
    User *user = [User new];
    user.name = [NSString stringWithFormat:@"%d%d%d", i,i,i];
    [array addObject:user];
}
  
//Context结构体实例  
MyConext context = {0};
context.object = @"hahaha";
context.value = @"hello world...";
    
//遍历Array数组元素，每一次传入一个函数中进行处理
CFArrayApplyFunction((CFArrayRef)array,
                     CFRangeMake(0, CFArrayGetCount((CFArrayRef)array)),
                     MyContextCllbackFunc2,
                     &context);
```

被回调的c函数

```
static void MyContextCllbackFunc2(const void *user, void *context) {
    User *aUser = (__bridge User *)(user);
    MyConext *ctx = context;
    
    NSLog(@"aUser.name = %@", aUser.name);
    NSLog(@"ctx = %p", ctx);
}
```

输出

```
2016-02-24 23:47:18.055 XZHRequest[3064:30469] aUser.name = 000
2016-02-24 23:47:18.055 XZHRequest[3064:30469] ctx = 0x7fff52afea18
2016-02-24 23:47:18.056 XZHRequest[3064:30469] aUser.name = 111
2016-02-24 23:47:18.056 XZHRequest[3064:30469] ctx = 0x7fff52afea18
2016-02-24 23:47:18.059 XZHRequest[3064:30469] aUser.name = 222
2016-02-24 23:47:18.059 XZHRequest[3064:30469] ctx = 0x7fff52afea18
```