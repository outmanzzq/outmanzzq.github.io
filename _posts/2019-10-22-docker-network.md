---
layout: post
title: Docker：网络模式详解
categories: docker,network
description: Docker 容器的网络默认与宿主机、与其他容器都是相互隔离,本文主要介绍 Docker 高级网络功能。
keywords: docker,network
---

> Docker 容器的网络默认与宿主机、与其他容器都是相互隔离,本文主要介绍 Docker 高级网络功能。

**Docker 自身的4种网络工作方式，和一些自定义网络模式:**

安装 Docker 时，它会自动创建三个网络，bridge（创建容器默认连接到此网络）、 none 、host

- host：容器将不会虚拟出自己的网卡，配置自己的 IP 等，而是使用宿主机的IP和端口。
- Container：创建的容器不会创建自己的网卡，配置自己的 IP，而是和一个指定的容器共享 IP、端口范围。
- None：该模式关闭了容器的网络功能。
- Bridge：此模式会为每一个容器分配、设置 IP 等，并将容器连接到一个 docker0 虚拟网桥，通过 docker0 网桥以及 Iptables nat 表配置与宿主机通信。

以上都是不用动手的，真正需要配置的是自定义网络。

## 一、前言

当你开始大规模使用 Docker 时，你会发现需要了解很多关于网络的知识。

Docker 作为目前最火的轻量级容器技术，有很多令人称道的功能，如 Docker 的镜像管理。

然而，Docker 同样有着很多不完善的地方，网络方面就是 Docker 比较薄弱的部分。

因此，我们有必要深入了解 Docker 的网络知识，以满足更高的网络需求。

本文首先介绍了 Docker 自身的4种网络工作方式，然后介绍一些自定义网络模式。

## 二、默认网络

当你安装 Docker 时，它会自动创建三个网络。你可以使用以下 `docker network ls`命令列出这些网络：

```bash
$ docker network ls
NETWORK ID          NAME                DRIVER
7fca4eb8c647        bridge              bridge
9f904ee27bf5        none                null
cf03ee007fb4        host                host
```

Docker 内置这三个网络，运行容器时，你可以使用该 --network 标志来指定容器应连接到哪些网络。

> 该 bridge 网络代表 docker0 所有 Docker 安装中存在的网络。除非你使用该 `docker run --network=<NETWORK>` 选项指定，否则 Docker 守护程序默认将容器连接到此网络。

我们在使用 docker run 创建 Docker 容器时，可以用 --net 选项指定容器的网络模式，Docker 可以有以下 4 种网络模式：

- host 模式：使用 --net=host 指定。
- none 模式：使用 --net=none 指定。
- bridge 模式：使用 --net=bridge 指定，默认设置。
- container 模式：使用 --net=container:NAME_or_ID 指定。

下面分别介绍一下 Docker 的各个网络模式。

### 2.1 Host

相当于 Vmware 中的桥接模式，与宿主机在同一个网络中，但没有独立 IP 地址。

众所周知，Docker 使用了 Linux 的 Namespaces 技术来进行资源隔离，如 PID Namespace 隔离进程，Mount Namespace 隔离文件系统，Network Namespace 隔离网络等。

一个 Network Namespace 提供了一份独立的网络环境，包括网卡、路由、Iptable 规则等都与其他的 Network Namespace 隔离。

一个 Docker 容器一般会分配一个独立的 Network Namespace。

但如果启动容器的时候使用 host 模式，那么这个容器将不会获得一个独立的 Network Namespace，而是和宿主机共用一个 Network Namespace。

容器将不会虚拟出自己的网卡，配置自己的 IP 等，而是使用宿主机的 IP 和端口。

例如，我们在 10.10.0.186/24 的机器上用 host 模式启动一个含有 nginx 应用的 Docker 容器，监听 tcp80 端口。

