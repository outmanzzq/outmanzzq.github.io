---
layout: post
title: Vagrant 学习 - Vagrantfile
categories: vagrant
description: Vagrant 学习 - Vagrantfile
keywords: Vagrant, Vagrantfile
---

> 有时候，希望对 vm 做更详尽的配置，比如配置一次创建一组 vm ，搭建一个 mfs 的测试环境，需要一台服务器做 mfsmaster ，两台服务器做 mfs chunk server ，一台服务器做 metalogger ，还有一台服务器做 mfs client 进行测试。

下面是一组服务测试 mfs 的 vagrant file 范例:

```YAML
# -*- mode: ruby -*-
# vi: set ft=ruby :

app_servers = {
    :mfschunk1 => '192.168.33.20',
    :mfschunk2 => '192.168.33.21'
}

Vagrant.configure("2") do |config|
    config.vm.box = "centos64"

    config.vm.define :mfsmaster do |mfsmaster_config|
        mfsmaster_config.vm.network :private_network, ip: "192.168.33.10"
        config.vm.provider :virtualbox do |vb|
            vb.name = "mfsmaster"
            vb.customize ["modifyvm", :id, "--cpuexecutioncap", "50"]
            vb.customize ["modifyvm", :id, "--memory", "1024"]
            vb.customize ["modifyvm", :id, "--cpus", "2"]
            vb.cpus = 2
        end
    end

    app_servers.each do |app_server_name, app_server_ip|
        config.vm.define app_server_name do |app_config|
            app_config.vm.hostname = "#{app_server_name.to_s}.vagrant.internal"
            app_config.vm.network :private_network, ip: app_server_ip
            app_config.vm.provider "virtualbox" do |vb|
                vb.name = app_server_name.to_s
            end
        end
    end

    config.vm.define :metalogger do |metalogger_config|
        metalogger_config.vm.hostname = "metalogger.vagrant.internal"
        metalogger_config.vm.network :private_network, ip: "192.168.33.30"
        metalogger_config.vm.provider "virtualbox" do |vb|
            vb.name = "metalogger"
            vb.customize ["modifyvm", :id, "--cpuexecutioncap", "50"]
            vb.customize ["modifyvm", :id, "--memory", "1024"]
            vb.customize ["modifyvm", :id, "--cpus", "2"]
            vb.cpus = 2
        end
    end

    config.vm.define :mfsclient do |mfsclient_config|
        mfsclient_config.vm.hostname = "mfsclient.vagrant.internal"
        mfsclient_config.vm.network :private_network, ip: "192.168.33.40"
        mfsclient_config.vm.provider "virtualbox" do |vb|
            vb.name = "mfsclient"
        end
    end
end
```

保存 Vagrantfile 文件以后，执行如下命令查看虚机配置：

```shell
$ vagrant status
Current machine states:

mfsmaster                 not created (virtualbox)
mfschunk1                 not created (virtualbox)
mfschunk2                 not created (virtualbox)
metalogger                not created (virtualbox)
mfsclient                 not created (virtualbox)
```

执行 up 命令，批量创建虚机并启动。

`$ vagrant up`

如果只想启动一台，执行：

`$ vagrant up mfsmaster`

# 一、vagrant基础配置

Vagrantfile 配置文件的格式简单介绍一下。

## 1.1 文件头

说明文件的格式信息，处理方式等。

```YAML
# -*- mode: ruby -*-
# vi: set ft=ruby :
```

指定使用 ruby 格式，vi 进行编辑的，所有文件都采用这个文件头即可。

## 1.2 通用数据

设置一些基础数据，供配置信息中调用。

```YAML
app_servers = {
    :mfschunk1 => '192.168.33.20',
    :mfschunk2 => '192.168.33.21'
}
```

这里是定义一个 hashmap，以 key-value 方式来存储 vm 主机名和 ip 地址。

## 1.3 配置信息

```YAML
ENV["LC_ALL"] = "en_US.UTF-8"
# 指定 vm 的语言环境，缺省地，会继承 host 的 locale 配置
Vagrant.configure("2") do |config|
    # ...
end
```

