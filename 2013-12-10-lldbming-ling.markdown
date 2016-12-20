---
layout: post
title: "lldb命令"
date: 2013-12-10 12:09:33 +0800
comments: true
categories: 
---

###常用断点调试命令

```
c : 过掉当前断点
```

```
n: 逐行执行代码
```

```
s: 进入当前行要调用的方法
```

```
finish: 跳出结束执行执行当前方法
```

```
continue: 结束当前次循环
```

```
po: 输出对象的值
```

```
frame variable（fr v）: 输出当前块的所有变量的信息
```

```
fr v -r 单独的变量
```

###符号断点

![](http://i5.tietuku.com/d79ed6e3ff1fa9e1.png)

###expr进行值修改或方法调用

```
expr (方法返回值类型)[Target SEL]
```

```
expr (void)[_view3 x_debug]
```

```
- (void)x_debug {	
	//..
}
```

###call，功能与expr类似

```
call (方法返回值类型)[Target SEL]
```

```
expr (void)[_view3 x_debug]
```

```
- (void)x_debug {	
	//..
}
```

###image寻找地址对应的代码

```
image lookup --address 0x0000000100004af8（地址）
```


###bt打印当前线程信息

```
bt
```

###打印当前进程所有的线程

```
thread list

输出的信息中，星号(*)表示thread #1为当前线程
```

```
获取线程的跟踪栈，一些变量值、调用的当前方法等数据

thread backtrace
```


###监测变量值变化

距离监测名为`global`的变量的写操作，并在`(global==5)`为`真`时停止监测

```
watch set var global
```

```
watch modify -c '(global==5)'
```

```
watch list
```

###启动程序

```
process launch
```

```
run 
```

```
r
```


###使用进程ID或进程名，将调试器挂在到这个进程的App程序

举例用于连接Sketch程序(假定其进程ID为123)

```
process attach --pid 123
```

```
process attach --name Sketch
```

```
process attach --name Sketch --waitfor
```

****

###list命令查看代码

```
list
```

```
list 文件名
```

```
list 行号
```

***

###下断点

```
C函数

breakpoint set --name main
```

```
C++类方法

breakpoint set --method foo
```

```
Objective-C选择器

kpoint set --selector alignLeftEdges:
```

```
根据某个函数调用语句下断点(Objective-C比较有用)

breakpoint set -n "-[SKTGraphicView alignLeftEdges:]"
```

***

###查看多线程的信息

```
输出当前断点处，所有的线程

thread backtrace all
```

![](http://i4.tietuku.com/12634d10c35b7614.png)

****

###从上面的输出结果可以得到:

- 使用 `* thread`开头，表示当前代码执行在`第几号线程`上
	- `* thread #3`，说明当前所在线程是3号

- 使用 `* frame`开头，表示当前代码执行在`第几号栈帧`上
	- `* frame #0`，说明处于0号栈帧上

****

###thread return 当不想让代码执行某个方法，或者要直接返回一个想要的值

比如断点到如下方法时

```
- (BOOL)isMan {
	.....
}
```

断点后，lldb执行`thread return NO`，那么就会结束方法执行，并将NO作为方法的返回值

****

###切换到另一个线程上进行调试

```
thread select 线程id
```

```
eg. thread select 1
```

![](http://i4.tietuku.com/3e39706cd5881d8f.png)

****

###watchpoint set观察某个变量值的改变

- 首先断点到要观察的变量的代码前一行
- 然后使用如下lldb命令观察变量值改变

```
watchpoint set variable 变量
```

eg、

```
(lldb) watchpoint set variable self->_string
Watchpoint created: Watchpoint 1: addr = 0x7fcf3959c418 size = 8 state = enabled type = w
    watchpoint spec = 'self->_string'
    new value: 0x0000000000000000
```

需要注意的是，这里不接受`方法`，所以不能使用watchpoint set variable `self.string or [self string]`，因为self.string调用的是string的getter方法，而必须使用`self->string`

****

###有时候在lldb执行expr时会报错，找不到属性或方法，但明明是存在属性和方法

就像如下，输出一个view的frame，但有时候就会报错，找不到属性或方法


```
(lldb) p self.view.frame
error: property 'frame' not found on object of type 'UIView *'
error: 1 errors parsing expression
```

出现这个问题的原因:

```
1. 每次run Xcode，LLDB的东西都会被清空
2. lldb没有导入UIKit，无法得到属性和方法的声明与实现
```

解决就是在执行p self.view.frame时，先导入UIKit或其他依赖的`.h`

```
expr @import UIKit（或其他类库.h、自定义类.h）
```

****

###Facebook开源的UI调试能力增强工具chisel

```
brew update
brew install chisel
```

```
echo  command script import /usr/local/Cellar/chisel/1.4.0/libexec/fblldb.py >> ~/.lldbinit
```

> 注意如上chisel版本1.4.0是我当前下载安装的版本，如果更新版本，也需要重新执行上面的命令修改版本号

chisel主要使用的命令

[![Snip20160319_3.md.png](http://img.ja.9xiqi.cn/2016/03/19/Snip20160319_3.md.png)](http://pic.9xiqi.cn/image/lVO6V)

当某个命令不会使用时，执行

```
help pviews
```

```
help border
```

等等，就会输出这个命令使用到的格式、以及一些例子.