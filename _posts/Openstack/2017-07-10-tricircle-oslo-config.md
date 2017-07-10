---
layout: post
#标题配置
title:  Openstack oslo.config note in Tricircle_jobpagination
#时间配置
date:   2017-07-10 10:45:00 +0800
#大类配置
categories: Tricircle
#小类配置
tag: Development notes
---

* content
{:toc}

# 1. Question
在job_pagination的开发过程中，需要使用 配置的max_pagination_limit（最大分页大小）这个配置参数，该配置参数的使用场景有两个（1）真正软件项目启动时候的注册，读取使用（2）在unit test，functional test
这种测试场景下，模拟进行参数的注册/读取使用

运行tox的时候，在本地正常，在网上出现registe duplication的异常

# 2. Config的使用方法和运行逻辑
[**参考**](http://www.cnblogs.com/Security-Darren/p/3854797.html)Openstack配置解析库的使用方法
## 2.1 基本概念和使用方法
**注意概念**

配置项的模式(option schemas)： 配置文件中是针对一个完整服务的里面有很多值，其他的模块不可能用到所有的值，需要指定自己用的那些值，这就是配置项的模式
```
# 声明配置项模式
# 单个配置项模式
enabled_apis_opt = cfg.ListOpt('enabled_apis',
                                   default=['ec2', 'osapi_compute'],
                                   help='List of APIs to enable by default.')
# 多个配置项组成一个模式
common_opts = [
        cfg.StrOpt('bind_host',
                   default='0.0.0.0',
                   help='IP address to listen on.'),
                
        cfg.IntOpt('bind_port',
                   default=9292,
                   help='Port number to listen on.')
    ]
# 配置组
rabbit_group = cfg.OptGroup(
    name='rabbit',
    title='RabbitMQ options'
)
# 配置组中的模式，通常以配置组的名称为前缀（非必须）
rabbit_ssl_opt = cfg.BoolOpt('use_ssl',
                             default=False,
                             help='use ssl for connection')
# 配置组中的多配置项模式
rabbit_Opts = [
    cfg.StrOpt('host',
                  default='localhost',
                  help='IP/hostname to listen on.'),
    cfg.IntOpt('port',
                 default=5672,
                 help='Port number to listen on.')
]
```
使用需要配置文件/声明配置项的模式/创建对象/适用对象的注册方法（没有解析配置文件）/调用对象传入配置文件路径
如果没有传入配置文件的则使用声明模式时候的默认值

## 2.2 Unit test异常解决
```buildoutcfg
class AsyncJobControllerTest(unittest.TestCase):
    def setUp(self):
        cfg.CONF.clear()
        cfg.CONF.register_opts(app.common_opts)
        core.initialize()
        core.ModelBase.metadata.create_all(core.get_engine())
        self.controller = FakeAsyncJobController()
        self.context = context.get_admin_context()
        self.job_resource_map = constants.job_resource_map
        policy.populate_default_rules()

```
在setup的开始行就对cfg.CONF进行清理和重新注册，如此下来py35正常

# 3. OSlo.config是如何设计的


#
虽一个小小的pagination，还是在开发中遇到很多问题，那些和paginate无关的东西耗费太多的时间，需要静下来好好补充，了解拓宽