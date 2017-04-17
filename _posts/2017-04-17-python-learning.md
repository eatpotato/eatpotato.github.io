---
layout:     post
title:      python学习
date:       2017-04-17
author:     xue
catalog:    true
tags:
    - python
---

## 一、标准数据类型

* Numbers（数字）
* String（字符串）
* List（列表）
* Tuple（元组）
* Dictionary（字典）


### Python数字

数字数据类型用于存储数值。  
他们是不可改变的数据类型，这意味着改变数字数据类型会分配一个新的对象。  
当你指定一个值时，Number对象就会被创建：

```
var1 = 1
```

可以通过使用del语句删除单个或多个对象的引用。例如：

```
del var
del var1, var2
```

Python支持四种不同的数字类型：

* int（有符号整型）
* long（长整型[也可以代表八进制和十六进制]）
* float（浮点型）
* complex（复数）

一些数值类型的实例：

|int|long|float|complex|
|--|--|--|--|
|10|51924361L|0.0|3.14j|
|-786|0122L|-21.9|9.322e-36j|

Python还支持复数，复数由实数部分和虚数部分构成，可以用a + bj,或者complex(a,b)表示， 复数的实部a和虚部b都是浮点型
 
### Python字符串

字符串或串(String)是由数字、字母、下划线组成的一串字符。

python的字串列表有2种取值顺序:

* 从左到右索引默认0开始的，最大范围是字符串长度少1
* 从右到左索引默认-1开始的，最大范围是字符串开头

实例：

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
str = 'Hello World!'
 
print str           # 输出完整字符串
print str[0]        # 输出字符串中的第一个字符
print str[2:5]      # 输出字符串中第三个至第五个之间的字符串
print str[2:]       # 输出从第三个字符开始的字符串
print str * 2       # 输出字符串两次
print str + "TEST"  # 输出连接的字符串
```

注： 加号（+）是字符串连接运算符，星号（*）是重复操作。

### Python列表

List（列表） 是 Python 中使用最频繁的数据类型。  
列表可以完成大多数集合类的数据结构实现。它支持字符，数字，字符串甚至可以包含列表（所谓嵌套）。  
列表用[ ]标识。是python最通用的复合数据类型。

实例：

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
list = [ 'runoob', 786 , 2.23, 'john', 70.2 ]
tinylist = [123, 'john']
 
print list               # 输出完整列表
print list[0]            # 输出列表的第一个元素
print list[1:3]          # 输出第二个至第三个的元素 
print list[2:]           # 输出从第三个开始至列表末尾的所有元素
print tinylist * 2       # 输出列表两次
print list + tinylist    # 打印组合的列表
```

### python元组

元组是另一个数据类型，类似于List（列表）。
元组用"()"标识。内部元素用逗号隔开。但是元组不能二次赋值，相当于只读列表。

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
tuple = ( 'runoob', 786 , 2.23, 'john', 70.2 )
tinytuple = (123, 'john')
 
print tuple               # 输出完整元组
print tuple[0]            # 输出元组的第一个元素
print tuple[1:3]          # 输出第二个至第三个的元素 
print tuple[2:]           # 输出从第三个开始至列表末尾的所有元素
print tinytuple * 2       # 输出元组两次
print tuple + tinytuple   # 打印组合的元组
```

以下是元组**无效**的，因为元组是不允许更新的。而列表是允许更新的：

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
tuple = ( 'runoob', 786 , 2.23, 'john', 70.2 )
list = [ 'runoob', 786 , 2.23, 'john', 70.2 ]
tuple[2] = 1000    # 元组中是非法应用
list[2] = 1000     # 列表中是合法应用
```

### python字典

字典(dictionary)是除列表以外python之中最灵活的内置数据结构类型。列表是有序的对象结合，字典是无序的对象集合。  
两者之间的区别在于：字典当中的元素是通过键来存取的，而不是通过偏移存取。  
字典用"{ }"标识。字典由索引(key)和它对应的值value组成。

实例：

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
dict = {}
dict['one'] = "This is one"
dict[2] = "This is two"
 
tinydict = {'name': 'john','code':6734, 'dept': 'sales'}
 
 
print dict['one']          # 输出键为'one' 的值
print dict[2]              # 输出键为 2 的值
print tinydict             # 输出完整的字典
print tinydict.keys()      # 输出所有键
print tinydict.values()    # 输出所有值
```



## 参考


