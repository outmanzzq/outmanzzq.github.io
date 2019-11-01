---
layout: post
title: 迁移 Docker 容器储存位置
categories: docker,volume
description: 一般来说我们需要将系统磁盘和应用数据盘进行分离，除了能够获得更好的性能，最关键的还是能够让数据更安全可靠：多数云服务数据盘支持备份快照、并且支持大容量 SSD 盘。
keywords: docker,volume
---

> 要将系统磁盘和应用数据盘进行分离，除了能够获得更好的性能，最关键的还是能够让数据更安全可靠：多数云服务数据盘支持备份快照、并且支持大容量 SSD 盘。

## 写在前面

先使用 df 了解下当前机器的分区状况。

```bash
# df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            2.0G     0  2.0G   0% /dev
tmpfs           395M  5.3M  390M   2% /run
/dev/vda1        40G  8.3G   30G  22% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/vdb1        20G   45M   19G   1% /data
tmpfs           395M     0  395M   0% /run/user/0
```

可以看到系统盘有 40G，挂载在 / 根目录，设备是 /dev/vda1，而数据盘有20G，挂载在 /data （个人习惯），设备为 /dev/vdb1。

如果是老机器，有运行中的容器，可能会看到类似下面的输出。

```bash
overlay         196G   24G  163G  13% /var/lib/docker/overlay2/69e985e9fbc2bbaee2fbdcd81c514d64c4ed9862233bf4797a75ac10df80ed1e/merged
shm              64M  4.0K   64M   1% /var/lib/docker/containers/14777d5d02f2600ea134a8eff061dc4d2fd440b747c936da6024386f457a9c2c/mounts/shm
```

在迁移之前，我们需要了解默认的容器数据保存位置。

```bash
# docker info | grep "Docker Root Dir"
Docker Root Dir: /var/lib/docker
```

通过 docker info 我们可以看到默认的安装位置在 /var/lib/docker，没错，默认是在系统盘，随着下载镜像越来越多，构建镜像、运行容器越来越多，系统盘可能会迅速被它蚕食而发生一些意料之外的事情: 系统无法启动、或者严重变慢，所以强烈建议对它进行迁移。

## 开始迁移

考虑到有一些同学并不是新机器，所以这里简单启动一个 Nginx 容器，来模拟“有数据”状态，帮助我们验证迁移结果。

`docker run -d -p 8080:80 nginx`

Nginx 启动之后，我们使用 curl 验证服务是否正常。

```bash
$ curl 127.0.0.1:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>
 
<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>
 
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

接着使用 du 命令来看看，上小节使用 docker info 了解到的 docker 默认数据目录有多大。

```bash
$ du -hs /var/lib/docker
4.3G  /var/lib/docker
```

如果你确定你的镜像都已经妥善保存好、或者用的都是公开的镜像，容器实例中没有存储特别的东西，可以考虑先执行 docker system prune 给 docker 数据目录先减个肥，再进行迁移。

要进行数据迁移，需要先暂停 docker 服务。

`service docker stop`

创建迁移目录（用来放新数据的目录），我个人习惯将可备份的用户数据存放于应用分区 /data 下。

`mkdir -p /data/docker/`

然后使用万能的 rsync 对数据进行迁移。

`rsync -avz /var/lib/docker/ /data/docker`

在长长的屏幕日志滚动之后，你将会看到类似下面的输出：

```bash
docker/tmp/
docker/trust/
docker/volumes/
docker/volumes/metadata.db
 
sent 1,514,095,568 bytes  received 3,096,373 bytes  4,998,984.98 bytes/sec
total size is 3,955,563,885  speedup is 2.61
```

数据就这样迁移完毕了，完整性由 rsync 保证。接下来要修改 docker 的配置，让 docker 从新的位置进行数据加载和存储。

编辑 /etc/docker/daemon.json 配置文件，如果没有这个文件，那么需要自己创建一个，根据上面的迁移目录，基础配置如下：

```json
{
    "data-root": "/data/docker",
    "registry-mirrors": [
        "http://YOUR_MIRROR_LINK"
    ]
}
```

将容器服务启动起来。

`service docker start`

使用文章开头的命令再次验证下 docker 数据存储设置，可以看到配置已经生效。

```bash
# docker info | grep "Docker Root Dir"
Docker Root Dir: /data/docker
```

还记得这小节开头提到的 Nginx 容器嘛，我们将它重新启动，来验证服务是否可用，先找到这个容器的“尸体”。

```bash
# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                      PORTS               NAMES
fd9b79ae8574        nginx               "nginx -g 'daemon of…"   44 minutes ago      Exited (0) 31 m
```

接着使用容器基础命令将实例启动。

`docker start fd9b79ae8574`

最后再使用 curl 验证一下结果：

```bash
$ curl 127.0.0.1:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>

<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
```

至此，迁移就大功告成啦。

对了，你还记得我们最开始看到的 /var/lib/docker 目录嘛，它现在已经完全无用了，可以使用 rm -rf /var/lib/docker  将它清理掉啦。

> 原文链接：<https://soulteary.com/2019/07/14/migrate-docker-container-storage-location.html>