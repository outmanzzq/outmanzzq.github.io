---
layout: post
title: 通过 Docker Swarm 集群部署 Traefik
categories: docker,swarm,traefik
description: 通过 Docker Swarm 集群部署 Traefik
keywords: docker,swarm,traefik
---

> 本节将测试通过 docker-machine 创建 Docker Swarm 集群，并部署 Traefik

# 环境介绍

- 宿主机： MacOS 10.14.6
  - VirtualBox: 5.2.18 r124319 (Qt5.6.3)
  - docker-machine: 0.16.1, build cce350d7
  - boot2docker: v19.03.2-rc1
  - Docker: 18.09.6, build 481bc77
  - traefik: v1.7.14

通过 docker-machine 创建 3 台服务器

主机名 | IP | Swarm 集群角色 | 备注
-|-
manager | 192.168.99.100 | manager | -
worker1 | 192.168.99.101 | worker1 | -
worker2 | 192.168.99.102 | worker2 | -

## 前期准备

1. 需安装 [docker-machine](https://docs.docker.com/machine/)
2. 需安装 [VirtualBox](https://www.virtualbox.org/wiki/Downloads)

> 为加快 docker-machine 创建 vm 速度，建议:
> - 手动下载 [boot2docker.iso](https://github.com/boot2docker/boot2docker/releases/download/v19.03.2-rc1/boot2docker.iso)
> - 放到 ~/.docker/machine/cache/boot2docker.iso 目录
> - 创建 alias

```bash
sudo bash -c 'cat >>~/.profile <<EOF
alias docker-machinec='docker-machine create -d virtualbox   --virtualbox-boot2docker-url ~/.docker/machine/cache/boot2docker.iso'
EOF

source  ~/.profile
```

# 一、具体操作

## 1.1 创建并初始化 Docker Swarm 集群

```bash
# create docker-machine manager,work1,work2
$ for host in manager worker1 worker2; do docker-machinec $host; done

# init cluster
##manager
$ docker-machine ssh manager "docker swarm init \
    --listen-addr $(docker-machine ip manager) \
    --advertise-addr $(docker-machine ip manager)"

$ export worker_token=$(docker-machine ssh manager "docker swarm join-token worker -q")

##worker1
$ docker-machine ssh worker1 "docker swarm join \
    --token=${worker_token} \
    --listen-addr $(docker-machine ip worker1) \
    --advertise-addr $(docker-machine ip worker1) \
    $(docker-machine ip manager)"

##worker2
$ docker-machine ssh worker2 "docker swarm join \
    --token=${worker_token} \
    --listen-addr $(docker-machine ip worker2) \
    --advertise-addr $(docker-machine ip worker2) \
    $(docker-machine ip manager)"

##check
$ docker-machine ssh manager docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
tnc7tqgwjp5eqba0i1zji0mnj *   manager             Ready               Active              Leader              18.09.6
lwvomgs80i85zzkh4ptwewpij     worker1             Ready               Active                                  18.09.6
w2cgm0zib6dxs9d6mn07ladsf     worker2             Ready               Active                                  18.09.6

```

## 1.2 部署 Traefik

```bash
#创建 traefik 使用的网络
docker-machine ssh manager "docker network create --driver=overlay traefik-net"

# manager 节点部署 traefik 服务
$ docker-machine ssh manager "docker service create \
    --name traefik \
    --constraint=node.role==manager \
    --publish 80:80 --publish 8080:8080 \
    --mount type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock \
    --network traefik-net \
    traefik \
    --docker \
    --docker.swarmMode \
    --docker.domain=traefik \
    --docker.watch \
    --api"v
```

启动参数说明：

选项 | 说明
-|-
--publish 80:80 --publish 8080:8080 | 80 端口为集群监听端口，8080 为 webUI 访问端口
--constraint=node.role==manager     | 指定在集群 manager 角色节点部署
--mount type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock | 监听 Docker socket API 事件（容器创建、删除等）
--network traefik-net               | 指定容器使用的网络名称
--docker                            | 指定为 docker 引擎
--api                               | 开启 WebUI 访问

## 1.3 部署应用

```bash
#app1
$ docker-machine ssh manager "docker service create \
    --name whoami0 \
    --label traefik.port=80 \
    --network traefik-net \
    containous/whoami"

#app2
$ docker-machine ssh manager "docker service create \
    --name whoami1 \
    --label traefik.port=80 \
    --network traefik-net \
    --label traefik.backend.loadbalancer.sticky=true \
    containous/whoami"

#检查服务
$ docker-machine ssh manager "docker service ls"
ID            NAME     MODE        REPLICAS  IMAGE                     PORTS
moq3dq4xqv6t  traefik  replicated  1/1       traefik:latest            *:80->80/tcp,*:8080->8080/tcp
ysil6oto1wim  whoami0  replicated  1/1       containous/whoami:latest
z9re2mnl34k4  whoami1  replicated  1/1       containous/whoami:latest
```

> 说明：通过 `--label traefik.backend.loadbalancer.sticky=true` 实现会话保持。

## 1.4. 通过 Traefik 访问应用

```bash
#app1: whoami0
$ curl -H Host:whoami0.traefik http://$(docker-machine ip manager)
Hostname: dfa25599e253
IP: 127.0.0.1
IP: 10.0.0.13
IP: 172.18.0.4
GET / HTTP/1.1
Host: whoami0.traefik
User-Agent: curl/7.54.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 10.255.0.2
X-Forwarded-Host: whoami0.traefik
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: 7403d79c27d0
X-Real-Ip: 10.255.0.2

#app2: whoami1
$ curl -H Host:whoami1.traefik http://$(docker-machine ip manager)
Hostname: 44e396b2b959
IP: 127.0.0.1
IP: 10.0.0.17
IP: 172.18.0.5
GET / HTTP/1.1
Host: whoami1.traefik
User-Agent: curl/7.54.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 10.255.0.2
X-Forwarded-Host: whoami1.traefik
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: 7403d79c27d0
X-Real-Ip: 10.255.0.2
```

> 如果是 Swarm 模式，访问集群任意节点都可得到相同结果

```bash
#cluster worker1
$ curl -H Host:whoami0.traefik http://$(docker-machine ip worker1)
Hostname: f1ecf18a1bd2
IP: 127.0.0.1
IP: 10.0.0.14
IP: 172.18.0.4
GET / HTTP/1.1
Host: whoami0.traefik
User-Agent: curl/7.54.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 10.255.0.3
X-Forwarded-Host: whoami0.traefik
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: 7403d79c27d0
X-Real-Ip: 10.255.0.3

#cluster worker2
$ curl -H Host:whoami1.traefik http://$(docker-machine ip worker2)
Hostname: a7f079a2ec71
IP: 127.0.0.1
IP: 10.0.0.18
IP: 172.18.0.6
GET / HTTP/1.1
Host: whoami1.traefik
User-Agent: curl/7.54.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 10.255.0.4
X-Forwarded-Host: whoami1.traefik
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: 7403d79c27d0
X-Real-Ip: 10.255.0.4
```

## 1.5 伸缩服务

```bash
$ docker-machine ssh manager "docker service scale whoami0=5"

$ docker-machine ssh manager "docker service scale whoami1=5"

#check
$ docker-machine ssh manager "docker service ls"
ID                  NAME                MODE                REPLICAS            IMAGE                      PORTS
k36uevddcmkc        traefik             replicated          1/1                 traefik:latest             *:80->80/tcp, *:8080->8080/tcp
tjby5a5gfrti        whoami0             replicated          5/5                 containous/whoami:latest
246bcqk4imw1        whoami1             replicated          5/5                 containous/whoami:latest
```

## 1.6 测试会话保持

### 因 whoami0 未设置会话保持，故每次访问返回容器节点 IP 不同

```bash
$ for n in `seq 5`
do
  echo "$n >>>"
  curl -c cookies.txt -sH Host:whoami0.traefik http://$(docker-machine ip manager) |grep -m 2 'IP'
done

#输出结果（第二行IP）
1 >>>
IP: 127.0.0.1
IP: 10.0.0.11
2 >>>
IP: 127.0.0.1
IP: 10.0.0.12
3 >>>
IP: 127.0.0.1
IP: 10.0.0.6
4 >>>
IP: 127.0.0.1
IP: 10.0.0.13
5 >>>
IP: 127.0.0.1
IP: 10.0.0.14
```

### 因 whoami1 已设置会话保持，故每次访问返回容器节点 IP 相同（会话保持）

```bash
#第一次访问,设置 cookie
$ curl -c cookies.txt -sH Host:whoami1.traefik http://$(docker-machine ip manager) |grep -m 2 'IP';

# 第二次访问
## 如果访问时指定 cookie 文件，则讲访问同一台服务器（容器）
$ for n in `seq 5`
do
    echo "$n >>>"
    curl -b cookies.txt -sH Host:whoami1.traefik http://$(docker-machine ip manager) |grep -m 2 'IP';
done

#输出结果
1 >>>
IP: 127.0.0.1
IP: 10.0.0.18
2 >>>
IP: 127.0.0.1
IP: 10.0.0.18
3 >>>
IP: 127.0.0.1
IP: 10.0.0.18
4 >>>
IP: 127.0.0.1
IP: 10.0.0.18
5 >>>
IP: 127.0.0.1
IP: 10.0.0.18
```

# 二、懒人版

> 在初始化 docker-machine 集群后，其中 1.2，1.3 步骤可通过 Docker-compse 编排文件部署 Traefik 及 app 服务 直接替代

## 2.1 manager 节点操作

### docker-compose 单机模式

```bash
#登录manager，并安装 docker-compose 工具
$ docker-machine ssh manager

$ sudo curl -o /usr/local/bin/docker-compose https://mirrors.aliyun.com/docker-toolbox/linux/compose/1.21.2/docker-compose-Linux-x86_64 && \
sudo chmod +x /usr/local/bin/docker-compose && \
docker-compose -v

#生成编排文件
$ echo '
version: "3"

services:
  reverse-proxy:
    image: traefik
    restart: always
    command: --api --docker
    networks:
      - traefik-net
    ports:
      - "80:80"
      - "8080:8080"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /dev/null:/traefik.toml
    deploy:
      mode: global
      placement:
        constraints:
          - node.role == manager

  whoami0:
    image: containous/whoami
    restart: always
    networks:
      - traefik-net
    labels:
      - "traefik.backend=whoami0"
      - "traefik.frontend.rule=Host:whoami0.traefik"

  whoami1:
    image: containous/whoami
    restart: always
    networks:
      - traefik-net
    labels:
      - "traefik.backend=whoami1"
      - "traefik.backend.loadbalancer.sticky=true"
      - "traefik.frontend.rule=Host:whoami1.traefik"

networks:
  traefik-net:
    driver: overlay

' > docker-compose.yml

#语法检查
$ docker-compose config

#运行
$ docker-compose up -d

#检查
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS                                                              NAMES
5451484ec7f9        traefik             "/traefik --api --do…"   About a minute ago   Up About a minute   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:8080->8080/tcp   docker_reverse-proxy_1
3f782ace7aaf        containous/whoami   "/whoami"                About a minute ago   Up About a minute   80/tcp                                                             docker_whoami1_1
30e1b5c721b4        containous/whoami   "/whoami"                About a minute ago   Up About a minute   80/tcp
```

> 后面测试参考 1.4 及以后步骤

### docker swarm 集群模式

待补充。。。

> 参考链接： <https://docs.traefik.io/user-guide/swarm-mode/>
