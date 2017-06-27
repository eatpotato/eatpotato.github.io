---
layout:     post
title:      python迭代器、生成器与装饰器
date:       2017-06-24
author:     xue
catalog:    true
tags:
    - python
---



# 可迭代对象与迭代器

#### 可迭代对象

可以直接作用于for循环的对象统称为可迭代对象，代码中可以通过isinstance()判断一个对象是否是Iterable对象

Example：


```
>>> from collections import Iterable
>>> isinstance([],Iterable)
True
>>> isinstance({}, Iterable)
True
>>> isinstance('test', Iterable)
True
>>> isinstance((x for x in range(10)), Iterable)
True
>>> isinstance(1, Iterable)
False

```
 
#### 迭代器

可以被next()函数调用并不断返回下一个值的对象称为迭代器，代码中可以使用isinstance()判断一个对象是否是Iterator对象

Example:

```
>>> from collections import Iterator
>>> isinstance((x for x in range(10)), Iterator)
True
>>> isinstance([], Iterator)
False
>>> isinstance({}, Iterator)
False
>>> isinstance('abc', Iterator)
False
```

可以看到，生成器都是Iterator对象，但list、dict、str虽然是Iterable，却不是Iterator。

#### 迭代器与列表

```
aList = ['a', 'b']
aIter = iter(aList)

while True:
    try:
        print aIter.next()
    except StopIteration:
        print 'Done'
        break
```

#### 迭代器与字典

```
aDict = {'a': 1, 'b': 2}

for key, value in aDict.iteritems():
    print 'key: %s, value: %s' % (key, value)
```

#### for循环原理

```
for x in [1, 2, 3, 4, 5]:
    pass
```

实际上完全等价于：

```
# 首先获得Iterator对象:
it = iter([1, 2, 3, 4, 5])
# 循环:
while True:
    try:
        # 获得下一个值:
        x = next(it)
    except StopIteration:
        # 遇到StopIteration就退出循环
        break
```

#### 自定义迭代器

根据上面的for循环原理，我们可以自定义迭代器,只需要实现iter,next方法

```
class MyRange(object):
    def __init__(self, end):
        self.curr = 0
        self.end = end

    def __iter__(self):
        return self

    def next(self):
        if self.curr < self.end:
            val = self.curr
            self.curr += 1
            return val
        else:
            raise StopIteration()

```

## 生成器

带有 yield 关键字的的函数在 Python 中被称之为 generator(生成器)

Example:

```
def test():
    for i in range(5):
        yield i


a = test()
print a.next()
print a.next()
print a.next()
print a.next()
print a.next()
print a.next()
```

输出结果：

```
0
1
2
3
4
Traceback (most recent call last):
  File "test8.py", line 12, in <module>
    print a.next()
StopIteration
```

代码中a = test()时，会返回一个generator对象，它保存的是算法，在我们调用a.next(),会执行至yield语句并返回，再次执行时从上次返回的yield语句处继续执行.

在nova加载extentions时的代码如下：

```
    def sorted_extensions(self):
        if self.sorted_ext_list is None:
            self.sorted_ext_list = sorted(self.extensions.iteritems())

        for _alias, ext in self.sorted_ext_list:
            yield ext
           
```

get_resource方法中调用了此函数

```
    for ext in self.sorted_extensions():
        resources.extend(ext.get_resources())
```

这样做的就是好处就是extensions 对象不会占用大量的内存，它们只会要调用的时候在内存里生成


#### 斐波拉契数列的实现

```
def fab(max):
    n, a, b = 0, 0, 1
    while n < max:
    	yield b
    	a, b = b, a+b
    	n += 1

```

## 装饰器

装饰器实际上就是一个函数, 而且是一个接收函数对象的函数. 有了装饰器我们可以在执行被装饰函数之前做一个预处理, 也可以在处理完函数之后做清除工作. 

一个直观的例子:

```
def hello(fn):
    def wrapper():
        print "hello, %s" % fn.__name__
        fn()
        print "goodby, %s" % fn.__name__
    return wrapper
    
@hello
def foo():
    print "i am foo"
 
foo()
```

