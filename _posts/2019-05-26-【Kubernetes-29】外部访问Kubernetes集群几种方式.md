---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 访问ApiServer
Apiserver 做为 Kubernetes master 的核心组件之一起着非常重要的作用，因此如何访问 apiserver 就变得非常的重要，我们分为2部分来介绍一下如何访问 apiserver。

## ① 容器外
容器外指的是 `我们并不是在 Pod 内容器访问 apiserver`，比如我们在集群的节点上通过console访问，或者在其它集群通过HTTP协议进行访问的方式。容器外访问 apiserver 主要有以下几种方式，这些使用方式主要用在运维排查场景。

1. [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/)  
   ```
   kubectl 是最常使用的方式，我们日常的运维开发过程中使用 kubectl 命令都是直接访问 apiserver，从而间接控制 etcd 的状态。
   kubectl 默认情况下会读取 ~/.kube/config 做为 kubeconfig 文件内容，我们也可以使用 --kubeconfig 选项来指定kubeconfig文件路径。  
   访问 apiserver 需要进行权限控制，因此我们需要有相关的访问凭证，而这些访问凭证会存储在 kubeconfig 文件内。
   ```
    
2. kubectl proxy  
   ```
   kubectl proxy 实际上是启动一个代理的 HTTP 服务，使得我们可以通过 HTTP 协议进行访问 apiserver，具体的步骤如下  
   1). 启动proxy: kubectl proxy --port=8080
   2). 访问api: curl http://localhost:8080/api/v1/namespaces/kube-system/services/hostnames 
   
   api url path跟具体的 Kubernetes API 对象有关，具体可以看 API 对象的 metadata.selfLink
   ```
   
3. 直接访问  
   ```
   APISERVER=$(kubectl config view --minify | grep server | cut -f 2- -d ":" | tr -d " ")
   SECRET_NAME=$(kubectl get secrets | grep ^default | cut -f1 -d ' ')
   TOKEN=$(kubectl describe secret $SECRET_NAME | grep -E '^token' | cut -f2 -d':' | tr -d " ")

   curl $APISERVER/api --header "Authorization: Bearer $TOKEN" --insecure
   ```

