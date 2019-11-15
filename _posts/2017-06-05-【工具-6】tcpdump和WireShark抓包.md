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
    -n  输出 IP 地址 而非域名
    -nn 显示端口号
    -t  在输出的每一行不打印时间戳
    -r  从指定的文件读取抓包内容 （-r data.pcap）
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
`tcpdump tcp -nn -i bond0 -tttt -s 0 -c 100 and dst port ! 22 and src net 10.10.1.0/24 -w 20170101.pcap`

1. tcp: 表示只抓取TCP协议的数据包
2. -nn: 表示输出IP地址和端口号 （通常在网络故障排查中，使用 IP 地址和端口号更便于分析问题）
3. -i bond0: 表示抓取经过 bond0 的数据包 (bond0是逻辑网卡)
4. -tttt: 表示时间戳格式为 2017-01-01 11:11:11.123456
5. -s 0: 表示抓取完整的数据包，默认抓取长度为 68 字节
6. -c 100: 表示抓取100个数据包
7. dst port ! 22: 表示抓取目标端口不是 22 的数据包
8. src net 10.10.1.0/24: 表示抓取源网络地址为 10.10.1.0/24 的数据包
9. -w 20170101.pcap: 表示抓取结果输出到 20170101.pcap 文件，方便使用 wireshark 进行分析

其它例子
1. 抓取包含 192.168.1.1 的数据包: tcpdump -i bond0 -nn host 192.168.1.1 
2. 抓取包含 192.168.1.0/24 网段的数据包: tcpdump -i bond0 -nn net 192.168.1.0/24  
3. 抓取包含端口 22 的数据包: tcpdump -i bond0 -nn port 22  
4. 抓取 udp 协议的数据包: tcpdump -i bond0 -nn udp  
5. 抓取 icmp 协议的数据包: tcpdump -i bond0 -nn icmp  
6. 抓取 arp 协议的数据包: tcpdump -i bond0 -nn arp  
7. 抓取 ip 协议的数据包: tcpdump -i bond0 -nn ip  
8. 抓取源 ip 是 192.168.1.1 数据包: tcpdump -i bond0 -nn src host 192.168.1.1
9. 抓取目的 ip 是 192.168.1.1 数据包: tcpdump -i bond0 -nn dst host 192.168.1.1 
10. 抓取源端口是 22 的数据包: tcpdump -i bond0 -nn src port 22  
11. 抓取源 ip 是 192.168.1.1 且目的 ip 是 22 的数据包: tcpdump -i bond0 -nn src host 192.168.1.1  and dst port 22  

注意 bond0 指的是网卡接口，默认情况下生产环境都会使用多网卡绑定bond，一般指定为 bond0 即可。

