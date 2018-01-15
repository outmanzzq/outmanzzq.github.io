---
layout: post
title: 通过 SSH 连接本机 Docker 容器
categories: ssh
description: 通过 SSH 连接本机 Docker 容器
keywords: ssh, docker
---

> 虽然 Docker 提供了 docker attach 和 docker exec 命令可直接进入容器，但有时候我们需要通过 SSH 方式连接容器，以方便做相关测试。

# 一、环境说明

- 系统： macOS 10.12.6
- Docker： 17.09.1-ce
- Docker Image:   hermsi/alpine-sshd:lastest

> 说明：本次使用镜像基于 alpine 内建 ssh 服务，当然也可用其他镜像手动安装 ssh 服务。

# 二、操作过程

## 1. 拉取镜像

```shell
➜ docker pull hermsi/alpine-sshd

# 检查
➜ docker images
REPOSITORY                                             TAG                 IMAGE ID            CREATED             SIZE
hermsi/alpine-sshd                                     latest              ea32527f361c        2 days ago          8.68MB
```

## 2. 启动容器，并将容器 SSH 服务端口（默认 22 ）映射本机 222 端口

```shell
# 启动容器，通过 --env 设置 root 初始密码
➜ docker run -dit --name 'remote-ssh' -p 222:22 --env ROOT_PASSWORD=root hermsi/alpine-sshd
519e948de7c5abd1b7d6df5f6aa32e55a68871d20cea937dbf4c67a3f5f98c1b

# 检查容器启动是否成功
➜ docker ps
CONTAINER ID        IMAGE                COMMAND             CREATED             STATUS              PORTS                 NAMES
519e948de7c5        hermsi/alpine-sshd   "entrypoint.sh"     6 seconds ago       Up 3 seconds        0.0.0.0:222->22/tcp   remote-ssh

```

## 3. ssh 连接容器

```shell
➜ ssh root@localhost -p 222

The authenticity of host '[localhost]:222 ([127.0.0.1]:222)' can't be established.
ECDSA key fingerprint is SHA256:ZymJaZDCryfrLcp/cgwQAZaYcsFCKbsGn1QdgbEblRo.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[localhost]:222' (ECDSA) to the list of known hosts.
root@localhost's password:
Welcome to Alpine!

The Alpine Wiki contains a large amount of how-to guides and general
information about administrating Alpine systems.
See <http://wiki.alpinelinux.org>.

You can setup the system with the command: setup-alpine

You may change this message by editing /etc/motd.

519e948de7c5:~# cat /etc/os-release
NAME="Alpine Linux"
ID=alpine
VERSION_ID=3.7.0
PRETTY_NAME="Alpine Linux v3.7"
HOME_URL="http://alpinelinux.org"
BUG_REPORT_URL="http://bugs.alpinelinux.org"
519e948de7c5:~# ps
PID   USER     TIME   COMMAND
    1 root       0:00 /usr/sbin/sshd -D -e
   14 root       0:00 sshd: root@pts/1
   16 root       0:00 -ash
   18 root       0:00 ps
519e948de7c5:~# exit
Connection to localhost closed.

```

# 三、进阶：采用 Docker-Compose 管理

## 1. docker-compose-alpine-ssh.yml 编排文件内容

```YAML
version: '2'
services:
  alpine-ssh:
    image: hermsi/alpine-sshd:latest
    container_name: remote-ssh
    environment:
      ROOT_PASSWORD: root
    ports:
      - "222:22"
```

## 2. 启动并测试 ssh 连接

```shell
# 启动并实时查看日志
➜  docker-compose -f docker-compose-alpine-ssh.yml up

# 后台运行
➜  docker-compose -f docker-compose-alpine-ssh.yml up -d
```