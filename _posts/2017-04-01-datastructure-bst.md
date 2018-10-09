---
layout: post
title: BST的递归实现
categories: [Algorithm]
description: 二叉查找树的递归版本实现
keywords: Data Structure, Algorithm, BST
---


## 引言
本文使用递归实现了BST，并测试了代码的正确性

## 插入
比较简单，比较插入值与根节点值的大小。相等，则无需操作；小于则插入根节点左子树；大于则插入根节点右子树。
代码：

```
参数：value, 待插入值
	 root， BST的根节点

返回值：插入后BST的根节点

public TreeNode insert(int value, TreeNode root){
	if(root == null){
		root = new TreeNode(value);
		return root;
	}
	
	if(value == root.value){
		return root;
	}else if(value < root.value){
		root.left = insert(value, root.left);
		return root;
	}else{
		root.right = insert(value, root.right);
		return root;
	}
	
}
```

## 查找
比较简单，与插入类似，直接上代码

```
参数：value, 查找的值
	 root， BST的根节点

返回值：找到则返回对应节点，否则返回null

public TreeNode find(int value, TreeNode root){
	if(root == null){
		return null;
	}
	
	if(value == root.value){
		return root;
	}else if(value < root.value){
		return find(value, root.left);
	}else{
		return find(value, root.right);
	}
}
```

## 删除
相对繁琐一点，先找到被删节点，被删节点没有孩子，直接删除，被删节点只有一个孩子，则返回该节点的孩子。被删节点有两个孩子，将右子树最小节点值覆盖当前节点，删除右子树最小节点。

```	
函数功能：删除BST最小节点
函数:BST根节点

返回值：长度为2的TreeNode数组，数字第0个元素为删除后BST的跟，第1个元素为被删的节点	

private TreeNode[] deleteMin(TreeNode root){
	if(root.left == null){
		return new TreeNode[]{root.right, root};
	}
	
	TreeNode[] res = deleteMin(root.left);
	root.left = res[0];
	
	return new TreeNode[]{root, res[1]};
}

函数功能：删除BST值为value的节点
函数:value, 待删除节点的值
     root, BST根节点

返回值：删除后BST的根
public TreeNode delete(int value, TreeNode root){
	if(root == null){
		return null;
	}
	
	if(value < root.value){
		root.left = delete(value, root.left);
		return root;
	}else if(value > root.value){
		root.right = delete(value, root.right);
		return root;
	}else {
		if(root.left != null && root.right != null){
			TreeNode[] res = deleteMin(root.right);
			root.right = res[0];
			root.value = res[1].value;
			
			return root;
		}else{
			if(root.left == null){
				return root.right;
			}
			
			return root.left;
		}
	}
}
```

## 测试
验证一个树是BST，中序遍历，遍历的值为升序

```
int last = Integer.MIN_VALUE; //用于记录上一次访问的值的大小
private boolean isBST(BST.TreeNode root){
	if(root == null){
		return true;
	}
	
	if(!isBST(root.left)){
		return false;
	}
	
	if(last >= root.value){
		return false;
	}
	
	last = root.value;
	
	return isBST(root.right);		
}
```

## 源码
BST：

```
public class BST {
	public class TreeNode{
		int value;
		TreeNode left;
		TreeNode right;
		
		public TreeNode(int value){
			this.value = value;
		}
	}
	
	public TreeNode insert(int value, TreeNode root){
		if(root == null){
			root = new TreeNode(value);
			return root;
		}
		
		if(value == root.value){
			return root;
		}else if(value < root.value){
			root.left = insert(value, root.left);
			return root;
		}else{
			root.right = insert(value, root.right);
			return root;
		}
		
	}

	public TreeNode find(int value, TreeNode root){
		if(root == null){
			return null;
		}
		
		if(value == root.value){
			return root;
		}else if(value < root.value){
			return find(value, root.left);
		}else{
			return find(value, root.right);
		}
	}
	
	private TreeNode[] deleteMin(TreeNode root){
		if(root.left == null){
			return new TreeNode[]{root.right, root};
		}
		
		TreeNode[] res = deleteMin(root.left);
		root.left = res[0];
		
		return new TreeNode[]{root, res[1]};
	}
	
	public TreeNode delete(int value, TreeNode root){
		if(root == null){
			return null;
		}
		
		if(value < root.value){
			root.left = delete(value, root.left);
			return root;
		}else if(value > root.value){
			root.right = delete(value, root.right);
			return root;
		}else {
			if(root.left != null && root.right != null){
				TreeNode[] res = deleteMin(root.right);
				root.right = res[0];
				root.value = res[1].value;
				
				return root;
			}else{
				if(root.left == null){
					return root.right;
				}
				
				return root.left;
			}
		}
	}

}
```

测试源码：

```
import static org.junit.Assert.*;

import org.junit.Before;
import org.junit.Test;

import junit.framework.Assert;

public class TestBST {
	private int[] nums;
	private int last ;
	
	@Before
	public void setUp() throws Exception {
		int len = (int)(Math.random() * 1000000);
		
		nums = new int[len];
		for(int i = 0; i < len; i++){
			nums[i] = (int)(Math.random() * 1000);
		}
	}
	
	
	
	private boolean isBST(BST.TreeNode root){
		if(root == null){
			return true;
		}
		
		if(!isBST(root.left)){
			return false;
		}
		
		if(last >= root.value){
			return false;
		}
		
		last = root.value;
		
		return isBST(root.right);		
	}

	@Test
	public void test() {
		BST binarySearchTree = new BST();
		
		BST.TreeNode root = null;
		
		for(int i = 0; i < nums.length; i++){
			root = binarySearchTree.insert(nums[i], root);
			
			last = Integer.MIN_VALUE;
			assertTrue(isBST(root));
		}
		
		for(int i = 0; i < nums.length; i++){
			assertTrue(binarySearchTree.find(nums[i], root) != null);
		}
		
		for(int i = 0; i < nums.length; i++){
			root = binarySearchTree.delete(nums[i], root);
			
			last = Integer.MIN_VALUE;
			assertTrue(isBST(root));
		}
	}

}
```


