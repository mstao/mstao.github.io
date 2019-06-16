---
title: 跳表的设计与实现
tags: [数据结构, 跳表]
author: Mingshan
categories: [数据结构, 链表]
date: 2019-06-02
---

链表作为一种数据结构我们是比较熟知的，相对数组来说插入和删除操作性能比较高，因为数组涉及到移位操作，但数组可以利用二分法进行快速查找，而在链表中想要获取当前元素，就必须知道该元素的上一个节点（头节点除外），这就限制了链表在查找操作的性能，试想有没有一种数据结构，在链表基础上也能实现类似二分查找这样较快的操作呢？这就要引出本篇要了解的**跳表(SkipList)**这种数据结构了。

何谓跳表？跳表通过维护一个多层次的链表，且每一层链表中的元素是前一层链表元素的子集。下面是一个已经排好序的链表：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/skiplist/skiplist_linklist.png?raw=true)

如果我们想快速定位某一节点，可以选取链表的某些节点重新生成一个新的链表，这个新提取的链表作为一层索引，当想访问原始链表的一个节点时，我们可以先访问索引链表，找出索引链表的符合条件某个节点后（注意该节点的值在原始链表必然存在），然后下沉到原始链表，接下来就在原始链表中寻找相应节点即可，如下图所示：

<!-- more -->

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/skiplist/skiplist_linklist_two_ahead.png?raw=true)

比如我们要找12这个节点，我们先遍历上层索引链表，找到符合条件的节点9，接着下沉到最底层链表，再往后遍历一个节点，就找到12了。加来一层索引之后，查找一个结点需要遍历的结点个数减少了，查找效率也会相应提高。如果我们在第二层索引上再一层索引呢？查找方式和上面类似，就不分析了。这里有个疑问，我怎么知道要有多少层索引链表？每一层选取哪些节点呢？这涉及到跳表查找的核心，如果索引链表选取不合适，那么查找效率也大打折扣，所以我们要保证索引链表合适，数据分布均匀。

## 跳表节点表示和随机算法选取

