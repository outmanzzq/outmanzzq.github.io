---
layout: post
title: consul 集群搭建与 Golang 服务发现示例
categories: consul,traefik,cluster,go
description: consul 集群搭建与 Golang 服务发现示例
keywords: consul,traefik,cluster,go
---

> 传统单机应用动态性不强，不会频繁地更新和重新发布，也较少地进行自动伸缩。但随着互联网分布式系统的普及，服务与服务之间的伸缩性和可扩展性的要求也越来越大。

 为了满足服务的垂直和水平的扩张，以往一般使用预定义的端口配置服务，当新的服务需要上线或当期服务需要冗余扩展的时候，我们需要静态化地“注册”相关 ip 与端口信息到一个地方，再通过程序之间定时“更新”的方法去同步信息。

 但这种手段问题是非常多的，例如我们需要连接 kafka 的 master 的时候，如果服务信息发生变更，客户端是很难知道的。

 其中一个简单粗暴的方法是配置 hosts 文件，使用预定义的“域名”作为服务的连接依据，但这样也是麻烦的，服务变更的时候，要手动更改 hosts 文件，当服务器上的服务相对多的情况下，维护量就想当恐怖了。

其实就一句话：服务多，问题多

> 服务发现不是万金油，任务服务提供方如果挂掉，其实很难做到百分百实时通知到调用方的，依然需要通过合理的架构或容灾手段，提高整体服务的可用性。

在微服务化的趋势下，为了最大限度增加扩容缩容的灵活性，名字服务和服务发现等方式就越来越受到青睐了。

