---
title: 图的数据结构及基础算法
tags: [数据结构, 图]
author: Mingshan
categories: [数据结构, 图]
date: 2020-01-19
mathjax: true
---

**图(Graph)**这个数据结构在平时开发中遇到的比较少，但我认为它是十分重要的，因为从真实的世界中来看，很多东西都可以抽象为图的表示，比如人际关系，地理位置，天马行空的东西都可以抽象为图，所以它比链表等基础数据结构高级一点点，也比较复杂，属于非线性结构。数学中有一个图论的分支也是与其有关。了解图在程序世界的存储方式，我们可以更加细致地刻画图的结构，让其为我所用，岂不妙哉？

<!-- more -->

## 定义和分类

那么图的定义是什么呢？我们直接看维基百科：

> 一张图 $G$ 是一个二元组$(V,E)$，其中$V$称为顶点集，$E$称为边集。它们亦可写成$V(G)$和$E(G)$。 $E$的元素是一个二元组数对，用$(x,y)$表示，其中${x,y} \in V$。

下面一个图的示例：

![image](https://github.com/mstao/static/blob/master/images/graph/graph.png?raw=true)

图中的元素被称为**顶点**（Vertex），顶点与顶点之间连线被称为**边**（Edge），每个顶点有多少连线被称称为**度**（Degree），比如a图的2节点，度为2。

图(a)顶点与顶点之间的边没有方向，这种图被称为**无向图**（Undirected graph），在现实中，一个顶点与另一个顶点之间也可以是有向的，比如我们熟悉的铁路网等，从一个地方到一个地方，必然是有指向的，这种叫**有向图**（Directed graph），如图(b)所示。当然一个更常见的场景，比如北京到上海的距离肯定比上海到苏州的距离长，如何描述边与边的权重呢？这种图被称为**带权图**（Weighted graph），1顶点到2顶点边的权重为1。这里的权重只是抽象出的概念，在现实中可根据不同场景代表不同的含义，比如距离，车票价格等。

在**有向图**中，指向这个顶点的边和从其出发的边是不一样的，所以用**出度**（Out-degree）和**入度**（In-degree）来具体描述顶点的度。出度指的是有多少边是从当前顶点出发指向其他顶点；入度指的是有多少边指向当前顶点。对于图(b)而言，1节点的出度和人度都是1。

当然图是有很多的概念，上面是最基本的，了解了这些，我们才可以探讨图在代码中如何表示（以无向图示例）。

## 存储表示

根据维基百科的介绍，图有如下存储表示：

- 邻接矩阵（Adjacency matrix）
- 邻接表（Adjacency list）
- 前向星
- 有向图的十字链表
- 无向图的邻接多重表

本文以邻接矩阵和邻接表为例，后面几种再单独探讨。

### 邻接矩阵

大学学过线性代数的同学对矩阵可能还有点印象，矩阵（Matrix）是一个按照长方阵列排列的复数或实数集合，映射到编程语言很像是二维数组，对于上图(a)而言，该图是一个无向不加权图，在矩阵中，其中的元素初始值都是0，我们规定从一个顶点到另一个顶点如果有边，这两个订点对应的矩阵位置的值记为1，由于是无向图，可能会映射到两个元素，下面是图(a)的邻接矩阵表示：

$$
    \begin{bmatrix}
    0 & 1 & 0 & 1 \\
    1 & 0 & 1 & 0 \\
    0 & 1 & 0 & 1 \\
    1 & 0 & 1 & 0 \\
    \end{bmatrix}
$$

### 邻接表

```Java
/**
 * 无向图 - 基于邻接矩阵(adjacency matrix)
 *
 * @author hanjuntao
 */
public class AMUndiGraph implements Graph {

  /**
   * 邻接矩阵长或宽最大长度
   */
  private static final int maxSideLength = 4;
  // 邻接矩阵长或宽长度
  private int sideLength;
  // 邻接矩阵
  private int[][] adjacencyMatrix;
  // 节点数目
  private int nodeSize;
  // 边数量
  private int edgeSize;

  public AMUndiGraph() {
    this(maxSideLength);
  }

  public AMUndiGraph(int sideLength) {
    if (sideLength <= 0) {
      throw new IllegalArgumentException("The sideLength [" + sideLength + "] is not valid number");
    }

    this.sideLength = sideLength;
    adjacencyMatrix = new int[sideLength][sideLength];
  }

  /**
   * 添加边
   *
   * @param start 起始节点位置
   * @param end   结束节点位置
   */
  public void addEdge(int start, int end) {
    checkPosition(start);
    checkPosition(end);

    if (this.adjacencyMatrix[start][end] == 0 && this.adjacencyMatrix[end][start] == 0) {
      this.adjacencyMatrix[start][end] = 1;
      this.adjacencyMatrix[end][start] = 1;
      this.edgeSize++;
    }
  }

  /**
   * 获取节点数量
   *
   * @return 节点数量
   */
  @Override
  public int getNodeSize() {
    int currNodeSize = 0;

    for (int i = 0; i < sideLength; i++) {
      for (int j = 0; j < i + 1; j++) {
        if (adjacencyMatrix[i][j] == 1) {
          currNodeSize++;
        }
      }
    }

    return currNodeSize;
  }

  /**
   * 获取边的数量
   *
   * @return 边的数量
   */
  @Override
  public int getEdgeSize() {
    return this.edgeSize;
  }

  private void checkPosition(int index) {
    if (index < 0 || index >= adjacencyMatrix.length) {
      throw new IllegalArgumentException("The index [" + index + "] is not valid number");
    }
  }

  @Override
  public String toString() {
    return "AMUndiGraph{" +
        "sideLength=" + sideLength +
        ", adjacencyMatrix=" + Arrays.deepToString(adjacencyMatrix) +
        ", nodeSize=" + nodeSize +
        ", edgeSize=" + edgeSize +
        '}';
  }
}
```

## References：

- https://zh.wikipedia.org/wiki/%E5%9B%BE_(%E6%95%B0%E5%AD%A6)
- https://en.wikipedia.org/wiki/Graph_(discrete_mathematics)
