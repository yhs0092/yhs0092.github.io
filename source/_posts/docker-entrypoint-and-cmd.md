---
title: Dockerfile 中的 ENTRYPOINT 和 CMD
tags: [docker]
date: 2020-08-20 20:53:17
categories:
- 软件技术
index_img: /img/blog/docker-entrypoint-and-cmd/banner.jpg
banner_img: /img/blog/docker-entrypoint-and-cmd/banner.jpg
---

关于 Dockerfile 中 ENTRYPOINT 和 CMD 的简要对比及使用说明
<!-- more -->

> 在 Dockerfile 中， `ENTRYPOINT` 命令和 `CMD` 命令都可以指定 docker 容器启动时运行的命令，它们之间有什么异同，使用上又有什么需要注意的呢？

## 简要对比

### `ENTRYPOINT` 命令和 `CMD` 命令都可用于指定 docker 容器启动时默认运行的命令，并且它们都是可以覆盖的

`ENTRYPOINT` 命令可以使用 `--entrypoint` 参数覆盖。
如果可执行命令还需要加上参数，记得参数是加在末尾的。
例如，执行 `docker run --entrypoint sleep test 3600` 命令，表示启动一个 `test:latest` 镜像的容器，执行的命令效果是 `sleep 3600`（注意这里作为命令的 `sleep` 和作为参数的 `3600` 的位置不是连在一起的！）。

`CMD` 命令不需要使用特别的参数覆盖，直接在 `docker run` 命令的末尾添加参数即可。
例如，执行 `docker run test sleep 3600` 命令，也表示启动一个 `test:latest` 镜像的容器，执行的命令是 `sleep 3600`。
（前提条件是镜像没有指定`ENTRYPOINT`，后文会展示 `ENTRYPOINT` 和 `CMD` 是可以共同生效的）

### 如果你希望打出的镜像只执行某个具体的程序，那么 `ENTRYPOINT` 更适合用于指定启动命令

如前文所述， `ENTRYPOINT` 指定的启动命令覆盖起来更麻烦一些。
该命令通常都是用于指定一个预设的启动命令的，用户一般不会修改此启动命令，但他们可能增加一些参数。
例如使用如下的 Dockerfile 打一个镜像：
```Dockerfile
FROM alpine

ENTRYPOINT ["ping"]
```
执行打包`docker build -t test -f Dockerfile .`。
则在运行 `test` 镜像时可以通过在 `docker run` 命令末尾追加参数来调整运行的`ping`命令。
例如可以执行 `docker run test localhost` 来运行 `ping localhost`，也可以执行 `docker run test www.baidu.com` 来运行 `ping www.baidu.com`。

### 如果你希望打出的镜像更灵活，那么使用 `CMD` 更合适

如前文所述，仅仅是在 `docker run` 命令的末尾添加参数即可覆盖 `CMD` 命令。
因此用户可以更加方便灵活地指定其他启动命令来运行镜像。

## 命令书写风格： shell 和 exec

在 Dockerfile 中，把启动命令写成 `ENTRYPOINT ifconfig` 和 `ENTRYPOINT ["ifconfig"]` 都是可以的，前者称为 shell 风格，后者称为 exec 风格，而且在运行 docker 镜像时看上去效果也一样。
但这两种写法打出的镜像中实际运行的启动命令并不相同。

使用 shell 风格的启动命令打出的镜像，会将 `ENTRYPOINT` 后面接的命令整体以 `/bin/sh -c` 的形式执行。
例如 Dockerfile 中写的是 `ENTRYPOINT ifconfig eth0` ，则打出的镜像中启动命令为 `/bin/sh -c 'ifconfig eth0'`。

使用 exec 风格的启动命令打出的镜像，会将其中的命令分段拼接起来执行。
例如 Dockerfile 中写的是 `ENTRYPOINT ["ifconfig", "eth0"]` ，则打出的镜像中启动命令为 `ifconfig eth0`。

直接执行这两种风格的 Dockerfile 打出的镜像，从效果上看似乎没有差别。
但是 shell 风格的写法会让 `docker run` 镜像时不好添加参数，而 exec 风格的写法则没有此问题。
因此，一般更推荐使用 exec 风格的写法书写 Dockerfile。

作为示例，我们分别以两种风格各书写一份 Dockerfile，内容如下：
- DockerfileShell:
  ```Dockerfile
  FROM alpine

  ENTRYPOINT ifconfig
  ```
- DockerfileExec:
  ```Dockerfile
  FROM alpine

  ENTRYPOINT ["ifconfig"]
  ```

