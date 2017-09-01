---
layout: post
#标题配置
title:  LearnPython(一)
#时间配置
date:   2017-06-05 10:45:00 +0800
#大类配置
categories: Python
#小类配置
tag: 
---

* content
{:toc}

# 1. 函数式编程
函数和函数式的关系就像计算和计算机的关系，C语言对应函数编程，Python较C更抽象对应函数式编程。

Python支持的函数式编程：<br/>
1. 不是纯的函数式编程，有变量
2. 支持高阶函数，函数可以作为变量传入
3. 支持闭包，有闭包就可以返回函数
4. 有限度支持匿名函数

# 2. 高阶函数
## 2.1 变量可以指向函数
```buildoutcfg
>>> abs
<built-in function abs>
>>> abs(-10)
10
>>> f=abs
>>> f(-10)
10

```
函数名就是指向函数的变量，和普通的函数没有什么区别，仅是指向的对象是函数
```buildoutcfg
>>> abs
<built-in function abs>
>>> abs(-10)
10
>>> len
<built-in function len>
>>> len([1,2,3])
3
>>> abs=len
>>> abs(-10)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: object of type 'int' has no len()
>>> abs([1,2,3])
3

```
## 2.2 高阶函数
变量可以指向函数，能接受函数做参数的函数就是高阶函数
```buildoutcfg
def add(x,y,f):
    return f(x)+f(y)

add(-5,9,abs)
14
```

## 2.3 常用内置高阶函数
### 2.3.1 map()
接受一个List和一个function，对List中的每个元素使用function处理得到新的List;可以理解为map()的作用就是转换List
```buildoutcfg
ex1:

def f(x):
    return x*x
    
print map(f, range[1,4])

ex2:
def f(s):
    return s[0].upper()+s[1:].lower()
    
print map(f, ['aafd', 'BBBD'])
```

### 2.3.2 reduce()
入参为 function + list +[初始值]，function 要有两个入参，对List中的每个元素反复调用

```buildoutcfg
ex1:
def f(x,y):
    return x+y
    
print reduce(f, range(1:4))
实现对List求和
print reduce(f, range(1:4), 100) 初始值默认设为１００，然后对每个元素一次调用、

ex2:
reduce(lambda x,y: x*y, range(1,4))
```

### 2.3.3 filter()
入参 function + list,对每个元素进行function过滤掉返回false　不符合的元素

```buildoutcfg
ex1:
def is_odd(x):
    return x % 2 == 1
    
filter(is_odd, range(1,7))

def is_not_empty(s):
    return s and len(s.strip()) > 0
filter(is_not_empty, ['test', None, '', 'str', '  ', 'END'])

s.strip(rm) 删除 s 字符串中开头、结尾处的 rm 序列的字符。
当rm为空时，默认删除空白符（包括'\n', '\r', '\t', ' '

```
### 2.3.4 sort()
比较函数的定义是，传入两个待比较的元素 x, y，如果 x 应该排在 y 的前面，返回 -1，如果 x 应该排在 y 的后面，返回 1。如果 x 和 y 相等，返回 0
```buildoutcfg
ex1:　实现倒序
def reversed_cmp(x, y):
    if x > y:
        return -1
    if x < y:
        return 1
    return 0
    
sorted([36, 5, 12, 9, 21], reversed_cmp)
```

# 3. 闭包
内层函数引用了外层函数的变量（参数也算变量），然后返回内层函数的情况，称为闭包（Closure）

# 3.1 返回函数
可以在一个函数内部定义一个新的函数，然后返回新的函数
```buildoutcfg
def myabs():
    return abs   # 返回函数
def myabs2(x):
    return abs(x)   # 返回函数调用的结果，返回值是一个数值
```

返回函数的作用：<br/>
对一些计算进行延迟计算，以及也可以想到在不修改原函数的情况下二次定义原有函数，为其添加新的功能特性
```buildoutcfg
def calc_sum(lst):
    def lazy_sum():
        return sum(lst)
    return lazy_sum
    
f = calc_sum([1, 2, 3, 4]) 调用　calc_sum()并没有计算，是、而是返回了求和计算的函数
f()　对返回的函数进行计算
```

闭包的特点：<br/>
返回函数引用外层函数的局部变量，因此使用闭包要确保引用的局部变量在函数返回后不能改变，返回闭包不能引用循环变量

```buildoutcfg
# 希望一次返回3个函数，分别计算1x1,2x2,3x3:
def count():
    fs = []
    for i in range(1, 4):
        def f():
             return i*i
        fs.append(f)
    return fs

f1, f2, f3 = count()
# 结果都返回９，因为调用时　i 已经变为　３

def count():
    fs = []
    for i in range(1, 4):
        def f(i):
            return lambda : i*i
        fs.append(f(i))
    return fs
f1, f2, f3 = count()
print f1(), f2(), f3()
#正常返回，对于循环的入参f(i)相当于进行函数的调用，再次进行内置的函数的使用，保证计算的延迟转移

```


# 4. 匿名函数
匿名函数lambda x: x*x表示，不用定义函数名简化代码
```buildoutcfg
sorted([1, 3, 9, 5, 0], lambda x,y: -cmp(x,y))

#返回函数的使用
myabs = lambda x: -x if x < 0 else x 
myabs(-1)

```
