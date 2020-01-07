---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 背景
前面关于Docker的文章里面我们提到了 `Docker容器本质是进程，它主要由Namespace、Cgroups、Rootfs这三种技术实现。Docker是最有名的容器运行时，它可以在宿主机上启动很多容器`。在 [kubernetes-对象简单介绍](https://chenguolin.github.io/2019/03/25/Kubernetes-14-Kubernetes%E5%AF%B9%E8%B1%A1%E7%AE%80%E5%8D%95%E4%BB%8B%E7%BB%8D/) 文章里面提到 `Pod是Kubernetes最小的调度单位`，可能大家都会很好奇 Pod 到底是什么，它和 Docker 的关系是怎么样的。所以，这篇文章会介绍 Kubernetes Pod，了解 Pod 是什么。

让我们来考虑一个使用场景 `假设有2个进程 P1和P2，为了协同工作，要求P1和P2必须部署在同一台机器，同时要能够网络互通，以及共享文件存储`，针对这种场景我们看下如果使用 Docker 部署的话该怎么实现。

显然我们需要运行2个容器，每个容器内跑进程P1和P2，两个容器需要使用相同的网络配置，同时需要挂载相同的Volume。另外我们还需要考虑2个容器的启动顺序，因为可能进程间会有启动依赖。

```这里不推荐一个容器内同时运行P1和P2，因为容器是单进程模型，也就是说容器没有管理多个进程的能力。这是因为容器里 PID=1 的进程就是应用本身，其他的进程都是这个 PID=1 进程的子进程。如果PID=1进程不会给子进程传系统信号的话，子进程是没有办法正常接收到系统信号的，也就是没有办法优雅退出，存在风险，详细可以参考 https://chenguolin.github.io/2019/06/22/Kubernetes-30-Kubernetes容器应用优雅退出机制/```

Docker的实现方案虽然可行，但是很麻烦，因为我们不仅要考虑多个容器共享网络、共享存储，还要考虑容器的启动顺序。但是到了 Kubernetes ，这样的问题就迎刃而解了，`Pod 是 Kubernetes 里的原子调度单位。这就意味着，Kubernetes 项目的调度器，是统一按照 Pod 而非容器的资源需求进行计算的`。

所以，在 Kubernetes 里面实现方案为 把2个容器绑定在同一个Pod上，同一个Pod 里的所有容器，共享同一个 Network Namespace，并且可以声明共享同一个 Volume。

# 二. Pod
Kubernetes 中 Pod 的实现需要使用一个中间容器，这个容器叫作 Infra 容器。在这个 Pod 中，Infra 容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过 Join Network Namespace 的方式，与 Infra 容器关联在一起。

在 Kubernetes 项目里，Infra 容器一定要占用极少的资源，所以它使用的是一个非常特殊的镜像，叫作 `k8s.gcr.io/pause`。这个镜像是一个用汇编语言编写的、永远处于 暂停 状态的容器，解压后的大小也只有 100~200 KB 左右。当 Infra 容器设置了 Network Namespace 后，用户容器就可以加入到 Infra 容器的 Network Namespace 当中了。

我们可以来验证一下

1. 查看 apiserver pod
   ```
   $ kubectl get pods -n kube-system -o wide | grep kube-apiserver-ecs-s6-large-2-linux-20200105130533
   kube-apiserver-ecs-s6-large-2-linux-20200105130533            1/1     Running   2          33h   192.168.0.14    ecs-s6-large-2-linux-20200105130533   <none>           <none>
   ```

2. 登录对应机器查看 docker容器进程  （发现有2个进程，其中5a894a10c697进程确实用pause镜像启动，为什么不是用k8s.gcr.io/pause。因为国内无法下载，用阿里云的镜像替代）
   ```
   $ docker ps -a | grep apiserver
   7589296a202d        0cae8d5cc64c                                        "kube-apiserver --ad…"   29 hours ago        Up 29 hours                                     k8s_kube-apiserver_kube-apiserver-ecs-s6-large-2-linux-20200105130533_kube-system_212183bed1727623dc1edaf897053c50_2
   5a894a10c697        registry.aliyuncs.com/google_containers/pause:3.1   "/pause"                 29 hours ago        Up 29 hours                                     k8s_POD_kube-apiserver-ecs-s6-large-2-linux-20200105130533_kube-system_212183bed1727623dc1edaf897053c50_2
   ```
   
3. 查看两个进程对应的 network namespace
   ```
   $ docker inspect 7589296a202d | grep Pid
     "Pid": 2861,
   $ ls -lsrt /proc/2861/ns/
   total 0
   0 lrwxrwxrwx 1 root root 0 1月   7 19:20 uts -> uts:[4026531838]
   0 lrwxrwxrwx 1 root root 0 1月   7 19:20 user -> user:[4026531837]
   0 lrwxrwxrwx 1 root root 0 1月   7 19:20 pid -> pid:[4026532263]
   0 lrwxrwxrwx 1 root root 0 1月   7 19:20 net -> net:[4026531956]
   0 lrwxrwxrwx 1 root root 0 1月   7 19:20 mnt -> mnt:[4026532262]
   0 lrwxrwxrwx 1 root root 0 1月   7 19:20 ipc -> ipc:[4026532246]
   
   $ docker inspect 5a894a10c697  | grep Pid
     "Pid": 2651
   $ ls -lsrt /proc/2651/ns/
   total 0
   0 lrwxrwxrwx 1 root root 0 1月   7 19:21 uts -> uts:[4026532245]
   0 lrwxrwxrwx 1 root root 0 1月   7 19:21 user -> user:[4026531837]
   0 lrwxrwxrwx 1 root root 0 1月   7 19:21 pid -> pid:[4026532247]
   0 lrwxrwxrwx 1 root root 0 1月   6 13:47 net -> net:[4026531956]
   0 lrwxrwxrwx 1 root root 0 1月   7 19:21 mnt -> mnt:[4026532244]
   0 lrwxrwxrwx 1 root root 0 1月   6 13:47 ipc -> ipc:[4026532246]
   ```
   
4. 通过第3步，我们可以确认容器进程 7589296a202d 和 5a894a10c697 对应的 user、net、ipc namespace相同，因此证实了 Infra容器 和 用户容器确实是共享network namespace。

`这也就意味着，如果 Pod 里的有多个容器，那么这些容器可以直接使用 localhost 进行通信，同时它们看到的网络设备跟 Infra 容器看到的完全一样。`

了解

# 三. 使用

# 四. 源码
