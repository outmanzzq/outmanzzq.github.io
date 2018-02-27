---
layout: wiki
title: PyCharm
categories: IDE
description: pycharm快捷键、常用设置、配置管理
keywords: python, pycharm, ide
---

> PyCharm是一个用于计算机编程的集成开发环境（IDE），主要用于Python语言开发，由捷克公司JetBrains开发。提供代码分析、图形化调试器，集成测试器、集成版本控制系统（Vcs），并支持使用Django进行网页开发。

# 一、PyCharm 3.0 默认快捷键

## 1. 编辑（Editing）

|快捷键|说明|
--|--
Ctrl + Space | 基本的代码完成（类、方法、属性）
Ctrl + Alt + Space | 快速导入任意类
Ctrl + Shift + Enter | 语句完成
Ctrl + P | 参数信息（在方法中调用参数）
Ctrl + Q | 快速查看文档
F1 | 外部文档
Shift + F1 | 外部文档，进入web文档主页
Ctrl + Shift + Z | Redo 重做
Ctrl + 悬浮/单击鼠标左键 | 简介/进入代码定义
Ctrl + F1 | 显示错误描述或警告信息
Alt + Insert | 自动生成代码
Ctrl + O | 重新方法
Ctrl + Alt + T | 选中
Ctrl + / | 行注释/取消行注释
Ctrl + Shift + / | 块注释
Ctrl + W | 选中增加的代码块
Ctrl + Shift + W | 回到之前状态
Ctrl + Shift + ]/[ | 选定代码块结束、开始
Alt + Enter | 快速修正
Ctrl + Alt + L | 代码格式化
Ctrl + Alt + O | 优化导入
Ctrl + Alt + I | 自动缩进
Tab / Shift + Tab | 缩进、不缩进当前行
Ctrl+X/Shift+Delete | 剪切当前行或选定的代码块到剪贴板
Ctrl+C/Ctrl+Insert | 复制当前行或选定的代码块到剪贴板
Ctrl+V/Shift+Insert | 从剪贴板粘贴
Ctrl + Shift + V | 从最近的缓冲区粘贴
Ctrl + D | 复制选定的区域或行
Ctrl + Y | 删除选定的行
Ctrl + Shift + J | 添加智能线
Ctrl + Enter | 智能线切割
Shift + Enter | 另起一行
Ctrl + Shift + U | 在选定的区域或代码块间切换
Ctrl + Delete | 删除到字符结束
Ctrl + Backspace | 删除到字符开始
Ctrl + Numpad+/- | 展开/折叠代码块（当前位置的：函数，注释等）
Ctrl + shift + Numpad+/- | 展开/折叠所有代码块
Ctrl + F4 | 关闭运行的选项卡

## 2. 查找/替换(Search/Replace)

|快捷键|说明|
--|--
F3 | 下一个
Shift+F3| 前一个
Ctrl +R | 替换
Ctrl+Shift+F | 或者连续2次敲击 shift 全局查找{可以在整个项目中查找某个字符串什么的，如查找某个函数名字符串看之前是怎么使用这个函数的}
Ctrl+Shift+R | 全局替换

## 3. 运行(Running)

|快捷键|说明|
--|--
Alt + Shift + F10 | 运行模式配置
Alt + Shift + F9 | 调试模式配置
Shift + F10 | 运行
Shift + F9 | 调试
Ctrl + Shift + F10 | 运行编辑器配置
Ctrl + Alt + R | 运行 manage.py 任务

## 4. 调试(Debugging)

|快捷键|说明|
--|--
F8 | 跳过
F7 | 进入
Shift + F8 | 退出
Alt + F9 | 运行游标
Alt + F8 | 验证表达式
Ctrl + Alt + F8 | 快速验证表达式
F9 | 恢复程序
Ctrl + F8 | 断点开关
Ctrl + Shift + F8 | 查看断点

## 5. 导航(Navigation)

|快捷键|说明|
--|--
Ctrl + N | 跳转到类
Ctrl + Shift + N | 跳转到符号
Alt + Right/Left | 跳转到下一个、前一个编辑的选项卡（代码文件）
Alt + Up/Down | 跳转到上一个、下一个方法
F12 | 回到先前的工具窗口
Esc | 从工具窗口回到编辑窗口
Shift + Esc | 隐藏运行的、最近运行的窗口
Ctrl + Shift + F4 | 关闭主动运行的选项卡
Ctrl + G | 查看当前行号、字符号
Ctrl + E | 当前文件弹出，打开最近使用的文件列表
Ctrl+Alt+Left/Right | 后退、前进
Ctrl+Shift+Backspace | 导航到最近编辑区域 {差不多就是返回上次编辑的位置}
Alt + F1 | 查找当前文件或标识
Ctrl+B / Ctrl+Click | 跳转到声明
Ctrl + Alt + B | 跳转到实现
Ctrl + Shift + I | 查看快速定义
Ctrl + Shift + B | 跳转到类型声明
Ctrl + U | 跳转到父方法、父类
Ctrl + ]/[ | 跳转到代码块结束、开始
Ctrl + F12 | 弹出文件结构
Ctrl + H | 类型层次结构
Ctrl + Shift + H | 方法层次结构
Ctrl + Alt + H | 调用层次结构
F2 / Shift + F2 | 下一条、前一条高亮的错误
F4 / Ctrl + Enter | 编辑资源、查看资源
Alt + Home | 显示导航条F11书签开关
Ctrl + Shift + F11 | 书签助记开关
Ctrl + #[0-9] | 跳转到标识的书签
Shift + F11 | 显示书签

## 6. 搜索相关(Usage Search)

|快捷键|说明|
--|--
Alt + F7/Ctrl + F7 | 文件中查询用法
Ctrl + Shift + F7 | 文件中用法高亮显示
Ctrl + Alt + F7 | 显示用法

## 7. 重构(Refactoring)

|快捷键|说明|
--|--
F5 | 复制
F6 | 剪切
Shift + F6 | 重命名
Ctrl + F6 | 更改签名
Ctrl + Alt + N | 内联
Ctrl + Alt + M | 提取方法
Ctrl + Alt + V | 提取属性
Ctrl + Alt + F | 提取字段
Ctrl + Alt + C | 提取常量
Ctrl + Alt + P | 提取参数

## 8. 控制VCS/Local History

|快捷键|说明|
--|--
Ctrl + K | 提交项目
Ctrl + T | 更新项目
Alt + Shift + C | 查看最近的变化
Alt + BackQuote(’) | VCS 快速弹出

## 9. 模版(Live Templates)

|快捷键|说明|
--|--
Ctrl + Alt + J | 当前行使用模版
Ctrl +Ｊ | 插入模版

## 10. 基本(General)

|快捷键|说明|
--|--
Alt + #[0-9] | 打开相应的工具窗口
Ctrl + Alt + Y | 同步
Ctrl + Shift + F12 | 最大化编辑开关
Alt + Shift + F | 添加到最喜欢
Alt + Shift + I | 根据配置检查当前文件
Ctrl + BackQuote(’) | 快速切换当前计划
Ctrl + Alt + S　| 打开设置页
Ctrl + Shift + A | 查找编辑器里所有的动作
Ctrl + Tab | 在窗口间进行切换

# 二、pycharm常用设置

> pycharm 中的设置是可以导入和导出的，file>export settings 可以保存当前pycharm 中的设置为jar文件，重装时可以直接import settings>jar 文件，就不用重复配置了。

## file -> Setting ->Editor

### 1. 设置Python自动引入包

要先在 >general > autoimport -> python :show popup
快捷键：Alt + Enter: 自动添加包

### 2. “代码自动完成”时间延时设置

Code Completion -> Auto code completion in (ms):0 -> Autopopup in (ms):500

### 3. Pycharm 中默认是不能用 Ctrl+ 滚轮改变字体大小的，可以在〉Mouse 中设置

### 4. 显示“行号”与“空白字符”

Appearance -> 勾选“Show line numbers”、“Show whitespaces”、“Show method separators”

### 5. 设置编辑器“颜色与字体”主题

Colors & Fonts -> Scheme name -> 选择”monokai”“Darcula”
说明：先选择“monokai”，再“Save As”为”monokai-pipi”，因为默认的主题是“只读的”，一些字体大小颜色什么的都不能修改，拷贝一份后方可修改！
修改字体大小
Colors & Fonts -> Font -> Size -> 设置为“14”

### 6. 设置缩进符为制表符“Tab”

File -> Default Settings -> Code Style
-> General -> 勾选“Use tab character”
-> Python -> 勾选“Use tab character”
-> 其他的语言代码同理设置

### 7. 去掉默认折叠

Code Folding -> Collapse by default -> 全部去掉勾选

### 8. pycharm默认是自动保存的，习惯自己按ctrl + s 的可以进行如下设置：

General -> Synchronization -> Save files on frame deactivation 和 Save files automatically if application is idle for .. sec 的勾去掉

Editor Tabs -> Mark modified tabs with asterisk 打上勾

### 9. >file and code template>python scripts

```python
#!/usr/bin/env python
#-- coding: utf-8 --

title = "$Package_name"
author = "$USER"
mtime = "$DATE"

code is far away from bugs with the god animal protecting

I love animals. They taste delicious.
          ┏┓      ┏┓
        ┏┛┻━━━┛┻┓
        ┃      ☃      ┃
        ┃  ┳┛  ┗┳  ┃
        ┃      ┻      ┃
        ┗━┓      ┏━┛
            ┃      ┗━━━┓
            ┃  神兽保佑    ┣┓
            ┃　永无BUG！   ┏┛
            ┗┓┓┏━┳┓┏┛
              ┃┫┫  ┃┫┫
              ┗┻┛  ┗┻┛
```

### 10 python 文件默认编码

File Encodings> IDE Encoding: UTF-8;Project Encoding: UTF-8;

### 11. 代码自动整理设置

这里 line breaks 去掉√，否则bar, 和 baz 会分开在不同行，不好看。

## File -> Settings -> appearance

### 1. 修改 IDE 快捷键方案

Keymap
1) execute selection in console : add keymap > ctrl + enter

