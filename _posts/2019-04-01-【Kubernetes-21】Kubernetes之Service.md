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

# 二. Service介绍
Kubernetes 中 Service 是一个抽象的概念，它定义了 Pod 的逻辑分组和一种可以访问它们的策略，这组 Pod 能被Service访问，使用YAML（优先）或JSON 来定义 Service，Service 所针对的一组 Pod 通常由 LabelSelector 选择。Service是一个抽象层，它定义了一组逻辑的Pods，这些Pod提供了相同的功能，借助Service，应用可以方便的实现服务发现与负载均衡，可以把 Service 加上一组 Pod 称作是一个微服务。

知道 Service 的基本定义后，我们来看一下如何创建一个 Service。我们先通过先创建一个 Deployment 对象管理3个 Pod，然后通过再创建 Service，通过  LabelSelector 选择 Pod。

Deployment yaml 配置如下，Pod 的作用是每次访问 9376 端口时，返回它自己的 hostname。

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
4. 查看Service，发现Service VIP地址为10.96.250.206，代理端口为9376
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

根据以上信息，确认 hostnames Service 的 Endpoints 正是 Deployment 所管理的3个 Pod。需要注意的是，只有处于 Running 状态，且 readinessProbe 检查通过的 Pod，才会出现在 Service 的 Endpoints 列表里。并且，当某一个 Pod 出现问题时，Kubernetes 会自动把它从 Service 里摘除掉。

通过 Service VIP 10.96.250.206 访问 Pod，这个 VIP 地址是 Kubernetes 自动为 Service 分配的。
```
$ curl "10.96.250.206:9376"
hostnames-85cd66c585-ch4zk
   
$ curl "10.96.250.206:9376"
hostnames-85cd66c585-4w9rh
   
$ curl "10.96.250.206:9376"
hostnames-85cd66c585-p99c5
```

通过三次连续不断地访问 Service 的 VIP 地址和代理端口 9376，它就为我们依次返回了三个 Pod 的 hostname，因为 Service 提供的是 Round Robin 方式的负载均衡。对于这种方式，我们称为 `ClusterIP 模式的 Service`。每个 Service 都会被分配一个唯一的IP地址，这个IP地址与 Service 的生命周期绑定在一起，当 Service 存在的时候它不会改变。

# 三. Service使用
Service 常用的命令如下

1. 创建Service: kubectl apply -f service.yaml
2. 列出Service: kubectl get service -n {kube-system}
3. 查看Service: kubectl get service -n {kube-system} {service-name} -o yaml
4. 描述Service: kubectl describe service -n {kube-system} {service-name}
5. 编辑Service: kubectl edit service -n {kube-system} {service-name}
6. 删除Service: kubectl delete service -n {kube-system} {service-name}

