---
layout:     post
title:      nova自定义API
date:       2017-06-16
author:     xue
catalog:    true
tags:
    - openstack
    - nova
---

以kilo版本nova为例

创建自定义的API有三种方式：

* 在原有的资源上增加函数，例如在servers上增加一个接口，查看虚拟机的资源利用情况
* 添加扩展资源，定义新的扩展资源
* 添加核心资源,定义新的核心资源


## method 1

对于第一种情况,具体可以参照

```
    @wsgi.response(202)
    @wsgi.action('revertResize')
    def _action_revert_resize(self, req, id, body):
        context = req.environ['nova.context']
        instance = self._get_server(context, req, id)
        try:
            self.compute_api.revert_resize(context, instance)
        except exception.MigrationNotFound:
            msg = _("Instance has not been resized.")
            raise exc.HTTPBadRequest(explanation=msg)
        except exception.FlavorNotFound:
            msg = _("Flavor used by the instance could not be found.")
            raise exc.HTTPBadRequest(explanation=msg)
        except exception.InstanceIsLocked as e:
            raise exc.HTTPConflict(explanation=e.format_message())
        except exception.InstanceInvalidState as state_error:
            common.raise_http_conflict_for_instance_invalid_state(state_error,
                    'revertResize', id)
        return webob.Response(status_int=202)
```

访问它的curl为： 

```
curl -XPOST http://192.168.138.122:8774/v2/321c7446162e431f91c69b60e64d605f/servers/97f532c4-9335-47e9-8d1c-9ba1da3666bf/action    -H "Content-type: application/json" -H "X-Auth-Token: 4bd43818ff7547d0ad92bae071bd4973" -d '{"revertResize": null}'
```

@wsgi.action装饰器里的名字与http请求body中的数据key对应

## mothod 2


添加新的扩展资源，我们需要写一个py文件，定义一个class，将其放在nova.api.openstack.compute.contrib目录下面，文件名小写，然后再文件中定义一个class，类名和文件一样，只是首字母大写，该class要继承于ExtensionDescriptor,并实现get_resources 或者get_controller_extensions


### 2.1 创建对应的objects

```
# nova/objects/documents.py
from nova import db
from nova.objects import base
from nova.objects import fields


class Documents(base.NovaPersistentObject, base.NovaObject,
             base.NovaObjectDictCompat):
    # Version 1.0: Initial version
    VERSION = '1.0'
    
    fields = {
        'id': fields.IntegerField(),
        'name': fields.StringField(nullable=False),
        }
        
    def __init__(self, *args, **kwargs):
        super(Documents, self).__init__(*args, **kwargs)
        self.obj_reset_changes()
        
    @base.remotable_classmethod
    def get_by_id(cls, context, id):
        return db.document_get_by_id(
            context, id)
        
    @base.remotable_classmethod
    def get_all(cls, context):
        return db.document_get_all(context)
               
    @base.remotable_classmethod
    def delete_by_id(cls, context, id):
        db.document_delete_by_id(context, id)
         
    @base.remotable_classmethod
    def update(cls, context, id, names):
        db.document_update(context, id, names)

    @base.remotable_classmethod
    def create(self, context, values):
        db_document = db.document_create(context, values)


```

### 2.2 创建Models

```
# nova/db/sqlalchemy/models.py
class Documents(BASE, NovaBase):
    __tablename__ = 'documents'
    id = Column(Integer, primary_key=True)
    name = Column(String(45), nullable=False)
```

### 2.3 创建对数据库的实现

```
# nova/db/api.py

def document_get_by_id(context, id):
    """Get document by id."""
    return IMPL.document_get_by_id(context, id)
    
def document_get_all(context):
    """Get all documents."""
    return IMPL.document_get_all(context)
    
def document_delete_by_id(context, id):
    """Delete a document record."""
    return IMPL.document_delete_by_id(context, id)
    
def document_update(context, id, name):
    """Update a document record."""
    return IMPL.document_update(context, id, name)
    
def document_create(context, values):
    """Create a document record."""
    return IMPL.document_create(context, values)
```

