---
layout:     post
title:      cinder服务启动源码分析
date:       2017-05-07
author:     xue
catalog:    true
tags:
    - openstack
    - cinder
---

本文以Newton版本的cinder为例
## 一、cinder api服务

根据setup.cfg找到cinder-api服务的启动入口

```
#setup.cfg
[entry_points]
console_scripts =
    cinder-api = cinder.cmd.api:main
```


```
#cinder/cmd/api.py
def main():
    # 加载辅助对象，封装与数据库相关的操作
    objects.register_all()
    gmr_opts.set_defaults(CONF)
    # 加载配置并设置日志
    CONF(sys.argv[1:], project='cinder',
         version=version.version_string())
    config.set_middleware_defaults()
    logging.setup(CONF, "cinder")
    python_logging.captureWarnings(True)
    utils.monkey_patch()

    gmr.TextGuruMeditation.setup_autorun(version, conf=CONF)
    # 初始化rpc
    rpc.init(CONF)
    launcher = service.process_launcher()
    
    # 创建一个WSGIService对象
    server = service.WSGIService('osapi_volume')
    launcher.launch_service(server, workers=server.workers)
    launcher.wait()
```

WSGIService关键代码如下：

```
#cinder/service.py
class WSGIService(service.ServiceBase):
    def _get_manager(self):
        
        fl = '%s_manager' % self.name
        if fl not in CONF:
            return None

        manager_class_name = CONF.get(fl, None)
        if not manager_class_name:
            return None

        manager_class = importutils.import_class(manager_class_name)
        return manager_class()
	def __init__(self, name, loader=None):
       
        ...
        self.name = name
        # 会加载名为`osapi_volume_manager`的管理器（或None）
        self.manager = self._get_manager()
        """创建WSGI应用加载器
    	并根据配置文件(`cinder.conf`)设置应用配置路径:
    	`config_path` = `/etc/cinder/paste-api.ini`
    	"""
        self.loader = loader or wsgi.Loader(CONF)
        """ 加载‘/etc/cinder/paste-api.ini’文件中‘[composite:osapi_volume]’中定义的app,
        具体的代码看下面
        self.app = self.loader.load_app(name)
        """根据配置文件(`cinder.conf`)设置监听地址及工作线程数
        如果未指定监听ip及端口就分别设置为`0.0.0.0`及`0`
        如果为指定工作线程数就设置为cpu个数
        如果设置的工作线程数小于1，则抛异常
        """
        self.host = getattr(CONF, '%s_listen' % name, "0.0.0.0")
        self.port = getattr(CONF, '%s_listen_port' % name, 0)
        self.use_ssl = getattr(CONF, '%s_use_ssl' % name, False)
        self.workers = (getattr(CONF, '%s_workers' % name, None) or
                        processutils.get_worker_count())
        if self.workers and self.workers < 1:
            worker_name = '%s_workers' % name
            msg = (_("%(worker_name)s value of %(workers)d is invalid, "
                     "must be greater than 0.") %
                   {'worker_name': worker_name,
                    'workers': self.workers})
            raise exception.InvalidInput(msg)
            
        """如果CONF.profiler.profiler_enabled = True就开启性能分析  
        创建一个类型为`Messaging`的通知器(`_notifier`)，将性能数据发送给
        ceilometer
        """
        setup_profiler(name, self.host)
          
        # 创建WSGI Server对象
        self.server = wsgi.Server(CONF,
                                  name,
                                  self.app,
                                  host=self.host,
                                  port=self.port,
                                  use_ssl=self.use_ssl)

```

load_app的代码如下所示：

```
#oslo.service/wsgi.py
class Loader(object):
    """Used to load WSGI applications from paste configurations."""

    def __init__(self, conf):
        """Initialize the loader, and attempt to find the config.
        :param conf: Application config
        :returns: None
        """
        conf.register_opts(_options.wsgi_opts)
        self.config_path = None

        #根据配置文件获得config_path的路径
        config_path = conf.api_paste_config
        if not os.path.isabs(config_path):
            self.config_path = conf.find_file(config_path)
        elif os.path.exists(config_path):
            self.config_path = config_path

        if not self.config_path:
            raise ConfigNotFound(path=config_path)

    def load_app(self, name):
        """Return the paste URLMap wrapped WSGI application.
        :param name: Name of the application to load.
        :returns: Paste URLMap object wrapping the requested application.
        :raises: PasteAppNotFound
        """
        try:
            LOG.debug("Loading app %(name)s from %(path)s",
                      {'name': name, 'path': self.config_path})
            return deploy.loadapp("config:%s" % self.config_path, name=name)
        except LookupError:
            LOG.exception("Couldn't lookup app: %s", name)
            raise PasteAppNotFound(name=name, path=self.config_path)
```

