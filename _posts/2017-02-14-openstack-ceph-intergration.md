---
layout:     post
title:      openstack与ceph集成
date:       2017-02-14
author:     xue
catalog:    true
tags:
    - ceph
    - openstack
---

## Openstack与ceph集成

说明：在进行openstack与ceph集成之前，请保证已有一套openstack环境和ceph环境，为了便于演示，本次openstack采用all-in-one环境，主机名为：server-31,部署的openstack版本为mitaka。

openstack all-in-one搭建请阅读:[openstack-all-in-one30分钟快速搭建](http://eatpotato.github.io/2016/11/15/openstack-all-in-one/)  
ceph集群搭建请阅读: [ceph总结](http://eatpotato.github.io/2017/01/18/ceph-summary/)

### 配置openstack为ceph客户端

1.安装ceph-common包

```
yum install -y ceph-common
```
2.拷贝ceph.conf文件

将Ceph配置文件ceph.conf从ceph集群拷贝至server-31的/etc/ceph/目录下, 这个配置文件帮助客户端访问Ceph monitor和osd设备。检查ceph.conf的权限是644。

3.创建存储池

为Cinder、Glance、nova创建ceph存储池。可以使用任何可用的存储池，但建议为不同的openstack组件分别创建不同的存储池

```
ceph osd pool create glance 128
ceph osd pool create nova 128
ceph osd pool create cinder 128
```

![](/img/openstack-ceph/ceph-pools.png)

4.创建用户

为Cinder、Glance、nova创建ceph新用户

```
ceph auth get-or-create client.openstack mon 'allow r' ods 'allow class-read object_prefix rbd_children, allow rwx pool=cinder, allow rwx pool=nova, allow rwx pool=glance'
```

![](/img/openstack-ceph/ceph-add-user.png)


5.将生成的client密钥写入ceph.client.openstack.keyring文件

```
cat > /etc/ceph/ceph.client.openstack.keyring << EOF
[client.openstack]
key = AQBHaKJYdnRPMxAAqzd07gn/Nf0DLDqJNqF0Xg==
EOF
```

6.使用client.openstack用户访问ceph集群

```
ceph -s --name client.openstack
```

![](/img/openstack-ceph/ceph-openstack-user-test.png)

7.给libvirt设置密钥

说明：这一步骤的前提是你的ceph使用了cephx认证


7.1）创建临时的openstack.key,用于后面的Cinder和Nova配置

```
ceph auth get-key client.openstack | tee /etc/ceph/client.openstack.key
```

7.2）生成uuid

```
uuidgen
```

7.3) 创建密钥文件,并将uuid设置给它

```
cat > /etc/nova/secret.xml << EOF
<secret ephemeral='no' private='no'>
  <usage type='ceph'>
    <name>client.openstack secret</name>
  </usage>
  <uuid>7200aea0-2ddd-4a32-aa2a-d49f66ab554c</uuid>
</secret>
EOF
```

7.4) 定义（define）密钥文件

```
virsh secret-define --file /etc/nova/secret.xml
```

7.5) 在virsh里设置好我们最后一步生成的保密字符串值

```
virsh secret-set-value --secret 7200aea0-2ddd-4a32-aa2a-d49f66ab554c --base64 $(cat /etc/ceph/client.openstack.key)
```

7.6) 验证

```
virsh secret-list
```

![](/img/openstack-ceph/virsh-list.png)


7.7) 删除临时的密钥副本

rm -f /etc/ceph/client.openstack.key

### 配置ceph作为glance后端存储

1.登录到server-31(Openstack all-in-one节点,版本为mitaka)，编辑/etc/glance/glance-api.conf文件并作如下修改：

```
[DEFAULT]
show_image_direct_url = True
...
[glance_store]
stores = rbd
default_store = rbd
bd_store_chunk_size = 8
rbd_store_pool = glance
rbd_store_user = openstack
rbd_store_ceph_conf = /etc/ceph/ceph.conf
```

参数说明如下：

