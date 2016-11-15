---
layout: post
title: HDFS原理
date:       2015-12-8
author:     xue
tags:
    - hadoop
---

## Hadoop与HDFS

&emsp;&emsp;Hadoop是一个由Apache基金会所开发的分布式系统基础架构。
用户可以在不了解分布式底层细节的情况下，开发分布式程序。充分利用集群的威力进行高速运算和存储。    

&emsp;&emsp;Hadoop实现了一个分布式文件系统（Hadoop Distributed File System），简称HDFS

&emsp;&emsp;Hadoop经典教材：\<\<Hadoop实战\>\>中写到:

![what-is-hdfs](/img/2015/12/8/what-is-hdfs.png "what-is-hdfs")<br>

&emsp;&emsp;Hadoop的框架最核心的设计就是：HDFS和MapReduce。HDFS为海量的数据提供了存储，则MapReduce为海量的数据提供了计算。

***
## HDFS的设计理念

1. 存储超大文件

    这里的“超大文件”是指几百MB、GB甚至TB级别的文件。

2. 最高效的访问模式是一次写入、多次读取(流式数据访问)

    HDFS存储的数据集作为hadoop的分析对象。在数据集生成后，长时间在此数据集上进行各种分析。每次分析都将设计该数据集的大部分数据甚至全部数据，因此读取整个数据集的时间延迟比读取第一条记录的时间延迟更重要。

3. 运行在普通廉价的服务器上

    HDFS设计理念之一就是让它能运行在普通的硬件之上，即便硬件出现故障，也可以通过容错策略来保证数据的高可用。

## HDFS不适合哪些场景

1. 将HDFS用于对数据访问要求低延迟的场景

     由于HDFS是为高数据吞吐量应用而设计的，必然以高延迟为代价。

2. 存储大量小文件
 
     HDFS中元数据（文件的基本信息）存储在namenode的内存中，而namenode为单点，小文件数量大到一定程度，namenode内存就吃不消了。

***
## HDFS的一些基本概念

&emsp;&emsp;数据块（block）：大文件会被分割成多个block进行存储，block大小默认为64MB。每一个block会在多个datanode上存储多份副本，默认是3份。

&emsp;&emsp;namenode：namenode负责管理文件目录、文件和block的对应关系以及block和datanode的对应关系。

&emsp;&emsp;datanode：datanode就负责存储了，当然大部分容错机制都是在datanode上实现的。

&emsp;&emsp;Secondary NameNode:从元数据节点。从元数据节点并不是NameNode出现问题时候的备用节，它是NameNode的冷备份，意思是NameNode如果坏掉，那么SecondaryNameNode不能马上代替namenode工作。但是SecondaryNameNode上存储namenode的一些信息，减少namenode坏掉之后的损失。

&emsp;&emsp;关系图如下：

![datanode-namenode-relation](/img/2015/12/8/datanode-namenode-relation.jpg "datanode-namenode-relation")<br>

***
##HDFS的副本机制
&emsp;&emsp;大文件会被分割成多个block进行存储，block大小默认为64MB。每一个block会在多个datanode上存储多份副本，默认是3份。

![replication-data](/img/2015/12/8/replication-data.png "replication-data")<br>

&emsp;&emsp;上图中显示了两个数据文件，一个位于/user/chuck/data1，一个位于/user/james/data2。data1被分成了三个数据块，分别用数字1,2,3表示。data2分成的数据块由4,5来表示。这些文件的内容分散在几个DataNode上。

##副本放置机制
&emsp;&emsp;HDFS运行在跨越大量机架的集群之上。两个不同机架上的节点是通过交换机实现通信的，在大多数情况下，相同机架上机器间的网络带宽优于在不同机架上的机器。所以如果将将副本放置在不同的机架上，最大化了可靠性，但是增加了写的成本，因为写的时候需要跨越多个机架传输文件块。而副本如果都在同一机架上，那么数据可靠应又得不到保障。

&emsp;&emsp;默认的HDFS block放置策略在最小化写开销和最大化数据可靠性、可用性以及总体读取带宽之间进行了一些折中，采取的策略是：

&emsp;&emsp;1/3在同一个节点上，1/3副本在同一个机架上，另外1/3均匀地分布在其他机架上。这种方式提高了写的性能，并且不影响数据的可靠性和读性能。

##心跳机制
&emsp;&emsp;一个数据节点周期性发送一个心跳包到名字节点。网络断开会造成一组数据节点子集和名字节点失去联系。名字节点根据缺失的心跳信息判断故障情况。名字节点将这些数据节点标记为死亡状态，不再将新的IO请求转发到这些数据节点上，这些数据节点上的数据将对HDFS不再可用，可能会导致一些块的复制因子降低，一旦发现就启动复制操作。

***
##HDFS写操作原理
![hdfs-write](/img/2015/12/8/hdfs-write.jpg "hdfs-write")<br>

有一个文件FileA，100M大小。Client将FileA写入到HDFS上。HDFS按默认配置。

HDFS分布在三个机架上Rack1，Rack2，Rack3。

   1. Client将FileA按64M分块。分成两块，block1和Block2;

   2. Client向nameNode发送写数据请求，如图蓝色虚线①------>。

   3. NameNode节点，记录block信息。并返回可用的DataNode，如粉色虚线②--------->。
      Block1: host2,host1,host3
      Block2: host7,host8,host4
    
   4. client向DataNode发送block1；发送过程是以流式写入。

