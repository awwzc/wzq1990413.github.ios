---
layout: post
title: virtualbox虚拟机固定ip
categories: [virtualbox虚拟机,固定ip]
description: virtualbox虚拟机分配固定ip
keywords: virtualbox， 固定ip
---

### 需求
在本地跑虚拟机搭建开发环境时新建的虚拟机需要一个固定ip,并且能够连接外网，跟主机，其他虚拟机网络互通


### 实现方式

1. 给虚拟机添加一个桥接连接方式的网卡用于连接外网
![bg]({{ site.url }}/images/posts/virtualbox/bg.png)

2. 给虚拟添加一个host-only连接方式的网卡用于跟主机，其他虚拟机互联，此网卡会分配固定ip
![bg]({{ site.url }}/images/posts/virtualbox/hostonly.png)

3. 最终虚拟机固定ip为网卡enp0s8:192.168.99.103

```

根据主机网卡vboxnet0，找到虚拟机网卡enp0s8
vboxnet0: flags=8943<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST> mtu 1500
	ether 0a:00:27:00:00:00
	inet 192.168.99.1 netmask 0xffffff00 broadcast 192.168.99.255

```
![gdip]({{ site.url }}/images/posts/virtualbox/gdip.png)

4. virtualbox 常用网络模式解释和配置，参考[地址](https://cizixs.com/2017/03/09/virtualbox-network-mode-explained/)