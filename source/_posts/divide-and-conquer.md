---
title: 分治算法理解及其应用
tags: [算法,分治]
author: Mingshan
categories: [算法,分治]
date: 2019-12-24
mathjax: true
---

在维基百科中，关于分治算法（divide-and-conquer）的介绍如下：在计算机科学中，分治法是建基于多项分支递归的一种很重要的算法范式。字面上的解释是“分而治之”，就是把一个复杂的问题分成两个或更多的相同或相似的子问题，直到最后子问题可以简单的直接求解，原问题的解即子问题的解的合并。这个概念非常好理解，一个操作的计算规模很大，直接计算比较困难，如果可以将该问题分解为多个子问题进行计算，并且合并这些计算结果与原计算结果期望一致，那么这样就再好不过了。

单从上述概念来理解，我们不难发现，利用分治算法有一个分解与合并的过程，至于分解后的小规模问题如果解决，可以继续利用分治思想继续进行分解，直至可以直接计算。这就形成了递归。所以利用分治算法求解问题一般有三个步骤：

1. 分解：将原问题分解为若干个规模较小，相对独立，与原问题形式相同的子问题。
2. 解决：若子问题规模较小且易于解决时，则直接解。否则，递归地解决各子问题。
3. 合并：将各子问题的解合并为原问题的解。

<!-- more -->


**通用模型**

```
Divide_and_Conquer(P){
    if(xxx) //递归出口：如果规模足够小，克制直接求解，则开始“治”
        return ADHOC(P); //ADHOC是治理可直接求解子问题的子过程
     
    <divide P into smaller subinstances P1,P2,...Pk>; //将P“分”解为k个子问题
     
    for(int i = 0; i < k; ++i)
        yi = Divide_and_Conquer(Pi); //递归求解各个子问题
     
    return merge(y1, y2, ..., yk); //将各个子问题的解“合”并为原问题的解
}
```


## 减而治之 Decrease and conquer

![image](https://github.com/mstao/static/blob/master/images/decrease-and-conquer.png?raw=true)


## 分而治之 Divide and conquer

![image](https://github.com/mstao/static/blob/master/images/divide-and-conquer.png?raw=true)


## References：

- [分治法 - 维基百科](https://zh.wikipedia.org/wiki/%E5%88%86%E6%B2%BB%E6%B3%95)
- [递归 & 分治](https://oi-wiki.org/basic/divide-and-conquer/)
- [分治算法学习](https://www.cnblogs.com/yuanyb/p/10567023.html)
- [](https://mingshan.fun/2019/01/20/tail-recursion/)
