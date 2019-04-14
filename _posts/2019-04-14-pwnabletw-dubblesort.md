---
layout: post
title:  "git使用"
categories: ctf
tags: pwnable
author: ble55ing
---

* content
{:toc}

##pwnable.tw dubblesort 分析

本题考察的是scanf的%d对于‘+’的处理，相当于没有输入

题目为一个排序算法，就如题目名称那样，dubblesort，32位程序。

利用思路为栈溢出，先是栈溢出泄露出栈上libc的相关数据从而获取libc地址，再是栈溢出劫持控制流。第二步时需要注意存在着一个canery需要绕过，可以使用scanf的%d输入为+时为没有输入的方式跳过（返回值会出错但本题没检查），就一个数字的长度，在第25个。
