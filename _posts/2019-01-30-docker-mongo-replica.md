---
layout: post
title: docker 部署 MongoDB 副本集
categories: docker,mongodb
description: some word here
keywords: docker,mongodb,replica
---

> 某些情况下，MongodB 副本可以提供更高的读取容量，就像客户端可以发送读操作到不同的服务器。在不同数据中心维护数据副本可以增加分布式应用的数据局部性和可用性。还可以因为其它目的保存额外的副本，比如灾难恢复、报告或备份。

# 副本集概述

## MongoDB 中的副本

1. 一个副本集就是一组维护相同数据集的 mongod 实例。

2. 一个数据集包含一些数据承载节点和一个可选的仲裁节点。数据承载节点中，有且只能有一个被认为是主承载节点，而其它节点被认为是次要节点。主节点接收所有写入操作。主节点将对其数据集所做的所有更改记录到其 oplog。

3. 次要节点复制主节点的 oplog 并将操作应用到其数据集，就跟次要数据集反映了主数据集一样。如果主节点不可用，一个合格的次要节点将被选为新的主节点。

4. 可以添加一个额外的 mongod 实例到副本集中来作为仲裁节点。仲裁节点不维护数据集。仲裁节点通过响应副本集其它成员的心跳和选举请求来达到维护副本集中法定成员数量的目的。因为它不需要存储数据集，比带有数据集的全功能副本集成员消耗更少的资源，所以它是一种提供副本集仲裁功能比较好的方式。如果你的副本集有偶数个成员，添加一个仲裁节点在主节点选举中来获得一个主节点的选票。仲裁节点不需要专用硬件。

5. 仲裁节点将永远是仲裁节点，但主节点可能变为次要节点，次要节点也可能通过选举成为主要节点。

## 异步复制

次要节点异步的应用来自主节点的操作。通过在主节点之后应用操作，副本集可以不管一个或多个成员的失败而继续实现其功能。

## 自动故障切换

- 当主节点超过十秒没有与副本集的其它成员通信，一个合格的次要节点将举行选举来将它自己选举为新的主节点。第一个次要节点举行选举，并在接收多数成员的投票后成为主节点。
- MongoDB 3.2 版本后引入了版本 1 的复制协议，它减少了副本集的故障切换时间并加速了多个同时存在的主节点的检测。
- 虽然时间不同，故障转移过程一般在一分钟之内完成。

## 读操作

- 默认的，客户端从主节点进行读操作，然而可以通过指定读偏好来将读操作发送到次要节点。次要节点的异步复制意味着从次要节点进行的读操作可能返回没有反映主节点上数据状态的数据。
- MongoDB 中，客户端可以在写操作持久化之前看到其结果。

## 额外的特性

副本集提供多种支持应用需求的选项。可以部署成员在多个数据中心的副本集，或通过调整一些成员的 members[n].priority 来控制选举主节点的结果。副本集也支持专用于报告、灾难恢复或备份功能的成员。

## 副本集成员

- 主节点接收所有写操作。
- 次要节点从主节点复制操作来保持与主节点相同的数据集。次要节点可以有特殊配置文件的附加配置。比如，次要节点可以对选举弃权或优先级为 0。
- 副本集的推荐最低配置是带有三个数据承载点三成员副本集：一个主节点和两个次要节点。也可以部署为两个数据承载点的三成员副本集：一个主节点，一个次要节点和一个仲裁节点。但这种模式不如三数据承载点的模式冗余性好。
- 3.0 版本的变化：副本集最多可以有 50 名成员，但只有 7 名投票成员。

## 主节点

主节点是副本集中接收写操作的唯一成员。MongoDB 在主节点上应用写操作，然后将操作记录在主节点的 oplog 上。次要节点复制 oplog 并将操作应用到它们的数据集上。

## 次要节点

- 次要节点维护主节点数据集的一个副本。复制数据时，次要节点在异步过程中从主节点的 oplog 应用操作到它自己的数据集。副本集可以有一个或多个次要节点。

- 次要节点可以通过配置来达到实现一些效果：
  - 防止被选举为主节点，以便允许它留在次要节点数据中心或作为冷备用。
  - 防止应用从中读取，以便允许它运行需要与正常流量分离的应用。
  - 保存一个运行的历史快照，以便从某些错误中恢复，如误删数据库。

## 仲裁节点

- 仲裁节点没有数据集的副本，并且不能变成主节点。副本集可以有仲裁节点来在选举主节点中投票。仲裁节点永远有一张选票，这样就允许副本集的投票成员数量不均等，而无需承担一个复制数据的成员的额外开销。
> 重要：不要在还承载副本集的主节点或次要节点的系统上运行仲裁节点。

