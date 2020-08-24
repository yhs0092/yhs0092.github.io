---
title: Docker 容器终止运行方式小记
tags: [docker,ServiceComb-Java-Chassis]
keywords: [docker,ServiceComb-Java-Chassis,CSE]
date: 2020-08-21 16:05:38
categories:
- 软件技术
index_img: /img/blog/docker-termination-description/banner.jpg
banner_img: /img/blog/docker-termination-description/banner.jpg
---
简要介绍 `docker stop/kill` ，以及如何更优雅地触发容器退出流程
<!-- more -->

## docker 容器根进程与容器退出机制

docker 作为一种轻量级的“虚拟化”技术，通过 Linux 提供的 cgroup 和 namespace 等机制限制容器内进程的资源占用及其对宿主机资源的可见性。当容器启动时，宿主机 Linux 内核会为此容器创建一个 PID namespace 。容器的启动进程在宿主机上看来是一个普通的进程，而在该容器的 PID namespace 中看上去 `PID` = 1 ，即它是此容器内的根进程。当容器内 PID=1 的进程停止运行时，容器便停止运行了。

作为例子，我们启动一个 alpine linux 的容器，使其执行 `ping localhost` 命令：
```shell
## --rm 表示容器结束时自动删除，免去手动清理无用容器的工作
docker run --rm --entrypoint ping alpine localhost
```

运行 `docker ps` 找到这个 alpine 容器，我们可以使用 `docker inspect` 命令查看到此容器的根进程在宿主机上的进程号：
```shell
# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                    NAMES
52100c256521        alpine              "ping localhost"    9 seconds ago       Up 8 seconds                                 zen_mccarthy

# docker inspect 52100 -f '{{.State.Pid}}'
9509
```
> `docker inspect <container_id>` 会以 json 格式打印容器的信息。`-f`参数后面接的格式化串声明 `docker inspect` 只将容器根进程的进程号打印出来。

然后在宿主机执行`ps <Pid>`，果然是能看到一个 `ping localhost` 进程的：
```shell
# ps 9509
  PID TTY      STAT   TIME COMMAND
 9509 ?        Ss     0:00 ping localhost
```

使用 `kill` 命令向其发送一个 `SIGINT` 信号令其终止运行（`SIGINT`信号代表键盘发送的终止信号 ctrl-C ， `ping` 命令接收到此信号后会正常退出并打印统计信息） `kill -s SIGINT 9509` ，最终在运行容器的控制台，我们看到的输出类似下面这样
```shell
# docker run --rm --entrypoint ping alpine localhost
PING localhost (127.0.0.1): 56 data bytes
64 bytes from 127.0.0.1: seq=0 ttl=64 time=0.032 ms
…… 忽略中间日志 ……
64 bytes from 127.0.0.1: seq=26 ttl=64 time=0.046 ms

--- localhost ping statistics ---
27 packets transmitted, 27 packets received, 0% packet loss
round-trip min/avg/max = 0.032/0.046/0.065 ms
```
说明容器中的 `ping` 进程确实是接收到 `SIGINT` 信号正常终止了。并且此时再次运行 `docker ps` 是看不到这个容器的，说明**容器根进程结束运行的话，容器也会终止运行**。

> 以上现象也能解释为什么容器的启动命令不能用 `nohup` 等形式将应用服务进程作为后台运行 —— 如果应用服务的进程不是作为主进程运行在前台，那么容器就会因为根进程执行完成而结束运行。

## docker stop 和 docker kill

`docker stop` 命令和 `docker kill` 命令都可以用于终止正在运行的 docker 容器。差别在于， `docker stop` 默认会先发送一个 `SIGTERM` 信号给容器内的根进程，如果一段时间内容器没有结束运行（默认10秒），则再向容器发送一个 `SIGKILL` 将容器根进程强制结束。而 `docker kill` 是默认直接给容器内的根进程发送一个 `SIGKILL` 信号，类似于 `kill -9` 强杀进程。

两者的设计目的并不相同， `docker stop` 主要是为了优雅停止容器的运行。它的选项 `--time , -t` 表示在强制杀进程之前等待多少秒。
而 `docker kill` 跟 Linux 的 `kill` 命令一样，可以用来给容器内的根进程发送进程信号。具体的信号内容通过参数 `--signal , -s` 来指定，参数值可以是*信号名*或者是它的*数字编号*。

需要注意的是，**容器只把进程信号发送给它的根进程**。这也比较容易理解，Linux宿主机上也是由 PID=1 的根进程负责处理宿主机上的进程结束工作。容器的 PID namespace 与此类似， docker 也只会把外来的进程信号传递给 PID=1 的进程，这个进程再来决定信号该如何处理。

