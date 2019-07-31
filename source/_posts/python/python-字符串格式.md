---
title: python-字符串格式
date: 2019-07-31 10:59:16
tags: python
---

# 格式化操作符（%）

"%"是Python风格的字符串格式化操作符，非常类似C语言里的printf()函数的字符串格式化（C语言中也是使用%）。

下面整理了一下Python中字符串格式化符合：

| 格式化符号 | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| %c         | 转换成字符（ASCII 码值，或者长度为一的字符串）               |
| %r         | 优先用repr()函数进行字符串转换                               |
| %s         | 优先用str()函数进行字符串转换                                |
| %d / %i    | 转成有符号十进制数                                           |
| %u         | 转成无符号十进制数                                           |
| %o         | 转成无符号八进制数                                           |
| %x / %X    | 转成无符号十六进制数（x / X 代表转换后的十六进制字符的大小写） |
| %e / %E    | 转成科学计数法（e / E控制输出e / E）                         |
| %f / %F    | 转成浮点数（小数部分自然截断）                               |
| %g / %G    | %e和%f / %E和%F 的简写                                       |
| %%         | 输出% （格式化字符串里面包括百分号，那么必须使用%%）         |

这里列出的格式化符合都比较简单，唯一想要强调一下的就是"%s"和"%r"的差别。

看个简单的代码：

```python
string = "Hello\tWill\n"

print("%s" %string)
print("%r" %string)
'''
Hello   Will

'Hello\tWill\n'
'''
```

补充：

Python打印值的时候会保持该值在Python代码中的状态，不是用户所希望看到的状态。而使用print打印值则不一样，print打印出来的值是用户所希望看到的状态。 str和repr的区别：

1. str

   把值转换为合理形式的字符串，给用户看的。str实际上类似于int，long，是一种类型。

   ```python
   print str("Hello,  world!")
   # Hello,  world!            
   print str(1000L)
   # 1000                         
   str("Hello, world!")
   # 'Hello, world!'               # 字符串转换之后仍然是字符串
   str(1000L)
   # '1000'
   ```

2. repr()

   创建一个字符串，以合法python表达式的形式来表示值。repr()是一个函数。

   ```python
   print repr("Hello,  world!")
   # 'Hello,  world!'
   print repr(1000L)
   # 1000L
   repr("Hello,  world!")
   # "'Hello,  world!'"
   repr(1000L)
   # '1000L'
   ```

# 格式化操作辅助符



通过"%"可以进行字符串格式化，但是"%"经常会结合下面的辅助符一起使用。

| **辅助符号** | **说明**                                                     |
| ------------ | ------------------------------------------------------------ |
| *****        | 定义宽度或者小数点精度                                       |
| **-**        | 用做左对齐                                                   |
| **+**        | 在正数前面显示加号(+)                                        |
| **#**        | 在八进制数前面显示零(0)，在十六进制前面显示"0x"或者"0X"（取决于用的是"x"还是"X"） |
| **0**        | 显示的数字前面填充"0"而不是默认的空格                        |
| **(var)**    | 映射变量（通常用来处理字段类型的参数）                       |
| **m.n**      | m 是显示的最小总宽度，n 是小数点后的位数（如果可用的话）     |

```python
num = 100

print("%d to hex is %x" %(num, num))
print("%d to hex is %X" %(num, num))
print("%d to hex is %#x" %(num, num))
print("%d to hex is %#X" %(num, num))

# 浮点数
f = 3.1415926
print("value of f is: %.4f" %f)

# 指定宽度和对齐
students = [{"name":"Wilber", "age":27}, {"name":"Will", "age":28}, {"name":"June", "age":27}]
print("name: %10s, age: %10d" %(students[0]["name"], students[0]["age"]))
print("name: %-10s, age: %-10d" %(students[1]["name"], students[1]["age"]))
print("name: %*s, age: %0*d" %(10, students[2]["name"], 10, students[2]["age"]))

# dict参数
for student in students:
    print("%(name)s is %(age)d years old" %student)
    
'''
100 to hex is 64
100 to hex is 64
100 to hex is 0x64
100 to hex is 0X64
value of f is: 3.1416
name:     Wilber, age:         27
name: Will      , age: 28        
name:       June, age: 0000000027
Wilber is 27 years old
Will is 28 years old
June is 27 years old
'''
```

# 字符串模板

其实，在Python中进行字符串的格式化，除了格式化操作符，还可以使用string模块中的字符串模板（Template）对象。下面就主要看看Template对象的substitute()方法：

```python
from string import Template
sTemp = Template('Hi ,$name,$$ ')
print(sTemp.substitute(name='wumu'))
'''
Hi ,wumu,$ 
'''
```

# format

```python
# 位置参数
print("{} is {} years old".format("Wilber", 28))
print("Hi, {0}! {0} is {1} years old".format("Wilber", 28))

# 关键字参数
print("{name} is {age} years old".format(name = "Wilber", age = 28))

# 下标参数
li = ["Wilber", 28]
print("{0[0]} is {0[1]} years old".format(li))

# 填充与对齐
# ^、<、>分别是居中、左对齐、右对齐，后面带宽度
# :号后面带填充的字符，只能是一个字符，不指定的话默认是用空格填充
print('{:>8}'.format('3.14'))
print('{:<8}'.format('3.14'))
print('{:^8}'.format('3.14'))
print('{:0>8}'.format('3.14'))
print('{:a>8}'.format('3.14'))

# 浮点数精度
print('{:.4f}'.format(3.1415926))
print('{:0>10.4f}'.format(3.1415926))

# 进制
# b、d、o、x分别是二进制、十进制、八进制、十六进制
print('{:b}'.format(11))
print('{:d}'.format(11))
print('{:o}'.format(11))
print('{:x}'.format(11))
print('{:#x}'.format(11))
print('{:#X}'.format(11))

# 千位分隔符
print('{:,}'.format(15700000000))

'''
Wilber is 28 years old
Hi, Wilber! Wilber is 28 years old
Wilber is 28 years old
Wilber is 28 years old
    3.14
3.14    
  3.14  
00003.14
aaaa3.14
3.1416
00003.1416
1011
11
13
b
0xb
0XB
15,700,000,000
'''
```



# 参考

> <https://www.cnblogs.com/wilber2013/p/4641616.html>