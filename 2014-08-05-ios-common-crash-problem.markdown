---
layout: post
title: "iOS_common_crash_problem"
date: 2014-08-05 11:47:42 +0800
comments: true
categories: 
---

###常见问题整理

***

###Array的一些崩溃

- (1) 数组对象是id类型，并不清楚到底是可变还是不可变

```objc
id array = [NSArray new];
[array addObject:@"111"];
```

- (2) 添加一个nil到数组对象

```objc
id array = [NSMutableArray new];
[array addObject:nil];
```

```objc
NSString *str = nil;
@[@"1", str, @"2"];
```

如果一定要加一个空，起到占位作用

```objc
id array = [NSMutableArray new];
[array addObject:(id)kCFNull];
[array addObject:[NSNull null]];
```

运行输出

```
<__NSArrayM 0x7fe5c8726450>(
<null>,
<null>
)
```

可以了解下 nil、Nil、kCFNull、NSNUll之间的区别。

- (3) 数组越界

```objc
id array = [NSMutableArray new];
[array objectAtIndex:100000];
```

```objc
id array = [NSMutableArray new];
[array insertObject:[NSObject new] atIndex:10000];
```

```objc
id array = [NSMutableArray new];
[array removeObjectAtIndex:10000];
```

总之一切有关index下标取值的方法，都要注意下标是否越界。还有有关`Rnage`操作数组对象的也要注意。


***

###Dic 的一些崩溃

- 使用@{}快速构建字典，value如果为nil，就会崩溃

```objc
id dic = @{
           	@"key" : [[NSUserDefaults standardUserDefaults] objectForKey:@"wocao"],
           };
```

修改就是对value做一次防止nil处理

```objc
id dic = @{
           	@"key" : ([[NSUserDefaults standardUserDefaults] objectForKey:@"wocao"] ? [[NSUserDefaults standardUserDefaults] objectForKey:@"wocao"] : (id)kCFNull),
           };
```

对于key也是不能为nil的，同样会崩溃

```objc
id<NSCopying> key = nil;
id dic = @{
            key :  (id)kCFNull,
           };
```

- 取出对象后必须判断对象类型再去发送消息给这个取出对象

```objc
NSDictionary *dic = @{@"obj" : [NSObject new]};
UIViewController *vc = [dic objectForKey:@"obj"];
// 当做UIViewController去操作
```

```objc
NSDictionary *dic = @{@"num" : @"Hello World"};
NSInteger count = [[dic objectForKey:@"num"] count];
```

所以一定要对从字典取出的对象进行类型判断之后再去发送消息

```objc
NSDictionary *dic = @{@"num" : @"Hello World"};
//    NSInteger count = [[dic objectForKey:@"num"] count];
NSArray *array = [dic objectForKey:@"num"];
if ([array isKindOfClass:[NSArray class]]) {
    NSInteger count = array.count;
}
```

注意对象类型比较问题，对于常见集合类型（Array、Set、Dic）都是属于`类簇类`，所以不能使用如下方法进行类型判断:

```objc
NSArray *array = @[@"1", @"2", @"3"];
if ([array class] == [NSArray class]) {
    [array count];
}
```

但是对于一些我们自己的一些类对象（非类簇类的对象）是可以的:

```objc
Car *car = [[Car alloc] initWithName:@"car1" cid:@"cid1"];
if ([car class] == [car class]) {
    NSLog(@"car.name = %@", car.name);
}
```

至于什么是类簇类可以参考网上其他文章，此处就细说。

***

###NSString的一些崩溃

```objc
[NSString stringWithString:nil];
```

```objc
NSString *str1 = nil;
NSString *str2 = @"Hello world!";

// 崩溃
if ([str2 hasPrefix:str1]) {
    //.....
}

// 崩溃
if ([str2 hasSuffix:str1]) {
 	//.....       
}

// 崩溃
[str2 containsString:str1];
```

总之对于NSString的这些方法，都应该防止传入一个nil的NSString对象。

***

###application windows are expected to have a root view controller 启动的时候可能由于一些提示框Window没有设置rootViewController导致程序崩溃

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:    (NSDictionary *)launchOptions {
   //....
   
   // 防止启动时Window缺少root vc导致程序崩溃
	NSArray *windows = [[UIApplication sharedApplication] windows];
	for(UIWindow *window in windows) {
	    if(window.rootViewController == nil){
	        UIViewController* vc = [[UIViewController alloc]initWithNibName:nil bundle:nil];
	        window.rootViewController = vc;
	    }
	}
	
	return YES;
}
```