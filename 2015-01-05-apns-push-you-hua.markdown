---
layout: post
title: "Apns Push 优化"
date: 2015-01-05 11:53:18 +0800
comments: true
categories: 
---


***

###苹果对ios push发送的长度有限制(官方文档提示ios 8以下是256byte，ios 8以上是2k，目前从发送反馈来看，一致定位256byte) ，其中包括push的属性字段。所以为了提高push中字段尽可能多一些，只能通过`缩减push属性字段的长度`

|序号|修改前|修改后|
|---|---|---|
|1|pushId|p|
|2| custom_property | cp |
|3| type |t|
|4| value | v |
|5| msg_type | mt |

注意，但是需要保证字段的唯一性.


***

###Apns推送通知，需要建立白名单

> 某条通知，决定只推送到指定的哪一些版本的App程序.

***

