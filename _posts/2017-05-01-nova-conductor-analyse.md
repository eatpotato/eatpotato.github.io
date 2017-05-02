---
layout:     post
title:      nova conductor服务
date:       2017-05-01
author:     xue
catalog:    true
tags:
    - openstack
---


以nova kilo版本为例
## nova主要组件及功能

|组件|功能|
|--|--|
|nova-api|负责接收和响应用户请求|
|nova-conductor|负责数据库的访问权限控制，避免nova-compute直接访问数据库|
|nova-scheduler|负责选择虚拟机的宿主机|
|nova-cert|用于管理证书|
|nova-novncproxy|提供vnc访问功能|
|nova-compute|管理虚拟机的整个生命周期（从创建到删除），并实现虚拟机迁移、快照等功能|
|nova-consoleauth|为vpn代理服务器提供token验证功能|


按功能划分其主要组件有：  
（1） 虚拟机管理： nova-api、nova-compute、nova-scheduler  
（2） 虚拟机VNC及日志管理： nova-console、nova-consoleauth  
（3） 数据库管理： nova-conductor  
（4） 安全管理： nova-consoleauth、nova-cert


## nova rpc 服务

nova的各个服务之间的通信都使用了基于AMQP实现的RPC机制，其中nova-compute、nova-conductor和nova-schduler在启动时都会注册一个RPC Server，而nova-api因为nova内部并没有服务会调用它提供的接口，所以无需注册。以nova-compute为例：

```
#nova/compute/rpcapi.py

class ComputeAPI(object):

    #定义了非常多的方法，每个方法里都有下面两行     cctxt = self.client.prepare(server=host, version=version)       #获得目标主机的RPC client
    #rpc cast是异步调用，第二个参数是RPC调用的函数名，后面的参数根据方法不同而不同     cctxt.cast(ctxt, 'live_migration', instance=instance,                            dest=dest, block_migration=block_migration,                migrate_data=migrate_data, **args)

```

nova/compute/rpcapi.py里面只是定义了服务的RPC调用接口，真正完成任务的是nova/compute/manager.py里面的方法

## nova conductor

nova/conductor/api.py定义了四个controller：LocalAPI、API、LocalComputeTaskAPI、ComputeTaskAPI


nova服务的conductor_api对象的类型可能是LocalAPI类或API类。API方法检查nova.conf配置文件中的use_local配置项，如果为True,则创建LocalAPI对象;否则，创建API对象。



nova/conductor/__init__.py定义如下：

```
def API(*args, **kwargs):
    use_local = kwargs.pop('use_local', False)
    if oslo_config.cfg.CONF.conductor.use_local or use_local:
        api = conductor_api.LocalAPI
    else:
        api = conductor_api.API
    return api(*args, **kwargs)


def ComputeTaskAPI(*args, **kwargs):
    use_local = kwargs.pop('use_local', False)
    if oslo_config.cfg.CONF.conductor.use_local or use_local:
        api = conductor_api.LocalComputeTaskAPI
    else:
        api = conductor_api.ComputeTaskAPI
    return api(*args, **kwargs)
```

所以可以看到conductor的API和ComputeTaskAPI是根据/etc/nova/nova.conf配置文件中的conductor字段中的use_local参数值或者从上层函数传递下来的use_local值进行设置的。有Local和没有Local的主要区别是：Local不会发起RPC请求。

**LocalAPI**


```
class LocalAPI(object):     """A local version of the conductor API that does database updates     locally instead of via RPC.     """      def __init__(self):         # 定义了一个ConductorManager对象，它的定义在nova/conductor/manager.py中，
        其中定义了许多数据库访问的方法，这些方法都在本机建立与数据库的连接。
        # LocalAPI类定义了许多接口方法供Nova其他服务（如Scheduler服务）调用，
        而这些方法底层都是直接调用了ConductorManager对象中定义的相应方法,并没有向nova conductor发送RPC请求         self._manager = utils.ExceptionHelper(manager.ConductorManager())
```


**API**

```
class API(object):
    def __init__(self):
    #API对象的初始化方法也很简单，它创建了一个Conductor RPC API对象。
    Conductor RPC API类中定义了许多接口方法向Nova conductor发送RPC请求。     self._manager = rpcapi.ConductorAPI()

```