```
# nova/db/sqlalchemy/api.py
I   
    
```



### 2.4 数据库升级

```
# nova/db/sqlalchemy/migrate_repo/versions/306_add_documents.py
from migrate.changeset import UniqueConstraint

from sqlalchemy import Column
from sqlalchemy import DateTime
from sqlalchemy import Integer
from sqlalchemy import MetaData
from sqlalchemy import String
from sqlalchemy import Table


def upgrade(migrate_engine):
    meta = MetaData()
    meta.bind = migrate_engine

    columns = [
    (('created_at', DateTime), {}),
    (('updated_at', DateTime), {}),
    (('deleted_at', DateTime), {}),
    (('deleted', Integer), dict(default=0)),

    (('id', Integer), dict(primary_key=True, nullable=False)),
    (('name', String(length=45)), {})
    ]
    
    basename = 'documents'
    _columns = [Column(*args, **kwargs) for args, kwargs in columns]
    table = Table(basename, meta, *_columns,
                          mysql_engine='InnoDB',
                          mysql_charset='utf8')
    table.create()
    



def downgrade(migrate_engine):
    meta = MetaData()
    meta.bind = migrate_engine
    
    table_name = 'documents'
    if migrate_engine.has_table(table_name):
    	instance_extra = Table(table_name, meta, autoload=True)
    	instance_extra.drop()          
```

执行命令：  su -s /bin/sh -c "nova-manage db sync" nova

验证：

```

root@localhost:(none) 11:20:57>use nova;
Database changed
root@localhost:nova 11:21:09>desc documents;
+------------+-------------+------+-----+---------+----------------+
| Field      | Type        | Null | Key | Default | Extra          |
+------------+-------------+------+-----+---------+----------------+
| created_at | datetime    | YES  |     | NULL    |                |
| updated_at | datetime    | YES  |     | NULL    |                |
| deleted_at | datetime    | YES  |     | NULL    |                |
| deleted    | int(11)     | YES  |     | NULL    |                |
| id         | int(11)     | NO   | PRI | NULL    | auto_increment |
| name       | varchar(45) | YES  |     | NULL    |                |
+------------+-------------+------+-----+---------+----------------+
6 rows in set (0.01 sec)
```


### 2.5 添加异常

```
# nova/exception.py
class DocumentsNotFoundByName(NotFound):
    msg_fmt = _("Documents %(name)s not found.")
    
class DocumentsNotFoundById(NotFound):
    msg_fmt = _("Documents %(id)s not found.")
```

### 2.6 添加documents.py

```
import webob

from nova import db
from nova import exception
from nova.objects import documents
from nova.api.openstack import extensions
from nova.api.openstack import wsgi

authorize = extensions.extension_authorizer('compute', 'documents')

class DocumentsController(wsgi.Controller):
        """the Documents API Controller declearation"""
         
        
        def index(self, req):
            documents_dict = {}
            documents_list = []

            context = req.environ['nova.context']
            authorize(context)

            document = documents.Documents.get_all(context)
            if document:
                for single_document in document:
                    id = single_document['id']
                    name = single_document['name']
                    document_dict = dict()
                    document_dict['id'] = id
                    document_dict['name'] = name
                    documents_list.append(document_dict)

            documents_dict['document'] = documents_list
            return documents_dict

        def create(self, req, body):
            values = {}
            context = req.environ['nova.context']
            authorize(context)

            id = body['id']
            name = body['name']
            values['id'] = id
            values['name'] = name

            try:
                documents.Documents.create(context, values)
            except :
                raise webob.exc.HTTPNotFound(explanation="Document not found")

            return webob.Response(status_int=202)


        def show(self, req, id):
            documents_dict = {}
            context = req.environ['nova.context']
            authorize(context)

            try:
                document = documents.Documents.get_by_id(context, id)
            except :
                raise webob.exc.HTTPNotFound(explanation="Document not found")

            documents_dict['document'] = document
            return documents_dict

        def update(self, req, id, body):
            context = req.environ['nova.context']
            authorize(context)
            name = body['name']

            try:
                documents.Documents.update(context, id, name)
            except :
                raise webob.exc.HTTPNotFound(explanation="Document not found")

            return webob.Response(status_int=202)

        def delete(slef, req, id):
            context = req.environ['nova.context']
            authorize(context)

            try:
                document = documents.Documents.delete_by_id(context, id)
            except :
                raise webob.exc.HTTPNotFound(explanation="Document not found")

            return webob.Response(status_int=202)
            

class Documents(extensions.ExtensionDescriptor):
        """Documents ExtensionDescriptor implementation"""

        name = "documents"
        alias = "os-documents"
        namespace = "www.www.com"
        updated = "2017-06-14T00:00:00+00:00"

        def get_resources(self):
            """register the new Documents Restful resource"""

            resources = [extensions.ResourceExtension('os-documents',
                DocumentsController())
                ]

            return resources

 ```



