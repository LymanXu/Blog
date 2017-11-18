---
layout: post
#标题配置
title:  Common install
#时间配置
date:   2017-11-14 16:40:00 +0800
#大类配置
categories: 
#小类配置
tag: 
---

* content
{:toc}

# 1. Jdk安装配置
```buildoutcfg
wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u151-b12/e758a0de34e24606bca991d704f6dcbf/jdk-8u151-linux-x64.tar.gz
后面的链接按照版本替换
tar -zxvf jdk-8u151-linux-x64.tar.gz
mv jdk1.8.0_151 /opt/jdk

vim /etc/profile
加入：
export JRE_HOME=${JAVA_HOME}/jre  
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib  
export PATH=${JAVA_HOME}/bin:$PATH  

生效服务
source /etc/profile  

```

# 2. Idea安装