| 配置项 | 含义 | 默认值 |
| --- | --- | --- |
| rbd_pool | 保存rbd卷的ceph pool名称 | rbd |
| rbd_user | 访问 RBD 的用户的 ID，仅仅在使用 cephx 认证时使用 | none |
| rbd_ceph_conf | Ceph 配置文件的完整路径 | ''，表示使用 librados 的默认ceph 配置文件 |
| rbd_secret_uuid | rbd secret uuid ||
| rbd_store_chunk_size | 每个 RBD 卷实际上就是由多个对象组成的，因此用户可以指定一个对象的大小来决定对象的数量，默认是 8 MB | 8 |


2.重启Openstack Glance服务：

```
service openstack-glance-api restart
```

3.下载cirros镜像

```
wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
```

4.创建一个glance镜像

```
openstack image create "cirros" --file cirros-0.3.4-x86_64-disk.img  --disk-format qcow2 --container-format bare --public
```

5.查看镜像

```
[root@server-31 ~]# glance image-list
+--------------------------------------+-------------+
| ID                                   | Name        |
+--------------------------------------+-------------+
| e3c098eb-f363-481a-a05e-8b9be7909d37 | cirros      |
| 65fc4bb9-2563-4157-a635-f91688c3c841 | cirros_alt  |
| 4b2faee6-aea4-4fda-ae93-cb905f6ebd55 | cirros_vmdk |
+--------------------------------------+-------------+
```

6.去ceph池验证新添加的镜像

```
[root@server-31 ~]# rados -p glance ls --name client.openstack | grep -i e3c098eb-f363-481a-a05e-8b9be7909d37
rbd_id.e3c098eb-f363-481a-a05e-8b9be7909d37
```

### 配置ceph作为cinder后端存储

1.登录到server-31(Openstack all-in-one节点,版本为mitaka)，编辑/etc/cinder/cinder.conf文件并作如下修改：

```
[DEFAULT]
default_volume_type = BACKEND_1
enabled_backends = BACKEND_1
rbd_store_chunk_size = 4
...
[BACKEND_1]
volume_backend_name=BACKEND_1
volume_driver=cinder.volume.drivers.rbd.RBDDriver
rbd_ceph_conf=/etc/ceph/ceph.conf
rbd_user=openstack
rbd_pool=cinder
rbd_secret_uuid=7200aea0-2ddd-4a32-aa2a-d49f66ab554c
backend_host=rbd:cinder
rbd_store_chunk_size=4
```

参数说明如下：

| 配置项 | 含义 | 默认值 |
| --- | --- | --- |
| rbd_pool | 保存rbd卷的ceph pool名称 | rbd |
| rbd_user | 访问 RBD 的用户的 ID，仅仅在使用 cephx 认证时使用 | none |
| rbd_ceph_conf | Ceph 配置文件的完整路径 | ''，表示使用 librados 的默认ceph 配置文件 |
| rbd_secret_uuid | rbd secret uuid ||
| rbd_store_chunk_size | 每个 RBD 卷实际上就是由多个对象组成的，因此用户可以指定一个对象的大小来决定对象的数量，默认是 8 MB | 8 |



2.重启Openstack Cinder服务

```
service openstack-cinder-volume restart
```

3.测试

```
[root@server-31 ~]# cinder create --display-name ceph-colume01 1
+--------------------------------+--------------------------------------+
|            Property            |                Value                 |
+--------------------------------+--------------------------------------+
|          attachments           |                  []                  |
|       availability_zone        |                 nova                 |
|            bootable            |                false                 |
|      consistencygroup_id       |                 None                 |
|           created_at           |      2017-02-14T17:01:57.000000      |
|          description           |                 None                 |
|           encrypted            |                False                 |
|               id               | 1bdcaf27-2b7d-4595-9a06-ed008d9bc3ba |
|            metadata            |                  {}                  |
|        migration_status        |                 None                 |
|          multiattach           |                False                 |
|              name              |            ceph-colume01             |
|     os-vol-host-attr:host      |                 None                 |
| os-vol-mig-status-attr:migstat |                 None                 |
| os-vol-mig-status-attr:name_id |                 None                 |
|  os-vol-tenant-attr:tenant_id  |   e6d19ba2f0b243489697887751b13264   |
|       replication_status       |               disabled               |
|              size              |                  1                   |
|          snapshot_id           |                 None                 |
|          source_volid          |                 None                 |
|             status             |               creating               |
|           updated_at           |                 None                 |
|            user_id             |   5cb9ffb529124f838031b7d79f924950   |
|          volume_type           |                 None                 |
+--------------------------------+--------------------------------------+
```

