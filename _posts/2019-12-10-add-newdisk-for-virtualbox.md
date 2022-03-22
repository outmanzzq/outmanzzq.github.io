---
layout: post
title: virtualbox 虚拟机命令行添加新硬盘
categories: vbox,virtualbox,vagrant
description: virtualbox 虚拟机命令行添加新硬盘
keywords: vbox,virtualbox,vagrant
---

> 出于通过 vagrant box 创建 VM ，测试 Ceph 需要，需给 vm 添加额外硬盘以做测试。

# 环境说明

- 宿主机OS：CentOS Linux 7.7.1908
- Vagrant：2.2.5
- VirtualBox：6.0.2r128162
- 工作目录：/server/vagrant/ceph
- Vagrantfile 文件内容

```bash
# -*- mode: ruby -*-
# vi: set ft=ruby :

node_servers = {
    :vm1-master => '172.18.128.254',
}

Vagrant.configure("2") do |config|
  config.vm.box = "centos/7.5_18.09_x86-64"

  node_servers.each do |node_servers_name, node_server_ip|
    config.vm.define node_servers_name do |node_config|
      node_config.vm.hostname = "#{node_servers_name.to_s}"
      node_config.vm.network :private_network, ip: node_server_ip
      node_config.vm.provider "virtualbox" do |vb|
        vb.name = node_servers_name.to_s
      end

      node_config.vm.provision "shell", inline: <<-SHELL
        sudo -i
        sed -i  's/^#PermitRootLogin.*/PermitRootLogin yes/g' /etc/ssh/sshd_config && \
        sed -n '/PermitRootLogin/p' /etc/ssh/sshd_config && \
        sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config && \
        sed -n '/^PasswordAuthentication yes/p' /etc/ssh/sshd_config && \
        systemctl restart sshd && \
        echo "root:vagrant" |sudo chpasswd
  
        # set synctime
        #yum install ntp -y
        #systemctl enable ntpd
        #sed -i 's/\(^OPTIONS=\).*/\1"-g -x"/g' /etc/sysconfig/ntpd
        #systemctl restart ntpd
        ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
  
        # stop firewall
        systemctl stop firewalld
        systemctl disable firewalld
  
        # disable selinux
        sed -i 's/\(^SELINUX=\).*/\1disabled/g' /etc/selinux/config
  
        # disable swap
        #swapoff -a
        #sed -i /swap/s/^/#/g /etc/fstab
        #sh /vagrant/scripts/install_dockerCE_docker-compose.sh
      SHELL
    end
  end
end
```

# 一、具体操作过程

> 添加硬盘分为3步：

1. 创建虚拟磁盘镜像
2.附加磁盘镜像到虚拟机存储控制器
3.进入虚拟机挂载新磁盘

## 1. 初始化虚拟机

```bash
# 进入工作目录
cd /server/vagrant/ceph

# 校验并初始化
vagrant validate

vagrant up

# 停机并准备添加新硬盘
vagrant halt
```

## 2. 创建虚拟磁盘

这一步最简单，在母机上执行命令:

```VBoxManage createhd --filename /storage/vms/disk50g --size 50000 --format VMDK```

这句的意思是，创建一个虚拟磁盘 /storage/vms/disk50g 容量 50000MB 格式是 VMDK ,执行成功后会有一个文件 /storage/vms/disk50g.vmdk 记好它，路径当然根据个人需求.

## 3. 添加磁盘到虚拟机

这个步骤相当于我们装电脑时把硬盘插到主板上，怎么插呢？首先你得有一块硬盘，就是上面创建的那个文件，然后你要插到主板上哪个口子？我们知道主板有 ide 或 sata 磁盘驱动器，假如是sata，它还有好几个口，这些口都有编号 1,2,3,4,5 。了解了这些，下面的命令就比较好理解了。

首先，看一下虚拟机的信息

```VBoxManage list vms #列出virtualbox下所有的虚拟机```

找到你想弄的虚拟机，记下它的名字 ，像我这里这样的

