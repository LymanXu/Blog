---
layout: post
#标题配置
title:  Tricircle多节点部署
#时间配置
date:   2017-07-04 15:17:00 +0800
#大类配置
categories: Tricircle
#小类配置
tag: 教程
---

* content
{:toc}

# 1. 问题List
1. Node的网卡配置
  **Node1的网卡配置**<br/>
  三个网卡：一个NAT用于访问互联网；一个Bridge用于和其他Neutron互联；一个Host Only用于ssh登录
  
  **Node2的网卡配置**<br/>
  四个网卡：一个NAT用于访问互联网；两个Bridge用于Tricircle;一个Host Only用于ssh登录