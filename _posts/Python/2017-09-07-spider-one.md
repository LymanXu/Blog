---
layout: post
#标题配置
title:  Python 信息爬虫(一)
#时间配置
date:   2017-09-07 15:30:00 +0800
#大类配置
categories: Python
#小类配置
tag: 
---

* content
{:toc}

# 1. 简单爬虫的架构
包括URL管理（用来生成资源的定位符）、网页下载器(资源定位后进行网页的下载)、网页解析(网页下载后进行有效信息的解析)

## 1.1 URL管理器
生成管理要爬取的资源定位符，防止重复爬取信息。对于url的集合可以放到内存中，关系数据库中，以及速度较快的缓存数据库中（redis中使用set）

## 1.2 urllib包
urllib提供一个高级的WEB通信库，支持最基本的Web协议，功能就是利用基本的协议从网络上下载这些数据。

下载打开网页的方式：<br/>
1. 最基础的直接使用f.open()
  ```buildoutcfg
import urllib2

f = urllib2.urlopen('http://www.xxx.con')

print f.getcode()
content = f.rend()
```

2. 使用Request进行带有HTTP验证信息
  ```buildoutcfg
import urllib2
from base64 import encodestring

req = urllib2.Request(url)
b64str = encodestring('%s:%s' % (Login,Password))[:-1]
req.addheader("Authorization", "Basic %s" % b64str)

f = urllib2.urlopen(req)

```

3. 使用开启器进行有HTTP登录信息的验证
  ```buildoutcfg
import urllib2

def handler_version(url):
    from urlparse import urlparse
    hdlr = urllib2.HTTPBasicAuthHandler()
    hdlr.add_password(REALM,
         urlparse(url)[1], Login, Password)
    opener = urllib2.build_opener(hdlr)
    urllib2.install_opener(openrc)
    
    f = urllib2.urlopen(url)
```

## 1.3 网页解析器
将整个网页下载为一个DOM对象树的形式，进行结构化解析（document object model)