---
title: odoo源码解析-启动web服务器
tags:
  - odoo
abbrlink: 25ec7414
date: 2019-11-16 14:41:42
---



# 启动

```python

def start(preload=None, stop=False):
    """ Start the odoo http server and cron processor.
    """
    global server
	## 这里加载两个模块web和web_kan，在这里加载模块才能够在用户没有登录的时候才能够访问路由
    load_server_wide_modules()
    odoo.service.wsgi_server._patch_xmlrpc_marshaller()
	
    """
    ·GeventServer
    ·PreforkServer
    ·ThreadedServer(默认)
    CommonServer是后面三个类的父类
	Odoo服务器通过ThreadedServer.run()运行
    """
    if odoo.evented:
        server = GeventServer(odoo.service.wsgi_server.application)
    elif config['workers']:
        if config['test_enable'] or config['test_file']:
            _logger.warning("Unit testing in workers mode could fail; use --workers 0.")

        server = PreforkServer(odoo.service.wsgi_server.application)

        # Workaround for Python issue24291, fixed in 3.6 (see Python issue26721)
        if sys.version_info[:2] == (3,5):
            # turn on buffering also for wfile, to avoid partial writes (Default buffer = 8k)
            werkzeug.serving.WSGIRequestHandler.wbufsize = -1
    else:
        server = ThreadedServer(odoo.service.wsgi_server.application)

    watcher = None
    if 'reload' in config['dev_mode'] and not odoo.evented:
        if inotify:
            watcher = FSWatcherInotify()
            watcher.start()
        elif watchdog:
            watcher = FSWatcherWatchdog()
            watcher.start()
        else:
            if os.name == 'posix' and platform.system() != 'Darwin':
                module = 'inotify'
            else:
                module = 'watchdog'
            _logger.warning("'%s' module not installed. Code autoreload feature is disabled", module)
    if 'werkzeug' in config['dev_mode']:
        server.app = DebuggedApplication(server.app, evalex=True)

    ##  启动web服务器
    rc = server.run(preload, stop)

    if watcher:
        watcher.stop()
    # like the legend of the phoenix, all ends with beginnings
    if getattr(odoo, 'phoenix', False):
        _reexec()

    return rc if rc else 0
```

# ThreadedServer(CommandServer)

## Run

```python
""" Start the http server and the cron thread then wait for a signal.

        The first SIGINT or SIGTERM signal will initiate a graceful shutdown while
        a second one if any will force an immediate exit.
        """
## 启动一个系统命令监测。。。
self.start(stop=stop)

## 安装、更新、加载模块
rc = preload_registries(preload)

if stop:
    self.stop()
    return rc

## 加载定时任务
self.cron_spawn()

# Wait for a first signal to be handled. (time.sleep will be interrupted
# by the signal handler)
try:
    while self.quit_signals_received == 0:
        self.process_limit()
        if self.limit_reached_time:
            has_other_valid_requests = any(
                not t.daemon and
                t not in self.limits_reached_threads
                for t in threading.enumerate()
                if getattr(t, 'type', None) == 'http')
            if (not has_other_valid_requests or
                (time.time() - self.limit_reached_time) > SLEEP_INTERVAL):
                # We wait there is no processing requests
                # other than the ones exceeding the limits, up to 1 min,
                # before asking for a reload.
                _logger.info('Dumping stacktrace of limit exceeding threads before reloading')
                dumpstacks(thread_idents=[thread.ident for thread in self.limits_reached_threads])
                self.reload()
                # `reload` increments `self.quit_signals_received`
                # and the loop will end after this iteration,
                # therefore leading to the server stop.
                # `reload` also sets the `phoenix` flag
                # to tell the server to restart the server after shutting down.
                else:
                    time.sleep(1)
                    else:
                        time.sleep(SLEEP_INTERVAL)
                        except KeyboardInterrupt:
                            pass

                        self.stop()
```

## start

