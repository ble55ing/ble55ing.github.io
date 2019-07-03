---
layout: post
title:  "starctf2018 babystack"
categories: ctf
tags: starctf pwn
author: ble55ing
---

* content
{:toc}

## starctf2018 babystack writeup

这是一道有趣的题目。一直以来，绕过stack canary是通过逐位爆破的方式进行的，总是想着，是否有一种方法，能够更改stack canary中的值，这样就能够直接绕过stack check了。

这就是这样的一道题目。

### 题目描述

题目本身关了ASLR，并会创建一个线程，线程中会读一个长度然后写入栈中。

很明显会有一个栈溢出。

### 题目调试

使用gdb加载程序，然后b在线程调用的读入数据的函数内部。单步程序，输入过长的数据，查看堆__stack_chk_fail的调用，

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/starctf20181.png)

可以看到这里是存储在fs:[0x28]中。gdb中可以通过info all-registers来查看所有寄存器中的值，但是并没有这个。。这个fs寄存器是由glibc定义的，存放Thread Local Storage （TLS）信息的，该结构体如下所示：

```
typedef struct
{
        void *tcb;        /* Pointer to the TCB.  Not necessarily the
                             thread descriptor used by libpthread.  */
        dtv_t *dtv;
        void *self;        /* Pointer to the thread descriptor.  */
        int multiple_threads;
        int gscope_flag;
        uintptr_t sysinfo;
        uintptr_t stack_guard;   /* canary，0x28偏移 */
        uintptr_t pointer_guard;
        ……
} tcbhead_t;
```

这个stack_guard 是在```__libc_start_main```中进行设置和赋值的，首先以_dl_random这个全局变量为入参（有没有可能得到这个值然后能够绕过canary），生成canary，然后通过THREAD_SET_STACK_GUARD宏将canary赋值给tls的stack_guard变量。 

### 题目解法

在使用pthread时，这个TLS会被定位到与线程的栈空间相接近的位置，所以如果输入的数据过长的话也可以把这里覆盖掉，就可以改掉stack_guard的值了。

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/starctf20182.png)

rcx中是xor之后的值，再异或回去就是了，可以看到原先的stack_guard是在$rbp+2008的位置。

### 扩展研究

stack canery的生成流程，即如何将stack canary的值生成并放入fs:[0x28]的

这一部分是在security_init 函数里，该函数开始的地方位于<dl_main+6725> 。

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/starctf20183.png)

这个值的生成是从```_dl_random```这个随机数生成的，它由```_dl_sysdep_start```函数从内核获取的。```_dl_setup_stack_chk_guard```函数使用这个随机数生成canary值，THREAD_SET_STACK_GUARD宏将canary设置到%fs:0x28位置。 

```
198	/* Set up the stack checker's canary.  */
199   uintptr_t stack_chk_guard = _dl_setup_stack_chk_guard (_dl_random);
200 # ifdef THREAD_SET_STACK_GUARD
201   THREAD_SET_STACK_GUARD (stack_chk_guard);
202 # else
203   __stack_chk_guard = stack_chk_guard;
204 # endif
```

总的来说，kernel初始化了跟TLS相关的寄存器gs, 并 且提供了canary这个随机值, glibc写入%gs:0x14这个保存随机值的位置并且提供变量定义和 打印函数定义, 最上面是gcc插入对canary的值的引用和出错函数到用户代码里. 

### 参考文章

<https://www.cnblogs.com/gm-201705/p/9864080.html> 

<https://hardenedlinux.github.io/2016/11/27/canary.html> 