+++
title = "在Emacs里面SSH管理远程服务器"
date = 2021-08-28
lastmod = 2021-08-28T22:00:41+08:00
tags = ["Emacs", "SSH"]
categories = ["Emacs", "SSH"]
draft = false
author = "7ym0n"
+++

## 为什么要造一个这样的轮子 {#为什么要造一个这样的轮子}

以前这个不是我的刚需，以前也业务代码都是有完整的 `CI` 流程，所以基本上用的不多，都是把脚本交给 `CICD` 。这次换工作后，开发环境系统，完全转移至 `Mac` 了，导致一些以前将就用的软件，没法用，用了一些蹩脚的办法实现，导致配置起来麻烦。


## 主要功能 {#主要功能}

1.  创建远程连接会话
2.  持久化保存会话信息
3.  使用历史会话信息直接连接
4.  支持TOTP两步验证
5.  支持多个会话批量执行同样的命令
6.  支持shell命令行、标记选择执行、整个buffer重新执行
7.  支持加入移除连接会话批量执行


## TODO {#todo}

-   待实现的功能，不一定啥时候会做。
    -   文件上传
    -   文件下载


## 安装 {#安装}

主要依赖使用命令行工具：

-   OpenSSH 7.3
-   SSHPass 支持TOTP需要编译安装 [sshpass](https://github.com/7ym0n/dotfairy/blob/master/lisp/init-ssh.el)
-   OathTool TOTP自动获取工具。


### 安装sshpass {#安装sshpass}

```nil
git clone https://github.com/dora38/sshpass.git
cd sshpass
./bootstrap
./configure --prefix=/usr/local/bin
sudo make install
```


### 安装oathtool {#安装oathtool}

注意不同平台包名不一样, `MacOS`:

```nil
brew install oath-toolkit
```

`ArchLinux`:

```nil
sudo pacman -Syy oath-toolkit
```


## 效果 {#效果}

[预览](/ssh-manager.png)
