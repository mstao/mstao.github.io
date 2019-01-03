---
title: php json_encode函数使用遇到的问题
categories: php
tags: php
author: Mingshan
date: 2017-07-01
---
在php中有一个函数可以将数组转化为json数据存储格式，这个函数就是json_encode
但在使用这个函数时转化的格式不一致，比如：
```php
//关联二位数组
$a2=array(    
  '1'=>array('name'=>'john','age'=>'32'),
  '2'=>array('name'=>'tom','age'=>'22')
);
$json2=json_encode($a2);
echo $json2."<br>"; //{"1":{"name":"john","age":"32"},"2":{"name":"tom","age":"22"}}
//索引二维数组
$a3=array(
   array('name'=>'zz','age'=>'31'),
   array('name'=>'we','AGE'=>'12')
  );
 $json3=json_encode($a3);
 echo $json3."<br>";//[{"name":"zz","age":"31"},{"name":"we","AGE":"12"}]
```
关联二维数组和索引二维数组转化为json数据格式不同，这时在前台用js解析json的时候就有差别

 - 对于关联数组生成的json数据格式 ，在前台直接用js的eval()将其转化为json对象，然后根据{key:value}取值
 - 对于索引数组生成的json数据格式，用js的eval()转为json对象后，由于[]代表数组格式，所以遇到[]还是按照数组取值，遇到{key:value}这种形式的按照对象取值就行了

当数组维数多的时候需要根据转换后的json数据格式用js进行相应的解析，避免出错。
