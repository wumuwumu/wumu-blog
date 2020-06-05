---
title: odoo的dbfilter配置
date: 2019-12-16 17:37:50
tags:
 - odoo
---

# 关于 Odoo 的 dbfilter 配置项

## 概述

默认情况下首次访问odoo页面时，会要求选择要访问的数据库，db中的所有库都会被列出来供选择，这种在生产环境下通常是不希望的看到，如果在启动时指定连接的数据库名可以解决这个问题

1. .conf文件中指定 `db_name = xxx`
2. 或者启动命令加参数`-d xxx`

## dbfilter

当我们需要根据域名来匹配数据库时（比如saas环境）这样就不适用了，这个时候就可以用 dbfilter 这个配置项来实现

dbfilter 默认值为 `.*`

eg: `dbfilter = ^%h$` 表示按域名精确匹配数据库服务器中名称为域名的数据库

启动参数 `--db-filter='^%d$'` 表示按二级域名前缀精确匹配对应名称的数据库（注意：127.0.0.1访问时会被匹配为 127 库名）

可用的匹配替代符号有 %h 和 %d

### %h

%h 代表访问访问的域名，比如 www.abc.com

### %d

当访问地址为 www.abc.com 时 %d 为 abc
当访问地址为 shop.abc.com 时 %d 为 shop

## 相关源代码

odoo中的相应的解析代码

```python
def db_filter(dbs, httprequest=None):
    httprequest = httprequest or request.httprequest
    h = httprequest.environ.get('HTTP_HOST', '').split(':')[0]
    d, _, r = h.partition('.')
    if d == "www" and r:
        d = r.partition('.')[0]
    r = openerp.tools.config['dbfilter'].replace('%h', h).replace('%d', d)
    dbs = [i for i in dbs if re.match(r, i)]
    return dbs
```



