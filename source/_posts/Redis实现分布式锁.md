---
title: Redis实现分布式锁
date: 2017-09-25 20:00:12
tags: [Redis,分布式]
categories: Redis

---
本篇记录一下如何用Redis实现分布式锁，参考书籍[《Redis实战》](https://item.jd.com/11791607.html)，采用Jedis实现。


# 分布式锁的概念
分布式锁是用来协调分布式系统当中各节点对于资源的获取、更新、删除。比较常用的是Redlock和Zookeeper(前者遭到了质疑)，以及可以直接将此任务托付给底层RDBMS，具体参见博客[聊一聊分布式锁的设计](http://www.weizijun.cn/2016/03/17/%E8%81%8A%E4%B8%80%E8%81%8A%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%E7%9A%84%E8%AE%BE%E8%AE%A1/)。
# Redis实现
下面来探讨Redis如何实现分布式锁。
## Redis自带的事务处理
Redis事务的核心是MULTI, EXEC, WATCH, DISCARD, UNWATCH, 通过乐观锁实现CAS，用法比较简单，参见官方文档:[事务](http://redisdoc.com/topic/transaction.html), [UNWATCH命令](http://redisdoc.com/transaction/unwatch.html)。 

提供的内容较为简单，易用但是对于高并发量其性能堪忧，因此有时候需要进行自行设计分布式锁。

## setNX命令
其语法为： ``SETNX key value`` 当且仅当key不存在设置成功返回1，存在时不做任何操作返回0。

当有多个进程同时调用此命令时，Redis保证了只有一个进程能够设置成功，即获取到了锁。当进程返回1时开始执行自己的工作，返回0时轮询查看锁的状态。

下面的代码通过setNX命令实现一个简易锁：

    public static String acquireLockWithTimeout(Jedis conn, long timeout) {
        long endTime = System.currentTimeMillis() + timeout;
        String identifier = UUID.randomUUID().toString();
        while (System.currentTimeMillis() < endTime) {
            long result = conn.setnx("lock", identifier);
            if (result == 1) {
                return identifier;
            }
        }

        return null;
    }

    public static boolean releaseLock(Jedis conn, String identifier) {
        conn.watch("lock");
        while (true) {
            try {
                if (conn.get("lock").equals(identifier)) {
                    Transaction tx =  conn.multi();
                    tx.del("lock");
                    tx.exec();
                    return true;
                }
                conn.unwatch();
                break;
            } catch (Exception ignore) {
                // 此时其他进程对锁进行了修改，重试
            }
        }
        return false;
    }

上述方法较为简单，加锁方法通过不断轮询setNX的返回值来判断是否获取到了锁，如果在指定时间内获得则返回生成的唯一标识，否则返回null。对于释放锁的方法来说，先比较lock的值是否是刚才加锁时获得的唯一标识符相同，相同则尝试在事务中删除锁，此时可能因为其余的进程对锁进行了修改并获取了锁，此时会放弃当前的事务，并进行重试。最外层的无限循环是为了之后实现超时锁使用的。

用此方法实现的简单锁可能会遇到死锁问题，当持有锁的进程意外退出的时候，此时锁不会得到释放，因此为锁设置一个超时时间是很重要的。

## setEX命令
可以用此命令来设置锁的过期时间，来解决上述提到的死锁问题，与NX连用的格式如下： ``SET key-with-expire-and-NX "hello" EX 10086 NX``，需要注意的是超时时间尽量设置得比要完成的所有任务中最耗时的任务时间要长，因为如果超时时间设置的过短的话，一个进程在完成任务之前锁就会被别的进程获取，此时还是会发生资源冲突，采用Jedis客户端实现超时锁如下：

    public static String acquireLockWithTimeout(Jedis conn, long timeout, long lockTimeout) {
        long endTime = System.currentTimeMillis() + timeout;
        String identifier = UUID.randomUUID().toString();
        int lockTimeInSeconds = (int) (lockTimeout / 1000L);
        // -1代表无限循环
        while (timeout == -1 || System.currentTimeMillis() < endTime) {
            // 成功加锁
            if (conn.setnx("lock", identifier) == 1) {
                conn.expire("lock", lockTimeInSeconds);
                return identifier;
            } else if (conn.ttl("lock") < 0) {
                conn.expire("lock", lockTimeInSeconds);
            }
        }

        return null;
    }