```
[root@server-31 ~]# rados -p pool-6ee65215e53546f58ee3c79325e4a923 ls | grep -i 1bdcaf27-2b7d-4595-9a06-ed008d9bc3ba
rbd_id.volume-1bdcaf27-2b7d-4595-9a06-ed008d9bc3ba 
```

### 配置ceph作为nova后端存储

1.登录到server-31(Openstack all-in-one节点,版本为mitaka)，编辑/etc/nova/nova.conf文件并作如下修改：

```
[libvirt]
inject_partition=-2
images_type=rbd
images_rbd_pool=nova
images_rbd_ceph_conf =/etc/ceph/ceph.conf
rbd_user=openstack
rbd_secret_uuid=7200aea0-2ddd-4a32-aa2a-d49f66ab554c
```

| 配置项 | 含义 | 默认值 |
| --- | --- | --- |
| images_type | 其值可以设为下面几个选项中的一个：raw、qcow2、lvm、rbd、default | default |
| images_rbd_pool | 存放 vm 镜像文件的 RBD pool | rbd |
| images_rbd_ceph_conf | Ceph 配置文件的完整路径 | ''|
| rbd_user | rbd user id，仅仅在使用 cephx 认证时使用 | none |
| rbd_secret_uuid | rbd secret uuid ||



2.重启Openstack Nova服务:

```
service openstack-nova-compute restart
```

3.测试

说明:要在ceph中启动虚拟机，Glance镜像的格式必须为RAW。
 
3.1)将cirros镜像从QCOW转换成RAW

```
qemu-img convert -f qcow2 -O raw cirros-0.3.4-x86_64-disk.img cirros-0.3.4-x86_64-disk.raw
```

3.2使用RAW镜像创建Glance镜像：

```
root@server-31 ~]# glance image-create --name "cirros_raw_image" --file cirros-0.3.4-x86_64-disk.raw --disk-format raw --container-format bare --visibility public --progress
[=============================>] 100%
+------------------+----------------------------------------------------------------------------------+
| Property         | Value                                                                            |
+------------------+----------------------------------------------------------------------------------+
| checksum         | 56730d3091a764d5f8b38feeef0bfcef                                                 |
| container_format | bare                                                                             |
| created_at       | 2017-02-14T17:13:47Z                                                             |
| direct_url       | rbd://fe06e0f0-8b35-42d5-ae67-ec9e64f29aaa/glance                            |
|                  | /2e32ffb5-684f-4070-9de1-a71832134a2f/snap
|                   
| disk_format      | raw                                                                              |
| id               | 2e32ffb5-684f-4070-9de1-a71832134a2f                                             |
| min_disk         | 0                                                                                |
| min_ram          | 0                                                                                |
| name             | cirros_raw_image                                                                 |
| owner            | e6d19ba2f0b243489697887751b13264                                                 |
| protected        | False                                                                            |
| size             | 41126400                                                                         |
| status           | active                                                                           |
| tags             | []                                                                               |
| updated_at       | 2017-02-14T17:14:05Z                                                             |
| virtual_size     | None                                                                             |
| visibility       | public                                                                           |
+------------------+----------------------------------------------------------------------------------+
```

3.3)创建一个可引导的卷来从Ceph卷启动虚拟机:

