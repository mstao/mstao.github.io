---
title: 树状数组BinaryIndexedTree
tags: [数据结构, 树状数组]
author: Mingshan
categories: [数据结构, 树状数组]
date: 2019-11-29
---

第一个看到树状数组（Binary Indexed Tree）这个数据结构时，真的被吸引了，设计真是简洁和优化，属于理论很复杂，但代码实现不复杂的那种算法。该算法多用于高效计算数列的前缀和， 区间和动态单点值的修改。要理解树状数组的工作原理，必须要知道二进制的运算法则，比如 `&`、 `-`、补码和反码等。下面先介绍下二进制的一些简单运算。

<!-- more -->

## 二进制的一些运算

**按位与 &**

我们知道Java中int类型占4字节，也就是32位，现在我们有两个数 10 和 4，按位与的运算法则为`同一得 1`，下面是运算过程：

```
  0000 0000 0000 0000 1100
& 0000 0000 0000 0000 0100
= 0000 0000 0000 0000 0100
```

结果是4。

**求正数的负数 -**

在二进制的世界里，没有减法运算，当做减法运算时，需要将其取反，然后加1，比如-4的二进制的表示，下面是运算过程：

```
   0000 0000 0000 0000 0100
~  1111 1111 1111 1111 1011
+1 1111 1111 1111 1111 1100
```
所以-4的二进制为：`1111 1111 1111 1111 1100`

## 树状数组（Binary Indexed Tree）概念和原理

了解了二进制基本运算，下面来看树状数组。首先看一张我画的图：

![image](https://github.com/mstao/static/blob/master/images/BinaryIndexedTree.png?raw=true)

或者下图

![image](http://community.topcoder.com/i/education/binaryIndexedTrees/bitval.gif)


上图中，我们有两个数组，**A**数组和**C**数组。**A**数组是原值，**C**数组是根据某一规则存的是A数组若干项的和。从上图来看，C数组似乎是呈对称的形态，比如C[8]表示A[1] ~ A[8]的和，而C[4]表示A[1] ~ A[4]的和，所以C[8]又可以表示C[4] + [6] + C[7]，这有什么规律呢？ 下面是C[1] ~ c[8]值计算方式：

- C[1] = A[1];
- C[2] = A[1] + A[2];
- C[3] = A[3];
- C[4] = A[1] + A[2] + A[3] + A[4];
- C[5] = A[5];
- C[6] = A[5] + A[6];
- C[7] = A[7];
- C[8] = A[1] + A[2] + A[3] + A[4] + A[5] + A[6] + A[7] + A[8];

下面是A[1] ~ A[16] 和 C[1] ~ C[16]数组中值的表格：

位置 |  1 |	2 |	3 |	4 | 5 |	6 |	7 |	8 |	9 |	10 | 11 | 12 |	13 | 14 | 15 | 16
---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---
A数组值 | 	1 |	0 |	2 |	1 |	1 |	3 |	0 |	4 |	2 |	5 |	2 |	2 |	3 |	1 |	0 |	2
C数组值   |   1 | 1 |	2 |	4 |	1 |	4 |	0 |	12 | 2 | 7 | 2 | 11 | 3 | 4 | 0 | 29


结合上面而言，如果数组位置是奇数，那么`C[i] = A[i]`；如果是偶数，可以形象地看作二分：`C[i] = C[i / 2] + A[i - (i / 2) + 1] + .... A[i]`。而`C[i / 2]`又可以利用上面的二分继续继续进行分割，直至不可分割。


## 完整代码


```Java
public class BinaryIndexedTree {
  private int length;
  private int[] tree;

  public BinaryIndexedTree(int length) {
    if (length <= 0) {
      throw new IllegalArgumentException("The length [" + length + "] must greater 0.");
    }

    this.length = length;
    tree = new int[length + 1];
  }

  /**
   * 更新
   *
   * @param index 指定位置
   * @param value 值
   */
  public void put(int index, int value) {
    checkPosition(index);

    while (index <= length) {
      tree[index] += value;
      index += lowBit(index);
    }
  }

  /**
   * 计算从1到{@code index}之间的和
   *
   * @param index 指定位置
   * @return 和
   */
  public int sum(int index) {
    checkPosition(index);

    int sum = 0;
    while (index > 0) {
      sum += tree[index];
      index -= lowBit(index);
    }

    return sum;
  }

  /**
   * 算从{@code start}到{@code index}之间的和
   *
   * @param start 起始位置
   * @param end 终止位置
   * @return 值
   */
  public int sum(int start, int end) {
    return sum(end) - sum(start - 1);
  }

  /**
   * 获取
   *
   * @param index 指定位置
   * @return 值
   */
  public int get(int index) {
    checkPosition(index);

    int sum = tree[index];
    int z = index - lowBit(index);
    index--;
    while (index != z) {
      sum -= tree[index];
      index -= lowBit(index);
    }
    return sum;
  }

  /**
   * (二进制)保留最低位的1及其后面的0，高位的1全部变为0，
   * 即得到该数的二进制从右往左第一个非0位所表示的10进制数。
   * 例如：
   * <pre>
   *  原值 0000 0110
   *  取反 1111 1001
   *  +1  1111 1010
   *  &   0000 0111
   *  结果 0000 0010
   * </pre>
   *
   * @param k 待处理的十进制数
   * @return 处理后的十进制数
   */
  private static int lowBit(int k) {
    return k & -k;
  }

  /**
   * 检测指定位置是否越界
   *
   * @param index 指定位置
   */
  private void checkPosition(int index) {
    if (index <= 0) {
      throw new IllegalArgumentException("The index [" + index + "] must greater 0.");
    }
    if (index > this.length) {
      throw new IllegalArgumentException("The index [" + index + "] is out of range.");
    }
  }

  @Override
  public String toString() {
    return Arrays.toString(tree);
  }
}
```



## References：

- [树状数组 - 维基百科](https://zh.wikipedia.org/wiki/%E6%A0%91%E7%8A%B6%E6%95%B0%E7%BB%84)
- [树状数组详解](https://www.cnblogs.com/xenny/p/9739600.html)
- [[数据结构]树状数组的基本应用](https://www.cnblogs.com/cyanigence-oi/p/11708420.html)
