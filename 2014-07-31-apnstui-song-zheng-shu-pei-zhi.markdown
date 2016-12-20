---
layout: post
title: "APNS推送证书配置"
date: 2014-07-31 22:13:18 +0800
comments: true
categories: 
---



###好久以前弄过APNS推送证书的配置，忘记了...今天又要搞，弄了好久才搞定，做个笔记.

***

主要关键:

1. Xcode工程的Bundle Id
2. Apple开发中中心配置的App Id
3. 推送证书对应的App Id

这三个Id必须一致.

***

下面给出操作步骤截图:

###No1. 进入Applde开发中中心，选择Xcode工程配置的Bundle Id一致的App Id.

![截图](http://i1.tietuku.com/9fb84b551ba15dc8.png)

###No2. 启用选中App Id证书中的Push Notification.

![截图](http://i1.tietuku.com/8904c229e6512d57.png)

![截图](http://i1.tietuku.com/fcae521e500c2579.png)

###No3. 分别创建用于【调试的开发推送证书】和用于【产品发布的推送证书】，二者创建过程是一样的.

举例创建开发推送证书.

![截图](http://i1.tietuku.com/d2ef522293159f4d.png)

![截图](http://i1.tietuku.com/4fc03d22aebd04dc.png)

![截图](http://i1.tietuku.com/09ce52e7074177c2.png)
**这一步的作用:将当前Mac机器可以推送远程通知到手机.**

![截图](http://i1.tietuku.com/b4f2b03a53478348.png)

![截图](http://i1.tietuku.com/c5ab816a5bb8e504.png)
**保存到本地**

###然后返回之前的步骤，选择本机申请的证书，即可完成创建配置推送证书.

###No4. 创建包含【推送调试】和【推送发布】的证书的描述文件.

![截图](http://i1.tietuku.com/a4730177c91cb194.png)

![截图](http://i1.tietuku.com/9dcfc2287c0328f8.png)

###No5. 下载证书描述文件安装

###No6. 将安装的证书导出为p12交换证书，能够在其他电脑调试推送通知.


总结: 1)推送证书的作用  2)描述文件profile作用

![](http://i3.tietuku.com/4fa21fef05bc7c76.png)