```
[root@server-31 ~]# glance image-list
+--------------------------------------+------------------+
| ID                                   | Name             |
+--------------------------------------+------------------+
| 2e32ffb5-684f-4070-9de1-a71832134a2f | cirros_raw_image |
+--------------------------------------+------------------+
[root@server-31 ~]# cinder create --image-id 2e32ffb5-684f-4070-9de1-a71832134a2f --display-name cirros-ceph-boot-volume 1
+--------------------------------+--------------------------------------+
|            Property            |                Value                 |
+--------------------------------+--------------------------------------+
|          attachments           |                  []                  |
|       availability_zone        |                 nova                 |
|            bootable            |                false                 |
|      consistencygroup_id       |                 None                 |
|           created_at           |      2017-02-14T17:20:19.000000      |
|          description           |                 None                 |
|           encrypted            |                False                 |
|               id               | 14e76e18-a38d-4455-a357-f4e1db53e516 |
|            metadata            |                  {}                  |
|        migration_status        |                 None                 |
|          multiattach           |                False                 |
|              name              |       cirros-ceph-boot-volume        |
|     os-vol-host-attr:host      |         cinder@ssd-ceph#ssd          |
| os-vol-mig-status-attr:migstat |                 None                 |
| os-vol-mig-status-attr:name_id |                 None                 |
|  os-vol-tenant-attr:tenant_id  |   e6d19ba2f0b243489697887751b13264   |
|       replication_status       |               disabled               |
|              size              |                  1                   |
|          snapshot_id           |                 None                 |
|          source_volid          |                 None                 |
|             status             |               creating               |
|           updated_at           |      2017-02-14T17:20:19.000000      |
|            user_id             |   5cb9ffb529124f838031b7d79f924950   |
|          volume_type           |                 None                 |
+--------------------------------+--------------------------------------+
[root@server-31 ~]# nova boot --flavor 1 --block-device-mapping vda=14e76e18-a38d-4455-a357-f4e1db53e516 --image 2e32ffb5-684f-4070-9de1-a71832134a2f --nic net-id=0c688876-b427-4093-bb9d-331d39d5f2b9 vm_on_ceph
+--------------------------------------+---------------------------------------------------------+
| Property                             | Value                                                   |
+--------------------------------------+---------------------------------------------------------+
| OS-DCF:diskConfig                    | MANUAL                                                  |
| OS-EXT-AZ:availability_zone          |                                                         |
| OS-EXT-SRV-ATTR:host                 | -                                                       |
| OS-EXT-SRV-ATTR:hypervisor_hostname  | -                                                       |
| OS-EXT-SRV-ATTR:instance_name        | instance-00000002                                       |
| OS-EXT-STS:power_state               | 0                                                       |
| OS-EXT-STS:task_state                | scheduling                                              |
| OS-EXT-STS:vm_state                  | building                                                |
| OS-SRV-USG:launched_at               | -                                                       |
| OS-SRV-USG:terminated_at             | -                                                       |
| accessIPv4                           |                                                         |
| accessIPv6                           |                                                         |
| adminPass                            | PzoUXRaQXRE6                                            |
| config_drive                         |                                                         |
| created                              | 2017-02-14T17:38:59Z                                    |
| flavor                               | m1.tiny (1)                                             |
| hostId                               |                                                         |
| id                                   | 6c26ba25-c032-4170-a12b-53531b3340ac                    |
| image                                | cirros_raw_image (2e32ffb5-684f-4070-9de1-a71832134a2f) |
| key_name                             | -                                                       |
| metadata                             | {}                                                      |
| name                                 | vm_on_ceph                                              |
| os-extended-volumes:volumes_attached | [{"id": "14e76e18-a38d-4455-a357-f4e1db53e516"}]        |
| progress                             | 0                                                       |
| security_groups                      | default                                                 |
| status                               | BUILD                                                   |
| tenant_id                            | e6d19ba2f0b243489697887751b13264                        |
| updated                              | 2017-02-14T17:39:00Z                                    |
| user_id                              | 5cb9ffb529124f838031b7d79f924950                        |
+--------------------------------------+---------------------------------------------------------+
[root@server-31 ~]# nova list
+--------------------------------------+------------+--------+------------+-------------+--------------------+
| ID                                   | Name       | Status | Task State | Power State | Networks           |
+--------------------------------------+------------+--------+------------+-------------+--------------------+
| 6c26ba25-c032-4170-a12b-53531b3340ac | vm_on_ceph | ACTIVE | -          | Running     | public=172.24.5.12 |
+--------------------------------------+------------+--------+------------+-------------+--------------------+
```

