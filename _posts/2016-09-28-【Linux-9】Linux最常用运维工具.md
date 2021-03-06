---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Linux
---

# 一. ldd
`ldd` 用来查看可执行文件运行所需的依赖库，常用来解决因缺少某个库文件而不能运行的一些问题。`ldd` 查看可执行文件的依赖库的工作原理，本质是通过`ld-linux.so` 来实现的。

```
$ ldd test
libstdc++.so.6 => /usr/lib64/libstdc++.so.6 (0x00000039a7e00000)
libm.so.6 => /lib64/libm.so.6 (0x0000003996400000)
libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00000039a5600000)
libc.so.6 => /lib64/libc.so.6 (0x0000003995800000)
/lib64/ld-linux-x86-64.so.2 (0x0000003995400000)
```

# 二. lsof
`lsof（list open files）`是一个查看当前系统文件的工具，在linux环境下任何事物都以文件的形式存在，通过文件不仅仅可以访问常规数据，还可以访问网络连接和硬件。

1. 查看所有打开的文件
   ```
   $ lsof
   COMMAND     PID      USER   FD      TYPE             DEVICE SIZE/OFF       NODE NAME
   init          1      root  cwd       DIR              253,0     4096          2 /
   init          1      root  rtd       DIR              253,0     4096          2 /
   init          1      root  txt       REG              253,0   150352    1310795 /sbin/init
   init          1      root  mem       REG              253,0    65928    5505054 /lib64/libnss_files-2.12.so
   ```
2. 查找打开某个文件相关的进程
   ```
   $ lsof /bin/sh
   COMMAND   PID USER  FD   TYPE DEVICE SIZE/OFF   NODE NAME
   bash     1264 root txt    REG    8,3   944184 392451 /bin/bash
   bash    24173  cgl txt    REG    8,3   944184 392451 /bin/bash
   bash    31744 zb10 txt    REG    8,3   944184 392451 /bin/bash
   ```
3. 列出某个用户打开的文件
   ```
   $ lsof -u cgl
   COMMAND   PID USER   FD   TYPE             DEVICE SIZE/OFF    NODE NAME
   sshd    24152  cgl  cwd    DIR                8,3     4096       2 /
   sshd    24152  cgl  rtd    DIR                8,3     4096       2 /
   sshd    24152  cgl  txt    REG                8,3   575192 1183078 /usr/sbin/sshd
   sshd    24152  cgl  DEL    REG                0,4          2344482 /dev/zero
   sshd    24152  cgl  mem    REG                8,3    18600 1444476 /lib64/security/pam_limits.so
   ...
   ```
4. 列出某个进程所打开的文件
   ```
   $lsof -c bash   //或使用 ($ lsof -p {pid})
   bash     1264 root    0u   CHR                5,1      0t0    5679 /dev/console
   bash     1264 root    1w  FIFO                0,8      0t0    9855 pipe
   bash     1264 root    2w  FIFO                0,8      0t0    9855 pipe
   bash    24173  cgl  cwd    DIR                8,3     4096 1314152 /home/cgl
   bash    24173  cgl  rtd    DIR                8,3     4096       2 /
   bash    24173  cgl  txt    REG                8,3   944184  392451 /bin/bash
   bash    24173  cgl  mem    REG                8,3   157072 1311011 /lib64/ld-2.12.so
   ...
   ```
5. 查看所有网络连接
   ```
   $ lsof -i  //($ lsof -i tcp 列出所有TCP连接)
   rsyslogd   1119   root    8u  IPv4   18499      0t0  UDP *:7139
   dnsmasq    1139 nobody    4u  IPv4    9392      0t0  UDP localhost:domain
   dnsmasq    1139 nobody    5u  IPv4    9393      0t0  TCP localhost:domain (LISTEN)
   sshd       1156   root    3u  IPv4    9439      0t0  TCP *:34815 (LISTEN)
   ...
   ```
6. 查看端口占用
   ```
   $ lsof -i:34815
   sshd     1156 root    3u  IPv4    9439      0t0  TCP *:34815 (LISTEN)
   sshd     1156 root    4u  IPv6    9441      0t0  TCP *:34815 (LISTEN)
   ```
7. 查看被显式删除但句柄还被进程持有的文件
   ```
   $ lsof | grep deleted
   sh        12967    cgl    1w      REG                8,3     1896 1315491 /home/cgl/log (deleted)
   sleep     13348    cgl    1w      REG                8,3     1896 1315491 /home/cgl/log (deleted)
   ```

# 三. pstack
`pstack` 命令用于跟踪进程栈，`pstack`命令必须由相应进程的属主或`root`用户 运行，可以使用`pstack`来确定进程挂起的位置。