## 副本集 oplog

- oplog：操作日志，一个特殊的固定集合，保持所有修改数据库上数据的操作的滚动记录。
- 副本集的所有成员都包含一份 oplog 的副本，在 local.oplog.rs 中，这允许它们维护数据库的当前状态。
- 为了减轻复制的困难，所有的副本集成员成员都发送心跳（ping）到所有其它成员。
- 任何成员都可以从其它成员那里导入 oplog。
- oplog 中的每个操作都是幂等（idempotent）的。即，oplog 对目标资料应用不管一次或多次操作，都产生相同的结果。

## oplog 大小

- 当第一次开始一个副本集成员时，MongoDB 以默认大小创建一个 oplog。
- 如果可以预料副本集工作量的大小，可以将 oplog 设置为比默认值大些。相反，如果应用主要用来执行读操作和少量写操作，一个小的 oplog 可能就足够了。
- 以下情况可能需要较大的 oplog：
  - 一次更新多个文档
  - 删除与插入量大致相同的数据时
  - 大量的就地更新

## oplog 状态

使用 rs.printReplicationInfo() 方法来查看 oplog 状态，包括其大小和操作的时间范围。

# 二、实战：在 docker 中创建副本集

## 准备工作

- 安装 docker <https://docs.docker.com/install/linux/docker-ce/centos/>
- 拉取 mongo 镜像，如有需求可加上版本号

```bash
# 创建工作目录
mkdir -p /server/docker-compose/mongo-replica && cd /server/docker-compose/mongo-replica

# 拉取镜像
docker pull mongo
```

## 概览

1. 我们要从 mongo 镜像创建三个容器，都处在它们的 docker 容器网络内。
2. 这三个容器将被命名为 mongo1、mongo2 和 mongo3。它们将作为副本集的三个 mongo 实例。
3. 暴露它们的端口到本地机器，以便可以从本地机器的 mongo shell 来访问它们中的任意一个。
4. 这三个 mongo 容器中的每一个都应该能与这个网络中的所有其它容器通信。

## 建立网络

1.创建一个名为 my-mongo-cluster 的网络：

```bash
docker network create my-mongo-cluster
```

2.查看当前系统中的网络：

```bash
docker network ls
```

## 创建容器

运行以下命令启动第一个容器：

```bash
docker run \
-dit \
-p 30001:27017 \
--name mongo1 \
--net my-mongo-cluster \
mongo mongod --replSet my-mongo-set
```

> - docker run: 从镜像启动容器
> - dit : 后台运行
> - -p 30001:27017：暴露容器中的 27017 端口，映射到本机的 30001 端口
> - --name mongo1：给这个容器命名为 mongo1
> - --net my-mongo-cluster：将此容器添加到 my-mongo-cluster 网络
> - mongo：用来生成此容器的镜像名称
> - mongod --replSet my-mongo-set：执行 mongod 命令，以将此 mongo 实例添加到名称为 my-mongo-se 的副本集。

启动其余两个容器：

```bash
docker run \
-dit \
-p 30002:27017 \
--name mongo2 \
--net my-mongo-cluster \
mongo mongod --replSet my-mongo-set

docker run \
-dit \
-p 30003:27017 \
--name mongo3 \
--net my-mongo-cluster \
mongo mongod --replSet my-mongo-set
```

## 配置副本集

现在我们需要的 mongo 实例已经运行起来了，现在来把它们编程副本集。

### 1. 这条命令将在运行的容器 mongo1 中打开 mongo shell：

```bash
docker exec -it mongo1 mongo
```

### 2. 在 mongo shell 中进行配置：

```mongo
> db = (new Mongo('localhost:27017')).getDB('test')
test
> config = {
     "_id" : "my-mongo-set",
     "members" : [
         {
             "_id" : 0,
             "host" : "mongo1:27017"
         },
         {
             "_id" : 1,
             "host" : "mongo2:27017"
         },
         {
             "_id" : 2,
             "host" : "mongo3:27017"
         }
     ]
  }
```

> - 第一个 _id 键应当和 —replSet 标签为 mongo 实例设置的值一样，这个例子中是 my-mongo-set。
> - 接下来列出了所有想放到副本集中的成员。
> - 将所有 mongo 实例添加到 docker 网络。在 my-mongo-cluster 网络中根据每个容器的名称各自分配到 ip 地址。

### 3. 通过以下命令启动副本集：

```mongo
rs.initiate(config)
```

如果一切顺利，提示符将变成这样：(需手动随便敲命令触发下)

