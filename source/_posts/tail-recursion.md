---
title: 递归、尾递归和使用Stream延迟计算优化尾递归
tags: [数据结构]
author: Mingshan
categories: [数据结构]
date: 2019-1-20
---

我们在学数据结构的时候必然会接触栈（Stack），而栈有一个重要的应用是在程序设计语言中实现递归。递归用途十分广泛，比如我们常见的阶乘，如下代码：

<!-- more -->

```Java
public static int func1(int n) {
    if (n == 1) return 1;
    return n * func1(n - 1);
}
```

就可以用递归实现，而且实现相当简洁。如果要计算n的阶乘，那么只需知道n-1的阶乘再乘以n，同理依次类推，直到当我们计算2的阶乘的时候，只需知道1的阶乘，显然这是递归终止条件，再层层向上返回，直至计算出n的阶乘即可。

从上面的分析可以看出，如果我们要进行递归求解某一问题，需要满足以下三个条件：

1. 能将一个问题转变为另一个新问题，而新问题的解法与原问题相同或者类同，并且新问题的数据规模更小，问题简化。

使用递归的情景是当前数据规模较大，直接计算比较困难，那么可以将该问题进行分解，数据规模越来越小，计算也越来越容易，其实这是“分治法”的体现。

2.  存在递归终止条件，或者说递归的边界。

递归的终止条件是必须的，既然当前问题可以分解，那么就必须存在一个“极限”，分解到什么程度？到哪里停止？

“分治法”求解递归问题算法有一个一般形式：

```
void p(参数列表) {
    if (递归终止条件成立) 直接求解；    // 递归终止条件
    else p(较小的参数)                  // 递归步骤
}
```

写递归代码要考虑两点：问题的**递推公式**以及**终止条件**。比如阶乘问题，它的递推公式为：

```
f(n) = n * f(n - 1)
```
这个自然很明显，如果遇到复杂的递归问题，那么推导问题的递推公式是十分重要的，同时推导出问题的终止条件，问题就好解决了。


**递归代码要警惕堆栈溢出**

就拿Java来说，栈帧（Stack Frame）是用于支持虚拟机进行方法调用和方法执行的数据结构。它是虚拟机运行时数据区中的虚拟机栈的栈元素。栈帧存储了方法的局部变量表、操作数栈、动态链接和方法返回地址等信息。每一个方法从调用开始至执行完成的过程，都对应着一个栈帧在虚拟机栈里入栈与出栈的过程。

在Java虚拟机规范的描述中，我们可以查到这样的描述：

```
If the computation in a thread requires a larger Java Virtual Machine stack than is permitted, the Java Virtual Machine throws a StackOverflowError.
```

上面的描述大意是：如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出StackOverflowError异常，如下所示：

```
Exception in thread "main" java.lang.StackOverflowError
```

所以我们在写递归代码时，时刻警惕堆栈溢出异常。当递归代码调用层次很深，超过了虚拟机所允许的最大深度，就会出现上面的异常了。

**递归代码要注意重复计算问题**

还有一个比较容易被忽略的问题是，我们在写递归代码时，有可能进行重复计算。就拿我们最熟悉的Fibonacci数列来说，下面是我们常见的Fibonacci数列递归代码：

```Java
public static int fib(int n) {
    if (n == 1 || n == 2) {
        return 1;
    }

    return fib(n - 1) + fib(n - 2);
}
```

观察上面的代码我们可以发现，每计算一个数，都要计算两个数的值，其中就会出现计算重复的情况，比如计算fib(10)， 需要计算fib(9) 和 fib(8)，计算fib(9)，需要计算fib(8) 和 fib(7)，发现fib(8)被计算了两遍，如下所示：

```
    10
   /  \
  9    8
 / \  / \
8   7 7  6
    ...
```

所以我们能不能将计算的结果缓存起来呢？这样当计算某一个数的时候，都去缓存里面查一下，来避免重复计算。如下代码所示：

```Java
public static int fib2(int n) {
    if (n == 1 || n == 2) {
        return 1;
    }

    if (cache.containsKey(n)) {
        return cache.get(n);
    }

    int res = fib(n - 1) + fib(n - 2);
    cache.put(n, res);
    return res;
}
```

