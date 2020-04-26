---
layout: post
title:  "docker使用"
categories: linux
tags:  tools
author: ble55ing
---

* content
{:toc}

## 安装

首先安装Docker CE 在ubantu上，参照https://docs.docker.com/install/linux/docker-ce/ubuntu/#set-up-the-repository一步一步来就可以了 。安装完成后使用docker pull xxx 或者下载好镜像并使用docker load < xxx 把镜像部署到本地，xxx为镜像名。

## 镜像和容器

然后为该镜像创建一个容器，docker run -it --privileged -v /home/path:/home/docker/path -p 8080:80 xxx bash 这样的目的在于将本地的path和该容器进行共享。

### 保存和导入镜像

docker save ID > xxx.tar

docker load < xxx.tar

### 保存和导入容器

docker export ID >xxx.tar

docker import xxx.tar containr:v1

###删除容器和镜像

docker rm ID

docker rmi ID

## 使用tips

使用tumx指令来使用tumx，主要是使用分屏功能。使用ctrlB+%增加分屏，使用ctrlB+space切换分屏模式。

docker run出来的虚拟机是拥有不同的虚拟ip的。

然后，可以从宿主机通过docker exec -it bd bash -c 'cd /home/docker/path && ./1.sh'这样来运行容器中的命令。

使用鼠标需要增加一个~/.tmux.conf的文件，其中内容为set -g mouse on。可以通过nano命令进行查看，这样就能够使用鼠标了。使用鼠标查看过往记录时是用滚轮向上。

查看docker网络环境  docker network ls

使用鼠标进行复制粘贴：选中文字，按住shift再按鼠标右键就能出来菜单了

## 使用docker Hub

docker tag container:v1 61355ing/container首先为docker打上tag，

然后使用docker login登录进去，使用docker push 61355ing/container 就能push上去了，下来的时候就是docker pull 61355ing/container 

### 使用dockerfile

命令 ： docker build -f aa_dockerfile .

需要注意的是，这命令的执行中会将该目录下所用文件上传到docker daemon，所以一定要新建一个目录。

不然就会出现如下情况

Sending build context to Docker daemon  15.84GB
Error response from daemon: Error processing tar file(exit status 1): write /pwndocker.tar: no space left on device

### docker 更改配置

比如要更改绑定的端口或者是映射的文件夹

```
docker inspect 897 # 用来查看
cd /var/lib/docker/containers/8973019dbb476fef2a3610da4156ff9a12827f27ddfd9d425ef504c93e1f9baf 
这个是docker的完整hash，在inspect里有
在其中有两个json文件，是用来做配置的。
一定要两个文件都改了，就比较稳
然后service docker restart
docker inspect 897 # 再次查看配置是否更改，没更改就再来一遍
```




