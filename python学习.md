## Python 是一个高层次的结合了解释性、编译性、互动性和面向对象的脚本语言。

Python 的设计具有很强的可读性，相比其他语言经常使用英文关键字，其他语言的一些标点符号，它具有比其他语言更有特色语法结构。
Python 是一种解释型语言： 这意味着开发过程中没有了编译这个环节。类似于PHP和Perl语言。
Python 是交互式语言： 这意味着，您可以在一个Python提示符，直接互动执行写你的程序。
Python 是面向对象语言: 这意味着Python支持面向对象的风格或代码封装在对象的编程技术。
Python 是初学者的语言：Python 对初级程序员而言，是一种伟大的语言，它支持广泛的应用程序开发，从简单的文字处理到 WWW 浏览器再到游戏。

## Python 特点

```
1.易于学习：Python有相对较少的关键字，结构简单，和一个明确定义的语法，学习起来更加简单。
2.易于阅读：Python代码定义的更清晰。
3.易于维护：Python的成功在于它的源代码是相当容易维护的。
4.一个广泛的标准库：Python的最大的优势之一是丰富的库，跨平台的，在UNIX，Windows和Macintosh兼容很好。
5.互动模式：互动模式的支持，您可以从终端输入执行代码并获得结果的语言，互动的测试和调试代码片断。
6.可移植：基于其开放源代码的特性，Python已经被移植（也就是使其工作）到许多平台。
7.可扩展：如果你需要一段运行很快的关键代码，或者是想要编写一些不愿开放的算法，你可以使用C或C++完成那部分程序，然后从你的Python程序中调用。
8.数据库：Python提供所有主要的商业数据库的接口。
9.GUI编程：Python支持GUI可以创建和移植到许多系统调用。
10.可嵌入: 你可以将Python嵌入到C/C++程序，让你的程序的用户获得"脚本化"的能力
```

## Python 中文编码

```
print "你好，世界";
```

运行后会报错

```
/System/Library/Frameworks/Python.framework/Versions/2.7/bin/python2.7 /Users/xiongzenghui/PycharmProjects/PythonDemos/demo1.py
  File "/Users/xiongzenghui/PycharmProjects/PythonDemos/demo1.py", line 1
SyntaxError: Non-ASCII character '\xe4' in file /Users/xiongzenghui/PycharmProjects/PythonDemos/demo1.py on line 1, but no encoding declared; see http://python.org/dev/peps/pep-0263/ for details
```

Python中默认的编码格式是 ASCII 格式，在没修改编码格式时无法正确打印汉字，所以在读取中文时会报错。
解决方法为只要在文件开头加入 `# -*- coding: UTF-8 -*-` 或者 `#coding=utf-8` 就行了。

```
# -*- coding: UTF-8 -*-

print "你好，世界"
```

就OK了。

## Python 引号

```
word = 'word'

sentence = "这是一个句子。"

paragraph = """这是一个段落。
            line1
            line2
            line3"""
```

三引号可以由多行组成，编写多行文本的快捷语法，常用语文档字符串，在文件的特定地点，被当做注释。

## Python注释

单行注释

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-
# 文件名：test.py

# 第一个注释
print "Hello, Python!";  # 第二个注释
```

多行注释使用三个单引号(''')或三个双引号(""")

```
'''
这是多行注释，使用单引号。
这是多行注释，使用单引号。
这是多行注释，使用单引号。
'''

"""
这是多行注释，使用双引号。
这是多行注释，使用双引号。
这是多行注释，使用双引号。
"""
```

## 等待用户输入

```
raw_input("\n\nPress the enter key to exit.")
```

## 行和缩进

Python的代码块不使用大括号（{}）来控制类，函数以及其他逻辑判断。python最具特色的就是用缩进来写模块。

所有代码块语句必须包含相同的缩进空白数量，这个必须严格执行，否则会编译报错。

正确写法：

```
if True:
    print "True"
else:
  print "False"
```

错误写法

```
if True:
    print "Answer"
    print "True"
else:
    print "Answer"
  print "False"	# 没有严格缩进，在执行时会报错
```

## 多个语句构成代码组

第一行以冒号( : )结束，该行之后的一行或多行代码构成代码组

```
if expression : 
   suite 
elif expression :  
   suite  
else :  
   suite 
