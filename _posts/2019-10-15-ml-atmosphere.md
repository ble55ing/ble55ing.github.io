---
layout: post
title:  "机器学习环境安装"
categories: DeepLearning
tags: DeepLearning atmosphere
author: ble55ing
---

* content
{:toc}
## 机器学习环境安装

由于机器学习所用的服务器一般是多人公用，所以很多地方涉及到低权限用户无法操作的问题，因此来记录一下。

### 安装Anaconda

Anaconda就相当于预装了很多环境，所以用来垫基础，这是清华的源

<https://mirrors.tuna.tsinghua.edu.cn/help/anaconda/> 

在其中的bin中有pip，可以将其目录添加到path中

```
export PATH=$PATH:/home/user/anaconda3/bin
source ~/.bashrc
```

### CUDA

<https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&target_distro=Ubuntu&target_version=1404&target_type=runfilelocal> 

chmod +x 然后./运行

只安装toolkit

```
export PATH=$HOME/cuda-10.0/bin:$PATH
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$HOME/cuda-10.0/lib64/
source ~/.bashrc
```

使用cuda-toolkit 看 安没安好

### CUDNN

<https://developer.nvidia.com/rdp/cudnn-download> 

```
cp cuda/include/cudnn.h include/
cp cuda/lib64/libcudnn* lib64/
chmod a+r include/cudnn.h lib64/libcudnn*
cat include/cuann.h | grep CUDNN_MAJOR -A5
```

### tensorflow

pip install tensorflow-gpu.

安装的是python3的版本

### 错误汇总

1. failed to allocate 200.06M (209780736 bytes) from device: CUDA_ERROR_OUT_OF_MEMORY: out of 		memory

   限定使用的GPU

   ```
   import os
   os.environ['CUDA_VISIBLE_DEVICES']='1'
   ```

2. 用不了pip时的替代

   python2.7 -m pip install 可替换pip install。

