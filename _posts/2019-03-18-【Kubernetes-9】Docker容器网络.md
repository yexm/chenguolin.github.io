---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

docker run 创建Docker容器时，可以用`--net`选项指定容器的网络模式，默认为 bridge 模式。Docker有以下4种网络模式，使用 `docker network ls` 可以查看当前宿主机的网络模式，关于Docker网络可以参考 [Docker network](https://docs.docker.com/network/)。

1. `host`: `--net=host`
2. `container`: ，`--net=container:xxx`
3. `none`: `--net=none`
4. `bridge`: `--net=bridge` (Docker默认设置)

# 一. host模式
如果启动容器的时候使用 `host` 模式，那么这个容器将不会获得一个独立的 Network Namespace，而是和宿主机共用网络栈。容器将不会虚拟出自己的网卡，配置自己的 IP 等，而是使用宿主机的 IP 和端口。

例如我们在 `10.10.101.105` 的机器上用 host 模式启动一个含有 web 应用的 Docker 容器，监听 tcp 80 端口。当我们在容器中执行任何类似 ifconfig 命令查看网络环境时，看到的都是宿主机上的信息。而外界访问容器中的应用，则直接使用`10.10.101.105:80`即可，不用任何 NAT 转换，就如直接跑在宿主机中一样。

但是，容器的其他方面，如文件系统、进程列表等还是和宿主机隔离的。

`当使用host模式网络时比bridge模式更快（因为没有路由开销），但是它将容器直接暴露在公共网络中，是有安全隐患的。`

```
$ docker run -d --net=host ubuntu:14.04 tail -f /dev/null
$ ip addr | grep -A 2 eth0:
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
link/ether 06:58:2b:07:d5:f3 brd ff:ff:ff:ff:ff:ff
inet **10.0.7.197**/22 brd 10.0.7.255 scope global dynamic eth0

$ docker ps
CONTAINER ID    IMAGE         COMMAND      CREATED        STATUS        PORTS         NAMES
b44d7d5d3903  ubuntu:14.04    tail -f    2 seconds ago  Up 2 seconds                jovial_blackwell

$ docker exec -it b44d7d5d3903 ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
link/ether 06:58:2b:07:d5:f3 brd ff:ff:ff:ff:ff:ff
inet **10.0.7.197**/22 brd 10.0.7.255 scope global dynamic eth0
```

# 二. container模式
这个模式指定新创建的容器和已经存在的一个容器共享一个 Network Namespace，而不是和宿主机共享。新创建的容器不会创建自己的网卡，配置自己的 IP，而是和一个指定的容器共享 IP、端口范围等。  
同样，两个容器除了网络方面，其他的如文件系统、进程列表等还是隔离的。两个容器的进程可以通过 lo 网卡设备通信。

```
$ docker run -d -P --net=bridge nginx:1.9.1
$ docker ps
CONTAINER ID  IMAGE        COMMAND   CREATED         STATUS            PORTS               NAMES
eb19088be8a0  nginx:1.9.1  nginx -g  3 minutes ago   Up 3 minutes    0.0.0.0:32769->80/tcp, 0.0.0.0:32768->443/tcp    admiring_engelbart

$ docker exec -it admiring_engelbart ip addr
...
link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff
inet **172.17.0.3**/16 scope global eth0

$ docker run -it --net=container:admiring_engelbart ubuntu:14.04 ip addr
...
link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff
inet **172.17.0.3**/16 scope global eth0
```

# 三. none模式
这个模式和前两个不同。在这种模式下，Docker 容器拥有自己的 Network Namespace，但是，并不为 Docker 容器进行任何网络配置。  
也就是说，这个 Docker 容器没有网卡、IP、路由等信息。需要我们自己为 Docker 容器添加网卡、配置 IP 等。

`实际上，该模式关闭了容器的网络功能，在以下两种情况下是有用的：容器并不需要网络（例如只需要写磁盘卷的批处理任务）; 你希望自定义网络`

```
$ docker run -d -P --net=none nginx:1.9.1
$ docker ps
CONTAINER ID  IMAGE          COMMAND     CREATED       STATUS         PORTS     NAMES
d8c26d68037c  nginx:1.9.1    nginx -g  2 minutes ago Up 2 minutes             grave_perlman

$ docker inspect d8c26d68037c | grep IPAddress
"IPAddress": "",
"SecondaryIPAddresses": null,
```

# 四. bridge模式
`bridge 模式是 Docker 默认的网络设置，此模式会为每一个容器分配 Network Namespace、设置 IP 等，并将一个主机上的 Docker 容器连接到一个虚拟网桥上`。

当 Docker server 启动时，会在主机上创建一个名为 `docker0` 的虚拟网桥，此主机上启动的 Docker 容器会连接到这个虚拟网桥上。虚拟网桥的工作方式和物理交换机类似，这样主机上的所有容器就通过交换机连在了一个二层网络中。

接下来就要为容器分配 IP 了，Docker 会从 RFC1918 所定义的私有 IP 网段中，选择一个和宿主机不同的IP地址和子网分配给 docker0，连接到 docker0 的容器就从这个子网中选择一个未占用的 IP 使用。一般 Docker 会使用 `172.17.0.0/16` 这个网段，并将`172.17.0.1`分配给docker0网桥。

```
$ docker run -d -P --net=bridge nginx:1.9.1
$ docker ps
CONTAINER ID   IMAGE                  COMMAND       CREATED         STATUS         PORTS                  NAMES
17d447b7425d   nginx:1.9.1            nginx -g   19 seconds ago  Up 18 seconds  0.0.0.0:49153->443/tcp, 0.0.0.0:49154->80/tcp  trusting_feynman
```

每个Linux容器都有一个网络栈，包括`网卡（Network Interface）、回环设备（Loopback Device）、路由表（Routing Table）和 iptables 规则`。我们可以通过 host 模式直接使用宿主机网络栈的，虽然可以为容器提供良好的网络性能，但也会不可避免地引入共享网络资源的问题，比如端口冲突。`所以，在大多数情况下，我们都希望容器进程能使用自己 Network Namespace 里的网络栈，拥有属于自己的 IP 地址和端口`。

我们其实可以把每一个容器看做一台主机，它们都有一套独立的网络栈。如果想要实现两台主机之间的通信，最直接的办法，就是把它们用一根网线连接起来。而如果想要实现多台主机之间的通信，那就需要用网线，把它们连接在一台交换机上。在 Linux 中，能够起到虚拟交换机作用的网络设备，是网桥（Bridge）。它是一个工作在数据链路层（Data Link）的设备，主要功能是根据 MAC 地址学习来将数据包转发到网桥的不同端口（Port）上。

Docker 项目会默认在宿主机上创建一个名叫 `docker0` 的网桥，凡是连接在 docker0 网桥上的容器，都可以通过它来进行通信。我们需要使用一种名叫 `Veth Pair` 的虚拟设备了。Veth Pair 设备的特点是: 它被创建出来后，总是以两张虚拟网卡（Veth Peer）的形式成对出现的。并且，从其中一个"网卡"发出的数据包，可以直接出现在与它对应的另一张"网卡"上，哪怕这两个"网卡"在不同的 Network Namespace 里。`这就使得 Veth Pair 常常被用作连接不同 Network Namespace 的"网线"`。

比如，现在我们启动了一个叫作 nginx-1 的容器，并进入到这个容器中查看一下它的网络设备
```
$ docker run -it -d --name nginx-1 nginx
# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.7  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:ac:11:00:07  txqueuelen 0  (Ethernet)
        RX packets 6512  bytes 8922774 (8.5 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 2578  bytes 140938 (137.6 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         172.17.0.1      0.0.0.0         UG    0      0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 eth0
```

可以看到，这个容器里有一张叫作 eth0 的网卡，它正是一个 Veth Pair 设备在容器里的这一端。通过 route 命令查看 nginx-1 容器的路由表，我们可以看到，这个 eth0 网卡是这个容器里的默认路由设备，所有对 172.17.0.0/16 网段的请求，也会被交给 eth0 来处理（第二条 172.17.0.0 路由规则）。

而这个 Veth Pair 设备的另一端，则在宿主机上，你可以通过查看宿主机的网络设备看到它，如下所示。
```
# ifconfig
veth821413b Link encap:Ethernet  HWaddr 16:42:A8:1F:B9:D9
          inet6 addr: fe80::1442:a8ff:fe1f:b9d9/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:18 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:1320 (1.2 KiB)
```

通过 ifconfig 命令的输出，你可以看到，nginx-1 容器对应的 Veth Pair 设备，在宿主机上是一张虚拟网卡。它的名字叫作 veth9c02e56。并且，通过 `brctl show` 的输出，你可以看到这张网卡被插在了 docker0 上。

这时候，如果我们再在这台宿主机上启动另一个 Docker 容器，比如 nginx-2，会发现一个新的、名叫 veth07a9016 的虚拟网卡，也被插在了 docker0 网桥上。
```
$ docker run –d --name nginx-2 nginx
$ brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.0242735ad13a       no              veth821413b
                                                        veth07a9016
```

在 nginx-1 容器里 ping 一下 nginx-2 容器的 IP 地址（172.17.0.3），会发现同一宿主机上的两个容器默认就是相互连通的。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker-network-1.png?raw=true)

