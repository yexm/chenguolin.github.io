---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 概述
我们知道 Kubernetes 中所有的 API 对象都保存在 Etcd 里，对这些 API 对象的操作一定都是通过访问 kube-apiserver 实现的。任何的中间层都是为了解耦，APIServer 也不例外，因为我们需要 APIServer 来帮助我们做鉴权、授权的工作。任何一个系统安全性都非常重要，Kubernetes 也不例外，所以任何操作 API 对象的请求都是通过 APIServer 而非直接操作 Etcd。

访问 APIServer 有以下几种方式
1. 容器外: kubectl、kubectl proxy 和 REST HTTP 请求
2. 容器内: client-go 等

但是，无论是容器内还是容器外，任何一个请求到 APIServer 之后它的处理流程如下，要经过 鉴权、授权 和 准入控制，这篇文章会重点介绍`鉴权`和`授权`，对于`准入控制`我们不做过多的介绍，详细可以参考 [Controlling Access to the Kubernetes API](https://kubernetes.io/docs/reference/access-authn-authz/controlling-access/)

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/Kubernetes-access-pipeline.png?raw=true)

`注意: 只有通过 HTTPS 访问 APIServer 的时候才需要鉴权和授权，HTTP 访问不需要。因为，正常来说 HTTPS 是外部请求需要考虑安全性，而 HTTP 是内部请求安全性上可以保证所以无需鉴权和授权。`

# 二. Authentication(鉴权)
APIServer 是一个提供 HTTP 接口的服务，为了安全性考虑任何一个请求到来时都需要经过鉴权。所谓鉴权指的是验证请求合法性，确认请求来自合法的客户端。之前我们在 [HTTP API接口安全性设计](https://chenguolin.github.io/2017/07/26/HTTP-API-2-HTTP-API%E6%8E%A5%E5%8F%A3%E5%AE%89%E5%85%A8%E6%80%A7%E8%AE%BE%E8%AE%A1/)提到过为了保证接口安全我们可以使用 `对称密钥签名` 或 `私钥签名公钥验签`，同时在[cookies和token鉴权区别](https://chenguolin.github.io/2017/07/29/HTTP-API-4-Cookies%E5%92%8CToken%E9%89%B4%E6%9D%83%E5%8C%BA%E5%88%AB/)中我们提到 Token鉴权 是目前用的最多的鉴权方式。

Kubernetes APIServer 使用 `client certificates` 和 `token` 2种方式对请求进行鉴权，默认使用 `client certificates` 方式，详情可以参考 [kubernetes authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)，如果请求鉴权失败，则会返回 HTTP 401 状态码。

## ① client certificates
client certificates 指的是客户端证书用于标识Client或者User，CA 机构会遵守 X.509 规范来签发客户端证书，证书用于请求 APIServer 时鉴权使用，关于证书相关的内容可以参考 [Client authenticated_TLS_handshake](https://en.wikipedia.org/wiki/Transport_Layer_Security#Client-authenticated_TLS_handshake)

我们知道 kubeconfig 文件默认是以证书的方式来访问 APIServer的，例如下面这个文件内容

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/ca.crt         #CA证书，CA指的是颁发数字证书的机构
    server: https://kubernetes.docker.internal:6443       #APIServer
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate: /etc/kubernetes/client.crt       #客户端证书
    client-key: /etc/kubernetes/client.key               #客户端私钥
```

有了ca.crt、client.crt、client.key 后我们就可以向 APIServer 发起请求，kubectl 默认情况下会根据 kubeconfig 配置文件的内容在请求 APIServer 的时候进行签名。除此之外，我们也可以使用 curl 自己请求 APIServer，例如下面这个例子。

```
$ APISERVER=$(kubectl config view --minify | grep server | cut -f 2- -d ":" | tr -d " ")
$ echo $APISERVER
https://kubernetes.docker.internal:6443

$ CACRT=$(kubectl config view --minify | grep certificate-authority | cut -f 2- -d ":" | tr -d " ")
$ echo CACRT
/etc/kubernetes/ca.crt

$ CLIENTCRT=$(kubectl config view --minify | grep client-certificate | cut -f 2- -d ":" | tr -d " ")
$ echo CLIENTCRT
/etc/kubernetes/client.crt

$ CLIENTKEY=$(kubectl config view --minify | grep client-key | cut -f 2- -d ":" | tr -d " ")
$ echo CLIENTKEY
/etc/kubernetes/client.key

$ curl $APISERVER/api --cacert $CACRT --cert $CLIENTCRT --key $CLIENTKEY
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "192.168.65.3:6443"
    }
  ]
}
```

## ② token
token鉴权 是我们比较熟悉的一种方式，在[cookies和token鉴权区别](https://chenguolin.github.io/2017/07/29/HTTP-API-4-Cookies%E5%92%8CToken%E9%89%B4%E6%9D%83%E5%8C%BA%E5%88%AB/)中我们仔细对比了这两种鉴权方式的区别。同时提到基于token鉴权方式是无状态的，所以基于JSON Web Tokens 已经成为事实标准。

Kubernetes token 跟 ServiceAccount 有关系，每个 ServiceAccount 都会关联一个 Secret，Secret 内正是 token 内容。这个 Secret 就是ServiceAccount 用来跟 APIServer 进行交互的授权文件，token 内容一般是证书或者密码，它以一个 Secret 对象的方式保存在 Etcd 当中。Kubernetes default namespace 默认有一个 ServiceAccount 为 default，这个 ServiceAccount 有访问 APIServer 的绝大多数权限（权限很大）。

我们可以来验证一下，查看一下 default ServiceAccount 所关联 Secret 的 token

```
$ kubectl get sa
NAME      SECRETS   AGE
default   1         34d

