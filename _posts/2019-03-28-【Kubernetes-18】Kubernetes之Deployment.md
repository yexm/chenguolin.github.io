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
Deployment 看似简单，但实际上它实现了 kubernetes 项目中一个非常重要的功能 `Pod 的水平扩展/收缩（horizontal scaling out/in）`。这个功能，是从 PaaS 时代开始，一个平台级项目就必须具备的编排能力。Deployment 实现上是通过控制 ReplicaSet 对象，然后通过 ReplicaSet 对象达到控制 Pod 的目的。

我们先来创建一个Deployment，yaml配置文件如下。当我们提交了一个 Deployment 对象后，Deployment Controller 就会立即创建一个 Pod 副本个数为 3 的 ReplicaSet。这个 ReplicaSet 的名字，则是由 Deployment 的名字和一个随机字符串共同组成。

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

1. 创建Deployment `kubectl apply -f deployment.yaml`  （使用默认namespace）
2. 查看集群状态，发现创建1个Deployment名为 nginx-deployment，1个ReplicaSet名为nginx-deployment-54f57cf6bf 以及 3个Pod名为nginx-deployment-54f57cf6bf-xxxx
   ```
   $ kubectl get all | grep nginx
   pod/nginx-deployment-54f57cf6bf-6775m   1/1     Running   0          76s
   pod/nginx-deployment-54f57cf6bf-6wngr   1/1     Running   0          76s
   pod/nginx-deployment-54f57cf6bf-bhjzw   1/1     Running   0          76s
   deployment.apps/nginx-deployment   3/3     3            3           76s
   replicaset.apps/nginx-deployment-54f57cf6bf   3         3         3       76s
   ```
3. 我们可以确认，Deployment实现逻辑为，先创建ReplicaSet，通过ReplicaSet创建Pod
4. 查看Deployment
   ```
   $ kubectl get deploy nginx-deployment
   NAME               READY   UP-TO-DATE   AVAILABLE   AGE
   nginx-deployment   3/3     3            3           5m7s
   
   NAME        Deployment的名称，通常是业务在yaml文件里面配置的（metadata.name字段）
   READY       指当前已经处于Ready的 Pod 个数 （Ready 指的是健康检查正确）
   UP-TO-DATE  指当前处于最新版本的 Pod 个数（所谓最新版本指的是 Pod 的 Spec 部分与 Deployment 里 Pod 模板里定义的完全一致）
   AVAILABLE   指当前可用的Pod个数 （既是 Running 状态，又是最新版本，并且已经处于 Ready 状态的 Pod 的个数）
   ```
5. 查看ReplicaSet
   ```
   $ kubectl get replicaset
   NAME                          DESIRED   CURRENT   READY   AGE
   nginx-deployment-54f57cf6bf   3         3         3       11m
   
   NAME        ReplicaSet的名称，格式为 nginx-deployment-{randstring}（randstring由kubernetes根据pod-template-hash生成）
   DESIRED     指用户期望的 Pod 个数，通常是业务在yaml文件里面配置的（spec.replicas字段）
   CURRENT     指当前处于 Running 状态的 Pod 的个数
   READY       指当前已经处于Ready的 Pod 个数 （Ready 指的是健康检查正确）
   ```
6. 查看Pod
   ```
   $ kubectl get pod
   NAME                                READY   STATUS    RESTARTS   AGE
   nginx-deployment-54f57cf6bf-6775m   1/1     Running   0          14m
   nginx-deployment-54f57cf6bf-6wngr   1/1     Running   0          14m
   nginx-deployment-54f57cf6bf-bhjzw   1/1     Running   0          14m
   
   NAME        Pod的名称，格式为 nginx-deployment-54f57cf6bf-{another random string}
   READY       指当前已经处于Ready的 Pod 个数 （Ready 指的是健康检查正确）
   STATUS      指Pod的状态
   RESTARTS    指Pod重启的次数
   ```
   
`Deployment 实际上是一个两层控制器。首先，它通过 ReplicaSet 的个数来描述应用的版本。然后，它再通过 ReplicaSet 的属性（比如 replicas 的值），来保证 Pod 的副本数量。Deployment 控制 ReplicaSet（版本），ReplicaSet 控制 Pod（副本数）。`

# 三. 使用
## ① 常用命令
1. 创建Deployment: `kubectl apply -f deployment.yaml`
2. 列出Deployment: `kubectl get deployment -n {kube-system}`
3. 查看Deployment: `kubectl get deployment -n {kube-system} {deployment-name} -o yaml`
4. 

## ② 属性字段