```bash
# 运行容器;
$ docker run --name=nginx_host --net=host -p 80:80 -d nginx
74c911272942841875f4faf2aca02e3814035c900840d11e3f141fbaa884ae5c

# 查看容器;
$ docker ps  
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
74c911272942        nginx               "nginx -g 'daemon ..."   25 seconds ago      Up 25 seconds                           nginx_host
```

当我们在容器中执行任何类似 ifconfig 命令查看网络环境时，看到的都是宿主机上的信息。

而外界访问容器中的应用，则直接使用 10.10.0.186:80 即可，不用任何 NAT 转换，就如直接跑在宿主机中一样。

但是，容器的其他方面，如文件系统、进程列表等还是和宿主机隔离的。

```bash
$ netstat -nplt | grep nginx
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      27340/nginx: master
```

### 2.2 Container

在理解了 host 模式后，这个模式也就好理解了。

这个模式指定新创建的容器和已经存在的一个容器共享一个 Network Namespace，而不是和宿主机共享。

新创建的容器不会创建自己的网卡，配置自己的 IP，而是和一个指定的容器共享 IP、端口范围等。

同样，两个容器除了网络方面，其他的如文件系统、进程列表等还是隔离的。两个容器的进程可以通过 lo 网卡设备通信。

### 2.3 None

该模式将容器放置在它自己的网络栈中，但是并不进行任何配置。

> 实际上，该模式关闭了容器的网络功能，在以下两种情况下是有用的：容器并不需要网络（例如只需要写磁盘卷的批处理任务）。

overlay

在 docker1.7 代码进行了重构，单独把网络部分独立出来编写，所以在 docker1.8 新加入的一个 overlay 网络模式。Docker 对于网络访问的控制也是在逐渐完善的。

### 2.4 Bridge

相当于 Vmware 中的 Nat 模式，容器使用独立 network Namespace，并连接到 docker0 虚拟网卡（默认模式）。

通过 docker0 网桥以及 Iptables nat 表配置与宿主机通信；

bridge 模式是 Docker 默认的网络设置，此模式会为每一个容器分配 Network Namespace、设置 IP 等，并将一个主机上的 Docker 容器连接到一个虚拟网桥上。下面着重介绍一下此模式。

## 三、Bridge 模式

### 3.1 Bridge 模式的拓扑

当 Docker server 启动时，会在主机上创建一个名为 docker0 的虚拟网桥，此主机上启动的 Docker 容器会连接到这个虚拟网桥上。

虚拟网桥的工作方式和物理交换机类似，这样主机上的所有容器就通过交换机连在了一个二层网络中。

接下来就要为容器分配 IP 了，Docker 会从 RFC1918 所定义的私有 IP 网段中，选择一个和宿主机不同的IP地址和子网分配给 docker0，连接到 docker0 的容器就从这个子网中选择一个未占用的 IP 使用。

> 如一般 Docker 会使用 172.17.0.0/16 这个网段，并将 172.17.0.1/16 分配给 docker0 网桥（在主机上使用 ifconfig 命令是可以看到 docker0 的，可以认为它是网桥的管理接口，在宿主机上作为一块虚拟网卡使用）。
>
> 单机环境下的网络拓扑如下，主机地址为 10.10.0.186/24。

![ ](/images/20191022-docker-network-01.jpg)

### 3.2 Docker：网络模式详解

Docker 完成以上网络配置的过程大致是这样的：

1. 在主机上创建一对虚拟网卡 veth pair 设备。veth 设备总是成对出现的，它们组成了一个数据的通道，数据从一个设备进入，就会从另一个设备出来。因此，veth 设备常用来连接两个网络设备。

2. Docker 将 veth pair 设备的一端放在新创建的容器中，并命名为 eth0。另一端放在主机中，以 veth65f9 这样类似的名字命名，并将这个网络设备加入到 docker0 网桥中，可以通过 brctl show 命令查看。

```bash
$ brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.02425f21c208       no
```

3. 从 docker0 子网中分配一个 IP 给容器使用，并设置 docker0 的 IP 地址为容器的默认网关。