## 尾调用（tail cal）与尾递归（tail recursion）

在了解尾递归之前，我们先了解下什么是尾调用？在阮一峰的[尾调用优化](http://www.ruanyifeng.com/blog/2015/04/tail-call.html)这篇文章中详细描述了尾调用，尾调用就是指某个函数的最后一步是调用另一个函数，如下所示：


```
function f(x){
  return g(x);
}
```

就是函数最后执行一步时直接调用了另一个函数，没有当前方法的局部变量再参与运算。

上面我们说到递归，因为递归是自己调用自己，如果尾调用自身，就称为尾递归。
这是什么意思呢？看如下代码：


```Java
public int func2(int n, int total) {
    if (n == 1) return total;
    return func2(n - 1, n * total);
}
```

我们将上面提到的阶乘用尾递归来改写下，发现多了一个参数，比如我们计算5的阶乘，计算情况如下：

```
func2(5, 1)
   ↓
func2(4, 5)
   ↓
func2(3, 20)
   ↓
func2(2, 60)
   ↓
func2(1, 120)
```

那么普通的阶乘递归和尾递归计算阶乘啥区别呢？

在普通递归中，比如计算func(n), func(n)  是依赖于 func(n-1) 的，func(n) 只有在得到 func(n-1) 的结果之后，才能计算它自己的返回值，因此理论上，在 func(n-1) 返回之前，func(n)，不能结束返回。因此func(n)就必须保留它在栈上的数据，直到func(n-1)先返回，这样就出现栈溢出的异常。

对于尾递归来说，我们从上面的尾递归程序可以看出将上一次的计算结果放到调用参数里，func2(n, total)必须要像以前那样，必须等到func2(n - 1, n * total)返回再计算自己，它的值其实就是func2(n - 1, n * total)的返回值。所以在func2(n, total) 等待 func2(n - 1, n * total)调用之前，就可以把自己栈上的数据销毁了。形象地说，只有向下计算，不需要再向上返回了。理论上说，尾递归不会出现栈溢出的异常。具体的尾递归在编译器的实现方式，需要编译器来支持。

现在有一个问题，Java现在支持尾递归吗？我们可以来运行一下上面的程序，发现没什么卵用。。

## 使用Stream延迟计算优化尾递归

如果我们对JDK8新加入的[Stream](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/stream/package-summary.html)有所了解的话，可以得知Stream支持延迟计算，比如Stream的Laziness-seeking特点是这样介绍的：

```
Many stream operations, such as filtering, mapping, or duplicate removal, can be implemented lazily, exposing opportunities for optimization
```

所以我们可以利用Stream将递归调用延迟化来避免栈帧的创建。Stream可以利用iterate方法生成无限长度的Stream，它有两个参数，第一个初始值，第二个是指定的生成函数，UnaryOperator类型，下面是利用iterate打印正整数的例子：


```Java
Stream.iterate(1, item -> item + 1).limit(10).forEach(System.out::println);
```

此时我们需要考虑以下几点：

1. 递归之间的关系表示
2. 判断递归结束标志
3. 获取递归计算结果
4. 触发递归函数

所以现在可以设计如下函数接口，用`@FunctionalInterface`注解声明：

```Java
@FunctionalInterface
public interface TailRecursion<T> {

    /**
     * 用于递归栈帧之间的连接,惰性求值
     *
     * @return 下一个递归栈帧
     */
    TailRecursion<T> apply();

    /**
     * 判断当前递归是否结束
     *
     * @return 默认为false
     */
    default boolean isFinished() {
        return false;
    }

    /**
     * 获得递归结果,只有在递归结束才能调用,这里默认给出异常,通过工具类的重写来获得值
     *
     * @return 递归最终结果
     */
    default T getResult() {
        throw new Error("递归还没有结束,调用获得结果异常!");
    }

    /**
     * 执行一系列的递归,因为栈帧只有一个,所以使用findFirst获得最终的栈帧,接着调用getResult方法获得最终递归值
     *
     * @return 及早求值,获得最终递归结果
     */
    default T invoke() {
        return Stream.iterate(this, TailRecursion::apply)
                .filter(TailRecursion::isFinished)
                .findFirst()
                .get()
                .getResult();
    }
}
```

上面的函数接口中，`apply()`方法和iterate的第二个参数`UnaryOperator`作用一致，用于将所有的递归计算放入到Stream里面。`isFinished()`用于判断是否计算结束，对于递归的中间步骤，该方法返回false，对于最后一步，该方法返回true。`getResult()`用于获取递归的最终结果（仅当递归结束）。

`invoke`方法则是最重要的一个方法，它会将所有的递归操作通过apply方法连接起来，通过没有栈帧的尾调用得到最后的结果。利用Stream类型提供的iterate方法，在所有的递归调用中，只有最后一个递归调用会在isFinished中返回true，当它被调用时，也就意味着整个递归调用链的结束。最后，通过findFirst来返回这个值。


**对尾递归调用的统一包装**

前面说到尾递归，用尾递归的一般形式来表示：


```Java
void p(参数列表) {
    if (递归终止条件成立) 直接求解；    // 递归终止条件
    else p(较小的参数)                  // 此处必须为尾调用
}
```

所以我们需要设计对上面尾递归函数式接口包装一下，使其能够符合用尾递归进行计算的一般形式，主要有两点：

- 调用下次递归
- 用来结束一系列的递归操作，得到最终的结果

下面是具体包装实现：

```Java
public class TailInvoke {

    /**
     * 获得当前递归的下一个递归
     *
     * @param nextFrame 下一个递归
     * @param <T>       T
     * @return 下一个递归
     */
    public static <T> TailRecursion<T> call(final TailRecursion<T> nextFrame) {
        return nextFrame;
    }

    /**
     * 结束当前递归，重写对应的默认方法的值,完成状态改为true,设置最终返回结果,设置非法递归调用
     *
     * @param value 最终计算结果
     * @param <T>   T
     * @return 一个isFinished状态true的尾递归, 外部通过调用接口的invoke方法及早求值, 启动递归求值。
     */
    public static <T> TailRecursion<T> done(T value) {
        return new TailRecursion<T>() {
            @Override
            public TailRecursion<T> apply() {
                throw new Error("递归已经结束,非法调用apply方法");
            }

            @Override
            public boolean isFinished() {
                return true;
            }

            @Override
            public T getResult() {
                return value;
            }
        };
    }
}
```

我们用上面封装的包装类来求解阶乘问题，代码如下：

```Java
public static TailRecursion<Long> factorialTailRecursion(final long n, final long total) {
    if (n == 1)
        return TailInvoke.done(total);
    else
        return TailInvoke.call(() -> factorialTailRecursion(n - 1, n * total));
}
```

函数式编程有一个概念，叫做柯里化（currying），意思是将多参数的函数转换成单参数的形式。这里也可以使用柯里化。

```Java
public static long factorial(final long number) {
    return factorialTailRecursion(number, 1).invoke();
}
```

## References：

- [尾调用优化](http://www.ruanyifeng.com/blog/2015/04/tail-call.html)
- [说说尾递归](https://www.cnblogs.com/catch/p/3495450.html)
- [Currying](https://en.wikipedia.org/wiki/Currying)
- [递归](https://time.geekbang.org/column/article/41440)
- [A Better Javascript Memoizer](http://unscriptable.com/2009/05/01/a-better-javascript-memoizer/)
- 数据结构（c语言版）（第二版）
- [Java8函数之旅 (六) -- 使用lambda实现Java的尾递归](https://www.cnblogs.com/invoker-/p/7723420.html)
- [Java8函数之旅 (七) - 函数式备忘录模式优化递归](https://www.cnblogs.com/invoker-/p/7728452.html)
- [详解Java 8中Stream类型的“懒”加载](https://www.cnblogs.com/heimianshusheng/p/5674372.html)
- [[Java 8] (7) 利用Stream类型的"懒"操作](https://blog.csdn.net/dm_vincent/article/details/40503685)
- [[Java 8] (8) Lambda表达式对递归的优化(上) - 使用尾递归](https://blog.csdn.net/dm_vincent/article/details/40581859)
- [Stream](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/stream/package-summary.html)
