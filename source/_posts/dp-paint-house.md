---
title: 动态规划之房屋染色
tags: [算法,动态规划]
author: Mingshan
categories: [算法,动态规划]
date: 2021-03-07
---

这里有n个房子在一列直线上，现在我们需要给房屋染色，共有k种颜色。每个房屋染不同的颜色费用也不同，你希望每两个相邻的房屋颜色不同

费用通过一个nxk 的矩阵给出，比如cost[0][0]表示房屋0染颜色0的费用，cost[1][2]表示房屋1染颜色2的费用。

样例：

```
输入:
costs = [[14,2,11],[11,14,5],[14,3,10]]
输出: 10
说明:
三个屋子分别使用第1,2,1种颜色，总花费是10。
```

原题链接：https://www.lintcode.com/problem/516/

<!-- more -->

## 确定状态

**最后一步**

这个题每个房子染的房屋颜色有k种选择，假设k=3，颜色为红，绿，蓝。由于相邻的房子颜色不能一样，对于最后一个房子n，如果染成了红色，那么倒数第二个房子n-1只能染成绿色或者蓝色，对于其他两种颜色，也是一样的，所以我们可以知道：

- 最后一个房子（n）为红色，倒数第二个房子（n-1）颜色为绿或者蓝
- 最后一个房子（n）为绿色，倒数第二个房子（n-1）颜色为红或者蓝
- 最后一个房子（n）为蓝色，倒数第二个房子（n-1）颜色为红或者绿

由此可知，当有k种颜色时，最后一栋房子染色就有k种情况，每一种情况对应的倒数第二个房屋染色也是不一样的。

**确定子问题**

由上面分析可知，假设该问题的最优解最小花费为A，最后一栋房子染色为kn，花费为An，我们必然就可以知道倒数第二栋房子染色选择就可以是 k1,k2,k3 ... kn-1。那么去掉最后一栋房子的花费An，前n-1栋房子的花费也必然是最小的，为`A-An`，这样我们就把求解前n栋房子的最小花费转为求解前n-1栋房子的最小花费，问题规模自然变小了。这也是我们要找的子问题。

这个有一个问题要注意下，假设我们知道了前n-1栋房子的最优解，但如果我们不知道第n-1栋房子染了什么颜色，那么第n栋房子染什么颜色我们就不能确定了，如果刚好最后一栋房子与倒数第二栋房子染色一样，且花费最小，那么我们得出的结果就是错的。所以我们必须要记录每个房子染了什么颜色，否则该问题无法用动态规划进行求解。

## 转移方程

根据上面我们确定了问题的解决方式，由于要记录前n栋房子的染色的最小花费且还要记录第n栋房子染了什么颜色，那么我们需要开辟了二维数组来记录这两个场景的数据，所以我们用`f[n][i]` 来表示前n栋房子的最小花费，且第n栋房子的染色方式为`i`。`cost`为题意给定的费用矩阵，转移方程为： 

```
f[n][i] = cost[n-1][i] + min{f[n-1][0], f[n-1][2] ... f[n-1][j]} 且 [j != i]
```

上面转移方式的含义为：如果要求前n栋房子（注意第n栋房子在cost数组里面下标为`n-1`）的最小花费`f[n][i]`，那么我们直接获取第n栋房子染为第i种颜色的花费为`cost[n-1][i]`，不过第n-1栋房子除了第i种房子不能染，其他颜色都可以选择来染，从中选择一个花费最小的，再加上第n栋房子最小花费，即为问题的解。

## 初始条件和边界情况

由于`f[n][i]`代表前n栋房子的染色情况，那么`f[0][i]`代表没有房子，花费必然是0。

## 代码实现

根据上述分析，代码就很简单了，下面是没有经过优化的代码，待会还要优化：

```java
 public static int minCostII(int[][] costs) {
    // write your code here

    if (costs == null || costs.length == 0) {
      return 0;
    }

    // 房子数
    int m = costs.length;
    // 颜色种类数
    int k = costs[0].length;

    // 转移方程, n 为前n房子的最小花费（n从1开始）
    // f[n][i] = [k != i]  const[n-1][i] + min{f[n-1][0], f[n-1][2] ... f[n-1][k]}

    int[][] f = new int[m+1][k];

    for (int i = 1; i <= m; i++) {
      for (int j = 0; j < k; j++) {
        f[i][j] = Integer.MAX_VALUE;

        for (int z = 0; z < k; z++) {
          if (z == j) {
            continue;
          }

          int v = f[i-1][z] + costs[i-1][j];
          if (f[i][j] > v) {
            f[i][j] = v;
          }
        }
      }
    }

    int result = f[m][0];

    for (int i = 1; i < k; i++) {
      if (f[m][i] < result) {
        result = f[m][i];
      }
    }

    return result;
  }
```

## 优化

我们直接来看转移方程：

```
f[n][i] = cost[n-1][i] + min{f[n-1][0], f[n-1][2] ... f[n-1][j]} 且 [j != i]
```

根据题意我们可以知道，颜色有k种，k的规模是没有说的，如果k为很小的数，那么求解`min{f[n-1][0], f[n-1][2] ... f[n-1][j]}` 是比较快的，如果k为很大的数字，第n栋房子每换一种颜色染，那么`min{f[n-1][0], f[n-1][2] ... f[n-1][j]}`都要重新计算一遍，区别就是从该集合中去掉颜色与第n栋染色一样的那一项，我们先来简化一下：

