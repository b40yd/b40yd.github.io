+++
title = "依赖包管理 - 如何自定义编辑器"
date = 2021-04-16
lastmod = 2021-04-17T23:41:05+08:00
tags = ["Emacs", "编辑器", "package"]
categories = ["Emacs", "编辑器", "package"]
draft = false
author = "7ym0n"
+++

## 设置软件仓库源 {#设置软件仓库源}

`Emacs` 默认使用的是 `https://elpa.gnu.org/` 和 `https://melpa.org` 。 由于 `FW` 的原因，访问速度非常慢，经常超时，所以需要设置国内源。


## 需求 {#需求}

-   通过自定义配置文件指定仓库源
-   解决包 `git` 地址问题
-   解决多源管理问题
-   解决包升级时下载阻塞问题


## 基础使用 {#基础使用}

`Emacs` 默认使用 `package-list` 浏览官方的仓库列表，可使用的包比较少，主要是因为 `LICENSE` 不是 `GPL` 自由软件授权。

{{< figure src="/manual/package-default-manage.png" >}}


### 软件包列表 {#软件包列表}

使用 `M-x list-packages RET` 即可，会打开一个 `*Packages*` 的 `buffer` 显示软件包列表。这个时候可以使用快捷键进行管理。

| 快捷键 | 说明      |
|-----|---------|
| i   | 标记需要安装的包 |
| u   | 取消标记的安装包 |
| x   | 安装已标记的安装包 |
| n   | 下一个包  |
| p   | 上一个包  |
| d   | 删除      |
| H   | 正则过滤包并隐藏 |
| g   | 刷新列表  |
|     | / 清除过滤 |
|     | k 基于关键字过滤 |
|     | n 基于名字过滤 |
| h   | 帮助文档  |


### 安装 {#安装}

使用安装命令 `M-x package-install RET <package_name>` 可以直接输入包名进行安装。下载安装的包会自动存放在 `/.emacs.d/elpa/`
目录下。


### 重新安装 {#重新安装}

重新安装使用 `M-x package-reinstall RET <package_name>` 就可以了。


### 更新 {#更新}

升级包输入 `M-x upgrade-package RET` 即可自动更新已安装的软件包。


### 删除 {#删除}

如果需要删除，可以直接输入 `M-x package-delete RET <package_name>` 指定包名，进行删除。


### 刷新列表 {#刷新列表}

在获取包列表后，会自动缓存信息，如果需要刷新包列表缓存，需要使用 `package-refresh-contents` 刷新即可


## 解决方案 {#解决方案}

由于使用默认的方式，正常情况下是不够用的，很多第三方包是在 `melpa` 仓库中，还有的可能需要手动下载，这导致包管理起来比较麻烦，特别是升级时。因为 `emacser` 们喜欢折腾，很多时候包根本不稳定，会持续性更新和 `bug` 的修复。这个时候就需要解决仓库源和包升级的问题了，同时还需要解决包下载网络速度慢的问题。 `Emacs` 默认提供了一个 `package-archives` 变量，来设置仓库源，可以通过设置该变量，解决仓库源和网络问题（在特殊情况下，还是没办法一些包的问题，比如需要的包只有 `git` 仓库地址，并未发布至软件仓库）。

```emacs-lisp
(setq package-archives '(("gnu" . "https://elpa.gnu.org/packages/") ;; GNU ELPA repository (Offical)
                         ("melpa" . "https://melpa.org/packages/") ;; MELPA repository
                         ("melpa-stable" . "https://stable.melpa.org/packages/") ;; MELPA Stable repository
                         ("org" . "http://orgmode.org/elpa/"))) ;; Org-mode's repository
```

需要引入 `use-package` 和 `straight.el` 两个包管理扩展来解决包 `git` 下载地址和管理更新等问题了。这两个包即可配合也可单独使用，如果单独使用， `straight.el` 需要配合 `straight.el` 来使用了。

这里就只介绍 `use-package` 和 `straight.el` 的配合使用，来解决下载地址和包升级管理。在引入这两个包前，需要回顾一下，前面基础配置时，有一个禁止加载已安装包的变量，它就是 `package-enable-at-startup` ，我们需要结合该变量进行对包管理的升级改造。

解决包下载升级时阻塞，导致无法正常工作，必须等待完成的问题。需要引入一个第三方 `paradox` 包，来解决该问题，注意使用该包后，对应的快捷键也有变化，因为它实际是对默认的管理器做了升级改造。


## 代码实现 {#代码实现}
