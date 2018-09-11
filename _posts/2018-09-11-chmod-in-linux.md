---
layout: post
title: linux用chmod修改文件权限详解(文件权限与特殊权限)
categories: chmod,linux
description: linux 下用 chmod 修改文件权限详解(文件权限与特殊权限)
keywords: chmod,linux
---

> chmod 是一条在 Unix 系统中用于控制用户对文件的权限的命令（change mode 单词前缀的组合）和函数。只有文件所有者和超级用户可以修改文件或目录的权限。可以使用绝对模式，符号模式指定文件的权限。

# 一、八进制语法

chmod 命令可以使用八进制数来指定权限。文件或目录的权限位是由9个权限位来控制，每三位为一组，它们分别是文件所有者(user)的读、写、执行，用户组(group)的读、写、执行以及(other)其它用户的读、写、执行。历史上，文件权限被放在一个比特掩码中，掩码中指定的比特位设为 1，用来说明一个类具有相应的优先级。

## chmod 的八进制语法的数字说明；

八进制|十进制
-|-
r|4
w|2
x|1
-|0

> 所有者的权限用数字表达：属主的那三个权限位的数字加起来的总和。如 rwx ，也就是 4+2+1 ，应该是 7。

与文件所有者同属一个用户组的其他用户的权限用数字表达：属组的那个权限位数字的相加的总和。

如 rw- ，也就是 4+2+0 ，应该是 6。

其它用户组的权限数字表达：其它用户权限位的数字相加的总和。如r-x ，也就是 4+0+1 ，应该是 5。

# 二、符号模式

