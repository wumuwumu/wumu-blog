---
title: frp搭建
date: 2020-06-05 15:54:05
tags:
- frp
---

# 简述

frp是有个内网穿透的工具，分为客户端和服务端。客户端的程序名称是frpc，服务端的程序名称是frps。

# 服务器

## 下载

```
https://github.com/fatedier/frp/releases
```

## 配置文件

```toml
# frps.ini
[common]
bind_port = 7000   # 用于与客户端之间通信
```

## 运行程序

```bash
./frps -c ./frps.ini
```

# 客户端

## 配置文件

详细看<https://github.com/fatedier/frp/blob/master/README_zh.md#dashboard>

```toml
# frpc.ini
[common]
server_addr = x.x.x.x
server_port = 7000

[web]
type = http
local_port = 80
custom_domains = www.yourdomain.com

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 6000
```

## 运行程序

```bash
./frpc -c ./frpc.ini
```

