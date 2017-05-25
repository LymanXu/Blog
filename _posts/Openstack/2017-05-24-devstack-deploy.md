---
layout: post
#标题配置
title:  Devstack 部署
#时间配置
date:   2017-05-24 23:00:00 +0800
#大类配置
categories: Openstack
#小类配置
tag: openstack搭建
---

* content
{:toc}

# 搭建Devstack

1. 新建vm进行配置

   网络：

        网卡1：连接方式：NAT    控制：IntelPRO/1000  混杂：拒绝   接入网线 true

        网卡2：连接方式：桥接网卡 控制： Qualcomm Atheros   混杂：全部允许  接入网线 true

        网卡3：连接方式：桥接网卡    控制：Realtek PCIe  混杂：全部允许  接入网线 true

        网卡4：连接方式：仅主机网络 控制：Virtual Host-Only   混杂：拒绝   接入网线 true

   系统：

        memory 7g, cpus 3, disk 40g

2. 安装Ubuntu系统时直接注册stack作为用户
3. git clone https://git.openstack.org/openstack-dev/devstack
4. cd devstack， 创建local.conf 配置文件（注意： =前后 不需要空格，第一次被坑了好久）
5. devstack安装时，觉费时的是git clone各种组件代码

   手动clone需要的各个组件代码，在 /opt/stack录下clone
   sudo git clone git://git.openstack.org/openstack/nova.git /opt/stack/nova

   sudo git clone git://git.openstack.org/openstack/glance.git /opt/stack/glance

   sudo git clone git://git.openstack.org/openstack/cinder.git /opt/stack/cinder

   sudo git clone git://git.openstack.org/openstack/horizon.git /opt/stack/horizon

   sudo git clone git://git.openstack.org/openstack/keystone.git /opt/stack/keystone

   sudo git clone git://git.openstack.org/openstack/neutron.git /opt/stack/neutron

   sudo git clone git://git.openstack.org/openstack/swift.git /opt/stack/swift

   sudo git clone git://git.openstack.org/openstack/heat.git /opt/stack/heat

   sudo git clone git://git.openstack.org/openstack/ceilometer.git /opt/stack/ceilometer

6. 进行安装
   ./stack.sh

# 镜像地址配置及ssh连接

1. Ubuntu常用国内源地址
   http://wiki.ubuntu.com.cn/%E6%BA%90%E5%88%97%E8%A1%A8
2. 安装ssh
   http://www.linuxidc.com/Linux/2016-12/137908.htm

# Linux命令所遇到的问题

1. 文件拒绝访问，添加权限
   Permission denied.

   Ø  ls –l /etc/sudoers 查看权限

   可以看到，/etc/sudoers为非目录，文件主有r(只读权限)，同组的用户有r权限，其他用户没有任何权限

   Ø  添加权限，这里会遇到一个操作不被允许的问题

   通过root用户进行权限修改
   为用户添加sudo权限（修改sudoers文件）
   http://www.cnblogs.com/youngerchina/p/5624473.html

   所遇问题：

   在sudoers文件中 加入
   stack ALL=(ALL) ALL
   切换到 stack用户，新建文件夹失败，感觉权限修改没生效，其实是在上一个用户的目录下，导致非root用户没有权限。
   注意在chmod 777 /etc/sudoers ，是所有用户对该文件都有权限，修改过后记得 恢复chmod 440 /etc/sudoers，否则会出现无法使用sudo命令。

# local.conf文件配置

[[local|localrc]]<br/>
HOST_IP=192.168.56.202<br/>
SERVICE_HOST=192.168.56.202<br/>
MYSQL_HOST=192.168.56.202<br/>
RABBIT_HOST=192.168.56.202<br/>
GLANCE_HOST=192.168.56.202<br/>
ADMIN_PASSWORD=password<br/>
DATABASE_PASSWORD=password<br/>
RABBIT_PASSWORD=password<br/>
SERVICE_PASSWORD=password<br/>
LOGFILE=/opt/stack/logs/stack.sh.log<br/>
VERBOSE=True<br/>
LOG_COLOR=True<br/>
SCREEN_LOGDIR=/opt/stack/logs<br/>
\# Use Neutron instead of nova-network<br/>
disable_service n-net<br/>
enable_service q-svc<br/>
enable_service q-dhcp<br/>
enable_service q-agt<br/>
enable_service q-l3<br/>
enable_service c-api<br/>
enable_service c-vol<br/>
enable_service c-sch<br/>
disable_service n-obj<br/>
disable_service c-bak<br/>
disable_service tempest<br/>
disable_service horizon<br/>
\#\# Neutron options<br/>
Q_USE_SECGROUP=False<br/>
NEUTRON_CREATE_INITIAL_NETWORKS=False<br/>
\# Open vSwitch provider networking configuration<br/>

Q_USE_PROVIDERNET_FOR_PUBLIC=True

OVS_PHYSICAL_BRIDGE=br-ex

PUBLIC_BRIDGE=br-ex

OVS_BRIDGE_MAPPINGS=extern:br-ex

PUBLIC_INTERFACE=enp0s8

Q_ML2_PLUGIN_VLAN_TYPE_OPTIONS=(network_vlan_ranges=bridge:2001:3000,extern:3001:4000)

enable_plugin neutron-lbaas https://github.com/openstack/neutron-lbaas.git

enable_plugin octavia https://github.com/openstack/octavia.git

ENABLED_SERVICES+=,q-lbaasv2

ENABLED_SERVICES+=,octavia,o-cw,o-hk,o-hm,o-api

注意点:
1.  (注意这里的bridge-mapping要和下面的vlan对应)
2. 在=后面千万不要有空格