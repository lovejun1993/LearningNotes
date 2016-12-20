---
layout: post
title: "APNS远程推送"
date: 2015-08-01 22:58:52 +0800
comments: true
categories: 
---

###推送通知简介

- 此通知不是`NSNotificationCenter`发出的通知`NSNotification`，而是`推送通知`。

- `推送通知`主要用途
	- App 与 APNS苹果服务器 之间的交互
	- 确切的说让`不在前台运行`的App被动触发做一些事情

- 推送通知的分类
	- 1) 远程推送通知（`只能走APNS苹果推送服务器发出的远程网络通知`）
	- 2) 本地推送通知 (自己代码触发的通知)

***

###应用程序的三种运行状态下时通知的执行差异

####获取App当前的运行状态

```
UIApplicationState state = [UIApplication sharedApplication].applicationState

typedef NS_ENUM(NSInteger, UIApplicationState) {

	//1. App运行在前台
    UIApplicationStateActive,
    
    //2. 	App非激活状态（被通知栏挡住，接电话..等）  		
    UIApplicationStateInactive, 
    
    //3. App运行在后台 
    UIApplicationStateBackground
    	
} NS_ENUM_AVAILABLE_IOS(4_0);
```


####App运行在`前台active` 或 `inActive 被通知栏挡住`


- 能够接受到推送通知（远程or本地）

- 推送通知`不会`产生提示框

- 回调函数`直接由系统调用`，`不需要点击就会调用`，如下回调函数:

```
//接收到【本地】通知的回调函数
- (void)application:(UIApplication *)application didReceiveLocalNotification:(UILocalNotification *)notification

//接收到【远程】通知的回调函数
- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler
```

####App运行在`后台backgroud`


- 能够接受到推送通知（远程or本地）

- 推送通知会产生提示框

- 必须手动点击推送通知提示框，然后触发回调函数获取远程通知，如下回调函数:

```
//接收到【本地】通知的回调函数
- (void)application:(UIApplication *)application didReceiveLocalNotification:(UILocalNotification *)notification

//接收到【远程】通知的回调函数
- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler
```

####App还`没有运行`

- 能够接受到推送通知（远程or本地）

- 推送通知会产生提示框

- 必须手动点击推送通知提示框，重新启动App程序，如下回调函数:

```
//App程序启动完毕的回调函数
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
```

- appDidFinishLaunching回调函数中，判断是由 本地通知启动 or 远程通知 启动

```
//1. 点击本地推送通知后启动App

NSDictionary *localNotification = [launchOptions valueForKey:UIApplicationLaunchOptionsLocalNotificationKey];
```

```
//2. 点击远程推送通知后启动App

NSDictionary *remoteNotification = [launchOptions valueForKey:UIApplicationLaunchOptionsRemoteNotificationKey];
```

- 将接收到的通知JSON显示在当前window显示的控制器的push出的子控制器界面

```
//1. 假设要显示通知JSON内容的界面
UIViewController *detailVC = [[UIViewController alloc] init];

//2. 将detailVCpush到最顶层控制器的上面
[[[self.window.rootViewController.childViewControllers firstObject] view] addSubview:detailVC.view];

//注意: [self.window.rootViewController.childViewControllers firstObject]，实际上是取得UINavigationController控制器栈顶的控制器.
```

***

###推送通知时的注意点

```	
1. 不管App是死是活，都会由系统弹出推送通知提示框.

2. 也就是说，接收通知是iOS手机系统自己管理的事情，不是某个App能够做的事情.
```

```
完成推送功能的证书中的`app id`必须是`精确完整`的，不能使用`模糊`的app id
```

```
推送消息JSON消息大小不能超过`2KB`
```

```
标准的JSON消息体格式

{  
    "aps": {  
        "content-available": 1, //此字段是接收后台推送通知时使用 
        "alert": {
        		
	        	"body" : "This is the alert text",  
	        	"action-loc-key" : "PLAY"
        }
        "badge": 1,  
        "sound": "default"  
    }  
    
	//我们自己定义的参数字典
	"userInfo" : {
		//各种key与value
	}
}
```

```
苹果不允许以任何形式获取iPhone设备的UUID`，只能使用苹果规定的`deviceToken`。
```

```
deviceToken代替UUID的作用:

