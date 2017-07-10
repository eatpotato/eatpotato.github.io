---
layout:     post
title:      keystone与ldap的集成
date:       2017-07-08
author:     xue
catalog:    true
tags:
    - openstack
    - keystone
---

以kilo版本的openstack为例

## ldap简介

LDAP是轻量目录访问协议(Lightweight Directory Access Protocol)的缩写，LDAP是从X.500目录访问协议的基础上发展过来的。

#### ldap的特点

* LDAP的结构用树来表示，而不是用表格。正因为这样，就不能用SQL语句了
* LDAP可以很快地得到查询结果，不过在写方面，就慢得多
* LDAP提供了静态数据的快速查询方式
* Client/server模型，Server 用于存储数据，Client提供操作目录信息树的工具
* 这些工具可以将数据库的内容以文本格式（LDAP 数据交换格式，LDIF）呈现
* LDAP是一种开放Internet标准，LDAP协议是跨平台的Interent协议

#### ldap基本概念

LDAP的一张示例图如下：

![LDAP示例图](/img/keystone/ldap-example.png)

**Entry**

条目，也叫记录项，是LDAP中最基本的颗粒，就像字典中的词条，或者是数据库中的记录。通常对LDAP的添加、删除、更改、检索都是以条目为基本对象的。

dn：每一个条目都有一个唯一的标识名（distinguished Name ，DN），如上图的一条记录为dn："cn=baby,ou=marketing,ou=people,dc=mydomain,dc=org" 。通过DN的层次型语法结构，可以方便地表示出条目在LDAP树中的位置，通常用于检索。


**Attribute**

  每个条目都可以有很多属性（Attribute），比如常见的人都有姓名、地址、电话等属性。每个属性都有名称及对应的值，属性值可以有单个、多个。比如你有多个电话。
  
  LDAP为人员组织机构中常见的对象都设计了属性(比如commonName，surname)。下面有一些常用的别名：
  
 
|属性|别名|语法|描述|值（举例）|
|--|--|--|--|--|
|commonName|cn|Directory String|姓名|baby|
|surname|sn|Directory String|姓|yang|
|organizationalUnitName|ou|Directory String|单位|Tech|
|organization|o|Directory String|组织|school|
|telephoneNumber||Telephone Number|电话号码|114|
|owner||DN|该条目的拥有者|cn=baby,ou=marketing,ou=people,dc=mydomain,dc=org|

**ObjectClass**

对象类（ObjectClass）是属性的集合，LDAP预想了很多人员组织机构中常见的对象，并将其封装成对象类。比如人员（person）含有姓（sn）、名（cn）、电话(telephoneNumber)、密码(userPassword)等属性。

单位职工(organizationalPerson)是人员(person)的继承类，除了上述属性之外还含有职务（title）、邮政编码（postalCode）、通信地址(postalAddress)等属性。

通过对象类可以方便的定义条目类型。每个条目可以直接继承多个对象类，这样就继承了各种属性。如果2个对象类中有相同的属性，则条目继承后只会保留1个属性。对象类同时也规定了那些属性是基本信息，必须含有(Must 或Required，必要属性)；哪些属性是扩展信息，可以含有（May或Optional，可选属性）。

**Schema**

Schema是一个数据模型，它被用来决定数据怎样被存储，被跟踪的数据的是什么类型，存储在不同的Entry下的数据之间的关系。


## centos服务器上搭建ldap

1.安装软件包

```
 yum install -y openldap openldap-clients openldap-servers migrationtools 
```

2.配置 OpenLDAP Server

```
vim /etc/openldap/slapd.d/cn\=config/olcDatabase\=\{2\}hdb.ldif
```

修改三行内容,密码根据自己需要修改：

![](/img/ldap/openldap-server-config.png)

3.更改配置 Monitoring Database

```
vim /etc/openldap/slapd.d/cn\=config/olcDatabase\=\{1\}monitor.ldif
```

修改下面的内容:

![](/img/ldap/monitor-database.png)

4.准备ldap数据库

```
cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
```

5.测试配置文件是否正确

```
slaptest -u 
```

6.启动服务

```
systemctl enable slapd
systemctl start sladp
```

7.将所有的配置LDAP server, 添加到LDAP schemas中

```
cd /etc/openldap/schema/
for i in `ls | grep ldif`; do ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f $i; done
```

8.生成文件base.ldif

```
cat >/root/base.ldif << EOF
# extended LDIF
#
# LDAPv3
# base <dc=openstack,dc=org> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#
# openstack.org
dn: dc=openstack,dc=org
objectClass: dcObject
objectClass: organizationalUnit
dc: openstack
ou: openstack

# UserGroups, openstack.org
dn: ou=groups,dc=openstack,dc=org
objectClass: organizationalUnit
ou: groups

# Users, openstack.org
dn: ou=users,dc=openstack,dc=org
objectClass: organizationalUnit
ou: users

# Roles, openstack.org
dn: ou=roles,dc=openstack,dc=org
ou: roles
objectClass: organizationalUnit

# Tenants, openstack.org
dn: ou=tenants,dc=openstack,dc=org
ou: tenants
objectClass: organizationalUnit
EOF
```

