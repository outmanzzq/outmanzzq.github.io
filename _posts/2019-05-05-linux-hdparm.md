---
layout: post
title: Linux 的 hdparm 工具参数详解：硬盘检查、测速、设定、优化
categories: hdparm,io
description: 在Linux下可以使用hdparm工具查看硬盘的相关信息或对硬盘进行测速、优化、修改硬盘相关参数设定。我主要常用这个工具来测试硬盘速度。
keywords: hdparm,io
---

> 在 Linux 下可以使用 hdparm 工具查看硬盘的相关信息或对硬盘进行测速、优化、修改硬盘相关参数设定。我主要常用这个工具来测试硬盘速度。

# hdparm (hard disk parameters)

## 功能说明

显示与设定硬盘的参数。

## 语法

`hdparm [-CfghiIqtTvyYZ][-a <快取分区>][-A <0或1>][-c ][-d <0或1>][-k <0或1>][-K <0或1>][-m <分区数>][-n <0或1>][-p ][-P <分区数>][-r <0或1>][-S <时间>][-u <0或1>][-W <0或1>][-X <传输模式>][设备]`

## 参数说明：

```shell
-a  <快取分区> 设定读取文件时，预先存入块区的分区数，若不加上<快取分区>选项，则显示目前的设定。
-A  <0或1> 启动或关闭读取文件时的快取功能。
-c  设定 IDE32位I/O 模式。
-C  检测 IDE 硬盘的电源管理模式。
-d  <0或1> 设定磁盘的 DMA 模式。
-f  将内存缓冲区的数据写入硬盘，并清楚缓冲区。
-g  显示硬盘的磁轨，磁头，磁区等参数。
-h  显示帮助。
-i  显示硬盘的硬件规格信息，这些信息是在开机时由硬盘本身所提供。
-I  直接读取硬盘所提供的硬件规格信息。
-k  <0或1> 重设硬盘时，保留 -dmu 参数的设定。
-K  <0或1> 重设硬盘时，保留 -APSWXZ 参数的设定。
-m  <磁区数> 设定硬盘多重分区存取的分区数。
-n  <0或1> 忽略硬盘写入时所发生的错误。
-p  设定硬盘的 PIO 模式。
-P  <磁区数> 设定硬盘内部快取的分区数。
-q  在执行后续的参数时，不在屏幕上显示任何信息。
-r  <0或1> 设定硬盘的读写模式。
-S  <时间> 设定硬盘进入省电模式前的等待时间。
-t  评估硬盘的读取效率。
-T  平谷硬盘快取的读取效率。
-u  <0或1> 在硬盘存取时，允许其他中断要求同时执行。
-v  显示硬盘的相关设定。
-W  <0或1> 设定硬盘的写入快取。
-X  <传输模式> 设定硬盘的传输模式。
-y  使 IDE 硬盘进入省电模式。
-Y  使 IDE 硬盘进入睡眠模式。
-Z  关闭某些 Seagate 硬盘的自动省电功能。
```

## hdparm 常用参数使用举例

```shell
# 1、显示硬盘的相关设置：
[root@oracle ~]# hdparm /dev/sda
/dev/sda:
IO_support = 0 (default 16-bit)
readonly = 0 (off)
readahead = 256 (on)
geometry = 19929［柱面数］/255［磁头数］/63［扇区数］, sectors = 320173056［总扇区数］, start = 0［起始扇区数］

# 2、显示硬盘的柱面、磁头、扇区数：
[root@oracle ~]# hdparm -g /dev/sda
/dev/sda:
geometry = 19929［柱面数］/255［磁头数］/63［扇区数］, sectors = 320173056［总扇区数］, start = 0［起始扇区数］

# 3、测试硬盘的读取速度：

[root@oracle ~]# hdparm -t /dev/xvda

/dev/xvda:
Timing buffered disk reads: 422 MB in 3.01 seconds = 140.20 MB/sec
[root@oracle ~]# hdparm -t /dev/xvda

/dev/xvda:
Timing buffered disk reads: 408 MB in 3.01 seconds = 135.59 MB/sec
[root@oracle ~]# hdparm -t /dev/xvda

/dev/xvda:
Timing buffered disk reads: 416 MB in 3.01 seconds = 138.24 MB/sec

# 4、测试硬盘缓存的读取速度：

[root@oracle ~]# hdparm -T /dev/xvda

/dev/xvda:
Timing cached reads: 11154 MB in 1.98 seconds = 5633.44 MB/sec
[root@oracle ~]# hdparm -T /dev/xvda

/dev/xvda:
Timing cached reads: 10064 MB in 1.98 seconds = 5077.92 MB/sec
[root@oracle ~]# hdparm -T /dev/xvda

/dev/xvda:
Timing cached reads: 10600 MB in 1.98 seconds = 5351.73 MB/sec

# 5、检测硬盘的电源管理模式：
[root@oracle ~]# hdparm -C /dev/sda
/dev/sda:
drive state is: standby [省电模式]

# 6、查询并设置硬盘多重扇区存取的扇区数，以增进硬盘的存取效率：
[root@oracle ~]# hdparm -m /dev/sda
[root@oracle ~]# hdparm -m 参数值为整数值如8 /dev/sda

```

# 附：硬盘坏道修复方法

- 检查：smartctl -l selftest /dev/sda
- 卸载：umount /dev/sda*
- 修复：badblocks /dev/sda

> 参考链接：<https://blog.csdn.net/AXW2013/article/details/80498777>
