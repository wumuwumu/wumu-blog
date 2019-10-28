---
title: php5环境搭建
date: 2019-09-02 23:14:11
tags:
---

# 安装nginx

```bash
yum install epel-release
yum install nginx

```

# 安装php

remi源可以获取更高的版本，php-fpm是要启动的

```bash
rpm -ivh http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
yum install --enablerepo=remi --enablerepo=remi-php56 php php-fpm
yum install --enablerepo=remi --enablerepo=remi-php56 php-opcache php-mbstring php-mysql* php-gd php-redis php-mcrypt php-xml php-redis
```

# nginx配置

```nginx
server {
    listen       80;
    server_name  www.test.com test.com;
    root     /data/www/Public;
    index  index.php index.html;

    location / {
            try_files $uri $uri/ /index.php?$args;
    }
    location ~ index.php {
        fastcgi_connect_timeout 20s;     # default of 60s is just too long
        fastcgi_read_timeout 20s;       # default of 60s is just too long
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_pass 127.0.0.1:9000;    # assumes you are running php-fpm locally on port 9000
        fastcgi_param  PHP_VALUE  "open_basedir=/data/www/:/data/www/Data:/tmp/";
    }
}
```

# 开启php的日志

1. 修改 php-fpm.conf 文件，添加（或修改）如下配置：

   ```nginx
   [global]
     error_log = log/error_log
   
     [www]
     catch_workers_output = yes
   ```

2. 修改 php.ini 文件，添加（或修改）如下配置：

   ```
     log_errors = On
     error_log = "/usr/local/lnmp/php/var/log/error_log"
     error_reporting=E_ALL&~E_NOTICE
   ```

3. 重启 php-fpm 