---
layout: post
title: Linux Firewalld 用法及案例
categories: linux,firewalld,iptables
description: Linux Firewalld 用法及案例
keywords: linux,firewalld,iptables
---

> CentOS 7 中防火墙 firewalld 是一个非常的强大的功能，在 CentOS 6.5 中在 iptables 防火墙中进行了升级了。

# 一、Firewalld 概述

- 动态防火墙管理工具
- 定义区域与接口安全等级
- 运行时和永久配置项分离
- 两层结构
  - 核心层 处理配置和后端，如 iptables、ip6tables、ebtables、ipset 和模块加载器
  - 顶层 D-Bus 更改和创建防火墙配置的主要方式。所有 firewalld 都使用该接口提供在线工具

## 1. 原理图

![ ](/images/20181129-linux-firewalld-01.png)

![ ](/images/20181129-linux-firewalld-02.png)

## 2. Firewalld与iptables对比

- firewalld 是 iptables 的前端控制器
- iptables 静态防火墙 任一策略变更需要 reload 所有策略，丢失现有链接
- firewalld 动态防火墙 任一策略变更不需要 reload 所有策略 将变更部分保存到 iptables,不丢失现有链接
- firewalld 提供一个 daemon 和 service 底层使用 iptables
- 基于内核的 Netfilter

## 3. 配置方式

- firewall-config 图形界面
- firewall-cmd 命令行工具
- 直接修改配置文件
  - /lib/firewalld 用于默认和备用配置
  - /etc/firewalld 用于用户创建和自定义配置文件 覆盖默认配置
  - /etc/firewalld/firewall.conf 全局配置

## 4. 运行时配置和永久配置

- firewall-cmd –zone=public –add-service=smtp 运行时配置，重启后失效
- firewall-cmd –permanent –zone=public –add-service=smtp 永久配置，不影响当前连接，重启后生效
- firewall-cmd –runtime-to-permanent 将运行时配置保存为永久配置

## 5. Zone（9种）

- 网络连接的可信等级,一对多，一个区域对应多个连接
  - drop.xml 拒绝所有的连接
  - block.xml 拒绝所有的连接
  - public.xml 只允许指定的连接 *默认区域
  - external.xml 只允许指定的连接
  - dmz.xml 只允许指定的连接
  - work.xml 只允许指定的连接
  - home.xml 只允许指定的连接
  - internal.xml 只允许指定的连接
  - trusted.xml 允许所有的连接
    - /lib/firewalld/zones 默认和备用区域配置
    - /etc/firewalld/zones 用户创建和自定义区域配置文件 覆盖默认配置

```xml
<?xml version="1.0" encoding="utf-8"?>
<zone>
  <short>Public</short>
  <description>For use in public areas. You do not trust the other computers on networks to not harm yo
ur computer. Only selected incoming connections are accepted.</description>
  <service name="ssh"/>
  <service name="dhcpv6-client"/>
</zone>
```

```xml
version="string" 版本
target="ACCEPT|%%REJECT%%|DROP" 默认REJECT 策略
short 名称
description 描述
interface 接口
    name="string"
source 源地址
    address="address[/mask]"
    mac="MAC"
    ipset="ipset"
service 服务
    name="string"
port 端口
    port="portid[-portid]"
    protocol="tcp|udp"
protocol 协议
    value="string"
icmp-block 
    name="string"
icmp-block-inversion
masquerade
forward-port
    port="portid[-portid]"
    protocol="tcp|udp"
    to-port="portid[-portid]"
    to-addr="address"
source-port
    port="portid[-portid]"
    protocol="tcp|udp"
rule
<rule [family="ipv4|ipv6"]>
  [ <source address="address[/mask]" [invert="True"]/> ]
  [ <destination address="address[/mask]" [invert="True"]/> ]
  [
    <service name="string"/> |
    <port port="portid[-portid]" protocol="tcp|udp"/> |
    <protocol value="protocol"/> |
    <icmp-block name="icmptype"/> |
    <masquerade/> |
    <forward-port port="portid[-portid]" protocol="tcp|udp" [to-port="portid[-portid]"] [to-addr="address"]/> |
    <source-port port="portid[-portid]" protocol="tcp|udp"/> |
  ]
  [ <log [prefix="prefixtext"] [level="emerg|alert|crit|err|warn|notice|info|debug"]/> [<limit value="rate/duration"/>] </log> ]
  [ <audit> [<limit value="rate/duration"/>] </audit> ]
  [
    <accept> [<limit value="rate/duration"/>] </accept> |
    <reject [type="rejecttype"]> [<limit value="rate/duration"/>] </reject> |
    <drop> [<limit value="rate/duration"/>] </drop> |
    <mark set="mark[/mask]"> [<limit value="rate/duration"/>] </mark>
  ]
</rule>

rich rule
<rule [family="ipv4|ipv6"]>
  <source address="address[/mask]" [invert="True"]/>
  [ <log [prefix="prefixtext"] [level="emerg|alert|crit|err|warn|notice|info|debug"]/> [<limit value="rate/duration"/>] </log> ]
  [ <audit> [<limit value="rate/duration"/>] </audit> ]
  <accept> [<limit value="rate/duration"/>] </accept> |
  <reject [type="rejecttype"]> [<limit value="rate/duration"/>] </reject> |
  <drop> [<limit value="rate/duration"/>] </drop>
</rule>
```

