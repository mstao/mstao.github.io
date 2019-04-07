---
title: 线索二叉树(Morris Traversal)
tags: [数据结构, 二叉树]
author: Mingshan
categories: [数据结构, 二叉树]
date: 2019-04-01
---

在前面的[文章中](https://mingshan.fun/2019/03/17/binary-tree/)总结了二叉树的一些操作，提供了二叉树前中后的递归和非递归的实现。在非递归的实现中，基本思想是利用栈来模拟递归调用遍历的过程，本质上和递归实现没有区别，空间复杂度为$O(n)$。是否存在一种算法，它不使用栈也不破坏二叉树结构，但是可以完成对二叉树的遍历？即：

1. 空间复杂度为$O(1)$
2. 二叉树的结构不能被破坏

Morris Traversal 能够解决上面的问题，James H. Morris 提出了二叉树线索化，利用结点的空链域来存储结点的前驱和后继信息。

<!-- more -->

## 线索二叉树结构

那么什么是线索二叉树呢？

- 如果结点有左子树，则其left域指示为其左孩子，否则指示其前驱；
- 如果结点有右子树，则其right域指示为其右孩子，否则指示其后继

对于上述的概念，我们可以用一个比较具体的例子来分析，下面是一个二叉树：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/BinaryTree-demo.png?raw=true)

该二叉树中序遍历的结果为：`4-2-8-5-1-6-3-7`, 上面提到的前驱和后继是针对遍历过程来说的，不同的遍历方式一个结点的前驱和后继可能会不同，前驱和后继被称为`线索`，对二叉树以某种次序遍历使其变为线索二叉树的过程叫做`线索化`。

下面是线索二叉树的结点Node，在决定 left 是指向左孩子还是前驱，right 是指向右孩子还是后继，我们是需要一个区分标志的。因此我们在 Node类中增加`leftThread`和`rightThread`布尔字段，其中：

1. leftThread 为 true 时指向前驱，为 false 时指向该结点的左子树；
2. rightThread 为 true 时指向后继，为 false 时指向该结点的右子树。

```Java
public static class Node<E extends Comparable<E>> {
    E item;
    Node<E> left;
    Node<E> right;
    boolean leftThread;
    boolean rightThread;

    public Node (E item) {
        this.item = item;
    }


    @Override
    public String toString() {
        return "item=" + item + " left="
                + ((left != null) ? left.item : "NULL") + " right=" + ((right != null) ? right.item : "NULL")
                + " leftThread=" + leftThread + " rightThread=" + rightThread;
    }
}
```


针对上面二叉树的中序遍历来说，我们需要先将二叉树线索化，形成线索二叉树：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/ThreadedBinaryTree.png?raw=true)


## 二叉树线索化

了解了二叉树的线索化后，接下来就是用代码来将二叉树线索化，这里我们依然采用中序遍历，并使用递归，代码如下：

```Java
/**
 * 中序遍历线索化
 *
 * @param root 二叉树根结点
 */
public void inThreading(Node root) {
    if (root == null) {
        return;
    }

    // 按照中序遍历方向，先处理左子树
    inThreading(root.left);

    // 判断当前节点的左孩子是否为空，为空的话则当前节点的左孩子指向其前驱
    if (root.left == null) {
        root.leftThread = true;
        root.left = pre; // 指向前驱
    }

    // pre的右孩子为空
    if (pre != null && pre.right == null) {
        pre.rightThread = true;
        pre.right = root; // pre的右孩子指向root（后继）
    }
    
    pre = root; 

    // 右子树线索化
    inThreading(root.right);
}
```

上面代码中，就是根据中序遍历的递归代码改写来的，对于任意一个结点，先访问右子树，接着访问该结点，最后访问右子树。pre代表上次访问的结点（相对于当前结点）。

## 中序遍历

构造完线索二叉树，我们就可以来利用Morris Traversal遍历线索二叉树了，遍历步骤如下：

1. 找到中序遍历的起点
2. 该结点不为空，访问该结点
3. 判断当前结点的是否有后继
   1. 如果有，那么获取其后继结点作为当前结点
   2. 如果没有，说明当前结点有右子树，那么按照中序遍历访问右子树
4. 重复3操作，直至线索二叉树输出完毕

代码如下：

```Java
/**
 * 中序遍历
 *
 * @param root 根结点
 */
public void inOrderNonRec(Node root) {
    if (root == null) {
        return;
    }
    
    // 查找中序遍历的起始节点
    while (root != null && !root.leftThread) {
        root = root.left;
    }
    
    while (root != null) {
        System.out.println(root);
        // 如果右孩子是线索
        if (root.rightThread) {
            root = root.right;
        } else { // 有右孩子
            root = root.right;
            while (root != null && !root.leftThread) {
                root = root.left;
            }
        }
    }
}
```


## References：

- 严蔚敏，《数据结构（C语言）第二版》
- [Threaded binary tree](https://en.wikipedia.org/wiki/Threaded_binary_tree)
- [Morris Traversal方法遍历二叉树（非递归，不用栈，O(1)空间）](https://www.cnblogs.com/AnnieKim/archive/2013/06/15/MorrisTraversal.html)
- http://www.geeksforgeeks.org/inorder-tree-traversal-without-recursion-and-without-stack/
- http://www.geeksforgeeks.org/morris-traversal-for-preorder/
- http://stackoverflow.com/questions/6478063/how-is-the-complexity-of-morris-traversal-on


[<font size=3 color="#409EFF">向本文提出修改或勘误建议</font>](https://github.com/mstao/mstao.github.io/blob/hexo/source/_posts/threaded-binary-tree.md)