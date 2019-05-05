---
layout: post
title: nodejs高大上的部署方式-PM2
categories: nodejs,PM2
description: 如果直接通过node app来启动，如果报错了可能直接停在整个运行，supervisor感觉只是拿来用作开发环境的。再网上找到pm2.
keywords: nodejs,PM2
---

> 如果直接通过node app来启动，如果报错了可能直接停在整个运行，supervisor感觉只是拿来用作开发环境的。再网上找到pm2.

# 简介

如果直接通过node app来启动，如果报错了可能直接停在整个运行，supervisor感觉只是拿来用作开发环境的。再网上找到pm2.目前似乎最常见的线上部署nodejs项目的有forever,pm2这两种。
使用场合:

- supervisor是开发环境用。
- forever管理多个站点，每个站点访问量不大，不需要监控。
- nodemon 是开发环境使用，修改自动重启。
- pm2 网站访问量比较大,需要完整的监控界面。

# PM2的主要特性

- 内建负载均衡（使用Node cluster 集群模块）
- 后台运行
- 0秒停机重载，我理解大概意思是维护升级的时候不需要停机.
- 具有Ubuntu和CentOS 的启动脚本
- 停止不稳定的进程（避免无限循环）
- 控制台检测
- 提供 HTTP API
- 远程控制和实时的接口API ( Nodejs 模块,允许和PM2进程管理器交互 )

## 1、最常用的属nohup了，其实就是在后台执行进程，末尾加个&

```shell
[zhoujie@ops-dev ~]$ nohup node /home/zhoujie/ops/app.js &
[1] 31490nohup: ignoring input and appending output to `nohup.out'
```

即此时程序已启动，直接访问即可，原程序的的标准输出被自动改向到当前目录下的nohup.out文件，起到了log的作用。该命令可以在你退出帐户/关闭终端之后继续运行相应的进程。
nohup就是不挂起的意思( no hang up)。

该命令的一般形式为：`nohup command &`

这个不太靠谱的样子，经常默默的进程在后台就挂了

## 2、用screen另开一个屏幕，这种方式可以直接在屏幕上看到程序运行情况

给该应用程序开个screen，如：screen -r ops ，用npm start启动，

退出该后台：ctrl + a，再按d，可不能直接ctrl +c，否则就退出了

这种方式很不专业，呵呵，不过方便看在生产环境的操作。

这个本质上用的forever，package.json里配置的：

```nodejs
  "scripts": {
    "start": "forever app.js",
    "test": "supervisor app.js"
  },
```

## 3、PM2

使用它要先安装它，用root账号和全局模式安装一下：

`npm install -g pm2`

用它来启动程序（在当前目录下可以直接启动，pm2 start app.js --name uops）

```shell
[zhoujie@ops-dev uops]$ pm2 start app.js 
[PM2] Spawning PM2 daemon
[PM2] Success
[PM2] Process app.js launched
┌──────────┬────┬──────┬─────┬────────┬───────────┬────────┬─────────────┬──────────┐
│ App name │ id │ mode │ PID │ status │ restarted │ uptime │      memory │ watching │
├──────────┼────┼──────┼─────┼────────┼───────────┼────────┼─────────────┼──────────┤
│ app      │ 0  │ fork │ 308 │ online │         0 │ 0s     │ 21.879 MB   │ disabled │
└──────────┴────┴──────┴─────┴────────┴───────────┴────────┴─────────────┴──────────┘
 Use `pm2 info <id|name>` to get more details about an app
