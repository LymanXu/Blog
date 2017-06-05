---
layout: post
#标题配置
title:  Python Collection相关总结
#时间配置
date:   2017-06-05 10:45:00 +0800
#大类配置
categories: Python
#小类配置
tag: Collection
---

* content
{:toc}

# List
## 基础概念
1. 列表的数据项不需要具有相同的类型
2. 把逗号分隔的不同的数据项使用方括号括起来即可

## 常用function
1. +号进行组合，*进行复制
2. len(), max(), min(), list()元组转列表


## 需要注意的method
**list.pop() and list.remove()**

pop入参索引，默认为-1，remove入参为元素（remove 一次操作移除第一个符合条件的元素）
```python
#!usr/bin/python

aList = ['a', 'b', 'c', 'a', 'b', 1, 2, 3]

for a in aList:
    if a == 'a':
        aList.remove(a)
print aList

aList.remove('b')
print aList

aList = ['a', 'b', 'c', 'a', 'b', 1, 2, 3]
delList = ['a', 'b']

for a in aList:
    if a in delList:
        aList.remove(a)
print aList
result:
['b', 'c', 'b', 1, 2, 3]    ----没有解释出使用remove无法循环删除的问题
['c', 'b', 1, 2, 3]
['b', 'c', 'b', 1, 2, 3]    ----b没有移除成功
```




