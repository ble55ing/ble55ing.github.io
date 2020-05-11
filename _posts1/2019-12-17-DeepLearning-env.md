---
layout: post
title:  "strcpy 函数逻辑分析"
categories: DeepLearning
tags: DeepLearning
author: ble55ing
---

* content
{:toc}
## tensorflow 2.0如何设置GPU内存占用

主要在于设置GPU内存使用

最近模型总是把内存跑满。。这不行啊。别人没法用了，所以做一些设置。

```
os.environ['CUDA_VISIBLE_DEVICES']='1' 
config = tf.compat.v1.ConfigProto(allow_soft_placement=True)
config.gpu_options.per_process_gpu_memory_fraction = 0.5
config.gpu_options.allow_growth=True 
sess =tf.compat.v1.Session(config=config)
```

从上到下依次是：

设置使用的显卡编号

初始化设置信息

配置gpu最大使用上限

配置动态增长

配置设置

