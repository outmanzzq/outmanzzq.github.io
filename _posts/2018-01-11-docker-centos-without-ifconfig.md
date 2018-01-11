---
layout: post
title: Docker安装 Centos 后没有 ifconfig 命令解决办法
categories: docker
description: sdocker 安装 centos 后没有 ifconfig 命令解决办法
keywords: docker, centos, ifconfig
---

> 使用 docker pull centos 命令下载下来的 centos 镜像是 centos7 的最小安装包，里面并没有携带 ifconfig 或 ip 命令，导致无法查看容器内的ip，可使用 yum provides ifconfig 查询依赖到软件包，yum 安装即可。

# 一、环境说明

- 系统： macOS 10.12.6
- Docker：17.09.1-ce
- Docker Image: CentOS 7

# 二、操作过程

## 1. 运行容器并查询 IP

```shell
➜  docker run -dit --name 'test-c7' centos:7
3d8e19cafde90a7d964f4b3a1bcf4ccdd6da4135c8f885fe5a098161e565d106
➜  docker ps
CONTAINER ID        IMAGE                       COMMAND                  CREATED             STATUS              PORTS                NAMES
3d8e19cafde9        centos:7                    "/bin/bash"              12 seconds ago      Up 9 seconds                             test-c7

# 查询容器 IP 地址
➜  docker exec test-c7 ifconfig
oci runtime error: exec failed: container_linux.go:265: starting container process caused "exec: \"ifconfig\": executable file not found in $PATH"
```

## 2. 进入容器查询命令依赖包并安装

```shell
# 可看到 ifconfig 命令依赖net-tools
➜  ~ docker attach test-c7
[root@3d8e19cafde9 /]# yum provides ifconfig
Loaded plugins: fastestmirror, ovl
base                                                                                                                                               | 3.6 kB  00:00:00
extras                                                                                                                                             | 3.4 kB  00:00:00
updates                                                                                                                                            | 3.4 kB  00:00:00
(1/4): base/7/x86_64/group_gz                                                                                                                      | 156 kB  00:00:00
(2/4): extras/7/x86_64/primary_db                                                                                                                  | 145 kB  00:00:00
(3/4): base/7/x86_64/primary_db                                                                                                                    | 5.7 MB  00:00:02
(4/4): updates/7/x86_64/primary_db                                                                                                                 | 5.2 MB  00:00:04
Determining fastest mirrors
 * base: ftp.sjtu.edu.cn
 * extras: mirrors.sohu.com
 * updates: mirrors.sohu.com
base/7/x86_64/filelists_db                                                                                                                         | 6.7 MB  00:00:01
extras/7/x86_64/filelists_db                                                                                                                       | 528 kB  00:00:00
updates/7/x86_64/filelists_db                                                                                                                      | 3.0 MB  00:00:00
net-tools-2.0-0.22.20131004git.el7.x86_64 : Basic networking tools
Repo        : base
Matched from:
Filename    : /sbin/ifconfig

# 安装依赖包
[root@3d8e19cafde9 /]# yum install -y net-tools
```

## 3. 再次查询，正确获取容器 IP，问题解决。

```shell
# 查询方式一：容器内直接查询
[root@3d8e19cafde9 /]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.2  netmask 255.255.0.0  broadcast 0.0.0.0
        ether 02:42:ac:11:00:02  txqueuelen 0  (Ethernet)
        RX packets 16185  bytes 23722029 (22.6 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 3937  bytes 216916 (211.8 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# 查询方式二：[ctrl+P][ctrl+Q] 优雅退出容器查询
[root@3d8e19cafde9 /]# read escape sequence

➜ docker exec test-c7 ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.2  netmask 255.255.0.0  broadcast 0.0.0.0
        ether 02:42:ac:11:00:02  txqueuelen 0  (Ethernet)
        RX packets 16188  bytes 23722163 (22.6 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 3937  bytes 216916 (211.8 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```

> 参考链接：<http://blog.csdn.net/Magic_YH/article/details/51292095>