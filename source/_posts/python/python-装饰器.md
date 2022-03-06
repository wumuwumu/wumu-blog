---
title: python-装饰器
tags: python
abbrlink: 3a5cdcf8
date: 2019-07-31 19:44:57
---

# 简单的装饰器

```python
import logging


def use_logging(func):

    def wrapper():
        logging.warning("%s is running" % func.__name__)
        return func()   # 把 foo 当做参数传递进来时，执行func()就相当于执行foo()
    return wrapper

def foo():
    print('i am foo')

foo = use_logging(foo)  # 因为装饰器 use_logging(foo) 返回的时函数对象 wrapper，这条语句相当于  foo = wrapper
foo()                   # 执行foo()就相当于执行 wrapper()
'''
WARNING:root:foo is running
i am foo
'''
```

# @ 语法糖

```python
def use_logging(func):

    def wrapper():
        logging.warn("%s is running" % func.__name__)
        return func()
    return wrapper

@use_logging
def foo():
    print("i am foo")

foo()
```

# *args、**kwargs

可能有人问，如果我的业务逻辑函数 foo 需要参数怎么办？比如：

```
def foo(name):
    print("i am %s" % name)
```

我们可以在定义 wrapper 函数的时候指定参数：

```python
def wrapper(name):
        logging.warn("%s is running" % func.__name__)
        return func(name)
    return wrapper
```

这样 foo 函数定义的参数就可以定义在 wrapper 函数中。这时，又有人要问了，如果 foo 函数接收两个参数呢？三个参数呢？更有甚者，我可能传很多个。当装饰器不知道 foo 到底有多少个参数时，我们可以用 *args 来代替：

```python
def wrapper(*args):
        logging.warn("%s is running" % func.__name__)
        return func(*args)
    return wrapper
```

如此一来，甭管 foo 定义了多少个参数，我都可以完整地传递到 func 中去。这样就不影响 foo 的业务逻辑了。这时还有读者会问，如果 foo 函数还定义了一些关键字参数呢？比如：

```python
def foo(name, age=None, height=None):
    print("I am %s, age %s, height %s" % (name, age, height))
```

这时，你就可以把 wrapper 函数指定关键字函数：

```python
def wrapper(*args, **kwargs):
        # args是一个数组，kwargs一个字典
        logging.warn("%s is running" % func.__name__)
        return func(*args, **kwargs)
    return wrapper
```

# 带参数的装饰器

装饰器还有更大的灵活性，例如带参数的装饰器，在上面的装饰器调用中，该装饰器接收唯一的参数就是执行业务的函数 foo 。装饰器的语法允许我们在调用时，提供其它参数，比如`@decorator(a)`。这样，就为装饰器的编写和使用提供了更大的灵活性。比如，我们可以在装饰器中指定日志的等级，因为不同业务函数可能需要的日志级别是不一样的。

```python
def use_logging(level):
    def decorator(func):
        def wrapper(*args, **kwargs):
            if level == "warn":
                logging.warn("%s is running" % func.__name__)
            elif level == "info":
                logging.info("%s is running" % func.__name__)
            return func(*args)
        return wrapper

    return decorator

@use_logging(level="warn")
def foo(name='foo'):
    print("i am %s" % name)

foo()
```

上面的 use_logging 是允许带参数的装饰器。它实际上是对原有装饰器的一个函数封装，并返回一个装饰器。我们可以将它理解为一个含有参数的闭包。当我 们使用`@use_logging(level="warn")`调用的时候，Python 能够发现这一层的封装，并把参数传递到装饰器的环境中。

```python
@use_logging(level="warn")`等价于`@decorator
```

# 类装饰器

没错，装饰器不仅可以是函数，还可以是类，相比函数装饰器，类装饰器具有灵活度大、高内聚、封装性等优点。使用类装饰器主要依靠类的`__call__`方法，当使用 @ 形式将装饰器附加到函数上时，就会调用此方法。

```python
class Foo(object):
    def __init__(self, func):
        self._func = func

    def __call__(self):
        print ('class decorator runing')
        self._func()
        print ('class decorator ending')

@Foo
def bar():
    print ('bar')

bar()
```

### functools.wraps

使用装饰器极大地复用了代码，但是他有一个缺点就是原函数的元信息不见了，比如函数的`docstring`、`__name__`、参数列表，先看例子：

```python
# 装饰器
def logged(func):
    def with_logging(*args, **kwargs):
        print func.__name__      # 输出 'with_logging'
        print func.__doc__       # 输出 None
        return func(*args, **kwargs)
    return with_logging

# 函数
@logged
def f(x):
   """does some math"""
   return x + x * x

logged(f)
```

不难发现，函数 f 被`with_logging`取代了，当然它的`docstring`，`__name__`就是变成了`with_logging`函数的信息了。好在我们有`functools.wraps`，`wraps`本身也是一个装饰器，它能把原函数的元信息拷贝到装饰器里面的 func 函数中，这使得装饰器里面的 func 函数也有和原函数 foo 一样的元信息了。

```python
from functools import wraps
def logged(func):
    @wraps(func)
    def with_logging(*args, **kwargs):
        print func.__name__      # 输出 'f'
        print func.__doc__       # 输出 'does some math'
        return func(*args, **kwargs)
    return with_logging

@logged
def f(x):
   """does some math"""
   return x + x * x
```

# 装饰器顺序

一个函数还可以同时定义多个装饰器，比如：

```python
@a
@b
@c
def f ():
    pass
```

它的执行顺序是从里到外，最先调用最里层的装饰器，最后调用最外层的装饰器，它等效于

```python
f = a(b(c(f)))
```

# 补充

## *与**区别

在Python的函数定义中使用*args和**kwargs可传递可变参数。*args用作传递非命名键值可变长参数列表（位置参数），**kwargs用作传递键值可变长参数列表。在函数调用的时候也有解构的使用

```python
def test_var_args(farg, *args):
    print "formal arg:", farg
    for arg in args:
        print "another arg:", arg
 
test_var_args(1, "two", 3)
'''
formal arg: 1
another arg: two
another arg: 3
'''
```

```python
def test_var_kwargs(farg, **kwargs):
    print "formal arg:", farg
    for key in kwargs:
        print "another keyword arg: %s: %s" % (key, kwargs[key])
 
test_var_kwargs(farg=1, myarg2="two", myarg3=3)
'''
Required argument:  1
Optional argument (*args):  2
Optional argument (*args):  3
Optional argument (*args):  4
Optional argument k2 (*kwargs): 6
Optional argument k1 (*kwargs): 5
'''
```

```python
def test_var_args_call(arg1, arg2, arg3):
    print "arg1:", arg1
    print "arg2:", arg2
    print "arg3:", arg3
 
args = ("two", 3)
test_var_args_call(1, *args)
```

```python
def test_var_args_call(arg1, arg2, arg3):
    print "arg1:", arg1
    print "arg2:", arg2
    print "arg3:", arg3
 
kwargs = {"arg3": 3, "arg2": "two"}
test_var_args_call(1, **kwargs)
```



# 参考

> <https://foofish.net/python-decorator.html>
>
> <https://www.biaodianfu.com/python-args-kwargs.html>
>
> <https://my.oschina.net/leejun2005/blog/477614> 例子介绍的很详细