---
title: ngrok环境搭建
tags:
  - linux
abbrlink: ef774ead
date: 2019-03-29 09:28:43
---

# 下载安装

1. 配置golang环境

   1. 安装go

      ```bash
      yum install golang
      ```

   2. 配置`GOPATH`

2. 安装git2

   ```ba&#39;sh
   sudo yum remove git
   sudo yum install epel-release
   sudo yum install https://centos7.iuscommunity.org/ius-release.rpm
   sudo yum install git2u
   ```

3. 下载ngrok

   ```bash
   go get github.com/inconshreveable/ngrok
   ```

# 生成证书

   1. 使用let’s encrypt证书

      1. 申请证书（具体看申请证书，主要通配符证书和三级域名）

      2. 修改证书

         客户端证书

         ```bash
         cd ngrok
         cp /etc/letsencrypt/live/xncoding.com/chain.pem assets/client/tls/ngrokroot.crt
         ```

         服务端证书

         ```bash
         cp /etc/letsencrypt/live/xncoding.com/cert.pem assets/server/tls/snakeoil.crt
         cp /etc/letsencrypt/live/xncoding.com/privkey.pem assets/server/tls/snakeoil.key
         ```

# 编译

1. 编译服务端

   ```bash
   make release-server
   ```

2. 编译客户端

   不同平台的客户端需要分开编译。不同平台使用不同的 GOOS 和 GOARCH，GOOS为go编译出来的操作系统 (windows,linux,darwin)，GOARCH, 对应的构架 (386,amd64,arm)

   ```bash
   GOOS=linux GOARCH=amd64 make release-client
   GOOS=windows GOARCH=amd64 make release-client
   GOOS=linux GOARCH=arm make release-client
   ```

# 启动服务器

在开启之前，请主要端口是否开放

```bash
./ngrokd -domain=ngrok.sciento.top -httpAddr=:9580 -httpsAddr=:9443 -tunnelAddr=":9444"
```

# 启动客户端

1. 配置文件,具体看官方文档

   ```
   server_addr: "ngrok.sciento.top:9444"
   trust_host_root_certs: false
   tunnels:
     http:
       subdomain: "demo"
       proto:
         http: "9000"
         
     https:
       subdomain: "demo"
       proto:
         https: "9000"
   
   ```

2. 启动

   ```bash
   ./ngrok -config=ngrok.cfg start http https
   ```

# nginx配置

1. 安装nginx

2. 配置

   ```nginx
   server {
       listen       80;
       server_name  demo.ngrok.xncoding.com;
       return       301 https://demo.ngrok.xncoding.com$request_uri;
   }
   
   server {
       listen       443 ssl http2;
       server_name  demo.ngrok.xncoding.com;
   
       charset utf-8;
   
       ssl_certificate /etc/letsencrypt/live/demo.ngrok.xncoding.com/fullchain.pem;
       ssl_certificate_key /etc/letsencrypt/live/demo.ngrok.xncoding.com/privkey.pem;
       ssl_trusted_certificate /etc/letsencrypt/live/demo.ngrok.xncoding.com/chain.pem;
   
       access_log /var/log/nginx/ngrok.log main;
       error_log /var/log/nginx/ngrok_error.log error;
   
       location / {
           proxy_pass http://127.0.0.1:5442;
           proxy_redirect off;
           proxy_set_header Host       $http_host:5442;
           proxy_set_header X-Real-IP  $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       }
   }
   ```


# 参考

https://www.xncoding.com/2017/12/29/web/ngrok.html

https://www.coldawn.com/how-to-issue-acmev2-wildcard-certificates-with-certbot-on-centos-7/

https://www.jianshu.com/p/c5c9d071e395

http://ngrok.cn/docs.html#tcp