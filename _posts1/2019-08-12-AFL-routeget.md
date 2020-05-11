---
layout: post
title:  "angr AFL插桩全路径获取"
categories: coding
tags: coding
author: ble55ing
---

* content
{:toc}
## angr AFL插桩全路径获取

近期需要做AFL的全路径获取，所以来做一做

### 构建程序控制流图

使用angr进行程序流程图的构建，在较细粒度上对程序进行静态分析，生成源码的程序控制流图，以基本块为单位，并记录了基本块之间的跳转关系。使用CFGAccurate生成控制流图，生成的文件为.dot的控制流图描述文件。这种粒度的控制流图将每条路径上对相同函数的调用都单独的区分出来了，很方便进行路径的分析。

### 获取全路径

首先根据控制流图描述文件的格式，使用正则表达式来做匹配，将控制流图中的每一条线表示为一个基本块间的跳转关系，将控制流图中的每一个节点表示为一个基本块，并将每个基本块中，存在的函数起始地址进行记录，尤其要记录主函数的地址和所在基本块，作为路径分析的起点。此处为了删除AFL插入的代码部分对源程序控制流的影响，记录每个进入了```__afl_maybe_log```的基本块。同时为了接下来的生成AFL路径信息做准备，将含有AFL插桩的随机数的基本块进行记录，记录时哪个基本块包含有哪些AFL随机数。

### 路径预处理

对收集完的原始路径进行预处理，方便接下来的工作。

首先是对函数表和调用表的预处理，预处理的主要目的是排序和去重。函数表使用地址进行排序，调用表使用进行基本块进行排序。

#### 对AFL插入函数调用的预处理

接下来将删除AFL插入的代码部分对源程序控制流的影响。这一部分是将所有基本块中，调用afl相关函数的调用关系删除。分析发现，afl的相关函数的调用与原程序逻辑呈现这样的关系：

对于代码块A：

```
.text:00000000004009D0                 lea     rsp, [rsp-98h]
.text:00000000004009D8                 mov     [rsp+98h+var_98], rdx
.text:00000000004009DC                 mov     [rsp+98h+var_90], rcx
.text:00000000004009E1                 mov     [rsp+98h+var_88], rax
.text:00000000004009E6                 mov     rcx, 0BB94h
.text:00000000004009ED                 call    __afl_maybe_log
.text:00000000004009F2                 mov     rax, [rsp+98h+var_88]
.text:00000000004009F7                 mov     rcx, [rsp+98h+var_90]
.text:00000000004009FC                 mov     rdx, [rsp+98h+var_98]
.text:0000000000400A00                 lea     rsp, [rsp+98h]
.text:0000000000400A08                 push    r13
.text:0000000000400A0A                 push    r12
.text:0000000000400A0C                 push    rbp
.text:0000000000400A0D                 push    rbx
.text:0000000000400A0E                 sub     rsp, 8
.text:0000000000400A12                 cmp     edi, 1
.text:0000000000400A15                 jle     loc_400E48
```

其控制流描述如下所示：

