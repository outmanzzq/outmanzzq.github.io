---
layout: post
title: 使用 Portainer 管理 Docker Swarm 集群
categories: docker,portainer,vagrant
description: 使用 Portainer 管理 Docker Swarm 集群
keywords: docker,portainer,vagrant
---

> Portainer 是 Docker 的图形化管理工具，提供状态显示面板、应用模板快速部署、容器镜像网络数据卷的基本操作（包括上传下载镜像，创建容器等操作）、事件日志显示、容器控制台操作、Swarm 集群和服务等集中管理和操作、登录用户管理和控制等功能。功能十分全面，基本能满足中小型单位对容器管理的全部需求。

## 一、环境说明

- 宿主机：MacOS 10.14.5 (需能访问公网)

- vagrant: 2.0.1
- virtualbox: 5.2.18
- box: centos/7.5_18.04_x86-64

- OS: CentOS 7.5_18.04_x86-64
- Docker: 18.09.6
- Docker-compose: 1.21.2

- Portainer: 1.21.0

- Docker Swarm 集群 IP 规划：
  - node01: 192.168.150.181 (master)
  - node02: 192.168.150.182 (worker01)
  - node03: 192.168.150.183 (worker02)

- 一键部署 docker + docker-compose 脚本: install_dockerCE_docker-compose.sh

> 脚本需放在 Vagrantfile 文件同级 scripts 文件夹内，vagrant 自动构建虚拟机后，会自动挂载在 /vagrant/scripts/install_dockerCE_docker-compose.sh

## 二、具体操作

### 1. install_dockerCE_docker-compose.sh 文件内容

```bash
#!/bin/bash

#生成工作目录
mkdir -p /server/{tools,scripts,backup,docker-compose}

##更换阿里云源
mv /etc/yum.repos.d/CentOS-Base.repo{,.backup}
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
yum update -y

##安装常用工具
yum install tree lrzsz -y

#安装docker
yum remove -y docker \
     docker-client \
     docker-client-latest \
     docker-common \
     docker-latest \
     docker-latest-logrotate \
     docker-logrotate \
     docker-selinux \
     docker-engine-selinux \
     docker-engine

yum install -y yum-utils \
     device-mapper-persistent-data \
     lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum makecache fast
yum -y install docker-ce
systemctl start docker
systemctl enable docker
docker version

##镜像加速
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<EOF
{
 "registry-mirrors": ["https://2apmvngw.mirror.aliyuncs.com"]
}
EOF

systemctl daemon-reload
systemctl restart docker

##安装最新 docker-compose
curl -L https://mirrors.aliyun.com/docker-toolbox/linux/compose/$(curl -s https://mirrors.aliyun.com/docker-toolbox/linux/compose/ |egrep '^<a' |awk -F '">|</a>' '{print $2}' |sort -V |tail -1)docker-compose-Linux-x86_64 -o  /usr/bin/docker-compose

chmod +x /usr/bin/docker-compose
docker-compose -v
```

### 2. Vagrantfile 文件内容

```Vagrantfile
# -*- mode: ruby -*-
# vi: set ft=ruby :

node_servers = {
    :node01 => '192.168.150.181',
    :node02 => '192.168.150.182',
    :node03 => '192.168.150.183',
}

Vagrant.configure("2") do |config|
  config.vm.box = "centos/7.5_18.04_x86-64"

  node_servers.each do |node_servers_name, node_server_ip|
    config.vm.define node_servers_name do |node_config|
      node_config.vm.hostname = "#{node_servers_name.to_s}"
      node_config.vm.network :private_network, ip: node_server_ip
      node_config.vm.provider "virtualbox" do |vb|
        vb.name = node_servers_name.to_s
      end
    end

    config.vm.provision "shell", inline: <<-SHELL
      sudo -i

      sed -i  's/^#PermitRootLogin.*/PermitRootLogin yes/g' /etc/ssh/sshd_config && \
      sed -n '/PermitRootLogin/p' /etc/ssh/sshd_config && \
      sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config && \
      sed -n '/^PasswordAuthentication yes/p' /etc/ssh/sshd_config && \
      systemctl restart sshd && \
      echo "root:vagrant" |sudo chpasswd

      ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

      # stop firewall
      systemctl stop firewalld
      systemctl disable firewalld

      # disable selinux
      sed -i 's/\(^SELINUX=\).*/\1disabled/g' /etc/selinux/config

      # disable swap
      #swapoff -a
      #sed -i /swap/s/^/#/g /etc/fstab
      sh /vagrant/scripts/install_dockerCE_docker-compose.sh
    SHELL
  end
end

```

### 3. 构建虚拟机集群

```bash
# 进行 Vagrantfile 语法检查
$ vagrant validate
Vagrantfile validated successfully

# 启动自动构建
$ vagrant up

# 稍等片刻检查结果
$ vagrant status
Current machine states:

node01                    running (virtualbox)
node02                    running (virtualbox)
node03                    running (virtualbox)

# 分别确认 node01,node02,node03 docker,docker-compose 已成功安装
$ vagrant ssh node01 -c 'docker --version && docker-compose -v'
Docker version 18.09.6, build 481bc77156
docker-compose version 1.21.2, build a133471
Connection to 127.0.0.1 closed.

$ vagrant ssh node02 -c 'docker --version && docker-compose -v'
Docker version 18.09.6, build 481bc77156
docker-compose version 1.21.2, build a133471
Connection to 127.0.0.1 closed.

$ vagrant ssh node03 -c 'docker --version && docker-compose -v'
Docker version 18.09.6, build 481bc77156
docker-compose version 1.21.2, build a133471
Connection to 127.0.0.1 closed.

```

