---
layout: post
title:  "pwnable.tw applestore"
categories: pwnable
tags:  pwn
author: ble55ing
---

* content
{:toc}

## pwnable.tw applestore 分析

此题第一步凑齐7174进入漏洞地点

然后可以把iphone8的结构体中的地址通过read修改为一个.got表地址，这样就能把libc中该函数地址打出来。这是因为read函数并不会在遇到\x00时截断（就是在read字符'y'的时候）。

然后还可以采用同样的办法把libc中的libc.symbols['environ']打出来，这个值存储着当前栈中环境变量所在的位置，因而可以得到栈的位置。

然后具体分析在进入delete之后的情景：

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/20190414213415.png)

将程序断在delete中的关键位置，此时ebp为0xffac6cc8，而v2的值为0xffac6ca8，即在atoi数字后面的值，因而可以控制v2和v4，v5

然后可以通过这个*(_DWORD *)(v5 + 8) = v4;，把ebp改掉。v5=$ebp-0x8，v4=atoi_got_addr + 0x22，这样在下次handler中就可以向atoi_got中写入system地址了。