```
0	 label=<{ <TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD >0x4009d0</TD><TD >(0x4009d0)</TD><TD ><B>main</B></TD><TD></TD></TR></TABLE>|<TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD ALIGN="LEFT">0x004009d0&#58;&nbsp;</TD><TD ALIGN="LEFT">lea</TD><TD ALIGN="LEFT">rsp, qword ptr [rsp - 0x98]</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x004009d8&#58;&nbsp;</TD><TD ALIGN="LEFT">mov</TD><TD ALIGN="LEFT">qword ptr [rsp], rdx</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x004009dc&#58;&nbsp;</TD><TD ALIGN="LEFT">mov</TD><TD ALIGN="LEFT">qword ptr [rsp + 8], rcx</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x004009e1&#58;&nbsp;</TD><TD ALIGN="LEFT">mov</TD><TD ALIGN="LEFT">qword ptr [rsp + 0x10], rax</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x004009e6&#58;&nbsp;</TD><TD ALIGN="LEFT">mov</TD><TD ALIGN="LEFT">rcx, 0xbb94</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x004009ed&#58;&nbsp;</TD><TD ALIGN="LEFT">call</TD><TD ALIGN="LEFT">0x4020e0</TD><TD></TD></TR></TABLE> }>,
1	 label=<{ <TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD >0x4020e0</TD><TD >(0x4020e0)</TD><TD ><B>__afl_maybe_log</B></TD><TD></TD></TR></TABLE>|<TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD ALIGN="LEFT">0x004020e0&#58;&nbsp;</TD><TD ALIGN="LEFT">lahf</TD><TD></TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x004020e1&#58;&nbsp;</TD><TD ALIGN="LEFT">seto</TD><TD ALIGN="LEFT">al</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x004020e4&#58;&nbsp;</TD><TD ALIGN="LEFT">mov</TD><TD ALIGN="LEFT">rdx, qword ptr [rip + 0x201005]</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x004020eb&#58;&nbsp;</TD><TD ALIGN="LEFT">test</TD><TD ALIGN="LEFT">rdx, rdx</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x004020ee&#58;&nbsp;</TD><TD ALIGN="LEFT">je</TD><TD ALIGN="LEFT">0x402110</TD><TD></TD></TR></TABLE> }>,
5	 label=<{ <TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD >0x4009f2</TD><TD >(0x4009d0)</TD><TD ><B>main+0x22</B></TD><TD></TD></TR></TABLE>|<TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD ALIGN="LEFT">0x004009f2&#58;&nbsp;</TD><TD ALIGN="LEFT">mov</TD><TD ALIGN="LEFT">rax, qword ptr [rsp + 0x10]</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x004009f7&#58;&nbsp;</TD><TD ALIGN="LEFT">mov</TD><TD ALIGN="LEFT">rcx, qword ptr [rsp + 8]</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x004009fc&#58;&nbsp;</TD><TD ALIGN="LEFT">mov</TD><TD ALIGN="LEFT">rdx, qword ptr [rsp]</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x00400a00&#58;&nbsp;</TD><TD ALIGN="LEFT">lea</TD><TD ALIGN="LEFT">rsp, qword ptr [rsp + 0x98]</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x00400a08&#58;&nbsp;</TD><TD ALIGN="LEFT">push</TD><TD ALIGN="LEFT">r13</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x00400a0a&#58;&nbsp;</TD><TD ALIGN="LEFT">push</TD><TD ALIGN="LEFT">r12</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x00400a0c&#58;&nbsp;</TD><TD ALIGN="LEFT">push</TD><TD ALIGN="LEFT">rbp</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x00400a0d&#58;&nbsp;</TD><TD ALIGN="LEFT">push</TD><TD ALIGN="LEFT">rbx</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x00400a0e&#58;&nbsp;</TD><TD ALIGN="LEFT">sub</TD><TD ALIGN="LEFT">rsp, 8</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x00400a12&#58;&nbsp;</TD><TD ALIGN="LEFT">cmp</TD><TD ALIGN="LEFT">edi, 1</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x00400a15&#58;&nbsp;</TD><TD ALIGN="LEFT">jle</TD><TD ALIGN="LEFT">0x400e48</TD><TD></TD></TR></TABLE> }>,
0 -> 5
0 -> 1
```

总共涉及代码段0,1,5三个，0->5，0->1两个基本块之间的跳转关系。

基本块0是从代码块A的开始到```call    __afl_maybe_log```，基本块1是```__afl_maybe_log```的开始地方，基本块5是从语句```mov     rax, [rsp+98h+var_88]```到代码块A结束的部分。因此，将代码块0->1的这个跳转关系删除并不会影响对原程序逻辑的分析，而且会简化程序的执行流程。

#### 对系统函数的预处理

系统函数并没有函数内部的逻辑实现 ，因此对其的调用也可以仿照AFL的进行操作：将所有进入函数内部的逻辑剪掉，不过如下所示，为getenv函数在控制流描述中的异常调用：

```
10	 label=<{ <TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD >0x400880</TD><TD >(0x400880)</TD><TD></TD><TD></TD></TR></TABLE>|<TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD ALIGN="LEFT">0x00400880&#58;&nbsp;</TD><TD ALIGN="LEFT">jmp</TD><TD ALIGN="LEFT">qword ptr [rip + 0x202792]</TD><TD></TD></TR></TABLE> }>,
13	 label=<{ <TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD >0x1000000</TD><TD >(0x1000000)</TD><TD ><B>getenv</B></TD><TD >SIMP</TD></TR></TABLE> }>,
10 -> 13
```

基本块10是PLT表中的内容，基本块13是函数体的内容，由于是系统的函数，所以没有函数体。基本块10只有对基本块13的调用关系，基本块13也只有基本块10会到达。

#### 对程序中函数的预处理

同样的问题当然也会发生在程序中的函数上。这里的函数是一定会运行到的，因此不可能存在跳过的情况。因此操作与前两者相反，要将跳过函数体的分支删除。如对于代码块B：

```
.text:0000000000400A65                 mov     edi, offset filename ; "all"
.text:0000000000400A6A                 mov     rbp, rax
.text:0000000000400A6D                 call    filedeal
.text:0000000000400A72                 mov     rdi, f1         ; stream
.text:0000000000400A75                 call    _fgetc
```

其控制流描述如下所示：

