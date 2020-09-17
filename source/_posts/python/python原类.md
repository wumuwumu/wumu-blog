---
title: python原类
date: 2019-11-03 10:05:32
tags:
- python
---

# Python中一切皆对象，类也是对象 
​    之前我们说Python中一切都是对象。对象从哪里来，对象是类的实例。如下，使用type()函数查看对象所属的类型。我们可以看到Python中所以实例都是类的对象。那么类呢，既然一切都是对象，那么类也应该是对象。如下代码中发现我们创建的Person类原来也是对象，是type的对象。

```python
a =10; b = 12.12; c="hello" ;d =[1,2,3,"rr"];e = {"aa":1,"bb":"cc"}
type(a);type(b);type(c);type(d);type(e)
<class 'int'>   #a = 10;a也是对象，即10是对象，是int类型的对象
<class 'float'> #float也是类，注意python很多类的写法是小写，有的则是大写
<class 'str'>
<class 'list'>
<class 'dict'>class Person(object):
    print("不调用类，也会执行我")
    def __init__(self,name):
        self.name = name
    def p(self):
        print("this is a  methond")
        
print(Person)  
tom = Person("tom")
print("tom实例的类型是：%s"%type(tom))  # 实例tom是Person类的对象。
print("Peron类的类型：%s"%type(Person))  #结果看出我们创建的类属于type类,也就是说Person是type类的对象
print("type的类型是：%s"%type(type))  #type是type自己的对象

不调用类，也会执行我
<class '__main__.Person'>
tom实例的类型是：<class '__main__.Person'>
Peron类的类型：<class 'type'>
type的类型是：<class 'type'>
```
# 动态创建类
## **通过class动态的构建需要的类**

因为类也是对象，你可以在运行时动态的创建它们，就像其他任何对象一样。首先，你可以在函数中创建类，使用class关键字即可。 

```python
def choose_class(name):
    if name == 'foo':
        class Foo(object):
            pass
        return Foo     # 返回的是类，不是类的实例
    else:
        class Bar(object):
            pass
        return Bar
MyClass = choose_class('foo')

print MyClass              # 函数返回的是类，不是类的实例
#输出：<class '__main__.Foo'>

print MyClass()            # 你可以通过这个类创建类实例，也就是对象
#输出：<__main__.Foo object at 0x1085ed950
```

## **通过type函数构造类**

但这还不够动态，因为你仍然需要自己编写整个类的代码。由于类也是对象，所以它们必须是通过什么东西来生成的才对。当你使用class关键字时，Python解释器自动创建这个对象。但就和Python中的大多数事情一样，Python仍然提供给你手动处理的方法。还记得内建函数type吗？这个古老但强大的函数能够让你知道一个对象的类型是什么，就像这样：

```python
print type(1)
#输出：<type 'int'>
print type("1")
#输出：<type 'str'>
print type(ObjectCreator)
#输出：<type 'type'>
print type(ObjectCreator())
#输出：<class '__main__.ObjectCreator'>
```

这里，type有一种完全不同的能力，它也能动态的创建类。type可以接受一个类的描述作为参数，然后返回一个类。（我知道，根据传入参数的不同，同一个函数拥有两种完全不同的用法是一件很傻的事情，但这在Python中是为了保持向后兼容性）

**type的语法：**

```
type(类名, 父类的元组（针对继承的情况，可以为空），包含属性的字典（名称和值）)
```

比如下面的代码：

```
class MyShinyClass(object):
    pass
```

可以手动通过type创建，其实

```python
MyShinyClass = type('MyShinyClass', (), {})  # 返回一个类对象
print MyShinyClass
#输出：<class '__main__.MyShinyClass'>
print MyShinyClass()  #  创建一个该类的实例
#输出：<__main__.MyShinyClass object at 0x1085cd810>
```

 你会发现我们使用“MyShinyClass”作为类名，并且也可以把它当做一个变量来作为类的引用。

接下来我们通过一个具体的例子看看type是如何创建类的，范例：javascript:void(0);)

