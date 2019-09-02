---
title: nginx配置
date: 2018-12-05 21:47:32
tags:
- web
- linux
---

## 配置web服务器

```
server {
    listen      80;
    server_name api.lufficc.com  *.lufficc.com;
    location /images/ {
        root /data;
    }

    location / {
        proxy_pass https://lufficc.com;
    }
}
```

## 反向代理

```
server{
      listen 80;
      server_name search.lufficc.com;
      location / {
              proxy_pass https://www.baidu.com;
      }
}
```

# 参考

> https://lufficc.com/blog/configure-nginx-as-a-web-server
>
> https://blog.csdn.net/hj7jay/article/details/53905943 http://www.nginx.cn/76.html