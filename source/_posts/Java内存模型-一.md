---
title: Java内存模型(一)
date: 2017-03-10 20:06:10
tags: [Java,MutiThread]
categories: Java

---
**注：**一下部分文字图片来自《Java并发编程的艺术》
# 概述
JMM(Java Memory Model),即Java内存模型，是Java实现并发的主要机制。多个线程通过JMM共享程序公有状态进而实现隐式通信。
# 抽象结构
程序的公有状态，共享变量等储存在主内存(Main Memory)当中。每个线程都有一个本地内存(Local Memory)来储存主内存中共享变量的副本，其结构如下图所示：
![](http://cdn.infoqstatic.com/statics_s1_20170228-0434_4/resource/articles/java-memory-model-1/zh/resources/11.png)
（本图作者为Java资深软件工程师：[程晓明](http://ifeve.com/author/25622818/)）

此时两个线程A,B的通信流程为：

- 线程A更新本地内存中共享变量的副本，并将其写入主内存。
- 线程B从主内存中读取更新过的共享变量。

# 重排序
重排序是指为了提高程序运行的性能，充分利用并行，来对指令进行重新排序，主要分为编译器优化重排序和指令级并行重排序。关于重排序的更多理解，参见：[理解重排序](http://blog.hesey.net/2011/07/reordering.html)
## 数据依赖性
数据依赖性分为三种情况：

- 写后读，对一个变量写之后再进行读取。
- 读后写，对一个变量读取之后再进行写入。
- 写后写，对一个变量写之后再进行写入。

以上三种情况若两个指令发生了重排序，则产生的结果与预期不一致，称以上三个指令对两指令之间存在数据依赖性，存在数据依赖性的两个指令不会发生重排序。
## as-if-serial
**as-if-serial**的含义是指，对于一个单线程而言，无论怎么重排序，其结果与顺序执行的结果一致。为了遵循这个原则，在单线程环境中，如果两个指令指令之间存在数据依赖性，那么这两个指令便不会发生重排序。
## 对多线程的影响
假设有以下程序：

	public class RecorderExample {
	    private boolean flag = false;
	    private int i = 0;
	
	    public void write() {
	        i = 1;       //1
	        flag = true; //2
	    }
	
	    public void read() {
	        if (flag) {
	            int a = i * i; 
	        }
	    }
	
	    public static void main(String[] args) {
	        RecorderExample re = new RecorderExample();
	        new Thread(re::write).start();
	        new Thread(re::read).start();
	    }
	}
在第一个线程里面，1、2指令由于没有数据依赖，所以可以进行重排序，若两个进行了重排序，**flag**设置成true之后，切换到第二个线程进行条件判断，判断完成后对a进行赋值，此时变量**i**并没有进行设置，所以会发生异常的结果，也就是说，**重排序会破坏多线程的语义**。
# 内存屏障
它指的是一组用来实现对内存操作的顺序限制的处理器指令，会根据需求禁止特定类型的重排序。先假定这样一个事实：内存数据被推送到缓冲区，就会有消息协议来保证缓存和内存的数据一致性，这种尽快保证数据可见性的技术称为内存屏障。

内存屏障有三种类型：

- Store屏障：在该屏障之前的Store指令都会被执行，即写操作都会刷新到主内存当中去。
- Load屏障，在该屏障之后的load指令在该屏障之后执行，保证了处理器缓存加载成功后进行读操作。
- Full Barrier:即结合了上述两种屏障功能的屏障。

一个比较典型的例子是Java当中的**volatile**关键字，对**volatile**变量写指令的后面会加上store屏障，对**volatile**变量读命令的前面会加上load屏障。

关于内存屏障的更多内容，参见[内存屏障与JVM并发](http://www.infoq.com/cn/articles/memory_barriers_jvm_concurrency)
# happens-before
如果一个操作的结果对另一个操作是可见的，那么这两个操作必须遵循**happens-before**关系。在这里两个操作可以在同一个线程里面，也可以在不同的线程里面，常见的规则如下：

- 单个线程中的操作，happens-before于其后续操作。

这句话很好理解，假定单个线程中的两条指令不存在数据依赖性，那这两条指令则不满足**happens-before**的先决条件；若存在数据的依赖性，则一条指令必定发生在另一条之前。

- 对一个锁的解锁，happens-before于对这个锁的加锁。
- 对一个**volatile**写操作happens-before于对这个变量的读操作。

# 顺序一致性模型
是一个理想化的理论模型，主要有两大特征：

- 单个线程的操作必须按照程序的顺序来执行。
- 每个线程都只有一个单一的执行序列（不管是否同步），且每个操作必须立刻对所有线程可见。

## JVM的实现
### 同步执行
还是刚才的RecorderExample,现在修改如下：

	public class RecorderExample {
	    private boolean flag = false;
	    private int i = 0;
	
	    public synchronized void write() {
	        i = 1; //1
	        flag = true; //2
	    }
	
	    public synchronized void read() {
	        if (flag) {
	            int a = i * i;
	        }
	    }
	
	    public static void main(String[] args) {
	        RecorderExample re = new RecorderExample();
	        new Thread(re::write).start();
	        new Thread(re::read).start();
	    }
	}
加上**synchronized**关键字之后，**write** happens-before **read**，但是在第一个线程当中，1和2操作是有可能被重排序的，但是对结果没有任何的影响，可以看成是和顺序一致性模型达到了相同的效果。
### 非同步执行
非同步的执行结果无法预知，也没有什么实际意义。JVM对非同步执行只提供一种安全性，即在读一个变量的时候，要么是某个其他线程已经设置好的值，要么是零值，这里无需再做分析。