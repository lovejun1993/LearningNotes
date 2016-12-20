---
layout: post
title: "Reachability源码学习"
date: 2013-12-30 12:08:22 +0800
comments: true
categories: 
---

###实时监测手机的联网状态，并作出对应的回调处理，源码学习子`AFNetworkReachabilityManager`.

***

- 首先定义网络状态的所有可能的状态的枚举

```
typedef NS_ENUM(NSInteger, AFNetworkReachabilityStatus) {
	//未知
    AFNetworkReachabilityStatusUnknown          = -1,
    
    //与App通信的服务器，不可达
    AFNetworkReachabilityStatusNotReachable     = 0,
    
    //与App通信的服务器，可达且通过3G，4G ...
    AFNetworkReachabilityStatusReachableViaWWAN = 1,
    
    //与App通信的服务器，可达且通过WiFi
    AFNetworkReachabilityStatusReachableViaWiFi = 2,
};
```

***

- 定义了网络状态变化后的`通知Key`

```
//状态改变后的通知key
extern NSString * const AFNetworkingReachabilityDidChangeNotification;

//从通知实例中获取到当前状态的key
extern NSString * const AFNetworkingReachabilityNotificationStatusItem;
```

***

- `AFNetworkReachabilityManager `的`向外暴露`的属性

```
/**
 The current network reachability status.
 */
@property (readonly, nonatomic, assign) AFNetworkReachabilityStatus networkReachabilityStatus;

/**
 Whether or not the network is currently reachable.
 */
@property (readonly, nonatomic, assign, getter = isReachable) BOOL reachable;

/**
 Whether or not the network is currently reachable via WWAN.
 */
@property (readonly, nonatomic, assign, getter = isReachableViaWWAN) BOOL reachableViaWWAN;

/**
 Whether or not the network is currently reachable via WiFi.
 */
@property (readonly, nonatomic, assign, getter = isReachableViaWiFi) BOOL reachableViaWiFi;

```

***

- `AFNetworkReachabilityManager `的`只向内暴露`的属性

```
//保存转换成oc对象后的SCNetworkReachabilityRef结构体实例
@property (readwrite, nonatomic, strong) id networkReachability;

@property (readwrite, nonatomic, assign) AFNetworkReachabilityAssociation networkReachabilityAssociation;

@property (readwrite, nonatomic, assign) AFNetworkReachabilityStatus networkReachabilityStatus;

@property (readwrite, nonatomic, copy) AFNetworkReachabilityStatusBlock networkReachabilityStatusBlock;
```

***

- `AFNetworkReachabilityManager`实例化的集中方式

	- 1) 使用默认的监测地址
	
	```
	+ (instancetype)sharedManager {
	    static AFNetworkReachabilityManager *_sharedManager = nil;
	    static dispatch_once_t onceToken;
	    dispatch_once(&onceToken, ^{
	    
	    	//1. 构造默认的地址
	        struct sockaddr_in address;
	        bzero(&address, sizeof(address));
	        address.sin_len = sizeof(address);
	        address.sin_family = AF_INET;
				
			//2. 调用传入ip地址的方法创建AFNetworkReachabilityManager实例
	        _sharedManager = [self managerForAddress:&address];
	    });

	    return _sharedManager;
	}
	```
	
	- 2) 使用指定的服务器域名作为监测地址（注意:`SCNetworkReachabilityCreateWithName`）

	```
	+ (instancetype)managerForDomain:(NSString *)domain {

		//1. 根据传入的域名字符串，创建SCNetworkReachabilityRef实例（网络连接）
	    SCNetworkReachabilityRef reachability = SCNetworkReachabilityCreateWithName(kCFAllocatorDefault, [domain UTF8String]);

		//2. 创建AFNetworkReachabilityManager实例，并保存上面创建的SCNetworkReachabilityRef实例
	    AFNetworkReachabilityManager *manager = [[self alloc] initWithReachability:reachability];
	    
	    //3. 标示来自域名
	    manager.networkReachabilityAssociation = AFNetworkReachabilityForName;

	    return manager;
	}		
	```
	
	- 3) 直接使用ip地址作为监测地址（注意:`SCNetworkReachabilityCreateWithAddress `）
	
	```
	+ (instancetype)managerForAddress:(const void *)address {

		//1. 根据传入的ip地址结构体实例，创建SCNetworkReachabilityRef实例（网络连接）
	    SCNetworkReachabilityRef reachability = SCNetworkReachabilityCreateWithAddress(kCFAllocatorDefault, (const struct sockaddr *)address);

		//2. 创建AFNetworkReachabilityManager实例，并保存上面创建的SCNetworkReachabilityRef实例
	    AFNetworkReachabilityManager *manager = [[self alloc] initWithReachability:reachability];
	    
	    //3. 标示来自ip地址
	    manager.networkReachabilityAssociation = AFNetworkReachabilityForAddress;

	    return manager;
	}
	```
	
