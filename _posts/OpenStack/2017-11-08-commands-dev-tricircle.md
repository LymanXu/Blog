---
layout: post
#标题配置
title:  Commands list in develop Tricircle
#时间配置
date:   2017-11-08 20:22:00 +0800
#大类配置
categories: Openstack
#小类配置
tag: 
---

* content
{:toc}

# 1. 
## 1.1 tricircle 的打包和安装更新

```buildoutcfg
打包：
sudo python setup.py bdist_wheel

更新：
sudo pip uninstall tricircleclient
sudo pip install /path/to/xxx.whl
```

## 1.2 vim查找
```buildoutcfg
查找到第n行：
命令模式下　:n

查找特定字符xx:
命令模式下　/xx         (note: n向下继续查找，N向上继续查找)
```

## 1.3 查看进程
```buildoutcfg
ps -aux


```

## 1.4 scp远程拷贝
```buildoutcfg
scp -r tricircle_requests/ stack@192.168.56.107:/opt/stack
```

## 1.5 openstack重启服务
```buildoutcfg
查看服务的运行命令：
systemctl show devstack@n-sch.service -p ExecStart --no-pager
（note: 用来使用命令重新运行服务）

停掉服务：
sudo systemctl stop devstack@n-cpu.service

查看服务日志：
sudo journalctl -f --unit devstack@n-cpu.service --unit devstack@n-cond.service
sudo journalctl -f --unit devstack@n-*
```

## 1.6 ovs: command not found
没有安装openvswitch<br/>
```buildoutcfg
sudo apt install openvswitch-switch
```

## 1.7 删除/添加默认网关
route del default gw 192.168.120.240
route add default gw 192.168.120.240

## 1.8 添加/删除路由
添加<br/>
route add -net 9.123.0.0 netmask 255.255.0.0 gw 9.123.0.1

删除<br/>
route del -net 9.123.0.0 netmask 255.255.0.0 gw 9.123.0.1

## 1.9 添加/删除virtual IP
linux添加和删除virtual ip
网卡上增加一个IP：
ifconfig eth0:1 192.168.0.1 netmask 255.255.255.0

删除网卡的一个IP地址:
ip addr del 192.168.0.1 dev eth0
r
## 1.10 添加/删除 Vitrual network interface
virtual network interface虚拟网卡的添加删除
查看ip
ip -o -f inet addr show

ifconfig tunl0 down

## 1.11 unable to lock the administration
找出并杀掉所有apt-get或者apt进程
```buildoutcfg
ps -A |grep apt

sudo kill -9 processnumber(上面显示的列表的第一列)
或
sudo kill -SIGKILL processnumber
```