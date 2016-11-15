---
layout:     post
title:      openstack-all-in-one30分钟快速搭建
subtitle:   
date:       2016-11-15 
author:     xue
header-img: img/post-bg-linux-interrupt.jpg
tags:
    - openstack
---

# puppet-openstack-intergration
 
## 项目简介
puppet-openstack-integration此模块确保社区可以持续地测试和验证使用Puppet modules部署的Openstack集群。  
支持环境：支持 Ubuntu 14.04或者CentOS 7.x

## 开始前准备工作
1. 解决跟linux内核版本有关的bug（在特定内核版本的虚拟机上执行会出现此错误）:  
   说明：目前在有云上创建的CentOS7.2的虚拟机都会出现此问题。  
   方法：在http://rpmfind.net/linux/rpm2html/search.php?query=kernel-devel中找到对应内核版本的kernel-devel rpm包并安装。

2. 开启虚拟机的selinux:  
   方法：更改/etc/selinux/config文件：SELINUX=enforcing,重启。
    
3. 更改gem source,pip镜像(建议执行此步骤，我好几次出错都是因为gem 安装失败，pip安装超时的原因而终止)：
 
```
gem sources -l #查看现有的gem源
gem source --remove https://rubygems.org/ #注意是source而非sources
gem source -a https://ruby.taobao.org/ #添加淘宝的源
 
mkdir /root/.pip #创建.pip目录
vim /root/.pip/pip.conf #内容如下：
[global]
timeout = 60 #设置超时时间
index-url = https://pypi.douban.com/simple  
``` 
 
4. 从github上将puppet-openstack-integration项目stable/mitaka分支克隆下来，命令：  

```
git clone -b stable/mitaka https://github.com/openstack/puppet-openstack-integration.git
```

5. 进入puppet-openstack-integration目录下，更改all-in-one.sh（重要）：   

```
git clone -b stable/mitaka https://github.com/openstack/puppet-openstack-integration.git #将第43行内容改为此行的内容
export SCENARIO=scenario001 #将第46行内容改为此行内容
```

## Installation

1. 进入puppet-openstack-integration目录下，执行：./all-in-one.sh
2. 等待n杯咖啡的时间,openstack(M版)+ceph就安装好了
3. 当你看到以下提示时，说明安装成功：

```
OpenStack Dashboard available: http://127.0.0.1/dashboard
To access through Horizon, use the following user/password:
  admin / a_big_secret
To use OpenStack through the CLI, run:
  source ~/openrc
```
4. 本次安装版本不带horizon组件，所以需要通过CLI访问。

## 执行过程分析
执行all-in-one.sh脚本：  
1. 从git上clone puppet-openstack-integration项目  
2. 根据funtion函数中的方法判断操作系统的类型，安装libxml2-devel libxslt-devel,ruby,gem等软件，卸载facter，puppet等软件  
3. 通过gem安装bundler  
4. 执行run_tests.sh

执行run_tests.sh脚本：  
1. export一些变量，判断fixtures/scenarioo×.pp是否存在  
2. 从git://git.openstack.org/openstack/tempest上clone下来Tempest and plugins  
3. 根据系统安装puppet,dstat 等等   
4. 执行install_modules.sh  

执行 install_modules.sh脚本  
1. export一些变量，通过gem安装r10k  
2. 调用function中install_modules方法，使用r10k puppetfile install -v 在/etc/puppet/module目录下安装puppet-openstack-integration/Puppetfile中定义的所有module  
3. 执行puppet module list命令  
4. install_modules.sh执行完毕，回到run_test.sh  

回到run_test.sh脚本：  
1. Install repo  
2. 通过run_puppet 方法，执行命令/usr/bin/puppet apply --detailed-exitcodes --color=false --test --trace --hiera_config /tmp/puppet-openstack-integration/hiera/hiera.yaml fixtures/scenario001.pp    
3. 第二次执行run_puppet ，根据执行返回值确定执行状态。  
4. 安装配置tempest并相应运行smoke测试

## 可能出现的问题
执行过程中，可能在git clone -b 12.0.0 git://git.openstack.org/openstack/tempest /tmp/openstack/tempest这一步等待较长时间然后报错  
解决方法：  
   在命令行执行：git clone -b 12.0.0 https://git.openstack.org/openstack/tempest /tmp/openstack/tempest  ,并注释掉run_test.sh的这一行代码（第62行）
