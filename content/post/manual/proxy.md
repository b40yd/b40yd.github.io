+++
title = "设置代理 - 如何自定义编辑器"
date = 2021-04-16
lastmod = 2021-04-16T23:52:51+08:00
draft = false
author = "7ym0n"
+++

## 场景 {#场景}

`Emacs` 在下载包，或者第三方包在安装时依赖额外的软件，一般会提供自动下载，比如 `lsp-mode` 补全相关的 `server` 。这些第三方软件大多数放在国外的服务器上，由于 `FW` 的原因，需要使用代理服务器进行下载。


## 需求 {#需求}

-   指定自己的代理服务器
-   使用时一键打开和一键关闭
-   不同类型代理一键切换


## 设置代理 {#设置代理}

设置代理，有很多解决方案，支持常见的 `http` 、 `https` 、 `socks` 、 `rlogin` 和 `telnet` 等， 参考官方手册[^fn:1]。下面介绍一些常见的。


### 环境变量 {#环境变量}

Shell环境中可以设置 `https_proxy` 和 `http_proxy` 变量，这样用命令行启动，请求 `web` 服务，就会通过代理服务器。缺点也很明显，每次需要代理都需要设置环境变量，不能做到需要时打开，不需要时自动关闭，所以只满足了第一条需求。

```shell
`https_proxy=https://proxy.com:1080 http_proxy=http://proxy.com:1080 emacs -nw`
```


### 代码设置 {#代码设置}

通过环境变量设置，非常不方便，每次都需要设置该参数，变得很麻烦，并且，一点也不像一个 `emacser` 的风格。使用 `Emacs` 的原则就是，一切问题皆可用编程来解决问题。幸运的是 `Emacs` 内置提供了代理相关的设置，我们只需要对相关变量做设置即可。由于 `emacs` 发起网络请求是基于 `make-network-process` 的，所以最根本的方法是临时设置环境变量到 `process-environment` 里。默认代理设置使用的是 `url-proxy-services` 变量，如果代理需要认证，则需要设置另一个 `url-http-proxy-basic-auth-storage` 变量，这样用户可以根据情况设置自己的代理服务器了。

```emacs-lisp
(setq url-proxy-services
   '(("no_proxy" . "^\\(localhost\\|10\\..*\\|192\\.168\\..*\\)")
     ("http" . "proxy.com:1080")
     ("https" . "proxy.com:1080")))

(setq url-http-proxy-basic-auth-storage
    (list (list "proxy.com:1080"
                (cons "Input your LDAP UID !"
                      (base64-encode-string "LOGIN:PASSWORD")))))
```

目前也只能满足第一条需求。如果这样设置后，每次启动都会使用代理请求 `web` 服务。


### 快速设置代理 {#快速设置代理}

实现快速设置 `SOCKS v5 protocol` 代理和 `http` 的代理。目前已知可以设置 `url-proxy-services` 和
`url-http-proxy-basic-auth-storage` 变量来启用代理。那么我们可以通过自定义函数来实现相关的自动实现。 `socks` 代理设置，可以通过 `url-gateway-method` 变量设置，并设置 `socks-server` 变量，如果需要认证，需要设置 `socks-username` 和 `socks-password` 变量。

代码参考来自 `Centaur`[^fn:2]，通过 `C-x C-f` 打开 `~/.emacs.d/lisp/init-proxy.el` 文件，添加以下内容，并通过 `C-x C-s` 保存。

```emacs-lisp
(defvar default-proxy "127.0.0.1:1080")
(defvar socks-server)
(defvar socks-noproxy)
;; Network Proxy
(defun proxy-http-show ()
  "Show HTTP/HTTPS proxy."
  (interactive)
  (if url-proxy-services
      (message "Current HTTP proxy is `%s'" default-proxy)
    (message "No HTTP proxy")))

(defun proxy-http-enable ()
  "Enable HTTP/HTTPS proxy."
  (interactive)
  (setq url-proxy-services
        `(("http" . ,default-proxy)
          ("https" . ,default-proxy)
          ("no_proxy" . "^\\(localhost\\|192.168.*\\|10.*\\)")))
  (proxy-http-show))

(defun proxy-http-disable ()
  "Disable HTTP/HTTPS proxy."
  (interactive)
  (setq url-proxy-services nil)
  (proxy-http-show))

(defun proxy-http-toggle ()
  "Toggle HTTP/HTTPS proxy."
  (interactive)
  (if (bound-and-true-p url-proxy-services)
      (proxy-http-disable)
    (proxy-http-enable)))

(defun proxy-socks-show ()
  "Show SOCKS proxy."
  (interactive)
  (when (fboundp 'cadddr)                ; defined 25.2+
    (if (bound-and-true-p socks-noproxy)
        (message "Current SOCKS%d proxy is %s:%d"
                 (cadddr socks-server) (cadr socks-server) (caddr socks-server))
      (message "No SOCKS proxy"))))

(defun proxy-socks-enable ()
  "Enable SOCKS proxy."
  (interactive)
  (require 'socks)
  (setq url-gateway-method 'socks
        socks-noproxy '("localhost")
        socks-server '("Default server" "127.0.0.1" 1086 5))
  (proxy-socks-show))

(defun proxy-socks-disable ()
  "Disable SOCKS proxy."
  (interactive)
  (setq url-gateway-method 'native
        socks-noproxy nil)
  (proxy-socks-show))

(defun proxy-socks-toggle ()
  "Toggle SOCKS proxy."
  (interactive)
  (if (bound-and-true-p socks-noproxy)
      (proxy-socks-disable)
    (proxy-socks-enable)))
(provide 'init-proxy)
```

可以通过 `M-x` 输入执行 `eval-buffer` 运行当前 `buffer` 。再次执行 `M-x` 输入实现的这些命令即可，例如输入 `proxy-http-toggle` 就会在 `minibuffer` 显示当前代理服务的信息。

为了让 `Emacs` 启动自动加载以上配置，需要在 `init.el` 中去加载当前文件。如，在 `init.el` 文件末尾加入以下代码：

```emacs-lisp
(require 'init-proxy)
```

由于我们在基本配置中设置了 `load-path` 代码的加载路径，它默认将自动加载，无需指定文件的路径。

[^fn:1]: 代理设置 <https://www.gnu.org/software/emacs/manual/html%5Fnode/url/Gateways-in-general.html>
[^fn:2]: Centaur配置 <https://github.com/seagle0128/.emacs.d/>