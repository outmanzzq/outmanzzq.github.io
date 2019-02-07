---
layout: post
title: linux getent 命令简介
categories: linux，getent
description: linux getent 命令简介
keywords: linux，getent
---

> getent 从某个解析库 LIB 中获的条目。该命令可以检测 nsswitch 配置是否正确

# 一、nsswitch之基本原理

### Authentication（认证）
用户的用户名和密码是否能通过检验。

### Authorizantion（授权）
用户是否被允许访问服务或资源。


### 应用程序的名称解析流程：

应用程序  --> nsswitch（配置文件（查询顺序）） --> 对应库文件 --> 解析库 --> 完成解析

### NSS （Name Switch Serviers 名称解析服务）

### nsswitch（network services switch 网络服务转换）
中间层，本质上上是一些库文件。提供了为应用程序向不同的解析库进行名称解析的手段和顺序。

> 注意：具体的解析过程是由对应的库文件完成。

###  配置文件
/etc/nsswitch.conf

#### 格式如下：

`应用程序： 查询顺序1 [VALUE=ACTION] 查询顺序2  ...`

#### 查询结果返回值 VALUE：

- SUCCESS  服务正常，找到名称
- NOTFOUND 服务正常，没有找到名称
- UNAVAIL 服务不可用
- TRYAGAIN  服务有临时性故障

#### 查询结果动作 ACTION：

- return 返回
- continue 继续

#### 查询结果的返回动作：

- 默认情况下，一旦出现第一个 SUCCESS，就直接 return，否则就 continue。

- 手工指定返回动作：`VALUE=ACTION`

### 库文件：

- /usr/lib/libnss*.so 不同查找方法对应的库文件
- /etc/protocols 网络和ip协议对应的端口表
- /etc/services  服务和协议对应的端口表


# 二、getent命令

getent 用来察看系统的数据库中的相关记录

### 用法： 

`getent [选项...] 数据库 [键 ...]`

```bash
Get entries from administrative database.

-s, --service=CONFIG       要使用的服务配置
-?, --help                 给出该系统求助列表
    --usage                给出简要的用法信息
-V, --version              打印程序版本号
```

### 支持的数据库：

`ahosts ahostsv4 ahostsv6 aliases ethers group gshadow hosts netgroup networks passwd protocols rpc services shadow`

### 举例

```bash
#查找 hostname 对应的 IP
ubuntu$ getent hosts ubuntu
127.0.1.1       ubuntu
192.168.0.2     ubuntu

#执行反响 DNS 查询（即根据域名查找对应 IP）
$ getent hosts myhost.mydomain.com
15.77.3.40       myhost.mydomain.com myhost

#根据用户名查找 UID
ubuntu$ getent passwd greys
greys:x:1000:1000:Gleb Reys,,,:/home/greys:/bin/bash

#根据 UID 查找用户名
ubuntu$ getent passwd 1000
greys:x:1000:1000:Gleb Reys,,,:/home/greys:/bin/bash

#获取当前登陆用户的信息
$ getent passwd `whoami`
root:x:0:0:root:/root:/bin/bash

#查找那个服务在使用特定端口
$ getent services 22
ssh                   22/tcp
$ getent services 21
ftp                   21/tcp
$ getent services 25
smtp                  25/tcp mail
```

> 参考链接：<https://www.cnblogs.com/kelamoyujuzhen/p/9815774.html>