1. 标识`哪一部iPhone设备`
2. iPhone设备上的`哪一个App`
3. 自己在通知消息体JSON设置的参数，区别登陆状态....
```

```
接收到远程通知时显示的`样式`，只能`用户自己在手机上对App进行设置`，App代码中是无法控制的
```

```	
用户可以设置禁止接收APNS对App的远程推送通知
```

****

###iOS8以上，不管是进行`本地推送通知` or `远程推送通知`之前，都必须`注册申请操作权限`，再程序启动时由系统弹出提示框，让用户选择是否进行授权推送操作

![](http://i5.tietuku.com/e5cd2affe65442d4.png)

####不带Action的注册推送通知操作

```
- (void)sendReceiveDeviceTokenRequest {
#if ( __IPHONE_OS_VERSION_MAX_ALLOWED > __IPHONE_7_0 )
    if ([[UIApplication sharedApplication] respondsToSelector:@selector(registerUserNotificationSettings:)])
    {
        //1.
        UIUserNotificationSettings *settings = [UIUserNotificationSettings settingsForTypes:UIUserNotificationTypeBadge|UIUserNotificationTypeSound|UIUserNotificationTypeAlert categories:nil];
        
        //2.
        [[UIApplication sharedApplication] registerUserNotificationSettings:settings];
    }
#else
    
    //1.
    UIRemoteNotificationType myTypes = UIRemoteNotificationTypeBadge | UIRemoteNotificationTypeAlert | UIRemoteNotificationTypeSound;
    
    //2.
    [[UIApplication sharedApplication] registerForRemoteNotificationTypes:myTypes];
#endif
}
```

####带Action的注册推送通知操作

- 第一步、创建包含了多个将要在通知上显示的Action的`UIMutableUserNotificationCategory`实例

```
- (UIMutableUserNotificationCategory *)customNotificationCategory {

	//1.创建消息上面要添加的动作(按钮的形式显示出来)  
    UIMutableUserNotificationAction *action = [[UIMutableUserNotificationAction alloc] init];  
    action.identifier = @"action";//按钮的标示  
    action.title=@"Accept";//按钮的标题  
    action.activationMode = UIUserNotificationActivationModeForeground;//当点击的时候启动程序  
    //    action.authenticationRequired = YES;  
    //    action.destructive = YES;  
      
    UIMutableUserNotificationAction *action2 = [[UIMutableUserNotificationAction alloc] init];  
    action2.identifier = @"action2";  
    action2.title=@"Reject";  
    action2.activationMode = UIUserNotificationActivationModeBackground;//当点击的时候不启动程序，在后台处理  
    action.authenticationRequired = YES;//需要解锁才能处理，如果action.activationMode = UIUserNotificationActivationModeForeground;则这个属性被忽略；  
    action.destructive = YES;  
      
    //2.创建动作(按钮)的类别集合  
    UIMutableUserNotificationCategory *category = [[UIMutableUserNotificationCategory alloc] init];  
    category.identifier = @"alert";//这组动作的唯一标示,推送通知的时候也是根据这个来区分  
    [category setActions:@[action,action2] forContext:(UIUserNotificationActionContextMinimal)]; 
    
    //3.
    return category;
} 
```

- 第二步、修改之前的注册推送的代码，在注册推送时传入`UIMutableUserNotificationCategory 数组`

```
- (void)sendReceiveDeviceTokenRequest {

#if ( __IPHONE_OS_VERSION_MAX_ALLOWED > __IPHONE_7_0 )
    if ([[UIApplication sharedApplication] respondsToSelector:@selector(registerUserNotificationSettings:)]) {
    	
		//改动一: 得到上面方法创建的categry
		UIMutableUserNotificationCategory *categorys = [self customNotificationCategory];
		    
		//改动二: 注册时，传入category
		UIUserNotificationSettings *settings = [UIUserNotificationSettings settingsForTypes:UIUserNotificationTypeBadge|UIUserNotificationTypeSound|UIUserNotificationTypeAlert categories: [NSSet setWithObjects:categorys, nil]];
        
		[[UIApplication sharedApplication] registerUserNotificationSettings:settings];
    }
#else
    UIRemoteNotificationType myTypes = UIRemoteNotificationTypeBadge | UIRemoteNotificationTypeAlert | UIRemoteNotificationTypeSound;
    [[UIApplication sharedApplication] registerForRemoteNotificationTypes:myTypes];
#endif
}
```

![通知上带Actions的效果图](http://i12.tietuku.com/98381c6a790604ca.png)

####注册推送通知操作之后的UIApplicationDelegate定义的回调函数，注意必须是`iOS8+`的sdk才有如下函数，在此回调函数中，继续`注册远程推送操作`


```
#if (__IPHONE_OS_VERSION_MAX_ALLOWED > __IPHONE_7_1)

