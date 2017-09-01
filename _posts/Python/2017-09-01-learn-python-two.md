---
layout: post
#标题配置
title:  LearnPython(二)－－装饰器
#时间配置
date:   2017-09-01 13:25:00 +0800
#大类配置
categories: Python
#小类配置
tag: 
---

* content
{:toc}

# 1. Decorator装饰器
装饰器用来做什么：不想改动原函数的情况下为函数动态的添加功能，通过高阶函数传入原函数进行包装返回新的函数

```buildoutcfg
ex1:包装函数，实现调用信息的打印

def f(x):
    return x*2
    
def log_f(f):
    def new_f(x):
        print 'call'+f.__name__+'()'
        return f(x)
    return new_f
    
```
使用＠语法简化装饰器调用
```buildoutcfg
@new_f
def f(x):
    return x*2
    
即等效于,函数f　被装饰后又重新赋给给　f
def f(x):
    return x*2
f = new_f(f)
```
简化代码，常用装饰器<br/>
1. @log , 打印日志
2. @performance ，检测性能
3. @transaction ,数据库事务
4. @post('/register') ,URL路由

# 2. 入参定义和可变参数
1. 默认值
  def f(a, b=0)  b有默认值是一个可选的参数，没有对应入参的话会默认为默认值
  
2. *可变参数
```buildoutcfg
def funcD(a, b, *c):
  print a
  print b
  print "length of c is: %d " % len(c)
  print c
调用funcD(1, 2, 3, 4, 5, 6)结果是
1
2
length of c is: 4
(3, 4, 5, 6)
```

3. **键值对参数
如果一个函数定义中的最后一个形参有 ** （双星号）前缀,所有正常形参之外的其他的关键字参数都将被放置在一个字典中传递给函数
```buildoutcfg
def funcF(a, **b):
  print a
  for x in b:
    print x + ": " + str(b[x])
funcF(100, c='你好', b=200)
```

# 3. 无入参的decorator
修饰器本身无可变入参，被修饰函数入参不确定的情况
```buildoutcfg
def log(f):
    def fn(x):
        print 'call ' + f.__name__ + '()...'
        return f(x)
    return fnimport functools
def log(f):
    @functools.wraps(f)
    def wrapper(*args, **kw):
        print 'call...'
        return f(*args, **kw)
    return wrapper
    
@log
def add(x, y):
    return x + y
print add(1, 2)
#如此调用会报错，入参不符合，fn(x)是一个参数，所以报错，对于多参数调用可以使用　*, **

def log(f):
    def fn(*args, **kw):
        print 'call ' + f.__name__ + '()...'
        return f(*args, **kw)
    return fn
使ｌｏｇ自适应任何参数定义的函数
```

# 4. 带参数Decorator
```buildoutcfg
@log('DEBUG')
def my_func():
    pass
    
相当于：
log_decorator = log('DEBUG')
@log_decorator
def my_func():
    pass
 

def log(prefix):
    def log_decorator(f):
        def wrapper(*args, **kw):
            print '[%s] %s()...' % (prefix, f.__name__)
            return f(*args, **kw)
        return wrapper
    return log_decorator

@log('DEBUG')
def test():
    pass
print test()
```
通过使用带参数的log 返回一个log_decorator(f),在log_decorator中接受函数并返回新的函数；这里借助了Ｐｙｔｈｏｎ的闭包特性，在内层函数中引用外层函数的变量

# 5. 函数包装后同步函数的属性
```buildoutcfg
def log(f):
    def wrapper(*args, **kw):
        print 'call...'
        return f(*args, **kw)
    return wrapper
@log
def f2(x):
    pass
print f2.__name__
```
输出的为wrapper,@log后返回的不再是f(),而是log内部定义的wrapper,导致依赖函数名的代码不可用，如何进行属性的自动同步呢
Python内置的functools可以用来自动化完成这个“复制”的任务
```buildoutcfg
import functools
def log(f):
    @functools.wraps(f)
    def wrapper(*args, **kw):
        print 'call...'
        return f(*args, **kw)
    return wrapper
```
注意点：参数名信息无法同步，即使采用固定的参数也无法同步复制

# 6. 偏函数
对于原函数有很多参数的时候，为了减少调用时提供的参数，简化调用
借助functools.partial实现
```buildoutcfg
import functools
int2 = functools.partial(int, base=2)
int2('1000000')
64
int2('1010101')
85
```
int()默认按十进制转化，当int('',base=2)时按二进制转化