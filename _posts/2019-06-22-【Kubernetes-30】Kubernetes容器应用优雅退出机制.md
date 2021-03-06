---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 信号
当我们的设备断电的时候，我们需要考虑的一个点是我们的应用程序应该要如何处理这个事件。需要注意的是，当我们把电源拔掉的时候，电源其实并不是立即断电的。
我们的应用程序需要能够去处理这种事件，同时保存相应的应用状态。

对应的，我们会有以下几种可能的处理方式

1. 将内存中的所有内容保存到磁盘，然后在下次启动时将存储在磁盘上的内容恢复到内存中，但是这种方案有个问题就是从存储到磁盘和从磁盘恢复是一个缓慢的过程。
2. 使用一个文件来存储电源状态，其中0表示电源打开，1表示电源关闭，但是这种方法意味着系统中运行的进程应该跟踪存储在文件中的这个位。
3. 系统内核发送类似 `SIGTERM` 的信号给进程，进程自己去处理这个信号

对于应用容器来说，优雅退出重不重要完全取决于应用本身，这篇文章我们会讨论在Kubernetes和Docker中如何优雅退出容器。

`信号是一种软件中断的方式，用于一个进程与另一个进程通信的一种方式，例如操作系统和硬件`。所谓中断，是指当进程接收到一个信号时。它会停止做它正在做的事情，并通过对它做一些事情或忽略它来处理它。

