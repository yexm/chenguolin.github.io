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
[iptables](https://en.wikipedia.org/wiki/Iptables) 是一个用户空间的命令行工具，通过 iptables 创建新规则，其实就是往 netfilter 中插入一个hook，从而实现修改数据包、控制数据包流向等。iptables 适用于用于 IPv4 数据包，如果是 IPv6 则需要使用 ip6tables。iptables 主要由 规则表（table）、规则链（chain）以及规则（rule）三部分组成，下面我们重点介绍一下。

因为 iptables 是一个命令行工具而非后台守护进程，因此我们不需要通过 systemctl 或者 service 等命令进行启动（不过，可能有些Linux发行版会需要），Linux已经内置了，通过 iptables 命令修改完规则后立即生效。

table 指的是规则表，用于存储具有相同功能的规则，不同的功能的规则放置在不同的 table 中。iptables 内置了5个 table，filter、nat、mangle、raw、security，最常用的是 filter 、nat 和 mangle 这3个 table，这3个 table 主要的作用如下。

1. filter: 负责IP数据包过滤（防火墙），是 iptables 命令默认查看的 table，内置了 `INPUT`、`FORWARD`、`OUTPUT` 3条规则链
2. nat: 负责网络地址转换即Network Address Translation，内置了 `ROUTING`、`OUTPUT`、`POSTROUTING` 3条规则链
3. mangle: 负责修改IP数据包，内置了`PREROUTING`、`POSTROUTING`、`OUTPUT`、`INPUT`、`FORWARD` 5条规则链


## ① filter


## ② nat


## ③ mangle

# 三. 使用
