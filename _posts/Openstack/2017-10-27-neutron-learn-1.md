---
layout: post
#标题配置
title:  Neutron learn(一)－－neutron的启动过程
#时间配置
date:   2017-10-27 16:53:00 +0800
#大类配置
categories: Neutron
#小类配置
tag: 
---

* content
{:toc}

# 1. Neutron整体启动流程



需要注意的是 1.1.2.1.2.1.1: load_paste_app(app_name), 会从api-paste.ini文件加载neutron的相关配置信息

# 1.1 load_paste_app()
通过api-paste.ini加载相关资源

api-paste.ini部分内容
```buildoutcfg
[composite:neutron]
use = egg:Paste#urlmap
/: neutronversions_composite
/v2.0: neutronapi_v2_0

[composite:neutronapi_v2_0]
use = call:neutron.auth:pipeline_factory
noauth = cors http_proxy_to_wsgi request_id catch_errors extensions neutronapiapp_v2_0
keystone = cors http_proxy_to_wsgi request_id catch_errors authtoken keystonecontext extensions neutronapiapp_v2_0

[app:neutronapiapp_v2_0]
paste.app_factory = neutron.api.v2.router:APIRouter.factory

```

可以看到对于composite:neutron, 无论是哪个分支，都需要从app=neutronapiapp_v2_0中加载资源