> 系统自带了好几种快捷键方案，下拉框中有如“defaul”,“Visual Studio”,在查找Bug时非常有用,“NetBeans 6.5”,“Default for GNOME”等等可选项，因为“Eclipse”方案比较大众，个人用的也比较多，最终选择了“Eclipse”。

还是有几个常用的快捷键跟 Eclipse 不一样，为了能修改，还得先对 Eclipse 方案拷贝一份：

#### (1).代码提示功能，默认是【Ctrl+空格】，现改为跟Eclipse一样，即【Alt+/】

Main menu -> code -> Completion -> Basic -> 设置为“Alt+/”
Main menu -> code -> Completion -> SmartType -> 设置为“Alt+Shift+/”
不过“Alt+/”默认又被
Main menu -> code -> Completion -> Basic -> Cyclic Expand Word 占用，先把它删除再说吧（单击右键删除）！

#### (2).关闭当前文档，默认是【Ctrl+F4】，现改为跟Eclipse一样，即【Ctrl+W】

Main menu -> Window -> Active Tool Window -> Close Active Tab -> 设置为 “Ctrl+F4”;
Main menu -> Window -> Editor -> Close -> 设置为 “Ctrl+W”;

### 2.设置 IDE 皮肤主题

Theme -> 选择“Alloy.IDEA Theme”
或者在setting中搜索theme可以改变主题，所有配色统一改变

