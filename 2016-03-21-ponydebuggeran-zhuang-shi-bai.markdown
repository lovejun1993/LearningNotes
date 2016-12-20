---
layout: post
title: "ponyDebugger安装失败"
date: 2016-03-21 19:48:58 +0800
comments: true
categories: 
---

###按照github上文档安装，一直失败报错如下

```
Could not find a version that satisfies the requirement pybonjour==1.1.1 (from ponyd) (from versions: )
No matching distribution found for pybonjour==1.1.1 (from ponyd)
```

###去对应的issues上搜到了解决，先安装pip

- 安装pip

```
sudo easy_install pip
```

- 然后通过pip安装ponyDebugger

```
pip install -e git+https://github.com/Eichhoernchen/pybonjour.git#egg=pybonjour
```

```
Obtaining pybonjour from git+https://github.com/Eichhoernchen/pybonjour.git#egg=pybonjour
  Cloning https://github.com/Eichhoernchen/pybonjour.git to /Users/xiongzenghui/Library/PonyDebugger/src/pybonjour
Installing collected packages: pybonjour
  Running setup.py develop for pybonjour
Successfully installed pybonjour
```

- 然后再使用文档上的命令

```
curl -s https://cloud.github.com/downloads/square/PonyDebugger/bootstrap-ponyd.py | \
  python - --ponyd-symlink=/usr/local/bin/ponyd ~/Library/PonyDebugger
```

****

###指定监听的地址

```
ponyd serve --listen-interface=127.0.0.1
```

***

###再打开浏览器输入`http://localhost:9000`

能够看到PonyGateway的网页，就说明安装成了.

****

###后来在github的issues中还找到了一种解决

```
git clone git@github.com:square/PonyDebugger.git
cd PonyDebugger
python setup.py install
```