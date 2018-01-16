---
layout: wiki
title: Vagrant
categories: vagrant
description: Vagrant 常用命令
keywords: vagrant
---

> vagrant 基本命令，根据操作的目的，可以对基本命令进行分类;

# 一、操作镜像

该命令有两个，用来管理本地镜像。

## 1. vagrant box

### 添加镜像到本地仓库

```shell
vagrant box add [box-name] [box 镜像文件或镜像名]
```

### 移除本地镜像

```shell
vagrant box remove [box-name]
```

### box 多版本共存的情况

如果 box 升级过，那么在 box list 中会出现两个同名，但版本不同的镜像。如：

```shell
$ vagrant box list |grp coreos
coreos-alpha                         (virtualbox, 745.1.0)
coreos-alpha                         (virtualbox, 928.0.0)
```

使用该镜像创建虚拟机的时候，默认会使用高版本的 box。
如果想使用低版本，需要修改 rantfile,指定 box-version
在 config.vm.box=xxx 下一行，如上面的例子中，在『config.vm.box = "coreos-alpha"』后面增加一行配置信息：
config.vm.box_version = "745.1.0"

同样，如果想删除一个 box，如下操作会失败：

```shell
$ vagrant box remove coreos-alpha
You requested to remove the box 'coreos-alpha' with provider
'virtualbox'. This box has multiple versions. You must
explicitly specify which version you want to remove with
the `--box-version` flag or specify the `--all` flag to remove all
versions. The available versions for this box are:

 * 745.1.0
 * 928.0.0
```

这时有两个选择：

- 删除所有同名镜像

>> vagrant box remove coreos-alpha --all

- 删除指定版本的镜像

>> vagrant box remove coreos-alpha --box-version=745.1.0

### 升级镜像

检查镜像是否有升级？

```shell
$ cd ~/vm/ubuntu
$ vagrant box outdated
Checking if box 'ubuntu/trusty64' is up to date...
A newer version of the box 'ubuntu/precise64' is available! You currently
have version '20160122.0.0'. The latest is version '20160209.0.0'. Run
`vagrant box update` to update.
```

中央仓库有新版更新了，手动更新 box 。更新的结果并不是替换旧版本，而是在本地仓库中增加了新版的 box 镜像。

```shell
$ cd ~/vm/ubuntu
$ vagrant box update
==> default: Checking for updates to 'ubuntu/precise64'
    default: Latest installed version: 20160120.0.0
    default: Version constraints:
    default: Provider: virtualbox
==> default: Updating 'ubuntu/precise64' with provider 'virtualbox' from version
==> default: '20160120.0.0' to '20160201.0.0'...
==> default: Loading metadata for box 'https://atlas.hashicorp.com/ubuntu/precise64?access_token=cXR0wCgWXoRpMw.atlasv1.ydBAS4ev1YCWzSK4S1l6iVjssRbO5Q50a8YVnEPqoyYjcQVeaMdEsiQ8rz8tOcSHLuY'
==> default: Adding box 'ubuntu/precise64' (v20160201.0.0) for provider: virtualbox
    default: Downloading: https://atlas.hashicorp.com/ubuntu/boxes/precise64/versions/20160201.0.0/providers/virtualbox.box
==> default: Successfully added box 'ubuntu/precise64' (v20160201.0.0) for 'virtualbox'!

$ vagrant box list | grep precise64
ubuntu/precise64                     (virtualbox, 20160120.0.0)
ubuntu/precise64                     (virtualbox, 20160201.0.0)
```

也可以检查本地仓库中的所有镜像是否有升级,使用 --global 开关，这时候不需要进入工作目录

```shell
$ vagrant box outdated --global
* 'ubuntu/trusty64' is outdated! Current: 20160122.0.0. Latest: 20160209.0.0
* 'ubuntu/precise64' (v20160201.0.0) is up to date
* 'pollyduan/bento_oracle_xe' wasn't added from a catalog, no version information
* 'phusion/ubuntu-14.04-amd64' (v2014.04.30) is up to date
* 'hashicorp/precise32' (v1.0.0) is up to date
* 'hashicorp/boot2docker' (v1.7.8) is up to date
* 'debian/jessie64' wasn't added from a catalog, no version information
* 'coreos-alpha' is outdated! Current: 928.0.0. Latest: 955.0.0
* 'centos7' wasn't added from a catalog, no version information
* 'centos65' wasn't added from a catalog, no version information
* 'centos64' wasn't added from a catalog, no version information
* 'bento/ubuntu-14.04' (v2.2.3) is up to date
* 'bento/opensuse-13.2-x86_64' (v2.2.1) is up to date
* 'bento/freebsd-10.2' (v2.2.3) is up to date
* 'bento/fedora-22' (v2.2.3) is up to date
* 'bento/debian-8.2' (v2.2.3) is up to date
* 'bento/centos-7.2' (v2.2.3) is up to date
* 'bento/centos-6.7' (v2.2.3) is up to date
```

