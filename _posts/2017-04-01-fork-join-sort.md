---
layout: post
title: ForkJoinSort
categories: [Java]
description: 
keywords: JDK
---

##1. 引言
ForkJoin框架在[这里](http://niceaz.com/java-executors%E6%A1%86%E6%9E%B6%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/#more-147)有过介绍，现在实现用ForkJoin框架实现快速排序，并用jUnit测试算法的正确性和效率。

##2. ForkJoinQuickSort源码与注释
	import java.util.concurrent.ForkJoinPool;
	import java.util.concurrent.RecursiveAction;
	
	public class ForkJoinQuickSort {
		
		private class ForkJoinTask extends RecursiveAction{
			private static final long serialVersionUID = -8563402994473910391L;
	
			private final int THRESHOLD = 5; //阈值
			
			private int left;
			private int right;
			private int[] arr;
			
			//构造函数
			public ForkJoinTask(int left, int right, int[] arr) {
				// TODO Auto-generated constructor stub
				this.left = left;
				this.right = right;
				this.arr = arr;
			}
			
			//插入排序
			private void InsertSort(int[] arr, int left, int right){
				if(left >= right){
					return ;
				}
				
				for(int i = left + 1; i <= right; i++){
					int tmp = arr[i];
					int j = i - 1;
					while(j >= left && arr[j] > tmp){
						arr[j + 1] = arr[j];
						j--;
					}
					
					arr[j + 1] = tmp;
				}			
			}
	
			@Override
			protected void compute() {
				//验证当前任务规模，小于阈值使用插入排序
				if(right - left <= this.THRESHOLD){
					InsertSort(arr, left, right);
					return ;
				}
				
				//选定枢纽元，并将枢纽元放于正确位置
				int povit = arr[left];
				int i = left;
				int j = right;
				while(i < j){
					while(i < j && arr[j] >= povit){
						j--;
					}
					arr[i] = arr[j];
					
					while(i < j && arr[i] <= povit){
						i++;
					}
					arr[j] = arr[i];
				}
				
				arr[i] = povit;
				
				//分解为两个子任务
				ForkJoinTask task1 = new ForkJoinTask(left, i - 1, arr);
				ForkJoinTask task2 = new ForkJoinTask(i + 1, right, arr);
				//执行子任务
				invokeAll(task1, task2);			
			}
			
		}
		
		//将QuickSort代理到ForkJoinTask
		public void QuickSort(int[] arr){
			if(arr == null || arr.length == 0){
				return ;
			}
			
			ForkJoinPool forkJoinPool = new ForkJoinPool();
			ForkJoinTask forkJoinTask = new ForkJoinTask(0, arr.length - 1, arr);
			
			forkJoinPool.invoke(forkJoinTask);
		}
	
	}

##3. jUnit验证准确性和效率
	import java.util.Arrays;
	
	import org.junit.Assert;
	import org.junit.Before;
	import org.junit.Test;
	
	public class TestForkJoinQuickSort {
		private int[] arr1;
		private int[] arr2;
	
		@Before
		public void setUp() throws Exception {
			int length = (int) (Math.random() * 100000000);
			arr1 = new int[length];
			arr2 = new int[length];
			
			for(int i = 0; i < length; i++){
				arr2[i] = arr1[i] = (int) (Math.random() * Integer.MAX_VALUE);
			}
		}
	
		@Test
		public void testQuickSort() {
			long begin = System.currentTimeMillis();
			Arrays.sort(arr1);
			System.out.println(System.currentTimeMillis() - begin);
			
			begin = System.currentTimeMillis();
			new ForkJoinQuickSort().QuickSort(arr2);
			System.out.println(System.currentTimeMillis() - begin);
			Assert.assertArrayEquals(arr1, arr2);
		}
	
	}
某次执行效果如下：</br>

![](http://niceaz.com/wp-content/uploads/2016/04/ForkJoinSort.png)</br>
当数据量大的时候，简单的并行的快速算法比经过优化的串行快速算法Arrays.sort算法快很多。


