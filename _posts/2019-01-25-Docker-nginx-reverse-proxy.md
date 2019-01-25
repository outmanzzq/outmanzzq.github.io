---
layout: post
title: Docker nginx 反向代理设置
categories: Docker,nginx,Proxy
description: Docker nginx 反向代理设置
keywords: Docker,nginx,Proxy
---

> 最近在公司搭建了一个基于 [Gogs](https://gogs.io/) 的代码管理系统，以及基于 [Kanboard](https://kanboard.org/) 的任务管理系统等几个内部系统。由于部署在同一台机器上，基于不同的端口区分不同的服务。比如：
>
> - Git 服务`http://10.10.1.110:10080`
> - 任务管理系统`http://10.10.1.110:8888`
> - 其他
>
> 为了更好的使用，通过内部域名区分，比如 ：
>
> - Git 服务`http://gogs.vking.io`
> - 任务管理系统 `http://task.vking.io`
> - 其他
>
> 注：vking.io 是内部域名，可通过 [dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html) 配置。

## 方案一

现有服务都是通过  [Docker](https://www.docker.com/)  部署，[nginx](https://www.nginx.com/) 同样通过 Docker 部署，使用官方提供的镜像即可。

- 新建 nginx 配置文件， `nginx.conf`，存放路径为 `/srv/docker/nginx/nginx.conf`

```nginx
# user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
# daemon off;
```

- 新建反向代理设置文件 `reverse-proxy.conf`，存放路径为 `/srv/docker/nginx/conf.d/reverse-proxy.conf`

```nginx
# If we receive X-Forwarded-Proto, pass it through; otherwise, pass along the
# scheme used to connect to this server
map $http_x_forwarded_proto $proxy_x_forwarded_proto {
  default $http_x_forwarded_proto;
  ''      $scheme;
}
# If we receive X-Forwarded-Port, pass it through; otherwise, pass along the
# server port the client connected to
map $http_x_forwarded_port $proxy_x_forwarded_port {
  default $http_x_forwarded_port;
  ''      $server_port;
}
# If we receive Upgrade, set Connection to "upgrade"; otherwise, delete any
# Connection header that may have been passed to this server
map $http_upgrade $proxy_connection {
  default upgrade;
  '' close;
}
# Apply fix for very long server names
server_names_hash_bucket_size 128;
# Default dhparam
ssl_dhparam /etc/nginx/dhparam/dhparam.pem;
# Set appropriate X-Forwarded-Ssl header
map $scheme $proxy_x_forwarded_ssl {
  default off;
  https on;
}
gzip_types text/plain text/css application/javascript application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;
log_format vhost '$host $remote_addr - $remote_user [$time_local] '
                 '"$request" $status $body_bytes_sent '
                 '"$http_referer" "$http_user_agent"';
access_log off;

# HTTP 1.1 support
proxy_http_version 1.1;
proxy_buffering off;
proxy_set_header Host $http_host;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $proxy_connection;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $proxy_x_forwarded_proto;
proxy_set_header X-Forwarded-Ssl $proxy_x_forwarded_ssl;
proxy_set_header X-Forwarded-Port $proxy_x_forwarded_port;
# Mitigate httpoxy attack (see README for details)
proxy_set_header Proxy "";
server {
    server_name _; # This is just an invalid value which will never trigger on a real hostname.
    listen 80;
    access_log /var/log/nginx/access.log vhost;
    return 503;
}

# gogs.vking.io
upstream gogs.vking.io {
    # gogs
    server gogs:3000;
}

server
{
    server_name gogs.vking.io;
    listen 80;

    location / {
        proxy_pass http://gogs.vking.io;
    }
    access_log /var/log/nginx/access.log vhost;
}

# task.vking.io
upstream task.vking.io {
    # kanboard
    server kanboard:80;
}

server
{
    server_name task.vking.io;
    listen 80;

    location / {
        proxy_pass http://task.vking.io;
    }
    access_log /var/log/nginx/access.log vhost;
}
```

- gogs 启动命令

```shell
docker container run -d --name gogs \
    --restart always \
    -p 10022:22  \
    -p 10080:3000 \
    --network gogs-net \
    -v /srv/docker/gogs:/data \
    gogs/gogs:latest
```

> 注： `upstream gogs.vking.io` 中的 `server` 中的 `gogs:3000` 分别指容器名称和原始expose 的端口。

- 启动容器

```shell
docker container run -d --name nginx \
    --restart always \
    -p 80:80 \
    --network gogs-net \
    -v /srv/docker/nginx/nginx.conf:/etc/nginx/nginx.conf \
    -v /srv/docker/nginx/conf.d:/etc/nginx/conf.d \
    nginx:alpines
```

> 注：`--network gogs-net` 指定三个容器在同一网络，如果用默认的 `bridge`的话，不需要设置

## 方案二

因为这些服务都是部署在一台机器上的，可以通过 Docker 的自动服务发现部署，原理[见此](http://jasonwilder.com/blog/2014/03/25/automated-nginx-reverse-proxy-for-docker/)。

- gogs

```shell
docker container run -d --name gogs \
  --restart always \
  -p 10022:22  -p 10080:3000 \
  --network gogs-net \
  -e VIRTUAL_HOST=gogs.vking.io \
  -e VIRTUAL_PORT=3000 \
  -v /opt/docker/gogs:/data \
  gogs/gogs:latest
```

- Kanboard

  ```shell
  docker container run -d --name kanboard \
      --restart always \
      -p 8888:80 \
      --network gogs-net \
      -e VIRTUAL_HOST=task.vking.io \
      -e VIRTUAL_PORT=80 \
      -v /srv/docker/kanboard/data:/var/www/app/data \
      -v /srv/docker/kanboard/plugins:/var/www/app/plugins \
      kanboard/kanboard:latest
  ```

- nginx

  ```shell
  docker container run -d --name nginx \
      --restart always \
      -p 80:80 \
      --network gogs-net \
      -v /var/run/docker.sock:/tmp/docker.sock:ro \
      jwilder/nginx-proxy:alpine
  ```

> 注：关键是容器通过 `-e VIRTUAL_HOST` 指定 url，通过 `-e VIRTUAL_PORT=80` 指定端口，同样端口也必须是原始镜像 expose 的端口。

## 延伸

目前服务都是通过 shell 启动，可改成通过 [Docker Compose](https://docs.docker.com/compose/) 统一编排任务，把 dnsmasq+nginx+gogs+... 等统一管理。如果是部署在公网上的话，还可以把 SSL 证书到期自动刷新等一起编排。

`---EOF---`

> 原文链接：<https://gythialy.github.io/Docker-nginx-reverse-proxy/>