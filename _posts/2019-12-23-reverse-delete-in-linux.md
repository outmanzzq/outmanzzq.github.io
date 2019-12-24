---
layout: post
title: Linux 反选删除文件
categories: rm,linux,delete
description: Linux rm 删除指定文件外的其他文件方法
keywords: rm,linux,delete
---

> Linux rm 删除指定文件外的其他文件方法

## 最简单的方法

```bash
# 打开extglob模式
$ shopt -s extglob

# 单文件排除删除
$ rm -fr !(file1)

# 如果是多个要排除的，可以这样
$ rm -rf !(file1|file2)
```

> 参考链接：<https://www.cnblogs.com/bierenbiewo11/p/11040413.html>