> **参数 2** ，表示的是当前配置文件使用的 vagrant configure 版本号为 Vagrant 1.1+
>> 如果取值为 1，表示为 Vagrant 1.0.x Vagrantfiles，旧版本暂不考虑，记住就写 2 即可。

本文只对 configure 2 作说明，旧版本不多解释了。

do … end 为配置的开始结束符，所有配置信息都写在这两段代码之间。
config 是为当前配置命名，你可以指定任意名称，如 myvmconfig ，在后面引用的时候，改为自己的名字即可。

### 1.3.1 基本配置信息

#### 镜像

`config.vm.box = "ubuntu/precise64"`

指定当前 vm 使用的 box 镜像名称，值为本地仓库的镜像名。

#### 定义vm的configure配置节点

```YAML
config.vm.define :mfsmaster do |mfsmaster_config|
# ...
end
```

表示在 config 配置中，定义一个名为 mfsmaster 的 vm 配置，该节点下的配置信息命名为 mfsmaster_config ；
如果该 Vagrantfile 配置文件只定义了一个 vm ，这个配置节点层次可忽略。

##### vm 网络环境配置

vagrant 的网络连接方式有三种：

- NAT : `缺省创建，用于让 vm 可以通过 host 转发访问局域网甚至互联网。`
- host-only : `只有主机可以访问 vm ，其他机器无法访问它。`
- bridge : `此模式下vm就像局域网中的一台独立的机器，可以被其他机器访问。`

`config.vm.network :private_network, ip: "192.168.33.10"`

配置当前 vm 的 host-only 网络的 IP 地址为 192.168.33.10

host-only 模式的 IP 可以不指定，而是采用 dhcp 自动生成的方式，如 :

  `config.vm.network "private_network", type: "dhcp”`

```shell
# config.vm.network "public_network", ip: "192.168.0.17"
# 创建一个 bridge 桥接网络，指定 IP
# config.vm.network "public_network", bridge: "en1: Wi-Fi (AirPort)"
# 创建一个 bridge 桥接网络，指定桥接适配器
config.vm.network "public_network"
# 创建一个 bridge 桥接网络，不指定桥接适配器
```

配置当前项以后，如果 host 有多个网络适配器，第一次启动会询问桥接到哪个网络，如：

```shell
==> mfsmaster: Available bridged network interfaces:
1) en1: Wi-Fi (AirPort)
2) en0: 以太网
3) en2: Thunderbolt 1
4) p2p0
==> mfsmaster: When choosing an interface, it is usually the one that is
==> mfsmaster: being used to connect to the internet.
    mfsmaster: Which interface should the network bridge to
```

我使用的是 wifi ，选择 1，继续。

##### 同步文件夹配置

用来让 host 与 vm 二者进行文件同步。

`config.vm.synced_folder "../data", "/vagrant_data"`

设置同步文件夹，让主机与 vm 中的一个文件夹内容保持一致。

缺省地，vagrant 会把工作目录映射到 vm 的 /vagrant 目录，如果需要增加更多同步文件夹，使用上面的配置，第一个文件夹为 host 主机的目录，第二个文件夹为 vm 中的目录。

##### 设置主机名

指定 vm 的 hostname ，会覆盖 vm 中 /etc/hostsname 中的设置。

`config.vm.hostname = “mfsmaster.vagrant.internal"`

##### 端口转发

指定将 host 的 8080 端口请求，转发到 vm 的 80 端口，这样访问 http://host:8080 就相当于访问 http://vm:80 了。结合《[WiKi-Vagrant](https://outmanzzq.github.io/wiki/vagrant/)》中的 share 共享 http 功能，我们就可以做到让 internet 每个角落的用户访问 vm 里的 http 服务了。

`config.vm.network: forwarded_port, guest: 80, host: 8080`

guest 和 host 是必须的，还有几个可选属性：

- guest_ip：`字符串，vm 指定绑定的 Ip，缺省为 0.0.0.0`
- host_ip：`字符串，host 指定绑定的 Ip，缺省为 0.0.0.0`
- protocol：`字符串，可选 TCP 或 UDP，缺省为 TCP`

