---
layout: post
title: yumdownloader:下载保存 Yum 包而不安装
categories: yum,rpm
description: yumdownloader:下载保存 Yum 包而不安装
keywords: yum,rpm
---

> 有时候安装 Mysql 或者 POSTGRESQL 这种较大的软件，在线开启仓库下载遇到网络源维护或者自己网络差的时候下载速度总是让人捉急，搭建本地 yum 源可以解决部分问题，如果我们需要的包在本地的 YUM 源里找不到或者版本不对，就不好办了。
>
> yum install 安装完之后会自动清理安装包，如果只想通过 yum 下载软件的安装包，但是不需要进行安装的话，可以使用 yumdownloader 命令。

## yumdownloader 软件安装（CentOS 7）

yumdownloader 命令在软件包 yum-utils 里面。先安装 yum-utils ：

`yum install yum-utils -y`

查看 yum-utils 软件包有没有 yumdownloader，如果有输出代表可用：

```bash
$ rpm -ql yum-utils |grep yumdownloader
/usr/bin/yumdownloader
/usr/share/man/man1/yumdownloader.1.gz
```

## 实战 （CentOS 7）

### 实战1: 离线安装 JDK

比如只想下载 java-1.8.0-openjdk ,但是不安装，可以执行如下指令：

`yumdownloader java-1.8.0-openjdk.x86_64`

通过 yumdownloader 就可以在已有的环境下下载 rpm 包待备用。也省得网络不好的时候下载一个几百兆的 NPM 包(比如 mysql )慢出翔来。

单纯的使用 yumdownloader 只会下载给定名称的既定 RPM 包，安装时候所需要的一些依赖不会被下载。如果要下载依赖加上 "--resolve" 参数，如果要指定下载目录。加上"--destdir" 参数

```bash
\$ yumdownloader java-1.8.0-openjdk.x86_64 --resolve --destdir=/opt/java/
$ ls /opt/java/

atk-2.28.1-1.el7.x86_64.rpm                                   libwayland-client-1.15.0-1.el7.x86_64.rpm
cairo-1.15.12-3.el7.x86_64.rpm                                libwayland-server-1.15.0-1.el7.x86_64.rpm
copy-jdk-configs-3.3-10.el7_5.noarch.rpm                      libXcomposite-0.4.4-4.1.el7.x86_64.rpm
cups-libs-1.6.3-35.el7.x86_64.rpm                             libXft-2.3.2-2.el7.x86_64.rpm
fribidi-1.0.2-1.el7.x86_64.rpm                                libXi-1.7.9-1.el7.x86_64.rpm
gdk-pixbuf2-2.36.12-3.el7.x86_64.rpm                          libXinerama-1.1.3-2.1.el7.x86_64.rpm
giflib-4.1.6-9.el7.x86_64.rpm                                 libXrandr-1.5.1-2.el7.x86_64.rpm
graphite2-1.3.10-1.el7_3.x86_64.rpm                           libxslt-1.1.28-5.el7.x86_64.rpm
gtk2-2.24.31-1.el7.x86_64.rpm                                 libXtst-1.2.3-1.el7.x86_64.rpm
gtk-update-icon-cache-3.22.30-3.el7.x86_64.rpm                lksctp-tools-1.0.17-2.el7.x86_64.rpm
harfbuzz-1.7.5-2.el7.x86_64.rpm                               mesa-libEGL-18.0.5-4.el7_6.x86_64.rpm
hicolor-icon-theme-0.12-7.el7.noarch.rpm                      mesa-libgbm-18.0.5-4.el7_6.x86_64.rpm
jasper-libs-1.900.1-33.el7.x86_64.rpm                         pango-1.42.4-2.el7_6.x86_64.rpm
java-1.8.0-openjdk-1.8.0.222.b10-0.el7_6.x86_64.rpm           pcsc-lite-libs-1.8.8-8.el7.x86_64.rpm
java-1.8.0-openjdk-headless-1.8.0.222.b10-0.el7_6.x86_64.rpm  pixman-0.34.0-1.el7.x86_64.rpm
javapackages-tools-3.4.1-11.el7.noarch.rpm                    python-javapackages-3.4.1-11.el7.noarch.rpm
jbigkit-libs-2.0-11.el7.x86_64.rpm                            python-lxml-3.2.1-4.el7.x86_64.rpm
libfontenc-1.1.3-3.el7.x86_64.rpm                             ttmkfdir-3.0.9-42.el7.x86_64.rpm
libglvnd-egl-1.0.1-0.8.git5baa1e5.el7.x86_64.rpm              tzdata-java-2019b-1.el7.noarch.rpm
libthai-0.1.14-9.el7.x86_64.rpm                               xorg-x11-fonts-Type1-7.5-9.el7.noarch.rpm
libtiff-4.0.3-27.el7_3.x86_64.rpm                             xorg-x11-font-utils-7.5-21.el7.x86_64.rpm

#离线安装
$ cd /opt/java
$ yum localinstall java-1.8.0-openjdk.x86_64 -y
```

### 实战2: 离线安装 nginx

