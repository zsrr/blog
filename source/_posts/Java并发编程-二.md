---
title: Java并发编程(二)
date: 2017-03-14 20:20:52
tags: [Java,MutiThread]
categories: 多线程

---
**注：**以下内容均摘自《Java并发编程的艺术》
# 等待／超时模式
## 介绍
等待／超时模式的典型应用场景是，等待一段时间，看是否能得到正确的结果，如果能，则将其返回，不能则返回默认值，其代码如下：

	public synchronized Object getObject(long millionSeconds) throws InterruptedException {
        long future = System.currentTimeMillis() + millionSeconds;
        long remainingTime = millionSeconds;
        while (obj == null && remainingTime > 0) {
            wait(remainingTime);
            remainingTime = future - System.currentTimeMillis();
        }

        return obj;
    }
## 应用：数据库连接池
下面来模拟一个数据库连接池，用来从中获取和释放连接，客户端获取连接被设置为超时等待，程序如下：

	public class ConnectionPool {
	    private final LinkedList<Connection> connections = new LinkedList<>();
	
	    public ConnectionPool(int initSize) {
	        if (initSize <= 0)
	            throw new IllegalArgumentException("Init size must > 0");
	        for (int i = 0; i < initSize; i++) {
	            connections.addLast(ConnectionDriver.getConnection());
	        }
	    }
	    
	    public void releaseConnection(Connection connection) {
	        synchronized (connections) {
	            connections.addLast(connection);
	            connections.notifyAll();
	        }
	    }
	    
	    public Connection getConnection(long millionSeconds) throws InterruptedException {
	        synchronized (connections) {
	            if (millionSeconds <= 0) {
	                while (connections.isEmpty()) {
	                    connections.wait();
	                }
	                return connections.removeFirst();
	            } else {
	                long future = System.currentTimeMillis() + millionSeconds;
	                long remaining = millionSeconds;
	                
	                while (connections.isEmpty() && remaining > 0) {
	                    connections.wait(remaining);
	                    remaining = future - System.currentTimeMillis();
	                }
	                
	                Connection connection = null;
	                if (!connections.isEmpty()) {
	                    connection = connections.removeFirst();
	                }
	                
	                return connection;
	            }
	        }
	    }
	}
ConnectionDriver通过动态代理技术返回一个Connection:

	public class ConnectionDriver {
	
	    public static class ConnectionHandler implements InvocationHandler {
	        @Override
	        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	            if (method.getName().equals("commit")) {
	                Thread.sleep(1000);
	            }
	
	            return null;
	        }
	    }
	
	    public static Connection getConnection() {
	        return (Connection) Proxy.newProxyInstance(Connection.class.getClassLoader(),
	                new Class[] {Connection.class},
	                new ConnectionHandler());
	    }
	}
上面的连接池通过构造函数传入的initSize来定义里面的连接数量，通过**release**方法来释放用过的连接，此时连接被重新加入到队列内部，然后通知在**connections**上等待的线程此时有连接可以进行复用。**getConnection**通过制定的参数设置超时时间，并判断队列是否为空，不为空则直接从队列里面取出第一个并返回，不是的话则在**connections**上进行等待，知道超时或者又有复用的connection为止。