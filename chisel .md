## chisel 


git clone chisel所有文件

```
.....
```

编辑配置环境变量的文件

```
vim ~/.lldbinit
```

在文件中写如下

```
command script import command script import /Users/xiongzenghui/Desktop/sources/chisel/fblldb.py
```

路径是clone chisel的目录。

> 然后在xcode运行时，断点后再lldb命令行调试中就可以输出一些chisel调试UI的命令了。


chisel的lldb调试命令：

- (1) pviews 打印某个view的所有层级结构、frame、layer等信息

```
pviews self.view
```

- (2) show/hide 显示/隐藏某个view

```
show _view1
```

```
hide _view2
```

- (3) border/unborder 边框

```
border -c 'blue' -w 2 _view1
unborder _view1
```

- (4) pinternals 打印view的所有信息

```
pinternals _view1
```

- (5) pclass 打印view的继承结构

```
pclass _view1
```

- (6) pvc 打印当前的控制器层级，如下图，我定义了一个UINavigationController，ViewController作为它的根控制器

```
pvc
```

- (7) pcells 打印层级最高的tableview当前可见的所有cell

```
pcells
```

- (8) pdata 解码打印一个NSData对象

```
pdata dataDemo_
```

- (9) pdocspath 打印应用程序的Documents目录路径

```
pdocspath -o 参数-o直接打开目录
```

- (10) pinternals 打印对象的成员变

```
pinternals self
```

- (11) pivar 

```
pivar 对象 某一个Ivar
```

- (12) presponder 打印一个继承于presponder的控件的响应链

