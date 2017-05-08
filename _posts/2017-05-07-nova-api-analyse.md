---
layout:     post
title:      深入理解nova api服务
date:       2017-05-07
author:     xue
catalog:    true
tags:
    - openstack
    - nova
---

## wsgi简介

在构建 Web 应用时，通常会有 Web Server 和 Application Server 两种角色。其中 Web Server 主要负责接受来自用户的请求，解析 HTTP 协议，并将请求转发给 Application Server，Application Server 主要负责处理用户的请求，并将处理的结果返回给 Web Server，最终 Web Server 将结果返回给用户。

由于有很多动态语言和很多种 Web Server，他们彼此之间互不兼容，给程序员造成了很大的麻烦。

WSGI是一种 web server or gateway 和 python web application or framework 之间简单通用的接口，符合这种接口的 application 可运行在所有符合该接口的 server 上。通俗的讲，WSGI 规范了一种简单的接口，解耦了 server 和 application，使得双边的开发者更加专注自身特性的开发。

WSGI 标准中主要定义了两种角色：

* “server” 或 “gateway” 端
* “application” 或 “framework” 端

**“application” 或 “framework” 端**

Application/framework 端必须定义一个 callable object，callable object 可以是以下三者之一：

* function, method
* class
* instance with a __call__ method

Callable object 必须满足以下两个条件：

* 接受两个参数：字典(environ)，回调函数(start_response，返回 HTTP status，headers 给 web server)
* 返回一个可迭代的值

一个简单的callable object 如下所示：

```
def simple_app(environ, start_response):
    """Simplest possible application object"""
    status = '200 OK'
    response_headers = [('Content-type', 'text/plain')]
    start_response(status, response_headers)
    return ['Hello World']
```

**“server” 或 “gateway” 端**

Server/gateway 端主要专注 HTTP 层面的业务，重点是接收 HTTP 请求和提供并发。每当收到 HTTP 请求，server/gateway 必须调用 callable object：

* 接收 HTTP 请求，但是不关心 HTTP url, HTTP method 等
* 为 environ 提供必要的参数，实现一个回调函数 
*  start_response，并传给 callable object
调用 callable object

一个简单的例子如下所示：

```
# server/gateway side
if __name__ == '__main__':
    from wsgiref.simple_server import make_server
    server = make_server('0.0.0.0', 8080, application)
    server.serve_forever()
```

## Paste Deployment简介

paste deployment是一个配置WSGI APP和发现服务的一个系统，对于WSGI APP的开发者，它提供了一个简单的函数(loadapp)，以便从配置文件和python egg中加载一个wsgi app，对于提供的WSGI APP，它只请求一个简单的入口，因此你不需要提供你的app详情给它。

通俗来讲：可以将它理解成是一种机制或者流程模式，在一个Server中，可以通过他将server中的app通过它进行连接组合。其中的连接方式他通过一个简单的接口提供给用户，用户只需要配置好paste deployment即可完成app的连接组合，这对用户完全是透明的。

与paste deployment的主要交互方式是配置文件，让整体server中app的流程处理按照配置文件中的流向去运行。

配置文件中由多个sections(段)组成，每个sections是由[type:name]这样的格式组成，如果不是这样的格式，将会被忽略

## 解读nova中api-paste.ini

/etc/nova/api-paste.ini的部分内容如下： 

