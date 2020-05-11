---
title: HttpMessageConverter转换问题
categories: [Java, Spring]
tags: [java, Spring]
author: Mingshan
date: 2020-05-12
---

熟悉SpringMVC的同学都清楚接口的请求（request）与响应（response）涉及序列化与反序列化操作，如果我们想根据项目需求做点定制化操作，保险起见我们需要了解下`HttpMessageConverter`接口工作流程及一些注意事项（仅针对HTTP）。

<!-- more -->

## Message Conversion

SpringMVC的请求与响应涉及HTTP协议，而HTTP请求和响应的传输是字节流，所以用Java编写的服务器必然涉及解析转换字节流的过程，当然这个过程Spring已经帮我们屏蔽了，我们接收请求时加一个注解`@RequestBody`，响应时我们直接返回实体，十分方便。这个过程需要`HttpMessageConverter`参与，`HttpMessageConverter`是一个接口，定义了几个方法，如下所示：

```
public interface HttpMessageConverter<T> {
    /**
     * 根据mediaType判断clazz是否可读
     */
	boolean canRead(Class<?> clazz, @Nullable MediaType mediaType);

    /**
     * 根据mediaType判断clazz是否可写
     */
	boolean canWrite(Class<?> clazz, @Nullable MediaType mediaType);

    /**
     * 获取受支持的mediaType
     */
	List<MediaType> getSupportedMediaTypes();

    /**
     * 将HttpInputMessage流中的数据绑定到clazz中
     */
	T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
			throws IOException, HttpMessageNotReadableException;

    /**
     * 将t对象写入到HttpOutputMessage流中
     */
	void write(T t, @Nullable MediaType contentType, HttpOutputMessage流中 outputMessage)
			throws IOException, HttpMessageNotWritableException;

}

```


熟悉IO编程的同学对`read/write`模式应该非常熟悉，这个也正是对应将数据从流中读取出来和将数据写入到流中，`MediaType`在HTTP中对应`Content-Type`，是从HTTP的header里面取的，这个也无需多说，详情可以参考RFC文档：https://tools.ietf.org/html/rfc7231#section-3.1.1.1。

关于`HttpMessageConverter`的实现，Spring为我们提供了超级多的实现，具体实现类参考：[HttpMessageConverter Implementations](https://docs.spring.io/spring/docs/5.2.6.RELEASE/spring-framework-reference/integration.html#rest-message-conversion)，比较常用的比如：`StringHttpMessageConverter`，`MappingJackson2HttpMessageConverter`等，同时，`RestTemplate`里面也是用到这些转换器的，这里提一下。我们来关注一下`AbstractHttpMessageConverter`抽象类，其他的具体实现类是继承了这个类的。

`AbstractHttpMessageConverter`提供了一个Map类型的变量`supportedMediaTypes`，用来表示HTTP请求与响应支持的`Content-Type`类型，所以在自定义`HttpMessageConverter`时，需要注意这个变量的赋值情况，否则会出现`'Content-Type' cannot contain wildcard type '*'`这种错误，此时需要正确赋值，具体赋值方式，可参考源码。

另一个需要关注的实现类是`MappingJackson2HttpMessageConverter`，我们知道Spring默认使用Jackson来做序列化与反序列化的，比如我们想格式化响应实体的时间格式，以及忽略值为空的字段等，我们可以利用`MappingJackson2HttpMessageConverter`来完成我们自定义需求，下面是一个示例：

```
@Bean
public MappingJackson2HttpMessageConverter getMappingJackson2HttpMessageConverter() {
    MappingJackson2HttpMessageConverter mappingJackson2HttpMessageConverter = new MappingJackson2HttpMessageConverter();
    // 设置日期格式
    ObjectMapper objectMapper = new ObjectMapper();
    SimpleDateFormat smt = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    objectMapper.setDateFormat(smt);
    objectMapper.disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);
    objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    objectMapper.setSerializationInclusion(JsonInclude.Include.NON_EMPTY);
    mappingJackson2HttpMessageConverter.setObjectMapper(objectMapper);
    return mappingJackson2HttpMessageConverter;
}
```

## References：

- https://docs.spring.io/spring/docs/5.2.6.RELEASE/spring-framework-reference/integration.html#rest-message-conversion
