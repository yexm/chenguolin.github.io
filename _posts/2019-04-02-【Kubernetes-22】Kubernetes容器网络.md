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

因此，这篇文章会通过 Docker网络、Kubernetes网络 2部分来介绍一下 Kubernetes容器网络，通过这篇文章可以知道在Docker环境下容器是如何通信的，在Kubernetes环境下Pod是如何通信的。

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
   bridge name	bridge id		    STP enabled	    interfaces
   docker0		8000.0242564f4703	no		        veth910802f
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
   
 

## ② 跨主机访问


# 三. Kubernetes网络
## ① flannel

## ② calico


