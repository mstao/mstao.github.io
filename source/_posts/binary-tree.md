---
title: 二叉树概念和操作
tags: [数据结构, 二叉树]
author: Mingshan
categories: [数据结构, 二叉树]
date: 2019-03-17
---

## 二叉树定义

**二叉树**（Binary Tree）是n（n >= 0）个结点所构成的集合，它或为空树（n=0）；或为非空树，对于非空树$T$：

1. 有且仅有一个称之为根的结点
2. 除根节点以外的其余结点分为两个互不相交的子集$T_1$和$T_2$，分别称为$T$的左子树和右子树，且$T_1$和$T_2$本身又是二叉树

<!-- more -->

## 二叉树性质及存储结构

通过上面的二叉树定义我们知道，二叉树的每个结点最多有两个子结点，同时具有递归性质。递归在二叉树的一些操作中使用非常频繁，对此不熟悉的同学请参考：[递归、尾递归和使用Stream延迟计算优化尾递归](https://mingshan.fun/2019/01/20/tail-recursion/)。接下来我们来看看二叉树的一些概念：

- 结点的高度：结点到叶子结点的最长路径（边数）
- 结点的深度：根结点到这个节点所经历的的边的个数
- 结点的层数：结点的深度 + 1
- 结点的度：结点拥有的子树数称为结点的度
- 树的高度: 根结点的高度

下面是一个二叉树及上面属性的具体值，就不解释了，比较简单。

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/BinaryTree.png?raw=true)




## References：

- [二叉树的各种操作](https://subetter.com/algorithm/various-operations-of-the-binary-tree.html)
- 严蔚敏，《数据结构（C语言）第二版》