Linux相关的信号如下所示，关于更多信号可以从 [sys/signal.h](https://unix.superglobalmegacorp.com/Net2/newsrc/sys/signal.h.html) 获取。

```
Signal     Value     Action   Comment
----------------------------------------------------------------------
SIGHUP        1       Term    Hangup detected on controlling terminal
                              or death of controlling process
SIGINT        2       Term    Interrupt from keyboard
SIGQUIT       3       Core    Quit from keyboard
SIGILL        4       Core    Illegal Instruction
SIGABRT       6       Core    Abort signal from abort(3)
SIGFPE        8       Core    Floating point exception
SIGKILL       9       Term    Kill signal
SIGSEGV      11       Core    Invalid memory reference
SIGPIPE      13       Term    Broken pipe: write to pipe with no
                              readers
SIGALRM      14       Term    Timer signal from alarm(2)
SIGTERM      15       Term    Termination signal
SIGUSR1   30,10,16    Term    User-defined signal 1
SIGUSR2   31,12,17    Term    User-defined signal 2
SIGCHLD   20,17,18    Ign     Child stopped or terminated
SIGCONT   19,18,25    Cont    Continue if stopped
SIGSTOP   17,19,23    Stop    Stop process
SIGTSTP   18,20,24    Stop    Stop typed at tty
SIGTTIN   21,21,26    Stop    tty input for background process
SIGTTOU   22,22,27    Stop    tty output for background process
```

举个例子

1. 当我们按下 `ctrl+c` 的时候相当于发送 `SIGINT` 信号
2. 当我们按下 `ctrl+z` 的时候相当于发送 `SIGTSTP` 信号
3. 当我们输入 `fg` 或 `bg` 命令的时候相当于发送 `SIGCONT` 信号

# 二. 容器优雅退出
对于Docker容器来说，当我们使用 `docker stop` 命令stop一个容器的时候，docker会先发送 `SIGTERM` 信号给容器内1号进程，如果在`10s内`进程没有及时结束，内核会继续发送`SIGKILL`信号直接结束进程。如果我们直接使用 `docker kill` 命令则会直接退出进程，容器内进程没有机会优雅退出。

对于Kubernetes的Pod来说，当我们使用 `kubectl delete pods mypod` 命令删除一个pod的时候，它会先发送 `SIGTERM` 信号给Pod内容器1号进程，如果在指定时间内进程没有结束，内核会继续发送 `SIGKILL` 信号直接结束进程，这个等待时间可以通过 Pod的配置文件定义 `terminationGracePeriodSeconds`，默认值为`30s`。如果应用进程没有处理 `SIGTERM` 信号，那么应用进程将会收到 `SIGKILL` 信号，容器进程将会立即从 etcd 删除，没有留时间给应用进程进行优雅退出。

正常情况下容器内1号进程为应用进程，但是很多时候我们期望在容器内启动更多进程，所以我们会使用shell脚本进程做为容器1号进程，通过shell脚本启动相应的应用进程和其它进程。这里面会有一个问题就是 `如果容器内1号进程是shell进程，我们的应用进程是shell进程的子进程`，那么应用进程是没有办法接收到 `SIGTERM` 信号的。这种情况下，应用进程将会直接接收到 `SIGKILL` 信号，应用进程就没有机会优雅退出，这个问题会对应用状态持久化会有影响。

# 三. dumb-init
我们通过一段 Golang 代码来验证以下3个case

```
package main

import (
     "fmt"
     "os"
     "os/signal"
     "runtime/debug"
     "syscall"
)

func main() {
     fmt.Println("Start ...")

     // block until shutdown channel closed
     shutdown := make(chan struct{})

     // wait signal
     c := make(chan os.Signal)
     signal.Notify(c, os.Interrupt, syscall.SIGHUP, syscall.SIGINT, syscall.SIGTERM)

     go func() {
         defer func() {
             if err := recover(); err != nil {
	         fmt.Println(err)
		 debug.PrintStack()
	     }
         }()

         for sig := range c {
	     fmt.Println(fmt.Sprintf("receive system signal: %s", sig.String()))
             // close shutdown channel
             close(shutdown)
	     break
         }
     }()

     // blocking unit receive signal
     <-shutdown
     
     fmt.Println("Shutdown ~")
}
```

代码编译: `GOOS=linux GOARCH=amd64 go build -o go-test *.go`

## ① case 1
case1: 验证业务应用进程为容器内1号进程，可以正常接收到 SIGTERM 信号，应用可以优雅退出

Dockerfile内容如下

```
FROM alpine:3.6

# WORKDIR
WORKDIR /www/cgl-test/
COPY go-test /www/cgl-test/
RUN chmod +x /www/cgl-test/go-test
COPY run.sh /www/cgl-test/
RUN chmod +x /www/cgl-test/run.sh

# ENTRYPOINT
ENTRYPOINT ["./go-test"]
```

1. 镜像构建: `$ docker build -t cgl-go-test:v1.0.0 .`
2. 运行容器: `$ docker run cgl-go-test:v1.0.0`
3. 查看容器内进程树，发现容器内1号进程为业务应用进程
   ```
   $ docker exec -it 8e9054162190 /bin/sh
   /www/cgl-test # ps axu
    PID   USER     TIME   COMMAND
      1   root     0:00   ./go-test
   ```
4. 使用Docker stop命令结束容器
   ```
   $ docker stop 8e9054162190
   Start ...
   receive system signal: terminated
   Shutdown ~
   ```
5. 由上可知，如果容器内1号进程是业务应用进程，那么应用进程是可以正常接收到 `SIGTERM` 信号的，应用进程可以优雅退出

## ② case 2
case2: 验证容器内1号进程为shell进程，业务应用进程为shell进程的子进程，业务应用进程没有办法正常接收到 `SIGTERM` 信号，应用无法优雅退出

Dockerfile内容如下

```
FROM alpine:3.6

# WORKDIR
WORKDIR /www/cgl-test/
COPY go-test /www/cgl-test/
RUN chmod +x /www/cgl-test/go-test
COPY run.sh /www/cgl-test/
RUN chmod +x /www/cgl-test/run.sh

# ENTRYPOINT
ENTRYPOINT ["./run.sh"]
```

run.sh脚本内容如下

```
#!/bin/sh

./go-test
```

1. 镜像构建: `$ docker build -t cgl-go-test:v2.0.0 .`
2. 运行容器: `$ docker run cgl-go-test:v2.0.0`
3. 查看容器内进程树，发现容器内1号进程为shell脚本进程，应用进程为shell脚本进程子进程
   ```
   $ docker exec -it da7d48ada838 /bin/sh
   /www/cgl-test # ps axu
   PID   USER     TIME   COMMAND
    1    root     0:00   {run.sh} /bin/sh ./run.sh
    6    root     0:00   ./go-test
   ```
4. 使用Docker stop命令结束容器
   ```
   $ docker stop da7d48ada838
   Start ...
   ```
5. 使用`Docker stop`停止容器进程的时候发现业务容器并没有正常打印出结束的日志，说明业务应用进程没有接收到 `SIGTERM` 信号。因此业务应用进程在`10s内`没有及时结束，内核会继续发送`SIGKILL`信号直接结束进程，导致业务应用进程没有优雅退出。

由上可知，当容器内`1`号进程为shell脚本进程，应用进程为shell进程的子进程的时候。shell进程无法向其子进程传递 `SIGTERM` 信号，导致容器发送 `SIGTERM ` 信号之后，父进程等待子进程退出。此时，如果父进程不能将信号传递到子进程，则整个容器就将无法正常退出，除非向父进程发送 `SIGKILL` 信号，使其强行退出。也就是我们使用 `docker stop` 命令结束容器的时候，等待超时时间之后，内核会发送 `SIGKILL` 信号强制结束进程。

所以如果容器内的进程树结构如下所示，都会有上面提到的问题
```
bash（PID 1）
   app（PID2）
```

bash 进程在接受到 `SIGTERM` 信号的时候，不会向应用进程传递这个信号，这会导致应用进程仍然不会退出。对于传统 Linux 系统（bash 进程 PID 不为 1），在 bash 进程退出之后，应用进程的父进程会被 `init` 进程（PID 为 1）接管，成为其父进程。但是在容器环境中，这样的行为会使应用进程失去父进程，因此 bash 进程不会退出。

## ③ case 3
case3: 使用`dumb-init`进程做为容器内`1`号进程，shell进程为`dumb-init`进程的子进程，通过shell进程内启动业务应用进程，相应的在shell脚本加入一些机制来保证应用可以优雅退出

为了解决这个问题，Yelp 提出了 [dumb-init](https://github.com/Yelp/dumb-init) ，默认情况下，dumb-init 会向子进程的进程组发送其收到的信号，dumb-init 也会接管失去父进程的进程，确保其能正常退出。

我们修改下Dockerfile的内容，加入dumb-init

```
FROM alpine:3.6

# WORKDIR
WORKDIR /www/cgl-test/

COPY go-test /www/cgl-test/
RUN chmod +x /www/cgl-test/go-test
COPY run.sh /www/cgl-test/
RUN chmod +x /www/cgl-test/run.sh

# Change 1: Download dumb-init
ADD https://github.com/Yelp/dumb-init/releases/download/v1.2.0/dumb-init_1.2.0_amd64 /usr/local/bin/dumb-init
RUN chmod +x /usr/local/bin/dumb-init

# ENTRYPOINT
ENTRYPOINT ["/usr/local/bin/dumb-init", "--"]
CMD ["./run.sh"]
```

run.sh脚本内容如下

```
#!/bin/sh

# 捕获Linux信号
trap 'stop' TERM HUP INT

# 优雅退出
stop() {
    pid=`ps axu | grep 'go-test' | awk '{print $1}'`
    kill -15 $pid
}

# 运行
./go-test
```

1. 镜像构建: `$ docker build -t cgl-go-test:v3.0.0 .`
2. 运行容器: `$ docker run cgl-go-test:v3.0.0`
3. 查看容器内进程树，发现容器内1号进程为dumb-init，dumb-init子进程为run.sh进程，业务应用进程为run.sh的子进程
   ```
   $ docker exec -it 5e10698aec5c /bin/sh
   /www/cgl-test # ps axu
   PID   USER     TIME   COMMAND
     1   root     0:00   /usr/local/bin/dumb-init -- ./run.sh
     7   root     0:00   {run.sh} /bin/sh ./run.sh
     8   root     0:00   ./go-test
   ```
4. 使用Docker stop命令结束容器
   ```
   $ docker stop 5e10698aec5c
   Start ...
   receive system signal: terminated
   Shutdown ~
   ```
5. 由上可知，使用dump-init做为容器内1号进程，同时用`trap`命令监听系统信号，可以保证业务应用进程能够接收到信号，保证应用进程可以优雅退出

一般来说遇到需要在容器内起两个或多个业务进程的场景，最好的做法是将这俩进程的启动放在一个shell脚本中，这个shell脚本扮演这俩进程的父进程，然后这个shell本身启动的时候还是直接挂在dumb-init进程下面，作为其唯一的子进程。这个shell脚本除了正常启动业务进程的逻辑之外，需要添加额外的处理逻辑

1. 实现一个stop函数，负责退出时的优雅逻辑
2. 在脚本中添加trap逻辑捕获退出信号并处理