$ kubectl get sa default -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2020-01-07T02:00:23Z"
  name: default
  namespace: default
  resourceVersion: "343"
  selfLink: /api/v1/namespaces/default/serviceaccounts/default
  uid: 7734964f-30f1-11ea-b94b-025000000001
secrets:
- name: default-token-htw52

$ kubectl get secret default-token-htw52 -o yaml | grep 'token:'
token: ZXlKaGJHY2lPaU...
```

拿到 token 之后，我们就可以通过 token 请求 APIServer，通过 token APIServer 就可以完成请求鉴权。

```
$ token=$(kubectl describe secret default-token-htw52 | grep 'token:' | cut -f2 -d':' | tr -d " ")
$ curl $APISERVER/api --header "Authorization: Bearer $token" --cacert /etc/kubernetes/ca.crt
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "192.168.65.3:6443"
    }
  ]
}
```

# 三. Authorization(授权)
鉴权是为了校验请求是否合法，Authorization 授权则是为了授予某个用户、对象某些权限，通过授权进行权限控制。Kubernetes 支持多种授权模式，包括 `Node`、`ABAC`、`RBAC` 和 `Webhook`，具体可以参考文档 [authorization mode](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#review-your-request-attributes)。如果请求授权校验失败，则会返回 HTTP 403 状态码。

在 Kubernetes 项目中，使用的授权（Authorization）模式是 [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)，通过 kube-apiserver 进程参数 `--authorization-mode=RBAC` 进行配置。关于 RBAC 的细节，我们分为以下2部分来介绍。

## ① role和rolebinding
Role 和 RoleBingding 是 Kubernetes 的 API 对象，Role 是一组对 Kubernetes API 对象的操作权限规则，RoleBingding 用于把 Role 绑定在指定对象上。指定对象可以是 User、Group 和 ServiceAccount，用的最多的是 ServiceAccount。但是 Kubernetes 中，其实并没有一个叫作 User 的 API 对象，实际上 Kubernetes 里的 User，只是一个授权系统里的逻辑概念（比如kubeconfig内的User），大部分情况下，我们只要使用 ServiceAccount 就足够了。

下面，我们我们分别介绍一下如何把 Role 绑定给 User、ServiceAccount 和 Group

### User
1. 先定义Role对象（只能作用于 default namespace，允许用户对 default namespace pod 进行 GET、WATCH 和 LIST 操作）
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default                 #允许作用namespace
  name: pod-reader                   #Role对象名称
rules:
- apiGroups: [""] #                  #空表示核心API对象
  resources: ["pods"]                #允许访问API对象
  verbs: ["get", "watch", "list"]    #允许操作列表，全集为 "get", "list", "watch", "create", "update", "patch", "delete"
```
   
2. 定义一个RoleBinding对象（把Role绑定在jane用户上）
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods                   #RoleBinding对象名称
  namespace: default                #允许作用namespace
subjects:                           #subjects定义要授予的用户或对象
- kind: User                        #类型，支持User、Group 和 ServiceAccount，用的最多的是ServiceAccount
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:                            #roleRef指向具体的Role对象
  kind: Role                         
  name: pod-reader 
  apiGroup: rbac.authorization.k8s.io
```
3. 通过RoleBinding对象，我们就可以把 pod-reader Role 定义的权限绑定给用户 jane，这样 jane 就可以对 default namespace pod 进行 GET、WATCH 和 LIST 操作。
4. Role 和 RoleBinding 对象它们对权限的限制规则仅在它们自己的 Namespace 内有效，roleRef 也只能引用当前 Namespace 里的 Role 对象。

### ServiceAccount
之前我们提到 User 其实用的不多，用的最多的是 ServiceAccount，ServiceAccount 可以简单的理解为 `Kubernetes内置用户` 类型。下面，我们看下如何为一个 ServiceAccount 授权。

1. 先创建一个 ServiceAccount
```
apiVersion: v1
kind: ServiceAccount
metadata: 
  namespace: default
  name: cgl-sa
```
2. 定义Role对象（只能作用于 default namespace，允许用户对 default namespace pod 进行 GET、WATCH 和 LIST 操作）
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default                 #允许作用namespace
  name: pod-reader                   #Role对象名称
rules:
- apiGroups: [""] #                  #空表示核心API对象
  resources: ["pods"]                #允许访问API对象
  verbs: ["get", "watch", "list"]    #允许操作列表，全集为 "get", "list", "watch", "create", "update", "patch", "delete"
```
3. 定义一个RoleBinding对象（把 Role 绑定在 cgl-sa 这个 ServiceAccount 上）
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods                   #RoleBinding对象名称
  namespace: default                #允许作用namespace
