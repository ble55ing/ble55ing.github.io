---
layout: post
title:  "cybricsctf Tone"
categories: ctf
tags: cybricsctf misc
author: ble55ing
---

* content
{:toc}

## cybricsctf Tone writeup

这个金砖5国ctf真的有意思，有很多新的有趣知识，这道题目是听声音辨别电话号码

zakukozh和battleship就不写wp了，比较常规，前者是文件的逐字节加密，y=ax+b的形式，后者是battleship游戏，电脑会一次打好多炮，可以在save的时候得到对面哪些地方有船的信息，还可以load一次地图。保存的文件一开始有两个字节分别两方的船数，然后后面有一段是每艘船的血量，然后是两个地图，然后最后是两个地图的crc32。根据每次save的结果进行比对就能得到各个位置的作用。

题目的文件可以在这里找到，https://github.com/ble55ing/ctfpwn/tree/master/2019cybricsctf

### 视频提取音频

现在的播放器很多都提供这种功能，如KMPlayer就是右键->声音->音轨->声音录制，得到MP4，再转成wav格式。

然后拿au打开，看其频谱图，是这样子的：

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20190721230931.png)

每个音由一个高频一个低频组成，其表如下所示：

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/20131209002603671.gif)

然后照着这个图比对，得到电话号码，222 999 22 777 444 222 7777 7777 33 222 777 33 8 8 666 66 2 555 333 555 2 4。根据9键键盘的输入方式，得到结果cybrics{secrettonalflag}



