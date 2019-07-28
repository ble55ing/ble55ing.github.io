---
layout: post
title:  "如何提升代码的可读性（+）--以python为例"
categories: coding
tags: coding
author: ble55ing
---

* content
{:toc}
## 如何提升代码的复杂性（-）--以python为例

最近总是读一些别人写的源码，也有一些心得，在这篇里记录一下。

### 变量名的命名

变量名的命名会很大的程度的减少代码的可读性，比如下面的代码，将变量命名为ll（小写的L），然后再在后面使用这个变量的时候就会出现情况，当看到循环的时候，首先就会认为这是十一，然后影响对代码逻辑的分析。

![](https://raw.githubusercontent.com/ble55ing/PicGo/master/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20190721232820.png)

这样的情况会出现比如l（小写的L）、I（大写的i）、和数字1的区别，O（大写的o）和数字0之类的。

### 不规范的函数返回值

python中也有很多种变量类型，将所有函数返回值或许多的数据结构都定义为str类型并使用字符串处理的方式编写程序也不是不行，但这样就会使得程序很复杂。

### 变量赋值、浅拷贝、深拷贝

pthon中的变量存储对于每个变量名都是类似于指针的方式，所以如果复制的方式不恰当很可能会导致对原数据一同进行了修改。来看下面的示例：

```python
import copy
list = [0,"000",[0,90,900]]
listcopy1 = list
listcopy2 = copy.copy(list)
listcopy3 = copy.deepcopy(list)
print(list)
listcopy1[0] = 1
listcopy1[1]="111"
listcopy1[2][0]=1
print(list,listcopy1)
listcopy2[0] = 2
listcopy2[1]="222"
listcopy2[2][0]=2
print(list,listcopy2)
listcopy3[0] = 3
listcopy3[1]="333"
listcopy3[2][0]=3
print(list,listcopy3)
#结果
[0, '000', [0, 90, 900]]
[1, '111', [1, 90, 900]] [1, '111', [1, 90, 900]]
[1, '111', [2, 90, 900]] [2, '222', [2, 90, 900]]
[1, '111', [2, 90, 900]] [3, '333', [3, 90, 900]]
```

有一个叫做copy的库，可以用来做拷贝；可以看到直接赋值是指针赋值，只变了地址；copy()是浅拷贝，变量其中包含的变量还是指针；deepcopy()是深拷贝，完全一份新的。

