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
除了 CPU 和 Memory 资源外，Kubernetes v1.8 之后加入了 [ephemeral-storage](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#local-ephemeral-storage) 资源类型，但是到目前 Kubernetes v1.17 还未正式 GA。除此之外，还可以自定义资源类型，详情可以参考 [extended-resources](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#extended-resources)。

# 三. Quality of Service(QoS)
在了解了 Kubernetes 的资源模型之后，我们来看一下 Kubernetes 的 Quality of Service(QoS)。在 Kubernetes 中，Pod 配置不同的 requests 和 limits 会将这个 Pod 划分到不同的 QoS 级别当中，目前支持 **Guaranteed、Burstable、Best-Effort** 这3个级别。

QoS 主要的作用是影响节点驱逐 Pod 的策略，当节点资源不足时，kubelet 会挑选一些 Pod 进行驱逐操作，这个时候会需要参考这些 Pod 的 QoS 级别。`Best-Effort` 级别 Pod 优先级最低，当节点出现资源不足的时候最先考虑驱逐这类 Pod。`Burstable` 级别 Pod 是中等优先级，当没有 Best-Effort 级别 Pod 的时候才会考虑驱逐资源使用超过 requests 的 Burstable Pod。`Guaranteed` 级别 Pod 优先级最高，Kubernetes 会保证只有当 Guaranteed 类别的 Pod 的资源使用量超过了其 limits 的限制，或者宿主机本身正处于 Memory Pressure 状态时，Guaranteed 的 Pod 才可能被驱逐。

因此，在实际生产环境中，针对 DaemonSet 这类的应用建议把 Pod 设置成 Guaranteed 级别，尽量减少 Pod 被驱逐的概率。

## ① Guaranteed
`Guaranteed 指的是所有容器的所有资源类型 requests 和 limits 都相同的 Pod`，如果一个 Pod 只有 limits 没有 requests，那么这个 Pod 也属于 Guaranteed（如果没有 requests Kubernetes 会默认设置成 limits）。下面这2个例子，都是 Guaranteed 级别 Pod。

1.没有 requests 配置
```
containers:
  name: foo
    resources:
      limits:
        cpu: 100m
        memory: 1Gi
  name: bar
    resources:
      limits:
        cpu: 100m
        memory: 100Mi
```

2.显式的配置 requests 和 limits
```
containers:
  name: foo
    resources:
      limits:
        cpu: 100m
        memory: 1Gi
      requests:
        cpu: 100m
        memory: 1Gi
  name: bar
    resources:
      limits:
        cpu: 100m
        memory: 100Mi
      requests:
        cpu: 100m
        memory: 100Mi
```

在使用容器的时候，我们还可以通过设置 [cpuset](https://chenguolin.github.io/2019/03/14/Kubernetes-5-Docker%E5%AE%B9%E5%99%A8%E6%8A%80%E6%9C%AF%E4%B9%8BCgroups/#-cpuset) 把容器绑定到某个 CPU 的核上，而不是像 cpushare 那样共享 CPU 的计算能力。由于绑定在某个 CPU 后，在 CPU 之间的上下文切换的次数大大减少，容器里应用的性能会得到大幅提升。

所以，只要 Pod 属于 Guaranteed 级别并且容器 CPU 资源的 requests 和 limits 设置为同一个相等的**整数值**即可。

## ② Best-Effort
`Best-Effort 指的是所有容器都没有配置 requests 和 limits 的 Pod `，如果没有配置 limits 表示容器可以使用当前节点所有可分配的资源。

```
containers:
  name: foo
    resources:
  name: bar
    resources:
```

## ③ Burstable
所有不满足 Guaranteed 和 Best-Effort 的 Pod 都属于 Burstable，例如下面这几个例子

1.有一个容器 requests 和 limits 不同
```
containers:
  name: foo
    resources:
      limits:
        cpu: 1000m
        memory: 1Gi
      requests:
        cpu: 100m
        memory: 100Mi
  name: bar
    resources:
      limits:
        cpu: 100m
        memory: 100Mi
      requests:
        cpu: 100m
        memory: 100Mi
```

2.有容器没有配置 requests 和 limits
```
containers:
  name: foo
    resources:
      limits:
        cpu: 1000m
        memory: 1Gi
      requests:
        cpu: 100m
        memory: 100Mi
  name: bar
    ...
```

# 四. Policies(策略)
除了对单 Pod 进行资源配置外，Kubernetes 还支持给某个 Namespace 进行资源限制，主要有 [LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/) 和 [ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/) 两种策略。我们知道，一个 Kubernetes 集群会有多个业务，每个业务属于不同 Namespace。默认情况下 Pod 资源 limits 配置可以使用整个节点的资源， 因此如果能够在 Namespace 级别进行资源的限制就显得非常重要，可以保证业务之间公平共享资源。

## ① Limit Ranges
[LimitRange](https://github.com/kubernetes/api/blob/02edbc1d22f3f0842d77b5cac047f25f8da1b6f9/core/v1/types.go#L5241)是Kubernetes 的API对象，可以用于限制某个 Namespace 下 Pod 容器资源使用，详细使用可以参考 [Kubernetes LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/)。

```
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range
  namespace: cgl
spec:
  limits:
  - max:                
      cpu: "800m"
      memory: "1Gi"
    min:                
      cpu: "100m"
      memory: "99Mi"
    default:            
      cpu: "700m"
      memory: "900Mi"
    defaultRequest:     
      cpu: "110m"
      memory: "111Mi"
    type: Container
```

1. max: 表示 Pod 内容器最大可配置的资源
2. min: 表示 Pod 内容器最小需要配置的资源
3. default: 表示 Pod 内容器没有配置 limits 的情况下，会自动设置的默认值
4. defaultRequest: 表示 Pod 内容器没有配置 requests 的情况下，会自动设置的默认值

配置 LimitRange 后我们可以通过 `kubectl describe ns cgl` 查看当前 Namespace 的详细信息

```
$ kubectl describe ns cgl
Name:         cgl
Labels:       <none>
Annotations:  <none>
Status:       Active

No resource quota.

Resource Limits
 Type       Resource  Min   Max   Default Request  Default Limit  Max Limit/Request Ratio
 ----       --------  ---   ---   ---------------  -------------  -----------------------
 Container  cpu       100m  800m  110m             700m           -
 Container  memory    99Mi  1Gi   111Mi            900Mi          -
```

根据上面信息，如果我们在 cgl Namespace 下启动一个 Pod 并且没有配置任何 limits 和 requests，那 Kubernetes 会根据 LimitRange 自动设置 limits 和 requests。

## ② Resource Quotas
除了 LimitRange 之外，还可以使用 [ResourceQuota](https://github.com/kubernetes/api/blob/02edbc1d22f3f0842d77b5cac047f25f8da1b6f9/core/v1/types.go#L5397) 来限制某个 Namespace 总的可以使用的资源。ResourceQuota 除了支持设置 cpu、memory 资源外，还支持设置 pods、services、replicationcontrollers 等API对象，详细可以参考 [Configure Quotas for API Objects
](https://kubernetes.io/docs/tasks/administer-cluster/quota-api-object/#notes)。

```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: resource-quota
  namespace: cgl
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    pods: "5"
    services: "4"
    replicationcontrollers: "3"
    resourcequotas: "2"
    secrets: "1"
    configmaps: "1"
    persistentvolumeclaims: "1"
    services.loadbalancers: "2"
    services.nodeports: "0"
```

1. requests.cpu: 表示 Namespace 下所有 Pod 可以配置 requests.cpu 的上限
2. requests.memory: 表示 Namespace 下所有 Pod 可以配置 requests.memory 的上限
3. limits.cpu: 表示 Namespace 下所有 Pod 可以配置 limits.cpu 的上限
4. limits.memory: 表示 Namespace 下所有 Pod 可以配置 limits.memory 的上限
5. pods: 表示 Namespace 可以启动 Pod 数的上限 （其它的字段类似，就不一一枚举介绍）

配置 LimitRange 后我们可以通过 `kubectl describe ns cgl` 查看当前 Namespace 的详细信息

```
kubectl describe ns cgl
Name:         cgl
Labels:       <none>
Annotations:  <none>
Status:       Active

Resource Quotas
 Name:                   resource-quota
 Resource                Used    Hard
 --------                ---     ---
 configmaps              0       1
 limits.cpu              1400m   2
 limits.memory           1800Mi  2Gi
 persistentvolumeclaims  0       1
 pods                    1       5
 replicationcontrollers  0       3
 requests.cpu            220m    1
 requests.memory         222Mi   1Gi
 resourcequotas          1       2
 secrets                 2       1
 services                0       4
 services.loadbalancers  0       2
 services.nodeports      0       0

Resource Limits
 Type       Resource  Min   Max   Default Request  Default Limit  Max Limit/Request Ratio
 ----       --------  ---   ---   ---------------  -------------  -----------------------
 Container  cpu       100m  800m  110m             700m           -
 Container  memory    99Mi  1Gi   111Mi            900Mi          -
```

根据上面的信息，cgl Namespace limits.cpu 已经使用了 1400m，那么只剩 600m 给剩下的 Pod 使用，limits.memory 已经使用了 1800Mi，那么只剩 200Mi 给剩下的 Pod 使用。同理，针对 pods、services 等 API 对象的限制也是一样的用法。

总结一下，**LimitRange** 用于限制 Pod 内容器资源可配置的最大值、最小值 或 默认值，**ResourceQuota** 用于限制 Namespace 总的可以使用的资源，以及某些API对象可创建的个数。实际生产环境，强烈建议针对每个 Namespace 都设置 LimitRange 和 ResourceQuota。

