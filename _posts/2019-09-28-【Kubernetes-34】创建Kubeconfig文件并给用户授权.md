---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 背景
Kubernetes 做为一个大的容器编排系统，安全是需要重点考虑的点，前面我们总结了关于 Kubernetes的权限控制，可以参考[kubernetes权限控制](https://chenguolin.github.io/2019/04/03/Kubernetes-23-Kubernetes%E6%9D%83%E9%99%90%E6%8E%A7%E5%88%B6/)。 做为一名 Kubernetes 系统的管理员或运维就需要考虑如何保证集群的安全，尤其在公有云的环境下如何隔离每个租户/用户权限应该重点考虑。

我们知道 Kubernetes 是一个处于 Paas 和 Iaas 中间的基础架构，一个 Kubernetes 集群会有管理员、运维、开发、业务方等几种角色。如果我们希望不同角色用户权限不一样，管理员应该具有整个集群admin权限，运维具有某些整个集群读权限同时具有某些Namespace的写权限，开发具有某些Namespace读写权限，业务方只能有自己所属业务Namespace的读写权限，那我们要如何实现呢？

一些大的公司或者公有云都会基于 Kubernetes 开发一个容器平台，通过平台屏蔽掉底层基础架构，通过平台解决一切问题，这是最理想的解决方案。但是，很多时候由于一些原因免不了我们还是需要上机器通过 kubectl 命令操作 Kubernetes API 对象。我们知道 Kubernetes 提供了 kubeconfig 文件用于访问集群 API 对象，所以我们可以根据不同用户角色创建 kubeconfig 并授予相应的权限就可以实现权限隔离。

# 二. 实现
我们知道 kubectl 命令需要读取 kubeconfig 文件的配置内容，根据配置文件内容对请求签名。[kubernetes权限控制](https://chenguolin.github.io/2019/04/03/Kubernetes-23-Kubernetes%E6%9D%83%E9%99%90%E6%8E%A7%E5%88%B6/) 这篇文章我们提到过任何外部的 HTTPS 请求到 APIServer 都需要经过 鉴权和授权。鉴权用的最多的是 Client Certificates 客户端证书的方式，授权则使用 ServiceAccount/User + Role/RoleBinding 或 ServiceAccount/User + ClusterRole/ClusterRoleBinding。

下面，我们就看一下如何使用 Client Certificates + User + ClusterRole/ClusterRoleBinding 如何创建一个 kubeconfig 文件，并为指定用户进行授权。

1.创建 Client Certificates （使用openssl，假设用户名为 cgl）
```
$ openssl genrsa -out cgl.key 2048
$ openssl req -new -key cgl.key -out cgl.csr -subj "/CN=cgl/O=eng"\n
$ kubectl apply -f - <<EOF
  apiVersion: certificates.k8s.io/v1beta1
  kind: CertificateSigningRequest
  metadata:
    name: cgl-csr
  spec:
    groups:
    - system:authenticated
    request: `(cat cgl.csr | base64 | tr -d '\n')`
    usages:
    - digital signature
    - key encipherment
    - server auth
    - client auth
  EOF
$ kubectl certificate approve $user-csr
$ kubectl get csr cgl-csr -o jsonpath='{.status.certificate}' | base64 --decode > cgl.crt
```

2.证书的内容可以通过以下命令查看: `$ openssl x509 -text -noout -in certificate.crt`

3.配置kubeconfig
```
$ kubectl config set-credentials cgl --client-certificate=cgl.crt --client-key=cgl.key
$ kubectl config set-context cgl-context --cluster={cluster-name} --user=cgl
$ kubectl config view
```

4.授权，把ClusterRole view 授权给用户 cgl
```
$ kubectl apply -f - <<EOF
  kind: ClusterRoleBinding
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    name: read-only-2-cgl
  subjects:
  - kind: User
    name: cgl
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: ClusterRole
    name: view
    apiGroup: rbac.authorization.k8s.io
  EOF
```

5.切换kubeconfig context
```
$ kubectl config get-contexts
$ kubectl config use-context $user-context
```
6.生成新的kubeconfig文件: `$ kubectl config view --minify > cgl-kubeconfig`

7.kubectl使用新kubeconfig文件，测试权限（确认用户 cgl 是否只有只读权限）
```
$ kubectl get pods -n kube-system --kubeconfig cgl-kubeconfig
NAME                                     READY   STATUS    RESTARTS   AGE
coredns-6dcc67dcbc-9zcb9                 1/1     Running   0          35d
coredns-6dcc67dcbc-mpsb7                 1/1     Running   0          77m
etcd-docker-desktop                      1/1     Running   0          35d
kube-apiserver-docker-desktop            1/1     Running   0          35d
kube-controller-manager-docker-desktop   1/1     Running   22         35d
kube-proxy-c2zfh                         1/1     Running   0          35d
kube-scheduler-docker-desktop            1/1     Running   27         35d

$ kubectl delete pod coredns-6dcc67dcbc-9zcb9 -n kube-system --kubeconfig cgl-kubeconfig
Error from server (Forbidden): pods "coredns-6dcc67dcbc-9zcb9" is forbidden: User "cgl" cannot delete resource "pods" in API group "" in the namespace "kube-system"
```

类似的如果我们想要为其它用户设置其它权限可以参考这个例子，主要的区别是在授权这一步。Kubernetes内置了很多 ClusterRole 和 ClusterRoleBinding 可以直接使用，除此之外我们也可以自己编写 ClusterRole 设定权限规则，通过 ClusterRoleBinding 给某个用户或 ServiceAccount 授权总体而言还是比较灵活的。

