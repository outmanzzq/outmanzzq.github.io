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
    curl -L https://mirrors.aliyun.com/docker-toolbox/linux/compose/$(curl -s https://mirrors.aliyun.com/docker-toolbox/linux/compose/ |egrep '^<a' |awk -F '">|</a>' '{print $2}' |sort -V |tail -1)docker-compose-Linux-x86_64 -o  /usr/bin/docker-compose

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
command=tail -f /www/wwwroot/app/worker.log
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

# 激活开机启动命令
systemctl enable supervisord

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

# 启动supervisor进程
systemctl start supervisord

# 如果修改了supervisor.service文件，可以通过reload命令来重新加载配置文件
systemctl reload supervisord.service
```

# 二、配置文件

## 1. 通过 yum/apt 安装后，supervisor 的配置文件在：

```bash
/etc/supervisor/supervisord.conf
```

supervisor 的配置文件默认是不全的，不过在大部分默认的情况下，上面说的基本功能已经满足。而其管理的子进程配置文件在：

```bash
/etc/supervisor/conf.d/*.conf
```

然后，开始给自己需要的脚本程序编写一个子进程配置文件，让 supervisor 来管理它，放在 /etc/supervisor/conf.d/ 目录下，以 .conf 作为扩展名
（每个进程的配置文件都可以单独分拆也可以把相关的脚本放一起）。如任意定义一个和脚本相关的项目名称的选项组（/etc/supervisor/conf.d/test.conf）：

```conf
#项目名
[program:blog]
#脚本目录
directory=/opt/bin
#脚本执行命令
command=/usr/bin/python /opt/bin/test.py
# supervisor 启动的时候是否随着同时启动，默认 True
autostart=true
#当程序 exit 的时候，这个program不会自动重启,默认 unexpected
#设置子进程挂掉后自动重启的情况，有三个选项，false,unexpected 和 true。如果为 false 的时候，无论什么情况下，都不会被重新启动，如果为 unexpected，只有当进程的退出码不在下面的 exitcodes 里面定义的
autorestart=false
#这个选项是子进程启动多少秒之后，此时状态如果是 running，则我们认为启动成功了。默认值为 1
startsecs=1
#日志输出 
stderr_logfile=/tmp/blog_stderr.log
stdout_logfile=/tmp/blog_stdout.log
#脚本运行的用户身份
user = zhoujy
#把 stderr 重定向到 stdout，默认 false
redirect_stderr = true
#stdout 日志文件大小，默认 50MB
stdout_logfile_maxbytes = 20M
#stdout 日志文件备份数
stdout_logfile_backups = 20


[program:zhoujy] #说明同上
directory=/opt/bin
command=/usr/bin/python /opt/bin/zhoujy.py
autostart=true
autorestart=false
stderr_logfile=/tmp/zhoujy_stderr.log
stdout_logfile=/tmp/zhoujy_stdout.log
#user = zhoujy
```

## 2. 通过 easy_install 安装后，配置文件不存在，需要自己导入

运行 echo_supervisord_conf 打印出一个配置文件的样本，要是设置样本为一个配置文件则：

```bash
#1：运行 echo_supervisord_conf，查看配置样本：
echo_supervisord_conf

#2：创建配置文件：
echo_supervisord_conf > /etc/supervisord.conf
```

配置子进程配置文件，可以直接在 supervisor 中的;[program:theprogramname]里设置。

详细的子进程配置文件：

样本：

```conf
;[program:theprogramname]
;command=/bin/cat              ; the program (relative uses PATH, can take args)
;process_name=%(program_name)s ; process_name expr (default %(program_name)s)
;numprocs=1                    ; number of processes copies to start (def 1)
;directory=/tmp                ; directory to cwd to before exec (def no cwd)
;umask=022                     ; umask for process (default None)
;priority=999                  ; the relative start priority (default 999)
;autostart=true                ; start at supervisord start (default: true)
;startsecs=1                   ; # of secs prog must stay up to be running (def. 1)
;startretries=3                ; max # of serial start failures when starting (default 3)
;autorestart=unexpected        ; when to restart if exited after running (def: unexpected)
;exitcodes=0,2                 ; 'expected' exit codes used with autorestart (default 0,2)
;stopsignal=QUIT               ; signal used to kill process (default TERM)
;stopwaitsecs=10               ; max num secs to wait b4 SIGKILL (default 10)
;stopasgroup=false             ; send stop signal to the UNIX process group (default false)
;killasgroup=false             ; SIGKILL the UNIX process group (def false)
;user=chrism                   ; setuid to this UNIX account to run the program
;redirect_stderr=true          ; redirect proc stderr to stdout (default false)
;stdout_logfile=/a/path        ; stdout log path, NONE for none; default AUTO
;stdout_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stdout_logfile_backups=10     ; # of stdout logfile backups (default 10)
;stdout_capture_maxbytes=1MB   ; number of bytes in 'capturemode' (default 0)
;stdout_events_enabled=false   ; emit events on stdout writes (default false)
;stderr_logfile=/a/path        ; stderr log path, NONE for none; default AUTO
;stderr_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stderr_logfile_backups=10     ; # of stderr logfile backups (default 10)
;stderr_capture_maxbytes=1MB   ; number of bytes in 'capturemode' (default 0)
;stderr_events_enabled=false   ; emit events on stderr writes (default false)
;environment=A="1",B="2"       ; process environment additions (def no adds)
;serverurl=AUTO                ; override serverurl computation (childutils)
```

### 参数说明：

```conf
;[program:theprogramname]      ;这个就是咱们要管理的子进程了，":"后面的是名字，最好别乱写和实际进程
                                有点关联最好。这样的 program 我们可以设置一个或多个，一个 program 就是
                                要被管理的一个进程
;command=/bin/cat              ; 这个就是我们的要启动进程的命令路径了，可以带参数
                                例子：/home/test.py -a 'hehe'
                                有一点需要注意的是，我们的 command 只能是那种在终端运行的进程，不能是
                                守护进程。这个想想也知道了，比如说 command=service httpd start。
                                httpd 这个进程被 linux 的 service 管理了，我们的 supervisor 再去启动这个命令
                                这已经不是严格意义的子进程了。
                                这个是个必须设置的项
;process_name=%(program_name)s ; 这个是进程名，如果我们下面的 numprocs 参数为 1 的话，就不用管这个参数
                                 了，它默认值 %(program_name)s 也就是上面的那个 program 冒号后面的名字，
                                 但是如果 numprocs 为多个的话，那就不能这么干了。想想也知道，不可能每个
                                 进程都用同一个进程名吧。

;numprocs=1                    ; 启动进程的数目。当不为1时，就是进程池的概念，注意 process_name 的设置
                                 默认为 1    。。非必须设置
;directory=/tmp                ; 进程运行前，会前切换到这个目录
                                 默认不设置。。。非必须设置
;umask=022                     ; 进程掩码，默认 none，非必须
;priority=999                  ; 子进程启动关闭优先级，优先级低的，最先启动，关闭的时候最后关闭
                                 默认值为 999 。。非必须设置
;autostart=true                ; 如果是 true 的话，子进程将在 supervisord 启动后被自动启动
                                 默认就是 true   。。非必须设置
;autorestart=unexpected        ; 这个是设置子进程挂掉后自动重启的情况，有三个选项，false,unexpected
                                 和 true。如果为 false 的时候，无论什么情况下，都不会被重新启动，
                                 如果为 unexpected，只有当进程的退出码不在下面的 exitcodes 里面定义的退
                                 出码的时候，才会被自动重启。当为 true 的时候，只要子进程挂掉，将会被无
                                 条件的重启
;startsecs=1                   ; 这个选项是子进程启动多少秒之后，此时状态如果是 running，则我们认为启
                                 动成功了
                                 默认值为 1 。。非必须设置
;startretries=3                ; 当进程启动失败后，最大尝试启动的次数。。当超过 3 次后，supervisor 将把
                                 此进程的状态置为 FAIL
                                 默认值为 3 。。非必须设置
;exitcodes=0,2                 ; 注意和上面的的 autorestart=unexpected 对应。。exitcodes 里面的定义的
                                 退出码是 expected 的。
;stopsignal=QUIT               ; 进程停止信号，可以为 TERM, HUP, INT, QUIT, KILL, USR1, or USR2 等信号
                                  默认为 TERM 。。当用设定的信号去干掉进程，退出码会被认为是 expected
                                  非必须设置
;stopwaitsecs=10               ; 这个是当我们向子进程发送 stopsignal 信号后，到系统返回信息
                                 给 supervisord，所等待的最大时间。 超过这个时间，supervisord 会向该
                                 子进程发送一个强制 kill 的信号。
                                 默认为 10 秒。。非必须设置
;stopasgroup=false             ; 这个东西主要用于，supervisord 管理的子进程，这个子进程本身还有
                                 子进程。那么我们如果仅仅干掉 supervisord 的子进程的话，子进程的子进程
                                 有可能会变成孤儿进程。所以咱们可以设置可个选项，把整个该子进程的
                                 整个进程组都干掉。 设置为 true 的话，一般 killasgroup 也会被设置为 true。
                                 需要注意的是，该选项发送的是 stop 信号
                                 默认为 false。。非必须设置。。
;killasgroup=false             ; 这个和上面的 stopasgroup 类似，不过发送的是 kill 信号
;user=chrism                   ; 如果 supervisord 是 root 启动，我们在这里设置这个非 root 用户，可以用来
                                 管理该 program
                                 默认不设置。。。非必须设置项
;redirect_stderr=true          ; 如果为 true，则 stderr 的日志会被写入 stdout 日志文件中
                                 默认为 false，非必须设置
;stdout_logfile=/a/path        ; 子进程的 stdout 的日志路径，可以指定路径，AUTO，none 等三个选项。
                                 设置为 none 的话，将没有日志产生。设置为 AUTO 的话，将随机找一个地方
                                 生成日志文件，而且当 supervisord 重新启动的时候，以前的日志文件会被
                                 清空。当 redirect_stderr=true 的时候，sterr 也会写进这个日志文件
;stdout_logfile_maxbytes=1MB   ; 日志文件最大大小，和 [supervisord] 中定义的一样。默认为 50
;stdout_logfile_backups=10     ; 和 [supervisord] 定义的一样。默认 10
;stdout_capture_maxbytes=1MB   ; 这个东西是设定 capture 管道的大小，当值不为 0 的时候，子进程可以从 stdout
                                 发送信息，而 supervisor 可以根据信息，发送相应的 event。
                                 默认为 0，为 0 的时候表达关闭管道。。。非必须项
;stdout_events_enabled=false   ; 当设置为 ture 的时候，当子进程由 stdout 向文件描述符中写日志的时候，将
                                 触发 supervisord 发送 PROCESS_LOG_STDOUT 类型的 event
                                 默认为 false。。。非必须设置
;stderr_logfile=/a/path        ; 这个东西是设置 stderr 写的日志路径，当 redirect_stderr=true。这个就不用
                                 设置了，设置了也是白搭。因为它会被写入 stdout_logfile 的同一个文件中
                                 默认为 AUTO ，也就是随便找个地存，supervisord 重启被清空。。非必须设置
;stderr_logfile_maxbytes=1MB   ; 这个出现好几次了，就不重复了
;stderr_logfile_backups=10     ; 这个也是
;stderr_capture_maxbytes=1MB   ; 这个一样，和 stdout_capture 一样。 默认为 0，关闭状态
;stderr_events_enabled=false   ; 这个也是一样，默认为 false
;environment=A="1",B="2"       ; 这个是该子进程的环境变量，和别的子进程是不共享的
;serverurl=AUTO                ;
```

# 三、常用命令

命令|说明
-|-
supervisorctl help                  |获取帮助
supervisorctl reread                |重新读取服务配置，不会重启
supervisorctl update                |更新配置，并重启服务
supervisorctl status                |查看服务状态
supervisorctl pid                   |获取 supervisord 守护进程 PID
supervisorctl pid+进程名             |根据进程名获取 PID
supervisorctl pid all               |获取所有服务进程 PID
supervisorctl clear all             |清除所有进程日志文件
supervisorctl fg+进程名              |将指定进程调到前台，按 Ctrl+C 退出前台
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