***

- 创建`AFNetworkReachabilityManager`时，传入`SCNetworkReachabilityRef`的初始化方法

```
- (instancetype)initWithReachability:(SCNetworkReachabilityRef)reachability {
    self = [super init];
    if (!self) {
        return nil;
    }
	
	//1. 将`SCNetworkReachabilityRef结构体实例`转换成oc中的对象
    self.networkReachability = CFBridgingRelease(reachability);
    
    //2. 初始网络状态设置为未知
    self.networkReachabilityStatus = AFNetworkReachabilityStatusUnknown;

    return self;
}
```

```
CFBridgingRelease()这个c函数主要作用:

1. 将【非Objective-C对象】转换为【Objective-C对象】
2. 同时将对象的管理权交给ARC，开发者无需手动管理内存
3. 效果和 __bridge_transfer 是一样的
```

***


- `AFNetworkReachabilityManager`的init函数与dealloc函数

	- init函数，在`ARC环境下被标记为不可用`

	```
	- (instancetype)init NS_UNAVAILABLE
	{
	    return nil;
	}
	```

	- dealloc函数，在执行时停止网络状态改变的监测

	```
	- (void)dealloc {
	    [self stopMonitoring];
	}
	```
	
	- `NS_UNAVAILABLE`这个宏的使用
		- ARC 状态时候 标记一些 函数不能够使用

***

- 使用`AFNetworkReachabilityManager实例`，获取当前网络状态的Api

```
- (BOOL)isReachable {
    return [self isReachableViaWWAN] || [self isReachableViaWiFi];
}

- (BOOL)isReachableViaWWAN {
    return self.networkReachabilityStatus == AFNetworkReachabilityStatusReachableViaWWAN;
}

- (BOOL)isReachableViaWiFi {
    return self.networkReachabilityStatus == AFNetworkReachabilityStatusReachableViaWiFi;
}
```

***


- 使用`AFNetworkReachabilityManager实例`开启网络状态改变监听（注意，AFN默认是没有开启网络状态监听的，需要自己start开启）