9.导入基本模板

```
ldapadd -x -D"cn=admin,dc=openstack,dc=org" -W -f base.ldif
```

10.查看entry是否添加成功

```
[root@localhost ~]# ldapsearch -x -LLL -Hldap:/// -b dc=openstack,dc=org dn
dn: dc=openstack,dc=org

dn: ou=groups,dc=openstack,dc=org

dn: ou=users,dc=openstack,dc=org

dn: ou=roles,dc=openstack,dc=org

dn: ou=tenants,dc=openstack,dc=org
```

## keystone与ldap的集成
以Mitaka版本openstack为例

先列一下踩的坑：

问题一. ERROR keystone UNDEFINED_TYPE: {'info': 'enabled: attribute type undefined', 'desc': 'Undefined attribute type'}

解决方法：

修改ldap schema:

vim /etc/openldap/schema/core.schema

```
# 修改以下代码 
objectclass ( 2.5.6.9 NAME 'groupOfNames' DESC 'RFC2256: a group of names (DNs)' SUP top STRUCTURAL MUST ( member $ cn ) MAY ( businessCategory $ seeAlso $ owner $ ou $ o $ description $ enabled ) ) 

# 添加下面的内容
attributetype ( 2.5.4.66 NAME 'enabled' DESC 'RFC2256: enabled of a group' EQUALITY booleanMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.7 SINGLE-VALUE )

```

vim /etc/openldap/schema/inetorgperson.schema 

```
# 修改以下代码 
objectclass ( 2.16.840.1.113730.3.2.2 NAME 'inetOrgPerson' DESC 'RFC2798: Internet Organizational Person' SUP organizationalPerson STRUCTURAL MAY ( audio $ businessCategory $ carLicense $ departmentNumber $ displayName $ employeeNumber $ employeeType $ givenName $ homePhone $ homePostalAddress $ initials $ jpegPhoto $ labeledURI $ mail $ manager $ mobile $ o $ pager $ photo $ roomNumber $ secretary $ uid $ userCertificate $ x500uniqueIdentifier $ preferredLanguage $ userSMIMECertificate $ userPKCS12 $ description $ enabled $ email ) ) 

```

重启ldap服务

```
service slapd restart
```


问题二.

```
OBJECT_CLASS_VIOLATION: {'info': "object class 'groupOfNames' requires attribute 'member'", 'desc': 'Object class violation'}
```

解决方法：

vim /etc/keystone/keystone.conf

```
use_dumb_member = True

dumb_member = cn=dumb,dc=openstack,dc=org
```

问题三.

```
 ERROR keystone   File "/usr/bin/keystone-manage", line 10, in <module>
 ERROR keystone     sys.exit(main())
 ERROR keystone   File "/usr/lib/python2.7/site-packages/keystone/cmd/manage.py", line 46, in main
 ERROR keystone     cli.main(argv=sys.argv, config_files=config_files)
 ERROR keystone   File "/usr/lib/python2.7/site-packages/keystone/cmd/cli.py", line 1024, in main
 ERROR keystone     CONF.command.cmd_class.main()
 ERROR keystone   File "/usr/lib/python2.7/site-packages/keystone/cmd/cli.py", line 375, in main
 ERROR keystone     klass.do_bootstrap()
 ERROR keystone   File "/usr/lib/python2.7/site-packages/keystone/cmd/cli.py", line 222, in do_bootstrap
 ERROR keystone     enabled = user['enabled']
 ERROR keystone KeyError: 'enabled'
```

解决方法：

vim /etc/keystone/keystone.conf

```
user_enabled_emulation = True
```

重启httpd服务

下面是keystone与ldap集成的过程：

修改keystone.conf

```
[identity]
driver = keystone.identity.backends.ldap.Identity
[ldap]
#如何ldap不是本机，将localhost改为相应ip即可
url = ldap://localhost
user = cn=admin,dc=openstack,dc=org
#修改密码为相应的值
password = xxx
suffix = dc=openstack,dc=org
use_dumb_member = True
dumb_member = cn=dumb,dc=openstack,dc=org
query_scope = one
user_tree_dn = ou=users,dc=openstack,dc=org
user_enabled_invert = False
user_enabled_default = True
user_allow_create = True
user_allow_update = True
user_allow_delete = True
user_enabled_emulation = True
group_tree_dn = ou=groups,dc=openstack,dc=org
group_allow_create = False
group_allow_update = False
group_allow_delete = False
use_tls = False
tls_req_cert = demand
use_pool = False
pool_size = 10
pool_retry_max = 3
pool_retry_delay = 0.1
pool_connection_timeout = -1
pool_connection_lifetime = 600
use_auth_pool = False
auth_pool_size = 100
auth_pool_connection_lifetime = 60
```

