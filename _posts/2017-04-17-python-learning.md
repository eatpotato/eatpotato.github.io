---
layout:     post
title:      python数据类型
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
 
print list                           # 输出完整列表
print list[0]                        # 输出列表的第一个元素
print list[1:3]                      # 输出第二个至第三个的元素 
print list[2:]                       # 输出从第三个开始至列表末尾的所有元素
print tinylist * 2                   # 输出列表两次
print list + tinylist                # 打印组合的列表
list[-2] = 'john'                    # 读取列表中倒数第二个元素
list[1:] = [785, 2,23, 'john', 70.2] # 从第二个元素开始截取列表
```


Python列表脚本操作符

|pyhton表达式|结果|描述|
|--|--|--|
|len([1, 2, 3])|3|长度|
|[1, 2, 3] + [4, 5, 6]|[1, 2, 3, 4, 5, 6]|组合|
|['Hi!'] * 4|['Hi!', 'Hi!', 'Hi!', 'Hi!']|重复|
|3 in [1, 2, 3]|True|元素是否存在于列表中|
|for x in [1, 2, 3]: print x,|1 2 3|迭代|

python包含以下函数:

|函数|描述|
|--|--|
|cmp(list1, list2)|比较两个列表的元素|
|len(list)|列表元素个数|
|min(list)|返回列表元素最小值|
|max(list)|返回列表元素最大值|
|list(seq)|将元组转换为列表|

列表有以下方法：

|方法|描述|
|--|--|
|list.append(obj)|在列表末尾添加新的对象|
|list.count(obj)|统计某个元素在列表中出现的次数|
|list.extend(seq)|在列表末尾一次性追加另一个序列中的多个值（用新列表扩展原来的列表）|
|list.index(obj)|从列表中找出某个值第一个匹配项的索引位置|
|list.insert(index, obj)|将对象插入列表|
|list.pop(obj=list[-1])|移除列表中的一个元素（默认最后一个元素），并且返回该元素的值|
|list.remove(obj)|移除列表中某个值的第一个匹配项|
|list.reverse()|反向列表中元素|
|list.sort([func])|对原列表进行排序|

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

创建元组：

``` 
tup1 = ()          # 创建空元组
tup2 = (50,)       # 元组中只包含一个元素时，需要在元素后面添加逗号  
```

修改元组  
元组中的元素值是不允许修改的，但我们可以对元组进行连接组合，如下实例:

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-

tup1 = (12, 34.56);
tup2 = ('abc', 'xyz');

# 以下修改元组元素操作是非法的。
# tup1[0] = 100;

# 创建一个新的元组
tup3 = tup1 + tup2;
print tup3;
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

Python字典包含了以下内置方法：

|函数|描述|
|--|--|
|dict.clear()|删除字典内所有元素|
|dict.copy()|返回一个字典的浅复制|
|dict.fromkeys(seq[, val]))|创建一个新字典，以序列 seq 中元素做字典的键，val 为字典所有键对应的初始值|
|dict.get(key, default=None)|返回指定键的值，如果值不在字典中返回default|
|dict.has_key(key)|如果键在字典dict里返回true，否则返回false|
|dict.items()|以列表返回可遍历的(键, 值) 元组数组|
|dict.keys()|以列表返回一个字典所有的键|
|dict.setdefault(key, default=None)|和get()类似, 但如果键不存在于字典中，将会添加键并将值设为default|
|dict.update(dict2)|把字典dict2的键/值对更新到dict里|
|dict.values()|以列表返回字典中的所有值|


## 二、循环语句
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


## 三、函数
定义一个函数，需要满足以下规则：

* 函数代码块以 def 关键词开头，后接函数标识符名称和圆括号()。
* 任何传入参数和自变量必须放在圆括号中间。圆括号之间可以用于定义参数。
* 函数的第一行语句可以选择性地使用文档字符串—用于存放函数说明。
* 函数内容以冒号起始，并且缩进。
* return [表达式] 结束函数，选择性地返回一个值给调用方。不带表达式的return相当于返回 None。

实例：

```
def printme( str ):
   "打印传入的字符串到标准显示设备上"
   print str
   return
```

**可更改(mutable)与不可更改(immutable)对象**

在 python 中，strings, tuples, 和 numbers 是不可更改的对象，而 list,dict 等则是可以修改的对象。

python 函数的参数传递：

* 不可变类型：类似 c 的值传递，如 整数、字符串、元组。如fun（a），传递的只是a的值，没有影响a对象本身。比如在 fun（a）内部修改 a 的值，只是修改另一个复制的对象，不会影响 a 本身。

* 可变类型：类似 c 的引用传递，如 列表，字典。如 fun（la），则是将 la 真正的传过去，修改后fun外部的la也会受影响

### 匿名函数

python 使用 lambda 来创建匿名函数。

* lambda只是一个表达式，函数体比def简单很多。
* lambda的主体是一个表达式，而不是一个代码块。仅仅能在lambda表达式中封装有限的逻辑进去。
* lambda函数拥有自己的命名空间，且不能访问自有参数列表之外或全局命名空间里的参数。
* 虽然lambda函数看起来只能写一行，却不等同于C或C++的内联函数，后者的目的是调用小函数时不占用栈内存从而增加运行效率。

如下实例：

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
# 可写函数说明
sum = lambda arg1, arg2: arg1 + arg2;
 
# 调用sum函数
print "相加后的值为 : ", sum( 10, 20 )
print "相加后的值为 : ", sum( 20, 20 )
```

以上实例输出结果：

```
相加后的值为 :  30
相加后的值为 :  40
```
## 参考
[菜鸟教程](http://www.runoob.com/python/)

