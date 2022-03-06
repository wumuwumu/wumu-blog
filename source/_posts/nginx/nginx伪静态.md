---
title: nginx伪静态
tags:
  - nginx
abbrlink: 66244ea6
date: 2019-09-02 22:26:40
---
# 伪静态

伪静态是一种可以把文件后缀改成任何可能的一种方法，如果我想把PHP文件伪静态成html文件，这种相当简单的。
nginx里使用伪静态是直接在nginx.conf 中写规则的，而apache要开启写模块(mod_rewrite)才能进行伪静态。
nginx只需要打开nginx.conf配置文件,然后在里面写需要的规则就可以了。

**1、Nginx伪静态案例：（Nginx用伪静态是不需要配置的）**

找到nginx.conf配置文件：nginx.conf，然后打开，找到server {} 在里面加上：

下面加的意思是隐藏掉index.php：

```nginx
location / {         
    # 其他的一些规则，自己加
    if(!-e $request_filename) {         
        rewrite  ^(.*)$  /index.php?s=$1  last; 
        break;  
    }
}
```

**2、每个网站独立的配置文件（独立的伪静态规则）：**

我们正常的时候每个网站都会有独立的配置文件，直接去改配置文件就好了。然后nginx.conf引入他们所有的配置文件就好了：

如：在nginx.conf配置文件最下面添加以下代码：

```nginx
include vhost/*.conf;
```

说明：引入nginx.conf配置文件所在目录下vhost目录下的所有以.conf的配置文件！

以下就是其中一个网站的配置文件内容：规则就是隐藏掉index.php

```nginx
server {
        listen       80;
        root /www/web/admin/public;
        server_name www.admin.com;
        index  index.html index.php index.htm;
        error_page  400 /errpage/400.html;
        error_page  403 /errpage/403.html;
        error_page  404 /errpage/404.html;
        error_page  503 /errpage/503.html;
        location ~ \.php$ {
                fastcgi_pass   127.0.0.1:9000;
                fastcgi_index  index.php;
                include fcgi.conf;
        }
        location ~ /\.ht {
                deny  all;
        }
        location / { 
            if (!-e $request_filename) {
                 rewrite  ^(.*)$  /index.php?s=$1  last;
                 break;
            }
        }
}
```

# nginx url重写

url重写是指通过配置conf文件，以让网站的url中达到某种状态时则定向/跳转到某个规则，比如常见的伪静态、301重定向、浏览器定向等

## rewrite

### 语法

在配置文件的`server`块中写，如：

```nginx
server {   
    rewrite 规则 定向路径 重写类型;
}
```

- 规则：可以是字符串或者正则来表示想匹配的目标url
- 定向路径：表示匹配到规则后要定向的路径，如果规则里有正则，则可以使用`$index`来表示正则里的捕获分组
- 重写类型：
  - last ：相当于Apache里德(L)标记，表示完成rewrite，浏览器地址栏URL地址不变
  - break；本条规则匹配完成后，终止匹配，不再匹配后面的规则，浏览器地址栏URL地址不变
  - redirect：返回302临时重定向，浏览器地址会显示跳转后的URL地址
  - permanent：返回301永久重定向，浏览器地址栏会显示跳转后的URL地址

### 简单例子

```nginx
server {
    # 访问 /last.html 的时候，页面内容重写到 /index.html 中
    rewrite /last.html /index.html last;
    # 访问 /break.html 的时候，页面内容重写到 /index.html 中，并停止后续的匹配
    rewrite /break.html /index.html break;
    # 访问 /redirect.html 的时候，页面直接302定向到 /index.html中
    rewrite /redirect.html /index.html redirect;
    # 访问 /permanent.html 的时候，页面直接301定向到 /index.html中
    rewrite /permanent.html /index.html permanent;
    # 把 /html/*.html => /post/*.html ，301定向
    rewrite ^/html/(.+?).html$ /post/$1.html permanent;
    # 把 /search/key => /search.html?keyword=key
    rewrite ^/search\/([^\/]+?)(\/|$) /search.html?keyword=$1 permanent;
}
```

#### last和break的区别

因为301和302不能简单的只返回状态码，还必须有重定向的URL，这就是return指令无法返回301,302的原因了。这里 last 和 break 区别有点难以理解：

- last一般写在server和if中，而break一般使用在location中
- last不终止重写后的url匹配，即新的url会再从server走一遍匹配流程，而break终止重写后的匹配
- break和last都能组织继续执行后面的rewrite指令

在`location`里一旦返回`break`则直接生效并停止后续的匹配`location`

```nginx
server {
    location / {
        rewrite /last/ /q.html last;
        rewrite /break/ /q.html break;
    }
    location = /q.html {
        return 400;
    }
}
```

