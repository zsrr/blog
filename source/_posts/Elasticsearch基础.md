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

## 运行

# 基本概念

# 对外接口
## Http接口
## Java接口