[zhoujie@ops-dev uops]$
```

看，它显示了Success，程序已经默默的成功的启动了，可以实时监控程序的运行，比如执行个pm2 restart，则上述restarted那栏变成1，可以显示程序运行了多长时间、占用内存大小，实在是太赞啦！

![ ](/images/20190505-pm2-for-nodejs-01.jpg)

终止程序也很简单：pm2 stop

![ ](/images/20190505-pm2-for-nodejs-02.jpg)

列举出所有用pm2启动的程序：pm2 list

```shell
[zhoujie@ops-dev uops]$ pm2 list
┌──────────┬────┬──────┬─────┬────────┬───────────┬────────┬─────────────┬──────────┐
│ App name │ id │ mode │ PID │ status │ restarted │ uptime │      memory │ watching │
├──────────┼────┼──────┼─────┼────────┼───────────┼────────┼─────────────┼──────────┤
│ app      │ 0  │ fork │ 984 │ online │         1 │ 3s     │ 64.141 MB   │ disabled │
└──────────┴────┴──────┴─────┴────────┴───────────┴────────┴─────────────┴──────────┘
 Use `pm2 info <id|name>` to get more details about an app
```

查看启动程序的详细信息：`pm2 describe id`

```shell
[zhoujie@ops-dev uops]$ pm2 desc 0
Describing process with pid 0 - name app
┌───────────────────┬─────────────────────────────────────────┐
│ status            │ online                                  │
│ name              │ app                                     │
│ id                │ 0                                       │
│ path              │ /home/zhoujie/uops/app.js               │
│ args              │                                         │
│ exec cwd          │ /home/zhoujie/uops                      │
│ error log path    │ /home/zhoujie/.pm2/logs/app-error-0.log │
│ out log path      │ /home/zhoujie/.pm2/logs/app-out-0.log   │
│ pid path          │ /home/zhoujie/.pm2/pids/app-0.pid       │
│ mode              │ fork_mode                               │
│ node v8 arguments │                                         │
│ watch & reload    │ ✘                                       │
│ interpreter       │ node                                    │
│ restarts          │ 1                                       │
│ unstable restarts │ 0                                       │
│ uptime            │ 93s                                     │
│ created at        │ 2015-01-07T09:41:25.672Z                │
└───────────────────┴─────────────────────────────────────────┘
[zhoujie@ops-dev uops]$
```

通过pm2 list命令来观察所有运行的进程以及它们的状态已经足够好了.但是怎么来追踪它们的资源消耗呢?别担心,用这个命令:pm2 monit

![ ](/images/20190505-pm2-for-nodejs-03.jpg)

可以得到进程(以及集群)的CPU的使用率和内存占用(ctrl +c 退出)

实时集中log处理：pm2 logs

![ ](/images/20190505-pm2-for-nodejs-04.jpg)

### 强大API： pm2 web

你想要监控所有被PM2管理的进程,而且同时还想监控运行这些进程的机器的状态，

```shell
[zhoujie@ops-dev uops]$ pm2 web
Launching web interface on port 9615
[PM2] Process /usr/local/node/lib/node_modules/pm2/lib/HttpInterface.js launched
[PM2] Process launched
┌────────────────────┬────┬──────┬──────┬────────┬───────────┬────────┬─────────────┬──────────┐
│ App name           │ id │ mode │ PID  │ status │ restarted │ uptime │      memory │ watching │
├────────────────────┼────┼──────┼──────┼────────┼───────────┼────────┼─────────────┼──────────┤
│ app                │ 0  │ fork │ 984  │ online │         1 │ 9m     │ 74.762 MB   │ disabled │
│ pm2-http-interface │ 1  │ fork │ 1878 │ online │         0 │ 0s     │ 15.070 MB   │ disabled │
└────────────────────┴────┴──────┴──────┴────────┴───────────┴────────┴─────────────┴──────────┘
 Use `pm2 info <id|name>` to get more details about an app