### 1.3.2 vm provider 配置

```YAML
config.vm.provider :virtualbox do |vb|
     # ...
end
```

#### vm provider 通用配置

虚机容器提供者配置，对于不同的 provider ，特有的一些配置，此处配置信息是针对 virtualbox 定义一个提供者，命名为 vb ，跟前面一样，这个名字随意取，只要节点内部调用一致即可。

配置信息又分为通用配置和个性化配置，通用配置对于不同 provider 是通用的，常用的通用配置如下：

```YAML
vb.name = "centos6"
# 指定 vm-name，也就是 virtualbox 管理控制台中的虚机名称，或者说是 virtual box 工作目录的名字，即 ~/VirtualBox\ VMs/centos6 。如果不指定该选项会生成一个随机的名字，不容易区分。
vb.gui = true
# vagrant up 启动时，是否自动打开 virtual box 的窗口，缺省为 false
vb.memory = "1024"
# 指定 vm 内存，单位为 MB
vb.cpus = 2
# 设置 CPU 个数
```

##### vm provider 个性化配置（virtualbox）

上面的 provider 配置是通用的配置，针对不同的虚拟机，还有一些的个性的配置，通过 vb.customize 配置来定制。

对 virtual box 的个性化配置，可以参考：VBoxManage modifyvm 命令的使用方法。其他虚机的 provider，暂时未做测试。

详细的功能接口和使用说明，可以参考 virtualbox 官方文档。

```YAML
# 修改 vb.name 的值
v.customize ["modifyvm", :id, "--name", "mfsmaster2"]

# 如修改显存，缺省为 8M，如果启动桌面，至少需要 10M，如下修改为 16M：
vb.customize ["modifyvm", :id, "--vram", "16"]

# 调整虚拟机的内存
 vb.customize ["modifyvm", :id, "--memory", "1024"]

# 指定虚拟 CPU 个数
 vb.customize ["modifyvm", :id, "--cpus", "2"]

# 增加光驱：
vb.customize ["storageattach",:id,"--storagectl", "IDE Controller","--port","0","--device","0","--type","dvddrive","--medium","/Applications/VirtualBox.app/Contents/MacOS/VBoxGuestAdditions.iso"]
# meduim 参数不可以为空，如果只挂载驱动器不挂在 iso，指定为“ emptydrive ”。如果要卸载光驱，medium 传入 none 即可。
# 从这个指令可以看出，customize 方法传入一个 json 数组，按照顺序传入参数即可。

# json 数组传入多个参数，是否意味着 VBoxManage 命令一样，同时操作多个属性呢？由此，我们可以做一个测试来验证一下：
v.customize ["modifyvm", :id, "--name", “mfsserver3", "--memory", “2048"]
```

扩展一下，如果创建的虚机很多，vm 都混杂在一起，我们都知道 virtualbox 支持对vm进行分组。要在 vagrant 使用分组，可以在 mfs的vagrantfile 中如下自定义：

`vb.customize ["modifyvm",:id,"--groups",”/mfs"]`

参数说明：

分组名是路径格式，/ 开始，表示第一级目录，可以指定多级目录，如 /mfs/chunk；
可以指定多个分组，用逗号分开，如:“ /dev,/mfs ”
每一个 vm 创建以后，都会放到 mfs 分组里。可以在 virtualbox 管理界面查看。

### 1.3.3 一组相同配置的 vm

前面配置了一组 vm 的 hash map ，定义一组 vm 时，使用如下节点遍历。

遍历 app_servers map ，将 key 和 value 分别赋值给 app_server_name 和 app_server_ip

```YAML
app_servers.each do |app_server_name, app_server_ip|
     # 针对每一个 app_server_name，来配置 config.vm.define 配置节点，命名为 app_config
     config.vm.define app_server_name do |app_config|
          # 此处配置，参考 3.1.2 config.vm.define
     end
end
```

如果不想定义 app_servers ，下面也是一种方案:

