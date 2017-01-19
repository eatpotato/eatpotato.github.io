---
layout:     post
title:      ceph总结
date:       2017-01-18
author:     xue
catalog:    true
tags:
    - ceph
---

## ceph简介

Ceph是一个分布式存储系统，诞生于2004年，最早致力于开发下一代高性能分布式文件系统的项目。随着云计算的发展，ceph乘上了OpenStack的春风，进而成为了开源社区受关注较高的项目之一。


### ceph基本结构

![](/img/ceph/ceph-architecture.png)

自下向上，可以将Ceph系统分为四个层次:

1.基础存储系统RADOS（Reliable, Autonomic, Distributed Object Store，即可靠的、自动化的、分布式的对象存储）  
RADOS本身也是分布式存储系统，CEPH所有的存储功能都是基于RADOS实现,RADOS由大量的存储设备节点组成，每个节点拥有自己的硬件资源（CPU、内存、硬盘、网络），并运行着操作系统和文件系统。

2.基础库librados  
这一层的功能是对RADOS进行抽象和封装，并向上层提供API，以便直接基于RADOS（而不是整个Ceph）进行应用开发。

3.高层应用接口  
这一层包括了三个部分：RADOS GW（RADOS Gateway）、 RBD（Reliable Block Device）和Ceph FS（Ceph File System），其作用是在librados库的基础上提供抽象层次更高、更便于应用或客户端使用的上层接口。  
其中，RADOS GW是一个提供与Amazon S3和Swift兼容的RESTful API的gateway，以供相应的对象存储应用开发使用。RADOS GW提供的API抽象层次更高，但功能则不如librados强大。因此，开发者应针对自己的需求选择使用。  
RBD则提供了一个标准的块设备接口，常用于在虚拟化的场景下为虚拟机创建volume。目前，Red Hat已经将RBD驱动集成在KVM/QEMU中，以提高虚拟机访问性能。  
Ceph FS是一个POSIX兼容的分布式文件系统。由于还处在开发状态，因而Ceph官网并不推荐将其用于生产环境中

4.应用层  
这一层就是不同场景下对于Ceph各个应用接口的各种应用方式，例如基于librados直接开发的对象存储应用，基于RADOS GW开发的对象存储应用，基于RBD实现的云硬盘等等。

### Ceph基本组件

* Osd  
用于集群中所有数据与对象的存储。处理集群数据的复制、恢复、回填、再均衡。并向其他osd守护进程发送心跳，然后向Mon提供一些监控信息。
* Monitor  
监控整个集群的状态，维护集群的cluster MAP二进制表，保证集群数据的一致性。ClusterMAP描述了对象块存储的物理位置，以及一个将设备聚合到物理位置的桶列表。
* MDS(可选)  
为Ceph文件系统提供元数据计算、缓存与同步。在ceph中，元数据也是存储在osd节点中的，mds类似于元数据的代理缓存服务器。MDS进程并不是必须的进程，只有需要使用CEPHFS时，才需要配置MDS节点。

### ceph数据的存储过程
Ceph系统中的寻址流程如下图所示：

![](/img/ceph/ceph-addressing.png)

无论使用哪种存储方式（对象、块、挂载），存储的数据都会被切分成对象（Objects）。Objects size大小可以由管理员调整，通常为2M或4M。每个对象都会有一个唯一的OID，由ino与ono生成。ino即是文件的File ID，用于在全局唯一标示每一个文件，而ono则是分片的编号。比如：一个文件FileID为A，它被切成了两个对象，一个对象编号0，另一个编号1，那么这两个文件的oid则为A0与A1。Oid的好处是可以唯一标示每个不同的对象，并且存储了对象与文件的从属关系。  

但是对象并不会直接存储进OSD中，因为对象的size很小，在一个大规模的集群中可能有几百到几千万个对象。这么多对象光是遍历寻址，速度都是很缓慢的；并且如果将对象直接通过某种固定映射的哈希算法映射到osd上，当这个osd损坏时，对象无法自动迁移至其他osd上面（因为映射函数不允许）。为了解决这些问题，ceph引入了归置组的概念，即PG。  

