---
layout: post
title: Heap的实现
categories: [Algorithm]
description: 
keywords: Data Structure, Algorithm
---

##Heap的实现

	public class Heap {
		private int[] arr;
		private int size = 0;
		
		public Heap(int capacity){
			arr = new int[capacity];
		}
		
		public boolean push(int value){
			if(size == arr.length){
				return false;
			}
			
			arr[size++] = value;
			int i = size - 1;
			
			while(i > 0){
				int father = (i - 1) / 2;
				if(arr[father] > value){
					arr[i] = arr[father];
					i = father;
				}else{
					break;
				}
			}
			
			arr[i] = value;
			return true;
		}
		
		private void swap(int[] nums, int i, int j){
			int tmp = nums[i];
			nums[i] = nums[j];
			nums[j] = tmp;
		}
		
		private void adjust(int[] nums, int root, int right){
			int i = root;
			int leftSon = i * 2 + 1;
			int rightSon = i * 2 + 2;
			
			while(i <= right){
				int min = i;
				if(leftSon <= right && arr[leftSon] < arr[min]){
					min = leftSon;
				}
				
				if(rightSon <= right && arr[rightSon] < arr[min]){
					min = rightSon;
				}
				
				if(min == i){
					return ;
				}
				
				swap(nums, i, min);
				i = min;
				leftSon = i * 2 + 1;
				rightSon = i * 2 + 2;
			}
		}
		
		public int pop() throws Exception{
			if(size == 0){
				throw new Exception("No Element");
			}
			
			if(size == 1){
				size--;
				return arr[0];
			}
			
			int ret = arr[0];		
			arr[0] = arr[size - 1];
			
			size--;
			
			adjust(arr, 0, size - 1);
			
			return ret;
		}		
	
	}

