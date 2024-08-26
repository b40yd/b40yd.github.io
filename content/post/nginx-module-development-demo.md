+++
title = "Nginx 模块开发实现配置开启命令Demo"
date = 2024-08-26T14:32:10+08:00
tags = ["Nginx"]
categories = ["Nginx"]
draft = false
author = "7ym0n"
+++

## 引言

Nginx 是一个高性能的 HTTP 服务器和反向代理服务器，广泛用于网站和应用的部署。通过开发自定义模块，您可以扩展 Nginx 的功能，以满足特定的需求。在本教程中，我们将演示如何开发一个简单的 Nginx 模块。

## 环境准备

在开始开发之前，请确保您已经安装了以下工具：

- **Nginx 源代码**：您可以从 [Nginx 官方网站](http://nginx.org/en/download.html) 下载最新版本的源代码。
- **GCC 编译器**：用于编译 C 语言代码。
- **Make 工具**：用于构建项目。

确保这些工具已经正确安装和配置。

## 创建 Nginx 模块

1. **下载 Nginx 源代码**

   ```sh
   wget http://nginx.org/download/nginx-1.26.2.tar.gz
   tar -zxvf nginx-1.26.2.tar.gz
   cd nginx-1.26.2
   ```

2. 创建模块目录和文件

在 Nginx 源代码目录下创建一个新的模块目录：
```sh
mkdir -p my_module
cd my_module
touch ngx_http_my_module.c
```

3. 编写模块代码

打开 ngx_http_my_module.c 文件，添加以下代码：
```c
#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>

typedef struct {
    ngx_flag_t my_bool;
} ngx_http_my_module_loc_conf_t;

static char *ngx_http_my_module_set_bool(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
static void *ngx_http_my_module_create_loc_conf(ngx_conf_t *cf);
static char *ngx_http_my_module_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child);
static ngx_int_t ngx_http_my_module_handler(ngx_http_request_t *r);

static ngx_command_t ngx_http_my_module_commands[] = {
    {
        ngx_string("my_bool"),
        NGX_HTTP_LOC_CONF|NGX_CONF_FLAG,
        ngx_http_my_module_set_bool,
        NGX_HTTP_LOC_CONF_OFFSET,
        offsetof(ngx_http_my_module_loc_conf_t, my_bool),
        NULL
    },
    ngx_null_command
};

static ngx_http_module_t ngx_http_my_module_ctx = {
    NULL,                          /* preconfiguration */
    NULL,                          /* postconfiguration */
    NULL,                          /* create main configuration */
    NULL,                          /* init main configuration */
    NULL,                          /* create server configuration */
    NULL,                          /* merge server configuration */
    ngx_http_my_module_create_loc_conf, /* create location configuration */
    ngx_http_my_module_merge_loc_conf   /* merge location configuration */
};

ngx_module_t ngx_http_my_module = {
    NGX_MODULE_V1,
    &ngx_http_my_module_ctx,       /* module context */
    ngx_http_my_module_commands,   /* module directives */
    NGX_HTTP_MODULE,               /* module type */
    NULL,                          /* init master */
    NULL,                          /* init module */
    NULL,                          /* init process */
    NULL,                          /* init thread */
    NULL,                          /* exit thread */
    NULL,                          /* exit process */
    NULL,                          /* exit master */
    NGX_MODULE_V1_PADDING
};

static ngx_int_t ngx_http_my_module_handler(ngx_http_request_t *r) {
    ngx_http_my_module_loc_conf_t *my_conf;
    my_conf = ngx_http_get_module_loc_conf(r, ngx_http_my_module);

    if (my_conf->my_bool) {
        ngx_buf_t *b;
        ngx_chain_t out;

        r->headers_out.content_type.len = sizeof("text/plain") - 1;
        r->headers_out.content_type.data = (u_char *) "text/plain";

        b = ngx_pcalloc(r->pool, sizeof(ngx_buf_t));
        out.buf = b;
        out.next = NULL;

        b->pos = (u_char *) "Feature is enabled";
        b->last = b->pos + sizeof("Feature is enabled") - 1;
        b->memory = 1;
        b->last_buf = 1;

        r->headers_out.status = NGX_HTTP_OK;
        r->headers_out.content_length_n = b->last - b->pos;

        ngx_http_send_header(r);
        return ngx_http_output_filter(r, &out);
    } else {
        return NGX_DECLINED;
    }
}

static char *ngx_http_my_module_set_bool(ngx_conf_t *cf, ngx_command_t *cmd, void *conf) {
    ngx_http_my_module_loc_conf_t *my_conf = conf;
    char *rv = ngx_conf_set_flag_slot(cf, cmd, conf);
    if (rv != NGX_CONF_OK) {
        return rv;
    }

    return NGX_CONF_OK;
}

static void *ngx_http_my_module_create_loc_conf(ngx_conf_t *cf) {
    ngx_http_my_module_loc_conf_t *conf;

    conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_my_module_loc_conf_t));
    if (conf == NULL) {
        return NGX_CONF_ERROR;
    }

    conf->my_bool = NGX_CONF_UNSET;

    return conf;
}

static char *ngx_http_my_module_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child) {
    ngx_http_my_module_loc_conf_t *prev = parent;
    ngx_http_my_module_loc_conf_t *conf = child;

    ngx_conf_merge_value(conf->my_bool, prev->my_bool, 0);

    return NGX_CONF_OK;
}
```

这个模块定义了一个 my_bool 配置指令，并在访问配置了该指令的 URL 时，根据 my_bool 的值返回不同的响应。

4. 编译和安装 Nginx

返回到 Nginx 源代码目录，编译并安装 Nginx：
```sh
cd ..
./configure --add-module=./my_module
make -j8
```

5. 配置 Nginx

编辑 Nginx 配置文件（nginx.conf），添加以下内容：
```nginx
http {
    server {
        listen 80;
        location /hello {
            my_bool on;
        }
    }
}
```

6. 启动 Nginx

启动或重启 Nginx：

```sh
./nginx 
```

7. 测试模块

在浏览器中访问 http://localhost/feature，你应该会看到配置的字符串 "Feature is enabled"。