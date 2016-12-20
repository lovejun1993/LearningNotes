---
layout: post
title: "XMPP学习"
date: 2015-11-13 01:05:15 +0800
comments: true
categories: 
---

###一、XMPP协议，是一套规范了一套用于即使聊天中，客户端与服务端进行传输数据的`数据传输格式定义`。

###二、XMPP协议实现之客户端，如: Adium，Spark，等。我们还可以使用XMPP组织提供的XMPPFramework for iOS来编写基于XMPP协议的客户端App程序。

###三、XMPP协议实现之服务端，如Openfire等。那么同理，服务端也可以使用XMPP组织提供用来扩展服务端程序的Library实现基于XMPP协议的服务端程序。

***

###XML作为XMPP协议规定的`数据传输格式`

- 比如，之前在Socket文章在，发送登陆、发送消息数据的格式

发送登陆的数据格式

```
iam:账号
```

发送聊天数据的数据格式

```
msg:你好啊!
```

- 那么在XMPP协议中，规定数据格式为XML，可以修改为如下格式

发送登陆的数据格式

```
<im>
	<iam>zhangsan</iam>
</im>
```

发送聊天数据的数据格式

```
<im>
	<msg>你好啊!</msg>
</im>
```

- 小结`使用XMPP`实现通讯与使用`Socket`实现通讯的区别

	- XMPP中使用`XML格式`的数据
	- Socket中使用`字符串格式`的数据

****

###先搭建一个实现了XMPP通讯协议的服务器后台使用

- 安装jdk

	- 1) 安装dmg

	- 2) 配置jdk mac系统环境变量

	- 3) 终端输入 `vi ~/.bash_profile`

	- 4) 输入 `export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_40.jdk/Contents/Home`，其中找自己机器上的安装路径

	- 5) 输入 `source ~/.bash_profile` 使设置的环境变量生效

	- 6) 输入 `echo $JAVA_HOME` 查看是否设置正确

	- 7) 输入 `java -version` 测试jdk是否配置成功

	- 8) 这种配置方法只适用于用户环境变量，如果系统更新，之前的配置可能失效，如果想要永久改变需要配置在`/etc/profile`文件中

- 安装mysql数据库，以及数据库开发工具navicat

	- 安装mysql的dmg

	- 打开终端，输入 `vim ~/.bash_profile`

	- 在打开的文件中输入如下shell代码（注意: =号两侧不能有空格）

	```
	#mysql环境变量配置
	
	#进入mysql环境命令行
	alias mysql='/usr/local/mysql/bin/mysql'
	
	#登陆mysql管理平台
	alias mysqladmin='/usr/local/mysql/bin/mysqladmin'
	#ls
	alias ls='ls -G'
	```

	- 输入 `source ~/.bash_profile` 使设置的环境变量生效

	- 修改root账户的登陆密码

	```
	mysqladmin -u root password "123456"
	```
	
	- 使用root账户登陆mysql服务器

	```
	mysql -u root -p
	
	输入密码123456
	```
	
	- 查看当前所有的数据库

	```
	show databases;
	```

- 打开`http://xmpp.org/`下载服务端`Openfire`，并安装

	- 点击安装dmg

	- 修改 `/usr/local/openfire/`目录的权限

	```
	1. cd /usr/local/
	2. open .
	3. 选择 openfire，右键，显示简介
	4. 解锁
	5. 在共享与权限中，添加一个Adnimistrator用户
	```
	
	- 进入到`/usr/local/openfire/resources/database/`目录下，其中的存放的是对应每一种数据库的初始化脚本
		- mysql的`openfire_mysql.sql`
		- oracle的`openfire_oracle.sql`
		- db2的`openfire_db2.sql`
		- sqlserver的`openfire_sqlserver.sql`
		- 把对应数据库的脚本sql文件，复制到桌面，以便导入到数据库

	- 安装`mysql workbench`图形化操作工具
		- 创建一个专门为`openfire`服务器使用的数据库`openfire`，注意`Collation`选择`utf8_bin`

		- 使用`mysql workbench`导入执行sql脚本
			- File -> Open SQL Script -> 选择sql脚本文件
			- 点击执行脚本，创建所有XMPP需要的表结构

- 接下来，设置里面打开Openfire，进行性openfire服务器配置
	- 点击 `Open Admin Console` ，进入Openfire服务器配置界面
	- 语言选择，简体中文
	- 设置一个域名，比如`webchat.im`
	- 选择标准数据库
	- 配置Openfire服务器连接到本地数据库的JDBC驱动，以及配置
		- 替换数据库URL选择中的，`host`与`database name`
				- host: "localhost"
				- database name: openfire
		- mysql数据库登陆账户: root
		- 登陆密码: 123456

	- openfire服务器登陆地址，http://localhost:9090/
		- 登陆名: admin
		- 密码: 123456
		- 添加几个用户

***

###然后再安装客户端，用来我们写的App程序进行通信

- 安装XMPP协议实现的mac客户端
	- Adium
	- Spark
	- mac自带的`信息`程序
		- 账号类型: Jabber（XMPP）
		- 账号:**zhangsan@webchat.im**，一定要加上`@webchat.im`
		- 密码: 123456
		- 服务器: `localhost或127.0.0.1`
		- 端口号: 登陆openfire后台界面，查看`服务器端口`栏
			- 选择5222，没有加密的，方便写代码

***

###下载XMPP提供的iOS客户端的类库Library代码

- 选择	`releases` 版本

- `XMPPFramework`代码结构
	- Authentication: 登陆、授权等
	- Categories: 一些扩展的分类
	- Core: 核心的XMPP操作类
	- `Extensions`: 让开发者自己去开启某些功能木块，默认情况下XMPP框架`不会使用`
	- Utilities: 常用工具类
	- Vendor: 依赖的第三方框架
	- Xcode: 提供一些demo实例代码


****

###依次列举XMPP类库中的主要Api类

- `XMPPStream`
	- XMPP代码开发中使用的做多的Api类，

- `XMPPParser`
	- 负责解析XMPPStream数据流

- `XMPPJID`
	- 表示XMPP账户登录的用户名，实现了NSCoping协议和NSCoding协议

- `XMPPElement`

	- 继承自`NSMXLElement`
	- 表示XML中的一个节点元素

	- `XMPPIQ`: 组装一个XMPP请求的XML元素
	- `XMPPMessage`: 
	- `XMPPPresence`: 代表一个`上线`操作的XML配置

- `XMPPModule`

- `XMPPLogging`

- `XMPPInternal`

****

###导入`Vender`文件夹下的库时出现的问题

- 问题1: `libxml/tree.h file not found`，原因是因为类库`KissXML`类库引起

	- 添加` libxml2.dylib`系统动态库
	- 工程 -> **Project** -> BuildSetting -> `Header Search Paths` -> 添加`/usr/include/libxml2`
	- 继续搜`other linker flags` -> 添加`-lxml2`
	- 如果编译还报错
		- 工程 -> **Targets** -> 选择对应的Target -> `Header Search Paths` -> 添加`$(SDKROOT)/usr/include/libxml2`

- 问题2: `Undefined symbols for architecture x86_64:
  "_dns_free_resource_record", referenced from:`，原因是因为类库`libidn.a`静态库引起的
	- 因为XMPP默认只支持`32位`，而使用的libdn.a就只支持`32位的cpu`
	- 因此，解决只能是重新编译`libldn`的源码，在`64位`编译器下，重新生成.a文件

	- 完成步骤:
		- 1) `http://ftp.gnu.org/gnu/libidn/`，下载libidn最新源码
		- 2) 将下载的libidn最新源码文件夹放到某个位置
		- 3) 终端命令cd进入到源码的根目录 `cd /Users/xiongzenghui/libidn-1.32 `
		- 4) 执行命令`./configure --prefix=/usr/local/iphone --host=arm-apple-darwin --disable-shared CC=/Developer/Platforms/iPhoneOS.platform/Developer/usr/bin/arm-apple-darwin9-gcc-4.0.1 CFLAGS="-arch armv6 -pipe -std=c99 -Wno-trigraphs -fpascal-strings -fasm-blocks -O0 -Wreturn-type -Wunused-variable -fmessage-length=0 -fvisibility=hidden -miphoneos-version-min=2.0 -gdwarf-2 -mthumb -isysroot /Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS2.0.sdk" CPP=/Developer/Platforms/iPhoneOS.platform/Developer/usr/bin/cpp AR=/Developer/Platforms/iPhoneOS.platform/Developer/usr/bin/ar`
		- 5) 编译报错，等找到解决方法再回来写
	