假设现在我们有一个数组`A[2,1,4,5,7,3]`，1就是最应的最小花费，如果1对应的颜色不能选择，那么A的最小值就是2，其余A的最小值都是1，所以我们只需要把A的最小值，次小值都求解出来，问题就解决了。不过怎么求解呢？下面我们用双指针的解法来求解。

首先初始化最小值，次小值（由于用了两个指针，需要初始化两个指针的位置）：

- 如果数组长度为0，没有最小值，更没有次小值；
- 如果数组长度为1，最小值和次小值都是第一位元素；
- 如果数组长度大于等于2，需要比较第一位和第二位元素的大小，然后给最小值，次小值赋上对应的值。

之所以初始化最小值，次小值，是因为我们要用双指针来求解，算法时间复杂度为O(n)，方式如下：

- 从数组第三位开始遍历，如果遍历的值比最小值小，那么就将最小值原来的值记录下来，当当前值赋给最小值
- 判断最小值无更新
    - 如果最小值无更新，且次小值大于当前值，那么就将当前值赋值给次小值
    - 如果最小值有更新，那么直接将最小值原来的值赋给次小值


代码如下：

```java
/**
   * 求解数组m的最小值，次小值
   *
   * @param m
   */
  public static void test(int[] m) {
    if (m == null || m.length == 0) {
      return;
    }

    // 第一小， 第二小
    int min1 = Integer.MAX_VALUE, min2 = Integer.MAX_VALUE;
    // 第一小颜色位置， 第二小颜色位置
    int k1 = 0, k2 = 0;

    if (m.length == 1) {
      min1 = min2 = m[0];
      k1 = k2 = 0;
    }

    if (m.length >= 2) {
      int curr0 = m[0];
      int curr1 = m[1];

      if (curr0 >= curr1) {
        min1 = curr1;
        k1 = 1;
        min2 = curr0;
        k2 = 0;
      } else {
        min1 = curr0;
        k1 = 0;
        min2 = curr1;
        k2 = 1;
      }

      for (int q = 2; q < m.length; q++) {
        int curr = m[q];

        // 选判断是否比次小值小
        int oldMin1 = min1;
        int oldK1 = k1;
        if (curr < min1) {
          min1 = curr;
          k1 = q;
        }

        // 最小值无更新，且min2大于当前值
        if (oldK1 == k1 && min2 > curr) {
          min2 = curr;
          k2 = q;
        }

        // 最小值有更新，那么原来的值就是第二小
        if (oldK1 != k1) {
          min2 = oldMin1;
          k2 = oldK1;
        }
      }
    }

    System.out.println("min1 = " + min1 + ", k1 = " + k1);
    System.out.println("min2 = " + min2 + ", k2 = " + k2);
  }
```

根据上面分析，在求解前n栋房子花费最小值是，我们直接将前n-1栋房子花费的最小值，次小值求出来，然后根据第n栋房子的颜色来进行合适判断即可，完整代码如下：

```java
  public static int minCostII2(int[][] costs) {
    // write your code here

    if (costs == null || costs.length == 0) {
      return 0;
    }

    // 房子数
    int m = costs.length;
    // 颜色种类数
    int k = costs[0].length;

    // 转移方程, n 为前n房子的最小花费（n从1开始）
    // f[n][i] = [j != i]  const[n-1][i] + min{f[n-1][0], f[n-1][2] ... f[n-1][k]}

    int[][] f = new int[m+1][k];

    for (int i = 1; i <= m; i++) {
      // 第一小， 第二小
      int min1 = Integer.MAX_VALUE, min2 = Integer.MAX_VALUE;
      // 第一小颜色位置， 第二小颜色位置
      int k1 = 0, k2 = 0;

      if (k == 1) {
        min1 = min2 = f[i-1][0];
        k1 = k2 = 0;
      }

      if (k >= 2) {
        int curr0 = f[i-1][0];
        int curr1 = f[i-1][1];

        if (curr0 >= curr1) {
          min1 = curr1;
          k1 = 1;
          min2 = curr0;
          k2 = 0;
        } else {
          min1 = curr0;
          k1 = 0;
          min2 = curr1;
          k2 = 1;
        }

        // 算出{f[i-1][0], f[i-1][2] ... f[i-1][k]} 的第一小，记录哪个颜色，第二小，记录哪个颜色，下面可以直接用
        for (int q = 2; q < k; q++) {
          int curr = f[i-1][q];

          // 选判断是否比次小值小
          int oldMin1 = min1;
          int oldK1 = k1;
          if (curr < min1) {
            min1 = curr;
            k1 = q;
          }

          // 最小值无更新，且min2大于当前值
          if (oldK1 == k1 && min2 > curr) {
            min2 = curr;
            k2 = q;
          }

          // 最小值有更新，那么原来的值就是第二小
          if (oldK1 != k1) {
            min2 = oldMin1;
            k2 = oldK1;
          }
        }
      }

      for (int j = 0; j < k; j++) {
        f[i][j] = Integer.MAX_VALUE;

        // 如果当前位置为k1,当前不能取，只能取次小的
        if (j == k1) {
          f[i][j] = min2 + costs[i-1][j];
        } else {
          f[i][j] = min1 + costs[i-1][j];
        }
      }
    }

    int result = f[m][0];

    for (int i = 1; i < k; i++) {
      if (f[m][i] < result) {
        result = f[m][i];
      }
    }

    return result;
  }
```
