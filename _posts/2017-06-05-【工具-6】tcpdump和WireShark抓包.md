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
`tcpdump` 是一个很常用的网络包分析命令行工具，可以用来显示通过网络传输到本系统的 TCP/IP 以及其他网络的数据包。tcpdump 使用 libpcap 库来抓取网络包，这个库在几乎在所有的 Linux/Unix 中都有。

`tcpdump, a powerful command-line packet analyzer; and libpcap, a portable C/C++ library for network traffic capture.`

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
        
-A	  以ASCII格式打印出所有报文
-B    缓冲buffer的大小
-c	  指定数量的报文后
-C	  在将一个原始分组写入文件之前，检查文件当前的大小是否超过了参数file_size 中指定的大小。如果超过了指定大小，则关闭当前文件，然后在打开一个新的文件。参数 file_size 的单位是兆字节（是1,000,000字节，而不是1,048,576字节）。
-d	将匹配信息包的代码以人们能够理解的汇编格式给出。
-dd	将匹配信息包的代码以c语言程序段的格式给出。
-ddd	将匹配信息包的代码以十进制的形式给出。
-D	打印出系统中所有可以用tcpdump截包的网络接口。
-e	在输出行打印出数据链路层的头部信息,也就是使用数据链路层的MAC数据包数据来显示.
-f	将外部的Internet地址以数字的形式打印出来。
-F	从指定的文件中读取表达式，忽略命令行中给出的表达式。
-i	指定监听的网络接口。
-l	使标准输出变为缓冲行形式，可以把数据导出到文件。
-L	列出网络接口的已知数据链路。
-b	在数据-链路层上选择协议，包括ip、arp、rarp、ipx都是这一层的。
-n	不把网络地址转换成名字。
-nn	不进行端口名称的转换。
-N	不输出主机名中的域名部分。例如，‘nic.ddn.mil‘只输出’nic‘。
-t	在输出的每一行不打印时间戳。
-O	不运行分组分组匹配（packet-matching）代码优化程序。
-P	不将网络接口设置成混杂模式。
-q	快速输出。只输出较少的协议信息。
-r	从指定的文件中读取包(这些包一般通过-w选项产生)。
-S	将tcp的序列号以绝对值形式输出，而不是相对值。
-s	从每个分组中读取最开始的snaplen个字节，而不是默认的68个字节。
-T	将监听到的包直接解释为指定的类型的报文，常见的类型有rpc远程过程调用）和snmp（简单网络管理协议)。
-t	不在每一行中输出时间戳。
-tt	在每一行中输出非格式化的时间戳
-ttt	输出本行和前面一行之间的时间差。
-tttt	在每一行中输出由date处理的默认格式的时间戳。
-v	输出一个稍微详细的信息，例如在ip包中可以包括ttl和服务类型的信息。
-vv	输出详细的报文信息。
-w	直接将分组写入文件中，而不是不分析并打印出来。
-X	可以列出16进制以及ASCII的数据包内容,对于监听数据包内容很有用.
```

# 三. wireshark

