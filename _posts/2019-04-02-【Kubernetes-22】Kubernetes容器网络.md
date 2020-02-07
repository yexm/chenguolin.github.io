---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 概述
我们知道任何一个系统都需要有网络，Kubernetes也一样。从之前的文章我们知道 Linux 容器能看见的`网络栈`，实际上是被隔离在它自己的 Network Namespace 当中的。而所谓`网络栈`，就包括了 网卡（Network Interface）、回环设备（Loopback Device）、路由表（Routing Table）和 iptables 规则。对于一个进程来说，这些要素，其实就构成了它发起和响应网络请求的基本环境。

所以，在大多数情况下，我们都希望容器进程能使用自己 Network Namespace 里的网络栈，也就是拥有属于自己的 IP 地址和端口。那问题来了，这个被隔离的容器进程，该如何跟其他 Network Namespace 里的容器进程进行交互呢？

因此，这篇文章会通过 Docker网络 部分介绍单宿主机容器之间通信、跨宿主机容器之间通信，通过 Kubernetes 网络 部分介绍 Kubernetes的网络模型以及社区比较有名的网络解决方案。

# 二. Docker网络
如果我们想要实现两台主机之间的通信，最直接的办法就是把它们用一根网线连接起来，而如果我们想要实现多台主机之间的通信，那就需要用网线，把它们连接在一台交换机上。在 Linux 中能够起到虚拟交换机作用的网络设备是网桥（Bridge），它是一个工作在数据链路层（Data Link）的设备，主要功能是根据 MAC 地址学习来将数据包转发到网桥的不同端口（Port）上。

通过之前的文章 [Docker容器网络](https://chenguolin.github.io/2019/03/18/Kubernetes-9-Docker%E5%AE%B9%E5%99%A8%E7%BD%91%E7%BB%9C/) 我们知道 Docker 有4种网络模式，默认是 bridge 的方式，也就是 Docker 会默认在宿主机上创建一个名叫 docker0 的网桥，凡是连接在 docker0 网桥上的容器，就可以通过它来进行通信。默认情况下 Docker 会使用 172.17.0.0/16 这个网段，并将 172.17.0.1 分配给 docker0 网桥。

下面，我们默认以 bridge 模式分为 单机访问、跨主机访问 这2种情况来讨论下Docker容器之间如何通信。

## ① 单机访问
`单机访问`指的是同一台宿主机上的容器如何进行互相访问，我们知道当 Docker engine 启动时，会在主机上创建一个名为 docker0 的虚拟网桥，此主机上启动的 Docker 容器会连接到这个虚拟网桥上。虚拟网桥的工作方式和物理交换机类似，这样主机上的所有容器就通过交换机连在了一个二层网络中。

把容器连接到 docker0 虚拟网络则是通过 Veth Pair（Virtual Ethernet Device Pairs）的虚拟设备。Veth Pair 设备的特点是它被创建出来后，总是以两张虚拟网卡（Veth Peer）的形式成对出现的，一个在容器内部默认为 eth0，另一个在宿主机被绑定在 docker0 网桥上。并且从其中一个`网卡`发出的数据包，可以直接出现在与它对应的另一张`网卡`上，这两张网卡允许在不同 Network Namespace 里，这就使得 Veth Pair 常常被用作连接不同 Network Namespace 的`网线`。

因此，根据以上的信息我们知道同一个宿主机上的网络可以通过 `Veth Pair + docker0` 实现网络互通，它的通信原理如下图所示

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker-network-1.png?raw=true)

基于以上的内容，我们做一下实验验证

1. 先查看当前宿主机是否有 docker0 网桥 (默认IP为172.17.0.1)
   ```
   $ ifconfig | grep docker0 -A 10
   docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:63:d6:bb:70  txqueuelen 0  (Ethernet)
        RX packets 10  bytes 541 (541.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 4  bytes 634 (634.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

   ```

