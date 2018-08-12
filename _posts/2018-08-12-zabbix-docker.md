---
layout: post
title: Docker Compose 快速搭建zabbix监控系统
categories: docker,zabbix
description: Docker Compose 快速搭建zabbix监控系统
keywords: docker,zabbix
---

> 在真实环境搭建一套zabbix系统是件费时费力的事情，本文内容就是用docker来缩减搭建时间，目标是让读者们尽快投入zabbix系统的体验和实践.

# 环境说明

- OS：Ubuntu 18.04 x86_64
- IP: 192.168.145.110
- Docker: version 17.12.1-ce
- Docker Compose: version 1.17.1

- zabbix-docker仓库：<https://github.com/outmanzzq/docker-compose.git> (zabbix-docker目录)
- 环境变量文件说明
  - .env_agent    zabbix-agent 配置文件
  - .env_db_mysql zabbix-server连接mysql配置文件
  - .env_srv      zabbix-server配置文件
  - .env_web      zabbix-server web 服务配置文件
  - zbx_env       zabbix-server启动后数据库及配置映射持久化存储目录
- simkai.ttf        zabbix-web 中文字体文件（解决中文乱码问题）
- docker-compose.yml zabbix for docker 容器编排配置文件

# 具体部署过程

## zabbix server 部署

### 1. 安装docker、docker-compose

```bash
# 创建工作目录
sudo mkdir -p /server/{tools,backup,scripts}
cd /server/

# 更换阿里源
sudo cp /etc/apt/sources.list{,.ori}

sudo cat > /etc/apt/sources.list <<EOF
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
EOF

sudo apt update

# 删除旧版本
sudo apt-get remove docker docker-engine docker.io

# 安装
sudo apt-get install docker-ce docker-compose git lrzsz -y

# 验证
$ docker -v
Docker version 17.12.1-ce, build 7390fc

$ docker-compose -v
docker-compose version 1.17.1, build unknown
```

> ubuntu 安装 docker 参考:  <https://docs.docker.com/v17.09/engine/installation/linux/docker-ce/ubuntu/#set-up-the-repository>
>
> CentOS 安装 docker 参考：<https://docs.docker.com/v17.09/engine/installation/linux/docker-ce/centos/>

### 2. 拉取 zabbix-docker docker-compose文件

```bash
sudo git clone https://github.com/outmanzzq/docker-compose.git

```

> 如果拉取失败，可直接访问仓库地址，然后选择下载zip文件，再通过unzip 解压。

### 3. 检查docker-compose配置文件，并启动zabbix-server服务

```bash
cd /server/docker-compose/zabbix-docker

# docker-compose.yml文件内容
version: '2'
services:
 zabbix-server:
  image: zabbix/zabbix-server-mysql:alpine-3.4-latest
  ports:
   - "10051:10051"
  volumes:
   - /etc/localtime:/etc/localtime:ro
   - /etc/timezone:/etc/timezone:ro 
   - ./zbx_env/usr/lib/zabbix/alertscripts:/usr/lib/zabbix/alertscripts:ro
   - ./zbx_env/usr/lib/zabbix/externalscripts:/usr/lib/zabbix/externalscripts:ro
   - ./zbx_env/var/lib/zabbix/modules:/var/lib/zabbix/modules:ro
   - ./zbx_env/var/lib/zabbix/enc:/var/lib/zabbix/enc:ro
   - ./zbx_env/var/lib/zabbix/ssh_keys:/var/lib/zabbix/ssh_keys:ro
   - ./zbx_env/var/lib/zabbix/mibs:/var/lib/zabbix/mibs:ro
  volumes_from:
   - zabbix-snmptraps:ro
  links:
   - mysql-server:mysql-server
   - zabbix-java-gateway:zabbix-java-gateway
  ulimits:
   nproc: 65535
   nofile:
    soft: 20000
    hard: 40000
  mem_limit: 512m
  env_file:
   - .env_db_mysql
   - .env_srv

 zabbix-web-nginx-mysql:
  image: zabbix/zabbix-web-nginx-mysql:alpine-3.4-latest
  ports:
   - "8081:80"
   - "8443:443"
  links:
   - mysql-server:mysql-server
   - zabbix-server:zabbix-server
  mem_limit: 512m
  volumes:
   - /etc/localtime:/etc/localtime:ro
   - /etc/timezone:/etc/timezone:ro
   - ./zbx_env/etc/ssl/nginx:/etc/ssl/nginx:ro
   - ./simkai.ttf:/usr/share/zabbix/fonts/graphfont.ttf:ro
  env_file:
   - .env_db_mysql
   - .env_web

 zabbix-agent:
  image: zabbix/zabbix-agent:alpine-3.4-latest
  ports:
   - "10050:10050"
  volumes:
   - /etc/localtime:/etc/localtime:ro
   - /etc/timezone:/etc/timezone:ro
   - ./zbx_env/etc/zabbix/zabbix_agentd.d:/etc/zabbix/zabbix_agentd.d:ro
   - ./zbx_env/var/lib/zabbix/modules:/var/lib/zabbix/modules:ro
   - ./zbx_env/var/lib/zabbix/enc:/var/lib/zabbix/enc:ro
   - ./zbx_env/var/lib/zabbix/ssh_keys:/var/lib/zabbix/ssh_keys:ro
  links:
   - zabbix-server:zabbix-server
  env_file:
   - .env_agent
  privileged: true
  pid: "host"

 zabbix-java-gateway:
   image: zabbix/zabbix-java-gateway:alpine-3.4-latest
   ports:
    - "10052:10052"
   env_file:
    - .env_java
 
 zabbix-snmptraps:
   image: zabbix/zabbix-snmptraps:alpine-3.4-latest
   ports:
    - "162:162/udp"
   volumes:
    - ./zbx_env/var/lib/zabbix/snmptraps:/var/lib/zabbix/snmptraps:rw

 mysql-server:
  image: mysql:5.7
  command: [mysqld, --character-set-server=utf8, --collation-server=utf8_bin]
  volumes_from:
    - db_data_mysql
  volume_driver: local
  env_file:
   - .env_db_mysql

 db_data_mysql:
    image: busybox
    volumes:
    - ./zbx_env/var/lib/mysql:/var/lib/mysql:rw

# 检查
sudo docker-compose config

# 启动zabbix-server服务
sudo docker-compose up -d

# 查看日志(此时会自动拉取相关docker镜像，最后如无报错，则说明启动成功)
sudo docker-compose logs -f

```

### 4. 通过web访问zabbix

- web访问地址：<192.168.145.110:8081> （Admin/zabbix)

### 5. 通过docker内置zabbix-agent服务监控zabbix-server自身

web管理界面点击: 配置=》主机=》zabbix-server, agent代理程序的接口选项：

- DNS名称：zabbix-agent,并选择 DNS
- 勾选： 已启用
- 点击：更新

最终效果：

![ ](/images/20180812-zabbix-config-01.png)

![ ](/images/20180812-zabbix-config-02.png)

## 被监控端 zabbix agent 安装

安装 zabbix-agent,并在/etc/zabbix/zabbix_agentd.conf 中将server 参数指定 zabbix-server IP即可。

> 参考链接：
> - 官方文档：<https://www.zabbix.com/documentation/3.4/zh/manual/installation/containers>
> - 官方仓库：<https://github.com/zabbix/zabbix-docker.git>