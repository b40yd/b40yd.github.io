---
title: "实战vector采集nginx日志，并使用clickhouse存储"
date: 2024-11-13T13:31:14+08:00
tags: ["Python", "vector", "nginx", "clickhouse", "docker"]
categories: ["Python", "vector", "nginx", "clickhouse", "docker"]
draft: false
author: "b40yd"
---

## 背景

针对web应用访问情况等，记录日志并使用分析。

## vector

对于日志采集并存储的方法非常多，这里主要介绍使用rust开发的vector。

[vector](https://vector.dev) 是什么？

    Vector 是一种高性能的可观察性数据管道，可以收集、转换所有日志、指标和跟踪信息（ logs, metrics, and traces），并将其写到您想要的存储当中；Vector 可以实现显着的成本降低、丰富的数据处理和数据安全.

需要使用的组件：

- vector
- nginx
- clickhouse
- docker

## 环境配置

使用 docker 安装等方式来进行演示。
需要用到自定义镜像、配置文件等。需要规划好目录。

```text
├── docker # 存放镜像构建的配置文件
│   ├── nginx_vector
│   │   ├── Dockerfile
│   │   └── entrypoint.sh
│   └── repos # 存放镜像源替换
│       └── debian.list
├── docker-compose.yaml
├── nginx.conf # 基础的nginx配置
├── nginx_conf.d
│   └── default.conf # 配置nginx server
└── vector.toml # vector配置
```

使用nginx镜像为基础镜像，构建一个vector-nginx的镜像。 文件`docker/nginx_vector/Dockerfile`的内容如下:

```Dockerfile
FROM nginx:latest
COPY docker/repos/debian.list /etc/apt/sources.list.d/debian.list
RUN apt-get update -y
RUN apt install -y apt-transport-https ca-certificates
RUN apt-get install -y curl procps net-tools vim
RUN bash -c "$(curl -L https://setup.vector.dev)"
RUN apt-get install -y vector
RUN rm -rf /var/log/nginx/*.log
COPY docker/nginx_vector/entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
```

文件`docker/nginx_vector/entrypoint.sh`的内容如下：
```sh
#!/bin/bash

echo "Starting Nginx and Vector..."
nginx & # 优先启动nginx
vector --config /etc/vector/vector.toml 
```

由于网络问题，需要将系统debian 12的源文件替换成国内的aliyun，也可以替换成其他源，`docker/repos/debian.list` 的内容如下：

```text
deb https://mirrors.aliyun.com/debian/ bookworm main non-free non-free-firmware contrib
deb-src https://mirrors.aliyun.com/debian/ bookworm main non-free non-free-firmware contrib
deb https://mirrors.aliyun.com/debian-security/ bookworm-security main
deb-src https://mirrors.aliyun.com/debian-security/ bookworm-security main
deb https://mirrors.aliyun.com/debian/ bookworm-updates main non-free non-free-firmware contrib
deb-src https://mirrors.aliyun.com/debian/ bookworm-updates main non-free non-free-firmware contrib
deb https://mirrors.aliyun.com/debian/ bookworm-backports main non-free non-free-firmware contrib
deb-src https://mirrors.aliyun.com/debian/ bookworm-backports main non-free non-free-firmware contrib
```

配置nginx，需要定义一个日志格式。这里采用json的方式记录。

```ngnix
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format json escape=json '{"timestamp":"$time_iso8601",'
            '"client_addr":"$remote_addr",'
            '"client_port":$remote_port,'
            '"client_user":"$remote_user",'
            '"request_body":"$request_body",'
            '"request":"$request",'
            '"request_length":$request_length,'
            '"server_addr":"$server_addr",'
            '"request_time":$request_time,'
            '"server_protocol":"$server_protocol",'
            '"ssl_protocol":"$ssl_protocol",'
            '"ssl_cipher":"$ssl_cipher",'
            '"upstream_addr":"$upstream_addr",'
            '"upstream_response_time":"$upstream_response_time",'
            '"upstream_status":"$upstream_status",'
            '"http_host":"$http_host",'
            '"http_referer":"$http_referer",'
            '"http_x_forwarded_for":"$http_x_forwarded_for",'
            '"http_user_agent":"$http_user_agent",'
            '"server_host":"$host",'
            '"server_port":$server_port,'
            '"uri":"$uri",'
            '"body_bytes_sent":$body_bytes_sent,'
            '"bytes_sent":$body_bytes_sent,'
            '"status":$status'
            '}';

    access_log  /var/log/nginx/access.log  json;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
```

定义一个基础的nginx server：

```nginx
server {
    listen 80;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
}
```

vector配置，主要将nginx的访问日志转换成clickhouse支持的数据格式，并记录。

注意： 

- 这里需要解析时间，如果格式不对，是无法被正确识别，clickhouse也会拒绝写入。

    `parse_timestamp`函数的用法： 

    1. 第一个参数，指定需要解析时间的字段名称
    2. 第二个参数，这里不是指定输出的格式，而是这个字段记录的时间格式（最开始以为是输出的格式，后来才发现是原时间time_iso8601的字符串格式）。

- 如果没有使用代理。upstream相关变量是没有值的，这里需要转换成数据库表的类型。


```toml
[sources.nginx_logs]
  type = "file"
  include = ["/var/log/nginx/access.log"]
  read_from = "beginning"

[transforms.nginx_logs_parser]
  type = "remap"
  inputs = ["nginx_logs"]
  source = ''' 
    . = parse_json!(string!(.message))
    .timestamp = format_timestamp!(parse_timestamp!(.timestamp, format: "%Y-%m-%dT%H:%M:%S%z"), format: "%Y-%m-%d %H:%M:%S")
    .client_addr = .client_addr
    .client_port = .client_port
    .client_user = .client_user
    .request_body = .request_body
    .request = .request
    .request_length = .request_length
    .request_time = .request_time
    .server_addr = .server_addr
    .server_protocol = .server_protocol
    .ssl_protocol = .ssl_protocol
    .ssl_cipher = .ssl_cipher
    .upstream_addr = .upstream_addr
    if .upstream_response_time == "" {
      .upstream_response_time = 0.0
    } else {
      .upstream_response_time = to_float!(.upstream_response_time)
    }
    if .upstream_status == "" {
      .upstream_status = 0
    } else { 
      .upstream_status = to_int!(.upstream_status)
    }
    .http_host = .http_host
    .http_referer = .http_referer
    .http_x_forwarded_for = .http_x_forwarded_for
    .http_user_agent = .http_user_agent
    .server_host = .server_host
    .server_port = .server_port
    .uri = .uri
    .body_bytes_sent = .body_bytes_sent
    .bytes_sent = .bytes_sent
    .status = .status
  '''

[sinks.clickhouse]
  type = "clickhouse"
  inputs = ["nginx_logs_parser"]
  endpoint = "http://clickhouse:8123"
  table = "nginx_logs"
  database = "default"
  compression = "none"
  skip_unknown_fields = true

[sinks.clickhouse.auth]
strategy = "basic"
user = "default"
password = "" 

[sinks.clickhouse.healthcheck]
  enabled = true

#[sinks.my_console]
#  type = "console"
#  inputs = [ "nginx_logs_parser" ]
#[sinks.my_console.encoding]
#  codec = "json"
```

最后准备好 docker-compose.yaml 文件。

```yaml
services:
  clickhouse:
    image: clickhouse/clickhouse-server
    restart: always
    environment:
      # - ALLOW_EMPTY_PASSWORD=yes
      - CLICKHOUSE_DB=default
      - CLICKHOUSE_USER=default
      - CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT=1
      - CLICKHOUSE_PASSWORD=
      - CLICKHOUSE_ADMIN_PASSWORD=
    ports:
      - '8123:8123'
      - '9000:9000'
    volumes:
      - ./clickhouse:/var/lib/clickhouse

  nginx_server:
    build:
      context: .
      dockerfile: docker/nginx_vector/Dockerfile
    restart: always
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./nginx_conf.d:/etc/nginx/conf.d
      - ./vector.toml:/etc/vector/vector.toml
    ports:
      - "8888:80"
    links:
      - clickhouse
    depends_on:
      - clickhouse
```

最后，别忘记创建clickhouse数据库表。

```sql
CREATE TABLE nginx_logs
(
	timestamp DateTime, 
	client_addr String, 
	client_port UInt64, 
	client_user String, 
	request_body String, 
	request String, 
	request_length UInt64, 
	server_addr String, 
	request_time Float64, 
	server_protocol String, 
	ssl_protocol String, 
	ssl_cipher String, 
	upstream_addr String, 
	upstream_response_time Float64, 
	upstream_status UInt16, 
	http_host String, 
	http_referer String, 
	http_x_forwarded_for String, 
	http_user_agent String, 
	server_host String, 
	server_port UInt64, 
	uri String, 
	body_bytes_sent UInt64, 
	bytes_sent UInt64, 
	status UInt64
)
ENGINE = MergeTree
ORDER BY timestamp;
```

可以关注[docker-dev-env](https://github.com/b40yd/docker-dev-env)项目，以上代码可以通过该项目，自动生成不同环境的配置。

### 使用docker-dev-env

通过git下载该项目源码：

```shell
git clone https://github.com/b40yd/docker-dev-env.git

pip install poetry

cd docker-dev-env

poetry install  # 安装依赖
```

生成关键配置文件。

```shell
poetry run python genenv.py --help 查看命令行帮助
poetry run python genenv.py gc # 可以加--help 查看子命令参数
```

输出：

```toml
[nginx]
port = 8888
path = "/etc/nginx"
config_path = "/etc/nginx/conf.d"
log_path = "/var/log/nginx"
worker_connections = "1024"

[clickhouse]
port = "8123"
database = "default"
user = "default"
password = ""
```

生成环境相关的文件配置。

```shell
poetry run python genenv.py gvnc #生成 vector-nginx-clickhouse相关的配置，如前面介绍的文件内容。
```

## 部署

使用docker compose部署环境。

```shell
cd outputs/vector-nginx-clickhouse

docker compose -f docker-compose.yaml up -d
```

导入clickhouse数据库表。

通过 `http://localhost:8123/play` 或者其他工具导入sql。

访问 `http://localhost:8888`。

通过 `http://localhost:8123/play` 查询是否成功记录日志。