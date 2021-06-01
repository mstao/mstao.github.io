---
title: i++与++i字节码分析
author: Mingshan
tags: JVM
categories: [JVM, Bytecode]
date: 2021-06-01
---

最近碰到几道关于i++与++i相关的题，我们从字节码角度来分析执行情况，该文章需要读者有字节码相关基础及了解方法调用机制。

# 分析

下面是一个题，请问下面代码输出什么？

```java
  public static void f() {
    int i = 1;
    System.out.println(i++ + i++);
  }
```

<!-- more -->

答案是3。相信有小伙伴会回答2，有此想法的是不了解其运算过程。

我们用`javap -c Test.class`先反编译下这个类，找到这个方法的字节码，如下所示：

```
  public static void f();
    Code:
       0: iconst_1
       1: istore_0
       2: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
       5: iload_0
       6: iinc          0, 1
       9: iload_0
      10: iinc          0, 1
      13: iadd
      14: invokevirtual #4                  // Method java/io/PrintStream.println:(I)V
      17: return
```

忽略`2: getstatic` 和 `14: invokevirtual`这两个调用输出的指令，我们来分析下其他相关指令：

![image](https://github.com/mstao/jvm-readings/blob/master/i%2B%2B.png?raw=true)

从图可以看出 在第一个i++之前，已经将数据从局部变量表加载到操作数栈了，如下指令：

```
5: iload_0
6: iinc          0, 1
```

当`6: iinc ` 执行完之后，局部变量表0位置的数改为2，然后第二个i++操作同样是这样操作的，先将局部变量表第0个位置的数加载到操作数栈，由于第一个i++已经将其改为2，此时再读取时，就是2。

操作数栈此时栈的元素为2，1。然后执行`iadd`指令，将栈顶两int型数值相加并将结果压入栈顶，此时栈顶元素就是3。进而输出3。

最后执行`return`，结束。


# 涉及到的字节码指令：


## iconst

当int取值-1~5时，JVM采用iconst指令将常量压入操作数栈中
int 取值 0~5 时，JVM采用 iconst_0、iconst_1、iconst_2、iconst_3、iconst_4、iconst_5指令将常量压入操作数栈中；

取值-1时，采用iconst_m1指令将常量压入操作数栈中。

当int取值 -128~127 时，JVM采用 bipush 指令将常量压入操作数栈中。

## iload
将一个局部变量加载到操作数栈
• 非静态方法：从iload_1开始的，默认第iload_0是this
• 静态方法：从iload_0开始的

## istore
将一个数值从操作数栈存储到局部变量表
istore_n，n代表局部变量表第几个位置，从0开始。

## iadd

将栈顶两int型数值相加并将结果压入栈顶

## return

从当前方法返回，void