- (void)application:(UIApplication *)application didRegisterUserNotificationSettings:(UIUserNotificationSettings *)notificationSettings
{
    
    //1. 用户对app通知的设置
    UIUserNotificationSettings *settings = [application currentUserNotificationSettings];
    
    //2. 用户对通知允许的类型（内容、声音、应用图标数字）
    UIUserNotificationType types = [settings types];
    
    //3. 如果types包含应用图标数字的type，那么清除之前的数字
    if (types == 5 || types == 7) {//5和7满足条件
        
        //注册推送时候，清除推送通知badge数字
        application.applicationIconBadgeNumber = 0;
    }
    
    //4. 如下这一句必须写，否则不会执行远程推送，也就获取不到deviceToken
    [application registerForRemoteNotifications];
}

#endif
```

###本地通知

####发送一个本地通知的步骤
	
- 1) 创建`UILocalNotification`实例

- 2) 设置由系统发送这个本地通知的时间`fireDate`

- 3) 配置本地通知的内容：`通知主体、通知声音、图标数字`等
	- timeZone
	- alertBody
	- category
	- soundName
	- region

- 4) 配置通知传递的自定义参数`userInfo`JSON字典（这一步可选）

```
UILocalNotification *notify = ...;
	
notify.userInfo = @{
                	 @"key":@"value"
  	              };
```

****

####本地通知使用示例

- 创建本地通知对象

```
UILocalNotification *notification = [[UILocalNotification alloc] init];  
```

- 这个方法可以设置当前App在以后某个时间触发本地通知（即使App未运行，也会由系统发送这个本地通知）

```
notification.fireDate = [NSDate dateWithTimeIntervalSinceNow:5];  
```

- 设置时区

```
notification.timeZone = [NSTimeZone defaultTimeZone];  
```

- 设置通知播放的mp3

```
notification.soundName = @"xxx.mp3";
```

- 设置通知显示的内容

```
notification.alertBody = @"测试推送的快捷回复";  
```

- 设置应用程序的投标上的badgeValue

```
notification.applicationIconBadgeNumber = 999;
```

- 设置通知每隔一段时间重复的发送

```
notification.repeatInterval = NSCalendarUnitMonth;//每月发一次

注意: NSCalendarUnit枚举变量
```

- 锁屏界面时的Action

```
notification.alertAction = @"锁屏界面时提示用户解锁后能做的事情";
```

- 设置本地通知附带的json参数

```
notification.userInfo = @{
       	                	 @"key":@"value"
          	              };
```
 
-  用这两个方法判断是否注册成功  

```
// NSLog(@"currentUserNotificationSettings = %@",[[UIApplication sharedApplication] currentUserNotificationSettings]);  
//[[UIApplication sharedApplication] isRegisteredForRemoteNotifications];  
```

- 发送本地通知一、在本地通知队列等待App程序来发送，就是说可能会有延迟

```
[[UIApplication sharedApplication]  scheduleLocalNotification:notification];  
```

- 发送本地通知二、立马发送本地通知，不会有延迟

```
[[UIApplication sharedApplication]  presentLocalNotificationNow:notification];
```

- 获取当前程序还`未被`调度的本地通知 （`一旦通知被发送，就会从通知调度队列移除`）

```
[UIApplication sharedApplication].scheduledLocalNotifications;
```

####接收到`本地通知`，分情况走如下代理函数

- App处于`前台`，不会弹出提示框，由系统直接调用如下回调函数
- App处于`后台`，会弹出提示框，需要用户点击提示框之后，才会回调如下函数

```
- (void)application:(UIApplication *)application
didReceiveLocalNotification:(UILocalNotification *)notification {
	
	//1. 判断App程序状态的代码，统一由别处封装
	
	//2. 取出通知中的JSON
	NSDictionary *json = notification.userInfo;
	
	//3. 调用封装好的处理接收到推送通知的方法
	[某个类或对象 handleNotificationInfo:json];
}
```

- App还没有启动，会弹出提示框，需要用户点击提示框之后，才会调用App启动回调函数

```
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
	
	//1. 判断launchOptions字典中是否包含推送通知JSON
	
	//2. 两种流程
	//2.1 正常点击App图标启动的App
	//初始化所有的UI结构
	
	//2.2 由点击推送通知启动的App
	//在2.1的基础上，在栈顶控制器继续push显示通知信息的界面
	
	return YES;
}
```

####点击`本地通知`上的`Actions`做出的处理

> 该方法必须当App在`已经运行`才会调用

```
#if (__IPHONE_OS_VERSION_MAX_ALLOWED > __IPHONE_7_1)

