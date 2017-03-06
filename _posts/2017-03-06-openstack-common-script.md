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

未完待续