```

启动程序的时候顺便在浏览器访问：http://localhost:9615

擦，我眼睛被亮瞎了，这么炫酷，竟然把部署的服务器的信息和程序的信息都显示出来了：

这东西对程序运行的监控页面的开发实在是太有帮助了，呵呵~~

监控：pm2 monit
实时集中log处理: pm2 logs
API:pm2 web (端口：9615 )

# 常用命令总结

```shell
$ pm2 logs 显示所有进程日志
$ pm2 stop all 停止所有进程
$ pm2 restart all 重启所有进程
$ pm2 reload all 0秒停机重载进程 (用于 NETWORKED 进程)
$ pm2 stop 0 停止指定的进程
$ pm2 restart 0 重启指定的进程
$ pm2 startup 产生 init 脚本 保持进程活着
$ pm2 web 运行健壮的 computer API endpoint (http://localhost:9615)
$ pm2 delete 0 杀死指定的进程
$ pm2 delete all 杀死全部进程
 

运行进程的不同方式：
$ pm2 start app.js -i max 根据有效CPU数目启动最大进程数目
$ pm2 start app.js -i 3 启动3个进程
$ pm2 start app.js -x 用fork模式启动 app.js 而不是使用 cluster
$ pm2 start app.js -x -- -a 23 用fork模式启动 app.js 并且传递参数 (-a 23)
$ pm2 start app.js --name serverone 启动一个进程并把它命名为 serverone
$ pm2 stop serverone 停止 serverone 进程
$ pm2 start app.json 启动进程, 在 app.json里设置选项
$ pm2 start app.js -i max -- -a 23 在--之后给 app.js 传递参数
$ pm2 start app.js -i max -e err.log -o out.log 启动 并 生成一个配置文件
```

# 配置pm2启动文件

```shell
在项目根目录添加一个processes.json：
内容如下:

```nodejs
{
  "apps": [
    {
      "name": "mywork",
      "cwd": "/srv/node-app/current",
      "script": "bin/www",
      "log_date_format": "YYYY-MM-DD HH:mm Z",
      "error_file": "/var/log/node-app/node-app.stderr.log",
      "out_file": "log/node-app.stdout.log",
      "pid_file": "pids/node-geo-api.pid",
      "instances": 6,
      "min_uptime": "200s",
      "max_restarts": 10,
      "max_memory_restart": "1M",
      "cron_restart": "1 0 * * *",
      "watch": false,
      "merge_logs": true,
      "exec_interpreter": "node",
      "exec_mode": "fork",
      "autorestart": false,
      "vizion": false
    }
  ]
}
```

> 说明:

- apps:json结构，apps是一个数组，每一个数组成员就是对应一个pm2中运行的应用

- name:应用程序名称

- cwd:应用程序所在的目录

- script:应用程序的脚本路径

- log_date_format:

- error_file:自定义应用程序的错误日志文件

- out_file:自定义应用程序日志文件

- pid_file:自定义应用程序的pid文件

- instances:

- min_uptime:最小运行时间，这里设置的是60s即如果应用程序在60s内退出，pm2会认为程序异常退出，此时触发重启max_restarts设置数量

- max_restarts:设置应用程序异常退出重启的次数，默认15次（从0开始计数）

- cron_restart:定时启动，解决重启能解决的问题

- watch:是否启用监控模式，默认是false。如果设置成true，当应用程序变动时，pm2会自动重载。这里也可以设置你要监控的文件。

- merge_logs:

- exec_interpreter:应用程序的脚本类型，这里使用的shell，默认是nodejs

- exec_mode:应用程序启动模式，这里设置的是cluster_mode（集群），默认是fork

- autorestart:启用/禁用应用程序崩溃或退出时自动重启

- vizion:启用/禁用vizion特性(版本控制)

可以通过pm2 start processes.json来启动。

也可以把命令写在package.json里。如下:

```nodejs
 "scripts": {
    "dev": "NODE_ENV=development nodemon src/server.js & NODE_ENV=development nodemon src/server/action-server.js & tools/redis/socket.js",
    "dev_read_redis": "NODE_ENV=development nodemon src/app.js",
    "start": "NODE_ENV=production nodemon src/app.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
```

通过npm run start来启动。

> 相关链接：
>
> - 关于pm2远程部署到多台机器 <http://pm2.keymetrics.io/docs/usage/deployment/>
> - 官网：<http://pm2.keymetrics.io/docs/usage/quick-start/#42-starts>