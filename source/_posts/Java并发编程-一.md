---
title: Java并发编程(一)
date: 2017-03-13 18:46:34
tags: [Java,MutiThread]
categories: 多线程

---
**注：**一下部分文字图片来自《Java并发编程的艺术》
# 多线程简介
线程是现代操作系统调度的最小单元，一个进程可以包含很多线程。在Java的内存区域里面，每一个线程都有自己的程序计数器和虚拟机栈。处理器通过时间切片的方法在各个线程之间来回切换，可以用下面的代码查看JVM运行时各个线程的信息：

	public class ThreadTest {
	    public static void main(String[] args) {
	        ThreadMXBean mxBean = ManagementFactory.getThreadMXBean();
	        ThreadInfo[] threadInfoArray = mxBean.dumpAllThreads(false, false);
	
	        for (ThreadInfo info : threadInfoArray) {
	            System.out.println("Thread id: " + info.getThreadId() + " Thread name: " +
	                    info.getThreadName() + " Thread state: " + info.getThreadState());
	        }
	    }
	}
运行结果如下：

	Thread id: 9 Thread name: Monitor Ctrl-Break Thread state: RUNNABLE
	Thread id: 4 Thread name: Signal Dispatcher Thread state: RUNNABLE
	Thread id: 3 Thread name: Finalizer Thread state: WAITING
	Thread id: 2 Thread name: Reference Handler Thread state: WAITING
	Thread id: 1 Thread name: main Thread state: RUNNABLE
也就是说，运行main函数时，不只是main线程在运行，Java天生就是多线程的。
# 线程的优先级
在Java程序里面，线程的优先级通过**setPriority(int priority)**来设置，**priority**的取值范围为1-10，默认为5，线程的优先级越高，处理器对此线程所切的时间片越长，但是不同的操作系统的表现不同，有些会忽略掉线程的优先级，因此，设置优先级不会保证多线程程序的正确性，在实际编程过程中不应该使用。
# 线程的状态
线程的状态分为以下几种：

- **NEW** 被创建成功，还没有调用**start()**方法。
- **RUNNABLE** 正在运行中。
- **BLOCKED** 正在处于阻塞状态。
- **WAITING** 等待状态，此线程在等待其他线程做出特定的动作。
- **TIME_WAITING** 超时等待状态。
- **TERMINATED** 终止，线程结束。

