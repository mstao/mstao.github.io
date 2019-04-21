---
title: 从Leetcode的一道LRU算法题说起
tags: [算法, LRU]
author: Mingshan
categories: [算法]
date: 2019-04-21
---

在刷Leetcode遇见一道LRU相关的题，[原题如下](https://leetcode-cn.com/problems/lru-cache/)：

运用你所掌握的数据结构，设计和实现一个  LRU (最近最少使用) 缓存机制。它应该支持以下操作： 获取数据 get 和 写入数据 put 。

<!-- more -->

获取数据 get(key) - 如果密钥 (key) 存在于缓存中，则获取密钥的值（总是正数），否则返回 -1。
写入数据 put(key, value) - 如果密钥不存在，则写入其数据值。当缓存容量达到上限时，它应该在写入新数据之前删除最近最少使用的数据值，从而为新的数据值留出空间。

进阶:

你是否可以在 O(1) 时间复杂度内完成这两种操作？

那么什么是LRU呢？LRU是Least recently used的所写，中文意思是最近最少使用，学过操作系统的同学可能对LRU有些印象，常用于页面置换算法。简单来说，根据数据的历史访问记录来进行淘汰数据，如果一个数据被访问到，那么下次被访问的几率比较高；如果长时间不被访问，就面临着被淘汰的危险。结合上面的算法题，我们可以分析到，假设数据是被存储在队列中的，那么每次访问一个元素，都将其移动到队列的头结点，当队列满时（通常有一个最大容量），移除队列尾部的元素。

了解了LRU的基本概念，那么我们该用什么结构去表示数据命中没命中呢？如果限制在O(1) 时间复杂度来完成命中操作的话，我们就首选不能出现遍历了。学过数据结构的都知道哈希表的查找时间复杂度为 O(1) ，所以我们可以哈希表来表示数据什么时候命中了（key 和 value），用一个队列存储数据的key。

**get方法流程：**

1. 判断key在哈希表中是否命中
    1. 如果命中，将队列中的key移动到队首，返回哈希表中key对应的value；
    2. 如果没有，直接返回-1 

**put方法流程：**

1. 判断key在哈希表中是否命中
   1. 如果命中，代表更新，将队列中的key移动到队首
   2. 如果没有，在队首添加新元素
2. 更新或者新增该key-value到哈希表
3. 检测队列容量，如果容量超过最大限制，移除队列最后一个元素


好了，实现就不说了，代码如下，都是现成的：

```Java
package me.mingshan.algorithm.lru;


import java.util.LinkedHashMap;
import java.util.LinkedList;
import java.util.Map;

/**
 * @author mingshan
 */
public class LRUCache {
    private static final int DEFAULT_CAPACITY = 16;

    private final Map<Integer, Integer> map;
    private final LinkedList<Integer> queue;

    private final int capacity;

    public LRUCache() {
        this(DEFAULT_CAPACITY);
    }

    public LRUCache(int capacity) {
        if (capacity <= 0) {
            throw new IllegalArgumentException("capacity <= 0");
        }
        this.map = new LinkedHashMap<>();
        this.queue = new LinkedList<>();
        this.capacity = capacity;
    }

    public void put(int key, int value) {
        // 首先判断当前数据是否存在
        if (!map.containsKey(key)) {
            queue.addFirst(key);
        } else {
            // 如果是修改，则需要将其移动到头部
            moveToHead(key);
        }

        map.put(key, value);
        // 如果不存在，首先判断队列中是否已经满了
        if (queue.size() > capacity) {
            // 移除队列中最后一个结点
            map.remove(queue.removeLast());
        }
    }

    public int get(int key) {
        if (map.containsKey(key)) {
            // 如果是修改，则需要将其移动到头部
            int value = map.get(key);
            moveToHead(key);
            return value;
        } else {
            return -1;
        }
    }

    private void moveToHead(int key) {
        queue.remove(Integer.valueOf(key));
        queue.addFirst(Integer.valueOf(key));
    }
}
```

后面准备写一个通用的实现，先挖个坑~

## References：

- [146. LRU缓存机制](https://leetcode-cn.com/problems/lru-cache/)
- [Least_recently_used_(LRU)](https://en.wikipedia.org/wiki/Cache_replacement_policies#Least_recently_used_(LRU))