```python
def start(self, stop=False):
    _logger.debug("Setting signal handlers")
    set_limit_memory_hard()
    if os.name == 'posix':
        signal.signal(signal.SIGINT, self.signal_handler)
        signal.signal(signal.SIGTERM, self.signal_handler)
        signal.signal(signal.SIGCHLD, self.signal_handler)
        signal.signal(signal.SIGHUP, self.signal_handler)
        signal.signal(signal.SIGXCPU, self.signal_handler)
        signal.signal(signal.SIGQUIT, dumpstacks)
        signal.signal(signal.SIGUSR1, log_ormcache_stats)
        elif os.name == 'nt':
            import win32api
            win32api.SetConsoleCtrlHandler(lambda sig: self.signal_handler(sig, None), 1)

            test_mode = config['test_enable'] or config['test_file']
            if test_mode or (config['http_enable'] and not stop):
                # some tests need the http deamon to be available...
                self.http_spawn()
```



# ThreadedWSGIServerReloadable

这个服务可以不启动也能够运行程序。他的作用是debug保持端口是开启的。

Werkzeug是Python的WSGI规范的实现函数库。基于BSD协议。
WSGI(Web Server Gateway Interface)
WSGI服务允许重用环境提供的监听套接字，它通过自动重加载使用，用于保持当有重加载的时候监听套接字是打开状态

```python

class ThreadedWSGIServerReloadable(LoggingBaseWSGIServerMixIn, werkzeug.serving.ThreadedWSGIServer):
    """ werkzeug Threaded WSGI Server patched to allow reusing a listen socket
    given by the environement, this is used by autoreload to keep the listen
    socket open when a reload happens.
    """
    def __init__(self, host, port, app):
        super(ThreadedWSGIServerReloadable, self).__init__(host, port, app,
                                                           handler=RequestHandler)

        # See https://github.com/pallets/werkzeug/pull/770
        # This allow the request threads to not be set as daemon
        # so the server waits for them when shutting down gracefully.
        self.daemon_threads = False

    def server_bind(self):
        SD_LISTEN_FDS_START = 3
        if os.environ.get('LISTEN_FDS') == '1' and os.environ.get('LISTEN_PID') == str(os.getpid()):
            self.reload_socket = True
            self.socket = socket.fromfd(SD_LISTEN_FDS_START, socket.AF_INET, socket.SOCK_STREAM)
            _logger.info('HTTP service (werkzeug) running through socket activation')
        else:
            self.reload_socket = False
            super(ThreadedWSGIServerReloadable, self).server_bind()
            _logger.info('HTTP service (werkzeug) running on %s:%s', self.server_name, self.server_port)

    def server_activate(self):
        if not self.reload_socket:
            super(ThreadedWSGIServerReloadable, self).server_activate()

    def process_request(self, request, client_address):
        """
        Start a new thread to process the request.
        Override the default method of class socketserver.ThreadingMixIn
        to be able to get the thread object which is instantiated
        and set its start time as an attribute
        """
        t = threading.Thread(target = self.process_request_thread,
                             args = (request, client_address))
        t.daemon = self.daemon_threads
        t.type = 'http'
        t.start_time = time.time()
        t.start()

    # TODO: Remove this method as soon as either of the revision
    # - python/cpython@8b1f52b5a93403acd7d112cd1c1bc716b31a418a for Python 3.6,
    # - python/cpython@908082451382b8b3ba09ebba638db660edbf5d8e for Python 3.7,
    # is included in all Python 3 releases installed on all operating systems supported by Odoo.
    # These revisions are included in Python from releases 3.6.8 and Python 3.7.2 respectively.
    def _handle_request_noblock(self):
        """
        In the python module `socketserver` `process_request` loop,
        the __shutdown_request flag is not checked between select and accept.
        Thus when we set it to `True` thanks to the call `httpd.shutdown`,
        a last request is accepted before exiting the loop.
        We override this function to add an additional check before the accept().
        """
        if self._BaseServer__shutdown_request:
            return
        super(ThreadedWSGIServerReloadable, self)._handle_request_noblock()


```





# 参考

> <https://blog.csdn.net/weixin_35737303/article/details/79038879>