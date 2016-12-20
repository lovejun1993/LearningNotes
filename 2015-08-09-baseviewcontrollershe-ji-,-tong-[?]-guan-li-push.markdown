---
layout: post
title: "BaseViewController设计、统一管理Push"
date: 2015-08-09 22:11:38 +0800
comments: true
categories: 
---


###在BaseViewController提供一个统一使用`push viewController`的方法，来统一管理控制器压栈.

####`BaseViewController.h`

- 每一个ViewController对应一个`唯一的Id`

```
typedef NS_ENUM(NSInteger, ZSYViewControllerId) {
    ZSYHomePage                         = 1,
    ZSYCustomerRegistAndLogin ,
    ZSYCustomerLogin ,
    ZSYCustomerRegist ,
    ZSYCustomerFindPwd ,
    ZSYCustomerIdentiVery ,
    //...
}
```

- 给子类ViewController提供的push方法

```
- (id)pushWithControllerId:(ZSYViewControllerId)controllerId
                 isAnimate:(BOOL)isAnimate;

- (id)pushWithControllerId:(ZSYViewControllerId)controllerId
                     Title:(NSString *)title
                  Argument:(NSDictionary *)argument
                 isAnimate:(BOOL)isAnimate;
```

####`BaseViewController.m`

- 每一个定义的controllerId对应的className定义

```
static NSString *const kCustomerLoginAndRegistViewClassName     = @"ZSYCustomerLoginAndRegistViewController";
static NSString *const kCustomerLoginViewClassName              = @"ZSYCustomerLoginViewController";
static NSString *const kCustomerRegistViewClassName             = @"ZSYCustomerRegistViewController";
static NSString *const kCustomerFindPwdViewClassName            = @"ZSYCustomerFindPwdViewController";
```

- push 方法的实现

```
- (id)pushWithControllerId:(ZSYViewControllerId)controllerId
                 isAnimate:(BOOL)isAnimate
{
    return [self pushWithControllerId:controllerId
                                Title:nil
                             Argument:nil
                            isAnimate:isAnimate];
}

- (id)pushWithControllerId:(ZSYViewControllerId)controllerId
                     Title:(NSString *)title
                  Argument:(NSDictionary *)argument
                 isAnimate:(BOOL)isAnimate {
    
    NSString *className = nil;
    id vcInstance = nil;
    
    //1. 获取controllerId对应的className（比如，我们让ZSYHomePage对应@"ZSYHomePage_RootViewController"）
    className = [ZSYBaseViewController viewControllerclassNameById:controllerId];
    
    //2. 获取当前ViewController实例的，自定义NavigationController实例
    ZSYBaseNavController *nav = [self findNavigationControllerByCurrentVC];
    
	//3. 从找到的的NavigationController的viewController栈中查找，是否已经有这个类的实例
    UIViewController *findVC = [nav pushToViewController:NSClassFromString(className)
                                                Animated:isAnimate];
                                                
	//4. 如果存在，直接返回找到的viewController实例
    if (findVC) {
    
    	 //5. 并填充传入的属性值
        [ZSYBaseViewController p_fillArgumentForInstance:findVC
                                               arguments:argument
                                                   title:title];
        return findVC;
    }
    
	//-------------- 如下从栈内未找到ViewController实例的情况 --------------
		
	//6. 如果是登陆，直接present 不用 push
    if (controllerId == ZSYCustomerRegistAndLogin) {
        vcInstance = [self handleWhenControllerIdIsLoginModules:className
                                                       Argument:argument
                                                         Tiltle:title
                                                      isAnimate:isAnimate];
        
        return vcInstance;
    }
    
    //7. 前面情况都不符合，那么就创建一个实例，并完成差异化创建NavigationBarItem
    vcInstance = [self createNormalViewControllerInstanceExcludeLoginModolesWithControllerId:controllerId
                                                                                   ClassName:className];
    
    //7. 填充实例的属性值（使用KVC）
    vcInstance = [ZSYBaseViewController p_fillArgumentForInstance:vcInstance
                                                        arguments:argument
                                                            title:title];
    
    //8. 如果创建失败，返回nil
    if (!vcInstance) {
        return nil;
    }
    
    //9. 调用UINavigationController的push方法
    [nav pushViewController:vcInstance animated:isAnimate];
    
    return vcInstance;
}
```

- 查找controllerId对应的className

```
+ (NSString *)viewControllerclassNameById:(ZSYViewControllerId)controllerId {
    
    NSString *className = @"";
    
    switch (controllerId) {
            
        case ZSYHomePage: {
            className = kHomePageRootViewClassName;
            break;
        }
            
        case ZSYCustomerRegistAndLogin: {
            className = kCustomerLoginAndRegistViewClassName;
            break;
        }
            
        case ZSYCustomerLogin: {
            className = kCustomerLoginViewClassName;
            break;
        }
            
        case ZSYCustomerRegist: {
            className = kCustomerRegistViewClassName;
            break;
        }
            
        case ZSYCustomerFindPwd: {
            className = kCustomerFindPwdViewClassName;
            break;
        }
            
       //..省略
       
  	}

	return className;
}
```

