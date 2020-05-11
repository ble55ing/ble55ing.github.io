---
layout: post
title:  "根据cfg分析afl的路径信息（2）"
categories: CodeAnalyze
tags: angr afl
author: ble55ing
---

* content
{:toc}
# 根据cfg分析afl的路径信息（2）

使用angr构建二进制程序的cfg图的第二部分

前文链接<https://ble55ing.github.io/2019/06/20/afl-cfgpath/> 

上回书讲到，跑了好久没有结果，这回分析了一下原因，是原程序C语言写的，有很多自妇产逐字节比较的内容，在进行编译时，就变成了很多很多的if，导致程序的cfg图非常庞大。因而去优化了一下程序，使用C++来编写，这样就可以通过调用库函数来实现功能，大大的简化了程序流程，就能够愉快的跑起来啦

本文介绍在跑起来之后还遇到的其他问题

### CFG图的简化

angr生成的CFG图会在每个函数的调用时生成两条路径，一条是调用了函数的一条是没调用的。这显然是不合适的，由于函数本身其时不可能不进行该函数的调用，所以要将跳过该函数的路径除去。

#### CFG图的情况

如果是完整的cfg图（full）的话就是这样的操作了，但这里的不计算库函数的，所以需要区分库函数和程序声明的函数。具体的区分方法为，库函数会在cfg图的dot文件中表示为其plt表的地址，然后有唯一一条路径指向0x1000xxx的地址的一个基本块（也不是全部，比如这次fgetc和fopen，没有这个0x1000xxx的地址），在这个基本块里才有函数名称；而对于程序声明的函数在其函数体的第一个基本块就会有该函数的名称。

```
	296	 [fontname=monospace,
		fontsize=8.0,
		height=0.65278,
		label=<{ <TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD >0x4008e0</TD><TD >(0x4008e0)</TD><TD></TD><TD></TD></TR></TABLE>|<TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD ALIGN="LEFT">0x004008e0&#58;&nbsp;</TD><TD ALIGN="LEFT">jmp</TD><TD ALIGN="LEFT">qword ptr [rip + 0x202762]</TD><TD></TD></TR></TABLE> }>,
		pos="10186,1.573e+05",
		shape=Mrecord,
		width=3.4861];
	296 -> 305	 [color=blue,
		fontname=monospace,
		fontsize=8.0,
		pos="e,10186,1.5722e+05 10186,1.5727e+05 10186,1.5726e+05 10186,1.5724e+05 10186,1.5723e+05"];
	305	 [fillcolor="#dddddd",
		fontname=monospace,
		fontsize=8.0,
		height=0.51389,
		label=<{ <TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD >0x1000030</TD><TD >(0x1000030)</TD><TD ><B>__stack_chk_fail</B></TD><TD >SIMP</TD></TR></TABLE> }>,
		pos="10186,1.572e+05",
		shape=Mrecord,
		style=filled,
		width=3.4306];
	305 -> 305	 [color=orange,
		fontname=monospace,
		fontsize=8.0,
		pos="e,10309,1.5719e+05 10309,1.5722e+05 10320,1.5721e+05 10327,1.5721e+05 10327,1.572e+05 10327,1.572e+05 10324,1.5719e+05 10319,1.5719e+\
05"];
```

如```__stack_chk_fail```函数，其调用是对296块的调用，然后296中并没有表示其函数名，事实上这个是```__stack_chk_fail```函数的plt表位置，而又唯一出路径296->305，305中保有其函数名。这个305

 -> 305的自调用是很特殊的，其实目前只发现```__stack_chk_fail```和```exit```这种不正常退出的函数会有。

```
0	 [fontname=monospace,
		fontsize=8.0,
		height=1.5556,
		label=<{ <TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD >0x4009d0</TD><TD >(0x4009d0)</TD><TD ><B>main</B></TD><TD></TD></TR></TABLE>|<TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD ALIGN="LEFT">0x004009d0&#58;&nbsp;</TD><TD ALIGN="LEFT">lea</TD><TD ALIGN="LEFT">rsp, qword ptr [rsp - 0x98]</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x004009d8&#58;&nbsp;</TD><TD ALIGN="LEFT">mov</TD><TD ALIGN="LEFT">qword ptr [rsp], rdx</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x004009dc&#58;&nbsp;</TD><TD ALIGN="LEFT">mov</TD><TD ALIGN="LEFT">qword ptr [rsp + 8], rcx</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x004009e1&#58;&nbsp;</TD><TD ALIGN="LEFT">mov</TD><TD ALIGN="LEFT">qword ptr [rsp + 0x10], rax</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x004009e6&#58;&nbsp;</TD><TD ALIGN="LEFT">mov</TD><TD ALIGN="LEFT">rcx, 0xbb94</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x004009ed&#58;&nbsp;</TD><TD ALIGN="LEFT">call</TD><TD ALIGN="LEFT">0x4020e0</TD><TD></TD></TR></TABLE> }>,
		pos="9883.5,1.8988e+05",
		shape=Mrecord,
		width=3.6389];
```

而程序声明的函数则不会出现这种情况，因为不是通过plt表做的调用。如上是main函数的表示，这是一个main函数开始的第一个基本块。

#### 预处理

鉴于上述问题，需要对CFG图进行预处理，实际上这个预处理的功能很大程度上提升了效率。。

预处理的内容为：

​	1将所有调用库函数的路径消去；

​	2将所有调用```afl_maybe_log```、```__afl_maybe_log0```等函数的路径消去

​	3将所有调用程序声明函数而没进入的路径消去

消去上述三种路径后，分析块多了。。

### 内部处理

#### 关于循环

循环会导致在进行总的路径提取时陷入循环，所以需要进行处理

处理的方式分为while型和do-while型，即有第一个基本块判断是否达到条件跳出循环的也有最后一块判断的。前者的特征是2进2出，后者是1进2出，都是会构成环路的情况。目前暂定的处理方式为：将循环分为走0遍和走1遍的情况进行路径的处理，再走第二遍的时候就跳出循环。前者由于在第二次到达时就已经走过一遍循环体了，直接返回就行了；而后者触发第二次到达的是循环中的第一个基本块，所以要找到这个基本块的前一跳的基本块然后跳出去。

上述的这种方式感觉效果不佳，。。而且其实while型和do-while型区别不大。所以决定采用在第三次到达某一位置时，查看这次到达是否在之前存在过的形式来判断。如果这种形式之前存在了，则舍弃这次跳转。

#### 关于自循环

在CFG图中发现了一些自循环的基本块，如repe cmpsb这样的指令，由于出度有两个，直接忽略掉去自循环的调用关系就好了。

