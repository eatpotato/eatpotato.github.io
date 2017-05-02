---
layout:     post
title:      nova定制调度算法
date:       2017-05-01
author:     xue
catalog:    true
tags:
    - openstack
---


以nova kilo版本为例
## nova调度过程分析

之前创建虚拟机流程分析提到过：

```

#nova/scheduler/manager.py
class SchedulerManager(manager.Manager):
        @messaging.expected_exceptions(exception.NoValidHost)
    def select_destinations(self, context, request_spec, filter_properties):
        '''driver其实就是调度算法实现，由配置文件决定，通常用的比较多的就是filter_scheduler，
        对应filter_scheduler.py模块,该模块首先通过host_manager拿到所有的计算节点信息，
        然后通过filters过滤掉不满足条件的计算节点，剩下的节点通过weigh方法计算权值，
        最后选择权值高的作为候选计算节点返回。nova-scheduler进程结束。'''
        dests = self.driver.select_destinations(context, request_spec,
            filter_properties)
        return jsonutils.to_primitive(dests)
```

driver在SchedulerManager 的__init__方法中定义

```
class SchedulerManager(manager.Manager):
    def __init__(self, scheduler_driver=None, *args, **kwargs):
        if not scheduler_driver:
            scheduler_driver = CONF.scheduler_driver
        #driver 是通过配置文件中的选项值指定的类来返回的对象
        self.driver = importutils.import_object(scheduler_driver)

```


在nova的配置文件中：

```
scheduler_driver=nova.scheduler.filter_scheduler.FilterScheduler
```

继续往下分析：


```
#nova/scheduler/filter_scheduler.py
class FilterScheduler(driver.Scheduler):
    def select_destinations(self, context, request_spec, filter_properties):
        ...
        # 需要创建的 Instances 的数量
        num_instances = request_spec['num_instances']
        # 获取满足笫一次过滤条件的主机列表 List
        selected_hosts = self._schedule(context, request_spec,
                                        filter_properties)
        # 当请求的 Instance 数量大于合适的主机数量时，不会创建 Instance 且输出错误信息
        if len(selected_hosts) < num_instances and\
            len(selected_hosts) >= min_instances:
            ...
        elif len(selected_hosts) < min_instances:
            ...
        dests = [dict(host=host.obj.host, nodename=host.obj.nodename,
                      limits=host.obj.limits) for host in selected_hosts]
        ...
        return dests 
        
        
    def _schedule(self, context, request_spec, filter_properties):  
        ...
        #获取所有Hosts 状态,主要用来去除不活跃的节点
        hosts = self._get_all_host_states(elevated)
        #调用HostManager的get_filtered_hosts方法来选择可用的计算节点
        hosts = self.host_manager.get_filtered_hosts(hosts,
                        filter_properties, index=num, context=context)
        ...
        #通过 Weighed 选取最优 Host
        weighed_hosts = self.host_manager.get_weighed_hosts(hosts,
                    filter_properties, context=context)
        ...
        return selected_hosts
```

再来看看get_filtered_hosts的调用过程：

```
#nova/scheduler/host_manager.py
class HostManager(object):
    def get_filtered_hosts(self, hosts, filter_properties,
            filter_class_names=None, index=0, context=None):
        ...
        if filter_class_names is None:
            filters = self.default_filters
        else:
            filters = self._choose_host_filters(filter_class_names)
        ...
        
    def _choose_host_filters(self, filter_cls_names):
        ...
        #将filter_cls_names封装成列表
        if not isinstance(filter_cls_names, (list, tuple)):
            filter_cls_names = [filter_cls_names]

        good_filters = []
        bad_filters = []
        for filter_name in filter_cls_names:
            if filter_name not in self.filter_obj_map:
                if filter_name not in self.filter_cls_map:
                    bad_filters.append(filter_name)
                    continue
                filter_cls = self.filter_cls_map[filter_name]
                self.filter_obj_map[filter_name] = filter_cls()
            good_filters.append(self.filter_obj_map[filter_name])
        if bad_filters:
            msg = ", ".join(bad_filters)
            raise exception.SchedulerHostFilterNotFound(filter_name=msg)
        return good_filters
        
    def __init__(self):
        self.filter_handler = filters.HostFilterHandler()
        #__init__方法中调用get_matching_classes方法去加载nova.conf配置文件的scheduler_available_filters属性设置的fileter
        filter_classes = self.filter_handler.get_matching_classes(
                CONF.scheduler_available_filters)
        self.filter_cls_map = {cls.__name__: cls for cls in filter_classes}
        self.default_filters = self._choose_host_filters(self._load_filters())
        
    def _load_filters(self):
        return CONF.scheduler_default_filters
        
```
nova.conf中的相关配置如下：

```
scheduler_available_filters=nova.scheduler.filters.all_filters
scheduler_default_filters=RetryFilter,AvailabilityZoneFilter,RamFilter,ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter
```

总结一下_choose_host_filters的工作流程：

1. 将filter_cls_names封装成列表
2. 依次检查filter_cls_names中的所有可用的filter列表（个人理解，其检查过程就是判断scheduler_default_filters定义的filter能不能在nova/filter/下找到相关类）
3. 返回可用filter列表


## 添加自定义filter

自定义一个filter类，specified_host_filter.py：

```

from oslo_log import log as logging

from nova.scheduler import filters

LOG = logging.getLogger(__name__)

#任何filter类必须继承filters.BaseHostFilter类
class SpecifiedHostFilter(filters.BaseHostFilter):
    def __init__(self):
        #通知成功加载SpecifiedHostFilter类
        LOG.info("SpecifiedHostFilter is initialized!")
    
    #host_passes是每个类必须实现的方法,host_state保存了询问的计算节点的信息
    #filter_properties保存了一些帮助nvoa scheduler完成调度的信息
    def host_passes(slef, host_state, filter_properties):
        #获取客户端要求的计算节点主机名
        scheduler_hints = filter_properties.get('scheduler_hints',{})
        requested_host = scheduler_hints.get('requested_host',None)
        #如果客户提供了要求的计算节点，则检查当前计算节点与客户要求的节点是否匹配
        if requested_host:
        	return requested_host ==host_state.host
        #如果客户没有提供要求的计算节点，则返回真
        return true
```

将自定义filter放在nova目录下，形成下面的目录结构:

```
├── scheduler
├── myproject
│   ├── __init__.py
│   ├── specified_host_filter.py
```

修改nova.conf配置文件，

```
scheduler_available_filters=nova.scheduler.filters.all_filters
scheduler_available_filters=nova.myproject.specified_host_filter.SpecifiedHostFilter
scheduler_default_filters=RetryFilter,AvailabilityZoneFilter,RamFilter,ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter,SpecifiedHostFilter
```

重启nova-scheduler服务


