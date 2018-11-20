---
layout: post
title: ssh 通过跳板机直连跳板机内网服务器
categories: ssh
description: ssh 通过跳板机直连跳板机内网服务器
keywords: ssh
---

> 如果公司的服务器在外网，一般会设置一个跳板机，访问公司其他服务器都需要从跳板机做一个 ssh 跳转，外网的服务器基本都要通过证书登录的。
>
> 于是我们面临一个情况: 本机 ssh->跳板机->目标机器。（需要先登录跳板机，然后通过跳板机登录内网目标服务器，很明显效率很低，不利于自动化部署！）

# SSH 直连跳板机内网服务器，图：

![ ](/images/20181120-ssh-connect-through-springboard-01.png)

## 方式一：

```bash
# 从 linux 客户端的 ssh 跳转时，执行命令：

$ ssh username@跳板机ip地址

# 然后在跳板机上跳转到目标机器

$ ssh username@服务器ip地址
```

> 跳板机 ip 和目标机器 ip，username 账户下已经在相应的 .ssh/authorized_keys 加入了公钥，配置是没有问题了，但是我们会遇到一个 Pubkey Unauthorization 的错误，因跳板机没有username 的私钥。问题总是会有，解决方法也总是有，ssh 是有转发密钥的功能，从本机跳转到跳板机时可以把私钥转发过去。

### 正确做法是，在本机 linux 客户端执行命令

```bash
# -A表示转发密钥，所以跳转到跳板机，密钥也转发了过来
ssh -A username`@跳板机ip

# 接下来我们再在跳板机执行命令 ：
ssh username@目标机器ip
```

> 另外可以配置本机客户端的默认配置文件，修改为默认转发密钥:
>
> 修改 ssh_config（不是sshd_config，一般在/etc或者/etc/ssh下）：
>
> 把 #ForwardAgent no 改为 ForwardAgent Yes

## 方法二:(推荐)

`ssh username@目标机器ip -p 22 -o ProxyCommand='ssh -p 22 username@跳板机ip -W %h:%p'`

```bash
ssh root@192.168.1.2 -p 22 -o ProxyCommand='ssh -p 22 root@10.1.1.1 -W %h:%p'
```

### 也可以修改配置文件 ~/.ssh/config ， 若没有则创建：

```bash

Host tiaoban   #任意名字，随便使用

    HostName 192.168.1.1   #这个是跳板机的IP，支持域名

    Port 22      #跳板机端口

    User username_tiaoban       #跳板机用户

Host nginx      #同样，任意名字，随便起

    HostName 192.168.1.2  #真正登陆的服务器，不支持域名必须IP地址

    Port 22   #服务器的端口

    User username   #服务器的用户

    ProxyCommand ssh username_tiaoban@tiaoban -W %h:%p

Host 10.10.0.*      #可以用*通配符

    Port 22   #服务器的端口

    User username   #服务器的用户

    ProxyCommand ssh username_tiaoban@tiaoban -W %h:%p

```

> 配置好后， 直接 ssh nginx 就可以登录 192.168.1.2 这台跳板机后面的服务器。
>
> 也可以用 ssh username@10.10.0.xx 来登录10.10.0.27, 10.10.0.33, …. 等机器。

> 参考链接：<https://blog.csdn.net/DiamondXiao/article/details/52474104>