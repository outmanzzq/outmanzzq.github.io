---
layout: post
title: Docker 进入已运行容器，exit 不中断容器运行
categories: docker
description: Docker 进入已运行容器，exit 不中断容器运行
keywords: docker
---

> 当进入容器完成相关操作，使用 [ctrl+d] 或 exit 命令退出容器后，使用 docker ps 查看发现容器已经退出,如何避免这种情况呢？

# 一、环境说明

- 系统： macOS 10.12.6
- Docker：17.09.1-ce
- Docker Image: CentOS 7

# 二、问题重现

## 1. 创建并运行容器

```shell
➜ docker images
REPOSITORY                                             TAG                 IMAGE ID            CREATED             SIZE
centos                                                 7                   ff426288ea90        2 days ago          207MB

# 运行容器
➜ docker run -dit --name 'test-c7-exit' centos:7
a053d7016ed133c969f26ac6c31013e351d3c646ba1ea94b0b906562deb97116

➜  docker git:(master) docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
a053d7016ed1        centos:7            "/bin/bash"         5 seconds ago       Up 2 seconds
```

# 2. 进入容器完成相关操作后，退出后发现容器停止

```shell
# 进入容器
➜ docker attach test-c7-exit
[root@a053d7016ed1 /]# pwd
/
# 按 [ctrl+d] 或 exit 退出容器
[root@a053d7016ed1 /]# exit

# 此时看到容器也停止运行
➜ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED              STATUS                     PORTS               NAMES
a053d7016ed1        centos:7            "/bin/bash"         About a minute ago   Exited (0) 9 seconds ago                       test-c7-exit
```

# 三、原因分析

> 退出时，使用 [ctrl + D]，这样会结束 docker 当前线程，容器结束，可以使用 [ctrl + P][ctrl + Q] 组合键退出而不终止容器运行

# 四、解决过程

```shell
# 重新启动容器
➜ docker start test-c7-exit
test-c7-exit

# 查看容器当前状态
➜ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED              STATUS              PORTS               NAMES
a053d7016ed1        centos:7            "/bin/bash"         About a minute ago   Up 5 seconds                            test-c7-exit

# 重新进入容器
➜ docker attach test-c7-exit

# 此时使用[ctrl + P][ctrl + Q]退出
[root@a053d7016ed1 /]# read escape sequence

# 再次检查容器状态，发现仍在运行！
➜  docker git:(master) docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED              STATUS              PORTS               NAMES
a053d7016ed1        centos:7            "/bin/bash"         About a minute ago   Up 26 seconds                           test-c7-exit

```

> 参考链接：<http://blog.csdn.net/o1_1o/article/details/52710733>