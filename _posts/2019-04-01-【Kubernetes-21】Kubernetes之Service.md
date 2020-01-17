---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 背景
前面几篇文章我们介绍了Pod、Deployment、Daemonset、Cronjob 和 Job 这些对象，最终的目的都是为了启动一序列的业务Pod，这篇文章来介绍一下 Kubernetes 另一个API对象 Service。

我们先来了解一下为什么 Kubernetes 需要 Service？

我们知道 Pod 是有生命周期的，大部分情况下我们都是通过 Deployment 来部署业务，这就意味着一个业务会对应有多个 Pod，每个 Pod 都有一个IP地址。一个很常见的场景是 前端服务调用后端服务，假设前端和后端服务都通过 Deployment 进行部署，那前端如何找到后端服务Pod，同时该调用哪个Pod？

所以，为了解决这个问题，我们需要使用 Kubernetes Service。`Kubernetes 之所以需要 Service，一方面是因为 Pod 的 IP 不是固定的，另一方面则是因为一组 Pod 实例之间总会有负载均衡的需求`。

# 二. Service
## ① 介绍

## ② 命令

## ③ 属性

# 三. 实现

