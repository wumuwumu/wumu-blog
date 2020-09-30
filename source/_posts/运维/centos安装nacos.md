---
title: centos安装nacos
date: 2020-9-19 18:00:00
---

# 下载

不同版本有点配置有点差别

```
https://github.com/alibaba/nacos/releases
https://github.com/alibaba/nacos/releases/download/1.3.2/nacos-server-1.3.2.zip
```

# 配置数据库

```properties
#*************** Config Module Related Configurations ***************#
### If use MySQL as datasource:
# spring.datasource.platform=mysql

### Count of DB:
# db.num=1

### Connect URL of DB:
# db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
# db.user=nacos
# db.password=nacos
```

# 初始化数据库

```
mysqldump -h -u -p nacos < xx.sql
```

# 启动

```bash
./startup.sh -m standalone
```

# 编写service

```

[Unit]
Description=nacos
After=network.target
 
[Service]
Type=forking
ExecStart=/opt/nacos/bin/startup.sh -m standalone
ExecReload=/opt/nacos/bin/shutdown.sh
ExecStop=/opt/nacos/bin/shutdown.sh
PrivateTmp=true
 
[Install]  
WantedBy=multi-user.target


```

```
systemctl start nacos
systemctl enable nacos
```

# 参考

https://blog.csdn.net/weihuaya/article/details/108060847