> PID1进程对于操作系统而言具有特殊意义。操作系统的PID1进程是init进程，以守护进程方式运行，是所有其他进程的祖先，具有完整的进程生命周期管理能力。在Docker容器中，PID1进程是启动进程，它也会负责容器内部进程管理的工作。而这也将导致进程管理在Docker容器内部和完整操作系统上的不同。
> —— [理解Docker容器的进程管理][]

## 正确编写启动脚本确保容器优雅退出

让我们首先来看这样一个启动脚本：
```shell
#!/bin/sh

ping localhost
```
Dockerfile 如下：
```shell
FROM openjdk:8-alpine

WORKDIR /home

COPY start.sh .

ENTRYPOINT ["sh", "./start.sh"]
```
打包为 test 镜像，执行 `docker run --rm test` 启动 test 镜像。此时如果使用 `docker kill -s SIGINT <container_id>` 以向容器内发送 `ctrl+C` ，能否触发 `ping` 进程打印统计信息并终止运行呢？

实际实验的结果是，不行。
查看容器内的进程情况，我们可以看到 PID=1 的进程是 `sh ./start.sh`，而 `ping localhost` 进程是它的子进程：
```shell
# docker exec ed882 ps -o pid,ppid,user,args
PID   PPID  USER     COMMAND
    1     0 root     sh ./start.sh
    8     1 root     ping localhost
    9     0 root     ps -o pid,ppid,user,args
```

很明显， `ping localhost` 进程不是容器的根进程，我们发送的 `SIGINT` 信号只会被 `sh ./start.sh` 进程接收，但它又不会将信号转发给 `ping localhost` 进程，于是 `SIGINT` 信号就这么被忽略了，好像什么都没发生过。

解决这个问题的方式通常是确保容器内的业务进程是容器的根进程（docker部署方式推荐一个容器一个业务进程，所以尽量避免将多个业务部署在同一个容器中），为此，需要修改启动脚本：
```shell
#!/bin/sh

exec ping localhost # 注意这一行的变化
```
`exec` 命令用于运行指定的命令，并以此命令的运行进程替换掉当前进程，继承当前进程的 PID。直观地说，运行上面的启动脚本时，当执行到 `exec ping localhost` ， `ping localhost` 进程将会取代 `sh ./start.sh` 进程在进程树中的位置，并沿用 `sh ./start.sh` 进程的 PID，也就是变成容器内的根进程了。

重新打个 test 镜像包试一下：
```shell
# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                    NAMES
71cde237ce32        test                "sh ./start.sh"     10 seconds ago      Up 9 seconds                                 confident_mcclintock

# docker exec 71cde cat /home/start.sh
#!/bin/sh

exec ping localhost

# docker exec 71cde ps -o pid,ppid,user,args
PID   PPID  USER     COMMAND
    1     0 root     ping localhost
   15     0 root     ps -o pid,ppid,user,args
```
现在 `ping localhost` 是容器的根进程了。运行一下 `docker kill -s 2 71cde`，容器立即结束运行，并且打出了数据包统计信息，说明它确实是接收到 `SIGINT` 信号退出的。控制台输出如下：
```shell
# docker run --rm test
PING localhost (127.0.0.1): 56 data bytes
64 bytes from 127.0.0.1: seq=0 ttl=64 time=0.029 ms
64 bytes from 127.0.0.1: seq=1 ttl=64 time=0.044 ms
64 bytes from 127.0.0.1: seq=2 ttl=64 time=0.047 ms
…… 忽略中间输出 ……
64 bytes from 127.0.0.1: seq=85 ttl=64 time=0.046 ms

--- localhost ping statistics ---
86 packets transmitted, 86 packets received, 0% packet loss
round-trip min/avg/max = 0.029/0.047/0.061 ms
```

> 上文演示的是 `docker kill` 发送信号，但 `docker stop` 优雅退出容器的逻辑也是一样的。

这一点在实际的业务系统运行中也是有重要意义的。通常业务进程在结束运行前有些收尾的事情要做，比如需要确保写文件都结束，网络服务器还要保证自己处理中的业务请求都处理完成，在此过程中还不能接收新的请求。这些都要求容器内的业务进程能感知到容器将要退出运行才行。
所以我们在打包镜像的时候需要注意，**要么在 Dockerfile 中直接启动业务进程，如果要以启动脚本拉起业务进程，需要在脚本内以 `exec` 命令启动业务。**

举个实际的例子， Java-Chassis 框架基于 JVM shutdown hook 实现了微服务进程的[优雅停机][]功能。该功能使得 Java-Chassis 可以在微服务进程退出时执行等待处理中的业务请求执行完成、返回 503 状态码拒绝新请求、注销微服务实例等一系列操作，这些操作可以尽量确保业务请求不受损，使 consumer 端能尽快感知到 provider 微服务实例的下线操作。用户也可以通过 `BootListener` 接口监听 `BEFORE_CLOSE` 事件来完成自定义的一些清理操作。

