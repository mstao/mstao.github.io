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

大学学过线性代数的同学对矩阵可能还有点印象，矩阵（Matrix）是一个按照长方阵列排列的复数或实数集合，映射到编程语言是二维数组。

对于上图(a)而言，该图是一个**无向不加权图**，在矩阵中，其中的元素初始值都是0，我们规定从一个顶点i到另一个顶点j如果有边，这两个顶点对应的矩阵位置G(i, j) 和 G(j, i)  的值记为1，由于是无向图，可能会映射到两个元素，下面是图(a)的邻接矩阵表示：

$$
    \begin{bmatrix}
    0 & 1 & 0 & 1 \\
    1 & 0 & 1 & 0 \\
    0 & 1 & 0 & 1 \\
    1 & 0 & 1 & 0 \\
    \end{bmatrix}
$$

对于图(b)，这是个**有向不加权图**，我们规定，，其中的元素初始值都是0，从一个顶点i指向另一个顶点j，这两个顶点对应的矩阵位置G(i, j)的值记为1，图(b)的邻接矩阵表示为：

$$
    \begin{bmatrix}
    0 & 0 & 0 & 1 \\
    1 & 0 & 0 & 0 \\
    0 & 1 & 0 & 1 \\
    0 & 0 & 0 & 0 \\
    \end{bmatrix}
$$

对于图(c)，这是个**无向加权图**，我们规定，元素初始值都是0，从一个顶点i到另一个顶点j如果有边，这两个顶点对应的矩阵位置G(i, j) 和 G(j, i)  的值记为该边的权重值，所以图(c)的邻接矩阵表示为：

$$
    \begin{bmatrix}
    0 & 1 & 0 & 2 \\
    1 & 0 & 2 & 0 \\
    2 & 0 & 0 & 5 \\
    2 & 0 & 5 & 0 \\
    \end{bmatrix}
$$

从上面的几个邻接矩阵来看，用邻接矩阵来表示图还是很方便的，虽然简单，但却有很大问题，假设现在图G有n个顶点，需要开辟$n^2$个空间，并且很多空间都没有存储值，白白被浪费掉了。所以用邻接矩阵来表示图还是需要考虑考虑的。

用邻接矩阵来表示图代码也很简单，我们需要声明一个二维数组：

```Java
  // 邻接矩阵
  private int[][] adjacencyMatrix;
```

接下来要做的一个操作是添加边，首先要知道起始顶点的位置和目标顶点的位置，无向图两个位置都要赋值为1，代码如下：

```
  /**
   * 添加边
   *
   * @param start 起始节点位置
   * @param end   结束节点位置
   */
  public void addEdge(int start, int end) {
    checkPosition(start);
    checkPosition(end);

    this.adjacencyMatrix[start][end] = 1;
    this.adjacencyMatrix[end][start] = 1;
    this.edgeSize++;
  }
```

下面是完整代码：

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

### 邻接表

用邻接矩阵表示图代码很简单，计算也很方便，如果边比较密集，还是可以考虑的。可以不可以不要二维数组只要一维数组表示图呢？知道散列表的拉链法的同学比较清楚当一个key出现哈希冲突时，会存到链表里面，邻接表与这个类似，我们需要一个一维数组来保存所有的顶点，每一个顶点对应一个链表，存储所以与该顶点存在边的顶点。当我们需要判断一个顶点与另一个顶点是否存在边时，我们可以定位到这个顶点，然后遍历其链表看看有没有另一个顶点即可。如下图所示：

