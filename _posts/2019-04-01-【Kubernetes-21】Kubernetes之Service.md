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

所以，为了解决这个问题，我们需要使用 Kubernetes Service。`Kubernetes 之所以需要 Service，一方面是因为 Pod 的 IP 不是固定的，另一方面则是因为一组 Pod 实例之间总会有负载均衡的需求，核心的需求是找到Pod。`

# 二. Service
## ① 介绍
Kubernetes 中 Service 是一个抽象的概念，它定义了 Pod 的逻辑分组和一种可以访问它们的策略，这组 Pod 能被Service访问，使用YAML（优先）或JSON 来定义 Service，Service 所针对的一组 Pod 通常由 LabelSelector 选择。Service是一个抽象层，它定义了一组逻辑的Pods，这些Pod提供了相同的功能，借助Service，应用可以方便的实现服务发现与负载均衡，可以把 Service 加上一组 Pod 称作是一个微服务。

知道 Service 的基本定义后，我们来看一下如何创建一个 Service。我们先通过先创建一个 Deployment 对象管理3个 Pod，然后通过再创建 Service，通过  LabelSelector 选择 Pod。

Deployment yaml 配置如下

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostnames
  namespace: kube-system
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hostnames
  template:
    metadata:
      labels:
        app: hostnames
    spec:
      containers:
      - name: hostnames
        image: k8s.gcr.io/serve_hostname
        ports:
        - containerPort: 9376
          protocol: TCP
        imagePullPolicy: IfNotPresent
```

Service yaml 配置如下

```
apiVersion: v1
kind: Service
metadata:
  name: hostnames
  namespace: kube-system
  labels:
    app: hostnames
spec:
  ports:
  - name: http
    port: 9376
    targetPort: 9376
    protocol: TCP
  selector:                 //筛选 app=hostnames 标签的Pod
    app: hostnames
```

1. 创建Deployment: `$ kubectl apply -f deployment.yaml`
2. 查看Deployment管理的Pod
   ```
   $ kubectl get po -n kube-system
   NAME                          READY   STATUS      RESTARTS   AGE     IP              NODE                                  NOMINATED NODE   READINESS GATES
   hostnames-85cd66c585-4w9rh    1/1     Running     0          2d23h   10.244.0.74     ecs-s6-large-2-linux-20200105130533   <none>           <none>
   hostnames-85cd66c585-ch4zk    1/1     Running     0          2d23h   10.244.0.73     ecs-s6-large-2-linux-20200105130533   <none>           <none>
   hostnames-85cd66c585-p99c5    1/1     Running     0          2d23h   10.244.0.72     ecs-s6-large-2-linux-20200105130533   <none>           <none>
   ```
3. 创建Service: `$ kubectl apply -f service.yaml`
4. 查看Service
   ```
   $ kubectl get service -n kube-system
   NAME          TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                  AGE
   hostnames     ClusterIP      10.96.250.206   <none>          9376/TCP                 2d23h
   ```
5. 查看Service筛选的Pod  (Service 选中的 Pod 称为Endpoints)
   ```
   $ kubectl get ep -n kube-system hostnames
   NAME        ENDPOINTS                                            AGE
   hostnames   10.244.0.72:9376,10.244.0.73:9376,10.244.0.74:9376   2d23h
   ```
6. 根据以上信息，确认 hostnames Service 的 Endpoints 正是 Deployment 所管理的3个 Pod。需要注意的是，只有处于 Running 状态，且 readinessProbe 检查通过的 Pod，才会出现在 Service 的 Endpoints 列表里。并且，当某一个 Pod 出现问题时，Kubernetes 会自动把它从 Service 里摘除掉。
7. 我们



每个 Service 都会被分配一个唯一的IP地址，这个IP地址与 Service 的生命周期绑定在一起，当 Service 存在的时候它不会改变。每个节点都运行了一个 kube-proxy 组件，kube-proxy 监控着 Kubernetes 增加和删除 Service，对于每个 Service kube-proxy 会随机开启一个本机端口，任何向这个端口的请求都会被转发到一个后台的Pod中，如何选择哪一个后台Pod是基于SessionAffnity进行的分配。

在创建Service的时候可以指定IP地址，将spec.clusterIP的值设置为我们想要的IP地址即可。

## ② 命令

## ③ 属性

# 三. 实现

