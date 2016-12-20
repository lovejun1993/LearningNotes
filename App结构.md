
- 公共基类
	- BaseViewController
	- BaseTableViewCell
	- BaseNavigationController
	- BaseViewModel >>> 统一处理网络请求错误、业务处理错误

- 数据缓存
	- 内存缓存
		- LRU 循环双向链表，淘汰最久未被访问的缓存项、超时项
	- 磁盘文件缓存
		- 对象的文件归档
		- 对象的sqlite表数据存储
		- 需要考虑版本升级时数据迁移的问题

- 网络库
	- Request
		- 作为实体类
			- request url
			- request method
			- request arguments
			- request completion block/delegate
		- 作为工具类
			- 提供start方法，开始网络请求
	- ChainRequest
		- req1->_next = req2;
	- BatchReuqest
		- Array of Request
	- Agent单例
		- 使用单例缓存dic临时缓存所有的Request对象
			- <request.url : Request对象>
		- 内部耦合AFNetworking完成具体的网络请求操作
		- 需要在Request执行完毕时，释放Request.block = nil; 避免循环引用

- 常用分类
	- NSArray/NSDictionary/NSSet
		- 安全读写item value
		- item value的数据类型转换
	- NSString
	- NSFileManager

- 对象之间的通信方式
	- 1 : 1
		- block 
		- delegate
	- 1 : n
		- kvo
		- NSNotificationCenter（同步） or NSNotificationQueue（异步）
		- 自己实现一个基于Ptotocol协议的`观察者List`模式

- 业务模块的结构
	- ViewController
	- Views
	- Request
	- Response
	- Models
	- 业务代码
		- 业务协议
		- 业务实现
		- Aspect绑定协议与实现类
	- 使用ReactiveCocoa这样提供KVO功能开源框架，让ViewModel与UI层、其他Service层、DAO层之间产生联动处理

	
![](http://b221.photo.store.qq.com/psb?/V11ePBui3l2qGa/kGom2RchRLBkTtAUTqRoB3K9*1xHj*3bwOYe.B8jvmc!/c/dN0AAAAAAAAA&bo=oQDLAKEAywAFACM!)
	
##向外暴露类簇类，来屏蔽内部多种版本的具体实现

```
- Tool
	- Tool_iOS7
	- Tool_iOS8
```

那么最理想的情况，指向外提供Tool这个类，而对于`Tool_iOS7`和`Tool_iOS8`这两个类向外屏蔽，由Tool这个类内部选择返回一个`Tool_iOS7或Tool_iOS8`的对象。

对一些容易废弃的api抽象类簇类（open url schme，推送）不同ios系统版本适配

##关于层与层之间、业务模块之间进行解耦

- 最直接想到的就是:
	- 只暴露`接口`，封装具体实现
	- 使用`类簇`
	- `工厂`

- 但更好的方式是，做一个`模块管理器`
	- 调用模块，找到模块管理器
	- 传给模块管理器一个标识，模块管理器找到对应的模块
	- 然后再传给模块管理器要执行的请求参数，模块管理通知对应的模块进行处理
	- 模块管理器将对应模块处理完的结果值，最后回调给调用的模块
	
个人想法，暂未实现，灵感来自`URL Scheme`来完成互不相干的两个App之间的交互、数据传输、业务调用，通过iOS系统作为连接根据`URL Scheme`找对对应的App完成调用。

##http session 登录态管理组件

有关登录态的代码，必须封装成一套独立的代码来管理。提供setter、getter来设置或读取登陆态，内部完成加解密。

##热修复原生objc代码

jspatch，基于runtime，交换`objc_method`中SEL指向的IMP。

##内存泄露监控