---
title: 树状数组BinaryIndexedTree
tags: [数据结构, 树状数组]
author: Mingshan
categories: [数据结构, 树状数组]
date: 2019-11-29
---


树状数组或二元索引树（英语：Binary Indexed Tree），又以其发明者命名为Fenwick树，最早由Peter M. Fenwick于1994年以A New Data Structure for Cumulative Frequency Tables为题发表在SOFTWARE PRACTICE AND EXPERIENCE。其初衷是解决数据压缩里的累积频率（Cumulative Frequency）的计算问题，现多用于高效计算数列的前缀和， 区间和。 n)时间内支持动态单点值的修改。

<!-- more -->

位置 |  1 |	2 |	3 |	4 | 5 |	6 |	7 |	8 |	9 |	10 | 11 | 12 |	13 | 14 | 15 | 16
---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---


A数组值 | 	1 |	0 |	2 |	1 |	1 |	3 |	0 |	4 |	2 |	5 |	2 |	2 |	3 |	1 |	0 |	2
TREE    |   1 | 1 |	2 |	4 |	1 |	4 |	0 |	12 | 2 | 7 | 2 | 11 | 3 | 4 | 0 | 29

![image](http://community.topcoder.com/i/education/binaryIndexedTrees/bitval.gif)
![image](https://github.com/mstao/static/blob/master/images/BinaryIndexedTree.png?raw=true)


## References：

- [树状数组 - 维基百科](https://zh.wikipedia.org/wiki/%E6%A0%91%E7%8A%B6%E6%95%B0%E7%BB%84)
