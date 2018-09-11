---
layout: post
title: 基于 Docker Compose 搭建 Redmine 项目管理系统
categories: docker,redmine
description: 基于 Docker Compose 搭建 Redmine 管理系统
keywords: docker,redmine
---

> Redmine 是用 Ruby 开发的基于 web 的项目管理软件，是用 ROR 框架开发的一套跨平台项目管理系统，据说是源于 Basecamp 的 ror 版而来，支持多种数据库，有不少自己独特的功能，例如提供 wiki、新闻台等，
>
> 还可以集成其他版本管理系统和 BUG 跟踪系统，例如 Perforce、SVN、CVS、TD 等等。
>
> 这种 Web 形式的项目管理系统通过“项目（Project）”的形式把成员、任务（问题）、文档、讨论以及各种形式的资源组织在一起，大家参与更新任务、文档等内容来推动项目的进度，同时系统利用时间线索和各种动态的报表形式来自动给成员汇报项目进度.

# 环境说明

- OS: Centos7 x86_64
- IP：192.168.61.33
- Docker: 18.06.0-ce
- Docker Compose: 1.22.0
- Redmine: 3.4.6
- mysql: 5.7

- docker compose 文件路径： /server/docker-compose/redmine/docker-compose.yml
- 数据存储路径：/server/docker-compose/datadir

# 一、具体操作过程

## 1. 更换 YUM 软件源（阿里云）

```bash
# 默认 root 用户，或采用 sudo -i 切换临时 root 用户
$ cp /etc/yum.repos.d/CentOS-Base.repo{,.backup}
$ curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
$ yum makecache
```

## 2. 安装 docker 及 docker-compose

```bash
# 卸载旧版 docker
$ yum remove -y docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine

# 安装依赖包
$ yum install -y yum-utils \
           device-mapper-persistent-data \
           lvm2

# 安装软件源
$ yum-config-manager \
    --add-repo \
    https://mirrors.ustc.edu.cn/docker-ce/linux/centos/docker-ce.repo

# 安装 docker
$ yum makecache fast && yum install -y docker-ce

# 安装 docker-compose
$ curl -L https://github.com/docker/compose/releases/download/1.17.1/docker-compose-`uname -s`-`uname -m` > /usr/bin/docker-compose
$ chmod +x /usr/bin/docker-compose

```

## 3. 部署 Redmine

/server/docker-compose/redmine/docker-compose.yml 文件内容：

```YAML
version: '3.1'

services:

  redmine:
    image: redmine
    restart: always
    ports:
      - 8080:3000
    environment:
      REDMINE_DB_MYSQL: db
      REDMINE_DB_PASSWORD: pwd_root
    volumes:
      - ./datadir:/usr/src/redmine/files

  db:
    image: mysql:5.7
    restart: always
    command: mysqld --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci --init-connect='SET NAMES utf8mb4;' --innodb-flush-log-at-trx-commit=0
    environment:
      MYSQL_ROOT_PASSWORD: pwd_root
      MYSQL_DATABASE: redmine
    volumes:
      - ./datadir:/var/lib/mysql:rw
```

## 4. 常用命令

```bash
# 进入工作目录
$ cd /server/docker-compose/redmine

# 验证
$ docker-compose config

# 启动(后台常驻模式)
$ docker-compose up -d

# 查看实时日志
$ docker-compose logs -f

```

## 5. web界面登陆

浏览器访问宿主机 IP:8080（默认admin/admin)

<http://192.168.61.33:8080>

# 二、常见问题

## 1. Redmine 平台无法输入中文项目名称？

报错信息：

```bash
Internal error
An error occurred on the page you were trying to access.
If you continue to experience problems please contact your Redmine administrator for assistance.

If you are the Redline administrator, check your log files for details about the error.
```

### 原因：

由于 MySQL 数据库(<=5.7)的字符集默认是 latin1，Redmine 官方给的脚本，默认是使用的 utf8 编码，所有的数据库表创建都是未指定字符集的!

故需要将所有的数据表的字符集改变为 utf-8。(docker-compose 文件中已通过 command 命令设置)

> 参考链接：
> - Redmine官方：<https://www.redmine.org>
> - Redmine使用：<http://www.redmine.org.cn>
> - Docker Hub: <https://hub.docker.com/_/redmine/>
