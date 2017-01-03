---
layout:     post
title:      ansible总结
date:       2017-01-07
author:     xue
tags:
    - ansible
---

## 安装ansible

从源码安装的步骤:

```
$ git clone git://github.com/ansible/ansible.git --recursive
$ cd ./ansible
$ source ./hacking/env-setup
```

通过pip安装:

```
pip install ansible
```

## 第一条ansible命令

编辑(或创建)/etc/ansible/hosts文件，在其中加入被管理的远程主机:
[merge]
10.0.81.31
10.0.81.32
10.0.81.33

注意：你的public SSH key必须在这些系统的“authorized_keys”中

现在对你的管理节点运行一个命令:

![](/img/ansible/ansible-first-command.png)

## inventory文件

### 主机与组
Ansible 可同时操作属于一个组的多台主机,组和主机之间的关系通过 inventory 文件配置. 默认的文件路径为 /etc/ansible/hosts

/etc/ansible/hosts 文件的格式与windows的ini配置文件类似:
 
```
[webservers]
foo.example.com
bar.example.com

[dbservers]
one.example.com
two.example.com
three.example.com
```

方括号[]中是组名,用于对系统进行分类,便于对不同系统进行个别的管理。  
一个系统可以属于不同的组,比如一台服务器可以同时属于 webserver组 和 dbserver组.这时属于两个组的变量都可以为这台主机所用

如果有主机的SSH端口不是标准的22端口,可在主机名之后加上端口号,用冒号分隔，如下：

```
badwolf.example.com:5309
```

假设你有一些静态IP地址,希望设置一些别名,但不是在系统的 host 文件中设置,又或者你是通过隧道在连接,那么可以设置如下:

```
jumper ansible_ssh_port=5555 ansible_ssh_host=192.168.1.50
```
在这个例子中,通过 “jumper” 别名,会连接 192.168.1.50:5555.

一组相似的 hostname , 可简写如下:

```
[webservers]
www[01:50].example.com
```
数字的简写模式中,01:50 也可写为 1:50,意义相同.你还可以定义字母范围的简写模式:

```
[databases]
db-[a:f].example.com
```
对于每一个host,你还可以选择连接类型和连接用户名：

```
[targets]

localhost              ansible_connection=local
other1.example.com     ansible_connection=ssh        ansible_ssh_user=mpdehaan
other2.example.com     ansible_connection=ssh        ansible_ssh_user=mdehaan    
```
### 主机变量
分配变量给主机的方式:

```
[atlanta]
host1 http_port=80 maxRequestsPerChild=808
host2 http_port=303 maxRequestsPerChild=909
```
### 组的变量
也可以定义属于整个组的变量:

```
[atlanta]
host1
host2

[atlanta:vars]
ntp_server=ntp.atlanta.example.com
proxy=proxy.atlanta.example.com
```
### 把一个组作为另一个组的子成员
可以把一个组作为另一个组的子成员,以及分配变量给整个组使用. 这些变量可以给 /usr/bin/ansible-playbook 使用,但不能给 /usr/bin/ansible 使用:

```
[atlanta]
host1
host2

[raleigh]
host2
host3

[southeast:children]
atlanta
raleigh

[southeast:vars]
some_server=foo.southeast.example.com
halon_system_timeout=30
self_destruct_countdown=60
escape_pods=2

[usa:children]
southeast
northeast
southwest
northwest
```