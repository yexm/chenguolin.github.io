---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 概述
Kubernetes 作为一个容器集群编排与管理系统为用户提供的基础设施能力，不仅包括了基本的API对象、网络管理、权限控制、日志监控还包括对应用的资源管理和调度。在 Kubernetes 中 Pod 是最小的原子调度单位，这也就意味着所有跟调度和资源管理相关的属性都应该是属于 Pod 对象的字段，关于 Pod 的介绍可以参考 [kubernetes之pod](https://chenguolin.github.io/2019/03/27/Kubernetes-17-Kubernetes%E4%B9%8BPod/)文章。

# 二. Resource model(资源模型)
Resource model 指的是 Kubernetes 的资源模型，Kubernetes 中资源指的是可以分配给 Pod 内容器使用的，最常见的有 `CPU` 和 `Memory`。每个 Kubernetes 节点都有一个可分配的资源上限，可以通过 `kubectl describe node {node-name}` 查看。一旦节点上的某个资源分配给某个 Pod 之后，它就无法再分配给其它 Pod ，直到当前 Pod 结束。

每个节点可分配的资源可以由这个公式计算得到 `[Allocatable] = [Node Capacity] - [Kube-Reserved] - [System-Reserved] - [Hard-Eviction-Threshold]`，Kube-Reserved 部分保留给 Kubernetes 组件（kubelet、docker等）使用，System-Reserved 部分是系统预留的，Hard-Eviction-Threshold 部分是可配置的阈值。关于节点的资源分配详细信息可以参考 [node resource allocatable](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/node-allocatable.md)

## ① CPU
`CPU` 是 Kubernetes 系统的一种资源类型，在 Kubernetes 中 1个 CPU 等价于 `1 AWS vCPU` 或 `1 GCP Core` 或 `1 Azure vCore` 或 `1 IBM vCPU` 或 `1 CPU Hyperthread`。在 Kubernetes 中，CPU 资源属于 **可压缩资源（compressible resources）**，当可压缩资源不足时 Pod 只会"饥饿" 但不会被Kill。

容器中跟 CPU 相关的字段有 `spec.containers[].resources.limits.cpu` 和 `spec.containers[].resources.requests.cpu`。`requests.cpu` 表示当前容器要申请的 CPU 资源，`limits.cpu` 表示当前容器最多能使用的 CPU 资源上限。每个 Pod 总

## ② Memory

## ③ Extended resources

# 三. Quality of Service(QoS)

# 四. Policies(策略)
## ① Limit Ranges

## ② Resource Quotas
