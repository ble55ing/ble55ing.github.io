---
layout: post
title:  "Windows Pwn相关知识"
categories: Windows
tags: pwn
author: ble55ing
---

* content
{:toc}
## Windows Pwn相关知识 

ctf比赛中的Windwos Pwn题目愈发的多了，学一波

杂乱的Windwos相关知识

### Windows x64平台fastcall调用约定 

Windwos有三种调用方式，待续

如何将值放入xmm的方式，待续

下面是参考资料1中的描述

1	前四个整型或指针类型参数由RCX,RDX,R8,R9依次传递，前四个浮点类型参数由XMM0,XMM1,XMM2,XMM3依次传递。

2	调用函数为前四个参数在调用栈上保留相应的空间，称作shadow space或spill slot。即使被调用方没有或小于4个参数，调用函数仍然保留那么多的栈空间，这有助于在某些特殊情况下简化调用约定。

3	除前四个参数以外的任何其他参数通过栈来传递，从右至左依次入栈。

4	由调用函数负责清理调用栈。

5	小于等于64位的整型或指针类型返回值由RAX传递。

6	浮点返回值由XMM0传递。

7	更大的返回值(比如结构体)，由调用方在栈上分配空间，并有RCX持有该空间的指针并传递给被调用函数，因此整型参数使用的寄存器依次右移一格，实际只可以利用3个寄存器，其余参数入栈。函数调用结束后，RAX返回该空间的指针。

8	除RCX,RDX,R8,R9以外，RAX、R10、R11、XMM4 和 XMM5也是易变化的(volatile)寄存器。

9	RBX, RBP, RDI, RSI, R12, R14, R14, and R15寄存器则必须在使用时进行保护。

10	在寄存器中，所有参数都是右对齐的。小于64位的参数并不进行高位零扩展，也就是高位是无法预测的垃圾数据。

### 虚表的vtable继承

1	当类继承了父类（父类含有虚函数），或者类中本来就有虚函数，那么编译之后就会产生虚表。

2	若A、B继承了父类C，那么在rdata段中将会有连续的空间存放A继承C的虚函数指针、B继承C的虚函数指针

3	实例化A或B时，将会在栈空间，则地址的首部放置指向虚表的指针，紧接着是各个局部变量;若是new()创建，则是堆空间，这时局部变量放置到栈空间，malloc的变量放置到堆空间。

### Windows的安全防护机制

参考另一篇文章请<https://ble55ing.github.io/2019/08/18/WindowsPwn0/> 

### 参考资料

<https://introspelliam.github.io/2017/08/05/pwn/XMAN%E4%B9%8B%E6%97%85-windows-exploit-technique/> 