```bash
# 运行容器;
$ docker run --name=nginx_bridge --net=bridge -p 80:80 -d nginx
9582dbec7981085ab1f159edcc4bf35e2ee8d5a03984d214bce32a30eab4921a

# 查看容器;
$ docker ps
CONTAINER ID        IMAGE          COMMAND                  CREATED             STATUS              PORTS                NAMES
9582dbec7981        nginx          "nginx -g 'daemon ..."   3 seconds ago       Up 2 seconds        0.0.0.0:80->80/tcp   nginx_bridge

# 查看容器网络;
$ docker inspect 9582dbec7981
"Networks": {
    "bridge": {
        "IPAMConfig": null,
        "Links": null,
        "Aliases": null,
        "NetworkID": "9e017f5d4724039f24acc8aec634c8d2af3a9024f67585fce0a0d2b3cb470059",
        "EndpointID": "81b94c1b57de26f9c6690942cd78689041d6c27a564e079d7b1f603ecc104b3b",
        "Gateway": "172.17.0.1",
        "IPAddress": "172.17.0.2",
        "IPPrefixLen": 16,
        "IPv6Gateway": "",
        "GlobalIPv6Address": "",
        "GlobalIPv6PrefixLen": 0,
        "MacAddress": "02:42:ac:11:00:02"
    }
}

$ docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "9e017f5d4724039f24acc8aec634c8d2af3a9024f67585fce0a0d2b3cb470059",
        "Created": "2017-08-09T23:20:28.061678042-04:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "Containers": {
            "9582dbec7981085ab1f159edcc4bf35e2ee8d5a03984d214bce32a30eab4921a": {
                "Name": "nginx_bridge",
                "EndpointID": "81b94c1b57de26f9c6690942cd78689041d6c27a564e079d7b1f603ecc104b3b",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```

网络拓扑介绍完后，接着介绍一下 bridge 模式下容器是如何通信的。

### 3.3 bridge 模式下容器的通信

在 bridge 模式下，连在同一网桥上的容器可以相互通信（若出于安全考虑，也可以禁止它们之间通信，方法是在 DOCKER_OPTS 变量中设置 –icc=false，这样只有使用 –link 才能使两个容器通信）。

Docker 可以开启容器间通信（意味着默认配置 --icc=true ），也就是说，宿主机上的所有容器可以不受任何限制地相互通信，这可能导致拒绝服务攻击。进一步地，Docker 可以通过 --ip_forward 和 --iptables 两个选项控制容器间、容器和外部世界的通信。

容器也可以与外部通信，我们看一下主机上的 Iptable 规则，可以看到这么一条

`-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE`

这条规则会将源地址为 172.17.0.0/16 的包（也就是从 Docker 容器产生的包），并且不是从 docker0 网卡发出的，进行源地址转换，转换成主机网卡的地址。

这么说可能不太好理解，举一个例子说明一下。

① 假设主机有一块网卡为 eth0，IP 地址为 10.10.101.105/24，网关为 10.10.101.254。

② 从主机上一个 IP 为 172.17.0.1/16 的容器中 ping 百度（180.76.3.151）。

③ IP 包首先从容器发往自己的默认网关 docker0，包到达 docker0 后，也就到达了主机上。

④ 然后会查询主机的路由表，发现包应该从主机的 eth0 发往主机的网关 10.10.105.254/24。

⑤ 接着包会转发给 eth0，并从 eth0 发出去（主机的 ip_forward 转发应该已经打开）。

这时候，上面的 Iptable 规则就会起作用，对包做 SNAT 转换，将源地址换为 eth0 的地址。

这样，在外界看来，这个包就是从 10.10.101.105 上发出来的，Docker 容器对外是不可见的。

那么，外面的机器是如何访问 Docker 容器的服务呢？我们首先用下面命令创建一个含有 web 应用的容器，将容器的 80 端口映射到主机的80端口。

