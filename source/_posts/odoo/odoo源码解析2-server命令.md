---
title: odoo源码解析2-server命令
date: 2019-11-16 11:40:07
tags:
- odoo
---

默认情况下的启动命令的server，这个是将odoo运行起来的命令。核心代码如下



```python
## 判断是否为root用户，如果为root用户就发送警告
check_root_user() 
## 解析命令行参数
odoo.tools.config.parse_config(args)
## 如果为postgres用户就停止运行
check_postgres_user()
report_configuration()

config = odoo.tools.config

# the default limit for CSV fields in the module is 128KiB, which is not
# quite sufficient to import images to store in attachment. 500MiB is a
# bit overkill, but better safe than sorry I guess
csv.field_size_limit(500 * 1024 * 1024)

## 创建加载的数据库
preload = []
if config['db_name']:
    preload = config['db_name'].split(',')
    for db_name in preload:
        try:
            odoo.service.db._create_empty_database(db_name)
            config['init']['base'] = True
        except ProgrammingError as err:
            if err.pgcode == errorcodes.INSUFFICIENT_PRIVILEGE:
                # We use an INFO loglevel on purpose in order to avoid
                # reporting unnecessary warnings on build environment
                # using restricted database access.
                _logger.info("Could not determine if database %s exists, "
                             "skipping auto-creation: %s", db_name, err)
            else:
                raise err
        except odoo.service.db.DatabaseExists:
            pass

if config["translate_out"]:
    export_translation()
    sys.exit(0)

if config["translate_in"]:
    import_translation()
    sys.exit(0)

# This needs to be done now to ensure the use of the multiprocessing
# signaling mecanism for registries loaded with -d
if config['workers']:
    odoo.multi_process = True

## 是否在启动服务后停止，用户创建更新数据库
stop = config["stop_after_init"]

## 设置pid文件
setup_pid_file()
## 启动server
rc = odoo.service.server.start(preload=preload, stop=stop)
sys.exit(rc)
```

参考

> <https://blog.csdn.net/weixin_35737303/article/details/79038671>

