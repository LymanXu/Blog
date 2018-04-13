---
layout: post
#标题配置
title:  Tricircle development[2]--Security group rule
#时间配置
date:   2017-09-21 09:58:00 +0800
#大类配置
categories: Openstack
#小类配置
tag: 开发笔记
---

* content
{:toc}

# 1. Question list
# 1.1 Local Neutron with central neutron

* 什么时候调local,什么时候调central
* 对security group的操作流程,exp create_sg
* RegionOne中的sg and sg rules are different with sg and sg rules in CentralRegion,为什么
* config sg的异步任务的处理逻辑是什么

# 1.2 RegionOne' sg and CentralRegion's sg
siglepod中创建了CentralRegion and RegionOne,　上发现数据库有两个neutron;neutron and neutron0（以及nova）
```buildoutcfg
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| cinder             |
| glance             |
| keystone           |
| mysql              |
| neutron            |
| neutron0           |
| nova_api           |
| nova_cell0         |
| nova_cell1         |
| performance_schema |
| sys                |
| tricircle          |
+--------------------+
```
# 1.2.1 neurtron0对应CentralRegion,neutron对应RegionOne
1. CentralRegion中的default sg,RegionOne中也有，而且具有相同的id和内容
2. CentralRegion中的rules，对于有remote_group_id的转化成remote_id_prefix,对于IPv6的不处理

通过下面的指令可以证明
```buildoutcfg
[stack@stack-node2:~/devstack]$neutron --os-region-name=CentralRegion security-group-list
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+--------------------------------------+---------+----------------------------------+----------------------------------------------------------------------+
| id                                   | name    | tenant_id                        | security_group_rules                                                 |
+--------------------------------------+---------+----------------------------------+----------------------------------------------------------------------+
| 17679411-f76c-4f7d-8831-af62c04ac901 | default | bb4349673696463bbc06218055ea37b3 | egress, IPv4                                                         |
|                                      |         |                                  | egress, IPv6                                                         |
|                                      |         |                                  | ingress, IPv4, remote_group_id: 17679411-f76c-4f7d-8831-af62c04ac901 |
|                                      |         |                                  | ingress, IPv6, remote_group_id: 17679411-f76c-4f7d-8831-af62c04ac901 |
+--------------------------------------+---------+----------------------------------+----------------------------------------------------------------------+
[stack@stack-node2:~/devstack]$neutron --os-region-name=RegionOne security-group-list
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+--------------------------------------+---------+----------------------------------+----------------------------------------------------------------------+
| id                                   | name    | tenant_id                        | security_group_rules                                                 |
+--------------------------------------+---------+----------------------------------+----------------------------------------------------------------------+
| 17679411-f76c-4f7d-8831-af62c04ac901 | default | bb4349673696463bbc06218055ea37b3 | egress, IPv4                                                         |
|                                      |         |                                  | egress, IPv6                                                         |
|                                      |         |                                  | ingress, IPv4, remote_ip_prefix: 10.0.0.0/24                         |
| ff60b1cf-8bf6-42dc-85bb-f3c0c22b97eb | default | bb4349673696463bbc06218055ea37b3 | egress, IPv4                                                         |
|                                      |         |                                  | egress, IPv6                                                         |
|                                      |         |                                  | ingress, IPv4, remote_group_id: ff60b1cf-8bf6-42dc-85bb-f3c0c22b97eb |
|                                      |         |                                  | ingress, IPv6, remote_group_id: ff60b1cf-8bf6-42dc-85bb-f3c0c22b97eb |
+--------------------------------------+---------+----------------------------------+----------------------------------------------------------------------+

mysql> use neutron;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A
mysql> select * from securitygroups;
+----------------------------------+--------------------------------------+---------+------------------+
| project_id                       | id                                   | name    | standard_attr_id |
+----------------------------------+--------------------------------------+---------+------------------+
| bb4349673696463bbc06218055ea37b3 | 17679411-f76c-4f7d-8831-af62c04ac901 | default |               15 |
| bb4349673696463bbc06218055ea37b3 | ff60b1cf-8bf6-42dc-85bb-f3c0c22b97eb | default |                6 |
+----------------------------------+--------------------------------------+---------+------------------+
2 rows in set (0.00 sec)

mysql> use neutron0;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> select * from securitygroups;
+----------------------------------+--------------------------------------+---------+------------------+
| project_id                       | id                                   | name    | standard_attr_id |
+----------------------------------+--------------------------------------+---------+------------------+
| bb4349673696463bbc06218055ea37b3 | 17679411-f76c-4f7d-8831-af62c04ac901 | default |                8 |
+----------------------------------+--------------------------------------+---------+------------------+

```
# 1.2.2 neutron数据库中有default_security_group
CentralRegion's security_group有一个对应neutron0中的default_security_group’s one row;
RegionOne‘s security_group has two 对应neutron0中的default_security_group’s one row 和自身neutron中的default_security_group’s one row，所以sg中有两个default_sg;

