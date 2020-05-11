---
layout: post
title:  "IDA Python实现控制流图生成"
categories: CodeAnalyze
tags: elf CodeAnalyze
author: ble55ing
---

* content
{:toc}
## IDA Python实现控制流图生成 

基于IDA Python的控制流图生成方案。

IDA Python生成的控制流图，其粒度大小与AFL识别的粒度大小基本相同。angr的粒度相比之下要小得多。因此angr所消耗的资源会更多，对于AFL插桩后的程序无法较好的完成控制流图 生成工作，因此尝试使用IDA Python进行控制流图的生成（AFL插桩后的程序大小竟然会扩到约10倍，有点恐怖）

#### IDA Python 介绍

IDA Python有三个库，其中提供一些API来提供一些功能

~~~
import idaapi
import idautils
import idc
~~~

使用IDA 打开程序后，按下Alt +F7选择脚本运行。

#### 需求功能

其实只要得到所有函数的F12的图的文件就可以了，那是WinGraph32的控制流图表示文件，用这个文件来得到控制流图会更简洁。。然后没找到批处理的方法。

所以就自己写了一个，处理跳转然后建立基本块间的可达关系然后得到控制流图。

### 附上依序处理基本块的代码

```
def main():

    start_time = time.time()

    all_funcs = idautils.Functions()
    GetDeclareFuncName()
    ii=0;
    for fn in all_funcs:
        ii+=1
        func_name = idc.GetFunctionName(fn)
        fflags = idc.GetFunctionFlags(fn)
        if func_name not in DeclareFuncName:
            continue
        #if func_name.find("main")!=0 or len(func_name)!=len("main"):
        #    continue

        fout = open("out.txt", "a")
        fout.write("FuncName: "+func_name +"\r\n")
        fout.close()
        DeclareFuncSit.append(blocknum)

        f_blocks = idaapi.FlowChart(idaapi.get_func(fn), flags=idaapi.FC_PREDS)
        
        for block in f_blocks:
            dealblocks()
```



#### 遇到的问题

##### 如何解决函数调用的问题

按照正常程序执行的过程进行，在函数中记录一个表示所有正常return的基本块的list，在函数调用结束后，根据这个list生成函数退出时，到达该基本块后一个基本块的可达关系。

##### jmp eax

关于jmp eax 这样的指令，在静态分析的时候不能很好的解析她接下来可能的跳转关系。可能可以做一个备选跳转路径的表？就是记录所有含有这种间接跳转的基本块的AFL桩点，然后生成这些基本块与所有可达基本块间的路径。图如果在程序的执行过程中，有出现这些路径中的一条，则将其也添加到可达路径的序列中。





