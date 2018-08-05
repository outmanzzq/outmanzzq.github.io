---
layout: post
title: Ubuntu 18.04 离线安装Kubernetes v1.11.1
categories: k8s,ubuntu
description: 在Ubuntu上使用离线方式快速安装K8S v1.11.1
keywords: k8s,ubuntu
---

> 由于GFW干扰的原因，按官方的文档很难正常安装。为了方便K8S入门同学的学习，特做了这个离线安装的文档。方便大家学习。

# 一、环境说明

- 离线安装包文件下载：<https://pan.baidu.com/s/1nmC94Uh-lIl0slLFeA1-qw>
- VMWare 14 pro 虚拟三台VM
  - 配置
    - CPU：1核
    - 内存：2GB
    - 硬盘：20G
    - OS：Ubuntu 18.04 LTS
  - K8S 主机名及IP
    - k8s-master001   192.168.98.110
    - k8s-node001      192.168.98.111
    - k8s-node002      192.168.98.112

# 二、准备工作(以下操作在三台机器中进行)

## 2.1 Ubuntu 18.04 LTS

默认安装 略

## 2.2 更换阿里源（国内用户）安装文件传输工具、禁用SWAP、关闭防火墙、关闭SELINUX、配置主机名、IP地址

```bash
# 更换阿里源
sudo cp /etc/apt/sources.list{,.ori}

sudo cat >/etc/apt/sources.list <<EOF
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

# 更新源
sudo apt update && sudo apt upgrade

# 安装openssh-server(开启ssh远程连接)
sudo apt install openssh-server -y

# 开启root远程登陆
sudo sed -i 's/\#\(PermitRootLogin \).*/\1 yes/g' /etc/ssh/sshd_config

sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/g' /etc/ssh/sshd_config

sudo systemctl restart ssh

# 修改root密码
echo '123456' | sudo passwd --stdin root

# 查看vm ip（此时就就可用ssh客户端工具远程连接啦，推荐secureCRT)
ip ad |grep 'inet '

# 安装必要工具及设置
sudo -i && \
apt install lrzsz ebtables ethtool -y && \
swapoff -a && \
sed -i '/ swap / s/^/#/' /etc/fstab && \

# 重启生效
reboot
```

## 2.3 安装Docker

```bash
# 生成工作目录
mkdir -p /server/{tools,backup,scripts}

cd /server/tools

# 使用 lrzsz 到 rz -y 根据上传网盘下载到文件夹中对应压缩包到／server／tools目录下
tar xzvf docker_v18.03.1_ce.tar.gz

cd docker_v18.03.1_ce && ./install.sh
```

## 2.4 安装Kubeadm等程序

```bash
tar xzvf 002.001.k8s.deb.v1.11.1.tar.gz

cd k8s.deb.v1.11.1  && ./install.sh
```

# 三、安装Kubeadm

## 3.1  On Master  导入镜像并初始化集群

### 3.1.1  导入镜像到Master

```bash
tar xzvf 002.002.k8s.master.v1.11.1.tar.gz

cd k8s.master.v1.11.1 && chmod +x loadall.sh && ./loadall.sh

tar xzvf 003.kubeadm_init.tar.gz

cd kubeadm_init  && chmod +x install.sh && ./install.sh  #注意修改脚本中初始化的网络地址(和你本地网段不同即可)

#通过LOG文件查看客户端加入的命令

#这时候主应该就可以了。
```

## 3.2  On node001 & node002  将NODE加入集群

### 3.2.1  导入镜像到所有Node

```bash
tar xzvf 002.002.k8s.node.v1.11.1.tar.gz

cd k8s.node.v1.11.1  && ./loadall.sh

```

> #使用初始化完成的命令加入集群。

## 3.3  On Master 安装Dashboard

### 3.3.1  执行安装脚本

```bash
tar xzvf 004.kubernetes-dashboard.tar.gz

cd kubernetes-dashboard && chmod +x install.sh && ./install.sh
```

## 3.4  安装Nginx-ingress

```bash
#先在所有节点上安装

tar xzvf 005.nginx-ingress.tar.gz

cd nginx-ingress && ./install_on_node.sh

#然后在所有Master节点上安装

tar xzvf 005.nginx-ingress.tar.gz

cd nginx-ingress && ./install_on_master.sh

# 最终结果
root@k8s-master001:/server/tools/nginx-ingress# kubectl get nodes
NAME            STATUS    ROLES     AGE       VERSION
k8s-master001   Ready     master    1h        v1.11.1
k8s-node001     Ready     <none>    51m       v1.11.1
k8s-node002     Ready     <none>    49m       v1.11.1

root@k8s-master001:/server/tools/nginx-ingress# kubectl get services
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   1h

root@k8s-master001:/server/tools/nginx-ingress# kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health": "true"}
```

## 3.5  入门命令

<https://kubernetes.io/docs/home/>
<http://docs.kubernetes.org.cn/>

> 参考链接：
> <https://www.kubernetes.org.cn/4387.html>
> <https://morphyhu.szitcare.com/wordpress/?p=1139>