```YAML
(1..3).each do |i|
        config.vm.define "app-#{i}" do |node|
        app_config.vm.hostname = "app-#{i}.vagrant.internal"
        app_config.vm.provider "virtualbox" do |vb|
            vb.name = app-#{i}
        end
  end
end
```

### 1.3.4 中央仓库配置

指定 box 镜像 push 发布的地址，供 box 镜像管理者使用。普通使用者不需关心。

```YAML
config.push.define "atlas" do |push|
     push.app = "YOUR_ATLAS_USERNAME/YOUR_APPLICATION_NAME"
end
```

### 1.3.5 vm 部署 ( provision )

用来加载 box 以后，对 vm 的环境进行一些定制，比如设置环境变量，安装软件，部署程序等。如：

```YAML
config.vm.provision "shell", inline: <<-SHELL
   sudo apt-get update
   sudo apt-get install -y apache2
SHELL
```

# 二、 进阶

## 2.1 部署两台 Heartbeat 主／备服务器

**要求**：

- 操作系统： CentOS 6.6 x64
- 主机创建完成后同步设置预定义网卡（ IP、VIP ）
- 基于辅助 IP （ ip addr 命令）配置 VIP
- 系统启动完毕后打印 IP 信息
- SSH 启用 root 登陆，并修改 root 密码为『root』

## 2.2 服务主机资源规划

VM 主机名 | 网卡 | IP | 用途
-|-|-|-
Master | eth1 | 10.0.0.7 | 外网管理 IP，用于 WAN 数据转发
-| eth2 |172.16.1.7 | 内网管理 IP，用于 LAN 数据转发
-| vip  |10.0.0.17  | 用于提供应用程序 A 挂载服务
BACKUP | eth1 | 10.0.0.8 | 外网管理 IP，用于 WAN 数据转发
-| eth2 | 172.16.1.8 | 内网管理 IP，用于 LAN 数据转发
-| vip  | 10.0.0.18  | 用于提供用于程序 B 挂载服务

## 2.3 Vagrantfile 文件内容

```YAML
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "centos-6.6-x86_64"

  # 定义 master
  config.vm.define :hbmaster do |hbmaster_config|
    hbmaster_config.vm.network :public_network, ip: "193.168.11.7"
    hbmaster_config.vm.network :private_network, ip: "172.16.1.7"
    hbmaster_config.vm.hostname = "master"
    hbmaster_config.vm.provision "shell", inline: <<-SHELL
      sudo ip addr add 10.0.0.17/24 dev eth2
      ip ad |grep 'inet'
      # set ssh root login
      sudo sed -i 's/#PermitRootLogin yes/PermitRootLogin yes/' /etc/ssh/sshd_config
      sudo /etc/init.d/sshd reload
      # set root password
      echo 'root' | sudo passwd --stdin root
    SHELL

    config.vm.provider :virtualbox do |vb|
      vb.name = "hbmaster"
      vb.customize ["modifyvm", :id, "--cpuexecutioncap", "50"]
      vb.customize ["modifyvm", :id, "--memory", "1024"]
      vb.customize ["modifyvm", :id, "--cpus", "2"]
      vb.cpus = 2
    end
  end

  # 定义 backup
  config.vm.define :hbbackup do |hbbackup_config|
    hbbackup_config.vm.network :public_network, ip: "192.168.11.8"
    hbbackup_config.vm.network :private_network, ip: "172.16.1.8"
    hbbackup_config.vm.hostname = "backup"
    hbbackup_config.vm.provision "shell", inline: <<-SHELL
      sudo ip addr add 10.0.0.18/24 dev eth2
      ip ad |grep 'inet'
      # set ssh root login
      sudo sed -i 's/#PermitRootLogin yes/PermitRootLogin yes/' /etc/ssh/sshd_config
      sudo /etc/init.d/sshd reload
      # set root password
      echo 'root' | sudo passwd --stdin root
    SHELL

    hbbackup_config.vm.provider "virtualbox" do |vb|
      vb.name = "hbbackup"
    end
  end
end
```

## 2.4 启动虚拟机

