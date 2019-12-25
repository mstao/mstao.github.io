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

在看邓公数据结构和算法课程时，提到了**减而治之**（减治）和**分而治之**（分治）。两者都是将问题规模缩小，但缩小的方式是不同的，适用场景也是不同的，这里结合教材来总结下这两者异同点。

<!-- more -->

## 减而治之 Decrease and conquer

所谓**减而治之**，是为求解一个大规模的问题，可以将其划分为两个子问题：其一**平凡**，另一个规模缩减，分别求解子问题，由子问题的解，得到原问题的解。如下图所示：

![image](https://github.com/mstao/static/blob/master/images/decrease-and-conquer.png?raw=true)

上面提到其一为**平凡问题**，这是个什么意思呢？从上图的意思来看，这个平凡问题不需要再进行分解了，可以直接求解。子问题可以继续进行减而治之，分解为更小的子问题与平凡问题，直至子问题无须再继续分解，这就是遇到了递归的终止条件。最终将子问题的解合并，即为原问题的解。

从减而治之的流程来看，这种递归似乎是线性的，即只在一条路上进行递归，这种也被称为**线性递归（linear recursion）**。正因为有此特性，减而治之对求解此类问题才如此犀利。如果有同学知道尾递归，那么可以形象地理解这个过程。下面看课程上提及的两道题。

**对于给定整数数组，利用减而治之进行求和**。对于此题，我们可以将整个计算任务分解为 [1 ~ n-1] 之间求和 与 最后以后之间的和， 对于[1 ~ n-1] 之间的求和，重复上面操作，当n 等于 0时，求出的值为0，这是该递归的终止条件。递归公式为：`sum(n) = sum(n - 1) + n.value`，代码很简单，如下所示：  

```Java
public static int sum(int[] source, int n) {
    return n < 1 ? 0 : sum(source, n - 1) + source[n - 1];
}
```

另一个问题是**给定一个数组，要求将数组反转**，这个问题也是典型的减而治之问题，先调换首尾两者的值，然后向中间进行递归。递归公式为： 
```
reverse(low, high) = {
  swap(low, high) 
  reverse(low + i, hi -1) 
}
```

代码很简单，不多介绍，如下：

```Java
private static void reverse(int[] a, int low, int hi) {
    if (low < hi) {
      swap(a, low, hi);
      reverse(a, low + 1, hi - 1);
    }
}

private static void swap(int[] source, int a, int b) {
    int temp = source[a];
    source[a] = source[b];
    source[b] = temp;
}
```

## 分而治之 Divide and conquer

所谓**分而治之**，是为求解一个大规模问题，可以将其划分为若干（通常两个）子问题，规模大体相当，分别求解子问题，由子问题的解，得到原问题的解。如下图所示：

![image](https://github.com/mstao/static/blob/master/images/divide-and-conquer.png?raw=true)

注意这里与**减而治之**不同的是，分解后的子问题都可以继续分解，相当于有多路递归情况，也被称为**多路递归（multi-way recursion）**。由于出现多路递归，有可能出现子问题的解被重复计算，如果计算规模很大的话，重复计算的耗时是非常庞大的，在写分治算法时应该极力避免此情况的发生。

如果要使用分治来解决问题，总结起来需要满足以下条件：

1. 该问题的规模缩小到一定程度就可以容易的解决。
2. 该问题可以分解为若干个规模较小的相同问题，即该问题具有**最优子结构**的性质。
3. 利用该问题分解出的子问题的解可以合并为该问题的解(**关键性质**)
4. 该问题所分解出的各个子问题最好是相互独立的，即子问题之间不包含公共的子问题。

第四条就是涉及到刚才提到的重复计算问题，稍不注意会使算法性能急剧下降。解决方案是将子问题的解进行缓存，或者转化为动态规划。

有了上面的理论基础，我们可以抽象出分治的通用模型，如下：

```
divide_and_conquer(problem) {
    if(|problem| < n) adhoc(problem);     /*解决小规模的问题*/
    divide Problem into smaller subinstances P1,P2,....Pk;  /*分解问题*/
    for (i = 1; i<= k(子问题个数); i++) {
        result[i] = divide_and_conquer(P[i]);   /*递归的解各个子问题*/
    }
    return merge(result[1]...result[k]);    /*将各个子问题的解合并为原问题的解*/
}
```

下面我们来几道分治的典型例题。

**利用分治求和**

上面我们利用了减而治之来对数组的元素进行求和，那么可不可以利用分治来进行求和呢？其实我们仔细分析下，一个数组可以一分为二，对这两部分进行求和，结果必然是原数组的和，而且这两部分没有交叉，不包含公共子问题。对两个部分继续分解，直至出现递归条件，那就是最后的数组只有一个元素，该数组的和，就是它自身，最后求出原问题的解。我们从分治的分治合三个部分来看下求和过程：

- **分**：给定一个整数集合 $A = {a[l], ..., a[n]}$，将其从中间一分为二，直至不可再分；
- **治**：通过递归对左右两部分分别求和，当数组只剩下一个元素时，其和就是自己的值；
- **合**：将子问题的和相加，就是原问题的解。

代码是很简单的，如下所示：

```Java
public static int sum(int[] source, int left, int right) {
    if (left == right) {
        return source[left];
    }

    int mid = (left + right) / 2;
    return sum(source, left, mid) + sum(source, mid + 1, right);
}
```

**快速排序**

快速排序想必大家都不陌生了，大学学习排序算法时必然会讲到快速排序。该排序算法思想比较好理解。首先有两个哨兵，初始情况下，哨兵i在数组的最左端，哨兵j在最右端，然后选择一个基准值（默认左边的第一位），哨兵i从左往右进行探测，找到第一个比基准值大的元素，哨兵j从右往左进行探测，找到第一个比基准值小的元素，然后两者交换位置，接着哨兵i，j继续朝着自己的方向进行探测，交换数据，直至两者相遇，交换相遇位置与基准位置的值，现在两者相遇的位置左边必然都比基准值小，右边都比基准值大。下面就好办了，基准值左边和右边各自重复以上步骤，那么什么时候递归结束呢？如果分解后数组只剩下一个元素，那么就不需要再进行排序了，这是递归的终止条件。

了解了快排的流程，那么它是如何利用了分治的呢？我们从分治的分治合三部分来整体分析下快排思想：

- **分**：给定一个整数集合 $A = {a[l], ..., a[n]}$，在`A[low..high]`中任选一个记录作为基准(Pivot)，以此基准将当前无序区划分为左、右两个较小的子区间`A[low..pivotpos-1]`和`A[pivotpos+1..high]`，并使左边子区间中所有记录的关键字均小于等于基准值，右边的子区间中所有记录的关键字均大于等于基准值，而基准值则位于正确的位置(pivotpos)上，它无须参加后续的排序；
- **治**：通过递归调用快速排序对左、右子区间`A[low..pivotpos-1]`和`A[pivotpos+1..high]`快速排序，当区间只剩下一个元素时，递归终止；
- **合**：　因为当"治"步骤中的两个递归调用结束时，其左、右两个子区间已有序。对快速排序而言，"组合"步骤无须做什么，可看作是空操作。

总结来说，快排思想就是 **从待排序数组中选一个值作为基准，使基准左边的所有数据都小于或等于该基准值，枢轴右边的数据都大于或等于该基准值，再用递归对这两个子数组进行快速排序，最终使得整个数组有序。**

了解完思想，代码其实非常简单明了，下面是Java实现的快排，基准值默认为当前数组的第一个元素：

```Java
  public static void sort(int[] source, int low, int high) {
    if (low < high) {
      int i = low;
      int j = high;
      // 找到基准值
      int base = source[low];

      // 两者不想遇
      while (i != j) {
        // 哨兵j从右往左走，找到比基准值第一个小的元素
        while (i < j && source[j] >= base) {
          j--;
        }

        // 哨兵i从右往左走，找到比基准值第一个大的元素
        while (i < j && source[i] <= base) {
          i++;
        }

        // 交换哨兵i，j的数据
        if (i < j) {
          swap(source, i, j);
        }
      }

      // 两个哨兵相遇后，交换基准与当前位置的数据
      swap(source, low, i);

      // 当前位置左边的数都比基准小，递归
      sort(source, low, i - 1);
      // 当前位置右边的数都比基准大，递归
      sort(source, i + 1, high);
    }
  }

  /**
   * 交换a,b两个位置的元素
   *
   * @param a 位置a
   * @param b 位置b
   */
  public static void swap(int[] source, int a, int b) {
    int temp = source[a];
    source[a] = source[b];
    source[b] = temp;
  }
```

**归并排序**

上面快速排序由于要选择一个基准值，基准值选择的不一样，时间复杂度也不一样，是一种不稳定的排序算法，有没有一种算法，不用选择基准值，并且使用分治算法，而且还是稳定的呢？这就是我们要谈到的归并排序算法了。该排序算法思想十分简单，**将一个数组从中间一份为而，分别对这两部分进行排序，然后将这已经排好序的两部分进行合并，递归以上步骤，结果原数组就是有序的了**。这里和快速排序有点非常不一样的地方，就是归并排序需要合并，给定两个数组，这两个数组都是有序的，需要将这两个数组合并为一个数组，且合并后是有序的。整个过程如下图所示：

<!--![image](https://img2018.cnblogs.com/blog/1458608/201903/1458608-20190320191753712-2113691492.png)-->

- **分**：给定一个整数集合 $A = {a[l], ..., a[n]}$，将其从中间一份为二；
- **治**：递归“分”的过程，直至当前数组只有一个元素，此时默认是有序的；
- **合**：合并上一层递归分解的两个数组，直至整个数组都是有序的。

“分”和“治”的过程是十分简单明了的，但“合”的过程相当于另一个算法，就是如何将两个有序数组合并为一个数组，这个在leetcode是一个problem，地址为：[88. 合并两个有序数组](https://leetcode-cn.com/problems/merge-sorted-array)，有兴趣的可以去尝试一下。这里给出利用一个temp数据来做数据中转，主要思路是：首先比较两个数组的元素，谁小就把谁先放到temp数组中，比较完毕后，两个数组可能有一个数组中的数据还没有移动到temp数组（两个数组长度不一致），不过剩下的部分不要担心，我们直接将其拷贝放到temp数组的后面就可以了，因为这部分的值必然是大于temp数组中已有的值的。整个排序算法的实现如下：

```Java
public static void sort(int[] source, int low, int high, int[] temp) {
    if (low < high) {
      int mid = (low + high) / 2;
      sort(source, low, mid, temp);
      sort(source, mid + 1, high, temp);
      merge(source, low, mid, high, temp);
    }
  }

  /**
   * 合并两个有序数组，需要辅助数组
   *
   * 两个数组的元素分别为：
   * source[low] ~ source[mid] 升序
   * source[low + 1] ~ source[high] 升序
   *
   * @param source 原数组
   * @param low 低位
   * @param mid 低位和高位的中间位置
   * @param high 高位
   * @param temp 辅助数组
   */
  private static void merge(int[] source, int low, int mid, int high, int[] temp) {
    int i = low;
    int j = mid + 1;
    int k = 0;

    // 将两个数组的元素一次比较，记录在temp 数组里面
    while (i <= mid && j <= high) {
      if (source[i] > source[j]) {
        temp[k++] = source[j++];
      } else {
        temp[k++] = source[i++];
      }
    }

    // 如果左半边数组还有剩余，直接追加到temp 数组里面
    while (i <= mid) {
      temp[k++] = source[i++];
    }

    // 如果右半边数组还有剩余，直接追加到temp 数组里面
    while (j <= high) {
      temp[k++] = source[j++];
    }

    // 将temp数组的元素移动原数组
//    for (i = 0; i < k; i++) {
//      source[low + i] = temp[i];
//    }
    // 使用内部数组copy，提高性能
    System.arraycopy(temp, 0, source, low, high - low + 1);
  }
```



参考：

- [分治法 - 维基百科](https://zh.wikipedia.org/wiki/%E5%88%86%E6%B2%BB%E6%B3%95)
- [递归 & 分治](https://oi-wiki.org/basic/divide-and-conquer/)
- [分治算法学习](https://www.cnblogs.com/yuanyb/p/10567023.html)
- [递归、尾递归和使用Stream延迟计算优化尾递归](https://mingshan.fun/2019/01/20/tail-recursion/)
- [递归问题（邓公数据结构1.4节笔记）](https://juejin.im/post/5b9f8ffae51d450e83776059)
- [数据结构MOOC](https://dsa.cs.tsinghua.edu.cn/~deng/ds/mooc/)
- [算法设计与分析学习笔记（一）：分治法](https://www.zybuluo.com/anboqing/note/53145)
- [【啊哈！算法】算法3：最常用的排序——快速排序](https://bbs.codeaha.com/forum.php?mod=viewthread&tid=4419&highlight=%BF%EC%CB%D9%C5%C5%D0%F2)
- [快速排序](https://www.cnblogs.com/lisijie/p/7259592.html)
