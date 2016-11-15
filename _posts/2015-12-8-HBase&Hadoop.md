---
layout: post
title: HBase原理及基本操作
category: 技术
comments: true
header-img: img/hbase-head.jpg
---

## 什么是HBase?

&emsp;&emsp;HBase是Apache Hadoop中的一个子项目，Hbase依托于Hadoop的HDFS作为最基本存储基础单元，它们之间的关系如图所示：

![hbase-hdfs-relation](/img/2015/12/8/hbase-hdfs-relation.png "hbase-hdfs-relation")<br>

&emsp;&emsp;HBase作为面向列的数据库运行在HDFS之上，它以Google BigTable为蓝本，以键值对的形式存储。项目的目标是快速在主机内数十亿行数据中定位所需的数据并访问它，是一个NoSql的数据库



## 为什么采用HBase？

&emsp;&emsp;HBase不同于一般的关系数据库,它是一个适合于非结构化或半结构化数据存储的数据库.另外HBase是基于列的而不是基于行的模式，这样就非常利于读大数据内容。

1. 结构化数据

&emsp;&emsp;比如我们做一个业务系统，要保存员工基本信息：工号、姓名、性别、出生日期等等；我们就会建立一个对应的staff表。

2. 非结构化数据

&emsp;&emsp;图片、声音、视频和一些没有固定格式的数据。像图片、声音、视频等数据我们通常无法直接知道他的内容，数据库也只能将它保存在一个BLOB字段中，对以后检索非常麻烦。一些没有固定格式的数据如一行有8个字段，之间用逗号分隔，而另一行有6个字段，用空格分隔等，它们之间基本上没有任何联系。

3. 半结构化数据

&emsp;&emsp;这样的数据和上面两种类别都不一样，它是结构化的数据，但是结构变化很大。因为我们要了解数据的细节所以不能将数据简单的组织成一个文件按照非结构化数据处理，由于结构变化很大也不能够简单的建立一个表和他对应。

&emsp;&emsp;先举一个半结构化的数据的例子，比如存储员工的简历。不像员工基本信息那样一致，每个员工的简历大不相同。有的员工的简历很简单，比如只包括教育情况；有的员工的简历却很复杂，比如包括工作情况、婚姻情况、出入境情况、户口迁移情况、党籍情况、技术技能等等，还有可能有一些我们没有预料的信息。通常我们要完整的保存这些信息并不是很容易的，因为我们不会希望系统中的表的结构在系统的运行期间进行变更。

***
## 什么是列存储？

&emsp;&emsp;列存储不同于传统的关系型数据库，其数据在表中是按行存储的，列方式所带来的重要好处之一就是，由于查询中的选择规则是通过列来定义的，因此整个数据库是自动索引化的。按列存储每个字段的数据聚集存储，在查询只需要少数几个字段的时候，能大大减少读取的数据量，一个字段的数据聚集存储，那就 更容易为这种聚集存储设计更好的压缩/解压算法。这张图讲述了传统的行存储和列存储的区别：

![column-storage](/img/2015/12/8/column-storage.png "column-storage")<br>

***
## HBase的数据类型

&emsp;&emsp;HBase是一个稀疏的、分布式的、持久化存储的多维度排序Map，Map的索引是行关键字、列关键字以及时间戳；Map中的每个value都是一个未经解析的byte数组:

&emsp;&emsp;(row:string, column:string,time:int64)->string

&emsp;&emsp;举个具体的例子，假设我们想要存储海量的网页及相关信息，我们姑且称这个特殊的表为Webtable，在Webtable里，我们使用URL作为行关键字，使用网页的某些属性作为列名，网页的内容存在“contents:”列中，并用获取该网页的时间戳作为标识

&emsp;&emsp;如图所示：

&emsp;&emsp;![hbase-storage-website](/img/2015/12/8/hbase-storage-website.jpg "hbase-storage-website")<br>

&emsp;&emsp;这是一个存储Web网页的例子的表的片断，行名是一个反向URL，contents列族存放的是网页的内容，anchor列族存放引用该网页的锚链接文本。CNN的主页被Sports Illustrater和MY-look的主页引用，因此包含了名为“anchor:cnnsi.com”和“anchhor:my.look.ca”的列。每个锚链接只有一个版本；而contents列则有三个版本，分别由时间戳t3，t5，和t6标识。

***
## HBase数据类型的几个基本概念

1. 行
 
&emsp;&emsp;表中的行关键字可以是任意的字符串（目前支持最大64KB的字符串，但是对大多数用户，10-100个字节就足够了）。对同一个行关键字的读或者写操作都是原子的（不管读或者写这一行里多少个不同列）。

