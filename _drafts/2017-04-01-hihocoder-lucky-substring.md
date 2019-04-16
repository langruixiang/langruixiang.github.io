---
layout: post
title: HiHoCoder题目链接 1152  Lucky Substrings
categories: [Algorithm]
description: 注意数据规模，数据规模也决定了题目的难度
keywords: Algorithm
---

tips: 注意数据规模，数据规模也决定了题目的难度

## HiHoCoder题目链接 [#1152 : Lucky Substrings](http://hihocoder.com/problemset/problem/1152)

>### 描述
A string s is LUCKY if and only if the number of different characters in s is a fibonacci number. Given a string consisting of only lower case letters, output all its lucky non-empty substrings in lexicographical order. Same substrings should be printed once.
>### 输入
A string consisting no more than 100 lower case letters.
>### 输出
Output the lucky substrings in lexicographical order, one per line. Same substrings should be printed once.
>### 样例输入
    aabcd
>### 样例输出
    a
    aa
    aab
    aabc
    ab
    abc
    b
    bc
    bcd
    c
    cd
    d
    
## 分析
题目限制输入字符串的长度不大于100，大大降低本题的难度，即便用O(n<sup>2</sup>)复杂度的算法也不会TLE，我们用双重循环便可以得到所有的子串，然后很容易得到子串中不同字符的个数，我们用set或者boolean数组存储100以内的斐波那契数，将符合条件的子串加入TreeSet中，由于TreeSet本身就是有序的，所以遍历输出即可。

## 源代码

	import java.io.File;
	import java.io.FileNotFoundException;
	import java.util.HashSet;
	import java.util.Iterator;
	import java.util.Scanner;
	import java.util.Set;
	import java.util.TreeSet;

	public class Main {
		private static int countCharacter(String str){
			Set<Character> set = new HashSet<Character>();
			
			for(int i = 0; i < str.length(); i++){
				set.add(str.charAt(i));
			}
			
			return set.size();
		}
		
	
		public static void main(String[] args) {
			// TODO Auto-generated method stub
			Scanner scanner = null;
			
			scanner = new Scanner(System.in);
		    boolean[] fib = new boolean[101]; //记录100以内的斐波那契数
			fib[1] = true;
			int pre1 = 1;
			int pre2 = 1;
			
			int num = pre1 + pre2;
			while(num <= 100){
				fib[num] = true;
				pre1 = pre2;
				pre2 = num;
				
				num = pre1 + pre2;
			}
			
			String line = scanner.nextLine();
			scanner.close();
			
			Set<String> res = new TreeSet<String>(); //记录结果
			
			for(int i = 0; i < line.length(); i++){
				for(int j = i + 1; j <= line.length(); j++){
					String subString = line.substring(i, j);
					if(fib[countCharacter(subString)]){
						res.add(subString);
					}
				}
			}
			
			Iterator<String> iterator = res.iterator(); //TreeSet有序，遍历输出即可
			while(iterator.hasNext()){
				System.out.println(iterator.next());
			}
		}

	}

