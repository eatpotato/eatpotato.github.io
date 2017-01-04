---
layout:     post
title:      ansible总结
date:       2017-01-07
author:     xue
tags:
    - ansible
---

## 安装ansible

1.yum安装:
RHEL()Centos)7版本：

```
rpm -Uvh http://mirrors.zju.edu.cn/epel/7/x86_64/e/epel-release-7-8.noarch.rpm
yum install ansible
```
2.Apt(Ubuntu)安装方式：

```
apt-get install software-properties-common
apt-add-repository ppa:ansible/ansible
apt-get update
apt-get install ansible
```

3.homebrew (Mac OSX)安装方式

```
brew update
brew install Ansible
```

4.通过pip安装:

```
easy_install pip
pip install ansible
```
如果是在OS X系统上安装，编译器可能会有警告或出错，需要设置CFLAGS、CPPFLAGS环境变量：

```
sudo CFLAGS=-Qunused-arguments CPPFLAGS=-Qunused-arguments pip install ansible
```

##  配置运行环境

### Ansible环境
Ansible 配置文件是以.ini格式存储配置数据的，在Ansible中，几乎所有的配置项都可以通过Ansible的playbook或环境变量来重新赋值。在运行Ansible命令式，命令将会按照预先设定的顺序查找配置文件，如下所示：

1）ANSIBLE_CONFIG: 首先，Ansible命令会检查环境变量，及这个环境变量指向的配置文件。  
2）./ansible.cfg: 其次，将会检查当前目录下的ansible.cfg配置文件  
3）~/ansible.cfg: 再次，将会检查当前用户home目录下的.ansible.cfg配置文件  
4) /etc/ansible/ansible.cfg: 最后，将会检查再用软件包管理工具安装Ansible时自动产生的配置文件。

### ansible.cfg常用配置参数

* inventory  这个参数表示资源清单inventory文件的位置，资源清单就是被管理主机列表。如：inventory = /etc/ansible/hosts
* forks  设置默认情况下Ansible最多能有多少个进程同时工作，默认设置最多5个进程并行处理。如： forks = 5
* sudo_user  这是设置默认执行命令的用户，如： sudo_user = root
* remote_port  这是制定连接被管节点的管理端口，默认是22。除非设置了特殊的SSH端口，不然这个参数一般是不需要修改的。如：reomte_ssh = 22
* timeout  这是设置SSH连接的超时间隔，单位是秒。配置示例如下：timeout = 60


## 第一条ansible命令

编辑(或创建)/etc/ansible/hosts文件，在其中加入被管理的远程主机:
[merge]
10.0.81.31
10.0.81.32
10.0.81.33

注意：你的public SSH key必须在这些系统的“authorized_keys”中

使用list-hosts参数进行验证：  
![](/img/ansible/ansible-list-hosts.png)

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

### 分文件定义 Host 和 Group 变量
在 inventory 主文件中保存所有的变量并不是最佳的方式.还可以保存在独立的文件中,这些独立文件与 inventory 文件保持关联. 不同于 inventory 文件(INI 格式),这些独立文件的格式为 YAML.
假设 inventory 文件的路径为:

```
/etc/ansible/hosts
```

假设有一个主机名为 ‘foosball’, 主机同时属于两个组,一个是 ‘raleigh’, 另一个是 ‘webservers’. 那么以下配置文件(YAML 格式)中的变量可以为 ‘foosball’ 主机所用.依次为 ‘raleigh’ 的组变量,’webservers’ 的组变量,’foosball’ 的主机变量:

```
/etc/ansible/group_vars/raleigh
/etc/ansible/group_vars/webservers
/etc/ansible/host_vars/foosball
```

Tip: Ansible 1.2 及以上的版本中,group_vars/ 和 host_vars/ 目录可放在 inventory 目录下,或是 playbook 目录下. 如果两个目录下都存在,那么 playbook 目录下的配置会覆盖 inventory 目录的配置

## Parallelism and Shell Commands
举一个例子

这里我们要使用 Ansible 的命令行工具来重启 Atlanta 组中所有的 web 服务器,每次重启10个.

我们先设置 SSH-agent,将私钥纳入其管理:

```
$ ssh-agent bash
$ ssh-add ~/.ssh/id_rsa
```