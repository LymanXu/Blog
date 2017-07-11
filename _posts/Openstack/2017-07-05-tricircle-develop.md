---
layout: post
#标题配置
title:  Tricircle开发问题List
#时间配置
date:   2017-07-05 15:17:00 +0800
#大类配置
categories: Tricircle
#小类配置
tag: 教程
---

* content
{:toc}

# 1. 问题List
## 1.1 import fixtures error
没有该种模块，进行pip安装到该项目下
```buildoutcfg
sudo pip install fixtures
```
# 2. Test问题
Python是个动态语言，很多问题都无法通过静态编译检查来发现，单元测试成了一个重要的确保质量的手段<br/>
[**tox单元测试过程&如何配置**](https://blog.apporc.org/2016/08/python-%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E5%B7%A5%E5%85%B7-tox/)<br/>
[**单元测试工具介绍**](http://www.tuicool.com/articles/UnQbyyv)<br/>
tox 用来管理和构建不同类型单元测试所需要的环境，如py27依赖一些库的环境，py35,pypy
## 2.2 tox pypy error
```buildoutcfg
sudo add-apt-repository ppa:pypy/ppa
sudo apt-get update
(或直接运行安装)
sudo apt-get install pypy pypy-dev
```


   