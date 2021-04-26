+++
title = "快捷键管理"
date = 2021-04-26
lastmod = 2021-04-26T23:58:17+08:00
tags = ["hydra"]
categories = ["hydra"]
draft = false
author = "7ym0n"
+++

## 快捷键管理 {#快捷键管理}

根据不同模式统一管理快捷键，这样可以不用复杂的组合快捷键来操作，有效的保护小手指。比如说，快捷键需要按 `C-c C-v C-j` 来操作某个命令时，如果按错键，会导致重复按多次，距离远的快捷键，小手指长期使用会很难受。


## 需求 {#需求}

-   根据主模式绑定一组快捷键，来减少组合键的使用


## 解决方案 {#解决方案}

可以使用 `pretty-hydra` 包来自定义快捷键，将组合键缩短至单键，会变得很容易。 使用 `pretty-hydra-define` 宏来定义一个模式的快捷键了，也可以在 `use-package` 中使用 `:pretty-hydra` 属性来定义快捷键。


## 实现代码 {#实现代码}

```emacs-lisp
(use-package pretty-hydra
  :bind ("C-c <f1>" . toggles-hydra/body)
  :init
  (pretty-hydra-define toggles-hydra (:title "Toggles" :color amaranth :quit-key "q")
    ("Basic"
     (("n" (if (fboundp 'display-line-numbers-mode)
               (display-line-numbers-mode (if display-line-numbers-mode -1 1))
             (global-linum-mode (if global-linum-mode -1 1)))
       "line number"
       :toggle (or (bound-and-true-p display-line-numbers-mode) global-linum-mode))
      ("a" global-aggressive-indent-mode "aggressive indent" :toggle t)
      ("d" global-hungry-delete-mode "hungry delete" :toggle t)
      ("e" electric-pair-mode "electric pair" :toggle t)
      ("c" flyspell-mode "spell check" :toggle t)
      ("s" prettify-symbols-mode "pretty symbol" :toggle t)
      ("l" global-page-break-lines-mode "page break lines" :toggle t)
      ("b" display-battery-mode "battery" :toggle t)
      ("i" display-time-mode "time" :toggle t)
      ("m" doom-modeline-mode "modern mode-line" :toggle t)))))
```
