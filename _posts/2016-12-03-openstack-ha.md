---
layout:     post
title:      openstack高可用
date:       2016-12-03
author:     xue
catalog:    true
tags:
    - openstack
---

# openstack ha
 
## 基础知识
  
### 高可用 （High Availability，简称 HA）
&emsp;&emsp;高可用性是指提供在本地系统单个组件故障情况下，能继续访问应用的能力，无论这个故障是业务流程、物理设施、IT软/硬件的故障。最好的可用性， 就是你的一台机器宕机了，但是使用你的服务的用户完全感觉不到。你的机器宕机了，在该机器上运行的服务肯定得做故障切换（failover），切换有两个维度的成本：RTO （Recovery Time Objective）和 RPO（Recovery Point Objective）。RTO 是服务恢复的时间，最佳的情况是 0，这意味着服务立即恢复；最坏是无穷大意味着服务永远恢复不了；RPO 是切换时向前恢复的数据的时间长度，0 意味着使用同步的数据，大于 0 意味着有数据丢失，比如 ” RPO = 1 天“ 意味着恢复时使用一天前的数据，那么一天之内的数据就丢失了。因此，恢复的最佳结果是 RTO = RPO = 0，但是这个太理想，或者要实现的话成本太高，全球估计 Visa 等少数几个公司能实现，或者几乎实现。

&emsp;&emsp;对 HA 来说，往往使用共享存储，这样的话，RPO =0 ；同时往往使用 Active/Active （双活集群） HA 模式来使得 RTO 几乎0，如果使用 Active/Passive 模式的 HA 的话，则需要将 RTO 减少到最小限度。HA 的计算公式是[ 1 - (宕机时间)/（宕机时间 + 运行时间）]，我们常常用几个 9 表示可用性：

* 2 个9：99% = 1% * 365 = 3.65 * 24 小时/年 = 87.6 小时/年的宕机时间
* 4 个9: 99.99% = 0.01% * 365 * 24 * 60 = 52.56 分钟/年
* 5 个9：99.999% = 0.001% * 365 = 5.265 分钟/年的宕机时间，也就意味着每次停机时间在一到两分钟。
* 11 个 9：几乎就是几年才宕机几分钟。 据说 AWS S3 的设计高可用性就是 11 个 9。 

### 服务的分类

HA 将服务分为两类：

有状态服务：后续对服务的请求依赖于之前对服务的请求。  
OpenStack有状态的服务包括OpenStack数据库和消息队列。  
无状态服务：对服务的请求之间没有依赖关系，是完全独立的。  
OpenStack无状态的服务包括nova-api、nova-conductor、glance-api、keystone-api、neutron-api、nova-scheduler。   

### HA 的种类
HA 需要使用冗余的服务器组成集群来运行负载，包括应用和服务。这种冗余性也可以将 HA 分为两类：

Active/Passive HA：集群只包括两个节点简称主备。在这种配置下，系统采用主和备用机器来提供服务，系统只在主设备上提供服务。在主设备故障时，备设备上的服务被启动来替代主设备提供的服务。典型地，可以采用 CRM 软件比如 Pacemaker 来控制主备设备之间的切换，并提供一个虚机 IP 来提供服务。  
Active/Active HA：集群只包括两个节点时简称双活，包括多节点时成为多主（Multi-master）。在这种配置下，系统在集群内所有服务器上运行同样的负载。以数据库为例，对一个实例的更新，会被同步到所有实例上。这种配置下往往采用负载均衡软件比如 HAProxy 来提供服务的虚拟 IP。

1、主动/被动（Active/Passive）配置
主备概念，主节点出问题时，备节点顶上。一般用VIP实现，使用Pacemaker和Corosync。

2、主动/主动（Active/Active）配置
无状态使用VIP进行负载平衡，可以使用HAProxy软件。

##Openstack controller ha实现方案
本次的ha测试环境采用五台物理机,其中三台为controller节点，一台为compute ,一台为network
controller节点上,整体的架构图如下：  
![](/img/openstack-controller-ha.jpg)  
三台物理服务器组成 pacemaker 集群，创建多个虚机，安装各种应用，具体如下：

| Service | Process | Mode  |  HA stragegy |
| --- | --- | --- | --- |
| keystone | httpd | AA | 负载均衡+多实例 |
| glance | openstack-glance-api  |  AA | 负载均衡+多实例 |
| glance | openstack-glance-registry | AA | 负载均衡+多实例 |
| nova | openstack-nova-api | AA | 负载均衡+多实例 |
| nova | openstack-nova-conductor | AA | AMQP+多实例 |
| nova | openstack-nova-scheduler | AA | AMQP+多实例 |
| nova | openstack-nova-consoleauth | AA | AMQP+多实例 |
| nova | openstack-nova-novncproxy | AA | 负载均衡+多实例 |
| neutron | neutron-server | AA | 负载均衡+多实例 |
| cinder | openstack-cinder-api | AA | 负载均衡+多实例 |
| cinder | openstack-cinder-scheduler | AA | AMQP+多实例 |
| cinder | openstack-cinder-volume | AA | 主备+pacemaker切换 |
| mysql | Mariadb-server | AA | Galera cluster |
| rabbitmq | rabbitmq-server | AA | cluster + mirror queue |
| haproxy | haproxy | AP | 多实例+pacemaker切换VIP |
| memcache | memcached | AA | 负载均衡+多实例 |

