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
2. sudo: ovs: command not found

  没有安装openvswitch<br/>
  sudo apt install openvswitch-switch
3. SSH登录
  
  命令格式： ssh username@IP (ssh stack@192.168.56.102)
  
# 2. Route添加/删除路由

**查看**<br/>
  route -n
  
  sudo vim /etc/network/interfaces

# 2.1 删除/添加默认网关
route del default gw 192.168.120.240
route add default gw 192.168.120.240

# 2.2 添加/删除路由
  添加<br/>
  route add -net 9.123.0.0 netmask 255.255.0.0 gw 9.123.0.1

  删除<br/>
  route del -net 9.123.0.0 netmask 255.255.0.0 gw 9.123.0.1


# 2. VirtualBox常见网络类型的区别