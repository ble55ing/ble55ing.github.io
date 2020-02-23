---
layout: post
title:  "十一月CTF部分题目汇总一"
categories: ctf pwn
tags: ctf pwn
author: ble55ing
---

* content
{:toc}
## pwn 线下使用 binwaf

众所周知web线下是可以通过安装waf来修补程序、截获流量的，而pwn由于权限限制往往做不到这些，因此往往需要对程序进行patch，在程序中得到流量。