****

###功能点、登陆

- 登陆完成的流程

	- 1) 客户端向服务端发送`XMPPJID`，请求建立连接
	- 2) 服务端建立与客户端的Socket连接，返回给客户端一个响应
		- 建立连接成功
		- 建立连接失败
	- 3) 客户端发送账号的密码给服务端验证

	
- 关于XMPP的相关代码，写在AppDelegate里面

- AppDelegate.h，提供XMPP相关功能入口

```
@interface AppDelegate : NSObject <UIApplicationDelegate>

/**
 *  XMPP客户端用户登陆
 */
- (void)xmppLoginWithAccount:(NSString *)account
                    Password:(NSString *)pwd;

@end
```

- AppDelegate.m，匿名分类声明`XMPPStream`等属性引用

```
#import "AppDelegate.h"

//导入XMPP Ai
#import "XMPPFramework.h"

@interface AppDelegate() <XMPPStreamDelegate> {
    
    NSString *_tmpLoginPwd;//临时保存登陆密码
}

/**
 *  所有XMPP操作都是使用这个Api类
 */
@property (nonatomic, strong) XMPPStream *xmppStream;

/**
 *  XMPPStream的delegate方法回调执行的所在队列
 */
@property (nonatomic, assign) dispatch_queue_t xmppStreamDelegateQueue;

@end
```

- AppDelegate.m，实现部分，使用懒加载实例化`XMPPStream`

```
@implementation AppDelegate

//省略其他代码....

#pragma mark - 懒加载

- (XMPPStream *)xmppStream {
    if (!_xmppStream) {
        
        //1. 实例化XMPPStream
        _xmppStream = [[XMPPStream alloc] init];
        
        //2.设置XMPPStream实例的代理、代理回调所在队列
        
        /*
         这样下，不分配一个回调方法执行的队列，那么XMPPStreamDelegate方法不会被执行
        [_xmppStream addDelegate:self
                   delegateQueue:nil];
         */
        
        //一定要分配一个队列
        [_xmppStream addDelegate:self
                   delegateQueue:dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)];
    }
    return _xmppStream;
}

//省略其他代码....

@end
``` 

- AppDelegate.m，实现部分，发起一个与XMPP服务器连接的`请求`

```
@implementation AppDelegate

//省略其他代码....

/**
 *  发起与XMPP服务器连接的请求
 */
- (void)sendConnectRequestWithAccount:(NSString *)account
{
    //1. 将传入的字符串账号转换成XMPPJID
    XMPPJID *jid = [XMPPJID jidWithUser:account domain:@"webchat.im" resource:@"iphone"];
    
    //2. 设置要建立连接JID
    self.xmppStream.myJID  = jid;
    
    //3. 设置xmppstream的host
    self.xmppStream.hostName = @"127.0.0.1";
    
    //4. 设置xmppstream的端口号
    self.xmppStream.hostPort = 5222;
    
    //5. 连接服务器
    NSError *error = nil;
    [self.xmppStream connectWithTimeout:-1 error:&error];
    
    if (error) {
        NSLog(@"建立XMPP连接失败: %@\n", [error localizedDescription]);
    } else {
        NSLog(@"【发起】建立XMPP服务器成功\n");
    }
}

//省略其他代码....

@end
```

- AppDelegate.m，实现部分，给外界暴露的登陆接口

```
@implementation AppDelegate

//省略其他代码....

#pragma mark - 登陆

- (void)xmppLoginWithAccount:(NSString *)account
                    Password:(NSString *)pwd
{
    
    //1. 发送连接XMPP服务器请求
    [self sendConnectRequestWithAccount:account];
    
    //2. 临时保存密码
    _tmpLoginPwd = pwd;
}

//省略其他代码....

@end
```

- AppDelegate.m，实现部分，发起连接请求之后的回调，实现`XMPPStreamDelegate`定义的回调函数

```
@implementation AppDelegate

//省略其他代码....

#pragma mark - XMPPStreamDelegate

#pragma mark 连接状态回调

#pragma mark - XMPPStreamDelegate

- (void)xmppStreamWillConnect:(XMPPStream *)sender
{
    NSLog(@"%s\n", __func__);
}

- (void)xmppStream:(XMPPStream *)sender socketDidConnect:(GCDAsyncSocket *)socket
{
    NSLog(@"%s\n", __func__);
}

- (void)xmppStreamDidConnect:(XMPPStream *)sender
{
    NSLog(@"%s\n", __func__);
    
    //发送验证登陆密码
    [self authentic];
}

@end

//省略其他代码....

@end
```

- AppDelegate.m，实现部分，完成发送验证密码

```
@implementation AppDelegate

//省略其他代码....

- (void)authentic
{
    NSError *error = nil;
    
    [self.xmppStream authenticateWithPassword:_tmpLoginPwd error:&error];
    
    if (error) {
        NSLog(@"发送登陆密码验证请求失败\n");
    } else {
        NSLog(@"发送登陆密码验证请求成功\n");
    }
}

//省略其他代码....

@end
```

- AppDelegate.m，实现部分，验证结果的回调中，发送一个`上线`请求

```
@implementation AppDelegate

//省略其他代码....

#pragma mark - XMPPStreamDelegate

#pragma mark 连接状态回调

//省略...

#pragma mark 登陆密码验证回调

//验证成功
- (void)xmppStreamDidAuthenticate:(XMPPStream *)sender {
    
	[self sendPresentNotify];
}

//验证失败
- (void)xmppStream:(XMPPStream *)sender didNotAuthenticate:(NSXMLElement *)error {
    
}

/**
 *  发送一个上线通知
 */
- (void)sendPresentNotify {
    
    //1. 创建一个代表上线的XML元素实例
	XMPPPresence *presence = [XMPPPresence presenceWithType:@"available"];
    
    //2. 发送这个XML元素到XMPP服务器
    [self.xmppStream sendElement:presence];
}

//省略其他代码....

@end
```

- AppDelegate.m，实现部分，修改之前的登陆接口，防止不断的建立连接造成报错`Attempting to connect while already connected or connecting`

```
@implementation AppDelegate

//省略其他代码....

#pragma mark - 登陆

- (void)xmppLoginWithAccount:(NSString *)account
                    Password:(NSString *)pwd
{
    //1.【重要】登陆前先断开之前的连接
    [self.xmppStream disconnect];
    
    //2. 发送连接XMPP服务器请求
    [self sendConnectRequestWithAccount:account];
    
    //3. 临时保存密码
    _tmpLoginPwd = pwd;
}

//省略其他代码....

@end
```

OK，登陆到此为止。

***

####功能点、注销登陆

- AppDelegate.m，实现部分，在AppDelegate.h中添加注销的入口

```
#import <UIKit/UIKit.h>


@interface AppDelegate : UIResponder <UIApplicationDelegate>


@property (strong, nonatomic) UIWindow *window;

/**
 *  XMPP客户端用户登陆
 */
- (void)xmppLoginWithAccount:(NSString *)account
                    Password:(NSString *)pwd;

/**
 *  注销登陆
 */
- (void)xmppLogout;

@end
```

- AppDelegate.m，实现部分，发送一个用户`注销`登陆的请求

```
@implementation AppDelegate

//省略其他代码....

- (void)xmppLogout
{
    //1. 创建一个代表下线的XML元素实例
    XMPPPresence *dePresence = [XMPPPresence presenceWithType:@"unavailable"];
    
    //2. 发送下线通知
    [self.xmppStream sendElement:dePresence];
    
    //3. 断开与服务器的连接
    [self.xmppStream disconnect];
    
}

//省略其他代码....

@end
```

- AppDelegate.m，实现部分，发起一个与XMPP服务器连接的`请求`

```
@implementation AppDelegate

//省略其他代码....


//省略其他代码....

@end
```

- AppDelegate.m，实现部分，发起一个与XMPP服务器连接的`请求`

```
@implementation AppDelegate

//省略其他代码....


//省略其他代码....

@end
```

****

###将XMPP操作代码，全部单独封装到一个单例类`XmppOperationManager`

