---
layout:     post
title:      nova创建虚拟机源码分析
date:       2017-04-26
author:     xue
catalog:    true
tags:
    - openstack
---


以nova kilo版本为例
## nova源码目录
**部分**源码目录如下所示：

```
.
├── CA
├── api               - Nova API服务
│   ├── ec2           - Amazon EC2 API 支持 
│   ├── metadata
│   ├── openstack     - Openstack API
│   ├── validation
├── cells             - nova-cells服务
├── cert              - nova-cert服务
├── cloudpipe         - cloudpipe支持，提供vpn服务
├── cmd               - 各个nova服务的入口程序,所有服务的main函数
├── compute           - nova-compute服务
├── conductor         - nova-conductor服务
├── console           - nova-console服务
├── consoleauth       - nova-consoleauth服务
├── db                - 数据库操作
├── hacking           - 编码规范检查
├── image             - glance接口抽象
├── ipv6
├── keymgr
├── locale            - 本地化处理
├── network           - nova-network服务
├── objects           - 对象模型，封装了所有实体对象的CURD操作，相对以前直接调用db的model更安全，并且支持版本控制
├── openstack         - 来自于Oslo的代码
│   └── common
├── pci               - SR-IOV支持
├── policies          - policy校验实现
├── scheduler         - nova-schedule服务
├── servicegroup
├── spice
├── tests             - 单元测试
├── virt              - Hypervisor driver
├── vnc
├── volume            - cinder抽象接口
```

下面是一个创建虚拟机的整体调用过程，分析的openstack版本为kilo

## 一、nova api
每个资源对应一个底层Controller类，处理虚拟机创建请求的方法为create方法，其定义如下：

```
#nova/api/openstack/compute/servers.py

class Controller(wsgi.Controller):

        def create(self, req, body):
        """Creates a new server for a given user."""
        # 检查HTTP消息体是否合法
        if not self.is_valid_body(body, 'server'):
            raise exc.HTTPUnprocessableEntity()
        # 获得客户端传入的虚拟机参数
        context = req.environ['nova.context']
        # 通过server_dict保存虚拟机参数
        server_dict = body['server']
        password = 	 self._get_server_admin_password(server_dict)
        if 'name' not in server_dict:
            msg = _("Server name is not defined")
            raise exc.HTTPBadRequest(explanation=msg)
        # 获取和检查虚拟机名
        name = server_dict['name']
        # 检查虚拟机名的长度是否越界
        self._validate_server_name(name)
        name = name.strip()
        # 获取 虚拟机磁盘镜像uuid
        image_uuid = self._image_from_req_data(body)
        ...
        # 获取客户端需求的网络
        requested_networks = self._determine_requested_networks(server_dict)
        # 获取和验证客户端指定的虚拟机IP
        (access_ip_v4, ) = server_dict.get('accessIPv4'),
        if access_ip_v4 is not None:
            self._validate_access_ipv4(access_ip_v4)
        # 获取虚拟机规格id
        flavor_id = self._flavor_id_from_req_data(body)
        try:
            #获取虚拟机规格信息
            _get_inst_type = flavors.get_flavor_by_flavor_id
            inst_type = _get_inst_type(flavor_id, ctxt=context,
                                       read_deleted="no")
            #调用了nova/compute/api.py的create方法,创建虚拟机
            (instances, resv_id) = self.compute_api.create(context,
                       ...)
            ...
        #将虚拟机信息转化为字典
        server = self._view_builder.create(req, instances[0])
        #将虚拟机信息封装成ResponseObject对象
        robj = wsgi.ResponseObject(server)
        #添加访问当前虚拟机资源的url
        return self._add_location(robj)

```

对Nova API中底层controller对象定义的HTTP请求的处理方法做一个小节，一般一个HTTP请求的处理方法需要完成如下工作：

1.检查客户端传入的参数是否合法  
2.调用Nova其他自服务的API处理客户端的HTTP请求  
3.将Nova其他自服务的API返回结果转化为可视化的字典  



Compute API类的create方法定义如下：

```
#nova/compute/api.py

class API(base.Base):

    @hooks.add_hook("create_instance")
    def create(self, context, instance_type,
          ...):
        #检查客户是否具有创建虚拟机的权限
    	self._check_create_policies(context, availability_zone,
                requested_networks, block_device_mapping)
        #进一步处理虚拟机创建请求
        return self._create_instance(
                ...)
                
    def _create_instance(self, context, instance_type,
          ...):
        #先做一些验证工作，这里不再赘述
        ...
        #更新InstanceAction表记录，将虚拟机操作状态设置为“开始创建”
        for instance in instances:
            self._record_action_start(context, instance,
                                      instance_actions.CREATE)
        #调用conductor/api.py的build_instances方法
        self.compute_task_api.build_instances(context,
                ...)
        return (instances, reservation_id)
        
    @property
    def compute_task_api(self):
        if self._compute_task_api is None:
            from nova import conductor
            self._compute_task_api = conductor.ComputeTaskAPI()
        return self._compute_task_api


```

conductor API类的build_instances方法:

```
#nova/conductor/api.py,实际上conductor的api对rpcapi做了一层封装

class ComputeTaskAPI(object):

	def build_instances(self, context, instances, image, filter_properties,
	           ...):
	     self.conductor_compute_rpcapi.build_instances(context,
	           ...)
	          
#nova/conductor/rpcapi.py
class ComputeTaskAPI(object):
    	          
	def build_instances(self, context, instances, image, filter_properties,
	          ...):
	    ...
	    #获得目标主机的RPC client
	    cctxt = self.client.prepare(version=version)
	    #rpc cast是异步调用，第二个参数是RPC调用的函数名，kw是传递的参数
        cctxt.cast(context, 'build_instances', **kw)
        '''截至到现在，虽然目录由api->compute->conductor，但仍在nova-api进程中运行，
        直到cast方法执行，该方法由于是异步调用，因此nova-api不会等待远程方法调用结果，直接返回结束。''''
```

## 二、nova conductor
由于是向nova-conductor发起的RPC调用,nova/conductor/rpcapi.py里面只是定义了服务的RPC调用接口，真正完成任务的是nova/conductor/manager.py里面的方法：

```
#nova/conductor/manager.py
class ComputeTaskManager(base.Base):
   def build_instances(self, context, instances, image, filter_properties,
       ...)
       ...
       try:
           #调用了nova/schduler/client/query.py里面的select_destinations
           hosts = self.scheduler_client.select_destinations(context,
                    request_spec, filter_properties)
                    
#nova/scheduler/client/query.py
class SchedulerQueryClient(object):
   def select_destinations(self, context, request_spec, filter_properties):
        #调用nova/scheduler/rpcapi.py里面的select_destinations方法
        return self.scheduler_rpcapi.select_destinations(
            context, request_spec, filter_properties)
            
#nova/scheduler/rpcapi.py
class SchedulerAPI(object):
    def select_destinations(self, ctxt, request_spec, filter_properties):
        cctxt = self.client.prepare(version='4.0')
        #这里是同步调用，此时nova-conductor并不会退出，而是堵塞等待直到nova-scheduler返回。
        return cctxt.call(ctxt, 'select_destinations',
            request_spec=request_spec, filter_properties=filter_properties)
        
```

## 三、nova scheduler

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

## 四、nova conductor

第三步执行完成后，返回到第二步的 hosts = self.scheduler_client.select_destinations(context,
                    request_spec, filter_properties)，继续往下执行
                    
```
#nova/conductor/manager.py
class ComputeTaskManager(base.Base):
   def build_instances(self, context, instances, image, filter_properties,
       ...)
       ...
       try:
           #调用了nova/schduler/client/query.py里面的select_destinations
           hosts = self.scheduler_client.select_destinations(context,
                    request_spec, filter_properties)
           ...
           #调用nova/compute/rpcapi.py的build_and_run_instance方法
           self.compute_rpcapi.build_and_run_instance(context,
                    ...)
                    
#nova/compute/rpc.api.py   
class ComputeAPI(object):
   def build_and_run_instance(self, ctxt, instance, host, image, request_spec,
       ...):
       ...
       cctxt = self.client.prepare(server=host, version=version)
       #异步调用，nova-conductor结束
       cctxt.cast(ctxt, 'build_and_run_instance', instance=instance,  
           ...)
                         
```

## 五、nova compute

```
#nova/compute/manager.py
class ComputeManager(manager.Manager):
   #build_and_run_instance会调用_build_and_run_instance，这里其调用过程
   def _build_and_run_instance(self, context, instance, image, request_spec,
      ...):
      ...
      image_name = image.get('name')
      # ==> nova.image.__init__
             #     ==> nova.image.api.API:get()
             #         ==> nova.image.api.API:_get_session_and_image_id()
             #             ==> nova.image.glance:get_remote_image_service()
             #                 ==> nova.image.glance.GlanceImageService:show()   
             #                     ==> nova.image.glance._tnslate_from_glanceranslate_from_glance
             # Return:ClanceImageService.show() 返回一个包含了 Image_Mete 信息的 Dict['name'] == uri_or_id
      
      # 向外部发出一个 start to create instance 的通知
      self._notify_about_instance_usage(context, instance, 'create.start',
                extra_usage_info={'image_name': image_name})
      ...
      '''
      该方法调用了driver的spawn方法，这里的driver就是各种hypervisor的实现，
      所有实现的driver都在virt目录下，入口为driver.py，比如libvirt driver实现对应为virt/libvirt/driver.py
      '''
      self.driver.spawn(context, instance, image,
                                      injected_files, admin_password,
                                      network_info=network_info,
                                      block_device_info=block_device_info)
                                      
                                      
#nova/virt/libvirt/driver.py
class LibvirtDriver(driver.ComputeDriver):

   def spawn(self, context, instance, image_meta, injected_files,
              admin_password, network_info=None, block_device_info=None):
        ...
        #创建虚拟机磁盘镜像
        self._create_image(context, instance,
                           ...)
        #创建虚拟机XML定义文件                   
        xml = self._get_guest_xml(context, instance, network_info,
                           ...)
        #创建虚拟机和虚拟机网络资源
        self._create_domain_and_network(context, xml, instance, network_info,
                           ...)
         
        def _wait_for_boot():
            """Called at an interval until the VM is running."""
            state = self.get_info(instance).state
            #检查虚拟机的power_state树形
            if state == power_state.RUNNING:
                LOG.info(_LI("Instance spawned successfully."),
                         instance=instance)
                raise loopingcall.LoopingCallDone()
        #调用上面定义的_wait_for_boot,创建定时线程等待虚拟机创建完毕
        timer = loopingcall.FixedIntervalLoopingCall(_wait_for_boot)
        timer.start(interval=0.5).wait()                                     
```
