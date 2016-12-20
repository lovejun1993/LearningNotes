---
layout: post
title: "DesignMode、Proxy"
date: 2014-01-10 11:14:16 +0800
comments: true
categories: 
---


###代理模式、可谓是在iOS用的最多、最经典的模式

具体应用场景

- NSProxy应用拦截目标对象方法
- UITableView中UI效果展示与数据源获取的逻辑分离

###小结代理的应用场景

- 需要拦截目标方法执行、面向切面编程
	- 代理对象
	- 继承类
	- swizzle方法实现
	- 消息转发（动态代理对象使用到）

- UI 与 数据源 分离
	- UITableView只负责显示界面效果
	- 显示的数据通过回调delegate对象来获取

###代理的种类

- 静态 代理对象
- 动态 代理对象

***

###静态代理、需要手动传入被代理的`Target对象`

####抽象接口

```objc
//
//  Human.h
//  demos
//

#import <Foundation/Foundation.h>

/**
 *  抽象接口
 */
@protocol Human <NSObject>

- (void)run;

@end
```

####接口的具体实现

```objc
//
//  XiaoMing.h
//  demos
//

#import <Foundation/Foundation.h>

//实现抽象接口
#import "Human.h"

@interface XiaoMing : NSObject <Human>

@end
```

```objc
//
//  XiaoMing.m
//  demos
//

#import "XiaoMing.h"

@implementation XiaoMing

- (void)run {
    NSLog(@"小明在跑步...");
}

@end
```

###抽象接口类型的 代理实现类

```objc
//
//  HumanProxy.h
//  demos
//

#import <Foundation/Foundation.h>

//实现抽象接口
#import "Human.h"

/**
 *  代理类也是抽象接口实现者
 */
@interface HumanProxy : NSObject <Human>

/**
 *  传入要被代理的对象
 */
- (instancetype)initWithTarget:(id<Human>)target;

@end
```

```objc
//
//  HumanProxy.m
//  demos
//

#import "HumanProxy.h"

@interface HumanProxy ()

@property (nonatomic, strong) id<Human> target;

@end

@implementation HumanProxy

- (instancetype)initWithTarget:(id<Human>)target
{
    self = [super init];
    if (self) {
        
        //保存原始对象
        _target = target;
    }
    return self;
}

- (void)run {
    
    //1. 之前做点什么
    //.....
    
    //2. 目标方法执行
    [_target run];
    
    //3. 之后做点什么
    //.....
}

@end
```

###Java中有动态代理，iOS也有动态代理，只不过实现有一点不同而已.

***

###首先看下Java中的动态代理

抽象接口

```java
public interface Subject   
{   
	public void doSomething();   
} 
```

目标实现类

```java
public class RealSubject implements Subject   
{   
  	public void doSomething()   
	{   
		System.out.println( "call doSomething()" );   
	}   
} 
```

代理类（注意: 实现的是 `InvocationHandler` 接口）

```java
public class ProxyHandler implements InvocationHandler   
{   

	//保存原始对象
	private Object proxied;   
   
   //构造函数传入原始对象  
	public ProxyHandler( Object proxied )   
	{   
   		this.proxied = proxied;   
  	}   
   
///////////////////////////////////////////////////////
//////实现InvocationHandler的接口方法，完成方法拦截
///////////////////////////////////////////////////////
	public Object invoke( Object proxy, Method method, Object[] args ) throws Throwable   
	{   
	//1. 在目标对象方法执行之前
	//....
	
	//2. 目标对象的方法执行
	return method.invoke( proxied, args);  
	    
	//3. 在目标对象方法执行之后
	//....
	}    
} 
```

代理类的使用demo示例

```java
import java.lang.reflect.InvocationHandler;   
import java.lang.reflect.Method;   
import java.lang.reflect.Proxy;   
import sun.misc.ProxyGenerator;   
import java.io.*; 
  
public class DynamicProxy   
{   
  public static void main( String args[] )   
  {   
  
  	//1. 目标对象
    RealSubject real = new RealSubject();   
    
    //2. 使用系统Proxy类，创建一个RealSubject对象的代理对象
    //2.1 指定Subject.class.getClassLoader()
    //2.2 指定Subject.class
    //2.3 指定InvocationHandler实现类对象
    Subject proxySubject = (Subject)Proxy.newProxyInstance(Subject.class.getClassLoader(), new Class[]{Subject.class}, new ProxyHandler(real));
         
    //3. 调用代理对象的目标方法     
    proxySubject.doSomething(); 
  }   
}  
```

通过InvocationHandler接口，来拦截Target对象的所有方法的执行.

***

###Objective-C中的动态代理，使用的是 `NSProxy` 类实现

套路差不多，使用`消息转发`完成，具体使用参考 NSProxy文章.

- forward方法
- methodSign方法