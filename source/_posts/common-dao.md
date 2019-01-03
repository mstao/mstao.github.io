---
title: 基于java中的泛型与反射构建通用DAO层
categories: Java
tags: java
author: Mingshan
date: 2017-07-23
---
在利用hibernate写通用DAO层时需要获取泛型的类型，比如我在写hql的update语句时需要获取泛型的实体类，由于泛型有擦除机制，所以与需要在运行过程中获取泛型的类型产生了矛盾。此时需要利用反射机制来实现此功能，下面来看一个小例子。

> 首先建一个实体类Dog
```java
import java.io.Serializable;

public class Dog implements Serializable {
    private static final long serialVersionUID = 8108340856807454651L;
	  private int age;
	  private String name;

  	public int getAge() {
  		return age;
  	}

  	public void setAge(int age) {
  		this.age = age;
  	}

  	public String getName() {
  		return name;
  	}

  	public void setName(String name) {
  		this.name = name;
  	}
}
```
<!-- more -->
>  然后写一个基类，在此类中可以获取泛型的类型
```java
package com.han.one;

import java.lang.reflect.Field;
import java.lang.reflect.ParameterizedType;
import java.util.HashMap;
import java.util.Map;

/**
 * 通过反射获取泛型实例
 */
public  class Genericity<T> {

    @SuppressWarnings("rawtypes")
    protected Class clazz;

    @SuppressWarnings("unchecked")
    /**
    * 把泛型的参数提取出来的过程放入到构造函数中写，因为
    * 当子类创建对象的时候，直接调用父类的构造函数
    */
    public Genericity() {
    	// 通过反射机制获取子类传递过来的实体类的类型信息
        ParameterizedType type = (ParameterizedType) this.getClass().getGenericSuperclass();
        //得到t的实际类型
        clazz = (Class<T>) type.getActualTypeArguments()[0];
    }

    /**
     * 获取指定实例的所有属性名及对应值的Map实例
     * @param entity 实例
     * @return 字段名及对应值的Map实例
     */
     protected Map<String, Object> getFieldValueMap(T entity) {
        // key是属性名，value是对应值
        Map<String, Object> fieldValueMap = new HashMap<String, Object>();

        // 获取当前加载的实体类中所有属性
        Field[] fields = this.clazz.getDeclaredFields();

        for (int i = 0; i < fields.length; i++) {
            Field f = fields[i];
            // 属性名
            String key = f.getName();
            //属性值
            Object value = null;
            // 忽略序列化版本ID号
            if (! "serialVersionUID".equals(key)) {
            	// 取消Java语言访问检查
            	f.setAccessible(true);
                try {
                    value =f.get(entity);
                } catch (IllegalArgumentException e) {
                    e.printStackTrace();
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
                fieldValueMap.put(key, value);
            }
        }
        return fieldValueMap;
    }
}
```
在此类的构造方法中利用反射获取子类传递过来的实体类的类型信息，getFieldValueMap方法用于获取该实体类的属性信息

> 最后写一个测试类
```java
package com.han.one;

import java.util.Map;
import java.util.Set;

/**
 * 测试：通过反射获取运行过程中泛型实例
 *
 */

public class GenericityTest extends Genericity<Dog> {

    public static void main(String[] args) {

        GenericityTest gt = new GenericityTest();

        //赋值
        Dog  dd = new Dog();
        dd.setAge(1);
        dd.setName("旺财");

        Map<String,Object> map = gt.getFieldValueMap(dd);
        //遍历
        Set<Map.Entry<String, Object>> entrySet = map.entrySet();
        for (Map.Entry<String, Object> entry : entrySet) {
            String key = entry.getKey();
            Object value = entry.getValue();
            System.out.println(key + "---" + value);
        }
    }
}
```
在这个测试类中，此类继承基类，并向其传递实体类，这样在父类中就可以通过反射获取泛型的类型了。

以此为基础，就可以构建通用的DAO了，代码如下：
```java
public class BaseDaoImpl<T>  implements IBaseDao<T> {
    @Autowired
    private SessionFactory  sessionFactory;

    @SuppressWarnings("rawtypes")
    private final Class clazz;

    @SuppressWarnings("unchecked")
    public BaseDaoImpl() {
        // 通过反射机制获取子类传递过来的实体类的类型信息
        ParameterizedType type = (ParameterizedType) this.getClass().getGenericSuperclass();
        clazz = (Class<T>) type.getActualTypeArguments()[0];
    }

    @Override
    public boolean update(T t) {
        StringBuffer stringBuffer = new StringBuffer();
        stringBuffer.append("update " + this.clazz.getSimpleName());
        stringBuffer.append(" u set u.itemTitle=:itemTitle ,u.itemContent=:itemContent,u.addTime=:addTime,u.isImage=:isImage,u.isPublish=:isPublish,u.author=:author  where u.id=:id");
        System.out.println(stringBuffer.toString());
        Query query  = sessionFactory.getCurrentSession().createQuery(stringBuffer.toString());
        query.setProperties(t);
        return (query.executeUpdate()>0);
    }
}
```
这里只是在update方法中利用反射获取实体类，通过拼装hql语句来达到重用目的，当然参数也可以动态获取，这里只是个小例子。

> 总结

java中泛型与反射的应用很广泛，想要完全掌握不是那么容易，多写多练是比较好的方式^_^