```python
1、构建Foo类
#构建目标代码
class Foo(object):
    bar = True
#使用type构建
Foo = type('Foo', (), {'bar':True})

2.继承Foo类
#构建目标代码：
class FooChild(Foo):
    pass
#使用type构建
FooChild = type('FooChild', (Foo,),{})

print FooChild
#输出：<class '__main__.FooChild'>
print FooChild.bar   # bar属性是由Foo继承而来
#输出：True

3.为Foochild类增加方法
def echo_bar(self):
    print self.bar

FooChild = type('FooChild', (Foo,), {'echo_bar': echo_bar})
hasattr(Foo, 'echo_bar')
#输出：False
hasattr(FooChild, 'echo_bar')
#输出：True
my_foo = FooChild()
my_foo.echo_bar()
#输出：True
```

可以看到，在Python中，类也是对象，你可以动态的创建类。这就是当我们使用关键字class时Python在幕后做的事情，而这就是通过元类来实现的。

### type创建类与class的比较

### **使用type创建带属性和方法的类**

```python

1.使用type创建带有属性的类,添加的属性是类属性，并不是实例属性
Girl = type("Girl",(),{"country":"china","sex":"male"})
girl = Girl()
print(girl.country,girl.sex)  #使用type创建的类，调用属性时IDE不会自动提示补全
print(type(girl),type(Girl))
'''
china male
<class '__main__.Girl'> <class 'type'>
'''
 
2.使用type创建带有方法的类
#python中方法有普通方法，类方法，静态方法。
def speak(self): #要带有参数self,因为类中方法默认带self参数。
    print("这是给类添加的普通方法")
 
@classmethod
def c_run(cls):
    print("这是给类添加的类方法")
 
@staticmethod
def s_eat():
    print("这是给类添加的静态方法")
 
#创建类，给类添加静态方法，类方法，普通方法。跟添加类属性差不多.
Boy = type("Boy",(),{"speak":speak,"c_run":c_run,"s_eat":s_eat,"sex":"female"})
boy = Boy()
boy.speak()
boy.s_eat() #调用类中的静态方法
boy.c_run() #调用类中类方法
print("boy.sex:",boy.sex)
print(type(boy),type(Boy))
'''
这是给类添加的普通方法
这是给类添加的静态方法
这是给类添加的类方法
boy.sex: female
<class '__main__.Boy'> <class 'type'>
'''
```

### **使用type定义带继承，属性和方法的类**

```python
class Person(object):
    def __init__(self,name):
        self.name = name
    def p(self):
        print("这是Person的方法")
class Animal(object):
    def run(self):
        print("animal can run ")
#定义一个拥有继承的类，继承的效果和性质和class一样。
Worker = type("Worker",(Person,Animal),{"job":"程序员"})
w1 = Worker("tom")
w1.p()
w1.run()
print(type(w1),type(Worker))
'''
这是Person的方法
animal can run 
<class '__main__.Worker'> <class 'type'>
<class '__main__.Person'>
'''
```

 总结：

通过type添加的属性是类属性，并不是实例属性
通过type可以给类添加普通方法，静态方法，类方法，效果跟class一样
type创建类的效果，包括继承等的使用性质和class创建的类一样。本质class创建类的本质就是用type创建。所以可以说python中所有类都是type创建的。

# 自定义元类

元类的主要目的就是为了当创建类时能够自动地改变类。通常，你会为API做这样的事情，你希望可以创建符合当前上下文的类。假想一个很傻的例子，你决定在你的模块里所有的类的属性都应该是大写形式。有好几种方法可以办到，但其中一种就是通过设定__metaclass__。采用这种方法，这个模块中的所有类都会通过这个元类来创建，我们只需要告诉元类把所有的属性都改成大写形式就万事大吉了。

__metaclass__实际上可以被任意调用，它并不需要是一个正式的类。所以，我们这里就先以一个简单的函数作为例子开始。

## 1、使用函数当做元类

