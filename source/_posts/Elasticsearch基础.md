---
title: Elasticsearch基础
date: 2017-09-16 21:41:08
tags: [搜索引擎]
categories: Elasticsearch

---
最近要对项目增加一个搜索的功能，MySQL自带的搜索引擎太差，所以将要搜索的关键数据存放到Elasticsearch做检索，下面来记录一下Elasticsearch的基本知识，Elasticsearch的版本为2.3.0。

<!--more-->

# 安装和配置
## 安装
从官网上下载Elasticsearch的压缩包并解压，mac版的下载地址为：[https://download.elasticsearch.org/elasticsearch/release/org/elasticsearch/distribution/tar/elasticsearch/2.3.0/elasticsearch-2.3.0.tar.gz](https://download.elasticsearch.org/elasticsearch/release/org/elasticsearch/distribution/tar/elasticsearch/2.3.0/elasticsearch-2.3.0.tar.gz)，不建议用浏览器下载实在是太慢了，把下载地址放到迅雷等专业的下载工具会好一点。

## 配置
### JVM配置
Elasticsearch是用Java开发的，所以配置好JVM参数对其运行的性能有绝对的影响，主要调节的有两个：Xms(初始运行内存),Xmx(最大内存)。配置在$ELASTICSEARCH_HOME/bin/elasticsearch.in.sh文件中：

    if [ "x$ES_MIN_MEM" = "x" ]; then
        ES_MIN_MEM=256m
    fi
    if [ "x$ES_MAX_MEM" = "x" ]; then
        ES_MAX_MEM=1g
    fi
    if [ "x$ES_HEAP_SIZE" != "x" ]; then
        ES_MIN_MEM=$ES_HEAP_SIZE
        ES_MAX_MEM=$ES_HEAP_SIZE
    fi

    # min and max heap sizes should be set to the same value to avoid
    # stop-the-world GC pauses during resize, and so that we can lock the
    # heap in memory on startup to prevent any of it from being swapped
    # out.
    JAVA_OPTS="$JAVA_OPTS -Xms${ES_MIN_MEM}"
    JAVA_OPTS="$JAVA_OPTS -Xmx${ES_MAX_MEM}"

这里最大内存最好设置为系统可用内存的一半，初始内存和最大内存设置成相同值。有关JVM优化，参见：[JVM优化总结][http://www.ibm.com/developerworks/cn/java/j-lo-jvm-optimize-experience/index.html],这里不再做专门介绍。

### 配置文件
配置文件在$ELASTICSEARCH_HOME/config文件夹下，一个是主要配置文件elasticsearch.yml，另外一个是日志文件logging.yml。

下面来看一下主要的配置说明。

#### cluster.name
集群的名称。要确保不同的环境下集群的名称不重复。

#### node.name
节点的名称。默认情况下当节点启动时elasticsearch会在3000个名字当中随机取出一个赋给node节点。

#### node.rack
节点描述。

#### path.data
数据存储位置，如果要设置多值用逗号分隔。

#### path.logs
日志存储位置。

#### bootstrap.mlockall
是否锁住内存。

#### network.host
绑定的IP地址。

#### http.port
Http所占用的端口。

#### discovery.zen.ping.unicast.hosts
开始发现新节点的地址。

#### discovery.zen.minimum_master_nodes
最多发现的master节点数。

#### gateway.recover_after_nodes
设置集群恢复所需要启动的最少的节点数。

#### node.max_local_storage_nodes
在一台机器上最多启动的节点数。

#### action.destructive_requires_name
删除索引时需要指定名称。

#### index.refresh_interval
设置索引刷新的时间。

## 运行
在$ELASTICSEARCH_HOME/bin当中执行elasticsearch命令即可：

    ./elasticsearch --cluster.name stephen_cluster --node.name zhangshirui

此时访问http://localhost:9200将会得到以下内容：

    {
      "name" : "zhangshirui",
      "cluster_name" : "stephen_cluster",
      "version" : {
        "number" : "2.3.0",
        "build_hash" : "8371be8d5fe5df7fb9c0516c474d77b9feddd888",
        "build_timestamp" : "2016-03-29T07:54:48Z",
        "build_snapshot" : false,
        "lucene_version" : "5.5.0"
      },
      "tagline" : "You Know, for Search"
    }

# 基本概念
## Lucene
Elaticsearch基于开源搜索引擎Lucene，有关Lucene的更多内容，参见[Lucene实例教程](blog.csdn.net/chenghui0317/article/details/10052103)及[倒排索引](https://wizardforcel.gitbooks.io/the-art-of-programming-by-july/content/06.11.html)。

## Elasticsearch术语
### 索引词(Term)
被索引的精确值，区分大小写，可用来进行精确匹配。
### 文本(Text)
一段非结构化文字。通常会被分词器分成一个个索引词，然后在底层建立倒排索引。
### 分析(Analysis)
将文本分成索引词的过程，主要依赖于分词器，比较著名的中文分词器有IK分词器，庖丁分词器。
### 集群(Cluster)
一个集群用cluster.name来进行标识，当有多个集群时，要确保集群的名称不重复，否则节点可能会加入错误的集群。
### 节点(Node)
集群的一部分，可以存储数据，并参加集群的索引和搜索过程，节点的名字用node.name来指定。
### 分片(Shard)
索引可以存储很大的数据，当索引的大小超过一个节点的物理存储上限时，就会将索引分成多个分片存储到不同的节点，
### 主分片(Primary Shard)
文档优先被存储到主分片当中，然后复制到其他不同的副本当中。默认的情况下一个索引包含5个主分片，分片数量应该预先指定，一旦建立则其数量无法更改。
### 副分片(Replica shard)
每一个分片可以有多个副本，复制的目的主要是两个：

- 提供HA，当主分片挂了副分片可以作为主分片使用。
- 提高查询性能，查询可以在主分片或者副分片上进行，主分片和副分片必须安置在不同的节点上。

### 索引(Index)
具备相同结构的文档集合，索引的名称全部小写，通过这个名称来执行索引的更新，删除操作。注意此处的索引和关系数据库中索引的概念不同。
### 类型(Type)
类型为索引的逻辑分区，是具有公共字段的文档。
### 文档(Document)
存储在Elasticsearch当中的JSON形式字符串，每一个存储在其中的文档包含有一个类型和ID，每个文档存储了多个字段，或者键值对。
### 映射(Mapping)
每一个索引包含一个映射，映射定义着索引中的字段类型，以及索引范围内的设置。
### 字段(Field)
每个文档包含有零个或者多个字段，相当于数据库中的列的概念。

# 使用Java客户端
Elasticsearch支持Rest风格API的调用，这里不再细谈，只看一下如何用Java客户端来操作Elasticsearch。调用存在两种方式，一种是作为节点加入，另外一种是作为客户端连接到集群(cs结构模式)，在这里就不记录Java API的使用了，详情见：[Elasticsearch Java Api](https://endymecy.gitbooks.io/elasticsearch-guide-chinese/content/java-api/README.html)