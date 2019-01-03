---
title: SpringBoot与RabbitMQ结合使用
tags: [RabbitMQ, java, SpringBoot]
author: Mingshan
categories: [RabbitMQ]
date: 2017-11-25
---

上一篇文章中是使用amqp-client来操作RabbitMQ的，但我们平常用SpringBoot比较多，SpringBoot也整合了RabbitMQ，用起来是十分方便的。

首先，我们先添加依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

然后在application.properties文件添加RabbitMQ的配置：

```
# rabbitmq
spring.application.name=spring-boot-rabbitmq
spring.rabbitmq.host=127.0.0.1
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
spring.rabbitmq.publisher-confirms=true

```

## Hello
我们先来个hello，首先在发送方注入**AmqpTemplate**，像其它Spring Framework提供的高级抽象一样， Spring AMQP 提供了扮演核心角色的模板. 定义了主要操作的接口称为AmqpTemplate. 这些操作包含了发送和接收消息的一般行为。


在配置类RabbitConfig类生成一个名为hello_rq的队列

```
@Configuration
public class RabbitConfig {

    @Bean
    public Queue helloQueue() {
        return new Queue("hello_rq");
    }

}

```


**发送方：**


```java
@Component
public class Sender {
    private final static Logger logger = LoggerFactory.getLogger(Sender.class);

    @Autowired
    private AmqpTemplate amqpTemplate;

    public void send() {
        String context = "hello " + new Date();
        logger.info("Sender : " + context);
        this.amqpTemplate.convertAndSend("hello_rq", context);
    }
}

```
这里我们用了convertAndSend方法进行消息的发送，将消息发送到hello_rq的队列中

**接收方：**

```java
@Component
@RabbitListener(queues = "hello_rq")
public class Receiver {
    private final static Logger logger = LoggerFactory.getLogger(Receiver.class);

    @RabbitHandler
    public void process(String hello) {
        logger.info("Receiver1  : " + hello);
    }

}

```
在接收方利用@RabbitListener注解来监听队列，利用@RabbitHandler来处理消息

<!-- more -->

**测试类：**


```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest
public class HelloTest {
    @Autowired
    private Sender sender;

    @Test
    public void hello() throws Exception {
        sender.send();
    }
}

```

## 四种 Exchange Types

### fanout

fanout类型的Exchange路由规则非常简单，它会把所有发送到该Exchange的消息路由到所有与它绑定的Queue中。

**生产者代码：**

```java
@Component
public class FanoutProducer {
    private final static Logger logger = LoggerFactory.getLogger(FanoutProducer.class);

    @Autowired
    private AmqpTemplate amqpTemplate;

    public void send() {
        String message = "This is topic message ====!";
        logger.info("message => " + message);
        this.amqpTemplate.convertAndSend("ex.fanout", "", message);
    }

}

```
上面代码中的convertAndSend有三个参数，第一个参数设置exchange名称，第二个参数设置routingKey，第三个参数设置要发送的消息，由于是fanout，所以没必要设置routingKey的值，其实在源码中，也是最终调用channel.basicPublish方法的。源码如下：

```java
BasicProperties convertedMessageProperties = this.messagePropertiesConverter
		.fromMessageProperties(messageProperties, this.encoding);
channel.basicPublish(exchange, routingKey, mandatory, convertedMessageProperties, messageToUse.getBody());
```

**消费者代码：**

```java
@Component
@RabbitListener(bindings = @QueueBinding(value = @Queue,
        exchange = @Exchange(value = "ex.fanout", type = ExchangeTypes.FANOUT)
))
public class FanoutConsumerA {
    private final static Logger logger = LoggerFactory.getLogger(FanoutConsumerA.class);

    @RabbitHandler
    public void process(String message) {
        logger.info("ConsumerA Receiver :" + message);
    }
}

```

这里用@RabbitListener注解来监听，利用@RabbitHandler来处理消息。其中在@RabbitListener中又为队列，交换器和绑定的@QueueBinding 注解中指定参数，一个参数比较全的例子如下：

