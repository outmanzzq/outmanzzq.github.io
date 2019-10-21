---
layout: post
title: docker 与 gosu
categories: docker,gosu
description: gosu 是个工具，用来提升指定账号的权限，作用与 sudo 命令类似，而 docker 中使用 gosu 的起源来自安全问题；
keywords: docker,gosu
---

> gosu 是个工具，用来提升指定账号的权限，作用与 sudo 命令类似，而 docker 中使用 gosu 的起源来自安全问题；

docker 容器中运行的进程，如果以 root 身份运行的会有安全隐患，该进程拥有容器内的全部权限，更可怕的是如果有数据卷映射到宿主机，那么通过该容器就能操作宿主机的文件夹了，一旦该容器的进程有漏洞被外部利用后果是很严重的。
因此，容器内使用非 root 账号运行进程才是安全的方式，这也是我们在制作镜像时要注意的地方。

这里有篇文章也推荐在容器中使用最小权限的账号：<https://snyk.io/blog/10-docker-image-security-best-practices/>

## 在镜像中创建非 root 账号

既然不能用 root 账号，那就要创建其他账号来运行进程了，以 redis 官方镜像的[Dockerfile](https://github.com/docker-library/redis/blob/6845f6d4940f94c50a9f1bf16e07058d0fe4bc4f/5.0/Dockerfile)为例，

来看看如何创建账号:

![ ](/images/20191021-gosu-for-docker-01.jpg)

可见 redis 官方镜像使用 groupadd 和 useradd 创建了名为 redis 的组合账号，接下来就是用 redis 账号来启动服务了，理论上应该是以下套路：

1. 用 USER redis 将账号切换到 redis。
2. 在 docker-entrypoint.sh 执行的时候已经是 redis 身份了，如果遇到权限问题，例如一些文件只有 root 账号有读、写、执行权限，用 sudo xxx 命令来执行即可。

`但事实并非如此！`

在 Dockerfile 脚本中未发现 USER redis 命令，这意味着执行 docker-entrypoint.sh 文件的身份是 root；
其次，在 docker-entrypoint.sh 中没有发现 su - redis 命令，也没有 sudo 命令；

在 Dockerfile 脚本中未发现 USER redis 命令，这意味着执行 docker-entrypoint.sh 文件的身份是 root；
其次，在 docker-entrypoint.sh 中没有发现 `su - redis` 命令，也没有 sudo 命令；

## 确认 redis 服务的启动账号

还是自己动手来证实一下吧，我的环境信息如下：

- 操作系统：CentOS Linux release 7.6.1810
- Docker： 1.13.1

### 操作步骤如下

```bash
#1 启动一个 redis 容器：
$ docker run --name myredis -idt redis

#2 进入容器：
$ docker exec -it myredis /bin/bash

#3 在容器内，先更新 apt：
$ apt-get update

#4 安装 ps 命令：
$ apt-get install procps

#5 执行命令 ps -ef 查看 redis 服务，结果如下：
$ root@122c2df16bbb:/data# ps -ef
UID         PID   PPID  C STIME TTY          TIME CMD
redis         1      0  0 09:22 ?        00:00:01 redis-server *:6379
root        287      0  0 09:36 ?        00:00:00 /bin/bash
root        293    287  0 09:39 ?        00:00:00 ps -ef
```

上面的结果展示了两个关键信息：

- 第一，redis 服务是 redis 账号启动的，并非 root；
- 第二，redis 服务的 PID 等于 1，这很重要，宿主机执行 `docker stop` 命令时，该进程可以收到 SIGTERM 信号量，于是 redis 应用可以做一些退出前的准备工作，例如保存变量、退出循环等，也就是优雅停机(Gracefully Stopping)；

现在我们已经证实了 redis 服务并非 root 账号启动，而且该服务进程在容器内还是一号进程，但是我们在 Dockerfile 和 docker-entrypoint.sh 脚本中都没有发现切换到 redis 账号的命令，也没有 sudo 和 su，这是怎么回事呢？

> **答案是 gosu**

再看一次[redis的docker-entrypoint.sh](https://github.com/docker-library/redis/blob/6845f6d4940f94c50a9f1bf16e07058d0fe4bc4f/5.0/docker-entrypoint.sh)文件，如下图:

![ ](/images/20191021-gosu-for-docker-02.jpg)

注意上图中的代码，我们来分析一下：

1. 假设启动容器的命令是 `docker run --name myredis -idt redis redis-server /usr/local/etc/redis/redis.conf`；
2. 容器启动后会执行 docker-entrypoint.sh 脚本，此时的账号是 root；
3. 当前账号是 root，因此会执行上图红框中的逻辑；
4. 红框中的 $0 表示当前脚本的名称,即 docker-entrypoint.sh；
5. 红框中的 `$@` 表示外部传入的所有参数，即redis-server /usr/local/etc/redis/redis.conf；
6. gosu redis “$0” “@”，表示以redis账号的身份执行以下命令：
`docker-entrypoint.sh redis-server /usr/local/etc/redis/redis.conf`
7. gosu redis “$0” "@" 前面加上个 exec，表示以 gosu redis “$0” "@" 这个命令启动的进程替换正在执行的 docker-entrypoint.sh 进程，这样就保证了 gosu redis “$0” "@" 对应的进程 ID 为 1；
8. gosu redis “$0” "@" 导致 docker-entrypoint.sh 再执行一次，但是当前的账号已经不是 root 了，于是会执行兜底逻辑 exec “$@”；
9. 此时的 $@ 是 redis-server /usr/local/etc/redis/redis.conf，因此 redis  服务会启动，而且账是 redis；
10. $@前面有个 exec，会用 redis-server 命令启动的进程取代当前的 docker-entrypoint.sh 进程，所以，最终 redis 进程的 PID 等于 1，而 docker-entrypoint.sh 这个脚本的进程已经被替代，因此就结束掉了；

## 关于 gosu

通过上面的分析，我们对 gosu 的作用有了基本了解：功能和 sudo 类似，提升指定账号的权限，用来执行指定的命令，其官网地址是：<https://github.com/tianon/gosu> ，如下图所示，官方的描述也是说 su 和 sudo 命令有一些问题，所以才有了 gosu 工具来作为替代品;

在 docker 的官方文档中，也见到了 gosu 的[使用示例](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)，和前面的 redis 很像，如下图:

![ ](/images/20191021-gosu-for-docker-03.jpg)

> 注意上图中底部的那段话：使用 exec XXX 命令以确保 XXX 对应的进程的 PID 保持为 1，这样该进程才能收到宿主机发送给容器的信号量；

## 为什么要用 gosu 取代 sudo

前面主要讲 gosu 的用法，但是为什么要用 gosu 呢？接下来通过实战对比来看看 sudo 的问题在哪：

### 1. 执行以下命令创建一个容器

`docker run --rm gosu/alpine gosu root ps aux`

上述命令会启动一个安装了 gosu 的 linux 容器，并且启动后自动执行命令 gosu root ps aux，作用是以 root 账号的身份执行 ps aux，也就是将当前进程都打印出来，执行结果如下：

```bash
[root@centos7 ~]# docker run --rm gosu/alpine gosu root ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 ps aux
```

上述信息显示，我们执行 docker run 时的 `gosu root ps aux` 会执行 ps 命令，该命令成了容器内的唯一进程，这说明通过 gosu 启动的是符合我们要求的（PID为1），接下来再看看用 sudo 执行 ps 命令的效果；

### 2. 执行以下命令创建一个容器

`docker run --rm ubuntu:trusty sudo ps aux`

上述命令会用 sudo 启动 ps 命令，结果如下：

```bash
USER        PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root          1  0.0  0.0  46012  1772 ?        Rs   12:05   0:00 sudo ps aux
root          6  0.0  0.0  15568  1140 ?        R    12:05   0:00 ps aux
```

尽管我们只想启动 ps 进程，但是容器内出现了两个进程，sudo 命令会创建第一个进程，然后该进程再创建了 ps 进程，而且 ps 进程的 PID 并不等于 1，这是达不到我们要求的，此时在宿主机向该容器发送信号量，收到信号量的是 sudo 进程。

**通过上面对可以小结：**

1. gosu 启动命令时只有一个进程，所以 docker 容器启动时使用 gosu，那么该进程可以做到 PID 等于 1；

2. sudo 启动命令时先创建 sudo 进程，然后该进程作为父进程去创建子进程，1 号 PID 被 sudo 进程占据；

综上所述，在 docker 的 entrypoint 中有如下建议：

1. 创建 group 和普通账号，不要使用 root 账号启动进程；
2. 如果普通账号权限不够用，建议使用 gosu 来提升权限，而不是 sudo；
3. entrypoint.sh 脚本在执行的时候也是个进程，启动业务进程的时候，在命令前面加上 exec，这样新的进程就会取代 entrypoint.sh 的进程，得到 1 号 PID；
4. exec "$@" 是个保底的逻辑，如果 entrypoint.sh 的入参在整个脚本中都没有被执行，那么 exec "$@" 会把入参执行一遍，如果前面执行过了，这一行就不起作用，这个命令的细节在 Stack Overflow 上有[详细的描述](https://stackoverflow.com/questions/39082768/what-does-set-e-and-exec-do-for-docker-entrypoint-scripts)

## 如何在镜像中安装 gosu

前面的 redis 例子中，我们看到 docker-entrypoint.sh 中用到了 gosu，那么是在哪里安装了 gosu 呢？自然是 Dockerfile 中，一起来看看 redis 的 Dockerfile 中是如何安装 gosu 的：

```Dockerfile
# grab gosu for easy step-down from root
# https://github.com/tianon/gosu/releases
ENV GOSU_VERSION 1.10
RUN set -ex; \
  \
  fetchDeps=" \
    ca-certificates \
    dirmngr \
    gnupg \
    wget \
    "; \
    apt-get update; \
    apt-get install -y --no-install-recommends $fetchDeps; \
    rm -rf /var/lib/apt/lists/*; \
    \
    dpkgArch="$(dpkg --print-architecture | awk -F- '{ print $NF }')"; \
    wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch"; \
    wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch.asc"; \
    export GNUPGHOME="$(mktemp -d)"; \
    gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4; \
    gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu; \
    gpgconf --kill all; \
    rm -r "$GNUPGHOME" /usr/local/bin/gosu.asc; \
    chmod +x /usr/local/bin/gosu; \
    gosu nobody true; \
    \
```

> 至此，gosu 在 docker 中的作用已经分析完毕，希望在您编写自定义镜像的时候，本文能给您带来一些参考；

> 原文链接：<https://blog.csdn.net/boling_cavalry/article/details/93380447s>