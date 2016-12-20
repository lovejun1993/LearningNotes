---
layout: post
title: "DesignMode、State"
date: 2014-01-16 16:11:54 +0800
comments: true
categories: 
---


###状态机、将一个事物的`不同的状态执行的业务逻辑`，单独封装到`一个对应状态实现类`中去.

***

###抽象状态

```objc
//
//  State.h
//  demos
//

#import <Foundation/Foundation.h>

//当前状态对象，需要设置到哪个状态机对象
/** 因为StateObject接口中已经#import了State接口，所以使用@protocol声明 */
@protocol StateObject;

/**
 *  抽象状态
 */
@protocol State <NSObject>

/**
 *  当前状态设置到哪一个状态机
 */
- (void)handleState:(id<StateObject>)object;

@end
```

###抽象状态机

```objc
//
//  StateObject.h
//  demos
//

#import <Foundation/Foundation.h>

//依赖一个状态
#import "State.h"

/**
 *  状态管理器
 */
@protocol StateObject <NSObject>

/**
 *  切换状态
 */
- (void)changeToState:(id<State>)state;

/**
 *  当前状态
 */
- (id<State>)currentState;

@end
```

###默认状态实现（抽象类）

```objc
//
//  AbstactState.h
//  demos
//

#import <Foundation/Foundation.h>

#import "State.h"

@interface AbstactState : NSObject <State>

//单例所有状态对象
+ (instancetype)sharedInstance;

@end
```

```objc
//
//  AbstactState.m
//  demos
//

#import "AbstactState.h"

#import "StateObject.h"

@implementation AbstactState

+ (instancetype)sharedInstance {
    static id state = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        state = [[self alloc] init];
    });
    return state;
}

- (void)handleState:(id<StateObject>)object {
    
    //将新的State设置给状态机对象
    //并由状态机提供对应的状态的附加代码
    [object changeToState:self];
}

@end
```

###具体状态1，提供切到此状态时的附加代码逻辑

```objc
//
//  State1.h
//  demos
//

#import <Foundation/Foundation.h>

#import "AbstactState.h"

@interface State1 : AbstactState

@end
```

```objc
//
//  State1.m
//  demos
//

#import "State1.h"

@implementation State1

- (void)handleState:(id<StateObject>)object {
    
    //1. 执行父类修改state
    [super handleState:object];
    
    //2. 做自己的处理
    //......
}

@end
```

###具体状态2，提供切到此状态时的附加代码逻辑

```objc
//
//  State2.h
//  demos
//

#import <Foundation/Foundation.h>

#import "AbstactState.h"

@interface State2 : AbstactState

@end
```

```objc
//
//  State2.m
//  demos
//

#import "State2.h"

#import "StateObject.h"

@implementation State2

- (void)handleState:(id<StateObject>)object {
    
    //1. 执行父类修改state
    [super handleState:object];
    
    //2. 做自己的处理
    //......
}


@end
```

###具体状态3，提供切到此状态时的附加代码逻辑

```objc
//
//  State3.h
//  demos
//

#import <Foundation/Foundation.h>

#import "AbstactState.h"

@interface State3 : AbstactState

@end
```

```objc
//
//  State3.m
//  demos
//

#import "State3.h"

#import "StateObject.h"

@implementation State3

- (void)handleState:(id<StateObject>)object {
    
    //1. 执行父类修改state
    [super handleState:object];
    
    //2. 做自己的处理
    //......
}


@end
```

###具体状态机1

```objc
//
//  StateObject1.h
//  demos
//

#import <Foundation/Foundation.h>

//实现接口
#import "StateObject.h"

/**
 *  一个具体的状态管理器
 */
@interface StateObject1 : NSObject <StateObject>

@end
```

```objc
//
//  StateObject1.m
//  demos
//

#import "StateObject1.h"

@interface StateObject1 ()

//保存当前状态对象
@property (nonatomic, strong) id<State> state;

@end

@implementation StateObject1

- (void)changeToState:(id<State>)state {
    _state = state;
}

- (id<State>)currentState {
    return _state;
}

@end
```

###demo示例

```objc
//
//  StateTest.m
//  demos
//

#import "StateTest.h"

#import "State1.h"
#import "State2.h"
#import "State3.h"

#import "StateObject1.h"

@implementation StateTest

- (void)test {
    
    //1. 状态机
    id<StateObject> manager = [StateObject1 new];
    
    //2. 所有的状态（附加逻辑）
    id<State> state1 = [State1 sharedInstance];
    id<State> state2 = [State2 sharedInstance];
    id<State> state3 = [State3 sharedInstance];
    
    //3. 切换状态
    [manager changeToState:state1];
    
    //4. 切换状态
    [manager changeToState:state2];
    
    //5. 切换状态
    [manager changeToState:state3];
}

@end
```

###好处: 每一种状态封装对应的处理逻辑.

###缺点: 状态很多时候，实现类也很多.