```

同样每一个代码块，都需要行号缩进一致。

## Python 命令行参数

Python 中也可以所用 `sys` 的 `sys.argv` 来获取命令行参数：

- (1) sys.argv 是命令行参数列表。
- (2) len(sys.argv) 是命令行参数个数。
- (3) `sys.argv[0]` 表示脚本名。

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-

import sys

print '参数个数为:', len(sys.argv), '个参数。'
print '参数列表:', str(sys.argv)
```

执行如下

```
python demo1.py arg1 arg2 arg3
```

输出如下

```
参数个数为: 4 个参数。
参数列表: ['demo1.py', 'arg1', 'arg2', 'arg3']
```

### getopt模块

getopt模块是专门处理命令行参数的模块，用于获取命令行选项和参数，也就是sys.argv。

命令行选项使得程序的参数更加灵活。支持短选项模式（`-`）和长选项模式（`--`）。

getopt.getopt 方法用于解析命令行参数列表，语法格式如下：

```
getopt.getopt(args, options[, long_options])
```

方法参数说明：

```
args: 要解析的命令行参数列表。一般是sys.argv[1:]，0为脚本本身的名字。

options: 以字符串的格式定义，options后的冒号(:)表示该选项必须有附加的参数，不带冒号表示该选项不附加参数。shortopts 短格式（“-”）。

long_options: 以列表的格式定义，long_options 后的等号(=)表示如果设置该选项，必须有附加的参数，否则就不附加参数。
该方法返回值由两个元素组成: 第一个是 (option, value) 元组的列表。 第二个是参数列表，包含那些没有'-'或'--'的参数。longopts 长格式（“--”）
```

python脚本如下

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-

import sys

