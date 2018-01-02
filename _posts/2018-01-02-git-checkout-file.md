---
layout: post
title: Git 从远程仓库检出指定目录或文件
categories: git
description: Git 从远程仓库检出指定目录或文件
keywords: Git, git checkout
---

> 对于大型 Git 仓库，每次执行 Git 命令，都需要经过漫长的等待，特别是要经常执行的 git status 命令。
> 从 1.7.0 开始，Git 引入 sparse checkout（稀疏检出） 机制，稀疏检出机制允许只检出指定目录或者文件，这在大型 Git 仓库中，将大幅度缩短 Git 执行命令的时间。

> 要想只检出指定的目录或文件，需要在 .git/info/sparse-checkout 文件中指定目录或文件的路径，下面将以一个具体例子介绍 如何使用 Git 的 sparse checkout 。

**目录**

* TOC
{:toc}

# 环境准备

- OS: MacOS 10.12.6
- Git: git version 2.14.3 (Apple Git-98)

# 准备远程仓库

初始化一个仓库，目录结构如下：

```shell
benbendeMac:git-sparse-checkout-study benben$ tree
.
├── LICENSE
├── README.md      # 目标检出文件1
└── dir1
    ├── file1      # 目标检出文件2
    └── file2

1 directory, 4 files
```

将其推送到 Github 上新建的一个仓库，地址『例如』：
https://github.com/outmanzzq/git-sparse-checkout-study.git

# 为Git配置稀疏检出

换一个目录，再初始化一个 Git 仓库，以便用稀疏检出的方式，检出刚才在 Github 上新建的 git-sparse-checkout-study 仓库：

```shell
benbendeMac:git-sparse-checkout-study benben$ mkdir -p ~/tmp/git-sparse-checkout-study
benbendeMac:git-sparse-checkout-study benben$ cd ~/tmp/git-sparse-checkout-study/
benbendeMac:git-sparse-checkout-study benben$ git init
Initialized empty Git repository in /Users/benben/tmp/git-sparse-checkout-study/.git/

# 编辑该仓库目录下的 .git/info/sparse-checkout 文件，指定检出规则
benbendeMac:git-sparse-checkout-study benben$ ls -A
.git
benbendeMac:git-sparse-checkout-study benben$ cat >>.git/info/sparse-checkout <<EOF
> /README.md
> /dir1/file1
EOF
benbendeMac:git-sparse-checkout-study benben$ cat .git/info/sparse-checkout
/README.md
/dir1/file1
```

使用 git config core.sparseCheckout true 命令开启 Git 稀疏检出模式。然后编辑该仓库目录下的 .git/info/sparse-checkout 文件，指定检出规则。这里只检出 git-sparse-checkout-study 仓库中的 dir1 目录下的file1文件和 根目录下的 README.md 文件：

```shell
# 开启稀疏检出选项
benbendeMac:git-sparse-checkout-study benben$ git config core.sparseCheckout true
# 检查
benbendeMac:git-sparse-checkout-study benben$ git config --list
credential.helper=osxkeychain
user.email=gmoutman@gmail.com
user.name=outmanzzq
core.repositoryformatversion=0
core.filemode=true
core.bare=false
core.logallrefupdates=true
core.ignorecase=true
core.precomposeunicode=true
core.sparsecheckout=true    # 可看到稀疏检出选项已开启
```

# 检出

添加远程仓库地址，并检出：

```shell
benbendeMac:git-sparse-checkout-study benben$ git remote add origin https://github.com/outmanzzq/git-sparse-checkout-study.git
benbendeMac:git-sparse-checkout-study benben$ git pull origin master
remote: Counting objects: 4, done.
remote: Compressing objects: 100% (3/3), done.
remote: Total 4 (delta 0), reused 4 (delta 0), pack-reused 0
Unpacking objects: 100% (4/4), done.
From https://github.com/outmanzzq/git-sparse-checkout-study
 * branch            master     -> FETCH_HEAD
 * [new branch]      master     -> origin/master
benbendeMac:git-sparse-checkout-study benben$ tree
.
├── README.md
└── dir1
    └── file1
```

可以看到，Git 只检出了根目录下的 README.md 文件和 dir1 目录下 file1 文件。

如果此时需要再检出，根目录下的 dir1 目录下 file2 文件，则需要将其加入到 .git/info/sparse-checkout 文件中：

```shell
# 增加检出文件列表
benbendeMac:git-sparse-checkout-study benben$ cat >>.git/info/sparse-checkout <<EOF
> /dir1/file2
EOF
benbendeMac:git-sparse-checkout-study benben$ cat .git/info/sparse-checkout
/README.md
/dir1/file1
/dir1/file2

#检出
benbendeMac:git-sparse-checkout-study benben$ git read-tree -mu HEAD
benbendeMac:git-sparse-checkout-study benben$ tree
.
├── README.md
└── dir1
    ├── file1
    └── file2
```

# 关闭稀疏检出

```shell
benbendeMac:git-sparse-checkout-study benben$ cat >.git/info/sparse-checkout <<EOF
> /*
EOF
benbendeMac:git-sparse-checkout-study benben$ cat .git/info/sparse-checkout
/*
benbendeMac:git-sparse-checkout-study benben$ git read-tree -mu HEAD
benbendeMac:git-sparse-checkout-study benben$ tree
.
├── LICENSE
├── README.md
└── dir1
    ├── file1
    └── file2

1 directory, 4 files
benbendeMac:git-sparse-checkout-study benben$ git config core.sparseCheckout false
benbendeMac:git-sparse-checkout-study benben$ rm -f .git/info/sparse-checkout

```

可以看到所有文件都已显示出来了。
最后不要忘了配置 Git 的 core.sparseCheckout 为 false 以及移除 .git/info/sparse-checkout 文件。

.git/info/sparse-checkout 中使用和 .gitignore 相同的匹配模式，例如 非匹配 !/dir2/* 以及 /*.java 等。