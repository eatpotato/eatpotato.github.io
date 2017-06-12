---
layout:     post
title:      cinder创建卷源码分析
date:       2017-06-11
author:     xue
catalog:    true
tags:
    - openstack
    - cinder
---

本文以kilo版本的cinder为例


代码整体流程如下：



![](/img/cinder/cinder-architecture.png)

如整体架构图所示，创建卷涉及的大体步骤主要有以下几步：  
a. Client发送请求，通过RESTFUL接口访问cinder-api。  
b. Api解析响应请求，api解析由Client发送来的请求，并通过rpc进一步调用cinder-scheduler。  
c. Scheduler对资源进行调度，scheduler选择合适的节点进行。  
d. Volume调用Driver创建卷，volume通过指定Driver进行卷的创建。  


## cinder api

```
cinder client
POST  /v2/{tenant_id}/volumes  
    ↓
cinder.api.v2.volumes.VolumeController.create:
    # 对volume_type、metadata、snapshot等信息进行检查
    new_volume = self.volume_api.create(context,
                                        size,
                                        volume.get('display_name'),
                                        volume.get('display_description'),
                                        **kwargs)
    ↓
cinder.volume.api.API.create:
    # 对source_volume、volume_type等参数进行进一步检查
    flow_engine = create_volume.get_flow()
    ↓
cinder.volume.flows.api.create_volume.get_flow:
    api_flow.add(ExtractVolumeRequestTask())
    api_flow.add(QuotaReserveTask(),
                 EntryCreateTask(db_api),
                 QuotaCommitTask())
    api_flow.add(VolumeCastTask())
    ↓
cinder.volume.api.API.create:
    flow_engine.run()
    ↓
cinder.volume.flows.api.create_volume.ExtractVolumeRequestTask.execute:
    # 获取 request 信息并返回
cinder.volume.flows.api.create_volume.QuotaReserveTask.execute:
    # 预留配额
cinder.volume.flows.api.create_volume.EntryCreateTask.execute:
    # 在数据库中创建 volume 条目,此时卷的状态为”creating”
cinder.volume.flows.api.create_volume.QuotaCommitTask.execute:
    # 确认配额
cinder.volume.flows.api.create_volume.VolumeCastTask.execute:
    self._cast_create_volume(context, request_spec, filter_properties)
    ↓
cinder.volume.flows.api.create_volume.VolumeCastTask._cast_create_volume:
    if not host: # 如果未传入host,则会经过调度进行创建卷
        self.scheduler_rpcapi.create_volume()
    else: # 如果传入host,直接交由Volume Manager去创建卷
        self.volume_rpcapi.create_volume()
    ↓
cinder.scheduler.rpcapi.SchedulerAPI.create_volume: # 以未传入host为例
    return cctxt.cast(ctxt, 'create_volume', ···)
```



## cinder scheduler

```
cinder.scheduler.manager.SchedulerManager.create_volume:
    flow_engine = create_volume.get_flow(context,···)
    ↓
cinder.scheduler.flows.create_volume.get_flow:
    scheduler_flow.add(ExtractSchedulerSpecTask())
    scheduler_flow.add(ScheduleCreateVolumeTask())
    ↓
cinder.scheduler.manager.SchedulerManager.create_volume:
    flow_engine.run()
    ↓
cinder.scheduler.flows.create_volume.ExtractSchedulerSpecTask.execute:
    # 获取用于调度的信息
    ↓ 
cinder.scheduler.flows.create_volume.ScheduleCreateVolumeTask.execute:
    self.driver_api.schedule_create_volume()
    ↓
cinder.scheduler.filter_scheduler.FilterScheduler.schedule_create_volume:
    weighed_host = self._schedule()
    # 更新数据库
    updated_volume = driver.volume_update_db(context, volume_id, host)
    self.volume_rpcapi.create_volume()
    ↓
cinder.volume.rpcapi.VolumeAPI.create_volume:
    cctxt.cast(ctxt, 'create_volume',···)
```

## cinder volume

```
cinder.volume.manager.VolumeManage.create_volume:
    flow_engine = create_volume.get_flow()
    ↓
cinder.volume.flows.manager.create_volume.get_flow:
    volume_flow.add(ExtractVolumeRefTask(db, host))
    if allow_reschedule and request_spec: # 如果允许重新调度
        volume_flow.add(OnFailureRescheduleTask())
    volume_flow.add(ExtractVolumeSpecTask(db),
                    NotifyVolumeActionTask(db, "create.start"),
                    CreateVolumeFromSpecTask(db, driver),
                    CreateVolumeOnFinishTask(db, "create.end"))
    ↓                 
cinder.volume.manager.VolumeManage.create_volume:
    flow_engine.run()
    ↓
cinder.volume.flows.manager.create_volume.ExtractVolumeRefTask.execute:
    #从数据库中获得volume信息
    volume_ref = self.db.volume_get(context, volume_id)
    return volume_ref
    ↓
cinder.volume.flows.manager.create_volume.OnFailureRescheduleTask.execute:
    # 当进行task恢复回滚操作的时候,触发一个发送进行重新调度的请求
    ↓ 
cinder.volume.flows.manager.create_volume.ExtractVolumeSpecTask.execute:
    # 返回要创建volume的通用结构规范
    return specs
    #
    ↓
cinder.volume.flows.manager.create_volume.NotifyVolumeActionTask.execute:
    # 执行关于给定卷的相关通知操作，获取指定卷的使用率信息，并进行通知操作
    ↓
cinder.volume.flows.manager.create_volume.CreateVolumeFromSpecTask.execute:
    # 根据创建的不同类别，去创建卷
    create_type = volume_spec.pop('type', None)
    if create_type == 'raw':
        model_update = self._create_raw_volume(context,···)
    elif create_type == 'snap':
        model_update = self._create_from_snapshot(context,···)
    elif create_type == 'source_vol':
        model_update = self._create_from_source_volume(···)
    elif create_type == 'source_replica':
        model_update = self._create_from_source_replica(···)
    elif create_type == 'image':
        model_update = self._create_from_image(context,···)
    ↓
cinder.volume.flows.manager.create_volume.CreateVolumeFromSpecTask._create_raw_volume:
    # 以原始创建方式为例
    return self.driver.create_volume(volume_ref)
    ↓
cinder.volume.flows.manager.create_volume.CreateVolumeOnFinishTask.execute: 
    # 当成功的建立卷之后，完成卷建立之后的通知操作,启动更新数据库，将卷更新为available状态。               
```

## 参考
[yikun's blog](http://yikun.github.io/2016/02/14/OpenStack%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-Cinder%E5%88%9B%E5%BB%BA%E5%8D%B7%E6%B5%81%E7%A8%8B/)
 