从前面的概念我们可以了解到，对于跳表中任意节点（原始链表和索引链表），都至少有两个指针，一个指针（next）指向当前层的下一个节点，一个指针（down）指向当前节点的下一层节点，只有这样我们才能应用上面的查找算法来定位到相应节点，这里参考leveldb里面的[Leveldb - skiplist实现](https://github.com/google/leveldb/blob/master/db/skiplist.h)，对于任意节点，我们都有一个`forward[]`数组，**用以存储该节点所有层的下一个节点的信息**。啥意思？就拿上面图中节点9来说，无论其是第一层还是第二层，该节点的`forward[]`数组存储的是第1层 forward[0] = 12，第二层 forward[1] = 17，每层依次往后存储..., 这样我们遍历到节点9后，就知道了当前9节点在所有层的后继节点情况，接着就可以使用我们的遍历算法了。节点的定义如下：

```Java
/**
 * 内部节点类
 *
 * @param <T> 泛型参数
 */
private static class Node<T> {
    int key;
    T value;
    /**
     * 每一层单链表指针：
     * 0：最底层
     * ......
     * i：第i层节点
     *
     * p = p.forwards[i] 表示第i层下一个节点
     */
    Node<T>[] forward;

    public Node(int key, T value, int level) {
        this.key = key;
        this.value = value;
        this.forward = new Node[level];
    }
}
```

在跳表的原始论文中（Algorithm to calculate a random level），给出了如何计算随机层数的伪代码，如下所示：

```
randomLevel()
    lvl := 1
    -- random() that returns a random value in [0...1)
    while random() < p and lvl < MaxLevel do
        lvl := lvl + 1
    return lvl
```

上面的随机层数算法设计到一个概率值p，我们其实可以猜想到，越往上的链表，节点应当越少，所以对于任意一个结点，其出现在层次概率其实是依次减少的（从下到上），我们来看看在Leveldb对skiplist实现中，也采用了随机算法来决定某一个节点该存在哪些索引层。下面是cpp的`RandomHeight`实现：

```cpp
template <typename Key, class Comparator>
int SkipList<Key, Comparator>::RandomHeight() {
  // Increase height with probability 1 in kBranching
  static const unsigned int kBranching = 4;
  int height = 1;
  while (height < kMaxHeight && ((rnd_.Next() % kBranching) == 0)) {
    height++;
  }
  assert(height > 0);
  assert(height <= kMaxHeight);
  return height;
}
```

我们还是按照论文给出的数据还定义下`randomLevel`的实现吧，下面是跳表的一些初始数据，包括最佳概率值：

```Java
// 最高层数
private final int MAX_LEVEL;
// 当前层数
private int listLevel;
// 表头
private Node<T> listHead;
// 表尾
private Node<T> NIL;
// 生成randomLevel用到的概率值
private final double P;
// 论文里给出的最佳概率值
private static final double OPTIMAL_P = 0.25;

public SkipList() {
    // 0.25, 15
    this(OPTIMAL_P, (int)Math.ceil(Math.log(Integer.MAX_VALUE) / Math.log(1 / OPTIMAL_P)) - 1);
}

public SkipList(double probability, int maxLevel) {
    P = probability;
    MAX_LEVEL = maxLevel;

    listLevel = 1;
    listHead = new Node<T>(Integer.MIN_VALUE, null, maxLevel);
    NIL = new Node<T>(Integer.MAX_VALUE, null, maxLevel);
    for (int i = listHead.forward.length - 1; i >= 0; i--) {
        listHead.forward[i] = NIL;
    }
}
```

下面是`randomLevel`的实现：

```Java
private int randomLevel() {
    int level = 1;
    while (level < MAX_LEVEL && Math.random() < P) {
        level++;
    }
    return level;
}
```

## 快速查找

经过上面对跳表结构和查找算法的分析，查找的代码其实已经比较清楚了，下面是论文中给出的查找伪代码：

```
Search(list, searchKey)
    x := list→header
    -- loop invariant: x→key < searchKey
    for i := list→level downto 1 do
        while x→forward[i]→key < searchKey do
            x := x→forward[i]
    -- x→key < searchKey ≤ x→forward[1]→key
    x := x→forward[1]
    if x→key = searchKey then return x→value
        else return failure
```

上面的查找过程为：

1. 从最上层链表开始遍历查找, x为头结点， x->forwards[i] 表示第i层下一个节点，条件是当前层的下一个节点小于查找key，查找到后更新x
2. 最上层链表查找完毕，下沉到下一层，继续重复上一步操作
3. 当全部层查找完毕，判断当前x的key是否与查找key相等，相等则查找成功，返回对应值；否则失败

完整代码如下：

```Java
public T search(int searchKey) {
    Node<T> curNode = listHead;

    for (int i = listLevel; i > 0; i--) {
        while (curNode.forward[i].key < searchKey) {
            curNode = curNode.forward[i];
        }
    }

    if (curNode.key == searchKey) {
        return curNode.value;
    } else {
        return null;
    }
}
```

## 高效插入和索引动态更新

插入操作比较复杂一点，因为要涉及到更新索引链表的问题，当我们用上面的查找算法查找当前待插入元素的合适位置时，

```Java
public void insert(int searchKey, T newValue) {
    // update数组为层级索引，用以存储新节点所有层数上，各自的前一个节点的信息
    Node<T>[] update = new Node[MAX_LEVEL];
    Node<T> curNode = listHead;

    // record every level largest value which smaller than insert value in update[]
    // 在update中纪录每一层中 小于value值的最大节点
    for (int i = listLevel - 1; i >= 0; i--) {
        while (curNode.forward[i].key < searchKey) {
            curNode = curNode.forward[i];
        }
        // use update save node in search path
        update[i] = curNode;
    }

    // in search path node next node become new node forwords(next)
    // 插入newNode 串联每一个层级的索引
    curNode = curNode.forward[0];

    if (curNode.key == searchKey) {
        curNode.value = newValue;
    } else {
        int lvl = randomLevel();

        // 随机的层数有可能会大于当前跳表的层数，那么多余的那部分层数对应的update[i]置为sl->head,后面用来初始化
        if (listLevel < lvl) {
            for (int i = listLevel; i < lvl; i++) {
                update[i] = listHead;
            }
            listLevel = lvl;
        }

        Node<T> newNode = new Node<T>(searchKey, newValue, lvl);

        // 逐层更新节点的指针(这里的层指的是随机的层，比如当前有4层，然后随机的层为2，则只会将新节点插入下面的两层)
        // 如果当前跳表层是4，随机的为6，则会把5、6层也赋值，用到update[i] = sl->head;这里的结果。
        for (int i = 0; i < lvl; i++) {
            // 这里就是说随机几层，就用到update中的那几层，插入到update[i]对应的节点之后
            newNode.forward[i] = update[i].forward[i];
            update[i].forward[i] = newNode;
        }
    }
}
```

## 删除


```Java
public void delete(int searchKey) {
    Node<T>[] update = new Node[MAX_LEVEL];
    Node<T> curNode = listHead;

    for (int i = listLevel - 1; i >= 0; i--) {
        while (curNode.forward[i].key < searchKey) {
            curNode = curNode.forward[i];
        }
        // curNode.key < searchKey <= curNode.forward[i].key
        update[i] = curNode;
    }

    curNode = curNode.forward[0];

    if (curNode.key == searchKey) {
        for (int i = 0; i < listLevel; i++) {
            if (update[i].forward[i] != curNode) {
                break;
            }
            update[i].forward[i] = curNode.forward[i];
        }

        while (listLevel > 0 && listHead.forward[listLevel - 1] == NIL) {
            listLevel--;
        }
    }
}
```


## References：

- [Skip Lists: A Probabilistic Alternative to
Balanced Trees](https://www.epaperpress.com/sortsearch/download/skiplist.pdf)
- [跳跃列表 - 维基百科](https://zh.wikipedia.org/wiki/%E8%B7%B3%E8%B7%83%E5%88%97%E8%A1%A8)
- [跳表：为什么Redis一定要用跳表来实现有序集合？](https://time.geekbang.org/column/article/42896)
- [Leveldb - skiplist](https://github.com/google/leveldb/blob/master/db/skiplist.h)
- [leveldb 源码分析 —— SkipList跳表](https://www.jianshu.com/p/e4b775870eea)
- [跳表的深入浅出——SkipList](https://simplecodesky.com/2018/04/16/skiplist-deep-learning/)
- [跳表(Skip List)的介绍以及查找插入删除等操作](http://www.spongeliu.com/63.html)
- [跳表SkipList](https://www.cnblogs.com/xuqiang/archive/2011/05/22/2053516.html)
- [使用CAS实现无锁的SkipList](https://www.cnblogs.com/fuzhe1989/p/3650303.html)
- [跳表 skiplist](https://segmentfault.com/a/1190000006024984)
- [跳表skiplist的理解](https://juejin.im/post/5c9fa06051882567d17c2e6e)

