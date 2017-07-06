---
title: Java非阻塞算法
date: 2017-06-22 16:18:13
tags: Java
categories: Java

---
啊，于今日可算是看完了《Java并发编程实战》这本书。
# 非阻塞算法简介
在基于锁的并发实现中，如果一个线程在持有锁时发生阻塞I/O，socket阻塞等等状况，那可能其他等待获取锁的线程都不能执行下去。如果某种算法使线程的失败不会导致此线程挂起或着阻塞至条件为真，只是单纯地返回，那么此算法称为非阻塞算法，通常在硬件层面得到支持。非阻塞算法在中低下的竞争环境下比锁的性能要好得多。
# 基于CAS的非阻塞算法
在大多数的处理器架构中采用比较并交换（Compare And Swap）的方法实现非阻塞算法。CAS包含了三个操作数——需要读写的内存位置V，进行的比较的值A和要设置的值B，只有当V中的值为A时，CAS才会通过原子的方式将V中的值设置为B。**java.util.concurrent.atomic**包下的各种原子类都提供了CAS操作。

下面通过实现一个非阻塞计数器来说明CAS非阻塞算法的用法：

	public class CASCounter {
	    private AtomicInteger counter = new AtomicInteger(0);
	    
	    public int getValue() {
	        return counter.get();
	    }
	    
	    public void increase() {
	        int v;
	        do {
	            v = counter.get();
	        } while (!counter.compareAndSet(v, v + 1));
	    }
	}
如果其他线程比当前线程提前更新了counter的值，那么将会导致此线程进行重试。CAS算法一个麻烦的地方就是要自己手动写重试的条件及内容。
# 非阻塞数据结构
在使用过程中不会使当前线程发生阻塞的数据结构为非阻塞数据结构。如**ArrayBlockingQueue**为阻塞数据结构，**ConcurrentLinkedQueue**为非阻塞数据结构。

非阻塞数据结构一般设计起来比较困难，需要专业算法工程师进行操作，下面通过两个简单的例子来说明非阻塞数据结构的设计使用。

## 非阻塞栈
下面通过**Treiber**算法实现一个非阻塞的栈：

	public class ConcurrentStack<E> {
	
	    AtomicReference<Node<E>> top = new AtomicReference<>();
	
	    public void push(E item) {
	        Node<E> newHead = new Node<>(item);
	        Node<E> oldHead = null;
	        do {
	            oldHead = top.get();
	            newHead.next = oldHead;
	        } while (!top.compareAndSet(oldHead, newHead));
	    }
	
	    public E pop() {
	        Node<E> oldHead, newHead;
	        do {
	            oldHead = top.get();
	            if (oldHead == null)
	                return null;
	            newHead = oldHead.next;
	        } while (!top.compareAndSet(oldHead, newHead));
	        return oldHead.item;
	    }
	
	    private static class Node<E> {
	        final E item;
	        Node<E> next;
	
	        Node(E item) {
	            this.item = item;
	        }
	    }
	}
栈是一种最简单的链式结构，无需多说，特别简单。

非阻塞栈和之前的技术器很好地说明了CAS的基本使用模式：在更新某个值时存在不确定性，尝试失败后重新尝试。技巧在于：将执行原子修改的范围缩小到单个变量上。
## 非阻塞链表
链表的实现比栈更加复杂，插入元素的时候，最后一个元素节点（尾节点）以及最后一个元素的下一个节点都需要更新。乍看起来，这两步操作不能通过原子变量来实现，这里需要两个技巧：第一个技巧是，即便在一个包含多个步骤的更新操作当中，需要确保数据结构处于一致的状态。当线程B到达时，如果发现线程A正在执行更新，那么线程B不能立即执行自己的更新操作；第二个技巧是，如果线程B发现线程A正在更新数据结构，那么线程B可以“帮助”线程A完成更新操作。代码如下：

	public class ConcurrentLinkedQueue<E> {
	    private static class Node<E> {
	        final E item;
	        final AtomicReference<Node<E>> next;
	
	        public Node(E item, Node<E> next) {
	            this.item = item;
	            this.next = new AtomicReference<>(next);
	        }
	    }
	    
	    private final Node<E> dummy = new Node<>(null, null);
	    private final AtomicReference<Node<E>> head = new AtomicReference<>(dummy);
	    private final AtomicReference<Node<E>> tail = new AtomicReference<>(dummy);
	    
	    private void push(E item) {
	        Node<E> newNode = new Node<>(item, null);
	        while (true) {
	            Node<E> curTail = tail.get();
	            Node<E> tailNext = curTail.next.get();
	            
	            if (curTail == tail.get()) {
	                //当前处于中间状态，此时其余线程正在修改数据结构
	                if (tailNext != null) {
	                    //尝试完成其他线程的工作
	                    tail.compareAndSet(curTail, tailNext);
	                } else {
	                    //尝试插入新节点
	                    if (curTail.next.compareAndSet(null, newNode)) {
	                        //推进旧节点
	                        tail.compareAndSet(curTail, newNode);
	                        return;
	                    }
	                }
	            }
	        }
	    }
	}
# ABA问题
在CAS算法当中，如果在内存V处的值A先变成B，然后再变成A，那么普通的方式会认为没有发生改变，那么就不会发生交换操作，但实际上已经发生了改变，属于CAS的一种异常情况。

常见的解决方案是通过版本号来标识对象，即便对象从A到B再到A，其版本号也是不同的。**AtomicStampedReference**支持在两个变量上执行原子的条件更新，通过更新引用的版本号，可以避免ABA问题。