```

#############
# OpenStack #
#############

[composite:osapi_compute]
use = call:nova.api.openstack.urlmap:urlmap_factory
/: oscomputeversions
/v1.1: openstack_compute_api_v2
/v2: openstack_compute_api_v2
/v2.1: openstack_compute_api_v21
/v3: openstack_compute_api_v3
这里利用composite将xxx/xxx形式的请求交给oscomputeversions，形似xxxx/v1.1/xxxxx请求交给openstack_compute_api_v2，来实现API版本控制


[composite:openstack_compute_api_v2]
use = call:nova.api.auth:pipeline_factory
noauth = compute_req_id faultwrap sizelimit noauth ratelimit osapi_compute_app_v2
noauth2 = compute_req_id faultwrap sizelimit noauth2 ratelimit osapi_compute_app_v2
keystone = compute_req_id faultwrap sizelimit authtoken keystonecontext ratelimit osapi_compute_app_v2
keystone_nolimit = compute_req_id faultwrap sizelimit authtoken keystonecontext osapi_compute_app_v2
针对openstack_compute_api_v2的实现来看，首先调用了nova.api.auth中pipeline_factory方法,从源代码可以看出，实际上_load_pipeline调用了keystone，
keystone属于是一个filter的集合，请求会依次通过前面的这些filter，最后到达osapi_compute_app_v2这个app

[composite:openstack_compute_api_v21]
use = call:nova.api.auth:pipeline_factory_v21
noauth = compute_req_id faultwrap sizelimit noauth osapi_compute_app_v21
noauth2 = compute_req_id faultwrap sizelimit noauth2 osapi_compute_app_v21
keystone = compute_req_id faultwrap sizelimit authtoken keystonecontext osapi_compute_app_v21

[composite:openstack_compute_api_v3]
use = call:nova.api.auth:pipeline_factory_v21
noauth = request_id faultwrap sizelimit noauth_v3 osapi_compute_app_v3
noauth2 = request_id faultwrap sizelimit noauth_v3 osapi_compute_app_v3
keystone = request_id faultwrap sizelimit authtoken keystonecontext osapi_compute_app_v3

[filter:request_id]
paste.filter_factory = oslo.middleware:RequestId.factory

[filter:compute_req_id]
paste.filter_factory = nova.api.compute_req_id:ComputeReqIdMiddleware.factory

[filter:faultwrap]
paste.filter_factory = nova.api.openstack:FaultWrapper.factory

[filter:noauth]
paste.filter_factory = nova.api.openstack.auth:NoAuthMiddlewareOld.factory

[filter:noauth2]
paste.filter_factory = nova.api.openstack.auth:NoAuthMiddleware.factory

[filter:noauth_v3]
paste.filter_factory = nova.api.openstack.auth:NoAuthMiddlewareV3.factory

[filter:ratelimit]
paste.filter_factory = nova.api.openstack.compute.limits:RateLimitingMiddleware.factory

[filter:sizelimit]
paste.filter_factory = oslo.middleware:RequestBodySizeLimiter.factory

[app:osapi_compute_app_v2]
paste.app_factory = nova.api.openstack.compute:APIRouter.factory
该app直接调用了nova.api.openstack.compute中的APIRouter类中的factory函数。

[app:osapi_compute_app_v21]
paste.app_factory = nova.api.openstack.compute:APIRouterV21.factory

[app:osapi_compute_app_v3]
paste.app_factory = nova.api.openstack.compute:APIRouterV3.factory

[pipeline:oscomputeversions]
pipeline = faultwrap oscomputeversionapp

[app:oscomputeversionapp]
paste.app_factory = nova.api.openstack.compute.versions:Versions.factory

##########
# Shared #
##########

[filter:keystonecontext]
paste.filter_factory = nova.api.auth:NovaKeystoneContext.factory

[filter:authtoken]
paste.filter_factory = keystonemiddleware.auth_token:filter_factory
```

从上面来看在OpenStack中nova-api的请求流程:依次经过filter[compute_req_id faultwrap sizelimit authtoken keystonecontext ratelimit]，最后到达osapi_compute_app_v2这个app。

## Nova API服务的启动

### WSGI Server
Nova-api(nova/cmd/api.py) 服务启动时，初始化 nova/wsgi.py 中的类 Server，建立了 socket 监听 IP 和端口，再由 eventlet.spawn 和 eventlet.wsgi.server 创建 WSGI server：

```
class Server(object):
    def __init__(self, name, app, host='0.0.0.0', port=0, pool_size=None,
        ...
        eventlet.wsgi.MAX_HEADER_LINE = CONF.max_header_line
        self.name = name
        self.app = app
        self._server = None
        bind_addr = (host, port)
        ...
        # 建立 socket，监听 IP 和端口
        self._socket = eventlet.listen(bind_addr, family, backlog=backlog)
    def start(self):
        ...
        # 构建所需参数
        wsgi_kwargs = {
            'func': eventlet.wsgi.server,
            'sock': dup_socket,
            'site': self.app,
            'protocol': self._protocol,
            'custom_pool': self._pool,
            'log': self._wsgi_logger,
            'log_format': CONF.wsgi_log_format,
            'debug': False,
            'keepalive': CONF.wsgi_keep_alive,
            'socket_timeout': self.client_socket_timeout
            }

        if self._max_url_len:
            wsgi_kwargs['url_length_limit'] = self._max_url_len
        # 由 eventlet.sawn 启动 server
        self._server = eventlet.spawn(**wsgi_kwargs)
        
```

