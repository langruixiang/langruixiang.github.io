---
layout: post
title: 并查集
categories: [Algorithm]
description: 并查集数据结构的实现
keywords: Data Structure, Union Find Set
---

## 并查集

## 概述
本文用数组arr实现了并查集：当arr[i] == i 时， i为集合的根节点，集合的标号为i。当arr[i] != i 时，则arr[i] 为i的父节点，沿着父节点向上寻找，直到arr[i] == i，即找到i所属的集合。主要操作如下：

* 初始化：最开始，所有的元素属于不同的集合，所以arr[i] = i
* Find(i): 如果arr[i] == i 则返回i；如果arr[i] != i， 则让i = arr[i], 重复执行该步骤。
* Union(i, j)：分别找到i和j所属的集合的根节点si、sj，如果si == sj，即i，j已经属于同一个集合，直接返回；如果si != sj，则将高度较小的集合根节点(设其是si)指向高度较大的集合的根节点，即arr[si] = sj,这样可以减少集合高度的增长，利于减少Find查找次数。

## 源码

```
public class UnionFindSet {
	int[] arr;
	int[] height;
	
	int size;
	//初始化：每个元素都是独立的集合，集合的高度都为0
	public UnionFindSet(int size){
		this.size = size;
		this.arr = new int[size];
		this.height = new int[size];
		
		for(int i = 0; i < size; i++){
			arr[i] = i;
			height[i] = 0;
		}
	}
	
	//沿着父节点向上寻找，直至arr[i] == i，i即为集合的标号
	public int Find(int num){
		if(num >= size){
			return -1;
		}
		
		int i = num;		
		while(arr[i] != i){
			i = arr[i];
		}
		
		return arr[i];
	}
	
	public boolean Union(int num1, int num2){
		int s1 = Find(num1);
		int s2 = Find(num2);
		
		if(s1 == -1 || s2 == -1){
			return false;
		}
		
		//num1, num2已经属于一个集合
		if(s1 == s2){
			return true;
		}
		
		if(height[s1] < height[s2]){//将高度较小的集合根节点指向高度较大集合的根节点
			arr[s1] = s2;
		}else if(height[s1] > height[s2]){
			arr[s2] = s1;
		}else{//集合高度一样，则随便选择集合的指向，并更新高度
			arr[s1] = s2;
			height[s2]++;
		}
		
		return true;
	}

}
```
