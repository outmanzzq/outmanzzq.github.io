---
layout: post
title: Docker 实现 Nginx proxy 多域名的自动反向代理、免费 SSL 证书
categories: docker,nginx,proxy
description: 在个人的小项目或者测试环境中，配置反向代理显得十分繁琐，而借助 Nginx-proxy 的镜像，即使是小白，也能快速实现域名转发。
keywords: docker,nginx,proxy
---

> 在个人的小项目或者测试环境中，配置反向代理显得十分繁琐，而借助 Nginx-proxy 的镜像，即使是小白，也能快速实现域名转发。

# 1.域名、IP自动转发

在开始之前，首先黑进了自家的路由器，将某个域名（甚至不存在），如 dotnet1.nginx-test.com 和 dotnet2.nginx-test.com 指向了局域网内 IP 为 "192.168.9.10" 的机器上（hosts、iptable等方式）。
接着，假设你已经安装了 Docker的基础上，只需再安装 docker-compose。如果你对这一切一无了解的话，可以使用daocloud提供的的 一键脚本。
回到本文讨论的重点，在不写任何 Nginx 配置的前提下，让相关的域名指向对应的应用。编写如下的 docker-compose.yml:

```YAML
version: '2'
services:
  nginx-proxy:
    image: jwilder/nginx-proxy
    container_name: nginx-proxy
    ports:
      - "80:80"
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro

  dotnet1:
    image: daocloud.io/koukouge/zhs:master
    container_name: dotnet1
    environment:
      - VIRTUAL_PORT=80 #监听的端口
      - VIRTUAL_HOST=dotnet1.nginx-test.com,192.168.9.10  #监听的地址
```

只需要一句 docker-compose up -d 就能启动对应的应用，实现自动转发。

# 2.零停机重载域名、IP

在上一节中，我们已经是在后台启动了 nginx-proxy 和 dotnet1 的应用，如果我们在新增或者修改原有的域名呢？假设在原有的 docker-compose.yml 增加一个 2048 的镜像：

```YAML
simple:
    image: alexwhen/docker-2048
    container_name: simple
    environment:
      - VIRTUAL_PORT=80
      - VIRTUAL_HOST=dotnet2.nginx-test.com
```

这种情况下，重启整个 docker-compose 显然不是最佳的方式。为了不影响已经运行中的应用，只需对新增或者需要修改的应用执行如下命令:

```sudo docker-compose up --build --no-deps -d simple # simple 为应用的名称```

# 3. Let's Encrypt 免费证书

随着网络安全意识的提高，Https 逐渐成为互联网的标配。特别是在国内的网络环境中，网络劫持现象愈演愈烈。即使是个人的小博客网站，博主并不接入广告的情况下，仍然可以无意中发现博客中居然有 "澳门在线赌场" 的广告，这时候使用 SSL 证书就显得十分必要了。

Let's Encrypt 是一家致力于推广 Https 技术的公益组织，其免费证书得到了几乎所有浏览器的支持，是目前最为流行、也是最大的免费证书提供者。
同样的，在我们之前基础上，我们同样可以实现对多个域名证书的傻瓜化配置。
在原有的基础下，我们将第一节中的 docker-compose.yml 修改为:

```YAML
version: '2'
services:
  nginx:
    restart: always
    image: nginx
    container_name: nginx
    ports:
    - 80:80
    - 443:443
    volumes:
    - /etc/nginx/conf.d
    - /etc/nginx/vhost.d
    - /usr/share/nginx/html
    - /etc/nginx/certs:/etc/nginx/certs:ro

  dotnet1:
    image: daocloud.io/koukouge/zhs:master
    container_name: dotnet1
    environment:
      - VIRTUAL_PORT=80 #监听的端口
      - VIRTUAL_HOST=dotnet1.nginx-test.com  #监听的地址
      - LETSENCRYPT_HOST=dotnet1.nginx-test.com #证书的域名
      - LETSENCRYPT_EMAIL=someone@simple.com #证书所有者的邮箱，快过期时会提醒

  nginx-gen:
    restart: always
    image: jwilder/docker-gen
    container_name: nginx-gen
    volumes:
    - /var/run/docker.sock:/tmp/docker.sock:ro
    - /etc/nginx/nginx.tmpl:/etc/docker-gen/templates/nginx.tmpl:ro
    volumes_from:
    - nginx
    entrypoint: /usr/local/bin/docker-gen -notify-sighup nginx -watch -wait 5s:30s
      /etc/docker-gen/templates/nginx.tmpl /etc/nginx/conf.d/default.conf

  letsencrypt-nginx-proxy-companion:
    restart: always
    image: jrcs/letsencrypt-nginx-proxy-companion
    container_name: letsencrypt-nginx-proxy-companion
    volumes_from:
    - nginx
    volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro
    - /etc/nginx/certs:/etc/nginx/certs:rw
    environment:
    - NGINX_DOCKER_GEN_CONTAINER=nginx-gen
```

先别急着启动，如果已经启动了就会发现 nginx-gen应用 缺失了nginx.tmpl 文件。所以我们需要将其下载并放置在相应的位置:

```curl https://raw.githubusercontent.com/jwilder/nginx-proxy/master/nginx.tmpl > /etc/nginx/nginx.tmpl```

最后，只要一声令下，就可以发现网站已经成功多了一个绿色的小锁。

> 原文链接：<https://www.cnblogs.com/chenug/p/6916639.html>
