---
layout: post
#标题配置
title:  Tricircle development(二)－－security group
#时间配置
date:   2017-09-21 09:58:00 +0800
#大类配置
categories: Tricircle
#小类配置
tag: Development notes
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
一个安装了siglepod的vm上发现数据库有两个neutron;neutron and neutron0
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
数据库neutron and neutron0的数据结构是相同的，应该是对应两个pod.Right????
```buildoutcfg
mysql> use neutron;

Database changed
mysql> show tables;
+-----------------------------------------+
| Tables_in_neutron                       |
+-----------------------------------------+
| address_scopes                          |
| agents                                  |
| alembic_version                         |
| allowedaddresspairs                     |
| arista_provisioned_nets                 |
| arista_provisioned_tenants              |
| arista_provisioned_vms                  |
| auto_allocated_topologies               |
| bgp_peers                               |
| bgp_speaker_dragent_bindings            |
| bgp_speaker_network_bindings            |
| bgp_speaker_peer_bindings               |
| bgp_speakers                            |
| brocadenetworks                         |
| brocadeports                            |
| cisco_csr_identifier_map                |
| cisco_hosting_devices                   |
| cisco_ml2_apic_contracts                |
| cisco_ml2_apic_host_links               |
| cisco_ml2_apic_names                    |
| cisco_ml2_n1kv_network_bindings         |
| cisco_ml2_n1kv_network_profiles         |
| cisco_ml2_n1kv_policy_profiles          |
| cisco_ml2_n1kv_port_bindings            |
| cisco_ml2_n1kv_profile_bindings         |
| cisco_ml2_n1kv_vlan_allocations         |
| cisco_ml2_n1kv_vxlan_allocations        |
| cisco_ml2_nexus_nve                     |
| cisco_ml2_nexusport_bindings            |
| cisco_port_mappings                     |
| cisco_router_mappings                   |
| consistencyhashes                       |
| default_security_group                  |
| dnsnameservers                          |
| dvr_host_macs                           |
| externalnetworks                        |
| extradhcpopts                           |
| firewall_policies                       |
| firewall_rules                          |
| firewalls                               |
| flavors                                 |
| flavorserviceprofilebindings            |
| floatingipdnses                         |
| floatingips                             |
| ha_router_agent_port_bindings           |
| ha_router_networks                      |
| ha_router_vrid_allocations              |
| healthmonitors                          |
| ikepolicies                             |
| ipallocationpools                       |
| ipallocations                           |
| ipamallocationpools                     |
| ipamallocations                         |
| ipamsubnets                             |
| ipsec_site_connections                  |
| ipsecpeercidrs                          |
| ipsecpolicies                           |
| logs                                    |
| lsn                                     |
| lsn_port                                |
| maclearningstates                       |
| members                                 |
| meteringlabelrules                      |
| meteringlabels                          |
| ml2_brocadenetworks                     |
| ml2_brocadeports                        |
| ml2_distributed_port_bindings           |
| ml2_flat_allocations                    |
| ml2_geneve_allocations                  |
| ml2_geneve_endpoints                    |
| ml2_gre_allocations                     |
| ml2_gre_endpoints                       |
| ml2_nexus_vxlan_allocations             |
| ml2_nexus_vxlan_mcast_groups            |
| ml2_port_binding_levels                 |
| ml2_port_bindings                       |
| ml2_ucsm_port_profiles                  |
| ml2_vlan_allocations                    |
| ml2_vxlan_allocations                   |
| ml2_vxlan_endpoints                     |
| multi_provider_networks                 |
| networkconnections                      |
| networkdhcpagentbindings                |
| networkdnsdomains                       |
| networkgatewaydevicereferences          |
| networkgatewaydevices                   |
| networkgateways                         |
| networkqueuemappings                    |
| networkrbacs                            |
| networks                                |
| networksecuritybindings                 |
| networksegments                         |
| neutron_nsx_network_mappings            |
| neutron_nsx_port_mappings               |
| neutron_nsx_router_mappings             |
| neutron_nsx_security_group_mappings     |
| nexthops                                |
| nsxv_edge_dhcp_static_bindings          |
| nsxv_edge_vnic_bindings                 |
| nsxv_firewall_rule_bindings             |
| nsxv_internal_edges                     |
| nsxv_internal_networks                  |
| nsxv_port_index_mappings                |
| nsxv_port_vnic_mappings                 |
| nsxv_router_bindings                    |
| nsxv_router_ext_attributes              |
| nsxv_rule_mappings                      |
| nsxv_security_group_section_mappings    |
| nsxv_spoofguard_policy_network_mappings |
| nsxv_tz_network_bindings                |
| nsxv_vdr_dhcp_bindings                  |
| nuage_net_partition_router_mapping      |
| nuage_net_partitions                    |
| nuage_provider_net_bindings             |
| nuage_subnet_l2dom_mapping              |
| poolloadbalanceragentbindings           |
| poolmonitorassociations                 |
| pools                                   |
| poolstatisticss                         |
| portbindingports                        |
| portdataplanestatuses                   |
| portdnses                               |
| portqueuemappings                       |
| ports                                   |
| portsecuritybindings                    |
| providerresourceassociations            |
| provisioningblocks                      |
| qos_bandwidth_limit_rules               |
| qos_dscp_marking_rules                  |
| qos_minimum_bandwidth_rules             |
| qos_network_policy_bindings             |
| qos_policies                            |
| qos_policies_default                    |
| qos_port_policy_bindings                |
| qospolicyrbacs                          |
| qosqueues                               |
| quotas                                  |
| quotausages                             |
| reservations                            |
| resourcedeltas                          |
| router_extra_attributes                 |
| routerl3agentbindings                   |
| routerports                             |
| routerroutes                            |
| routerrules                             |
| routers                                 |
| securitygroupportbindings               |
| securitygrouprules                      |
| securitygroups                          |
| segmenthostmappings                     |
| serviceprofiles                         |
| sessionpersistences                     |
| standardattributes                      |
| subnet_service_types                    |
| subnetpoolprefixes                      |
| subnetpools                             |
| subnetroutes                            |
| subnets                                 |
| subports                                |
| tags                                    |
| trunks                                  |
| tz_network_bindings                     |
| vcns_router_bindings                    |
| vips                                    |
| vpnservices                             |
+-----------------------------------------+
165 rows in set (0.00 sec)

```

