---
layout: post
title: CentOS 7 下安装和使用 Fail2ban
categories: centos,fail2ban,iptables
description: 本文主要介绍一下 CentOS 7 下 Fail2ban 安装以及如何和 iptables 联动来阻止恶意扫描和密码猜测等恶意攻击行为。
keywords: centos,fail2ban,iptables
---

> 从 CentOS 7 开始，官方的标准防火墙设置软件从 iptables 变更为 firewalld。 为了使 Fail2ban 与 iptables 联动，需禁用自带的 firewalld 服务，同时安装 iptables 服务。
>
> 因此，在进行 Fail2ban 的安装与使用前需根据博客[ CentOS 7 安装和配置 iptables 防火墙](https://bolerolily.github.io/2018/09/06/CentOS7安装和配置iptables防火墙) 进行环境配置。

# 关于 Fail2ban

- Fail2ban 可以监视你的系统日志，然后匹配日志的错误信息（正则式匹配）执行相应的屏蔽动作（一般情况下是调用防火墙屏蔽）
  如:当有人在试探你的 HTTP、SSH、SMTP、FTP 密码，只要达到你预设的次数，fail2ban 就会调用防火墙屏蔽这个 IP，而且可以发送 e-mail 通知系统管理员，是一款很实用、很强大的软件！

- Fail2ban 由 python 语言开发，基于 logwatch、gamin、iptables、tcp-wrapper、shorewall 等。如果想要发送邮件通知道，那还需要安装 postfix 或 sendmail。

- 在外网环境下，有很多的恶意扫描和密码猜测等恶意攻击行为，使用 Fail2ban 配合 iptables，实现动态防火墙是一个很好的解决方案。

# 安装 Fail2ban

本文中通过稳定版 Fail2ban-0.8.14 做演示。

```bash
#首先需要到 Fail2ban 官网下载程序源码包
wget https://codeload.github.com/fail2ban/fail2ban/tar.gz/0.8.14

#解压安装
tar -xf 0.8.14.tar.gz
cd fail2ban-0.8.14
python setup.py install

#手动生成一下程序的启动脚本
cp files/redhat-initd /etc/init.d/fail2ban

#加入开机启动项
chkconfig --add fail2ban
#检查
chkconfig --list fail2ban

```

![ ](/images/20191009-fail2ban-for-centos7-01.jpg)

安装完成后程序文件都是保存在 /etc/fail2ban 目录下，目录结构如下图所示。

![ ](/images/20191009-fail2ban-for-centos7-02.jpg)

其中 jail.conf 为主配置文件，相关的正则匹配规则位于 filter.d 目录，其它目录/文件一般很少用到，如果需要详细了解可自行搜索。

# 配置 Fail2ban

## 联动 iptables

新建 jail.local 来覆盖 Fail2ban 在 jail.conf 的默认配置。

```bash
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
vim /etc/fail2ban/jail.local
```

对 jail.conf 做以下两种修改。

### 1. 防止 SSH 密码爆破

[ssh-iptables]模块的配置修改如下。其中，port 应按照实际情况填写。

```bash
[ssh-iptables]

 enabled  = true
 filter   = sshd
 action   = iptables[name=SSH, port=22, protocol=tcp]
 logpath  = /var/log/secure
 maxretry = 3
 findtime  = 300
 ```

### 2. 阻止恶意扫描

新增 [nginx-dir-scan] 模块，配置信息如下。此处，port 和 logpath 应按照实际情况填写。

 ```bash
[nginx-dir-scan]

 enabled = true
 filter = nginx-dir-scan
 action   = iptables[name=nginx-dir-scan, port=443, protocol=tcp]
 logpath = /path/to/nginx/access.log
 maxretry = 1
 bantime = 172800
 findtime  = 300
```

然后在 filter.d 目录下新建 nginx-dir-scan.conf。

```bash
cp /etc/fail2ban/filter.d/nginx-http-auth.conf /etc/fail2ban/filter.d/nginx-dir-scan.conf
vim /etc/fail2ban/filter.d/nginx-dir-scan.conf
```

对 nginx-dir-scan.conf 进行修改，具体配置信息如下。

```bash
[Definition]

 failregex = <HOST> -.*- .*Mozilla/4.0* .* .*$
 ignoreregex =
```

此处的正则匹配规则是根据 nginx 的访问日志进行撰写，不同的恶意扫描有不同的日志特征。

本文采用此规则是因为在特殊的应用场景下有绝大的把握可以肯定 Mozilla/4.0 是一些老旧的数据采集软件使用的UA，所以就针对其做了屏蔽。不可否认 Mozilla/4.0 这样的客户端虽然是少数，但仍旧存在。因此，此规则并不适用于任何情况。

使用如下命令，可以测试正则规则的有效性。

`fail2ban-regex /path/to/nginx/access.log /etc/fail2ban/filter.d/nginx-dir-scan.conf`

Fail2ban 已经内置很多匹配规则，位于 filter.d 目录下，包含了常见的 SSH/FTP/Nginx/Apache 等日志匹配，如果都还无法满足需求，也可以自行新建规则来匹配异常IP。

总之，使用 Fail2ban+iptables 来阻止恶意 IP 是行之有效的办法，可极大提高服务器安全。

## 变更 iptables 封禁策略

Fail2ban 的默认 iptables 封禁策略为 REJECT --reject-with icmp-port-unreachable，需要变更 iptables 封禁策略为 DROP。

在 /etc/fail2ban/action.d/ 目录下新建文件 iptables-blocktype.local。

```bash
cd /etc/fail2ban/action.d/
cp iptables-blocktype.conf iptables-blocktype.local
vim iptables-blocktype.local
```

修改内容如下：

```bash
[INCLUDES]

after = iptables-blocktype.local

[Init]

blocktype = DROP
```

最后，别忘记重启 fail2ban 使其生效。

`systemctl restart fail2ban`

# Fail2ban 常用命令

```bash
#启动 Fail2ban
systemctl start fail2ban

#停止 Fail2ban
systemctl stop fail2ban

#开机启动 Fail2ban
systemctl enable fail2ban

#查看被 ban IP，其中 ssh-iptables 为名称，比如上面的 [ssh-iptables] 和 [nginx-dir-scan]
fail2ban-client status ssh-iptables

#添加白名单
fail2ban-client set ssh-iptables addignoreip IP地址

#删除白名单
fail2ban-client set ssh-iptables delignoreip IP地址

#查看被禁止的 IP 地址
iptables -L -n
```

> 原文连接：<https://www.jianshu.com/p/4fdec5794d08>