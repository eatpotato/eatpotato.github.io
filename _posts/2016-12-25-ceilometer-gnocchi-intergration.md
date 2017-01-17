---
layout:     post
title:      gnocchi原理及与ceilometer的集成
date:       2016-12-25
author:     xue
tags:
    - openstack
---

## gnocchi数据的存储

Gnocchi是把索引存在了其他数据库里面，然后针对实际的操作数据存储在自己的数据库中，自己的数据库中又以:  resource -> metric -> measure的数据分离形式来存储实际的监控数据，一般提取数据的时候只需要从metric层开始提取。

### Resource
resource是gnocchi对openstack监控数据的一个大体的划分,有磁盘的数据，镜像的数据，网络的数据，接口的数据等等.通过下面的命令可以看到实际使用的resource的名字

![](/img/gnocchi/gnocchi-resource-list.png)

### metric
metric是gnocchi对数据的第二层划分，归属与resource，如resource为image的项目下面，就可能有image.download,image.serve,image.size等等，我们可以通过下面的命令看到一个resource下面的所有的metric

![](/img/gnocchi/gnocchi-metric-list.png)

### measures
measures是gnocchi对数据的第三层划分也是最后一层，归属于metric，保存了实际的监控数据，可以通过下面的命令去查看某一个metric的所有数据:

![](/img/gnocchi/gnocchi-measures-show.png)

## gnocchi存储引擎

就像mysql一样，Gnocchi也有自己的存储引擎：

* File
* Swift
* Ceph
* InfluxDB

前面3种都不是专业的的处理时序序列的存储引擎，而且都依赖一个名为Carbonara的中间件来实现存储。InfluxDB虽然是一个专业的处理时序数据的数据库，但是由于InfluxDB本身就是一个开源软件，到现在虽然集成上Gnocchi了，但是还是有很多bug。

Gnocchi在存储数据的时候采用了压缩存储的技术，所以计算方式可以大致参照下面的计算方法:

```
# 需要存储的空间
number of points × 9 bytes = size in bytes
 
# 你需要存储的点,如:1个小时的监控数据,然后你的采集的密度是60s，那就是60*60 / 60 = 60个点
number of points = timespan ÷ granularity
 
# 下面如果你要存储1年的数据，采集的数据密度是5s，那么一年后你将会有那么大的数据量:
size in bytes = (365 * 24 * 60 * 60)/5  × 9 = 56 764 800 bytes = 55 434 KiB = 54 MiB

```

上面是按照一个监控项(metric)做计算,如果你要监控N个metrics（就是对应ceilometer的meter）,就要对应乘上N。

## gnocchi 集成ceilometer

gnocchi安装文档可以参考：[Ceilometer + Gnocchi + Aodh](http://www.cnblogs.com/multi-task/p/5553830.html)  
ceilometer.conf需要修改配置:


```
[Default]
meter_dispatchers=gnocchi
event_dispatchers=gnocchi

[api]
gnocchi_is_enabled = true

[dispatcher_gnocchi]
filter_service_activity=False
filter_project=services
url=http://lb.81.merge.polex.io:8041
archive_policy=low
resources_definition_file=gnocchi_resources.yaml
```

ceilometer与gnocchi集成之后，除了ceilometer-alarm之外的api将变得不可用，dashboard也无法再查看到相应的记录。

## gnocchi_resources.yaml
配置resources_definition_file=gnocchi_resources.yaml后，监控项在gnocchi_resources.yaml文件定义，如：


```

---

resources:
  - resource_type: instance
    archive_policy: 1days_per_sec
    metrics:
      - 'memory.usage'
      - 'cpu_util'
      - "disk.read.bytes"
      - "disk.read.requests"
      - "disk.write.bytes"
      - "disk.write.requests"
      - 'disk.read.requests.rate'
      - 'disk.write.requests.rate'
      - 'disk.read.bytes.rate'
      - 'disk.write.bytes.rate'
      - 'disk.capacity'
      - 'disk.allocation'
      - 'disk.usage'
    attributes:
      host: resource_metadata.host
      image_ref: resource_metadata.image_ref
      display_name: resource_metadata.display_name
      flavor_id: resource_metadata.(instance_flavor_id|(flavor.id))
      server_group: resource_metadata.user_metadata.server_group

  - resource_type: instance_network_interface
    archive_policy: 1days_per_sec
    metrics:
      - 'network.outgoing.packets.rate'
      - 'network.incoming.packets.rate'
      - 'network.outgoing.bytes.rate'
      - 'network.incoming.bytes.rate'
    attributes:
      name: resource_metadata.vnic_name
      instance_id: resource_metadata.instance_id

  - resource_type: instance_disk
    archive_policy: 1days_per_sec
    metrics:
      - 'disk.device.read.requests.rate'
      - 'disk.device.write.requests.rate'
      - 'disk.device.read.bytes.rate'
      - 'disk.device.write.bytes.rate'
      - 'disk.device.capacity'
      - 'disk.device.allocation'
      - 'disk.device.usage'
    attributes:
      name: resource_metadata.disk_name
      instance_id: resource_metadata.instance_id

  - resource_type: network
    archive_policy: 1days_per_sec
    metrics:
      - 'bandwidth'
      - 'network.services.lb.outgoing.bytes'
      - 'network.services.lb.incoming.bytes'
      - 'network.services.lb.total.connections'
      - 'network.services.lb.active.connections'
```
  
  gnocchi_resources.yaml文件默认定了较多的监控项，上面的例子定义了一些常用监控项。
  
  archive_policy为采用的监控策略，可以通过下面的命令常看:   
  ![](/img/gnocchi/gnocchi-archive-policy.png)  
  granularity为监控的时间间隔，timespan为保留时间
  
  创建archive_policy的命令为：
  
```
gnocchi archive-policy create -d granularity:1s,points:86400 1days_per_sec
```
创建archive_policy只需要指定granularity和points ，它们的乘积即为timespan
  
  
  