查询数据库tricircle中维护的security group的信息
```buildoutcfg
mysql> use tricircle;
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

mysql> select * from resource_routings where resource_type='security_group';
+----+--------------------------------------+--------------------------------------+--------------------------------------+----------------------------------+----------------+---------------------+------------+
| id | top_id                               | bottom_id                            | pod_id                               | project_id                       | resource_type  | created_at          | updated_at |
+----+--------------------------------------+--------------------------------------+--------------------------------------+----------------------------------+----------------+---------------------+------------+
|  7 | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb | 87ae2008-d3fa-4800-bfab-86b69dc44241 | 3787a7ad3f9f45c599647b190e4bcbc9 | security_group | 2017-09-20 05:27:01 | NULL       |
+----+--------------------------------------+--------------------------------------+--------------------------------------+----------------------------------+----------------+---------------------+------------+
发现resource_routings中只有一个安全组，bottom__id和　pod_id接下来有用

mysql> select * from pods;
+--------------------------------------+---------------+-------------+---------+---------+
| pod_id                               | region_name   | pod_az_name | dc_name | az_name |
+--------------------------------------+---------------+-------------+---------+---------+
| 87ae2008-d3fa-4800-bfab-86b69dc44241 | RegionOne     |             |         | az1     |
| e6edcd4b-7085-4c4f-b4c4-7d61eb060fcf | CentralRegion |             |         |         |
+--------------------------------------+---------------+-------------+---------+---------+

```