2. 先启动一个容器 nginx-1 （容器IP为172.17.0.2，容器内eth0 它正是一个 Veth Pair 设备在容器里的这一端）
   ```
   $ docker run -d --name nginx-1 nginx:1.7.9
   65d408db681dbb53f79d543e21b104cb5df9a85b3bb81e2849b6578abf3166c5
   
   $ docker exec -it nginx-1 /bin/sh
   / # ifconfig                          //查看容器内网络配置
   eth0   Link encap:Ethernet  HWaddr 02:42:AC:11:00:02
          inet addr:172.17.0.2  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

   lo     Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

   / # route                            //查看容器路由表
   Kernel IP routing table
   Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
   default         172.17.0.1      0.0.0.0         UG    0      0        0 eth0
   172.17.0.0      *               255.255.0.0     U     0      0        0 eth0
   
   通过 route 命令查看 nginx-1 容器的路由表，eth0 网卡是这个容器里的默认路由设备
   所有对 172.17.0.0/16 网段的请求，也会被交给 eth0 来处理（第二条 172.17.0.0 路由规则）
   ```

3. 查看物理机网络设备（veth95dc570是 nginx-1 容器对应的 Veth Pair 设备在宿主机上的虚拟网卡）
   ```
   $ ifconfig | grep veth -A 10
   veth95dc570: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether 72:fa:55:08:60:de  txqueuelen 0  (Ethernet)
        RX packets 12  bytes 750 (750.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 9  bytes 609 (609.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
   ```

4. 再启动另一个容器 nginx-2  （容器IP为172.17.0.3，容器内eth0 它正是一个 Veth Pair 设备在容器里的这一端）
   ```
   $ docker run -d --name nginx-2 nginx:1.7.9
   9b5a2b85d2be20a27c5e751388f07a517d3bd81c33aaa9afb2e2ed13360c8afa
   
   $ docker exec -it nginx-2 /bin/sh
   / # ifconfig                         //查看容器内网络配置
   eth0   Link encap:Ethernet  HWaddr 02:42:AC:11:00:03
          inet addr:172.17.0.3  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

   lo     Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
          
   / # route                            //查看容器路由表
   Kernel IP routing table
   Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
   default         172.17.0.1      0.0.0.0         UG    0      0        0 eth0
   172.17.0.0      *               255.255.0.0     U     0      0        0 eth0
   
   通过 route 命令查看 nginx-1 容器的路由表，eth0 网卡是这个容器里的默认路由设备
   所有对 172.17.0.0/16 网段的请求，也会被交给 eth0 来处理（第二条 172.17.0.0 路由规则）
   ```
   
5. 再查看物理机网络设备（veth910802f是 nginx-2 容器对应的 Veth Pair 设备在宿主机上的虚拟网卡）
   ```
   $ ifconfig | grep veth -A 10
   veth910802f: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether f2:9b:e5:ba:74:df  txqueuelen 0  (Ethernet)
        RX packets 4  bytes 250 (250.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 3  bytes 167 (167.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

   veth95dc570: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether 72:fa:55:08:60:de  txqueuelen 0  (Ethernet)
        RX packets 12  bytes 750 (750.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 10  bytes 651 (651.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
   ```

6. 查看下 docker0 网桥信息（发现veth910802f和veth95dc570都成功绑定在docker0网桥上了）
   ```
   $ brctl show
   bridge name  bridge id            STP enabled      interfaces
   docker       8000.0242564f4703    no               veth910802f
                                                      veth95dc570
   ```

7. 所以我们可以在容器1 nginx-1 和 容器2 nginx-2 内部互相访问对方
   ```
   $ docker exec -it nginx-1 /bin/sh
   / # ping 172.17.0.3
   PING 172.17.0.3 (172.17.0.3): 56 data bytes
   64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.087 ms
   ^C
   --- 172.17.0.3 ping statistics ---
   1 packets transmitted, 1 packets received, 0% packet loss
   round-trip min/avg/max = 0.087/0.087/0.087 ms
   
   $ docker exec -it nginx-2 /bin/sh
   / # ping 172.17.0.2
   PING 172.17.0.2 (172.17.0.2): 56 data bytes
   64 bytes from 172.17.0.2: seq=0 ttl=64 time=0.094 ms
   ^C
   --- 172.17.0.2 ping statistics ---
   1 packets transmitted, 1 packets received, 0% packet loss
   round-trip min/avg/max = 0.094/0.094/0.094 ms
   ```
   
这其中的原理其实非常简单，具体的流程如下

