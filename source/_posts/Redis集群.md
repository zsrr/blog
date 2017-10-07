---
title: Redis集群解决方案
date: 2017-09-27 16:01:52
tags: [Redis,分布式]
categories: Redis

---
本篇主要介绍Redis集群解决方案，主要介绍客户端集群技术和服务端集群技术。

<!--more-->

# 客户端实现
Redis绝大多数客户端实现了Shard的功能，采用[一致性哈希算法](http://blog.csdn.net/cywosp/article/details/23397179)，会根据key来决定所要访问的节点，缺点就是客户端需要知道服务器的状态，若是一台服务器宕机(没有实现HA的前提下)则会丢失数据，拿Jedis举例，实现此种方案的是采用[ShardedJedis](www.cnblogs.com/vhua/p/redis_1.html)。

# 服务端实现
自Redis 3.0开始Redis实现了自己的集群方案——Redis Cluster。此方案允许客户端以正常方式访问其中集群中的任意一个节点，关于其概述参见：[Redis Cluster Tutorial](https://redis.io/topics/cluster-tutorial)

其缺点为Pipeline和事务得不到良好的实现(满足CAP理论)，Pipeline实现可以见这篇：[Redis Cluster批量操作实现](http://trumandu.github.io/2016/06/05/Redis-Cluster-%E6%89%B9%E9%87%8F%E6%93%8D%E4%BD%9C%E5%AE%9E%E7%8E%B0/)，对于事务来讲笔者有一个方法，接着前面提到过的[Redis实现分布式锁](http://www.stephenzhang.me/2017/09/25/Redis%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81/)，可以用这种悲观锁的方式强行进行事务操作。