```
$ pstack 25480
#0  0x000000331e8ac7be in waitpid () from /lib64/libc.so.6
#1  0x000000000043f162 in ?? ()
#2  0x00000000004403ff in wait_for ()
#3  0x0000000000430ec9 in execute_command_internal ()
#4  0x0000000000433563 in ?? ()
#5  0x000000000043036d in execute_command_internal ()
#6  0x00000000004310be in execute_command ()
#7  0x00000000004319c7 in ?? ()
#8  0x0000000000430515 in execute_command_internal ()
#9  0x00000000004310be in execute_command ()
#10 0x000000000041d906 in reader_loop ()
#11 0x000000000041d128 in main ()
```

# 四. strace
`strace` 常用来跟踪进程执行时的系统调用和所接收的信号，在Linux世界进程不能直接访问硬件设备，当进程需要访问硬件设备(比如读取磁盘文件，接收网络数据等等)时，必须由用户态模式切换至内核态模式，通过系统调用访问硬件设备。`strace`可以跟踪到一个进程产生的系统调用，包括参数、返回值、执行消耗的时间。

1. 跟踪可执行文件
   ```
   $ strace -f -T -tt -e trace=all ./kafkactl
   10:42:48.992077 execve("./kafkactl", ["./kafkactl"], [/* 26 vars */]) = 0 <0.000080>
   10:42:48.992325 arch_prctl(ARCH_SET_FS, 0x102a150) = 0 <0.000006>
   10:42:48.992392 sched_getaffinity(0, 8192, {3, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, ...}) = 512 <0.000006>
   10:42:48.992557 mmap(NULL, 262144, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f0ef31b1000 <0.000007>
   ...
   ```
2. 跟踪指定进程
   ```
   $ sudo strace -f -T -tt -e trace=all -p 1282 | head -10
   Process 1282 attached with 25 threads
   [pid  7990] 10:46:26.900631 futex(0x7f47e0002374, FUTEX_WAIT_PRIVATE, 187, NULL <unfinished ...>
   [pid  6278] 10:46:26.900705 restart_syscall(<... resuming interrupted call ...> <unfinished ...>
   [pid  4429] 10:46:26.900720 futex(0x7f47d0002e34, FUTEX_WAIT_PRIVATE, 189, NULL <unfinished ...>
   [pid  2766] 10:46:26.900730 futex(0x7f482037e844, FUTEX_WAIT_PRIVATE, 189, NULL <unfinished ...>
   [pid  1343] 10:46:26.900739 restart_syscall(<... resuming interrupted call ...> <unfinished ...>
   [pid  1342] 10:46:26.900747 restart_syscall(<... resuming interrupted call ...> <unfinished ...>
   [pid  1341] 10:46:26.900756 restart_syscall(<... resuming interrupted call ...> <unfinished ...>
   ...
   ```

# 五. top
`top` 是Linux下常用的性能分析工具，能够实时显示系统中各个进程的资源占用状况。

```
$ top
top - 11:10:44 up 22:20,  4 users,  load average: 0.44, 0.51, 0.49
Tasks: 114 total,   1 running, 113 sleeping,   0 stopped,   0 zombie
Cpu(s):  3.3%us,  0.8%sy,  0.0%ni, 95.8%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Mem:   3924416k total,   922532k used,  3001884k free,   160632k buffers
Swap: 16383996k total,        0k used, 16383996k free,   402864k cached

PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
1   root      20   0 19356 1520 1228 S  0.0  0.0   0:00.79 init 
2   root      20   0     0    0    0 S  0.0  0.0   0:00.00 kthreadd
3   root      RT   0     0    0    0 S  0.0  0.0   0:01.74 migration/0
4   root      20   0     0    0    0 S  0.0  0.0   0:00.37 ksoftirqd/0
5   root      RT   0     0    0    0 S  0.0  0.0   0:00.00 stopper/0
```

1. top最常用交互操作指令
   + `P`：按CPU使用率排行
   + `T`：按TIME+排行
   + `M`：按MEM使用排行
2. 查看完整的程序命令
   ```
   $ top -c
   PID   USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
   1282  root     20   0 2956m 126m  13m S  0.0  3.3   0:37.07 /usr/java/jdk1.8.0_51/bin/java -Dhome=/www/cgl/agent
   21165 cgl     20   0 58484  27m  13m S  0.0  0.7   0:00.34 kubectl exec -it api-test-pre-664bbd669-fc9qd /bin/bash -n cgl
   ...
   ```
3. 查看指定的进程信息: `$ top -p {pid}`

# 六. vmstat
`vmstat`(Virtual Meomory Statistics) 可实时动态监视操作系统的虚拟内存、进程、CPU活动。

