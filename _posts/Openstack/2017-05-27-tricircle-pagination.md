---
layout: post
#标题配置
title:  Tricircle pagination 分页问题
#时间配置
date:   2017-05-27 10:49:00 +0800
#大类配置
categories: Tricircle
#小类配置
tag: 技术文档
---

* content
{:toc}

# 任务
1. implement routing pagination
2. implemnet async job cli in python-tricircleclient

# 参考资料
read neutron pagination source code

refer to pod routing cli implemention in python-tricircleclient<br/>
[github-tricircleclient](https://github.com/openstack/python-tricircleclient)<br/>
some patches to implement pod routing cli<br/>
[review patches](https://review.openstack.org/#/q/project:openstack/python-tricircleclient)<br/>
developer guide pagination的参数<br/>
[pagination API 参数](https://developer.openstack.org/api-ref/networking/v2/#pagination)<br/>

# Neutron中的分页实现
## oslo.db底层paginate_query源码 
github地址[link](https://github.com/openstack/oslo.db/commit/dea700d13e4c152bf1a484b62cc2bce329ef6fa9#diff-7d84830d3af287b534a2f8a6d74e2de5R151)<br/>