## puppet中的配置项

下面会介绍如果openstack使用puppet搭建，那么为了集成ceph，需要修改哪些配置项信息

### puppet-glance

转发层的部分代码如下：

```
  case $backend {
    # 如果后端存储为本地文件
    'file': {
      include ::glance::backend::file
      $backend_store = ['file']
    }
    # 如果后端存储使用ceph
    'rbd': {
      class { '::glance::backend::rbd':
        rbd_store_user => 'openstack',
        rbd_store_pool => 'glance',
      }
      $backend_store = ['rbd']
      # make sure ceph pool exists before running Glance API
      Exec['create-glance'] -> Service['glance-api']
    }
    # 如果后端存储使用swift
    'swift': {
      Service<| tag == 'swift-service' |> -> Service['glance-api']
      $backend_store = ['swift']
      class { '::glance::backend::swift':
        swift_store_user                    => 'services:glance',
        swift_store_key                     => 'a_big_secret',
        swift_store_create_container_on_put => 'True',
        swift_store_auth_address            => "${::openstack_integration::config::proto}://127.0.0.1:5000/v2.0",
      }
    }
    default: {
      fail("Unsupported backend (${backend})")
    }
  }
```

下面仅贴出glance/backend/rbd.pp的相关内容：

```
  glance_api_config {
    'glance_store/rbd_store_ceph_conf':    value => $rbd_store_ceph_conf;
    'glance_store/rbd_store_user':         value => $rbd_store_user;
    'glance_store/rbd_store_pool':         value => $rbd_store_pool;
    'glance_store/rbd_store_chunk_size':   value => $rbd_store_chunk_size;
    'glance_store/rados_connect_timeout':  value => $rados_connect_timeout;
  }
...
  glance_api_config { 'glance_store/default_store': value => 'rbd'; }
...
  package { 'python-ceph':
    ensure => $package_ensure,
    name   => $::glance::params::pyceph_package_name,
  }
```


### puppet-cinder

转发层的部分代码如下：

```
  case $backend {
    'iscsi': {
      class { '::cinder::setup_test_volume':
        size => '15G',
      }
      cinder::backend::iscsi { 'BACKEND_1':
        iscsi_ip_address => '127.0.0.1',
      }
    }
    'rbd': {
      cinder::backend::rbd { 'BACKEND_1':
        rbd_user        => 'openstack',
        rbd_pool        => 'cinder',
        rbd_secret_uuid => '7200aea0-2ddd-4a32-aa2a-d49f66ab554c',
      }
      # make sure ceph pool exists before running Cinder API & Volume
      Exec['create-cinder'] -> Service['cinder-api']
      Exec['create-cinder'] -> Service['cinder-volume']
    }
    default: {
      fail("Unsupported backend (${backend})")
    }
  }
  class { '::cinder::backends':
    enabled_backends => ['BACKEND_1'],
  }
  cinder_type { 'BACKEND_1':
    ensure     => present,
    properties => ['volume_backend_name=BACKEND_1'],
  }
```

下面仅贴出cinder/backend/rbd.pp的相关内容：