```
@RabbitListener(bindings = @QueueBinding(
        value = @Queue(value = "auto.headers", autoDelete = "true",
                        arguments = @Argument(name = "x-message-ttl", value = "10000",
                                                type = "java.lang.Integer")),
        exchange = @Exchange(value = "auto.headers", type = ExchangeTypes.HEADERS, autoDelete = "true"),
        arguments = {
                @Argument(name = "x-match", value = "all"),
                @Argument(name = "foo", value = "bar"),
                @Argument(name = "baz")
        })
)
public class HeadersConsumerA { {
    ...
}
```
注意队列的x-message-ttl 参数设为了10秒钟，因为参数类型不是String， 因此我们指定了它的类型，在这里是Integer。有了这些声明后，如果队列已经存在了，参数必须匹配现有队列上的参数。对于header交换器，我们设置binding arguments 要匹配头中foo为bar，且baz可为任意值的消息。 x-match 参数则意味着必须同时满足两个条件。

**测试类：**

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest
public class FanoutTest {
    @Autowired
    private FanoutProducer fanoutProducer;

    @Test
    public void test() {
        fanoutProducer.send();
    }
}
```


### direct

direct类型的Exchange路由规则也比较简单，它会把消息路由到那些binding key与routing key完全匹配的Queue中。

**生产者代码：**


```java
@Component
public class DirectProducer {
    private final static Logger logger = LoggerFactory.getLogger(DirectProducer.class);

    @Autowired
    private AmqpTemplate amqpTemplate;

    public void send() {
        String message = "This is topic message @====!";
        logger.info("message => " + message);

        // 参数意义
        // 第一个： exchange 名称
        // 第二个： routingKey
        // 第三个： 发送的消息
        this.amqpTemplate.convertAndSend("ex.direct", "error", message)里面的第二个参数为routingKey,设置为
    }
}
```
此时convertAndSend("ex.direct", "error", message)里面的第二个参数为routingKey,设置为error，说明消费者的bindingKey须为error才能接受到消息，其他的接收不到。

**消费者代码:**


```java
@Component
@RabbitListener(bindings = @QueueBinding(value = @Queue,
        exchange = @Exchange(value = "ex.direct", type = ExchangeTypes.DIRECT),
        key = "info"
))
public class DirectConsumerA {
    private final static Logger logger = LoggerFactory.getLogger(DirectConsumerA.class);

    @RabbitHandler
    public void process(String message) {
        logger.info("ConsumerA Receiver :" + message);
    }
}
```
在消费者A中，routingKey设置为info，自然接受不到消息了。

**测试类：**


```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest
public class DirectTest {
    @Autowired
    private DirectProducer directProducer;

    @Test
    public void test() {
        directProducer.send();
    }
}

```
### topic

由于direct的匹配规则需要完全配置，没有灵活性，所以topic就弥补了这一缺点， routingKey 必须是由点分隔的单词列表。这些单词可以是任何东西，但通常它们指定连接到消息的一些功能。一些有效的路由键例子：“ stock.usd.nyse ”，“ nyse.vmw ”，“ quick.orange.rabbit ”。在路由选择键中可以有任意数量的字，最多255个字节。

绑定键也必须是相同的形式。binding key中可以存在两种特殊字符“*”与“#”，用于做模糊匹配：

- "*" 可以代替一个字。
- "#" 可以代替零个或多个单词。

**生产者代码:**

```java
@Component
public class TopicProducer {
    private final static Logger logger = LoggerFactory.getLogger(TopicProducer.class);

    @Autowired
    private AmqpTemplate amqpTemplate;

    public void send() {
        String message = "This is topic message @====!";
        logger.info("message => " + message);

        // 参数意义
        // 第一个： exchange 名称
        // 第二个： routingKey
        // 第三个： 发送的消息
        this.amqpTemplate.convertAndSend("ex.topic", "quick.orange.rabbit", message);
    }
}
```
此时routingKey为quick.orange.rabbit，消费者可以对这个routingKey进行模糊匹配。

**消费者代码：**


```java
@Component
@RabbitListener(bindings = @QueueBinding(
        //value = @Queue(value = "myQueue", durable = "true"),
        value = @Queue, // 自动生成， 自动删除
        exchange = @Exchange(value = "ex.topic", ignoreDeclarationExceptions = "true", type = ExchangeTypes.TOPIC),
        key = "*.orange.*")
)
public class TopicConsumerA {
    private final static Logger logger = LoggerFactory.getLogger(TopicConsumerA.class);

