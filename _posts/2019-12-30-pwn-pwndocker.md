---
layout: post
title:  "pwn docker 搭建"
categories: pwn
tags: atmosphere
author: ble55ing
---

* content
{:toc}
## pwn docker 搭建

年底了，清理一下之前的底稿，这是10月10号就想发了的就，结果就拖到现在了。。

然后去年年底就想发了，结果就拖到现在了。。春节是无论如何拖不过去了，就现在了。

pwn题目所涉及的libc版本多变，不同的版本添加了不同的安全策略，

### docker安装

网上找的个集成工具，这个是该项目<https://github.com/skysider/pwndocker> 

docker pull skysider/pwndocker 

sudo docker run -i -t --privileged -v /home/w1804/www:/var/www skysider/pwndocker bash

这个项目能把大多数的运行环境搭建起来了，但是还没有调试环境，用起来不方便。

### 遇到的问题

error：

```Error response from daemon: Get https://registry-1.docker.io/v2/: net/http: TLS handshake timeout```

使用dig @114.114.114.114 registry-1.docker.io 

找到一个可以ping通的地址，将其添加到/etc/hosts 里。

```100.24.246.89 registry-1.docker.io```

```
vim /etc/docker/daemon.json 

{

    "graph": "/mnt/docker-data",

    "storage-driver": "overlay",

    "registry-mirrors": ["https://registry.docker-cn.com"]

}

```

之后重启

service docker restart 

systemctl restart docker 

### 使用

进入docker，为程序绑定运行时的库，

```
patchelf --set-interpreter /ctf/work/ld-linux-x86-64.so.2 printf
patchelf --set-rpath /ctf/work:/libc.so.6 printf
```

注意第二个是需要有冒号的，这两个命令也要添加到py里，就可以调试了

这就比较方便

### 额外

在使用gdb调试的时候，如果是给的libc，就无法通过main_arena或者heapinfo查看堆的信息，因为libc缺少符号表，gdb加载不出来。这时候有一个方法能够调试起来，那就是编译库文件，然后将所有库文件放入到工作目录的.debug文件夹下。

<https://github.com/matrix1001/glibc-all-in-one> 这里有一个能够简洁编译glibc的项目，非常有帮助