## File > settings > build.excution

每次打开 python 控制台时自动执行代码

```python
console > pyconsole
import sys

print(‘Python %s on %s’ % (sys.version, sys.platform))

sys.path.extend([WORKING_DIR_AND_PYTHON_PATHS])
import os
print(‘current workdirectory : ‘, os.getcwd() )
import numpy as np
import scipy as sp
import matplotlib as mpl
```

如果安装了 ipython，则在 pyconsole 中使用更强大的 ipython

console
选中 use ipython if available

这样每次打开 pyconsole 就会打开 ipython

> Note: 在virtualenv中安装 ipython: (ubuntu_env) pika:/media/pika/files/mine/python_workspace/ubuntu_env$pip install ipython

## File > settings > Languages & Frameworks

如果在项目设置中开启了 django 支持，打开 python console 时会自动变成打开django console，当然如果不想这样就关闭项目对 django 的支持：

如果打开支持就会在 settings > build.excution > console 下多显示一个django console:

Django console 设置如下

```python
import sys
print(‘Python %s on %s’ % (sys.version, sys.platform))
import django
print(‘Django %s’ % django.get_version())
sys.path.extend([WORKING_DIR_AND_PYTHON_PATHS])
if ‘setup’ in dir(django): django.setup()
import django_manage_shell; django_manage_shell.run(PROJECT_ROOT)
```

## File > settings > Project : initial project

project dependencies > LDA > project depends on these projects >

选择 sim_cluster 就可以在 LDA 中调用 sim_cluster 中的包

[Configure PyCharm]

pycharm 环境和路径配置

python 解释器路径

python 项目解释器路径

用于配置 python 项目执行的 python 路径

比如，有的项目是运行的是系统 python2.7 下的环境；有的是3.4；有的项目使用的是virtualenv 的 python 环境[python 虚拟环境配置 - pycharm 中的项目配置]

在 pycharm > file > settings > project:pythonworkspace > project

interpreter > 选择对应项目 > project interpreter 中指定 python 解释器

pycharm 中运行 configuration 有一个选项 add content roots to pythonpath
选中后 sys.path 中会多一整个项目 project 的路径