```
- (void)startMonitoring {

	//1. 如果当前有正在监听，先停止当前的监听
    [self stopMonitoring];

	//2. 如果没有要监听的网络连接，就结束此次start操作
    if (!self.networkReachability) {
        return;
    }

	//3. 设置当使用ios c函数库的回调的回调Block
    __weak __typeof(self)weakSelf = self;
    AFNetworkReachabilityStatusBlock callback = ^(AFNetworkReachabilityStatus status) {
        __strong __typeof(weakSelf)strongSelf = weakSelf;

		// 保存当前改变后的网络状态
        strongSelf.networkReachabilityStatus = status;
        
        // 执行使用AFNetworkReachabilityManager传入的用户回调Block
        if (strongSelf.networkReachabilityStatusBlock) {
            strongSelf.networkReachabilityStatusBlock(status);
        }

    };

	//4. 取出AFNetworkReachabilityManager实例保存的SCNetworkReachabilityRef实例
    id networkReachability = self.networkReachability;
    
    //5. 建立 SCNetworkReachabilityContext 实例，需要使用几个Block
    //先将如下这几个Block存入到context，后面再从context中取出来
    //回调Block1: 前面创建的AFNetworkReachabilityStatusBlock类型的callback
    //回调Block2: AFNetworkReachabilityRetainCallback
    //回调Block3: AFNetworkReachabilityReleaseCallback
    SCNetworkReachabilityContext context = {0, (__bridge void *)callback, AFNetworkReachabilityRetainCallback, AFNetworkReachabilityReleaseCallback, NULL};
    
    //6. 设置使用ios c函数库，监听到网络连接状态改变后的回调函数
    //参数1: 网络连接
    //参数2: AFNetworkReachabilityCallback回调函数，这个函数是在使用c函数库，监听到网络状态改变时由iOS sdk触发的回调
    //参数3: 指定为上面创建的Context
    SCNetworkReachabilitySetCallback((__bridge SCNetworkReachabilityRef)networkReachability, AFNetworkReachabilityCallback, &context);
    
    //7. 执行网络监测的操作
    //在`主线程的runloop`上调度
    //`runloop mode`为kCFRunLoopCommonModes，也就是任何情况下都要进行调度
    SCNetworkReachabilityScheduleWithRunLoop((__bridge SCNetworkReachabilityRef)networkReachability, CFRunLoopGetMain(), kCFRunLoopCommonModes);

	//8. 判断当前网络监听是来自 1)domain 2)ip地址
    switch (self.networkReachabilityAssociation) {
    	
    	//如果是domain，就不发通知
        case AFNetworkReachabilityForName:
            break;
            
        //如果是ip地址，就发送通知
        case AFNetworkReachabilityForAddress:
        case AFNetworkReachabilityForAddressPair:
        default: {
            dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0),^{
            	
            	//首先得到网络连接的flags
                SCNetworkReachabilityFlags flags;
                SCNetworkReachabilityGetFlags((__bridge SCNetworkReachabilityRef)networkReachability, &flags);
                
                //根据flags得到当前网络状态
                AFNetworkReachabilityStatus status = AFNetworkReachabilityStatusForFlags(flags);
                
                //然后再回到主队列，发送通知，告知网络状态已经改变
                dispatch_async(dispatch_get_main_queue(), ^{
                    callback(status);

                    NSNotificationCenter *notificationCenter = [NSNotificationCenter defaultCenter];
                    [notificationCenter postNotificationName:AFNetworkingReachabilityDidChangeNotification object:nil userInfo:@{ AFNetworkingReachabilityNotificationStatusItem: @(status) }];

                });
            });
        }
            break;
    }
}
```

```
这个【函数】是在使用ios sdk 的 c函数库监听到网络状态改变时由iOS sdk触发的回调执行的函数

static void AFNetworkReachabilityCallback(SCNetworkReachabilityRef __unused target, SCNetworkReachabilityFlags flags, void *info) {

	//1. 根据flags得到网络状态
    AFNetworkReachabilityStatus status = AFNetworkReachabilityStatusForFlags(flags);
    
    //2. 取出之前设置的callback，并执行
    AFNetworkReachabilityStatusBlock block = (__bridge AFNetworkReachabilityStatusBlock)info;
    if (block) {
        block(status);
    }

	//3. 发送网络状态改变的通知
    dispatch_async(dispatch_get_main_queue(), ^{
        NSNotificationCenter *notificationCenter = [NSNotificationCenter defaultCenter];
        NSDictionary *userInfo = @{ AFNetworkingReachabilityNotificationStatusItem: @(status) };
        [notificationCenter postNotificationName:AFNetworkingReachabilityDidChangeNotification object:nil userInfo:userInfo];
    });

}
```

```
创建SCNetworkReachabilityContext时，指定的retain Block

static const void * AFNetworkReachabilityRetainCallback(const void *info) {
    return Block_copy(info);
}
```

```
创建SCNetworkReachabilityContext时，指定的release Block

static void AFNetworkReachabilityReleaseCallback(const void *info) {
    if (info) {
        Block_release(info);
    }
}

```

***

- 停止网络监听

```
- (void)stopMonitoring {
    if (!self.networkReachability) {
        return;
    }
	
	//从主线程的runloop unschedule 监听操作
    SCNetworkReachabilityUnscheduleFromRunLoop((__bridge SCNetworkReachabilityRef)self.networkReachability, CFRunLoopGetMain(), kCFRunLoopCommonModes);
}
```