    @RabbitHandler
    public void process(String message) {
        logger.info("ConsumerA Receiver :" + message);
    }

}
```
此时bindingKey为*.orange.*，那么可以与生产者的routingKey匹配，那么这个消费者可以接受到消息。

### headers

headers类型的Exchange不依赖于routing key与binding key的匹配规则来路由消息，而是根据发送的消息内容中的headers属性进行匹配。 在绑定Queue与Exchange时指定一组键值对；当消息发送到Exchange时，RabbitMQ会取到该消息的headers（也是一个键值对的形式），消费者会根据设置x-match设置的配置类型(all,any)来进行匹配。

**生产者代码：**


```java
@Component
public class HeadersProducer {
    private final static Logger logger = LoggerFactory.getLogger(TopicProducer.class);

    @Autowired
    private AmqpTemplate amqpTemplate;

    public void send() {
        String mes = "This is headers message ====!";
        logger.info("message => " + mes);

        // 构建消息
        Message message = MessageBuilder.withBody(mes.getBytes())
                .setContentType(MessageProperties.CONTENT_TYPE_TEXT_PLAIN)
                .setMessageId("123")
                .setHeader("xiaoming", "123456")
                .build();

        // 参数意义
        // 第一个： exchange 名称
        // 第二个： routingKey
        // 第三个： 发送的消息
        this.amqpTemplate.convertAndSend("ex.headers", "", message);
    }
}
```
生产者的代码比较特殊，首先我们需要构建消息，需要用到**Message Builder API**， MessageBuilder 和 MessagePropertiesBuilder提供了消息构建API; 它们提供了更加方便地创建消息和消息属性的方法:



```java
// 构建消息
Message message = MessageBuilder.withBody(mes.getBytes())
        .setContentType(MessageProperties.CONTENT_TYPE_TEXT_PLAIN)
        .setMessageId("123")
        .setHeader("xiaoming", "123456")
        .build();

// 或者 ============================

MessageProperties props = MessagePropertiesBuilder.newInstance()
        .setContentType(MessageProperties.CONTENT_TYPE_TEXT_PLAIN)
        .setMessageId("123")
        .setHeader("xiaoming", "123456")
        .build();
Message message2 = MessageBuilder.withBody(mes.getBytes())
        .andProperties(props)
        .build();

```
其中**MessageProperties.CONTENT_TYPE_TEXT_PLAIN**代表 **text/plain**, 当然还有其他格式：

```java
public static final String CONTENT_TYPE_BYTES = "application/octet-stream";

public static final String CONTENT_TYPE_TEXT_PLAIN = "text/plain";

public static final String CONTENT_TYPE_SERIALIZED_OBJECT = "application/x-java-serialized-object";

public static final String CONTENT_TYPE_JSON = "application/json";

public static final String CONTENT_TYPE_JSON_ALT = "text/x-json";

public static final String CONTENT_TYPE_XML = "application/xml";

public static final String SPRING_BATCH_FORMAT = "springBatchFormat";

public static final String BATCH_FORMAT_LENGTH_HEADER4 = "lengthHeader4";

public static final String SPRING_AUTO_DECOMPRESS = "springAutoDecompress";

public static final String X_DELAY = "x-delay";
```

**消费者代码：**


```java
@Component
@RabbitListener(bindings = @QueueBinding(value = @Queue,
        exchange = @Exchange(value = "ex.headers", type = ExchangeTypes.HEADERS),
        arguments = {
                @Argument(name = "x-match", value = "any"),
                @Argument(name = "xiaoming", value = "123456"),
                @Argument(name = "bbb", value = "1234567")
        }
))
public class HeadersConsumerA {
    private final static Logger logger = LoggerFactory.getLogger(HeadersConsumerA.class);

    @RabbitHandler
    public void process(String message) {
        logger.info("ConsumerA Receiver :" + message);
    }
}

```
此时headers的key-value形式映射为@Argument，x-match指明匹配模式，这里为any，代表只要有一个匹配到就可以接收到消息。


## 源码地址

你可以在这里看到本博文的源代码：

https://github.com/mstao/spring-boot-learning/tree/master/spring-boot-rabbitmq

## 参考

http://www.blogjava.net/qbna350816/archive/2016/08/13/431562.html
