---
title: python原类实践
date: 2019-11-03 10:24:27
tags:
- python
---

# 使用原来创建ORM的实例 

我们通过创建一个类似Django中的ORM来熟悉一下元类的使用，通常元类用来创建API是非常好的选择，使用元类的编写很复杂但使用者可以非常简洁的调用API。

```python
#我们想创建一个类似Django的ORM，只要定义字段就可以实现对数据库表和字段的操作。
class User(Model):
    # 定义类的属性到列的映射：
    id = IntegerField('id')
    name = StringField('username')
    email = StringField('email')
    password = StringField('password')
```

例如：

```python
# 创建一个实例：
u = User(id=12345, name='Michael', email='test@orm.org', password='my-pwd')
# 保存到数据库：
u.save()
```

接下来我么来实现这么个功能：

```python
#coding:utf-8
#一、首先来定义Field类，它负责保存数据库表的字段名和字段类型：
class Field(object):
    def __init__(self, name, column_type):
        self.name = name
        self.column_type = column_type
    def __str__(self):
        return '<%s:%s>' % (self.__class__.__name__, self.name)

class StringField(Field):
    def __init__(self, name):
        super(StringField, self).__init__(name, 'varchar(100)')

class IntegerField(Field):
    def __init__(self, name):
        super(IntegerField, self).__init__(name, 'bigint')

#二、定义元类，控制Model对象的创建
class ModelMetaclass(type):
    '''定义元类'''
    def __new__(cls, name, bases, attrs):
        if name=='Model':
            return super(ModelMetaclass,cls).__new__(cls, name, bases, attrs)
        mappings = dict()
        for k, v in attrs.iteritems():
            # 保存类属性和列的映射关系到mappings字典
            if isinstance(v, Field):
                print('Found mapping: %s==>%s' % (k, v))
                mappings[k] = v
        for k in mappings.iterkeys():
            #将类属性移除，使定义的类字段不污染User类属性，只在实例中可以访问这些key
            attrs.pop(k)
        attrs['__table__'] = name.lower() # 假设表名和为类名的小写,创建类时添加一个__table__类属性
        attrs['__mappings__'] = mappings # 保存属性和列的映射关系，创建类时添加一个__mappings__类属性
        return super(ModelMetaclass,cls).__new__(cls, name, bases, attrs)

#三、编写Model基类
class Model(dict):
    __metaclass__ = ModelMetaclass

    def __init__(self, **kw):
        super(Model, self).__init__(**kw)

    def __getattr__(self, key):
        try:
            return self[key]
        except KeyError:
            raise AttributeError(r"'Model' object has no attribute '%s'" % key)

    def __setattr__(self, key, value):
        self[key] = value

    def save(self):
        fields = []
        params = []
        args = []
        for k, v in self.__mappings__.iteritems():
            fields.append(v.name)
            params.append('?')
            args.append(getattr(self, k, None))
        sql = 'insert into %s (%s) values (%s)' % (self.__table__, ','.join(fields), ','.join(params))
        print('SQL: %s' % sql)
        print('ARGS: %s' % str(args))

#最后，我们使用定义好的ORM接口，使用起来非常的简单。
class User(Model):
    # 定义类的属性到列的映射：
    id = IntegerField('id')
    name = StringField('username')
    email = StringField('email')
    password = StringField('password')

# 创建一个实例：
u = User(id=12345, name='Michael', email='test@orm.org', password='my-pwd')
# 保存到数据库：
u.save()

#输出
# Found mapping: email==><StringField:email>
# Found mapping: password==><StringField:password>
# Found mapping: id==><IntegerField:id>
# Found mapping: name==><StringField:username>
# SQL: insert into User (password,email,username,id) values (?,?,?,?)
# ARGS: ['my-pwd', 'test@orm.org', 'Michael', 12345]
```

# 使用__new__方法和元类方式分别实现单例模式

## 1、__new__、__init__、__call__的介绍

在讲到使用元类创建单例模式之前，比需了解__new__这个内置方法的作用，在上面讲元类的时候我们用到了__new__方法来实现类的创建。然而我在那之前还是对__new__这个方法和__init__方法有一定的疑惑。因此这里花点时间对其概念做一次了解和区分。

__new__方法负责创建一个实例对象，在对象被创建的时候调用该方法它是一个类方法。__new__方法在返回一个实例之后，会自动的调用__init__方法，对实例进行初始化。如果__new__方法不返回值，或者返回的不是实例，那么它就不会自动的去调用__init__方法。

__init__ 方法负责将该实例对象进行初始化，在对象被创建之后调用该方法，在__new__方法创建出一个实例后对实例属性进行初始化。__init__方法可以没有返回值。

__call__方法其实和类的创建过程和实例化没有多大关系了，定义了__call__方法才能被使用函数的方式执行。

```python
例如：
class A(object):
    def __call__(self):
        print "__call__ be called"

a = A()
a()
#输出
#__call__ be called 
```

打个比方帮助理解：如果将创建实例的过程比作建一个房子。

- 那么class就是一个房屋的设计图，他规定了这个房子有几个房间，每个人房间的大小朝向等。这个设计图就是累的结构
- __new__就是一个房屋的框架，每个具体的房屋都需要先搭好框架后才能进行专修，当然现有了房屋设计才能有具体的房屋框架出来。这个就是从类到类实例的创建。
- __init__就是装修房子的过程，对房屋的墙面和地板等颜色材质的丰富就是它该做的事情，当然先有具体的房子框架出来才能进行装饰了。这个就是实例属性的初始化，它是在__new__出一个实例后才能初始化。
- __call__就是房子的电话，有了固定电话，才能被打电话嘛（就是通过括号的方式像函数一样执行）。

