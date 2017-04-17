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

常用的数学函数

|函数|返回值（描述）|
|--|--|
|abs(x)|返回数字的绝对值，如abs(-10) 返回 10|
|ceil(x)|返回数字的上入整数，如math.ceil(4.1) 返回 5|
|cmp(x, y)|如果 x < y 返回 -1, 如果 x == y 返回 0, 如果 x > y 返回 1|
|floor(x)|返回数字的下舍整数，如math.floor(4.9)返回 4|
|max(x1, x2,...)|返回给定参数的最大值，参数可以为序列。|
|min(x1, x2,...)|返回给定参数的最小值，参数可以为序列。|
|pow(x, y)|x**y 运算后的值。|
|round(x [,n])|返回浮点数x的四舍五入值，如给出n值，则代表舍入到小数点后的位数。|
|sqrt(x)|返回数字x的平方根，数字可以为负数，返回类型为实数，如math.sqrt(4)返回 2+0j|
 
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

```
print 1,
print 2
```

输出结果为：
1 2

加了逗号之后 换行就变成了空格  

**python三引号**


python三引号允许一个字符串跨多行，字符串中可以包含换行符、制表符以及其他特殊字符。  
三引号的语法是一对连续的单引号或者双引号（通常都是成对的用）

```
 >>> hi = '''hi 
there'''
>>> hi   # repr()
'hi\nthere'
>>> print hi  # str()
hi 
there  
```


常见的字符串运算符：
下表实例变量 a 值为字符串 "Hello"，b 变量值为 "Python"：

|操作符|描述|实例|
|--|--|--|
|+|字符串连接|>>>a + b 'HelloPython'|
|*|重复输出字符串|>>>a * 2 'HelloHello'|
|[]|通过索引获取字符串中字符|>>>a[1] 'e'|
|[ : ]|截取字符串中的一部分|>>>a[1:4] 'ell'|
|in|成员运算符 - 如果字符串中包含给定的字符返回 True|>>>"H" in a True|
|not in|成员运算符 - 如果字符串中不包含给定的字符返回 True|>>>"M" not in a True|


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


## 循环语句
Python提供了for循环和while循环（在Python中没有do..while循环）:

while语法跟java相似（{}改为：，再注意一下语句的缩进即可），所以不做过多的介绍。

### 循环使用 else 语句
在 python 中，while … else 在循环条件为 false 时执行 else 语句块：

```
#!/usr/bin/python

count = 0
while count < 5:
   print count, " is  less than 5"
   count = count + 1
else:
   print count, " is not less than 5"
```

### for循环语句
for循环的语法格式如下：

```
for iterating_var in sequence:
   statements(s)
```

通过序列索引迭代:

```
for num in range(10,20):  # 迭代 10 到 20 之间的数字
	print num
for num in range(10):	# 迭代 0 到 10 之间的数字
	print num
```

range()函数实例:

```
range(1,5)            # 代表从1到5(不包含5)
[1, 2, 3, 4]
>>> range(1,5,2)      # 代表从1到5，间隔2(不包含5)
[1, 3]
>>> range(5)          # 代表从0到5(不包含5)
[0, 1, 2, 3, 4]
```

### pass语句

Python pass是空语句，是为了保持程序结构的完整性。  
pass 不做任何事情，一般用做占位语句。  
当你在编写一个程序时，执行语句部分思路还没有完成，这时你可以用pass语句来占位，也可以当做是一个标记，是要过后来完成的代码

## 参考