不进入工作目录进行升级

```shell
$ vagrant box update --box coreos-alpha
Checking for updates to 'coreos-alpha'
Latest installed version: 928.0.0
Version constraints: > 928.0.0
Provider: virtualbox
Updating 'coreos-alpha' with provider 'virtualbox' from version
'928.0.0' to '955.0.0'...
Loading metadata for box 'http://alpha.release.core-os.net/amd64-usr/current/coreos_production_vagrant.json'
Adding box 'coreos-alpha' (v955.0.0) for provider: virtualbox
Downloading: http://alpha.release.core-os.net/amd64-usr/955.0.0/coreos_production_vagrant.box
Box download is resuming from prior download progress
Progress: 1% (Rate: 92912/s, Estimated time remaining: 0:38:09)
...
```

## 2. 打包镜像

创建 vm 以后，我们自己根据需要安装软件，配置环境，都一切就绪了。如何分发给小伙伴使用呢？这里就要涌到 package 命令了，把镜像打包分发。

```shell
$ ll ~/VirtualBox\ VMs/ |grep ubuntu
drwx------  6 pollyduan  staff  204  2  2 11:18 ubuntu_default_1453944793418_7699
```

先看一下我们的 vm 目录名，这里有个容易混淆的目录：

- Vagrantfile所在的目录——vagrant的工作目录
- 虚拟机文件所在的目录——virtualbox的工作目录

这两个目录名不一定相同，如果在 Vagrantfile 中指定了 vb.name，virtualbox 工作目录就取这个名字；否则，命名为：vagrant 工作目录_随机字符串。
或者，也可以使用 virtual box 的管理工具来看 vm 的名称。

```shell
$ VBoxManage list vms|grep ubuntu
"ubuntu_default_1453944793418_7699" {0362edc2-548b-400b-a55d-776b0a24cd8d}
```

package 命令要使用的是 virtual box 工作目录。

> 格式：vagrant package --base [virtualbox的工作目录] --output [保存的文件名，缺省为 package.box]

```shell
$ vagrant package --base ubuntu_default_1453944793418_7699 --output ubuntu_myproject_test.box
==> ubuntu_default_1453944793418_7699: Clearing any previously set forwarded ports...
==> ubuntu_default_1453944793418_7699: Exporting VM...
==> ubuntu_default_1453944793418_7699: Compressing package to: /Users/pollyduan/vm/tmp/ubuntu_myproject_test.box
```

导出后，可以通过 IM、ftp 或其他方式分发给小伙伴，那么大家使用的环境就是一致的了。

# 二、操作虚拟机

## 1. 启动 vm

```shell
# 对于单虚拟机
$ vagrant up

# 如果同一个 Vagrantfile 定义了一个以上的虚拟机，则：
$ vagrant up [vm-name]

# 其他命令类似。如果省略 vm-name ，则依次启动所有 vm。

# 重启
$ vagrant reload [vm-name]

# 关机
$ vagrant halt [vm-name]

# 销毁虚拟机
$ vagrant destroy [vm-name]

# ssh 登录虚拟机
$ vagrant ssh [vm-name]
```

## 2. 休眠与唤醒

这一对冤家也无需多说，对于开发环境来说，个人觉得用处不是很大。

```shell
$ vagrant suspend
==> default: Saving VM state and suspending execution...
$ vagrant status
Current machine states:

default                   saved (virtualbox)

To resume this VM, simply run `vagrant up`.
$ vagrant resume
==> default: Resuming suspended VM...
==> default: Booting VM...
…...
```

## 3. 快照

vagrant snapshot 命令是 vm 的月光宝盒，如果 vm 中有任务没有跑完，需要关闭 virtual box ，就可以给 vm 做一个快照，保存 vm 当前所有的状态，在 virtualbox 重新启动后，再回复快照。

```shell
# 查看当前保存的快照
$ vagrant snapshot list
==> default: No snapshots have been taken yet!

# 创建一个命名快照
$ vagrant snapshot save shot1
==> default: Snapshotting the machine as 'shot1'...
$ vagrant snapshot list
shot1

# 恢复快照
$ vagrant snapshot restore shot1
==> default: Forcing shutdown of VM...
==> default: Restoring the snapshot 'shot1'...
…...

# 恢复后，快照会一直存在，直到你手动删除它。

# 删除快照
$ vagrant snapshot delete shot1
==> default: Deleting the snapshot 'shot1'...
Progress: 90%
Progress: 100%
==> default: Snapshot deleted!
```

这个操作会删除持久化的数据文件，稍微有点慢，耐心等待。这个内在的原理没有深入研究，有点不太理解，删除一个文件理论上应该比保存一个文件更快才对。

## 4. 盗梦空间