`$ docker run --name=nginx_bridge --net=bridge -p 80:80 -d nginx`

然后查看 Iptable 规则的变化，发现多了这样一条规则：

`-A DOCKER ! -i docker0 -p tcp -m tcp --dport 80 -j DNAT --to-destination 172.17.0.2:80`

此条规则就是对主机 eth0 收到的目的端口为 80 的 tcp 流量进行 DNAT 转换，将流量发往 172.17.0.2:80，也就是我们上面创建的 Docker 容器。所以，外界只需访问 10.10.101.105:80 就可以访问到容器中的服务。

除此之外，我们还可以自定义 Docker 使用的 IP 地址、DNS 等信息，甚至使用自己定义的网桥，但是其工作方式还是一样的。

## 四、自定义网络

建议使用自定义的网桥来控制哪些容器可以相互通信，还可以自动 DNS 解析容器名称到 IP 地址。

Docker 提供了创建这些网络的默认网络驱动程序，你可以创建一个新的 Bridge 网络，Overlay 或 Macvlan 网络。你还可以创建一个网络插件或远程网络进行完整的自定义和控制。

你可以根据需要创建任意数量的网络，并且可以在任何给定时间将容器连接到这些网络中的零个或多个网络。

此外，您可以连接并断开网络中的运行容器，而无需重新启动容器。当容器连接到多个网络时，其外部连接通过第一个非内部网络以词法顺序提供。

接下来介绍 Docker 的内置网络驱动程序。

### 4.1 bridge

一个 bridge 网络是 Docker 中最常用的网络类型。桥接网络类似于默认 bridge 网络，但添加一些新功能并删除一些旧的能力。以下示例创建一些桥接网络，并对这些网络上的容器执行一些实验。

`$ docker network create --driver bridge new_bridge`

创建网络后，可以看到新增加了一个网桥（172.18.0.1）。

```bash
$ ifconfig
br-f677ada3003c: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.18.0.1  netmask 255.255.0.0  broadcast 0.0.0.0
        ether 02:42:2f:c1:db:5a  txqueuelen 0  (Ethernet)
        RX packets 4001976  bytes 526995216 (502.5 MiB)
        RX errors 0  dropped 35  overruns 0  frame 0
        TX packets 1424063  bytes 186928741 (178.2 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 0.0.0.0
        inet6 fe80::42:5fff:fe21:c208  prefixlen 64  scopeid 0x20<link>
        ether 02:42:5f:21:c2:08  txqueuelen 0  (Ethernet)
        RX packets 12  bytes 2132 (2.0 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 24  bytes 2633 (2.5 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

### 4.2 Macvlan

Macvlan 是一个新的尝试，是真正的网络虚拟化技术的转折点。

Linux 实现非常轻量级，因为与传统的 Linux Bridge 隔离相比，它们只是简单地与一个 Linux 以太网接口或子接口相关联，以实现网络之间的分离和与物理网络的连接。

Macvlan 提供了许多独特的功能，并有充足的空间进一步创新与各种模式。

这些方法的两个高级优点是绕过 Linux 网桥的正面性能以及移动部件少的简单性。

删除传统上驻留在 Docker 主机 NIC 和容器接口之间的网桥留下了一个非常简单的设置，包括容器接口，直接连接到 Docker 主机接口。

> 由于在这些情况下没有端口映射，因此可以轻松访问外部服务。

#### 4.2.1 Macvlan Bridge 模式示例用法

Macvlan Bridge 模式每个容器都有唯一的 MAC 地址，用于跟踪 Docker 主机的 MAC 到端口映射。

Macvlan 驱动程序网络连接到父 Docker 主机接口。示例是物理接口，

例如:

eth0，用于 802.1q VLAN 标记的子接口 eth0.10（.10代表VLAN 10）或甚至绑定的主机适配器，将两个以太网接口捆绑为单个逻辑接口。 指定的网关由网络基础设施提供的主机外部。

每个 Macvlan Bridge 模式的 Docker 网络彼此隔离，一次只能有一个网络连接到父节点。

每个主机适配器有一个理论限制，每个主机适配器可以连接一个 Docker 网络。

同一子网内的任何容器都可以与没有网关的同一网络中的任何其他容器进行通信 macvlan bridge。

相同的 docker network 命令适用于 vlan 驱动程序。

> 在 Macvlan 模式下，在两个网络/子网之间没有外部进程路由的情况下，单独网络上的容器无法互相访​​问。这也适用于同一码头网络内的多个子网。

在以下示例中，eth0在docker主机网络上具有IP地址172.16.86.0/24，默认网关为172.16.86.1，网关地址为外部路由器172.16.86.1。

![ ](/images/20191022-docker-network-02.jpg)

注意对于 Macvlan 桥接模式，子网值需要与 Docker 主机的 NIC 的接口相匹配。例如，使用由该 -o parent= 选项指定的 Docker 主机以太网接口的相同子网和网关。

> 此示例中使用的父接口位于 eth0 子网上 172.16.86.0/24，这些容器中的容器 docker network 也需要和父级同一个子网 -o parent=。
>
> 网关是网络上的外部路由器，不是任何 ip 伪装或任何其他本地代理。
>
> 驱动程序用 -d driver_name 选项指定，在这种情况下 -d macvlan。

父节点 -o parent=eth0 配置如下：

```bash
$ ip addr show eth0
3: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    inet 172.16.86.250/24 brd 172.16.86.255 scope global eth0
```

创建 macvlan 网络并运行附加的几个容器：

```bash
# Macvlan  (-o macvlan_mode= Defaults to Bridge mode if not specified)
docker network create -d macvlan \
    --subnet=172.16.86.0/24 \
    --gateway=172.16.86.1  \
    -o parent=eth0 pub_net

