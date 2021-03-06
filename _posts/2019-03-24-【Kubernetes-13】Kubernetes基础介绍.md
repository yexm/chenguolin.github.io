---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 什么是Kubernetes
`Kubernetes` 是容器编排管理系统，是一个开源的平台，可以实现容器集群的自动化部署、自动扩缩容、维护等功能。  
`Kubernetes` 是Google 2014年创建管理的，是Google 10多年大规模容器管理技术Borg的开源版本。  
`Kubernetes` 基本上已经是私有云部署的一个标准。

Kubernets有以下几个特点
1. 可移植: 支持公有云，私有云，混合云，多重云（multi-cloud）
2. 可扩展: 模块化, 插件化, 可挂载, 可组合
3. 自动化: 自动部署，自动重启，自动复制，自动伸缩/扩展

`K8s` 是将8个字母 "ubernete" 替换为 "8" 的缩写，后续我们将使用 `K8s` 代替 Kubernetes。

# 二. 为什么要使用容器
![](https://d33wubrfki0l68.cloudfront.net/e7b766e0175f30ae37f7e0e349b87cfe2034a1ae/3e391/images/docs/why_containers.svg)

1. 传统的应用部署方式是所有应用共享一个底层操作系统，这样做不利于应用的升级更新和回滚，虽然可以通过创建虚拟机来实现隔离，但是虚拟机非常的重，不利于移植
2. 新的部署方式采用容器，每个容器之间互相隔离，每个容器有自己的文件系统，容器之间进程不会相互影响，能区分计算资源。相对于虚拟机，容器能快速部署，由于容器与底层设施、机器文件系统解耦的，所以它能在不同云、不同版本操作系统间进行迁移。

# 三. K8s架构
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/k8s-architecture.png?raw=true)

## ① master
Master节点组件提供集群的管理控制中心，通常在一台VM/机器上启动所有Master组件，并且不会在此VM/机器上运行用户容器。

1. `Etcd`: 是用来存储所有`Kubernetes`集群状态的，它除了具备状态存储的功能，还有事件监听和订阅、Leader选举的功能。事件监听和订阅指，其他组件各个通信并不是互相调用API来完成的，而是把状态写入`Etcd`（相当于写入一个消息），其他组件通过监听`Etcd`的状态的的变化（相当于订阅消息），然后做后续的处理，然后再一次把更新的数据写入`Etcd`。Leader选举指，其它一些组件比如 Scheduler，为了做实现高可用，通过`Etcd`从多个（通常是3个）实例里面选举出来一个做Master，其他都是Standby。
2. `API Server`: `Etcd`是整个系统的最核心，所有组件之间通信都需要通过`Etcd`。实际上组件并不是直接访问`Etcd`，而是访问一个代理，这个代理是通过标准的RESTFul API，重新封装了对`Etcd`接口调用，除此之外，这个代理还实现了一些附加功能，比如身份的认证、缓存等，这个代理就是 API Server。
3. `Controller manager`: 负责任务调度，简单说直接请求`Kubernetes`做调度的都是任务，例如Deployment 、DeamonSet、Pod等等，每一个任务请求发送给`Kubernetes`之后，都是由`Controller Manager`来处理的，每一种任务类型对应一个`Controller Manager`，比如Deployment对应一个叫做Deployment Controller，DaemonSet对应一个DaemonSet Controller。
4. `Scheduler`: 负责资源调度，Controller Manager 会把Pod对资源要求写入到`Etcd`里面，`Scheduler`监听到有新的Pod需要调度，就会根据整个集群的状态，把Pod分配到具体的worker节点上。
5. `Kubectl`: 是一个命令行工具，它会调用`API Server`发送请求写入状态到`Etcd`，或者查询`Etcd`的状态。

## ② worker
worker节点组件运行在每个k8s Node上，提供K8s运行时环境，以及维护Pod。

1. `Kubelet`: 运行在每一个worker节点上的Agent，它会监听`Etcd`中的`Pod`信息，运行分配给它所在节点的`Pod`，并把状态更新回`Etcd`。通过`docker`部署
2. `Kube-proxy`: 负责为Service提供cluster内部的服务发现和负载均衡。通过`k8s`部署
3. `Docker`: Docker引擎，负责容器运行。
4. `Container`: 负责镜像管理以及Pod和容器的真正运行（CRI）。

