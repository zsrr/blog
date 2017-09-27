---
title: Redis故障恢复
date: 2017-09-27 15:50:54
tags: Redis
categories: Redis

---
本篇记录Redis故障恢复措施，参考书籍[《Redis实战》](https://item.jd.com/11791607.html)。

<!--more-->

# Redis持久化
在谈及故障恢复之前，首先要了解Redis的持久化选项，Redis提供了两个持久化选项，一种是快照(Snapshotting),一种是只追加文件(append-only file,aof)，两种方法可以同时使用也可以单独使用，下面分别介绍这两种方法的使用。

## 快照持久化
快照是指内存中的数据在某个时间点的副本，用户可以将快照进行备份，可以将其移植到其他的服务器来创建相同的数据副本，在Redis目录下的redis.conf文件中存储着快照备份有关的配置：

    save 60 1000 # 表示60s内有1000次写入的话将会创建快照，有多个save条件时，满足任意一个即可创建快照
    stop-writes-on-bgsave-error no # 当最后一次快照存储失败时，是否停止写入
    rdbcompression yes # 是否压缩
    dbfilename dump.rdb # 快照文件名

    dir ./ # 存储位置

在新的快照被创建之前，如果Redis发生崩溃的话，那么Redis将会丢失最后一次创建快照成功后的所有数据。

创建快照有以下几种办法：

- 直接发送BGSAVE指令来创建快照，此时Redis会从父进程fork出子进程来进行快照的创建父进程继续处理请求。
- SAVE指令，阻塞创建快照，创建成功后返回。
- 在配置文件当中存在save选项，根据配置条件触发BGSAVE指令。
- 在通过SHUTDOWN关闭Redis时，或者收到标准信号TERM时，Redis会发出SAVE指令创建快照文件，此时不再接受来自客户端的任何请求。
- 当从服务器向主服务器发送SYNC指令时，如果主服务器没有进行BGSAVE操作，则会发出BGSAVE指令。

利用快照来进行持久化，主要用于那些对数据丢失可以容忍，并且数据量少的情形，因为如果数据量过大的话，Redis占用的内存也随之增加，BGSAVE创建子进程也会使系统变得卡顿。

## AOF持久化
采用此种持久化方式，Redis会将写命令写到AOF文件的末尾，当执行数据恢复的时候，只需要重新执行AOF文件当中的指令即可。下面是关于AOF持久化的配置：

    appendonly no # 是否开启此种持久化方式，默认不会开启
    appendfilename # 文件的名称
    appendfsync everysec # 写入的频率
    no-appendfsync-on-rewrite no
    auto-aof-rewrite-percentage 100
    auto-aof-rewrite-min-size 64mb

写入频率appendsync有三个选项：always, no, everysec，always是每接受到写命令都将其写入磁盘，是最为安全的策略，但是会严重影响性能。no选项是将写入托付给底层操作系统，是最快的策略。everysec做了中和，因此是绝佳选择。

AOF的缺陷就是随着Redis运行时间的增长，AOF文件将会变得越来越大，最后变得不可控制，为了解决这个问题，用户可以发送BGREWRITEAOF指令重新写AOF文件，删除冗余AOF文件减小体积，BGREWRITEAOF指令也会fork一个子进程进行重写，如果旧的AOF文件非常大，则会造成系统挂起数秒。

像之前设置save选项一样，我们可以通过设置auto-aof-rewrite-percentage和auto-aof-rewrite-min-size选项来让Redis自动进行AOF文件重写，两个选项的含义如下：

- **auto-aof-rewrite-percentage**: 重写时AOF文件大小比上一次重写增加的比例。
- **auto-aof-rewrite-min-size**: 重写时AOF文件大小达到的最小值。

举个例子，如果按照之前展示的默认配置，则当AOF大小为64m且比上一次重写时AOF大小大了整整一倍时，会自动执行BGREWRITEAOF。

看完了持久化机制，现在来看Redis的故障恢复。

# 故障恢复
## 检查AOF文件和快照文件
当Redis重启的时候，可以用redis-check-aof和redis-check-dump来恢复Redis，用法如下：

    redis-check-aof [--fix] <file>
    redis-check-dump <file>
当为redis-check-aof指令指定fix参数的时候，此时Redis将尝试恢复aof文件，恢复的方法为找到aof文件当中最先错误的指令并将后面的指令全部抛弃，只保留之前正确的部分。

## 更换主服务器
假设A,B是一个Redis集群，A为master,B为slave，此时若A宕机，改为C当作主服务器，此时可以在B之上运行SAVE指令生成快照，然后传到机器C上，最后将B作为C的Slave。
