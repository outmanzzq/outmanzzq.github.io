---
layout: post
title: Ubuntu小技巧：合上笔记本，系统不睡眠
categories: [ubuntu]
description: Ubuntu 小技巧：合上笔记本，系统不睡眠
keywords: ubuntu, 休眠，wifi 断网
---

> 大多数现代操作系统（包括 Windows）会在笔记本合上时进入睡眠状态。Ubuntu 也是如此。如果你想让你的笔记本盖子合上时不睡眠，就跟着我们学习吧。

# 环境说明

- 笔记本：Thinkpad E470
- 系统：Ubuntu 18.04 LTS

# 具体操作

## 方法一：

> 打开 System Settings –> Power（中文版是打开 系统设置 -> 电源），然后进行设置。一些用户设置后不会生效。

## 方法二：

> 直接编辑 Login Manager 的配置文件（logind.conf）。这个方法基本能生效，建议使用这个。按下 Ctrl – Alt – T 组合键，打开终端,执行命令：

```bash
sudo sed -i 's/#\(HandleLidSwitch=\).*/\1ignore/g' /etc/systemd/logind.conf

# 重启生效
reboot
```

配置文件的 “ignore” 值告诉 Ubuntu 当笔记本合上后不要睡眠或挂起.

> 参考链接：<https://blog.csdn.net/cblou/article/details/43191651>