---
layout: post
#标题配置
title:  Tricircle development[3]--安全组资源删除
#时间配置
date:   2018-01-29 16:40:00 +0800
#大类配置
categories: Openstack
#小类配置
tag: 开发笔记
---

* content
{:toc}

# 1. Background
## Security group的资源关系映射
在tricircle中，如果用户在CentralRegion新建了安全组，当用户在RegionOne中boot vm过程中，当RegionOne中的Nova send a request to create_port时创建的过程会先到CentralRegion进行中创建（这时使用用户定义的安全组），然后再在RegionOne中创建对应的port和绑定的安全组。RegionOne中的Nova调用update_port时更新绑定信息时，会先到CentralRegion中更新bind这时就会在resource_routing中添加top-bottom security group的一条信息，然后再在RegionOne中更新绑定[1]。于是对security group资源进行删除时，也先要删除bottom中的资源然后再删除top中的资源。

# 2. Current situation and problems
## 2.1 Get resource导致bottom资源重建
** 删除过程中调用Get resource会导致bottom重新创建资源 **
上面也提到了用户在CentralRegion定义的安全组，当用户在RegionOne中起虚拟机时，绑定的是CentralRegion中定义的安全组，这是由于将CentralRegion中的安全组资源在RegionOne中进行了重建，那么此时就需要维持一个Top-Bottom资源映射的管理，另外在RegionOne中重建的逻辑是在local查询没有的话到top查询查到就重建。那么再删除的过程中如果在local中有该种资源的查询操作那么就会在local中重建，导致删除失败，因此添加了resource_deleting表标志资源正在删除。

## 2.2 Top-bottom id不一致（bug）
** 对于Security group Top-bottom对应的映射在resource_routing中id不同（遗留bug） **
![Security]({{'/styles/images/OpenStack/2018-01-29-1.jpg' | prepend: site.baseurl }} "安全组信息")
上图说明了相关联的安全组在CentralRegion和RegionOne中的信息

![Security]({{'/styles/images/OpenStack/2018-01-29-2.jpg' | prepend: site.baseurl }}  "资源映射")
而在resource_routing表这条security group映射中的bottom_id并不是真实的，所以要检查security group资源的创建同步，以及resource_routing信息的维持。

## 2.3 原有security group删除的逻辑测试结果
操作：成功起vm后，在CentralRegion新建Test安全组后，将vm的port重新绑定到Test安全组，
Result：在RegionOne中成功同步新的Test安全组，resource_routings中有相应的记录

操作：将Test安全组删除（此时vm’ port正绑定）
    select * from  securitygroupportbindings;

Result：CentralRegion中成功删除，而此时的vm’s port依然还在绑定，RegionOne中安全组依然存在，resource_routings中的记录依然存在
1.	资源Top-bottom的同步删除没有实现
2.	安全组删除前没有进行port绑定检查（或失败），因为securitygroupportbindings中没有对port绑定的安全组进行更新，导致安全组删除前的port bingdings检查无效。

![Security]({{'/styles/images/OpenStack/2018-01-29-3.jpg' | prepend: site.baseurl }} "port绑定的security group")
![Security]({{'/styles/images/OpenStack/2018-01-29-4.jpg' | prepend: site.baseurl }} "port绑定的security group")

# 3. Solution
## 3.1 线上环境测试
基本起vm:
参考[2]tricircle install
```

新建用户定义的安全组：
neutron --os-region-name=CentralRegion security-group-create SGTest39 --description "0309-1 for security group deleting test"
neutron --os-region-name=CentralRegion security-group-list
```

查看资源映射信息：
登入MySQL，查看tricircle数据库中的resource_routing
`select * from resource_routings;`

更新虚拟机的端口绑定的安全组：
```
nova list
neutron --os-region-name=CentralRegion port-list
neutron --os-region-name=CentralRegion port-update --security-group a9ffc5a4-62a4-4b79-9e50-d7bbfece06e4 39e165c9-67a5-4d5a-bb79-34a9bab56b4c
(neutron --os-region-name=CentralRegion port-update --security-group security_group_id  port_id)
```

检查RegionOne中是否生成对应的security group资源，以及资源映射关系：
`neutron --os-region-name=RegionOne security-group-list`

测试出Bug: 资源重创建成功，但是resource_routing没有相应的记录。
`neutron --os-region-name=CentralRegion security-group-delete xxx`

查看日志信息：
`sudo journalctl -f --unit devstack@q-svc0.service`

使用Pdb进行在线调试：
```
systemctl show devstack@q-svc0.service -p ExecStart --no-pager
python -m pdb /usr/local/bin/neutron-server --config-file /etc/neutron/neutron.conf.0 --config-file /etc/neutron/plugins/ml2/ml2_conf.ini
打断点：
b tricircle/network/central_plugin.py:836
b tricircle/network/central_plugin.py:843
b tricircle/network/central_plugin.py:906

b tricircle/network/security_groups.py:101
```

发现问题
1.	当security group delete时，向deleting_resources中插入一条记录，当由于资源正在使用，无法删除报异常时，刚才插入的deleting_resources中的记录并没有删除掉，这里应该使用事务

# 4. Harvest
## 4.1 分布式通信
1. RPC说白就是提供参数调用栈的封装+底层使用TCP或HTTP传递在服务端解封调用，并进行结果封装传递，客户端再解封的过程。
     RPC框架实现[3] http://javatar.iteye.com/blog/1123915
2. Tricircle中远程进程通信
    Tricircle中当CentralRegion neutron调用BottomRegion neutron或其他服务的时候，是直接使用的HTTP消息。
    Tricircle/common/client.py:Client包装所有的OpenStack service clients，调用client.xx会转到resource_handle中处理，会生成neutronclient/neutron/client.py,然后在neutronclient中对方法进行HTTPClient request。对于bottom region neutron服务端对这些REST api的处理还没看。

# 5. References
[1] https://wiki.openstack.org/wiki/TricircleHowToReadCode
[2] https://docs.openstack.org/tricircle/pike/install/installation-guide.html#single-pod-installation-with-devstack
[3] http://javatar.iteye.com/blog/1123915