- 查找当前控制器所在的导航栈（UINavigationController是自己定义的）

```
- (ZSYBaseNavController *)findNavigationControllerByCurrentVC {
    
    ZSYBaseNavController *nav = nil;
    
    if (self.navigationController) {
        if ([self.navigationController isKindOfClass:[ZSYBaseNavController class]]) {
            nav = (ZSYBaseNavController*)self.navigationController;
        }
    }
    
    return nav;
}
```

- 填充控制器需要的属性值，使用的一个第三方KVC框架

```
+ (id)p_fillArgumentForInstance:(id)instance
                      arguments:(NSDictionary *)argument
                          title:(NSString *)title
{
    if (title) {
        [instance setTitle:title];
    }
    
    if (argument) {
        @try {
            [instance mts_setValuesForKeysWithDictionary:argument];
        }
        @catch (NSException *exception) {
            DLog(@"%@ 设置 %@ 出错!\n", instance, argument);
        }
    }
    return instance;
}
```

- 如果当前要跳转的是`登陆控制器`就使用 `presneViewControlelr:`

```
- (id)handleWhenControllerIdIsLoginModules:(NSString *)className
                                  Argument:(NSDictionary *)argument
                                    Tiltle:(NSString *)title
                                 isAnimate:(BOOL)isAnimate
{
    
    id vcInstance = nil;
    
    vcInstance = [self createLoginModulesInstance:className
                                         animated:isAnimate];
    
    vcInstance = [ZSYBaseViewController p_fillArgumentForInstance:vcInstance
                                                        arguments:argument
                                                            title:title];
    
    return vcInstance;
}

- (id)createLoginModulesInstance:(NSString *)className
                        animated:(BOOL)animated
{
    
    id retVal = nil;
    Class myClass = NSClassFromString(className);
    
    if ([[[self class] presentingVC] isKindOfClass:myClass]) {
        return [[self class] presentingVC];
    }
    
    if ([self p_isLoginModule]) {//push
    
        retVal = [[myClass alloc] _initWithRightNone];
        [self.navigationController pushViewController:retVal animated:animated];
        
    } else {//present
        
        retVal = [[myClass alloc] _initWithCloseButton];
        [ZSYBaseViewController presentVC:retVal];
        
    }
    
    return retVal;
}
```

- 完成差异化创建`NavigationBarButtonItem`，来决定UINavigationBarButton的显示样式

```
- (id)createNormalViewControllerInstanceExcludeLoginModolesWithControllerId:(ZSYViewControllerId)controllerId
                                                                  ClassName:(NSString *)className
{
    id instance = nil;
    
    if ([className isEqualToString:className1]) {
    	
    	//实例化右侧按钮 - 地图
        instance = [[NSClassFromString(className) alloc] _initWithMapButton];
        
	} else if ([className isEqualToString:className2]) {
		
		//实例化右侧按钮 - 关于我们
		instance = [[NSClassFromString(className) alloc] _initWithAoutUsButton];
		
	} else if ([className isEqualToString:className3]) {
		
		//实例化右侧按钮 - 搜索
		instance = [[NSClassFromString(className) alloc] _initWithSearchButton];
		
	} else  {
		
		//实例化右侧按钮 - 什么也没有
		instance = [[NSClassFromString(className) alloc] _initWithUseNone];
		
	}
	
	return instance;	
}
```

```
- (ZSYBaseViewController *)_initWithRightNone {
    self = [super init];
    if (self) {
        //不做神马....
    }
    return self;
}

- (ZSYBaseViewController *)_initWithUseButton {
    self = [super init];
    
    if (self) {
        UIBarButtonItem *item = [UIBarButtonItem createNormalButtonItemWithTitle:@"权益使用"
                                                                           Target:self
                                                                         Selector:@selector(useButtonClick:)];
        [self.navigationItem setRightBarButtonItem:item];
    }
    
    return self;
}

- (ZSYBaseViewController *)_initWithAboutButton {
    self = [super init];
    if (self) {
        UIBarButtonItem *item = [UIBarButtonItem createNormalButtonItemWithTitle:@"关于"
                                                                          Target:self
                                                                        Selector:@selector(aboutButtomClick:)];
        [self.navigationItem setRightBarButtonItem:item];
    }
    return self;
}

- (ZSYBaseViewController *)_initWithCloseButton {
    self = [super init];
    if (self) {
        UIBarButtonItem *close = [UIBarButtonItem createGoBackViewControllerButtonItemWithTarget:self
                                                                                        Selector:@selector(closeButtonClick:)];
        [self.navigationItem setLeftBarButtonItem:close];
    }
    return self;
}
```