```
vmstat 5 5
procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu-----
r  b   swpd   free   buff  cache    si   so    bi    bo   in   cs us sy id wa st 
0  0      0 3004920 161136 402916    0    0     2    11  148  111  3  1 96  0  0
0  0      0 3004212 161140 402916    0    0     0    29  485 1212  5  2 93  0  0
1  0      0 3005072 161144 402516    0    0     0    14  241  646  3  1 96  0  0
0  0      0 3004420 161144 402916    0    0     0     5  309  761  3  1 96  0  0
0  0      0 3004404 161148 402916    0    0     0    29  333  883  4  1 95  0  0
```

1. `Procs（进程）`
   + r: 运行队列中进程数量
   + b: 等待IO的进程数量
2. `Memory（内存）`
   + swpd: 使用虚拟内存大小
   + free: 可用内存大小
   + buff: 用作缓冲的内存大小
   + cache: 用作缓存的内存大小
3. `Swap`
   + si: 每秒从交换分区写到内存的大小
   + so: 每秒写入交换分区的内存大小
4. `IO：（现在的Linux版本块的大小为1024bytes）`
   + bi: 每秒读取的块数
   + bo: 每秒写入的块数
5. `system`
   + in: 每秒中断数，包括时钟中断
   + cs: 每秒上下文切换数
6. `CPU（以百分比表示）`
   + us: 用户进程执行时间(user time)
   + sy: 系统进程执行时间(system time)
   + id: 空闲时间(包括IO等待时间)
   + wa: 等待IO时间

# 七. iostat
`iostat`是I/O statistics（输入/输出统计）的缩写，用来动态监视系统的磁盘活动。

1. 显示所有设备负载情况
   ```
   $ iostat
   Linux 4.4.113-1.el7.elrepo.x86_64 (cgl.test.com)     07/31/2019 	_x86_64_	(32 CPU)
   avg-cpu:  %user   %nice %system %iowait  %steal   %idle
             4.41    0.00    3.23    1.40    0.00   90.96

   Device:   tps       kB_read/s    kB_wrtn/s    kB_read      kB_wrtn
   sda       88.65     777.25       1812.72      29728729225  69334319625
   sdb       3.64      3.95         128.93       151130648    4931488699
   ```
   
   cpu属性值说明:  
   1). `%user`：CPU处在用户模式下的时间百分比  
   2). `%nice`：CPU处在带NICE值的用户模式下的时间百分比  
   3). `%system`：CPU处在系统模式下的时间百分比  
   4). `%iowait`：CPU等待IO完成时间的百分比  
   5). `%steal`：管理程序维护另一个虚拟处理器时，虚拟CPU的无意识等待时间百分比  
   6). `%idle`：CPU空闲时间百分比
   
   `如果 %iowait 的值过高，表示硬盘存在 I/O 瓶颈；%idle 值高，表示CPU较空闲；如果 %idle 值高但系统响应慢时，有可能是CPU等待分配内存，此时应加大内存容量；%idle 值如果持续低于10，那么系统的CPU处理能力相对较低，表明系统中最需要解决的资源是CPU。`

   disk属性值说明:  
   1). `tps`: 设备每秒的I/O请求次数  
   2). `kB_read/s`: 每秒从设备读取的数据，单位KB  
   3). `kB_wrtn/s`: 每秒向设备写入的数据量，单位KB  
   4). `kB_read`: 读取的总数据量，单位KB  
   5). `kB_wrtn`: 写入的总数据量，单位KB  
    
2. 定时显示所有设备负载情况（每2秒刷新一次，共3次）: `$ iostat 2 3`
3. 查看设备使用率（%util）和响应时间（await）
   ```
   $ iostat -d -x -k   //或 iostat -d -x -k 2 3 (每2秒刷新一次，共3次)
   Linux 2.6.32-642.el6.x86_64 (cgl.test.com) 	07/31/2019 	_x86_64_	(2 CPU)

   Device:   rrqm/s   wrqm/s  r/s     w/s    rkB/s   wkB/s   avgrq-sz avgqu-sz  await  r_await  w_await  svctm  %util
   sda       0.07     3.60    0.22    1.86   4.29    21.12   24.46    0.02      8.43   11.05    8.12     2.27   0.47
   ```
   
   1). `rrqm/s`: 每秒进行 merge 的读操作数目  
   2). `wrqm/s`: 每秒进行 merge 的写操作数目  
   3). `r/s`: 每秒完成的读 I/O 设备次数  
   4). `w/s`: 每秒完成的写 I/O 设备次数  
   5). `rkB/s`: 每秒读K字节数  
   6). `wkB/s`: 每秒写K字节数  
   7). `avgrq-sz`: 平均每次设备I/O操作的数据大小  
   8). `avgqu-sz`: 平均I/O队列长度  
   9). `await`: 平均每次设备I/O操作的等待时间(毫秒)  
   10). `svctm`: 平均每次设备I/O操作的服务时间(毫秒)  
   11). `%util`: 一秒中有百分之多少的时间用于 I/O 操作
   
   `如果 %util 接近 100%，说明产生的I/O请求太多，I/O系统已经满负荷，该磁盘可能存在瓶颈。`