使用符号模式可以设置多个项目：who (用户类型），operator (操作符）和 permission (权限）,每个项目的设置可以用逗号隔开。 命令 chmod 将修改 who 指定的用户类型对文件的访问权限，用户类型由一个或者多个字母在 who 的位置来说明,如:

## who 的符号模式表所示

who|用户类型|说明
-|-|-
u | user   | 文件所有者
g | group  | 文件所有者所在组
o | others | 所有其他用户
a | all    | 所用用户, 相当于 ugo

## operator 的符号模式表

Operator | 说明
-|-
\+ | 为指定的用户类型增加权限
\- | 去除指定用户类型的权限
= | 设置指定用户权限的设置，即将用户类型的所有权限重新设置

## permission 的符号模式表

模式 | 名字 | 说明
-|-|-
r | 读 | 设置为可读权限
w | 写 | 设置为可写权限
x | 执行权限 | 设置为可执行权限
X | 特殊执行权限 | 只有当文件为目录文件，或者其他类型的用户有可执行权限时，才将文件权限设置可执行
s | setuid/gid | 当文件被执行时，根据who参数指定的用户类型设置文件的 setuid 或者 setgid 权限
t | 粘贴位 | 设置粘贴位，只有超级用户可以设置该位，只有文件所有者 u 可以使用该位

## 符号模式实例

对目录的所有者 u 和关联组 g 增加读 r 和写 w 权限:

```bash
$ chmod ug+rw mydir
$ ls -ld mydir
drw-rw----   2 unixguy  uguys  96 Dec 8 12:53 mydir</span>
```

# 三、Linux 中的文件特殊权限

## 1. setuid、setgid

先看个实例，查看你的 /usr/bin/passwd 与 /etc/passwd 文件的权限

```bash
[root@MyLinux ~]# ls -l /usr/bin/passwd /etc/passwd
-rw-r--r-- 1 root root  1549 08-19 13:54 /etc/passwd
-rwsr-xr-x 1 root root 22984 2007-01-07 /usr/bin/passwd
```

众所周知，/etc/passwd 文件存放的各个用户的账号与密码信息，/usr/bin/passwd 是执行修改和查看此文件的程序，但从权限上看，/etc/passwd 仅有 root 权限的写（w）权，可实际上每个用户都可以通过 /usr/bin/passwd 命令去修改这个文件，于是这里就涉及了 linux 里的特殊权限 setuid，正如 -rwsr-xr-x 中的 s

setuid 就是：让普通用户拥有可以执行“只有 root 权限才能执行”的特殊权限，setgid 同理指”组“

作为普通用户是没有权限修改 /etc/passwd 文件的，但给 /usr/bin/passwd 以 setuid 权限后，普通用户就可以通过执行 passwd 命令，临时的拥有 root 权限，去修改 /etc/passwd 文件了

## 2. stick bit （粘贴位)

再看个实例，查看你的 /tmp 目录的权限

```bash
[root@MyLinux ~]# ls -dl /tmp
drwxrwxrwt 6 root root 4096 08-22 11:37 /tmp
```

tmp 目录是所有用户共有的临时文件夹，所有用户都拥有读写权限，这就必然出现一个问题，A用户在 /tmp 里创建了文件 a.file，此时B用户看了不爽，在 /tmp 里把它给删了（因为拥有读写权限），那肯定是不行的。实际上是不会发生这种情况，因为有特殊权限 stick bit（粘贴位）权限，正如 drwxrwxrwt 中的最后一个 t

stick bit (粘贴位)就是：除非文件的属主和 root 用户有权限删除它，除此之外其它用户不能删除和修改这个文件。

也就是说，在 /tmp 目录中，只有文件的拥有者和 root 才能对其进行修改和删除，其他用户则不行，避免了上面所说的问题产生。用途一般是把一个文件夹的的权限都打开，然后来共享文件，象 /tmp 目录一样。

## 3. 如何设置以上特殊权限

- setuid：chmod u+s xxx

- setgid: chmod g+s xxx

- stick bit : chmod o+t xxx

或者使用八进制方式，在原先的数字前加一个数字，三个权限所代表的进制数与一般权限的方式类似，如下:

```script
suid   guid    stick bit

  1        1          1
```

所以：

- suid 的二进制串为：100，换算十进制为：4

- guid 的二进制串为:010,换算：2

- stick bit 二进制串：001，换算：1

于是也可以这样设:

```bash
setuid: chmod 4755 xxx

setgid: chmod 2755 xxx

stick bit: chmod 1755 xxx
```

最后，在一些文件设置了特殊权限后，字母不是小写的 s 或者 t，而是大写的 S 和 T，那代表此文件的特殊权限没有生效，是因为你尚未给它对应用户的 x 权限  （可以参考文章末尾的英文）

# 四、文件详细信息里各个符号的解释

```-rw-r--r-- 1 vivek admin 2558 Jan  8 07:41 filename```

- -rw-r--r-- : file mode+permission(下面有介绍)
- 1 - number of links（硬链接数）
- vivek - Owner name (if user name is not a known user, the numeric user id displayed)
- admin - Group name (if group name is not a known group, the numeric group id displayed)
- 2558 - number of bytes in the file (file size)
- Jan 8 07:41 - abbreviated month, day-of-month file was
last modified, hour file last modified, minute file last modified
- filename - File name / pathname

## ls -l file mode (permissions)

Quoting from the unix ls command man page - the file mode printed under the -l option consists of the entry type and the permissions. The entry type character describes the type of file, as follows:

char | remark
-|-
\- | Regular file.
b | Block special file.
c | Character special file.
d | Directory.
l | Symbolic link.
p | FIFO.
s | Socket.
w | Whiteout.

> d：表示是一个目录，事实上在 ext2fs 中，目录是一个特殊的文件。
> －：表示这是一个普通的文件。
> l: 表示这是一个符号链接文件，实际上它指向另一个文件。（即软连接）
> b、c：分别表示区块设备和其他的外围设备，是特殊类型的文件。
> s、p：这些文件关系到系统的数据结构和管道，通常很少见到。

The next three fields are three characters each: owner permissions, group permissions, and other permissions. Each field has three character positions:

1. If r, the file is readable; if -, it is not readable.
2. If w, the file is writable; if -, it is not writable.
3. The first of the following that applies:

- S : If in the owner permissions, the file is not executable and set-user-ID mode is set. If in the group permissions, the file is not executable and set-group-ID mode is set.
- s : If in the owner permissions, the file is executable and set-user-ID mode is set. If in the group permissions, the file is executable and set group-ID mode is set.
- x : The file is executable or the directory is searchable.
- \- : The file is neither readable, writable, executable, nor set-user-ID nor set-group-ID mode, nor sticky.

4.&nbsp;These next two apply only to the third character in the last group (other permissions).
- T : The sticky bit is set (mode 1000), but not execute or search permission.
- t : The sticky bit is set (mode 1000), and is search able or executable.

> 原文链接：<https://blog.csdn.net/gscaiyucheng/article/details/21367977>