```mongo
my-mongo-set:PRIMARY>
```

> 这意味着 shell 现在与 my-mongo-set 集群中的 PRIMARY 数据库进行了关联。

### 4. 下面测试副本集是否工作，先在 primary 数据库中插入数据：

```mongo
> db.mycollection.insert({name : 'sample'})
WriteResult({ "nInserted" : 1 })
> db.mycollection.find()
{ "_id" : ObjectId("57761827767433de37ff95ee"), "name" : "sample" }

```

### 5. 然后新建一个与 secondary 数据库的连接，并测试文档是否在那里复制：

```mongo
> db2 = (new Mongo('mongo2:27017')).getDB('test')
test
> db2.setSlaveOk()
> db2.mycollection.find()
{ "_id" : ObjectId("57761827767433de37ff95ee"), "name" : "sample" }
```

> 执行 db2.setSlaveOk() 命令来让 shell 知道我们故意在查询非 primary 的数据库。

# 三、高级篇：采用 docker-compose 全自动配置

## 1. 创建工作目录

```bash
mkdir -p /server/docker-compose/mongo-replica/{scripts,data}

cd /server/docker-compose/mongo-replica
```

## 2. 生成 setup.sh 自动配置脚本

```bash
$ cat > scripts/setup.sh <<EOF
#!/bin/bash

mongodb1=`getent hosts ${MONGO1} | awk '{ print $1 }'`
mongodb2=`getent hosts ${MONGO2} | awk '{ print $1 }'`
mongodb3=`getent hosts ${MONGO3} | awk '{ print $1 }'`

port=${PORT:-27017}

echo "Waiting for startup.."
until mongo --host ${mongodb1}:${port} --eval 'quit(db.runCommand({ ping: 1 }).ok ? 0 : 2)' &>/dev/null; do
  printf '.'
  sleep 1
done

echo "Started.."

echo setup.sh time now: `date +"%T" `
mongo --host ${mongodb1}:${port} <<EOF
   var cfg = {
        "_id": "${RS}",
        "protocolVersion": 1,
        "members": [
            {
                "_id": 0,
                "host": "${mongodb1}:${port}"
            },
            {
                "_id": 1,
                "host": "${mongodb2}:${port}"
            },
            {
                "_id": 2,
                "host": "${mongodb3}:${port}"
            }
        ]
    };
    rs.initiate(cfg, { force: true });
    rs.reconfig(cfg, { force: true });
EOF

```

## 3. 生成 docker-compose.yml 文件

```bash
$ cat > docker-compose.yml <<EOF
version: '3'
services:
  mongo2:
    container_name: "mongo2"
    image: mongo:4.1.7
    ports:
      - "30012:27017"
    command: mongod --replSet vision-set  --bind_ip 0.0.0.0
    restart: always

  mongo3:
    container_name: "mongo3"
    image: mongo:4.1.7
    ports:
      - "30013:27017"
    command: mongod --replSet vision-set  --bind_ip 0.0.0.0
    restart: always

  mongo1:
    container_name: "mongo1"
    image: mongo:4.1.7
    ports:
      - "30011:27017"
    command: mongod --replSet vision-set --bind_ip 0.0.0.0
    links:
      - mongo2:mongo2
      - mongo3:mongo3
    restart: always

  mongo-vision-set-setup:
    container_name: "mongo-vision-set-setup"
    image: mongo:4.1.7
    depends_on:
      - "mongo1"
      - "mongo2"
      - "mongo3"
    links:
      - mongo1:mongo1
      - mongo2:mongo2
      - mongo3:mongo3
    volumes:
        - ./scripts:/scripts
    environment:
      - MONGO1=mongo1
      - MONGO2=mongo2
      - MONGO3=mongo3
      - RS=vision-set
    entrypoint: [ "/scripts/setup.sh" ]
EOF

# 检查配置
docker-compose config

# 启动
docker-compose up -d

# 查看日志
docker-compose logs -f

```

## 4. 检查 mongo replica 配置是否成功

```bash
# 进入 mongo1 如果提示符变为 primary，则初始化基本成功
docker exec -it mongo1 mongo

vision-set:PRIMARY>

# 先在 primary 数据库中插入数据：

vision-set:PRIMARY> db.mycollection.insert({name : 'sample'})
WriteResult({ "nInserted" : 1 })
vision-set:PRIMARY> db.mycollection.find()
{ "_id" : ObjectId("57761827767433de37ff95ee"), "name" : "sample" }

# 然后新建一个与 secondary 数据库的连接，并测试文档是否在那里复制：

vision-set:PRIMARY> db2 = (new Mongo('mongo2:27017')).getDB('test')
test
vision-set:PRIMARY> db2.setSlaveOk()
vision-set:PRIMARY> db2.mycollection.find())
{ "_id" : ObjectId("57761827767433de37ff95ee"), "name" : "sample" }
```