```
166	 label=<{ <TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD >0x400a65</TD><TD >(0x4009d0)</TD><TD ><B>main+0x95</B></TD><TD></TD></TR></TABLE>|<TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD ALIGN="LEFT">0x00400a65&#58;&nbsp;</TD><TD ALIGN="LEFT">mov</TD><TD ALIGN="LEFT">edi, 0x402603</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x00400a6a&#58;&nbsp;</TD><TD ALIGN="LEFT">mov</TD><TD ALIGN="LEFT">rbp, rax</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x00400a6d&#58;&nbsp;</TD><TD ALIGN="LEFT">call</TD><TD ALIGN="LEFT">0x401000</TD><TD></TD></TR></TABLE> }>,
167	 label=<{ <TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD >0x401000</TD><TD >(0x401000)</TD><TD ><B>filedeal</B></TD><TD></TD></TR></TABLE>|<TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD ALIGN="LEFT">0x00401000&#58;&nbsp;</TD><TD ALIGN="LEFT">lea</TD><TD ALIGN="LEFT">rsp, qword ptr [rsp - 0x98]</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x00401008&#58;&nbsp;</TD><TD ALIGN="LEFT">mov</TD><TD ALIGN="LEFT">qword ptr [rsp], rdx</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x0040100c&#58;&nbsp;</TD><TD ALIGN="LEFT">mov</TD><TD ALIGN="LEFT">qword ptr [rsp + 8], rcx</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x00401011&#58;&nbsp;</TD><TD ALIGN="LEFT">mov</TD><TD ALIGN="LEFT">qword ptr [rsp + 0x10], rax</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x00401016&#58;&nbsp;</TD><TD ALIGN="LEFT">mov</TD><TD ALIGN="LEFT">rcx, 0xaf75</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x0040101d&#58;&nbsp;</TD><TD ALIGN="LEFT">call</TD><TD ALIGN="LEFT">0x4020e0</TD><TD></TD></TR></TABLE> }>,
293	 label=<{ <TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD >0x400a72</TD><TD >(0x4009d0)</TD><TD ><B>main+0xa2</B></TD><TD></TD></TR></TABLE>|<TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD ALIGN="LEFT">0x00400a72&#58;&nbsp;</TD><TD ALIGN="LEFT">mov</TD><TD ALIGN="LEFT">rdi, rbp</TD><TD></TD></TR><TR><TD ALIGN="LEFT">0x00400a75&#58;&nbsp;</TD><TD ALIGN="LEFT">call</TD><TD ALIGN="LEFT">0x4008f0</TD><TD></TD></TR></TABLE> }>,
166 -> 167
166 -> 293
```

基本块167是一个函数filedeal的开始部分，跳转到它的路径只有基本块166。基本块166是在代码块B中到调用filedeal函数的部分，基本块293是之后的部分。分析程序流程，程序不可能在部调用基本块167的filedeal函数的情况下走到基本块293，所以这其实是一条不可达的路径，应当删除。

#### 自循环指令的处理

在程序中是有一些能够产生自循环的指令，它们本身不会产生路径，因此也可以删除。

```
510	 label=<{ <TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD >0x400d96</TD><TD >(0x4009d0)</TD><TD ><B>main+0x3c6</B></TD><TD></TD></TR></TABLE>|<TABLE BORDER="0" CELLPADDING="1" ALIGN="LEFT"><TR><TD ALIGN="LEFT">0x00400d96&#58;&nbsp;</TD><TD ALIGN="LEFT">repe cmpsb</TD><TD ALIGN="LEFT">byte ptr [rsi], byte ptr [rdi]</TD><TD></TD></TR></TABLE> }>,
510 -> 510
```

比如在0x400d96位置的基本块510，指令为repe cmpsb，这条指令是一条比较指令，把ESI指向的数据与EDI指向的数一个一个的进行比较。  这条指令就存在循环调用自身的情况，而我们不必去关注这情况，所以可以将这个自身的调用关系除去。

### 全部路径获取

接下来要通过基本块的调用关系，得到全部的程序执行路径。这些执行路径要保证能够做到不重不漏。

决定采用从主函数基本块开始递归进行路径查询，在到达路径结束位置的时候返回的方式来生成全部路径。由于在之前的预处理中的处理方法是将进入的路径全部去除，所以能够在这种遍历方法中起到有用的效果。

这里值得注意的是对循环操作的处理办法，因为在处理到循环的时候，即使循环体执行一轮，执行的路径也不会是新的路径，而且就路径而言循环体的存在会产生一个路径的环路，如果不处理，就会影响到递归的效果。因此，如何处理循环，是对全路径获取的一个难题。

处理循环时的关注点有三个：第一次进行循环，第二次进行循环，和离开循环。第一次进入循环时，产生的都是新的路径，因此必须予以记录；离开循环时也会产生新的路径，也需要予以记录。而第二次运行循环时，所走的路径与第一次循环时可能相同也可能不同，而从循环底部到循坏头部的路径是新的路径。因此，拟将一个循环抽象为第一次进行循环、第二次进行循环、循环过程和离开循环三个过程，其中，循环过程为第二次循环以后的在循环中的部分的全部集合。一种可行的做法为，当到达一个基本块非第一次也非第二次时，如果所执行的路径是第一次或自二次执行过的，则可认为其路径的遍历工作在之前的路径遍历中已经进行完了，因此可将这此的路径遍历舍弃。如果这个路径是之前遍历中没有进行的，则将其视作循环过程中的一部分。即循环过程中可以有多种执行执行循环的方式，但均不能重复，也不与第一次循环和第二次循环重复。

### AFL路径获取

根据获取的每一条路径，按照AFL的路径计算方式，使用AFL插入的随机数，对AFL的路径进行计算。由于已将所有的路径遍历过了，所以生成的AFL路径也是全部的AFL路径。之后就可以使用这个AFL的全路径进行路径分析了。