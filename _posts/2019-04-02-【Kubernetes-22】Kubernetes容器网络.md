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

## ① 单机访问


## ② 跨主机访问


# 三. Kubernetes网络
## ① flannel

## ② calico
