---
layout: post
title: 虚拟化
date:       2015-12-24
author:     xue
catalog:    true
tags:
    - 虚拟化
---
## 虚拟化概述

&emsp;&emsp;对云计算而言，特别是提供基础架构即服务的云计算，更关心的是硬件抽象层上的虚拟化。因为，只有把物理计算机系统虚拟化为多台虚拟计算机系统，通过网络将这些虚拟计算机系统互联互通，才能够形成现代意义上的基础架构即服务云计算系统。  
&emsp;&emsp;硬件上的虚拟化又被称为系统虚拟化，是指将一台物理计算机系统虚拟化为一台或多台虚拟硬件，如CPU、内存和设备等，并提供一个独立的虚拟机执行环境。通过虚拟机监控器（Virtual Machine Monitor,VMM,也可以称为Hypervisor）的模拟，虚拟机中的操作系统（Guest OS,客户端操作系统）认为自己仍然是独占一个系统在运行。在一台物理计算机上运行的每个虚拟机中的操作系统是可以完全不同的，并且它们的执行环境是完全独立的。  
&emsp;&emsp;典型的虚拟化产品有：

- VMware
- Microsoft
- Xen
- KVM

## 虚拟机的实现方式

&emsp;&emsp;当前两主流的虚拟化按照实现方式可以分为两种：  

- VMM直接运行在硬件平台上，控制所有硬件并管理客户操作系统。客户操作系统运行在比VMM更高的级别。这个模型也是虚拟化历史里的经典模型，很多著名虚拟机都是根据这个模型来实现的，比如说Xen。
- VMM运行在一个传统的操作系统里（第一层软件），可以把这个VMM看作是第二层软件层，而客户机操作系统则是第三软件层。KVM和VirtualBox就是这种实现。

&emsp;&emsp;两种实现方式的具体区别如下图所示：

![different-way-of-achieving-virtualization](/img/2015/12/24/hypertype-1.jpg "different-way-of-achieving-virtualization")<br>

## 高层管理工具

&emsp;&emsp;VMM本身只是提供了虚拟化的基础机构，很多和最终用户相关的工作，比如配置、启动虚拟机，都要作为高层的管理工具来实现。这些管理哦给你工具存在于宿主机中，一般包括各种应用程序和库文件。

### XenAPI

&emsp;&emsp;XenAPI是由Xen社区开发出来的一套接口，用于远程或本地配置和管理Xen的虚拟机，工作在比Xen更高级的层面，比如集群、网络、存储等。在很多前空下，XenAPI也被看作一整套的工具栈（Tool Stack),包括了管理接口正常工作需要的所有组件，比如一个守护进程（Daemon）、xe命令行工具等。

### Libvirt


&emsp;&emsp;Libvirt是由Redhat开发的一套开源的软件工具，目标是提供一个通用和稳定的软件库来高效、安全地管理一个节点上的虚拟机，并支持远程操作。它由以下模块组成：

- 一个库文件，实现管理接口
- 一个守护进程（libvirtd）
- 一个命令行工具


&emsp;&emsp;基于可移植性和高可靠性的考虑，Libvirt采用C语言开发，但是也提供了对其他编程语言的绑定，包括Python,Perl，OCaml,Ruby,Java和PHP。因此Libvirt的调用可以被集成到各种编程语言中，适合不同的环境。另一方面，不像XenAPI只管理Xen,Libvirt支持多种VMM，包括：

- LXC,轻量级的Linux容器（Container）
- OpenVZ,基于Linux内核的轻量级Linux容器（Container）
- KVM/QEMU,基于Linux的类型2的VMM
- Xen,开源的类型1的VMM
- User-mode Linux(UML)，系统调用级别的Linux虚拟机
- VirtualBox,Oracle开发的类型2的VMM，可以运行在Windows,Linux和Mac OS X上。
- VMware ESX and GSX，VMware虚拟化的服务器版本
- VMware Workstation and Player,VMware虚拟化的桌面版本呢
- Hyper-V,微软开发的VMM
- PowerVM,IBM开发的VMM，可以运行在AIX和Linux上
- Parallels Workstation,Parallels为Mac开发的VMM
- Bhyve,FreeBSD 9+上的VMM


&emsp;&emsp;为了支持多种的VMM，Libvirt采用了基于驱动（Driver）的框架，如下图所示。也就是说，每种VMM需要听一个Driver,和Libvirt进行通信来操控特定的VMM。这也意味着，通用的Libvirt提供的API和某种VMM可能不完全一样。

![libvirt_architecture](/img/2015/12/24/libvirt_architecture.png "libvirt_architecture")<br>