拿上面的运行结果的线程名为**Finalizer**的线程来说，运行时处于**WAITING**状态，它在等待有对象进入它内部持有的**ReferenceQueue**，并执行清理动作，有关Finalizer的更多信息，查看：[GC执行finalize的过程](http://blog.csdn.net/rsljdkt/article/details/12242007)

为了说明各大状态，我先用开几个线程，然后用[jstack](http://blog.csdn.net/fenglibing/article/details/6411940)指令查看各线程的状态。

代码如下：

	class SleepUtils {
	    public static void sleep(int millionSeconds) {
	        try {
	            Thread.sleep(millionSeconds);
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        }
	    }
	}

	public class ThreadStateTest {
	
	    static class TimeWaitingThread implements Runnable {
	        @Override
	        public void run() {
	            while (true) {
	                SleepUtils.sleep(100);
	            }
	        }
	    }
	
	    static class PauseThread implements Runnable {
	        @Override
	        public void run() {
	            while (true) {
	                long current = System.currentTimeMillis();
	                while (System.currentTimeMillis() - current <= 3000);
	            }
	        }
	    }
	
	    static class Waiting implements Runnable {
	        @Override
	        public void run() {
	            while (true) {
	                synchronized (Waiting.class) {
	                    try {
	                        Waiting.class.wait();
	                    } catch (InterruptedException e) {
	                        e.printStackTrace();
	                    }
	                }
	            }
	        }
	    }
	
	    static class Block implements Runnable {
	        @Override
	        public void run() {
	            synchronized (Block.class) {
	                while (true) {
	                    SleepUtils.sleep(4000);
	                }
	            }
	        }
	    }
	
	    public static void main(String[] args) {
	        new Thread(new TimeWaitingThread(), "TimedWaitingThread").start();
	        new Thread(new PauseThread(), "PauseThread").start();
	        new Thread(new Waiting(), "WaitingThread").start();
	        new Thread(new Block(), "Block1").start();
	        new Thread(new Block(), "Block2").start();
	    }
	}
运行之后，先在终端运行[jps](http://blog.csdn.net/fenglibing/article/details/6411932)命令，结果如下：

	37729 
	38982 Launcher
	38983 AppMain
	38986 Jps
其中**AppMain**就是刚才开启的进程pid，现在运行**jstack 38983**，运行结果如下（部分）：

	"Block2" #14 prio=5 os_prio=31 tid=0x00007fc65c835000 nid=0x5703 waiting for monitor entry [0x0000700002ba2000]
	   java.lang.Thread.State: BLOCKED (on object monitor)
		at com.stephen.ThreadStateTest$Block.run(ThreadStateTest.java:58)
		- waiting to lock <0x00000007956f6ea8> (a java.lang.Class for com.stephen.ThreadStateTest$Block)
		at java.lang.Thread.run(Thread.java:745)
	
	"Block1" #13 prio=5 os_prio=31 tid=0x00007fc65d82a000 nid=0x5503 waiting on condition [0x0000700002a9f000]
	   java.lang.Thread.State: TIMED_WAITING (sleeping)
		at java.lang.Thread.sleep(Native Method)
		at com.stephen.SleepUtils.sleep(ThreadStateTest.java:10)
		at com.stephen.ThreadStateTest$Block.run(ThreadStateTest.java:58)
		- locked <0x00000007956f6ea8> (a java.lang.Class for com.stephen.ThreadStateTest$Block)
		at java.lang.Thread.run(Thread.java:745)
	
	"WaitingThread" #12 prio=5 os_prio=31 tid=0x00007fc65d82f800 nid=0x5303 in Object.wait() [0x000070000299c000]
	   java.lang.Thread.State: WAITING (on object monitor)
		at java.lang.Object.wait(Native Method)
		- waiting on <0x00000007956f37f0> (a java.lang.Class for com.stephen.ThreadStateTest$Waiting)
		at java.lang.Object.wait(Object.java:502)
		at com.stephen.ThreadStateTest$Waiting.run(ThreadStateTest.java:44)
		- locked <0x00000007956f37f0> (a java.lang.Class for com.stephen.ThreadStateTest$Waiting)
		at java.lang.Thread.run(Thread.java:745)
	
	"PauseThread" #11 prio=5 os_prio=31 tid=0x00007fc65e01e000 nid=0x5103 runnable [0x0000700002899000]
	   java.lang.Thread.State: RUNNABLE
		at com.stephen.ThreadStateTest$PauseThread.run(ThreadStateTest.java:33)
		at java.lang.Thread.run(Thread.java:745)
	
	"TimedWaitingThread" #10 prio=5 os_prio=31 tid=0x00007fc65c80e000 nid=0x4f03 waiting on condition [0x0000700002796000]
	   java.lang.Thread.State: TIMED_WAITING (sleeping)
		at java.lang.Thread.sleep(Native Method)
		at com.stephen.SleepUtils.sleep(ThreadStateTest.java:10)
		at com.stephen.ThreadStateTest$TimeWaitingThread.run(ThreadStateTest.java:23)
		at java.lang.Thread.run(Thread.java:745)
现在来逐一解释：

- **Block2**：由于**Block1**已经获得**Block.class**锁，所以此线程处于阻塞状态。
- **Block1**：此线程不断的进行睡眠，且睡眠操作具有时间限制，因此处在超时等待操作状态。
- **WaitingThread**：此线程在**Waiting.class**上等待，且没有时间限制，所以处在waiting状态。
- **PauseThread**：此线程和**TimedWaitingThread**线程达到的效果相同，但是实现的手段不同，没有调用**Thread.sleep(int millionSeconds)**方法，处在**Runnable**状态。
- **TimedWaitingThread**：此线程不断调用**Thread.sleep(int millionSeconds)**方法，处于超时等待状态。

# 线程的中断
## interrupt()
其他线程可以调用另一个线程的**interrupt()**方法来中断线程，可以调用线程的**isInterrupted()**来判断一个线程是否被中断，但是这个方法不总是返回true:

- 当线程终止时，此方法会返回false。
- 抛出**InterruptedException**的方法，在抛出异常之前，会将标识位设置为false，因此方法也总是返回false。

现在用如下程序做出证明：

	public class InterruptTest {
	    static class SleepThread implements Runnable {
	        @Override
	        public void run() {
	            while (true) {
	                SleepUtils.sleep(3000);
	            }
	        }
	    }
	
	    static class NormalThread implements Runnable {
	        @Override
	        public void run() {
	            while (true);
	        }
	    }
	
	    public static void main(String[] args) {
	        Thread sleepThread = new Thread(new SleepThread(), "Sleep");
	        Thread normalThread = new Thread(new NormalThread(), "Normal");
	
	        sleepThread.start();
	        normalThread.start();
	
	        SleepUtils.sleep(2000);
	
	        sleepThread.interrupt();
	        normalThread.interrupt();
	
	        System.out.println("Sleep thread interrupted is " + sleepThread.isInterrupted());
	        System.out.println("Normal thread interrupted is " + normalThread.isInterrupted());
	    }
	}
结果如下：

	Sleep thread interrupted is false	
	Normal thread interrupted is true
## 设置boolean变量
还可以设置boolean变量来通知线程的结束，这种方式比上面的**interrupt**方法更为优雅：

	public class BooleanFlagThread {
	    static class CounterRunnable implements Runnable {
	        long i = 0L;
	        private volatile boolean isCancelled = false;
	
	        public void cancel() {
	            isCancelled = true;
	        }
	
	        @Override
	        public void run() {
	            while (!isCancelled && !Thread.currentThread().isInterrupted()) {
	                i++;
	            }
	        }
	    }
	
	    public static void main(String[] args) {
	        CounterRunnable cr = new CounterRunnable();
	        Thread countThread = new Thread(cr, "Count");
	        countThread.start();
	
	        SleepUtils.sleep(2000);
	        cr.cancel();
	
	        System.out.println("" + cr.i);
	    }
	}
# 线程之间的通信：notify(), wait()
线程除了用**synchronized**,**volatile**来进行通信之外，还可以用notify和wait方法，使用这两个方法的前提是某个线程获取了相应对象的锁，典型应用的示例代码如下：

	public class NotifyWaitTest {
	    static Object lock = new Object();
	    static boolean flag = true;
	
	    static class Wait implements Runnable {
	        @Override
	        public void run() {
	            //获取对象的锁
	            synchronized (lock) {
	                while (flag) { // 2
	                    System.out.println("不满足相应的条件");
	                    try {
	                        //进入wait状态，同时释放对象的锁
	                        lock.wait();
	                    } catch (InterruptedException e) {
	                        e.printStackTrace();
	                    }
	                }
	                System.out.println("条件满足");
	            }
	        }
	    }
	
	    static class Notify implements Runnable {
	        @Override
	        public void run() {
	            //获取对象的锁
	            synchronized (lock) {
	                if (flag) {
	                    flag = false; // 1
	                    //通知改变
	                    lock.notify();
	                }
	
	                System.out.println("条件已改变");
	            }
	        }
	    }
	
	    public static void main(String[] args) {
	        Thread waitThread = new Thread(new Wait(), "Wait");
	        Thread notifyThread = new Thread(new Notify(), "Notify");
	        waitThread.start();
	        //保证wait先执行的方式
	        SleepUtils.sleep(2000);
	        notifyThread.start();
	    }
	}
输出的结果如下：

	不满足相应的条件
	条件已改变
	条件满足
需要注意的是：

- 在调用notify, wait方法之前，要先获得对象的锁。
- wait方法，会释放相应的锁。
- notify方法会将调用wait的线程从等待队列到同步队列，此时wait线程将从WAITING状态转到BLOCK状态。
- 只有当notify的线程释放对象锁时，另一个wait线程才会返回继续工作。
- notify和wait具有happens-before关系。