```shell
# 进行语法检查
$ vagrant validate
Vagrantfile validated successfully.

# 查看 vm 状态
$ vagrant status
Current machine states:

hbmaster                  not created (virtualbox)
hbbackup                  not created (virtualbox)

# 启动 (重载使用命令： vagrant reload --provision)
$ vagrant up
Bringing machine 'hbmaster' up with 'virtualbox' provider...
Bringing machine 'hbbackup' up with 'virtualbox' provider...
==> hbmaster: Importing base box 'centos-6.6-x86_64'...
==> hbmaster: Matching MAC address for NAT networking...
==> hbmaster: Setting the name of the VM: hbmaster
==> hbmaster: Clearing any previously set forwarded ports...
==> hbmaster: Clearing any previously set network interfaces...
==> hbmaster: Preparing network interfaces based on configuration...
    hbmaster: Adapter 1: nat
    hbmaster: Adapter 2: bridged
    hbmaster: Adapter 3: hostonly
==> hbmaster: Forwarding ports...
    hbmaster: 22 (guest) => 2222 (host) (adapter 1)
==> hbmaster: Running 'pre-boot' VM customizations...
==> hbmaster: Booting VM...
==> hbmaster: Waiting for machine to boot. This may take a few minutes...
    hbmaster: SSH address: 127.0.0.1:2222
    hbmaster: SSH username: vagrant
    hbmaster: SSH auth method: private key
    hbmaster:
    hbmaster: Vagrant insecure key detected. Vagrant will automatically replace
    hbmaster: this with a newly generated keypair for better security.
    hbmaster:
    hbmaster: Inserting generated public key within guest...
    hbmaster: Removing insecure key from the guest if its present...
    hbmaster: Key inserted! Disconnecting and reconnecting using new SSH key...
==> hbmaster: Machine booted and ready!
==> hbmaster: Checking for guest additions in VM...
    hbmaster: The guest additions on this VM do not match the installed version of
    hbmaster: VirtualBox! In most cases this is fine, but in rare cases it can
    hbmaster: prevent things such as shared folders from working properly. If you see
    hbmaster: shared folder errors, please make sure the guest additions within the
    hbmaster: virtual machine match the version of VirtualBox you have installed on
    hbmaster: your host and reload your VM.
    hbmaster:
    hbmaster: Guest Additions Version: 4.3.28
    hbmaster: VirtualBox Version: 5.1
==> hbmaster: Setting hostname...
==> hbmaster: Configuring and enabling network interfaces...
    hbmaster: SSH address: 127.0.0.1:2222
    hbmaster: SSH username: vagrant
    hbmaster: SSH auth method: private key
==> hbmaster: Mounting shared folders...
    hbmaster: /vagrant => F:/study/vagrant
==> hbmaster: Running provisioner: shell...
    hbmaster: Running: inline script
==> hbmaster:     inet 127.0.0.1/8 scope host lo
==> hbmaster:     inet6 ::1/128 scope host
==> hbmaster:     inet 10.0.2.15/24 brd 10.0.2.255 scope global eth0
==> hbmaster:     inet6 fe80::a00:27ff:fe2d:823a/64 scope link
==> hbmaster:     inet 192.168.11.7/24 brd 192.168.11.255 scope global eth1
==> hbmaster:     inet6 fe80::a00:27ff:feb3:8f14/64 scope link
==> hbmaster:     inet 172.16.1.7/24 brd 172.16.1.255 scope global eth2
==> hbmaster:     inet 10.0.0.17/24 scope global eth2
==> hbmaster:     inet6 fe80::a00:27ff:feda:b200/64 scope link
==> hbmaster: sshd neu laden:
==> hbmaster: [  OK  ]
==> hbmaster: ändere Passwort für Benutzer root.
==> hbmaster: passwd: alle Authentifizierungsmerkmale erfolgreich aktualisiert.
==> hbbackup: Importing base box 'centos-6.6-x86_64'...
==> hbbackup: Matching MAC address for NAT networking...
==> hbbackup: Setting the name of the VM: hbbackup
==> hbbackup: Clearing any previously set forwarded ports...
==> hbbackup: Fixed port collision for 22 => 2222. Now on port 2200.
==> hbbackup: Clearing any previously set network interfaces...
==> hbbackup: Preparing network interfaces based on configuration...
    hbbackup: Adapter 1: nat
    hbbackup: Adapter 2: bridged
    hbbackup: Adapter 3: hostonly
==> hbbackup: Forwarding ports...
    hbbackup: 22 (guest) => 2200 (host) (adapter 1)
==> hbbackup: Running 'pre-boot' VM customizations...
==> hbbackup: Booting VM...
==> hbbackup: Waiting for machine to boot. This may take a few minutes...
    hbbackup: SSH address: 127.0.0.1:2200
    hbbackup: SSH username: vagrant
    hbbackup: SSH auth method: private key
    hbbackup: Warning: Connection reset. Retrying...
    hbbackup: Warning: Connection aborted. Retrying...
    hbbackup: Warning: Remote connection disconnect. Retrying...
    hbbackup: Warning: Connection aborted. Retrying...
    hbbackup:
    hbbackup: Vagrant insecure key detected. Vagrant will automatically replace
    hbbackup: this with a newly generated keypair for better security.
    hbbackup:
    hbbackup: Inserting generated public key within guest...
    hbbackup: Removing insecure key from the guest if its present...
    hbbackup: Key inserted! Disconnecting and reconnecting using new SSH key...
==> hbbackup: Machine booted and ready!
==> hbbackup: Checking for guest additions in VM...
    hbbackup: The guest additions on this VM do not match the installed version of
    hbbackup: VirtualBox! In most cases this is fine, but in rare cases it can
    hbbackup: prevent things such as shared folders from working properly. If you see
    hbbackup: shared folder errors, please make sure the guest additions within the
    hbbackup: virtual machine match the version of VirtualBox you have installed on
    hbbackup: your host and reload your VM.
    hbbackup:
    hbbackup: Guest Additions Version: 4.3.28
    hbbackup: VirtualBox Version: 5.1
==> hbbackup: Setting hostname...
==> hbbackup: Configuring and enabling network interfaces...
    hbbackup: SSH address: 127.0.0.1:2200
    hbbackup: SSH username: vagrant
    hbbackup: SSH auth method: private key
==> hbbackup: Mounting shared folders...
    hbbackup: /vagrant => F:/study/vagrant
==> hbbackup: Running provisioner: shell...
    hbbackup: Running: inline script
==> hbbackup:     inet 127.0.0.1/8 scope host lo
==> hbbackup:     inet6 ::1/128 scope host
==> hbbackup:     inet 10.0.2.15/24 brd 10.0.2.255 scope global eth0
==> hbbackup:     inet6 fe80::a00:27ff:fe2d:823a/64 scope link
==> hbbackup:     inet 192.168.11.8/24 brd 192.168.11.255 scope global eth1
==> hbbackup:     inet6 fe80::a00:27ff:fec0:3738/64 scope link
==> hbbackup:     inet 172.16.1.8/24 brd 172.16.1.255 scope global eth2
==> hbbackup:     inet 10.0.0.18/24 scope global eth2
==> hbbackup:     inet6 fe80::a00:27ff:fed5:fbaf/64 scope link
==> hbbackup: sshd neu laden:
==> hbbackup: [  OK  ]
==> hbbackup: ändere Passwort für Benutzer root.
==> hbbackup: passwd: alle Authentifizierungsmerkmale erfolgreich aktualisiert.
```

## 2.5 关于 provision 常用命令

- vagrant provision `执行所有 vm provision 定义 shell 命令。`
- vagrant provision vm-machine `执行指定 vm provision 定义 shell 命令。`
- vagrant reload ---provision `重建所有 vm ，并执行所有 vm provision 定义 shell 命令.`
- vagrant reload vm-machine ---provision `重建指定 vm，并执行其 provision 定义 shell 命令。`

> 原文链接： <http://blog.csdn.net/54powerman/article/details/50676320>