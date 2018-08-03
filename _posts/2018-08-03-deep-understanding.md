---
layout: post
title: 深入理解 Docker Volume
categories: docker,volume
description: 深入理解 Docker Volume
keywords: docker,volume,volume_from
---

> 想要了解 Docker Volume,首先我们需要知道 Docker 的文件系统是如何工作的.Docker 镜像是由多个文件系统(只读层)叠加而成.

> 当我们启动一个容器的时候,Docker 会加载镜像层并在其上添加一个读写层.

> 如果运行中的容器修改了现有的一个已存在的文件,那该文件将会从读写层下的只读层复制到读写层,该文件的只读版本仍然存在,只是已经被读写层中该文件的副本所隐藏.

> 当删除 Docker 容器,并通过该镜像重新启动时,之前的更改将会丢失.在 Docker 中,只读层以及在顶部的读写层的组合被称为 Union FIle System(联合文件系统).

# 一、基础篇

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

```bash
FROM ubunut VOLUME /data
```

**但是还有另一件只有 -v 参数能够做到而 Dockerfile 是做不到的事情就是在容器上挂载指定的主机目录.**例如:

```bash
root@syx-VB:~# docker run -v /home/syx/dockerfile:/data ubuntu ls /data  
df_test1
```

该命令将挂载主机的 /home/syx/dockerfile 目录到容器内的 /data 目录上.

任何在 /home/syx/dockerfile 目录下的文件都会出现在容器内.这对于在主机和容器之间共享文件是非常有用的,例如挂载需要编译的源代码.

为了保证可移植性,挂载主机目录不需要从 Dockerfile 指定.当使用 -v 参数时,镜像目录下的任何文件都不会被复制到 Volume 中.

## 数据共享

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

## 数据容器

常见的使用场景是使用纯数据容器来持久化数据库,配置文件或者数据文件等.例如:

```bash
$docker run --name dbdate postgres echo “Data-Only container for postgres”
```

该命令将会创建一个已经包含在 Dockerfile 里定义过 Volume 的 postgres 镜像,运行 echo 命令然后退出.当我们运行 docker ps 命令时,echo 可以帮助我们识别某镜像的用途.我们可以用 -volume-from 命令来识别其他容器的 Volume:

```bash
$docker run -d --volumes-from dadate --name db1 postgres  
```

**使用数据容器的两个注意点:**

1. 不要运行数据容器,这纯属是在两非自愿
2. 不要为了数据容器而使用”最小的镜像”,如 busybox 或 scratch,只使用数据库镜像本身就可以了.

> 你已经拥有了该镜像,所以不需要占用额外的空间.

## 备份

如果你在用数据容器,那做备份是相当容易的.

```bash
root@syx-VB:~# docker run --rm --volumes-from dbdate -v $(pwd):/backup ubuntu tar cvf /backup/backup.tar /var/lib/postgresql/data
```

该命令会将 Volume 里所有的东西压缩为一个 tar 包(官方的 postgres Dockerfile 在 /var/lib/postgresql/data 目录下定义了一个 Volume).

```bash
root@syx-VB:~# ls  
backup.tar  
```

## 删除Volumes

这个功能太重要了,如果你已经使用 docker run 来删除你的容器,那可能会有很多孤立的 Volume 仍在占用着空间.

**Voulume 可以被删除的条件:**

1. 该容器可以用 docker rm -v来删除且没有其他容器连接到该 Volume (以及主机目录是也没被指定为 Volume ).注意, -v 是必不可少的.
2. docker run 中使用 rm 参数.

# 二、进阶篇

> 一开始,认为 Volume 是用来持久化的,但是这实际上不对,因为认为 Volume 是用来持久化的同学一定是认为容器无法持久化,所以有了 Volume 来帮助容器持久化,事实上,容器会一直存在,除非你删除他们.

容器是持久的,直到你删除他们,并且你只能这么做:

```bash
$docker rm my_contariner
```

如果你没有执行此命令,那么你的容器会一直存在,依旧可以启动,停止等.如果你找不到容器,可以运行

```bash
$docker ps -a
```

Docker ps 只能显示正在运行的容器,但是容器也会处于停止状态,这种情况下,上面的命令会显示所有的容器,无论他们处于什么状态. docker run... 命令可以有很多的组合(它提供了 Docker 容器从创建到启动的所有功能),它会创建一个新的容器,然后启动它.

**所以说,Volume 不是为了持久化.**

## 什么是 Voume

Volume 可以将容器以及容器缠身的数据分离开来,这样的话,当你使用 `docker rm my_container` 删除容器时,不会影响相关的数据.

Volume 可以使用下面两种方式创建:

1. 在 Dockerfile 指定 VOLUME /some/dir
2. 执行 `docker run -v /some/dir` 命令指定

无论哪种方式都是做了同样的事情.

他们告诉 Docker 在主机上创建一个目录(默认情况下是在 /var/lib/docker ),然后将其挂载到指定的路径(本例中是: /some/dir ).

当删除使用该 Volume 的容器时,Volume 本身不会受到影响,它可以一直存在下去.

如果在容器中不存在指定的目录,那么该目录将会被自动创建.

你可以告诉 Docker 同时删除容器和 Volume:

```bash
$docker rm -v my_container
```

有时候,你想在容器中使用主机上的某个目录,你可以通过其他的参数来指定:

```bash
$docker run -v /host/path:/some/path...
```

这就明确的告诉 Docker 使用指定的主机路径来代替 Docker 床架你的根路径并挂载到容器内指定的路径(上例中是 /some/dir ).

> 需要注意的是,这种方式同样支持问文件.在 Docker 术语中,这通常被称为bind-mounts.如果主机上的路径不存在,目录将自动在给定的路径中创建.

容器也可以与其他容器共享 Volume:

```bash
$docker run --name my_container -v /some/path ...  
$docker run --volumes-from my_container --name my_contaner2 ...  
```

上面你的案例将告诉 Docker 从第一个容器挂载相同的 Volume 到第二个容器,它可以在两个容器之间共享数据.

- 如果你执行 `docker rm -v my_container` 命令,而上面的第二容器依然存在,那 Volume 就不会删除,
- 如果你不使用 `docker rm -v my_container2` 删除第二个容器,那么这个 Volume 就是一直存在.

**VOLUME 实际上就是在本地新建了一个文件夹挂载了，那么实际上容器内部的文件夹有三种情况:**

1. 没有指定 VOLUME 也没有指定 -v，这种是普通文件夹。
2. 指定了 VOLUME 没有指定 -v，这种文件夹可以在不同容器之间共享，但是无法在本地修改。
3. 指定了 -v 的文件夹，这种文件夹可以在不同容器之间共享，且可以在本地修改。

> 原文链接：
- <https://www.cnblogs.com/ilinuxer/p/6613904.html>
- <http://www.cnblogs.com/ilinuxer/p/6613910.html>
