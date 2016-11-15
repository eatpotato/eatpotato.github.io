---
layout: post
title: HDFS完全分布式安装及基本操作
date:       2015-12-8
author:     xue
tags:
    - hadoop
---

## 机器配置和安装说明

&emsp;&emsp;我用了1台机器，win7操作系统，i5处理器。

&emsp;&emsp;使用VMware虚拟机，安装了3台linux虚拟机，Linux操作系统，使用的Ubuntu12.04系统。

&emsp;&emsp;Hadoop版本，使用1.0.4

## 选择安装环境

&emsp;&emsp;下载 hadoop-1.0.4.tar.gz、VMware、ubuntu镜像、jdk、 winscp

&emsp;&emsp;WinSCP是一个Windows环境下使用SSH的开源图形化SFTP客户端。同时支持SCP协议。它的主要功能就是在本地与远程计算机间安全的复制文件。

&emsp;&emsp;Vmware，收费产品，占内存较大。

&emsp;&emsp;Oracle的VirtualBox，开源产品，占内存较小，但安装ubuntu过程中，重启会出错。而且在VirtualBox使用NAT方式联网时，和本地网络连接时，没有VMWare支持的好！

&emsp;&emsp;但是可以下载vm10keygen.exe自动生成序列码，所以我选Vmware。

&emsp;&emsp;Centos，红帽开源版，接近于生产环境。

&emsp;&emsp;Ubuntu，操作简单，方便，界面友好。可以用于实验环境。

&emsp;&emsp;我选Ubuntu12.04.5 32位，可以直接在学校网站上下载，网址是：ftp://ftp.ustb.edu.cn/pub/ubuntu-releases/12.04.5

***
## 更新软件源并安装vim,ssh

1. 在每台linux虚拟机上，更新软件源后，安装：vim，ssh

     命令：sudo apt-get update

	   sudo apt-get upgrade

	   sudo apt-get install vim

   	   sudo apt-get install ssh


2. 在客户端，安装Winscp或putty这两个程序，它们都是依靠ssh服务来操作的，所以前提必须安装ssh服务。

 
     service ssh status   查看ssh状态。

     如果关闭，使用service ssh start  开启服务。

     SecureCRT，可以通过ssh远程访问linux虚拟机。

     winSCP或putty，可以从win7向linux上传文件，或是从linux下载文件。

***
## 修改主机名和网络配置

1. 在每台linux虚拟机上，分别修改主机名为：hostaa，hostbb，hostcc

     命令：sudo vim /etc/hostname

2. 在每台linux虚拟机上 ，配置静态IP

     命令：sudo vim /etc/network/interfaces
     ![set-static-ip](/img/2015/12/8/set-static-ip.png "set-static-ip")<br>

3. 在每台linux虚拟机上 ，修改/etc/hosts文件，hosts文件和windows上的功能是一样的。存储主机名和ip地址的映射。

     命令: sudo vim /etc/hosts 编写hosts文件。结果如下：
     ![modify-hosts](/img/2015/12/8/modify-hosts.png "modify-hosts")<br>

***
## 配置ssh，实现无密码登陆

&emsp;&emsp;无密码登陆，效果也就是在hostaa上，通过 ssh hostbb 或 ssh hostcc 就可以登陆到对方计算机上。而且不用输入密码。

1. 三台虚拟机上，使用 ssh-keygen -t rsa，一路按回车就行了。这部主要是设置ssh的密钥和密钥的存放路径。 路径为~/.ssh下。打开~/.ssh 下面有三个文件：authorized_keys(已认证的keys),id_rsa(私钥),id_rsa.pub(公钥) 

2. 在master上将公钥放到authorized_keys里。命令：sudo cat id_rsa.pub>>authorized_keys
     
	将master上的authorized_keys放到其他linux的~/.ssh目录下。

	命令：sudo scp authorized_keys xue@192.168.209.130:~/.ssh       

	用法：sudo scp authorized_keys 远程主机用户名@远程主机名或ip:存放路径。

3. 修改authorized_keys权限，命令：chmod 644 authorized_keys

