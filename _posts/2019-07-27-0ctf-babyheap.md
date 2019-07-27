---
layout: post
title:  "cybricsctf Tone"
categories: ctf
tags: cybricsctf misc
author: ble55ing
---

* content
{:toc}

## 0ctf2019的babyheap--exhaust topchunk

这道题是耗尽topchunk，当top chunk不足以分配需要的size时，会触发malloc_consolidate ，然后加以利用的。

### 题目描述

存在offbynull漏洞，libc 2.28

### 题解

#### exhaust topchunk

首先先耗尽topchunk，申请一个块，将topchunk末位置为0，这样，由于有tcache，0x28，0x38，0x48，0x58先申请完填满tcache。

接下来申请删除就会进fastbin了，注意不要释放最后一块。

#### malloc_consolidate

功能有二：

​	检查fastbin并进行初始化；

​	如果fastbin初始化，则按照一定的顺序合并fastbin中的chunk放入unsorted bin中。

具体如下：

1. 判断fastbin是否初始化，如果未初始化，则进行初始化然后退出。

2. 按照fastbin由小到大的顺序（0x20 ,0x30 ,0x40这个顺序）合并chunk，每种相同大小的fastbin中chunk的处理顺序是从fastbin->fd开始取，下一个处理的是p->fd，依次类推。

3. 首先尝试合并pre_chunk。

4. 然后尝试合并next_chunk：如果next_chunk是top_chunk，则直接合并到top_chunk，然后进行第六步；如果next_chunk不是top_chunk，尝试合并。

5. 将处理完的chunk插入到unsorted bin头部。

6. 获取下一个空闲的fastbin，回到第二步，直到清空所有fastbin中的chunk，然后退出。

   如参考资料一中所说。

   尝试将topchunk耗尽，看fastbin的合并。

   ![](https://raw.githubusercontent.com/ble55ing/PicGo/master/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20190727083128.png)

   一直申请0x30大小的块直到耗尽，然后将编号1-9的块删除，保留最前和最后两个块，此时topchunk为0x20大小。申请0x40的块，触发合并。

   合并、申请完之后，最后一块的prev_size是0x140，然后再利用offbynull将0x140块变为0x100块，然后再在里边申请几个块，把第一个块释放掉，这样在再触发malloc_consolidate时就会将这个块0x140与最后的0x30合并，就构成了overlap。但是这个第一个块是不能在fastbin里的，要在unsortedbin，所以必须先将这个块和之前的一些块合并，触发一次malloc_consolidate再将前面的空间分配出去才行。

   由于有个overlap，所以可以通过题目给的View函数泄露libc了。

   #### 劫持topchunk

   在main_arena+96的地方存着topchunk的位置，通过0x56的堆头地址得到其之前的一个位置（fastbin），然后修改这个topcuhunk的地址，到malloc_hook前面一点的地方，即IO_wide_data_0+301处，找到一个0x7f，将其当做topchunk，然后再申请时切割topchunk就能修改malloc_hook了。当然这题还需要realloc中转一下。

   

### 参考资料

<https://www.anquanke.com/post/id/176139> 