# 八. wget
`wget` 是Linux下的一个下载工具

1. 下载单个文件: `$ wget http://www.minjieren.com/wordpress-3.1-zh_CN.zip`
2. 下载并以不同的文件名保存: `$ wget http://www.minjieren.com/wordpress-3.1-zh_CN.zip -O wordpress.zip`
3. 限速下载: `$ wget --limit-rate=300k http://www.minjieren.com/wordpress-3.1-zh_CN.zip`
4. 断点续传: `$ wget -c http://www.minjieren.com/wordpress-3.1-zh_CN.zip`
5. 后台下载: `$ wget -b http://www.minjieren.com/wordpress-3.1-zh_CN.zip`
6. 设置代理下载: `$ wget --user-agent="Mozilla/5.0 (Windows; U; Windows NT 6.1; en-US) AppleWebKit/534.16 (KHTML, like Gecko) Chrome/10.0.648.204 Safari/534.16" http://www.minjieren.com/wordpress-3.1-zh_CN.zip`
7. 下载多个文件: `$ wget -i filelist.txt`  (filelist.txt为下载链接文件)

# 九. scp
1. 本地拷贝到远程服务器
   + 文件: `$ scp file.dat root@12.13.14.15:/root/chenguolin`
   + 目录: `$ scp -r dir root@12.13.14.15:/root/chenguolin`
2. 远程服务器拷贝到本地
   + 文件: `$ scp root@12.13.14.15:/root/chenguolin/file.dat .`
   + 目录: `$ scp -r root@12.13.14.15:/root/chenguolin/dir .`
3. scp指定私钥文件
    + 本地拷贝到远程: `$ scp -i ssh-key.pem file.dat root@12.13.14.15:/root/chenguolin`
    + 远程拷贝到本地: `$ scp -i ssh-key.pem root@12.13.14.15:/root/chenguolin/file.dat .`

# 十. systemd
systemd（System Management Daemon）是Linux 系统工具，用来启动守护进程，已成为大多数发行版的标准配置，systemctl 则是命令行工具。很多时候我们期望部署的服务可以用操作系统来托管，并且开机自启动，这个时候用 systemd 就非常合适。`推荐使用systemctl命令，如果没有可以使用service代替` [difference-between-systemctl-and-service-command](https://stackoverflow.com/questions/43537851/difference-between-systemctl-and-service-command)

1. 启动服务: `$ systemctl start [name.service]`
2. 停止服务: `$ systemctl stop [name.service]`
3. 重启服务: `$ systemctl restart [name.service]`
4. 重新加载服务: `$ systemctl reload [name.service]`
5. 查看服务状态: `$ systemctl status [name.service]`
6. 设置开机启动: `$ systemctl enable [name.service]` (确认 /lib/systemd/system/*.service /etc/systemd/system/*.service 是否有软链接)
7. 撤销开机启动: `$ systemctl disable [name.service]`
8. 查看开启启动服务列表: `$ systemctl list-unit-files | grep enabled`
9. systemd重新加载配置文件: `$ systemctl daemon-reload`
10. 查看服务是否处于运行状态: `$ systemctl is-active [name.service]`
11. 查看服务是否处于启动失败状态: `$ systemctl is-failed [name.service]`
12. 查看服务是否开机启动: `$ systemctl is-enabled [name.service]`
13. 查看服务启动参数: `$ systemctl show [name.service]`

# 十一. journalctl
journalctl 命令用来查看 systemd 所管理守护进程的日志，以及系统相关日志。

1. 查看所有日志: `$ journalctl -f` （-f 实时滚动显示最新日志）
2. 查看内核日志: `$ journalctl -k -f`
3. 查看指定时间的日志
   ```
   $ journalctl --since="2012-10-30 18:17:16" -f
   $ journalctl --since "20 min ago" -f
   $ journalctl --since yesterday -f
   $ journalctl --since "2015-01-10" --until "2015-01-11 03:00" -f
   $ journalctl --since 09:00 --until "1 hour ago" -f
   $ journalctl --since=today -f
   $ journalctl --since "2015-06-01 01:00:00" -f
   $ journalctl --since "2015-06-01" --until "2015-06-13 15:00" -f
   $ journalctl --since 09:00 --until "1 hour ago" -f
   ```
4. 查看指定服务日志: `$ journalctl -u docker.service -f`
5. 查看指定进程的日志: `$ journalctl _PID=1 -f`
6. 查看日志占用磁盘空间: `$ journalctl --disk-usage`


