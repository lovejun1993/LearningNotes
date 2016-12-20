---
layout: post
title: "重构代码二、UIView的数据适配"
date: 2016-01-26 11:11:48 +0800
comments: true
categories: 
---


###常见在UIView中直接耦合实体类型对象

```objc
@interface ShopModel : NSObject
@property (nonatomic, copy) NSString *shopName;
@property (nonatomic, copy) NSString *shopAdress;
//....等等属性
@end
@implementation ShopModel
@end
```

```objc
@interface ShopEntryView : UIView

- (void)setupWithShopModel:(ShopModel *)shopModel;

@end
@implementation ShopEntryView {
    UILabel *_shopNameLabel;
    UILabel *_shopAdressLabel;
}

- (void)setupWithShopModel:(ShopModel *)shopModel {
    _shopNameLabel.text = shopModel.shopName;
    _shopAdressLabel.text = shopModel.shopAdress;
    [self setNeedsLayout];
}

- (void)layoutSubviews {
    [super layoutSubviews];
    
    //frame计算
    //......
}

@end
```

```objc
@implementation UIAdaptorDemo {
    ShopEntryView *_shopView;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    [self apiCall];
}

- (void)apiCall {
    ShopModel *model = nil;//假设从服务器接口获取得到
    
    [_shopView setupWithShopModel:model];
}

@end
```

这样比很多人还在使用NSDictionary传给UI再进行解析赋值给UI对象，已经进步了很多了，但是还是有不好的地方。

因为UI直接强耦合了`ShopModel`这个类型，就是说这个`ShopEntryView`的对象就只能够为`ShopModel`这个类型对象进行显示了，就再不能显示其他实体类对象的数据了。

***

###改进一、使用抽象接口隔离

数据类的抽象

```objc
@protocol ShopModelInterface <NSObject>
@property (nonatomic, copy) NSString *shopName;
@property (nonatomic, copy) NSString *shopAdress;
@end

或

@protocol ShopModelInterface <NSObject>
- (NSString *)shopName;
- (NSString *)shopAdress;
@end
```

建议还是第二种吧，一般还是不要在Protocol中使用属性。Protocol一般只应该定义一些抽象方法，而属性会带有实例变量。

下面是抽象的具体实现

```objc
@interface ShopModel : NSObject <ShopModelInterface>
@property (nonatomic, copy) NSString *shopName;
@property (nonatomic, copy) NSString *shopAdress;
//....等等属性
@end
@implementation ShopModel
@end
```

UI改成依赖抽象

```objc
@interface ShopEntryView : UIView

- (void)setupWithShopModel:(id<ShopModelInterface>)shopModel;

@end
@implementation ShopEntryView {
    UILabel *_shopNameLabel;
    UILabel *_shopAdressLabel;
}

- (void)setupWithShopModel:(id<ShopModelInterface>)shopModel {
    _shopNameLabel.text = [shopModel shopName];
    _shopAdressLabel.text = [shopModel shopAdress];
    [self setNeedsLayout];
}

- (void)layoutSubviews {
    [super layoutSubviews];
    
    //frame计算
    //......
}

@end
```

UIVieController使用抽象类型转换对象，并传给UI

```objc
@implementation UIAdaptorDemo {
    ShopEntryView *_shopView;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    [self apiCall];
}

- (void)apiCall {
    ShopModel *model = nil;//假设从服务器接口获取得到
    
    [_shopView setupWithShopModel:(id<ShopModelInterface>)model];
}

@end
```

这样比之前直接耦合`强数据类型 ShopModel`又好多了，但是仍然还是有一个问题没有解决。

仍然只能显示 `id<ShopModelInterface>` 这一类类型对象的数据。而如果没有实现`ShopModelInterface`抽象定义的实体类对象，就不能让`ShopEntryView`去显示数据。

****

###UI自己定义出自己能够显示数据项，让外界自己按照U给出的规则进行数据适配

首先UI自己定义出所有能够显示的数据规则。

```objc
@interface ShopEntryViewDataSource : NSObject

@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *address;

@end
@implementation ShopEntryViewDataSource
@end
```

然后UI依赖上面的数据格式进行数据显示

```objc
@interface ShopEntryView : UIView

- (void)setupWithDataSource:(ShopEntryViewDataSource *)dataSource;

@end
@implementation ShopEntryView {
    UILabel *_shopNameLabel;
    UILabel *_shopAdressLabel;
}

- (void)setupWithDataSource:(ShopEntryViewDataSource *)dataSource {
    _shopNameLabel.text = [dataSource name];
    _shopAdressLabel.text = [dataSource address];
    [self setNeedsLayout];
}

- (void)layoutSubviews {
    [super layoutSubviews];
    
    //frame计算
    //......
}

@end
```