```
.h

#import <Foundation/Foundation.h>

@interface XmppOperationManager : NSObject

+ (instancetype)manager;

/**
 *  XMPP客户端用户登陆
 */
- (void)xmppLoginWithAccount:(NSString *)account
                    Password:(NSString *)pwd;

/**
 *  注销登陆
 */
- (void)xmppLogout;

@end
```


```
.m

#import "XmppOperationManager.h"
#import "XMPPFramework.h"

@interface XmppOperationManager () <XMPPStreamDelegate> {
    
    NSString *_tmpLoginPwd;//临时保存登陆密码
}

@property (nonatomic, strong) XMPPStream *xmppStream;

@end

@implementation XmppOperationManager

+ (instancetype)manager {
    static XmppOperationManager *manager = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        manager = [[XmppOperationManager alloc] init];
    });
    return manager;
}

#pragma mark - 懒加载

- (XMPPStream *)xmppStream {
    if (!_xmppStream) {
        
        //1. 实例化XMPPStream
        _xmppStream = [[XMPPStream alloc] init];
        
        //2.设置XMPPStream实例的代理、代理回调所在队列
        
        /*
         这样下，不分配一个回调方法执行的队列，那么XMPPStreamDelegate方法不会被执行
         [_xmppStream addDelegate:self
         delegateQueue:nil];
         */
        
        //一定要分配一个队列
        [_xmppStream addDelegate:self
                   delegateQueue:dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)];
    }
    return _xmppStream;
}

#pragma mark - 登陆

- (void)xmppLoginWithAccount:(NSString *)account
                    Password:(NSString *)pwd
{
    //1. 登陆前先断开之前的连接
    [self.xmppStream disconnect];
    
    //2. 发送连接XMPP服务器请求
    [self sendConnectRequestWithAccount:account];
    
    //3. 临时保存密码
    _tmpLoginPwd = pwd;
}

#pragma mark - private

/**
 *  发起与XMPP服务器连接的请求
 */
- (void)sendConnectRequestWithAccount:(NSString *)account
{
    //1. 将传入的字符串账号转换成XMPPJID
    XMPPJID *jid = [XMPPJID jidWithUser:account domain:@"webchat.im" resource:@"iphone"];
    
    //2. 设置要建立连接JID
    self.xmppStream.myJID  = jid;
    
    //3. 设置xmppstream的host
    self.xmppStream.hostName = @"127.0.0.1";
    
    //4. 设置xmppstream的端口号
    self.xmppStream.hostPort = 5222;
    
    //5. 连接服务器
    NSError *error = nil;
    [self.xmppStream connectWithTimeout:-1 error:&error];
    
    if (error) {
        NSLog(@"建立XMPP连接失败: %@\n", [error localizedDescription]);
    } else {
        NSLog(@"【发起】建立XMPP服务器成功\n");
    }
}

/**
 *  验证登陆密码
 */
- (void)authentic
{
    NSError *error = nil;
    
    [self.xmppStream authenticateWithPassword:_tmpLoginPwd error:&error];
    
    if (error) {
        NSLog(@"发送登陆密码验证请求失败\n");
    } else {
        NSLog(@"发送登陆密码验证请求成功\n");
    }
}

/**
 *  发送一个上线通知
 */
- (void)sendPresentNotify {
    
    //1. 创建一个代表上线的XML元素实例
    XMPPPresence *presence = [XMPPPresence presenceWithType:@"available"];
    
    //2. 发送这个XML元素到XMPP服务器
    [self.xmppStream sendElement:presence];
}

- (void)xmppLogout
{
    //1. 创建一个代表下线的XML元素实例
    XMPPPresence *dePresence = [XMPPPresence presenceWithType:@"unavailable"];
    
    //2. 发送下线通知
    [self.xmppStream sendElement:dePresence];
    
    //3. 断开与服务器的连接
    [self.xmppStream disconnect];
    
}

#pragma mark - XMPPStreamDelegate

#pragma mark 连接状态回调

- (void)xmppStreamWillConnect:(XMPPStream *)sender
{
    NSLog(@"%s\n", __func__);
}

- (void)xmppStream:(XMPPStream *)sender socketDidConnect:(GCDAsyncSocket *)socket
{
    NSLog(@"%s\n", __func__);
}

- (void)xmppStreamDidConnect:(XMPPStream *)sender
{
    NSLog(@"%s\n", __func__);
    
    //发送验证登陆密码的请求
    [self authentic];
}

#pragma mark 登陆密码验证回调

//验证成功
- (void)xmppStreamDidAuthenticate:(XMPPStream *)sender {
    NSLog(@"%s\n", __func__);
    
    [self sendPresentNotify];
}

//验证失败
- (void)xmppStream:(XMPPStream *)sender didNotAuthenticate:(NSXMLElement *)error {
    NSLog(@"%s\n", __func__);
}

@end
```

***

###功能点、注册XMPP账号

- **代码改进**，添加表示注册、登陆操作的标志位，区分是注册还是登陆
	- 因为登陆、注册，首先`都需要建立一个XMPP服务器连接`
	- 然后再验证密码或注册密码
	- 所以，在建立连接的回调函数中，判断进行登陆还是注册

- 并且针对方块块，使用不同的方法`前缀`

```
.h

#import <Foundation/Foundation.h>

@interface XmppOperationManager : NSObject

+ (instancetype)manager;

/**
 *  XMPP用户注册
 */
- (void)xmpp_registWithAccount:(NSString *)account
                      Password:(NSString *)pwd;

/**
 *  XMPP客户端用户登陆
 */
- (void)xmpp_loginWithAccount:(NSString *)account
                    Password:(NSString *)pwd;

/**
 *  注销登陆
 */
- (void)xmpp_logout;

@end
```

