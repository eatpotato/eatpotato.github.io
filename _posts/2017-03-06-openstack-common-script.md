---
layout:     post
title:      openstack常用脚本
date:       2017-03-04
author:     xue
catalog:    true
tags:
    - openstack
---

## 镜像下载与上传


```
wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img -P /tmp

openstack image create "cirros" --file /tmp/cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare --public

```


## Openstack服务检查脚本

```
for service in `systemctl list-units | grep -P "(openstack)|(neutron)"| sed 's/●//g' | awk  '{print $1}'`; do
        if systemctl is-active $service &>/dev/null; then
                echo -e "\e[32m$service is OK!\e[0m"
        else
                echo -e "\e[31m$service fail to start!\e[0m"
        fi
done
```

## Glance服务重启

```
service openstack-glance-registry restart

service openstack-glance-registry status

service openstack-glance-api restart

service openstack-glance-api status
```

## 创建private网络和路由器

```
  neutron net-create test
  neutron subnet-create --name test --gateway 192.168.1.1 test 192.168.1.0/24
  neutron router-create router
  neutron router-interface-add router test
  neutron router-gateway-set router public
```

## 创建安全组

```
nova secgroup-add-rule default tcp 22 22 0.0.0.0/0

nova secgroup-add-rule default tcp 80 80 0.0.0.0/0

nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
```

## 创建虚拟机

```
openstack server create --flavor m1.large --image $IMAGE_NAME --nic net-id=$NET_ID --security-group default $HOST_NAME

```

未完待续