`熟悉了 docker0 网桥的工作方式，我们就可以理解，在默认情况下，被限制在 Network Namespace 里的容器进程，实际上是通过 Veth Pair 设备 + 宿主机网桥的方式，实现了跟同其他容器的数据交换。`

与之类似地，当你在一台宿主机上，访问该宿主机上的容器的 IP 地址时，这个请求的数据包，也是先根据路由规则到达 docker0 网桥，然后被转发到对应的 Veth Pair 设备，最后出现在容器里。这个过程的示意图，如下所示

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker-network-2.png?raw=true)

同样地，当一个容器试图连接到另外一个宿主机时，比如 ping 10.168.0.3，它发出的请求数据包，首先经过 docker0 网桥出现在宿主机上。然后根据宿主机的路由表里的直连路由规则（10.168.0.0/24 via eth0)），对 10.168.0.3 的访问请求就会交给宿主机的 eth0 处理。所以接下来，这个数据包就会经宿主机的 eth0 网卡转发到宿主机网络上，最终到达 10.168.0.3 对应的宿主机上。当然，这个过程的实现要求这两台宿主机本身是连通的。这个过程的示意图，如下所示

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker-network-3.png?raw=true)

`所以说，当你遇到容器连不通"外网"的时候，你都应该先试试 docker0 网桥能不能 ping 通，然后查看一下跟 docker0 和 Veth Pair 设备相关的 iptables 规则是不是有异常，往往就能够找到问题的答案了。`


