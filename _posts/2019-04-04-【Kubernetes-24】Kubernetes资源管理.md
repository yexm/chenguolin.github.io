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

容器中跟 CPU 相关的字段有 `spec.containers[].resources.limits.cpu` 和 `spec.containers[].resources.requests.cpu`。`requests.cpu` 表示当前容器要申请的 CPU 资源（Kubernetes 调度是根据 requests.cpu 值），`limits.cpu` 表示当前容器最多能使用的 CPU 资源上限（没有配置表示可以使用节点所有CPU资源）。每个 Pod 总 `requests.cpu` 等于所有容器 `requests.cpu` 相加，同理 `limits.cpu` 也类似。

CPU 资源单位可以是浮点数，例如 **0.5** 表示半个CPU资源，除了浮点数也可以使用 **500m** 来表示。需要注意的是，CPU 资源最小可分配的是 **1m**，就是说如果我们配置了小于 1m 的CPU资源，那 Kubernetes 会自动设置为 1m。`强烈建议使用 100m 这种方式来表示 CPU 资源，因为这是 Kubernetes 内部通用方式。`

了解这些内容之后，我们可以看以下这个例子，Pod 内有2个容器，每个容器配置了 requests.cpu 和 limits.cpu。当 Pod 创建之后可以通过 `kubectl describe pod myapp-pod` 查看对应的容器状态，可以发现2个容器配置的CPU资源是一样的。

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod-cpu
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container-1
    image: busybox
    resources:
      limits:
        cpu: 2000m
      requests:
        cpu: 100m
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
  - name: myapp-container-2
    image: busybox
    resources:
      limits:
        cpu: "2"
      requests:
        cpu: "0.1"
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
```

我们在 [docker容器技术](https://chenguolin.github.io/2019/03/13/Kubernetes-3-Docker%E5%AE%B9%E5%99%A8%E6%8A%80%E6%9C%AF/) 文章里面提到过容器的本质是一个特殊进程，这个进程由 Namespace 进行隔离，同时使用 Cgroups 进行资源限制。所以，Pod 容器配置的 requests.cpu 和 limits.cpu 实际上是在设置容器进程 Cgroups 的 cpu 子资源限制配置，它的原理总结起来为以下2点。

1. requests.cpu: 字段设置的是 cpu.shares 的值，值计算公式为 **requests.cpu * 1024**。例如 requests.cpu 值为100m 转换为小数就是 0.1，则cpu.shares 的值为102（取整）。[cpu.shares](https://docs.docker.com/engine/reference/run/#cpu-share-constraint) 值表示进程使用CPU的权重设置，默认值为1024，最小值为2。
2. limits.cpu: 字段设置的是 cpu.cfs_quota_us 的值，值计算公式为 **limits.cpu * cpu.cfs_period_us**，cpu.cfs_period_us 默认为100000。cpu.cfs_quota_us 值表示进程每个周期能使用CPU时间，配合 cpu.cfs_period_us 使用，默认为 -1 表示不限制。例如要限制进程只能使用 50% CPU 那可以设置该值为 cpu.cfs_period_us 的1/2，如果要设置进程能使用多核CPU那可以设置该值为cpu.cfs_period_us 的 n 倍。

当上面这个 Pod 启动之后，我们来确认一下这2个容器的对应进程的 Cgroups 的 cpu 资源限制配置情况。

```
$ docker ps -a | grep myapp-pod-cpu
ed39ee83a27f        busybox             "sh -c 'echo Hello K…"   24 minutes ago ...
14e3bb16e4e2        busybox             "sh -c 'echo Hello K…"   25 minutes ago ...

$ cat /sys/fs/cgroup/cpu/.../ed39ee83a27f.../cpu.shares    
102              //正确  100m * 1024 = 102
$ cat /sys/fs/cgroup/cpu/.../ed39ee83a27f.../cpu.cfs_quota_us
200000           //正确  2 * 100000 = 200000

