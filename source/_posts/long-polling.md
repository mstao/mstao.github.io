---
title: 网站消息推送之long polling（长轮询）初探
categories: php
tags: php
author: Mingshan
date: 2017-07-09
---
网站的消息推送功能应用很广泛，比如论坛，问答网站等等都需要推送消息，那么采用什么样的推送方式更加便捷，更加节省服务器资源呢，这个需要根据网站的流量和规模来决定，因为long polling是我最先接触到的，我就来谈谈它吧。

长轮询初看像是轮流查询的意思，其实不是，它是客户端通过ajax发出请求，然后客户端挂起，等待服务器端响应，服务器端会检测有无新消息，如果有消息，服务器端会将新消息推送给客户端，结束本次请求，如果在有效请求期内没有新消息出现，那么会一直检测有无新消息出现。连接会保持一段时间周期直到数据或状态改变或者时间过期，通过这种机制来减少无效的客户端和服务器间的交互。

虽然长轮循比传统的轮询性能会有些提高，但在服务器端数据变化非常频繁的情况下，两者的性能并不能差多少，因为都是客户端先请求，服务器再响应，只是两者服务器端响应的机制不同。
下面来说说代码，服务器端我用的是php，客户端用的是jQuery


服务器端代码：

```php
/**
 * @desc ajax长轮询 来获取通知消息信息
 * @return 通知信息数量>o
 */
public function longPolling() {
    if(!$_GET['timed']) exit();
    date_default_timezone_set("PRC");
    session_write_close(); //防止session访问互斥问题
    set_time_limit(0);//无限请求超时时间
    $timed = $_GET['timed'];
    while (true) {
        sleep(3); // 休眠3秒
        //判断有无新通知出现
         $no_count=D('Notifications')->getNotificationsCount($this->uid);
         if ($no_count>0) {
            $responseTime = time();
            // 返回数据信息，请求时间、返回数据时间、耗时
            $content=array(
                'result'         =>$no_count,
                'reponse_time'   =>$responseTime,
                'request_time'   =>$timed,
                'use_time'       =>($responseTime - $timed)
            );
            echo $this->ajaxReturn($content);
            exit();
        } else { // 模拟没有数据变化，将休眠 hold住连接
            sleep(13);
            exit();
        }
    }
}

```
<!-- more -->
从服务器段代码可以看出，里面有个while(true){}死循环，只有有新信息或者连接失效时会退出循环。

客户端代码：
```javascript
$(function(){
	/**
	 * 消息的处理 递归调用
	 */
	 (function longPolling() {  
         $.ajax({  
             url: MODULE+"/Notifications/longPoll",  
             data: {"timed": Date.parse(new Date())/1000},  
             dataType: "json",  
             timeout: 70000,//单位毫秒
             error: function (XMLHttpRequest, textStatus, errorThrown) {  

            	 if (textStatus == "timeout") { // 请求超时  
                     longPolling(); // 递归调用  
                 } else { // 其他错误，如网络错误等  
                     longPolling();  
                 }  
             },  
             success: function (data, textStatus) {  
                 //此时已有消息过来了，将消息数量显示
                 $('.nav-counter').text(data.result);
                 if (textStatus == "success") {
                         // 请求成功，继续请求
                    longPolling();
                 }  
             }  
         });  

     })();
});
```

客户端代码调用ajax进行处理，逻辑已经很清楚了。

以上就是我对long polling的理解，虽然长轮询较轮询有了不错的改进，但还是会消耗很多的服务器资源，并不是十分理想的网站消息推送方案。
