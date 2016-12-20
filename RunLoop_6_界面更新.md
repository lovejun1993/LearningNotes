##界面更新机制 就是使用了 CFRunLoopObserver 来实现的
	
- 在`主线程的RunLoop`添加了二个 `Observer 观察者`

	- 一个`最高`优先级Observer
		- 监听`RunLoop Entry` 状态，runloop即将进入
		- 回调主要做的事情
			- 创建主线程的自动释放池

	- 一个`最低` 优先级Observer
		- 监听`RunLoop  BeforeWaiting`状态，runloop将睡
			- 首先销毁旧的自动释放池
			- 再创建新的自动释放池
		- 监听`RunLoop Exit 线程即将退出`状态，runloop将退出
			- 销毁自动释放池


有时间仔细看看....