- 访问`/last/`时重写到`/q.html`，然后使用新的`uri`再匹配，正好匹配到`locatoin = /q.html`然后返回了`400`
- 访问`/break`时重写到`/q.html`，由于返回了`break`，则直接停止了

## if判断

只是上面的简单重写很多时候满足不了需求，比如需要判断当文件不存在时、当路径包含xx时等条件，则需要用到`if`

### 语法

```undefined
if (表达式) {}
```

- 当表达式只是一个变量时，如果值为空或任何以0开头的字符串都会当做false
- 直接比较变量和内容时，使用=或!=
- ~正则表达式匹配，~*不区分大小写的匹配，!~区分大小写的不匹配

一些内置的条件判断：

- -f和!-f用来判断是否存在文件
- -d和!-d用来判断是否存在目录
- -e和!-e用来判断是否存在文件或目录
- -x和!-x用来判断文件是否可执行

### 内置的全局变量

```
$args ：这个变量等于请求行中的参数，同$query_string
$content_length ： 请求头中的Content-length字段。
$content_type ： 请求头中的Content-Type字段。
$document_root ： 当前请求在root指令中指定的值。
$host ： 请求主机头字段，否则为服务器名称。
$http_user_agent ： 客户端agent信息
$http_cookie ： 客户端cookie信息
$limit_rate ： 这个变量可以限制连接速率。
$request_method ： 客户端请求的动作，通常为GET或POST。
$remote_addr ： 客户端的IP地址。
$remote_port ： 客户端的端口。
$remote_user ： 已经经过Auth Basic Module验证的用户名。
$request_filename ： 当前请求的文件路径，由root或alias指令与URI请求生成。
$scheme ： HTTP方法（如http，https）。
$server_protocol ： 请求使用的协议，通常是HTTP/1.0或HTTP/1.1。
$server_addr ： 服务器地址，在完成一次系统调用后可以确定这个值。
$server_name ： 服务器名称。
$server_port ： 请求到达服务器的端口号。
$request_uri ： 包含请求参数的原始URI，不包含主机名，如：”/foo/bar.php?arg=baz”。
$uri ： 不带请求参数的当前URI，$uri不包含主机名，如”/foo/bar.html”。
$document_uri ： 与$uri相同。
```

如：

```stylus
访问链接是：http://localhost:88/test1/test2/test.php 
网站路径是：/var/www/html
$host：localhost
$server_port：88
$request_uri：http://localhost:88/test1/test2/test.php
$document_uri：/test1/test2/test.php
$document_root：/var/www/html
$request_filename：/var/www/html/test1/test2/test.php
```

### 例子

```nginx
# 如果文件不存在则返回400
if (!-f $request_filename) {
    return 400;
}
# 如果host不是xuexb.com，则301到xuexb.com中
if ( $host != 'xuexb.com' ){
    rewrite ^/(.*)$ https://xuexb.com/$1 permanent;
}
# 如果请求类型不是POST则返回405
if ($request_method = POST) {
    return 405;
}
# 如果参数中有 a=1 则301到指定域名
if ($args ~ a=1) {
    rewrite ^ http://example.com/ permanent;
}
```

在某种场景下可结合`location`规则来使用，如：

```nginx
# 访问 /test.html 时
location = /test.html {
    # 默认值为xiaowu
    set $name xiaowu;
    # 如果参数中有 name=xx 则使用该值
    if ($args ~* name=(\w+?)(&|$)) {
        set $name $1;
    }
    # 301
    rewrite ^ /$name.html permanent;
}
```

上面表示：

- /test.html => /xiaowu.html
- /test.html?name=ok => /ok.html?name=ok

## location

### 语法

在`server`块中使用，如：

```nginx
server { 
    location 表达式 {    }
}
```

location表达式类型

- 如果直接写一个路径，则匹配该路径下的
- ~ 表示执行一个正则匹配，区分大小写
- ~* 表示执行一个正则匹配，不区分大小写
- ^~ 表示普通字符匹配。使用前缀匹配。如果匹配成功，则不再匹配其他location。
- = 进行普通字符精确匹配。也就是完全匹配。

### 优先级

1. 等号类型（=）的优先级最高。一旦匹配成功，则不再查找其他匹配项。
2. ^~类型表达式。一旦匹配成功，则不再查找其他匹配项。
3. 正则表达式类型（~ ~*）的优先级次之。如果有多个location的正则能匹配的话，则使用正则表达式最长的那个。
4. 常规字符串匹配类型。按前缀匹配。

### 例子 - 假地址掩饰真地址

```nginx
server {
    # 用 xxoo_admin 来掩饰 admin
    location / {
        # 使用break拿一旦匹配成功则忽略后续location
        rewrite /xxoo_admin /admin break;
    }
    # 访问真实地址直接报没权限
    location /admin {
        return 403;
    }
}
```

# 参考

<https://www.toolnb.com/tools/rewriteTools.html>