```bash
"vm2_default_1471395575217_38235" {98d2cc97-beda-4be1-876d-d5cd7200837e}
"vm1-master" {9f172263-c4d6-4af5-a6ba-7ff0df695d37}
```

我要弄的是 "vm1-master"

```VBoxManage showvminfo vm1-master```

根据输出信息，大概找到 Storage Controller 那一块：

```bash
Storage Controller Name (0):            IDE Controller
Storage Controller Type (0):            PIIX4
Storage Controller Instance Number (0): 0
Storage Controller Max Port Count (0):  2
Storage Controller Port Count (0):      2
Storage Controller Bootable (0):        on
```

我发现这个虚拟机只有 IDE Controller ，要插硬盘还得先关闭虚拟机，麻烦啊，为了以后一劳永逸，我决定给虚拟机增加一个 Sata Controller ，首先 我关闭了虚拟机，当然不关闭虚拟机不知道行不行，下次试试。

```VBoxManage storagectl vm1-master --name "SATA Controller" --add sata --portcount 5 --controller IntelAhci --bootable on```

这样就增加了个 sata 驱动器。

继续我们把硬盘插上去

```VBoxManage storageattach vm1-master --storagectl "SATA Controller" --type hdd --medium /storage/vms/disk50g.vmdk --port 1 --device 0```

需要了解的有几个参数:

- vm1-master 就是你虚拟机的名字，上面有提到
- --storagectl 参数就是磁盘控制器的名字，参考上一条命令中的 --name
- --type hdd 表示是硬盘，因为这个命令不仅仅可以插硬盘，还能插光驱等，所以要指定插的是什么
- --medium 这里指定的是虚拟磁盘的文件名，想想一下你手里拿着的硬盘
- --port 表示插在哪个端口，上面我们创建 sata 控制器的时候通过 --portcount 5 开放了5个端口，相当于主板上的5个sata- 接口，此时我们用第1个
- --device 0 设备id，设为0 就可以了，我也不知道什么意思
- 如果没有错误的话，这里就成功添加了一块硬盘到虚拟机了，启动虚拟机，这里我用的是vagrant来管理虚拟机的。

```bash
vagrant up
vagrant ssh
```

登陆虚拟机后，执行 sudo fdisk -l ，可以看到我们刚添加的磁盘 /dev/sdb

下面我们把它分区

```sudo fdisk /dev/sdb```

出现提示，"command m for help .." ，直接输入 "n" ，

进行分区，如果只要分一个区的话，最好办，一路enter ,

又回到 "command m for help .." 的时候，输入 "w" 并回车，分区完成。

格式化分区:

```bash
sudo  mkfs.xfs  /dev/sdb1
```

挂载分区，并设置开机自动挂载：

```bash
# 设置开机自动挂载
sudo echo '/dev/sdb1    /home/wwwroot    ext4    defaults 1 2' >/etc/fstab

# 挂载分区
sudo mount /dev/sdb1  /home/wwwroot
```

# 二、进阶篇

通过 Vagrantfile 自动添加硬盘,目标：

- Software:  CentOS 7.6 on all nodes
- Host Services:  
  - node01: Management Server (IP:172.16.80.111)
  - node02: Metadata Server (IP:172.16.80.112)   # 额外添加 2 块硬盘
  - node03: Storage Server (IP:172.16.80.113)    # 额外添加 4 块硬盘
  - node04: Client (IP:172.16.80.114)
  - node05: Admon Server (IP:172.16.80.115) (可选项)

- Storage:
  - Storage servers with RAID-6, xfs, mounted to "/data".
  - Metadata servers with RAID-1, ext4, mounted to "/mnt/md0".

Vagrantfile 文件内容

