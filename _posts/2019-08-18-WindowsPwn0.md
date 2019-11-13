---
layout: post
title:  "Windows PWn"
categories: Windows pwn 
tags: pwn
author: ble55ing
---

* content
{:toc}
## Windows Pwn安全防护机制 

### PE 文件格式

DOS 头：MZ标识

PE 文件头：入口点、数据表

Section表：Section Headers的表

导入地址表：与ELF的GOT类似；只读

导出地址表：Exported	functions  of a  Module   只读

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/windows1.png)

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/windwos2.png)

### DEP

Linux上的NX，栈不可执行保护

绕过方式：

​	ROP	

​	JIT	page，VirualProtect![](https://raw.githubusercontent.com/ble55ing/PicGo/master/windwos3.png)

### ASLR

和Linux上的PIE&ASLR略有不同

镜像/程序的随机化只在系统重启时；

TEB/PEB/堆/栈的随机化是每次进程重启时。

一些内核相关的dll（如ntdldl.dll，kernel32.dll）与所有进程分享地址。

绕过方式：

​	信息泄露(包括跨进程) 

​	爆破(win7 x64,  win10 x86) ：64位操作系统的前32位是不变的，后16bit也是不变的，只有两字节会变。

​	攻击 Non-ASLR 的镜像或 top down alloc(win7) 

​	堆喷定位内存：申请很大的空间去放shellcode，通常大小在200M左右，主要目的是覆盖0x0c0c0c0c。shellcode前也是填充大量的0x0c0c0c0c。0x0c0c这条指令对应的汇编指令是or al,0ch。这条指令与0x90的NOP指令类似。有文章说堆喷要求攻击者能够控制单次申请的堆块的大小，所以基本上只有在支持动态语言脚本（如JavaScript，ActionScript等）的程序中才能使用

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/windwos4.png)

### Control	Flow Guard 

所有的直接call都会被CFG检查，有一个预先设置的read-only的bitmap。

编译器将间接函数调用的目的地址保存在GuardCFFunction Table中。在执行间接调用函数之前，会检查跳转的这些地址是否存在于表里

攻击vtable已经是历史了

绕过方式：

​	覆盖CFG不保护的return address, SEH handler 等

​	覆盖CFG	disabled	module 

​	COOP++ 

### 栈

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/windows5.png)

#### GS

类似于Stack Canery

绕过方式

​	利用未被GS保护的内存模块：GS机制只有在缓冲区大小大于4字节的函数中才存在，可以寻找缓冲区大小不大于4字节的函数

​	覆盖虚函数表：

​	覆盖SEH(x86) ：这个位置在GS上方

​	替换掉.data中的cookie值：将.data段的备份数据也改成用来覆盖SecurityCookie的值，那么就可以绕过检测

​	Stack underflow 

​	非线性写

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/windwos6.png)

#### SafeSEH(x86)

在call异常处理handler之前检查handler是否合法

绕过方式：

​	覆盖一个在有SEH但没启用SafeSEH的镜像里的handler

​	利用加载内存模块外的地址：程序加载时，内存除了exe和dll外还存在其他的映射文件，异常处理函数指针指向这些地址时不会进行有效性验证，如果找到jmp esp之类的指令就很好控制流程（CFG呢？存疑）

​	覆盖虚函数表

​	使用堆地址覆盖SEH：SafeSEH允许其异常处理句柄位于除栈空间之外的非映像页面，将shellcode写在堆空间中 ，再覆盖SEH链表的地址。使程序异常处理句柄指向堆空间，就可以绕过SafeSEH的检测了。

 ![](https://raw.githubusercontent.com/ble55ing/PicGo/master/windows7.png)

#### SEHOP(x86)

```
  1.所有SEH结构体必须存在栈上
  2. 所有SEH结构体必须四字节对齐
  3.所有SEH结构体中处理异常的函数必须不在栈上
  4.检测整个SEH链中最后一个结构体，其next指针必须指向0xffffffff，且其异常处理函数必须是
  ntdll!FinalExceptionHandler
  5.攻击者将SEH指针劫持到堆空间中运行shellcode。
  6.有了SEHOP机制以后，由于ASLR的存在，攻击者很难将伪造的SEH链表的最后一个节点指到
  ntdll!FinalExceptionHandler上所以在检测最后一个节点的时候会被SEHOP机制发现异常
```

绕过方式：

​	泄露栈信息并修复SEH chain

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/windows8.png)

### 堆

#### Windows堆排布

增长方式由低地址到高地址。

块表位于堆区的起始位置，用于索引堆区中所有堆块的信息

	##### 空表

按照堆块的大小不同，空表分为128条；其中空表索引的第一项是所有大于等于1024字节的堆块。

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20190830222244.png)

	##### 快表

加速堆块分配采用的策略，这类单向链表不会发生堆块合并；快表初始化为空，每条快表最多只有4个结点 

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20190830222547.png)

##### 分配模式

- 快表分配：寻找 -> 将其改为占用态 -> 从堆表中卸下 -> 返回一个指针指向自身块身给程序使用

- 普通空表分配：使用最小满足来寻找最优的空闲块分配

- 0号空表分配：反向查找，先找最大，然后看是否满足，再找最合适的

- 释放时，将状态改位0（0x8,0x9和0x0？）

