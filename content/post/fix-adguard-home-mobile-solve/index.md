+++
date = '2025-04-18T23:22:56+08:00'
draft = false
title = '记一次 Adguard Home 移动端解析失效的处理过程'
slug = 'fix-adguard-home-mobile-resolve'
description = ''
image= ''
license=''
tags = [
    '问题记录',
    'DNS',
    'adguardhome'
]
categories = [
    '网络',
]
+++
## 文前注  
这大概是2月份处理的事，趁着这次正经搞了一下博客，顺道整理一下写出来吧，不然博客搭完又要空荡荡在这里挂着  

## 起因  
为了过滤广告和方便查看DNS查询日志  
在原有分流DNS+代理的基础上，部署了 Adguard Home ，并已配置好指向分流DNS的地址和DNS黑名单规则  

在pc和服务器上引用 Adguard Home 的DNS均没有问题  
当在手机上设置 Adguard Home 的DNS后  
需要科学上网的网站出现了间歇性无法访问的情况  

## 问题调查记录 
之前几次搭建都因为这个问题被迫撤掉 Adguard Home 的部署，于是打算详细查一下原因  
既然访问有问题，那么先查询一下浏览器的解析情况  
### 查询浏览器解析
在浏览器地址栏输入`chrome://net-internal`，并切换到DNS页面，输入访问出现问题的网站 
 
![DNS查询结果为错误的IPV4地址](https://r2-static.daiyousei.moe/e2be084271b7c6c1efdf1ea6c9d02c972ab3d98c.jpg)

很显然，这个IP绝对不是通过 Adguard Home 解析出来的  

作为对比，这是使用分流DNS+代理的解析结果  

![DNS查询结果为代理的FakeIP](https://r2-static.daiyousei.moe/59911734f2f4eadc4a92e79a6975cd17d5303e5a.jpg)

### 获取浏览器网络日志
现在需要获取浏览器究竟使用了什么DNS来解析地址  
在地址栏输入`chrome://net-export`，进入 Capture Network Log 页面  
点击 `Start Logging to Disk` 开始记录，新建页面并访问有问题的网站  
回到日志记录页面，点击 `Stop Logging` ，点击 `Email Log` ，将日志发送到电脑上  

![开始记录](https://r2-static.daiyousei.moe/5d5783f265c7589a55838072f6d15db8dde2e976.jpg) ![停止记录](https://r2-static.daiyousei.moe/5bd212a7cb68a779991dc7e190ee49006639bf99.jpg) ![发送日志文件](https://r2-static.daiyousei.moe/bdab862f13e40337b64859be2d2b2e96c26618b0.jpg)

在电脑上的浏览器地址栏输入 `chrome://net-internal` ，点击 `netlog_viewer` 链接，进入日志解析网站  
上传发送到电脑的日志，选择DNS选项卡  

![日志获取到的浏览器使用的DNS信息](https://r2-static.daiyousei.moe/757361224ed0584153d9722280894a1fe8648a12.png) 

![问题网站的解析记录](https://r2-static.daiyousei.moe/3b1793de095d62f19ef67b803c8d85960129ba69.png)

看见了熟悉的面孔

在网上使用关键词搜索了一下，发现了这样一则[讨论](https://github.com/pymumu/smartdns/issues/712)[^1]

[^1]: [小米9故意内置隐藏的114DNS污染了路由器海外DNS解析结果 · Issue #712 · pymumu/smartdns](https://github.com/pymumu/smartdns/issues/712)

那么问题原因就很清晰了

### 额外试错：搭建转发DNS验证缺省DNS问题  
现在让我们回到获取并解析浏览器日志这步  
当时的猜想是当前手机网络设置只指定了1个DNS，需要确认一下114DNS是否为缺省DNS填充  

暂时懒得写了，先咕了

## 问题处理    
大概原因是 Adguard Home 黑名单规则屏蔽了手机厂商的遥测域名，系统自动添加了114DNS作为备用DNS  
导致需要科学上网的网站间歇性无法访问  

处理方式为屏蔽114DNS或者将114DNS转发到 Adguard Home DNS  
以下操作以`openwrt`为基准
* 屏蔽114 DNS  
  * `网络 > 防火墙 > 通信规则` 下添加规则，规则内容如下图所示  

![通信规则设置](https://r2-static.daiyousei.moe/eb5474cd84380e681dea048bd996ef1b375833be.png)

* 转发114 DNS 至 Adguard Home  DNS  
  * `网络 > 防火墙 > 端口转发` 下添加规则，规则内容如下图所示  

![端口转发基本设置](https://r2-static.daiyousei.moe/2e151214dd7353756a2f6bbe8a72efdb64b3ab51.png)

![端口转发高级设置](https://r2-static.daiyousei.moe/c46e37e8f9e57821f83429017aaf274f65dd5e91.png)

> [!NOTE]
> 由于本人的整套解析链均未在openwrt上部署，所以关闭了openwrt的dns重定向防止上述配置失效。  
> 请读者根据实际运行环境情况进行调整。


