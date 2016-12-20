---
layout: post
title: "git使用遇到的问题"
date: 2015-08-20 18:51:28 +0800
comments: true
categories: 
---

###git log 查看前所有的提交

```
git log 
```

```
git log --oneline
```

图标形式输出所有分支的提交

```
git log --oneline --decorate --graph --all
```

查看最近2次的提交


```
git log -2
```

查看file1提交记录

```
git log file1
```

查找关键key相关的提交

```
git log --S关键key
```

git log 增强

```
git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"

在终端输入git lg，就能看到下面漂亮的git log了

想看到git log的变化的行数，输入git lg -p 

```

***

###git show 

显示具体的代码改动情况

```
git show
```

显示指定 commit 修改的内容

```
git show 某一个commitId
```

统计并一行输出某一个commit的修改的内容


```
git show --stat --oneline commitId
```

***


###git revert的使用

会回退到指定Id的代码（本地和远程都会删掉之前的代码）

```
git revert c011eb3c20ba6fb38cc94fe5a8dda366a3990c61
```

将当前回退的代码作为一个【新】的commitId提交

```
git push origin 分支名

```

***

###git diff的使用

* 查看暂存区与最后提交的差异

```
git diff HEAD
```

* 统计一下有哪些文件被改动

```
git diff --stat
```

* 查看当前目录和另外一个分支的差别

```
git diff 其他分支的名字
```

* 比较两个历史版本之间的差异

```
git diff SHA1 SHA2
```

```
git diff SHA1 SHA2 --oneline
```

第一部分: 显示修改的文件.

如:

```
--- a/basic.lua
+++ b/basic.lua
```

"---" ， 表示变动前的文件.
"+++" ， 表示变动后的文件.

第二部分: 修改文件中变动的位置.

如: `@@ -1,7 +1,7 @@`

1. 文件变动的位置使用 `@@`起始，并以`@@`结束.
2. `-1`中的负号，表示修改前的文件
3. `+1`中的正号，表示修改后的文件
4. `-1,7`表示修改之【前】的文件中，第1行到第7行
5. `+1,7`表示修改之【后】的文件中，第1行到第7行
6. `-1,7 +1,7`表示修改之【前】文件的第一行到第7行，变成了，修改之【后】文件的第1行到第7行.




***

###git blame使用


* 查看某个文件的每一部分代码，分别是谁提交的

```
git blame ./basic.lua
```


* 定位某个文件第1行到第3行，是谁修改提交的

```
git blame ./basic.lua -L 1,3
```

***

###git bisect查找问题引入的版本

第一步、 找到【正常】的commit和【出现问题】的commit，从前往后的顺序.

示例如下：

```
git log --oneline


adc1783 第十次提交		当前最新提交是有问题的提交
744f942 第九次提交
b789ced 第八次提交
b7c093c 第七次提交
24f0941 第六次提交
e871681 第五次提交
41b15b8 第四次提交
66ab858 第三次提交
5cd7090 第二次提交
ee36a22 第一次提交    	很久之前没有问题的提交
```

OK，这一步就是要找到

1. 【最早之前】，正常的commit.
2. 当前出现问题的，【最后】的commit.

第二步、`开始git bisect` 查找问题版本.

```
git bisect start
```

第三步、`标记当前出现问题的commit`.

```
git bisect bad
```

第三步、`标记最早正常的commit`.

假设是第一次提交，commitId是ee36a22.

```
git bisect good ee36a22
```

第四步、如上三个步骤之后，`git会自动将HEAD指向bad与good中间的commit`.

```
命令行输出如下:

Bisecting: 4 revisions left to test after this (roughly 2 steps)
[e8716810840d33cfc540bc6599a344c827d6e16e] 第五次提交
```

第五步、`测试当前HEAD指向commit的代码是否是正常的`.

```
1. 如果正常，标记为good.

git bisect good
```

```
2. 如果不正常，标记为bad.

git bisect bad
```

第六步、`HEAD继续移动到当前的bad与good中间的commit`.（查找过程类似二分法）

继续操作第五步，检查当前HEAD指向的commit是否是正常.

第七步、`最终定位到出现问题的commit`.

大概如下:

```
41b15b83a6f9592a751e5b181419709fb76b7d61 is the first bad commit
commit 41b15b83a6f9592a751e5b181419709fb76b7d61
Author: xiongzenghui <xiongzenghui@myhome.163.com>
Date:   Mon Sep 21 21:36:04 2015 +0800

    第四次提交（此次提交的描述信息...）

:100644 100644 3790ef4f06d98c7000bb04563d1ef80e05301383 a0f420a2bc00e5d2cab3e20eb0f65c47afb54972 M	demo.lua

```

第八步、使用`git diff`找出这次提交做出的变化，就是产生问题的代码.

```
git diff 3790ef a0f420
```

```
命令行输出如下:

diff --git a/3790ef4 b/a0f420
index 3790ef4..a0f420a 100644
--- a/3790ef4
+++ b/a0f420
@@ -1,4 +1,5 @@
 
 1
 2
-3
\ No newline at end of file
+3
+4
\ No newline at end of file
```

即可找到出现问题: 删除3，又添加了3，然后添加4.

注意:（找到了问题所在后，不要在当前HEAD指向的commit做修改，可以单独检出当前commit的一个分支进行修改，然后合并回主分支.）

第九步、让HEAD恢复指向最新commit，结束git bisect查找问题commit操作.

```
git bisect reset
```

***

###`git grep`查询指定内容的位置

查询指定内容所在行号

```
git grep -n '要查找的内容'
```

带单个正则表达式的查找

```
git grep -e 正则表达式

git grep -e 'zhangsan'
```

多个正则表达式

```
git grep -e 正则式1 --or -2 正则式2

git grep -e 'zhangsan' --or -e 'lisi'
```

```
git grep -e 正则式1 --and -2 正则式2


git grep -e 正则式1 --and \( -e 正则式2 --or --not -e 正则式3 \)
```