```YAML
# -*- mode: ruby -*-
# vi: set ft=ruby :

common_servers = {
    :bgfsmanager => '172.16.80.111', # Management Server
    :bgfsclient => '172.16.80.114', # Client
    :bgfsadmon => '172.16.80.115'  # Admon Server
}

metadata_servers = {
    :bgfsmetadata => '172.16.80.112', # Metadata Server with RAID-1, ext4, mounted to "/mnt/md0"
}

storage_servers = {
    :bgfstorage => '172.16.80.113', # Storage Server with RAID-6, xfs, mounted to "/data"
}

metadata_secondDisk='./msecondDisk.vdi'   # metadata 第二颗硬盘
metadata_thirdDisk='./mthirdDisk.vdi'     # metadata 第三颗硬盘

storage_secondDisk='./ssecondDisk.vdi'   # storage 第二颗硬盘
storage_thirdDisk='./sthirdDisk.vdi'     # storage 第三颗硬
storage_fourthDisk='./sfourthDisk.vdi'   # storage 第四颗硬盘
storage_fivethDisk='./sfivethDisk.vdi'   # storage 第五颗硬盘


Vagrant.configure("2") do |config|
  config.vm.box = "CentOS/7.6_19.02_x86-64"

  common_servers.each do |beegfs_servers_name, beegfs_server_ip|
    config.vm.define beegfs_servers_name do |beegfs_config|
      beegfs_config.vm.hostname = "#{beegfs_servers_name.to_s}"
      beegfs_config.vm.network :private_network, ip: beegfs_server_ip
      beegfs_config.vm.provider "virtualbox" do |vb|
        vb.name = beegfs_servers_name.to_s
        #vb.memory = 4096

        if beegfs_servers_name == 'bgfsmanager' then
           echo beegfs_servers_name
           exit
        end
      end
    end

    config.vm.provision "shell", inline: <<-SHELL
      set -e
      sudo -i

      sed -i  's/^#PermitRootLogin.*/PermitRootLogin yes/g' /etc/ssh/sshd_config && \
      sed -n '/PermitRootLogin/p' /etc/ssh/sshd_config && \
      sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config && \
      sed -n '/^PasswordAuthentication yes/p' /etc/ssh/sshd_config && \
      systemctl restart sshd && \
      echo "root:vagrant" | chpasswd

      # set synctime
      yum install ntp -y
      systemctl enable ntpd
      sed -i 's/\(^OPTIONS=\).*/\1"-g -x"/g' /etc/sysconfig/ntpd
      systemctl restart ntpd
      ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

      # stop firewall
      systemctl stop firewalld

      # disable selinux
      sed -i 's|^SELINUX=.*|SELINUX=disabled|g' /etc/selinux/config

      # disable swap
       swapoff -a
       sed -i /swap/s/^/#/g /etc/fstab

    SHELL
  end

  metadata_servers.each do |beegfs_servers_name, beegfs_server_ip|
    config.vm.define beegfs_servers_name do |beegfs_config|
      beegfs_config.vm.hostname = "#{beegfs_servers_name.to_s}"
      beegfs_config.vm.network :private_network, ip: beegfs_server_ip
      beegfs_config.vm.provider "virtualbox" do |vb|
        vb.name = beegfs_servers_name.to_s
        #vb.memory = 4096

        # add secondDisk
        unless File.exist?(metadata_secondDisk)
          vb.customize ['createhd', '--filename', metadata_secondDisk, '--variant', 'Fixed', '--size', 1 * 1024 ]
        end
        vb.customize ['storageattach', :id,  '--storagectl', 'IDE', '--port', 0, '--device', 1, '--type', 'hdd', '--medium', metadata_secondDisk]

        # add thirdDisk
        unless File.exist?(metadata_thirdDisk)
          vb.customize ['createhd', '--filename', metadata_thirdDisk, '--variant', 'Fixed', '--size', 1 * 1024 ]
        end
        vb.customize ['storageattach', :id,  '--storagectl', 'IDE', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', metadata_thirdDisk]

      end
    end

    config.vm.provision "shell", inline: <<-SHELL
      set -e
      sudo -i

      sed -i  's/^#PermitRootLogin.*/PermitRootLogin yes/g' /etc/ssh/sshd_config && \
      sed -n '/PermitRootLogin/p' /etc/ssh/sshd_config && \
      sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config && \
      sed -n '/^PasswordAuthentication yes/p' /etc/ssh/sshd_config && \
      systemctl restart sshd && \
      echo "root:vagrant" | chpasswd

      # set synctime
      yum install ntp -y
      systemctl enable ntpd
      sed -i 's/\(^OPTIONS=\).*/\1"-g -x"/g' /etc/sysconfig/ntpd
      systemctl restart ntpd
      ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

      # stop firewall
      systemctl stop firewalld

      # disable selinux
      sed -i 's|^SELINUX=.*|SELINUX=disabled|g' /etc/selinux/config

      # disable swap
       swapoff -a
       sed -i /swap/s/^/#/g /etc/fstab

    SHELL
  end

  storage_servers.each do |beegfs_servers_name, beegfs_server_ip|
    config.vm.define beegfs_servers_name do |beegfs_config|
      beegfs_config.vm.hostname = "#{beegfs_servers_name.to_s}"
      beegfs_config.vm.network :private_network, ip: beegfs_server_ip
      beegfs_config.vm.provider "virtualbox" do |vb|
        vb.name = beegfs_servers_name.to_s
        #vb.memory = 4096

        # add secondDisk
        unless File.exist?(storage_secondDisk)
          vb.customize ['createhd', '--filename', storage_secondDisk, '--variant', 'Fixed', '--size', 1 * 1024 ]
        end
        vb.customize ['storageattach', :id,  '--storagectl', 'IDE', '--port', 0, '--device', 1, '--type', 'hdd', '--medium', storage_secondDisk]

        # add thirdDisk
        unless File.exist?(storage_thirdDisk)
          vb.customize ['createhd', '--filename', storage_thirdDisk, '--variant', 'Fixed', '--size', 1 * 1024 ]
        end
        vb.customize ['storageattach', :id,  '--storagectl', 'IDE', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', storage_thirdDisk]

        # add fourthDisk
        unless File.exist?(storage_fourthDisk)
          vb.customize ['createhd', '--filename', storage_fourthDisk, '--variant', 'Fixed', '--size', 1 * 1024 ]
        end
        vb.customize ['storageattach', :id,  '--storagectl', 'IDE', '--port', 1, '--device', 1, '--type', 'hdd', '--medium', storage_fourthDisk]

        # add fivethDisk
        unless File.exist?(storage_fivethDisk)
          vb.customize ['createhd', '--filename', storage_fivethDisk, '--variant', 'Fixed', '--size', 1 * 1024 ]
        end
        #vb.customize ['storageattach', :id,  '--storagectl', 'IDE', '--port', 1, '--device', 1, '--type', 'hdd', '--medium', storage_fivethDisk]

      end
    end

    config.vm.provision "shell", inline: <<-SHELL
      set -e
      sudo -i

      sed -i  's/^#PermitRootLogin.*/PermitRootLogin yes/g' /etc/ssh/sshd_config && \
      sed -n '/PermitRootLogin/p' /etc/ssh/sshd_config && \
      sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config && \
      sed -n '/^PasswordAuthentication yes/p' /etc/ssh/sshd_config && \
      systemctl restart sshd && \
      echo "root:vagrant" | chpasswd

      # set synctime
      yum install ntp -y
      systemctl enable ntpd
      sed -i 's/\(^OPTIONS=\).*/\1"-g -x"/g' /etc/sysconfig/ntpd
      systemctl restart ntpd
      ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

      # stop firewall
      systemctl stop firewalld

      # disable selinux
      sed -i 's|^SELINUX=.*|SELINUX=disabled|g' /etc/selinux/config

      # disable swap
       swapoff -a
       sed -i /swap/s/^/#/g /etc/fstab

    SHELL
  end
end
```

> 说明：
> 1. 受 IDE 控制器盘符号限制，只能添加 4 块硬盘（00，01，10，11）
> 2. SATA 控制器可添加多个，但目前未找到添加方法。。
> 3. 查看 VBox 硬盘控制器命令:  

```bash
VBoxManager showvminfo + vmname |grep 'Storage Controller'
```

> 参考链接：<https://my.oschina.net/cxz001/blog/734167>
> <http://www.kclouder.cn/centos7-beegfs/>