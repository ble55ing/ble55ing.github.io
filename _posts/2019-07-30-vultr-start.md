---
layout: post
title:  "使用Vultr搭建VPS"
categories: paper
tags: paper cfg
author: ble55ing
---

* content
{:toc}
## 使用Vultr搭建VPS

国际赛没有VPS网卡的真的是。。所以还是要搭一个

### Vultr

一个卖服务器的网站，:[http://www.vultr.com](https://www.vultr.com/?ref=7775345-4F)，目前支持支付宝和微信了

服务器纽约的有优惠，但2.5美元的只支持ipv6，也有3.5美元的，其他地区最便宜5美元

看网上的文章建议centos 6 x64

建完服务器可以在Products里看到，并在detail里面做设置，这里有IP和password

### SSR

用用户名和密码ssh上去，用户是默认root。

登上去首先是配置SSR，

```
yum -y install wget
wget --no-check-certificate https://freed.ga/github/shadowsocksR.sh; bash shadowsocksR.sh
```

然后根据提示输入密码和端口

然后部署Google的BBR拥塞控制算法

```
wget --no-check-certificate https://github.com/teddysun/across/raw/master/bbr.sh
chmod +x bbr.sh
./bbr.sh
```

然后就可以使用SSR的客户端去设置代理了。

SSR客户端需要NET Framework 4.5.2以上，下载链接：

<https://dotnet.microsoft.com/download/thank-you/net472>

###参考资料

<https://juejin.im/post/5bbdbffcf265da0ac2568a80>

<https://www.baishitou.cn/1524.html>