**而要确保 Java-Chassis 的优雅停机功能在容器化部署场景下正常工作，就需要用户基于上述内容调整业务进程启动方式，使得业务进程是容器内的根进程。**

当然 Java-Chassis 的优雅停机操作仍然只是尽力保护业务请求不受损（配合 Java-Chassis 的[实例隔离][]和[重试策略][]）。 Java-Chassis 默认的重试策略是 `tryOnSame=0,tryOnNext=1` ，因此只能保证 provider 端实例一个接一个下线时业务调用不受损，而且要求实例下线的时间间隔要大于 consumer 端的 instance pull 时间间隔。否则 consumer 端本地缓存的 provider 实例列表中实际已下线的实例数大于1，就有可能造成请求路由给一个已下线的实例，又重试到另一个已下线的实例的情况，最终导致业务请求失败。

要从理论上保证实例缩容、滚动升级等涉及 provider 端实例下线的场景中微服务业务调用不受损，最好是在部署系统下线微服务实例之前将对应的实例记录状态置为不可用，让 consumer 端微服务从服务中心感知到实例状态变化后再真正下线对应的实例。但 ServiceComb 本身并没有对用户的部署方式做假设，让部署系统和服务中心联动起来也并不是一件简单的事情。 为了能以尽可能低的成本达到类似的效果， Java-Chassis 框架在新版本引入了退出前将本实例置为 `DOWN` 状态并阻塞等待的机制。
此机制默认不开启，配置 `servicecomb.boot.turnDown.waitInSeconds` 设置阻塞等待时间大于零即可启用。当 JVM 触发 shutdown hook 运行时，该功能会向服务中心发请求将本实例的状态置为 `DOWN`，并根据用户配置的阻塞时长进行等待。在此期间， shutdown hook 的运行线程是被阻塞住的，因此 JVM 还不会退出，微服务业务还能继续处理请求。阻塞完成后 JVM shutdown hook 逻辑放通继续执行，微服务真正退出。在阻塞的这一段时间里， consumer 端微服务就能够从服务中心查询到即将下线的 provider 实例状态变为不可用了，于是业务流量便提前绕过此实例，从根源上避免了业务调用失败。
使用此功能还需要部署系统给予业务服务足够的退出时间，这可能需要用户根据 consumer 端的 instance pull 时间间隔进行设置，以确保 docker / K8s 等部署运行系统不会提前强制结束进程。

> 该功能的代码在 `SCBEngine#blockShutDownOperationForConsumerRefresh` 方法中。

## 补充说明

在 docker 文档 [docker kill][] 中有说明如果 `ENTRYPOINT` 和 `CMD` 是以 shell 风格书写的，会导致启动命令以 `/bin/sh -c` 方式运行，是 `sh` 进程的子进程，无法获得信号。但笔者在实测时发现即使以 shell 风格书写 Dockerfile 中的启动命令， `docker history` 查看镜像分层看到的也确实是 `ENTRYPOINT ["/bin/sh" "-c" "sh ./start.sh"]` ，启动容器后仍然能看到启动脚本中拉起的业务进程的 PID=1 （前提条件是启动脚本以 `exec` 命令运行业务）。推测是内核或者 docker 版本不同导致与文档所描述的行为不同。
即使如此，书写 Dockerfile 时仍然建议遵循 exec 风格。（见[上一篇博客][Dockerfile 中的 ENTRYPOINT 和 CMD]的描述）

## 扩展阅读

- [docker kill 官方文档][docker kill]
- [docker stop 官方文档][docker stop]
- [理解Docker容器的进程管理][]
- [优雅停机][]

## 本文验证环境

同[Dockerfile 中的 ENTRYPOINT 和 CMD][]。

[docker kill]: https://docs.docker.com/engine/reference/commandline/kill/ "docker kill | Docker Documentation"
[docker stop]: https://docs.docker.com/engine/reference/commandline/stop/ "docker stop | Docker Documentation"
[理解Docker容器的进程管理]: https://www.cnblogs.com/ilinuxer/p/6188303.html "理解Docker容器的进程管理"
[Dockerfile 中的 ENTRYPOINT 和 CMD]: /2020/08/20/docker-entrypoint-and-cmd/ "Dockerfile 中的 ENTRYPOINT 和 CMD"
[优雅停机]: https://docs.servicecomb.io/java-chassis/zh_CN/general-development/shutdown/ "Java-Chassis 优雅停机"
[实例隔离]: https://docs.servicecomb.io/java-chassis/zh_CN/references-handlers/loadbalance/#_5 "Java-Chassis 实例隔离"
[重试策略]: https://docs.servicecomb.io/java-chassis/zh_CN/references-handlers/loadbalance/#_7 "Java-Chassis 重试策略"
