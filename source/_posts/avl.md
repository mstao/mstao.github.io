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

如果一棵树不符合AVL的性质，我们该怎么办呢？这就需要动态地调整二叉树，称之为旋转，通过旋转最小失衡树来调整。在新插入的结点向上查找，以第一个平衡因子的绝对值超过1的结点为根的子树称为最小不平衡子树。对于AVL树来说，失衡的情况可以分如下几种：LL，RR，LR，RL。

### LL

对于任意节点，如果左子树的高度与右子树的高度差大于1，并且子树没有失衡的情况，这种被称为LL型，如下图所示：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/avl/ll.png?raw=true)

此时需要进行一次右旋操作：

1. 以当前根结点(3)的左子树根结点(2)作为根结点；
2. 将原来的根结点(3)作为现在根结点(2)的右子树根结点；
3. 将原来左子树根结点(2)的右子树作为原来根结点的左子树

上面转移的规则其实很好理解，由于左子树较高，所以需要向右侧转移，达到平衡效果，由于AVL树拥有二叉搜索树的特性，左子树的值比右子树的值小，所以当前根结点(3)必大于2的右子树根结点(这里为NIL)，所以3作为2的左子树根结点，那么2的右子树根结点就只能作为3的左子结点了。下面的都是这种规则，不再进行分析了。旋转代码如下：


```Java
/**
 * 右旋，失衡情况对应LL
 *
 * @param node 当前子树根结点
 * @return 旋转后的根结点
 */
public AVLNode<E> rotateRight(AVLNode<E> node) {
    Objects.requireNonNull(node, "node must be not null");

    // 暂存当前节点
    AVLNode<E> originNode = node;
    // 当前节点的左子节点
    AVLNode<E> leftNode = node.left;
    if (leftNode == null)
        return node;
    originNode.left = leftNode.right;
    // 替换根结点
    node = leftNode;
    node.right = originNode;

    return node;
}
```


### RR

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/avl/rr.png?raw=true)

此时需要进行一次左旋操作：
1. 以当前根节点(1)的右子树根结点(2)作为根结点；
2. 将原来的根结点(1)作为现在根结点(2)的左子树根结点；
3. 将原来右子树根结点(2)的左子树作为原来根结点的右子树


```Java
/**
 * 左旋，失衡情况对应RR
 *
 * @param node 当前子树根结点
 * @return 旋转后的根结点
 */
public AVLNode<E> rotateLeft(@NotNull AVLNode<E> node) {
    Objects.requireNonNull(node, "node must be not null");

    // 暂存当前节点
    AVLNode<E> originNode = node;
    // 当前节点的右子结点
    AVLNode<E> rightNode = node.right;
    if (rightNode == null)
        return node;
    // 以当前根节点的右子树根结点作为根结点
    originNode.right = rightNode.left;
    // 替换根结点
    node = rightNode;
    node.left = originNode;

    return node;
}
```


### LR

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/avl/lr-1.png?raw=true)

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/avl/lr-2.png?raw=true)

LL 和 RR在进行一次旋转操作之后，就达到了平衡，

### RL

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/avl/rl-1.png?raw=true)

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/avl/rl-2.png?raw=true)

参考：

- [维基百科-AVL树](https://zh.wikipedia.org/wiki/AVL%E6%A0%91)
- [平衡二叉树,AVL树之图解篇](https://www.cnblogs.com/suimeng/p/4560056.html)
- [AVL](https://www.jianshu.com/p/65c90aa1236d)