重启

接着创建nova、glance等用户.具体的操作可以参考[官方安装手册](https://docs.openstack.org/mitaka/install-guide-rdo/keystone-users.html)


对接完之后结果如下：

```
[root@localhost ~]# openstack user list
+----------------------------------+---------+
| ID                               | Name    |
+----------------------------------+---------+
| 2b68a7415f7b41babc8b493b83878286 | admin   |
| 2ed4fc05512a46bfbba22d20a2928e29 | neutron |
| f2b302600645405bbeacbd563e39467f | nova    |
| b81233ec879742dfb063be73e61a4058 | glance  |
| da6a7e4ebd924d1b81e21b4513a30d56 | cinder  |
| 21aabb86372942229b92eb41c27c911a | demo    |
```

查看ldap记录

```
[root@localhost ~]# ldapsearch -x -b 'dc=openstack,dc=org' '(objectclass=*)'
# extended LDIF
#
# LDAPv3
# base <dc=openstack,dc=org> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# openstack.org
dn: dc=openstack,dc=org
objectClass: dcObject
objectClass: organizationalUnit
dc: openstack
ou: openstack

# groups, openstack.org
dn: ou=groups,dc=openstack,dc=org
objectClass: organizationalUnit
ou: groups

# users, openstack.org
dn: ou=users,dc=openstack,dc=org
objectClass: organizationalUnit
ou: users

# roles, openstack.org
dn: ou=roles,dc=openstack,dc=org
ou: roles
objectClass: organizationalUnit

# tenants, openstack.org
dn: ou=tenants,dc=openstack,dc=org
ou: tenants
objectClass: organizationalUnit

# 2b68a7415f7b41babc8b493b83878286, users, openstack.org
dn: cn=2b68a7415f7b41babc8b493b83878286,ou=users,dc=openstack,dc=org
objectClass: person
objectClass: inetOrgPerson
sn: admin
cn: 2b68a7415f7b41babc8b493b83878286
mail: root@localhost
userPassword:: xxx

# enabled_users, users, openstack.org
dn: cn=enabled_users,ou=users,dc=openstack,dc=org
objectClass: groupOfNames
member: cn=2b68a7415f7b41babc8b493b83878286,ou=users,dc=openstack,dc=org
member: cn=2ed4fc05512a46bfbba22d20a2928e29,ou=users,dc=openstack,dc=org
member: cn=f2b302600645405bbeacbd563e39467f,ou=users,dc=openstack,dc=org
member: cn=b81233ec879742dfb063be73e61a4058,ou=users,dc=openstack,dc=org
member: cn=da6a7e4ebd924d1b81e21b4513a30d56,ou=users,dc=openstack,dc=org
member: cn=21aabb86372942229b92eb41c27c911a,ou=users,dc=openstack,dc=org
cn: enabled_users

# 2ed4fc05512a46bfbba22d20a2928e29, users, openstack.org
dn: cn=2ed4fc05512a46bfbba22d20a2928e29,ou=users,dc=openstack,dc=org
objectClass: person
objectClass: inetOrgPerson
userPassword:: xxx
mail: neutron@localhost
sn: neutron
cn: 2ed4fc05512a46bfbba22d20a2928e29

# f2b302600645405bbeacbd563e39467f, users, openstack.org
dn: cn=f2b302600645405bbeacbd563e39467f,ou=users,dc=openstack,dc=org
objectClass: person
objectClass: inetOrgPerson
userPassword:: xxx
mail: nova@localhost
sn: nova
cn: f2b302600645405bbeacbd563e39467f

# b81233ec879742dfb063be73e61a4058, users, openstack.org
dn: cn=b81233ec879742dfb063be73e61a4058,ou=users,dc=openstack,dc=org
objectClass: person
objectClass: inetOrgPerson
userPassword:: xxx
mail: glance@localhost
sn: glance
cn: b81233ec879742dfb063be73e61a4058

# da6a7e4ebd924d1b81e21b4513a30d56, users, openstack.org
dn: cn=da6a7e4ebd924d1b81e21b4513a30d56,ou=users,dc=openstack,dc=org
objectClass: person
objectClass: inetOrgPerson
userPassword:: xxx
mail: cinder@localhost
sn: cinder
cn: da6a7e4ebd924d1b81e21b4513a30d56

# 21aabb86372942229b92eb41c27c911a, users, openstack.org
dn: cn=21aabb86372942229b92eb41c27c911a,ou=users,dc=openstack,dc=org
objectClass: person
objectClass: inetOrgPerson
userPassword:: xxx
cn: 21aabb86372942229b92eb41c27c911a
sn: demo

# search result
search: 2
result: 0 Success

# numResponses: 13
# numEntries: 12
```

