---
layout: post
title: 使用 Docker Compose 一键搭建本地代码质量检测平台 SonarQube
categories: docker-compose
description: 使用 Docker Compose 一键搭建本地代码质量检测平台 SonarQube
keywords: docker, sonarqube, 代码质量
---

> 目前代码分析工具首推的也是 sonarqube，支持各种语言的程序检测，使用简单方便，非常适合微服务的代码评审，强烈推荐！

# 一、环境说明

- OS: MacOS 10.12.6
- Docker: 17.09.1-ce
- Docker Compose: 1.17.1

# 二、部署过程

## 1. docker-compose-sonarqube.yml 内容

```shell
# 创建工作目录(非必须)
mkdir -p ~/Documet/docker
cd  ~/Documet/docker

➜  docker cat docker-compose-sonarqube.yml
version: '2'
services:
  db:
    image: postgres
    container_name: db
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar
    expose:
      - "5432"

  sq:
    image: sonarqube
    container_name: sq
    environment:
      SONARQUBE_JDBC_URL: jdbc:postgresql://db:5432/sonar
    ports:
      - "9000:9000"
    links:
      - db
```

## 2. 进行语法检查

```shell
➜  docker docker-compose -f docker-compose-sonarqube.yml config
services:
  db:
    container_name: db
    environment:
      POSTGRES_PASSWORD: sonar
      POSTGRES_USER: sonar
    expose:
    - '5432'
    image: postgres
  sq:
    container_name: sq
    environment:
      SONARQUBE_JDBC_URL: jdbc:postgresql://db:5432/sonar
    image: sonarqube
    links:
    - db
    ports:
    - 9000:9000/tcp
version: '2.0'
```

## 3. 启动并显示实时日志

```shell
➜  docker docker-compose -f docker-compose-sonarqube.yml up
Creating network "docker_default" with the default driver
Creating db ...
Creating db ... done
Creating sq ...
Creating sq ... done
Attaching to db, sq
...
```

## 4. 启动服务，并转入后台运行

```shell
➜  docker docker-compose -f docker-compose-sonarqube.yml  up -d
```

## 5. 前台web访问（默认用户密码：admin/admin）

<http://localhost:9000>

> 参考链接：<https://www.jianshu.com/p/a1450aeb3379>