那么现在UI就不关心任何的外部数据类型了，只关心自己依赖的`ShopEntryViewDataSource`类型的对象进行数据显示。

此时，如果有具体数据实现类型1，实现了抽象接口

```objc
@protocol ShopEntryViewDataSourceInterface <NSObject>
- (NSString *)shopName;
- (NSString *)shopAdress;
@end
```

```objc
@interface ShopModel : NSObject <ShopEntryViewDataSourceInterface>
@property (nonatomic, copy) NSString *shopName;
@property (nonatomic, copy) NSString *shopAdress;
//....等等属性
@end
@implementation ShopModel
@end
```

如果有一些数据类型是没有实现抽象接口的

```objc
@interface CompanyModel : NSObject
@property (nonatomic, strong) NSString *companyName;
@property (nonatomic, strong) NSString *companyAddress;
@end
@implementation CompanyModel
@end
```

对于如上两种类型数据，让UI都去适配显示

```objc
@implementation UIAdaptorDemo {
    ShopEntryView *_shopView1;
    ShopEntryView *_shopView2;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    [self apiCall];
}

- (void)apiCall {
    
    // 数据类型一适配UI
    ShopModel *model1 = nil;
    ShopEntryViewDataSource *dataSource1 = [ShopEntryViewDataSource new];
    dataSource1.name = model1.shopName;
    dataSource1.address = model1.shopAdress;
    [_shopView1 setupWithDataSource:dataSource1];
    
    // 数据类型二适配UI
    CompanyModel *model2 = nil;
    ShopEntryViewDataSource *dataSource2 = [ShopEntryViewDataSource new];
    dataSource2.name = model2.companyName;
    dataSource2.address = model2.companyAddress;
    [_shopView2 setupWithDataSource:dataSource2];
}

@end
```

这样此时可以显示任意类型的实体类对象的数据了，而不再是依赖某一类的对象。


***

###还有一种VC之间跳转时设置参数，我看到有一些时使用一个字典传递过去，然后目标VC按照固定的key值来读取参数字典中参数值。

有一种我觉得比传字典使用固定key去取值好的方式，仍然是使用抽象隔离。

- (1) 首先是跳转到的目标控制器

```objc
#import <UIKit/UIKit.h>

/**
 *  当前VC需要的参数集合抽象
 */
@protocol Pass2VCArguments <NSObject>

@required

- (NSString *)username;
- (NSString *)password;

@optional

- (NSString *)validCode;

@end

/**
 *  跳转到的目标控制器
 */
@interface Pass2VC : UIViewController

@property (nonatomic, strong) id<Pass2VCArguments> arguments;

@end

@implementation Pass2VC

- (void)viewDidLoad {
    [super viewDidLoad];
    
    //取出参数
    NSString *username = [_arguments username];
    NSString *password = [_arguments password];
    NSString *validCode = nil;
    if ([_arguments respondsToSelector:@selector(validCode)]) {
        validCode = [_arguments validCode];
    }
    
    //执行业务处理
    //....
}

@end
```

- (2) 执行跳转逻辑的VC

```objc
#import <UIKit/UIKit.h>

@interface Pass1VC : UIViewController

@end

#import "Pass1VC.h"

//导入跳转到VC
#import "Pass2VC.h"

@interface Pass2VCArgumentsImpl : NSObject <Pass2VCArguments> {
    @package
    NSString *_username;
    NSString *_password;
    NSString *_validCode;
}
@end
@implementation Pass2VCArgumentsImpl

- (NSString *)username {
    return _username;
}
- (NSString *)password {
    return _password;
}
- (NSString *)validCode {
    return _validCode;
}
@end

@implementation Pass1VC

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    //1. 构建参数
    Pass2VCArgumentsImpl *arguments = [Pass2VCArgumentsImpl new];
    arguments->_username = @"username";
    arguments->_password = @"password";
    arguments->_validCode = @"验证码";
    
    //2. 跳转VC
    Pass2VC *vc = [[Pass2VC alloc] init];
    vc.arguments = arguments;
    
    [self.navigationController pushViewController:vc animated:YES];
}

@end
```

参数传递的是一个符合VC2的理想的参数集合的具体实现类对象，这样一来双方就达成一致的约定了。