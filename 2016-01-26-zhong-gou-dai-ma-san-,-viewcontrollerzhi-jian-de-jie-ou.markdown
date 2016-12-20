---
layout: post
title: "重构代码三、ViewController之间的解耦"
date: 2016-01-26 12:18:46 +0800
comments: true
categories: 
---


###一般的App中，ViewControllerA要push到ViewControllerB，为如下几步:

```
1. ViewControllerA.m 导入 ViewControllerB.h
2. new 一个 ViewControllerB 的实例
3. [ViewControllerA实例 pushViewController: ViewControllerB实例];
```

毋庸置疑如上的方法 简单明了快速，但是有个最大的缺点

> ViewControllerA.m 强依赖 ViewControllerB.h

如果哪天希望单独将 `ViewControllerA` 复用到一个新的App项目中时，基本上是不可能。因为ViewControllerA 强依赖了ViewControllerB.h，或许还有ViewControllerC.h、ViewControllerD.h、ViewControllerE.h...

那么这样的 ViewControllerA.h与ViewControllerA.m 几乎是 `无法复用`.

###因重构现有的App项目，希望把其中一些ViewController、业务类单独抽出封装成SDK。可是一看项目的代码，全部是A依赖B，B依赖C，C依赖A的形式，几乎是无法单独抽出某一个部分，深深感觉到这种 `强依赖` 的代码结构，在需要 `将某一部分代码单独剥离出来` 的时候真的很恶心，造成这样的原因:

- 过度 依赖 `具体类`
- 具体类 之间的 互相引用

如下这种，一个A类里面，直接导入其他 `具体类.h`

```
#import "具体类1.h"
#import "具体类2.h"
#import "具体类3.h"

@implementation 类A

//使用具体类1、具体类2、具体类3的实现代码

@emd
```

如上结构的代码造成无法单独剥离 类A实现代码 

- 如上结构代码在需要单独将`类A的 .h与.m`拖到`另一个工程`中复用时
- 就必须同时拖入 `类A .h与.m 所依赖的其他具体类.h与.m`
- 那拖入的这些.h与.m又可能会依赖其他的.h与.m


###对一个功能模块实现类最理想的复用状态:

> 从当前App项目中拷贝出 ViewController.h与ViewController.m 或 ManagerA.h与ManagerB.m 到另一个App项目中直接编译成功.


###而能够达到完全的复用程度必须遵循

- 依赖抽象、不依赖具体。每一层的类都最好都是依赖抽象。

- 尽可能少的与其他模块实现类产生关系。

***

###解耦方式一、一个UIViewController具体类 <--映射一个--> 一个整形ViewControllerId枚举值

###之前自己的App中，使用了一个BaseViewController添加了一个自己的push方法，然后让`具体ViewController类` 对应一个 `整形唯一Id值`

UIViewController具体类 对应的 整形ViewControllerId枚举定义

```objc
typedef NS_ENUM(NSInteger, ViewControllerId) {
    ViewControllerIdLogin = 0x01,
    ViewControllerIdRegist,
    ViewControllerIdModify,
    //...
};
```

基类BaseViewController提供统一pushViewController的方法，通过id值找到对应的ViewController类

```
+ (instancetype)pushWithControllerId:(ViewControllerId)controlelrId
{
    //1. 根据controlelrId得到对应Class
    //2. 实例化Class
    //3. 设置得到的实例
    //4. push viewController
    //5. 返回实例
}
```

我自己觉的还是比 直接强依赖 好很多了。
至少ViewControllerA.m中 耦合的仅仅只是一个 `整形 ViewControllerId 枚举值`，没有依赖任何的 `ViewControllerB.h、ViewControllerC.h、ViewControllerD.h`。


但是觉得这种方法也有几个问题:

- ViewControllerId 枚举值 越来越多
- 传入的参数没有解耦、打包成一体
	- 参考之前的《参数封装》文章

****

###解耦方式二、使用抽象接口protocol隔离

> 举例一个简单的例子 ViewControllerA需要跳转到ViewControllerB显示最新新闻列表

####角色1、`显示新闻列表功能` 的ViewController 抽象接口协议定义

> 主要针对 `功能点` 的抽象

```objc
#import <Foundation/Foundation.h>

/**
 *  抽象获取最新新闻的ViewController接口
 */
@protocol NewsListlization <NSObject>

- (void)receiveNewsList;

@end
```

####角色2、新闻列表抽象接口协议的 `实现类` ViewControllerB

```objc
#import <UIKit/UIKit.h>

#import "NewsListlization.h"

@interface ViewControllerB : UIViewController <NewsListlization>

@end
```

```objc
#import "ViewControllerB.h"

@implementation ViewControllerB

- (void)receiveNewsList {
    //1. 获取最新数据
    //2. reload UI
}

@end
```

###角色3、ViewControllerA跳转到 `一个能够显示新闻列表的Controller（获得一个抽象接口实现类对象完成对应的功能）`

```objc
#import "ViewControllerA.h"

//依赖的是一个中间容器对象
#import "ViewControllerManager.h"

@implementation ViewControllerA

- (void)buttonClick{
    
    //1. 获取到单例容器对象
    ViewControllerManager *manager = [ViewControllerManager sharedInstance];
    
    //2. 从容器对象获取 一个可以加载最新新闻列表的ViewController抽象的实现类对象
    id<NewsListlization> controller = [manager newsListViewController];
    
    //3. push 得到的这个 ViewController实例
    [self.navigationController pushViewController:(UIViewController *)controller
                                         animated:YES];
}

@end
```

从上看到，ViewControllerA中没有强依赖ViewControllerB.h，只是依赖一个中间容器类。后续复用这个ViewControllerA时，只需要维护这个中间容器类，而不需要关系其他的`具体类`.


####角色4、中间容器类，负责管理 抽象与具体实现的映射

```objc
#import <Foundation/Foundation.h>

#import "NewsListlization.h"

@interface ViewControllerManager : NSObject

+ (instancetype)sharedInstance;

- (id<NewsListlization>)newsListViewController;

@end
```

```objc
#import "ViewControllerManager.h"

//导入抽象的实现类
#import "ViewControllerB.h"

@implementation ViewControllerManager

+ (id<NewsListlization>)newsListViewController {
    
    //返回抽象的实现类的一个实例
    return [[ViewControllerB alloc] init];
}

@end
```

- 依赖 `具体类` 的代码都交给 这个`中间容器类`.
- 并且由中间容器类对象 管理 所有的抽象与具体类的 映射关系.

####对于中间容器类，可以使用一个开源类库 `Objection` 来代替.
