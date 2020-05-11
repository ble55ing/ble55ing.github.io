---
layout: post
title:  "de1ctf Mine sweeping"
categories: ctf
tags: cybricsctf misc
author: ble55ing
---

* content
{:toc}
## de1ctf的Mine sweeping

勇气、危机、未知、热血、谋略，3A级游戏大作——扫雷

### 题目分析

题目是一个Unity游戏，将其Assembly-CSharp.dll放到dnSpy里，看到其地图分析的逻辑。找到其地图相关的信息。

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20190803134124.png)

找到了一个DevilsInHeaven数组，但这个数组并不是按照顺序来的，其中的每一个数据，是从下往上的某一列的数据，1为有雷，0为没有。

然后还找到了Changemap的函数，该函数说明了这个雷的分布也不是完全和前面那个数组一样的，有一些位置（6个）被进行了随机。

### 得到二维码

这个扫雷雷太多了，所以是不可能正常的扫出来的。

由于ChangeMap改的非常少，所以每次的图其实差别不大。

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20190803134110.png)

发现了左上左下和右下的大方框和右上的小方框，感觉是向左旋转90度的二维码。

然后一列一列试DevilsInHeaven数组中的数据，找到对应的列

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20190803134133.png)

编了个小程序去找合适的列，有的列的前几位这中有被修改了部分，比如最后一列，就没有在这里显示出来。

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20190803135226.png)

整理出来就是这样的一个二维码，其中灰色的框为随机数的，这里有5个，最后一个在左下的最后一行里，由于是大方框的一部分就没有标出来。

扫描二维码得网址得flag

