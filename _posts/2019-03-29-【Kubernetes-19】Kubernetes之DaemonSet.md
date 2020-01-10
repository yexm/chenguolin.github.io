---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 背景
上一篇文章 [kubernetes deployment](https://chenguolin.github.io/2019/03/28/Kubernetes-18-Kubernetes%E4%B9%8BDeployment/) 我们学习了 Deployment 这个API对象，是kubernetes集群中用的最多的对象类型。Deployment 支持同时运行多个Pod，以及Pod弹性伸缩，kubernetes 基于Pod粒度进行调度，对业务屏蔽掉基础环境，业务不需要关心运行在哪个节点上。

但是，有些情况我们希望能够在 kubernetes 集群的每个节点上都启动一个 Pod，做为节点的 Daemon 进程，例如以下几个例子

1. 各种网络插件的 Agent 组件，必须运行在每一个节点上，用来处理这个节点上的容器网络
2. 各种存储插件的 Agent 组件，必须运行在每一个节点上，用来在这个节点上挂载远程存储目录，操作容器的 Volume 目录
3. 各种监控组件和日志组件，也必须运行在每一个节点上，负责这个节点上的监控信息和日志采集

Kubernetes DaemonSet 正是用来解决以上问题的，通过 DaemonSet 在 Kubernetes 集群每个节点都运行一个 Daemon Pod。当集群加入一个新节点的时候，会自动创建 DaemonSet 定义的 Pod。当删除节点的时候，会自动删除 DaemonSet 定义的 Pod。



# 二. Daemonset

# 三. 使用
