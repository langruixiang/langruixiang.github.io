---
layout: post
title: 二叉树非递归遍历
categories: [Algorithm]
description: 二叉树的前序，中序，后序非递归遍历
keywords: Data Structure, Algorithm, Binary Tree
---

##二叉树非递归遍历

##1. 前序

```
public List<Integer> preorderTraversal(TreeNode root) {
    List<Integer> ret = new ArrayList<>();
    Stack<TreeNode> stack = new Stack<>();
    
    TreeNode ite = root;
    while(ite != null || !stack.isEmpty()){
        if(ite == null){
            ite = stack.pop();
        }else{
            ret.add(ite.val);
            if(ite.right != null){
                stack.push(ite.right);
            }
            ite = ite.left;
        }
    }
    
    return ret;
}
```


##2. 中序

```
public List<Integer> inorderTraversal(TreeNode root) {
    List<Integer> ret = new ArrayList<Integer>();
    Stack<TreeNode> stack = new Stack<TreeNode>();
    
    TreeNode iterator = root;
    while(iterator != null || !stack.isEmpty()){
        if(iterator == null){
            iterator = stack.pop();
            ret.add(iterator.val);
            iterator = iterator.right;
        }else{
            stack.push(iterator);
            iterator = iterator.left;
        }
    }
    
    return ret;
}
```

##3. 后序

```
public List<Integer> postorderTraversal(TreeNode root) {
    List<Integer> ret = new ArrayList<>();
    Stack<TreeNode> stack = new Stack<>();
    
    TreeNode lastNode = new TreeNode(1); //lastNode初始化为任意非树中已有节点
    
    TreeNode ite = root;
    while(ite != null || !stack.isEmpty()){
        if(ite == null){
            ite = stack.pop();
        }else{
            if(lastNode == ite.left || lastNode == ite.right || (ite.left == null && ite.right == null)){ //三个条件不能遗忘
                ret.add(ite.val);
                lastNode = ite;
                ite = null;
            }else{
                stack.push(ite);
                if(ite.right != null){
                    stack.push(ite.right);
                }
                
                ite = ite.left;
            }
        }
    }
    
    return ret;
}
```
