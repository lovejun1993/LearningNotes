---
layout: post
title: "DesignMode、Command"
date: 2014-01-13 13:12:26 +0800
comments: true
categories: 
---

###命令模式、

将如下两个部分的工作分割开来

- 一、谁发出的命令
- 二、谁来执行命令

也就是说将二者分开:

- 命令发出者
- 接收到一个命令后的处理者

命令模式的大概过程:

- 首先对一条命令进行封装，包含:
	- 指定命令的接收者
	- 指定命令要执行的处理

- 命令发送方，发送一个命令

- 命令接收方接收到一个命令，直接执行命令，而接收方
	- 不必关心 命令是谁发送的
	- 只需要直接 让命令Invoke执行即可，做自己应该的处理即可

***

###命令模式需要的角色

- 角色一、命令
- 角色二、命令接收者
- 角色三、命令与接收者的对于关系的管理者

***

###`命令接收者`的抽象

```objc
//
//  Receiver.h
//  demos
//
//  Created by xiongzenghui on 14/2/1.
//  Copyright © 2014年 xiongzenghui. All rights reserved.
//

#import <Foundation/Foundation.h>

/**
 *  命令接收者
 */
@protocol Receiver <NSObject>

// 接收到命令后的处理
- (void)handle;

@end
```

###`命令`的抽象

```objc
//
//  Command.h
//  demos
//
//  Created by xiongzenghui on 14/2/1.
//  Copyright © 2014年 xiongzenghui. All rights reserved.
//

#import <Foundation/Foundation.h>

//依赖一个接收者
#import "Receiver.h"

/**
 *  命令
 */
@protocol Command <NSObject>

/**
 *  命令需要绑定一个接受者对象作为处理者
 */
- (void)bindReceiver:(id<Receiver>)receiver;

/**
 *  命令执行
 */
- (void)execute;

@end
```

###`命令与接收者的管理者`抽象

```objc
//
//  Invoker.h
//  demos
//
//  Created by xiongzenghui on 14/2/1.
//  Copyright © 2014年 xiongzenghui. All rights reserved.
//

#import <Foundation/Foundation.h>

//包含一个命令
#import "Command.h"

//包含一个接收者
#import "Receiver.h"

@protocol Invoker <NSObject>

/**
 *  绑定 命令与接收者 的关系
 */
- (void)bindCommand:(id<Command>)cmd
           Receiver:(id<Receiver>)receiver;

/**
 *  执行命令
 */
- (void)invoke;

@end
```

下面是抽象的实现部分

###抽象命令默认实现

```objc
//
//  AbstractCommand.h
//  demos
//
//  Created by xiongzenghui on 14/2/1.
//  Copyright © 2014年 xiongzenghui. All rights reserved.
//

#import <Foundation/Foundation.h>

//依赖一个接收者
#import "Receiver.h"

//实现命令接口
#import "Command.h"

@interface AbstractCommand : NSObject <Command>

//持有一个接收者
@property (nonatomic, strong, readonly) id<Receiver> receiver;

@end
```

```objc
//
//  AbstractCommand.m
//  demos
//
//  Created by xiongzenghui on 14/2/1.
//  Copyright © 2014年 xiongzenghui. All rights reserved.
//

#import "AbstractCommand.h"

@implementation AbstractCommand

//绑定接收者
- (void)bindReceiver:(id<Receiver>)receiver {
	//持有接收者对象
    _receiver = receiver;
}

//命令被执行
- (void)execute {
    
    //通知接收者对象处理
    [_receiver handle];
}

@end
```

###具体命令1

```objc
//
//  Command1.h
//  demos
//
//  Created by xiongzenghui on 14/2/1.
//  Copyright © 2014年 xiongzenghui. All rights reserved.
//

#import <Foundation/Foundation.h>

#import "AbstractCommand.h"

@interface Command1 : AbstractCommand

@end
```

```objc
//
//  Command1.m
//  demos
//
//  Created by xiongzenghui on 14/2/1.
//  Copyright © 2014年 xiongzenghui. All rights reserved.
//

#import "Command1.h"

@implementation Command1

@end
```

###具体的命令接收者1

```objc
//
//  Receiver1.h
//  demos
//
//  Created by xiongzenghui on 14/2/1.
//  Copyright © 2014年 xiongzenghui. All rights reserved.
//

#import <Foundation/Foundation.h>

#import "Receiver.h"

@interface Receiver1 : NSObject <Receiver>

@end
```

```objc
//
//  Receiver1.m
//  demos
//
//  Created by xiongzenghui on 14/2/1.
//  Copyright © 2014年 xiongzenghui. All rights reserved.
//

#import "Receiver1.h"

@implementation Receiver1

- (void)handle {
    NSLog(@"Receiver1 正在处理...");
}

@end
```

###具体的绑定器，绑定命令与接收者

```objc
//
//  ComcreatInvoker.h
//  demos
//
//  Created by xiongzenghui on 14/2/1.
//  Copyright © 2014年 xiongzenghui. All rights reserved.
//

#import <Foundation/Foundation.h>

#import "Invoker.h"

@interface ComcreatInvoker : NSObject <Invoker>

/**
 *  持有一个命令，再触发时刻执行命令
 */
@property (nonatomic, strong, readonly) id<Command> command;

@end
```

```objc
//
//  ComcreatInvoker.m
//  demos
//
//  Created by xiongzenghui on 14/2/1.
//  Copyright © 2014年 xiongzenghui. All rights reserved.
//

#import "ComcreatInvoker.h"

@implementation ComcreatInvoker

- (void)bindCommand:(id<Command>)cmd
           Receiver:(id<Receiver>)receiver
{
    //1. 但其概念Invoker对象，持有传入的命令，再后续使用
    _command = cmd;
    
    //2. 让命令 持有一个 接收者
    [_command bindReceiver:receiver];
}

- (void)invoke {
    
    //执行持有的命令
    [_command execute];
}

@end
```

###demo示例

```objc
//
//  CommandTest.m
//  demos
//
//  Created by xiongzenghui on 14/2/1.
//  Copyright © 2014年 xiongzenghui. All rights reserved.
//

#import "CommandTest.h"

//具体命令
#import "Command1.h"

//具体接收者
#import "Receiver1.h"

//具体绑定器
#import "ComcreatInvoker.h"

@implementation CommandTest

- (void)test {
    
    //1. 实例化一个具体命令
    id<Command> command = [Command1 new];
    
    //2. 实例化一个具体接收者（可以使用配置文件动态切换）
    id<Receiver> receiver = [Receiver1 new];
    
    //3. 包装器
    id<Invoker> invoker = [ComcreatInvoker new];
    
    //4. 包装器绑定 命令与接收者
    [invoker bindCommand:command
                Receiver:receiver];
    
    //5. 将这个Invoker对象发送给其他对象
    //...
    
    //6. 接收到这个Invoker对象的对象
    [invoker invoke];
}

@end
```

***

###小结发送命令与接收命令要做的事情:

- 发送方命令的代码处要做的事情
	- 构造 Receiver对象
	- 构造 Command对象
	- 构造 Invoker对象
	- 上面三个对象的关系:
		- Invoker对象 持有 Command对象
		- Command对象 持有 Receiver对象

- 命令接收方要做的事情
	- 只需要关系`-[Receiver handle]`的具体实现逻辑

就是说`命令发送`方`与命令接收处理者`之间通过一个相同的`抽象接口`进行关联.


