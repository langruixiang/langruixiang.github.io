---
layout: post
title: HiHoCoder 题目链接 HiHoCoder Week 78
categories: [Algorithm]
description: Trie的应用和实现
keywords: Algorithm
---

### HiHoCoder 题目链接：[HiHoCoder Week 78](http://hihocoder.com/contest/hiho78/problem/1)
>### 描述
<a href="http://i4.tietuku.com/8e622dec30fdfc71.png" title="点击显示原始图片"><img src="http://i4.tietuku.com/8e622dec30fdfc71t.jpg"></a></br>
Query auto-completion(QAC) is widely used in many search applications. The basic idea is that when you type some string s in the search box several high-frequency queries which have s as a prefix are suggested. We say string s1 has string s2 as a prefix if and only if the first |s2| characters of s1 are the same as s2 where |s2| is the length of s2.</br>
These days Little Hi has been working on a way to improve the QAC performance. He collected N high-frequency queries. We say a string s is a proper prefix if there are no more than 5 collected queries have s as a prefix. A string s is a shortest proper prefix if s is a proper prefix and all the prefixes of s(except for s itself) are not proper prefixes. Little Hi wants to know the number of shortest proper prefixes given N collected queries.

>Hint: the 4 shortest proper prefixes for Sample Input are "ab", "bb", "bc" and "be". Empty string "" is not counted as a proper prefix even if N <= 5.
>### 输入
The first line contains N(N <= 10000), the number of collected queries.</br>
The following N lines each contain a query.</br>
Each query contains only lowercase letters 'a'-'z'.</br>
The total length of all queries are no more than 2000000.</br>
Input may contain identical queries. Count them separately when you calculate the number of queries that have some string as a prefix.</br>
>### 输出
Output the number of shortest proper prefixes.
>### 样例输入
    12
    a
    ab
    abc
    abcde
    abcde
    abcba
    bcd
    bcde
    bcbbd
    bcac
    bee
    bbb
>### 样例输出
    4

## 分析
使用前缀树，插入单词的同时统计经过该节点的单词的个数，BFS或者DFS Trie树：

- 如果当前节点单词个数不大于5，则计数器加一，并且该节点的子节点一定不是shortest proper prefix（是proper prefix）,所以,不用访问其子节点

- 如果当前节点单词个数大于5，则访问其子节点

***tips:*** HiHoCoder本题Java编写的DFS只能通过90%的数据，BFS可以通过100%

##源代码：

	import java.io.*;
	import java.util.*;
	
	public class Main {
		private static class TrieNode{ //Trie树节点
			int count = 0;
			TrieNode[] sons = new TrieNode[26];
		}
		
		private static TrieNode root = new TrieNode();
		private static int ret = 0;
		
		private static void insert(TrieNode root,String word){ //插入单词，并统计单词频数
			TrieNode iterator = root;
			for(int i = 0; i < word.length(); i++){
				iterator.count++;
				int index = word.charAt(i) - 'a';
				if(iterator.sons[index] == null){
					TrieNode tmp = new TrieNode();
					iterator.sons[index] = tmp;
					iterator = tmp;
				}else{
					iterator = iterator.sons[index];
				}
			}
			
			iterator.count++;
		}
		
		private static void count(TrieNode root){	
	//		if(root.count <= 5){  //递归的DFS
	//			ret++;
	//		}else{
	//			for(int i = 0; i < 26; i++){
	//				if(root.sons[i] != null){
	//					count(root.sons[i]);
	//				}
	//			}
	//		}		
			
			Queue<TrieNode> queue = new LinkedList<TrieNode>(); //BFS
			queue.add(root);
			while(!queue.isEmpty()){
				TrieNode tmp = queue.poll();
				if(tmp.count <= 5){
					ret++;
				}else{
					for(int i = 0; i < 26; i++){
						if(tmp.sons[i] != null){
							queue.add(tmp.sons[i]);
						}
					}
				}
			}
		}	
	
		public static void main(String[] args) {
			// TODO Auto-generated method stub
			Scanner scanner = null;
			
			scanner = new Scanner(System.in);
			int N = scanner.nextInt();
			scanner.nextLine();
			while(N-- > 0){
				String word = scanner.nextLine();
				insert(root, word);
			}
			
			if(root.count <= 5){
				System.out.println("0");
			}else{
				count(root);
				System.out.println(ret);
			}
	
		}
	
	}


