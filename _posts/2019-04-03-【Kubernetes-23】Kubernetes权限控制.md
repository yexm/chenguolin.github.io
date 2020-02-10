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

# 二. Authentication(鉴权)
APIServer 是一个提供 HTTP 接口的服务，为了安全性考虑任何一个请求到来时都需要经过鉴权。所谓鉴权指的是验证请求合法性，确认请求来自合法的客户端。之前我们在 [HTTP API接口安全性设计](https://chenguolin.github.io/2017/07/26/HTTP-API-2-HTTP-API%E6%8E%A5%E5%8F%A3%E5%AE%89%E5%85%A8%E6%80%A7%E8%AE%BE%E8%AE%A1/)提到过为了保证接口安全我们有2种方式 `对称密钥签名` 和`私钥签名公钥验签`，这是最常见的接口鉴权方式。

Kubernetes APIServer 则使用 `client certificates` 和 `token` 2种方式进行请求鉴权，`client certificates` 是用的最多的方式。

## ① client certificates
client certificates 指的是客户端证书用于标识Client或者User，CA 机构会遵守 X.509 规范来签发客户端证书，证书用于请求 APIServer 时鉴权使用，关于证书相关的内容可以参考 [Client authenticated_TLS_handshake](https://en.wikipedia.org/wiki/Transport_Layer_Security#Client-authenticated_TLS_handshake)

我们知道 kubeconfig 文件默认是以证书的方式来访问 APIServer的，例如下面这个文件内容

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/ca.crt         //CA证书，CA指的是颁发数字证书的机构
    server: https://kubernetes.docker.internal:6443       //APIServer
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
    client-certificate: /etc/kubernetes/client.crt       //客户端证书
    client-key: /etc/kubernetes/client.key               //客户端私钥
```

有了ca.crt、client.crt、client.key 后我们就可以向 APIServer 发起请求，kubectl 默认情况下会根据 kubeconfig 配置文件的内容再请求 APIServer 的时候进行签名。除此，之外我们也可以使用 curl 自己请求 APIServer，例如下面这个例子。

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


# 三. Authorization(授权)
## ① role和rolebinding

## ② clusterrole 和 clusterrolebinding

# 四. 举例

