---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 概述
从前面总结的文章，我们可以知道容器其实可以分为两个部分 `容器镜像` 和 `容器运行时`。容器实际上是一个由 Linux Namespace、Linux Cgroups 和 rootfs 三种技术构建出来的进程的隔离环境，作为一名开发者，我们其实并不需要关心容器运行时的差异。因为在整个`开发 - 测试 - 发布`的流程中，真正承载着容器信息进行传递的，是容器镜像，而不是容器运行时。

Kubernetes 项目在 Borg 体系的指导下，体现出了一种独有的 先进性 与 完备性，而这些特质才是一个基础设施领域开源项目赖以生存的核心价值。kubernetes 项目的架构，跟它的原型项目 Borg 非常类似，都由 Master 和 Node 两种节点组成，而这两种角色分别对应着控制节点和计算节点。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/kubernetes-jiagou-1.png?raw=true)

1. `控制节点`: 即 Master 节点，由三个紧密协作的独立组件组合而成，它们分别是负责 API 服务的 [kube-apiserver](https://github.com/kubernetes/kubernetes/tree/master/cmd/kube-apiserver)、负责调度的 [kube-scheduler](https://github.com/kubernetes/kubernetes/tree/master/cmd/kube-scheduler)，以及负责容器编排的 [kube-controller-manager](https://github.com/kubernetes/kubernetes/tree/master/cmd/kube-controller-manager)。整个集群的持久化数据，则由 kube-apiserver 处理后保存在 Etcd 中。
2. `计算节点`: 最核心的部分则是一个叫作 [kubelet](https://github.com/kubernetes/kubernetes/tree/master/cmd/kubelet) 的组件。在 Kubernetes 项目中，kubelet 主要负责同容器运行时（比如 Docker 项目）打交道。而这个交互所依赖的，是一个称作 [CRI](https://github.com/kubernetes/cri-api)（Container Runtime Interface）的远程调用接口，这个接口定义了容器运行时的各项核心操作，比如启动一个容器需要的所有参数。

所以，Kubernetes 项目并不关心你部署的是什么容器运行时，使用的什么技术实现，只要这个容器运行时能够运行标准的容器镜像，它就可以通过实现 CRI 接入到 Kubernetes 项目当中。而具体的容器运行时，比如 Docker 项目，则一般通过 [containerd/cri](https://github.com/containerd/cri) 这个容器运行时规范同底层的 Linux 操作系统进行交互，即把 CRI 请求翻译成对 Linux 操作系统的调用（操作 Linux Namespace 和 Cgroups 等）。

所以，`Kubernetes 项目最主要的设计思想是，从更宏观的角度，以统一的方式来定义任务之间的各种关系，并且为将来支持更多种类的关系留有余地。`

正常情况下，这些应用往往会被直接部署在同一台机器上，通过 Localhost 通信，通过本地磁盘目录交换文件。而在 Kubernetes 项目中，这些容器则会被划分为一个Pod，Pod 里的容器共享同一个 Network Namespace、同一组数据卷，从而达到高效率交换信息的目的。Pod 是 Kubernetes 项目中最基础的一个对象，源自于 Google Borg 论文中一个名叫 Alloc 的设计。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/kubernetes-jiagou-2.png?raw=true)

在 Kubernetes 项目中，我们所推崇的使用方法是

1. 首先通过一个 编排对象，比如 Pod、Job、CronJob 等来描述我们要管理的应用
2. 然后再为它定义一些 服务对象，比如 Service、Secret、Horizontal Pod Autoscaler（自动水平扩展器）等。这些对象，会负责具体的平台级功能。

过去很多的集群管理项目（比如 Yarn、Mesos，以及 Swarm）所擅长的，都是把一个容器按照某种规则，放置在某个最佳节点上运行起来，这种功能我们称为`调度`。
而 Kubernetes 项目所擅长的，是按照用户的意愿和整个系统的规则，完全自动化地处理好容器之间的各种关系。这种功能，就是我们经常听到的一个概念 `编排`。

Kubernetes 项目的本质: `是为用户提供一个具有普遍意义的容器编排工具。不过，更重要的是 Kubernetes 项目为用户提供的不仅限于一个工具。它真正的价值，乃在于提供了一套基于容器构建分布式系统的基础依赖。容器的本质是进程，kubernetes则是操作系统。`

