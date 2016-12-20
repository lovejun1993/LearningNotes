---
layout: post
title: "DesignMode、Strategy"
date: 2014-01-17 16:21:54 +0800
comments: true
categories: 
---


###策略、将多种条件路径if-elseif中的`业务逻辑`，封装成一个单独的策略实现类.

```
if (满足条件1) {
	执行逻辑1
} else if (满足条件2) {
	执行逻辑2
} else if (满足条件3) {
	执行逻辑3
} else if (满足条件4) {
	执行逻辑4
} else if (满足条件5) {
	执行逻辑5
} else if (满足条件6) {
	执行逻辑6
} else if (满足条件7) {
	执行逻辑7
}
```

###下面模拟一个`不同等级会员，对书原价打折`的例子.

***

###使用if-else-if的代码

```objc
if (低级会员) {
	return 原价 * 0.95;
} else if (中级会员) {
	return 原价 * 0.80;
} else if (高级会员) {
	return 原价 * 0.60;
}
```
###有一些问题:

- 随着更多的新的会员等级增加，那么这个if-elseif会越来越长...
	- 这个问题如果每一个条件都是`不同的情况`，那么无可避免
	
- 每一个if里面的`执行逻辑`，没有`复用性`
	- 对应每一个if的逻辑，就很有必要单独封装，提高复用性
***

###抽象会员等级打折接口

```objc
//
//  MemberStrategy.h
//  demos
//

#import <Foundation/Foundation.h>

/**
 *  抽象会员
 */
@protocol MemberStrategy <NSObject>

/**
 *  计算当前会员，打折后的图书价格
 */
- (float)calcPrice:(float)price;

@end
```


###低级会员、单独逻辑

```objc
//
//  PrimaryMemberStrategy.h
//  demos
//

#import <Foundation/Foundation.h>

//实现会员接口
#import "MemberStrategy.h"

@interface PrimaryMemberStrategy : NSObject <MemberStrategy>

@end
```

```objc
//
//  PrimaryMemberStrategy.m
//  demos
//

#import "PrimaryMemberStrategy.h"

@implementation PrimaryMemberStrategy

- (float)calcPrice:(float)price {
    
    //低级会员，打95折
    return price * 0.95;
}

@end
```

###中级会员、单独逻辑

```objc
//
//  MediateMemberStrategy.h
//  demos
//

#import <Foundation/Foundation.h>

//实现会员接口
#import "MemberStrategy.h"

@interface MediateMemberStrategy : NSObject <MemberStrategy>

@end
```

```objc
//
//  MediateMemberStrategy.m
//  demos
//

#import "MediateMemberStrategy.h"

@implementation MediateMemberStrategy

- (float)calcPrice:(float)price {
    
    //中级会员，打8折
    return price * 0.80;
}


@end
```

###高级会员、单独逻辑

```objc
//
//  AdvancedMemberStrategy.h
//  demos
//

#import <Foundation/Foundation.h>

//实现会员接口
#import "MemberStrategy.h"

@interface AdvancedMemberStrategy : NSObject <MemberStrategy>

@end
```

```objc
//
//  AdvancedMemberStrategy.m
//  demos
//

#import "AdvancedMemberStrategy.h"

@implementation AdvancedMemberStrategy

- (float)calcPrice:(float)price {
    
    //高级会员，打6折
    return price * 0.60;
}

@end
```