```
- (void)useButtonClick:(id)sender {
    [self pushWithControllerId:ZSYXFXTBenefitUse isAnimate:YES];
}

- (void)aboutButtomClick:(id)sender {
    
}

- (void)closeButtonClick:(id)sender {
    
    BOOL isNeedForWardToHomePage = [self isForwardToHomePage];
    
    [self dismissViewControllerAnimated:YES completion:nil];

    if (isNeedForWardToHomePage) {
        [[APP_DELEGATE contentVC] popAllToRootViewControllerAnimated:NO];
        
        // CJT:此句代码应该放入到这个位置
        ZSYBaseNavController *nav = (ZSYBaseNavController *)[APP_DELEGATE contentVC].selectedViewController;
        [nav removeCaptureViewWhenLoginExpirateViewClickClose];
    }
}
```

- 还可以继续添加不同控制器的`navigationBar`进行差异化添加不同的UI（搜索栏、下拉框、标题栏...）

	- 通过修改ViewController实例的`navigationItem`模型实例的属性

```
//1. 创建一个输入框
//..

//2. 
self.navigationItem.titleView = 输入框;
```

***

###这样做的好处

- 防止push死循环

- ViewController实例，统一在一个地方完成创建

- 可以在一个地方来完成，对不同的ViewController的导航栏按钮差异化、或者其他的UI差异化设置

- 使用一个Id来代替一个ClassName，外界使用push时，只是使用了一个`枚举值Controllerid`而并不知道具体使用的是哪一个`ViewController Class`，从而达到解耦
    
   
****

###自定义一个UINavigationController也可以完成导航栏按钮、导航栏输入框..等等的差异化样式or统一样式

- 定义一个UINavigationController，重写父类的各种系统方法，来完成我们自己操作之后，再让父类的代码继续执行

```
@interface ZSYBaseNavController : UINavigationController

@end
```

```
@implementation ZSYBaseNavController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    //设置UINavigationControllerDelegate为ZSYBaseNavController实例
    self.delegate = self;
    
}

#pragma mark - UINavigationControllerDelegate

- (void)navigationController:(UINavigationController *)navigationController
      willShowViewController:(UIViewController *)viewController
                    animated:(BOOL)animated 
{
                    
}   

- (void)navigationController:(UINavigationController *)navigationController
       didShowViewController:(UIViewController *)viewController
                    animated:(BOOL)animated 
{                 

}

#pragma mark - 重写系统方法，进行拦截

//拦截pushViewController方法
- (void)pushViewController:(UIViewController *)viewController animated:(BOOL)animated 
{
	//1. 做点什么
	//...
	
	//2, 执行父类的push操作
	[super pushViewController:viewController animated:animated];
}

//拦截popToViewController操作
- (NSArray *)popToViewController:(UIViewController *)viewController animated:(BOOL)animated 
{
	//1. 做点什么
	//...
	
	//2, 执行父类的pop操作
	return [super popToViewController:viewController animated:animated];
}

//拦截popToRootViewControllerAnimated操作
- (NSArray *)popToRootViewControllerAnimated:(BOOL)animated 
{
    //1. 做点什么
	//...
	
	//2, 执行父类的pop操作
    return [super popToRootViewControllerAnimated:animated];
}

//拦截popViewControllerAnimated操作
- (UIViewController *)popViewControllerAnimated:(BOOL)animated 
{
   //1. 做点什么
	//...
	
	//2, 执行父类的pop操作    
	return [super popViewControllerAnimated:animated];
}

//拦截setViewControllers操作
- (void)setViewControllers:(NSArray *)viewControllers animated:(BOOL)animated 
{
    //1. 做点什么
	 //...
	
	//2, 执行父类的set操作
    [super setViewControllers:viewControllers animated:animated];
}

@end
```

- 可以重写如上的`pushViewController:animated:`方法，完成差异化`导航栏UI`、各种UI设置

```
- (void)pushViewController:(UIViewController *)viewController animated:(BOOL)animated {
    
    //1. 给当前push的viewController实例
    // 添加左侧导航按钮、有导航按钮、中间的导航输入框
    //..
    
    //2.    
    [super pushViewController:viewController animated:animated];
}

```

- 仍然重写`pushViewController:animated:`方法，来完成系统TabBarConttroller是否隐藏tabBar

```
- (void)pushViewController:(UIViewController *)viewController animated:(BOOL)animated {
    
    //1. 判断push的viewController实例是不是rootVC
	if (self.veiwControllers.count > 1) {
	
		//不是root viewController，那么添加设置自动隐藏tabBar的设置
		viewController.hidesBottomBarWhenPushed = YES;
	}
    
    //2. 执行父类的push代码  
    [super pushViewController:viewController animated:animated];
```