1. nginx-1 容器里访问 nginx-2 容器的 IP 地址（ping 172.17.0.3）的时候，这个目的 IP 地址会匹配到 nginx-1 容器里的第二条路由规则（172.17.0.0  *  255.255.0.0  U  0 0  0  eth0）。
2. 根据这条路由规则，凡是匹配到这条规则的 IP 包，应该经过本机的 eth0 网卡，通过二层网络直接发往目的主机。我们前面提到过容器nginx-1 eth0 网卡是一个 Veth Pair 在容器内的一端，而另一端则位于宿主机上被绑定 docker0 网桥上即 veth95dc570。
3. 数据包会从容器nginx-1 eho0网卡流入宿主机 docker0 网桥上veth95dc570，docker0 处理转发的过程会扮演二层交换机的角色。docker0 网桥根据数据包的目的 MAC 地址（也就是 nginx-2 容器的 MAC 地址），在它的 CAM 表（即交换机通过 MAC 地址学习维护的端口和 MAC 地址的对应表）里查到对应的端口（Port）为veth910802f，然后把数据包发往这个端口。
4. veth910802f是 nginx-2 容器绑定在 docker0 网桥上一个 Veth Pair 设备的虚拟网卡。因此，数据包就进入到了 nginx-2 容器的 Network Namespace 里。
5. 所以nginx-2 容器看到它自己的 eth0 网卡上出现了流入的数据包，容器 nginx-2 的网络协议栈就会对请求进行处理，最后将响应返回到容器 nginx-1。

所以单宿主机容器要想跟外界进行通信，它发出的 IP 包就必须从它的 Network Namespace 里出来，来到宿主机上。而解决这个问题的方法就是为容器创建一个一端在容器里充当默认网卡、另一端在宿主机上 docker0 网桥的 Veth Pair 设备。

## ② 跨主机访问
`跨主机访问`指的是不同宿主机上的容器如何进行互相访问，Docker 默认配置下一台宿主机上的 docker0 网桥，和其他宿主机上的 docker0 网桥，它们互相之间是没办法连通。所以，不同宿主机上的容器通过 IP 地址进行互相访问是做不到的。

Docker 支持[overlay](https://docs.docker.com/network/overlay/) 驱动，在已有的宿主机网络上再通过软件构建一个覆盖在已有宿主机网络之上的可以把所有容器连通在一起的虚拟网络。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker-overlay-network.png?raw=true)

实际在生产环境中很少只使用 Docker 来搭建应用部署环境，目前用的最多的是通过 Kubernetes+Docker（容器运行时） 来搭建应用部署环境，因此跨主机容器网络访问方案会和Kubernetes结合，下文会仔细介绍社区几个容器网络方案。

# 三. Kubernetes网络
我们知道 Kubernetes 创建一个 Pod 的第一步就是创建并启动一个 Infra 容器，然后业务容器加入 Infra 容器的 Network Namespace。从 Docker 网络我们了解到 Docker 默认会创建 docker0 网桥，所有容器都连接在 docker0 网桥上。但是 Kubernetes 则是通过一个叫作 CNI 的接口，维护了一个单独的网桥来代替 docker0，这个网桥的名字就叫作 CNI 网桥，它在宿主机上的设备名称默认是 cni0。

注意 CNI 网桥只是接管所有 CNI 插件负责的、即 Kubernetes 创建的容器（Pod）。如果我们还是用 docker run 单独启动一个容器，那么 Docker 项目还是会把这个容器连接到 docker0 网桥上，所以这个容器的 IP 地址还是属于 docker0 网桥的 172.17.0.0/16 网段。

Kubernetes 之所以要设置这样一个与 docker0 网桥功能几乎一样的 CNI 网桥，主要原因包括两个方面
1. Kubernetes 项目并没有使用 Docker 的网络模型（CNM），所以它并不希望、也不具备配置 docker0 网桥的能力
2. 与 Kubernetes 如何配置 Pod，也就是 Infra 容器的 Network Namespace 密切相关。CNI 的设计思想就是 Kubernetes 在启动 Infra 容器之后，就可以直接调用 CNI 网络插件，为这个 Infra 容器的 Network Namespace，配置符合预期的网络栈。

社区比较出名的 CNI 网络插件有以下几种 [Flannel](https://github.com/coreos/flannel)、[Calico](https://github.com/projectcalico/calico)、[Canal](https://github.com/projectcalico/canal)

## ① Flannel
Flannel 项目是 CoreOS 公司主推的容器网络方案，Flannel 项目本身只是一个框架真正为我们提供容器网络功能的是 Flannel 的后端实现。目前 Flannel 支持三种后端实现分别是 VXLAN、host-gw 和 UDP。



## ② Calico


