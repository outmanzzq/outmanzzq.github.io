---
layout: wiki
title: Linux 命令之 diff
categories: diff
description: diff 命令在最简单的情况下，比较两个文件的不同
keywords: diff,compare
---

> diff 命 令是 linux 上非常重要的工具，用于比较文件的内容，特别是比较两个版本不同的文件以找到改动的地方。diff 在命令行中打印每一个行的改动。最新版 本的diff还支持二进制文件。diff 程序的输出被称为补丁 (patch)，因为Linux系统中还有一个 patch 程序，可以根据 diff 的输出将 a.c 的文件内容更新为 b.c。diff 是 svn、cvs、git 等版本控制工具不可或缺的一部分。

# 1. 格式：

diff [参数][文件1或目录1][文件2或目录2]

# 2. 功能：

diff 命令能比较单个文件或者目录内容。如果指定比较的是文件，则只有当输入为文本文件时才有效。以逐行的方式，比较文本文件的异同处。如果指定比较的是目录的 的时候，diff 命令会比较两个目录下名字相同的文本文件。列出不同的二进制文件、公共子目录和只在一个目录出现的文件。

# 3. 常用参数说明

```bash
-a 预设只会逐行比较文本文件

-b 忽略行尾的空格
-B 不检查空白行

-c 用上下文输出格式，提供 n 行上下文
-C 执行与 -c 命令相同

-d 使用不同的演算法，以较小的单位来做比较

-f 输出的格式类似于ed 的script，但按照原来文件的顺序来显示不同处

-H 比较大文件时可以加快速度

-l 若比较的文件在某几行有所不同，而这几行同时都包含了选项中指定的字符或字符串，则不显示这两个文件的差异。

-i 不检查大小写的不同
-I 讲结果交由 pr 程序来分页

-n 将比较结果以 RCS 的格式来显示
-N 在比较目录时，若文件 A 仅出现在某个目录中，则预设会显示。

-p 若比较的文件为 C语言 的程序源文件时，则预设会显示
-P 与 -N 类似，但只有当第二个目录包含了包含了第一个目录中没有的文件按时，才会将这个文件与空白的文件比较。

-q 仅显示有无差异，不显示详细信息

-r 比较子目录中的文件

-s 若没有发现任何差异，任然显示信息
-S 在比较目录时，从指定的文件开始比较

-t 在输出时将 tab 字符展开
-T 在每行前面加上tab 字符以便对齐

-u，-U 以合并的方式来显示文件内容的不同

-v 显示版本信息

-w 忽略全部的空格字符
-W 在使用 -y 参数时，指定栏宽

-x 不比较选项中指定的文件或目录
-X 可以将文件或目录类型存成文本文件，然后指定此文本文件

-y 以并列的方式显示文件的异同之处

--help 显示帮助
```

# 4. 实例

## 在没有任何参数时，diff 命令比较

```bash
[root@CentOS6]# cat>t1.txt<<EOF
this is a text!

Name is t1.txt!
The extra content!
I am MenAngel!
EOF

[root@CentOS6]# cat> t2.txt　　　　//第二种cat建文件方式，Ctrl+Z 退出编辑
this is a text!

Name is t2.txt!
[root@CentOS6]# diff t1.txt t2.txt
3,5c3　　　　    //文件1中的第 3 行到第5行（3,5）改为（c）文件2中的第3行（3），则两个文件相同。
< Name is t1.txt!　　　　//<表示 FILE1 的行
< The extra content!
< I am MenAngel!
---
> Name is t2.txt!　　　　//>表示 FILE2 的行
```

## 以并列输出格式展示两个文件的不同

```bash
[root@CentOS6]# diff -y t1.txt t2.txt
this is a text!                         this is a text!

Name is t1.txt!                           | Name is t2.txt!
The extra content!                        <
I am MenAngel!                            <
```

## 并列输出格式下控制宽度

```bash
[root@CentOS6]# diff -y -W 40 t1.txt t2.txt
this is a text!     this is a text!

Name is t1.txt!    |    Name is t2.txt!
The extra conten   <
I am MenAngel!     <
```

## 以上下文输出格式展示两个文件的不同