print str(sys.argv)
```

执行如下

```
python demo1.py -c -b 5 --mm --lala /home
```

在上面的命令中`sys.argv`中存的是


```
['demo1.py', '-c', '-b', '5', '--mm', '--lala', '/home']
```

```
'-c'代表不需要附加参数的短选项；
'-b'代表需要附加参数的短选项，同时5就是其附加参数
'--mm'代表不需要附加参数的长选项
'--lala'代表需要附加参数的长选项，同时/home就是其附加参数
```

getopt函数的格式是:

```
getopt.getopt(命令行参数，"短选项",[长选项]）
```

```
其中如果短选项需要附加参数则在短选项后面加 >>> ：
如果长选项需要添加参数则在长选项后添加>>> =
```

获取上面的参数的python代码如下

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-

import sys
import  getopt

print str(sys.argv)

# c、b两个短命令行参数，且b需要带值
# mm、lala两个长命令行参数，且lala需要带值
opts, args = getopt.getopt(sys.argv[1:],'cb:',['mm','lala='])

print opts
print args
```

执行如下

```
python demo1.py -c -b 5 --mm --lala /home
```

输出如下

```
['demo1.py', '-c', '-b', '5', '--mm', '--lala', '/home']
[('-c', ''), ('-b', '5'), ('--mm', ''), ('--lala', '/home')]
[]
```

## Python 变量类型

### 自动类型推断

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-

counter = 100 # 赋值整型变量
miles = 1000.0 # 浮点型
name = "John" # 字符串

print counter
print miles
print name
```

### Python有五个标准的数据类型

```
Numbers（数字）
String（字符串）
List（列表）
Tuple（元组）
Dictionary（字典）
```

### Python数字

```
var1 = 1
var2 = 10
```

您也可以使用del语句删除一些对象的引用

```
del var1
```

Python支持四种不同的数字类型

```
int（有符号整型）
long（长整型[也可以代表八进制和十六进制]）
float（浮点型）
complex（复数）
```

### Python字符串

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-

str = 'Hello World!'
print str  # 输出完整字符串
print str[0]  # 输出字符串中的第一个字符
print str[2:5]  # 输出字符串中第三个至第五个之间的字符串
print str[2:]  # 输出从第三个字符开始的字符串
print str * 2  # 输出字符串两次
print str + "TEST"  # 输出连接的字符串
```

输出如下

```
Hello World!
H
llo
llo World!
Hello World!Hello World!
Hello World!TEST
```

字符串格式化

```
print "My name is %s and weight is %d kg!" % ('Zara', 21) 
```

### Python列表

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-

list = ['runoob', 786, 2.23, 'john', 70.2]
tinylist = [123, 'john']

print list  # 输出完整列表
print list[0]  # 输出列表的第一个元素
print list[1:3]  # 输出第二个至第三个的元素
print list[2:]  # 输出从第三个开始至列表末尾的所有元素
print tinylist * 2  # 输出列表两次
print list + tinylist  # 打印组合的列表
```

输出如下

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-

['runoob', 786, 2.23, 'john', 70.2]
runoob
[786, 2.23]
[2.23, 'john', 70.2]
[123, 'john', 123, 'john']
['runoob', 786, 2.23, 'john', 70.2, 123, 'john']
```

list还有成员运算（int 、not in）

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-

a = 10
b = 20
list = [1, 2, 3, 4, 5 ];

if ( a in list ):
   print "1 - 变量 a 在给定的列表中 list 中"
else:
   print "1 - 变量 a 不在给定的列表中 list 中"

if ( b not in list ):
   print "2 - 变量 b 不在给定的列表中 list 中"
else:
   print "2 - 变量 b 在给定的列表中 list 中"

# 修改变量 a 的值
a = 2
if ( a in list ):
   print "3 - 变量 a 在给定的列表中 list 中"
else:
   print "3 - 变量 a 不在给定的列表中 list 中"
```

### Python元组

```
tuple = ('runoob', 786, 2.23, 'john', 70.2)
tinytuple = (123, 'john')

print tuple  # 输出完整元组
print tuple[0]  # 输出元组的第一个元素
print tuple[1:3]  # 输出第二个至第三个的元素
print tuple[2:]  # 输出从第三个开始至列表末尾的所有元素
print tinytuple * 2  # 输出元组两次
print tuple + tinytuple  # 打印组合的元组
```

输出如下

```
('runoob', 786, 2.23, 'john', 70.2)
runoob
(786, 2.23)
(2.23, 'john', 70.2)
(123, 'john', 123, 'john')
('runoob', 786, 2.23, 'john', 70.2, 123, 'john')
```

> 元组是不允许更新。

### Python 字典

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-

dict = {}
dict['one'] = "This is one"
dict[2] = "This is two"
 
tinydict = {'name': 'john','code':6734, 'dept': 'sales'}
  
print dict['one']          # 输出键为'one' 的值
print dict[2]              # 输出键为 2 的值
print tinydict             # 输出完整的字典
print tinydict.keys()      # 输出所有键
print tinydict.values()    # 输出所有值
```

## Python身份运算符

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-

a = 20
b = 20

if ( a is b ):
   print "1 - a 和 b 有相同的标识"
else:
   print "1 - a 和 b 没有相同的标识"

if ( id(a) is not id(b) ):
   print "2 - a 和 b 有相同的标识"
else:
   print "2 - a 和 b 没有相同的标识"

# 修改变量 b 的值
b = 30
if ( a is b ):
   print "3 - a 和 b 有相同的标识"
else:
   print "3 - a 和 b 没有相同的标识"

if ( a is not b ):
   print "4 - a 和 b 没有相同的标识"
else:
   print "4 - a 和 b 有相同的标识"
```

## Python 函数

### 定义一个函数的规则

```
1. 函数代码块以 def 关键词开头，后接函数标识符名称和圆括号()。
2. 任何传入参数和自变量必须放在圆括号中间。圆括号之间可以用于定义参数。
3. 函数的第一行语句可以选择性地使用文档字符串—用于存放函数说明。
4. 函数内容以冒号起始，并且缩进。
5. return [表达式] 结束函数，选择性地返回一个值给调用方。不带表达式的return相当于返回 None。
```

demo

```
def functionname( parameters ):
   "函数_文档字符串"
   function_suite
   return [expression]
```

比如

```
def printme( str ):
   "打印传入的字符串到标准显示设备上"
   print str
   return
```

### 函数调用

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
# 定义函数
def printme( str ):
   "打印任何传入的字符串"
   print str;
   return;
 
# 调用函数
printme("我要调用用户自定义函数!");
printme("再次调用同一函数");
```

### 可更改(mutable)与不可更改(immutable)对象

在 python 中，strings, tuples, 和 numbers 是不可更改的对象，而 list,dict 等则是可以修改的对象:


- 不可变类型：

变量赋值 a=5 后再赋值 a=10，这里实际是新生成一个 int 值对象 10，再让 a 指向它，而 5 被`丢弃`，不是改变a的值，相当于新生成了a。

- 可变类型：

变量赋值 la=[1,2,3,4] 后再赋值 la[2]=5，则是将 list la 的第二个元素值更改，本身la没有动，只是其内部的一部分值被修改了。


python 函数的参数传递：

- 不可变类型：

类似 c++ 的值传递，如 整数、字符串、元组。如fun（a），传递的只是a的值，没有影响a对象本身。比如在 fun（a）内部修改 a 的值，只是修改另一个复制的对象，不会影响 a 本身。

- 可变类型：

类似 c++ 的引用传递，如 列表，字典。如 fun（la），则是将 la 真正的传过去，修改后fun外部的la也会受影响。

### 调用函数时可使用的正式参数类型：

```
必备参数
关键字参数
默认参数
不定长参数
```

- 必备参数，就是必须要传递的...

- 关键字参数

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
#可写函数说明
def printinfo( name, age ):
   "打印任何传入的字符串"
   print "Name: ", name;
   print "Age ", age;
   return;
 
#调用printinfo函数，关键字参数顺序不重要，用参数名匹配参数值
printinfo( age=50, name="miki" );
```

- 缺省参数

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
#可写函数说明
def printinfo( name, age = 35 ):
   "打印任何传入的字符串"
   print "Name: ", name;
   print "Age ", age;
   return;
 
#调用printinfo函数
printinfo( age=50, name="miki" );
printinfo( name="miki" );
```

- 不定长参数

语法格式

```
def functionname([formal_args,] *var_args_tuple ):
   "函数_文档字符串"
   function_suite
   return [expression]
```

例子

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
# 可写函数说明
def printinfo( arg1, *vartuple ):
   "打印任何传入的参数"
   print "输出: "
   print arg1
   for var in vartuple:
      print var
   return;
 
# 调用printinfo 函数
printinfo( 10 );
printinfo( 70, 60, 50 );
```

### 匿名函数 lambda

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
# 可写函数说明
sum = lambda arg1, arg2: arg1 + arg2;
 
# 调用sum函数
print "相加后的值为 : ", sum( 10, 20 )
print "相加后的值为 : ", sum( 20, 20 )
```

## Python 文件I/O

http://www.runoob.com/python/python-files-io.html

## 面向对象

### 创建类

```
# !/usr/bin/python
# -*- coding: UTF-8 -*-

class Employee:
    '所有员工的基类'

    # 类变量，它的值将在这个类的所有实例之间共享
    empCount = 0

    # 构造函数
    def __init__(self, name, salary):
        self.name = name
        self.salary = salary
        Employee.empCount += 1

    # 功能函数1
    def displayCount(self):
        print "Total Employee %d" % Employee.empCount

    # 功能函数2
    def displayEmployee(self):
        print "Name : ", self.name, ", Salary: ", self.salary
```

### 创建实例对象

```
# 创建 Employee 类的第一个对象
emp1 = Employee("Zara", 2000)

# 创建 Employee 类的第二个对象
emp2 = Employee("Manni", 5000)
```

### 访问属性

```
emp1.displayEmployee()
emp2.displayEmployee()
print "Total Employee %d" % Employee.empCount
```

你可以添加，删除，修改类的属性，如下所示：

```
emp1.age = 7  # 添加一个 'age' 属性
emp1.age = 8  # 修改 'age' 属性
del emp1.age  # 删除 'age' 属性
```

## python对象销毁(垃圾回收)

类似苹果的引用计数器

```
a = 40      # 创建对象  <40>
b = a       # 增加引用， <40> 的计数
c = [b]     # 增加引用.  <40> 的计数

del a       # 减少引用 <40> 的计数
b = 100     # 减少引用 <40> 的计数
c[0] = -1   # 减少引用 <40> 的计数
```

同样存在循环引用的问题。

## 类的继承

```
class A:        # 定义类 A
.....

class B:         # 定义类 B
.....

class C(A, B):   # 继承类 A 和 B
.....
```

你可以使用issubclass()或者isinstance()方法来检测。

```
issubclass() - 布尔函数判断一个类是另一个类的子类或者子孙类，语法：issubclass(sub,sup)
isinstance(obj, Class) 布尔函数如果obj是Class类的实例对象或者是一个Class子类的实例对象则返回true。
```

## Python执行shell



安装pip

```
sudo easy_install pip 
```

在使用pip安装sh

```
https://github.com/amoffat/sh
```

```
sudo pip install sh
```

clone sh

```
xiongzenghuis-MacBook-Pro:Desktop xiongzenghui$ cd ~/Desktop/
xiongzenghuis-MacBook-Pro:Desktop xiongzenghui$ git clone https://github.com/amoffat/sh.git
```

进入clone好的sh目录，然后执行如下install

```
sudo pip install -r requirements-dev.txt
```

执行测试

```
python sh.py test
```


## xcode编译程序时调用python脚本  

<img src="./python1.png" alt="" title="" width="700"/>

### which python来查看python的安装位置

```
which python
```