- (void)application:(UIApplication *)application
handleActionWithIdentifier:(NSString *)identifier
forLocalNotification:(UILocalNotification *)notification
  completionHandler:(void (^)())completionHandler
{

	//在其他App界面显示时收到本地消息，下拉消息会有快捷回复的按钮，点击按钮后调用的方法，根据identifier来判断点击的哪个按钮，notification为消息内容  
    if ([identifier isEqulToString:@"action1"]) {
    
    	//点中action1的处理
    	//....
    	
    } else if ([identifier isEqulToString:@"action2"]) {
    	
    	//点中action2的处理    
    	//....
    	
    }

	//如下代码必须要写
    if (completionHandler)
        completionHandler();
}

#endif
```

***

###远程推送

####让服务器能够进行APNS通知推送，需要给服务器推送证书的p12交换证书
- 开发推送证书的p12
- 发布推送证书的p12

####注册完远程推送之后`[application registerForRemoteNotifications]`，那么会回调执行获取deviceToken回调函数（可在`UIApplicationDelegate`找到）
	
- 获取DeviceToken`成功`的回调函数，`将得到的deviceToken发送给到服务器`
	
```
- (void)application:(UIApplication *)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken
{    
    //将当前将得到的deviceToken发送给到服务器
    [_remoteManager registWithDeviceToken:deviceToken];
}
```
	
- 获取DeviceToken`失败`的回调函数
	
```
- (void)application:(UIApplication *)application didFailToRegisterForRemoteNotificationsWithError:(NSError *)error
{
    DLog(@"获取DeviceToken失败 -- Error: %@\n", [error localizedDescription]);
}
```

####接收到`远程通知`，分情况走如下代理函数

- App处于`前台`，不会弹出提示框，由系统直接调用如下回调函数
- App处于`后台`，会弹出提示框，需要用户点击提示框之后，才会回调如下函数
	

```
这个方法建议不使用，当`App程序处于后台`的时候是无法接收到推送信息
	
- (void)application:(UIApplication *)application
didReceiveRemoteNotification:(NSDictionary *)userInfo 
{
	// 调用封装好的处理接收到推送通知的方法，该方法会判断App程序状态分情况处理
	[某个类或对象 handleNotificationInfo:userInfo];
}
```	

```
最好使用这个回调函数代替上面的那个，因为上面那个函数具备如下优点:
1. 当`App程序处于后台`时也可以接收远程通知
2. 并可以`操作界面UIView实例`
3. 系统给出30s的时间对推送的消息进行处理
4. 苹果推荐使用如下这个回调函数处理`远程通知`

注意: 最后必须执行`CompletionHandler`这个Block，告诉iOS系统开始处理更新UI

- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler
{
	
	//代码块1: 处理远程消息
	{
		//1.【重要】调用封装好的处理接收到推送通知的方法
		[某个类或对象 handleNotificationInfo:userInfo];
		
		//2.【重要】在这里可以直接更新App的界面UI显示.
		//修改UI属性的代码
	}
    
    
    //代码块2: 一定要写如下其中一句，告诉iOS系统去更新UI，如下可以告诉SDK三种结果
    {			  
	    //2.1 结果1: 接收到远程通知
		completionHandler(UIBackgroundFetchResultNewData);
		
		//2.2 结果2: 没有接收到远程通知
	  	//completionHandler(UIBackgroundFetchResultNoData);
	  	
	  	//2.3 结果3: 接收远程通知失败
	  	//completionHandler(UIBackgroundFetchResultFailed);
	  }
}
```

- App还没有启动，会弹出提示框，需要用户点击提示框之后，才会调用App启动回调函数

```
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
	
	//1. 判断launchOptions字典中是否包含推送通知JSON
	
	//2. 两种流程
	//2.1 正常点击App图标启动的App
	//初始化所有的UI结构
	
	//2.2 由点击推送通知启动的App
	//在2.1的基础上，在栈顶控制器继续push显示通知信息的界面
	
	return YES;
}
```

####点击`远程通知`上的`Actions`做出的处理

```
#if (__IPHONE_OS_VERSION_MAX_ALLOWED > __IPHONE_7_1)
	
