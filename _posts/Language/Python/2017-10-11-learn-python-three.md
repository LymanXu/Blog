---
layout: post
#标题配置
title:  LearnPython(三)－－异常
#时间配置
date:   2017-10-10 16:50:00 +0800
#大类配置
categories: Python
#小类配置
tag: 
---

* content
{:toc}

# 1. 异常处理
```buildoutcfg
try:
    try_suite
except ExceptionType [,e]:
    exception block
else:
    no_exception_block
finally:
    fianlly_block
```
对于finally需要注意，使用finally遇到异常的时候，会先执行finally的代码块然后再将异常抛出
