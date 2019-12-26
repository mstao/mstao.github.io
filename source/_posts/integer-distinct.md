---
title: 有序整数集合去重算法
tags: [算法]
author: Mingshan
categories: [算法]
date: 2019-12-23
mathjax: true
---

最近在看邓俊辉邓公的《数据结构和算法》课程，其中在讲Vector时提到了唯一化算法，也就是我们熟悉的去重算法，该算法实现相当简单，但却很高效。如果我们不借助Java任何已有的数据结构和方法，想对一个集合去重，最低效的算法是遍历其集合，找出每个重复元素存在的区间，每个区间只保留一个元素即可，这就涉及到remove 操作，最坏的情况下，每次都要调用remove方法，时间复杂度累计到$O(2^n)$，显然十分低效。

<!-- more -->

对于上面低效操作，我们可以利用已有的集合空间，结合两个指针，一个指针i一直指向非重复元素的最后一位，一个指针j用来向前探测重复的元素，当j遇到一个新的元素时，就将其位置的元素拷贝到i的下一位，同时i自增，指向元素，然后j跳过该元素的重复区间，继续探测下一个不同的元素，直至列表末尾，最后输出0~i这个区间的元素即可。下面是邓公的PPT演示，相信大家都可以看得懂。如下图。

![image](https://github.com/mstao/static/blob/master/images/distinct.png?raw=true)

课程是用C++写的，我这里用Java重写下，理解了原理，分分钟就可以写得出来。

```Java
/**
* 将已排好序的数列去重
*
* @param source 源
* @return 结果
*/
public static List<Integer> distinct(List<Integer> source) {
    if (source == null || source.isEmpty()) {
      return Collections.emptyList();
    }
    
    int i = 0, j;
    
    for (j = 1; j < source.size(); j++) {
      int next = i + 1;
      if (!source.get(j).equals(source.get(i))) {
        if (!source.get(j).equals(source.get(next))) {
          source.set(next, source.get(j));
        }
        i = next;
      }
    }
    
    return source.subList(0, i + 1);
}
```

References：

- [https://www.bilibili.com/video/av49361421?p=53](清华大学-邓俊辉MOOC数据结构与算法全套)
