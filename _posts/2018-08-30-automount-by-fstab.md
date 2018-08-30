---
layout: post
title: Linux 命令-自动挂载文件 /etc/fstab 功能详解
categories: linux,fstab
description: some word here
keywords: keyword1, keyword2
---

> 磁盘被手动挂载之后都必须把挂载信息写入 /etc/fstab 这个文件中，否则下次开机启动时仍然需要重新挂载。

# 一、/etc/fstab 文件的作用

系统开机时会主动读取 /etc/fstab 这个文件中的内容，根据文件里面的配置挂载磁盘。

这样我们只需要将磁盘的挂载信息写入这个文件中我们就不需要每次开机启动之后手动进行挂载了。

# 二、挂载的限制

在说明这个文件的作用之前我想先强调一下挂载的限制。

1. 根目录是必须挂载的，而且一定要先于其他 mount point 被挂载。因为 mount 是所有目录的跟目录，其他木有都是由根目录 /衍生出来的。

2. 挂载点必须是已经存在的目录。

3. 挂载点的指定可以任意，但必须遵守必要的系统目录架构原则

4. 所有挂载点在同一时间只能被挂载一次

5. 所有分区在同一时间只能挂在一次

6. 若进行卸载，必须将工作目录退出挂载点（及其子目录）之外。

# 三、/etc/fstab 文件中的参数

下面我们看看看 /etc/fstab 文件，这是我的 linux 环境中 /etc/fstab 文件中的内容:

```bash
# 查看当前系统已经存在的挂载信息
$ cat /etc/fstab
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/sda1 during installation
UUID=f0d9b5f8-24ef-4aba-b3ce-f4bf0a0c231a /               ext4    errors=remount-ro 0       1
/swapfile                                 none            swap    sw              0       0
# mount second disk /dev/sdb1
UUID=12f3d1ee-4058-42b7-a83c-83edfb896206 /data           ext4    errors=remount-ro 0       1
```

在文件中我已经把每一列都做出来表示方便识别，我们可以看到一共有六列。

## 第一列：Device：磁盘设备文件或者该设备的 Label 或者 UUID

### 1）查看分区的 label 和 uuid

Label 就是分区的标签，在最初安装系统时填写的挂载点就是标签的名字。

可以通过查看一个分区的 superblock 中的信息找到 UUID 和 Label name。

例如:我们要查看 /dev/sdb1 这个设备的uuid和label name

```bash
# 切换 root
$ sudo -i

# 查询 /dev/sdb1 UUID
$ blkid /dev/sdb1
/dev/sdb1: UUID="12f3d1ee-4058-42b7-a83c-83edfb896206" TYPE="ext4" PARTUUID="40901347-01"
```

### 2）使用设备名和 label 及 uuid 作为标识的不同

使用设备名称（ /dev/sda )来挂载分区时是被固定死的，一旦磁盘的插槽顺序发生了变化，就会出现名称不对应的问题。因为这个名称是会改变的。

不过使用 label 挂载就不用担心插槽顺序方面的问题。不过要随时注意你的 Label name。至于 UUID，每个分区被格式化以后都会有一个 UUID 作为唯一的标识号。使用 uuid 挂载的话就不用担心会发生错乱的问题了。

## 第二列：Mount point：设备的挂载点，就是你要挂载到哪个目录下。

## 第三列：filesystem：磁盘文件系统的格式，包括 ext2、ext3、reiserfs、nfs、vfat等

## 第四列：parameters：文件系统的参数

参数 | 说明
-|-
Async/sync | 设置是否为同步方式运行，默认为 async
auto/noauto | 当下载 mount -a 的命令时，此文件系统是否被主动挂载。默认为 auto
rw/ro | 是否以以只读或者读写模式挂载
exec/noexec | 限制此文件系统内是否能够进行"执行"的操作
user/nouser | 是否允许用户使用 mount 命令挂载
suid/nosuid | 是否允许 SUID 的存在
Usrquota | 启动文件系统支持磁盘配额模式
Grpquota | 启动文件系统对群组磁盘配额模式的支持
Defaults | 同事具有 rw,suid,dev,exec,auto,nouser,async 等默认参数的设置

## 第五列：能否被 dump 备份命令作用：dump 是一个用来作为备份的命令。通常这个参数的值为 0 或者 1

参数 | 说明
-|-
0 | 代表不要做 dump 备份
1 | 代表要每天进行 dump 的操作
2 | 代表不定日期的进行 dump 操作

## 第六列：是否检验扇区：开机的过程中，系统默认会以 fsck 检验我们系统是否为完整（clean）。

参数 | 说明
-|-
0 | 不要检验
1 | 最早检验（一般根目录会选择）
2 | 1级别检验完成之后进行检验

# 二、Linux 磁盘分区 UUID 的获取及其 UUID 的作用

> UUID-Universally Unique IDentifiers 全局唯一标识符

## 1. Linux 磁盘分区 UUID 的获取方法

- **方法1:**  ls -l /dev/disk/by-uuid/

```bash
# 需要 root 权限
$ ls -l /dev/disk/by-uuid/ 
总用量 0
lrwxrwxrwx 1 root root 10 8月  30 18:57 12f3d1ee-4058-42b7-a83c-83edfb896206 -> ../../sdb1
lrwxrwxrwx 1 root root 10 8月  30 18:57 f0d9b5f8-24ef-4aba-b3ce-f4bf0a0c231a -> ../../sda1
```

- **方法2:**  通过 blkid 命令

```bash
# 需要 root 权限
$ blkid /dev/sda1
/dev/sda1: UUID="f0d9b5f8-24ef-4aba-b3ce-f4bf0a0c231a" TYPE="ext4" PARTUUID="e3d6d3a9-01"
```

## 2. Linux UUID 的作用及意义

### 原因1：它是真正的唯一标志符

UUID 为系统中的存储设备提供唯一的标识字符串，不管这个设备是什么类型的。如果你在系统中添加了新的存储设备如硬盘，很可能会造成一些麻烦，比如说启动的时候因为找不到设备而失败，而使用UUID则不会有这样的问题。

### 原因2：设备名并非总是不变的

自动分配的设备名称并非总是一致的，它们依赖于启动时内核加载模块的顺序。如果你在插入了 USB 盘时启动了系统，而下次启动时又把它拔掉了，就有可能导致设备名分配不一致。

使用 UUID 对于挂载移动设备也非常有好处──例如我有一个 24 合一的读卡器，它支持各种各样的卡，而使用 UUID 总可以使同一块卡挂载在同一个地方。

### 原因3：ubuntu 中的许多关键功能现在开始依赖于 UUID

例如 grub ──系统引导程序，现在可以识别 UUID，打开你的 /boot/grub/menu.lst，你可以看到类似如下的语句：

```bash
$ cat /boot/grub/menu.lst
title Ubuntu hardy (development branch), kernel 2.6.24-16-generic
root (hd2,0)
kernel /boot/vmlinuz-2.6.24-16-generic root=UUID=c73a37c8-ef7f-40e4-b9de-8b2f81038441 ro quiet splash
initrd /boot/initrd.img-2.6.24-16-generic
quiet
```

> 参考链接：
>
> <https://www.cnblogs.com/qiyebao/p/4484047.html>
> <https://www.cnblogs.com/itcomputer/p/4664511.html>