流式写入过程：

   1. 将64M的block1按64k的package划分;

   2. 然后将第一个package发送给host2;

   3. host2接收完后，将第一个package发送给host1，同时client想host2发送第二个package；

   4. host1接收完第一个package后，发送给host3，同时接收host2发来的第二个package。

   5. 以此类推，如图红线实线所示，直到将block1发送完毕。

   6. host2,host1,host3向NameNode，host2向Client发送通知，说“消息发送完了”。如图粉红颜色实线所示。

   7. client收到host2发来的消息后，向namenode发送消息，说我写完了。这样就真完成了。如图黄色粗实线

   8. 发送完block1后，再向host7，host8，host4发送block2，如图蓝色实线所示。

   9. 发送完block2后，host7,host8,host4向NameNode，host7向Client发送通知，如图浅绿色实线所示。

   10. client向NameNode发送消息，说我写完了，如图黄色粗实线。这样FileA就写入完成了。

***
##HDFS读操作原理
![hdfs-read](/img/2015/12/8/hdfs-read.jpg "hdfs-read")<br>
读操作就简单一些了，如图所示，client要从datanode上，读取FileA。而FileA由block1和block2组成。 

操作流程为：

   1. client向namenode发送读请求

   2. namenode查看Metadata信息，返回fileA的block的位置。
     block1:host2,host1,host3
     block2:host7,host8,host4

   3. block的位置是有先后顺序的，先读block1，再读block2。而且block1去host2上读取；然后block2，去host7上读取；
   上面例子中，client位于机架外，那么如果client位于机架内某个DataNode上，例如,client是host6。那么读取的时候，遵循的规律是：优选读取本机架上的数据。

##HDFS访问方式

&emsp;&emsp;HDFS给应用提供了多种访问方式。用户可以通过Java API接口访问，也可以通过C语言的封装API访问，还可以通过浏览器的方式访问HDFS中的文件。

1. DFSShell
    HDFS以文件和目录的形式组织用户数据。它提供了一个命令行的接口(DFSShell)让用户与HDFS中的数据进行交互。命令的语法和用户熟悉的其他shell(例如 bash, csh)工具类似。
    
2. 浏览器接口
    一个典型的HDFS安装会在一个可配置的TCP端口开启一个Web服务器用于暴露HDFS的名字空间。用户可以用浏览器来浏览HDFS的名字空间和查看文件的内容。
    
3. Java API
   HDFS提供为程序提供java api，为c语言包装的java api也是可用的

##HDFS的优缺点
优点：处理超大文件、流式的访问数据、运行于廉价的商用机器集群上

缺点：

   1. 不适合低延迟数据访问

&emsp;&emsp;如果要处理一些用户要求时间比较短的低延迟应用请求，则HDFS不适合。
  
&emsp;&emsp; 改进策略：对于那些有低延时要求的应用程序，HBase是一个更好的选择



   2. 无法高效存储大量小文件
   
&emsp;&emsp;因为Namenode把文件系统的元数据放置在内存中，所以文件系统所能容纳的文件数目是由Namenode的内存大小来决定。一般来说，每一个文件、文件夹和Block需要占据150字节左右的空间，所以，如果你有100万个文件，每一个占据一个Block，你就至少需要300MB内存。当前来说，数百万的文件还是可行的，当扩展到数十亿时，对于当前的硬件水平来说就没法实现了。
　　

   3. 不支持多用户写入及任意修改文件
   
&emsp;&emsp;在HDFS的一个文件中只有一个写入者，而且写操作只能在文件末尾完成，即只能执行追加操作。目前HDFS还不支持多个用户对同一文件的写操作，以及在文件任意位置进行修改。

##HDFS特性

&emsp;&emsp;Hadoop（包括HDFS）非常适合在商用硬件（commodity hardware）上做分布式存	储和计算，因为它不仅具有容错性和可扩展性，而且非常易于扩展。Map-Reduce框架以其在大型分布式系统应用上的简单性和可用性而著称，这个框架已经被集成进Hadoop中。
  
&emsp;&emsp;HDFS的可配置性极高，同时，它的默认配置能够满足很多的安装环境。多数情况下，这些参数只在非常大规模的集群环境下才需要调整。
 用Java语言开发，支持所有的主流平台，支持类Shell命令，可直接和HDFS进行交互。
 NameNode和DataNode有内置的Web服务器，方便用户检查集群的当前状态。
 
&emsp;&emsp;还有一些的新特性：
 
&emsp;&emsp;1. 文件权限和授权
 
&emsp;&emsp;2. 机架感知（Rack awareness）：在调度任务和分配存储空间时考虑节点的物理位置

&emsp;&emsp;3.安全模式：一种维护需要的管理模式。

&emsp;&emsp;4. fsck：一个诊断文件系统健康状况的工具，能够发现丢失的文件或数据块。
    
&emsp;&emsp;5. Rebalancer：当datanode之间数据不均衡时，平衡集群上的数据负载。
    
&emsp;&emsp;6. 升级和回滚：在软件更新后有异常发生的情形下，能够回滚到HDFS升级之前的状态      
    
&emsp;&emsp;7. Secondary Namenode：对文件系统名字空间执行周期性的检查点，将Namenode上HDFS改动日志文件的大小控制在某个特定的限度下。
