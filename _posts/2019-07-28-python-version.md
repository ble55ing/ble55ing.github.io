---
layout: post
title:  "python 2.7到3.7的升级之路"
categories: coding
tags: coding
author: ble55ing
---

* content
{:toc}
## python 2.7到3.7的升级之路

2020年1月1日起，python2.7就停止维护了。虽然是好多功能和库并没有完全支持python3的今天，也还是来学习一下如何将python2.7修改为python3的程序

### 运行环境

python 3的安装包先在官网[https://www.python.org/downloads/](https://link.jianshu.com/?t=https%3A%2F%2Fwww.python.org%2Fdownloads%2F)下载了，然后安装。安装时注意一下选项，改一下安装位置，把环境变量啥的添上。然后将其安装目录下的python.exe更名为python3.exe，就可以用了。

使用的编译器是pycharm，在file->setting里换编译器。

然后更新一下pip，毕竟之后要安装库的

```
python2 -m pip install --upgrade pip --force-reinstall
python3 -m pip install --upgrade pip --force-reinstall
```

之后要安装什么库的时候，python -m pip install xxx或者python3 -m pip install xxx，感觉比pip3稳定（也可能是我pip3太久没更新了），所有库python3和python2都是要分开装的

### 语法变更

#### print 要加括号，exec也是 

#### raw_input()直接写input()

本来的raw_input能够进行格式的预测，如默认int，有小数点是float，有引号是str，input默认全为str

#### 数字全部为float型

python2中，整数相除为整数，python3中，是真正的除法，全为float型，想达到python2中的```/```的效果，可以用```\\```。

### 新的报错

#### 编码格式

报错，UnicodeDecodeError: 'gbk' codec can't decode byte 0xa2 in position 49: illegal multibyte sequence，需要更改为fr = open(file_path,'r',encoding='UTF-8')

### 参考资料

<https://www.jianshu.com/p/544d8bf98cd8> 