查询数据库neutron中关于security groups的信息
```buildoutcfg
mysql> use neutron;

mysql> select * from securitygroups;
+----------------------------------+--------------------------------------+---------+------------------+
| project_id                       | id                                   | name    | standard_attr_id |
+----------------------------------+--------------------------------------+---------+------------------+
| 3787a7ad3f9f45c599647b190e4bcbc9 | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb | default |               15 |
| 3787a7ad3f9f45c599647b190e4bcbc9 | 8b09c296-ff1a-492b-a385-561c1b4e4b64 | default |                6 |
+----------------------------------+--------------------------------------+---------+------------------+

mysql> select * from default_security_group;
+----------------------------------+--------------------------------------+
| project_id                       | security_group_id                    |
+----------------------------------+--------------------------------------+
| 3787a7ad3f9f45c599647b190e4bcbc9 | 8b09c296-ff1a-492b-a385-561c1b4e4b64 |
+----------------------------------+--------------------------------------+

mysql> select * from securitygrouprules;
+----------------------------------+--------------------------------------+--------------------------------------+-----------------+-----------+-----------+----------+----------------+----------------+------------------+------------------+
| project_id                       | id                                   | security_group_id                    | remote_group_id | direction | ethertype | protocol | port_range_min | port_range_max | remote_ip_prefix | standard_attr_id |
+----------------------------------+--------------------------------------+--------------------------------------+-----------------+-----------+-----------+----------+----------------+----------------+------------------+------------------+
| 3787a7ad3f9f45c599647b190e4bcbc9 | 7ad4e4a1-1f9c-4367-abaf-354f983f5c4c | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb | NULL            | egress    | IPv4      | NULL     |           NULL |           NULL | NULL             |               16 |
| 3787a7ad3f9f45c599647b190e4bcbc9 | 7fe6d23d-3ca0-4de4-8465-1ab692b992d3 | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb | NULL            | ingress   | IPv4      | icmp     |           NULL |           NULL | 0.0.0.0/0        |               20 |
| 3787a7ad3f9f45c599647b190e4bcbc9 | 8382d48d-beb8-41e3-8231-14fe308ab23c | 8b09c296-ff1a-492b-a385-561c1b4e4b64 | NULL            | egress    | IPv4      | NULL     |           NULL |           NULL | NULL             |                8 |
| 3787a7ad3f9f45c599647b190e4bcbc9 | c5f12f46-2503-4526-9662-32890a473ed7 | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb | NULL            | egress    | IPv6      | NULL     |           NULL |           NULL | NULL             |               17 |
| 3787a7ad3f9f45c599647b190e4bcbc9 | ceb6d629-47d0-4388-9f2d-bba3ccc6b9fe | 8b09c296-ff1a-492b-a385-561c1b4e4b64 | NULL            | egress    | IPv6      | NULL     |           NULL |           NULL | NULL             |               10 |
| 7480c987587f418c81f328316eb6481b | ffe6dbbf-8f47-461d-a5cf-2196e36634d9 | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb | NULL            | ingress   | IPv4      | NULL     |           NULL |           NULL | 10.0.0.0/24      |               19 |
+----------------------------------+--------------------------------------+--------------------------------------+-----------------+-----------+-----------+----------+----------------+----------------+------------------+------------------+

```

查询数据库neutron0中关于security group的信息(对应CentralRegion)
```buildoutcfg
mysql> use neutron0;

mysql> select * from securitygroups;
+----------------------------------+--------------------------------------+---------+------------------+
| project_id                       | id                                   | name    | standard_attr_id |
+----------------------------------+--------------------------------------+---------+------------------+
| 3787a7ad3f9f45c599647b190e4bcbc9 | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb | default |                8 |
+----------------------------------+--------------------------------------+---------+------------------+

mysql> select * from default_security_group;
+----------------------------------+--------------------------------------+
| project_id                       | security_group_id                    |
+----------------------------------+--------------------------------------+
| 3787a7ad3f9f45c599647b190e4bcbc9 | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb |
+----------------------------------+--------------------------------------+

mysql> select * from securitygrouprules;
+----------------------------------+--------------------------------------+--------------------------------------+--------------------------------------+-----------+-----------+----------+----------------+----------------+------------------+------------------+
| project_id                       | id                                   | security_group_id                    | remote_group_id                      | direction | ethertype | protocol | port_range_min | port_range_max | remote_ip_prefix | standard_attr_id |
+----------------------------------+--------------------------------------+--------------------------------------+--------------------------------------+-----------+-----------+----------+----------------+----------------+------------------+------------------+
| 3787a7ad3f9f45c599647b190e4bcbc9 | 791cc717-3a7d-4148-9c2c-b20b8bbd7681 | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb | NULL                                 | egress    | IPv4      | NULL     |           NULL |           NULL | NULL             |               10 |
| 3787a7ad3f9f45c599647b190e4bcbc9 | 7e5fbc92-708c-433e-99ce-95e4a2326b5f | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb | ingress   | IPv6      | NULL     |           NULL |           NULL | NULL             |               11 |
| 3787a7ad3f9f45c599647b190e4bcbc9 | 8b1a9484-9542-49ff-aa47-0cb23130306d | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb | ingress   | IPv4      | NULL     |           NULL |           NULL | NULL             |                9 |
| 3787a7ad3f9f45c599647b190e4bcbc9 | 9e6ef456-ee80-45df-bf68-fc25219a67a3 | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb | NULL                                 | ingress   | IPv4      | icmp     |           NULL |           NULL | 0.0.0.0/0        |               13 |
| 3787a7ad3f9f45c599647b190e4bcbc9 | bbfba375-0bd2-4945-a0f0-6e0d76b31a9b | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb | NULL                                 | egress    | IPv6      | NULL     |           NULL |           NULL | NULL             |               12 |
+----------------------------------+--------------------------------------+--------------------------------------+--------------------------------------+-----------+-----------+----------+----------------+----------------+------------------+------------------+

```
在数据库tricircle-resource_routings表中的security group记录，在neutron,neutron0中的security group都存在，也就是tricircle中维持的各个pod中都有；neurtron0对应CentralRegion,neutron对应RegionOne