```
.m

#import "XmppOperationManager.h"
#import "XMPPFramework.h"

/**
 * 用来标记当前所造的XMPP操作的标记位
 */
typedef NS_ENUM(NSInteger, XMPPOperationType) {
    XMPPOperationTypeNone = 0x1,
    XMPPOperationTypeRegist ,
    XMPPOperationTypeLogin ,
};

@interface XmppOperationManager () <XMPPStreamDelegate> {
    
    //临时保存登陆密码
    NSString *_tmpLoginPwd;
    
    //临时保存所做的XMPP操作类型
    XMPPOperationType _currntOpType;
}

//XMPP操作Api的最核心的类
@property (nonatomic, strong) XMPPStream *xmppStream;

@end

@implementation XmppOperationManager

+ (instancetype)manager {
    static XmppOperationManager *manager = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        manager = [[XmppOperationManager alloc] init];
    });
    return manager;
}

#pragma mark - 懒加载

- (XMPPStream *)xmppStream {
    if (!_xmppStream) {
        
        //1. 实例化XMPPStream
        _xmppStream = [[XMPPStream alloc] init];
        
        //2.设置XMPPStream实例的代理、代理回调所在队列
        
        /*
         这样下，不分配一个回调方法执行的队列，那么XMPPStreamDelegate方法不会被执行
         [_xmppStream addDelegate:self
         delegateQueue:nil];
         */
        
        //一定要分配一个队列
        [_xmppStream addDelegate:self
                   delegateQueue:dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)];
    }
    return _xmppStream;
}

#pragma mark - tools工具函数

- (XMPPJID *)tool_jidWithAccount:(NSString *)account {
    return [XMPPJID jidWithUser:account domain:@"webchat.im" resource:@"iphone"];
}

- (void)tool_setRegistOperation {
    _currntOpType = XMPPOperationTypeRegist;
}

- (BOOL)tool_isRegistOperation {
    if (_currntOpType == XMPPOperationTypeRegist) {
        return YES;
    }
    
    return NO;
}

- (void)tool_setLoginOperation {
    _currntOpType = XMPPOperationTypeLogin;
}

- (BOOL)tool_isLoginOperation {
    if (_currntOpType == XMPPOperationTypeLogin) {
        return YES;
    }
    
    return NO;
}

- (void)tool_resetOperation {
    _currntOpType = XMPPOperationTypeNone;
}

#pragma mark - Private私有XMPP操作函数

/**
 *  发起与XMPP服务器连接的请求
 */
- (void)private_sendConnectRequestWithAccount:(NSString *)account
{
    //1. 将传入的字符串账号转换成XMPPJID
    XMPPJID *jid = [self tool_jidWithAccount:account];
    
    //2. 设置要建立连接JID
    self.xmppStream.myJID  = jid;
    
    //3. 设置xmppstream的host
    self.xmppStream.hostName = @"127.0.0.1";
    
    //4. 设置xmppstream的端口号
    self.xmppStream.hostPort = 5222;
    
    //5. 连接服务器
    NSError *error = nil;
    [self.xmppStream connectWithTimeout:-1 error:&error];
    
    if (error) {
        NSLog(@"建立XMPP连接失败: %@\n", [error localizedDescription]);
    } else {
        NSLog(@"【发起】建立XMPP服务器成功\n");
    }
}

/**
 *  验证登陆密码
 */
- (void)private_authentic
{
    NSError *error = nil;
    
    [self.xmppStream authenticateWithPassword:_tmpLoginPwd error:&error];
    
    if (error) {
        NSLog(@"发送登陆密码验证请求失败\n");
    } else {
        NSLog(@"发送登陆密码验证请求成功\n");
    }
}

/**
 *  发送一个上线通知
 */
- (void)private_sendPresentNotify {
    
    //1. 创建一个代表上线的XML元素实例
    XMPPPresence *presence = [XMPPPresence presenceWithType:@"available"];
    
    //2. 发送这个XML元素到XMPP服务器
    [self.xmppStream sendElement:presence];
}

/**
 *  发送一个上线通知
 */
- (void)private_regist {
    
    NSError *error = nil;
    
    [self.xmppStream registerWithPassword:_tmpLoginPwd error:&error];
    
    if (error) {
        NSLog(@"发送注册密码验证请求失败\n");
    } else {
        NSLog(@"发送注册密码验证请求成功\n");
    }
}

#pragma mark - 对外暴露的XMPP操作入口

- (void)xmpp_registWithAccount:(NSString *)account
                 Password:(NSString *)pwd
{
    //1. 标记当前进行注册操作
    [self tool_setRegistOperation];
    
    //2. 断开之前的XMPP服务器连接
    [self.xmppStream disconnect];
    
    //3. 再发起新的XMPP服务器连接请求
    [self private_sendConnectRequestWithAccount:account];
    
    //4. 临时保存密码
    _tmpLoginPwd = pwd;
}

- (void)xmpp_loginWithAccount:(NSString *)account
                         Password:(NSString *)pwd
{
    //1. 记录当前为登陆操作
    [self tool_setLoginOperation];
    
    //2. 登陆前先断开之前的连接
    [self.xmppStream disconnect];
    
    //3. 发送连接XMPP服务器请求
    [self private_sendConnectRequestWithAccount:account];
    
    //4. 临时保存密码
    _tmpLoginPwd = pwd;
}

- (void)xmpp_logout
{
    //1. 创建一个代表下线的XML元素实例
    XMPPPresence *dePresence = [XMPPPresence presenceWithType:@"unavailable"];
    
    //2. 发送下线通知
    [self.xmppStream sendElement:dePresence];
    
    //3. 断开与服务器的连接
    [self.xmppStream disconnect];
    
}

#pragma mark - XMPPStreamDelegate

#pragma mark 连接状态回调

- (void)xmppStreamWillConnect:(XMPPStream *)sender
{
    NSLog(@"%s\n", __func__);
}

- (void)xmppStream:(XMPPStream *)sender socketDidConnect:(GCDAsyncSocket *)socket
{
    NSLog(@"%s\n", __func__);
}

- (void)xmppStreamDidConnect:(XMPPStream *)sender
{
    NSLog(@"%s\n", __func__);
    
    //建立连接是进行登陆
    if ([self tool_isRegistOperation]) {
        [self private_regist];
    }

    //建立连接是进行注册
    if ([self tool_isLoginOperation]) {
        [self private_authentic];
    }
}

#pragma mark 登陆密码验证结果回调

//验证成功
- (void)xmppStreamDidAuthenticate:(XMPPStream *)sender {
    NSLog(@"%s\n", __func__);
    
    //发送上线通知
    [self private_sendPresentNotify];
    
    //重置操作标记
    [self tool_resetOperation];
}

//验证失败
- (void)xmppStream:(XMPPStream *)sender didNotAuthenticate:(NSXMLElement *)error {
    NSLog(@"%s\n", __func__);
    
    //重置操作标记
    [self tool_resetOperation];
}

#pragma mark 注册结果回调

- (void)xmppStreamDidRegister:(XMPPStream *)sender {
    
    NSLog(@"注册成功...\n");
    
    //重置操作标记
    [self tool_resetOperation];
}

- (void)xmppStream:(XMPPStream *)sender didNotRegister:(NSXMLElement *)error {
    
    NSLog(@"注册失败...\n");
    
    //重置操作标记
    [self tool_resetOperation];
}

@end
```

****

###功能点、使用XMPPFramework中的extensions的模块的步骤

- 第一步、修改`XMPPFramework.h`导入需要的extension模块的头文件

```
#import "XMPP.h"

//如下为手动导入extensions中的功能木块
#import "extensions模块头文件.h"
```

- 第二步、修改`XmppOperationManager.m`，在其匿名分类声明扩展模块需要的类实例属性

```
@interface XmppOperationManager () <XMPPStreamDelegate> 

@property (nonatomic, strong) XMPPStream *xmppStream;

@property (nonatomic, strong) 实体类 *实体;
@property (nonatomic, strong) 实体类CoreDataStorage *实体Storage;

@end
```

- 第三步、修改`XmppOperationManager.m`，在初始化`XMPPStream`时，注册其extension模块

```
- (void)setupXmppStream {
    
    if (_xmppStream) {
        return;
    }
    
    //1. 实例化XMPPStream
    _xmppStream = [[XMPPStream alloc] init];
    
    //2.设置XMPPStream实例的代理、代理回调所在队列（一定要分配一个回调代理队列）
    [_xmppStream addDelegate:self
               delegateQueue:dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)];
    
    //3. 注册其他扩展模块到XMPPStream实例
    {
		//3.1 XxxCoreDataStorage负责实体类实例的持久化
	    _rosterStorage = [XMPPRosterCoreDataStorage sharedInstance];
	    
	    //3.2 使用Storage创建实体类实例
	    _roster = [[XMPPRoster alloc] initWithRosterStorage:_rosterStorage];
	    
	    //3.3 设置实体类代理对象
	    [_roster addDelegate:self delegateQueue:dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)];
	    
	    //3.4 注册到XMPPStream实例
	    [_roster activate:_xmppStream];
    }
}
```

- 第四步、修改`XmppOperationManager.m`，在`XmppOperationManager实例dealloc`时，释放掉一切关于XMPP的对象

```
- (void)destroyXMPPStream {
    
    //1. 移除XMPPStream实例的代理
    [_xmppStream removeDelegate:self];
    
    //2. 取消注册的各种模块
    {	
        //停止电子名片模块
        [_vCardModule deactivate];
        
        //停止头像模块
        [_vCardAvatar deactivate];
    }
    
    //3. 断开连接
    [_xmppStream disconnect];
    
    //4. 释放内存
    [_xmppStream removeDelegate:self];
    _xmppStream = nil;
    
    _vCardStorage = nil;
    
    [_vCardModule removeDelegate:self];
    _vCardModule = nil;
    
    [_vCardAvatar removeDelegate:self];
    _vCardAvatar = nil;
}
```

****

###功能点、自动进行XMPPStream重连

- 修改`XMPPFramework.h`导入消息扩展模块头文件

```
#import "XMPP.h"

//如下为手动导入extensions中的功能木块

//当断开连接时自动重新
#import "XMPPReconnect.h"
```

- 修改`XmppOperationManager.m`匿名分类，添加属性

```
@interface XmppOperationManager () <XMPPStreamDelegate, XMPPReconnectDelegate> 

@property (nonatomic, strong) XMPPStream *xmppStream;

@property (nonatomic, strong) XMPPReconnect *reconnect;

@end
```

- 修改`XmppOperationManager.m`实例化时，创建`XMPPStream`时注册消息模块

```
@implementation XmppOperationManager

//...

- (void)setupXmppStream {
    
    if (_xmppStream) {
        return;
    }
    
    //1. 实例化XMPPStream
    _xmppStream = [[XMPPStream alloc] init];
    
    //2.设置XMPPStream实例的代理、代理回调所在队列
    [_xmppStream addDelegate:self
               delegateQueue:dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)];
    
    //3.
    _reconnect = [[XMPPReconnect alloc] init];
    [_reconnect addDelegate:self delegateQueue:dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)];
    [_reconnect activate:_xmppStream];
    
    //...
}

//...

@end
```