```python
# 元类会自动将你通常传给‘type’的参数作为自己的参数传入
def upper_attr(future_class_name, future_class_parents, future_class_attr):
    '''返回一个类对象，将属性都转为大写形式'''
    #选择所有不以'__'开头的属性
    attrs = ((name, value) for name, value in future_class_attr.items() if not name.startswith('__'))
    # 将它们转为大写形式
    uppercase_attr = dict((name.upper(), value) for name, value in attrs)
    #通过'type'来做类对象的创建
    return type(future_class_name, future_class_parents, uppercase_attr)#返回一个类

class Foo(object):
    __metaclass__ = upper_attr
    bar = 'bip' 
```

```python
print hasattr(Foo, 'bar')
# 输出: False
print hasattr(Foo, 'BAR')
# 输出:True
 
f = Foo()
print f.BAR
# 输出:'bip'
```

## 2、使用class来当做元类

由于__metaclass__必须返回一个类。

```python
# 请记住，'type'实际上是一个类，就像'str'和'int'一样。所以，你可以从type继承
# __new__ 是在__init__之前被调用的特殊方法，__new__是用来创建对象并返回之的方法，__new_()是一个类方法
# 而__init__只是用来将传入的参数初始化给对象，它是在对象创建之后执行的方法。
# 你很少用到__new__，除非你希望能够控制对象的创建。这里，创建的对象是类，我们希望能够自定义它，所以我们这里改写__new__
# 如果你希望的话，你也可以在__init__中做些事情。还有一些高级的用法会涉及到改写__call__特殊方法，但是我们这里不用，下面我们可以单独的讨论这个使用

class UpperAttrMetaClass(type):
    def __new__(upperattr_metaclass, future_class_name, future_class_parents, future_class_attr):
        attrs = ((name, value) for name, value in future_class_attr.items() if not name.startswith('__'))
        uppercase_attr = dict((name.upper(), value) for name, value in attrs)
        return type(future_class_name, future_class_parents, uppercase_attr)#返回一个对象，但同时这个对象是一个类
```

 但是，这种方式其实不是OOP。我们直接调用了type，而且我们没有改写父类的__new__方法。现在让我们这样去处理:

```python
class UpperAttrMetaclass(type):
    def __new__(upperattr_metaclass, future_class_name, future_class_parents, future_class_attr):
        attrs = ((name, value) for name, value in future_class_attr.items() if not name.startswith('__'))
        uppercase_attr = dict((name.upper(), value) for name, value in attrs)
 
        # 复用type.__new__方法
        # 这就是基本的OOP编程，没什么魔法。由于type是元类也就是类，因此它本身也是通过__new__方法生成其实例，只不过这个实例是一个类.
        return type.__new__(upperattr_metaclass, future_class_name, future_class_parents, uppercase_attr)
```

你可能已经注意到了有个额外的参数upperattr_metaclass，这并没有什么特别的。类方法的第一个参数总是表示当前的实例，就像在普通的类方法中的self参数一样。当然了，为了清晰起见，这里的名字我起的比较长。但是就像self一样，所有的参数都有它们的传统名称。因此，在真实的产品代码中一个元类应该是像这样的：

```python
class UpperAttrMetaclass(type):
    def __new__(cls, name, bases, dct):
        attrs = ((name, value) for name, value in dct.items() if not name.startswith('__')
        uppercase_attr  = dict((name.upper(), value) for name, value in attrs)
        return type.__new__(cls, name, bases, uppercase_attr)
```

如果使用super方法的话，我们还可以使它变得更清晰一些。

```python
class UpperAttrMetaclass(type):
    def __new__(cls, name, bases, dct):
        attrs = ((name, value) for name, value in dct.items() if not name.startswith('__'))
        uppercase_attr = dict((name.upper(), value) for name, value in attrs)
        return super(UpperAttrMetaclass, cls).__new__(cls, name, bases, uppercase_attr)
```

# 参考

<https://www.cnblogs.com/tkqasn/p/6524879.html>

<https://blog.csdn.net/qq_26442553/article/details/82459234>