push 和 pop ，每调用一次 push 命令会自动创建一个命名快照，名为：push+一串随机数，如：push_1455524411_6632；每调用一次 pop ，会逐级恢复到最新的快照，并删除快照。看例子：

```shell
$ vagrant snapshot list
==> default: No snapshots have been taken yet!
$ vagrant snapshot push
==> default: Snapshotting the machine as 'push_1455525041_2882'...
$ vagrant snapshot list
push_1455525041_2882
$ vagrant snapshot push
==> default: Snapshotting the machine as 'push_1455525049_7456'...
$ vagrant snapshot push
==> default: Snapshotting the machine as 'push_1455525058_6891'...
$ vagrant snapshot list
push_1455525041_2882
push_1455525049_7456
push_1455525058_6891
$ vagrant snapshot pop
==> default: Forcing shutdown of VM...
==> default: Restoring the snapshot 'push_1455525058_6891'...
==> default: Deleting the snapshot 'push_1455525058_6891'...
==> default: Snapshot deleted!
$ vagrant snapshot list
push_1455525041_2882
push_1455525049_7456
```

> Tips: 文本的日志看起来还不够形象，在 push 三个 snapshot 后在 virtual box 中是树形显示的；每次 pop ，树枝会逐级退回，看起来更像穿越的感觉。
> Tips: 在 pop 清空之前，随时可以通过 restore 恢复其中一个快照，同样快照不会删除；不影响 pop 的测试。

## 5. 远程连接分享

远程连接通过 share connect 两个命令可以实现通过本机 vagrant 连接另外一台 host 上的虚机。

```shell
# 允许 ssh 连接
$ vagrant share --ssh
==> default: Detecting network information for machine...
    default: Local machine address: 127.0.0.1
    default:
    default: Note: With the local address (127.0.0.1), Vagrant Share can only
    default: share any ports you have forwarded. Assign an IP or address to your
    default: machine to expose all TCP ports. Consult the documentation
    default: for your provider ('virtualbox') for more information.
    default:
    default: An HTTP port couldn't be detected! Since SSH is enabled, this is
    default: not an error. If you want to share both SSH and HTTP, please set
    default: an HTTP port with `--http`.
    default:
    default: Local HTTP port: disabled
    default: Local HTTPS port: disabled
    default: SSH Port: 2200
    default: Port: 2200
==> default: Generating new SSH key...
    default: Please enter a password to encrypt the key: [输入授权密码]
    default: Repeat the password to confirm:[再输一次]
    default: Inserting generated SSH key into machine...
==> default: Checking authentication and authorization...
==> default: Creating Vagrant Share session...
    default: Share will be at: vile-ibex-8238
==> default: Your Vagrant Share is running! Name: vile-ibex-8238
==> default:
==> default: You're sharing your Vagrant machine in "restricted" mode. This
==> default: means that only the ports listed above will be accessible by
==> default: other users (either via the web URL or using `vagrant connect`).
==> default:
==> default: You're sharing with SSH access. This means that another user
==> default: simply has to run `vagrant connect --ssh vile-ibex-8238`
==> default: to SSH to your Vagrant machine.
==> default:
==> default: Because you encrypted your SSH private key with a password,
==> default: the other user will be prompted for this password when they
==> default: run `vagrant connect --ssh`. Please share this password with them
==> default: in some secure way.
```

> Tips: 你可以通过--name指定一个名称，否则会随机生成一个共享名，如本例中的 vile-ibex-8238

```shell
# 连接远端 ssh 虚拟机
$ vagrant connect --ssh vile-ibex-8238 --static-ip 10.2.136.211
Loading share 'vile-ibex-8238'...
The SSH key to connect to this share is encrypted. You will require
the password entered when creating to share to decrypt it. Verify you
access to this password before continuing.

Press enter to continue, or Ctrl-C to exit now.[回车]
Password for the private key:[输入授权密码]
Executing SSH...
vagrant@vagrant-ubuntu-trusty-64:~$
```

## 6. 共享http

vagrant share 可以把 host 主机的 http 开放到远端，供任何人访问，这好像跟 vm 没什么关系，但的确它发生了。

```shell
$ ~/apache-tomcat-8.0.28/bin/startup.sh
$ vagrant share --http 80
==> default: Detecting network information for machine...
    default: Local machine address: 127.0.0.1
    default:
    default: Note: With the local address (127.0.0.1), Vagrant Share can only
    default: share any ports you have forwarded. Assign an IP or address to your
    default: machine to expose all TCP ports. Consult the documentation
    default: for your provider ('virtualbox') for more information.
    default:
    default: Local HTTP port: 80
    default: Local HTTPS port: disabled
    default: Port: 2200
==> default: Checking authentication and authorization...
==> default: Creating Vagrant Share session...
    default: Share will be at: enchanting-buffalo-1493
==> default: Your Vagrant Share is running! Name: enchanting-buffalo-1493
==> default: URL: http://enchanting-buffalo-1493.vagrantshare.com
==> default:
==> default: You're sharing your Vagrant machine in "restricted" mode. This
==> default: means that only the ports listed above will be accessible by
==> default: other users (either via the web URL or using `vagrant connect`).
```

