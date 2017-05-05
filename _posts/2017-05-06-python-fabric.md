---
layout:     post
title:      python fabric详解
date:       2017-05-06
author:     xue
catalog:    true
tags:
    - python
---

## 简介
Fabirc是基于python实现的SSH命令行工具，非常适合应用的自动化部署，或者执行系统管理任务。

简单的例子：

```
root@openstack:~# cat fabfile.py
def hello():
    print 'hello world!'
 
root@openstack:~# fab hello
hello world!

Done.
```

这个fab简单地导入了fabfile,并执行定义的hello函数。

fab作为Fabric程序的命令行入口，提供了丰富的参数调用，命令格式如下：

```
root@openstack:~# fab --help
Usage: fab [options] <command>[:arg1,arg2=val2,host=foo,hosts='h1;h2',...] ...
```

各参数含义如下：

|参数项|含义|
|--|--|
|-l|显示可用任务函数名|
|-f|指定fab入口文件,默认为fabfile.py|
|-g|指定网关（中转设备），比如堡垒机环境，填写堡垒机IP即可|
|-H|指定目标主机，多台主机用“，”分隔|
|-P|以异步并行方式运行多台主机任务，默认为串行运行|
|-R|指定角色（Role）|
|-t|设置设备连接超时时间|
|-T|设置远程主机命令执行超时时间|
|-w|当命令执行失败，发出告警，而非默认终止任务|




## 实例分析

```
#自定义fabfile文件如下
#/root/fab.py
from fabric.api import *

#local_node和remote_node都可以为多个
local_node = ['127.0.0.1']
remote_node = ['10.116.97.30']
#定义local和remote角色
env.roledefs['local'] = local_node
env.roledefs['remote'] = remote_node
#定义用户名
env.user = 'root'
#定义密码
env.password = 'xxx'

#不同的角色执行不同的命令
@roles('local')
def test_local():
    #run命令在roles为local的主机上执行命令"hostname"
    run("hostname")

@roles('remote')
def test_remote():
    run("hostname")

```

在python中调用：

```
root@openstack:~# python
Python 2.7.6 (default, Oct 26 2016, 20:30:19)
[GCC 4.8.4] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> from fabric.api import execute
>>> import sys
>>> sys.path.append('/root')
>>> import fab
>>> execute(fab.test_local)
[127.0.0.1] Executing task 'test_local'
[127.0.0.1] run: hostname
[127.0.0.1] out: openstack
[127.0.0.1] out:

{'127.0.0.1': None}

>>> execute(fab.test_remote)
[10.116.97.30] Executing task 'test_remote'
[10.116.97.30] run: hostname
[10.116.97.30] out: openstack
[10.116.97.30] out:

{'10.116.97.30': None}
```

## fabfile全局属性设定

env对象的作用是定义fabfile的全局设定，各属性说明如下：

|属性|含义|
|--|--|
|env.host|定义目标主机,以python的列表表示，如env.host=['xx.xx.xx.xx','xx.xx.xx.xx']|
|env.exclude_hosts|排除指定主机，以python的列表表示|
|env.port|定义目标主机端口，默认为22|
|env.user|定义用户名|
|env.password|定义密码|
|env.passwords|与password功能一样，区别在于不同主机配置不同密码的应用情景,配置此项的时候需要配置用户、主机、端口等信息，如：env.passwords = {'root@xx.xx.xx.xx:22': '123', 'root@xx.xx.xx.xx':'234'}|
|env.getway|定义网关|
|env.deploy_release_dir|自定义全局变量|
|env.roledefs|定义角色分组|

## 常用的API

Fabric支持常用的方法及说明如下：

|方法|说明|
|--|--|
|local|执行本地命令，如:local('hostname')|
|lcd|切换本地目录,lcd('/root')|
|cd|切换远程目录,cd('cd')|
|run|执行远程命令，如：run('hostname')|
|sudo|sudo执行远程命令，如：sudo('echo “123456″ | passwd --stdin root')|
|put|上传本地文件到远程主机,如：put(src,des)|
|get|从远程主机下载文件到本地，如：get(des,src)|
|prompt|获取用户输入信息，如：prompt（‘please enter a new password:’）|
|confirm|获取提示信息确认，如：confirm('failed.Continue[Y/n]?')|
|reboot|重启远程主机，reboot()|
|@task|函数修饰符，标识的函数为fab可调用的|
|@runs_once|函数修饰符，表示的函数只会执行一次|