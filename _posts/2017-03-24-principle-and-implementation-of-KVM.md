---
layout:     post
title:      KVM的原理与使用
date:       2017-03-25
author:     xue
catalog:    true
tags:
    - 虚拟化
---

## 一、KVM简介

KVM的全称是Kernel Virtual Machine，翻译过来就是内核虚拟机。它是一个 Linux 的一个内核模块，该内核模块使得 
Linux 变成了一个 Hypervisor：

* 它由 Quramnet 开发，该公司于 2008年被 Red Hat 收购。
* 它支持 x86 (32 and 64 位), s390, Powerpc 等 CPU。
* 它从 Linux 2.6.20 起就作为一模块被包含在 Linux 内核中。
* 它需要支持虚拟化扩展的 CPU。
* 它是完全开源的。[官网](http://www.linux-kvm.org/page/Main_Page)

### 1.1 KVM的功能列表

* 支持CPU 和 memory 超分（Overcommit）
* 支持半虚拟化I/O （virtio）
* 支持热插拔 （cpu，块设备、网络设备等）
* 支持对称多处理（Symmetric Multi-Processing，缩写为 SMP ）
* 支持实时迁移（Live Migration）
* 支持 PCI 设备直接分配和 单根I/O 虚拟化 （SR-IOV）
* 支持 内核同页合并 （KSM ）
* 支持 NUMA （Non-Uniform Memory Access，非一致存储访问结构 ）

### 1.2 KVM原理

KVM 本身不执行任何模拟，需要用户空间应用程序 QEMU 通过 /dev/kvm 接口设置一个客户机虚拟服务器的地址空间，向它提供模拟的 I/O，KVM 模块实现处理器的虚拟化和内存虚拟化。在硬件虚拟化技术的支持下，内核的 KVM 模块与 QEMU 的设备模拟协同工作，构成一套和物理计算机系统完全一致的虚拟化计算机软硬件系统。

![](/img/kvm/kvm-principle.png)

## 二、KVM基础功能

### 2.1 CPU

在 QEMU/KVM 中，QEMU 提供对 CPU 的模拟，展现给客户机一定的 CPU 数目和 CPU 的特性。在 KVM 打开的情况下，客户机中 CPU 指令的执行由硬件处理器的虚拟化功能「如 Intel VT-x 和 AMD AMD-V」辅助执行，具有非常高的执行效率。

在 KVM 环境中，每个客户机都是一个标准的 Linux 进程(QEMU 进程)，而每一个 vCPU 在宿主机中是 QEMU 进程派生的一个普通线程。

在 Linux 中，一般进程有两种执行模式：

* 内核模式
* 用户模式

而在 KVM 环境中，增加了第三条模式：客户模式。vCPU 在三种执行模式下的分工如下：

* 用户模式
	* 主要处理 I/O 的模拟和管理，由 QEMU 的代码实现
* 内核模式
	* 主要处理特别需要高性能和安全相关的指令，如处理客户模式到内核模式的转换，处理客户模式下的 I/O 指令或其它特权指令引起的 VM-Exit，处理影子内存管理 （shadow MMU）
* 客户模式
	* 主要执行 Guest 中的大部分指令，I/O 和一些特权指令除外「它们会引起 VM-Exit，被 hypervisor 截获并模拟」
	
![](/img/kvm/kvm-cpu.png)

### 2.2 内存

内存是一个非常重要的部件，它是与 CPU 沟通的一个桥梁。

在通过 QEMU 命令行启动客户机时设置内存的参数是 -m:

* -m megs # 设置客户机的内存为 megs MB 大小

**EPT 和 VPID**

EPT（Extended Page Tables，扩展页表），属于 Intel 的第二代硬件虚拟化技术，它是针对内存管理单元（MMU）的虚拟化扩展。如果只是一台物理服务器，这个物理地址就只为一个操作系统服务，但如果进行了虚拟化部署，有多个虚拟机时，就存在着稳定性的隐患。因为在进行 VM Entry（虚拟机进入）与 VM Exit（虚拟机退出）时（尤其是后者），都要对内存页进行修改。但物理内存是多个虚拟机共享的，因此不能让虚拟机直接访问物理地址，否则一个虚拟机出现内存错误，就会殃及整个物理服务器的运行。所以必须要采取虚拟地址，而 EPT 的作用就在于加速从虚拟机地址至主机物理地址的转换过程，节省传统软件处理方式的系统开销。

VPID（Virtual-Processor Identifiers，虚拟处理器标识）。是对现在的 CPUID 功能的一个强化，因为在每个 CPU 中都有一个 TLB，用来缓存逻辑地址到物理地址的转换表，而每个虚拟机都有自己的虚拟 CPU 来对应。所以，在进行迁移时要进行 TLB 的转存和清除。而 VPID 则会跟踪每个虚拟 CPU 的 TLB，当进行虚拟机迁移或 VM Entry 与 VM Exit 时，VMM可以动态的分配非零虚拟处理器的 ID 来迅速匹配（0 ID 给 VMM 自己使用），从而避免了 TLB 的转存与清除的操作，节省了系统开销，并提高了迁移速度，同时也降低对系统性能的影响。

```
# grep -E 'ept|vpid' /proc/cpuinfo                  # 查看 cpu 是否支持相应特性
# cat /sys/module/kvm_intel/parameters/{ept,vpid}   # 确认是否开启 ept 和 vpid
```


 

## 三、CentOS上安装KVM功能模块步骤

### 3.1 KVM 需要有 CPU 的支持（Intel VT 或 AMD SVM），在安装 KVM 之前检查一下 CPU 是否提供了虚拟技术的支持。

* 基于 Intel 处理器的系统，运行grep vmx /proc/cpuinfo查找 CPU flags 是否包括 vmx 关键词
* 基于 AMD 处理器的系统，运行grep svm /proc/cpuinfo查找 CPU flags 是否包括 svm 关键词
* 检查BIOS，确保BIOS里开启VT选项

### 3.2 安装KVM所需组件

```
yum install seabios  qemu-kvm virt-manager libvirt -y
```

启动libvirt服务：

```
systemctl start libvirtd
```


### 3.3 创建并启动bridge设备
创建/etc/sysconfig/network-scripts/ifcfg-br0

```
DEVICE=br0
TYPE=Bridge
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=static
PEERDNS=no
DELAY=0
STP=yes
BRIDGING_OPTS=""
NM_CONTROLLED=no
IPADDR=x.x.x.x
NETMASK=255.255.255.0
GATEWAY=x.x.x.x
```

修改/etc/sysconfig/network-scripts/ifcfg-eth0

```
DEVICE=eth0
HWADDR=24:6e:96:27:65:5c
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=static
HOTPLUG=yes
PEERDNS=no
BRIDGE=br0
NM_CONTROLLED=no
```

重启eth0:

```
ifdown eth0
ifup eth0
```

启动br0:

```
ifup br0
```



### 3.4 创建虚拟机：

**基于iso文件创建**

划分磁盘：

```
qemu-img create -f qcow2 /data/centos-6.4.qcow2 10G
```

创建虚拟机（本命令的虚拟机的网络模式基于网桥）

```
virt-install --virt-type kvm --name centos-6.4 --ram 1024 \
--cdrom=/data/CentOS-6.4-x86_64-netinstall.iso \
--disk path=/data/centos-6.4.qcow2,size=10,format=qcow2 \
--network network=default \
--graphics vnc,listen=0.0.0.0 --noautoconsole \
--os-type=linux --os-variant=rhel6

```



**基于现在的虚拟机创建**

拷贝已有虚拟机的disk文件：

```
cp old_centos7.2.qcow2 new_centos7.2.qcow2
```

创建new_centos7.2.xml文件如下：

```
<domain type='kvm'>
<name>new_centos7.2</name>
<memory>16097152</memory>
<currentMemory>16097152</currentMemory>
<vcpu>2</vcpu>
<os>
<type arch='x86_64' machine='pc'>hvm</type>
<boot dev='hd'/>
</os>
<features>
<acpi/>
<apic/>
<pae/>
</features>
<clock offset='localtime'/>
<on_poweroff>destroy</on_poweroff>
<on_reboot>restart</on_reboot>
<on_crash>destroy</on_crash>
<devices>
<disk type='file' device='disk'>
<driver name='qemu' type='qcow2'/>
<source file='/var/new_centos7.2.qcow2'/>
<target dev='vda' bus='virtio'/>
</disk>
<interface type="bridge">
<model type="virtio"/>
<source bridge="br0"/>
</interface>

<input type='tablet' bus='usb'/>
<graphics type='vnc' port='-1' listen = '0.0.0.0' keymap='en-us'/>
</devices>
</domain>
```
启动虚拟机

```
virsh define new_centos7.2.xml
virsh start new_centos7.2
```

查看创建的虚拟机：

```
[root@server-31 var]# virsh list --all
 Id    Name                           State
----------------------------------------------------

 11    centos.7.2                     running
```

/etc/libvirt/qemu目录下会发现centos.7.2.xml，内容如下：


```
<domain type='kvm'>
  <name>centos.7.2</name>
  <uuid>727ade52-43ba-46a0-8865-1198498af22f</uuid>
  <memory unit='KiB'>4194304</memory>
  <currentMemory unit='KiB'>4194304</currentMemory>
  <vcpu placement='static'>1</vcpu>
  <os>
    <type arch='x86_64' machine='pc-i440fx-rhel7.0.0'>hvm</type>
    <boot dev='hd'/>
  </os>
  ...
  <devices>
    <emulator>/usr/libexec/qemu-kvm</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/var/Centos-7.2.qcow2'/>
      <target dev='vda' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
    </disk>
    ...
    <interface type='bridge'>
      <mac address='52:54:00:76:c4:91'/>
      <source bridge='br0'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
    ...
  </devices>
</domain>
```

查看网桥信息

```
root@server-31 qemu]# brctl show
bridge name	bridge id		STP enabled	interfaces
br0		8000.246e9627655c	yes		eth0
                                    vnet0
```


后期可能会因为需求对KVM重新配置，这个时候我们只需要修改对应的xml文件即可

修改配置的时候需要关闭KVM： virsh destroy centos.7.2

使用virsh define centos.7.2.xml使配置生效

使用virsh start centos.7.2启动虚拟机

## 四、虚拟机的上网方式
### 4.1 nat

1.nat模式配置

KVM默认采用nat模式，用户网络（User Networking）：让虚拟机访问主机、互联网或本地网络上的资源的简单方法，但是不能从网络或其他的客户机访问客户机。在公网IP不够使用KVM还需要上网的时候可以使用，大大节省了公网IP！同时这种模式也使得KVM不用暴露在公网之中，也增加了安全性。

下图是虚拟机管理模块产生的接口关系：

![](/img/kvm/kvm-nat.png)

其中virbr0是由宿主机虚拟机支持模块安装时产生的虚拟网络接口，也是一个switch和bridge，负责把内容分发到各虚拟机。

从图上可以看出，虚拟接口和物理接口之间没有连接关系，所以虚拟机只能在通过虚拟的网络访问外部世界，无法从网络上定位和访问虚拟主机。

下面是nat的default配置，可以自定义地址池

```
[root@server-31 networks]# cat /etc/libvirt/qemu/networks/default.xml
<network>
  <name>default</name>
  <uuid>71c7ed8d-08aa-44d0-8f76-cc4fd2dd4380</uuid>
  <forward mode='nat'/>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:91:d4:95'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>
```
确认路由转发开启:

```
[root@server-31 networks]# cat /proc/sys/net/ipv4/ip_forward
1
```

使用virsh net-start default来启动nat网络

### 4.2 bridge

虚拟网桥（Virtual Bridge）：设置好后客户机与互联网，客户机与主机之间都可以通信，试用于需要多个公网IP的环境。

Bridge方式即虚拟网桥的网络连接方式，是客户机和子网里面的机器能够互相通信。可以使虚拟机成为网络中具有独立IP的主机。

桥接网络（也叫物理设备共享）被用作把一个物理设备复制到一台虚拟机。网桥多用作高级设置，特别是主机多个网络接口的情况。

配置步骤见3.3小节


网络配置可以同时存在nat和Bridge.

## 五、QEMU

### 5.1 qemu-img

qemu-img 是 QEMU 的磁盘管理工具，支持多种虚拟镜像格式

```
[root@server-31 ~]# qemu-img -h | grep Support
Supported formats: vvfat vpc vmdk vhdx vdi ssh sheepdog rbd raw host_cdrom host_floppy host_device file qed qcow2 qcow parallels nbd iscsi gluster dmg tftp ftps ftp https http cloop bochs blkverify blkdebug
```

简单介绍一下qcow2和raw格式镜像：

qcow2 是 qcow 的一种改进，是 QEMU 实现的一种虚拟机镜像格式。更小的虚拟硬盘空间（尤其是宿主分区不支持 hole 的情况下），支持压缩、加密、快照功能，磁盘读写性能较 raw 差。


raw是原始的磁盘镜像格式，它的优势在于它非常简单而且非常容易移植到其他模拟器（emulator，QEMU 也是一个 emulator）上去使用。如果客户机文件系统（如 Linux 上的 ext2/ext3/ext4、Windows 的 NTFS）支持“空洞” （hole），那么镜像文件只有在被写有数据的扇区才会真正占用磁盘空间，从而有节省磁盘空间的作用。qemu-img 默认的 raw 格式的文件其实是稀疏文件（sparse file）「稀疏文件就是在文件中留有很多空余空间，留备将来插入数据使用。如果这些空余空间被 ASCII 码的 NULL 字符占据，并且这些空间相当大，那么这个文件就被称为稀疏文件，而且，并不分配相应的磁盘块」，dd 命令创建的也是 raw 格式，不过 dd 一开始就让镜像实际占用了分配的空间，而没有使用稀疏文件的方式对待空洞而节省磁盘空间。尽管一开始就实际占用磁盘空间的方式没有节省磁盘的效果，不过它在写入新的数据时不需要宿主机从现有磁盘空间中分配，从而在第一次写入数据时性能会比稀疏文件的方式更好一点。简单来说，raw 有以下几个特点：

* 寻址简单，访问效率高
* 可以通过格式转换工具方便地转换为其它格式
* 格式实现简单，不支持压缩、快照和加密
* 能够直接被宿主机挂载，不用开虚拟机即可在宿主和虚拟机间进行数据传输

格式转换
以raw转换为qcow2为例

```
qemu-img convert -f raw -O qcow2 test.raw test.qcow2
# -f fmt
# -O output_fmt

```

## 参考
[世民谈云计算](http://www.cnblogs.com/sammyliu/p/4543110.html)  
[KVM虚拟化技术实战与原理解析](https://read.douban.com/ebook/10181845/)  
[枯木的博客](http://blog.opskumu.com/kvm-notes.html)  
[segmentfault](https://segmentfault.com/a/1190000000644069)
