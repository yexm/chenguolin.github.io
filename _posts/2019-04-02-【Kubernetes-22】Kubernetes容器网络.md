---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 概述
我们知道任何一个系统都需要有网络，Kubernetes也一样。从之前的文章我们知道 Linux 容器能看见的`网络栈`，实际上是被隔离在它自己的 Network Namespace 当中的。而所谓`网络栈`，就包括了 网卡（Network Interface）、回环设备（Loopback Device）、路由表（Routing Table）和 iptables 规则。对于一个进程来说，这些要素，其实就构成了它发起和响应网络请求的基本环境。

所以，在大多数情况下，我们都希望容器进程能使用自己 Network Namespace 里的网络栈，也就是拥有属于自己的 IP 地址和端口。那问题来了，这个被隔离的容器进程，该如何跟其他 Network Namespace 里的容器进程进行交互呢？

因此，这篇文章会通过 Docker网络、Kubernetes网络 2部分来介绍一下 Kubernetes容器网络，通过这篇文章可以知道在Docker环境下容器是如何通信的，在Kubernetes环境下Pod是如何通信的。

# 二. Docker网络
如果我们想要实现两台主机之间的通信，最直接的办法就是把它们用一根网线连接起来，而如果我们想要实现多台主机之间的通信，那就需要用网线，把它们连接在一台交换机上。在 Linux 中能够起到虚拟交换机作用的网络设备是网桥（Bridge），它是一个工作在数据链路层（Data Link）的设备，主要功能是根据 MAC 地址学习来将数据包转发到网桥的不同端口（Port）上。

通过之前的文章 [Docker容器网络](https://chenguolin.github.io/2019/03/18/Kubernetes-9-Docker%E5%AE%B9%E5%99%A8%E7%BD%91%E7%BB%9C/) 我们知道 Docker 有4种网络模式，默认是 bridge 的方式，也就是 Docker 会默认在宿主机上创建一个名叫 docker0 的网桥，凡是连接在 docker0 网桥上的容器，就可以通过它来进行通信。

下面，我们分为 单机访问、跨主机访问 这2种情况来讨论下Docker容器之间的通信模式。

## ① 单机访问
`单机访问`指的是同一台宿主机上的容器如何进行互相访问，我们知道当 Docker engine 启动时，会在主机上创建一个名为 docker0 的虚拟网桥，此主机上启动的 Docker 容器会连接到这个虚拟网桥上。虚拟网桥的工作方式和物理交换机类似，这样主机上的所有容器就通过交换机连在了一个二层网络中。把容器连接到 docker0 虚拟网络则是通过 Veth Pair（Virtual Ethernet Device Pairs）的虚拟设备。

Veth Pair 设备的特点是它被创建出来后，总是以两张虚拟网卡（Veth Peer）的形式成对出现的。并且从其中一个`网卡`发出的数据包，可以直接出现在与它对应的另一张`网卡`上，这两张网卡允许在不同 Network Namespace 里。这就使得 Veth Pair 常常被用作连接不同 Network Namespace 的`网线`。

因此，根据以上的信息我们知道同一个宿主机上的网络可以通过 `Veth Pair + docker0` 实现网络互通，它的通信原理如下图所示

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker-network-1.png?raw=true)

## ② 跨主机访问


# 三. Kubernetes网络
## ① flannel

## ② calico
