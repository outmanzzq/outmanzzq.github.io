---
layout: post
title: Nginx 代理 WebSocket
categories: nginx,websocket
description: Nginx 代理 WebSocket 的要点是设置Upgrade和Connection响应头。
keywords: nginx,websocket
---

> NGINX 通过允许一个在客户端和后端服务器之间建立的隧道来支持 WebSocket。为了 NGINX 发送来至于客户端 Upgrade 请求到后端服务器，Upgrade 和 Connection 头部必须被设置明确。

# 一、配置 Nginx 根据 Upgrade（即$http_upgrade）来设置 Connection：

- 如果请求头中有 Upgrade，就直接设置到响应头中，并把 Connection 设置为 upgrade。如 WebSocket 请求头会带上Upgrade: websocket，则响应头有

```nginx
Upgrade: websocket
Connection: upgrade
```

- 否则把 Connection 设置为 close。如普通 HTTP 请求。

## 最终 Nginx 配置如下：

```nginx

map $http_upgrade $connection_upgrade {
  default upgrade;
  '' close;
}

server {
  listen 8000;
  location / {
    proxy_pass http://localhost:4000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
  }
}
```

# 二、配置 websocket 负载均衡

在实际的生产环境中，要求多个 WebSocket 服务器必须具有高性能和高可用，那么 WebSocket 协议就需要一个负载均衡层，NGINX从1.3开始支持 WebSocket，其可以作为一个反向代理和为 WebSocket 程序做负载均衡。

```nginx
upstream wsbackend {
  server 127.0.0.1:8080;
  server 127.0.0.1:8081;
}

server {
  listen  80;
  server_name ws.domain.com;
  location / {
   proxy_pass http://wsbackend;
   proxy_http_version 1.1;
   proxy_set_header Upgrade $http_upgrade;
   proxy_set_header Connection "upgrade";
  }
}
```

## 反向代理服务器在支持 WebSocket 时面临的挑战

- WebSocket 是端对端的，所以当一个代理服务器从客户端拦截一个 Upgrade 请求，它需要去发送它自己的 Upgrade 请求到后端服务器，也包括合适的头。

- 因为 WebSocket 是一个长连接，不像 HTTP 那样是典型的短连接，所以反向代理服务器需要允许连接保持着打开，而不是在它们看起来空闲时就将它们关闭。

# 三、nginx 配置 http 访问自动跳转到 https

按照如下格式修改 nginx.conf 配置文件，80 端口会自动转给 443 端口，这样就强制使用 SSL 证书加密了。访问 http 的时候会自动跳转到 https 上面。

```nginx
server {
listen 80;
server_name www.域名.com;
rewrite ^(.*) https://$server_name$1 permanent;
}

server {
listen 443;
server_name www.域名.com;
root /home/www;
ssl on;
ssl_certificate /etc/nginx/certs/server.crt;
ssl_certificate_key /etc/nginx/certs/server.key;
}
```

> 参考链接：
> - <https://www.jianshu.com/p/07f37dd4657f>
> - <https://www.cnblogs.com/shansongxian/p/7120359.html>