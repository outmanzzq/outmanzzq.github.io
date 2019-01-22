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

# 3. 参数

```bash
-a,--text 把所有文件当做文本文件逐行比较
-b,--ignore-space-change 忽略空格产生的变化
-B,--ignore-blank-lines 忽略空白行的变化
-c,–C NUM，--context[=NUM] 使用上下文输出格式（文件1在上，文件2在下，在差异点会标注出来），输出NUM（默认3）行的上下文（上下各NUM行，不包括差异行）
-d,--minimal 使用不同的算法，努力寻找一个较小的变化集合。这会使 diff 变慢（有时更慢）
-D NAME,--ifdef=NAME 合并if-then-else 格式输出，预处理宏（由 NAME 参数提供）条件
-e,--ed 输出一个ed格式的脚本文件
-E,--ignore-all-space 忽略由于 Tab 扩展而导致的变化
-F RE,--show-function-line=RE 在上下文输出格式（文件1在上，文件2在下）和统一输出格式中，对于每一大块的不同，显示出匹配RE（regexp 正则表达式）最近的行
-i,--ignore-case 忽略大小写的区别
-I RE,--ignore-matching-lines=RE 忽略所有匹配RE（regexp 正则表达式）的行的更改
-l,--paginate 通过 pr 编码传递输出，使其分页
-n,--rcs 输出 RCS 格式差异

-N,--new-file 把缺少的文件当做空白文件处理

-p,--show-c-function 显示带有C函数的变化

-q,--brief 仅输出文件是否有差异，不报告详细差异

-r,--recursive 当比较目录时，递归比较所有找到的子目录

-s,--report-identical-files 当两个文件相同时报告

-S FILE,--starting-file=FILE 在比较目录时，从 FILE 开始。用于继续中断的比较

-t,--expand-tabs 将输出时扩展Tab转换为空格，保护输入文件的 tab 对齐方式

-T,--initial-tab 通过预先设置的 tab 使选项卡对齐（？？？）

-u,-U NUM,--unified[=NUM] 使用统一输出格式（输出一个整体，只有在差异的地方会输出差异点，并标注出来），输出NUM（默认3）行的上下文（上下各NUM行，不包括差异行）

-v,--version 输出版本号

-w,--ignore-all-space 比较时忽略所有空格

-W NUM,--width=NUM 在并列输出格式时，指定列的宽度为 NUM（默认130）

-x PAT,--exclude=PAT 排除与 PAT（pattern样式）匹配的文件

-X FILE,--exclude-from=FILE 排除与 FILE 中样式匹配的文件

-y,--side-by-side 使用并列输出格式

--from-file=FILE1 FILE1 与所有操作对象比较，FILE1 可以是目录

--help 输出帮助信息

--horizon-lines=NUM 保留 NUM 行的公共前缀和后缀

--ignore-file-name-case 比较时忽略文件名大小写

--label LABEL 使用 LABEL（标识）代替文件名

--left-column （在并列输出格式中）只输出左列的公共行

--no- ignore-file-name-case 比较时考虑文件名大小写

--normal 输出一个正常的差异

--speed-large-files 假设文件十分大，而且有许多微小的差异

--strip-trailing-cr 去掉在输入时尾随的回车符

--suppress-common-lines 不输出公共行

--to-file=FILE2 所有操作对象与 FILE2 比较，FILE2 可以是目录

--unidiredtional-newfile 将缺少的第一个文件视为空文件



--GTYPE-group-format=GFMT 以 GFMT 格式化 GTYPE 输入组

--line-format=LFMT 以 LFMT 格式化输入所有行

--LTYPE-line-format=LFMT 以 LFMT 格式化 LTYPE 输入行

LTYPE 可以是’old’,’new’或’unchanged’。GTYPE 可以是 LTYPE 选项或’changed’

       GFMT 包括：

              %< 该行属于FILE1

              %> 该行属于FILE2

              %= 该行属于FILE1和FILE2公共行

              %[-][WIDTH（宽度）][.[PREC（精确度）]]{doxX}LETTER（字母） printf 格式规范 LETTER。如下字母表示属于新的文件，小写表示属于旧的文件：

                     F 行组中第一行的行号

                     L 行组中最后一行的行号

                     N 行数（=L-F+1）

                     E F-1

                     M L+1

       LFMT 可包含：

              %L 该行内容

              %l 该行内容，但不包括尾随的换行符

              %[-][WIDTH][.[PREC]]{doxX}n printf 格式规范输入行编号

       GFMT 或 LFMT 可包含：

       %% %

%c’C’ 单个字符C

%c’\000’ 八进制 000 所代表的字符
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

> 原文链接：<https://www.cnblogs.com/diantong/p/9238348.html>