为了更好的了解 Service，我们需要熟悉 Service yaml 配置文件相关属性字段的含义，我们通过下面这个例子来了解一下 Service。有关 Service 属性字段的相关含义，可以参考 [kubernetes api core/v1/types Service](https://github.com/kubernetes/api/blob/master/core/v1/types.go#L4023)

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app":"hostnames"},"name":"hostnames","namespace":"kube-system"},"spec":{"ports":[{"name":"http","port":9376,"protocol":"TCP","targetPort":9376}],"selector":{"app":"hostnames"}}}
  creationTimestamp: "2020-01-14T01:59:23Z"
  labels:
    app: hostnames
  name: hostnames
  namespace: kube-system
  resourceVersion: "2353260"
  selfLink: /api/v1/namespaces/kube-system/services/hostnames
  uid: 3585fd62-3df4-4ef4-9b99-ac6716f8fffb
spec:
  clusterIP: 10.96.250.206
  ports:
  - name: http
    port: 9376
    protocol: TCP
    targetPort: 9376
  selector:
    app: hostnames
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

## ① type字段
type相关的字段的定义可以参考 [kubernetes apimachinery meta/v1/types TypeMeta](https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L41) 主要是以下字段

1. apiVersion: 使用的API对象的版本，可以参考kubernetes github源码 v1beta1 就是表示版本
2. kind: API对象类型，对于Service来说一直是 Service

## ② meta字段
meta相关的字段的定义可以参考 [kubernetes apimachinery meta/v1/types ObjectMeta](https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L110) 主要是以下字段

1. name: Service名称，一般由业务在yaml文件里面设置
2. namespace: Service所属的namespace （kubernetes namespace类似组的概念，和linux namespace不同）
3. creationTimestamp: Service创建时间
4. labels: Service相关的label，有些是用户设置的，有些则是kubernetes自动设置的
5. uid: kubernetes自动生成的唯一的Service id
6. selfLink: 当前Service的url，可以通过访问该url获取到Service的相关信息
7. annotations: 用户自己设置的一些key-value键值对注释，类似labels

## ③ spec字段
spec相关的字段的定义可以参考 [kubernetes api core/v1/types ServiceSpec](https://github.com/kubernetes/api/blob/master/core/v1/types.go#L3836) 主要是以下字段

1. selector: 用于筛选Pod
2. clusterIP: Service的VIP地址，如果用户没有设置 kubernetes 会随机生成一个
3. type: Service类型，主要有以下4种类型 ClusterIP、NodePort、LoadBalancer 和 ExternalName （ClusterIP是Service默认类型，Service只能在集群内被访问）
4. ports: Service暴露的一序列端口，关键的字段如下
    + name: 这组Port的配置名称，只能包含`小写字母`和`-`，并且只能以字母开头
    + protocol: 协议名称，默认tcp
    + port: Service暴露的端口
    + TargetPort: Pod提供访问的端口
5. externalIPs: Service配置的外部IP，通过外部的 IP 路由到集群中一个或多个 Node 上
6. loadBalancerIP: type为LoadBalancer时必须要设置的LoadBalancer IP地址（常用于公有云场景）
7. externalName: type为ExternalName时必须要设置的Service name （类似CNAME，使用方式可以参考 [Service externalname](https://kubernetes.io/docs/concepts/services-networking/service/#externalname)）

## ④ status字段
status相关的字段的定义可以参考 [kubernetes api core/v1/types ServiceStatus](https://github.com/kubernetes/api/blob/master/core/v1/types.go#L3793) 主要是以下字段

只有一个字段 loadBalancer，默认情况下为空，spec.type 为 LoadBalancer 时该字段会有值。

# 四. Service实现
Service 实现由 kube-proxy 组件负责的，每个 kubernetes 节点都运行了一个 kube-proxy 组件，kube-proxy 组件目前支持3种模式 `userspace`、`iptables`、`ipvs`，可以参考 [kube-proxy --proxy-mode ProxyMode](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)。所谓 Service，其实就是 Kubernetes 为 Pod 分配的、固定的、基于 iptables（或者 IPVS）的访问入口。而这些访问入口代理的 Pod 信息，则来自于 Etcd，由 kube-proxy 通过控制循环来维护。

## ① userspace
userspace 模式是 Kubernetes v1.0 之前 kube-proxy 默认的实现方案，现在已经不再使用了。实现方案是每个节点都运行了一个 kube-proxy 组件，kube-proxy 通过 Service 的 Informer 感知到一个 Service 对象的创建或删除，然后在宿主机上创建或删除 iptables 规则，请求经过 iptables 后，再转发给kube-proxy端口。默认情况下，kube-proxy 请求后端 Pod 会使用 `round-robin` 算法实现负载均衡。关于这部分的源码可以参考 [kubernetes/pkg/proxy/iptables](https://github.com/kubernetes/kubernetes/tree/master/pkg/proxy/userspace)

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/service-userspace-mode.png?raw=true)

## ② iptables
iptables 模式是 Kubernetes v1.1 版本新增的，从 Kubernetes v1.2 起，kube-proxy 默认实现模式为 iptables。每个节点都运行了一个 kube-proxy 组件，kube-proxy 通过 Service 的 Informer 感知到一个 Service 对象的创建或删除，然后在宿主机上创建或删除 iptables 规则。关于这部分的源码可以参考 [kubernetes/pkg/proxy/iptables](https://github.com/kubernetes/kubernetes/tree/master/pkg/proxy/iptables)，关于 iptables 可以参考 [Linux iptables介绍](https://chenguolin.github.io/2016/09/30/Linux-10-Linux-iptables%E4%BB%8B%E7%BB%8D/)

我们看下 hostnames Service 在宿主机创建的 iptables 规则，从 [kubernetes/pkg/proxy/iptables](https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/iptables/proxier.go#L222) 源码可以知道 kube-proxy 在写 iptables 的时候只会更新 filter、nat 这两个规则表，但是我们主要看的还是 nat 这个表。

```
Chain PREROUTING (policy ACCEPT)
num  target                prot opt source               destination
1    KUBE-SERVICES         all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */

Chain OUTPUT (policy ACCEPT)
num  target                prot opt source               destination
1    KUBE-SERVICES         all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */

Chain POSTROUTING (policy ACCEPT)
num  target                prot opt source               destination
1    KUBE-POSTROUTING      all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes postrouting rules */
2    MASQUERADE            all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match src-type LOCAL
3    RETURN                all  --  10.244.0.0/16        10.244.0.0/16
4    MASQUERADE            all  --  10.244.0.0/16       !224.0.0.0/4
5    RETURN                all  -- !10.244.0.0/16        10.244.0.0/24
6    MASQUERADE            all  -- !10.244.0.0/16        10.244.0.0/16

Chain KUBE-MARK-DROP (0 references)
num  target                prot opt source               destination
1    MARK                  all  --  0.0.0.0/0            0.0.0.0/0            MARK or 0x8000

Chain KUBE-MARK-MASQ (27 references)
num  target                prot opt source               destination
1    MARK                  all  --  0.0.0.0/0            0.0.0.0/0            MARK or 0x4000

Chain KUBE-POSTROUTING (1 references)
num  target                     prot opt source               destination
1    MASQUERADE                 all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service traffic requiring SNAT */ mark match 0x4000/0x4000

Chain KUBE-SERVICES (2 references)
num  target                     prot opt source               destination
1    KUBE-MARK-MASQ             tcp  -- !10.244.0.0/16        10.96.250.206        /* kube-system/hostnames:http cluster IP */ tcp dpt:9376
1    KUBE-SVC-7HKTMCUATIQNHXGP  tcp  --  0.0.0.0/0            10.96.250.206        /* kube-system/hostnames:http cluster IP */ tcp dpt:9376

Chain KUBE-SVC-7HKTMCUATIQNHXGP (1 references)
num  target                     prot opt source               destination
1    KUBE-SEP-OH65HOU34ABHBAW4  all  --  0.0.0.0/0            0.0.0.0/0            statistic mode random probability 0.33333333349
2    KUBE-SEP-26C2QCRVMOQSAR6D  all  --  0.0.0.0/0            0.0.0.0/0            statistic mode random probability 0.50000000000
3    KUBE-SEP-5ASQB7OILPK5E4ER  all  --  0.0.0.0/0            0.0.0.0/0

Chain KUBE-SEP-OH65HOU34ABHBAW4 (1 references)
num  target                     prot opt source               destination
1    KUBE-MARK-MASQ             all  --  10.244.0.72          0.0.0.0/0
2    DNAT                       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp to:10.244.0.72:9376

Chain KUBE-SEP-26C2QCRVMOQSAR6D (1 references)
num  target                     prot opt source               destination
1    KUBE-MARK-MASQ             all  --  10.244.0.73          0.0.0.0/0
2    DNAT                       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp to:10.244.0.73:9376

Chain KUBE-SEP-5ASQB7OILPK5E4ER (1 references)
num  target                     prot opt source               destination
1    KUBE-MARK-MASQ             all  --  10.244.0.74          0.0.0.0/0
2    DNAT                       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp to:10.244.0.74:9376
```

从上面的规则，参考 [Linux iptables介绍](https://chenguolin.github.io/2016/09/30/Linux-10-Linux-iptables%E4%BB%8B%E7%BB%8D/) 内容，我们可以知道当我们在集群内访问 hostnames Service 的时候，IP数据包的流程是这样的 `PREROUTING -> KUBE-SERVICES）-> KUBE-SVC-7HKTMCUATIQNHXGP -> KUBE-SEP-OH65HOU34ABHBAW4 / KUBE-SEP-26C2QCRVMOQSAR6D / KUBE-SEP-5ASQB7OILPK5E4ER`。

10.96.250.206 是 hostnames 这个 Service 的 VIP，所以 `KUBE-SVC-7HKTMCUATIQNHXGP  tcp  --  0.0.0.0/0      10.96.250.206     /* kube-system/hostnames:http cluster IP */ tcp dpt:9376` 这一条规则就为这个 Service 设置了一个固定的入口地址。而最终指向三条自定义链，这3个自定义链其实就是这个 Service 代理的三个 Pod，所以这一组规则，就是 Service 实现负载均衡的位置。

所以，访问 Service VIP 的 IP数据包经过 iptables 规则链处理之后，就已经变成了访问具体某一个后端 Pod 的 IP数据包了。这些 iptables 规则，正是 kube-proxy 通过监听 Pod 的变化事件，在宿主机上生成并维护的。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/service-iptables-mode.png?raw=true)

## ③ ipvs
kube-proxy 通过 iptables 处理 Service 的过程，需要在`每台宿主机`上设置相当多的 iptables 规则。当宿主机上有大量 Pod 的时候，成百上千条 iptables 规则不断地被刷新，会大量占用该宿主机的 CPU 资源，甚至会让宿主机卡在这个过程中。因此，基于 iptables 的 Service 实现，是制约 Kubernetes 项目承载更多量级的 Pod 的主要障碍。

从 Kubernetes v1.11 之后，kube-proxy 新增了 ipvs 模式。ipvs 模式的工作原理跟 iptables 模式类似，当我们创建了 Service 之后，kube-proxy 首先会在宿主机上创建一个虚拟网卡 kube-ipvs0 ，并为它分配 Service VIP 作为 IP 地址，kube-proxy 就会通过 Linux 的 ipvs 模块，为这个 IP 地址设置 N 个 ipvs 虚拟主机，并使用轮询模式 (rr) 来作为负载均衡策略。

相比于 iptables，ipvs 在内核中的实现其实也是基于 Netfilter 的 NAT 模式，所以在转发这一层上，理论上 ipvs 并没有显著的性能提升。但是，ipvs 并不需要在每个宿主机上为每个 Pod 设置 iptables 规则，而是把对这些规则的处理放到了内核态，从而极大地降低了维护这些规则的代价。ipvs 模块只负责上述的负载均衡和代理功能。而一个完整的 Service 流程正常工作所需要的包过滤、SNAT 等操作，还是要靠 iptables 来实现。只不过，这些辅助性的 iptables 规则数量有限，也不会随着 Pod 数量的增加而增加。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/service-ipvs-mode.png?raw=true)

`因此，在大规模集群里，强烈建议 kube-proxy 使用 ipvs 这个模式（通过kube-proxy --proxy-mode参数进行设置），它为 Kubernetes 集群规模带来的提升，还是非常巨大的。`

# 五. Service类型
Kubernetes 支持4种 Service 的访问方式 `ClusterIP`、`NodePort`、`LoadBalancer` 和 `ExternalName`

## ① ClusterIP
默认情况下 Service type 为 ClusterIP，在集群内部暴露 Service 的IP，集群外无法访问该 Service

1. 创建Deployment
   ```
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: my-nginx
   spec:
     selector:
       matchLabels:
         run: my-nginx
     replicas: 2
     template:
       metadata:
         labels:
           run: my-nginx
       spec:
         containers:
         - name: my-nginx
           image: nginx
           ports:
           - containerPort: 80
   ```

   $ kubectl apply -f  my-nginx.yaml
   $ kubectl get pods -l run=my-nginx -o wide  
   
2. 创建Service
   ```
   apiVersion: v1
   kind: Service
   metadata:
     name: my-nginx
     labels:
       run: my-nginx
   spec:
     ports:
     - port: 80
       protocol: TCP
     selector:
       run: my-nginx 
     type: ClusterIP
   ```

3. 查看Service
   ```
   $ kubectl get svc my-nginx
   NAME       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
   my-nginx   ClusterIP   10.0.162.149   <none>        80/TCP    21s
   ```

4. 集群内访问Service
   ```
   $ curl "http://10.0.162.149:80"
   ...
   ```

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/Kubernetes-ClusterIP-Service.png?raw=true)

## ② NodePort

## ③ LoadBalancer

## ④ ExternalName


# 六. 集群内访问Service
Kubernetes 支持3种 Service 的访问方式 `ClusterIP`、`环境变量` 和 `DNS`，但是我们推荐使用 DNS 的方式。

## ① ClusterIP
默认情况下 Service type 为 ClusterIP，在集群内我们可以通过直接访问 ClusterIP和端口 访问业务 Service。
   
```
$ kubectl get service -n kube-system
NAME        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
hostnames   ClusterIP   10.96.250.206   <none>        9376/TCP                 5d4h
   
$ curl "http://10.96.250.206:9376"
hostnames-85cd66c585-ch4zk
```

因此，在`容器外`我们可以使用 ClusterIP 来访问 Service。

## ② 环境变量
每创建一个 Pod 的时候，kubelet 会给每个容器添加当前namespace Service 相关的环境变量，只能添加当前已经存在的 Service 相关的环境变量，如果 Service 是在 Pod 启动之后创建，那么在 Pod 内是不会有关于该 Service 的环境变量的。

我们查看下当前 Service 列表 
```
$ kubectl get service -n kube-system 
  NAME        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
  hostnames   ClusterIP   10.96.250.206   <none>        9376/TCP                 3d8h
  kube-dns    ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP,9153/TCP   11d
```

部署一个新的Pod，然后进入Pod打印环境变量
```
env | grep HOSTNAMES
HOSTNAMES_SERVICE_PORT=9376
HOSTNAMES_PORT=tcp://10.96.250.206:9376
HOSTNAMES_PORT_9376_TCP_ADDR=10.96.250.206
HOSTNAMES_PORT_9376_TCP_PORT=9376
HOSTNAMES_PORT_9376_TCP_PROTO=tcp
HOSTNAMES_PORT_9376_TCP=tcp://10.96.250.206:9376
HOSTNAMES_SERVICE_PORT_HTTP=9376
HOSTNAMES_SERVICE_HOST=10.96.250.206
```

因此，在`容器内`我们可以使用内置的环境变量来访问 Service。

## ③ DNS
通常我们都会在 Kubernetes 安装 DNS 插件，例如 CoreDNS，DNS 插件会监听 ApiServer Service的创建事件，当有新 Service 创建的时候就会生成一个新的 DNS 域名 `{service-name}.{namespace}.svc.cluster.local`。因此，我们可以在 Pod 内通过 DNS 域名访问一个 Service 服务。

例如 hostsname Service 在 kube-system 下，那么 DNS 域名为 `hostsname.kube-system.svc.cluster.local`，这个域名的解析结果为 Service 的 VIP 地址。

因此，在`容器内`我们可以使用 DNS 域名 来访问 Service。

# 七. 总结
在理解了 Kubernetes Service 机制的工作原理之后，很多与 Service 相关的问题，其实都可以通过分析 Service 在宿主机上对应的 iptables 规则得到解决。
参考 [debug-service](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/)，可以用以下步骤来排查 Service 相关问题。

1. Service是否存在: `$ kubectl get service -n {namespace} {service}`
2. 查看Service yaml 确认配置是否正确: `$ kubectl get service -n {namespace} {service} -o yaml`
3. 查看Service EndPoints: `$ kubectl get ep -n {namespace} {service}`
4. 确认Service所筛选的Pod是否正常: `$ kubectl get pods -n {namespace} -l app=hostnames`
5. kube-proxy是否在运行: `$ ps auxw | grep kube-proxy`
6. 查看iptables规则是否正确: `$ iptables-save | grep hostnames` (主要看 nat 表规则)

