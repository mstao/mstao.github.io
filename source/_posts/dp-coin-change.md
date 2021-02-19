---
title: 动态规划之零钱兑换
tags: [算法,动态规划]
author: Mingshan
categories: [算法,动态规划]
date: 2021-02-19
---

先来看一下这道题简化的描述：

现有2元，5元，7元三种硬币，假设硬币都足够多，现求解：最少用多少枚上述硬币拼出27块钱？

<!-- more -->

# 递归解法

看到这个问题，我们一下子就可以想到一个树形结构，27块钱可以分别减去上面三种硬币的面额，剩下的值可以继续减去上面三种硬币的面额，直至无法再减，计算出刚好能减完的路线（从顶点到叶子节点）的节点数，最小的即是本题答案的解，图如下所示：

![image](https://github.com/mstao/static/blob/master/images/ds/MinCoin.png?raw=true)

从上面的分析来看，很明显这道题可以用递归来做，而且比较简单，代码如下：

```Java
  public static int solution1(int x) {
    if (x == 0) {
      return 0;
    }

    int res = x + 1;

    if (x >= 2) {
      res = Math.min(solution1(x - 2) + 1, res);
    }

    if (x >= 5) {
      res = Math.min(solution1(x - 5) + 1, res);
    }

    if (x >= 7) {
      res = Math.min(solution1(x - 7) + 1, res);
    }

    return res;
  }
```

找到了递推公式，递归解法是很简单的，不过要注意下返回条件：`if (x == 0) { return 0; }`，可直接拼出来的，返回就好了。

除了递归解法，我们还有其他方式吗？当然有，下面我们动态规划分析下这道题的解题思路。

原题是在leetcode上面，地址：https://leetcode-cn.com/problems/coin-change/

原题描述：

> 给定不同面额的硬币 coins 和一个总金额 amount。编写一个函数来计算可以凑成总金额所需的最少的硬币个数。如果没有任何一种硬币组合能组成总金额，返回 -1。
你可以认为每种硬币的数量是无限的。
 
 
# 动态规划解法

## 确定状态

**确定子问题**

由题可知，需要若干枚硬币来拼出27块钱，且硬币数量最小，假设现在最优解为`A1, A2 ... Ak`， 最优解的最后一枚硬币为Ak，那么剩下的`27-Ak`块钱必然是由`A1, A2 ... Ak-1`枚硬币拼出来的最优解，用的硬币数量也是最小的，如下图所示：

![image](https://github.com/mstao/static/blob/master/images/ds/MinCoin2.png?raw=true)

所以到这里，我们将求解27转化为求解`27-Ak`的子问题，问题规模变小了。

**最后一步**

在上面确定子问题时，我们知道最优解最后一个硬币为`Ak`，但在问题中，我们知道硬币可以是2元，5元，7元三种，所以`Ak`的可能情况也会是这三种，最终需要在这三种拿到最优解。

## 转移方程

确定状态之后，我们还需要将问题抽象一下，用`f(x)`代表用多少枚硬币拼出x的最优解，所以分析上面的，就变成了`f(27) = min{ f(27-2) + 1, f(27-5) + 1, f(27-7) + 1 }`，将27替换成x，转移方程如下：

```
f(x) = min{ f(x-2) + 1, f(x-5)+1, f(x-7)+1 }
```

## 初始条件和边界情况

初始条件和边界情况对动态规划的正确性至关重要，动态规划题目大部分是从小到下计算，此时需要对一些值进行初始化，然后才能进行计算；一些转移方程需要加条件限制才能够成立。对于此题，如果x为0时，f(0) 是多少呢？显然是0，因为没有硬币可以拼出来是0。

边界情况是每次拼时，必须用提供的硬币面额，且刚好能够拼出给定的面额。

## 代码实现

如果上面的分析理解了，代码是很简单的，不过有一个问题需要注意下，如果`f(i-1)` 为`Integer.MAX_VALUE`，那么`f(i-1) + 1` 结果就是负数，会导致计算错误，所以需要先判断`f(i-1)`是否为`Integer.MAX_VALUE`，然后再进行比较，代码如下：

```Java
  public static int solution3(int[] coins, int amount) {
    // f(x) 代表x面额最少用多少枚硬币拼出
    int[] f = new int[amount + 1];
    // 初始化
    f[0] = 0;

    // 数值从小到大进行计算
    // f[x] = min{ f[x - coin1] + 1 , f[x - coin2] + 1, ....}
    // 1,2..27
    for (int i = 1; i <= amount; i++) {
      // 先设置f(i) 为最大值，好进行比较更新
      f[i] = Integer.MAX_VALUE;

      for (int j = 0; j < coins.length; j++) {
        int coin = coins[j];
        // 当前待拼的面额必须大于等于硬币的面额
        // 并且f[x - coin1]不为Integer.MAX_VALUE，注意不能直接用 f[i] > (f[i - coin] + 1) 来判断，
        // 因为 f[i - coin]为Integer.MAX_VALUE， + 1 就是负数了，此时的f[i]计算结果为负数
        if (i >= coin && f[i - coin] != Integer.MAX_VALUE) {
          f[i] = Math.min(f[i - coin] + 1, f[i]);
        }
      }
    }

    // 不能拼出，返回-1
    if (f[amount] == Integer.MAX_VALUE) {
      return -1;
    }

    return f[amount];
  }
```
