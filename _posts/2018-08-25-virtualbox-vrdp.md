---
layout: post
title: VirtualBox的远程桌面功能
categories: VirtualBox,RDP
description: VirtualBox的远程桌面功能
keywords: VirtualBox,RDP
---

> 启动VirtualBox Remote Display Protocol (VRDP)远程桌面功能后，可以使用远程桌面连
接到虚拟机，VirtualBox 4.0之后需要安装扩展。

**特别说明:**

1. 这个远程桌面和系统的远程桌面无关，不需要虚拟机系统设置密码都可以使用。

2. VirtualBox的远程桌面允许多连接，Guest OS是Windows桌面版系统时，远程登录后也不会注销系统当前的用户。

# 虚拟机使用Host-Only网络时的环境设置

-|系统|局域网IP|Host-Only网络IP|系统远程桌面|VRDP远程桌面
-|-|-|-|-|-
Host OS|Win7 x64|192.168.1.10|192.168.56.1|端口3389|-
Guest OS|XP |- |192.168.56.102|端口3389|端口5000

# 虚拟机使用Bridged网络时的环境设置

-|系统|局域网IP|系统远程桌面|VRDP远程桌面
-|-|-|-|-|-
 Host OS|Win7 x64|192.168.1.10|端口3389| -
 Guest OS|Xubuntu|192.168.1.12|-| 端口5001

## 假设局域网内有一台安装有Vista系统的电脑，如何远程到虚拟机里的系统呢？

- **Host-Only网络时**

```bash
  - mstsc /v:192.168.1.10         其他电脑连接到Host OS(Win7 x64)
  - mstsc /v:192.168.56.102       Host系统连接到虚拟机系统XP
  - mstsc /v:192.168.1.10:5000    其他电脑(包括Host OS)连接到虚拟机系统XP
```

- **Bridged网络时**

```bash
  - mstsc /v:192.168.1.10         其他电脑连接到Host OS(Win7 x64)
  - mstsc /v:192.168.1.10:5001    其他电脑(包括Host OS)连接到虚拟机系统Xubuntu
```

### **注意：**

>1. 外部连接的地址是Host机器的IP地址，不是虚拟机的机器名。
>2. 如果想开启两个以上的虚拟机的“开关远程桌面”的功能，需要不同的端口，默认使用3389端口，这样只能连接一台虚拟机(第一台开启该功能的虚拟机)。避免和Host系统以及Guest系统自带远程桌面的端口冲突，推荐使用5000~5050之间的端口。
>3. 注意Host系统的防火墙要允许VirtualBox访问网络。
>4. 如果使用VBoxHeadless后台方式启动虚拟机，VRDP将会自动启用，防止VRDP端口被外界访问到，添加参数--vrde=off。

原文链接：<http://yyimen.blog.163.com/blog/static/179784047201251701852363/>