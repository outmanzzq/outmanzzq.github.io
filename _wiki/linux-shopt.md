---
layout: wiki
title: linux 命令-shopt
categories: shopt,rm
description: 反向删除 Linux 文件或目录
keywords: shopt,rm
---

> 今天碰到一个需求，删除当前目录下除指定名称之外所有的文件,当时想到了这种写法：`rm -f !(a)`
> 没想到回报错了，利用 shopt 命令增强 shell 易用性，分享一波用法

# shopt 命令

shopt 命令是 set 命令的一种替代,很多方面都和 set 命令一样,但它增加了很多选项。可以使用

- -p  选项来查看 shopt 选项的设置(选项较多自行查看)

- -u  表示关闭一个选项

- -s  表示开启一个选项

> shopt命令若不带任何参数选项，则可以显示所有可以设置的 shell 操作选项。

## 以下是 shopt 命令的选择介绍：

选项 | 含义
-|-
cdable_vars     | 如果给 cd 内置命令的参数不是一个目录,就假设它是一个变量名,变量的值是将要转换到的目录
cdspell         | 纠正 cd 命令中目录名的较小拼写错误.检查的错误包括颠倒顺序的字符,遗漏的字符以及重复的字符.如果找到一处需修改之处,正确的路径将打印出,命令将继续.只用于交互式 shell
checkhash       | bash 在试图执行一个命令前,先在哈希表中寻找,以确定命令是否存在.如果命令不存在,就执行正常的路径搜索
checkwinsize    | bash 在每个命令后检查窗口大小,如果有必要,就更新 LINES 和 COLUMNS 的值
cmdhist         | bash 试图将一个多行命令的所有行保存在同一个历史项中.这是的多行命令的重新编辑更方便
dotglob         | Bash 在文件名扩展的结果中包括以点(.)开头的文件名
execfail        | 如果一个非交互式 shell 不能执行指定给 exec 内置命令作为参数的文件,它不会退出.如果 exec 失败,一个交互式 shell 不会退出
expand_aliases  | 别名被扩展.缺省为打开
extglob         | 打开扩展的模式匹配特性(正常的表达式元字符来自 Korn shell 的文件名扩展)
histappend      | 如果 readline 正被使用,用户有机会重新编辑一个失败的历史替换
histverify      | 如果设置,且 readline 正被使用,历史替换的结果不会立即传递给 shell 解释器.而是将结果行装入 readline 编辑缓冲区中,允许进一步修改
hostcomplete    | 如果设置,且 readline 正被使用,当正在完成一个包含@的词时 bash 将试图执行主机名补全.缺省为打开
huponexit       | 如果设置,当一个交互式登录 shell 退出时,bash 将发送一个 SIGHUP (挂起信号)给所有的作业
interactive_comments    | 在一个交互式 shell 中.允许以#开头的词以及同一行中其他的字符被忽略.缺省为打开
lithist         | 如果打开,且 cmdhist 选项也打开,多行命令讲用嵌入的换行符保存到历史中,而无需在可能的地方用分号来分隔
mailwarn        | 如果设置,且 bash 用来检查邮件的文件自从上次检查后已经被访问,将显示消息 ”The mail in mailfile has been read”
nocaseglob      | 如果设置,当执行文件名扩展时,bash 在不区分大小写的方式下匹配文件名
nullglob        | 如果设置,bash 允许没有匹配任何文件的文件名模式扩展成一个空串,而不是他们本身
promptvars      | 如果设置,提示串在被扩展后再进行变量和参量扩展.缺省为打开
restricted_shell    | 如果 shell 在受限模式下启动就设置这个选项.该值不能被改变.当执行启动文件时不能复位该选项,允许启动文件发现shell是否受限
shift_verbose   | 如果该选项设置,当移动计数超出位置参量个数时,shift 内置命令将打印一个错误消息
sourcepath      | 如果设置,source 内置命令使用PATH的值来寻找作为参数提供的文件的目录.缺省为打开
source          | 点(.)的同义词

## 开启之后，以下 5 个模式匹配操作符将被识别：

- ?(pattern-list)   #所给模式匹配 0 次或 1 次；
- *(pattern-list)   #所给模式匹配 0 次以上包括 0 次；
- +(pattern-list)   #所给模式匹配 1 次以上包括 1 次；
- @(pattern-list)   #所给模式仅仅匹配 1 次；
- !(pattern-list)   #不匹配括号内的所给模式。

## 实例：

删除文件名不以 jpg 结尾的文件：

`rm -rf !(*jpg)`

删除文件名以 jpg 或 png 结尾的文件：

`rm -rf *@(jpg|png)`

# 其他反向删除方法

```bash
#方法一
find ./ -maxdepth 1 -type f ! -name 'a' -exec rm -f {} \;

#方法二
ls|grep -v 'a'|xargs -i rm -f {}
```

> 参考链接：<https://www.cnblogs.com/tdcqma/p/5841256.html>