输出如下：

```
➜  python python hello.py                                                                                                                      
hello, foo
i am foo
goodby, foo
```

你可以看到如下的东西：

1）函数foo前面有个@hello的“注解”，hello就是我们前面定义的函数hello

2）在hello函数中，其需要一个fn的参数（这就用来做回调的函数）

3）hello函数中返回了一个inner函数wrapper，这个wrapper函数回调了传进来的fn，并在回调前后加了两条语句。

对于Python的这个@注解语法糖- Syntactic Sugar 来说，这个例子实际上被解释成了:

```
foo = hello(foo)
```

它的执行过程相当于

```
# 理解函数其实是一个对象的概念
fn = foo
print "hello, %s" % fn.__name__
fn()
print "goodby, %s" % fn.__name__

```

当有多个decorator或是带参数的decorator时，如：

```
@decorator_one
@decorator_two
def func():
    pass
```

相当于

```
func = decorator_one(decorator_two(func))
```

文本转换的例子：
 
```
def makeHtmlTag(tag, *args, **kwds):
    def real_decorator(fn):
        css_class = " class='{0}'".format(kwds["css_class"]) \
                                     if "css_class" in kwds else ""
        def wrapped(*args, **kwds):
            return "<"+tag+css_class+">" + fn(*args, **kwds) + "</"+tag+">"
        return wrapped
    return real_decorator
 
@makeHtmlTag(tag="b", css_class="bold_css")
@makeHtmlTag(tag="i", css_class="italic_css")
def hello():
    return "hello world"
 
print hello()
 
# 输出：
# <b class='bold_css'><i class='italic_css'>hello world</i></b>
```


下面，我们来看看用类的方式来重写上面的html.py的代码：


```
class makeHtmlTagClass(object):
 
    def __init__(self, tag, css_class=""):
        self._tag = tag
        self._css_class = " class='{0}'".format(css_class) \
                                       if css_class !="" else ""
 
    def __call__(self, fn):
        def wrapped(*args, **kwargs):
            return "<" + self._tag + self._css_class+">"  \
                       + fn(*args, **kwargs) + "</" + self._tag + ">"
        return wrapped
 
@makeHtmlTagClass(tag="b", css_class="bold_css")
@makeHtmlTagClass(tag="i", css_class="italic_css")
def hello(name):
    return "Hello, {}".format(name)
 
print hello("Hao Chen")
```

上面这段代码中，我们需要注意这几点：  
1）如果decorator有参数的话，__init__() 成员就不能传入fn了，而fn是在__call__的时候传入的。  
2）这段代码还展示了 wrapped(*args, **kwargs) 这种方式来传递被decorator函数的参数。


  
#### functools的wraps

```
def hello(fn):
    def wrapper():
        print "hello, %s" % fn.__name__
        fn()
        print "goodby, %s" % fn.__name__
    return wrapper

@hello
def foo():
    print "i am foo"

foo()
print foo.__name__
```

来看看这段代码的输出：

```
➜  python python hello.py                                                                                                                     
hello, foo
i am foo
goodby, foo
wrapper
```

会发现其输出的是“wrapper”，而不是我们期望的“foo”,因为我们前面提到过,这样的装饰器相当于foo = hello(foo), 而hello函数中返回值是wrapper, 所以foo.__name__ 自然是wrapper。所以，Python的functool包中提供了一个叫wrap的decorator来消除这样的副作用。下面是我们新版本的hello.py。

```
from functools import wraps
def hello(fn):
    @wraps(fn)
    def wrapper():
        print "hello, %s" % fn.__name__
        fn()
        print "goodby, %s" % fn.__name__
    return wrapper
 
@hello
def foo():
    print "i am foo"
 
foo()
print foo.__name__
```

## 参考
[Jmilk's blog](http://blog.csdn.net/jmilk/article/details/52560837)  
[廖雪峰的官方网站](http://www.liaoxuefeng.com/)  
[酷 壳 – COOLSHELL](http://coolshell.cn/articles/11265.html)
