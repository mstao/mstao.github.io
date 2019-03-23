---
title: 二叉树概念和操作
tags: [数据结构, 二叉树]
author: Mingshan
categories: [数据结构, 二叉树]
date: 2019-03-17
---

## 二叉树定义

**二叉树**（Binary Tree）是n（n >= 0）个结点所构成的集合，它或为空树（n=0）；或为非空树，对于非空树$T$：

1. 有且仅有一个称之为根的结点
2. 除根节点以外的其余结点分为两个互不相交的子集$T_1$和$T_2$，分别称为$T$的左子树和右子树，且$T_1$和$T_2$本身又是二叉树

<!-- more -->

## 二叉树性质及存储结构

通过上面的二叉树定义我们知道，二叉树的每个结点最多有两个子结点，同时具有递归性质。递归在二叉树的一些操作中使用非常频繁，对此不熟悉的同学请参考：[递归、尾递归和使用Stream延迟计算优化尾递归](https://mingshan.fun/2019/01/20/tail-recursion/)。接下来我们来看看二叉树的一些概念：

- 结点的高度：结点到叶子结点的最长路径（边数）
- 结点的深度：根结点到这个节点所经历的的边的个数
- 结点的层数：结点的深度 + 1
- 结点的度：结点拥有的子树数量称为结点的度
- 树的高度：根结点的高度

下面是一个二叉树及上面属性的具体值，就不解释了，比较简单。

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/BinaryTree.png?raw=true)

**二叉树具有以下重要特性：**

- 性质1：在二叉树的第i层上最多有$2^{i-1}$个结点（i>=1）
- 性质2：深度为k的二叉树至多有$2^k-1$个结点

二叉树既可以用数组来存储，又可以用链式结构来存储。

其中用链式结构来存储二叉树我们平时用的比较多，也好理解，我们可以把树的结点看个一个对象结点，其中有三个属性，数据区域、左孩子指针和右孩子指针，我们只需要根据根结点就可以利用结点的左右指针将整个树串起来。

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/BinaryTree-linked.png?raw=true)

对于用数组存储来说，我们需要将数组的第一位空出来，把根结点存储在下标为1的位置，对于任意一个结点，它在数组的存储位置为i，那么它的左结点存储的位置为`2i`，右结点为`2i + 1`。这样就可以将整个二叉树存储在数组中了。不过从上面的逻辑来看，用数组来存储二叉树会有空间的浪费，因为同一层有些结点有子结点，有些没有，这样就会浪费空间。

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/BinaryTree-array.png?raw=true)

## 二叉树操作

### 遍历

二叉树的概念和存储方式我们了解过了，那么对于二叉树而言，最重要的操作莫过于对其的遍历。在数据结构这门课上，我们学过对二叉树的遍历方式有三种：**前序遍历**、**中序遍历**以及**后序遍历**，其中，前、中、后序，表示的是节点与它的左右子树节点遍历打印的先后顺序。

- 前序遍历：对于树中的任意节点来说，先打印这个节点，然后再打印它的左子树，最后打印右子树
- 中序遍历：对于树中的任意节点来说，先打印它的左子树，然后再打印它本身，最后打印右子树
- 后序遍历：对于树中的任意节点来说，先打印它的左子树，然后再打印它的右子树，最后打印它本身

从上面三种遍历方式来看，前、中和后序遍历其实是递归的过程，下面是三种遍历的递推公式：

```
前序遍历的递推公式：
preOrder(r) = print r->preOrder(r->left)->preOrder(r->right)

中序遍历的递推公式：
inOrder(r) = inOrder(r->left)->print r->inOrder(r->right)

后序遍历的递推公式：
postOrder(r) = postOrder(r->left)->postOrder(r->right)->print r
```

接下来我们来仔细分析下这三种遍历的递归和非递归的方式。依下面的二叉树来分析：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/BinaryTree-demo.png?raw=true)

下面是代表二叉树的节点的Node类：

