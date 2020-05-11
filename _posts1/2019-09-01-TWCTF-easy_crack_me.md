---
layout: post
title:  "TWCTF的Easy_crack_me"
categories: ctf
tags: twctf aeg
author: ble55ing
---

* content
{:toc}
## TWCTF的Easy_crack_me

Tokyo Western的CTF，一道逆向题，队里没人做就给做了。

使用的是Z3求解器。。做完了和大佬说大佬推荐Sage。。下次再学。

### 使用注意

* 多解的情况无法将多解显示出来，需要逐步添加条件
* 没有能够限制某字符出现次数的方法，目前采用的方法是计算乘积。比如'a'有3个，'b'有2个，c有'1'个，```solver.add(12==(flag[0]-96)*(flag[1]-96)*(flag[2]-96)*(flag[3]-96)*(flag[4]-96)*(flag[5]-96)```
* 乘积的值大于了位数表示最大值后就没有意义了，如8位是255（就是这条限制条件相当于没写）

### 题目描述

flag是32位的16机制字符串，这个比赛的flag都是这个样子。

程序中加了好多的的限制条件，如每个数字的出现次数、部分位置的加和结果、部分位置的异或结果等

### 解法

使用Z3约束求解，由于是字符串所以使用了BitVec，（这么说难道用Int就可以进行正常的乘法运算了？，好像有点道理）

```
from z3 import *

flag = [BitVec('flag%d' % i,8) for i in range(32)]
solver = Solver()
ccc = [48,49,50,51,52,53,54,55,56,57,97,98,99,100,101,102]
mc = [0,1,3,8,11,12,15,18,20,21,26,30]
mi = []
f40 = [0x15e,0xda,0x12f,0x131,0x100,0x131,0xfb,0x102]
f60 = [0x52,0xc,0x1,0xf,0x5c,0x5,0x53,0x58]
f80 = [0x1,0x57,0x7,0xd,0xd,0x53,0x51,0x51]
fa0 = [0x129,0x103,0x12b,0x131,0x135,0x10b,0xff,0xff]
fnum = [3,2,2,0,3,2,1,3,3,1,1,3,1,2,2,3]
#54,57,97,99
mf40 = [BitVecVal(i,8) for i in f40]
mf60 = [BitVecVal(i,8) for i in f60]
mf80 = [BitVecVal(i,8) for i in f80]
mfa0 = [BitVecVal(i,8) for i in fa0]
mfnum = [BitVecVal(i,8) for i in fnum]
mccc = [BitVecVal(i,8) for i in ccc]

xx = 1
for i in range(11,16):
	for k in range(fnum[i]):
		xx*=ccc[i]
xx/=102
print xx
print 97*98*98*98*99*100*100*101*101*102*102
print 97+98+98+98+99+100+100+101+101+102+102
print 48*3+49*2+50+52*2+53+54*1+55*2+56*2+57

solver.add(flag[31]==53)
solver.add(flag[1]==102)
solver.add(flag[5]==56)
solver.add(flag[6]==55)
solver.add(flag[17]==50)
solver.add(flag[25]==52)
for i in range(32):
	solver.add(flag[i]!=51)
	if i in mc:
		solver.add(flag[i]<=102)
		solver.add(flag[i]>=97)
	else:
		solver.add(flag[i]<=57)
		solver.add(flag[i]>=48)
for k in range(0,7):
	solver.add(mf40[k]==flag[4 * k + 0]+flag[4 * k + 1]+flag[4 * k + 2]+flag[4 * k + 3])
	solver.add(mf60[k]==flag[4 * k + 0]^flag[4 * k + 1]^flag[4 * k + 2]^flag[4 * k + 3])
	
	#solver.add(v11 == f60[k])
for k in range(0,7):
	solver.add(mfa0[k]==flag[k + 0]+flag[k + 8]+flag[k + 16]+flag[k+24])
	solver.add(mf80[k]==flag[k + 0]^flag[k + 8]^flag[k + 16]^flag[k+24])

m1096 = BitVecVal(1096,8)
m782 = BitVecVal(782,8)
m1160 = BitVecVal(1160,8)
m96 = BitVecVal(96,8)

solver.add(m1096==flag[0]+flag[3]+flag[8]+flag[11]+flag[12]+flag[15]+flag[18]+flag[20]+flag[21]+flag[26]+flag[30])
solver.add(m782==flag[2]+flag[4]+flag[7]+flag[9]+flag[10]+flag[13]+flag[14]+flag[16]+flag[19]+flag[22]+flag[23]+flag[24]+flag[27]+flag[28]+flag[29])

solver.add(m96==(flag[8]-97)*(flag[11]-97)*(flag[12]-97)*(flag[15]-97)*(flag[26]-97)*(flag[20]-97))
#1,5,6,17,25,31
#solver.add(flag[0]==100)
#solver.add(flag[3]==98)
#solver.add(flag[21]!=102)
#solver.add(flag[30]!=97)
#solver.add(flag[18]!=102)
solver.add()
#solver.add(flag[2]==50)

solver.add(m1160==flag[0]+flag[2]+flag[4]+flag[6]+flag[8]+flag[10]+flag[12]+flag[14]+flag[16]+flag[18]+flag[20]+flag[22]+flag[24]+flag[26]+flag[28]+flag[30])
if solver.check() == sat:
    m = solver.model() 
    s = []
    char =""
    for i in range(32):
        s.append(m[flag[i]].as_long())
        char+=chr(m[flag[i]].as_long())
    print s
    print char
    #print(bytes(s))
else:
    print('unsat') 


```



