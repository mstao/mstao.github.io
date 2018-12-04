---
title: java枚举探究
date: 2017-07-26 10:11:07
tags: java
categories: Java
---

在jdk1.5中引入枚举这个小功能，这个功能虽然用的不多，但是却给我们的开发带来很多便利，我们
今天来看看java的枚举是个什么样子。
## 枚举的主要操作方法
```java
protected Enum(String name,int ordinal)  //接受枚举的名称和枚举的常量创建枚举对象  
protected final Object clone()throws CloneNotSupportedException  //克隆枚举对象  
public final int compareTo(E o) //比较枚举与指定对象的顺序
public final boolean equals(Object other)  //比较两个枚举对象  
public final int hashCode()  //返回枚举常量的哈希码  
public final String name()  //返回枚举类的名称  
public final int ordinal()  //返回枚举常量的序号  
public static <T extends Enum<T>> T valueOf(Class<T> enum Type,String name)  //返回带指定名称的指定枚举类型的枚举常量  
```

## 先定义一个枚举，用enum关键字

```java
/**
 * 定义枚举
 * @author mingshan
 *
 */
public enum EnumTest {
	 MON, TUE, WED, THU, FRI, SAT, SUN;
}
```

这里将星期定义为枚举类型，但没有赋值，既然已经定义好了，那么就先测试一下吧。

```java
/**
 * 枚举测试
 * @author mingshan
 *
 */
public class Test {

	public static void main(String[] args) {

		//遍历枚举
		for(EnumTest e : EnumTest.values()) {
			System.out.println(e.toString());
		}

		System.out.println("我是分割线------");


		//switch 操作
		EnumTest fri = EnumTest.FRI;

		switch(fri){
			case MON :
				System.out.println("今天是星期一"); break;
			case FRI :
				System.out.println("今天是星期五"); break;
		    default :
		    	System.out.println("-----"); break;
		}

		//返回
		System.out.println(fri.getDeclaringClass());

		//利用compareTo进行比较
		switch (fri.compareTo(EnumTest.SAT)) {
		case -1:
			System.out.println("之前");
			break;
		case 1:
			System.out.println("之后");
			break;
		default:
			break;
		}
	}
}
```

我们可以遍历枚举，用java的foreach进行遍历，调用枚举的values方法获取定义的枚举列表，但当
我们编写自定义enum时，却不包含values这个方法，这个方法是当我门编译文件时，编译器自动帮我
们加上的。枚举还可以进行switch操作，可以对获取的枚举进行判断。利用compareTo函数进行比较两个
枚举的顺序

<!-- more -->

## 给 enum 对象加一下 value 的属性和 getValue() 的方法

```java
/**
 * 赋初值
 * 给 enum 对象加一下 value 的属性和 getValue() 的方法
 * @author mingshan
 *
 */
public enum EnumTest2 {
    MON(1), TUE(2), WED(3), THU(4), FRI(5), SAT(6) {
        @Override
	    public boolean isRest() {

	    	return true;
	    }
    },
    SUN(0) {
        @Override
	    public boolean isRest() {

	    	return true;
	    }

    };

	private int value;

	private EnumTest2(int value) {
		this.value = value;
	}

	public int getValue() {
		return value;
	}

	public boolean isRest() {
        return false;
    }


}

```

> 获取属性值

```java
/**
 * 获取属性值
 */
System.out.println(EnumTest2.FRI.getValue());
```

> EnumSet的使用

```java
//EnumSet的使用
EnumSet<EnumTest2> allOf = EnumSet.allOf(EnumTest2.class);

//遍历枚举
for (EnumTest2 enumTest2 : allOf) {
	System.out.println(enumTest2.toString());
}
```

> EnumMap的使用

```java
EnumMap<EnumTest2, Object> enumMap = new EnumMap<>(EnumTest2.class);

enumMap.put(EnumTest2.FRI, "星期五");
enumMap.put(EnumTest2.SUN, "星期天");

//遍历map
for (Entry<EnumTest2, Object> enumTest2 : enumMap.entrySet()) {
	System.out.println(enumTest2.getKey()+"---"+enumTest2.getValue());

}
```
