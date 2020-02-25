---
layout: post
title:  "strcpy 函数逻辑分析"
categories: linux pwn coding 
tags: pwn
author: ble55ing
---

* content
{:toc}
## strcpy 函数逻辑分析

strcpy函数是string.h中的一个字符串拷贝的函数，估计大家都很常用到就不多说了。

### 分析原因

起因是注意到了这样的情况：如果strcpy(buf,buf+1)，即复制的地址有重叠的话，会出现复制数据的结果有问题的情况，所以就对strcpy的实现方式进行了分析。

注意：本分析中的情况只适用于32位程序，即gcc -m32 11.c -o 11编译出来的程序，而64位编译出来的程序则不会出现这样的问题。具体原因会在下面进行说明。

### 情况描述

```
// gcc -m32 11.c -o 11 ok 234678901234567789012345678901234
// gcc 11.c -o 11 fail    234567890123456789012345678901234
#include <stdio.h>
#include <string.h>

int main()
{
    unsigned char buf[0x100] = "1234567890123456789012345678901234";
    strcpy(buf, buf + 1);
    puts(buf);
    return 0;
}
```

如上代码所示，为32位编译和64位编译的不同结果。

可见32位情况下存在着重叠产生错误的情况。

### 故障分析

32位情况下的执行为：

1、	首先判断字符串长度是否小于16，如果是，则直接进入步骤5；如不是，进入步骤2。

2、	将前16个字符进行复制。

```
0xf7e88ec5 <__strcpy_sse2+181>    movdqu xmm1, xmmword ptr [ecx]
0xf7e88ec9 <__strcpy_sse2+185>    movdqu xmmword ptr [edx], xmm1
```

此时ecx值为0x1c，edx为0x1d，23456789 01234567 78901234 56789012 34

3、	对字符串进行对齐操作，将原字符串的下一个0x10位置找到，作为ecx的值0x20

```
   0xf7e88ef0 <__strcpy_sse2+224>    movdqa xmm1, xmmword ptr [ecx]
 ► 0xf7e88ef4 <__strcpy_sse2+228>    movaps xmm2, xmmword ptr [ecx + 0x10]
   0xf7e88ef8 <__strcpy_sse2+232>    movdqu xmmword ptr [edx], xmm1
```

然后进行复制，

此时ecx为0x20,edx为0x1f，23467890 12345677 89001234 56789012 34

4、如果其值长度还是超长，则会继续进行复制，这里倒是很安全，和64位的处理方式是一样的，把新值放入新的寄存器，然后把旧寄存器的值复制过去。

```
   0xf7e88f0f <__strcpy_sse2+255>    movaps xmm3, xmmword ptr [ecx + ebx + 0x10]
   0xf7e88f14 <__strcpy_sse2+260>    movdqu xmmword ptr [edx + ebx], xmm2
```

5、对于字符串最后的部分，用如下的方式将值复制过去。

```
  movlpd  xmm0, qword ptr [ecx]
  movlpd  qword ptr [edx], xmm0
  movlpd  xmm0, qword ptr [ecx+6]
  movlpd  qword ptr [edx+6], xmm0
较短的时候用这个
   0xf7e89150 <__strcpy_sse2+832>    mov    ax, word ptr [ecx]
   0xf7e89153 <__strcpy_sse2+835>    mov    word ptr [edx], ax
 ► 0xf7e89156 <__strcpy_sse2+838>    mov    al, byte ptr [ecx + 2]
   0xf7e89159 <__strcpy_sse2+841>    mov    byte ptr [edx + 2], al
```

可以发现，出现故障的原因是在于在对齐的时候，使用了已在前16字符进行了覆盖的字符，导致这个的覆盖出现了错误。而在64位的系统中，则不存在这种问题，字符串从一开始就是对齐了的，所以就没有问题。

因此，如果强行将字符串不对齐，则也会出现这样的问题，如

```

#include <stdio.h>
#include <string.h>

int main()
{
    unsigned char buf[0x100] = "12345678901234567890123456789012345678";
    strcpy(buf+1, buf + 2);
    puts(buf);
    return 0;
}
```

最终结果是1345678901234568890123456789012345678。

### 进阶扩展——memcpy

memcpy也是有同样的问题，不过这个64位是真没有问题。

但这个对齐操作就很迷了

```
#include <stdio.h>
#include <string.h>

int main()
{
    unsigned char buf[0x100] = "12345678901234567890123456789012345678";
    memcpy(buf+1, buf + 2,35);
    //strcpy(buf+1,buf+2)
    puts(buf);
    return 0;
}
```

strcpy操作：1346789012345678890123456789012345678

memcpy：13457890123456788902345678901234556778

memcpy的这个是怎么处理的，。看不懂是真的。

```
   0xf7f28dcc <__memcpy_sse2_unaligned+76>     movdqu xmm0, xmmword ptr [eax + 0x10]
   0xf7f28dd1 <__memcpy_sse2_unaligned+81>     movdqu xmm1, xmmword ptr [eax + ecx - 0x20]
   0xf7f28dd7 <__memcpy_sse2_unaligned+87>     cmp    ecx, 0x40
   0xf7f28dda <__memcpy_sse2_unaligned+90>     movdqu xmmword ptr [edx + 0x10], xmm0
   0xf7f28ddf <__memcpy_sse2_unaligned+95>     movdqu xmmword ptr [edx + ecx - 0x20], xmm1
```

memcpy的处理逻辑，ecx为长度，所以，memecpy是从头和尾两方向进行的memcpy，先是0x20，复制前0x10和后0x10，然后是这个。因此长度小于0x20时就不会有问题。