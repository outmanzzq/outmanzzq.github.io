---
layout: post
title: Mac 使用 Shuttle 优雅切换 http(s) 代理
categories: mac,proxy
description: Mac 使用 Shuttle 优雅切换 http(s) 代理
keywords: mac,proxy
---

> 在 MacOS 中配置 http(s) 代理时，通常的做法是在系统偏好设置 - 网络中进行操作，选择一个网络，点击高级按钮，点击代理选项卡，分别勾选网页代理（HTTP）和安全网页代理（HTTPS）然后填写上代理信息。
>
> 这种配置方式虽然可以实现需求，但缺点在于操作比较繁琐，特别是在需要频繁切换的情况下，效率极其低下。
>
> 基于该痛点，我们希望能避免重复操作，实现快速切换配置。

## Terminal 中设置代理

要避免在 GUI 进行重复的配置操作，比较好的简化方式是在 Terminal 中通过命令实现同样的功能。

事实上，在 MacOS 系统中的确是存在配置网络代理的命令，该命令即是 networksetup。

### 获取系统已有的网络服务

首先需要明确的是，macOS 系统中针对不同网络服务（networkservice）的配置是独立的，因此在配置 Web 代理时需要进行指定。

而要获取系统中存在哪些网络服务，可以通过如下命令查看：

```bash
$ networksetup -listallnetworkservices
An asterisk (*) denotes that a network service is disabled.
LPSS Serial Adapter (1)
LPSS Serial Adapter (2)
USB 10/100 LAN
iPhone USB
Wi-Fi
Bluetooth PAN
Thunderbolt Bridge
```

> 如果计算机是通过 Wi-Fi 上网的，那么我们设置网络代理时就需要指定 Wi-Fi 进行设置。

### 开启 http(s) 代理

通过 networksetup 命令对 HTTP 接口设置代理时，可以采用如下命令：

```bash
$ sudo networksetup -setwebproxy <networkservice> <domain> <port number> <authenticated> <username> <password>
# e.g. sudo networksetup -setwebproxy "Wi-Fi" 127.0.0.1 8080
```

执行该命令时，会开启系统的 Web HTTP Proxy，并将 Proxy 设置为 127.0.0.1:8080。

如果是对 HTTPS 接口设置代理时，命令为：

```bash
$ networksetup -setsecurewebproxy <networkservice> <domain> <port number> <authenticated> <username> <password>
# e.g. sudo networksetup -setsecurewebproxy "Wi-Fi" 127.0.0.1 8080
```

### 关闭 http(s) 代理

相应地，关闭 HTTP 和 HTTPS 代理的命令为：

```bash
# 关闭 http 代理
$ sudo networksetup -setwebproxystate <networkservice> <on off>
# e.g. sudo networksetup -setwebproxystate "Wi-Fi" off

# 关闭 https 代理
$ networksetup -setsecurewebproxystate <networkservice> <on off>
# e.g. sudo networksetup -setsecurewebproxystate "Wi-Fi" off
```

## 使用 Shuttle 一键设置代理

现在我们已经知道如何通过 networksetup 命令在 Terminal 中进行 http(s) 代理切换了，但如果每次都要重新输入命令和密码，还是会很麻烦，并没有真正地解决我们的痛点。

而且在实际场景中，我们通常需要同时开启或关闭代理，这类操作如此高频，要是还能通过点击一个按钮就实现切换，那就优雅多了。

幸运的是，这种优雅的方式还真能实现，只需要结合使用 Shuttle 这么一款小工具。

[Shuttle](http://fitztrev.github.io/shuttle/)，简而言之，它可以将一串命令映射到 MacOS 顶部菜单栏的快捷方式。我们要做的很简单，只需要将要实现的任务拼接成一条串行的命令即可，然后就可以在系统菜单栏中点击按钮运行整条命令。

例如，在Terminal中，要想在不手动输入sudo密码的情况下实现同时关闭代理，就可以通过如下串行命令实现。

```bash
$ echo <password> | sudo -S networksetup -setwebproxystate 'Wi-Fi' off && sudo networksetup -setsecurewebproxystate 'Wi-Fi' off
```

### 配置Shuttle

打开Shuttle配置文件，

编辑内容，修改hosts字段内容为以下配置：

```json
"hosts": [
    {
        "HTTP(S) Proxy - Wi-Fi": [
            {
                "name": "Turn on HTTP(S) Proxy",
                "cmd": "echo <password> | sudo -S networksetup -setwebproxy 'Wi-Fi' 127.0.0.1 8080 && sudo networksetup -setsecurewebproxy 'Wi-Fi' 127.0.0.1 8080 && exit"
            },
            {
                "name": "Turn off HTTP(S) Proxy",
                "cmd": "echo <password> | sudo -S networksetup -setwebproxystate 'Wi-Fi' off && sudo networksetup -setsecurewebproxystate 'Wi-Fi' off && exit"
            }
        ]
    }
]
```

配置十分简洁清晰，不用解释也能看懂。完成配置后，点击 MacOS 顶部菜单栏的 Shuttle 图标就会出现如下效果的快捷方式。

## 完整 Shuttle 配置文件 Demo

```json
{
  "_comments": [
    "Valid terminals include: 'Terminal.app' or 'iTerm'",
    "In the editor value change 'default' to 'nano', 'vi', or another terminal based editor.",
    "Hosts will also be read from your ~/.ssh/config or /etc/ssh_config file, if available",
    "For more information on how to configure, please see http://fitztrev.github.io/shuttle/"
  ],
  "editor": "default",
  "launch_at_login": true,
  "terminal": "Terminal.app",
  "iTerm_version": "nightly",
  "default_theme": "Homebrew",
  "open_in": "new",  
  "show_ssh_config_hosts": false,
  "ssh_config_ignore_hosts": [  ],
  "ssh_config_ignore_keywords": [  ],
  "hosts": [
    {
      "HTTP(S) Proxy - Wi-Fi": [
        {
            "name": "Turn on HTTP(S) Proxy",
            "cmd": "echo 'password' |  sudo -S networksetup -setsecurewebproxy 'Wi-Fi' 127.0.0.1 8080 && exit"
        },
        {
            "name": "Turn off HTTP(S) Proxy",
            "cmd": "echo 'password' | sudo -S networksetup -setwebproxystate 'Wi-Fi' off && sudo networksetup -setsecurewebproxystate 'Wi-Fi' off && exit"
        }
      ]
    }
  ]
}
```

## FAQ

### 1. 关于通过 Shuttle 开启/关闭 proxy 后，调用系统 Terminal 窗口无法自动关闭问题

在Terminal.app中，选择左上角菜单：

终端=》偏好设置=》shell

![ ](/images/20190813-auto-switchproxy-for-macos-01.jpg)

> 原文链接：<https://tidyko.com/posts/67b16286.html>
