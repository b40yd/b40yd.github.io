+++
title = "基础配置 - 如何自定义编辑器"
date = 2021-04-16
lastmod = 2021-04-16T18:13:10+08:00
tags = ["Emacs", "编辑器"]
categories = ["Emacs", "编辑器"]
draft = false
author = "7ym0n"
+++

## 配置文件 {#配置文件}

`Emacs` 启动会自动查找 `~/.emacs` ， `~/.emacs.el` 或者 `~/.emacs.d/init.el` 配置文件。 `Emacs` 还可以有一个默认的初始化文件
`default.el` ，位于 `Emacs` 的任何标准的 `package` 搜索目录下，其中 `Emacs` 的 `package` 搜索目录由 `load-path` 变量定义。除此之外，
`Emacs` 还有配置文件 (`site-wide startup file`)，称为 `site-start.el` ，也位于 `Emacs` 的任何标准的 `package` 搜索目录下。

`Emacs` 加载 `package` 中的配置是优先加载 `site-start.el` , 最后加载 `default.el` 。 `Emacs` 启动时，可以使用 `-q` 或 `–no-init-file`
选项来阻止 `Emacs` 加载初始化文件;如果初始化文件中将 `inhibit-default-init` 设置为 `t` ，那么 `Emacs` 不会加载 `default.el` ;最后，可以使用 `–no-site-file` 来禁止 `Emacs` 加载 `site-start.el` 配置文件。

还有一个特殊的 `early-init.el` 特殊的初始化配置文件.该配置文件在初始化 `package` 系统和 `GUI` 之前加载。


## 配置入口 {#配置入口}

`Emacs` 启动时自动调用 `package-initialize` 来导入已经安装了的包。 在运行 `after-init-hook` 之后完成的（参看 `startup` 简介）如果用户选项 `package-enable-at-startup` 被禁用，也就是 `package-enable-at-startup` 的值为 `nil` ，那么自动加载就不会被执行。 所以可以控制 `package` 的加载。为了方便管理及安装自定义的配置，通常会采用 `~/.emacs.d/init.el` 来作为配置文件的入口。主要用来初始化基础配置，然后自定义引导配置加载。


## 基础配置 {#基础配置}

在启动时，禁止加载已安装的 `package` 。禁止改变 `frame` 大小。为了好看，让滚动条和菜单栏、工具栏隐藏。这些都需要在 `GUI` 启动时设置，避免 `GUI` 启动时抖动。在 `~/.emacs.d/` 中创建一个 `early-init.el` 配置文件

```emacs-lisp
;; In Emacs 27+, package initialization occurs before `user-init-file' is
;; loaded, but after `early-init-file'. Doom handles package initialization, so
;; we must prevent Emacs from doing it early!
(setq package-enable-at-startup nil)

;; Inhibit resizing frame
(setq frame-inhibit-implied-resize t)

(push '(menu-bar-lines . 0) default-frame-alist)
(push '(tool-bar-lines . 0) default-frame-alist)
(push '(vertical-scroll-bars) default-frame-alist)
```

创建 `init.el` 入口配置文件。设置其他配置加载路径。把核心配置放在 `lisp` 目录下，自定义的一些配置放入 `site-lisp` 目录下。在
`~/.emacs.d` 分别创建 `lisp` 和 `site-lisp` 文件夹。

```emacs-lisp
;; (add-to-list 'load-path (expand-file-name "lisp" user-emacs-directory))
(defun custom-config-load-path (&rest _)
  "Load lisp path"
  (dolist (dir '("lisp" "site-lisp"))
    (push (expand-file-name dir user-emacs-directory) load-path)))

(advice-add #'package-initialize :after #'custom-config-load-path)
(custom-config-load-path)
```
