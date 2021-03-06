---
title: 关于首页feed流如何展示和数据库如何设计问题
categories: php
tags: php
author: Mingshan
date: 2017-07-15
---
最近在做一个简单的问答网站，首页的内容需要根据用户关注的话题和关注的问题等来展示最新的动态信息，刚开始的时候我想数据库表的设计是关键。

这里涉及到两个概念 **推模式（push）**和**拉模式（pull）**，这里有两篇大神分享的知识[ 新浪微博架构和FEED架构分析--人人架构_paper0023_新浪博客][1]和[微博feed系统的推(push)模式和拉(pull)模式和时间分区拉模式架构探讨][2]，讲解的比较清楚


  [1]: http://blog.sina.com.cn/s/blog_53b95aec0100ujim.html
  [2]: http://www.cnblogs.com/sunli/archive/2010/08/24/twitter_feeds_push_pull.html

> 简单来说，什么是推，什么是拉呢？



 1. **推模式**是一个用户发表了一条动态，那么后台就遍历关注该用户的所有用户，向他们的feed中推送一条动态
 2. **拉模式**与推模式相反，当用户刷新首页时，后台会遍历该用户关注的用户的动态信息，并将动态信息压入到该用户的feed中

简单介绍完推拉模式后，下面就要考虑数据库表该怎么设计了，我采用的是最简单的推模式，毕竟新手嘛，先掌握实现流程。


----------
首先设计feed表，这里我设计一个feed表来存储推送的信息，该表主要有以下几个字段

 - id  自增id
 - suid  推送者uid
 - ruid  接收者uid
 - item_id  推送的信息id
 - type   推送信息类型
 - add_time  推送时间

这是我感觉很简单的feed表，毕竟我那个问答站推送类型不是太多，当然还需要为这个表设计索引哦。
设计完数据库表后，下面该考虑后台推送逻辑和代码如何实现以及前台首页如何渲染feed流信息。


----------
后台我用的是PHP的ThinkPHP框架，新手表示该框架很好用，用该框架可以快速实现的我的想法，我感觉这一点还是很好的。首先在推送类型的选择中我选择了以下几种推送类型

 - 当一个话题下有新话题发起时，推送给关注该话题的用户
 - 当一个问题有回答时，推送给关注该问题的用户
 - 当一个话题的问题有新回答时，推送给关注该话题的用户

上面推送过程中会产生大量的重复信息，所以需要在推送时对推送信息进行过滤，以避免重复的推送信息出现。代码就是当上面的推送类型产生时，将信息写入到feed表中，这里并没有对推送用户进行筛选（对推送用户的筛选可以降低数据库的压力）。

前台渲染的话需要对信息进行排序整合，并对每一条动态信息进行标记，以便在模板渲染时匹配对应的模板。
我写的简单部分整合代码，需要对信息进行遍历整合（这里没写）

     //如果推送类型 为a  则代表推送信息类型为  用户关注的话题有关的问题或关注的问题产生的回答
     $aid=$val['item_id'];
     /**根据回答id获取获取与此回答有关的信息**/
     //获取推送人的uid
     $suid=$val['suid'];
     //获取当前用户对回答的赞同状态
     $upvote_status=$this->getUpvoteStatusByAid($aid, $uid);
    //获取feed流 回答信息
     $a_info_all=$this->getFeedAnswerInfo($aid);
     $question_id=$a_info_all[0]['question_id'];
     //根据问题id获取与此问题相关的话题信息
     $tinfo_a=D('Topic')->getFeedTopicByQuestion($question_id);
     //将话题信息追加到回答信息数组中
     $a_info_all[0]['topic']= $tinfo_a;
     //将当前用户对回答的赞同状态最佳到信息数组中
     $a_info_all[0]['upvote_status']=$upvote_status;
     //将整理后的信息添加到feed数组中，并做一个标记 a,以便在模板中判断解析
     $arr_fd['answer']=$a_info_all;
     $arr_fd['feed_flag']='a';
     $feed_return_arr[]=  $arr_fd;


这里 $feed_return_arr[]是一个三维数组，在模板渲染的时候要注意一下。
<!-- more -->
上面是我对feed流简单的思考，如果是真实网络环境下这种简单的实现有许多大问题，比如feed表数据量过大，是否设定一个时间阀对表中超过该时间阀的推送信息进行删除以减少feed表的记录量等等。所以这种方式并不适合真实的网络环境，需要将推拉模式结合进行使用。