通过openstack security group查询不同region's security group
```buildoutcfg
stack@stack:~/devstack$ openstack --os-region-name=CentralRegion security group list
+--------------------------------------+---------+------------------------+----------------------------------+
| ID                                   | Name    | Description            | Project                          |
+--------------------------------------+---------+------------------------+----------------------------------+
| 383b75ab-5b6a-4985-b3cf-41fa70fec3cb | default | Default security group | 3787a7ad3f9f45c599647b190e4bcbc9 |
+--------------------------------------+---------+------------------------+----------------------------------+
stack@stack:~/devstack$ openstack security group list
+--------------------------------------+---------+------------------------+----------------------------------+
| ID                                   | Name    | Description            | Project                          |
+--------------------------------------+---------+------------------------+----------------------------------+
| 383b75ab-5b6a-4985-b3cf-41fa70fec3cb | default | Default security group | 3787a7ad3f9f45c599647b190e4bcbc9 |
| 8b09c296-ff1a-492b-a385-561c1b4e4b64 | default | Default security group | 3787a7ad3f9f45c599647b190e4bcbc9 |
+--------------------------------------+---------+------------------------+----------------------------------+

stack@stack:~/devstack$ openstack security group rule list
+--------------------------------------+-------------+-------------+------------+-----------------------+--------------------------------------+
| ID                                   | IP Protocol | IP Range    | Port Range | Remote Security Group | Security Group                       |
+--------------------------------------+-------------+-------------+------------+-----------------------+--------------------------------------+
| 7ad4e4a1-1f9c-4367-abaf-354f983f5c4c | None        | None        |            | None                  | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb |
| 7fe6d23d-3ca0-4de4-8465-1ab692b992d3 | icmp        | 0.0.0.0/0   |            | None                  | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb |
| 8382d48d-beb8-41e3-8231-14fe308ab23c | None        | None        |            | None                  | 8b09c296-ff1a-492b-a385-561c1b4e4b64 |
| c5f12f46-2503-4526-9662-32890a473ed7 | None        | None        |            | None                  | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb |
| ceb6d629-47d0-4388-9f2d-bba3ccc6b9fe | None        | None        |            | None                  | 8b09c296-ff1a-492b-a385-561c1b4e4b64 |
| ffe6dbbf-8f47-461d-a5cf-2196e36634d9 | None        | 10.0.0.0/24 |            | None                  | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb |
+--------------------------------------+-------------+-------------+------------+-----------------------+--------------------------------------+
stack@stack:~/devstack$ openstack --os-region-name=CentralRegion security group rule list
+--------------------------------------+-------------+-----------+------------+--------------------------------------+--------------------------------------+
| ID                                   | IP Protocol | IP Range  | Port Range | Remote Security Group                | Security Group                       |
+--------------------------------------+-------------+-----------+------------+--------------------------------------+--------------------------------------+
| 791cc717-3a7d-4148-9c2c-b20b8bbd7681 | None        | None      |            | None                                 | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb |
| 7e5fbc92-708c-433e-99ce-95e4a2326b5f | None        | None      |            | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb |
| 8b1a9484-9542-49ff-aa47-0cb23130306d | None        | None      |            | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb |
| 9e6ef456-ee80-45df-bf68-fc25219a67a3 | icmp        | 0.0.0.0/0 |            | None                                 | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb |
| bbfba375-0bd2-4945-a0f0-6e0d76b31a9b | None        | None      |            | None                                 | 383b75ab-5b6a-4985-b3cf-41fa70fec3cb |
+--------------------------------------+-------------+-----------+------------+--------------------------------------+--------------------------------------+


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