# Run a container on the new network specifying the --ip address.
docker  run --net=pub_net --ip=172.16.86.10 -itd alpine /bin/sh

# Start a second container and ping the first
docker  run --net=pub_net -it --rm alpine /bin/sh
ping -c 4 172.16.86.10
```

看看容器 ip 和路由表：

```bash
ip a show eth0
    eth0@if3: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UNKNOWN
    link/ether 46:b2:6b:26:2f:69 brd ff:ff:ff:ff:ff:ff
    inet 172.16.86.2/24 scope global eth0

ip route
    default via 172.16.86.1 dev eth0
    172.16.86.0/24 dev eth0  src 172.16.86.2

# NOTE: the containers can NOT ping the underlying host interfaces as
# they are intentionally filtered by Linux for additional isolation.
# In this case the containers cannot ping the -o parent=172.16.86.250
```

#### 4.2.2 Macvlan 802.1q Trunk Bridge 模式示例用法

VLAN（虚拟局域网）长期以来一直是虚拟化数据中心网络的主要手段，目前仍在几乎所有现有的网络中隔离广播的主要手段。

常用的 VLAN 划分方式是通过端口进行划分，尽管这种划分 VLAN 的方式设置比较很简单，但仅适用于终端设备物理位置比较固定的组网环境。

随着移动办公的普及，终端设备可能不再通过固定端口接入交换机，这就会增加网络管理的工作量。

比如，一个用户可能本次接入交换机的端口 1，而下一次接入交换机的端口 2，由于端口1和端口 2 属于不同的 VLAN，若用户想要接入原来的 VLAN 中，网管就必须重新对交换机进行配置。

显然，这种划分方式不适合那些需要频繁改变拓扑结构的网络。

而 MAC VLAN 可以有效解决这个问题，它根据终端设备的 MAC 地址来划分 VLAN。这样，即使用户改变了接入端口，也仍然处在原 VLAN 中。

Mac vlan 不是以交换机端口来划分 vlan。因此，一个交换机端口可以接受来自多个 mac 地址的数据。一个交换机端口要处理多个 vlan 的数据，则要设置 trunk 模式。

在主机上同时运行多个虚拟网络的要求是非常常见的。Linux 网络长期以来一直支持 VLAN 标记，也称为标准 802.1q，用于维护网络之间的数据路由隔离。连接到 Docker 主机的以太网链路可以配置为支持 802.1q VLAN ID，方法是创建 Linux 子接口，每个子接口专用于唯一的VLAN ID。

![ ](/images/20191022-docker-network-03.jpg)

**创建 Macvlan 网络**(VLAN ID 10)

```bash
$ docker network create \
  --driver macvlan \
  --subnet=10.10.0.0/24 \
  --gateway=10.10.0.253 \
  -o parent=eth0.10 macvlan10

