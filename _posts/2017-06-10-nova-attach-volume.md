---
layout:     post
title:      nova 挂载卷源码分析
date:       2017-06-10
author:     xue
catalog:    true
tags:
    - openstack
    - nova
---

本文以kilo版本的nova为例


## nova挂载卷源码分析

nova/api/openstack/compute/contrib/volumes.py   
在Nova Client进程中，由VolumeAttachmentController接受挂载请求

```
nova client
    ↓
POST /v2.1/servers/c2486c45-5a8f-4028-8cd1-51c0425f677a/os-volume_attachments
    ↓
nova.api.openstack.compute.volumes.VolumeAttachmentController.create
    self.compute_api.attach_volume
        self.compute_api 可用的有两个：
            1. nova.compute.cells_api.ComputeCellsAPI
            2. nova.compute.api.API
        根据 nova.conf 配置项决定使用哪一个，默认情况下是 nova.compute.api.API
```

```
nova.compute.api.API.attach_volume
    self._attach_volume
        ↓
nova.compute.api.API._attach_volume
    step1: volume_bdm = self.compute_rpcapi.reserve_block_device_name
    step2: volume = self.volume_api.get(context, volume_id)
    step3: self.volume_api.check_attach(context, volume, instance=instance)
    step4: self.volume_api.reserve_volume(context, volume_id)
    step5: self.compute_rpcapi.attach_volume(context, instance=instance,
                    volume_id=volume_id, mountpoint=device, bdm=volume_bdm)      
```

step1: 创建 BDM 数据库表数据

```
nova.compute.api.API._attach_volume
    volume_bdm = self.compute_rpcapi.reserve_block_device_name
        ↓
nova.compute.rpcapi.ComputeApi.reserve_block_device_name
    volume_bdm = cctxt.call(ctxt, 'reserve_block_device_name', **kw)
        ↓
nova.compute.manager.reserve_block_device_name
    bdm = objects.BlockDeviceMapping(...)
    bdm.create() 创建 BDM 数据库表数据  
```

step2: 调用 cinderclient 获取需要挂载的 volume 信息

```
nova.compute.api.API._attach_volume
    volume = self.volume_api.get(context, volume_id)
        ↓
nova.volume.API.get
    item = cinderclient(context).volumes.get(volume_id)
         
```

step3: 检查cinder status和可用域

```
nova.compute.api.API._attach_volume
    self.volume_api.check_attach(context, volume, instance=instance)
        ↓
nova.volume.API.check_attach
    if (volume['status'] not in ("available", 'in-use', 'attaching'))
    if instance_az != volume['availability_zone']
```

step4: 调用 cinderclient 更新 cinder DB -> 'status': 'attaching'（以防止其他操作进入）

```
nova.compute.api.API._attach_volume
    self.volume_api.reserve_volume(context, volume_id)
        ↓
nova.volume.API.reserve_volume
     cinderclient(context).volumes.reserve(volume_id)
```

step5: 挂载卷

```
nova.compute.api.API._attach_volume
    self.compute_rpcapi.attach_volume(context, instance=instance,
                    volume_id=volume_id, mountpoint=device, bdm=volume_bdm)  
        ↓
nova.compute.rpcapi.ComputeAPI.attach_volume
     cctxt = self.client.prepare(server=_compute_host(None, instance),
                version=version)
     cctxt.cast(ctxt, 'attach_volume', **kw)
        ↓
nova.compute.manager.ComputeManager.attach_volume
     step6: driver_bdm = driver_block_device.convert_volume(bdm) 
     step7: self._attach_volume(context, instance, driver_bdm)       

```

step6: 根据 source_type 获取 BDM 驱动 DriverVolumeBlockDevice

```
nova.compute.manager.ComputeManager.attach_volume
     driver_bdm = driver_block_device.convert_volume(bdm) 
        ↓
nova.virt.block_device.convert_volume
     source_volume = convert_volumes(volume_bdms)
     convert_volumes = functools.partial(_convert_block_devices,
                                   DriverVolumeBlockDevice)
                     
```

step7: BDM driver attach

```
nova.compute.manager.ComputeManager.attach_volume
     self._attach_volume(context, instance, driver_bdm)
        ↓
nova.compute.manager.ComputeManager._attach_volume  
     bdm.attach(context, instance, self.volume_api, self.driver,
                       do_check_attach=False, do_driver_attach=True)
        ↓           
nova.virt.block_device.DriverVolumeBlockDevice.attach
     volume_api.check_attach(context, volume, instance=instance) #检查cinder status和可用域
     step8:  connector = virt_driver.get_volume_connector(instance)
     step9:  connection_info = volume_api.initialize_connection(context,volume_id,connector)
     step10: virt_driver.attach_volume()  
```