- 修改`XmppOperationManager.m`实例销毁时，从`XMPPStream`中取消注册、移除代理对象、释放资源

```
- (void)destroyXMPPStream {
    
    //1. 移除XMPPStream实例的代理
    [_xmppStream removeDelegate:self];
    
    //2. 取消注册的各种模块
    {
        //停止重连
        [_reconnect deactivate];
        
        //...
    }
    
    //3. 断开连接
    [_xmppStream disconnect];
    
    //4. 释放内存
    [_xmppStream removeDelegate:self];
    _xmppStream = nil;
    
    [_reconnect removeDelegate:self];
    _reconnect = nil;
    
    //...
    
}

```

- 修改`XmppOperationManager.m`实例方法，实现XMPPReconnectDelegate

```
#pragma mark - XMPPReconnectDelegate

- (void)xmppReconnect:(XMPPReconnect *)sender didDetectAccidentalDisconnect:(SCNetworkReachabilityFlags)connectionFlags
{
	//已经重新建立连接
}

- (BOOL)xmppReconnect:(XMPPReconnect *)sender shouldAttemptAutoReconnect:(SCNetworkReachabilityFlags)reachabilityFlags
{
	//是否进行重连
	return YES;
}
```


****


###功能点、获取电子名片、头像

- 关于电子名片的注意点
	- 1) 电子名片的功能，属于XMPP的`Extensions`中的，默认下XMPP框架并未开启使用
	- 2) 所在模块是`XEP-0054`

#### 开启`Extentions`中未使用的模块步骤

- 第一步、在`XMPPFramework.h`中加入功能木块的头文件

```
#import "XMPP.h"

//如下为手动导入extensions中的功能木块

//导入电子名片
#import "XMPPvCardCoreDataStorage.h"
#import"XMPPvCardTempModule.h"

// 头像模块
#import "XMPPvCardAvatarModule.h"

//花名册
#import "XMPPRoster.h"
#import "XMPPRosterCoreDataStorage.h"
```

- 第二步、在`XmppOperationManager`定义电子名片模块需要的属性，声明`XMPPvCardTempModuleStorage`和`XMPPvCardAvatarDelegate`

```
@interface XmppOperationManager () <XMPPStreamDelegate, XMPPvCardTempModuleStorage, XMPPvCardAvatarDelegate> 

//电子名片使用的属性
@property (nonatomic, strong) XMPPvCardCoreDataStorage *vCardStorage;//持久化存储器
@property (nonatomic, strong) XMPPvCardTempModule *vCardModule;//全局电子名片模块

//头像使用的属性
@property (nonatomic, strong) XMPPvCardAvatarModule *vCardAvatar;

@end

//省略...
```

- 第三步、修改之前的代码`XmppOperationManager`

	- 修改1、XmppOperationManager实例的初始化`init`创建XMPPStream实例
	- 修改2、在实例化XMPPStream实例时，向其添加`注册电子名片模块`的代码
	- 修改3、XmppOperationManager实例的初始化`dealloc`销毁XMPPStream实例、取消注册功能模块、释放对象

```
//省略...

@implementation XmppOperationManager

+ (instancetype)manager {
    static XmppOperationManager *manager = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        manager = [[XmppOperationManager alloc] init];
    });
    return manager;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        [self setupXmppStream];
    }
    return self;
}

/**
 *	常见XMPPStream实例、以及注册其他未开启的extensions模块
 */
- (void)setupXmppStream {

	if (_xmppStream) {
		return;
	}
    
    //1. 实例化XMPPStream
    _xmppStream = [[XMPPStream alloc] init];
    
    //2.设置XMPPStream实例的代理、代理回调所在队列
    
    /*
     这样下，不分配一个回调方法执行的队列，那么XMPPStreamDelegate方法不会被执行
     [_xmppStream addDelegate:self
     delegateQueue:nil];
     */
    
    //一定要分配一个队列
    [_xmppStream addDelegate:self
               delegateQueue:dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)];
    
    //3. 注册电子名片功能，并设置代理对象
    _vCardStorage = [XMPPvCardCoreDataStorage sharedInstance];
    _vCardModule = [[XMPPvCardTempModule alloc] initWithvCardStorage:_vCardStorage];
    [_vCardModule addDelegate:self delegateQueue:dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)];
    [_vCardModule activate:_xmppStream];
    
    //4. 注册头像功能（使用电子名片功能完成）
    _vCardAvatar = [[XMPPvCardAvatarModule alloc] initWithvCardTempModule:_vCardModule];
	[_vCardAvatar addDelegate:self delegateQueue:dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)];
    [_vCardAvatar activate:_xmppStream];
}

/**
 *	销毁XMPPStream实例做的事情
 */
- (void)destroyXMPPStream {
    
    //1. 移除XMPPStream实例的代理
    [_xmppStream removeDelegate:self];
    
    //2. 取消注册的各种模块
    {
        //停止电子名片模块
        [_vCardModule deactivate];
        
        //停止头像模块
        [_vCardAvatar deactivate];
        
        //其他添加的extensions模块
        //...
    }
    
    //3. 断开连接
    [_xmppStream disconnect];
    
     //4. 释放内存
    [_xmppStream removeDelegate:self];
    _xmppStream = nil;
    
    _vCardStorage = nil;
    
    [_vCardModule removeDelegate:self];
    _vCardModule = nil;
    
    [_vCardAvatar removeDelegate:self];
    _vCardAvatar = nil;
}

//省略...

@end
```

- 第四步、读取电子名片与头像，在`XmppOperationManager`提供获取电子名片与头像的入口函数

```
#import <Foundation/Foundation.h>

//电子名片与头像
#import "XMPPvCardTemp.h"

@interface XmppOperationManager : NSObject

//省略...

/**
 *  获取电子名片与头像相关数据
 */
- (XMPPvCardTemp *)xmpp_vCardTemp;

//省略...

@end
```

- 第五步、在`XmppOperationManager`添加获取`XMPPvCardTemp实例`的方法

```
@implementation XmppOperationManager

//省略...

#pragma mark - 对外暴露的XMPP操作入口

//省略...

- (XMPPvCardTemp *)xmpp_vCardTemp {

	//很简单，直接返回，不需要任何处理
    return _vCardModule.myvCardTemp;
}

//省略...

@end
```

####分析如上第五步中，获取电子名片与头像的数据为何这么简单？

当我们开启了电子名片和头像的功能后，使用XMPPFramewoek API进行注册的操作之后，XMPPFrmaework为我们做的事情:

- 事情1、 在沙盒目录`Library`目录，创建`/Application Support/项目名`两个层级的子目录
	- 如: `Library/Application Support/XMPPDemo/`目录

- 事情2、 建立sqlite数据库: `XMPPvCard.sqlite`，有三个文件
	- 文件1，XMPPvCard.sqlite
	- 文件2，XMPPvCard.sqlite-shm
	- 文件3，XMPPvCard.sqlite-wal

- 事情3、 从XMPP服务器，读取电子名片和头像的数据
- 事情4、 将读取的数据，使用CoreData写入到数据库表中

所以，我们使用`-[XMPPvCardTempModule myvCardTemp]`获取数据时，直接使用CoreData从本地数据库读取的数据
	- `-[XMPPvCardTemp photo]`
	- `-[XMPPvCardTemp nikename]`
	- 等等

- 但是注意使用`-[XMPPvCardTemp jid]`返回的是nil
	- 原因当前使用的XMPP服务器`Openfire`返回的`响应XML`中，没有`JABBERID`节点
	- 如下如`-[XMPPvCardTemp jid]`实现
	
	```
	- (XMPPJID *)jid {
		XMPPJID *jid = nil;
		
		//读取响应XML中的名字为JABBERID的节点元素
		NSXMLElement *elem = [self elementForName:@"JABBERID"];
			
		if (elem != nil) {
			jid = [XMPPJID jidWithString:[elem stringValue]];
		}
			
		return jid;
	}
	```

****

###功能点、配置XMPP日志框架

