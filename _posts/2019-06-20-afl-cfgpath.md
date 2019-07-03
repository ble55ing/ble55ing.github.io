---
layout: post
title:  "根据cfg分析afl的路径信息"
categories: afl
tags: afl
author: ble55ing
---

* content
{:toc}
## 根据cfg分析afl的路径信息

前文使用angr获取了二进制程序的cfg图，接下来要依据控制流图，提取出afl的路径信息。

前文链接：<https://ble55ing.github.io/2019/06/18/angr-cfg/>

### 先提取普通的cfg图

首先还是要使用angr进行分析，得到描述cfg的dot文件

然后要对dot文件进行处理，提取有用的部分。有用的部分分为三种：当前处理基本块的id，基本块间的上下文关系和基本块中的汇编指令。前两项是要记录的，后面的汇编指令要处理。

处理的部分首先是获得各个函数的地址和名称，以及main函数的id。（这个后面感觉好像没什么用）

然后获取哪些基本块中被afl插入了以及其随机数。并将所有是afl添加进来的块的id记录下来，以便在后面将afl中的互相调用删去（不对这里好像有问题）仔细分析了一下angr生成的cfg描述，精简的解决方式为以```__afl_maybe_log```为目标的分支直接删去，因为angr是为每个调用函数都生成一个分支的，我们不用去考虑调用```__afl_maybe_log```之后在其函数内部的逻辑，而且最终这个分支还是会回去的。所以没问题。

然后就是顺着每一条路径，找到其中被afl插入的位置，按照AFL的分支计算逻辑来计算一下分支的情况。改天再验证一下出来的结果是否正确。

额复杂程序分析时内存不够你敢信。。OSError: [Errno 12] Cannot allocate memory

额然后虚拟机就启动不起来了```[/dev/sda1: clean, */* files, */* block```，Ctrl Alt +F2进命令行看看。```df -h```。竟然空间全占满了。删一个文件然后```sudo rm -rf ~/.local/shared/Trash/*```。reboot重启，等以会儿。。好了。

但这样下去就没法运行了。。只好新建一个虚拟机，待续

### 后文链接
<https://ble55ing.github.io/2019/07/03/afl-cfgpath2/>

