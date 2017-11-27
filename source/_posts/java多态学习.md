---
title: java多态学习
categories: Java
tags: java
author: Mingshan
---
在java多态中，引用与对象可以是不同的类型，如:

```java
A b=new B();

```
运用多态时，引用类型可以是实际对象类型的父类，即实际对象类型已经是一个比较具体的类，而引用类型则是一个比较抽象的类，任何extends过声明引用类型的对象都可以赋值给这个引用变量，这样就可以做出类似动态数组的东西，如下:

```java
Animal[] a=new Animal[2];
a[0]=new Dog();
a[1]=new Cat();
for(int i=0;i<a.length;i++){
    a[i].eat();
}

```
a数组里面可以放任何Animal的子类对象，调用的时候可以把子类都当作Animal来操作，实际上调用的是子类的方法，是不是很好玩呢→_→

当然，多态的应用很广泛呢，参数和返回类型也可以多态，如下:

```java
class Vet{
   public void giveShot(Anmial a){

      a.makeNoise();
   }
}

class Pet{
   public void a(){
      Vet v=new Vet();
      Dog dog=new Dog();
      Cat cat=new Cat();
      v.giveShot(dog);
      v.giveShot(cat);
   }

}
```
giveShot会接受任何Animal的子类的对象实例，根据传入的参数不同，会调用不同对象的方法。
