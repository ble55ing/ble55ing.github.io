---
layout: post
title:  "cybricsctf Tone"
categories: ctf
tags: cybricsctf misc
author: ble55ing
---

* content
{:toc}

## 西湖论剑2019的Storm_Note--largebin attack

当时做完了对largebin有了一个了解，现在又记得不深了，还是补一篇记录出来。

### 题目分析

题目有一个offbynull，而且给了一个0xABCD0100开始的空间，并且给了一个后门函数，后门函数会比较在0xABCD0100位置的值与输入的值做比较，一样就返回shell，没有show。此题将M_MXFAST设置为0，禁了fastbin。 

### 解题流程

#### offbynull

首先利用offbynull得到overlap，具体方法为首先构造0x20-0x510-0x20的连续分配，然后在0x510的最后伪造堆头，删除0x510，使用第一个块进行offbynull，然后申请两个块，0x20和0x4e0把0x500的大小分了，把这个0x20和后面的那个0x20删除，就得到了一个0x530的合并大块，其中间是overlap的0x4e0。

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20190726192105.png)

如图为构造的在进行overlap最后堆块合并前的堆空间。

#### largebin atatck

使用两次overlap，分别从0x530里分出一个0x4f0和0x4e0的块，将申请的0x4e0释放，该块会进入unsorted bin，再释放0x4f0的块，该块也会在unsorted bin，然后申请0x4f0的块，这时会将0x4e0的块放入largebin。

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20190726201059.png)

然后通过两个overlap去修改这两个块的指针，使得得到0xABCD0100，然后将其值修改了就ok了。

首先来捋一下接下来申请时的逻辑，目的是将0x56写到0xABCD0100前面一点的地址，然后得到这个块。申请首先找unsorted bin，然后把0x4f0的块放入large bin，然后找它的下一块。在放入largebin的过程中，会将bk的fd和bk_size的fd_size都更新，所以控制0x4e0的bk是fake_chunk+8，bk_size是ake_chunk-0x18-5；0x4f0的bk是fake_chunk。fake_chunk为0xABCD00e0。所以申请0x40时申请到的块如下所示：

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20190726205111.png)

### 题目文件

由于有一段时间了，就放在这里了<https://github.com/ble55ing/ctfpwn/tree/master/pwnable/ctf/x64> Storm_Note