---
layout: post
title: windows10 + Ubuntu 双系统引导修复（Boot Repair)
categories: os
description: windows10 + Ubuntu 双系统引导修复（Boot Repair) 
keywords: os,boot repair,ubuntu,bootloader
---

> 安装完双系统，如果在使用过程中不小心删除了 Ubuntu 引导项，则会导致开机后无法选择进入 Ubuntu 系统。或者当我们重装了 windows 系统后，也会发现原来的 Ubuntu 引导不见了，当出现这两种情况之一时，最好的解决办法不是重新把 Ubuntu 系统装一遍，我们只需要冲洗修复一下 Ubuntu 引导文件，就可以把问题解决了。

# 环境说明

- 笔记本：ThinkPad E47
- OS1: windows 10 pro(1703)
- OS2: ubuntu 18.04 LTS

> 因重装 win10，导致系统启动，直接进入 win10 系统，ubuntu 系统菜单丢失（通过分区工具确认分区及数据都在）

# 一、具体修复过程

## 1. 第一步：制作 Ubuntu U盘安装盘

- 准备一个4G以上U盘
- Ubuntu 18.04 LTS ISO镜像
- YUMI U盘烧录工具
- 一台 windows 系统电脑

利用 YUMI 工具 ，制作 Ubuntu U盘安装盘，制作过程参考：: https://www.pendrivelinux.com/yumi-multiboot-usb-creator/

## 2. 第二步：U盘启动预览版 Ubuntu 系统

还是需要进入 Ubuntu 界面，但是并不需要安装（如果直接安装的话，以前在 Ubuntu 里面的文件可全部都没有了，所以万不得已，千万别这样做）。

## 3. 第三步：进入预览版 Ubuntu 系统

打开终端，终端快捷键是 Ctrl+Alt+T，输入命令，添加 boot-repair 所在的源：(确保可以联网)
`sudo add-apt-repository ppa:yannubuntu/boot-repair && sudo apt-get update`

## 4. 第四步：执行修复命令

待上面命令执行完毕后，继续输入以下命令，安装 boot-repair 并且开启 boot-repair：

`sudo apt-get install -y boot-repair && boot-repair`

等一会，会出现如下的界面：

![ ](/images/20180711-repair-bootloader-01.png)

就会出现这个，点击 Recommended repair，过几分钟重启就行了。

# 二、常见问题

## 1. 按照以上操作，修复未成功！

如果上面已经执行成功了，可以跳过此部，否则，我们可以自己输入命令进行修复：

`sudo recommended repair`

成功后，就会弹出我们的盘的各种信息以及引导的信息。

## 2. 误点击了第二项 Create a BootInfo summary

那你的开机启动界面将会出来一大堆你以前没见过的东西。
那样的话，你可以输入名令：

`cd /boot/grub`

接着输入:

`sudo gedit grub.cfg`

打开 grub.cfg 文件后，通过搜索找到 windows，然后把下面这些删去就和原来一样了。

```bash

### BEGIN /etc/grub.d/25_custom ###
menuentry "efi/EFI/Boot/bootx64.efi" {
search --fs-uuid --no-floppy --set=root d000ed6a-5303-40aa-a517-af50e807c0e9
chainloader (${root})/efi/EFI/Boot/bootx64.efi
}

menuentry "efi/EFI/ubuntu/MokManager.efi" {
search --fs-uuid --no-floppy --set=root d000ed6a-5303-40aa-a517-af50e807c0e9
chainloader (${root})/efi/EFI/ubuntu/MokManager.efi
}

menuentry "Windows UEFI recovery bootmgfw.efi" {
search --fs-uuid --no-floppy --set=root A603-846C
chainloader (${root})/EFI/Microsoft/Boot/bootmgfw.efi
}

menuentry "Windows Boot UEFI recovery" {
search --fs-uuid --no-floppy --set=root A603-846C
chainloader (${root})/EFI/Boot/bkpbootx64.efi
}

menuentry "EFI/ubuntu/MokManager.efi sda2" {
search --fs-uuid --no-floppy --set=root A603-846C
chainloader (${root})/EFI/ubuntu/MokManager.efi
}

menuentry "Windows UEFI recovery LrsBootmgr.efi" {
search --fs-uuid --no-floppy --set=root 7607-5674
chainloader (${root})/efi/Microsoft/Boot/LrsBootmgr.efi
}

menuentry "Windows Boot UEFI recovery bkpbootx64.efi" {
search --fs-uuid --no-floppy --set=root 7607-5674
chainloader (${root})/efi/Boot/bkpbootx64.efi
}
### END /etc/grub.d/25_custom ###
```

> 参考连接：<https://blog.csdn.net/piaocoder/article/details/50589667>
