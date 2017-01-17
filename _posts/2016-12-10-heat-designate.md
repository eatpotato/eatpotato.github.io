---
layout:     post
title:      通过Heat快速搭建分布式designate服务
date:       2016-12-10
author:     xue
catalog:    true
tags:
    - openstack
---

## heat基础知识
本文可以在已安装openstack的环境下，快速搭建分布式designate服务。

### 使用到的heat 内部函数
* get_attr

作用：获取所创建资源的属性。


```
get_attr:

  - <resource name>

  - <attribute name>

  - <key/index 1> (optional)

  - <key/index 2> (optional)

  - ...

```
Resource name：必须是模板 resouce 段中指定的资源。  
Attribute name：要获取的属性，如果属性对应的值是list 或map， 则可以指定key/index来获取具体的值。  
示例：

```
resources:

  my_instance:

    type: OS::Nova::Server

 ...



outputs:

  instance_ip:

    description: IP address of the deployed compute instance

    value: { get_attr: [my_instance, first_address] }

  instance_private_ip:

    description: Private IP address of the deployed compute instance

    value: { get_attr: [my_instance, networks, private, 0] }
```

* get_param

作用：引用模板中指定的参数。
语法：


```
get_param:

 - <parameter name>

 - <key/index 1> (optional)

 - <key/index 2> (optional)

 - ...
```
示例：

```
parameters:

   instance_type:

    type: string

    label: Instance Type

    description: Instance type to be used.

  server_data:

    type: json



resources:

  my_instance:

    type: OS::Nova::Server

    properties:

      flavor: { get_param: instance_type}

      metadata: { get_param: [ server_data, metadata ] }

      key_name: { get_param: [ server_data, keys, 0 ] }

```
输入参数是：

```
{"instance_type": "m1.tiny",
{"server_data": {"metadata": {"foo": "bar"},
                 "keys": ["a_key","other_key"]}}}

```

* get_resource
作用：获取模板中指定的资源。
语法：

```
get_resource: <resource ID>
```
示例：

```
resources:

  instance_port:

    type: OS::Neutron::Port

    properties: ...



  instance:

    type: OS::Nova::Server

    properties:

      ...

      networks:

        port: { get_resource: instance_port }
```

## heat模板分析
模板中各参数（parameters）的含义

| 参数名称 | 参数含义 |
|---|---|
| key_name | Name of key-pair to be used for compute in |
| image_id | 镜像id  |
| instance_type | Type of instance (flavor) to be used |
| secgroup_name | security groups ，可以是一个列表 |
| password | root登录密码 |
| service_passwd | designate mysql rabbitmq keystone 相关服务的密码 |
| network_id | network id |

### 模板中各资源(resources)的含义

| 资源名称 | 资源类型| 资源作用 | 
|---|---|---|
| dns_secgroup | OS::Neutron::SecurityGroup | 为创建的两台虚拟机配置安全组规则，允许53端口的入口流量，即允许外部向它们发送dns请求 |
| designate-slave | OS::Nova::Server | 此虚拟机安装bind服务 |
| designate-master | OS::Nova::Server | 此虚拟机安装mysql rabbitmq keystone designate bind 服务 |

### 搭建步骤
1.创建下面的yaml文件：

[designate-final.yaml](/files/designate-final.yaml)

2. 执行命令(根据具体需要修改参数)：

```
openstack stack create -t designate-final.yaml -p "key_name=$KEY_NAME image_id=$IMAGE_ID instance_type=$INSTANCE_TYPE network_id=$NETWORK_ID" STACK_NAME
```
注意：由于我事先在镜像里安装了designate rabbitmq mysql bind keystone httpd 等相关包，所以此heat模板中并没有包的安装过程，如果用此模板安装，必须要使用安装了所有相关软件包的镜像（镜像必须为centos7系统）。等待两分钟左右便可。关于designate的原理和操作可以查看我的另一篇博客： [puppet-designate](http://xuefy.cn/2016/11/14/puppet-designate/)  
另外如果镜像不包含相应的软件包，也可以创建下面的heat模板，此模板自带相应软件包的安装：
[designate-final-full.yaml](/files/designate-final-full.yaml)


### 扩展节点
如果后面觉得两个节点的不够使，再扩展slave节点的话也比较简单，再扩展的designate-slave节点的内容完全一致，直接copy一份再改一下name。designate-master也很简单，只需要在最后的params中加上$slave_new_ip_address: {get_attr: [designate-slave-new, first_address]},然后把$slave_new_ip_address加入到bind配置文件中即可。