PG是一个逻辑概念，我们linux系统中可以直接看到对象，但是无法直接看到PG。它在数据寻址时类似于数据库中的索引：每个对象都会固定映射进一个PG中，所以当我们要寻找一个对象时，只需要先找到对象所属的PG，然后遍历这个PG就可以了，无需遍历所有对象。而且在数据迁移时，也是以PG作为基本单位进行迁移，ceph不会直接操作对象。  

对象映射进PG的方式：使用静态hash函数对OID做hash取出特征码，用特征码与PG的数量去模，得出PGID。  
最后PG会根据管理员设置的副本数量进行复制，然后通过crush算法存储到不同的OSD节点上。

## ceph-deploy快速安装
说明，此例中一个mon节点，两个osd节点，hostname分别为：test-ceph-1,test-ceph-2,test-ceph-3   
mons节点必须能够通过 SSH 无密码地访问各 Ceph 节点  
配置好相应的源，国内推荐使用有云UDS的源，创建/etc/yum.repos.d/ceph.repo如下：

```
[ceph]
name=UDS Packages for CentOS 7
baseurl=http://uds.ustack.com/repo/Azeroth/el7/
enabled=1
gpgcheck=0
priority=1
[ceph-noarch]
name=UDS Packages for CentOS 7
baseurl=http://uds.ustack.com/repo/Azeroth/el7/
enabled=1
gpgcheck=0
priority=1
[ceph-source]
name=UDS Packages for CentOS 7
baseurl=http://uds.ustack.com/repo/Azeroth/el7/
enabled=1
gpgcheck=0
priority=1
```

安装 ceph-deploy：

```
yum install ceph-deploy
```
如果安装了firewalld,那么需要：

```
firewall-cmd --zone=public --add-port=6789/tcp --permanent
```


 
若使用 iptables,要开放 Ceph Monitors 使用的 6789 端口和 OSD 使用的 6800:7300 端口范围:
 
```
iptables -A INPUT -i {iface} -p tcp -s {ip-address}/{netmask} --dport 6789 -j ACCEPT
/sbin/service iptables save
```

在 CentOS 和 RHEL 上， SELinux 默认为 Enforcing 开启状态。为简化安装，我们建议把 SELinux 设置为 Permissive 或者完全禁用，也就是在加固系统配置前先确保集群的安装、配置没问题。用下列命令把 SELinux 设置为 Permissive ：

```
setenforce 0
```

在mon节点上，用 ceph-deploy 执行如下步骤：

1.创建集群

```
ceph-deploy new test-ceph-1
```

2.把 Ceph 配置文件里的默认副本数从 3 改成 2 ，这样只有两个 OSD 也可以达到 active + clean 状态。把下面这行加入/etc/ceph/ceph.conf的[global]段：

```
osd_pool_default_size = 2
```

3.安装 Ceph 

```
ceph-deploy install test-ceph-1 test-ceph-2 test-ceph-3
```

4.配置初始 monitor(s)、并收集所有密钥：

```
ceph-deploy mon create-initial
```

完成上述操作后，当前目录里应该会出现这些密钥环：
{cluster-name}.client.admin.keyring
{cluster-name}.bootstrap-osd.keyring
{cluster-name}.bootstrap-mds.keyring
{cluster-name}.bootstrap-rgw.keyring


5.添加两个 OSD 
登录test-ceph-2:

```
mkdir /var/local/osd0
```

登录test-ceph-3:

```
mkdir /var/local/osd1
```

然后，从mon节点执行 ceph-deploy 来准备 OSD:

```
ceph-deploy osd prepare test-ceph-2:/var/local/osd0 test-ceph-3:/var/local/osd1
```

激活 OSD :

```
ceph-deploy osd activate test-ceph-2:/var/local/osd0 test-ceph-3:/var/local/osd1
```

6.用 ceph-deploy 把配置文件和 admin 密钥拷贝到mon节点和osd节点，这样你每次执行 Ceph 命令行时就无需指定 monitor 地址和 ceph.client.admin.keyring 了:

```
ceph-deploy admin server-52 test-ceph-1 test-ceph-2 test-ceph-3
```

