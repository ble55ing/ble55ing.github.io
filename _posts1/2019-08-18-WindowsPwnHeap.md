---
layout: post
title:  "Windows Pwn Heap 分析"
categories: Windows pwn 
tags: pwn
author: ble55ing
---

* content
{:toc}
## Windows Pwn 堆原理分析 

### Windwos 堆块格式

### Heap结构

每个HEAP有一个HEAP结构，一个heap结构有多个heap_segment 

heap结构在每个堆的起始地址，由每个堆的0号堆段和自己的结构拼接而成，由图可见堆段大小为0x40，前0x40字节属于0号堆段。 0x40之后的heap结构用来保存堆的资产及必要信息。  

```
+0x040 Flags : 2
+0x044 ForceFlags : 0
+0x048 CompatibilityFlags : 0
+0x04c EncodeFlagMask : 0x100000
+0x050 Encoding : _HEAP_ENTRY
+0x058 Interceptor : 0
+0x05c VirtualMemoryThreshold : 0xfe00
+0x060 Signature : 0xeeffeeff
+0x064 SegmentReserve : 0x1fd0000
+0x068 SegmentCommit : 0x2000
+0x06c DeCommitFreeBlockThreshold : 0x800
+0x070 DeCommitTotalFreeThreshold : 0x2000
+0x074 TotalFreeSize : 0x3c440
+0x078 MaximumAllocationSize : 0xfffdefff
+0x07c ProcessHeapsListIndex : 1
+0x07e HeaderValidateLength : 0x248
+0x080 HeaderValidateCopy : (null)
+0x084 NextAvailableTagIndex : 0
+0x086 MaximumTagIndex : 0
+0x088 TagEntries : (null)
+0x08c UCRList : _LIST_ENTRY [ 0x5a4ffe8 - 0x5a4ffe8 ]
+0x094 AlignRound : 0xf
+0x098 AlignMask : 0xfffffff8
+0x09c VirtualAllocdBlocks : _LIST_ENTRY [ 0x71009c - 0x71009c ]
+0x0a4 SegmentList : _LIST_ENTRY [ 0x710010 - 0x5760010 ]
+0x0ac AllocatorBackTraceIndex : 0
+0x0b0 NonDedicatedListLength : 0
+0x0b4 BlocksIndex : 0x00710260 Void
+0x0b8 UCRIndex : (null)
+0x0bc PseudoTagEntries : (null)
+0x0c0 FreeLists : _LIST_ENTRY [ 0x3c2b140 - 0x59ed748 ]
+0x0c8 LockVariable : 0x00710248 _HEAP_LOCK
+0x0cc CommitRoutine : 0x1bb2ba92 long +1bb2ba92
+0x0d0 StackTraceInitVar : _RTL_RUN_ONCE
+0x0d4 FrontEndHeap : 0x00150000 Void
+0x0d8 FrontHeapLockCount : 0
+0x0da FrontEndHeapType : 0x2 ‘’
+0x0db RequestedFrontEndHeapType : 0x2 ‘’
+0x0dc FrontEndHeapUsageData : 0x007161b0 -> 0
+0x0e0 FrontEndHeapMaximumIndex : 0x802
+0x0e2 FrontEndHeapStatusBitmap : [257] “???”
+0x1e4 Counters : _HEAP_COUNTERS
+0x240 TuningParameters : _HEAP_TUNING_PARAMETERS
+0x040 Flags 堆的flag由heapcreate时的flag参数控制，其中HEAP_GROWABLE（0x2）属性是默认的。且私有堆的flag要 or 0x1000.
创建时参数为4，flag为 2or 4 or 0x1000=0x1006
+0x050为之后解密HEAP_ENTRY头部的密钥
+0x060 Signature : 0xeeffeeff heap结构头签名
+0x078 MaximumAllocationSize 允许分配的最大空间，由于heapcreate时参数选择了0，这里允许分配整个地址空间大小。
+0x07c heap在peb堆数组中的索引
+0x074 TotalFreeSize : 0x3c440 全部free堆块的总大小。
+0x07e HeaderValidateLength heap结构头的大小
+0x0a4 SegmentList 堆段的链表，前向指针指向0号堆段，后向指针指向最后一个堆段。
+0x0c0 FreeLists 保存整个堆的空闲堆块，所有堆块的空闲堆块的集合
```



#### Heap Segment

每个程序会有多个Heap segment，当堆的大小不足以分配时，创建新的堆段分配新的空间。堆段通过链表进行连接。 

heap_segment分配在堆里，所以整体也算是一个HEAP_ENTRY，前8个字节为HEAP_ENTRY头部。 

堆段的签名为0Xffeeffee 

偏移为0x10为堆段的链表。

 0x18为堆段隶属的heap。

 0x20为堆块分配的页的数目。当需要分配的空间大小大于这个页大小时会创建一个新的heap_segment并链入链表中。 

FirstEntry，LastValidEntry 分别为堆段中第一个，及最后一个HEAP_ENTRY结构。 

堆段中主要保存堆段的起始及范围。堆段隶属的heap，堆段的链表。 

#### Heap Entry

类似linux 的chunk头，有与HEAP结构0x50偏移处八个字节的异或。

一共8个字节，解密后得到的8个字节中：

前两个字节是大小，单位是8个字节

第三个字节是标志位，*0×01 该块处于占用状态；0×02 该块存在额外描述；0×04 使用固定模式填充堆块；0×08 虚拟分配；0×10 该段最后一个堆块* 

第四字节是用于检测堆有效性的cookie

第五六字节为上一个块的大小

第7字节 堆块所在段的序号，未验证 

第8字节为UnusedBytes，未用到的字节，如分配出来的堆块头部及最后的16字节填充未使用故，第八字节为0x18. 

### Windows堆排布

增长方式由低地址到高地址。

块表位于堆区的起始位置，用于索引堆区中所有堆块的信息

#### 空表

按照堆块的大小不同，空表分为128条；其中空表索引的第一项是所有大于等于1024字节的堆块。

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20190830222244.png)

#### 快表

加速堆块分配采用的策略，这类单向链表不会发生堆块合并；快表初始化为空，每条快表最多只有4个结点 

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20190830222547.png)

#### 分配模式

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

### 参考资料

<https://www.pwndog.top/2018/08/08/windows%E4%B8%8B%E5%A0%86%E7%BB%93%E6%9E%84%E8%B0%83%E8%AF%95%E7%AC%94%E8%AE%B0/> 