具体的配置过程如下：

```
# 在所有lb节点，安装pacemaker组件，启动pcsd服务，并配置hacluster账户
$ yum install -y pacemaker pcs psmisc policycoreutils-python
$ systemctl start pcsd.service
$ systemctl enable pcsd.service
$ echo <some_password> | passwd hacluster --stdin

# 在任意一个节点初始化lb集群
$ pcs cluster auth ip1 ip2 ip3
Username: hacluster
Password:
server-33: Authorized
server-34: Authorized
server-35: Authorized
$ pcs cluster setup --name lb_cluster ip1 ip2 ip3

# 在所有lb节点，启动corosync & pacemaker服务
$ systemctl start corosync.service
$ systemctl enable corosync.service
$ systemctl start pacemaker.service
$ systemctl enable pacemaker.service


# 配置lb集群的特性
$ pcs property set stonith-enabled=false
$ pcs property set no-quorum-policy=ignore
$ pcs property set start-failure-is-fatal=false
$ pcs resource defaults resource-stickiness=10


# 创建vip与haproxy资源，并添加限制，确保p_vip运行在haproxy服务正常的节点
$ pcs resource create p_vip ocf:heartbeat:IPaddr2 ip=ip4 cidr_netmask=24 op monitor interval=2s
$ pcs resource create p_haproxy systemd:haproxy op monitor interval=2s --clone
$ pcs constraint colocation add p_vip with p_haproxy-clone --force
$ pcs resource meta p_vip migration-threshold=3 failure-timeout=60s
```




```
# 下面是pacemaker+corosync+galera的安装过程
# 在此之前先创建集群，创建方法参考pacemaker+corosync+haproxy
# 安装必要的包
$ yum install -y MariaDB-server MariaDB-client percona-xtrabackup socat #MariaDB的版本不要是5.0.x  ，使用10.1.x,否则会报错
$ yum install -y pacemaker pcs psmisc policycoreutils-python
$ yum install -y openstack-resource-agent


# 创建用户wsrep_sst
$ mysql -e 'grant all on *.* to "wsrep_sst"@"%" identified by "DJ1JAs8z"'
$ mysql -e 'grant all on *.* to "wsrep_sst"@"localhost" identified by "DJ1JAs8z"'


# 写入配置 /etc/my.cnf.d/wsrep.cnf，注意后缀是cnf，完成garela集群的配置
[mysqld]
binlog_format=row
wsrep_on=ON
wsrep_cluster_address="gcomm://ip1,ip2,ip3?pc.wait_prim=no"  # 三台 MariaDB 的 IP
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
wsrep_cluster_name="openstack"
wsrep_slave_threads=4
wsrep_sst_method=xtrabackup-v2
wsrep_sst_auth=wsrep_sst:DJ1JAs8z
wsrep_node_address=ip1      # 本机的地址
[xtrabackup]
parallel=2
[sst]
streamfmt=xbstream
transferfmt=socat
sockopt=,nodelay,sndbuf=1048576,rcvbuf=1048576


# 配置pacemaker管理Galera集群
$ pcs resource create p_mysql ocf:mysql-wss binary=/usr/sbin/mysqld test_passwd=DJ1JAs8z test_user=wsrep_sst socket=/var/lib/mysql/mysql.sock --disabled --clone
# 注意，这一步的mysql-wss可以在github上搜到，需要在/usr/lib/ocf/resource.d/放入mysql-wss
$ pcs resource update p_mysql op monitor interval=90 timeout=300
$ pcs resource update p_mysql op stop interval=0 timeout=120
$ pcs resource update p_mysql op start interval=0 timeout=300

# 启动数据库服务
$ pcs resource enable p_mysql

```


使用 HAProxy 的反向代理功能代理后端的 Galera 集群

```
# 在galera 节点安装xinetd服务，并在/etc/xinetd.d/目录下创建galeracheck:


service galeracheck
{
        port            = 49000
        disable        = no
        socket_type    = stream
        protocol        = tcp
        wait            = no
        user            = nobody
        group          = nobody
        groups          = yes
        server          = /usr/bin/galeracheck
        bind            = 0.0.0.0
        only_from      = 0.0.0.0
        per_source      = UNLIMITED
        cps            = 512 10
        flags          = IPv4
        instances      = UNLIMITED
}

# galeracheck的来源及配置项：https://github.com/olafz/percona-clustercheck

# 最后 /etc/haproxy/haproxy.cfg:
listen mysql
  bind 0.0.0.0:3307
  mode tcp
  balance source
  option tcplog
  option clitcpka
  option srvtcpka
  option httpchk
  timeout client 48h
  timeout server 48h
  server mysql-2 192.168.0.7:3306 check port 49000 inter 5000 rise 2 fall 3 backup
  server mysql-3 192.168.0.8:3306 check port 49000 inter 5000 rise 2 fall 3 backup
  server mysql-1 192.168.0.6:3306 check port 49000 inter 10s fastinter 2s downinter 3s rise 3 fall 2s
```


参考：  
[世民谈云计算](http://www.cnblogs.com/sammyliu/p/4741967.html)
