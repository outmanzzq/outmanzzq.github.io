---
layout: post
title: Linux进程管理利器——supervisor使用
categories: Linux,supervisor
description: Supervisord是用Python实现的一款非常实用的进程管理工具
keywords: Linux,supervisor
---

> supervisor管理进程，是通过fork/exec的方式将这些被管理的进程当作supervisor的子进程来启动，所以我们只需要将要管理进程的可执行文件的路径添加到supervisor的配置文件中就好了。此时被管理进程被视为supervisor的子进程，若该子进程异常中断，则父进程可以准确的获取子进程异常中断的信息，通过在配置文件中设置autostart=ture，可以实现对异常中断的子进程的自动重启。
> 本文基于vagrant+vbox构建Centos7系统，安装测试supervisor工具使用。

- vagrant：1.9.5
- vbox: 5.1.30
- vagrant Centos7.5 box [下载地址](https://app.vagrantup.com/centos/boxes/7/versions/1809.01/providers/virtualbox.box)

> box 下载地址拼接规则：https://my.oschina.net/cxgphper/blog/1940644

- Centos 7.5
  - IP: 192.168.11.150
  - user: root
  - passwd: root
- supervisor: 3.1.4-1

> supervisor官网地址：http://supervisord.org/

# 一、操作过程

## 1. vbox、vagrant 安装略。

Vagrantfile 文件内容：

```YAML
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  config.vm.box = "centos/7.5-1809"
  config.vm.network "public_network", ip: "192.168.11.150"
  config.vm.synced_folder "../_sharefolders", "/vagrant", SharedFoldersEnableSymlinksCreate: true

  config.vm.provision "shell", inline: <<-SHELL
  set -ex
  sudo -i

  ##设置时区
  timedatectl set-timezone Asia/Shanghai

  ##生成工作目录
  mkdir -p /server/{tools,scripts,backup,docker-compose}

  ##开启ssh root密码登录
  sed -i 's/#PermitRootLogin yes/PermitRootLogin yes/' /etc/ssh/sshd_config 
  sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
  systemctl restart sshd
  # set root password
  echo 'root' |  passwd --stdin root

  ##更换阿里云源
  mv /etc/yum.repos.d/CentOS-Base.repo{,.backup}
  curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
  yum update -y

  ##安装常用工具
  yum install epel-release -y
  yum install tree lrzsz dos2unix net-tools htop -y

  #安装docker
  yum remove -y docker \
       docker-client \
       docker-client-latest \
       docker-common \
       docker-latest \
       docker-latest-logrotate \
       docker-logrotate \
       docker-selinux \
       docker-engine-selinux \
       docker-engine

  yum install -y yum-utils \
       device-mapper-persistent-data \
       lvm2
  yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
  yum makecache fast
  yum -y install docker-ce
  systemctl start docker
  systemctl enable docker
  docker version
  ##镜像加速
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<EOF
{
 "registry-mirrors": ["https://2apmvngw.mirror.aliyuncs.com"]
}
EOF

    systemctl daemon-reload
    systemctl restart docker

    ##安装最新docker-compose
    curl -L https://mirrors.aliyun.com/docker-toolbox/linux/compose/\`curl -s https://mirrors.aliyun.com/docker-toolbox/linux/compose/ |egrep '^<a' |awk -F '">|</a>' '{print $2}' |sort -V |tail -1`docker-compose-Linux-x86_64 -o  /usr/bin/docker-compose

    [ ! -f /usr/bin/docker-compose ] && !!
    chmod +x /usr/bin/docker-compose
    docker-compose -v
    exit
  SHELL
end
```

启动vagrant 主机

```bash
vagrant up

#进入centos
vagrant ssh
```

## 2. Centos7 安装并配置supervisor

```bash
# su root user
su

# 安装supervisor
yum install -y supervisor

# 生成模拟tomcat进程
mkdir -p /www/wwwroot/app/

cat > /etc/supervisord.d/tomcat.ini <<EOF
[program:tomcat]
process_name=%(program_name)s_%(process_num)02d
command=watch 'free -m'
autostart=true
autorestart=true
user=vagrant
numprocs=5
redirect_stderr=true
stdout_logfile=/www/wwwroot/app/worker.log
EOF

# 开启web管理，浏览器访问地址：192.168.11.150:9001 (user/123)
sed -i '/\[supervisord\]/i\\[inet_http_server\]\nport=*:9001\nusername=user\npassword=123\n' supervisord.conf

# 添加supervisord服务开机自启动 https://www.jianshu.com/p/e1c3e6fbae80

cat >/usr/lib/systemd/system/supervisord.service <<EOF
# supervisord service for systemd (CentOS 7.0+)
# by ET-CS (https://github.com/ET-CS)
[Unit]
Description=Supervisor daemon

[Service]
Type=forking
ExecStart=/usr/bin/supervisord -c /etc/supervisord.conf
ExecStop=/usr/bin/supervisorctl $OPTIONS shutdown
ExecReload=/usr/bin/supervisorctl $OPTIONS reload
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
EOF

# 激活开机启动命令
systemctl enable supervisord

# 启动supervisor进程
systemctl start supervisord

# 关闭supervisor进程
systemctl stop supervisord

# 如果修改了supervisor.service文件，可以通过reload命令来重新加载配置文件
systemctl reload supervisord.service
```

# 二、 常用命令

命令|说明
-|-
supervisorctl help                  |获取帮助
supervisorctl reread                |重新读取服务配置，不会重启
supervisorctl update                |更新配置，并重启服务
supervisorctl status                |查看服务状态
supervisorctl pid                   |获取supervisord 守护进程PID
supervisorctl pid+进程名             |根据进程名获取PID
supervisorctl pid all               |获取所有服务进程PID
supervisorctl clear all             |清除所有进程日志文件
supervisorctl fg+进程名              |将指定进程调到前台，按Ctrl+C 退出前台
supervisorctl restart               |重启一个进程（不会重新读取进程配置文件）
supervisorctl restart+gname         |重启一个组里所有进程（不会重新读取进程配置文件）
supervisorctl restart <name\><name\> |重启多个进程或组（不会重新读取配置文件）
supervisorctl restart all           |重启所有进程，(不会重新读取配置文件）
supervisorctl start <name\>         |启动一个进程
supervisorctl start <gname\>        |启动一个组里所有进程
supervisorctl start <name\><name\>  |启动多个进程或组
supervisorctl start all             |启动所有进程
supervisorctl stop <name\>          |停止一个进程
supervisorctl stop <gname\>:*       |停止一个组里所有进
supervisorctl stop <name\><name\>   |停止多进程或组
supervisorctl stop all              |停止所有服务（不会自动重启）
supervisorctl tail [-f] <name\> [stdout\|stderr] (default stdout)  |倒序持续输出进程最新日志
