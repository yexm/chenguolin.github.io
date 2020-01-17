---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Linux
---

# 一. 概述
在介绍iptables之前，我们先来了解一下`防火墙`相关的概念，在计算机领域 "防火墙" 指的是网络防护。

1. 从逻辑上讲防火墙可以分为 主机防火墙 和 网络防火墙。主机防火墙一般是针对单个主机进行防护，网络防火墙则是处在网络的入口处，针对网络入口进行防护。
2. 从物理上讲防火墙可以分为 硬件防火墙 和 软件防火墙。硬件防火墙是在硬件级别实现防火墙功能，软件防火墙则是通过软件处理逻辑实现防火墙功能。

任何一个Linux发行版本 Red Hat、CentOS、Fedora Linux 等都内置了防火墙的功能，Linux内置的防火墙可以对IP数据包做一系列如过滤、更改、转发这样的操作，Linux内置的防火墙实际上由 `netfilter` 和 `iptables` 两个组件组成，属于主机级别的软件防火墙。

1. `netfilter` 组件位于内核空间，由一些数据包过滤表组成，这些表包含内核用来控制数据包过滤处理的规则集。
2. `iptables` 组件位于用户空间，实际上是一个命令行工具，它使插入、修改和删除数据包过滤表中的规则变得容易。

所以，Linux内置的防火墙实际上是通过 iptables 控制 netfilter 内核数据包过滤表中的规则，从而达到对IP数据包过滤、更改、转发的目的。

# 二. iptables简介
[iptables](https://en.wikipedia.org/wiki/Iptables) 是一个用户空间的命令行工具，通过 iptables 创建新规则，其实就是往 netfilter 中插入一个hook，从而实现修改数据包、控制数据包流向等。iptables 适用于用于 IPv4 数据包，如果是 IPv6 则需要使用 ip6tables。iptables 主要由 规则表（table）、规则链（chain）以及规则（rule）三部分组成。

`table` 指的是规则表，用于存储具有相同功能的规则，不同的功能的规则放置在不同的 table 中。iptables 内置了5个 table，filter、nat、mangle、raw、security，最常用的是 filter 、nat 和 mangle 这3个 table，这3个 table 主要的作用如下。

1. `filter`: 负责IP数据包过滤，是 iptables 命令默认查看的 table，支持 `INPUT`、`FORWARD`、`OUTPUT` 3条规则链。（我们通常所说的防火墙指的是filter表的过滤规则）
2. `nat`: 负责网络地址转换即Network Address Translation，包括Source NAT 和 Destination NAT，支持 `PREROUTING`、`OUTPUT`、`POSTROUTING` 3条规则链。
3. `mangle`: 负责修改IP数据包，支持 `PREROUTING`、`INPUT`、`FORWARD`、`POSTROUTING`、`OUTPUT` 5条规则链。

`chain` 指的是规则链，iptables内置了5条规则链，用于存储具有相同功能的规则，不同表共享这些规则链。注意每个IP数据包经过一条规则链的时候，都需要将当前链的所有规则检测一遍。`除了内置的5条规则链，用户还可以在table里面创建自定义的规则链，但是自定义规则链不能直接使用，只能被当做某个内置链的执行动作才能起作用。因为，自定义的规则链都是针对应用程序进行定制的，通过链接到iptables默认的规则链起作用。`

1. `INPUT`: 发送到当前机器的IP数据包，都要经过INPUT规则链所有规则进行检测一遍，存放 filter、mangle 表的规则
2. `FORWARD`: 通过当前机器转发的IP数据包（目的地不是当前机器 同时 也不是当前机器生成的），都要经过FORWARD规则链所有规则进行检测一遍，存放 filter、mangle 表的规则
3. `OUTPUT`: 当前机器生成的IP数据包，都要经过OUTPUT规则链所有规则进行检测一遍，存放 filter、nat、mangle 表的规则
4. `PREROUTING`: 当前机器在接收IP数据包之前，都会经过PREROUTING规则链所有规则进行检测一遍，存放 nat、mangle 表的规则
5. `POSTROUTING`: 当前机器在发送IP数据包之后，都会经过POSTROUTING规则链所有规则进行检测一遍，存放 nat、mangle 表的规则

`rule` 指的具体的规则，规则其实就是用户自定义的检测条件，表示如果IP数据包符合某个条件就执行某种处理动作。规则实际是存储在内核空间的 netfilter 的包过滤表中，如果IP数据包与规则匹配，则会根据规则定义的执行动作来进行处理，例如接收（ACCEPT）、丢弃（DROP）、返回（RETURN）或自定义的执行动作。

1. `ACCEPT`: 接收IP数据包
2. `DROP`: 直接丢弃IP数据包
3. `RETURN`: 停止当前规则检测，检测当前链下一个规则
4. `SNAT`: 源地址转换（局域网内所有用户共用同一个公网IP地址）
5. `DNAT`: 目标地址转换

基于上诉内容，规则表（table）、规则链（chain）、规则（rule） 这3者的关系为

1. table 用于存储具有相同功能的规则，每个 table 可以包含多个规则
2. iptables 内置5条规则链，所有 table 共享这5条规则链
3. 每条规则链包含一序列规则，每个IP数据包都要经过所有的规则进行检测

所以，根据以上的介绍，我们IP数据包的流向如下图所示
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/iptables-4.png?raw=true)

