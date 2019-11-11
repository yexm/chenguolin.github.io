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
    -A  用ASCII码打印每个数据包
    -B  设置操作系统缓冲区大小
    -c  抓包次数
    -C  输出到每个文件的大小
    -F  从指定的文件中读取表达式，忽略命令行中给出的表达式
    -G  
    -i  指定监听的网卡
    -t  在输出的每一行不打印时间戳
    -r  从指定的文件中读取包
    -s  设置每个数据包的大小，可以使得抓到的数据包不被截断，完整反映数据包的内容
    -T  将监听到的包直接解释为指定的类型的报文，常见的类型有rpc远程过程调用 和snmp
    -v  输出稍微详细的信息
    -vv 输出详细的报文信息
    -w  江抓包内容写入文件
    -X  列出16进制以及ASCII的数据包内容
    ```

2. expression指的是表达式
   

# 三. wireshark

