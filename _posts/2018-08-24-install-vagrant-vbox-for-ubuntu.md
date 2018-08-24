---
layout: post
title: Ubuntu 18.04 LTS 安装 Vagrant + VirtualBox
categories: ubuntu,vagrant
description: Ubuntu 18.04 LTS 安装 Vagrant + VirtualBox
keywords: ubuntu,vagrant
---

> Vagrant是一个搭建完整的虚拟开发环境的工具，通常简写为VDE（Virtual Development Environment）。Vagrant 节省大量重建操作系统环境的时间，它也是一个配置中心，允许你使用一个相同的配置管理和部署多个 VDE。安装 Vagrant 的同时，你也需要安装 VirtualBox，因为它是 Vagrant 的核心功能组建。

# 环境说明

- 系统：Ubuntu 18.04 LTS
- Vagrant: 2.1.2
- VirtualBox: 5.2.18 r124319
- 确保 CPU 支持并开启虚拟化！

## 软件包

- Vagrant：<https://releases.hashicorp.com/vagrant/2.1.2/vagrant_2.1.2_x86_64.deb>
- VirtualBox: <https://vagrantcloud.com/generic/boxes/alpine37/versions/1.8.24/providers/virtualbox.box>

# 一、具体安装过程

## 1. 安装前初始化工作

```bash
#1. 更换阿里云软件源
# change to root
$ sudo -i
$ cp /etc/apt/sources.list{,.ori}

$ cat > /etc/apt/sources.list <<EOF
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

$ apt update && sudo apt upgrade -y

#2. 安装常用软件
$ apt install git lrzsz tree curl wget build-essential linux-headers-4.15.0-20-generic -y

#3. 生成工作目录
$ mkdir -p /server/{tools,scripts,backup} && cd /server/tools
```

## 2. 下载并安装软件包

```bash
#下载
$ wget https://releases.hashicorp.com/vagrant/2.1.2/vagrant_2.1.2_x86_64.deb

$ wget https://download.virtualbox.org/virtualbox/5.2.18/virtualbox-5.2_5.2.18-124319~Ubuntu~bionic_amd64.deb

#检查
$ ll
total 109060
drwxr-xr-x 2 root root     4096 8月  23 23:45 ./
drwxr-xr-x 6 root root     4096 8月  23 23:48 ../
-rw-r--r-- 1 root root 43525726 7月  31 05:04 vagrant_2.1.2_x86_64.deb
-rw-r--r-- 1 root root 68127004 8月  23 23:40 virtualbox-5.2_5.2.18-124319~Ubuntu~bionic_amd64.deb

#安装
$ dpkg -i virtualbox-5.2_5.2.18-124319~Ubuntu~bionic_amd64.deb

$ dpkg -i vagrant_2.1.2_x86_64.deb

#如报错，则执行此命令修复安装
$ apt install -f -y

#加载 vboxdrv kernel 模块
$ /sbin/vboxconfig

# 检查
$ vagrant -v
Vagrant 2.1.2

```

## 3. 在执行 vagrant 之前首先配置 Box

```bash
# 格式
$ vagrant box add {title} {url}
$ vagrant init {title}
$ vagrant up
```

## 4. 示例（ Alpine 镜像）

```bash
$ mkdir -p /server/vagrant && cd /server/vagrant

$ vagrant init
$ cat > Vagrantfile <<EOF 
Vagrant.configure("2") do |config|
  config.vm.box = "generic/alpine37"
  config.vm.box_version = "1.8.24"
end
EOF

$ vagrant status
Current machine states:

default                   not created (virtualbox)

The environment has not yet been created. Run `vagrant up` to
create the environment. If a machine is not created, only the
default provider will be shown. So if a provider is not listed,
then the machine is not created for that environment.

#启动（如果 box 下载不下来，挂代理！或直接将 xxx.box 链接手动下载，再导入 box）
$ vagrant up

$ vagrant status

Current machine states:

default                   running (virtualbox)

The VM is running. To stop this VM, you can run `vagrant halt` to
shut it down forcefully, or you can run `vagrant suspend` to simply
suspend the virtual machine. In either case, to restart it again,
simply run `vagrant up`.

#进入虚拟机
$ vagrant ssh
localhost:~$ df -hP
Filesystem                Size      Used Available Capacity Mounted on
devtmpfs                 10.0M         0     10.0M   0% /dev
shm                    1001.4M         0   1001.4M   0% /dev/shm
/dev/sda3                31.3G    258.2M     29.4G   1% /
tmpfs                   200.3M    140.0K    200.1M   0% /run
df: /sys/kernel/security: Permission denied
cgroup_root              10.0M         0     10.0M   0% /sys/fs/cgroup
/dev/sda1                92.8M     14.0M     71.8M  16% /boot

#退出
localhost:~$ exit
logout
Connection to 127.0.0.1 closed.
```

至此，ubuntu 安装配置 vagrant 完毕。

# 二、vagrant 常用命令

```bash
#
$ vagrant init  　 # 初始化
$ vagrant up  　　 # 启动虚拟机
$ vagrant halt  　# 关闭虚拟机
$ vagrant reload  # 重启虚拟机
$ vagrant ssh  　  # SSH 至虚拟机
$ vagrant status   # 查看虚拟机运行状态
$ vagrant destroy  # 销毁当前虚拟机
```

> Vagrant 更多命令：<https://outmanzzq.github.io/wiki/vagrant/>
