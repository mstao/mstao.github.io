---
title: 分组计算问题
tags: [算法]
author: Mingshan
categories: [算法]
date: 2019-04-02
---

在平时编程中遇到这样一个问题：给定一个list，如何将list中的元素的state属性只保留最后一个为true, 其它统统置为false(state只是一个例子)？假设元素中依据type属性进行分类，如何根据type属性将上面的操作进行单独计算？

<!-- more -->

初看上面的问题很熟悉，就是如何将一批数据进行处理，如果再遇到分组情况，如何做到计算性能最高。我们还是先来考虑单独计算情况。

假设我们现在有一个实体Person，然后我们再构造一些测试数据：

```Java
public class Person {
    private String name;
    public int age;
    public String type;
    public boolean state;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public boolean isState() {
        return state;
    }

    public void setState(boolean state) {
        this.state = state;
    }

    public String getType() {
        return type;
    }

    public void setType(String type) {
        this.type = type;
    }
    
    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", type='" + type + '\'' +
                ", state=" + state +
                '}';
    }
}
```

模拟数据：

```Java
Person person1 = new Person();
person1.setName("1");
person1.setState(true);
Person person2 = new Person();
person2.setName("2");
person2.setState(false);
Person person3 = new Person();
person3.setName("3");
person3.setState(true);
Person person4 = new Person();
person4.setName("4");
person4.setState(true);
List<Person> persons = new ArrayList<>();
persons.add(person1);
persons.add(person2);
persons.add(person3);
persons.add(person4);
```

我们可以清晰地看到每一个实体都有一个state属性，现在我们要做的是将list的元素state属性只保留最后一个为true, 其它统统置为false，可以用JDK8的流操作来做，代码如下:


```Java
List<Person> st = new ArrayList<>();
persons
    .stream()
    .collect(partitioningBy(Person::isState))
    .forEach((k, v) -> {
        // 如果key为true
        if (k) {
            v.forEach(item -> item.setState(false));
            v.get(v.size() - 1).setState(true);
            st.addAll(v);
        } else {
            st.addAll(v);
        }
    });
```

在上述代码中，我们首先引入了stream，这个就没必要在说了，接着`collect(partitioningBy(Person::isState))`看着比较陌生，partitioningBy的作用是分区，所谓分区，就是根据传入的条件将集合一份分为二，key为true和true，所以上面的操作就是将集合中的元素根据state一份为二，我们只需对state为true的元素处理就可以了，接着我们遍历上面分区产生的map，只处理state为true的情况，然后再遍历其内部集合，将state全部设置为false，然后再将最后一个设置为true，最后将处理过后的数据放入到新集合中即可。


上面的操作很简单，也很好想，毕竟遍历好多次list，效率其实是比较低的。试想一下，有么有一种比较简便的方法，只遍历list一次实现上述操作呢？答案是可以的，不过需要引入一个指针来指向上次state为true的元素，随着遍历的进行，依次将元素state为true的改为false，并且将指针指向上一个元素，直至遍历到最后一个state为true的元素，并将上一个记录的元素的state改为false即可，这样最多只遍历一次，符合我们的预期，代码如下：

```Java
int[] lastIndex = new int[1];

for (int i = 1; i <= persons.size(); i++) {
    Person person = persons.get(i - 1);
    if (person.isState()) {
        int index = lastIndex[0];
        if (index != 0) {
            persons.get(index - 1).setState(false);
        }
        lastIndex[0] = i;
    }
}
```

回到最初我们的问题，如果要根据type进行分组处理，该怎么做呢？这个时候我们用到stream中分组就可以实现了，代码如下：

```Java
List<Person> tempList = new ArrayList<>(persons.size());
persons
    .parallelStream()
    .collect(Collectors.groupingBy(Person::getType))
    .forEach((k, v) -> {
        int[] lastIndex = new int[1];
        for (int i = 1; i <= v.size(); i++) {
            Person item = v.get(i - 1);
            if (item.isState()) {
                int index = lastIndex[0];
                if (index != 0) {
                    v.get(index - 1).setState(false);
                }
                lastIndex[0] = i;
            }
        }
        tempList.addAll(v);
        lastIndex = null;// help gc
    });
```

References：

- [partitioningBy](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/stream/Collectors.html#partitioningBy(java.util.function.Predicate))


[<font size=3 color="#409EFF">向本文提出修改或勘误建议</font>](https://github.com/mstao/mstao.github.io/blob/hexo/source/_posts/groupby-calculate.md)