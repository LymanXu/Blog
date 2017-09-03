---
layout: post
#标题配置
title:  Iptables/netfilter(一)
#时间配置
date:   2017-09-03 09:35:00 +0800
#大类配置
categories: Network
#小类配置
tag: Iptables
---

* content
{:toc}

# 1. What are netfilter and iptables
Netfilter 是一系列的 hooks位于Linux kernel，kernel可以注册回调函数结合network stack的关键位置.
A registered callback function is then called back for every packet that traverses(横跨) the respective（各自） hook within the network stack.

Iptables 是定义规则集的通用表结构。每条规则包括一些匹配的分类，和匹配后的去向。

Netfilter是一系列的位置和network stack结合的一系列的位置，Iptables是定义的一系列的规则，用于这些位置上。

# 2. Netfilter hooks
5个回调函数可以注册的hooks。当数据包经过network stack时会触发相应的hooks。

The following hooks represent various well-defined points in the networking stack:

NF_IP_PRE_ROUTING: This hook will be triggered by any incoming traffic very soon after entering the network stack. This hook is processed before any routing decisions have been made regarding where to send the packet.
NF_IP_LOCAL_IN: This hook is triggered after an incoming packet has been routed if the packet is destined for the local system.
NF_IP_FORWARD: This hook is triggered after an incoming packet has been routed if the packet is to be forwarded to another host.
NF_IP_LOCAL_OUT: This hook is triggered by any locally created outbound traffic as soon it hits the network stack.
NF_IP_POST_ROUTING: This hook is triggered by any outgoing or forwarded traffic after routing has taken place and just before being put out on the wire.

现在对hooks和表、链的概念依然有些模糊，通过数据在协议中的发送过程可以更好的了解.
数据在协议栈里的发送过程中，从上至下依次是“加头”的过程接受数据方就是个“剥头”的过程，最终到达用户那儿的就是裸数据了。
![协议栈的5个关键点]({{'/styles/images/Python/2017-09-03-iptables-netfilter-one-01.png' | prepend: site.baseurl }})

对于收到的每个数据包，都从“A”点进来，经过路由判决，如果是发送给本机的就经过“B”点，然后往协议栈的上层继续传递；否则，如果该数据包的目的地是不本机，那么就经过“C”点，然后顺着“E”点将该包转发出去。
对于发送的每个数据包，首先也有一个路由判决，以确定该包是从哪个接口出去，然后经过“D”点，最后也是顺着“E”点将该包发送出去。
协议栈那五个关键点A，B，C，D和E就是netfilter hooks作用的地方。
![For in 数组赋值]({{'/styles/images/Python/2017-09-03-iptables-netfilter-one-02.png' | prepend: site.baseurl }})

Netfilter 将这几个地方的hooks重新定义了名称。

**Hooks上的回调函数**<br/>
在这些关键点上，定义了好多具有优先级的回调函数，通过埋伏在这些关键点上的回调函数形成一条链。

# 3. Tables and Chains
表和链有什么联系，又是如何组成工作的呢<br/>
Iptables使用table组织路由规则，这些table按照处理功能进行分类。
For instance, if a rule deals with network address translation, it will be put into the nat table. If the rule is used to decide whether to allow the packet to continue to its destination, it would probably be added to the filter table.

Tables中定义的规则进一步和chains结合。Chains决定rules什么时候执行。Chains代表 netfilter hooks.

chains的名称和hooks的名称相对应：<br/>
1. PREROUTING: Triggered by the NF_IP_PRE_ROUTING hook.
2. INPUT: Triggered by the NF_IP_LOCAL_IN hook.
3. FORWARD: Triggered by the NF_IP_FORWARD hook.
4. OUTPUT: Triggered by the NF_IP_LOCAL_OUT hook.
5. POSTROUTING: Triggered by the NF_IP_POST_ROUTING hook.

一个table可以有多个chains，表中的规则可以在多个点发挥作用，因为
