---
layout: post
title:  "NEUZZ总结性分析"
categories: paper
tags: paper DeepLearning
author: ble55ing
---

* content
{:toc}
## NEUZZ总结性分析

和大佬讨论了一下还做了汇总，需要整理一下想一想下一步该做些什么

## 程序平滑模型

创新性的使用了深度神经网络搭建程序平滑模型，解决了目前使用符号执行时面临的约束求解和梯度爆炸两个开放性难题的问题。

本文使用模型训练的目的是降低损失函数，即降低模拟输出与实际输出之间的距离。

## 梯度引导变异

梯度大的地方，可以认为其变异更容易产生路径的变化，变异方向与梯度的方向相同时，可以更可能的触发新的代码块覆盖。

## 源码实现细节

程序在进行变异的过程中，变异所基于的梯度是各个方向的梯度，即变异为从每次随机选择一个输出神经元开始，计算各个输如字节在该神经元上的梯度，然后根据排序，以2的各个次方的分组进行按组的变异。

## 一些问题

### 这个模型为什么需要afl的样本作为输入

模型的学习是较快的，但模型的增长是相对缓慢的，所以需要预先准备能达到一定覆盖率的样本。

### 模型的输出神经元个数是由什么决定的

模型的输出神经元是在变化的，因而采用参数axis=1的np.unique进行bitmap的整合，能将相关联的基本块整合到一起。由于程序中很多基本块之间的上下文关系都很紧密，所以将这些基本块合并，在接触到其中的区别的时候再分开，可以有效的降低模型训练的难度。因而如果想知道某一基本块是整合后的哪一个块需要用原bitmap进行比对。

### 增量学习的过程

由于某些代码块很显然是会同时出现的，这被称为多重共线性，会阻碍模型的收敛。因此本模型会仅考虑哪些至少激活了一次的边缘，并将其汇总在一起出现的边缘合并为一个边缘，0和0归类到一起，1和1归类到一起。 