- (void)application:(UIApplication *)application
handleActionWithIdentifier:(NSString *)identifier
forRemoteNotification:(NSDictionary *)userInfo
  completionHandler:(void (^)())completionHandler
{
	//1. 当App运行在后台时收到远程通知
	//2. 下拉消息会有快捷回复的按钮，点击按钮后调用的方法
	//3. 根据identifier来判断点击的哪个按钮
	//4. notification为消息内容  
    
    if ([identifier isEqulToString:@"action1"]) {
    
    	//点中action1的处理
    	//....
    	
    } else if ([identifier isEqulToString:@"action2"]) {
    	
    	//点中action2的处理    
    	//....
    	
    }

	//如下代码必须要写
    if (completionHandler)
        completionHandler();
}

#endif
```

****

###统一处理`推送通知`的代码封装`handleNotificationInfo:`

####根据`App程序状态`与`远程与本地通知`组合起来，总共有`6`种情况

- 情况1: 【接收到`远程通知`】 + 【App在`前`台】
- 情况2: 【接收到`远程通知`】 + 【App在`后`台】
- 情况3: 【接收到`本地通知`】 + 【App在`前`台】
- 情况4: 【接收到`本地通知`】 + 【App在`后`台】
- 情况5: 【接收到`远程通知`】 + 【App`未启动`】
- 情况6: 【接收到`本地通知`】 + 【App`未启动`】

####处理方法A: （可封装到某个单独的类的一个方法中，主要处理App处于`前台`和`后台`）

- 情况1: 【接收到`远程通知`】 + 【App在`前`台】
- 情况2: 【接收到`远程通知`】 + 【App在`后`台】
- 情况3: 【接收到`本地通知`】 + 【App在`前`台】
- 情况4: 【接收到`本地通知`】 + 【App在`后`台】

```
/**
 *	对传入的通知的userInfo字典进行处理
 */
+ (void)handleNotificationInfo:(NSDictionary *)notificationJSON
{
	
	//1. 通知字典必须存在
	if (!notificationJSON) {
		return;
	}
    
	//2. 判断接收到的通知是是【远程通知】or 【本地通知】
	// 远程通知，通知的JSON包含 @"apns" 的key
	// 本地通知，规定通知的JSON不包含 @"apns" 这个key
	NSDictionary *apsInfo = [pushInfo objectForKey:@"aps"];
	
	//3. 判断是否为远程通知or本地通知？
	BOOL isRemote = (apsInfo != nil);
	
	//4. 获取当前App的状态
	UIApplicationState appState = [UIApplication sharedApplication].applicationState;
	
    if (state == UIApplicationStateActive)
     {
		//App处于前台时，如何处理通知...
		//注意: 不会由系统字典弹出提示框
		//处于前台时，接收到通知，通常不会做什么处理
		
    } else if (state == UIApplicationStateBackground)
    {
       //App处于后台时，如何处理通知...
    	//显示到某个控制器界面，显示通知中的数据
    	//在topViewController上push或present其他控制器界面
		
    } else if (state == UIApplicationStateInActive) 
    {
    	//App处于启动但是未激活时，如何处理通知...
    	//...
    }
}
```

- 这个方法可以封装在`BaseViewController`
	- `BaseViewController`初始化时，注册监听`接收远程推送通知`这个通知KEY
	- 然后`BaseViewController`在NSNotificationCenter通知回调函数中，调用一个`子类重写方法`
	- 某个子类ViewController重写的这个方法，用于处理`接收到远程通知`的逻辑

- 如果直接写在AppDelegate中的一个方法，就很简单了...

```
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{

	//对App主界面UI结构进行初始化完毕
	//tabbarController，rootViewController，rootNav...

	//处理由接收到推送通知，而启动App程序（接收到通知时，App未启动）
	//-------------------------------------------------------
	//1. 本地推送通知
	NSDictionary *localNotification = [options valueForKey:UIApplicationLaunchOptionsLocalNotificationKey];
	
	//2. 远程推送通知
	NSDictionary *remoteNotification = [options valueForKey:UIApplicationLaunchOptionsRemoteNotificationKey];
	
	//3. 确定是【本地推送通知】or【远程推送通知】
	NSDictionary *json = nil;
	
	if (localNotification && !remoteNotification) {
		
		//3.1 接收到的是【本地推送通知】
		json = localNotification;
		
		//【重要】调用封装好的处理接收到推送通知的方法
		[某个类或对象 handleNotificationInfo:json];
		
	} else if (!localNotification && remoteNotification) {
		
		//3.2 接收到的是【远程推送通知】
		json = remoteNotification;
	
		//【重要】调用封装好的处理接收到推送通知的方法
		[某个类或对象 handleNotificationInfo:json];
		
	} else {
	
		//3.3 同时接收到【本地推送通知】and【远程推送通知】
		//并且触发启动App程序
		//这里如何处理....
	}
	    
    //4. 每次都注册远程通知，怕deviceToken变化
	 [self registerRemoteNotificationToReceiveDeviceToken];
	
	return YES;
}
```

####处理方法B: 直接在AppDelegate回调函数`application: didFinishLaunchingWithOptions:`中，调用上面封装的方法主要处理`App未启动`的情况

- 情况5: 【接收到`远程通知`】 + 【App`未启动`】
- 情况6: 【接收到`本地通知`】 + 【App`未启动`】

***

###iOS7以上可以支持`后台接收远程通知`后，让处于后台的App也可以更新界面元素
 
- 1) 配置后台接收消息

![](http://i13.tietuku.com/f058c5e9a00cef9e.png)


- 2) 服务器往APNS推送的消息格式必须是如下
	- 关键点: `必须在apns为key的JSON中，加上content-available=1字段`
	
```
{  
    "aps": {  
        "content-available": 1,  
        "alert": "This is the alert text",  
        "badge": 1,  
        "sound": "default"  
    }  
    "acme1" : "bar",
	"acme2" : 42,
	//自他字段
}

