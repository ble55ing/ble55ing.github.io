---
layout: post
title:  "从零开始aeg"
categories: paper
tags: paper DTA
author: ble55ing
---

* content
{:toc}
## 环境配置

各种环境的配置，起源于强网杯的babyaeg，另外可能考虑做一下之后强网杯的AI aeg。

### 环境一

#### pypy

pypy是一个高效的python脚本优化器。

sudo apy-get install pypy

然后pypy使用的包和python是独立的，所以需要把python的支持包拷贝一份到pypy，位置是

```
cp -R /usr/local/lib/python2.7/dist-packages /usr/local/lib/pypy2.7/
cp -R /usr/local/lib/python2.7/site-packages /usr/local/lib/pypy2.7/
```

这样还是不够的，总需要有安装新的第三方库，因而要给pypy安装pip，安装方法为pypy get-pip.py，get-pip.py的地址为****地址，安装完之后pip就只给pypy安装了。有源码的可以通过pypy setup.py install的方式进行，



#### elftools和capstone

可以通过这两个工具进行python的二进制程序分析，诸位爆破。

## 环境二

sad

### 使用angr解决问题

cannot import name '_psutil_linux' 可以 pip install --ignore-installed psutil 更新一下

No module named six  可以pip install matplotlib解决

其他就直接pip install 就成

然后运行报错， Cannot find radare2 in PATH

需要apt-get update，然后apt-get install radare2.

遇到angr的CFGFast KeyError 34494976L错误，没解决，决定跳过这部分

使用angr感觉会慢一点，可能5s真的有点不够了。。