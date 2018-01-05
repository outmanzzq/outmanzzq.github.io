---
layout: post
title: 关闭ext4文件系统的日志功能
categories: ext4
description: 关闭ex4文件系统的日志功能
keywords: ext4, format
---

> 手头有一32G TF卡，准备插在路由器USB口做网盘使用，因fat32 不支持单文件超过4G的缺点，查阅各磁盘文件系统特点后，最终决定格式化成ext4，同时关闭日志功能，提高IO利用率

# 常见磁盘文件系统简介

- **FAT16**

我们以前用的DOS、Windows 95都使用FAT16文件系统，现在常用的Windows 98/2000/XP等系统均支持FAT16文件系统。它最大可以管理大到2GB的分区，但每个分区最多只能有65525个簇(簇是磁盘空间的配置单位)。随着硬盘或分区容量的增大，每个簇所占的空间将越来越大，从而导致硬盘空间的浪费。

- **FAT32**

随着大容量硬盘的出现，从Windows 98开始，FAT32开始流行。它是FAT16的增强版本，可以支持大到2TB(2048G的分区。FAT32使用的簇比FAT16小，从而有效地节约了硬盘空间。

- **NTFS**

最大分区2TB。另：简单卷最大2TB，动态卷最大16TB。

- **Ext4**

是 Ext3 的改进版，修改了 Ext3 中部分重要的数据结构，而不仅仅像 Ext3 对 Ext2 那样，只是增加了一个日志功能而已。Ext4 可以提供更佳的性能和可靠性，还有更为丰富的功能：

1. 与 Ext3 兼容。执行若干条命令，就能从 Ext3 在线迁移到 Ext4，而无须重新格式化磁盘或重新安装系统。原有 Ext3 数据结构照样保留，Ext4 作用于新数据，当然，整个文件系统因此也就获得了 Ext4 所支持的更大容量。

2. 更大的文件系统和更大的文件。较之 Ext3 目前所支持的最大 16TB 文件系统和最大 2TB 文件，Ext4 分别支持 1EB（1,048,576TB， 1EB=1024PB， 1PB=1024TB）的文件系统，以及 16TB 的文件。

3. 无限数量的子目录。Ext3 目前只支持 32,000 个子目录，而 Ext4 支持无限数量的子目录。

> Ext2 是 GNU/Linux 系统中标准的文件系统，其特点为存取文件的性能极好，对于中小型的文件更显示出优势，这主要得利于其簇快取层的优良设计。其单一文件大小与文件系统本身的容量上限与文件系统本身的簇大小有关，在一般常见的 x86 电脑系统中，簇最大为 4KB, 则单一文件大小上限为 2048GB, 而文件系统的容量上限为 16384GB。但由于目前核心 2.4 所能使用的单一分割区最大只有 2048GB，因此实际上能使用的文件系统容量最多也只有 2048GB。

# 一、环境说明

- OS： CentOS 7 x64 （vmware虚拟机）
- TF卡：32G (金士顿)

# 二、操作过程

## 1、格式化TF卡为 ext4

```shell
# 查看磁盘，TF卡为/dev/sdb
[root@centos7 ~]# fdisk -l

磁盘 /dev/sda：21.5 GB, 21474836480 字节，41943040 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x000695a8

   设备 Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     1026047      512000   83  Linux
/dev/sda2         1026048    41943039    20458496   8e  Linux LVM

磁盘 /dev/mapper/centos-root：18.8 GB, 18756927488 字节，36634624 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节


磁盘 /dev/mapper/centos-swap：2147 MB, 2147483648 字节，4194304 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节


磁盘 /dev/sdb：31.4 GB, 31439454208 字节，61405184 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x0e8ca368

   设备 Boot      Start         End      Blocks   Id  System
/dev/sdb1   *          63    61405183    30702560+   c  W95 FAT32 (LBA)

# 格式化
[root@centos7 ~]# mkfs.ext4 /dev/sdb
mke2fs 1.42.9 (28-Dec-2013)
/dev/sdb is entire device, not just one partition!
无论如何也要继续? (y,n) n
[root@centos7 ~]# mkfs.ext4 /dev/sdb
mke2fs 1.42.9 (28-Dec-2013)
/dev/sdb is entire device, not just one partition!
无论如何也要继续? (y,n) y
文件系统标签=
OS type: Linux
块大小=4096 (log=2)
分块大小=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
1921360 inodes, 7675648 blocks
383782 blocks (5.00%) reserved for the super user
第一个数据块=0
Maximum filesystem blocks=2155872256
235 block groups
32768 blocks per group, 32768 fragments per group
8176 inodes per group
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,

Allocating group tables: 完成
正在写入inode表: 完成
Creating journal (32768 blocks): 完成
Writing superblocks and filesystem accounting information: 完成
```

## 2、关闭 ext4 日志功能

```shell
# 查看是否开启日志功能
[root@centos7 ~]# dumpe2fs /dev/sdb | grep 'Filesystem features' | grep 'has_journal'
dumpe2fs 1.42.9 (28-Dec-2013)
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype extent 64bit flex_bg sparse_super large_file huge_file uninit_bg dir_nlink extra_isize

# 关闭日志功能
[root@centos7 ~]# tune2fs -O ^has_journal /dev/sdb
tune2fs 1.42.9 (28-Dec-2013)

#检查
[root@centos7 ~]# dumpe2fs /dev/sdb | grep 'Filesystem features' | grep 'has_journal'
dumpe2fs 1.42.9 (28-Dec-2013)

# 弹出TF卡
[root@centos7 ~]# eject /dev/sdb
```

## 3、如果要开启日志功能

```shell
# 开启日志
[root@centos7 ~]# tune2fs -O has_journal /dev/sdb
tune2fs 1.42.9 (28-Dec-2013)
Creating journal inode: 完成

#检查
[root@centos7 ~]# dumpe2fs /dev/sdb | grep 'Filesystem features' | grep 'has_journal'
dumpe2fs 1.42.9 (28-Dec-2013)
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype extent 64bit flex_bg sparse_super large_file huge_file uninit_bg dir_nlink extra_isize
```

## 4、Windows 系统读取 ext4 格式磁盘工具ext2fsd （注意：ext4 只读不支持写入！）

<https://sourceforge.net/projects/ext2fsd/files/latest/download?source=files>

> 参考链接：<http://www.cnblogs.com/jusonalien/p/5032973.html>