- 先开启XMPP日志功能
	
	- 找到`XMPPLogging.h`，看如下代码宏定义是1，就表示开启日志

	```
	#ifndef XMPP_LOGGING_ENABLED
		#define XMPP_LOGGING_ENABLED 1
	#endif
	```
	
	- 配置XMPP依赖的第三方日志框架`CocoaLumberjack `

	```
	Common.h，配置CocoaLumberjack日志框架的等级
	
	#import "DDLog.h"
	#import "DDTTYLogger.h"
	#import "DDFileLogger.h"

	#ifdef DEBUG
	static const int ddLogLevel = LOG_LEVEL_VERBOSE;
	#else
	static const int ddLogLevel = LOG_LEVEL_OFF;
	#endif
	```
		
	```
	AppDelegate.m，添加DDLog
	
	- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
	{
		[DDLog addLogger:[DDTTYLogger sharedInstance]];
		
		//省略...
	}
	```

- 查看控制台输出如下，是用来获取XMPP服务器上的电子名片和头像的请求`XML数据`

```
SEND: <iq type="get" to="haha5@webchat.im"><vCard xmlns="vcard-temp"/></iq>
```

所以，以后如果组装XMPP请求，就按照这种XML格式
	
***

###功能点、更新电子名片

- MainViewController.m 测试代码

```
/**
 *  重设电子名片
 */
- (void)resetVCard {
    
    XMPPvCardTemp *vCard = [[XmppOperationManager manager] xmpp_vCardTemp];
    
    vCard.name = @"";
    vCard.orgName = @"";
    vCard.title = @"";
    vCard.note = @"";
    vCard.mailer = @"";
    
    [[XmppOperationManager manager] xmpp_updateVCardTemp:vCard];
}
```

- `XmppOperationManager`添加暴露的更新函数入口

```
#import <Foundation/Foundation.h>

//电子名片与头像
#import "XMPPvCardTemp.h"

@interface XmppOperationManager : NSObject

//省略...

/**
 *  修改电子名片
 */
- (void)xmpp_updateVCardTemp:(XMPPvCardTemp *)temp;

//省略...

@end
```

```
//省略...

@interface XmppOperationManager () <XMPPStreamDelegate> 

//省略...

@property (nonatomic, strong) XMPPvCardTempModule *vCardModule;//全局电子名片模块

//省略...

@end

@implementation XmppOperationManager

//省略...

#pragma mark - 对外暴露的XMPP操作入口

//省略...

- (void)xmpp_updateVCardTemp:(XMPPvCardTemp *)temp {
    [_vCardModule updateMyvCardTemp:temp];
}

//省略...

@end
```

***

###功能点、上传图片用来设置头像

- 修改头像，也就是重设电子名片，类似上面的代码，只是修改`photo`属性

- MainViewController.m 测试代码

```

/**
 *  上传头像
 */
- (void)testUpLoadVCardAvator {
    UIImagePickerController *imgVC = [[UIImagePickerController alloc] init];
    
    imgVC.delegate = self;
    
    imgVC.editing = YES;
    
    //imgVC.sourceType = UIImagePickerControllerSourceTypeCamera;//拍照
    imgVC.sourceType = UIImagePickerControllerSourceTypePhotoLibrary;//图库
    
    [self presentViewController:imgVC animated:YES completion:nil];
}

#pragma mark - UIImagePickerControllerDelegate
- (void)imagePickerController:(UIImagePickerController *)picker
didFinishPickingMediaWithInfo:(NSDictionary *)info
{
    //获取选择图片
    UIImage *img = info[UIImagePickerControllerEditedImage];
    
    //dismiss
    [picker dismissViewControllerAnimated:YES completion:nil];
    
    //更新图像到XMPP服务器
    [self uploadImage:img];   
}

- (void)uploadImage:(UIImage *)img {
    //1.
    NSData *data = UIImagePNGRepresentation(img);
    
    //2.
    XMPPvCardTemp *vCard = [[XmppOperationManager manager] xmpp_vCardTemp];
    
    vCard.photo = data;
    
    [[XmppOperationManager manager] xmpp_updateVCardTemp:vCard];
}
```

***

###功能点、花名册

- 第一步、`XMPPFramework.h`导入其扩展模块

```
#import "XMPP.h"

//如下为手动导入extensions中的功能木块

//省略...

//花名册
#import "XMPPRoster.h"
#import "XMPPRosterCoreDataStorage.h"
```

- 第二步、在`XmppOperationManager`声明其需要的属性引用

```
@property (nonatomic, strong) XMPPRoster *roster;
@property (nonatomic, strong) XMPPRosterCoreDataStorage *rosterStorage;
```

- 第三步、将其模块注册到`XMPPStream`实例化的代码处

```
@implementation XmppOperationManager

//省略....

- (void)setupXmppStream {
    
    if (_xmppStream) {
        return;
    }
    
    //1. 实例化XMPPStream
    _xmppStream = [[XMPPStream alloc] init];
    
    //2.设置XMPPStream实例的代理、代理回调所在队列
    
    /*
     这样下，不分配一个回调方法执行的队列，那么XMPPStreamDelegate方法不会被执行
     [_xmppStream addDelegate:self
     delegateQueue:nil];
     */
    
    //一定要分配一个队列
    [_xmppStream addDelegate:self
               delegateQueue:dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)];
    
    //3. 注册电子名片功能
	//...
    
    //4. 注册头像功能（使用电子名片功能完成）
    //...
    
    //5. 花名册
    _rosterStorage = [XMPPRosterCoreDataStorage sharedInstance];
    _roster = [[XMPPRoster alloc] initWithRosterStorage:_rosterStorage];
    [_roster addDelegate:self delegateQueue:dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)];
    [_roster activate:_xmppStream];
}


//省略....

@end
```

####获取好友列表（花名册）

- 第一步、修改`XmppOperationManager.h`，添加用来暴露获取花名册入口函数

```
#import <Foundation/Foundation.h>

@interface XmppOperationManager : NSObject

//省略...

//花名册操作

//1)被动接受变化处理，在回调Block中调用如上方法，获取最新的数据
@property (nonatomic, copy) void (^onRosterDidChanged)(void);

//2) 主动获取数据
- (NSInteger)xmpp_sections;
- (NSInteger)xmpp_rowsForSection:(NSInteger)section;
- (id)xmpp_rosterForSection:(NSInteger)section Row:(NSInteger)row;

//省略...

@end
```

- 第二步、修改`XmppOperationManager.m`，在其匿名分类中，声明XMPPRoster操作相关的类属性，以及实现`NSFetchedResultsControllerDelegate`协议方法

```
//1. 实现NSFetchedResultsControllerDelegate，在花名册数据改变时回调执行
@interface XmppOperationManager () <XMPPStreamDelegate, NSFetchedResultsControllerDelegate>

//省略...


//2. 花名册相关的属性
@property (nonatomic, strong) XMPPRoster *roster;
@property (nonatomic, strong) XMPPRosterCoreDataStorage *rosterStorage;
@property (nonatomic, strong) NSFetchedResultsController *rosterResultController;

//省略...

@end
```

- 第三步、修改`XmppOperationManager.m`，在`XMPPStream`实例化时注册XMPPRoster模块

```
@implementation XmppOperationManager

//...

- (void)setupXmppStream {
    
    if (_xmppStream) {
        return;
    }
    
    //1. 实例化XMPPStream
    _xmppStream = [[XMPPStream alloc] init];
    
    //2. 设置XMPPStream实例的代理、代理回调所在队列
    [_xmppStream addDelegate:self
               delegateQueue:dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)];
               
	//3. 注册花名册扩展模块
	_rosterStorage = [XMPPRosterCoreDataStorage sharedInstance];
    _roster = [[XMPPRoster alloc] initWithRosterStorage:_rosterStorage];
    //设置XMPPRoster实例的代理方法实现的实现类实例
	[_roster addDelegate:self delegateQueue:dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)];
    [_roster activate:_xmppStream];
}

//..

@end
```

- 第四步、修改`XmppOperationManager.m`，实现查询花名册代码
	- 1) 使用`NSFetchResultController`这个CoreData Api查询`XMPPRoster.sqlite`数据库
	- 2) 获取某个组内的某一行的具体实例`XMPPUserCoreDataStorageObject`
	- 3) 当`XMPPRoster.sqlite`数据库中的数据发生变化后的回调

