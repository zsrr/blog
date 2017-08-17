---
title: 使用Fork/Join架构进行归并排序
date: 2017-03-14 23:37:51
tags: [Java,MutiThread]
categories: 多线程

---
多线程排序算法，有一点分治的思想
<!--more-->

**注：**以下部分文字，图片摘自《Java并发编程的艺术》
# Fork/Join框架简介
Fork/Join框架是jdk7中提供的并行框架，其主要的特点是把一个大任务分割成几块不同的小任务，由不同的线程去执行这一系列的小任务，最终结果合并成大任务的结果。

其主要原理如图所示：
![](http://cdn.infoqstatic.com/statics_s1_20170314-0434/resource/articles/fork-join-introduction/zh/resources/21.png)
# 工作窃取算法
工作窃取算法的意思是每个线程的任务队列执行完毕之后，此线程会将其他线程的任务队列中的任务“窃取”到本线程的任务队列中来。使用窃取算法的好处就是充分利用了线程的并行计算，有关工作窃取算法的更多内容，请看：[Work Stealing](https://en.wikipedia.org/wiki/Work_stealing)
# 实际应用
再实际应用上，要首先确定一个大的任务是否能够分成几个小的任务，是否能把小任务的结果进行合并，这样就自然而然的想到递归。下面笔者实现一个用Fork/Join架构实现的归并排序算法：

	public class ForkJoinPractice {
	    static class SortTask<T extends Comparable<T>> extends RecursiveAction {
	        T[] array;
	        int start;
	        int end;
	        int mid;
	
	        public SortTask(T[] array, int start, int end) {
	            this.array = array;
	            this.start = start;
	            this.end = end;
	            this.mid = (this.start + this.end) / 2;
	        }
	
	        @Override
	        protected void compute() {
	            int mid;
	            if (start < end) {
	                mid = (start + end) / 2;
	                //将任务分解
	                invokeAll(new SortTask<T>(array, start, mid), new SortTask<T>(array, mid + 1, end));
	                merge();
	            }
	        }
	
	        private void merge() {
	            int n1 = mid - start + 1;
	            int n2 = end - mid;
	            int i, j, k;
	
	            List<T> leftList = new ArrayList<>(n1);
	            List<T> rightList = new ArrayList<>(n2);
	
	            for (i = 0; i < n1; i++) {
	                leftList.add(array[start + i]);
	            }
	
	            for (j = 0; j < n2; j++) {
	                rightList.add(array[mid + 1 + j]);
	            }
	
	            i = j = 0;
	            k = start;
	
	            while (i < n1 && j < n2) {
	                if (leftList.get(i).compareTo(rightList.get(j)) < 0) {
	                    array[k++] = leftList.get(i++);
	                } else {
	                    array[k++] = rightList.get(j++);
	                }
	            }
	
	            while (i < n1) {
	                array[k++] = leftList.get(i++);
	            }
	
	            while (j < n2) {
	                array[k++] = rightList.get(j++);
	            }
	        }
	    }
	
	    public static void main(String[] args) {
	        Integer[] array = new Integer[90];
	        for (int i = 0; i < 90; i++) {
	            array[i] = 90 - i;
	        }
	
	        SortTask<Integer> sortTask = new SortTask<>(array, 0, array.length - 1);
	        ForkJoinPool pool = new ForkJoinPool();
	        pool.submit(sortTask);
	        try {
	        	//等待任务结束
	            pool.awaitTermination(3, TimeUnit.SECONDS);
	            pool.shutdown();
	            for (int i :  array) {
	                System.out.println(i);
	            }
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        }
	    }
	}