# 三. iptables命令
iptables 命令如下，可以通过 `man iptables` 进行查看，因为 iptables 是一个命令行工具而非后台守护进程，因此我们不需要通过 systemctl 或者 service 等命令进行启动（不过，可能有些Linux发行版会需要），Linux已经内置了，通过 iptables 命令修改完规则后立即生效。

```
iptables v1.4.21

Usage: iptables -[ACD] chain rule-specification [options]
       iptables -I chain [rulenum] rule-specification [options]
       iptables -R chain rulenum rule-specification [options]
       iptables -D chain rulenum [options]
       iptables -[LS] [chain [rulenum]] [options]
       iptables -[FZ] [chain] [options]
       iptables -[NX] chain
       iptables -E old-chain-name new-chain-name
       iptables -P chain target [options]
       iptables -h (print this help information)
       
Commands:
  --append     -A chain              Append to chain                                  //新增规则到某条规则链
  --check      -C chain              Check for the existence of a rule                //检查规则是否存在
  --delete     -D chain              Delete matching rule from chain                  //删除某条规则
  --delete     -D chain rulenum      Delete rule rulenum (1 = first) from chain       //删除某条规则（根据编号）
  --insert     -I chain [rulenum]    Insert in chain as rulenum (default 1=first)     //插入新的规则
  --replace    -R chain rulenum      Replace rule rulenum (1 = first) in chain        //替换某条规则（根据编号）
  --list       -L [chain [rulenum]]  List the rules in a chain or all chains          //列出所有规则
  --list-rules -S [chain [rulenum]]  Print the rules in a chain or all chains         //打印所有规则
  --flush      -F [chain]            Delete all rules in  chain or all chains         //删除所有规则
  --zero       -Z [chain [rulenum]]  Zero counters in chain or all chains 
  --new        -N chain              Create a new user-defined chain                  //创建用户新的自定义规则链
  --delete-chain  -X [chain]         Delete a user-defined chain                      //删除用户自定义的规则链
  --policy        -P chain target    Change policy on chain to target                 //变更规则链的执行动作 ACCEPT 或 DROP
  --rename-chain                     Change chain name, (moving any references)       //重命名规则链
				
                
Options:
  --ipv4          -4                       Nothing (line is ignored by ip6tables-restore)    
  --ipv6          -6                       Error (line is ignored by iptables-restore)
  --protocol      -p   proto               protocol: by number or name, eg. `tcp'                //协议类型 tcp, udp, icmp, ssh 等
  --source        -s   address[/mask][...] source specification                                  //源地址
  --destination   -d   address[/mask][...] destination specification                             //目的地址
  --in-interface  -i   input               name[+] network interface name ([+] for wildcard)     //输入网卡名称
  --jump          -j   target              target for rule (may load target extension)           //执行动作 ACCEPT、DROP、QUEUE
  --goto          -g   chain               jump to chain with no return
  --match         -m   match               extended match (may load extension)
  --numeric       -n	                     numeric output of addresses and ports                 
  --out-interface -o   output              name[+] network interface name ([+] for wildcard)     //输出网卡名称
  --table         -t   table               table to manipulate (default: `filter')               //指定table，默认filter
  --verbose       -v                       verbose mode
  --wait          -w  [seconds]            maximum wait to acquire xtables lock before give up
  --wait-interval -W  [usecs]	             wait time to try to acquire xtables lock default is 1 second
  --line-numbers                           print line numbers when listing                       //打印行号
  --exact         -x                       expand numbers (display exact values)
  --fragment      -f                       match second or further fragments only
  --version       -V                       print package version.
```

iptables 常用命令如下

1. 查看当前机器iptables所有规则: `$ iptables-save`
2. 查看指定table的所有规则: `$ iptables -t {table} -nL --line-numbers`
3. 删除所有的规则: `$ iptables -t {table} -F`
4. 保存当前机器iptables所有规则到文件: `$ iptables-save > filename`
5. 从指定文件恢复iptables规则: `$ iptables-restore < fileName`

## ① filter
filter table 规则主要的功能是`防火墙`，主要用于过滤IP数据包，是Linux防火墙主要的功能。

1. 删除所有规则链
   ```
   $ iptables -t filter -F   （内置规则链）
   $ iptables -t filter -X   （自定义规则链）
   ```

2. 删除所有规则
   ```
   $ iptables -t filter -P INPUT DROP
   $ iptables -t filter -P FORWARD DROP
   $ iptables -t filter -P OUTPUT DROP
   ```

3. 规则配置
   ```
   // case 1: 允许 loopback 设备
   $ iptables -t filter -A INPUT -i lo -j ACCEPT
   $ iptables -t filter -A OUTPUT -o lo -j ACCEPT
    
   // case 2: allow ssh over eth0
   $ iptables -t filter -A INPUT -i eth0 -p tcp --dport 22 -j ACCEPT
   $ iptables -t filter -A OUTPUT -o eth0 -p tcp --sport 22 -j ACCEPT

   // case 3: Allow icmp(ping) anywhere
   $ iptables -t filter -A INPUT -p icmp --icmp-type any -j ACCEPT
   $ iptables -t filter -A FORWARD -p icmp --icmp-type any -j ACCEPT
   $ iptables -t filter -A OUTPUT -p icmp --icmp-type any -j ACCEPT

   // case 4: allow http from internal(leftnet) to external(rightnet)
   $ iptables -t filter -A FORWARD -i eth1 -o eth2 -p tcp --dport 80 -j ACCEPT
   $ iptables -t filter -A FORWARD -i eth2 -o eth1 -p tcp --sport 80 -j ACCEPT

   // case 5: allow ssh from internal(leftnet) to external(rightnet)
   $ iptables -t filter -A FORWARD -i eth1 -o eth2 -p tcp --dport 22 -j ACCEPT
   $ iptables -t filter -A FORWARD -i eth2 -o eth1 -p tcp --sport 22 -j ACCEPT

   // case 6: allow http from external(rightnet) to internal(leftnet)
   $ iptables -t filter -A FORWARD -i eth2 -o eth1 -p tcp --dport 80 -j ACCEPT
   $ iptables -t filter -A FORWARD -i eth1 -o eth2 -p tcp --sport 80 -j ACCEPT

   // case 7: allow rpcinfo over eth0 from outside to system
   $ iptables -t filter -A INPUT -i eth2 -p tcp --dport 111 -j ACCEPT
   $ iptables -t filter -A OUTPUT -o eth2 -p tcp --sport 111 -j ACCEPT
   ```
   
## ② nat
nat table 规则主要的功能是网络地址转换，用于变更IP数据包中的源/目标IP地址，常见的 nat 有 SNAT (Source NAT) 和 DNAT（Destination NAT）。`SNAT` 常用于IP数据包离开主机之前改变源IP地址，`DNAT` 常用于IP数据包到达主机之前改变目的IP地址。

举个例子，我们在局域网内某个机器访问公网某个服务的时候，对外发请求的时候IP数据包从局域网到达网关的时候会执行 SNAT，把源IP地址改成网关的IP地址。当请求响应回来的时候，网关会执行 DNAT 把目的IP地址改成局域网内机器IP地址。（通常网关会在内存中维护这些映射关系）

1. 删除所有规则链
   ```
   $ iptables -t nat -F   （内置规则链）
   $ iptables -t nat -X   （自定义规则链）
   ```

2. 删除所有规则
   ```
   $ iptables -t nat -P INPUT DROP
   $ iptables -t nat -P FORWARD DROP
   $ iptables -t nat -P OUTPUT DROP
   ```

3. 规则配置
   ```
   // case 1: 配置SNAT coming from 10.1.1.0/24 network and exiting via eth1 will get the source ip-address set to 11.12.13.14
   $ iptables -t nat -A POSTROUTING -o eth1 -s 10.1.1.0/24 --to-source 11.12.13.14 -j SNAT 
   
   // case 2: 配置DNAT destination port 22 all packet destination ip-address set to 10.1.1.99
   $ iptables -t nat -A PREROUTING -i eth1 -p tcp --dport 22 --to-destination 10.1.1.99 -j DNAT
   ```

# 四. iptables使用
参考公有云 ECS 安全组的规则，我们来看下如何使用 iptables 实现主机的安全组。

IP数据包 `入方向` 规则如下

| 协议   |      端口      |  源地址 |   描述    |
|----------|:-------------|:------|:------|
| TCP |  22 | 0.0.0.0/0  （表示所有IP地址） |  SSH 远程登录  | 
| TCP |  23 | 0.0.0.0/0   |  Telnet 远程连接  | 
| TCP |  80 | 0.0.0.0/0   |  HTTP 服务端口  | 
| TCP |  443 | 0.0.0.0/0   |  HTTPS 服务端口  | 
| TCP |  3306 | 0.0.0.0/0   |  Mysql 端口  | 
| TCP |  6379 | 0.0.0.0/0   |  Redis 端口  | 

IP数据包 `出方向` 规则如下

| 协议   |      端口      |  目的地址 |
|----------|:-------------|------|
| all |  all | 0.0.0.0/0  （表示所有IP地址） |

# 五. iptables总结
综上所述，IP数据包的整体流向如下图所示

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/iptables-3.png?raw=true)


