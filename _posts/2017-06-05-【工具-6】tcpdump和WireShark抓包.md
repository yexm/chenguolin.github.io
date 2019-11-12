---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - 工具
---

# 一. 背景介绍
之前我们介绍过 [Mac下用charles抓包](https://chenguolin.github.io/2017/06/03/%E5%B7%A5%E5%85%B7-3-Mac%E4%B8%8B%E7%94%A8Charles%E6%8A%93%E5%8C%85/)，我们一般使用Charles用于抓HTTP/HTTPS协议的包，但是很多时候抓HTTP包并没有办法真正看出问题所在，所以我们会需要上机器抓下TCP/UDP包，确认最终的问题出在哪里。

无论是开发还是运维我认为学会抓包技术都是必备的技能，所以这篇文章会介绍如何使用`tcpdump`和`wireshare`进行抓包和分析。`tcpdump`是Linux下的网络包分析命令行工具，`wireshark`则是可视化的网络部分析工具，通常我们是先使用`tcpdump`抓包并保存到文件，然后使用`wireshark`进行分析。

# 二. tcpdump
## ① 介绍
`tcpdump` 是一个很常用的网络包分析命令行工具，可以用来显示通过网络传输到本系统的 TCP/IP 以及其他网络的数据包。tcpdump 使用 libpcap 库来抓取网络包，这个库在几乎在所有的 Linux/Unix 中都有。英文定义为 `tcpdump - dump traffic on a network, a powerful command-line packet analyzer and libpcap, a portable C/C++ library for network traffic capture.`

tcpdump的命令格式 `tcpdump [options] [expression]`   [tcpdump](https://www.tcpdump.org/manpages/tcpdump.1.html#lbAE)

1. options指的是选项参数，具体如下
   ```
   [cgl@localhost ~]$ tcpdump -h
   tcpdump version 4.5.1
   libpcap version 1.5.3
   Usage: tcpdump [-aAbdDefhHIJKlLnNOpqRStuUvxX] [ -B size ] [ -c count ]
		[ -C file_size ] [ -E algo:secret ] [ -F file ] [ -G seconds ]
		[ -i interface ] [ -j tstamptype ] [ -M secret ]
		[ -P in|out|inout ]
		[ -r file ] [ -s snaplen ] [ -T type ] [ -V file ] [ -w file ]
		[ -W filecount ] [ -y datalinktype ] [ -z command ]
		[ -Z user ] [ expression ]
    -A  以 ASCII 值显示抓到的包 （比如和 MySQL 的交互时可以通过 -A 查看包的文本内容）
    -B  设置操作系统缓冲区大小
    -c  抓包次数
    -C  输出到每个文件的大小
    -D  列出所有可用的网卡
    -F  从指定的文件中读取表达式，忽略命令行中给出的表达式
    -G  设置输出文件的滚动切割的间隔
    -i  指定监听的网卡
    -j  时间戳格式
    -n  禁用域名解析 tcpdump 直接输出 IP 地址
    -t  在输出的每一行不打印时间戳
    -r  从指定的文件中读取包
    -s  设置每个数据包的大小，可以使得抓到的数据包不被截断，完整反映数据包的内容（0表示抓取完整数据）
    -T  将监听到的包直接解释为指定的类型的报文，常见的类型有rpc远程过程调用 和snmp
    -v  输出稍微详细的信息
    -vv 输出详细的报文信息
    -w  抓包内容输出到文件
    -X  以 16 进制格式输出数据包的内容，不加该参数会只输出 iptcp/udp 头部信息
    -z  文件滚动切割之后的后置命令
    ```

2. expr=ession指的是表达式，用于配置要抓取哪些数据包，例如
    ```
    1) 指定host: host 10.215.20.13
    2) 指定网络地址: net 10.215.20.0
    3) 指定端口: port 8711
    4) 指定源地址: src 10.215.20.13
    5) 指定目标地址: dst 10.215.20.0
    6) 同时指定源地址和目标地址: src 10.215.20.13 and dst 10.215.20.0
    
    tcpdump默认会抓取所有协议的数据包，支持以下协议: ip、arp、tcp、udp、icmp
    tcpdump支持三种逻辑运算
    1) 非: ! 或 not
    2) 与: && 或 and
    3) 或: || 或 or
    ```

## ② 举例
`tcpdump tcp -i bond0 -tttt -s 0 -c 100 and dst port ! 22 and src net 10.10.1.0/24 -w 20190131.tcpdump`

1. tcp: 表示只抓取TCP协议的数据包
2. -i bond0: 表示抓取经过 bond0 的数据包 (bond0是逻辑网卡)
3. -tttt: 表示时间戳格式为 2017-01-01 11:11:11.123456
4. -s 0: 表示抓取完整的数据包，默认抓取长度为 68 字节
5. -c 100: 表示抓取100个数据包
6. dst port ! 22: 表示抓取目标端口不是 22 的数据包
7. src net 10.10.1.0/24: 表示抓取源网络地址为 10.10.1.0/24 的数据包
8. -w 20190131.tcpdump: 表示保存成 tcpdump 文件中, 方便使用 wireshark 分析抓包结果。

# 三. wireshark