> 执行 db2.setSlaveOk() 命令来让 shell 知道我们故意在查询非 primary 的数据库。

# 附：mongodb mongod 启动参数

> 我们可以通过 mongod --help 查看 mongod 的所有参数说明，以下是各参数的中文解释。

## 基本配置

```bash
–quiet
# 安静输出

–port arg
# 指定服务端口号，默认端口27017

–bind_ip arg
# 绑定服务IP，若绑定127.0.0.1，则只能本机访问，不指定默认本地所有IP

–logpath arg
# 指定MongoDB日志文件，注意是指定文件不是目录

–logappend
# 使用追加的方式写日志

–pidfilepath arg
# PID File 的完整路径，如果没有设置，则没有PID文件

–keyFile arg
# 集群的私钥的完整路径，只对于Replica Set 架构有效

–unixSocketPrefix arg
# UNIX域套接字替代目录,(默认为 /tmp)

–fork
# 以守护进程的方式运行MongoDB，创建服务器进程

–auth
# 启用验证

–cpu
# 定期显示CPU的CPU利用率和iowait

–dbpath arg
# 指定数据库路径

–diaglog arg
# diaglog选项 0=off 1=W 2=R 3=both 7=W+some reads

–directoryperdb
# 设置每个数据库将被保存在一个单独的目录

–journal
# 启用日志选项，MongoDB的数据操作将会写入到journal文件夹的文件里

–journalOptions arg
# 启用日志诊断选项

–ipv6
# 启用IPv6选项

–jsonp
# 允许JSONP形式通过HTTP访问（有安全影响）

–maxConns arg
# 最大同时连接数 默认2000

–noauth
# 不启用验证

–nohttpinterface
# 关闭http接口，默认关闭27018端口访问

–noprealloc
# 禁用数据文件预分配(往往影响性能)

–noscripting
# 禁用脚本引擎

–notablescan
# 不允许表扫描

–nounixsocket
# 禁用Unix套接字监听

–nssize arg (=16)
# 设置信数据库.ns文件大小(MB)

–objcheck
# 在收到客户数据,检查的有效性，

–profile arg
# 档案参数 0=off 1=slow, 2=all

–quota
# 限制每个数据库的文件数，设置默认为8

–quotaFiles arg
# number of files allower per db, requires –quota

–rest
# 开启简单的rest API

–repair
# 修复所有数据库run repair on all dbs

–repairpath arg
# 修复库生成的文件的目录,默认为目录名称dbpath

–slowms arg (=100)
# value of slow for profile and console log

–smallfiles
# 使用较小的默认文件

–syncdelay arg (=60)
# 数据写入磁盘的时间秒数(0=never,不推荐)

–sysinfo
# 打印一些诊断系统信息

–upgrade
# 如果需要升级数据库
```

## Replicaton 参数

```bash
–fastsync
# 从一个dbpath里启用从库复制服务，该dbpath的数据库是主库的快照，可用于快速启用同步

–autoresync
# 如果从库与主库同步数据差得多，自动重新同步，

–oplogSize arg
# 设置oplog的大小(MB)
```

## 主/从参数

```bash
–master
# 主库模式

–slave
# 从库模式

–source arg
# 从库 端口号

–only arg
# 指定单一的数据库复制

–slavedelay arg
# 设置从库同步主库的延迟时间
```

## Replica set(副本集)选项

```bash
–replSet arg
# 设置副本集名称
```

## Sharding(分片)选项

```bash
–configsvr
# 声明这是一个集群的config服务,默认端口27019，默认目录/data/configdb

–shardsvr
# 声明这是一个集群的分片,默认端口27018

–noMoveParanoia
# 关闭偏执为moveChunk数据保存?
```

**示例：**

```bash
./mongod -shardsvr -replSet shard1 -port 16161 -dbpath /data/mongodb/data/shard1a -oplogSize 100 -logpath /data/mongodb/logs/shard1a.log -logappend -fork -rest

# 上述参数都可以写入 mongod.conf 配置文档里例如：

dbpath = /data/mongodb
logpath = /data/mongodb/mongodb.log
logappend = true
port = 27017
fork = true
auth = true
```


> 参考链接：
> - <https://www.jianshu.com/p/ec53891d638b>
> - <https://github.com/senssei/mongo-cluster-docker>