---
title: Java内存模型(二)
date: 2017-03-11 11:08:01
tags: [Java,MutiThread]
categories: Java

---
**注：**一下部分文字图片来自《Java并发编程的艺术》
# volatile
## 特性
**volatile**的典型特性之一是一个线程对**volatile**变量的写入对其他线程来说是立即可见的。假设有以下程序：

	public class VolatileFeaturesExample {
	    private volatile int a;
	
	    public int getA() {
	        return a;
	    }
	
	    public void setA(int a) {
	        this.a = a;
	    }
	}
与下面这段代码的执行效果是等效的：

	public class VolatileFeaturesExample {
	    private int a;
	
	    public synchronized void setA(int a) {
	        this.a = a;
	    }
	
	    public synchronized int getA() {
	        return a;
	    }
	}
根据happens-before规则，锁的解锁happens before于此锁被加锁之前，所以一个线程对**a**变量的写入对另一个线程是可见的。

另外，锁的语义决定了**volatile**的读写具备原子性，也就是说即便对于**long**和**double**类型，只要有**volatile**关键字修饰，对其赋值也是原子操作。
## 限制重排序
为了实现volatile的内存语义， JMM针对volatile变量制定了一套特殊的重排序规则，规则如下表：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-03-11%20%E4%B8%8B%E5%8D%884.22.57.png)

从图中可以看出：

- 若第二个操作是对volatile对象的写操作，那么无论第一个操作是什么，都不会发生重排序。
- 若第一个操作是对volatile变量的读操作，那么无论第二个操作是什么，都不会发生重排序。

上一篇博客曾经提到**volatile**的实现是通过增加内存屏障来实现的，针对**volatile**的内存屏障策略如下：

- 在volatile变量写之前加入store屏障。
- 在volatile变量写之后加入store-load屏障。
- 在读操作的后面加上load屏障。
- 在读操作的后面加上load-store屏障。

# 锁
## 特性
先看下面这段代码：

	public class LockExample {
	    private int a;
	    
	    public synchronized void increase() { // 1
	        a++; // 2
	    } // 3
	    
	    public synchronized void get() { // 4
	        int i = a; // 5
	    } // 6
	}
根据happens-before原则，1发生在2之前，2发生在3之前……最终得出的结论是2发生在5之前，也就是说，两个线程，线程1获得锁之后对所有共享变量的修改，对下一个线程获得锁的时候是可见的。
## 内存语义
锁的内存语义如下：

- 一个线程释放锁，会将本地内存中的变量推送到主内存当中。
- 一个线程获得锁的时候，会将本地内存的变量设置为无效状态，强制从主内存中读取。

可见，锁的内存语义和**volatile**的内存语义差不多，区别就是锁对于共享变量的刷新和重新读取是全局的，volatile是局部的。
## 内存语义的实现
下篇再更新，可以先看这篇文章：[ReentrantLock源码分析](http://blog.csdn.net/yuhongye111/article/details/39053067)
# final
## 特性
被final修饰的变量只要在构造器之内进行合理的初始化并且没有发生[this引用逃逸](http://blog.csdn.net/flysqrlboy/article/details/10607295)，那么此字段对其他线程来说就是可见的。
## 写重排序规则
对final域进行写入的重排序规则为：在构造函数内对一个final域的写入，与随后把这个被构造对象的引用赋值给另一个引用，这两个指令不能发生重排序。

如下代码：

	public class FinalExample {
	    int i = 0;
	    final int j;
	    static FinalExample fe;
	
	    public FinalExample() {
	        i = 1;
	        j = 1;
	    }
	
	    public static void writer() {
	        fe = new FinalExample();
	    }
	
	    public static void reader() {
	        FinalExample obj = fe; //1
	        int a = obj.i; //2
	        int b = obj.j; //3
	    }
	}
假设线程A执行writer函数，final的重排序规则保证了在对fe赋值之前j已经得到了合理的初始化，但是对于普通域而言，在fe被赋值时，i可能未被得到合理的初始化，原因是i = 1可能因为重排序的关系放在了构造函数之外。
## 读重排序规则
在一个线程中，初次读对象引用和读取对象的final域，这两条指令不能发生重排序。还是上面的代码，若线程B执行reader函数，假设此时fe已经不是null，那么1和3操作就不能发生重排序，b的值便是j的值。但是对于普通域i来说情况就不一定了，因为此时i可能因为重排序的关系，初始化过程还没有进行。
## final引用类型
对于引用类型，final重排序规则如下：

在构造器内对final引用进行修改，与之后将此被构造对象赋值给其他引用，这两个操作不能发生重排序。例如：

	public class FinalExample {
	    final int[] finalArrayReference;
	    static FinalExample fe;
	    
	    public FinalExample() {
	        finalArrayReference = new int[10];
	        finalArrayReference[0] = 1;
	    }
	    
	    public static void writer() {
	        fe = new FinalExample();
	    }
	}
此时若有线程A执行writer函数，那么对于此时fe的finalArrayReference域不为空，其第一个位置的值也得到了初始化。