```
@implementation XmppOperationManager

//...

#pragma mark - Private私有XMPP操作函数

/**
 *  完成查询花名册数据库（XMPPRoster.sqlite）表数据结果集
 */
- (NSFetchedResultsController *)rosterResultController {
    
    if (!_rosterResultController) {
        
        //1. 获取RosterCoreDataStorage的Context
        NSManagedObjectContext *moc = [_rosterStorage mainThreadManagedObjectContext];
        
        //2. 设置要查询的实体类
        NSString *entityClass = @"XMPPUserCoreDataStorageObject";
        NSEntityDescription *entity = [NSEntityDescription entityForName:entityClass
                                                  inManagedObjectContext:moc];
        
        //3. 设置结果集排序字段
        NSSortDescriptor *sd1 = [[NSSortDescriptor alloc] initWithKey:@"sectionNum" ascending:YES];
        NSSortDescriptor *sd2 = [[NSSortDescriptor alloc] initWithKey:@"displayName" ascending:YES];
        NSArray *sortDescriptors = @[sd1,sd2];
        
        //4. 创建查询请求
        NSFetchRequest *fetchRequest = [[NSFetchRequest alloc] init];
        [fetchRequest setEntity:entity];
        [fetchRequest setSortDescriptors:sortDescriptors];
        [fetchRequest setFetchBatchSize:10];
        
        //5. 过滤掉因为添加不存在的好友时，仍然显示这个好友
        NSPredicate *predicate = [NSPredicate predicateWithFormat:@"subscription != %@", @"none"];
        fetchRequest.predicate = predicate;
        
        //6. 创建FetchResultController，并设置代理
        _rosterResultController = [[NSFetchedResultsController alloc] initWithFetchRequest:fetchRequest managedObjectContext:moc sectionNameKeyPath:@"sectionNum" cacheName:nil];
        _rosterResultController.delegate = self;
        
        //7. 执行查询
        NSError *error = nil;
        if (![_rosterResultController performFetch:&error])
        {
            NSLog(@"Error performing fetch: %@", error);
        }
        
    }
    
    return _rosterResultController;
}

//...

@end
```

```
@implementation XmppOperationManager

//....

#pragma mark - 对外暴露的XMPP操作入口

//....

- (NSInteger)xmpp_sections {
    
    //1. 所有组
    NSArray *sections = [self.rosterResultController sections];
    
    //2. 所有组个数
    return [sections count];
}

- (NSInteger)xmpp_rowsForSection:(NSInteger)section {
    
    //1. 所有组
    NSArray *sections = [self.rosterResultController sections];
    
    //2. 指定组下标符合标准
    if (section < [sections count]) {
        
        //2.1 具体某个组
        id <NSFetchedResultsSectionInfo> sectionInfo = sections[section];
        
        //2.2 组内个数
        return [sectionInfo numberOfObjects];
    }
    return 0;
}

- (id)xmpp_rosterForSection:(NSInteger)section Row:(NSInteger)row {
    
    //1. 所有组
    NSArray *sections = [self.rosterResultController sections];
    
    //2. 组个数
    NSInteger sectionCount = [sections count];
    
    //3. 指定组下标符合标准
    if (section < sectionCount) {
        
        //3.1 获取组内总个数
        NSInteger rowCount = [self xmpp_rowsForSection:section];
        
        //3.2 组内数符合标准
        if (row < rowCount) {
            
            //3.2.1 构造NSIndexPath
            NSIndexPath *indexPath = [NSIndexPath indexPathForRow:row inSection:section];
            
            //3.2.2 从fetch result controller 获取指定path的实例
            XMPPUserCoreDataStorageObject *user = [self.rosterResultController objectAtIndexPath:indexPath];
            
            return user;
        }
    }
    
    return nil;
}

//...

@end
```

```
@implementation XmppOperationManager

//....

#pragma mark - NSFetchedResultsControllerDelegate

- (void)controllerDidChangeContent:(NSFetchedResultsController *)controller
{
    //花名册数据库中数据发送变化的回调，回调通知处理
    if (self.onRosterDidChanged) {
        self.onRosterDidChanged();
    }
}

//....

@end
```

- 第五步、修改`XmppOperationManager.m`，在XmppOperationManager实例dealloc时解除注册、释放内存

```
@implementation XmppOperationManager

//....

- (void)dealloc {
    [self destroyXMPPStream];
}

#pragma mark 销毁XMPPStream实例

- (void)destroyXMPPStream {
    
    //1. 移除XMPPStream实例的代理
    [_xmppStream removeDelegate:self];
    
    //2. 取消注册的各种模块
    {
        //....
        
        //花名册
        [_roster deactivate];
    }
    
    //3. 断开连接
    [_xmppStream disconnect];
    
    //4. 释放内存
    [_xmppStream removeDelegate:self];
    _xmppStream = nil;
    
    //..
    
    [_roster removeDelegate:self];
    _roster = nil;
    
    _rosterResultController.delegate = nil;
    _rosterResultController = nil;
    _onRosterDidChanged = nil;
}


//....

@end
```

####好友添加与删除

- 删除好友

```
.h

#import <Foundation/Foundation.h>

@interface XmppOperationManager : NSObject

+ (instancetype)manager;

//...

/**
 *  移除好友
 */
- (void)xmpp_removeRoster:(NSString *)account;

//...

@end
```

```
.m

//...

@implementation XmppOperationManager


- (void)xmpp_removeRoster:(NSString *)account {
    
    //1. 先构造处要删除的XMPPJID
    XMPPJID *jid = [self tool_jidWithAccount:account];
	
	 //2. 使用XMPPRoster实例删除好友的API    
    [_roster removeUser:jid];
}

#pragma mark - NSFetchedResultsControllerDelegate

/**
 *	只有XMPPRoster.sqlite数据库中的表数据变化，都会执行这个回调函数
 * 		> 添加好友
 * 	  	> 删除好友
 */
- (void)controllerDidChangeContent:(NSFetchedResultsController *)controller
{
    //花名册数据库中数据发送变化的回调，回调通知处理
    if (self.onRosterDidChanged) {
        self.onRosterDidChanged();
    }
}

```

- 添加好友，分几步

#####第一步、`XmppOperationManager.h`暴露添加好友的入口

```
.h

#import <Foundation/Foundation.h>

@interface XmppOperationManager : NSObject

+ (instancetype)manager;

//...

/**
 *  添加好友
 */
- (void)xmpp_addRoster:(NSString *)account;

//...

@end
```
	
#####第二步、`XmppOperationManager.m`，完成A账户添加B账户
	
```
.m

//...

@implementation XmppOperationManager

//...

- (void)xmpp_addRoster:(NSString *)account {
    
    //1. 先判断要添加的JID是否与当前用户JID一样，即不能自己添加自己
    if ([_xmppStream.myJID.user isEqualToString:account]) {
        return;
    }
    
    //2. 构造处要添加的XMPPJID
    XMPPJID *desJID = [self tool_jidWithAccount:account];
    
    //3. 添加JID（订阅别人）
    [_roster subscribePresenceToUser:desJID];
}

//...

@end
```

	
#####第三步、`XmppOperationManager.m`，在实现的`XMPPRosterDelegate`回调函数中，做`B账户处理A账户的添加请求`的逻辑

```
.m

//...

@implementation XmppOperationManager

//...

#pragma mark - XMPPRosterDelegate

#pragma mark 当前XMPP登陆用户接收到别人的添加好友的请求
- (void)xmppRoster:(XMPPRoster *)sender didReceivePresenceSubscriptionRequest:(XMPPPresence *)presence
{
    //1. 获取A账户的状态
    NSString *presenceType = [NSString stringWithFormat:@"%@", [presence type]]; //online或offline
    
    //2. 获取A账户的用户名
    NSString *presenceFromUser =[NSString stringWithFormat:@"%@", [[presence from] user]];
    
    //3. 将A账户转换为XMPPJID
    XMPPJID *jid = [XMPPJID jidWithString:presenceFromUser];
    
    //4. 当前B账户同意，A账户的添加好友请求
    [_roster acceptPresenceSubscriptionRequestFrom:jid andAddToRoster:YES];
}

//...

@end

```
#####第四步、当B账户同意添加A账户之后，那么A账户就会接收到一个`B账户的上线通知`，执行`XMPPStreamDelegate`中定义的回调函数

