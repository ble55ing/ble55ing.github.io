---
layout: post
title:  "VMware Workstation 未能启动 VMware Authorization Service"
categories: linux
tags: atmosphere
author: ble55ing
---

* content
{:toc}
## VMware Workstation 未能启动 VMware Authorization Service

今天启动vmware的时候，突然弹出来一个框告诉我这个，然后查的也没人写过禁用怎么办，所以记录一下。

首先是windows +r 在运行菜单输入services.msc ，打开服务项管理。

找到VMware Authorization Service，这里此时是禁用

双击把其修改为自动或手动，应用启动确定应该就成了，但我应用时弹出了禁止访问，有的博客写了有这种情况，但没写怎么解决。

这应该是由于杀毒软件将其禁用了的关系，在其中找到改成允许启动就好了。

我用的火绒是在启动项管理->服务项里要进行的修改。



