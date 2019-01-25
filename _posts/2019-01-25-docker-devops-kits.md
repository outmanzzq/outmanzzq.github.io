---
layout: post
title: Docker Compose 编排 DevOps 工具
categories: Docker,Devops
description: some word here
keywords: docker,devops,git,jenkins,nginx
---

> 在 [Docker nginx 反向代理设置](https://outmanzzq.github.io/2019/01/25/Docker-nginx-reverse-proxy/) 介绍了通过 [nginx](https://nginx.org/en/) 反向代理关联容器。此为真实的使用场景。通过 [Gitea](https://github.com/go-gitea/gitea) 作为代码管理工具；[Kanboard](https://github.com/kanboard/kanboard) 作为任务管理；[Jenkins](https://jenkins.io/) 作为 CI 工具。这样的组合比较适合小型团队使用，相比起 [GitLab](https://gitlab.com/) 这种巨无霸来说，部署简单，使用简单。

## 准备

- 安装 Docker

```shell
$ curl -fsSL get.docker.com -o get-docker.sh
$ sudo sh get-docker.sh

<output truncated>

If you would like to use Docker as a non-root user, you should now consider
adding your user to the "docker" group with something like:

sudo usermod -aG docker your-user

Remember to log out and back in for this to take effect!

WARNING: Adding a user to the "docker" group grants the ability to run
        containers which can be used to obtain root privileges on the
        docker host.
        Refer to https://docs.docker.com/engine/security/security/#docker-daemon-attack-surface
        for more information.
```

- 安装 Docker Compose

```shell
$ sudo curl -L https://github.com/docker/compose/releases/download/1.19.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
$ docker-compose --version
```

> 注：Docker 以及 Docker Compose 的安装，官方文档讲得非常清晰，在此不再赘述。

## `docker-compose.yml` 文件

```yml
version: "3.5"

services:
  mysql:
    image: mysql:latest
    container_name: mysql
    ports:
      - "3306:3306"
    networks:
      - devops
    environment:
      - MYSQL_ROOT_PASSWORD=/run/secrets/db_root_password
    volumes:
      - type: bind
        source: ./mysql/conf.d
        target: /etc/mysql/conf.d
      - type: bind
        source: ./mysql/data
        target: /var/lib/mysql
      # - ./mysql/conf.d:/etc/mysql/conf.d
      # - ./mysql/data:/var/lib/mysql
    secrets:
      - db_root_password
    restart: always

  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    ports:
      - "10080:3000"
      - "10022:22"
    networks:
      - devops
    environment:
      - VIRTUAL_HOST=git.vking.io
      - VIRTUAL_PORT=3000
      - GITEA_CUSTOM=/etc/gitea
    depends_on: 
      - mysql
    volumes:
      - type: bind
        source: ./gitea
        target: /data
      - type: bind
        source: ./gitea/custom
        target: /etc/gitea
      # - ./gitea:/data
      # - ./gitea/custom:/etc/gitea
    restart: always

  task:
    image: kanboard/kanboard:latest
    container_name: kanboard
    ports:
      - "8888:80"
    networks:
      - devops
    environment:
      - VIRTUAL_HOST=task.vking.io
      - VIRTUAL_PORT=80
    volumes:
      - type: bind
        source: ./kanboard/data
        target: /var/www/app/data
      - type: bind
        source: ./kanboard/plugins
        target: /var/www/app/plugins
      # - ./kanboard/data:/var/www/app/data
      # - ./kanboard/plugins:/var/www/app/plugins
    restart: always

  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    ports:
      - "8081:8080"
      - "50000:5000"
    networks:
      - devops
    environment:
      - VIRTUAL_HOST=jenkins.vking.io
      - VIRTUAL_PORT=8080
    volumes:
      - type: bind
        source: ./jenkins/data
        target: /var/jenkins_home
      # - ./jenkins/data:/var/jenkins_home
    restart: always

  nginx:
      image: jwilder/nginx-proxy:alpine
      container_name: nginx
      ports:
        - "80:80"
      depends_on: 
        - gitea
        - task
        - jenkins
      networks:
        - devops
      volumes:
        - type: bind
          source: /var/run/docker.sock
          target: /tmp/docker.sock
        # - /var/run/docker.sock:/tmp/docker.sock
      restart: always

secrets:
  db_root_password:
    file: ./mysql/my_secret.txt

networks:
  devops:
    name: devops-network
```

> 注: 通过 volumes bind 方式挂载的外部文件/目录，如果不存在的话，不会自动创建。

## 使用

- MySQL 的管理员密码，通过 `mysql/my_my_secret.txt` 设置，构建容器的时候会自动加载并设置。
- 不同 services 管理的域名，通过环境变量设置 `VIRTUAL_HOST=域名；VIRTUAL_PORT=端口`
- 创建镜像并执行 `docker-compose up -d`
- 删除容器及 `volumn` 数据  `docker-compose down -v`

## 后记

因为通过反向代理隐藏了暴露端口的细节，如果没有外部注册的域名的话，还需要通过 [Dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html) 进行内部域名解析。

`---EOF---`

> 原文链接：<https://gythialy.github.io/docker-devops-kits/#more>