7.确保你对 ceph.client.admin.keyring 有正确的操作权限。

```
chmod +r /etc/ceph/ceph.client.admin.keyring
```

8.验证

![](/img/ceph/ceph-test-health.png)

## ceph手动部署

mon节点为test-ceph-1，osd节点为test-ceph-2，test-ceph-3  
mons节点必须能够通过 SSH 无密码地访问各 Ceph 节点   
配置好相应的ceph源  
密钥下载：  
不管你是用仓库还是手动下载，你都需要用密钥校验软件包。如果你没有密钥，就会收到安全警告。有两个密钥：一个用于发布（常用）、一个用于开发（仅适用于程序员和 QA ）  
执行下列命令安装 release.asc 密钥：

```
sudo rpm --import 'https://download.ceph.com/keys/release.asc'
```

执行下列命令安装 autobuild.asc 密钥（仅对 QA 和开发者）：

```
sudo rpm --import 'https://download.ceph.com/keys/autobuild.asc'
```

部署步骤如下：

1.mon节点和osd节点安装 yum-plugin-priorities、ceph和依赖包：

```
yum install yum-plugin-priorities snappy leveldb gdisk python-argparse gperftools-libs ceph 
```

2.mon节点部署

2.1登录到监视器节点， 确保保存 Ceph 配置文件的目录存在
2.2创建 Ceph 配置文件， Ceph 默认使用 ceph.conf 
2.3给集群分配惟一 ID （即 fsid ）,并把此 ID 写入 Ceph 配置文件

```
uuidgen
```

写入ceph.conf

```
[global]
fsid = {UUID}
```

2.4把初始监视器写入 Ceph 配置文件

```
mon_initial_members = test-ceph-1
```

2.5初始监视器的 IP 地址写入 Ceph 配置文件

```
mon_host = 10.0.86.23
```

2.6为此集群创建密钥环、并生成监视器密钥

```
ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
```

2.7生成管理员密钥环，生成 client.admin 用户并加入密钥环


```
ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --set-uid=0 --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow'

```

2.8把 client.admin 密钥加入 ceph.mon.keyring 

```
ceph-authtool /tmp/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
```

2.9用规划好的主机名、对应 IP 地址、和 FSID 生成一个监视器图，并保存为 /tmp/monmap

```
monmaptool --create --add test-ceph-1 10.0.86.23 --fsid {UUID} /tmp/monmap
```

2.10在监视器主机上分别创建数据目录。

```

mkdir /var/lib/ceph/mon/ceph-test-ceph-1
```

2.11创建一个boot引导启动osd的key

```
mkdir -p /var/lib/ceph/bootstrap-osd/
ceph-authtool -C /var/lib/ceph/bootstrap-osd/ceph.keyring
```

2.12用监视器图和密钥环组装守护进程所需的初始数据。

```
ceph-mon --mkfs -i test-ceph-1 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
```

2.13仔细斟酌 Ceph 配置文件，公共的全局配置包括这些：

```
[global]
fsid = xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
mon_initial_members = test-ceph-1
mon_host = 10.0.86.23
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
auth_supported = cephx
osd_pool_default_size = 2
```
把监视器节点的/etc/ceph/ceph.conf 和ceph.client.admin.keyring 拷贝到test-ceph-2,test-ceph3,这一步不要忘记！！！

2.14建一个空文件 done ，表示监视器已创建、可以启动了

```
touch /var/lib/ceph/mon/ceph-test-ceph-1/done
touch /var/lib/ceph/mon/ceph-test-ceph-1/sysvinit
```

2.15启动监视器

```
service ceph start mon.test-ceph-1
```

3.osd节点部署
注意：如果osd盘不是本地目录，那么请忽略3.1-3.3，直接执行3.4即可。
3.1创建本地目录当做osd盘  
在 test-ceph-2 上执行：

```
mkdir /var/local/osd0
```

在 test-ceph-3 上执行：

```
mkdir /var/local/osd1
```

3.2准备OSD  
在test-ceph-2、test-ceph-3上执行：

```
ceph osd create
```
这条命令会输出创建的osd number


3.3创建软链接。

在 test-ceph-2 上执行：