```Java
public static class Node<E extends Comparable<E>> {
    E item;
    Node<E> parent;
    Node<E> left;
    Node<E> right;

    public Node (Node<E> parent, E item) {
        this.parent = parent;
        this.item = item;
    }

    @Override
    public String toString() {
        return "item=" + item + " parent=" + ((parent != null) ? parent.item : "NULL") + " left="
                + ((left != null) ? left.item : "NULL") + " right=" + ((right != null) ? right.item : "NULL");
    }
}
```

#### 前序

前序遍历方式：对于树中的任意节点来说，先打印这个节点，然后再打印它的左子树，最后打印右子树。

我们先以递归的方式来思考整个遍历过程：

1. 输出1，接着左孩子
2. 输出2，接着左孩子
3. 输出4，左孩子为空，右孩子为空，此时2的左子树全部输出，接着输出2的右子树
4. 输出5，接着左孩子
5. 输出8，左孩子为空，右孩子为空，此时1的左子树全部输出完了，接着输出1的右子树
6. 输出3，接着左孩子
7. 输出6，左孩子为空，右孩子为空，3的左子树全部输出完了，接着输出3的右子树
8. 输出7，7的左孩子为空，右孩子为空，此时整个树输出完毕
 
**递归**代码比较简单，如下所示：

```java
/**
 * 前序遍历：
 *
 * 对于当前结点，先输出该结点，然后输出它的左孩子，最后输出它的右孩子
 */

/**
 * 前序遍历（递归）
 *
 * @param node
 */
public void preOrderRec(Node node) {
    if (node == null) {
        return;
    }

    System.out.println(node); // 先输出该结点
    preOrderRec(node.left);   // 输出它的左孩子
    preOrderRec(node.right);  // 输出它的右孩子
}
```

对于**非递归**的实现来说，其实就是模拟上面递归入栈出栈过程，在写代码之前，我们先牢记前序遍历的原则：**对于当前结点，先输出该结点，然后输出它的左孩子，最后输出它的右孩子**。所以对任意节点，我们都可以先把它看成父结点，输出该结点后，把它入栈，然后接着它的左孩子，此时这个左孩子又可以作为父结点，就这样一直遍历下去，直至当前结点的左孩子为空，上述过程用代码（**代码片段Ⅰ**）描述为：

```Java
while (node != null) {
    System.out.println(node); // 先输出当前结点
    stack.push(node);         // 当前结点入栈
    node = node.left;         // 遍历左孩子
}
```


此时会出现两种情况：

1. 当前结点的左孩子为空，但右孩子不为空
2. 当前结点的左右孩子都为空

**情况1**如下图所示：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/BinaryTree_node1.png?raw=true)

注意此时栈顶元素为下图中**栈顶节点**执行的结点，接着我们该如何进行呢？此时我们需要将栈顶元素出栈并赋值给node，接着我们就要访问当前节点的右孩子了（当前结点输出过了，左孩子为空），如果右孩子有左孩子，继续重复上面的步骤，代码（**代码片段Ⅱ**）如下：

```Java
node = stack.pop(); // 栈顶元素出栈
node = node.right; // 继续访问其右孩子
```

**情况2**如下图所示：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/BinaryTree_node2.png?raw=true)


对于情况2，同样此时栈顶元素为下图中**栈顶节点**执行的结点，此时我们需要将栈顶元素出栈并赋值给node，接着访问该结点的右孩子为空，此时相当于当前的所以我们就可以继续将栈顶元素出栈，继续访问当前栈顶元素的右孩子（左子树全部输出完毕），即：

```Java
node = stack.pop(); // 栈顶元素出栈
node = node.right; // 继续访问其右孩子
node = stack.pop(); // 右孩子为空， 继续出栈
node = node.right; // 继续访问其右孩子
```

我们发现上面的代码其实重复的，只不过需要加个空值判断就可以了，那么这个空值判断加在什么地方呢？仔细看看前面的，我们发现空值判断在**代码片段Ⅰ**
已经处理过了，只有在结点不为空时才能继续访问它的左孩子，所以，综合上面的分析，我们可以写出：


