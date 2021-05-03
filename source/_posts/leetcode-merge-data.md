---
title: 合并有序数组,链表,二叉树
tags: [算法,LeetCode,数组,链表,二叉树]
author: Mingshan
categories: [算法,LeetCode]
date: 2021-05-03
---

LeetCode有两道合并数据相关的题，分别是：

- [21.合并两个有序链表](https://leetcode-cn.com/problems/merge-two-sorted-lists/)
- [88.合并两个有序数组](https://leetcode-cn.com/problems/merge-sorted-array)
- [617.合并二叉树](https://leetcode-cn.com/problems/merge-two-binary-trees)

这些题数据结构不同，但是算法的目标大致一致，即将给定的两个相同的数据结构，合并为一个数据结构。其中会用到递归等思想，下面先从最简单的`合并两个有序数组`开始分析吧。

<!-- more -->

# 88.合并两个有序数组

原题：

> 给你两个有序整数数组 nums1 和 nums2，请你将 nums2 合并到 nums1 中，使 nums1 成为一个有序数组。<br/>
初始化 nums1 和 nums2 的元素数量分别为 m 和 n 。你可以假设 nums1 的空间大小等于 m + n，这样它就有足够的空间保存来自 nums2 的元素。

如果研究过归并排序的同学对这个算法肯定不会陌生，因为该排序算法也需要将两个已排序的数组合并为一个数组。根据题意可知道，我们只需要将nums2的数组拷贝到nums1数组中（有足够的的空间），并保持有序性，不需要额外开辟空间。但是如果不用新的数组来做的话，我们就不能从前往后遍历了。

## 开辟新数组解法

我们先看开辟新数组怎么解。此种解法是最好想的，就是首先从前往后同步遍历两个数组，考虑下面两种情况：

- 如果nums1[i] 比nums2[j]的值大，那么将j所处位置数据拷贝到新数组里面，同时j++，i不动；
- 如果nums1[i] 比nums2[j]的值小或相等，那么将i所处位置数据拷贝到新数组里面，同时i++，j不动

为什么要按照上面的逻辑处理呢？因为合并后的数组要保持原来的有序性，所以两个数组之间对应的值也应该进行比较，谁的值小，谁就应该继续前进，相应大的一方原地不动，这个很好理解，因为数组是从大到小排列的，只有小的先处理完，才能处理大的元素。

当上面的操作处理完后，如果两个数组其中有一个剩余，说明里面的元素全部都大于合并后数组里面的元素，我们直接将剩余的元素直接拷贝到合并后数组即可。

上面操作全部完毕，将合并后的数组直接从索引位置0拷贝到nums1即可。

代码如下：

```java
  /**
   * 思路：新建一个数组，然后按照两个数组从前往后的顺序遍历，每次小的数组指针向前移动，当有一个数组遍历完后，
   * 剩下的另一个数组的元素直接拷贝到目标数组即可，注意
   *
   * @param nums1
   * @param m
   * @param nums2
   * @param n
   */
  public static void merge(int[] nums1, int m, int[] nums2, int n) {
    int last = 0;

    int[] temp = nums1.clone();

    int i = 0;
    int j = 0;
    while (i <= m -1 && j <= n -1) {
      if (nums1[i] > nums2[j]) {
        temp[last++] = nums2[j++];
      } else {
        temp[last++] = nums1[i++];
      }
    }

    // nums1数组原数据有剩余
    while (i <= m - 1) {
      temp[last++] = nums1[i++];
    }

    // nums2数组原数据有剩余
    while (j <= n - 1) {
      temp[last++] = nums2[j++];
    }

    System.arraycopy(temp, 0, nums1, 0, m + n);
  }
```

## 不需要新数组解法

上面的解法开辟了一个新数组，空间复杂度为O(m+n)，算法比较好理解。但nums1的数组长度足够容纳两个数组的元素，所以这个开辟新数组有点多余，如何做到空间复杂度为O(1)呢？首先一点要明确，就是两个数组都是从小到大排列。思路如下：

初始i指向num1最后一位元素末尾（m-1），j指向num2的末尾（n-1）

- 两个数组从后往前遍历，谁大谁放到num1的后面，然后指针递减
- 如果遍历完后发现j不等于0，说明num2剩余的值比num1所有的值都要小，直接将剩余0~j的元素拷贝到数组num1的0~j位置即可

代码如下：


```Java
  /**
   * 不需要额外数组，从后往前遍历，如果遍历结束，nums2的指针没有置为0，那么直接将剩余0~j的元素拷贝到数组num1的0~j位置即可
   *
   * @param nums1
   * @param m
   * @param nums2
   * @param n
   */
  public static void merge2(int[] nums1, int m, int[] nums2, int n) {
    if (nums2 == null || nums2.length == 0) {
      return;
    }

    int i = m == 0 ? 0 : m - 1;
    int j = n - 1;

    int currIndex = m + n - 1;

    // 同步比较
    while (i >= 0 && j >= 0) {
      if (nums1[i] > nums2[j]) {
        nums1[currIndex] = nums1[i];
        i--;
      } else {
        nums1[currIndex] = nums2[j];
        j--;
      }

      currIndex--;
    }

    // 如果有nums2的j指针没有到0，那么将0~j的全部拷贝到nums1的前0~j位
    while (j >= 0) {
      nums1[j] = nums2[j];
      j--;
    }
  }
```

# 21.合并两个有序链表

原题：

> 将两个升序链表合并为一个新的 升序 链表并返回。新链表是通过拼接给定的两个链表的所有节点组成的。

这个题和上面合并升序数组类似，只是数据结构是单向链表。

## 从前往后迭代解法

根据上面的数组的迭代解法，我们同样可以用此思想来处理两个链表，算法流程不再赘述，代码如下：

```Java
  /**
   * 依次比较两个链表的值，谁的值小，谁的指针向前，最后将某一个链表剩余的元素全部拷贝过去即可
   *
   * @param l1
   * @param l2
   * @return
   */
  public static ListNode mergeTwoLists(ListNode l1, ListNode l2) {
    if (l1 == null) {
      return l2;
    }

    if (l2 == null) {
      return l1;
    }

    ListNode next = null;
    ListNode head = null;

    ListNode l1Next = l1;
    ListNode l2Next = l2;

    while (l1Next != null && l2Next != null) {
      if (l1Next.val > l2Next.val) {
        if (next == null) {
          next = new ListNode(l2Next.val);
        } else {
          next.next = new ListNode(l2Next.val);
        }

        // 谁的值小，谁继续前进
        l2Next = l2Next.next;
      } else {
        if (next == null) {
          next = new ListNode(l1Next.val);
        } else {
          next.next = new ListNode(l1Next.val);
        }

        // 谁的值小，谁继续前进
        l1Next = l1Next.next;
      }

      if (head == null) {
        head = next;
      }

      if (next.next != null) {
        next = next.next;
      }
    }

    while (l1Next != null) {
      next.next = new ListNode(l1Next.val);
      l1Next = l1Next.next;

      next = next.next;
    }

    while (l2Next != null) {
      next.next = new ListNode(l2Next.val);
      l2Next = l2Next.next;

      next = next.next;
    }

    return head;
  }
```

## 递归解法

思路：比较两个链表的当前节点，如果l1的节点小于l2的节点，那么我们需要合并l1.next 和l2，两个合并之后再作为一个链表，且l1.next指向该链表，代码如下：

```Java
public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
  if (l1 == null) {
    return l2;
  }

  if (l2 == null) {
    return l1;
  }

  if (l1.val < l2.val) {
    l1.next = mergeTwoLists(l1.next, l2);
    return l1;
  } else {
    l2.next = mergeTwoLists(l1, l2.next);
    return l2;
  }
}
```

# 617.合并二叉树

原题：

> 给定两个二叉树，想象当你将它们中的一个覆盖到另一个上时，两个二叉树的一些节点便会重叠。<br/>
你需要将他们合并为一个新的二叉树。合并的规则是如果两个节点重叠，那么将他们的值相加作为节点合并后的新值，否则不为 NULL 的节点将直接作为新二叉树的节点。

这个二叉树的合并和链表的合并有点类似，只不过相当应的节点值相加，然后再处理左右子树，思想是一致的，代码如下：

```Java
public TreeNode mergeTrees(TreeNode root1, TreeNode root2) {
  if (root1 == null) {
    return root2;
  }

  if (root2 == null) {
    return root1;
  }

  root1.val = root1.val + root2.val;

  root1.left = mergeTrees(root1.left, root2.left);
  root1.right = mergeTrees(root1.right, root2.right);
  return root1;
}
```