```
ln -s /var/local/osd0 /var/lib/ceph/osd/ceph-0
```

在 test-ceph-3 上执行：

```
ln -s /var/local/osd0 /var/lib/ceph/osd/ceph-0
```

3.4如果osd不是本地目录，需要以下操作：

在 test-ceph-2 上执行：  
根据设备名替换下面的/dev/vdb

```
mkfs.xfs -f  /dev/vdb
mkdir -p /var/lib/ceph/osd/ceph-0
mount /dev/vdb /var/lib/ceph/osd/ceph-0
mount -o remount,user_xattr /var/lib/ceph/osd/ceph-0
vi /etc/fstab
>>
/dev/vdb /var/lib/ceph/osd/ceph-0 xfs defaults 0 0
/dev/vdb /var/lib/ceph/osd/ceph-0 xfs remount,user_xattr 0 0
<<
```

在 test-ceph-3 上执行：  
根据设备名替换下面的/dev/vdc

```
mkfs.xfs -f  /dev/vdc
mkdir -p /var/lib/ceph/osd/ceph-1
mount /dev/vdc /var/lib/ceph/osd/ceph-1
mount -o remount,user_xattr /var/lib/ceph/osd/ceph-0
vi /etc/fstab
>>
/dev/vdc /var/lib/ceph/osd/ceph-0 xfs defaults 0 0
/dev/vdc /var/lib/ceph/osd/ceph-0 xfs remount,user_xattr 0 0
<<
```

3.5初始化 OSD 数据目录：  
在test-ceph-2、test-ceph-3上执行：

```
ceph-osd -i {osd-num} --mkfs --mkkey
```

3.6注册此 OSD 的密钥  
在test-ceph-2、test-ceph-3上执行：  
```
ceph auth add osd.{osd-num} osd 'allow *' mon 'allow profile osd' -i /var/lib/ceph/osd/{cluster-name}-{osd-num}/keyring

```

3.7把此节点加入 CRUSH 图  
在 test-ceph-2 上执行： 

```
ceph osd crush add-bucket test-ceph-2 host
```

在 test-ceph-3 上执行： 

```
ceph osd crush add-bucket test-ceph-3 host
```

3.8把此 Ceph 节点放入 default 根下  
在 test-ceph-2 上执行：

```
ceph osd crush move test-ceph-2 root=default
```

在 test-ceph-3 上执行：

```
ceph osd crush move test-ceph-2 root=default
```

3.9.分配权重、重新编译、注入集群

在 test-ceph-2 上执行:  
1.0表示权重，需要根据磁盘大小自行调整权重  

```
ceph osd crush add osd.0 1.0 host=test-ceph-2
```

在 test-ceph-3 上执行:

```
ceph osd crush add osd.0 1.0 host=test-ceph-3
```


3.10.创建一个空文件：  
在test-ceph-2、test-ceph-3上执行：  

```
touch /var/lib/ceph/osd/{cluster-name}-{osd-num}/sysvinit

```

3.11.用 sysvinit 启动  
在test-ceph-2、test-ceph-3上执行： 

```
service ceph start osd.{osd-num}
```

## puppet-ceph部署

目前部署方式可以有Puppet和Ceph-deploy, 目前生产环境中主要使用Puppet来进行部署。Ceph-deploy的部署方式主要是为了满足快速验证的需求。一般推荐在生产环境中使用Puppet来部署，这样方便后续环境的维护。

