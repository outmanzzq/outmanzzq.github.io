---
layout: post
title: zerotier 充当网关实现内网互联,访问其它节点内网
categories: vpn,nat
description: zerotier 是一个软交换机，使用 zerotier 可以让多台内网机器组成一个局域网。
keywords: vpn,nat,zerotier
---
> 本节知识点:建议学习和理解并掌握 iptables/route 运行原理和机制.

- 官网: <https://www.zerotier.com>
- 文档: <https://docs.zerotier.com>

## 一、网络三层 NAT 配置方法（LINUX主机）[推荐]

假设 Zerotier 虚拟局域网的网段是 192.168.88.0/24

- 局域网 A: 192.168.1.0/24
- 局域网 B: 192.168.2.0/24

> (如果需要互联)在局域网 A 和 B 中需要各有一台主机安装 Zerotier 并作为两个内网互联的网关

分别是: (括号里面为虚拟局域网的IP地址)

- 192.168.1.10 <192.168.88.10>
- 192.168.2.10 <192.168.88.20>

### 1.在 Zerotier 网站的 Networks 里面的 Managed Routes下配置路由表,增加如下内容

```sh
#如果单向连接,仅需填写下方一个即可.

192.168.1.0/24 via 192.168.88.10 
192.168.2.0/24 via 192.168.88.20
```

### 2.开启内核转发

```sh
#echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
#sysctl -p
```

### 3.防火墙设置

```sh
#其中的 ztyqbub6jp 在不同的机器中不一样，你可以在路由器ssh环境中用 zerotier-cli listnetworks 或者 ifconfig 查询zt开头的网卡名
iptables -I FORWARD -i ztyqbub6jp -j ACCEPT
iptables -I FORWARD -o ztyqbub6jp -j ACCEPT
iptables -t nat -I POSTROUTING -o ztyqbub6jp -j MASQUERADE

#保存配置到文件,否则重启规则会丢失.
iptables-save 
```

## 二、网络二层桥接方式（linux主机）[未测试,谨慎尝试!]

### 1.设置桥接

在官网的 Networks 里面，在 Members 选择两个节点前面的小扳手，然后勾选 Allow Ethernet Bridging

### 2.配置网桥模式

请注意,如果你的设备仅有一个物理网卡,下方配置可能会断网噢.

配置桥接前,请先清空物理网卡的 ip,否则会影响路由出口选择.

```sh
# 创建桥接网卡

##添加桥接网卡br0
brctl addbr br0

 ##查看
brctl show
ifconfig br0 172.25.47.104/24 ##给br0配置ip172.25.7.11
brctl addif br0 eth0 #添加真实物理网卡到桥接br0上
brctl addif br0 ztxxxx #添加zerotier网卡到桥接br0上
ping 172.25.7.254 ##测试，是否可以正常使用。

#删除桥接网卡
brctl delif br0 eth0 #从桥接中移出物理网卡eth0
brctl delif br0 ztxxx #从桥接网卡中移除zt网卡
ifconfig br0 down ## 关闭桥接网卡br0
brctl delbr br0 ##删除桥接网卡br0
brctl show ##查看桥接是否存在

#以上命令创建的网卡,会在重启丢失,下面是修改配置文件来实现持久化.

vim ifcfg-enp4s0f2 #编写物理网卡网络配置文件

 DEVICE=enp4s0f2
 ONBOOT=yes
 BOOTPROTO=none
 BRIDGE=br0

vim ifcfg-br0 #编写桥接配置文件

   DEVICE=br0
  ONBOOT=yes
  BOOTPROTO=none
  IPADDR=172.25.7.254
  PREFIX=24
  TYPE=Bridge

systemctl restart network #|重新启动网络服务
```

> 原文链接: <https://www.cnblogs.com/jonnyan/p/14175136.html>
