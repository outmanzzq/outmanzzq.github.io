---
layout: post
title: 如何在 CentOS / RHEL 8上安装 Fail2Ban 保护 SSH
categories: fail2ban,centos
description: 本文介绍了如何安装和配置 fail2ban 来保护 SSH 并提高 SSH 服务器的安全性，以防止在 CentOS / RHEL 8 上遭受暴力攻击
keywords: fail2ban,centos
---

> 在本文中，我们将解释如何安装和配置 fail2ban 来保护 SSH 并提高 SSH 服务器的安全性，以防止对 CentOS / RHEL 8 的暴力攻击。

Fail2ban 是一个免费的开放源代码且广泛使用的入侵防御工具，它可以扫描日志文件中的 IP 地址，这些 IP 地址显示出恶意迹象，例如密码失败过多等等，并禁止它们（更新防火墙规则以拒绝IP地址）。 。 默认情况下，它附带用于各种服务的过滤器，包括 sshd 。

另请参阅 ： [使用CentOS / RHEL 8进行初始服务器设置](https://www.howtoing.com/initial-server-setup-with-centos-rhel-8/)

## 在 CentOS / RHEL 8 上安装 Fail2ban

fail2ban 软件包不在官方存储库中，但在 EPEL 存储库中可用。 登录系统后，访问命令行界面，然后如图所示在系统上启用 EPEL 存储库。

```bash
$ dnf install -y epel-release

# 通过运行以下命令来安装 Fail2ban 软件包
$ dnf install -y fail2ban
```

## 配置 Fail2ban 保护 SSH

fail2ban 配置文件位于 /etc/fail2ban/ 目录中，过滤器存储在 /etc/fail2ban/filter.d/ 目录中（sshd 的过滤器文件为 /etc/fail2ban/filter.d/sshd.conf ）。 。

fail2ban 服务器的全局配置文件是 /etc/fail2ban/jail.conf ，但是，不建议直接修改此文件，因为将来在升级程序包时可能会覆盖或改进该文件。

或者，建议在 /etc/fail2ban/jail.d/ 目录下的 jail.local 文件或单独的 .conf 文件中创建和添加配置。 请注意，在 jail.local 中设置的配置参数将覆盖 jail.conf 中定义的任何参数。

对于本文，我们将在 /etc/fail2ban/ 目录中创建一个名为 jail.local 的单独文件，如下所示。

```bash
$ echo '
[DEFAULT]
ignoreip = 192.168.56.2/24
bantime  = 21600
findtime  = 300
maxretry = 3
banaction = iptables-multiport
backend = systemd

[sshd]
enabled = true
' > /etc/fail2ban/jail.local
```

### Fail2ban 配置

让我们简要解释一下上述配置中的选项：

- ignoreip ：指定不禁止的 IP 地址或主机名列表。
- bantime ：指定禁止主机的秒数（即有效禁止持续时间）。
- maxretry ：指定禁止主机之前的故障数。
- findtime ：fail2ban 将禁止主机，如果主机在最后一个“ findtime ”秒内生成了“ maxretry ”。
- banaction：禁止行动。
- backend ：指定用于修改日志文件的后端。

因此，上述配置意味着，如果IP在最近 5 分钟内发生3次故障，则将其禁止 6 个小时，并忽略 IP 地址 192.168.56.2 。

接下来，立即启动并启用 fail2ban 服务，并使用以下 systemctl 命令检查它是否已启动并正在运行。

```bash
#启动 Fail2ban s服务
$ systemctl start fail2ban
$ systemctl enable fail2ban
$ systemctl status fail2ban
```

## 使用 fail2ban-client 监视失败和禁止的 IP 地址

将 fail2ban 配置为保护 sshd 后 ，可以使用 fail2ban-client 监视失败和被禁止的IP地址。 要查看 fail2ban 服务器的当前状态，请运行以下命令。

```$ fail2ban-client status```

![ ](/images/20191030-fail2ban-for-centos8-01.png)

检查 Fail2ban 监狱状态,要监视 sshd 监狱，请运行。

```$ fail2ban-client status sshd```

![ ](/images/20191030-fail2ban-for-centos8-02.png)

使用 Fail2ban 监视 SSH s失败的登录

要在 fail2ban（在所有监狱和数据库）中取消禁止 IP 地址，请运行以下命令。

```$ fail2ban-client unban 192.168.56.1```

有关 fail2ban 的更多信息，请阅读以下手册页。

```bash
# man jail.conf
# man fail2ban-client
```

> 原文链接：<https://www.howtoing.com/install-fail2ban-to-protect-ssh-on-centos-rhel>
>
> [fail2ban 文档](http://www.fail2ban.org/wiki/index.php/MANUAL_0_8#Installing_from_sources_on_a_GNU.2FLinux_system)