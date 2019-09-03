---
title: 平衡二叉树AVL
tags: [数据结构, 二叉树]
author: Mingshan
categories: [数据结构, 二叉树]
date: 2019-08-20
---

在计算机科学中，AVL树是最早被发明的自平衡二叉查找树。在AVL树中，任一节点对应的两棵子树的最大高度差为1，因此它也被称为高度平衡树。查找、插入和删除在平均和最坏情况下的时间复杂度都是 $O(logn)$。所以我们可知，AVL树首先是二叉查找树(BST)，不了解BST的同学可以[了解一下](https://mingshan.fun/2017/12/24/binary-search-tree-structure/)，因为AVL树在添加节点和删除节点时，要保持平衡性质，就需要根据BST的性质来调整。节点的**平衡因子**是它的左子树的高度减去它的右子树的高度（有时相反），带有平衡因子1、0或-1的节点被认为是平衡的。下面是一个AVL树的例子：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/avl/avl.png?raw=true)

<!-- more -->

所以根据以上描述，AVL树有以下性质：

1. 若任一节点的左子树不空，则左子树上所有结点的值均小于它的根结点的值；
2. 若任一节点的右子树不空，则右子树上所有结点的值均大于它的根结点的值；
3. 任意节点的左、右子树也分别为二叉查找树；
4. 没有键值相等的节点；
5. 任一节点对应的两棵子树的最大高度差为1。

## AVL树旋转

如果一棵树不符合AVL的性质，我们该怎么办呢？这就需要动态地调整二叉树，称之为旋转，通过旋转最小失衡树来调整。主要有以下旋转方式：LL，RR，LR，RL。

### LL

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/avl/ll.png?raw=true)

### RR

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/avl/rr.png?raw=true)

### LR

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/avl/lr-1.png?raw=true)

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/avl/lr-2.png?raw=true)

### RL

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/avl/rl-1.png?raw=true)

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/avl/rl-2.png?raw=true)

参考：

- [维基百科-AVL树](https://zh.wikipedia.org/wiki/AVL%E6%A0%91)
- [平衡二叉树,AVL树之图解篇](https://www.cnblogs.com/suimeng/p/4560056.html)
- [AVL](https://www.jianshu.com/p/65c90aa1236d)



