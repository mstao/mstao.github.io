---
title: 设计模式之适配器模式
tags: [设计模式,适配器模式,java]
author: Mingshan
categories: 设计模式
date: 2018-6-25
---

在我们日常生活中可以看到许多例子，例如我们的手机需要用充电器来充电，因为220v电压手机是承受不住的；笔记本电脑连接投影仪可以是HDMI，需要转接头等等例子，从这些例子中我们可以发现目标对象与源对象无法直接交互，需要一个中间层来作为一个桥梁达到让两者完美交互的效果，从这种就可以看到适配器模式的影子了。

<!-- more -->

那么什么是适配器模式呢? 将一个类的接口转换成客户希望的另外一个接口。Adapter模式使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。

适配器包含类适配器和对象适配器，类适配器是类间继承，对象适配器是类的关联关系，这是两者的区别。


我们先来看看适配器模式的角色
1. Target目标角色
   
    该角色定义把其他类转化为何种接口，也就是我们最终期望的接口，需要Adapter实现该接口
2. Adaptee源角色

    想把谁转换为目标角色，此时就是源角色
3. Adapter适配器角色

    适配器模式的核心角色，它的职责非常简单，将源角色转化为目标角色

### 类适配器模式

通过继承来实现适配器模式，由于Java是属于单继承，所以这个使用限制很大。

Adaptee源角色，里面包含原来的业务逻辑

```Java
/**
 * 适配器源角色
 * 
 * @author mingshan
 *
 */
public class Adaptee {

    /**
     * 原有的业务逻辑
     */
    public void doSomething() {
        System.out.println("源角色 do something。。。");
    }
}

```

Target目标角色， 包含现有的业务逻辑

```Java
/**
 * 适配器目标角色
 * 
 * @author mingshan
 *
 */
public interface Target {

    /**
     *  目标角色有自己的方法
     */
    void request();
}


/**
 * 目标角色实现类， 现有的业务逻辑
 * 
 * @author mingshan
 *
 */
public class ConcreteTarget implements Target {

    @Override
    public void request() {
        System.out.println("xxxxxxxxxxxxxxxxxx");
    }

}


```

适配器角色，需要继承源角色，拿到其里面的方法，然后实现目标角色接口，让源角色与目标角色进行交互

```Java
/**
 * 适配器角色
 * 
 * @author mingshan
 *
 */
public class Adapter extends Adaptee implements Target {

    @Override
    public void request() {
        super.doSomething();
    }

}
```

测试一下

```Java
/**
 * 类适配器 Test
 * 
 * @author mingshan
 *
 */
public class Client {

    public static void main(String[] args) {
        // 原有的业务逻辑
        Target target = new ConcreteTarget();
        target.request();
        // 现在增加了适配器角色后的业务逻辑
        Target target2 = new Adapter();
        target2.request();
    }
}

```

### 对象适配器模式

对象适配器是通过类的关联关系来进行的，是为了解决类适配器模式的问题而出现的，它比类适配器模式灵活，扩展性强，实际运用的比较多。

这里只解释一些Adapter适配器角色的实现，其他的和类适配器模式一致。

通过构造器将两个源角色传递进来，实现Target接口，然后就可以进行交互啦

```Java
/**
 * 适配器角色
 * 
 * @author mingshan
 *
 */
public class Adapter implements Target {

    private Adaptee1 adaptee1;
    private Adaptee2 adaptee2;

    public Adapter(Adaptee1 adaptee1, Adaptee2 adaptee2) {
        this.adaptee1 = adaptee1;
        this.adaptee2 = adaptee2;
    }

    @Override
    public void request() {
        adaptee1.doSomething();
        adaptee2.doSomething();
    }
    
}

```

### JDK中用到适配器的地方

在阅读JDK并发相关的源码时，发现FutureTask 的构造函数既可以传Runnable，又可以传Callable，如下：

```Java
FutureTask(Runnable runnable, V result)
FutureTask(Callable<V> callable) 
```

我们知道Callable 是带返回值的，并且是在JDK1.5引入的，而Runnable很早就有了，且无返回值，那么FutureTask是如何支持这两个接口的呢？

其实在FutureTask的内部，实际运行的Callable，调用的也是Callable的call方法。所以我们就可以猜想到，应该是在内部将Runnable转为Callable了，事实就是如此，我们来看下构造器代码：

```Java
public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}
```

发现调用了`Executors.callable`来将Runnable转成了callable，代码如下

```Java
public static <T> Callable<T> callable(Runnable task, T result) {
    if (task == null)
        throw new NullPointerException();
    return new RunnableAdapter<T>(task, result);
}
```

上面代码所示，是返回了一个RunnableAdapter对象，该类源码如下：

```Java
/**
 * A callable that runs given task and returns given result
 */
static final class RunnableAdapter<T> implements Callable<T> {
    final Runnable task;
    final T result;
    RunnableAdapter(Runnable task, T result) {
        this.task = task;
        this.result = result;
    }
    public T call() {
        task.run();
        return result;
    }
}
```

RunnableAdapter其实是实现了Callable接口，在call方法内部其实是调用Runnable的run方法来调用的，返回传入的result。


### 代码参考

https://github.com/mstao/java-explore/tree/master/DesignPattern/src/pers/han/adapter


[<font size=3 color="#409EFF">向本文提出修改或勘误建议</font>](https://github.com/mstao/mstao.github.io/blob/hexo/source/_posts/design-pattern-adapter.md)