4. 测试是否成功
	
	ssh hostbb 输入用户名密码，然后退出，再次ssh hostbb不用密码，直接进入系统。这就表示成功了。
     ![ssh-rsa-test](/img/2015/12/8/ssh-rsa-test.png "ssh-rsa-test")<br>

***
## 上传jdk，并配置环境变量

1. 通过winSCP将文件上传到linux中。将文件放到/usr/lib/java中，三个linux都要操作。

     ![winscp-ftp](/img/2015/12/8/winscp-ftp.png "winscp-ftp")<br>

2. 解压缩：

     命令：tar -zxvf jdk-7u67-linux-i586.tar.gz 

3. 设置环境变量

     命令：sudo vim ~/.bashrc

     在最下面添加：

     export JAVA_HOME=/usr/lib/java/jdk1.7.0_67

     export PATH=$JAVA_HOME/bin:$PATH

     修改完后，用下面的命令让配置文件生效。
 
     source ~/.bashrc

***
## 上传hadoop并添加环境变量

1. 通过winSCP，上传hadoop，到/usr/local/下，解压缩tar -zxvf hadoop-1.0.4.tar.gz

2. 修改环境变量，将hadoop加进去（最后三个linux都操作一次），设置环境变量

     命令：sudo vim ~/.bashrc

     在最下面添加：

     export PATH=$HADOOP_HOME/bin:$PATH

     export HADOOP_HOME=/usr/local/hadoop

     修改完后，用下面的命令让配置文件生效。
     
     source ~/.bashrc

***
## 修改hadoop配置
1. 进入/usr/local/hadoop/conf目录下，修改core_site.xml文件如下：
     ![modify-core-site](/img/2015/12/8/modify-core-site.png "modify-core-site")<br>

     fs.default.name 的值是Name Node的URL+端口号
2. 进入/usr/local/hadoop/conf目录下，修改hdfs_site.xml文件如下：

     ![modify-hdfs-site](/img/2015/12/8/modify-hdfs-site.png "modify-hdfs-site")<br>

     dfs.name.dir 是设定DFS Name节点中的命名空间表格文件，在本地系统中保存的位置。

3. 进入/usr/local/hadoop/conf目录下，修改mapred_site.xml文件如下：

     ![modify-mapred-site](/img/2015/12/8/modify-mapred-site.png "modify-mapred-site")<br>

     mapred.job.tracker是指JobTracker的URL+端口号

4. 进入/usr/local/hadoop/conf目录下，修改masters文件如下：

     ![modify-masters](/img/2015/12/8/modify-masters.png "modify-masters")<br>

     masters的作用是记录namenode的IP

5. 进入/usr/local/hadoop/conf目录下，修改slaves文件如下:

     ![modify-slaves](/img/2015/12/8/modify-slaves.png "modify-slaves")<br>

     slaves的作用是dataNode的IP

6. 修改hadoop-env.sh文件,在最下面加入：

     ![modify-slaves](/img/2015/12/8/modify-hadoop-env.png "modify-hadoop-env")<br>

7. 更改hadoop目录的属主

     命令： sudo chown -R xue:hadoop hadoop

     解释：sudo chown -R 用户名@用户组 目录名

8. 让hadoop配置生效

     命令：source hadoop-env.sh

9. 格式化namenode，只格式一次

     命令：hadoop namenode -format

10. 启动hadoop

     切到/usr/local/hadoop/bin目录下,命令：./start-all.sh

11. 查看进程，是否启动

     命令： jps		    

     ![namenode-jps](/img/2015/12/8/namenode-jps.png "namenode-jps")<br>

     ![datanode-jps](/img/2015/12/8/datanode-jps.png "datanode-jps")<br>

***
## 用Web查看Hadoop运行状态

1. 查看Map/Reduce的管理情况
     
     202.204.62.86:50030

2. 查看HDFS的管理情况

     202.204.62.86:50070


## HDFS Shell 基本操作

1. 创建目录

     命令:hadoop fs -mkdir /目录名称

2. 上传本地文件至HDFS

     命令：hadoop fs -put 文件名称 目录名称

3. 列出某目录下所有文件

     命令：hadoop fs -ls 目录名称

4. 查看HDFS运行情况

     命令：hadoop dfsadmin -report