```Java
while (node != null || !stack.isEmpty()) {
    while (node != null) {
        System.out.println(node); // 先输出当前结点
        stack.push(node);         // 当前结点入栈
        node = node.left;         // 遍历左孩子
    }

    node = stack.pop();
    node = node.right;
}
```

好了，这里总结一下，主要有三个步骤：

1. 对于任何结点node，如果该结点不为空，打印当前结点node后将自己压入栈内，然后将当前结点的左子结点赋值给node，直至node为null
2. 若左子树为空，则栈顶元素出栈，并将当前node的右子结点赋值给node
3. 重复1，2步操作，直至node为空，并且栈为空

此时二叉树输出完毕，代码如下：

```Java
/**
 * 前序遍历（非递归）<br/>
 *
 * <ul>
 *  <li>1. 对于任何结点node，如果该结点不为空，打印当前节点将自己压入栈内，然后将当前结点的左子结点赋值给node，直至node为null</li>
*   <li>2. 若左子树为空，则栈顶元素出栈，并将当前node的右子结点赋值给node</li>
*   <li>3. 重复1，2步操作，直至node为空，并且栈为空</li>
 * <ul/>
 *
 * @param node
 */
public void preOrderNonRec(Node node) {
    if (node == null) {
        return;
    }

    System.out.println(node); // 先输出当前结点

    Stack<Node> stack = new Stack<>();
    stack.push(node);
    node = node.left;

    while (node != null || !stack.isEmpty()) {

        while (node != null) {
            System.out.println(node); // 先输出当前结点
            stack.push(node);         // 入栈
            node = node.left;         // 输出左孩子
        }                             // 循环结束，节点左子树全部输出

        node = stack.pop(); // 依次出栈
        node = node.right;  // 输出右孩子
    }
}
```

#### 中序

中序遍历方式：对于树中的任意节点来说，先打印它的左子树，然后再打印它本身，最后打印右子树。

我们先以递归的方式来思考整个遍历过程：

1. 从1开始，遍历节点的左子树，遍历到4，4的左孩子为空，右孩子为空
2. 输出4, 接着父结点
3. 输出2，接着右孩子，不为空，遍历5的左子树，遍历到8，8的左孩子为空，右孩子为空
4. 输出8，接着父结点
5. 输出5，5的右结点为空，回到2，2的所有子树输出完毕，接着2的父结点
6. 输出1，接着1的右孩子
7. 遍历3的左子树，遍历到6，6的左孩子为空，右孩子为空
8. 输出6，接着父结点
9. 输出3，接着右孩子
10. 输出7，7的左孩子为空，右孩子为空，此时整个树输出完毕

**递归**代码：


```Java
/**
 * 中序遍历（递归）
 *
 * @param node
 */
public void inOrderRec(Node node) {
    if (node == null) {
        return;
    }

    inOrderRec(node.left);
    System.out.println(node);
    inOrderRec(node.right);
}

```

**非递归**逻辑就是用栈模拟上述递归调用的过程，经过前序非递归实现的思考过程，相信我们对于中序遍历的非递归实现已经胸有成竹了，下面还是仔细分析一波。我们还是牢记中序遍历的原则：**对于树中的任意节点来说，先打印它的左子树，然后再打印它本身，最后打印右子树。**

对于任意节点都可以把它看成父结点，由于中序遍历是先输出结点的左子树，所以我们就可以遍历，将遍历的轨迹记录到栈内，直至结点为空，我们就可以写出如下代码（代码片段Ⅲ）：

```Java
while (node != null) { // 判断当前结点点是否为空
    stack.push(node); // 当前结点入栈
    node = node.left; // 访问其左孩子
}
```

当遍历结束后，和前序遍历一样，也会出现两种情况：

1. 当前结点的左孩子为空，但右孩子不为空
2. 当前结点的左右孩子都为空

**情况1**如下图所示：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/BinaryTree_node1.png?raw=true)