## ② 容器内
容器内大部分场景指的是我们在代码层面访问 apiserver，例如我们需要实现一个 Kubernetes watcher 组件用于监听 Pod 事件，那这个时候我们就需要访问 apiserver 获取 Pod 的事件。由于 Kubernetes 是 Golang 开发的，建议业务代码使用 Golang 开发，那么就可以直接 Kubernetes 社区提供的 [client-go](https://github.com/kubernetes/client-go)。初此之外，还有 [python client](https://github.com/kubernetes-client/python) 可以使用。

最常见的代码如下，相关的使用例子可以参考 [client-go examples](https://github.com/kubernetes/client-go/tree/master/examples)

```
package main

import (
    "fmt"
    "time"

    "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/rest"
)

func main() {
    // creates the in-cluster config
    config, err := rest.InClusterConfig()
    if err != nil {
        panic(err.Error())
    }
    
    // creates the clientset
    clientset, err := kubernetes.NewForConfig(config)
    if err != nil {
	panic(err.Error())
    }
    
    for {
	// get pods in all the namespaces by omitting namespace
	// Or specify namespace to get pods in particular namespace
	pods, err := clientset.CoreV1().Pods("").List(metav1.ListOptions{})
	if err != nil {
	    panic(err.Error())
	}
	fmt.Printf("There are %d pods in the cluster\n", len(pods.Items))
      
        // TODO @cgl
        
	time.Sleep(10 * time.Second)
    }
}
```

我们知道访问 apiserver 需要有访问凭证，rest.InClusterConfig() 可以自动生成 kubeconfig 配置文件用于访问 apiserver。通过参考 [InClusterConfig()](https://github.com/kubernetes/client-go/blob/a432bd9ba7da427ae0a38a6889d72136bce4c4ea/rest/config.go#L451)实现我们知道它实际上是根据 `/var/run/secrets/kubernetes.io/serviceaccount/` 这个目录生成 kubeconfig 配置文件。

那 `/var/run/secrets/kubernetes.io/serviceaccount/` 这个目录是怎么来的呢？

通过查看每个 Pod 的yaml，我们会发现 Kubernetes 会自动给每个 Pod 的 spec 加上 以下配置。这个配置在 Kubernetes 对应的实际是 [service account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)，通过 serviceAccount 我们可以控制权限，从而达到访问 apiserver 的目的。

```
spec:
  containers:
  - image: nginx:1.7.9
    imagePullPolicy: IfNotPresent
    ...
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-n4slb
      readOnly: true
  ...
  serviceAccount: default
  serviceAccountName: default
  ...
  volumes:
  - name: default-token-n4slb
    secret:
      defaultMode: 420
      secretName: default-token-n4slb
```

默认情况下，Kubernetes 会给每个 Pod 加上 `default` 这个 ServiceAccount，默认情况下 default 具有 admin 的权限。我们也可以创建自定义的 ServiceAccount，绑定相关的Role来控制访问 apiserver 的权限。

# 二. 访问业务Service
之前我们在 [kubernetes之service](https://chenguolin.github.io/2019/04/01/Kubernetes-21-Kubernetes%E4%B9%8BService/) 文章学习了 Kubernetes Service 对象，了解了 Service 的基本原理、类型、实现机制等等。

通过 [Service 实现](https://chenguolin.github.io/2019/04/01/Kubernetes-21-Kubernetes%E4%B9%8BService/#%E5%9B%9B-service%E5%AE%9E%E7%8E%B0) 我们知道 `所谓 Service 的访问入口，其实就是每台宿主机上由 kube-proxy 生成的 iptables 规则，以及 dns组件 生成的 DNS 域名，一旦离开了这个集群，这些信息就无效了。`，因此，如果要在集群外访问业务 Service 有以下几种方案。

## ② NodePort 
NodePort的方式指的是 `每个节点通过kube-proxy和指定端口代理业务Service`，通过 `nodeIp:port` 就可以直接访问 Service。

```
apiVersion: v1
kind: Service
metadata:
  name: nodeport-hostnames
  namespace: kube-system
  labels:
    app: hostnames
spec:
  type: NodePort          //指定 NodePort 类型
  ports:
  - name: http
    nodePort: 30001       //每个节点指定端口为 30001 （如果不显式地声明 Kubernetes 随机分配一个可用端口，范围是 30000-32767）
    port: 9376
    targetPort: 9376
    protocol: TCP
  selector:
    app: hostnames
```

在这个 Service 的定义里，我们声明它的类型是，type=NodePort。然后在 ports 字段里声明了 Service 的 9376 端口代理 Pod 的 9376 端口，同时每个节点30001 端口代理了 Service 的 9376 端口。

1. 创建Service: `$ kubectl apply -f service.yaml`
2. 查看Service
   ```
   $ kubectl get service -n kube-system
   NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
   nodeport-hostnames   NodePort    10.96.50.225    <none>        9376:30001/TCP           8s
   ```
3. 访问Service （node-ip指的是节点IP）
   ```
   $ curl "http://{node-ip}:30001"
   hostnames-85cd66c585-4w9rh
   ```

NodePort的原理实际上是在每个节点上通过 iptables 添加了一下这些规则，然后剩下的规则和 ClusterIP 是一样的，通过 iptables nat 来控制IP数据包的流向。新增的规则为 `KUBE-NODEPORTS  all  --  0.0.0.0/0       0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL`

```
Chain PREROUTING (policy ACCEPT)
target         prot opt source               destination
KUBE-SERVICES  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */

Chain KUBE-SERVICES (2 references)
target          prot opt source          destination
KUBE-NODEPORTS  all  --  0.0.0.0/0       0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL

Chain KUBE-NODEPORTS (1 references)
target                     prot opt source        destination
KUBE-MARK-MASQ             tcp  --  0.0.0.0/0     0.0.0.0/0            /* kube-system/nodeport-hostnames:http */ tcp dpt:30001
KUBE-SVC-US4ITFTMXMOU3US2  tcp  --  0.0.0.0/0     0.0.0.0/0            /* kube-system/nodeport-hostnames:http */ tcp dpt:30001

Chain KUBE-MARK-MASQ (32 references)
target     prot opt source               destination
MARK       all  --  0.0.0.0/0            0.0.0.0/0            MARK or 0x4000

Chain KUBE-SVC-US4ITFTMXMOU3US2 (2 references)
target                     prot opt source               destination
KUBE-SEP-AHHOLAQDDJX3HTPO  all  --  0.0.0.0/0            0.0.0.0/0            statistic mode random probability 0.33333333349
KUBE-SEP-SZF5QQDOLQINGNE2  all  --  0.0.0.0/0            0.0.0.0/0            statistic mode random probability 0.50000000000
KUBE-SEP-IYZ6I5322MC53TDK  all  --  0.0.0.0/0            0.0.0.0/0

Chain KUBE-SEP-AHHOLAQDDJX3HTPO (1 references)
target          prot opt source               destination
KUBE-MARK-MASQ  all  --  10.244.0.72          0.0.0.0/0
DNAT            tcp  --  0.0.0.0/0            0.0.0.0/0            tcp to:10.244.0.72:9376

Chain KUBE-SEP-SZF5QQDOLQINGNE2 (1 references)
target          prot opt source               destination
KUBE-MARK-MASQ  all  --  10.244.0.73          0.0.0.0/0
DNAT            tcp  --  0.0.0.0/0            0.0.0.0/0            tcp to:10.244.0.73:9376

Chain KUBE-SEP-IYZ6I5322MC53TDK (1 references)
target          prot opt source               destination
KUBE-MARK-MASQ  all  --  10.244.0.74          0.0.0.0/0
DNAT            tcp  --  0.0.0.0/0            0.0.0.0/0            tcp to:10.244.0.74:9376
```

因此，实际的IP数据包流向为 `PREROUTING -> KUBE-SERVICES -> KUBE-NODEPORTS -> KUBE-SVC-US4ITFTMXMOU3US2 -> KUBE-SEP-xxxx`

如下图所示，集群外部可以访问任意个Node节点的30080端口达到访问业务Service的目的。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/Kubernetes-NodePort-Service.png?raw=true)

## ② LoadBalancer
LoadBalancer 适用于公有云上的 Kubernetes 服务，可以参考 [华为云Kubernetes LoadBalancer Service](https://support.huaweicloud.com/usermanual-cce/cce_01_0014.html)、[阿里云Kubernetes LoadBalancer Service](https://help.aliyun.com/document_detail/86531.html)

```
apiVersion: v1
kind: Service
metadata:
  name: loadbalance-hostnames
  namespace: kube-system
  labels:
    app: hostnames
spec:
  type: LoadBalancer
  loadBalancerIP: 121.36.85.100     //共有云/私有云 LB IP地址
  ports:
  - name: http
    port: 9376
    targetPort: 9376
    protocol: TCP
  selector:
    app: hostnames
```

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/Kubernetes-Loadbalance-Service.png?raw=true)

## ③ Ingress
上面提到的2种方案实际上在生产环境用的不多，作为用户，其实更希望看到 Kubernetes 为我们内置一个全局的负载均衡器。然后通过我访问的 URL，把请求转发给不同的后端 Service。`这种全局的、为了代理不同后端 Service 而设置的负载均衡服务，就是 Kubernetes 里的 Ingress 服务。所谓 Ingress，就是 Service 的 Service`。由于Ingress比较复杂，具体可以参考 [kubernetes ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/Kubernetes-Ingress.png?raw=true)