## ③ 分析
tcpdump 支持抓取多种协议的数据包，如 TCP、UDP、ICMP 等等，详情可以参考 [www.tcpdump.org](https://www.tcpdump.org/manpages/tcpdump.1.html#lbAE)。下面我们以 TCP 协议为例，分析一下 tcpdump 抓取到的数据包内容。

这是一段抓包的内容
```
2017-06-03 09:18:07.929532 IP 192.168.10.13.55446 > 192.168.102.35.41916: Flags [P.], seq 1461:2782, ack 78, win 916, options [nop,nop,TS val 117964079 ecr 816509256], length 1321
```
`2017-06-03 09:18:07.929532`: 系统时间戳  
`IP`: 网络层协议类型，表示IPv4，IP6表示为IPv6  
`192.168.10.13.55446`: 源IP地址和端口  
`192.168.102.35.41916`: 目标IP地址和端口  
`Flags [P.]`: TCP报文类型，表示传送数据，常见类型在下文介绍
`seq 1461:2782`: 数据流的序列号，源和目标IP发送的第一个数据包该序列号为绝对值，后续使用相对数值，seq 1461:2782 代表该数据包包含该数据流的第 1461 到 2782 字节
`ack 78`: 表示已经确认的数据流编号
`win 916`: 表示TCP接收窗口大小为 916 字节，即接收缓冲区中可用的字节数
`options ...`: 表示TCP选项
`length 1321`: 表示数据包字节长度

常见的TCP报文的标记类型如下
1. Flags [S] 表示SYN，发起连接
2. Flags [F] 表示FIN 关闭连接
3. Flags [P] 表示PUSH 传送数据
4. Flags [R] 表示RST 异常关闭连接
5. Flags [.] 表示ACK 确认数据
6. Flags [S.] 表示SYN+ACK
7. Flags [P.] 表示PUSH+ACK

下面我们来看一个具体的例子，我们在 192.168.10.13 通过命令 `curl "http://www.baidu.com"` 访问百度首页内容，同时使用tcpdump抓取TCP数据包，我们通过过滤得到抓包数据如下，为了方便分析我们对每一行进行了编号
```
1. 2019-11-13 09:46:41.573678 IP 192.168.10.13.32768 > 61.135.169.121.80: Flags [S], seq 4091238127, win 29200, options [mss 1460,sackOK,TS val 2415029947 ecr 0,nop,wscale 9], length 0
2. 2019-11-13 09:46:41.577160 IP 61.135.169.121.80 > 192.168.10.13.32768: Flags [S.], seq 2587209100, ack 4091238128, win 8192, options [mss 1452,sackOK,nop,nop,nop,nop,nop,nop,nop,nop,nop,nop,nop,wscale 5], length 0
3. 2019-11-13 09:46:41.577178 IP 192.168.10.13.32768 > 61.135.169.121.80: Flags [.], ack 1, win 58, length 0
4. 2019-11-13 09:46:41.577213 IP 192.168.10.13.32768 > 61.135.169.121.80: Flags [P.], seq 1:78, ack 1, win 58, length 77
5. 2019-11-13 09:46:41.580876 IP 61.135.169.121.80 > 192.168.10.13.32768: Flags [.], ack 78, win 916, length 0
6. 2019-11-13 09:46:41.583144 IP 61.135.169.121.80 > 192.168.10.13.32768: Flags [.], seq 1:1461, ack 78, win 916, length 1460
7. 2019-11-13 09:46:41.583152 IP 192.168.10.13.32768 > 61.135.169.121.80: Flags [.], ack 1461, win 63, length 0
8. 2019-11-13 09:46:41.583241 IP 61.135.169.121.80 > 192.168.10.13.32768: Flags [P.], seq 1461:2782, ack 78, win 916, length 1321
9. 2019-11-13 09:46:41.583251 IP 192.168.10.13.32768 > 61.135.169.121.80: Flags [.], ack 2782, win 69, length 0
10. 2019-11-13 09:46:41.583331 IP 192.168.10.13.32768 > 61.135.169.121.80: Flags [F.], seq 78, ack 2782, win 69, length 0
11. 2019-11-13 09:46:41.586804 IP 61.135.169.121.80 > 192.168.10.13.32768: Flags [.], ack 79, win 916, length 0
12. 2019-11-13 09:46:41.586977 IP 61.135.169.121.80 > 192.168.10.13.32768: Flags [F.], seq 2782, ack 79, win 916, length 0
13. 2019-11-13 09:46:41.586992 IP 192.168.10.13.32768 > 61.135.169.121.80: Flags [.], ack 2783, win 69, length 0
```

我们来仔细分析一下上面的TCP数据包

1. 第1行表示TCP建连第一次握手: 192.168.10.13:32768 发SYN给 61.135.169.121:80，数据序列号为4091238127，TCP接收窗口大小为29200，数据包长度
为0 （为什么是源IP地址是32768端口，因为curl用的是临时端口，可以通过 --local-port 61235指定端口号）
2. 第2行表示TCP建连第二次握手: 61.135.169.121.80 发SYN+ACK给 192.168.10.13.32768，数据序列号为2587209100，同时确认收到4091238127期望下一个数据流编号为4091238128，TCP接收窗口为8192，数据包长度为0
3. 第3行表示TCP建连第三次握手: 192.168.10.13.32768 发ACK给 61.135.169.121.80，TCP接收窗口为58，数据包长度为0
4. 第4行表示192.168.10.13.32768发送数据，发77个字节数据给61.135.169.121.80
5. 第5行表示61.135.169.121.80确认收到数据
6. 第6行表示61.135.169.121.80发送数据，发1460字节数据给192.168.10.13.32768
7. 第7行表示192.168.10.13.32768确认收到数据
8. 第8行表示61.135.169.121.80发送数据，发1321字节数据给192.168.10.13.32768
9. 第9行表示192.168.10.13.32768确认收到数据
10. 第10行表示TCP关闭第一次挥手: 192.168.10.13.32768 发送FIN+ACK给 61.135.169.121:80
11. 第11行表示TCP关闭第二次挥手: 61.135.169.121:80 发送ACK给 192.168.10.13.32768
12. 第12行表示TCP关闭第三次挥手: 61.135.169.121:80 发送FIN+ACK给 192.168.10.13.32768
13. 第13行表示TCP关闭第四次挥手: 192.168.10.13.32768 发送ACK给 61.135.169.121:80

# 三. wireshark
如果需要使用图形工具来抓包请参考 Wireshark。
Wireshark 还可以用来读取 tcpdump 保存的 pcap 文件。你可以使用 tcpdump 命令行在没有 GUI 界面的远程机器上抓包然后在 Wireshark 中分析数据包。