***

- 发现AFNetworkReachabilityManager.m有一个关于kvo的函数

	- 源码如下

	```
	+ (NSSet *)keyPathsForValuesAffectingValueForKey:(NSString *)key {
		
		//1. 如果当前被修改的key是: 1) reachable 2) reachableViaWWAN 3) reachableViaWiFi 其中的一种，那么就发出networkReachabilityStatus属性被修改的通知
	    if ([key isEqualToString:@"reachable"] || [key isEqualToString:@"reachableViaWWAN"] || [key isEqualToString:@"reachableViaWiFi"]) {
	        return [NSSet setWithObject:@"networkReachabilityStatus"];
	    }
		
		//2. 返回当前被修改的属性
	    return [super keyPathsForValuesAffectingValueForKey:key];
	}
	```
	
	- 这个方法的作用
	
	```
	1. 如果一个@property的变化，依赖于某个对象的一个或者多个@property的变化
	2. 一旦这些一个或者多个@property的值发生变化，那么之前的主动依赖的@property也表示发生变化，并发送KVO修改通知
	```
	
	- 上如源码的作用

	```
	1. networkReachabilityStatus属性变化，依赖于 1) reachable 2) reachableViaWWAN 3) reachableViaWiFi 这三个属性值的变化

	2. 一旦后三个其中一个属性值变化，就代表networkReachabilityStatus属性值发生变化
	```
	
	- 使用这个方法注意:
	
	```
	属性必须实现 setter方法与gettter方法
	```
	
****

- 将`SCNetworkReachabilityFlags`解析成网络状态枚举

```
static AFNetworkReachabilityStatus AFNetworkReachabilityStatusForFlags(SCNetworkReachabilityFlags flags) {

	//1. 
    BOOL isReachable = ((flags & kSCNetworkReachabilityFlagsReachable) != 0);
    BOOL needsConnection = ((flags & kSCNetworkReachabilityFlagsConnectionRequired) != 0);
    BOOL canConnectionAutomatically = (((flags & kSCNetworkReachabilityFlagsConnectionOnDemand ) != 0) || ((flags & kSCNetworkReachabilityFlagsConnectionOnTraffic) != 0));
    BOOL canConnectWithoutUserInteraction = (canConnectionAutomatically && (flags & kSCNetworkReachabilityFlagsInterventionRequired) == 0);
    BOOL isNetworkReachable = (isReachable && (!needsConnection || canConnectWithoutUserInteraction));

	//2. 
    AFNetworkReachabilityStatus status = AFNetworkReachabilityStatusUnknown;
    if (isNetworkReachable == NO) {
        status = AFNetworkReachabilityStatusNotReachable;
    }
#if	TARGET_OS_IPHONE
    else if ((flags & kSCNetworkReachabilityFlagsIsWWAN) != 0) {
        status = AFNetworkReachabilityStatusReachableViaWWAN;
    }
#endif
    else {
        status = AFNetworkReachabilityStatusReachableViaWiFi;
    }

    return status;
}
```

***