# 开启一个桥接 Macvlan 的容器
$ docker run --net=macvlan10 -it --name macvlan_test1 --rm alpine /bin/sh
/ # ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
21: eth0@if13: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UNKNOWN
    link/ether 02:42:0a:0a:00:01 brd ff:ff:ff:ff:ff:ff
    inet 10.10.0.1/24 scope global eth0
       valid_lft forever preferred_lft forever

# 可以看到分配了一个 10.10.0.1 的地址，然后看一下路由地址。

/ # ip route
default via 10.10.0.253 dev eth0
10.10.0.0/24 dev eth0  src 10.10.0.1
```

然后再开启一个桥接 Macvlan 的容器：

```bash
$ docker run --net=macvlan10 -it --name macvlan_test2 --rm alpine /bin/sh
/ # ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
22: eth0@if13: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UNKNOWN
    link/ether 02:42:0a:0a:00:02 brd ff:ff:ff:ff:ff:ff
    inet 10.10.0.2/24 scope global eth0
       valid_lft forever preferred_lft forever

# 可以看到分配了一个 10.10.0.2 的地址，然后可以在两个容器之间相互 ping，是可以 ping 通的。
/ # ping 10.10.0.1
PING 10.10.0.1 (10.10.0.1): 56 data bytes
64 bytes from 10.10.0.1: seq=0 ttl=64 time=0.094 ms
64 bytes from 10.10.0.1: seq=1 ttl=64 time=0.057 ms
```

经过上面两个容器的创建可以看出，容器IP是根据创建网络时的网段从小往大分配的。

当然，在创建容器时，我们也可以使用 --ip 手动执行一个 IP 地址分配给容器，如下操作。

```bash
$ docker run --net=macvlan10 -it --name macvlan_test3 --ip=10.10.0.189 --rm alpine /bin/sh
/ # ip addr show eth0
24: eth0@if13: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UNKNOWN
    link/ether 02:42:0a:0a:00:bd brd ff:ff:ff:ff:ff:ff
    inet 10.10.0.189/24 scope global eth0
       valid_lft forever preferred_lft forever
```

**VLAN ID 20**

接着可以创建由 Docker 主机标记和隔离的第二个 VLAN 网络，该 macvlan_mode 默认是 macvlan_mode=bridge，如下：

```bash
$ docker network create \
  --driver macvlan \
  --subnet=192.10.0.0/24 \
  --gateway=192.10.0.253 \
  -o parent=eth0.20 \
  -o macvlan_mode=bridge macvlan20
```

当我们创建完 Macvlan 网络之后，在 docker 主机可以看到相关的子接口，如下：

```bash
$ ifconfig
eth0.10: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether 00:0c:29:16:01:8b  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 18  bytes 804 (804.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0.20: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether 00:0c:29:16:01:8b  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# 在 /proc/net/vlan/config 文件中，还可以看见相关的 Vlan 信息，如下：
$ cat /proc/net/vlan/config
VLAN Dev name    | VLAN ID
Name-Type: VLAN_NAME_TYPE_RAW_PLUS_VID_NO_PAD
eth0.10        | 10  | eth0
eth0.20        | 20  | eth0
```

> 原文链接：<https://www.cnblogs.com/zuxing/articles/8780661.html>