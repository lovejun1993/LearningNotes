---
layout: post
title: "Runtime Part9  Resolve"
date: 2014-09-24 00:18:44 +0800
comments: true
categories: 
---

###在`没有找到`发送消息中的`SEL`对应的`Method`时，系统会调用指定的系统函数，来处理这种情况.

> 可以是实现方法拦截的思路之一.

本文介绍使用 `Resolve 动态决议`的方式处理 没有找到 SEL对应的方法实现时进行消息转发.


###动态方法决议、可以对`类方法与对象方法`进行拦截

对象方法、没有找到`对象`的方法Method，会由系统执行

```
+ (BOOL)resolveInstanceMethod:(SEL)sel;
```

类方法、没有找到`类`的方法Method，会由系统执行

```
+ (BOOL)resolveClassMethod:(SEL)sel;
```

这两个方法拦执行时，返回YES或NO，告诉系统是否继续执行

- 返回YES，表示继续执行当前SEL对应的方法（可能崩溃）
- 返回NO，进入消息转发阶段二forward

***


###eg. 没有找到对象实现方法时的拦截处理

```
@implementation Person
	
- (void)dynamicAddMethodDemo {
    
    [self performSelector:@selector(noExistMethod:) withObject:@"test"];
}
	

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    
    if ([NSStringFromSelector(sel) isEqualToString:@"noExistMethod:"]) {
        
        // 和本地预先写好的函数实现关联起来
        [self addMethod:sel];
        
        return YES;
    }
    
    return [super resolveInstanceMethod:sel];
}
	
+ (void)addMethod:(SEL)sel {
    class_addMethod(self, sel, (IMP)runAddMethod, "v@:*");
}
	
// 之前就写好的预备处理方法，只需要在运行时添加进来
void runAddMethod(id self, SEL _cmd, NSString *string){
	//当某个SEL未找到方法实现时走的默认方法
}
```	

动态决议适合将预先使用代码写好的函数实现，在运行期进入动态方法决议时，将一些特定的SEL消息转入我们预先写好的函数实现中。

****