验证：



创建documents记录

```
curl -XPOST http://10.160.57.85:8774/v2/4e4bed4036b446e996ad99fc5522f086/os-documents  -H "Content-Type: application/json" -H "X-Auth-Token: 6e336e1b3121440891c06150fb3bd757" -d '{"id": 1, "name": "test1"}'

curl -XPOST http://10.160.57.85:8774/v2/4e4bed4036b446e996ad99fc5522f086/os-documents  -H "Content-Type: application/json" -H "X-Auth-Token: 6e336e1b3121440891c06150fb3bd757" -d '{"id": 2, "name": "test2"}'

```

查看数据库

```
root@localhost:nova 05:33:55>select * from documents;
+---------------------+------------+------------+---------+----+-------+
| created_at          | updated_at | deleted_at | deleted | id | name  |
+---------------------+------------+------------+---------+----+-------+
| 2017-06-14 09:33:36 | NULL       | NULL       |       0 |  1 | test1 |
| 2017-06-14 09:34:20 | NULL       | NULL       |       0 |  2 | test2 |
+---------------------+------------+------------+---------+----+-------+
```

查看所有documents记录


```
[root@controller1 contrib]# curl -XGET http://10.160.57.85:8774/v2/4e4bed4036b446e996ad99fc5522f086/os-documents  -H "Content-Type: application/json" -H "X-Auth-Token: 6e336e1b3121440891c06150fb3bd757"
{"document": [{"id": 1, "name": "test1"}, {"id": 2, "name": "test2"}]
```

修改一条documents记录

```
[root@controller1 contrib]# curl -XPUT http://10.160.57.85:8774/v2/4e4bed4036b446e996ad99fc5522f086/os-documents/1  -H "Content-Type: application/json" -H "X-Auth-Token: 6e336e1b3121440891c06150fb3bd757" -d '{"name": "new-test1"}'

# 确认是否修改
[root@controller1 contrib]# curl -XGET http://10.160.57.85:8774/v2/4e4bed4036b446e996ad99fc5522f086/os-documents  -H "Content-Type: application/json" -H "X-Auth-Token: 6e336e1b3121440891c06150fb3bd757"
{"document": [{"id": 1, "name": "new-test1"}, {"id": 2, "name": "test2"}]}
```

删除一条documents记录

```
[root@controller1 contrib]# curl -XDELETE http://10.160.57.85:8774/v2/4e4bed4036b446e996ad99fc5522f086/os-documents/1  -H "Content-Type: application/json" -H "X-Auth-Token: 6e336e1b3121440891c06150fb3bd757"

# 确认是否删除
[root@controller1 contrib]# curl -XGET http://10.160.57.85:8774/v2/4e4bed4036b446e996ad99fc5522f086/os-documents  -H "Content-Type: application/json" -H "X-Auth-Token: 6e336e1b3121440891c06150fb3bd757"
{"document": [{"id": 2, "name": "test2"}]}[root@controller1 contrib]#
```

