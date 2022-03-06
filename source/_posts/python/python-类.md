---
title: python-类
tags: python
abbrlink: 59b83510
date: 2019-08-04 18:12:22
---

# 类中默认函数

## ____new____和____init____区别

__new__:创建对象时调用，会返回当前对象的一个实例

__init__:创建完对象后调用，对当前对象的一些实例初始化，无返回值

1、在类中，如果__new__和__init__同时存在，会优先调用__new__


```python
class Data(object):
     def __new__(self):
             print "new"
     def __init__(self):
             print "init"
 
data = Data()
# new
```

2、__new__方法会返回所构造的对象，__init__则不会。__init__无返回值。

```python
class Data(object):
     def __init__(cls):
            cls.x = 2
             print "init"
            return cls

data = Data()
'''
init
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: __init__() should return None, not 'Data'
'''
```

```python
class Data(object):
    def __new__(cls):
        print("new")
        cls.x = 1
        return cls
def __init__(self):
    print("init")


data = Data()
print(data.x)
# new
# 1
data.x =2
print(data.x)
# 2
```

If __new__() returns an instance of cls, then the new instance’s __init__() method will be invoked like __init__(self[, ...]), where self is the new instance and the remaining arguments are the same as were passed to __new__().

如果__new__返回一个对象的实例，会隐式调用__init__

If __new__() does not return an instance of cls, then the new instance’s __init__() method will not be invoked.

如果__new__不返回一个对象的实例，__init__不会被调用

```python
class A(object):
     def __new__(Class):
             object = super(A,Class).__new__(Class)
             print "in New"
             return object
     def __init__(self):
             print "in init"
 
A()
# in New
# in init

class A(object):
     def __new__(cls):
             print "in New"
             return cls
     def __init__(self):
             print "in init"
 
a = A()      
# in New 
```

object.__init__(self[, ...])
Called when the instance is created. The arguments are those passed to the class constructor expression. If a base class has an __init__() method, the derived class’s __init__() method, if any, must explicitly call it to ensure proper initialization of the base class part of the instance; for example: BaseClass.__init__(self, [args...]). As a special constraint on constructors, no value may be returned; doing so will cause a TypeError to be raised at runtime.

在对象的实例创建完成后调用。参数被传给类的构造函数。如果基类有__init__方法，子类必须显示调用基类的__init__。

没有返回值，否则会再引发TypeError错误。

