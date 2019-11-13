---
layout: post
title:  "De1CTF unprintable分析"
categories: ctf pwn
tags: pwn
author: ble55ing
---

* content
{:toc}
## De1CTF unprintable分析

这题很有意思，相对巧妙，比赛时没有想到最后，现在复现一下

### 题目介绍

禁用了标准输出流，给了栈地址，有格式化字符串漏洞

### 劫持控制流--exit

在exit中会调用```_di_fini```，而在其中会有靠偏移来进行调用的位置，而这个位置是可以用的

```
   0x7f0bbf8e9dc4 <_dl_fini+788>    add    r12, qword ptr [rbx] <0x600dd8>
   0x7f0bbf8e9dc7 <_dl_fini+791>    mov    rdx, qword ptr [rax + 8]
   0x7f0bbf8e9dcb <_dl_fini+795>    shr    rdx, 3
   0x7f0bbf8e9dcf <_dl_fini+799>    test   edx, edx
   0x7f0bbf8e9dd1 <_dl_fini+801>    lea    r13d, [rdx - 1]
   0x7f0bbf8e9dd5 <_dl_fini+805>    je     _dl_fini+832 <0x7f0bbf8e9df0>
 
   0x7f0bbf8e9dd7 <_dl_fini+807>    nop    word ptr [rax + rax]
   0x7f0bbf8e9de0 <_dl_fini+816>    mov    edx, r13d
   0x7f0bbf8e9de3 <_dl_fini+819>    call   qword ptr [r12 + rdx*8]
```

在这里，rbx的值在栈上有，所以可以通过格式化字符串漏洞改写其中的值，从而达到控制控制流的目的。此时的r12是buf中的地址，所以可以在buf中一定偏移处写入地址即可。这里将控制流劫持到main函数开始的位置

### 劫持控制流--printf

同时可以使用格式化字符串漏洞修改printf函数返回时的位置，这样就可以在不动栈帧的情况下做利用了（之前有一道题想过这种方法，但当时是栈溢出最后没做成）。

方法是在栈中找到printf的返回地址，然后使用格式化字符串将其改掉，这样在printf返回的时候，就会回到改写后的地址，如改写到printf格式化字符串输入之前，就可以一直使用这个漏洞了。

### 构造ROP-chain

本题目中，每次printf格式化字符串长度不能超过0x2000，所以为了控制栈方便，找栈地址小于0x2000的搞。然后找一个pop rsp;ret的gadget，将其作为printf的返回地址，然后在这个返回地址的下一个8位放上输入的buf的地址，这样在pop rsp;ret后就到了rop chain上。

然后就是没有syscall的问题，可以使用经典64位优秀gadget来更改控制流到一个syscall上，可以使用read读一个0x3b的长度就可以控制rax，然后构造好rdi等参数就ok了

### 获得flag

关了标准控制流，将其重定向到输出流或错误流就可以了

```
cat flag > &0/2
```

### 参考资料

<https://www.jianshu.com/p/9fc6a4e98ecb> 