我们可以通过部署一个多实例Nginx服务来描述Kubernets内部的流程
1. 创建一个nginx_deployment.yaml配置文件。
2. 通过`kubectl`命令行创建一个包含Nginx的`Deployment`对象，`kubectl`会调用`API Server`往`Etcd`里面写入一个Deployment对象。
3. `Deployment Controller`监听到有新的`Deployment`对象被写入，获取到`Deployment`对象信息然后根据对象信息来做任务调度，创建对应的`Replica Set`对象。
3. `Replica Set Controller`监听到有新的对象被创建，获取到`Replica Set`对象信息然后根据对象信息来做任务调度，创建对应的`Pod`对象。
4. `Scheduler`监听到有新的`Pod`被创建，获取到`Pod`对象信息，根据集群状态将`Pod`调度到某一个worker节点上，然后更新`Pod`。
5. `Kubelet`监听到当前的节点被指定了的`Pod`，就根据对象信息运行`Pod`。

## ③ K8s与Docker
K8s在初期版本里，就对多个容器引擎做了兼容，因此可以使用docker、rkt对容器进行管理。以docker为例，kubelet中会启动一个docker manager，通过直接调用docker的api进行容器的创建等操作。

在k8s 1.5版本之后，kubernetes推出了自己的运行时接口 [cri-api(container runtime interface](https://github.com/kubernetes/cri-api/)。cri接口的推出，隔离了各个容器引擎之间的差异，而通过统一的接口与各个容器引擎之间进行互动。cri不仅定义了容器的生命周期的管理，还引入了k8s中pod的概念，并定义了管理pod的生命周期。在kubernetes中，pod是由一组进行了资源限制的，在隔离环境中的容器组成。而这个隔离环境，称之为PodSandbox。kubernetes孵化了[containerd/cri](https://github.com/containerd/cri)项目，是cri-api的具体实现。初此之外，docker 自己把运行时相关的从 daemon 剥离出来，实现了 [containerd](https://github.com/containerd/containerd) 项目。

![](https://github.com/containerd/cri/blob/master/docs/cri.png?raw=true)

为了进一步与oci进行兼容，kubernetes还孵化了[cri-o](https://github.com/cri-o/cri-o)，成为了架设在cri和oci之间的一座桥梁。通过这种方式，可以方便更多符合oci标准的容器运行时，接入kubernetes进行集成使用。可以预见到，通过cri-o，kubernetes在使用的兼容性和广泛性上将会得到进一步加强。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/k8s-docker-cri-containerd-2.png?raw=true)
  
# 四. K8s核心概念
1. Cluster: 集群指的是由K8s使用一序列的物理机、虚拟机和其它基础资源来运行你的应用程序。
2. Node: 一个Node就是一个运行着K8s的物理机或虚拟机，并且Pod可以在其上面被调度。
3. Pod: 一个Pod对应一个由相关容器和卷组成的容器组。
4. Label: 一个label是一个被附加到资源上的键值对，比如附加到一个Pod上为它传递一个用户自定的属性，label还可以被应用来组织和选择子网中的资源。
5. Selector: 是一个通过匹配labels来定义资源之间关系的表达式，例如为一个负载均衡的service指定目标Pod。
6. Replication Controller: replication controller 是为了保证Pod一定数量的复制品在任何时间都能正常工作，它不仅允许复制的系统易于扩展，还会处理当Pod在机器重启或发生故障的时候再创建一个。
7. Service: 一个service定义了访问Pod的方式，就像单个固定的IP地址和与其相对应的DNS名之间的关系。
8. Volume: 一个Volume是一个目录。
9. Kubernets Volume: 构建在Docker Volumes之上，并且支持添加和配置Volume目录或者其他存储设备。
10. Secret: Secret存储了敏感数据，例如能运行容器接受请求的权限令牌。
11. Name: 用户为Kubernets中资源定义的名字。
12. Namespace: namespace好比一个资源名字的前缀，帮助不同的项目可以共享cluster，防止出现命名冲突。
13. Annotation: 相对于label来说可以容纳更大的键值对，它对我们来说是不可读的数据，只是为了存储不可识别的辅助数据，尤其是一些被工具或系统扩展用来操作的数据。

# 五. 基于Docker本地运行K8s
1. 最终的效果图如下，只有一个节点运行了K8s master和node相关的组件
   ![](https://raw.githubusercontent.com/chenguolin/chenguolin.github.io/master/data/image/docker-k8s.png?raw=true)
2. 下载最新的Docker for Mac，可以看到内置的K8s集群，直接点击安装既可在本地搭建好单节点的K8s环境
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker-run-k8s.png?raw=true)
3. 设置Docker使用的内存和CPU来保证有足够的内存和CPU运行K8s相关的容器
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker-advanced.png?raw=true)
4. K8s安装完成后查看自动安装的K8s相关的容器  
   `$ docker container ls --format "table{{.Names}}\t{{.Image }}\t{{.Command}}"`
5. 使用kuberctl安装kubernets-dashbord服务
   ```
   $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
   secret "kubernetes-dashboard-certs" created
   serviceaccount "kubernetes-dashboard" created
   role.rbac.authorization.k8s.io "kubernetes-dashboard-minimal" created
   rolebinding.rbac.authorization.k8s.io "kubernetes-dashboard-minimal" created
   deployment.apps "kubernetes-dashboard" created
   service "kubernetes-dashboard" created
   ```
6. 查看K8s部署的容器与服务  
   ```
   $ kubectl get deployments --namespace kube-system
     NAME                             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
     kube-dns                           1         1           1            1        2d
     kubernetes-dashboard               1         1           1            0        1m
 
   $ kubectl get services --namespace kube-system
     NAME                   TYPE          CLUSTER-IP     EXTERNAL-IP      PORT(S)           AGE
     kube-dns               ClusterIP     10.96.0.10       <none>        53/UDP,53/TCP       2d
     kubernetes-dashboard   NodePort      10.107.94.60     <none>        80:32297/TCP        2m
               
   $ kubectl get pods -n kube-system
     NAME                                         READY     STATUS    RESTARTS   AGE
     etcd-docker-for-desktop                      1/1       Running      4        3d
     kube-apiserver-docker-for-desktop            1/1       Running      16       2d
     kube-controller-manager-docker-for-desktop   1/1       Running      10       2d
     kube-dns-86f4d74b45-pz86h                    3/3       Running      0        3d
     kube-proxy-sdp6v                             1/1       Running      0        3d
     kube-scheduler-docker-for-desktop            1/1       Running      0        3d
     kubernetes-dashboard-7c867b6d97-t9zn5        1/1       Running      0        43s
   ```
7. 使用kuberctl启动proxy  
   ```
   $ kubectl proxy
    Starting to serve on 127.0.0.1:8001
   ```
8. 浏览器访问k8s dashbord  
   http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/K8s-dashboard.png?raw=true)
   
   需要使用Kubeconfig或者Token登录，我们这里使用Token登录  
   1). 创建一个admin-role.yaml文件，详见[admin-role.yaml](https://github.com/rootsongjc/kubernetes-handbook/blob/master/manifests/dashboard-1.7.1/admin-role.yaml)  
   2). 创建serviceaccount和角色绑定  
       `$ kubectl create -f admin-role.yaml`  
   3). 获取secret中的token  
      ```
      $ kubectl -n kube-system get secret |grep admin-token  
        admin-token-2s8jh     kubernetes.io/service-account-token   3         11s 
      $ kubectl -n kube-system describe secret admin-token-2s8jh | grep token
      ```
9. 删除kubernets-dashboard相关的deployment和service   
   `$ kubectl delete deployment kubernetes-dashboard --namespace=kube-system`   
   `$ kubectl delete service kubernetes-dashboard --namespace=kube-system`
10. 可以使用`kubectl`操作K8s，部署各种资源包括Pods、Services、Deployments、ReplicaSet等等

# 六. K8s技能图谱
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/k8s-skill.jpg?raw=true)