## 6. services

```xml
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>MySQL</short>
  <description>MySQL Database Server</description>
  <port protocol="tcp" port="3306"/>
</service>
```

```xml
version="string"
short
description
port
    port="string"
    protocol="string"
protocol
    value="string"
source-port
    port="string"
    protocol="string"
module
    name="string"
destination
    ipv4="address[/mask]"
    ipv6="address[/mask]"
```

## 7. ipset 配置

```xml
系统默认没有 ipset 配置文件，需要手动创建 ipset 配置文件

mkdir -p /etc/firewalld/ipsets/mytest.xml mytest就是 ipset 名称

根据官方手册提供的配置模板

<?xml version="1.0" encoding="utf-8"?>
<ipset type="hash:net">
  <short>white-list</short>
  <entry>192.168.1.1</entry>
  <entry>192.168.1.2</entry>
  <entry>192.168.1.3</entry>
</ipset>

entry 也就是需要加入的 IP 地址

firewall-cmd --get-ipsets 显示当前的 ipset
firewall-cmd --permanent --add-rich-rule 'rule family="ipv4" source ipset="mytest" port port=80 protocol=tcp accept' 将 ipset 应用到策略中
```

## 8. 服务管理

```bash
 #安装firewalld
yum -y install firewalld firewall-config

#开机启动
systemctl enable|disable firewalld

#启动、停止、重启firewalld
systemctl start|stop|restart firewalld
```

> 如果想使用 iptables 配置防火墙规则，要先安装 iptables 并禁用 firewalld

```bash
#安装iptables
yum -y install iptables-services

#开机启动
systemctl enable iptables

#启动、停止、重启iptables
systemctl start|stop|restart iptables
```

# 二、firewall-cmd常用命令

```bash

firewall-cmd --version 查看 firewalld 版本
firewall-cmd --help 查看 firewall-cmd 用法
man firewall-cmd

firewall-cmd --state #查看 firewalld 的状态
systemctl status firewalld #查看 firewalld 的状态,详细

firewall-cmd --reload 重新载入防火墙配置，当前连接不中断
firewall-cmd --complete-reload 重新载入防火墙配置，当前连接中断

firewall-cmd --get-services 列出所有预设服务
firewall-cmd --list-services 列出当前服务
firewall-cmd --permanent --zone=public --add-service=smtp 启用服务
firewall-cmd --permanent --zone=public --remove-service=smtp 禁用服务

firewall-cmd --zone=public --list-ports
firewall-cmd --permanent --zone=public --add-port=8080/tcp 启用端口
firewall-cmd --permanent --zone=public --remove-port=8080/tcp 禁用端口
firewall-cmd --zone="public" --add-forward-port=port=80:proto=tcp:toport=12345 同服务器端口转发 80端口转发到12345端口
firewall-cmd --zone=public --add-masquerade 不同服务器端口转发，要先开启 masquerade
firewall-cmd --zone="public" --add-forward-port=port=80:proto=tcp:toport=8080:toaddr=192.168.1.1 不同服务器端口转发，转发到192.168.1.1的 8080 端口

firewall-cmd --get-zones 查看所有可用区域
firewall-cmd --get-active-zones 查看当前活动的区域,并附带一个目前分配给它们的接口列表
firewall-cmd --list-all-zones 列出所有区域的所有配置
firewall-cmd --zone=work --list-all 列出指定域的所有配置
firewall-cmd --get-default-zone 查看默认区域
firewall-cmd --set-default-zone=public 设定默认区域

firewall-cmd --get-zone-of-interface=eno222
firewall-cmd [--zone=<zone>] --add-interface=<interface> 添加网络接口
firewall-cmd [--zone=<zone>] --change-interface=<interface> 修改网络接口
firewall-cmd [--zone=<zone>] --remove-interface=<interface> 删除网络接口
firewall-cmd [--zone=<zone>] --query-interface=<interface> 查询网络接口

firewall-cmd --permanent --zone=internal --add-source=192.168.122.0/24 设置网络地址到指定的区域
firewall-cmd --permanent --zone=internal --remove-source=192.168.122.0/24 删除指定区域中的网路地址

firewall-cmd --get-icmptypes
```

