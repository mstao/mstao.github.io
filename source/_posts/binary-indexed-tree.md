---
title: 树状数组BinaryIndexedTree
tags: [数据结构, 树状数组]
author: Mingshan
categories: [数据结构, 树状数组]
date: 2019-11-29
mathjax: true
---


第一个看到树状数组（Binary Indexed Tree）这个数据结构时，真的被吸引了，设计真是简洁，属于理论很复杂，但实现不复杂的那种算法。该算法多用于高效计算数列的前缀和， 区间和动态单点值的修改。要理解树状数组的工作原理，必须要知道二进制的运算法则，比如 `&`、 `-`、补码和反码等。下面先介绍下二进制的一些简单运算。

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


上图中，我们有两个数组，**A**数组和**C**数组。**A**数组是原值，**C**数组是根据某一规则存的是A数组若干项的和。从上图来看，C数组似乎是呈对称的形态，比如C[8]表示A[1] ~ A[8]的和，而C[4]表示A[1] ~ A[4]的和，所以C[8]又可以表示C[4] + [6] + C[7]。一个C数组的元素只有一个父结点，但却有好多子节点，可以形象地理解为一个C数组元素管着一片区域，怎么去知道一个元素到底在管着哪些A数组的元素呢？ 下面是C[1] ~ c[8]值计算方式：

- C[1] = A[1];
- C[2] = A[1] + A[2];
- C[3] = A[3];
- C[4] = A[1] + A[2] + A[3] + A[4];
- C[5] = A[5];
- C[6] = A[5] + A[6];
- C[7] = A[7];
- C[8] = A[1] + A[2] + A[3] + A[4] + A[5] + A[6] + A[7] + A[8];

我们把上面索引的十进制，全部换成二进制，如下：

- C[1] = C[0001] = A[0001];
- C[2] = C[0010] = A[0001]+A[0010];
- C[3] = C[0011] = A[0011];
- C[4] = C[0100] = A[0001]+A[0010]+A[0011]+A[0100];
- C[5] = C[0101] = A[0101];
- C[6] = C[0110] = A[0101]+A[0110];
- C[7] = C[0111] = A[0111];
- C[8] = C[1000] = A[0001]+A[0010]+A[0011]+A[0100]+A[0101]+A[0110]+A[0111]+A[1000];


仔细观察上面的二进制表示，我们可以发现，**C[i]管的范围就是这个i二进制表示数的从低位到高位第一个为1的位置，和其低位所有二进制表示情况A数组元素之和**。什么意思？比如`C[6]=C[0110]`，它管的范围就是`0001 ~ 0010`。很神奇对不对？估计第一个想出树状数组的人是一个喜欢瞎琢磨的天才，这个规律不从二进制来看是不容易被发现的。那么问题来了，随便给定一个数，我怎么知道从低位起第一个为1的数怎么表示？放心，这个前辈们已经总结出来了，这个操作有个名字叫做lowbit，计算方式为`lowbit(i) = i&(-i)`， 计算该数的二进制从右往左第一个非0位所表示的10进制数，如下所示：

```
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
```


再从图片宏观来看，我们发现整个结构似乎有点二分的味道，具有某种对称性。如果数组位置是奇数，那么`C[i] = A[i]`；如果是偶数，可以形象地看作二分：`C[i] = C[i / 2] + A[i - (i / 2) + 1] + ....  + A[i]`。而`C[i / 2]`又可以利用上面的二分继续继续进行分割，直至不可分割。上面可以推导出如下公式：$C[i] = A[i - 2^k +1] + A[i - 2^k +2] + ... + A[i]$ ，其中k为i的二进制中从最低位到高位连续零的长度。其中这个$2^k$就是上面提到的lowbit(i)。所以我们就知道即C[i]数组中保存的就是数组A的区间`[i-lowbit(i)+1, i]`中所有数的和，公式如下：

$$C[x] = \sum_{i=x-lowbit(x)+1}^x{A[i]}$$

除此之外，我们还可以根据上述的二进制列表得到以下结论：

- C[i]保存的是以它为根的子树中所有叶节点的和。
- C[i]的子节点数量=lowbit(i)
- 除根结点，C[i]的父节点为C[x+lowbit(i)]

## 修改

了解了树状数组中C数组和A数组的关系后，没有提到的一点是，在实际编码中，是没有A数组的，只有C数组，数据是保存在C数组的，但逻辑上的操作是针对A数组，比如获取和更新某个索引位置的元素。现在修改某个位置的元素，需要考虑哪些东西呢？

比如我们要修改A[1]的元素，假设现在数组长度为N，因为C[1] ~ C[N]元素里面有些位置是包含A[1]的值的，所以我们只要找到其中的元素即可，从图中可以观察出，C[1] C[2] C[4] C[8] C[16]这几个包含A[1]的值，这有什么规律呢？我们发现这几个索引是2的N次方，这就相当于把最后的1一直左移，遇到前面也是1的就也消去它再往前走。由于我们知道lowbit的操作就是只保留二进制最后一位为1的数，其实这个问题就转化为，如何去寻找C数组某个位置的父结点呢？上面总结过：**C[i]的父节点为C[x+lowbit(i)]**，所以每次向上层寻找时，只需当前索引i加上lowbit(i)就行了，代码如下：

```Java
while (index <= length) {
  tree[index] += value;
  index += lowBit(index);
}
```

## 查询

现在想直接获取某个索引A数组的值，该怎么做呢？比如我想获取A[4]的值，我们需要将目光转移至C[4]，而C[4] 是由 C[2] + A[3] + A[4]的值之和，所以需要用C[4] - C[2] - A[3]

## 求和

如果想计算从索引1到指定位置i之间的元素和，该怎么做呢？比如A[1] ~ A[6]的和，从图上看出，我们只需要计算出C[4] + C[6] 即可，有没有发现这个查找路径有点像单点更新的逆过程？确实是这样，我们只需要找子元素索引位置为`index -= lowBit(index)`，一直递减到1，就能把前缀和给求出来啦：


```Java
int sum = 0;
while (index > 0) {
  sum += tree[index];
  index -= lowBit(index);
}
```


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
- [浅谈树状数组](https://www.cnblogs.com/wkfvawl/p/9445376.html)
- [树状数组（Binary Indexed Tree）](https://qoogle.top/binary-indexed-tree/)
