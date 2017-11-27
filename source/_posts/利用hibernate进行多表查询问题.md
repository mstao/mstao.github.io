---
title: 利用hibernate进行多表查询问题
categories: Java
tags: Hibernate
author: Mingshan
---
在Hibernate框架中，一个实体类映射为一个数据库表，在进行多表查询时,如何将不同表中的数据整合起来，并且映射为一个实体类是利用Hibernate进行多表查询的关键，根据我的理解，先将代码整理一下：

> 实体类
```java
@Entity
@Table(name = "ps_trends")
public class Trends implements Serializable {
    private static final long serialVersionUID = -2228382525594394975L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private int id;

    @Column(name = "item_title")
    private String itemTitle;

    @Column(name = "item_content")
    private String itemContent;

    @Column(name = "type_id")
    private int  typeId;

    @Column(name = "add_time")
    private String addTime;

    @Column(name = "view_count")
    private int viewCount;

    @Column(name = "is_image")
    private int isImage;

    @Column(name = "is_publish")
    private int isPublish;

    //临时属性
    @Transient
    private String itemTypeFlag;

    @Transient
    private String itemTypeName;

    public Trends() {}

    public Trends(int id, String itemTitle, String itemContent, String addTime, int viewCount,
    	  String itemTypeName,String itemTypeFlag) {
        super();
        this.id = id;
        this.itemTitle = itemTitle;
        this.itemContent = itemContent;
        this.addTime = addTime;
        this.viewCount = viewCount;
        this.itemTypeName = itemTypeName;
        this.itemTypeFlag = itemTypeFlag;

    }
    setter ，getter方法
}
```
 <!-- more -->
这里有两个属性注解为Transient，因为它们不是主表的映射字段。同时写一个有参构造方法，构造方法的参数列表即为要查询的映射字段。

> DaoImpl方法
```java
@Override
public Trends findTrendsInfoById(int id) {
    String hql="select new com.primaryschool.home.entity.Trends(t.id,t.itemTitle,t.itemContent,t.addTime,t.viewCount,tt.itemTypeName,tt.itemTypeFlag)from Trends t,TrendsType tt  where tt.id=t.typeId and t.id=? and t.isPublish=1";
    Query query=sessionFactory.getCurrentSession().createQuery(hql);
    query.setInteger(0, id);
    return (Trends) query.uniqueResult();
}
```
在findTrendsInfoById(int id)方法中，hql语句有些特别，它是将两个表的需要字段传入到Trends实体类的构造方法中，这样就可以直接利用getter方法进行取值了。