从上面图中可以看出，此时node指向为空，我们需要将栈顶元素出栈并赋值为node，此时node的左孩子为空，我们无法输出，所以我们就输出当前节点node，然后访问当前节点的右孩子，如果右孩子有左孩子，继续重复上面的步骤，代码（代码片段Ⅳ）如下：

```Java
node = stack.pop(); // 栈顶结点出栈
System.out.println(node); // 打印栈顶结点
node = node.right; // 访问其右孩子
```

**情况2**如下图所示：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/data-structure/BinaryTree_node2.png?raw=true)

对于情况2，当访问的结点为空时，我们就和情况1一样将栈顶元素出栈并赋值给node，将其输出，接着我们访问当前节点node的右孩子，发现右孩子为空，此时该怎么办呢？有了前序遍历的经验，我们继续将栈顶元素出栈，并赋值给node，注意此时node的左子树输出完毕了，输出该节点，继续访问node的右孩子：

```Java
node = stack.pop(); // 栈顶结点出栈
System.out.println(node); // 打印栈顶结点
node = node.right; // 访问其右孩子
node = stack.pop(); // 右孩子为空，栈顶元素继续出栈
System.out.println(node); // 打印栈顶元素
node = node.right; // 继续访问其右孩子
```

我檫，我们又发现重复，根据我们的经验，凡是重复的代码都不是最优代码，其实造成重复的原因是当前节点的右孩子为空，没法继续访问了，只能继续将栈顶结点出栈，此时的栈顶的结点的左子树已经输出完了，输出自己后，再访问其右孩子。这个判空在**代码片段Ⅲ**中。

好了，不分析了，此时我们可以写出如下代码：

```Java
while (node != null || !stack.isEmpty()) {
    while (node != null) {
        stack.push(node);
        node = node.left;
    }
    
    node = stack.pop();
    System.out.println(node);
    node = node.right;
}
```


总结一下，考虑以下三点：

1. 对于任何结点node，如果该结点不为空，将当前结点的左子结点赋值给node，直至node为null
2. 若左子结点为空，栈顶结点出栈，输出该结点后将该结点的右子结点赋值给node
3. 重复1，2操作

代码如下：

```Java
/**
 * 中序遍历（非递归）
 *
 * <ul>
 *  <li>1. 对于任何结点node，如果该结点不为空，将当前结点的左子结点赋值给node，直至node为null</li>
 *  <li>2. 若左子结点为空，栈顶节点出栈，输出该结点后将该结点的右子结点置为node</li>
 *  <li>3. 重复1，2操作</li>
 * </ul>
 *
 * @param node
 */
public void inOrderNonRec(Node node) {
    if (node == null) {
        return;
    }

    Stack<Node> stack = new Stack<>();
    stack.push(node);
    node = node.left;

    while (node != null || !stack.isEmpty()) {
        while (node != null) {
            stack.push(node);
            node = node.left;
        }

        node = stack.pop();
        System.out.println(node);
        node = node.right;
    }
}
```

#### 后序

后序遍历：对于树中的任意节点来说，先打印它的左子树，然后再打印它的右子树，最后打印它本身。

我们先以递归的方式来思考整个遍历过程：

1. 从1开始，遍历节点的左子树，遍历到4，4的左孩子为空，右孩子为空
2. 输出4， 接着寻找2的右子树，找到5
3. 遍历5的左子树，遍历到8，8的的左孩子为空，右孩子为空
4. 输出8，寻找5的右孩子，发现为空，接着父结点
5. 输出5，接着5的父结点2，2的左右子树输出完毕
6. 输出2，接着2的父结点，此时2的左子树输出完毕，接着2的右子树，找到3
7. 遍历3的左子树，找到6，6的左孩子为空，右孩子为空
8. 输出6，接着3的右孩子
9. 输出7，接着7的父结点
10. 输出3，接着3的父结点，此时1的左右子树输出完毕
11. 输出1，此时整个树输出完毕


**递归**代码非常简单，如下代码：