- 主要使用的是iOS sdk提供的`SCNetworkReachability.h`
	
	- SCNetworkReachabilityContext结构体

	```
	typedef struct {
		CFIndex		version;
		void *		info;
		const void	*(*retain)(const void *info);
		void		(*release)(const void *info);
		CFStringRef	(*copyDescription)(const void *info);
	} SCNetworkReachabilityContext;
	``` 
	
	- SCNetworkReachabilityFlags枚举值

	```
	enum {
		kSCNetworkReachabilityFlagsTransientConnection	= 1<<0,
		kSCNetworkReachabilityFlagsReachable		= 1<<1,
		kSCNetworkReachabilityFlagsConnectionRequired	= 1<<2,
		kSCNetworkReachabilityFlagsConnectionOnTraffic	= 1<<3,
		kSCNetworkReachabilityFlagsInterventionRequired	= 1<<4,
		kSCNetworkReachabilityFlagsConnectionOnDemand	= 1<<5,	// __OSX_AVAILABLE_STARTING(__MAC_10_6,__IPHONE_3_0)
		kSCNetworkReachabilityFlagsIsLocalAddress	= 1<<16,
		kSCNetworkReachabilityFlagsIsDirect		= 1<<17,
	#if	TARGET_OS_IPHONE
		kSCNetworkReachabilityFlagsIsWWAN		= 1<<18,
	#endif	// TARGET_OS_IPHONE

		kSCNetworkReachabilityFlagsConnectionAutomatic	= kSCNetworkReachabilityFlagsConnectionOnTraffic
	};
	typedef	uint32_t	SCNetworkReachabilityFlags;
	```
	
	- 获取SCNetworkReachabilityGetFlags的Api

	```
	Boolean//返回bool，表示获取是否成功
	SCNetworkReachabilityGetFlags			(
						SCNetworkReachabilityRef	target,
						SCNetworkReachabilityFlags	*flags
						)				__OSX_AVAILABLE_STARTING(__MAC_10_3,__IPHONE_2_0);
	```
	
	- SCNetworkReachabilityCallBack，block声明

	```
	typedef void (*SCNetworkReachabilityCallBack)	(
						SCNetworkReachabilityRef	target,
						SCNetworkReachabilityFlags	flags,
						void				*info
						);
	```
	
	- 创建`SCNetworkReachabilityRef`的Api

	```
	方式1:
	
	SCNetworkReachabilityRef//返回值类型
	SCNetworkReachabilityCreateWithName		(
						CFAllocatorRef			allocator,
						const char			*nodename
						)				__OSX_AVAILABLE_STARTING(__MAC_10_3,__IPHONE_2_0);
	```

	```
	方式2:
	
	SCNetworkReachabilityRef//返回值类型
	SCNetworkReachabilityCreateWithAddress		(
						CFAllocatorRef			allocator,
						const struct sockaddr		*address
						)				__OSX_AVAILABLE_STARTING(__MAC_10_3,__IPHONE_2_0);
	```
	
	```
	方式3:
	
	SCNetworkReachabilityRef//返回值类型
	SCNetworkReachabilityCreateWithAddressPair	(
						CFAllocatorRef			allocator,
						const struct sockaddr		*localAddress,
						const struct sockaddr		*remoteAddress
						)				__OSX_AVAILABLE_STARTING(__MAC_10_3,__IPHONE_2_0);
	```
	
	- 设置使用`SCNetworkReachability`监听到状态改变后的回调函数

	```
	Boolean//返回一个bool，返回是否设置成功
	SCNetworkReachabilitySetCallback		(
						SCNetworkReachabilityRef	target,
						SCNetworkReachabilityCallBack	callout,
						SCNetworkReachabilityContext	*context
						)				__OSX_AVAILABLE_STARTING(__MAC_10_3,__IPHONE_2_0);
	```
	
	- 在一个runloop开始调度执行网络状态监听

	```
	Boolean//返回一个bool，返回是否开始调度成功
SCNetworkReachabilityScheduleWithRunLoop	(
						SCNetworkReachabilityRef	target,
						CFRunLoopRef			runLoop,
						CFStringRef			runLoopMode
						)				__OSX_AVAILABLE_STARTING(__MAC_10_3,__IPHONE_2_0);
	```
	
	- 在一个runloop移除结束执行网络状态监听

	```
	Boolean//返回一个bool，返回是否移除调度成功
SCNetworkReachabilityUnscheduleFromRunLoop	(
						SCNetworkReachabilityRef	target,
						CFRunLoopRef			runLoop,
						CFStringRef			runLoopMode
						)				__OSX_AVAILABLE_STARTING(__MAC_10_3,__IPHONE_2_0);
	```
	
	- 暂时不知道做什么的...

	```
	Boolean
SCNetworkReachabilitySetDispatchQueue		(
						SCNetworkReachabilityRef	target,
						dispatch_queue_t		queue
						)				__OSX_AVAILABLE_STARTING(__MAC_10_6,__IPHONE_4_0);
	```