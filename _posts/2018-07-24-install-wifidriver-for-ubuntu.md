---
layout: post
title: Thinkpad E470 安装 Ubuntu 18.04 LTS 安装 WiFi 模块驱动
categories: ubuntu
description: Thinkpad E470 安装 Ubuntu 18.04 LTS 安装 WiFi 模块驱动
keywords: wifi,ubuntu
---

> 单位发的 Thinkpad E470C i5-6125U 商务极速版 一块ssd256G 安装ubuntu 18.04 LTS，发现没有wifi驱动，无线网是 Retaltek Semicondutor Co. Ltd Device c821【就是rtl8821ce】，没错就是这个坑比的网卡，linux现在不支持这个型号的网卡，网上找了好久终于解决，在此记录以便日后查阅。

# 环境说明

- 电脑：Thinkpad E470C i5-6125U 商务极速版
- wifi驱动：<https://github.com/outmanzzq/rtl8821ce.git>

# 具体操作过程

## 第一步：[更新阿里云APT源](https://opsx.alibaba.com/mirror)(点对应系统版本后面的“帮助”按钮)

```bash
# 备份
sudo cp /etc/apt/sources.list{,.ori}

# ubuntu 18.04 LTS 源
sudo cat > /etc/apt/sources.list <<EOF
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
EOF

#更新源
sudo apt update
```

## 第二步：安装依赖包

```bash
sudo apt-get install build-essential git -y
```

## 第三步：下载并编译wifi驱动源码包

```bash
# 创建工作目录
sudo mkdir -p /server/{tools,backup,scripts}
cd /server/tools

# 下载wifi驱动源码包
git clone https://github.com/outmanzzq/rtl8821ce.git

cd rtl8821ce

unzip rtl8821ce.zip

cd rtl8821ce

make

sudo make install

sudo modprobe -a 8821ce

# 重启生效
reboot

```

> 参考链接：
> 1. <https://blog.csdn.net/fljhm/article/details/79281655>
> 2. <http://forum.ubuntu.com.cn/viewtopic.php?f=116&t=485936>