## 1. Rich Rules

```bash
firewall-cmd –list-rich-rules 列出所有规则
firewall-cmd [–zone=zone] –query-rich-rule=’rule’ 检查一项规则是否存在
firewall-cmd [–zone=zone] –remove-rich-rule=’rule’ 移除一项规则
firewall-cmd [–zone=zone] –add -rich-rule=’rule’ 新增一
```

## 2. 复杂规则配置案例

```bash
firewall-cmd --zone=public --add-rich-rule 'rule family="ipv4" source address=192.168.0.14 accept' 允许来自主机 192.168.0.14 的所有 IPv4 流量

firewall-cmd --zone=public --add-rich-rule 'rule family="ipv4" source address="192.168.1.10" port port=22 protocol=tcp reject' 拒绝来自主机 192.168.1.10 到 22 端口的 IPv4 的 TCP 流量

firewall-cmd --zone=public --add-rich-rule 'rule family=ipv4 source address=10.1.0.3 forward-port port=80 protocol=tcp to-port=6532' 许来自主机 10.1.0.3 到 80 端口的 IPv4 的 TCP 流量，并将流量转发到 6532 端口上

firewall-cmd --zone=public --add-rich-rule 'rule family=ipv4 forward-port port=80 protocol=tcp to-port=8080 to-addr=172.31.4.2' 将主机 172.31.4.2 上 80 端口的 IPv4 流量转发到 8080 端口（需要在区域上激活 masquerade）

firewall-cmd --add-rich-rule='rule family="ipv4" source address="192.168.122.0" accept' 允许192.168.122.0/24主机所有连接

firewall-cmd --add-rich-rule='rule service name=ftp limit value=2/m accept' 每分钟允许2个新连接访问ftp服务

firewall-cmd --add-rich-rule='rule service name=ftp log limit value="1/m" audit accept' 同意新的IPv4和IPv6连接FTP ,并使用审核每分钟登录一次

firewall-cmd --add-rich-rule='rule family="ipv4" source address="192.168.122.0/24" service name=ssh log prefix="ssh" level="notice" limit value="3/m" accept' 允许来自1192.168.122.0/24地址的新IPv4连接连接TFTP服务,并且每分钟记录一次

firewall-cmd --permanent --add-rich-rule='rule protocol value=icmp drop' 丢弃所有icmp包

firewall-cmd --add-rich-rule='rule family=ipv4 source address=192.168.122.0/24 reject' --timeout=10 当使用 source 和 destination 指定地址时,必须有family参数指定 ipv4 或 ipv6。如果指定超时,规则将在指定的秒数内被激活,并在之后被自动移除

firewall-cmd --add-rich-rule='rule family=ipv6 source address="2001:db8::/64" service name="dns" audit limit value="1/h" reject' --timeout=300 拒绝所有来自2001:db8::/64子网的主机访问 dns 服务,并且每小时只审核记录1次日志

firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=192.168.122.0/24 service name=ftp accept' 允许192.168.122.0/24网段中的主机访问 ftp 服务

firewall-cmd --add-rich-rule='rule family="ipv6" source address="1:2:3:4:6::" forward-portto-addr="1::2:3:4:7" to-port="4012" protocol="tcp" port="4011"' 转发来自 ipv6 地址1:2:3:4:6::TCP端口4011,到1:2:3:4:7的TCP端口4012
```

## 3. Direct Rules

```bash
firewall-cmd –direct –add-rule ipv4 filter IN_public_allow 0 -p tcp –dport 80 -j ACCEPT 添加规则

firewall-cmd –direct –remove-rule ipv4 filter IN_public_allow 10 -p tcp –dport 80 -j ACCEPT 删除规则

firewall-cmd –direct –get-all-rules 列出规则
```

> 原文链接： <https://blog.csdn.net/xiazichenxi/article/details/80169927>