```python
#coding:utf-8
class Foo(object):
    def __new__(cls, *args, **kwargs):
        #__new__是一个类方法，在对象创建的时候调用
        print "excute __new__"
        return super(Foo,cls).__new__(cls,*args,**kwargs)


    def __init__(self,value):
        #__init__是一个实例方法，在对象创建后调用，对实例属性做初始化
        print "excute __init"
        self.value = value


f1 = Foo(1)
print f1.value
f2 = Foo(2)
print f2.value

#输出===：
excute __new__
excute __init
1
excute __new__
excute __init
2
#====可以看出new方法在init方法之前执行
```

 子类如果重写__new__方法，一般依然要调用父类的__new__方法。

```python
class Child(Foo):
    def __new__(cls, *args, **kwargs):        
        return suyper(Child, cls).__new__(cls, *args, **kwargs)
```

 必须注意的是，类的__new__方法之后，必须生成本类的实例才能自动调用本类的__init__方法进行初始化，否则不会自动调用__init__.

```python
class Foo(object):
    def __init__(self, *args, **kwargs):
        print "Foo __init__"
    def __new__(cls, *args, **kwargs):
        return object.__new__(Stranger, *args, **kwargs)

class Stranger(object):
    def __init__(self,name):
        print "class Stranger's __init__ be called"
        self.name = name

foo = Foo("test")
print type(foo) #<class '__main__.Stranger'>
print foo.name #AttributeError: 'Stranger' object has no attribute 'name'

#说明：如果new方法返回的不是本类的实例，那么本类（Foo）的init和生成的类(Stranger)的init都不会被调用
```

## 2.实现单例模式

依照Python官方文档的说法，__new__方法主要是当你继承一些不可变的class时(比如int, str, tuple)， 提供给你一个自定义这些类的实例化过程的途径。还有就是实现自定义的metaclass。接下来我们分别通过这两种方式来实现单例模式。当初在看到cookbook中的元类来实现单例模式的时候对其相当疑惑，因此才有了上面这些对元类的总结。

简单来说，单例模式的原理就是通过在类属性中添加一个单例判定位ins_flag，通过这个flag判断是否已经被实例化过了,如果被实例化过了就返回该实例。

### __new__方法实现单例：

```python
class Singleton(object):
    def __new__(cls, *args, **kwargs):
        if not hasattr(cls,"_instance"):
            cls._instance = super(Singleton, cls).__new__(cls, *args, **kwargs)
        return cls._instance


s1 = Singleton()
s2 = Singleton()

print s1 is s2
```



因为重写__new__方法，所以继承至Singleton的类，在不重写__new__的情况下都将是单例模式。

### 元类实现单例

当初我也很疑惑为什么我们是从写使用元类的__init__方法，而不是使用__new__方法来初为元类增加一个属性。其实我只是上面那一段关于元类中__new__方法迷惑了，它主要用于我们需要对类的结构进行改变的时候我们才要重写这个方法。

```python
class Singleton(type):
    def __init__(self, *args, **kwargs):
        print "__init__"
        self.__instance = None
        super(Singleton,self).__init__(*args, **kwargs)

    def __call__(self, *args, **kwargs):
        print "__call__"
        if self.__instance is None:
            self.__instance = super(Singleton,self).__call__(*args, **kwargs)
        return self.__instance


class Foo(object):
    __metaclass__ = Singleton #在代码执行到这里的时候，元类中的__new__方法和__init__方法其实已经被执行了，而不是在Foo实例化的时候执行。且仅会执行一次。


foo1 = Foo()
foo2 = Foo()
print Foo.__dict__  #_Singleton__instance': <__main__.Foo object at 0x100c52f10> 存在一个私有属性来保存属性，而不会污染Foo类（其实还是会污染，只是无法直接通过__instance属性访问）

print foo1 is foo2  # True

# 输出
# __init__
# __call__
# __call__
# {'__module__': '__main__', '__metaclass__': <class '__main__.Singleton'>, '_Singleton__instance': <__main__.Foo object at 0x100c52f10>, '__dict__': <attribute '__dict__' of 'Foo' objects>, '__weakref__': <attribute '__weakref__' of 'Foo' objects>, '__doc__': None}
# True 
```

基于这个例子：

- 我们知道元类(Singleton)生成的实例是一个类(Foo),而这里我们仅仅需要对这个实例(Foo)增加一个属性(__instance)来判断和保存生成的单例。想想也知道为一个类添加一个属性当然是在__init__中实现了。
- 关于__call__方法的调用，因为Foo是Singleton的一个实例。所以Foo()这样的方式就调用了Singleton的__call__方法。不明白就回头看看上一节中的__call__方法介绍。

假如我们通过元类的__new__方法来也可以实现，但显然没有通过__init__来实现优雅，因为我们不会为了为实例增加一个属性而重写__new__方法。所以这个形式不推荐。

```python
class Singleton(type):
    def __new__(cls, name,bases,attrs):
        print "__new__"
        attrs["_instance"] = None
        return  super(Singleton,cls).__new__(cls,name,bases,attrs)

    def __call__(self, *args, **kwargs):
        print "__call__"
        if self._instance is None:
            self._instance = super(Singleton,self).__call__(*args, **kwargs)
        return self._instance

class Foo(object):
    __metaclass__ = Singleton

foo1 = Foo()
foo2 = Foo()
print Foo.__dict__ 
print foo1 is foo2  # True

# 输出
# __new__
# __call__
# __call__
# {'__module__': '__main__', '__metaclass__': <class '__main__.Singleton'>, '_instance': <__main__.Foo object at 0x103e07ed0>, '__dict__': <attribute '__dict__' of 'Foo' objects>, '__weakref__': <attribute '__weakref__' of 'Foo' objects>, '__doc__': None}
# True
```