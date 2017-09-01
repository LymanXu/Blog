---
layout: post
title:  Container(一)　What's Container
date:   2017-08-24 11:00:00 +0800
categories: Container
tag: 教程
---

* content
{:toc}

# 1. What's Container
Container: package software into standardized unit for development. 一个container image包括代码、系统工具、系统依赖库、
设置，构成一个软件运行的完整环境，像一个什么都装好的集装箱，因此可以猜到具有方便性、标准性、隔离性。

1. 轻量级
runing on a single machine share tht machine's operating system kernel.

2. 标准性
符合开元标准，可以运行在linux,windows,vms

3. 安全性
isolate applications from one another and from the underlying infrastructure(应用之间的隔离，应用和系统之间的隔离)

# 2. Container VS VM
Containers 是在同一台machine 并且共享OS kernel.是app layer上的一个抽象, each container作为隔离的进程在user space下。<br/>
VM 将single machine的硬件资源进行划分，是physical hardware的一个抽象，讲一个server分割成多个servers.每个vm包含一个OS的copy.

Containers and VM 这两者最大的相同有两点：
1. 都是提供一个运行应用的隔离环境environment
2. environment代表一个二进制的artifact,能在主机之间迁移

## 2.1 Container are not VMs
VM比作houses,Docker比作apartments. Houses有自己的基础设施，管道系统，供暖系统、电力系统，有基本的房屋布局卧室、客厅、卫生间、厨房。
就算是最简单的单身公寓的话，你也要为房屋外的好多付费。

而apartment同样实现了居住的功能，但是公寓共享一套基础设施系统，并且公寓提供不同类型大小的房间，你只需要租你需要的。

在VM下，抽象单元是庞大的vm,存储着application code和 statefull data, a vm 包括运行在硬件服务器上的所有内容；<br/>
而在container下：<br/>
1. 抽象单元是application（组成应用的服务），为服务架构中很多小的服务(Docker container)协作构成应用。应用也可以
deconstructed into components.新的应用部署方式（导致应用开发部署过程中根本的改变）
2. 应用数据不放在container中而是放在有多个container共享的 container volume.因此
不需要备份container,而是备份container volumne.
3. pathches不需要用在container下，通过更新docker image，停掉原来的container，重新启动新的(container轻量级启动速度很快)

## 2.2 Can Containers and VMs coexist
Containers和VM都是提供一个应用运行的隔离的环境，而且看来containers更加的轻量级地实现了此类功能，那么Containers能取代VM吗

Dockeer和VM可以共同存在，简单的来说<br/>
1. VM上可以很好的运行Docker,
2. Docker上的Service和VM上的service可以交流通信，
3. Mixing and matching Docker with VM（两者的混合使用）可以更好地优化硬件的利用率
4. running docker hosts on wide variety，包括VM、Microsoft Hyper-V,Azure,AWS,提供了不同组合的灵活性，保证敏捷性、可移植、可控制

这几点理由解释可以共存，不过没让我觉得有一起使用的必要。

Containers可以在VM和bare metal physical servers上都可以运行，有了不同就有了选择，怎么选不会。。。感觉不是技术问题