### Application Side & Middlewar
nova/wsgi.py的__init__方法中加载了Loader类，Application 的加载就是由它完成的，Loader 的 load_app 方法调用了 paste.deploy.loadapp 加载了 WSGI 的配置文件 /etc/nova/api-paste.ini：

```
class Loader(object):
    """Used to load WSGI applications from paste configurations."""

    def __init__(self, config_path=None):

        # 获取 WSGI 配置文件的路径
        self.config_path = config_path or CONF.api_paste_config

    def load_app(self, name):

        # paste.deploy 读取配置文件并加载该配置
        return paste.deploy.loadapp("config:%s" % self.config_path, name=name)
```

前面的api-paste.ini提到过，nova-api请求会到osapi_compute_app_v2这个app,根据api-paste.ini里面的定义：

```
[app:osapi_compute_app_v2]
paste.app_factory = nova.api.openstack.compute:APIRouter.factory
```

nova/api/openstack/compute/__init__.py下APIRouter类中的factory方法继承自nova.api.openstack.APIRouter，factory方法创建了一个APIRouter对象，在APIRouter的__init__方法中调用了_setup_routes方法。其方法定义如下：

```
#nova/api/openstack/compute/__init__.py
class APIRouter(nova.api.openstack.APIRouter):
    def _setup_routes(self, mapper, ext_mgr, init_only):
        ...
        if init_only is None or 'consoles' in init_only:
            self.resources['consoles'] = consoles.create_resource()
            mapper.resource("console", "consoles",
                        controller=self.resources['consoles'],
                        parent_resource=dict(member_name='server',
                        collection_name='servers'))

        if init_only is None or 'consoles' in init_only or \
                'servers' in init_only or 'ips' in init_only:
            #ext_mgr是一个ExtensionManager对象，定义在nova/api/openstack/compute/extensions.py
            self.resources['servers'] = servers.create_resource(ext_mgr)
            mapper.resource("server", "servers",
                            controller=self.resources['servers'],
                            collection={'detail': 'GET'},
                            member={'action': 'POST'})
        ...

```

在_setup_routes方法中定义了许多nova资源的url映射，以上代码中给出了console,server资源的url映射。可以看到，对于每种资源，都是先调用资源的create_resource方法。返回一个resource对象。然后再调用mapper.resource添加资源的url.

以server资源为例：

```
mapper.resource("server", "servers",
                            controller=self.resources['servers'],
                            collection={'detail': 'GET'},
                            member={'action': 'POST'})
```

为了让代码更加简洁，mapper.resource将一些最基本的url映射封装了起来，即它已经封装了一些PUT,GET,DELETE之类的常见方法映射。  
mapper.resource的参数含义为：server是成员名，servers是集合名，collection参数指定额外的集合操作，member指定额外的成员操作。  
上面的代码的含义是，在最基本的url映射方法的基础上，额外添加了如下url映射：

```
map.connect("servers","/servers/detail",controller=self.resources['servers'],action="detail",conditions=dict(method=["GET"]))
map.connect("server","/server/{id}/action",controller=self.resources['servers'],action="action",conditions=dict(method=["POST"]))
```

## 处理HTTP请求的流程


```
#nova/api/openstack/compute/__init__.py
class APIRouter(nova.api.openstack.APIRouter):
    def _setup_routes(self, mapper, ext_mgr, init_only):
        ...
        if init_only is None or 'consoles' in init_only:
            self.resources['consoles'] = consoles.create_resource()
            mapper.resource("console", "consoles",
                        controller=self.resources['consoles'],
                        parent_resource=dict(member_name='server',
                        collection_name='servers'))


#nova/api/openstack/compute/consoles.py
def create_resource():
    return wsgi.Resource(Controller())
```
从上面的代码可以看到，console资源的controller对象是一个wsgi.Resource对象。因此，当客户端发送HTTP请求后，

APIRouter类继承了nova/wsgi.py中的Router类，__call__方法



## 参考

[Paste Deployment简介及nova-api-paste.ini解析](http://ju.outofmemory.cn/entry/178432)  
[JiYou](https://github.com/JiYou/openstack)