/media/pika/files/mine/python_workspace，里面的目录就被当成包使用，这样就可以通过 from SocialNetworks.SocialNetworks 引入不是 python 包的目录中的文件了。

不过最好使用 sys.path.append(os.path.join(os.path.split(os.path.realpath(file))[0],”../..”))来添加，这样在 pycharm 外也可以运行不出错 。

## [python 模块导入及属性：import ]

pycharm 中进行 python 包管理
pycharm 中的项目中可以包含 package、目录（目录名可以有空格）、等等

目录的某个包中的某个py文件要调用另一个py文件中的函数，首先要将目录设置为source root，这样才能从包中至上至上正确引入函数，否则怎么引入都出错：
SystemError: Parent module ” not loaded, cannot perform relative import

> Note:目录 > 右键 > make directory as > source root

## python 脚本解释路径

ctrl + shift + f10 / f10 执行 python 脚本时

当前工作目录 cwd 为 run/debug configurations 中的 working directory
可在 edit configurations > project or defaults 中配置

console 执行路径和当前工作目录
python console 中执行时

cwd为 File > settings > build.excution > console > pyconsole 中的working directory
并可在其中配置

## pycharm 配置 os.environ 环境

pycharm 中 os.environ 不能读取到 terminal 中的系统环境变量
pycharm 中 os.environ 不能读取 .bashrc 参数

使用 pycharm，无论在 python console 还是在 module 中使用 os.environ 返回的 dict 中都没有 ~/.bashrc 中的设置的变量，但是有 /etc/profile 中的变量配置。然而在terminal中使用 python，os.environ 却可以获取 ~/.bashrc 的内容。

解决方法1：
在 ~/.bashrc 中设置的系统环境只能在 terminal shell 下运行 spark 程序才有效，因为.bashrc is only read for interactive shells.

如果要在当前用户整个系统中都有效（包括 pycharm 等等IDE），就应该将系统环境变量设置在 ~/.profile 文件中。

如果是设置所有用户整个系统，修改/etc/profile或者/etc/environment吧。
如 SPARK_HOME 的设置[Spark：相关错误总结 ]

解决方法2：在代码中设置，这样不管环境有没有问题了.

## spark environment settings

```python
import sys, os
os.environ[‘SPARK_HOME’] = conf.get(SECTION, ‘SPARK_HOME’)
sys.path.append(os.path.join(conf.get(SECTION, ‘SPARK_HOME’), ‘python’))
os.environ[“PYSPARK_PYTHON”] = conf.get(SECTION, ‘PYSPARK_PYTHON’)
os.environ[‘SPARK_LOCAL_IP’] = conf.get(SECTION, ‘SPARK_LOCAL_IP’)
os.environ[‘JAVA_HOME’] = conf.get(SECTION, ‘JAVA_HOME’)
os.environ[‘PYTHONPATH’] = ‘SPARKHOME/python/lib/py4j−0.10.3−src.zip:PYTHONPATH’
```

> pycharm 配置第三方库代码自动提示,参考[Spark 安装和配置]

## Pycharm 实用拓展功能

pycharm 中清除已编译.pyc中间文件
选中你的 workspace > 右键 > clean python compiled files

还可以自己写一个清除代码

pycharm 设置外部工具
[python 小工具 ]针对当前pycharm中打开的py文件对应的目录删除其中所有的 pyc文件。如果是直接运行（而不是在下面的 tools 中运行），则删除E:\mine\python_workspace\WebSite 目录下的 pyc 文件。

将上面的删除代码改成外部工具
PyCharm > settings > tools > external tools > +添加
Name: DelPyc
program: PyInterpreterDirectory/python Python安装路径
Parameters: ProjectFileDir/Oth/Utility/DelPyc.py FileDir
Work directory: FileDir

> Note:Parameters 后面的 FileDir 参数是说，DelPyc 是针对当前 pycharm 中打开的py文件对应的目录删除其中所有的pyc文件。

之后可以通过下面的方式直接执行

> Note:再添加一个 Tools 名为 DelPycIn

program: Python 安装路径，e.g. D:\python3.4.2\python.exe
Parameters: E:\mine\python_workspace\Utility\DelPyc.py
Work directory 使用变量 FileDir

参数中没有 FileDir，这样就可以直接删除常用目录r’E:\mine\python_workspace\WebSite’了，两个一起用更方便
代码质量

当你在打字的时候，PyCharm 会检查你的代码是否符合 PEP8。它会让你知道，你是否有太多的空格或空行等等。如果你愿意，你可以配置 PyCharm 运行 pylint 作为外部工具。

python2 转 python3 最快方式
/usr/bin/2to3 -wn FileDir
