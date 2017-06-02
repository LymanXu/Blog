---
layout: post
#标题配置
title:  Python 编程skills
#时间配置
date:   2017-06-02 11:44:00 +0800
#大类配置
categories: Python
#小类配置
tag: Skills
---

* content
{:toc}

# 常用基础skill
## 三元表达式

对于类似C语言的三元条件表达式condition ? true_part : false_part，虽然Python没有三目运算符(?:)，但也有类似的替代方案，那就是true_part if condition else false_part。


> "Fire" if True else "Water"  
 'Fire'  
> "Fire" if False else "Water"  
 'Water' 

 
## For in理解

zip(seq1[,seq2,seq3])将多个数组中对应的元素打包成一个Tuple，然后返回一个list<br/>
For in 在数组中赋值时，for in出来的元素个数和新生成的集合中的元素个数相等 

![For in 数组赋值]({{'/styles/images/Python/0602-little-01.JPG' | prepend: site.baseurl }})
