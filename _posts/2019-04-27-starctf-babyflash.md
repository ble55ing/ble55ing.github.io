---
layout: post
title:  "*ctf babyflash"
categories: ctf
tags: *ctf misc
author: ble55ing
---

* content
{:toc}

## *ctf babyflash writeup

一道flash题目，本来只是随手打开的，结果就做出来了，还拿了一血 ）逃（

## 上半部分

首先看一下这是一个swf文件，打开是黑白黑白闪烁的图片。一数，诶，441张，这不就是一个21*21的二维码吗，试了一下第一行，还真是，于是解出了前半部分的flag

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/qrcode8.png)

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/qian4.png)

## 下半部分

再看一下swf文件，打开时其中是有音频的，把音频提取出来，然后放到[Audacity](https://www.audacityteam.org/) 里，把波形图改为频谱图，就能看到下半部分的flag了

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/rst9.png)



