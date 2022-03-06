---
title: python中and和or用法
tags:
  - python
abbrlink: db7d259c
date: 2019-10-25 15:41:30
---

在[Python](http://lib.csdn.net/base/python) 中，and 和 or 执行布尔逻辑演算，如你所期待的一样。但是它们并不返回布尔值，而是返回它们实际进行比较的值之一。

（类似C++里面的&&和||的短路求值）

（ 在布尔环境中，0、”、[]、()、{}、None为假；其它任何东西都为真。但是可以在类中定义特定的方法使得类实例的演算值为假。）

# and实例：

```python
>>> 'a' and 'b'
'b'
>>> '' and 'b'
''
>>> 'a' and 'b' and 'c'
'c'12345
```

从左到右扫描，返回第一个为假的表达式值，无假值则返回最后一个表达式值。

# or实例：

```python
>>> 'a' or 'b'
'a'
>>> '' or 'b'
'b'
>>> '' or [] or{}
{}12345
```

从左到右扫描，返回第一个为真的表达式值，无真值则返回最后一个表达式值。

# and-or搭配：

```python
>>> a = "betabin"
>>> b = "python"
>>> 1 and a or b
'betabin'
>>> 0 and a or b
'python'12345
```

看起来类似于于我们Ｃ＋＋中的条件运算符（bool？a：b），是的，当a为true的时候是一样的。但是，当a为false的时候，就明显不同了。

如果坚持要用and-or技巧来实现条件运算符的话，可以用种安全的方法：

```python
>>> a = ""
>>> b = "betabin"
>>> (1 and [a] or [b])[0]
''123
```

就是万能的[]，把a为假的可能性给抹杀掉，然后通过[0]再获得（因为要通过[0]获得元素，所以b也得加上[]）。



这个and-or技巧主要在lambda中使用。