```
@implementation XmppOperationManager

//...

#pragma mark - XMPPStreamDelegate

//...

#pragma mark 接收到其他好友上线的通知

- (void)xmppStream:(XMPPStream *)sender didReceivePresence:(XMPPPresence *)presence
{
    //1. 取得好友状态
    NSString *presenceType = [NSString stringWithFormat:@"%@", [presence type]]; //online/offline
    
    //2. 获取当前登陆用户
    //NSString *userId = [NSString stringWithFormat:@"%@", [[sender myJID] user]];
    
    //3. 上线通知来自B账户
    //NSString *presenceFromUser =[NSString stringWithFormat:@"%@", [[presence from] user]];
    
    //4. A账户也必须同意B账户的添加
    if ([presenceType isEqualToString:@"subscribed"]) {//接收到的添加好友后的回应
        
        XMPPJID *jid = [XMPPJID jidWithString:[NSString stringWithFormat:@"%@",[presence from]]];
        
        [_roster acceptPresenceSubscriptionRequestFrom:jid andAddToRoster:YES];
    }
}

//...

@end
```

- 小结添加好友的步骤:(A添加B)
	- 1) A发送一个添加B的请求到XMPP服务器
	- 2) B接收到A的添加请求，并同意A的添加
	- 3) A接收到B的上线通知，A最后还需要同意B的添加

***

###功能点、消息

- 修改`XMPPFramework.h`导入消息扩展模块头文件

```
#import "XMPP.h"

//如下为手动导入extensions中的功能木块

//...

//消息
#import "XMPPMessageArchiving.h"
#import "XMPPMessageArchivingCoreDataStorage.h"
```

- 修改`XmppOperationManager.m`匿名分类，添加属性，添加`XMPPMessageArchivingStorage`声明

```
@interface XmppOperationManager () <XMPPStreamDelegate,
                                    XMPPMessageArchivingStorage>
                                    
@property (nonatomic, strong) XMPPStream *xmppStream;

//消息
@property (nonatomic, strong) XMPPMessageArchiving *msgArchiving;
@property (nonatomic, strong) XMPPMessageArchivingCoreDataStorage *msgArchivingStorage;
@property (nonatomic, strong) NSFetchedResultsController *msgResultController;//查询消息（1.从XMPP读取数据 2.写入到数据库）

@end
```

- 修改`XmppOperationManager.m`实例化时，创建`XMPPStream`时注册消息模块

```
@implementation XmppOperationManager

//...

- (void)setupXmppStream {
    
    if (_xmppStream) {
        return;
    }
    
    //1. 实例化XMPPStream
    _xmppStream = [[XMPPStream alloc] init];
    
    //2.设置XMPPStream实例的代理、代理回调所在队列
    [_xmppStream addDelegate:self
               delegateQueue:self.globalQueue];
    
    //...
        
    //注册消息模块
    _msgArchivingStorage = [XMPPMessageArchivingCoreDataStorage sharedInstance];
    _msgArchiving = [[XMPPMessageArchiving alloc] initWithMessageArchivingStorage:_msgArchivingStorage];
    [_msgArchiving addDelegate:self delegateQueue:self.globalQueue];
    [_msgArchiving activate:_xmppStream];
}

//...

@end
```

- 修改`XmppOperationManager.m`实例销毁时，从`XMPPStream`中取消注册、移除代理对象、释放资源

```
- (void)destroyXMPPStream {
    
    //1. 移除XMPPStream实例的代理
    [_xmppStream removeDelegate:self];
    
    //2. 取消注册的各种模块
    {
        //...
        
        //停止消息
        [_msgArchiving deactivate];
    }
    
    //3. 断开连接
    [_xmppStream disconnect];
    
    //4. 释放内存
    [_xmppStream removeDelegate:self];
    _xmppStream = nil;
    
    //....
    
    _msgArchivingStorage = nil;
    [_msgArchiving removeDelegate:self];
    _msgArchiving = nil;
}
```

####读取消息列表

- 修改`XmppOperationManager.m`，初始化查询消息列表的NSFetchedResultsController实例

```
@implementation XmppOperationManager

#pragma mark - Private私有XMPP操作函数

//...

/**
 *  完成消息数据查询（1.读取XMPP服务器 2.写入数据库）
 */
- (NSFetchedResultsController *)msgResultController {
    if (!_msgResultController) {
        
        //1. 获取context
        NSManagedObjectContext *context = _msgArchivingStorage.mainThreadManagedObjectContext;
        
        //2. 设置请求对象
        NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"XMPPMessageArchiving_Message_CoreDataObject"];
        
        //3. 设置查询条件，登录 + 指定JID账号，的消息
        NSPredicate *pre = [NSPredicate predicateWithFormat:@"streamBareJidStr = %@ AND bareJidStr = %@",@"当前登录的XMPP账号"];
        request.predicate = pre;
        
        //4. 查询结果集按照时间升序
        NSSortDescriptor *timeSort = [NSSortDescriptor sortDescriptorWithKey:@"timestamp" ascending:YES];
        request.sortDescriptors = @[timeSort];
        
        //5. 创建
        _msgResultController = [[NSFetchedResultsController alloc] initWithFetchRequest:request managedObjectContext:context sectionNameKeyPath:nil cacheName:nil];
        
        NSError *err = nil;
        
        //6. 设置代理
        _msgResultController.delegate = self;
        
        //7. 执行Fetch操作
        [_msgResultController performFetch:&err];
    }
    
    return _msgResultController;
}

@end
```

- 修改`XmppOperationManager.h`，暴露查询好友的入口

```
#import <Foundation/Foundation.h>
#import "XMPPMessageArchiving_Message_CoreDataObject.h"

@interface XmppOperationManager : NSObject

/**
 *  获取指定登录用户的消息
 */
- (NSArray *)xmpp_messages:(NSString *)account;

@end
```

- 修改`XmppOperationManager.m`，实现入口函数

```
@implementation XmppOperationManager

- (NSArray *)xmpp_messages:(NSString *)account {

	//直接返回NSFetchResultController的执行结果数组
    return [self.msgResultController fetchedObjects];
}

@end
```

####发送一个文本消息

```
@interface XmppOperationManager () <XMPPStreamDelegate,XMPPMessageArchivingStorage>

@property (nonatomic, strong) XMPPStream *xmppStream;

@end

@implementation XmppOperationManager

- (void)xmpp_sendMessage {
    
    //1. 构造发送目标XMPPJID
    XMPPJID *desJID = nil;
    
    //2. 构造消息
    XMPPMessage *msg = [XMPPMessage messageWithType:@"chat" to:desJID];
    
    //3. 设置消息体
    [msg addBody:@"你好啊!"];
    
    //4. 发送消息
    [_xmppStream sendElement:msg];
    
}

@end
```

***

###功能点、发送文件 -- 图片

####方法一、将图片文件转换成`字符串`之后传输

```
- (void)xmpp_sendFileMessage:(NSString *)base64Str {
    
    //1. 构造发送目标XMPPJID
    XMPPJID *desJID = nil;
    
    //2. 构造消息
    XMPPMessage *msg = [XMPPMessage messageWithType:@"chat" to:desJID];
    
    //3. 设置消息属性区别当前消息是发送的哪一类文件
    [msg addAttributeWithName:@"messageType" stringValue:@"image"];//图片文件
//    [msg addAttributeWithName:@"messageType" stringValue:@"pdf"];//pdf
//    [msg addAttributeWithName:@"messageType" stringValue:@"excel"];//excel
//    [msg addAttributeWithName:@"messageType" stringValue:@"word"];//word
//    [msg addAttributeWithName:@"messageType" stringValue:@"txt"];//txt
    
    //4. 设置消息体（一定要设置）
    [msg addBody:@"你好啊!"];
    
    //5. 发送消息
    [_xmppStream sendElement:msg];
}
```

- 方式1: 客户端直接将转换后的字符串发送给Openfire服务器.
	- 缺点1) 如果文件过大，那么转换出来的字符串也是很大的，那么传输的XML流量也是很大的
	- 缺点2) Openfire承受的压力比较大

![](http://i12.tietuku.com/4f39e74b6c75fcde.png)


- 方式2: 客户端先将文件单独上传到另一个`文件服务器`，然后将得到的`文件路径`作为消息值发送给Openfire服务器.

![](http://i12.tietuku.com/87f7790dd355066d.png)

****

