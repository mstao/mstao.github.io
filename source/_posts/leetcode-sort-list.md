---
title: 合并有序数组,链表,二叉树
tags: [算法,LeetCode,链表,分而治之]
author: Mingshan
categories: [算法,LeetCode,链表]
date: 2021-05-11
---

LeetCode一道对链表排序的题：[148.排序链表](https://leetcode-cn.com/problems/sort-list/)，原题如下：
> 你链表的头结点 head ，请将其按 升序 排列并返回 排序后的链表 。
进阶：
你可以在 O(n log n) 时间复杂度和常数级空间复杂度下，对链表进行排序吗？

我们直接整进阶的。

<!-- more -->

## 题目分析

如果做这个题之前知道归并排序和合并两个排序链表两个算法的思路，该题就非常容易O.O。下面说下我的思路：

- 由于是单向链表，我们可以采取分而治之的思想，将一个链表从中间一份为二，分别对这个两个链表排序，然后再将排序好的两个链表合并为一个链表
- 对于上面的两个链表，可以同样采取第一步的策略，出现了递归。
合并两个排序链表也是采用递归，比较简单可以参考代码。

这里主要思考难点是要想到用分而治之的思想，先分再合，思路比较清晰，代码不是很复杂。如下：

```Java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode() {}
 *     ListNode(int val) { this.val = val; }
 *     ListNode(int val, ListNode next) { this.val = val; this.next = next; }
 * }
 */
class Solution {
    public ListNode sortList(ListNode head) {
      if (head == null || head.next == null) {
        return head;
      }

      // 先统计总节点数
      int nodeCount = 0;

      ListNode curr = head;
      while (curr != null) {
        nodeCount++;
        curr = curr.next;
      }

      // 获取中间节点位置
      int mid = nodeCount % 2 == 0 ? nodeCount / 2 : nodeCount / 2 + 1;

      // 找到中间节点
      int count = 1;
      curr = head;
      while (count != mid) {
        count++;
        curr = curr.next;
      }

      // 下一个链表的头结点
      ListNode nextHead = curr.next;
      // 引用置空
      curr.next = null;

      // 原来的头结点到中间位置的链表排序，返回新的头结点
      ListNode listNode1 = sortList(head);
      // 中间到末尾的节点排序，返回新的头结点
      ListNode listNode2 = sortList(nextHead);

      // 合并两个已排序的链表
      return merge(listNode1, listNode2);
    }

  /**
   * 合并两个排序链表
   *
   * @param head
   * @param nextHead
   * @return
   */
  private static ListNode merge(ListNode head, ListNode nextHead) {
    if (head == null) {
      return nextHead;
    }

    if (nextHead == null) {
      return head;
    }

    if (head.val > nextHead.val) {
      nextHead.next = merge(nextHead.next, head);
      return nextHead;
    } else {
      head.next = merge(nextHead, head.next);
      return head;
    }
  }
}
```


