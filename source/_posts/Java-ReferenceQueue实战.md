---
title: Java ReferenceQueue实战
date: 2017-01-22 22:28:09
tags: Java
categories: Java

---
## 浅谈ReferenceQueue
先看官方文档对这个类的介绍：  
> Reference queues, to which registered reference objects are appended by the
garbage collector after the appropriate reachability changes are detected.  

这个的大体意思是这样的，当一个Reference类所指向的实际对象被回收的时候，这个Referece便会加入到这个队列里面，这个队列提供了三个操作：

	public Reference<? extends T> poll();
	
	public Reference<? extends T> remove(long timeout);
	
	public Reference<? extends T> remove();
三个都是出队操作，后两个remove是阻塞方法，具体请看官方文档，懒得解释。

## ReferenceQueue实战
笔者将用一个简单粗暴的例子来解释这个类的实际用途。假设在一个公司的内部，有员工(Staff)，有事务(Work)，每一个事务都必须对应一个员工去做。假如这个员工突然有急事请假了，或者工作太累不小心嗝屁了（相应的Staff类实例被回收了），那这个Work类实例可以做一些清理工作，或者通知其他相关类它所对应的Staff已经被回收。那么这个模型就可以完美地应用ReferenceQueue，下面是代码实现：

Work类：

	public class Work {
    	static class StaffWeakReference extends WeakReference<Staff> {
        	Work work;

        	public StaffWeakReference(Work work, Staff referent, ReferenceQueue<? super Staff> q) {
            	super(referent, q);
            	this.work = work;
        	}
    	}

    	StaffWeakReference swReference;

    	Work(Staff staff, ReferenceQueue<? super Staff> referenceQueue) {
        	swReference = new StaffWeakReference(this, staff, referenceQueue);
    	}

    	public void cancel() {
        	// 可以做一些清理工作，或者通知其他相关类它所对应的Staff已经被回收
    	}
	}
Company类：

	public class Company {

    	static class CleanUpThread extends Thread {
        	ReferenceQueue<?> referenceQueue;
        	static final long TIME_OUT = 1000;

        	CleanUpThread(ReferenceQueue<?> referenceQueue) {
            	this.referenceQueue = referenceQueue;
        	}

        	@Override
        	public void run() {
            	while (true) {
                	try {
                    	Work.StaffWeakReference reference = (Work.StaffWeakReference) referenceQueue.remove(TIME_OUT);
                    	if (reference != null) {
                        	reference.work.cancel();
                    	}
                	} catch (InterruptedException e) {
                    	e.printStackTrace();
                	}
            	}
        	}
    	}

    	public static void main(String[] args) {
    		ReferenceQueue<Staff> referenceQueue = new ReferenceQueue<>();
        	CleanUpThread cleanUpThread = new CleanUpThread(referenceQueue);
        	cleanUpThread.start();
        	Staff staff = new Staff();
        	Work work = new Work(staff, referenceQueue);
        	...
        	...
    	}
	}
当main开始运行的时候，先开启一个CleanUpThread，这个类的run方法内部则是不断的从传入的ReferenceQueue当中读取所指向Staff对象已经被回收的引用，如果某个Staff已经被回收，那它在对应的Work里面对应的弱引用便会读取出来，这时候就可以调用读出的弱引用对应的Staff已经被回收的Work的cancel方法了。