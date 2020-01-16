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

所以，Linux内置的防火墙实际上是通过 iptables 控制 netfilter 数据包过滤表中的规则，从而达到对IP数据包过滤、更改、转发的目的。

# 二. iptables
[iptables](https://en.wikipedia.org/wiki/Iptables) 是一个用户空间的命令行工具，通过 iptables 创建新规则，其实就是往 netfilter 中插入一个hook，从而实现修改数据包、控制数据包流向等。iptables 适用于用于 IPv4 数据包，如果是 IPv6 则需要使用 ip6tables。iptables 主要由 规则表（table）、规则链（chain）以及规则（rule）三部分组成。

`table` 指的是规则表，用于存储具有相同功能的规则，不同的功能的规则放置在不同的 table 中。iptables 内置了5个 table，filter、nat、mangle、raw、security，最常用的是 filter 、nat 和 mangle 这3个 table，这3个 table 主要的作用如下。

1. filter: 负责IP数据包过滤（防火墙），是 iptables 命令默认查看的 table，内置了 `INPUT`、`FORWARD`、`OUTPUT` 3条规则链
2. nat: 负责网络地址转换即Network Address Translation，内置了 `ROUTING`、`OUTPUT`、`POSTROUTING` 3条规则链
3. mangle: 负责修改IP数据包，内置了`PREROUTING`、`POSTROUTING`、`OUTPUT`、`INPUT`、`FORWARD` 5条规则链

`chain` 指的是规则链，用于存储具有相同功能的规则，我们知道防火墙的作用就是对经过的IP数据包根据规则进行检测，然后执行相应的动作。可能有不止一条规则，因此我们把这些规则串到一条链上，每个经过的IP数据包都要经过该链所有规则进行检测一遍

1. INPUT: 发送到当前机器的IP数据包，都要经过INPUT规则链所有规则进行检测一遍
2. FORWARD: 所有通过当前机器转发的IP数据包，都要经过FORWARD规则链所有规则进行检测一遍
3. OUTPUT: 当前机器生成的IP数据包，都要经过OUTPUT规则链所有规则进行检测一遍

`rule` 指的

iptables 常用的命令如下，可以通过 `man iptables` 进行查看，因为 iptables 是一个命令行工具而非后台守护进程，因此我们不需要通过 systemctl 或者 service 等命令进行启动（不过，可能有些Linux发行版会需要），Linux已经内置了，通过 iptables 命令修改完规则后立即生效。

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

## ① filter
filter 表存储一序列的IP数据包过滤规则列表，内核模块 netfilter 会根据这些规则决定如何处理每个IP数据包。filter table 内置了 `INPUT`、`FORWARD`、`OUTPUT` 3条规则链，可以毫无问题地对包进行 接收（ACCEPT）、丢弃（DROP）、返回（RETURN）以及自定义的执行动作。`每个数据包通过网卡进入之后，`


我们先看一下当前机器的 filter table 的规则

```
$ iptables -t filter -L --line-numbers
Chain INPUT (policy ACCEPT)
num  target                  prot opt source       destination
1    KUBE-SERVICES           all  --  anywhere     anywhere             ctstate NEW /* kubernetes service portals */
2    KUBE-EXTERNAL-SERVICES  all  --  anywhere     anywhere             ctstate NEW /* kubernetes externally-visible service portals */
3    KUBE-FIREWALL           all  --  anywhere     anywhere

Chain FORWARD (policy ACCEPT)
num  target                    prot opt source               destination
1    KUBE-FORWARD              all  --  anywhere             anywhere             /* kubernetes forwarding rules */
2    KUBE-SERVICES             all  --  anywhere             anywhere             ctstate NEW /* kubernetes service portals */
3    DOCKER-USER               all  --  anywhere             anywhere
4    DOCKER-ISOLATION-STAGE-1  all  --  anywhere             anywhere
5    ACCEPT                    all  --  anywhere             anywhere             ctstate RELATED,ESTABLISHED
6    DOCKER                    all  --  anywhere             anywhere
7    ACCEPT                    all  --  anywhere             anywhere
8    DROP                      all  --  anywhere             anywhere
9    ACCEPT                    all  --  ecs-s6-large-2-linux-20200105130533/16  anywhere
10   ACCEPT                    all  --  anywhere             ecs-s6-large-2-linux-20200105130533/16

Chain OUTPUT (policy ACCEPT)
num  target         prot opt source       destination
1    KUBE-SERVICES  all  --  anywhere     anywhere             ctstate NEW /* kubernetes service portals */
2    KUBE-FIREWALL  all  --  anywhere     anywhere
```

每条规则除了 接收（ACCEPT）、丢弃（DROP）、返回（RETURN）这3个内置的执行动作，还可以配置用户自定义的执行动作。

1. ACCEPT: 接收IP数据包
2. DROP: 直接丢弃IP数据包
3. RETURN: 停止当前规则检测，检测当前链下一个规则

因此，filter表的规则的处理

使用举例
```
```

## ② nat


## ③ mangle

# 三. 使用