### 4. 构建 Docker Swarm 集群

#### node01 上操作

```bash
#登录 node01 虚拟机
$ vagrant ssh node01
$ sudo -i

# 查看 ip
$ ip ad |grep 'inet '
    inet 127.0.0.1/8 scope host lo
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
    inet 192.168.150.181/24 brd 192.168.150.255 scope global noprefixroute eth1
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
    inet 172.18.0.1/16 brd 172.18.255.255 scope global br-94e764529623
#过滤
$ ip ad |grep 'inet ' |awk '{print$2}' |cut -f1 -d '/'
127.0.0.1
10.0.2.15
192.168.150.181
172.17.0.1
172.18.0.1

#切换成 docker swarm 模式(因通过 vagrant 创建到虚拟机有多个网卡，故需指定网卡)
$ docker swarm init --advertise-addr eth1
Swarm initialized: current node (bc8bh2460wx968h6o0jjaf7nk) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-58u3bbfqu3dcujxmgdn7aa65l2xhv3oputd65ueesjwk6cvr3x-ato9p28nf0fw794t7dc3hi3ev 192.168.150.181:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

# 查看加入 docker swarm manager 节点 token（具体按实际生成的 token 为准）
$ docker swarm join-token manager
To add a manager to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-58u3bbfqu3dcujxmgdn7aa65l2xhv3oputd65ueesjwk6cvr3x-dj4q0bknc9hy083nzqh5ezqd1 192.168.150.181:2377

$ docker swarm join-token worker
To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-58u3bbfqu3dcujxmgdn7aa65l2xhv3oputd65ueesjwk6cvr3x-ato9p28nf0fw794t7dc3hi3ev 192.168.150.181:2377

#查看集群状态
$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
bc8bh2460wx968h6o0jjaf7nk *   node01              Ready               Active              Leader              18.09.6
```

#### node02 上操作

```bash
# 登录 node02
$ vagrant ssh node02

# 作为 worker 节点加入集群
$ docker swarm join --token SWMTKN-1-58u3bbfqu3dcujxmgdn7aa65l2xhv3oputd65ueesjwk6cvr3x-ato9p28nf0fw794t7dc3hi3ev 192.168.150.181:2377
This node joined a swarm as a worker
```

> worker 节点提权为 manager 节点，在 manager 节点执行： `docker node promote node02`
> manager 节点降权为 worker 节点，在 manager 节点执行： `docker node demote node02`

#### node03 上操作

```bash
# 登录 node03
$ vagrant ssh node03
$ sudo -i

# 作为 worker 节点加入集群
$ docker swarm join --token SWMTKN-1-58u3bbfqu3dcujxmgdn7aa65l2xhv3oputd65ueesjwk6cvr3x-ato9p28nf0fw794t7dc3hi3ev 192.168.150.181:2377
This node joined a swarm as a worker
```

#### node01 上检查结果

```bash
$ vagrant ssh node01
$ sudo -i

$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
bc8bh2460wx968h6o0jjaf7nk *   node01              Ready               Active              Leader              18.09.6
rq9jeup2d9twv5tlfaeysoipp     node02              Ready               Active              Reachable           18.09.6
fvnuqlvsd1flvxaxe5uwa5zkk     node03              Ready               Active                                  18.09.6
```

### 5. node01 上部署 Portainer

```bash
$ vagrant ssh node01
$ sudo -i

$ mkdir -p /server/docker-compose/portainer && cd /server/docker-compose/portainer

$ cat >> docker-compose.yml <<EOF
version: "3.2"

services:
  portainer:
    image: portainer/portainer
    restart: always
    volumes:
      - ./data:/data
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - '9000:9000'
    networks:
      - overlay
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]

#  agent:
#    image: portainer/agent
#    environment:
#      AGENT_CLUSTER_ADDR: tasks.agent
#    volumes:
#      - /var/run/docker.sock:/var/run/docker.sock
#      - /var/lib/docker/volumes:/var/lib/docker/volumes
#    ports:
#      - target: 9001
#        published: 9001
#        protocol: tcp
#        mode: host
#    networks:
#      - portainer_agent
#    deploy:
#      mode: global
#      placement:
#        constraints: [node.platform.os == linux]

networks:
  overlay:
EOF

# 语法检查并启动
$ docker-compose config
$ docker-compose up -d
```

> 此时宿主机即可通过 192.168.150.183:9000 访问 portainer

通过Portainer查看 docker swarm 集群，图：

![ ](/images/20190614-portainer-for-docker-swarm.jpg)

> 参考链接：

- Portainer 官网: <https://www.portainer.io/>
- storidge: <https://storidge.com>
- <https://www.jianshu.com/p/507f9283b4d5>