```Java
/**
 * 后序遍历（递归）
 *
 * @param node
 */
public void postOrderRec(Node node) {
    if (node == null) {
        return ;
    }

    postOrderRec(node.left);
    postOrderRec(node.right);
    System.out.println(node);
}
```

 非递归实现是比上面前中非递归遍历要复杂一点，现在还是考虑后序遍历的原则：**对于树中的任意节点来说，先打印它的左子树，然后再打印它的右子树，最后打印它本身。**

对于任意结点，总是先遍历其左孩子，再遍历右孩子，最后再遍历父结点，仔细思考这个过程和前中遍历有什么不同呢？对于**前序遍历**，父结点首先被访问到；对于**中序遍历**，当访问完左孩子，就可以访问父结点了；对于**后序遍历**，访问完左孩子，要去访问右孩子，注意，此时这个右孩子如果还有左孩子，那么还要继续遍历下去。说到这里我们就知道问题了，最初左孩子的父结点啥时候访问呢？就是最初的右孩子的左右子树都访问完了，再访问这个右孩子，最后才会访问到这个父结点。

所以根据以上分析，我们需要**判断上次访问的结点是位于左子树，还是右子树**。如果是左子树，那么需要跳过父结点，去访问右孩子；如果上次是访问的右孩子，我们就可以访问父结点了。

总结来说，**对于任意结点，只有它既没有左孩子也没有右孩子或者它有孩子但是它的孩子已经被输出，才会输出这个结点**，所以我们可以整一个变量（pre）来记录上次访问的是哪一个结点。若非上述两种情况，则将该结点的右孩子和左孩子依次入栈，这样就保证了每次取栈顶元素的时候, 先依次遍历左子树和右子树。代码如下：

```Java
if ((node.left == null && node.right == null) ||
    (node.right == null && pre == node.left) || (pre == node.right)) {
    System.out.println(node);
    pre = node;
    stack.pop();
} else {
    // 右孩子先入栈，才会先访问结点的左孩子
    if (node.right != null) {
        stack.push(node.right);
    }

    if (node.left != null) {
        stack.push(node.left);
    }
}
```

总结以上，pre首先赋初值为二叉树的根结点，栈初始值也为二叉树的根结点，所以在栈不为空的情况下，进行以上判断操作。所以完整代码如下：

```Java
/**
 * 后序遍历（非递归）
 *
 * 对于结点node，可分三种情况考虑：
 *
 * 1. node如果是叶子结点，直接输出
 * 2. node如果有孩子，且孩子没有被访问过，则按照右孩子，左孩子的顺序依次入栈
 * 3. node如果有孩子，而且孩子都已经访问过，则访问node节点
 *
 * 注意结点的右孩子先入栈，左孩子再入栈，这样才会先访问左孩子
 *
 * @param node
 */
public void postOrderNonRec(Node node) {
    if (node == null) {
        return ;
    }

    Stack<Node> stack = new Stack<>();
    Node pre = root;
    stack.push(node);

    while (!stack.isEmpty()) {
        node = stack.peek();

        if ((node.left == null && node.right == null) ||
            (node.right == null && pre == node.left) || (pre == node.right)) {
            System.out.println(node);
            pre = node;
            stack.pop();
        } else {
            // 右孩子先入栈，才会先访问结点的左孩子
            if (node.right != null) {
                stack.push(node.right);
            }

            if (node.left != null) {
                stack.push(node.left);
            }
        }

    }
}
```

## References：

- [二叉树基础（上）：什么样的二叉树适合用数组来存储？](https://time.geekbang.org/column/article/67856)
- [二叉树的各种操作](https://subetter.com/algorithm/various-operations-of-the-binary-tree.html)
- 严蔚敏，《数据结构（C语言）第二版》
- [二叉树的后序遍历--非递归实现](https://www.cnblogs.com/rain-lei/p/3705680.html)
- [二叉树前序、中序、后序遍历非递归写法的透彻解析](https://blog.csdn.net/zhangxiangdavaid/article/details/37115355)


[<font size=3 color="#409EFF">向本文提出修改或勘误建议</font>](https://github.com/mstao/mstao.github.io/blob/hexo/source/_posts/binary-tree.md)

