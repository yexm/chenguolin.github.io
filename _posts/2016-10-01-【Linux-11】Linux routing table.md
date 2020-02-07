---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Linux
---

# 一. routing table
routing table 指的是路由表，所有的网络设备无论是主机、路由器还是其他设备都需要有配置决定如何将 TCP/IP 数据包路由到何处，路由表正是用来存储这些配置的。路由表包含一序列路由规则，当收到一个数据包的时候它会根据路由表规则决定把数据包发送到哪里去。除此之外，路由表还包含到目的地址的距离信息。

Linux routing table 文件存储在内存中，路由表规则可以动态配置也可以静态配置。动态配置是通过动态路由协议自学习的方式，例如[OSPF and BGP](https://www.lartc.org/howto/lartc.dynamic-routing.html)，静态配置一般由网络管理员配置。由于内存有限没有办法存储大量设备的路由信息，因此路由表中的路由规则使用的是IP段也就是[Classless Inter-Domain Routing (CIDR)](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing)的方式。

Linux 下我们可以使用 `route` 命令查看当前内核的路由表的内容，如下所示

```
$ route
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         192.168.65.1    0.0.0.0         UG    0      0        0 eth0
10.1.0.0        *               255.255.0.0     U     0      0        0 cni0
127.0.0.0       *               255.0.0.0       U     0      0        0 lo
172.17.0.0      *               255.255.0.0     U     0      0        0 docker0
192.168.65.0    *               255.255.255.0   U     0      0        0 eth0
```

其中重要的几个字段含义如下
1. Destination: 目标网络或目标主机
2. Gateway: 网关地址 (*表示直连，不需要经过网关)
3. Genmask: 子网掩码
4. Flags: 路由标识 `U`表示路由规则可用，`G`表示网关，`H`表示目标是一个主机，`!`表示路由规则关闭
5. Iface: 网卡名称

这几行配置对应的含义如下
1. 第一行: 默认路由规则，没有匹配到路由表规则的情况下使用默认路由规则，把数据包通过eth0发送到网关192.168.65.1
2. 第二行: 所有目标IP为10.1.0.0/16的数据包，都会通过cni0直接发送出去（不需要经过网关）
3. 第三行: 所有目标IP为127.0.0.0/8的数据包，都会通过lo直接发送出去（不需要经过网关）
4. 第四行: 所有目标IP为172.17.0.0/16的数据包，都会通过docker0直接发送出去（不需要经过网关）
5. 第五行: 所有目标IP为192.168.65.0/24的数据包，都会通过eth0直接发送出去（不需要经过网关）

所以，正常来说路由表的路由规则很简单，简单的可以分为以下2类
1. 如果目标网络地址是本地网络（局域网），则直接通过网卡发送给目标地址
2. 如果目标网络地址是远程网络，则直接通过网卡发送到网关，通过网关转发

# 二. route command
类 Unix 系统 [route](https://en.wikipedia.org/wiki/Route_(command)) 命令用于显示和操作路由表路由规则，属于静态路由配置。直接在命令行下执行 route 命令来添加路由规则是不会永久保存，当网卡重启或者机器重启之后该路由规则就失效了，可以在 /etc/rc.local 中添加 route 命令来保证该路由规则永久有效。

route 命令的语法如下
```
route [-nNvee] [-FC] [<AF>]           查看路由表规则
route [-v] [-FC] {add|del|flush} ...  更新路由表规则
route {-h|--help} [<AF>]              使用帮助
route {-V|--version}                  版本信息
```

1. 查看路由表规则
   ```
   $ route
   Kernel IP routing table
   Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
   default         192.168.65.1    0.0.0.0         UG    0      0        0 eth0
   10.1.0.0        *               255.255.0.0     U     0      0        0 cni0
   127.0.0.0       *               255.0.0.0       U     0      0        0 lo
   172.17.0.0      *               255.255.0.0     U     0      0        0 docker0
   192.168.65.0    *               255.255.255.0   U     0      0        0 eth0

   $ route -n
   Kernel IP routing table
   Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
   0.0.0.0         192.168.65.1    0.0.0.0         UG    0      0        0 eth0
   10.1.0.0        0.0.0.0         255.255.0.0     U     0      0        0 cni0
   127.0.0.0       0.0.0.0         255.0.0.0       U     0      0        0 lo
   172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
   192.168.65.0    0.0.0.0         255.255.255.0   U     0      0        0 eth0
   
   $ route -e
   Kernel IP routing table
   Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
   default         192.168.65.1    0.0.0.0         UG        0 0          0 eth0
   10.1.0.0        *               255.255.0.0     U         0 0          0 cni0
   127.0.0.0       *               255.0.0.0       U         0 0          0 lo
   172.17.0.0      *               255.255.0.0     U         0 0          0 docker0
   192.168.65.0    *               255.255.255.0   U         0 0          0 eth0
   ```

2. 更新路由表规则
   ```
   1). 添加路由规则
   $ route add -net 192.56.76.0 netmask 255.255.255.0 dev eth0
   $ route -n
   route
   Kernel IP routing table
   Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
   default         192.168.65.1    0.0.0.0         UG    0      0        0 eth0
   192.56.76.0     *               255.255.255.0   U     0      0        0 eth0
   ...
   
   2). 删除路由规则
   $ route del -net 192.56.76.0 netmask 255.255.255.0
   $ route -n
   route
   Kernel IP routing table
   Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
   default         192.168.65.1    0.0.0.0         UG    0      0        0 eth0
   ...
   ```


