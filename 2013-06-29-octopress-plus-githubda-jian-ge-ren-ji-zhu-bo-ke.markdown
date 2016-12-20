---
layout: post
title: "octopress+github搭建个人技术博客"
date: 2013-06-29 21:48:00 +0800
comments: true
categories: 
---

###搭建博客


首先是克隆octopress的所有代码

```
git clone git://github.com/imathis/octopress.git octopress
```

然后进入目录下执行如下命令


```
cd octopress
sudo gem install bundler
bundle install
```

登录Github，假设你的用户名是username，首先要新建一个命名为 `username.github.com` 的Repo，命名必须是这个格式，如果不这样命名的话，在运行命令 `rake setup_github_pages` 之后不能够自动创建后面提到的master和source 分支，而是作为普通仓库生成 gh-pages 分支。

```
rake setup_github_pages

输入刚才创建的代码仓库的ssh地址
```

安装默认的主题

```
rake install
rake generate
rake preview

将博客发布到Github上
rake deploy
```

别忘了把所有源文件发布到 source 分支下面：

```
git add .
git commit -m "your message"
git push origin source
``` 

生成本机的ssh秘钥，并在github上设置

```
cd ~/.ssh
ssh-keygen -t rsa -C 你注册github时的email
```

弹出Enter file in which to save the key (/Users/twer/.ssh/id_rsa):直接按空格

弹出Enter passphrase (empty for no passphrase):输入你github账号的密码。Enter same passphrase again:再次输入你的密码。

打开`~/.ssh`下的id_rsa.pub文件复制里面的全部内容。
登陆github，选择Account Settings-->SSH Public Keys 添加ssh，把剪切板的内容复制到key输入框内直接保存。

测试shh:

```
ssh git@github.com
```

写博客

```
rake generate
git add .
git commit -am "Some comment here." 
git push origin source
rake deploy
```

***

###其他电脑上使用

> 一、Octopress目录结构

Octopress的仓库目录下有两个branch， source 和 master 。

**source** 分支下保存Octopress的源代码，我们需要用他们生成博客，该分支保存在Octopress本地仓库的根目录下；

**master** 分支下保存生成的博客内容，该分支在Octopress本地仓库的根目录下一个叫 `_deploy` 得文件夹中。该文件夹是以下划线开头的，会在执行 `git push origin source` 命令时被忽略，这也是为什么一个目录中能同时存在两个不同分支的文件夹的原因。

> 二、在本地重建Octopress仓库

####第一步: clone source 分支

```
git clone -b source git@github.com:username/username.github.com.git octopress
```

注意:
如上的 username 替换成你自己github用户名。 另外还要注意的是，clone的地址不能是 http 而必须得是 ssh 的。

如果执行时提示以下错误：

```
Cloning into 'octopress'...
The authenticity of host 'github.com (192.30.252.131)' can't be established.
RSA key fingerprint is 16:27:ac:a5:76:28:2d:36:63:1b:56:4d:eb:df:a6:48.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'github.com,192.30.252.131' (RSA) to the list of known hosts.
Permission denied (publickey).
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

说明你的电脑不被github信任，需要在你电脑上创建 ssh key 并添加到github中。

解决:

1. 生成key

```
ssh-keygen -t rsa -C "github账号申请的email"
```
如上提示，全部回车即可.

2. 把生成的key添加到 ssh-agent 中

```
ssh-add ~/.ssh/id_rsa
```

3. copy key内容

```
pbcopy < ~/.ssh/id_rsa.pub
```

4. 把生成的key添加到github

打开github登录， 选择Account Settings，选择 SSH KEYS ，点击 Add SSH key 按钮。

5. 验证key可用性

```
ssh -T git@github.com
```

####第二步: clone master分支

```
cd octopress
```

```
git clone git@github.com:username/username.github.com.git _deploy
```

####第三步: 配置环境

```
sudo gem install bundler
```

```
bundle install
```

```
rake setup_github_pages 
```

然后输入github中博客仓库地址。

***

###防止冲突

```
cd octopress
```

```
git pull origin source  
```

```
cd ./_deploy
```

```
git pull origin master 
```