load_app完成了如下操作:
1.根据/etc/cinder/api-paste.ini中的Paste规则加载相应的过滤器和app
2.设置路由映射(Router)

这些具体的过程可以参考我的另一篇博客： [深入理解nova api服务](http://xuefy.cn/2017/05/07/nova-api-analyse/)

上面的内容是创建创建一个WSGIService对象并初始化的过程，接着往下分析：

```
#cinder/cmd/api.py
def main():
    ...
    #通过进程启动器 - 在新的进程中启动服务程序，并等待服务启动结束
    #最终会调用`WSGIService.start`方法启动服务
    launcher.launch_service(server, workers=server.workers)
    
#cinder/service.py
from oslo_service import wsgi

class WSGIService(service.ServiceBase):
    ...
    def start(self):
        ...
        if self.manager:
            self.manager.init_host()
        """这里的server就是上面初始化时创建的wsgi Server对象
        self.server = wsgi.Server(CONF,
                                  name,
                                  self.app,
                                  host=self.host,
                                  port=self.port,
                                  use_ssl=self.use_ssl)
        """
        self.server.start()
        self.port = self.server.port
        

#接上文的start方法
#oslo.service/wsgi.py
class Server(service.ServiceBase):
    def start(self):
        ...

        self.dup_socket = self.socket.dup()

        if self._use_ssl:
            self.dup_socket = sslutils.wrap(self.conf, self.dup_socket)

        wsgi_kwargs = {
            'func': eventlet.wsgi.server,
            'sock': self.dup_socket,
            'site': self.app,
            'protocol': self._protocol,
            'custom_pool': self._pool,
            'log': self._logger,
            'log_format': self.conf.wsgi_log_format,
            'debug': False,
            'keepalive': self.conf.wsgi_keep_alive,
            'socket_timeout': self.client_socket_timeout
            }

        if self._max_url_len:
            wsgi_kwargs['url_length_limit'] = self._max_url_len

        self._server = eventlet.spawn(**wsgi_kwargs)


```

这样，cinder-api中的socket监听就准备就绪了：基于配置cinder-api会启动多个监听线程，每个客户端连接用来一个线程来处理.


## 二、cinder scheduler服务

同理，找到启动入口：

```
#cinder/cmd/scheduler.py
def main():
    objects.register_all()
    gmr_opts.set_defaults(CONF)
    CONF(sys.argv[1:], project='cinder',
         version=version.version_string())
    logging.setup(CONF, "cinder")
    python_logging.captureWarnings(True)
    utils.monkey_patch()
    gmr.TextGuruMeditation.setup_autorun(version, conf=CONF)
    server = service.Service.create(binary='cinder-scheduler')
    service.serve(server)
    service.wait()
```

service.Service.create部分代码如下：

```
class Service(service.Service):
	@classmethod
    def create(cls, host=None, binary=None, topic=None, manager=None,
               report_interval=None, periodic_interval=None,
               periodic_fuzzy_delay=None, service_name=None,
               coordination=False, cluster=None):
        ...
        if not host:
            host = CONF.host
        if not binary:
            binary = os.path.basename(inspect.stack()[-1][1])
        if not topic:
            topic = binary
        if not manager:
            subtopic = topic.rpartition('cinder-')[2]
            manager = CONF.get('%s_manager' % subtopic, None)
        #实例化Service类,会执行Service类的__init__方法
        service_obj = cls(host, binary, topic, manager,
                          report_interval=report_interval,
                          periodic_interval=periodic_interval,
                          periodic_fuzzy_delay=periodic_fuzzy_delay,
                          service_name=service_name,
                          coordination=coordination,
                          cluster=cluster)

        return service_obj
```
接着是service.serve(server)的部分代码：

```
#cinder/cmd/scheduler.py
service.serve(server)


#cinder/service.py
def serve(server, workers=None):
    global _launcher
    if _launcher:
        raise RuntimeError(_('serve() can only be called once'))

    _launcher = service.launch(CONF, server, workers=workers)
    
#最终会调用Service类的start方法,部分代码如下
class Service(service.Service):

    def start(self):
    
        ctxt = context.get_admin_context()
        endpoints = [self.manager]
        endpoints.extend(self.manager.additional_endpoints)
        obj_version_cap = objects.Service.get_minimum_obj_version(ctxt)
        

        target = messaging.Target(topic=self.topic, server=self.host)
        self.rpcserver = rpc.get_server(target, endpoints, serializer)
        #创建RPC连接，启动消费者线程,等待队列消息
        self.rpcserver.start()

```


