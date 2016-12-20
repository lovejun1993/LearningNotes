---
layout: post
title: "重构代码六、业务逻辑代码"
date: 2016-01-27 16:28:44 +0800
comments: true
categories: 
---

###针对量大、复杂的业务逻辑性判断、处理代码进行比较好的隔离，有很多的好处:

- 需要直接将某一块业务处理，直接拖到一个其他的工程，能够直接使用
- 多个业务逻辑实现版本更替
- 帮助新进员工快速熟悉业务代码流程

***

###改进一、抽象业务逻辑 与 具体业务逻辑 

- 业务接口抽象

```objc
#import <Foundation/Foundation.h>

/**
 *  业务处理类回调获取登陆验证码
 */
typedef NSStrig* (^SMSCodeGetBlock)(void);

/**
 *  登陆业务抽象
 */
@protocol LoginBusinessInterface <NSObject>


- (void)loginWithUsername:(NSStrig *)name
                 password:(NSStrig *)password
             successBlock:(void (^)(void))success
                failBlock:(void (^)(void))fail;

@end

```
- 业务具体实现

```objc
#import <Foundation/Foundation.h>
#import "LoginBusinessInterface.h"

@interface LoginViewModelV1_0 : NSObject <LoginBusinessInterface>

/**
 *  由UI设置
 */
@property (nonnull, nonatomic, copy) SMSCodeGetBlock sMSCodeGetBlock;

@end
```

```objc
#import "LoginViewModelV1_0.h"

@implementation LoginViewModelV1_0

- (void)loginWithUsername:(id)name
                 password:(id)password
             successBlock:(void (^)(void))success
                failBlock:(void (^)(void))fail
{
    //1. 执行block 获取sms code
    NSString *smsCode = nil;
    if (self.smsCodeGetBlock) {
        smsCode = self.smsCodeGetBlock();
    }
    
    //2. 验证登陆
    //用户名
    //密码
    //短信验证码
}

@end
```

- ViewController/View 依赖 `抽象接口`

```objc
#import "ViewController.h"

//导入抽象接口
#import "LoginBusinessInterface.h"

@interface ViewController ()

//UI
@property (nonatomic, strong) UITextField *usernameFiled;
@property (nonatomic, strong) UITextField *passwordFiled;
@property (nonatomic, strong) UITextField *smsFiled;

//业务
@property (nonatomic, strong) id<LoginBusinessInterface> loginViewModel;

@end

@implementation ViewController

- (void)loginButtonClick:(id)sender {
    
    //1.
    NSString *username = self.usernameFiled.text;
    NSString *password = self.passwordFiled.text;
    
    //2. 预先设置获取sms code 的block
    [_loginViewModel setSmsCodeGetBlock:^() {
        return self.smsFiled.text;
    }];
    
    //3.
    [_loginViewModel loginWithUsername:username
                              password:password
                          successBlock:^{
                            //...
                          } failBlock:^{
                            //...
                          }];
}

@end
```

- 对如上可以对 `接口获取实现类对象` 做成依赖注入

参考开源类库 `Objection`

***

###改进二、禁止 业务逻辑判断 与 业务逻辑处理 混在一起

####因为当业务代码很多很复杂时，如果将如下`两个方面`的代码揉在一起，那么对后期新进员工熟悉代码就造成很大的影响

- 第一种、逻辑步骤判断的代码
- 第二种、对应某一种逻辑之后的处理代码

####那么必须将如上两种类型代码分开

```objc
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    //一、逻辑性选择
    if (满足业务条件一) {
        [self _handle业务1];
    } else if (满足业务条件二) {
        [self _handle业务2];
    }
    //....很多复杂的业务代码
}

//二、具体某一种的逻辑的处理
- (void)_handle业务1 {
    ....
}

- (void)_handle业务2 {
    ....
}

@end
```