分别执行 `docker build -t test-shell -f DockerfileShell .` 和 `docker build -t test-exec -f DockerfileExec .` 打出镜像。
可以先使用 `docker history` 观察镜像的差别：
```shell
# docker history --no-trunc test-shell
IMAGE                                                                     CREATED              CREATED BY                                                                                          SIZE                COMMENT
sha256:3ff667ccad9ec9e5cdd1450bcfe9cd5463fc91c2ddd7f71221b6e65d20aa1eea   About a minute ago   /bin/sh -c #(nop)  ENTRYPOINT ["/bin/sh" "-c" "ifconfig"]                                           0B
sha256:a24bb4013296f61e89ba57005a7b3e52274d8edd3ae2077d04395f806b63d83e   2 months ago         /bin/sh -c #(nop)  CMD ["/bin/sh"]                                                                  0B
<missing>                                                                 2 months ago         /bin/sh -c #(nop) ADD file:c92c248239f8c7b9b3c067650954815f391b7bcb09023f984972c082ace2a8d0 in /    5.57MB

# docker history --no-trunc test-exec
IMAGE                                                                     CREATED              CREATED BY                                                                                          SIZE                COMMENT
sha256:e885515099ca3fa7cd55a40597f960f0660ea040dfe304888e12fedd3bec8748   About a minute ago   /bin/sh -c #(nop)  ENTRYPOINT ["ifconfig"]                                                          0B
sha256:a24bb4013296f61e89ba57005a7b3e52274d8edd3ae2077d04395f806b63d83e   2 months ago         /bin/sh -c #(nop)  CMD ["/bin/sh"]                                                                  0B
<missing>                                                                 2 months ago         /bin/sh -c #(nop) ADD file:c92c248239f8c7b9b3c067650954815f391b7bcb09023f984972c082ace2a8d0 in /    5.57MB
```

无论是执行 `docker run --rm test-shell` 还是 `docker run --rm test-exec` 都会将容器中全部的网卡打印出来。
但如果我们指定打印 `eth0` 网卡的信息，两个容器的差异就会显现出来了：
- shell 风格：
  ```shell
  # docker run --rm test-shell eth0
  eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:03
            inet addr:172.17.0.3  Bcast:172.17.255.255  Mask:255.255.0.0
            UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
            RX packets:1 errors:0 dropped:0 overruns:0 frame:0
            TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
            collisions:0 txqueuelen:0
            RX bytes:90 (90.0 B)  TX bytes:0 (0.0 B)

  lo        Link encap:Local Loopback
            inet addr:127.0.0.1  Mask:255.0.0.0
            UP LOOPBACK RUNNING  MTU:65536  Metric:1
            RX packets:0 errors:0 dropped:0 overruns:0 frame:0
            TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
            collisions:0 txqueuelen:1000
            RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
  ```
- exec 风格：
  ```shell
  # docker run --rm test-exec eth0
  eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:03
            inet addr:172.17.0.3  Bcast:172.17.255.255  Mask:255.255.0.0
            UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
            RX packets:1 errors:0 dropped:0 overruns:0 frame:0
            TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
            collisions:0 txqueuelen:0
            RX bytes:90 (90.0 B)  TX bytes:0 (0.0 B)
  ```

也就是说， shell 风格打出的镜像执行的命令类似于 `bash -c 'ifconfig' eth0` ，结尾的 `eth0` 根本没有作为参数追加到 `ifconfig` 后面。
而 exec 风格打出的镜像，参数才能正确地追加到启动命令的后面，达到 `ifconfig eth0` 的效果。

## ENTRYPOING 和 CMD 联用

`ENTRYPOINT` 和 `CMD` 命令联用，可以实现由 `ENTRYPOINT` 命令指定预设的应用程序启动命令，由 `CMD` 声明默认启动参数的效果。
我们用这样一份 Dockerfile 打出名为 `test` 的镜像：
```Dockerfile
FROM alpine

ENTRYPOINT ["ifconfig"]
CMD ["lo"]
```
这样此镜像的启动程序就是 `ifconfig` 命令，默认的参数是用来显示本地回环地址的。而我们也可以通过在 `docker run` 命令后追加参数来打印其他网卡：
默认参数效果：
```shell
# docker run --rm test
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```
手动指定其他参数：
```shell
# docker run --rm test eth0
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:03
          inet addr:172.17.0.3  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:1 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:90 (90.0 B)  TX bytes:0 (0.0 B)
```
可以看到追加到命令末尾的 `eth0` 参数覆盖了 `CMD` 指定的默认参数 `lo` ，但是 `ENTRYPOINT` 指定的启动命令 `ifconfig` 是不变的。

**注意**：要达到这种效果，**必须以 exec 形式书写 `CMD` 和 `ENTRYPOINT` 命令**，使用 shell 形式书写达不到效果的。

## 本文验证环境：

- 宿主机：Ubuntu 18.04

- docker：

  ```shell
  # docker version
  Client:
   Version:           18.06.1-ce
   API version:       1.38
   Go version:        go1.10.4
   Git commit:        e68fc7a
   Built:             Fri Oct 19 19:43:14 2018
   OS/Arch:           linux/amd64
   Experimental:      false

  Server:
   Engine:
    Version:          18.06.1-ce
    API version:      1.38 (minimum version 1.12)
    Go version:       go1.10.4
    Git commit:       e68fc7a
    Built:            Thu Sep 27 02:39:50 2018
    OS/Arch:          linux/amd64
    Experimental:     false
  ```

## 参考文档

- [Dockerfile: ENTRYPOINT和CMD的区别][]

[Dockerfile: ENTRYPOINT和CMD的区别]: https://zhuanlan.zhihu.com/p/30555962