&emsp;&emsp;HBase通过行关键字的字典顺序来组织数据，用户可以通过选择合适的行关键字，在数据访问时有效利用数据的位置相关性。举例来说，在Webtable里，通过反转URL中主机名的方式，可以把同一个域名下的网页聚集起来组织成连续的行，行关键字为com.google.maps/index.html和com.google.maps/information.html很可能存储在连续的区域。把相同的域中的网页存储在连续的区域可以让基于主机和域名的分析更加有效。

2. 族

&emsp;&emsp;列关键字组成的集合叫做“列族”，列族是访问控制的基本单位。存放在同一列族下的所有数据通常都属于同一个类型（我们可以把同一个列族下的数据压缩在一起）。列族在使用之前必须先创建，然后才能在列族中任何的列关键字下存放数据；列族创建后，其中的任何一个列关键字下都可以存放数据。一张表中的列族不能太多（最多几百个），并且列族在运行期间很少改变。与之相对应的，一张表可以有无限多个列

&emsp;&emsp;比如，Webtable有个列族language，language列族用来存放撰写网页的语言。我们在language列族中只使用一个列关键字，用来存放每个网页的语言标识ID。Webtable中另一个有用的列族是anchor；这个列族的每一个列关键字代表一个锚链接，如图一所示。Anchor列族的限定词是引用该网页的站点名；Anchor列族每列的数据项存放的是链接文本。

3. 时间戳

&emsp;&emsp;在HBase中，表的每一个数据项都可以包含同一份数据的不同版本；不同版本的数据通过时间戳来索引。HBase时间戳的类型是64位整型。HBase可以给时间戳赋值，用来表示精确到毫秒的“实时”时间；用户程序也可以给时间戳赋值。如果应用程序需要避免数据版本冲突，那么它必须自己生成具有唯一性的时间戳。数据项中，不同版本的数据按照时间戳倒序排序，即最新的数据排在最前面。

***
## HBase对半结构化数据的存储

&emsp;&emsp;对于两条记录：

     name:Tom,math:90,english:88;
     name:Jim,math:66,english:80;

&emsp;&emsp;在Mysql中他们可以这样存：

![mysql-storage-data](/img/2015/12/8/mysql-storage-data.png "mysql-storage-data")<br>

&emsp;&emsp如果一条新纪录为：name:Jack,age:14,math:100,art:97

&emsp;&emsp;不能存入刚才设计的数据库，因为schema定义了name,math,english之后，列是固定的，如果要插入到这张表中必须修改Schema,并且在项目上线后表字段是不能动态增加的。

&emsp;&emsp;但如果使用Hbase,对于上面的三条记录

![hbase-storage-data](/img/2015/12/8/hbase-storage-data.png "hbase-storage-data")<br>

&emsp;&emsp;hbase表中的每个列，都归属与某个列族。列族是表schema的一部分(而列不是)。列名都以列族作为前缀。例如course:math,course:art 都属于course这个列族。访问控制、磁盘和内存的使用统计都是在列族层面进行的,列族由任意多列组成，无需预先定义列的数量以及类型，列族支持列的动态扩展。在列导向的存储机制下对于Null值得存储是不占用任何空间的.而mysql中的Null是占用空间的

&emsp;&emsp;下面是来自于MYSQL官方的解释

![mysql-storage-null](/img/2015/12/8/mysql-storage-null.png "mysql-storage-null")<br>

***
## HBase的存储方式

&emsp;&emsp;当表的大小超过设置值得时候，HBase会自动地将表划分为不同的区域，每个区域包含所有行的一个子集。对用户来说，每个表是一堆数据的集合，靠主键来区分。从物理上来说，一张表被拆分成了多块，每一块就是一个HRegion。我们用表名+开始/结束主键([startkey, endkey])来区分每一个HRegion，一个HRegion会保存一个表里面某段连续的数据，从开始主键到结束主键，一张完整的表格是保存在多个HRegion上面。


&emsp;&emsp;不同的region会被Master分配给响应的RegionServer进行管理,如下图所示：

![hbase-storage-mode](/img/2015/12/8/hbase-storage-mode.jpg "hbase-storage-mode")<br>

&emsp;&emsp;所有的数据库数据一般都是保存在Hadoop分布式文件系统上面，用户通过一系列HRegion服务器获取这些数据，一台机器上面一般只运行一个HRegionServer。每一个HRegion块在物理上会被分为三个部分：Hmemcache(缓存)、Hlog(日志)、HStore(持久层)

