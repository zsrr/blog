---
title: ActiveMQ集群及负载均衡
date: 2017-09-12 20:41:53
tags: [分布式，消息中间件]
categories: 分布式

---
本篇记录Apache ActiveMQ的负载均衡及集群配置，参考书籍《ActiveMQ In Action》以及大牛[KimmKing的技术博客](http://blog.csdn.net/kimmking/article/details/8440150)。

<!--more-->

# Master/Slave
ActiveMQ Master/Slave有三种实现方式：

- 基于共享文件实现。
- 基于共享数据实现。
- Pure Master/Slave 问题较多已被废弃。

如图所示，对于基于共享数据库的实现，Master启动时，会获得排它锁，直至Master宕机，某个Slave获得此排它锁成为新的Master，所以此实现方案诣在解决单点故障问题。

![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-09-14%20%E4%B8%8A%E5%8D%8810.42.04.png)

客户端要配合**failover**协议使用才能正确使用Master/Slave，关于此协议查看官方文档：[http://activemq.apache.org/failover-transport-reference.html](http://activemq.apache.org/failover-transport-reference.html)

客户端所使用的URL如下面的格式所示：

    failover:(tcp://master:61616,tcp://slave1:61616,tcp://slave2:61616)?key=value

# 负载均衡
实现负载均衡的主要实现是通过ActiveMQ提供的network connector来实现，存在动态配置及静态配置两种，参见：

静态配置：[http://activemq.apache.org/static-transport-reference.html](http://activemq.apache.org/static-transport-reference.html)

动态配置：[http://activemq.apache.org/multicast-transport-reference.html](http://activemq.apache.org/multicast-transport-reference.html)