step8: 获取 connector，为 initialize_connection 提供参数

```
nova.virt.block_device.DriverVolumeBlockDevice.attach
     connector = virt_driver.get_volume_connector(instance)
        ↓
nova.virt.libvirt.driver.LibvirtDriver.get_volume_connector
     self._initiator = libvirt_utils.get_iscsi_initiator()
     self._fc_wwnns = libvirt_utils.get_fc_wwnns()
     self._fc_wwpns = libvirt_utils.get_fc_wwpns()
     return connector
```

step9: 存储端进行挂载返回 connection_info

```
nova.virt.block_device.DriverVolumeBlockDevice.attach
     connection_info = volume_api.initialize_connection(context,volume_id,connector)
        ↓
nova.volume.cinder.API.initialize_connection
     return cinderclient(context).volumes.initialize_connection(volume_id,
                                                                   connector)
```

step10: 主机端根据 connection_info 挂载卷到目标主机（iSCSI 为例）

```
nova.virt.block_device.DriverVolumeBlockDevice.attach
     virt_driver.attach_volume() 
        ↓
nova.virt.libvirt.driver.LibvirtDriver.attach_volume
     self._connect_volume(connection_info, disk_info)
        ↓
nova.virt.libvirt.volume.iscsi.LibvirtISCSIVolumeDriver.connect_volume
     # 接下来分为多路径和单路径两种情况，开启多路径的配置见文章最后的参考
     if self.use_multipath:
        run_iscsiadm_update_discoverydb()
        out = self._run_iscsiadm_bare()
        self._connect_to_iscsi_portal(props)
        self._rescan_iscsi()
     else:
        self._connect_to_iscsi_portal(iscsi_properties)
        self._run_iscsiadm(iscsi_properties, ("--rescan",))
     host_device = self._get_host_device(iscsi_properties)
                
```

step10中涉及的命令有：

其中牵扯到的指令有：  
a. 尝试连接  
iscsiadm -m node -T target_iqn -p target_protal  
b. 连接失败重新建立连接  
iscsiadm -m node -T target_iqn -p target_protal -op new  
iscsiadm -m node -T target_iqn -p target_protal —op update -n node.session.auth.authmethod -v auth_method  
iscsiadm -m node -T target_iqn -p target_protal —op update -n node.session.auth.username -v auth_username  
iscsiadm -m node -T target_iqn -p target_protal —op update -n node.session.auth.password -v auth_password  
c. 检查session，登陆  
iscsiadm -m session检查是否登录成功  
iscsiadm –m node –T targetname –p ip —login 登陆建立session  
d. 设置为随机器启动而启动  
iscsiadm -m node -T target_iqn -p target_protal —op update -n node.startup -v automatic  
iscsiadm -m node -T target_iqn -p target_protal –rescan
 
 
step11: 更新 cinder DB 数据

```
nova.virt.block_device.DriverVolumeBlockDevice.attach
     volume_api.attach(context, volume_id, instance.uuid,
                       self['mount_device'], mode=mode)
        ↓
nova.volume.API.attach()
     cinderclient(context).volumes.attach(volume_id, instance_uuid,
                                          mountpoint, mode=mode)
```


step10 提到的多路径：

普通的电脑主机都是一个硬盘挂接到一个总线上，这里是一对一的关系。而到了有光纤组成的SAN环境，或者由iSCSI组成的IPSAN环境，由于主机和存储通过了光纤交换机或者多块网卡及IP来连接，这样的话，就构成了多对多的关系。也就是说，主机到存储可以有多条路径可以选择。主机到存储之间的IO由多条路径可以选择。每个主机到所对应的存储可以经过几条不同的路径，如果是同时使用的话，I/O流量如何分配？其中一条路径坏掉了，如何处理？还有在操作系统的角度来看，每条路径，操作系统会认为是一个实际存在的物理盘，但实际上只是通向同一个物理盘的不同路径而已，这样是在使用的时候，就给用户带来了困惑。多路径软件就是为了解决上面的问题应运而生的。

多路径的主要功能就是和存储设备一起配合实现如下功能：

1. 故障的切换和恢复
2. IO流量的负载均衡
3. 磁盘的虚拟化

## 参考
[开启多路径](https://docs.openstack.org/liberty/config-reference/content/config-iscsi-multipath.html)  
[yikun's blog](http://yikun.github.io/2016/03/05/OpenStack%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E6%8C%82%E8%BD%BD%E5%8D%B7%E6%B5%81%E7%A8%8B/)
 