```

****

###地理位置的推送

- 使用`CoreLocation`发起定位

```
注意: iOS7以上要求，使用定位相关功能时，必须调用如下代码让用户完成授权

CLLocationManager *manager = [[CLLocationManager alloc] init];  

manager.delegate = self; 
 
//请求用户授权
[manager requestWhenInUseAuthorization];
```

- `用户授权`回调函数，在用户授权成功之后，发送一个`本地的地理通知`

```
- (void)locationManager:(CLLocationManager *)manager didChangeAuthorizationStatus:(CLAuthorizationStatus)status  
   
{  
	//获取授权结果
    BOOL canUseLocationNotifications = (status == kCLAuthorizationStatusAuthorizedWhenInUse);  
    
    //如果已经授权，开始发送一个地理本地通知
    if (canUseLocationNotifications) {  
        [self startShowLocationNotification];  
    }  
}  
```

- 发送一个`本地的地理通知`

```
- (void)startShowLocationNotification  
   
{  
	//1. 创建一个定理位置
    CLLocationCoordinate2D local2D ;  
    local2D.latitude = 123.0;  
    local2D.longitude = 223.0;  
    
    //2. 创建一个本地通知
    UILocalNotification *locNotification = [[UILocalNotification alloc] init];  
    locNotification.alertBody = @"你接收到了";  
    locNotification.regionTriggersOnce = YES;  
    locNotification.region = [[CLCircularRegion alloc] initWithCenter:local2D radius:45 identifier:@"local-identity"];  
    
    //3. 发送一个本地通知
    [[UIApplication sharedApplication] scheduleLocalNotification:locNotification];  
}
```

- 接收到`地理位置`的`本地`的推送通知

```
- (void)application:(UIApplication *)application didReceiveLocalNotification:(UILocalNotification *)notification  
   
{  
    CLRegion *region = notification.region;  
    if (region) 
    {
    	//处理远程推送的地理位置通知
    	//...  
    }  
}  
```

- 如上看出，`地理位置`的推送通知，只能是`本地`通知

***

###使用工具模拟调试APNS远程推送通知

- 完成`模拟远程推送`，可以使用`pushMeBaby`工具，步骤如下:
	- 1) `推送证书的导出的p12交换证书文件`放到`pushMeBaby`工程中
	- 2) 执行代码获取当前调试手机对应的
		- `deviceToken`
	- 2) 将之前获取的`deviceToken`，配置到`pushMeBaby`代码中的`deviceToken`选项
	- 3) 配置`pushMeBaby`代码中的`certificate`的`证书名字`
	- 4) 运行`pushMeBaby`代码，在弹出框中输入消息JSON即可

***