![hreserver-hregion](/img/2015/12/8/hreserver-hregion.png "hreserver-hregion")<br>
     
***
## HBase的访问接口
1. Native Java API，最常规和高效的访问方式，适合Hadoop MapReduce Job并行批处理HBase表数据

2. HBase Shell，HBase的命令行工具，最简单的接口，适合HBase管理使用

3. Thrift Gateway，利用Thrift序列化技术，支持C++，PHP，Python等多种语言，适合其他异构系统在线访问HBase表数据

4. REST Gateway，支持REST 风格的Http API访问HBase, 解除了语言限制

5. Pig，可以使用Pig Latin流式编程语言来操作HBase中的数据，和Hive类似，本质最终也是编译成MapReduce Job来处理HBase表数据，适合做数据统计

6. Hive，当前Hive的Release版本尚没有加入对HBase的支持，但在下一个版本Hive 0.7.0中将会支持HBase，可以使用类似SQL语言来访问HBase

***
## HBase的优缺点

&emsp;&emsp;缺点：不支持条件查询以及orderby等查询

&emsp;&emsp;优点：

1. 列可以动态增加，列为空则不存储数据，节省存储空间

2. 会自动切分数据

3. 可以提供高并发读写操作的支持

&emsp;&emsp;需要注意的一点：HBase在存储时以表的形式存储数据，数据按照Row key的字典序(byte order)排序存储。设计key时，要充分排序存储这个特性，将经常一起读取的行存储放到一起。(位置相关性) 

***
## 什么情况下需要HBase

1. 半结构化或非结构化数据

&emsp;&emsp;对于数据结构字段不够确定或杂乱无章很难按一个概念去进行抽取的数据适合用HBase。

2. 记录非常稀疏

&emsp;&emsp;RDBMS的行有多少列是固定的，为null的列浪费了存储空间。HBase为null的Column不会被存储，这样既节省了空间又提高了读性能。

3. 多版本数据

&emsp;&emsp;根据Row key和Column key定位到的Value可以有任意数量的版本值，因此对于需要存储变动历史记录的数据，用HBase就非常方便了。比如某表的Address是会变动的，业务上一般只需要最新的值，但有时可能需要查询到历史值。

4. 超大数据量

&emsp;&emsp;当数据量越来越大，RDBMS数据库撑不住了，就出现了读写分离策略，通过一个Master专门负责写操作，多个Slave负责读操作，服务器成本倍增。随着压力增加，Master又撑不住了，这时就要分库了，把关联不大的数据分开部署，一些join查询不能用了，需要借助中间层。随着数据量的进一步增加，一个表的记录越来越大，查询就变得很慢，于是又得搞分表，比如按ID取模分成多个表以减少单个表的记录数。采用HBase就简单了，只需要加机器即可，HBase会自动水平切分扩展，跟Hadoop的无缝集成保障了其数据可靠性（HDFS）和海量数据分析的高性能（MapReduce）。

***
## HBase shell基本操作

1. 建立一个表ScoreTable，有两个列族student和course

     ![hbase-create-table](/img/2015/12/8/hbase-create-table.png "hbase-create-table")<br>

2. 使用list命令来查看当前HBase里有哪些表

     ![hbase-table-list](/img/2015/12/8/hbase-table-list.png "hbase-table-list")<br>

3. 按设计的表结构插入值

     命令：put '表名','主键','列名','value'

     ![hbase-put](/img/2015/12/8/hbase-put.png "hbase-put")<br>

4. 查某一行记录

     命令：get '表名','行关键字'

     ![hbase-get-onerow](/img/2015/12/8/hbase-get-onerow.png "hbase-get-onerow")<br>

5. 查某一行的某一列族记录

     命令：get '表名','行关键字'，'列族名'

     ![get-table-row-column](/img/2015/12/8/get-table-row-column.png "get-table-row-column")<br>

6. 查看表中的所有数据

     命令：scan '表名'

     ![hbase-scan-table](/img/2015/12/8/hbase-scan-table.png "hbase-scan-table")<br>

7. 查看表中的某一列族的所有数据

     命令：scan '表名',{COLUMN=>'列族名'}

     ![hbase-scan-column](/img/2015/12/8/hbase-scan-column.png "hbase-scan-column")<br>

8. 扫描表查看特定的数据
 
     命令：scan '表名',{COLUMN=>'列族名',LIMIT =>num, STARTROW => '行关键字'}

     ![hbase-scan-filter](/img/2015/12/8/hbase-scan-filter.png "hbase-scan-filter")<br>