```bash
# step1: 查询本机 nginx 是否已安装
$ rpm -qa |grep nginx

# step2: 查询 nginx rpm 包
$ yum search nginx
Loaded plugins: fastestmirror
Repodata is over 2 weeks old. Install yum-cron? Or run: yum makecache fast
Determining fastest mirrors
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
====================================================================================================================== N/S matched: nginx ======================================================================================================================
pcp-pmda-nginx.x86_64 : Performance Co-Pilot (PCP) metrics for the Nginx Webserver

  Name and summary matches only, use "search all" for everything.

# step3: 缓存 nginx 及相关依赖包到本地（不安装）
$ yumdownloader pcp-pmda-nginx.x86_64 --resolve --destdir=/opt/nginx/
$ ll
-rw-r--r-- 1 root root  31264 Jul  4  2014 mailcap-2.1.41-2.el7.noarch.rpm
-rw-r--r-- 1 root root  35504 Nov 21  2018 pcp-conf-4.1.0-5.el7_6.x86_64.rpm
-rw-r--r-- 1 root root 448164 Nov 21  2018 pcp-libs-4.1.0-5.el7_6.x86_64.rpm
-rw-r--r-- 1 root root  23504 Nov 21  2018 pcp-pmda-nginx-4.1.0-5.el7_6.x86_64.rpm
-rw-r--r-- 1 root root  25372 Jul  4  2014 perl-Business-ISBN-2.06-2.el7.noarch.rpm
-rw-r--r-- 1 root root  24908 Jul  4  2014 perl-Business-ISBN-Data-20120719.001-2.el7.noarch.rpm
-rw-r--r-- 1 root root  33172 Jul  4  2014 perl-Compress-Raw-Bzip2-2.061-3.el7.x86_64.rpm
-rw-r--r-- 1 root root  58788 Jul  4  2014 perl-Compress-Raw-Zlib-2.061-4.el7.x86_64.rpm
-rw-r--r-- 1 root root  23384 Jul  4  2014 perl-Digest-1.17-245.el7.noarch.rpm
-rw-r--r-- 1 root root  30468 Jul  4  2014 perl-Digest-MD5-2.52-3.el7.x86_64.rpm
-rw-r--r-- 1 root root  15984 Jul  4  2014 perl-Encode-Locale-1.03-5.el7.noarch.rpm
-rw-r--r-- 1 root root  13520 Jul  4  2014 perl-File-Listing-6.04-7.el7.noarch.rpm
-rw-r--r-- 1 root root 117360 Jul  4  2014 perl-HTML-Parser-3.71-4.el7.x86_64.rpm
-rw-r--r-- 1 root root  18464 Jul  4  2014 perl-HTML-Tagset-3.20-15.el7.noarch.rpm
-rw-r--r-- 1 root root  26588 Jul  4  2014 perl-HTTP-Cookies-6.01-5.el7.noarch.rpm
-rw-r--r-- 1 root root  21224 Nov 12  2018 perl-HTTP-Daemon-6.01-8.el7.noarch.rpm
-rw-r--r-- 1 root root  14404 Jul  4  2014 perl-HTTP-Date-6.02-8.el7.noarch.rpm
-rw-r--r-- 1 root root  83600 Jul  4  2014 perl-HTTP-Message-6.06-6.el7.noarch.rpm
-rw-r--r-- 1 root root  17324 Jul  4  2014 perl-HTTP-Negotiate-6.01-5.el7.noarch.rpm
-rw-r--r-- 1 root root 266004 Jul  4  2014 perl-IO-Compress-2.061-2.el7.noarch.rpm
-rw-r--r-- 1 root root  23496 Jul  4  2014 perl-IO-HTML-1.00-2.el7.noarch.rpm
-rw-r--r-- 1 root root  36660 Apr 25  2018 perl-IO-Socket-IP-0.21-5.el7.noarch.rpm
-rw-r--r-- 1 root root 117320 Apr 25  2018 perl-IO-Socket-SSL-1.94-7.el7.noarch.rpm
-rw-r--r-- 1 root root 210260 Jul  4  2014 perl-libwww-perl-6.05-2.el7.noarch.rpm
-rw-r--r-- 1 root root  24384 Jul  4  2014 perl-LWP-MediaTypes-6.02-2.el7.noarch.rpm
-rw-r--r-- 1 root root  11544 Jul  4  2014 perl-Mozilla-CA-20130114-5.el7.noarch.rpm
-rw-r--r-- 1 root root  29328 Jul  4  2014 perl-Net-HTTP-6.06-2.el7.noarch.rpm
-rw-r--r-- 1 root root  29096 Jul  4  2014 perl-Net-LibIDN-0.12-15.el7.x86_64.rpm
-rw-r--r-- 1 root root 292308 Aug 11  2017 perl-Net-SSLeay-1.55-6.el7.x86_64.rpm
-rw-r--r-- 1 root root  62356 Nov 21  2018 perl-PCP-PMDA-4.1.0-5.el7_6.x86_64.rpm
-rw-r--r-- 1 root root  52744 Jul  4  2014 perl-TimeDate-2.30-2.el7.noarch.rpm
-rw-r--r-- 1 root root 109040 Jul  4  2014 perl-URI-1.60-9.el7.noarch.rpm
-rw-r--r-- 1 root root  18324 Jul  4  2014 perl-WWW-RobotRules-6.02-5.el7.noarch.rpm

# step3: 离线安装 nginx
$ cd /opt/nginx
$ yum localinstall pcp-pmda-nginx.x86_64 -y

# step4: 查询
$ rpm -qa |grep nginx
pcp-pmda-nginx-4.1.0-5.el7_6.x86_64

# step5: 启动 nginx 服务,并设置开机自启动
$ systemctl start nginx
$ systemctl enable nginx
```

> 原文链接：<https://www.linuxprobe.com/yumdownloader-download-rpm.html>