目前，主流的服务发现组件有：consul、etcd、zookeeper,其中的区别这里就不展开说明了，有兴趣的可以阅读一下 consul 官方的介绍： [《Consul vs. ZooKeeper, doozerd, etcd》](https://www.consul.io/intro/vs/zookeeper.html)

本文主要目的是通过学习 consul 的原理，动手搭建集群方案，实现服务注册与发现的功能，让我们对微服务架构的名字服务有一个清晰的印象。

# consul服务架构和核心概念

下图是 consul 以数据中心维度的架构图，图中的 SERVER 是 consul 服务端高可用集群，CLIENT 是 consul 客户端。consul 客户端不保存数据，客户端将接收到的请求转发给响应的 Server 端。Server 之间通过局域网或广域网通信实现数据一致性。每个 Server 或Client 都是一个 consul agent，或者说 server 和 client 只是 agent 所扮演的不同角色罢了。

![ ](/images/20190826-consul-cluster.png)

从上图可知，consul 使用 gossip 协议管理成员关系、广播消息到整个集群。

Consul 利用两个不同的 gossip pool。我们分别把他们称为局域网池(LAN Gossip Pool)或广域网池(WAN Gossip Pool)。每个 Consul 数据中心（ Datacenter ）都有一个包含所有成员（ Server和Client ）的LAN gossip pool。

LAN Pool有如下目的：

- 成员关系允许 Client 自动发现 Server 节点，减少所需的配置量

- 分布式故障检测允许的故障检测的工作在某几个 Server 几点执行，而不是集中整个集群所有节点上

- gossip 允许可靠和快速的事件广播，比如 Leader 选举

WAN Pool 是全局唯一的，无论属于哪一个数据中心，所有 Server 应该加入到 WAN Pool。由 WAN Pool 提供会员信息让 Server 节点可以执行跨数据中心的请求。也就是说这个 Pool 不同于 LAN Pool，它的目的是为了允许数据中心能够以 low-touch 的方式发现彼此。当数据中心的 server 收到来自不同数据中心的请求时，它可转发请求到数据中心的 leader。

在每个数据中心，client 和 server 是混合的。一般建议有 3-5 台 server。这是基于有故障情况下的可用性和性能之间的权衡结果，因为越多的机器加入达成共识越慢。然而，并不限制 client 的数量，它们可以很容易的扩展到数千或者数万台。

# consul 集群搭建

## 环境准备

为了学习 Consul 提供的服务实现服务的注册与发现，我们需要建立 Consul Cluster 的测试集群方案，要搭建集群，其实很容易，每台服务器只要启动 consul agent 的命令，然后按照角色划分好 server 和 client 即可。这边我们的测试部署计划是一个 datacenter 里包含3 个 server 和 1 个 client 的架构。

为了演示集群的搭建，我们准备四台测试的服务器：

环境 | 服务器 IP | 节点名称 | 角色
-|-
centos 7.2 | 172.17.0.3 | s1 | server
centos 7.2 | 172.17.0.4 | s2 | server
centos 7.2 | 172.17.0.5 | s3 | server
centos 7.2 | 172.17.0.6 | s4 | client

分别在每个服务器上下载 consul 的软件包：

> 本例子使用的 consul 是 1.1.0 版本

```bash
wget https://releases.hashicorp.com/consul/1.1.0/consul_1.1.0_linux_amd64.zip

unzip consul_1.1.0_linux_amd64.zip
```

解压后，可以看到只有一个 consul 的二进制文件，我们移动到 /usr/local/bin 目录下：

```cp consul /usr/local/bin/```

接着通过执行命令检查 consul 是否安装成功：

```bash
consul -h

Usage: consul [--version] [--help] <command> [<args>]

Available commands are:
    agent          Runs a Consul agent
    catalog        Interact with the catalog
    event          Fire a new event
    exec           Executes a command on Consul nodes
    force-leave    Forces a member of the cluster to enter the "left" state
    info           Provides debugging information for operators.
    join           Tell Consul agent to join cluster
    keygen         Generates a new encryption key
    keyring        Manages gossip layer encryption keys
    kv             Interact with the key-value store
    leave          Gracefully leaves the Consul cluster and shuts down
    lock           Execute a command holding a lock
    maint          Controls node or service maintenance mode
    members        Lists the members of a Consul cluster
    monitor        Stream logs from a Consul agent
    operator       Provides cluster-level tools for Consul operators
    reload         Triggers the agent to reload configuration files
    rtt            Estimates network round trip time between nodes
    snapshot       Saves, restores and inspects snapshots of Consul server state
    validate       Validate config files/directories
    version        Prints the Consul version
    watch          Watch for changes in Consul
```

每个集群测试服务器安装consul后，我们可以开始集群的搭建。

### s1运行consul服务

consul执行命令如下：

```bash
consul agent -server -bootstrap-expect 1 -data-dir /data/consul/ -node=s1 -bind=172.17.0.3  -rejoin -config-dir=/etc/consul.d/ -client 0.0.0.0
```

上面的命令中，我们会使用到几个重要的参数：

- server ： 定义 agent 运行在 server 模式，如果是 client 模式则不需要添加这个参数

- bootstrap-expect ：datacenter 中期望提供的 server 节点数目，当该值提供的时候，consul 一直等到达到指定 sever 数目的时候才会引导（启动）整个集群，为了测试演示，我们这里使用 1

- bind：该地址用来在集群内部的通讯，集群内的所有节点到地址都必须是可达的，默认是 0.0.0.0

- node：节点在集群中的名称，在一个集群中必须是唯一的，默认是该节点的主机名

- rejoin：使 consul 忽略先前的离开，在 agent 再次启动后仍旧尝试加入集群中。也就是说如果不加入这个参数，当前节点一旦退出，下次重启后是不会自动加入到集群中去的，除非是手动触发 consul join xxxx ，所以为了降低重启后对本身服务的影响，这里统一使用 -rejoin 参数。

- config-dir：配置文件目录，里面文件统一规定是以 .json 结尾才会被自动加载并读取服务注册信息的

- client：consul 服务侦听地址，处于 client mode 的 Consul agent 节点比较简单，无状态，仅仅负责将请求转发给 Server agent 节点

当服务启动后，后台输出类似以下日志信息：

```bash
$ consul agent -server -bootstrap-expect 1 -data-dir /data/consul/ -node=s1 -bind=172.17.0.3  -rejoin -config-dir=/etc/consul.d/ -client 0.0.0.0

BootstrapExpect is set to 1; this is the same as Bootstrap mode.
bootstrap = true: do not enable unless necessary
==> Starting Consul agent...
==> Consul agent running!
           Version: 'v1.1.0'
           Node ID: 'cccf0fec-2fd9-30d0-9105-8dd2ae6480d6'
         Node name: 's1'
        Datacenter: 'dc1' (Segment: '<all>')
            Server: true (Bootstrap: true)
       Client Addr: [0.0.0.0] (HTTP: 8500, HTTPS: -1, DNS: 8600)
      Cluster Addr: 172.17.0.3 (LAN: 8301, WAN: 8302)
           Encrypt: Gossip: false, TLS-Outgoing: false, TLS-Incoming: false

==> Log data will now stream in as it occurs:

    2018/05/31 03:54:36 [INFO] raft: Initial configuration (index=1): [{Suffrage:Voter ID:cccf0fec-2fd9-30d0-9105-8dd2ae6480d6 Address:172.17.0.3:8300}]
    2018/05/31 03:54:36 [INFO] raft: Node at 172.17.0.3:8300 [Follower] entering Follower state (Leader: "")
    2018/05/31 03:54:36 [INFO] serf: EventMemberJoin: s1.dc1 172.17.0.3
    2018/05/31 03:54:36 [WARN] serf: Failed to re-join any previously known node
    2018/05/31 03:54:36 [INFO] serf: EventMemberJoin: s1 172.17.0.3
    2018/05/31 03:54:36 [WARN] serf: Failed to re-join any previously known node
    2018/05/31 03:54:36 [INFO] consul: Adding LAN server s1 (Addr: tcp/172.17.0.3:8300) (DC: dc1)
    2018/05/31 03:54:36 [INFO] consul: Handled member-join event for server "s1.dc1" in area "wan"
    2018/05/31 03:54:36 [INFO] agent: Started DNS server 0.0.0.0:8600 (udp)
    2018/05/31 03:54:36 [INFO] agent: Started DNS server 0.0.0.0:8600 (tcp)
    2018/05/31 03:54:36 [INFO] agent: Started HTTP server on [::]:8500 (tcp)
    2018/05/31 03:54:36 [INFO] agent: started state syncer
    2018/05/31 03:54:43 [ERR] agent: failed to sync remote state: No cluster leader
    2018/05/31 03:54:45 [WARN] raft: Heartbeat timeout from "" reached, starting election
    2018/05/31 03:54:45 [INFO] raft: Node at 172.17.0.3:8300 [Candidate] entering Candidate state in term 3
    2018/05/31 03:54:45 [INFO] raft: Election won. Tally: 1
    2018/05/31 03:54:45 [INFO] raft: Node at 172.17.0.3:8300 [Leader] entering Leader state
    2018/05/31 03:54:45 [INFO] consul: cluster leadership acquired
    2018/05/31 03:54:45 [INFO] consul: New leader elected: s1
    2018/05/31 03:54:45 [INFO] agent: Synced node info
```

我们查看一下集群成员组成信息：

```bash
$ consul members

[root@f147f8eb5593 /]# consul members
Node  Address          Status  Type    Build  Protocol  DC   Segment
s1    172.17.0.3:8301  alive   server  1.1.0  2         dc1  <all>
```

命令输出信息中可以看到当前的集群中只有 Node 名称为s1这一个成员，它的状态是 alive,代表是正常运行的。dc1 是默认的 datacenter 名称；Type 会列出成员的角色，当前是 server；Address 中 8301 这个端口是 agent 默认的 LAN 端口（ WAN 端口默认是 8300 ）; Protocol 只是 consul 集群不同 agent 版本兼容性设计的，不同版本的 agent, protocol 的值可能不一样，这个我们先不管。

我们先回到运行日志中，我们发现有这么一句话 :

```New leader elected: s1```

这表明目前集群启动后，在只有一个成员的情况下，也会发生选举，s1自动当选。

为了确认，我们可以使用 consul 的 api 进行 leader 信息的查询，加以确认：

```bash
curl http://localhost:8500/v1/status/leader

"172.17.0.3:8300"

```

除了 consul members 命令外，也可以通过官方的 api 查询集群内的成员信息（目前只有自己这一个成员）:

```bash
curl http://localhost:8500/v1/status/peers

["172.17.0.3:8300"]
```

经过上面的工夫，我们的集群的第一个成员算是部署完成了，不要让它停下来，我们继续往集群添加不同的成员：）

### s2 运行 consul 服务

跟 s1 类似，s2 的 consul 执行命令如下：

```bash
consul agent -server  -data-dir /data/consul/ -node=s2 -bind=172.17.0.4  -rejoin -config-dir=/etc/consul.d/ -client 0.0.0.0
```

当我们执行完上面的命令的时候，我们在 s2 的机器上查看下集群信息：

```bash
//172.17.0.4 s2

$ consul members

Node  Address          Status  Type    Build  Protocol  DC   Segment
s2    172.17.0.4:8301  alive   server  1.1.0  2         dc1  <all>
```

这个时候，显示 s2 所在的集群中只有它自己本身，并没有发现之前我们启动的 s1

我们回到 s1 那边看下成员信息：

```bash
//172.17.0.3 s1

$ consul members

Node  Address          Status  Type    Build  Protocol  DC   Segment
s1    172.17.0.3:8301  alive   server  1.1.0  2         dc1  <all>
```

从上面的现象可以了解到，s2 启动后，并未自动加入之前的 cluster,这个时候，我们需要手动对 s2 的 consul agent 进行加入：

```bash
//172.17.0.4 s2

$ consul join 172.17.0.3

successfully joined cluster by contacting 1 nodes.
```

这样，无论s1或者是 s2,当我们再次查看成员信息的时候，双方都分别发现到对方了

```bash
$ consul members

Node  Address          Status  Type    Build  Protocol  DC   Segment
s1    172.17.0.3:8301  alive   server  1.1.0  2         dc1  <all>
s2    172.17.0.4:8301  alive   server  1.1.0  2         dc1  <all>
```

还记得我们命令的 -rejoin 参数吗？ 我们测算一下当我们停止 s2 再启动 s2，它会不会重新自动加入到之前的集群中去

我们在 s2 中执行以下命令：

```bash
//172.17.0.4 s2

$ consul leave

Graceful leave complete
```

观察 s1 的成员信息：

```bash
$ consul members

Node  Address          Status  Type    Build  Protocol  DC   Segment
s1    172.17.0.3:8301  alive   server  1.1.0  2         dc1  <all>
s2    172.17.0.4:8301  left    server  1.1.0  2         dc1  <all>
```

这时，s2 的状态已经是离线状态（left）， 我们再重写“拉起”它后，s2 的日志显示：

```bash
2018/05/31 05:48:03 [INFO] serf: Attempting re-join to previously known node: s1.dc1: 172.17.0.3:8302
2018/05/31 05:48:03 [INFO] agent: Started DNS server 0.0.0.0:8600 (udp)
2018/05/31 05:48:03 [INFO] serf: Attempting re-join to previously known node: s1: 172.17.0.3:8301
2018/05/31 05:48:03 [INFO] consul: Adding LAN server s2 (Addr: tcp/172.17.0.4:8300) (DC: dc1)
2018/05/31 05:48:03 [INFO] agent: Started DNS server 0.0.0.0:8600 (tcp)
2018/05/31 05:48:03 [INFO] agent: Started HTTP server on [::]:8500 (tcp)
2018/05/31 05:48:03 [INFO] agent: started state syncer
2018/05/31 05:48:03 [INFO] serf: EventMemberJoin: s1.dc1 172.17.0.3
2018/05/31 05:48:03 [INFO] serf: Re-joined to previously known node: s1.dc1: 172.17.0.3:8302
2018/05/31 05:48:03 [INFO] serf: EventMemberJoin: s1 172.17.0.3
2018/05/31 05:48:03 [INFO] serf: Re-joined to previously known node: s1: 172.17.0.3:8301
2018/05/31 05:48:03 [INFO] consul: Adding LAN server s1 (Addr: tcp/172.17.0.3:8300) (DC: dc1)
2018/05/31 05:48:03 [INFO] consul: New leader elected: s1

... ...
```

重启后，可以看到关键的日志：

```Re-joined to previously known node: s1: 172.17.0.3:8301```

表面，节点能自动回到之前的集群中去，紧接着，我们看下成员信息：

```bash
$ consul members

Node  Address          Status  Type    Build  Protocol  DC   Segment
s1    172.17.0.3:8301  alive   server  1.1.0  2         dc1  <all>
s2    172.17.0.4:8301  alive   server  1.1.0  2         dc1  <all>
```

s2 确实是会重新回归到集群中去，rejoin 的效果达到了。

### s3运行consul服务

consul执行命令如下：

```bash
 consul agent -server  -data-dir /data/consul/ -node=s3 -bind=172.17.0.5  -rejoin -config-dir=/etc/consul.d/ -client 0.0.0.0
```

和s2一样，首次启动的时候，记得记得记得要 join 到集群中去：

```consul join 172.17.0.3```

确认成员信息：

```bash
$ consul members
Node  Address          Status  Type    Build  Protocol  DC   Segment
s1    172.17.0.3:8301  alive   server  1.1.0  2         dc1  <all>
s2    172.17.0.4:8301  alive   server  1.1.0  2         dc1  <all>
s3    172.17.0.5:8301  alive   server  1.1.0  2         dc1  <all>
```

到这来为止我们集群中已经成功启动并运行了 3 个 consul server 了，接下来，我们试试不一样的角色节点，我们尝试添加一个 client 角色的 consul agent。

### c1 运行 consul 服务

consul 执行命令如下:

```bash
consul agent  -data-dir /data/consul/ -node=c1 -bind=172.17.0.6  -rejoin -config-dir=/etc/consul.d/ -client 0.0.0.0
```

和 s2、s3 一样，首次启动的时候，记得要 join 到集群中去：

```consul join 172.17.0.3```

查看集群信息：

```bash
$ consul members
Node  Address          Status  Type    Build  Protocol  DC   Segment
s1    172.17.0.3:8301  alive   server  1.1.0  2         dc1  <all>
s2    172.17.0.4:8301  alive   server  1.1.0  2         dc1  <all>
s3    172.17.0.5:8301  alive   server  1.1.0  2         dc1  <all>
c1    172.17.0.6:8301  alive   client  1.1.0  2         dc1  <default>
```

由于 c1 的启动命令中，没有 -server 参数，所以它会默认以 client 的角色运行。

为了完整的测试，我们尝试再测试关闭 leader（s1）的 consul 服务，然后观察选举请求

```bash
memberlist: Suspect s1 has failed, no acks received
2018/05/31 10:17:53 [INFO] memberlist: Suspect s1 has failed, no acks received
2018/05/31 10:17:54 [INFO] memberlist: Marking s1 as failed, suspect timeout reached (2 peer confirmations)
2018/05/31 10:17:54 [INFO] serf: EventMemberFailed: s1 172.17.0.3
2018/05/31 10:17:54 [INFO] consul: removing server s1 (Addr: tcp/172.17.0.3:8300) (DC: dc1)
2018/05/31 10:18:04 [INFO] consul: New leader elected: s3
2018/05/31 10:18:37 [INFO] serf: attempting reconnect to s1 172.17.0.3:8301
2018/05/31 10:19:07 [INFO] serf: attempting reconnect to s1 172.17.0.3:8301


curl http://localhost:8500/v1/status/leader

"172.17.0.5:8300"
```

从上面的日志和 api 返回信息了解到，s1 挂掉后，集群会选举新的 leader 为 s3

这样我们的集群节点部署和主体测试工作就基本可以了，感觉上通过 consul 的集群搭建还算相当轻松的。

# 服务发现示例

## 往 consul 注册一个测试服务

建立 Consul Cluster 目的是实现服务的注册和发现。

Consul 支持两种服务注册的方式：

- 通过 Consul 的服务注册 HTTP API，由服务自身在启动后调用 API 注册自己

- 通过在配置文件中定义服务的方式进行注册。

Consul 文档中建议使用后面一种方式来做服务配置和服务注册，而且配置很简单，只需往直前启动命令中的 --config-dir 指定的目录下新建 json 格式的注册文件即可，举个例子：

s3 上注册（部署）一个服务

```bash
//172.17.0.5 s3

touch /etc/consul.d/web.json

```

然后填写以下注册信息（必须是标准 Json 格式）

```json
{
  "service": {
    "name": "web",
    "tags": ["master"],
    "address": "172.17.0.5",
    "port": 9999,
    "checks": [
      {
        "http": "http://localhost:9999/health",
        "interval": "10s"
      }
    ]
  }
}
```

这个配置就是我们在 s3 节点上配置了一个 web 服务做，这个服务中我们定义了服务的 name、address、port 等，还包含一个 checks 配置用于健康检查，上面我们的示例会每隔 10 秒进行一次 http://localhost:9999/health 请求作为 healthcheck

## c1 上注册（部署）一个服务

同样，我们在 c1 上，也注册一个类似的 web 服务：

```bash
//172.17.0.6 c1

touch /etc/consul.d/web.json
```

填写下面的内容

```json
{
  "service": {
    "name": "web",
    "tags": ["master"],
    "address": "172.17.0.6",
    "port": 9999,
    "checks": [
      {
        "http": "http://localhost:9999/health",
        "interval": "10s"
      }
    ]
  }
}
```

s3 和 c1 都创建完服务定义文件后，我们需要对两者的 consul agent 进行重启，才能起到服务更新的效果。

当 s3 和 c1 的 consul agent 启动后，观察日志信息我们发现两者不约而同报出以下信息：

```bash
2018/05/31 06:50:02 [INFO] agent: Synced service "web"
2018/05/31 06:50:03 [WARN] agent: Check "service:web" HTTP request failed: Get http://localhost:9999/health: dial tcp 127.0.0.1:9999: connect: connection refused
2018/05/31 06:50:03 [INFO] serf: EventMemberLeave: c1 172.17.0.6
2018/05/31 06:50:03 [DEBUG] raft-net: 172.17.0.5:8300 
accepted connection from: 172.17.0.3:45189
2018/05/31 06:50:08 [INFO] serf: EventMemberJoin: c1 172.17.0.6
2018/05/31 06:50:13 [WARN] agent: Check "service:web" HTTP request failed: Get http://localhost:9999/health: dial tcp 127.0.0.1:9999: connect: connection refused
```

这些报错日志很容易理解，因为我们服务中定义了健康检查的请求 API，但目前我们还没提供这个服务，所以一直报 connection refused

## 编写健康监测程序

我们这个 web 服务，为了测试方便，简单地提供两个接口，一个用于健康检查，一个用于测试请求输出

Go 代码如下：

```go
package main

import (
  "fmt"
  "net/http"
  "os"
  "log"
)

var hostname, _ = os.Hostname()

func handler(w http.ResponseWriter, r *http.Request) {
    log.Println("receive a request")
    fmt.Fprintf(w, "result from %s\n", hostname)
}

func healthCheckHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Println("health check")
}

func main() {
    http.HandleFunc("/", handler)
    http.HandleFunc("/health", healthCheckHandler)
    http.ListenAndServe(":9999", nil)
}
```

编译后放到 s3 和 c1 任意目录中去

```go build -o hc```

s3 和 c1 分别后台启动这个 web 服务：

```bash
chmod +x hc

nohup /apps/svr/healthcheck/hc  > /apps/svr/healthcheck/log.out &
```

web 服务启动后，s3 和 c1 的日志会显示：

```2018/05/31 07:02:03 [INFO] agent: Synced check "service:web"```

为了完整查看集群中所提供的服务，可以通过特定的 API：

```bash
$ curl http://localhost:8500/v1/catalog/service/web\?pretty

[
    {
        "ID": "a6d4ea77-47f4-7b7f-480c-dce6f705f091",
        "Node": "c1",
        "Address": "172.17.0.6",
        "Datacenter": "dc1",
        "TaggedAddresses": {
            "lan": "172.17.0.6",
            "wan": "172.17.0.6"
        },
        "NodeMeta": {
            "consul-network-segment": ""
        },
        "ServiceID": "web",
        "ServiceName": "web",
        "ServiceTags": [
            "master"
        ],
        "ServiceAddress": "172.17.0.6",
        "ServiceMeta": {},
        "ServicePort": 10000,
        "ServiceEnableTagOverride": false,
        "CreateIndex": 591,
        "ModifyIndex": 591
    },
    {
        "ID": "7e836265-b197-89fb-e96a-26912ce69954",
        "Node": "s3",
        "Address": "172.17.0.5",
        "Datacenter": "dc1",
        "TaggedAddresses": {
            "lan": "172.17.0.5",
            "wan": "172.17.0.5"
        },
        "NodeMeta": {
            "consul-network-segment": ""
        },
        "ServiceID": "web",
        "ServiceName": "web",
        "ServiceTags": [
            "master"
        ],
        "ServiceAddress": "172.17.0.5",
        "ServiceMeta": {},
        "ServicePort": 10000,
        "ServiceEnableTagOverride": false,
        "CreateIndex": 587,
        "ModifyIndex": 587
    }
]
```

从 API 返回的信息知道，我们当前 s3 和 c1 都是 web 服务的提供方了，这个时候，我们心里应该开始想做那么一件事：既然 web 服务有了，如何通过名字调用它们,而非使用传统的 IP 直连的方式

其实脑袋里第一反应也许会想到：DNS 解析

## DNS 方式的服务发现

我们看看当前集群中谁是 leader

```bash
curl http://localhost:8500/v1/status/leader

"172.17.0.3:8300"
```

由于当前 leader 是 s1,所以接下来我们的 dns 名字服务测试就基于 s1 展开了

在 s1 中，执行 dig 命令

```bash
$ dig @127.0.0.1 -p 8600 web.service.consul SRV

; <<>> DiG 9.9.4-RedHat-9.9.4-61.el7 <<>> @127.0.0.1 -p 8600 web.service.consul SRV
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 14484
;; flags: qr aa rd; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 5
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web.service.consul.        IN  SRV

;; ANSWER SECTION:
web.service.consul. 0   IN  SRV 1 1 10000 s3.node.dc1.consul.
web.service.consul. 0   IN  SRV 1 1 10000 c1.node.dc1.consul.

;; ADDITIONAL SECTION:
s3.node.dc1.consul. 0   IN  A   172.17.0.5
s3.node.dc1.consul. 0   IN  TXT "consul-network-segment="
c1.node.dc1.consul. 0   IN  A   172.17.0.6
c1.node.dc1.consul. 0   IN  TXT "consul-network-segment="

;; Query time: 1 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Thu May 31 07:11:29 UTC 2018
;; MSG SIZE  rcvd: 227
```

可以看到在 ANSWER SECTION 中，我们得到了两个结果：s3 和 c1 上各有一个 web 的服务。在 dig 命令中我们用了 SRV 标志，那是因为我们需要的服务信息不仅有 ip 地址，还需要有端口号

这个时候，我们尝试停止 c1 上的 web 服务：

```pkill hc```

再重写执行 dig 命令

```bash
$ dig @127.0.0.1 -p 8600 web.service.consul SRV

; <<>> DiG 9.9.4-RedHat-9.9.4-61.el7 <<>> @127.0.0.1 -p 8600 web.service.consul SRV
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 59668
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 3
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web.service.consul.        IN  SRV

;; ANSWER SECTION:
web.service.consul. 0   IN  SRV 1 1 10000 s3.node.dc1.consul.

;; ADDITIONAL SECTION:
s3.node.dc1.consul. 0   IN  A   172.17.0.5
s3.node.dc1.consul. 0   IN  TXT "consul-network-segment="

;; Query time: 0 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Thu May 31 07:13:30 UTC 2018
;; MSG SIZE  rcvd: 137

```

结果显示，只有 s3 上这一个web服务可用了。当我们再次恢复 c1 的 web 服务，s1 的服务信息也会立刻更新：

```bash
web.service.consul. 0   IN  SRV 1 1 10000 c1.node.dc1.consul.
web.service.consul. 0   IN  SRV 1 1 10000 s3.node.dc1.consul.
```

还记得 c1 和 s3 上的 web 服务的 api 吗？正常来说可以通过调用 http://ip:9999/ 来获取服务器 hostname 信息的，既然 s1 上已经能“发现”web 的服务了，我们尝试通过以下请求测试一下：

```curl http://web.service.consul:9999/```

好遗憾，我们立刻收到错误信息：

```curl: (6) Could not resolve host: web.service.consul; Unknown error```

目前服务名字虽然已经在 consul 内部注册和发现了，但服务器上的程序仍然不能识别到的，这个时候，我们需要一些补充手段来解决这个问题

### 安装 dnsmasq

```bash
yum -y install dnsmasq

#验证
$ dnsmasq -v

Dnsmasq version 2.76  Copyright (c) 2000-2016 Simon Kelley
```

接下来，修改 /etc/dnsmasq.conf

主要修改几个关键配置：

下面的配置修改主要是方便我们自己的测试，更加规范的配置建议参考 dnsmasq 的相关文档自行配置

```bash
resolv-file=/etc/resolv.conf

strict-order

server=/consul/172.17.0.3#8600
```

接着需要在 /etc/resolv.conf 修改几行配置：

```bash
nameserver 127.0.0.1
nameserver 172.17.0.3
#nameserver 8.8.8.8
```

然后启动 dnsmasq, 再次dig一下：

```bash
dig @127.0.0.1 -p 8600 web.service.consul SRV

; <<>> DiG 9.9.4-RedHat-9.9.4-61.el7 <<>> @127.0.0.1 -p 8600 web.service.consul SRV
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 20604
;; flags: qr aa rd; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 5
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web.service.consul.        IN  SRV

;; ANSWER SECTION:
web.service.consul. 0   IN  SRV 1 1 10000 s3.node.dc1.consul.
web.service.consul. 0   IN  SRV 1 1 10000 c1.node.dc1.consul.

;; ADDITIONAL SECTION:
s3.node.dc1.consul. 0   IN  A   172.17.0.5
s3.node.dc1.consul. 0   IN  TXT "consul-network-segment="
c1.node.dc1.consul. 0   IN  A   172.17.0.6
c1.node.dc1.consul. 0   IN  TXT "consul-network-segment="

;; Query time: 0 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Thu May 31 08:57:25 UTC 2018
;; MSG SIZE  rcvd: 227
```

然后直接 dig web.service.consul，验证机器最终是否能识别名字服务：

```bash
$ dig web.service.consul SRV

; <<>> DiG 9.9.4-RedHat-9.9.4-61.el7 <<>> web.service.consul SRV
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 61000
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 5

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web.service.consul.        IN  SRV

;; ANSWER SECTION:
web.service.consul. 0   IN  SRV 1 1 10000 s3.node.dc1.consul.
web.service.consul. 0   IN  SRV 1 1 10000 c1.node.dc1.consul.

;; ADDITIONAL SECTION:
s3.node.dc1.consul. 0   IN  A   172.17.0.5
s3.node.dc1.consul. 0   IN  TXT "consul-network-segment="
c1.node.dc1.consul. 0   IN  A   172.17.0.6
c1.node.dc1.consul. 0   IN  TXT "consul-network-segment="

;; Query time: 2 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Thu May 31 08:58:25 UTC 2018
;; MSG SIZE  rcvd: 227
```

好样的，DNS 查询解析 web 的名字服务目前看来是成功的，最后我们尝试直接请求 web api 看是否能正常返回信息：

```bash
$ curl http://web.service.consul:9999
result from 5cade4f672b6

$ curl http://web.service.consul:9999
result from 5cade4f672b6

$ curl http://web.service.consul:9999
result from 579c1deb914f

$ curl http://web.service.consul:9999
result from 5cade4f672b6

$ curl http://web.service.consul:9999
result from 5cade4f672b6

$ curl http://web.service.consul:9999
result from 579c1deb914f
```

上面的请求知道 s1 对 web.service.consul 的名字解析成功，并能成功调用到 s3 和 c1 的 web 服务。

到这来为止还不能松懈，还有最后一步需要测试的，接下来我们关闭 s3 的服务，观察下解析调用的情况：

```bash
//s3

pkill hc
```

这时候，我们查询下 web 的名字解析情况：

```bash
$ dig web.service.consul SRV

; <<>> DiG 9.9.4-RedHat-9.9.4-61.el7 <<>> web.service.consul SRV
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 650
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web.service.consul.        IN  SRV

;; ANSWER SECTION:
web.service.consul. 0   IN  SRV 1 1 10000 c1.node.dc1.consul.

;; ADDITIONAL SECTION:
c1.node.dc1.consul. 0   IN  A   172.17.0.6
c1.node.dc1.consul. 0   IN  TXT "consul-network-segment="

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Thu May 31 09:03:44 UTC 2018
;; MSG SIZE  rcvd: 137
```

我们连续访问 curl http://web.service.consul:9999 的时候，输出都是：

```result from 5cade4f672b6 //c1```

可以看到，s1 当前只能解析 / 发现 c1 的服务，也就是说我们做到了服务的动态“下线”了。

当我们再次恢复 s3 的时候：

```bash
$ dig web.service.consul SRV

; <<>> DiG 9.9.4-RedHat-9.9.4-61.el7 <<>> web.service.consul SRV
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 9970
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 5

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web.service.consul.        IN  SRV

;; ANSWER SECTION:
web.service.consul. 0   IN  SRV 1 1 10000 c1.node.dc1.consul.
web.service.consul. 0   IN  SRV 1 1 10000 s3.node.dc1.consul.

;; ADDITIONAL SECTION:
c1.node.dc1.consul. 0   IN  A   172.17.0.6
c1.node.dc1.consul. 0   IN  TXT "consul-network-segment="
s3.node.dc1.consul. 0   IN  A   172.17.0.5
s3.node.dc1.consul. 0   IN  TXT "consul-network-segment="

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Thu May 31 09:06:47 UTC 2018
;; MSG SIZE  rcvd: 227
```

s3 的 web 服务又能自动注册到服务集群上去了，这个示例的演示也结束了。

## 总结

到目前为止，我们基本演示了 web 服务的服务注册和服务发现，虽然这示例比较简单，但实际项目中的服务化原理也是万变不离其宗的，结合相关架构设计，我们就可以搭建满足业务需要的服务发现和服务注册功能。

从集群的构建到服务发现的提供来看，consul 这个工具使用上还是挺简单的，不过还是那句，服务的高可用是全方位的，上线生产环境时需要注意 dns 缓存时间的配置，以及 DNS 域名管理方面的支持。consul 不是银弹，我们设计架构的时候更多要考虑服务之间的合理交互和更具体的容错方案。不过，不可否认的是以服务注册与发现的手段上，consul 是一套优秀的解决方案。

> 原文链接：<https://lihaoquan.me/2018/5/31/consul-in-action.html>