在puppet master module目录下下载[puppet-ceph](https://github.com/openstack/puppet-ceph/tree/stable/hammer) 

openstack/puppet-ceph 使用ceph版本为hammer
 
部署节点同上，mon节点为test-ceph-1，osd节点为test-ceph-2，test-ceph-3 

各节点加载的类：

```
node /^test-ceph-1$/ {
$ceph_pools = ['test']
ceph::pool { $ceph_pools: }
class { '::ceph::profile::mon': }
}
node /^test-ceph-[2-3]$/ {
class { '::ceph::profile::osd': }
}
```

传的hieradata:


common/ceph.yaml:

```
---
######## Ceph
ceph::profile::params::release: 'hammer'

######## Ceph.conf
ceph::profile::params::fsid: '4b5c8c0a-ff60-454b-a1b4-9747aa737d19'
ceph::profile::params::authentication_type: 'cephx'
ceph::profile::params::mon_initial_members: 'test-ceph-1'
ceph::profile::params::mon_host: '10.0.86.23:6789'
ceph::profile::params::osd_pool_default_size: '2'

######## Keys
ceph::profile::params::mon_key: 'AQATGHJTUCBqIBAA7M2yafV1xctn1pgr3GcKPg==' 
ceph::profile::params::client_keys:
  'client.admin':
    secret: 'AQATGHJTUCBqIBAA7M2yafV1xctn1pgr3GcKPg=='
    mode: '0600'
    cap_mon: 'allow *'
    cap_osd: 'allow *'
    cap_mds: 'allow *'
  'client.bootstrap-osd':
    secret: 'AQATGHJTUCBqIBAA7M2yafV1xctn1pgr3GcKPg=='
    keyring_path: '/var/lib/ceph/bootstrap-osd/ceph.keyring'
    cap_mon: 'allow profile bootstrap-osd'
```

test-ceph-2.yaml:

```
ceph::profile::params::osds:
  '/dev/sdb':
    journal: ''
```

test-ceph-3.yaml:

```
ceph::profile::params::osds:
  '/dev/sdc':
    journal: ''
```

puppet会执行 ceph-disk prepare /dev/sdc ，如果journal为空，它会把自动把这块盘分成两个分区,一个为ceph data ,一个为ceph journal  journal分区大小默认为5G，剩下的都分给ceph data.

Journal的作用类似于mysql innodb引擎中的事物日志系统。当有突发的大量写入操作时，ceph可以先把一些零散的，随机的IO请求保存到缓存中进行合并，然后再统一向内核发起IO请求。journal的io是非常密集的,很大程度上也损耗了硬件的io性能，所以通常在生产环境中，使用ssd来单独存储journal文件以提高ceph读写性能。

journal也可以使用单独的数据盘，只需要在hieradata中传递相应的设备名即可。

openstack/puppet-ceph 传osds参数不支持wwn的方式,因为ceph-disk当前不支持使用wwn来作为磁盘标识的输入参数

## puppet执行过程分析
创建mon的大致过程如下：

1.安装包

```
package { $::ceph::params::packages :
    ensure => $ensure,
    tag    => 'ceph'
  }
```

2.是否开启认证

```
# [*authentication_type*] Activate or deactivate authentication
#   Optional. Default to cephx.
#   Authentication is activated if the value is 'cephx' and deactivated
#   if the value is 'none'. If the value is 'cephx', at least one of
#   key or keyring must be provided.
if $authentication_type == 'cephx' {
      ceph_config {
        'global/auth_cluster_required': value => 'cephx';
        'global/auth_service_required': value => 'cephx';
        'global/auth_client_required':  value => 'cephx';
        'global/auth_supported':        value => 'cephx';
      }
```

3.生成mon密钥

```
cat > ${keyring_path} << EOF
[mon.]
key = ${key}
caps mon = \"allow *\"
EOF
chmod 0444 ${keyring_path}
```

4.生成/etc/ceph/ceph.client.admin.keyring文件

```
touch /etc/ceph/${cluster_name}.client.admin.keyring
```

5.初始化monitor服务，创建done,sysvinit空文件

```
mon_data=\$(ceph-mon ${cluster_option} --id ${id} --show-config-value mon_data) 
if [ ! -d \$mon_data ] ; then
  mkdir -p \$mon_data
  if ceph-mon ${cluster_option} \
        --mkfs \
        --id ${id} \
        --keyring ${keyring_path} ; then
    touch \$mon_data/done \$mon_data/${init} \$mon_data/keyring
  else
    rm -fr \$mon_data
  fi
fi
```

6.启动mon服务：

```
service ceph start mon.test-ceph-xue-1
```

创建osd的大致过程如下：

1.安装包

```
package { $::ceph::params::packages :
    ensure => $ensure,
    tag    => 'ceph'
  }
  
```

2.是否开启认证

```
# [*authentication_type*] Activate or deactivate authentication
#   Optional. Default to cephx.
#   Authentication is activated if the value is 'cephx' and deactivated
#   if the value is 'none'. If the value is 'cephx', at least one of
#   key or keyring must be provided.
if $authentication_type == 'cephx' {
      ceph_config {
        'global/auth_cluster_required': value => 'cephx';
        'global/auth_service_required': value => 'cephx';
        'global/auth_client_required':  value => 'cephx';
        'global/auth_supported':        value => 'cephx';
      }
```

3.创建keyring file

```
if ! defined(File[$keyring_path]) {
    file { $keyring_path:
      ensure  => file,
      owner   => $user,
      group   => $group,
      mode    => $mode,
      require => Package['ceph'],
    }
  }
```

4.生成管理员密钥环，生成 client.admin 用户并加入密钥环

```
ceph-authtool \$NEW_KEYRING --name '${name}' --add-key '${secret}' ${caps}
```

5.把 client.admin 密钥加入 ceph.mon.keyring

```
ceph ${cluster_option} ${inject_id_option} ${inject_keyring_option} auth import -i ${keyring_path}"
```

6.ceph 0.94版本下禁用udev rules,否则，可能会导致ceph-disk activate失败

```
mv -f ${udev_rules_file} ${udev_rules_file}.disabled && udevadm control --reload
```

7.使用ceph-disk prepare 做预处理  
预处理用作 Ceph OSD 的目录、磁盘。它会创建 GPT 分区、给分区打上 Ceph 风格的 uuid 标记、创建文件系统、把此文件系统标记为已就绪、使用日志磁盘的整个分区并新增一分区。可单独使用，也可由 ceph-deploy 用。

```
if ! test -b ${data} ; then
mkdir -p ${data}
fi
ceph-disk prepare ${cluster_option} ${data} ${journal}
udevadm settle
```



8.激活 Ceph OSD  
激活 Ceph OSD 。先把此卷挂载到一临时位置，分配 OSD 惟一标识符（若有必要），重挂载到正确位置 

```
ceph-disk activate ${data}
```


## ceph常见命令



查看ceph集群的运行状态信息:

```
root@test-ceph-1 ~]# ceph -s
    cluster b60db21f-6735-4909-a0bb-c550b4659bfc
     health HEALTH_OK
     monmap e1: 1 mons at {test-ceph-xue-1=30.20.10.9:6789/0}
            election epoch 2, quorum 0 test-ceph-xue-1
     osdmap e10: 1 osds: 1 up, 1 in
      pgmap v642: 256 pgs, 4 pools, 0 bytes data, 0 objects
            6865 MB used, 13603 MB / 20469 MB avail
                 256 active+clean
  
```

         
### auth命令
1.查看认证状态

```
[root@test-ceph-1 ~]# ceph auth list
installed auth entries:

osd.0
	key: AQDRPn9YgQESIhAA/n4KnNPlS3OXwX5W5c2s9w==
	caps: [mon] allow profile osd
	caps: [osd] allow *
client.admin
	key: AQATGHJTUCBqIBAA7M2yafV1xctn1pgr3GcKPg==
	caps: [mds] allow *
	caps: [mon] allow *
	caps: [osd] allow *
client.bootstrap-mds
	key: AQATGHJTUCBqIBAA7M2yafV1xctn1pgr3GcKPg==
	caps: [mon] allow profile bootstrap-mds
client.bootstrap-osd
	key: AQATGHJTUCBqIBAA7M2yafV1xctn1pgr3GcKPg==
	caps: [mon] allow profile bootstrap-osd
```

2.添加指定实例的认证信息

```
# 使用方法: ceph auth get-or-create 实例名称 对象1 权限1 对象2 权限2
[root@test-ceph-1 ~]# ceph auth get-or-create client.admin mds 'allow *' osd 'allow *' mon 'allow *'

```

3.删除指定实例及其认证信息

```
# 使用方法: ceph auth del 实例名称
ceph auth del client.bootstrap-mds
```

### pool命令

1.打印pool列表

```
ceph osd lspools
```

2.创建pool

通常在创建pool之前，需要覆盖默认的pg_num，官方推荐：

若少于5个OSD， 设置pg_num为128。  
5~10个OSD，设置pg_num为512。  
10~50个OSD，设置pg_num为4096。  
超过50个OSD，可以参考[pgcalc](http://ceph.com/pgcalc/)计算。


```
ceph osd pool create {pool-name} {pg-num} [{pgp-num}] [replicated] \
     [crush-ruleset-name] [expected-num-objects]
ceph osd pool create {pool-name} {pg-num}  {pgp-num}   erasure \
     [erasure-code-profile] [crush-ruleset-name] [expected_num_objects]
```

创建一个test-pool，pg_num为128：


```
ceph osd pool create test-pool 128
```

3.重命名pool

```
ceph osd pool rename test-pool test-pool-new
```

4.删除pool

删除一个pool会同时清空pool的所有数据，因此非常危险。(和rm -rf /类似）。因此删除pool时ceph要求必须输入两次pool名称，同时加上--yes-i-really-really-mean-it选项。


```
ceph osd pool delete test-pool test-pool  --yes-i-really-really-mean-it
```

5.显示所有pool详细信息

```
rados df
```

6.元数据信息

通过以下语法设置pool的元数据：

```
ceph osd pool set {pool-name} {key} {value}
```

比如设置pool的冗余副本数量为3:

```
ceph osd pool set test-pool size 3
```

通过get操作能够获取pool的配置值,比如获取当前pg_num：

```
ceph osd pool get test-pool pg_num
```

获取当前副本数:

```
ceph osd pool get test-pool size
```

### mon命令

1.显示mon的状态信息

```
[root@test-ceph-1 ~]# ceph mon stat
e1: 1 mons at {test-ceph-xue-1=30.20.10.9:6789/0}, election epoch 2, quorum 0 test-ceph-xue-1
```

2.格式化输出mon map信息

```
[root@test-ceph-1 ~]# ceph mon dump
dumped monmap epoch 1
epoch 1
fsid b60db21f-6735-4909-a0bb-c550b4659bfc
last_changed 0.000000
created 0.000000
0: 30.20.10.9:6789/0 mon.test-ceph-1
```

3.删除当前集群中指定的mon

```
ceph mon remove test-ceph-1
```

### osd命令

1.显示OSD map的汇总信息

```
[root@test-ceph-1 ~]# ceph osd stat
     osdmap e10: 1 osds: 1 up, 1 in
```

2.显示OSD tree

```
[root@test-ceph-1 ~]# ceph osd tree
ID WEIGHT  TYPE NAME                UP/DOWN REWEIGHT PRIMARY-AFFINITY
-1 0.01999 root default
-2 0.01999     host test-ceph-1
 0 0.01999         osd.0                 up  1.00000          1.00000
```

3.显示OSD的延迟汇总信息

```
ceph osd perf
```

4.查看OSD的使用率

```
ceph osd df
```

5.将指定OSD置为down状态

```
ceph osd down {osd-num}
```

6.将指定OSD置为out状态

```
ceph osd out {osd-number}
```


## 部署ceph all-in-one遇到的问题
在测试环境中，我用openstack/puppet-ceph 部署的ceph all-in-one遇到了问题：

我使用了三块ssd作为osd的三块盘，副本数为3，但是部署完有很多pg未处于active+clean 的状态

![](/img/ceph/ceph-allinone-problem.png)

后来发现问题出在这里：

![](/img/ceph/ceph-where-is-problem.png)

因为只有一个主机，bucket （桶）类型只有一个host,我设置的副本数是3，副本策略默认是type: host。


解决方法：

1.获取当前CRUSH map文件

```
ceph osd getcrushmap -o crushmap
```

2.反编译成可编辑文件

```
crushtool -d crushmap -o crushmap.txt
```

3.编辑文件，将type类型改为osd 

```
vim crushmap.txt
```

4.重新编译

```
crushtool -c crushmap.txt -o newcrushmap
````


5.往集群中注入CRUSH map

```
ceph osd setcrushmap -i newcrushmap
```

6.重启mon和osd服务

```
```

```
```
