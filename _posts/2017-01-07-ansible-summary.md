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
RHEL(Centos)7版本：

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
sodu easy_install pip
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

1.inventory  这个参数表示资源清单inventory文件的位置，资源清单就是被管理主机列表。如：inventory = /etc/ansible/hosts  
2.forks  设置默认情况下Ansible最多能有多少个进程同时工作，默认设置最多5个进程并行处理。如： forks = 5  
3.sudo_user  这是设置默认执行命令的用户，如： sudo_user = root  
4.remote_port  这是制定连接被管节点的管理端口，默认是22。除非设置了特殊的SSH端口，不然这个参数一般是不需要修改的。如：reomte_ssh = 22  
5.timeout  这是设置SSH连接的超时间隔，单位是秒。配置示例如下：timeout = 60  


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
目前ansible自带很多模块，我们可以使用ansible-doc -l显示所有自带模块，还可以通过ansible-doc "模块名"，查看模块的介绍以及案例。

下面举一些常见的命令：

1.shell命令  
![](/img/ansible/ansible-shell-command.png)  

-o, --one-line  Try to output everything on one line.

2.复制文件  
![](/img/ansible/ansible-copy-command.png)

3.包和服务管理  
![](/img/ansible/ansible-yum-command.png)


## playbook
Playbooks 是 Ansible的配置,部署,编排语言.他们可以被描述为一个需要希望远程主机执行命令的方案,或者一组IT程序运行的命令集合.
playbook 文件格式为YAML语法，在编写playbook前需要对YAML有一定的了解，关于YAML语法可以通过 [yaml官网](http://yaml.org/spec/1.2/spec.html)进行学习。

playbook 由一个或多个 ‘plays’ 组成.它的内容是一个以 ‘plays’ 为元素的列表。在 play 之中,一组机器被映射为定义好的角色.在 ansible 中,play 的内容,被称为 tasks,即任务.在基本层次的应用中,一个任务是一个对 ansible 模块的调用,

下面一个 playbook示例,其中仅包含一个 play:  

```
---
- hosts: webservers
  vars:
    http_port: 80
    max_clients: 200
  remote_user: root
  tasks:
  - name: ensure apache is at the latest version
    yum: pkg=httpd state=latest
  - name: write the apache config file
    template: src=/srv/httpd.j2 dest=/etc/httpd.conf
    notify:
    - restart apache
  - name: ensure apache is running
    service: name=httpd state=started
  handlers:
    - name: restart apache
      service: name=httpd state=restarted
```

在 [ansible-examples](https://github.com/ansible/ansible-examples) 中有很多实例，如果你希望深入学习可以在单独的页面打开它。

### Task
每一个 play 包含了一个 task 列表（任务列表）.一个 task 在其所对应的所有主机上（通过 host pattern 匹配的所有主机）执行完毕之后,下一个 task 才会执行.

### playbook变量与引用
1.通过inventory文件定义主机以及主机变量  
在/etc/ansible/hosts中定义主机组和组变量如下：

```
merge]
10.0.81.31
10.0.81.32
10.0.81.33

[merge:vars]
key=test
```
创建playbook文件test.yaml如下：

```
---
- hosts: merge
  gather_facts: False
  tasks:
  - name: display Host Variable from hostfile
    debug: msg="The key value is {{ key }}"
```

运行playbook文件：

![](/img/ansible/ansible-playbook-example.png)

2.通过文件定义主机以及主机组变量
我们可以通过host_vars和group_vars目录来针对主机和主机组定义变量。

使用yum安装Ansible的配置文件在/etc/ansible/目录下，我们在该目录下新建host_vars和group_vars目录,如下：
![](/img/ansible/ansible-tree.png)

3.通过ansible-playbook命令行传入
可以通过-e 命令传入变量

```
ansbile-playbook test.yaml -e "key=test"
```
ansible-playbook目前还支持YAML和JSON的方式传入变量

```
cat var.yaml
---
key: test
cat var.json
{"key": "test"}
```
4.可以在playbook文件内通过vars字段定义变量
5.可以在playbook文件内通过vars_files字段引用变量，首先把所有的变量定义在某个文件内，然后再playbook文件内使用vars_files参数引用这个变量文件
6.使用register内的变量
Ansible playbook内task之间可以互相传递数据，比如第2个task需要获得第1个task的执行结果。

我们可以通过下面的方式：

```
vim test01.yaml
---
- hosts: merge
  gather_facts: False
  tasks:
  - name: register variable
    shell: hostname
    register: info
  - name: display variable
    debug: msg="The key value is {{ info }}"
```

![](/img/ansible/ansible-register-command.png)

info的结果是一段python字典数据，里面存储着很多信息，包括执行时间状态变化输出等。

### playbook 循环

1.标准loops  
example:


```
---
- hosts: merge
  gather_facts: False
  tasks:
  - name: debug loops
    debug: msg="name ---------->  {{ item }}"
    with_items:
      - one
      - two
```

![](/img/ansible/ansible-standard-loops.png)

with_items的值是python list数据结构， 可以理解为每个task会循环读取list里面的值，然后key的名称是item

2.嵌套loops

```
---
- hosts: merge
  gather_facts: False
  tasks:
  - name: debug loops
    debug: msg="name ---------->  {{ item[0] }} ---------> {{ item[1] }}"
    with_nested:
      - ['A']
      - ['a','b','c']
```

![](/img/ansible/ansible-nesting-loops.png)

3.条件判断loops
有时候执行一个task之后，我们需要检测这个task的结果是否达到了预想状态，如果没有就需要退出整个playbook执行，这个时候我们就需要对某个task结果一直循环检测了，如下所示:

```
---
- hosts: merge
  gather_facts: False
  tasks:
  - name: debug loops
    shell: cat /root/test.txt
    register: host
    until: host.stdout.startswith("test")
    retries: 5
    delay: 5
```

5秒执行一次 cat /root/test.txt将结果register给host，然后判断host.stdout.startwith的内容是不是test字符串开头，如果条件成立，此task运行完成，如果条件不成立，5秒以后重试，5此后还不满足条件，此task运行失败。

其他的循环还有：散列loops、文件匹配loops、随机选择loops、文件优先匹配loops、register loops

### playbook lookups
Ansible还支持从外部数据拉取信息，比如我们可以从数据库拉取信息，然后赋值给一个变量。

1.lookups file

```
---
- hosts: merge
  gather_facts: False
  vars:
    contents: "{{ lookup('file', '/root/openrc') }}"
  tasks:
  - name: debug lookups
    debug: msg="The contents is {% for i in contents.split("\n") %} {{ i }} {% endfor %}"
```

2.lookups password
lookup('password', 'file_path')
它会对传入的内容进行加密处理

### playbook conditionals
目前Ansible所有conditionals方式都是通过使用when进行判断，when的值是一个条件表达式，如果条件判断成立，这个task就执行某个操作，否则，该task不执行。

```
tasks:
  - name: "shutdown Debian flavored systems"
    command: /sbin/shutdown -t now
    when: ansible_os_family == "Debian"
```
如果想查看哪些facts变量可以引用,可以在命令行上通过调用setup module命令可以查看
  
```
ansible hostname -m setup 
```

### TAGS
如果你有一个很大的playbook，而你只想run其中的某个task，这个时候tags是你的最佳选择。

一、最常见的使用形式:

```
tasks:
 
    - yum: name={{ item }} state=installed
      with_items:
         - httpd
         - memcached
      tags:
         - packages
 
    - template: src=templates/src.j2 dest=/etc/foo.conf
      tags:
         - configuration
    
```
此时若你希望只run其中的某个task，这run 的时候指定tags即可:

```
ansible-playbook example.yml --tags "configuration,packages"   #run 多个tags
ansible-playbook example.yml --tags packages                   # 只run 一个tag
```

相反，也可以跳过某个task

```
ansible-playbook example.yml --skip-tags configuration
```

二、tags 和role 结合使用
tags 这个属性也可以被应用到role上，例如:

```
roles:
  - { role: webserver, port: 5000, tags: [ 'web', 'foo' ] }
```

三、tags和include结合使用

```
- include: foo.yml tags=web,foo
```
这样，fool.yml 中定义所有task都将被执行

四、系统中内置的特殊tags：

  always、tagged、untagged、all 是四个系统内置的tag，有自己的特殊意义。  
  always: 指定这个tag 后，task任务将永远被执行，而不用去考虑是否使用了--skip-tags标记  
  tagged: 当 --tags 指定为它时，则只要有tags标记的task都将被执行,--skip-tags效果相反  
  untagged: 当 --tags 指定为它时，则所有没有tag标记的task 将被执行,--skip-tags效果相反  
  all: 这个标记无需指定，ansible-playbook 默认执行的时候就是这个标记.所有task都被执行  

## Roles

怎样组织 playbook 才是最好的方式呢？简单的回答就是：使用 roles ! Roles 基于一个已知的文件结构，去自动的加载某些 vars_files，tasks 以及 handlers。基于 roles 对内容进行分组，使得我们可以容易地与其他用户分享 roles 。  
一个项目的结构如下:

```
site.yml
webservers.yml
fooservers.yml
roles/
   common/
     files/
     templates/
     tasks/
     handlers/
     vars/
     defaults/
     meta/
   webservers/
     files/
     templates/
     tasks/
     handlers/
     vars/
     defaults/
     meta/
```

如果 roles 目录下有文件不存在，这些文件将被忽略。比如 roles 目录下面缺少了 ‘vars/’ 目录，这也没关系。
当一些事情不需要频繁去做时，你也可以为 roles 设置触发条件，像这样:

```
---

- hosts: webservers
  roles:
    - { role: some_role, when: "ansible_os_family == 'RedHat'" }
```

## Ansible最佳实践
### 优化Ansible速度
1.开启SSH长连接
如果被管理机器的SSH -V版本高于5.6时，我们可以直接在ansible.cfg文件中设置SSH长连接。设置参数如下：

```
sh_args = -o ControlMaster=auto -o Controlpersist=5d
```
Controlpersist=5d  设置整个长连接保持时间为5天

2.开启pipelining
如果开启了pipelining ,生成好的本地Python脚本PUT到远端服务器的过程会在SSH的会话中进行，这样可以大大提高整个执行效率。在ansible.cfg中设置：

```
pipelining = True
```

3.设置facts缓存
如果playbook不需要facts信息，可以在playbook中设置gather_facts:Fasle来提高playbook效率。但是如果我们既想在每次执行playbook的时候能搜集facts,又想加速这个收集过程，那么就需要配置facts缓存了。
可以在ansible.cfg中配置fact缓存使用redis:

```
[defaults]
gathering = smart
fact_caching = redis  #jsonfile等，使用json 需要指定fact_caching_connection指定存放路径
fact_caching_timeout = 86400

```

### 灰度发布与检测
1.语法检测
在编写完playbook或者role之后一定要养成进行语法检测的习惯，直接使用ansible-playbook命令的 --syntax-check参数即可。
2.灰度发布
进行预运行之前，我们需要把一个或者多个task使用delegate_to参数指定到一台设备上进行测试。

## Ansible与openstack

```
---
- hosts: localhost
  gather_facts: False
  vars:
    auth_url: http://10.0.84.52:35357/v3
    login_username: admin
    login_tenant_name: admin
    login_password: 
    image_id: 63a7ea8d-2fc4-442b-8455-261b93aeb909
    keypair_name: mykey
    private_net: 1b3c98f2-7cbb-459f-bb1f-3dbb4facee13
    flavor_id: 1
  connection: local
  tasks:
  - name: create compute
    nova_compute:
      auth_url: "{{ auth_url }}"
      login_username: "{{ login_username }}"
      login_password: "{{ login_password }}"
      login_tenant_name: "{{ login_tenant_name }}"
      security_groups: default
      state: present
      name: "test-xue-01"
      image_id: "{{ image_id }}"
      key_name: "{{ keypair_name }}"
      flavor_id: "{{ flavor_id }}"
      region_name: RegionOne
      nics:
        - net-id: "{{ private_net }}"
    register: openstack
  - name: pull floating ip address
    local_action: floating
      auth_url={{ auth_url }} username={{ login_username }} password={{ login_password }} tenant={{ login_tenant_name }}
    register: floating_ip

  - name: bond flaoting ip to instance
    local_action: bondip_openstack
      auth_url={{ auth_url }} username={{ login_username }} password={{ login_password }} tenant={{ login_tenant_name }} instance_id={{ openstack.id }}floating_ip={{ floating_ip.res }}
```

更多资料可以查看：[openstack-ansible](https://github.com/openstack/openstack-ansible)

## 几个实用的ansible命令：

1.管理域名解析器:


```
- name: Create DNS File
  register: config_resolv
  template: src=resolv.conf.j2 dest=/etc/resolv.conf
  tags: firsthyper_init
```
由于我使用了role来组织playbook,roleresolv.conf.j2在相应role的templates目录下.

2.安装常用软件：

```
- name: Install ntp,puppet,rsync
  register: install_package
  yum: name={{item}} state=latest
  with_items:
     - ntpdate
     - puppet
     - rsync
```
3.同步时间

```
- name: Sync time
  register: sync_time
  shell: systemctl stop ntpd && sleep 5 && /usr/sbin/ntpdate {{ ntp_server }} 1 >/dev/null 2>&1 && systemctl start ntpd
```

ntp_server定义在group_vars中.


4.查看数据库日志文件名称

```
- name: Fetch Mysql Master Log File
  register: master_log_file
  shell: mysql -e "show master status" | grep mysql | awk '{print $1}'
  delegate_to: "{{ mysql_master }}"
```

参考：  
[Ansible中文权威指南](http://www.ansible.com.cn/)  
[ “诡迹” 博客](http://unixman.blog.51cto.com/10163040/1674198)  
[Ansible自动化运维技术与最佳实践](http://product.dangdang.com/23958049.html)
