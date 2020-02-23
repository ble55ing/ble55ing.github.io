---
layout: post
title:  "十一月CTF部分题目汇总一"
categories: ctf pwn
tags: ctf pwn
author: ble55ing
---

* content
{:toc}
## 十一月CTF线下部分题目汇总

湖湘杯和红帽杯的比赛题目，把做的中比较有意思的记录一下，这个是线下

### 湖湘杯

住的竟然是五星级宾馆，但65元水准的五星级自助餐就。。emmm

#### pwn1 

久违的菜单题，程序除了常规的1234选项外，还藏了一个选项5：

```
        else
        {
          if ( v2 == 4 )
          {
            puts("bye!");
            exit(0);
          }
LABEL_16:
          _fprintf_chk((__int64)stderr, 1LL, (__int64)"unknown options, foolish %s", (__int64)byte_202100);
          fflush(stderr);
```

很关键。这里由于输入的name在byte_201000是0x50，结合前面的 字符正好0x69，能构造offbyone，可以这样构造overlap然后泄露程序libc地址然后fastbin attack。

程序的脆弱点是在这个地方，signed int 的v3存在整数溢出

```
  v3 = getint();
  if ( v3 > 3 )
  {
    fwrite("Invalid person :P\n", 1uLL, 0x12uLL, stderr);
    fflush(stderr);
    exit(0);
  }
  ((void (__fastcall *)(__int64, _QWORD, _QWORD))*(&fastcaller + v3))(
    a1,
    *(_QWORD *)&pointer[4 * v2 + 2],
    (unsigned int)pointer[4 * v2 + 1]);
```

这里有got表的函数的任意调用。注意：fprintf的格式化字符串输出是打在本地的，所以远程打的时候不能用这个泄露地址。这个题目中的got表中的很多函数是chk的版本，所以比如sprintf可以写入一个地址的。可以通过setbuf把stderr里写入堆块开始地址，从而可以和前面的选项5一起使用。

注：经大佬提醒，stderr的输出是默认不显示在远端的，但可以设置让其显示在远端。

### 红帽杯

### pwn2

一个base64，

然后system("/bin/sh")就能getshell，因为远端的defense是空的

这道题是修复方通过改defense来进行防御的

很好的出题思路，不过由于是AWD plus机制，没有起到应有效果

### pwn3

baby vm，之前没做过这种，比赛上强行做的一波，疯狂绕远路。

有一个指令是能putchar，造成任意读， 另一个指令getchar，构成任意写，最后的payload就是发一个字符串。类似'\x03'+'\x77'+p64(0x8048323)+'\x03'+..