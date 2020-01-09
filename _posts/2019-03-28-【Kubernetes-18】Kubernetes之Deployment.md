---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 背景
上一篇文章 [kubernetes之Pod](https://chenguolin.github.io/2019/03/27/Kubernetes-17-Kubernetes%E4%B9%8BPod/) 我们了解了 `Pod 这个API 对象，实际上是对容器的进一步抽象和封装而已`。Pod 对象，其实就是容器的升级版。它对容器进行了组合，添加了更多的属性和字段。这篇文章我们来了解一下 kubernetes 的 Deployment 这个API对象。

我们先来回答一个问题 `有了Pod，为什么我们还需要Deployment？`

我们知道生产环境上大部分的业务都是需要高可用的，通常实现高可用的方案是通过多副本机制。也就是说，如果我们现在有一个业务通过Pod部署，要实现高可用的话，我们就必须通过保证集群有多个相同的Pod，但是通过上一篇文章 [kubernetes之Pod](https://chenguolin.github.io/2019/03/27/Kubernetes-17-Kubernetes%E4%B9%8BPod/) 我们发现并没有一个属性字段允许同时运行多个Pod。因此，我们需要通过额外的手段来保证集群有多个Pod，这不仅给业务引入了复杂性，也这不符合kubernetes的设计原则。

因此，kubernetes 提供了多副本控制机制，常见的有 ReplicationController、ReplicaSet、Deployment、DaemonSet、Job 等。如果使用 Deployment 的话，那么业务便可以快速实现高可用，同时部署多个相同的Pod，还允许弹性伸缩。

我们在 [kubernetes架构](https://chenguolin.github.io/2019/03/24/Kubernetes-13-Kubernetes%E5%9F%BA%E7%A1%80%E4%BB%8B%E7%BB%8D/#%E4%B8%89-k8s%E6%9E%B6%E6%9E%84) 文章里提到 master 里面有一个 [kube-controller-manager](https://github.com/kubernetes/kubernetes/tree/master/cmd/kube-controller-manager)，简单的说 kube-controller-manager 用来编排某种 API 对象的。ReplicationController、ReplicaSet、Deployment、DaemonSet、Job 等对象都是通过 kube-controller-manager 内部的控制器负责的。

Deployment API 对象，它的控制器流程源码可以参考 [kubernetes controller deployment](https://github.com/kubernetes/kubernetes/tree/master/pkg/controller/deployment)。像 Deployment 这种控制器的设计原理，是 `用一种对象管理另一种对象`的艺术，控制器对象本身负责定义被管理对象的期望状态。

实际上，所有控制器最后的执行结果，要么是创建、更新一些 Pod（或者其他的 API 对象、资源），要么就是删除一些已经存在的 Pod（或者其他的 API 对象、资源）。

# 二. Deployment
Deployment 看似简单，但实际上它实现了 kubernetes 项目中一个非常重要的功能 `Pod 的水平扩展/收缩（horizontal scaling out/in）`。这个功能，是从 PaaS 时代开始，一个平台级项目就必须具备的编排能力。

Deployment 

# 三. 使用

# 四. 流程