- 合并时，将两个块从空闲链表卸下，调整块信息，将新块链入空闲

  ![](https://raw.githubusercontent.com/ble55ing/PicGo/master/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20190830223519.png)

  堆分配的函数，LocalAlloc()，GlobalAlloc()，HeapAlloc()，malloc()，都是使用ntdll.dll里的RtAllocateHeap()函数进行的分配（并不是具体实现，而是在二进制程序里就是调的RtAllocateHeap()函数）

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/windows9.png)

#### DWORD SHOOT 

发生在堆块的释放时，由于是双向链表，所以就会在自己bk的fd写入fd的值，在fd的bk写入bk的值

常用的目标：

​	内存变量：能影响程序执行的重要标志位
	代码逻辑：修改代码段的关键逻辑部分，比如NOP掉
	函数返回地址：修改函数返回地址来劫持进程（不固定）
	攻击异常处理机制：当产生异常时，会转入异常处理机制，将里面的数据结构作为目标
	函数指针：调用动态链接库的函数，C++中的虚函数调用
	PEB线程同步函数的入口地址：每个进程的PEB都存放着一对同步函数指针，指向RtlEnterCriticalSection()和RtlLeaveCriticalSection()，并且进程退出的时候会被ExitProcess()调用（固定PEB的地址不会变）

注意：

​	shellcode中修复堆区和DF的值

​	使用跳板进行定位：call DWORD PTR [EDI+0x78] call DWORD PTR [EBP+0x74] 这种的

​	指针反射：如果将shellcode地址写入RtlEnterCriticalSection()的指针位置，这个指针位置也会被写入shellcode+4的位置，注意避免

#### Metadata check & hardening

几乎不可能攻击堆的meta-data

安全的unlink

Replace	lookaside	lists	with	LFH 

Heap	cookies	&	Guard	pages ：

​	Heap	cookies	are	checked	in	some	places	such	as	entry	free 

​	Zero	Permission	Guard	pages	after	VirtualAlloc	memory 

Metadata	encoding 

Pointer	encoding 

​	Almost	all	function	pointer	are	encoded	such	as	VEH,	UEF,	CommitRoutine,	etc.绕过方式：

​	溢出用户数据

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/windows10.png)

#### LFH	allocation randomization 

GetNextFreedLFHblock(random_start_index)    

绕过方法：

​	allocate	LFH	unhandled	size(larger	than	0x4000) 

​	allocate	LFH	disabled	size(specific-sized LFH	will	enable only if allocation times exceeded	some	threshold)    

​	堆喷、爆破

#### VirtualAlloc	randomization 

Ptr=VirtualAlloc(size+random),	return	ptr+random   

### Windwos上的利用方式

#### Linux的信息泄露方式在Windows上的用法

没PIE二进制程序的基址：二进制程序的基址每次系统重启才更新

泄露加载库的地址：OK，GOT/GOT_PLT 即IAT    仍是可读的

动态链接相关技术如DYNELF,ret2dlresolve   

​	No	lazy	binding,	Ret2dlresolve	related	techniques	are	unavailable •

​	IAT	EAT	are	readable,	DYNELF-like	things	are	still	available    

#### Linux的控制流劫持方式在Windows上的用法

GOT表覆写 ：不可能，IAT是只读的

插入的函数指针覆写：IO_FILE_JUMP   ，free	hook   等，：

​	困难，一些函数指针被加密或移除了UEF VEH encoded,	PEB	 RtlEnterCriticalSection, RtlLeaveCriticalSection Removed    。

​	SEH handler还能用

vtable覆写：难，CFG限制覆写的值为函数开始。

返回地址的非线性写：OK

使用函数指针进行覆写：OK

#### Linux的利用方式在Windows上的用法

堆攻击方式：off-by-one,	house	of	xxx,	xxxbin attack    

堆操作：堆风水等；A	little hard due to	LFH	allocation randomization   

栈Canery 覆写：Stack	cookie	on	.data section	and	writeable    

#### Linux的exp生成方式在Windwos上的应用

ROP：有些难，indirect	calls	are	protected	by	CFG    

Disable	DEP	via mprotect like function    ：OK,	VirtualProtect	on	windows    

System	call	style shellcode    ：Hard,	Windwos系统调用不被好好的被文档记录而且每个版本不同

#### Windwos特有的利用技巧

#### SEH结构

```
struct VC_ EXCEPTION REGISTRATION{
	vC_ EXCEPTION_ REGISTRATION* prev;
	FARPROC handler;
	scopetable_ entry* scopetable; / /指向scopetable数組指針
	int_ index; / /在scopetable_ entry中索引
	DWORD_ ebp; //当前EBP 値
}
```

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20190830215823.png)

#### 通过SEH绕过GS

SEH会在有try...except结构的时候，一个VC_EXCEPTION_REGISTRATION结构被压入栈。覆写handler然后触发exception就能劫持控制流。

绕过SafeSEH：覆盖一个镜像的handler，要有SEH而没有safeSEH(only	way,	 see	ntdll.dll!RtlIsValidHandler)    

绕过SEHOP：泄露栈地址然后修复SEH chains。

有点难，但不是不行。

#### x86地址爆破

反正就8位，3位固定的

#### 跨进程泄露

一些内核相关dll如ntdll.dll，kernel32.dll，是全进程共享库的基址的。

#### 同进程泄露

镜像加载基址只有在系统重启是才会改变。

### 参考资料

Atum的WinPWN分享

<https://blog.csdn.net/m0_37809075/article/details/82849388> 