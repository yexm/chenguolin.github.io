---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 概述
我们知道 Kubernetes 中所有的 API 对象都保存在 Etcd 里，对这些 API 对象的操作一定都是通过访问 kube-apiserver 实现的。我们知道任何的中间层都是为了解耦，APIServer 也不例外，因为我们需要 APIServer 来帮助我们做鉴权、授权的工作。我们知道任何一个系统安全性都非常重要，Kubernetes 也不例外，所以任何操作 API 对象的请求都是通过 APIServer 而非直接操作 Etcd。

访问 APIServer 有以下几种方式
1. 容器外: kubectl、kubectl proxy 和 REST HTTP 请求
2. 容器内: client-go 等

但是，无论是容器内还是容器外，任何一个请求到 APIServer 之后它的处理流程如下，要经过 鉴权、授权 和 准入控制，这篇文章会重点介绍`鉴权`和`授权`，对于`准入控制`我们不做过多的介绍，详细可以参考 [Controlling Access to the Kubernetes API](https://kubernetes.io/docs/reference/access-authn-authz/controlling-access/)

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/Kubernetes-access-pipeline.png?raw=true)

# 二. Authentication(鉴权)
APIServer 是一个提供 HTTP 接口的服务，我们在 


# 三. Authorization(授权)
