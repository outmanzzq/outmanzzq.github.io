---
layout: post
title: 深入理解 Docker Volume (一)
categories: docker,volume
description: 深入理解 Docker Volume(一)
keywords: docker,volume,volume_from
---

> 想要了解 Docker Volume,首先我们需要知道 Docker 的文件系统是如何工作的.Docker 镜像是由多个文件系统(只读层)叠加而成.

> 当我们启动一个容器的时候,Docker 会加载镜像层并在其上添加一个读写层.

> 如果运行中的容器修改了现有的一个已存在的文件,那该文件将会从读写层下的只读层复制到读写层,该文件的只读版本仍然存在,只是已经被读写层中该文件的副本所隐藏.

> 当删除 Docker 容器,并通过该镜像重新启动时,之前的更改将会丢失.在 Docker 中,只读层以及在顶部的读写层的组合被称为 Union FIle System(联合文件系统).

**为了能够保存(持久化)数据以及共享容器间的数据,Docker 提出了 Volume 的概念.简单来说,Volume 就是目录或者文件,它可以绕过默认的联合文件系统,而以正常的文件或者目录的形式存在于宿主机上.**

我们可以通过两种方式来初始化 Volume,这两种方式有些细小而又重要的差别.我们可以在运行时使用 -v 来声明 Volume:

```bash
root@syx-VB:/home/syx# docker run -it --name container-test -h CONTAINER -v /data ubuntu /bin/bash  
root@CONTAINER:/# ls /data/  
root@CONTAINER:/#
```

**上面的命令会将 /data 挂载到容器中,并绕过联合文件系统,我们可以在主机上直接操作该目录.**

任何在该镜像 /data 路径的文件的文件都会被复制到 Volume.我们可以使用 docker inspect 命令找到 Volume 在主机上的存储位置:

```bash
$docker inspect container-test  
    "Mounts": [  
        {  
            "Name": "6407cbb6700a4076cdeeef60629f1748ff34310102480a3f702fd3fee9e69134",  
            "Source": "/var/lib/docker/volumes/6407cbb6700a4076cdeeef60629f1748ff34310102480a3f702fd3fee9e69134/_data",  
            "Destination": "/data",  
            "Driver": "local",  
            "Mode": "",  
            "RW": true  
        }  
    ],  
```

这说明 Docker 把在 /var/lib/docker 下的某个目录挂载到了容器内的 /data 目录下.让我们从主机添加文件都此文件夹下:

```bash
root@syx-VB:~# touch /var/lib/docker/volumes/6407cbb6700a4076cdeeef60629f1748ff34310102480a3f702fd3fee9e69134/_data/test-file  
```

进入容器

```bash
root@syx-VB:~# docker attach container-test  
root@CONTAINER:/# ls /data/  
test-file
```

只要将主机的目录挂载到容器的目录上,那改变就会立即生效.我们可以在 Dockerfile 中通过使用 VOLUME 指令来达到相同的目的:

> FROM ubunut VOLUME /data

**但是还有另一件只有 -v 参数能够做到而 Dockerfile 是做不到的事情就是在容器上挂载指定的主机目录.**例如:

```bash
root@syx-VB:~# docker run -v /home/syx/dockerfile:/data ubuntu ls /data  
df_test1
```

**该命令将挂载主机的 /home/syx/dockerfile 目录到容器内的 /data 目录上.

任何在 /home/syx/dockerfile 目录下的文件都会出现在容器内.这对于在主机和容器之间共享文件是非常有用的,例如挂载需要编译的源代码.为了保证可移植性,挂载主机目录不需要从 Dockerfile 指定.当使用 -v 参数时,镜像目录下的任何文件都不会被复制到 Volume 中.**

# 数据共享

 如果要授权一个容器访问另一个容器的 Volume ,我们可以使用 -volumes-from 参数来执行 docker run

 ```bash
root@syx-VB:~# docker run -it -h NEWCONTAINER --volumes-from container-test ubuntu /bin/bash  
root@NEWCONTAINER:/# ls /data/  
test-file
 ```

> 值得注意的是,就算你这个时候把 container-test 停止了,它仍然会起作用.只要有容器连接 Volume,他就不会被删除,如果这个时候你执行:

```bash
root@syx-VB:~# docker rm container-test  
Error response from daemon: Conflict, You cannot remove a running container. Stop the container before attempting removal or use -f  
Error: failed to remove containers: [container-test]  
```

# 数据容器

常见的使用场景是使用纯数据容器来持久化数据库,配置文件或者数据文件等.例如:

> $docker run --name dbdate postgres echo “Data-Only container for postgres”

该命令将会创建一个已经包含在 Dockerfile 里定义过 Volume 的 postgres 镜像,运行 echo 命令然后退出.当我们运行 docker ps 命令时,echo 可以帮助我们识别某镜像的用途.我们可以用 -volume-from 命令来识别其他容器的 Volume:

> $docker run -d --volumes-from dadate --name db1 postgres  

**使用数据容器的两个注意点:**

1. 不要运行数据容器,这纯属是在两非自愿
2. 不要为了数据容器而使用”最小的镜像”,如 busybox 或 scratch,只使用数据库镜像本身就可以了.你已经拥有了该镜像,所以不需要占用额外的空间.

# 备份

如果你在用数据容器,那做备份是相当容易的.

```bash
root@syx-VB:~# docker run --rm --volumes-from dbdate -v $(pwd):/backup ubuntu tar cvf /backup/backup.tar /var/lib/postgresql/data
```

该命令会将 Volume 里所有的东西压缩为一个 tar 包(官方的 postgres Dockerfile 在 /var/lib/postgresql/data 目录下定义了一个 Volume).

```bash
root@syx-VB:~# ls  
backup.tar  
```

# 删除Volumes

这个功能太重要了,如果你已经使用 docker run 来删除你的容器,那可能会有很多孤立的 Volume 仍在占用着空间.

**Voulume 可以被删除的条件:**

1. 该容器可以用 docker rm -v来删除且没有其他容器连接到该 Volume (以及主机目录是也没被指定为 Volume ).注意, -v 是必不可少的.
2. docker run 中使用 rm 参数.

> 原文链接：<https://www.cnblogs.com/ilinuxer/p/6613904.html>