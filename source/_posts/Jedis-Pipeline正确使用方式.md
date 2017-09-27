---
title: Jedis Pipeline正确使用方式
date: 2017-09-27 11:35:10
tags: [Redis,Jedis,Pipeline]
categories: Redis

---
笔者在Jedis使用过程中脑子中风，发现Python的Redis客户端和Jedis的流水线不太相同，下面来记一下Jedis Pipeline的正确使用方式。

<!--more-->

# Pipeline的概念
在我们直接用Jedis对象的各个方法的时候，指令是直接发送到服务端并且返回的，如果指令数量过多的话会造成客户端和服务器之间的大量通信，Pipeline的设计就是为了减少通信次数：首先将多个命令存到本地Pipeline对象当中，当执行**sync**方法时，所有的命令会一并发送到服务端，可以选择事务型流水线(所有的指令一块执行，不会被打断)和非事务型流水线(所有的指令执行过程中可能会被打断)。
# 使用
使用较为简单，下面来看三个示例：

示例一：

    private void test1() {
        Pipeline pipeline = jedis.pipelined();
        pipeline.set("test", "test");
        Response<String> response = pipeline.get("test");
        pipeline.sync();
        // sync之后get()方法才能够被调用
        System.out.println(response.get());
    }
示例二：

    private void test2() {
        Pipeline pipeline = jedis.pipelined();
        pipeline.set("test", "test");
        pipeline.get("test");
        // 返回所有的结果
        System.out.println(pipeline.syncAndReturnAll());
    }
示例三：

    private void test3() {
        Pipeline pipeline = jedis.pipelined();
        // 开启一段事务
        pipeline.multi();
        pipeline.set("test", "test");
        pipeline.get("test");
        pipeline.exec();
        // 返回所有的结果
        System.out.println(pipeline.syncAndReturnAll());
    }

# Pipeline事务和Jedis直接使用事务的区别
当直接调用Jedis对象的时候，jedis.multi()和jedis.exec()指令和它们两个之间的指令是直接发送到服务端并且保存到服务端的队列之中的(此时会有返回值Queued)，而用Pipeline对象的事务方法，所有的指令先在本地保存，最后sync方法才一并发送到远程服务端。这样Pipeline的实现方式看起来更好。
