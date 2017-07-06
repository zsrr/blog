---
title: Java内存模型(三)
date: 2017-03-13 18:44:31
tags: [Java,MutiThread]
categories: Java

---
**注：**一下部分文字图片来自《Java并发编程的艺术》
# 双重检查锁解决方案
Java程序有时需要用到单例模式，通常用延迟初始化的方式实现单例模式，但是只是简单的延迟初始化可能会有线程安全问题，例如：

	public class Instance {
	    private static Instance instance;
	    
	    public static Instance getInstance() {
	        if (instance == null) { //1
	            instance = new Instance();
	        }
	        
	        return instance;
	    }
	}
这段代码有很严重的问题，加入两个线程同时处于1操作并且判断instance为空的话，那么instance就会被初始化两次。
## synchronized解决方案
解决上述问题最简单的方法便是用**synchronized**关键字标注**getInstance**方法：

	public synchronized static Instance getInstance() {
        if (instance == null) {
            instance = new Instance();
        }

        return instance;
    }
此方法解决了多线程冲突的问题，但是如果线程太多且**instance**初始化过程太耗时的话，就会转成重量级锁，性能问题随之而来。
## 双重检查锁定(Double Checked Lock)
为了解决性能的开销，采取了下述办法来解决：

	public class Instance {
	    private static Instance instance;
	
	    public static Instance getInstance() {
	        if (instance == null) { //1
	            synchronized (Instance.class) { //2
	                if (instance == null) { //3
	                    instance = new Instance();
	                }
	            }
	        }
	        
	        return instance;
	    }
	}
采用上述办法不能完美地解决问题，出现的问题是返回的引用可能指向一个未被完全正确初始化的对象，下面来分析一下。

假设初始有两个线程A,B同时调用**getInstance**方法，判断instance为空之后开始竞争**Instance**锁，设B线程首先获得了锁，并在成功初始化**instance**对象之后释放锁，此时线程A再获取锁，根据happens-before关系，此时线程A对**instance**对象的改变是可以看到的，因此整个过程只初始化了一次，截止到现在是没有问题的。

### 对象的初始化过程
上述办法的问题来自于对象初始化过程的重排序，**instance**初始化可以分成以下三个过程：

- 分配对象的存储空间。
- 进行初始化。
- 将对象的地址赋值给引用。

问题来自于第二步和第三步的重排序。加入现在有两个线程A,B同时执行，B 先获得了锁先执行，此时A执行到上面的1操作，但是B线程此刻已经把对象地址赋值给了引用，但是并没有正确的初始化（3和1不具备happens-before）关系，此刻A线程读到的**instance**引用是不为空的，但是对象并没有正确的初始化，便即刻将其暴露给外部，问题就出现了。
### 基于volatile的解决方案
在**instance**对象前面加上**volatile**关键字可以禁止上述初始化过程的重排序：

	public class Instance {
	    private volatile static Instance instance;
	
	    public static Instance getInstance() {
	        if (instance == null) {
	            synchronized (Instance.class) {
	                if (instance == null) {
	                    instance = new Instance();
	                }
	            }
	        }
	
	        return instance;
	    }
	}
**volatile**关键字保证了读取到引用不为空时对象已经被正确地初始化，问题解决。
### 基于类初始化的解决方案

	public class Instance {
	    private static class InstanceHolder {
	        private static Instance instance = new Instance();
	    }
	    
	    public static Instance getInstance() {
	        return InstanceHolder.instance;
	    }
	}
此种方法适合包含无参构造函数的类的单例模式，原理如下：

假设有两个线程A,B当两个线程同时开始访问**Instance**类的时候，开始类的初始化，此时JVM将尝试获取**Instance**类上的锁，防止多个线程同时初始化。

假定线程B获取了初始化的锁，当执行完**instance = new Instance()**语句时，释放锁，线程A执行时将会看到已经初始化的**instance**实例并将其返回，初始化过程的重排序对于线程A此时是不可见的。