```buildoutcfg
mysql> use neutron0;
Database changed

mysql> select * from default_security_group;
+----------------------------------+--------------------------------------+
| project_id                       | security_group_id                    |
+----------------------------------+--------------------------------------+
| bb4349673696463bbc06218055ea37b3 | 17679411-f76c-4f7d-8831-af62c04ac901 |
+----------------------------------+--------------------------------------+

mysql> use neutron;
mysql> select * from default_security_group;
+----------------------------------+--------------------------------------+
| project_id                       | security_group_id                    |
+----------------------------------+--------------------------------------+
| bb4349673696463bbc06218055ea37b3 | ff60b1cf-8bf6-42dc-85bb-f3c0c22b97eb |
+----------------------------------+--------------------------------------+


```
# 1.2.3数据库tricircle中维护的security group top-bottom的对应信息
```buildoutcfg
mysql> use tricircle
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+---------------------+
| Tables_in_tricircle |
+---------------------+
| async_job_logs      |
| async_jobs          |
| cached_endpoints    |
| migrate_version     |
| pods                |
| recycle_resources   |
| resource_routings   |
| shadow_agents       |
+---------------------+
8 rows in set (0.00 sec)

mysql> select * from resource_routings where resource_type='security_group';
+----+--------------------------------------+--------------------------------------+--------------------------------------+----------------------------------+----------------+---------------------+------------+
| id | top_id                               | bottom_id                            | pod_id                               | project_id                       | resource_type  | created_at          | updated_at |
+----+--------------------------------------+--------------------------------------+--------------------------------------+----------------------------------+----------------+---------------------+------------+
|  7 | 17679411-f76c-4f7d-8831-af62c04ac901 | 17679411-f76c-4f7d-8831-af62c04ac901 | 0e5c9ad4-f103-4184-9d88-c2dd38de33e6 | bb4349673696463bbc06218055ea37b3 | security_group | 2017-10-18 07:12:59 | NULL       |
+----+--------------------------------------+--------------------------------------+--------------------------------------+----------------------------------+----------------+---------------------+------------+
1 row in set (0.00 sec)

mysql> select * from pods;
+--------------------------------------+---------------+-------------+---------+---------+
| pod_id                               | region_name   | pod_az_name | dc_name | az_name |
+--------------------------------------+---------------+-------------+---------+---------+
| 0e5c9ad4-f103-4184-9d88-c2dd38de33e6 | RegionOne     |             |         | az1     |
| 76b2054b-5679-4cdd-a555-6934051275b0 | CentralRegion |             |         |         |
+--------------------------------------+---------------+-------------+---------+---------+

```

# 1.3 对security group的操作流程
区别：<br/>
1. openstack --os-region-name=CentralRegion security group rule create (group为default)报错无法添加，证明了tricircle　security group中的代码有效；
openstack --os-region-name=RegionOne security group rule create (group为default)添加成功，证明在neutron中已经实现了支持对default sg的create rules
2. 无论是对CentralRegion,RegionOne中新增安全组，在sg rules均自动增加了两条rule(对应新增的sg)????

3. 在对centralRegion　add security group rule, the bottom pods' security group rules don't added.　So we should find what resource will be added into tricircle_resource_mapping.


按照代码的逻辑，在centralRegion add a rule to the default sg, a new rule will be added to the sg of Region. But 事实上并没有添加该规则，所以我需要在代码中加入Debug信息进行确认。

# 2. Tricircle中如何实现异步任务


# 3. Security group的测试
# 3.1 流程
1. test_central_plugin

对Plugin, NeutronContext, dbContext进行　fake; 传入被调函数的入参为ResourceStroe中的属性

* ResourceStroe其实实现一个关于top,pod1,pod2模拟场景的某些资源信息的存储。store_map{tablename:[];tablename2:[]}对于对top的信息的存储，
pod_store_map{'top':{资源名：[]},'pod_1': {}, 'pod_2': {}}存储每个pod的对应资源的信息（资源集合小于table集合）；还有一点是
```buildoutcfg
__init():
.....
 store_name = '%s_%s' % (prefix, table.upper())
                setattr(self, store_name, [])
```
将每个pod_prefix:每个table_name作为一个类型为[]的属性，添加到ResourceStroe，exem: TOP_NETWORK, BOTTOM1_SECURITYGROUP.....




