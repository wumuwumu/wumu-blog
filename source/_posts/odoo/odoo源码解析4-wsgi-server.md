---
title: odoo源码解析4-wsgi_server
tags:
  - odoo
abbrlink: fa23f810
date: 2019-11-16 15:35:05
---

# application

```python
def application(environ, start_response):
    ## 是否启动代理
    # FIXME: is checking for the presence of HTTP_X_FORWARDED_HOST really useful?
    #        we're ignoring the user configuration, and that means we won't
    #        support the standardised Forwarded header once werkzeug supports
    #        it
    if config['proxy_mode'] and 'HTTP_X_FORWARDED_HOST' in environ:
        return ProxyFix(application_unproxied)(environ, start_response)
    else:
        return application_unproxied(environ, start_response)
```

# application_unproxied

清除数据库和用户的追踪
清除动作在application方法的结尾不能完成，因为werkzeu在后面还会生成有关的日志。

```python
def application_unproxied(environ, start_response):
    """ WSGI entry point."""
    # cleanup db/uid trackers - they're set at HTTP dispatch in
    # web.session.OpenERPSession.send() and at RPC dispatch in
    # odoo.service.web_services.objects_proxy.dispatch().
    # /!\ The cleanup cannot be done at the end of this `application`
    # method because werkzeug still produces relevant logging afterwards
    if hasattr(threading.current_thread(), 'uid'):
        del threading.current_thread().uid
    if hasattr(threading.current_thread(), 'dbname'):
        del threading.current_thread().dbname
    if hasattr(threading.current_thread(), 'url'):
        del threading.current_thread().url

    with odoo.api.Environment.manage():
        result = odoo.http.root(environ, start_response)
        if result is not None:
            return result

    # We never returned from the loop.
    return werkzeug.exceptions.NotFound("No handler found.\n")(environ, start_response)
```

# 参考

> https://blog.csdn.net/weixin_35737303/article/details/79038982