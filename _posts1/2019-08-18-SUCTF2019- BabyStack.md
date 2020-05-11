---
layout: post
title:  "SUCTF2019 Windows BabyStack"
categories: ctf Windows pwn
tags: pwn Windows
author: ble55ing
---

* content
{:toc}
## SUCTF2019 Windows BabyStack

明明是safeSEH的题目，竟然叫BabyStack，栈溢出都没啥用。。

### 题目分析

主函数有栈溢出，但是是exit的所以不能控制ret。

程序本身是32位程序，很大，分析起来很困难。

### 漏洞分析

#### 除零错误

主函数中有一处问题代码，在0x408552处，F5不会显示，这个代码会将输入减去此时栈顶数据然后用栈顶数除以这个差，所以会存在除0错误

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20190823234039.png)

然后会触发异常处理函数

#### 异常处理函数

异常处理函数在0x407f60，触发除零错误会调用。

在x407f81处，函数一开始设置了___security_cookie，将一个全局变量异或ebp-8放入ebp-8，再异或ebp放入ebp-0x1c作为canery。

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20190823234123.png)

异常处理函数中，当输入为非yes也非no的值的时候，会触发能够在栈上写数据的函数，存在栈溢出。

异常处理函数中有打印flag的语句，但是需要1+1=3，而这两个数都是写死的，也溢出不到。

### 漏洞利用

除0错误只要输入与减去的值相同的值就可以了。

异常处理函数中能够获取任意地址数据，和栈溢出。

考虑在栈上布置一个SEH结构体，然后在任意地址读的时候读一个错误地址，触发异常执行get flag的语句。

构造方式为：从ebp-0x1c开始为：

### SEH利用

```
payload += p32(cookie1)
payload += p32(getflag)
payload += "C"*0x4
payload += p32(next_SEH)
payload += p32(this_SEH_ptr)
payload += p32((buffer_start + 8)^security_cookie)
```

其中，nextSEH和this_SEH_ptr的值不要动，cookie1和fake_struct_addr在本题中都是异或的全局变量，同样也要异或。注意最后程序会走到这个getfalg的位置getflag，而不是SEH里的。。？不知道为什么。

### SEH结构体构造

关于fake_struct的构造：

```
payload = ""
payload += p32(0xFFFFFFE4)
payload += p32(0)
payload += p32(0xFFFFFF0C)
payload += p32(0)
payload += p32(0xFFFFFFFE)
payload += p32(mov_eax1_ret)
payload += p32(getflag)
```

（这个构造好像毫无用处啊。。）把最后俩值改成“CCCCCCCC”也没事。。