我的 host 电脑在内网，在外网的任意一台电脑上，访问：

 <http://enchanting-buffalo-1493.vagrantshare.com>

奇迹发生了。

话说这个有什么用呢？别忘记，vagrant 有一个端口映射的功能，在后面的 Vagrantfile 配置里会提到，这样做的结果，就是可以在互联网任意一个角落可以访问到你的虚机的 http 服务。

## 7. windows 相关的操作

powershelgl 和 rdp 是 windows vm 相关的操作，未做测试，忽略。

## 8. 虚机环境部署

provision 用于通过 Vagrantfile 配置文件，对 vm 进行部署，如安装软件，发布应用等，在这里不多说，专门一章来记录。

## 9. 指定vmid操作虚拟机

在 [查看虚拟机状态](#3. 查看虚拟机状态) 中，我们可以看到当前工作机中的所有虚机，其中第一列数据为 vmid，我们可以无需进入 vagrant 工作目录，操作这些虚机。如：

```shell
vagrant up 63093ce
```

该方式适用于前面提到的 up、reload、halt、destroy 等命令。

# 三、监控虚拟机

## 1. 查看 sshd 配置信息

```shell
$ vagrant ssh-config
Host default
  HostName 127.0.0.1
  User vagrant
  Port 2222
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile "/Users/pollyduan/vm/ubuntu/.vagrant/machines/default/virtualbox/private_key"
  IdentitiesOnly yes
  LogLevel FATAL
```

## 2. 查看虚拟机开放的端口

```shell
$ vagrant port
The forwarded ports for the machine are listed below. Please note that
these values may differ from values configured in the Vagrantfile if the
provider supports automatic port collision detection and resolution.

    22 (guest) => 2222 (host)
```

## 3. 查看虚拟机状态

```shell
# 查看当前 vm 状态
$ cd ~/vm/ubuntu
$ vagrant status
Current machine states:

default                   poweroff (virtualbox)

The VM is powered off. To restart the VM, simply run `vagrant up`
```

状态可能是：

- not create  `执行 vagrant init 命令后，从未启动过`
- poweroff    `关机`
- running     `运行中`
- saved       `休眠`

## 4. 查看全部虚机状态

此命令无需进入 vagrant 工作目录。

```shell
$ vagrant global-status
id       name       provider   state    directory
--------------------------------------------------------------------------------
be5dee2  mfsmaster  virtualbox poweroff /Users/pollyduan/vm/mfs
a523de6  mfschunk1  virtualbox poweroff /Users/pollyduan/vm/mfs
8377e0d  mfschunk2  virtualbox poweroff /Users/pollyduan/vm/mfs
b772b1f  metalogger virtualbox poweroff /Users/pollyduan/vm/mfs
8a5f10e  mfsclient  virtualbox poweroff /Users/pollyduan/vm/mfs
63093ce  default    virtualbox poweroff /Users/pollyduan/vm/ubuntu
```

你可能会发现，为什么有的是 default ，有的是有名字的。这就是因为 mfsxxxx 是在 vagrantfile 中指定了 vb.name ，他对应的 virtualbox 工作目录也是这个值，而 ubuntu 这个虚机没有指定，所以是 default，而且其 virtualbox 工作目录也是比较长的——ubuntu_default_1453944793418_7699。

global-status 统计信息不是实时的，所有不能保证数据是绝对准确的。如果在 vagrant up 启动后，我们在 virtualbox 管理终端关闭 vm ，global-status 是捕获不到的，它还是会显示 running 状态。
截至 1.8.1 还是这样的，应该算是一个 bug 。处女座可能无法接受这个现实，那么你可以进入 vagrant 工作目录，手动再指定一次 vagrant halt ，状态就同步了。

# 四、其他命令

## 1. 获取帮助

help

## 2. 登录到中央仓库

login

## 3. 插件管理

plugin

## 4. 发布镜像到中央仓库

push

## 5. 获取版本信息

version

# 五、相关资源

## 软件下载

- 官网 <http://www.vagrantup.com>
- 官方下载地址 <https://www.vagrantup.com/downloads.html>
- 旧版本下载 <https://releases.hashicorp.com/vagrant/>

## Box下载

- 官方仓库 <https://atlas.hashicorp.com/boxes/search>
- 官方镜像 <https://vagrantcloud.com/boxes/search>
- 第三方仓库 <http://www.vagrantbox.es/>

> 原文链接：<http://blog.csdn.net/54powerman/article/details/50669807>