$ cat /sys/fs/cgroup/cpu/.../14e3bb16e4e2.../cpu.shares
102              //正确  0.1 * 1024 = 102
$ cat /sys/fs/cgroup/cpu/.../14e3bb16e4e2.../cpu.cfs_quota_us
200000           //正确  2 * 100000 = 200000
```

总的来说，强烈建议每个容器都需要配置 requests.cpu 和 limits.cpu。根据实践经验是可以把 requests.cpu 值设置的小一点保证能够成功调度到某个节点上，把 limits.cpu 设置大一点保证在高峰时期应用还能够继续运行，但是这2个字段设置的值会影响Pod Qos，这块会在下文介绍。

## ② Memory
`Memory` 是 Kubernetes 系统的一种资源类型，在 Kubernetes 中 1个 CPU 等价于 `1 AWS vCPU` 或 `1 GCP Core` 或 `1 Azure vCore` 或 `1 IBM vCPU` 或 `1 CPU Hyperthread`。在 Kubernetes 中，Memory 资源属于 **不可压缩资源（compressible resources）**，当不可压缩资源不足时 Pod 会被Kill，常见的有 Pod 因为 OOM（Out-Of-Memory）被内核杀掉。

容器中跟 Memory 相关的字段有 `spec.containers[].resources.limits.memory` 和 `spec.containers[].resources.requests.memory`。`requests.memory` 表示当前容器要申请的 Memory 资源（Kubernetes 调度是根据 requests.memory 值），`limits.memory` 表示当前容器最多能使用的 Memory 资源上限（没有配置表示可以使用节点所有CPU资源）。每个 Pod 总 `requests.memory` 等于所有容器 `requests.memory` 相加，同理 `limits.memory` 也类似。

Memory 资源单位默认是 bytes ，例如 **1000000** 表示 1000000 bytes 内存。除处值为还可以使用特殊的单位，例如 **E, P, T, G, M, K** 或 **Ei, Pi, Ti, Gi, Mi, Ki** 来表示。但是需要注意的是 Mi 和 M 是不同的，**1Mi=1024*1024、1M=1000*1000**。`强烈建议使用 Mi、Ki 这种方式来表示 Memory 资源，因为这比较符合程序员的思想。`

了解这些内容之后，我们可以看以下这个例子，Pod 内有2个容器，每个容器配置了 requests.memory 和 limits.memory。当 Pod 创建之后可以通过 `kubectl describe pod myapp-pod` 查看对应的容器状态，实际上这2个容器配置的 Memory 资源是一样的。

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod-memory
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container-1
    image: busybox
    resources:
      limits:
        memory: 1073741824
      requests:
        memory: 10485760
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
  - name: myapp-container-2
    image: busybox
    resources:
      limits:
        memory: "1Gi"
      requests:
        memory: "10Mi"
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
```

我们在 [docker容器技术](https://chenguolin.github.io/2019/03/13/Kubernetes-3-Docker%E5%AE%B9%E5%99%A8%E6%8A%80%E6%9C%AF/) 文章里面提到过容器的本质是一个特殊进程，这个进程由 Namespace 进行隔离，同时使用 Cgroups 进行资源限制。所以，Pod 容器配置的 requests.memory 和 limits.memory 实际上是在设置容器进程 Cgroups 的 memory 子资源限制配置。`limits.memory 字段实际设置的是 memory.limit_in_bytes 的值，表示进程能够使用的内存上限。`

当上面这个 Pod 启动之后，我们来确认一下这2个容器的对应进程的 Cgroups 的 memory 资源限制配置情况，发现两个容器进程对应的 memory.limit_in_bytes  值一样，这也证实了 1Gi = 1073741824。

```
$ docker ps -a | grep myapp-pod-cpu
31e87e590f2d        busybox             "sh -c 'echo Hello K…"   24 minutes ago ...
adcb91fdab07        busybox             "sh -c 'echo Hello K…"   25 minutes ago ...

$ cat /sys/fs/cgroup/memory/.../31e87e590f2d.../memory.limit_in_bytes    
1073741824
$ cat /sys/fs/cgroup/memory/.../adcb91fdab07.../memory.limit_in_bytes
1073741824
```

总的来说，强烈建议每个容器都需要配置 requests.memory 和 limits.memory。根据实践经验是可以把 requests.memory 值设置的小一点保证能够成功调度到某个节点上，把 limits.memory 设置大一点保证在高峰时期应用还能够继续运行，否则应用容器很容易因为 OOM（Out-Of-Memory）被内核 Kill。

## ③ Extended resources


# 三. Quality of Service(QoS)

# 四. Policies(策略)
## ① Limit Ranges

## ② Resource Quotas
