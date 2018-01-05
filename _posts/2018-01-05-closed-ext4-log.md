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

## 1、TF卡分区

```shell
[root@centos7 ~]# fdisk -l

Disk /dev/sda: 8589 MB, 8589934592 bytes
255 heads, 63 sectors/track, 1044 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x0001a415

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *           1          26      204800   83  Linux
Partition 1 does not end on cylinder boundary.
/dev/sda2              26          91      524288   82  Linux swap / Solaris
Partition 2 does not end on cylinder boundary.
/dev/sda3              91        1045     7658496   83  Linux

Disk /dev/sdb: 31.4 GB, 31439454208 bytes
64 heads, 32 sectors/track, 29983 cylinders
Units = cylinders of 2048 * 512 = 1048576 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x0635d4c6

   Device Boot      Start         End      Blocks   Id  System
[root@centos7 ~]# fdisk /dev/sdb

WARNING: DOS-compatible mode is deprecated. Its strongly recommended to
         switch off the mode (command 'c') and change display units to
         sectors (command 'u').

Command (m for help): p

Disk /dev/sdb: 31.4 GB, 31439454208 bytes
64 heads, 32 sectors/track, 29983 cylinders
Units = cylinders of 2048 * 512 = 1048576 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x0635d4c6

   Device Boot      Start         End      Blocks   Id  System

Command (m for help): n
Command action
   e   extended
   p   primary partition (1-4)
p
Partition number (1-4): 1
First cylinder (1-29983, default 1):
Using default value 1
Last cylinder, +cylinders or +size{K,M,G} (1-29983, default 29983):
Using default value 29983

Command (m for help): p

Disk /dev/sdb: 31.4 GB, 31439454208 bytes
64 heads, 32 sectors/track, 29983 cylinders
Units = cylinders of 2048 * 512 = 1048576 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x0635d4c6

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1               1       29983    30702576   83  Linux

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
```

## 2、格式化TF卡为 ext4

```shell
[root@centos7 ~]# mkfs.ext4 /dev/sdb1
mke2fs 1.41.12 (17-May-2010)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
1921360 inodes, 7675644 blocks
383782 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=4294967296
235 block groups
32768 blocks per group, 32768 fragments per group
8176 inodes per group
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000

Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done

This filesystem will be automatically checked every 21 mounts or
180 days, whichever comes first.  Use tune2fs -c or -i to override.
```

## 2、关闭 ext4 日志功能

```shell
# 查看是否开启日志功能
[root@centos7 ~]# dumpe2fs /dev/sdb1 | grep 'Filesystem features' | grep 'has_journal'
dumpe2fs 1.41.12 (17-May-2010)
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype extent flex_bg sparse_super large_file huge_file uninit_bg dir_nlink extra_isize

# 关闭日志功能
[root@centos7 ~]# tune2fs -O ^has_journal /dev/sdb1
tune2fs 1.41.12 (17-May-2010)

#检查
[root@centos7 ~]# dumpe2fs /dev/sdb1 | grep 'Filesystem features' | grep 'has_journal'
dumpe2fs 1.41.12 (17-May-2010)

# 弹出TF卡
[root@centos7 ~]# eject /dev/sdb
```

## 3、如果要开启日志功能

```shell
# 开启日志
[root@centos7 ~]# tune2fs -O has_journal /dev/sdb1
tune2fs 1.42.9 (28-Dec-2013)
Creating journal inode: 完成

#检查
[root@centos7 ~]# dumpe2fs /dev/sdb1 | grep 'Filesystem features' | grep 'has_journal'
dumpe2fs 1.42.9 (28-Dec-2013)
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype extent 64bit flex_bg sparse_super large_file huge_file uninit_bg dir_nlink extra_isize
```

## 4、Windows 系统读取 ext4 格式磁盘工具ext2fsd （注意：ext4 只读不支持写入！）

<https://sourceforge.net/projects/ext2fsd/files/latest/download?source=files>

> 参考链接：<http://www.cnblogs.com/jusonalien/p/5032973.html>