subjects:                           #subjects定义要授予的用户或对象
- kind: ServiceAccount              #ServiceAccount类型
  name: cgl-sa
  apiGroup: rbac.authorization.k8s.io
roleRef:                            #roleRef指向具体的Role对象
  kind: Role                         
  name: pod-reader 
  apiGroup: rbac.authorization.k8s.io
```
4. 查看cgl-sa ServiceAccount
```
$ kubectl describe sa cgl-sa
Name:                cgl-sa
Namespace:           default
Labels:              <none>
Annotations:         kubectl.kubernetes.io/last-applied-configuration:
                    {"apiVersion":"v1","kind":"ServiceAccount","metadata":{"annotations":{},"name":"cgl- sa","namespace":"default"}}
Image pull secrets:  <none>
Mountable secrets:   cgl-sa-token-f8ppj
Tokens:              cgl-sa-token-f8ppj
Events:              <none>
```
5. 根据鉴权模块提到每个 ServiceAccount 都会关联一个 Secret，每个Secret都会有 token，使用这个token我们就可以访问 APIServer。下面我们测试一下 cgl-sa 这个ServiceAccount的权限（default namespace pod 进行 GET、WATCH 和 LIST 操作）
```
$ token=$(kubectl describe secret cgl-sa-token-f8ppj | grep 'token:' | cut -f2 -d':' | tr -d " ")
$ curl $APISERVER/api/v1/namespaces/default/pods/ --header "Authorization: Bearer $token" --insecure
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces/kube-system/pods",
    "resourceVersion": "1111035"
  },
  ...
}
```

ServiceAccount 用的最多的场景是在用户 Pod 内，通过在用户的 Pod 声明使用某个 ServiceAccount 从而达到访问某个 API 对象的目的，例如下面这个 Pod 配置使用 cgl-sa 这个 ServiceAccount。如果 Pod 没有声明 serviceAccountName，Kubernetes 会自动在它的 Namespace 下创建一个名叫 default 的默认 ServiceAccount，然后分配给这个 Pod。这个默认 ServiceAccount 并没有关联任何 Role，此时它有访问 APIServer 的绝大多数权限（权限很大）。

```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: default
spec:
  serviceAccountName: cgl-sa
  ...
```

当这个 Pod 运行起来之后，我们就可以看到该 ServiceAccount 的 Secret 对象被 Kubernetes 自动挂载到了容器的 `/var/run/secrets/kubernetes.io/serviceaccount` 目录下，这个目录包含 `ca.crt`、`namespace` 和 `token` 3个文件。容器里的应用，就可以使用这个 ca.crt 来访问 APIServer 了。

### Group
前面我们提到 RoleBinding 指定对象可以是 User、Group 和 ServiceAccount，现在我们来看一下 Group。Group 指的是用户组，实际上，一个 ServiceAccount 名称在 Kubernetes 中对应的是 `system:serviceaccount:{namespace}:{serviceaccount-name}`。所以某个 Namespace 下对应的内置用户组的名称为 `system:serviceaccount:{namespace}`，通过在 RoleBinding 配置用户组可以把 Role 绑定给某个 Namespace 下所有 ServiceAccount。

举个例子，如果我们想配置 default 下所有 ServiceAccount 绑定某个Role
   
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: all-sa-in-default
  namespace: default
subjects:
- kind: Group
  name: system:serviceaccounts:default 
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: {role name}
  apiGroup: rbac.authorization.k8s.io
```

除了用户自定义的 Role 之外，Kubernetes 内置了很多为系统保留的 Role 和 RoleBinding，默认 `system:` 开头，我们可以通过 `kubectl get role -n kube-system` 和 `kubectl get rolebinding -n kube-system` 进行查看。

## ② clusterrole 和 clusterrolebinding
对于非 Namespaced 对象比如Node，或者说某一个 Role 想要作用于所有的 Namespace 的时候 Role 和 RoleBinding 就无能为力了。这个时候我们可以使用 ClusterRole 和 ClusterRoleBinding，这2个 API 对象的用法和 Role 和 RoleBinding 一致，只不过不需要配置 Namespace 字段。

Kubernetes 内置了很多为系统保留的 ClusterRole 和 ClusterRoleBinding，默认 `system:` 开头，我们可以通过 `kubectl get clusterrole -n kube-system` 和 `kubectl get cluterrolebinding -n kube-system` 进行查看。有几个内置的 ClusterRole 我们需要了解，`admin`、`cluster-admin`、`edit`、`view`，可以通过 `kubectl describe clusterrole` 命令查看详细的权限规则，通过名字我们能够知道 view 这个 ClusterRole 正是用来提供只读权限的。

例如我们希望把 view ClusterRole 绑定给某个 ServiceAccount，那 ClusterRoleBinding 配置样例如下。这个例子，cgl Namespace 下的 cgl-ca ServiceAccount 可以读取整个集群所有API对象，因为它绑定了 view ClusterRole。

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-only
subjects:
- kind: ServiceAccount
  name: cgl-ca
  namespace: cgl
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