```
  cinder_config {
    "${name}/volume_backend_name":              value => $volume_backend_name;
    "${name}/volume_driver":                    value => 'cinder.volume.drivers.rbd.RBDDriver';
    "${name}/rbd_ceph_conf":                    value => $rbd_ceph_conf;
    "${name}/rbd_user":                         value => $rbd_user;
    "${name}/rbd_pool":                         value => $rbd_pool;
    "${name}/rbd_max_clone_depth":              value => $rbd_max_clone_depth;
    "${name}/rbd_flatten_volume_from_snapshot": value => $rbd_flatten_volume_from_snapshot;
    "${name}/rbd_secret_uuid":                  value => $rbd_secret_uuid;
    "${name}/rados_connect_timeout":            value => $rados_connect_timeout;
    "${name}/rados_connection_interval":        value => $rados_connection_interval;
    "${name}/rados_connection_retries":         value => $rados_connection_retries;
    "${name}/rbd_store_chunk_size":             value => $rbd_store_chunk_size;
  }
  ...
  if $backend_host {
    cinder_config {
      "${name}/backend_host": value => $backend_host;
    }
  } else {
    cinder_config {
      "${name}/backend_host": value => "rbd:${rbd_pool}";
    }
  }
```

### puppet-nova

转发层的部分代码如下：

```
if $libvirt_rbd {
    class { '::nova::compute::rbd':
      libvirt_rbd_user        => 'openstack',
      libvirt_rbd_secret_uuid => '7200aea0-2ddd-4a32-aa2a-d49f66ab554c',
      libvirt_rbd_secret_key  => 'AQD7kyJQQGoOBhAAqrPAqSopSwPrrfMMomzVdw==',
      libvirt_images_rbd_pool => 'nova',
      rbd_keyring             => 'client.openstack',
      # ceph packaging is already managed by puppet-ceph
      manage_ceph_client      => false,
    }
    # make sure ceph pool exists before running nova-compute
    Exec['create-nova'] -> Service['nova-compute']
  }
```

下面仅贴出nova/compute/rbd.pp的相关内容：

```
nova_config {
    'libvirt/rbd_user': value => $libvirt_rbd_user;
  }

  if $libvirt_rbd_secret_uuid {
    nova_config {
      'libvirt/rbd_secret_uuid': value => $libvirt_rbd_secret_uuid;
    }

    file { '/etc/nova/secret.xml':
      content => template('nova/secret.xml-compute.erb'),
      require => Anchor['nova::config::begin'],
    }

    exec { 'get-or-set virsh secret':
      command => '/usr/bin/virsh secret-define --file /etc/nova/secret.xml | /usr/bin/awk \'{print $2}\' | sed \'/^$/d\' > /etc/nova/virsh.secret',
      unless  => "/usr/bin/virsh secret-list | grep ${libvirt_rbd_secret_uuid}",
      require => [File['/etc/nova/secret.xml'], Service['libvirt']],
    }

    if $libvirt_rbd_secret_key {
      $libvirt_key = $libvirt_rbd_secret_key
    } else {
      $libvirt_key = "$(ceph auth get-key ${rbd_keyring})"
    }
    exec { 'set-secret-value virsh':
      command => "/usr/bin/virsh secret-set-value --secret ${libvirt_rbd_secret_uuid} --base64 ${libvirt_key}",
      unless  => "/usr/bin/virsh secret-get-value ${libvirt_rbd_secret_uuid} | grep ${libvirt_key}",
      require => Exec['get-or-set virsh secret'],
      before  => Anchor['nova::config::end'],
      }
  }

  if $ephemeral_storage {
    nova_config {
      'libvirt/images_type':          value => 'rbd';
      'libvirt/images_rbd_pool':      value => $libvirt_images_rbd_pool;
      'libvirt/images_rbd_ceph_conf': value => $libvirt_images_rbd_ceph_conf;
    }
  } else {
    nova_config {
      'libvirt/images_rbd_pool':      ensure => absent;
      'libvirt/images_rbd_ceph_conf': ensure => absent;
    }
  }
```

可以看到上面的内容除了修改/etc/nova/nova.conf配置项意以外，还给libvirt设置了密钥，这些内容我们在上面的也有讲到具体的实现过程。

## 参考：
[Ceph Cookbook](http://product.dangdang.com/24001633.html)