![image](https://github.com/mstao/static/blob/master/images/graph/adjacency-list.png?raw=true)


首先我们声明邻接表：

```Java
// 邻接表
private LinkedList<Integer>[] adj;
```

接着需要添加边，对于无向图，我们只需要将目标顶点放入到顶点对应的链表中就行了，代码如下：

```Java
public void addEdge(int start, int end) {
  checkPosition(start);
  checkPosition(end);
  adj[start].add(end);
  adj[end].add(start);
  this.edgeSize++;
}
```

完整代码如下：

```
/**
 * 无向图 - 基于邻接表(adjacency list)
 *
 * @author hanjuntao
 */
public class AJUndiGraph implements Graph {
  // 邻接表
  private LinkedList<Integer>[] adj;
  // 边数量
  private int edgeSize;

  public AJUndiGraph(int arrSize) {
    if (arrSize <= 0) {
      throw new IllegalArgumentException("The arrSize [" + arrSize + "] is not valid number");
    }

    adj = new LinkedList[arrSize];
    for (int i = 0; i < arrSize; i++) {
      adj[i] = new LinkedList<>();
    }
  }

  public void addEdge(int start, int end) {
    checkPosition(start);
    checkPosition(end);

    adj[start].add(end);
    adj[end].add(start);
    this.edgeSize++;
  }

  private void checkPosition(int index) {
    if (index < 0 || index >= adj.length) {
      throw new IllegalArgumentException("The index [" + index + "] is not valid number");
    }
  }

  @Override
  public int getNodeSize() {
    return adj.length;
  }

  @Override
  public int getEdgeSize() {
    return this.edgeSize;
  }

  @Override
  public String toString() {
    return "AJUndiGraph{" +
        "adj=" + Arrays.deepToString(adj) +
        ", nodeSize=" + getNodeSize() +
        ", edgeSize=" + edgeSize +
        '}';
  }
}
```

## 广度和深度优先搜索

### 广度优先搜索

广度优先搜索（Breadth-First-Search）被称为**bfs**，是一种比较好理解的图搜索方式，从图的一个顶点开始，向外一层一层地搜索。对于这种搜索，我们需要记录当前的节点是否被访问过，以及标识层次访问的情形。所以我们用一个**数组**记录节点是否被访问过，用一个**队列**来记录每一层的顶点。下面提供一个无向图，我们想要搜索到值为7的节点，如下图所示：

![image](https://github.com/mstao/static/blob/master/images/graph/bfs_1.png?raw=true)

首先我们访问值为1的节点，我们需要先将1节点入队。节点1相当于下一层的顶点，该层只有1这个节点，接下来将1节点出队，访问1的第二层的节点。接着访问该层的所有节点，包括2,3,4，并依次入队。

![image](https://github.com/mstao/static/blob/master/images/graph/bfs_2.png?raw=true)

第二层访问完毕，接下来我们要访问第三层。先从队列中取出2节点，访问2节点的下一层节点，图中只有5节点，该节点没有被访问过，将节点5加入到队列中。接着分别访问3节点和4节点的下一层节点，操作同上。

![image](https://github.com/mstao/static/blob/master/images/graph/bfs_3.png?raw=true)


第三层节点访问完毕，接下来访问第四层节点，从队列中取出节点5，访问其下一层节点7，发现是我们要搜索的，搜索结束。

![image](https://github.com/mstao/static/blob/master/images/graph/bfs_4.png?raw=true)

通过上面的图片演示，我们已经对bfs有一个大致的印象，下面我们来详细总结下bfs的搜索过程：

1. 初始一个队列用来记录待访问节点的顶点，初始一个数据用来节点是否被访问过；将起始节点入队；
2. 将队列中的队首元素出队，访问其连接的下一层节点，并将访问过的节点入队；
3. 重复第二步，直至搜索到终止节点。

下面我们通过用领接表来表示无向图实现bfs，代码很容易读，至于如何利用广度优先搜索寻找起始节点和终止节点的最短路径，下一篇文章再系统讨论。bfs代码如下：

```Java
  /**
   * 广度优先搜索 Breadth-First-Search
   *
   * @param s 起始顶点
   * @param t 终止顶点
   */
  public void bfs2(int s, int t) {
    int len = adj.length;
    // 记录节点是否被访问过
    boolean[] visited = new boolean[len];

    // 存储每一层的顶点
    Queue<Integer> queue = new LinkedList<>();
    visited[s] = true;
    queue.add(s);

    while (!queue.isEmpty()) {
      Integer vertex = queue.poll();
      for (int i = 0; i < adj[vertex].size(); i++) {
        Integer curr = adj[vertex].get(i);
        if (!visited[curr]) {
          visited[curr] = true;
          System.out.println("当前顶点：" + vertex + "，当前节点：" + curr);

          if (curr == t) {
            return;
          }

          queue.add(curr);
        }
      }
    }
  }
```

### 深度优先搜索

深度优先搜索（Depth-First-Search）被称为**dfs**，和广度优先搜索是截然不同的搜索算法，在深度优先搜索中，假设沿着某条路径可以找到想要的节点，那么就一直走下去，如果目标不存在，那么就返回上一个节点，重复以上步骤，直至匹配到目标或遍历整个图。

从上面的描述来看，很明显是用到了递归，这里的递归终止条件有两个：

1. 匹配到目标，遍历结束
2. 遍历整个图，遍历结束

bfs伪代码（图的存储结构采用邻接表）如下：

```
visited[]; 

dfs = (s, t) {
    visited[s] = true;
    if (match) {
        return;
    }
    
    for i : adj[s]
        if (!visited[i]) {
            visited[i] = true;
            dfs(i, t)
        }
}
```


同广度优先搜索类似，我们也需要一个数组记录图中的节点是否被访问过，如果被访问过，那么就需要跳过，感觉有点点回溯法的味道，这里确实用到了回溯的思想。

```
  /**
   * 深度优先搜索 Depth-First-Search
   *
   * 打印走过的路径
   *
   * @param s 起始顶点
   * @param t 终止顶点
   */
  public void dfs(int s, int t) {
    int len = adj.length;
    boolean[] visited = new boolean[len];
    recurDfs(s, t, visited);
  }

  private void recurDfs(int vertex, int t, boolean[] visited) {
    visited[vertex] = true;
    if (vertex == t) {
      return;
    }

    for (int i = 0; i < adj[vertex].size(); ++i) {
      int curr = adj[vertex].get(i);
      if (!visited[curr]) {
        System.out.println("当前顶点：" + vertex + "，当前节点：" + curr);
        recurDfs(curr, t, visited);
      }
    }
  }
```


## References：

- https://zh.wikipedia.org/wiki/%E5%9B%BE_(%E6%95%B0%E5%AD%A6)
- https://en.wikipedia.org/wiki/Graph_(discrete_mathematics)
- https://time.geekbang.org/column/article/70891
