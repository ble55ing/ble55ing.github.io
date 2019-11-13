---
layout: post
title:  "HITB2017 Windows babyshellcode"
categories: ctf
tags: pwn Windows
author: ble55ing
---

* content
{:toc}
## HITB2017 babyshellcode

古老的Windows题，希望容易一点。

### 题目描述

32位PE文件，可以申请20个节点，在0x4053fb的位置是Memory，记录着每个节点的各项内容的指针。

在0x5448位置有一个守卫变量，不为0的时候就没法执行shellcode。

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/babyshellcode1.png)

功能有add，list，delete，run，set guard和exit。list可以得到node的信息。

### 漏洞分析

主函数有栈溢出但是exit所以过。

#### allocsc函数的整数溢出

本题中使用的是allocsc进行内存分配（分配的内存都是连着的没有堆头啥的）

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20190823232657.png)

如图为每次一个A然后最后是3次got表地址

接下来看allocsc的实现

```
int __cdecl allocsc(int size)
{
  int v1; // edx@1
  int result; // eax@2

  v1 = size + NEXT;                             // int overflow
  if ( size + NEXT < (unsigned int)(HEAP + HEAP_SIZE) )
  {
    NEXT += size;
    result = v1 - size;
  }
  else
  {
    result = -1;
  }
  return result;
}
```

这里的size+NEXT是存在整数溢出的，为负数时总能通过校验。

### 漏洞利用

使用这个allocsc的漏洞，将内存分配到Memory处，就是申请一个足够大的内存，将堆的接下来分配到note的起始处，然后再分配就可以改掉note中指针的值。然后可以用list函数去泄露内容，得到got表中的值，进而计算出配套dll中get_shell代码的位置。然后再分配一个较大的空间，去覆盖掉0x5448位置的守卫，就可以用run去运行get_shell了！！