```bash
[root@CentOS6]# diff -c t1.txt t2.txt
*** t1.txt  2018-06-23 21:44:15.171207940 +0800　　　　//*代表t1.txt
--- t2.txt  2018-06-23 21:44:54.987207910 +0800　　　　//-代表t2.txt
***************
*** 1,5 ****　　　　//从第1行到第5行
  this is a text!

! Name is t1.txt!　　　　//！标注差异行
! The extra content!
! I am MenAngel!
--- 1,3 ----　　　　//从第1行到第3行
  this is a text!

! Name is t2.txt!
```

## 以统一输出格式展示两个文件的不同

```bash
[root@CentOS6]# diff -u t1.txt t2.txt
--- t1.txt  2018-06-23 21:44:15.171207940 +0800　　　　//-代表t1.txt
+++ t2.txt  2018-06-23 21:44:54.987207910 +0800　　　　//+代表t2.txt
@@ -1,5 +1,3 @@　　　　//t1.txt 第1行到第5行，t2.txt 第1行到第3行
 this is a text!
  
-Name is t1.txt!　　　　//- 代表前面的文件比后面的文件多一行，+ 代表后面的文件比前面的文件多一行，一般一一对应
-The extra content!
-I am MenAngel!
+Name is t2.txt!
```

## 文件交换对比位置所带来的差距

```bash
[root@CentOS6]# diff -y -W 40 t2.txt t1.txt
this is a text!     this is a text!

Name is t2.txt!    |    Name is t1.txt!
           > The extra conten
           > I am MenAngel!
[root@CentOS6]# diff -c t2.txt t1.txt
*** t2.txt  2018-06-23 21:44:54.987207910 +0800
--- t1.txt  2018-06-23 21:44:15.171207940 +0800
***************
*** 1,3 ****
  this is a text!

! Name is t2.txt!
--- 1,5 ----
  this is a text!

! Name is t1.txt!
! The extra content!
! I am MenAngel!
[root@CentOS6]# diff -u t2.txt t1.txt
--- t2.txt  2018-06-23 21:44:54.987207910 +0800
+++ t1.txt  2018-06-23 21:44:15.171207940 +0800
@@ -1,3 +1,5 @@
 this is a text!
  
-Name is t2.txt!
+Name is t1.txt!
+The extra content!
+I am MenAngel!
```

## 目录与目录间的比较

```bash
[root@CentOS6]# mkdir dir1 dir2
[root@CentOS6]# cd dir1
[root@CentOS6 dir1]# cat>text1<<EOF
> dir:dir1
> name:text1
>
> Total 4!
> EOF
[root@CentOS6 dir1]# cat>text2<<EOF
> I am MenAngel!
> I am studying the order of Linux!
> EOF
[root@CentOS6 dir1]# cd ../dir2
[root@CentOS6 dir2]# cat>text1<<EOF
> dir:dir2
> name:text1
>
>
> Total 5!
> EOF
[root@CentOS6 dir2]# cat>text3<<EOF
> Working hard makes success!
> I am MenAngel!
> EOF
[root@CentOS6 dir2]# cd ../
[root@CentOS6]# diff dir1 dir2
diff dir1/text1 dir2/text1　　　　//有相同文件名时自动比较
1c1
< dir:dir1
---
> dir:dir2
4c4,5
< Total 4!
---
>
> Total 5!
Only in dir1: text2　　　　//只有 dir1 存在 text2
Only in dir2: text3　　　　//只有 dir2 存在 text3
```

## 生成 log 文件，diff的输出文件被称为补丁（patch），可以使用 patch 命令将文件内容更新

```bash
[root@CentOS6]# diff dir1 dir2>dir.log
[root@CentOS6]# cat dir.log
diff dir1/text1 dir2/text1
1c1
< dir:dir1
---
> dir:dir2
4c4,5
< Total 4!
---
>
> Total 5!
Only in dir1: text2
Only in dir2: text3
[root@CentOS6]# diff t1.txt t2.txt >t12.log
[root@CentOS6]# cat t12.log
3,5c3
< Name is t1.txt!
< The extra content!
< I am MenAngel!
---
> Name is t2.txt!
```

# vimdiff 也可以比较文件

`[xf@xuexi ~]$ vimdiff 1.txt 2.txt`

> 参考链接：<https://www.cnblogs.com/ay-a/p/8232584.html>