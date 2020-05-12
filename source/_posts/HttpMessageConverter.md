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

```Java
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
    void write(T t, @Nullable MediaType contentType, HttpOutputMessage outputMessage)
			throws IOException, HttpMessageNotWritableException;
}

```


熟悉IO编程的同学对`read/write`模式应该非常熟悉，这个也正是对应将数据从流中读取出来和将数据写入到流中，`MediaType`在HTTP中对应`Content-Type`，是从HTTP的header里面取的，这个也无需多说，详情可以参考RFC文档：https://tools.ietf.org/html/rfc7231#section-3.1.1.1。

关于`HttpMessageConverter`的实现，Spring为我们提供了超级多的实现，具体实现类参考：[HttpMessageConverter Implementations](https://docs.spring.io/spring/docs/5.2.6.RELEASE/spring-framework-reference/integration.html#rest-message-conversion)，比较常用的比如：`StringHttpMessageConverter`，`MappingJackson2HttpMessageConverter`等，同时，`RestTemplate`里面也是用到这些转换器的，这里提一下。我们来关注一下`AbstractHttpMessageConverter`抽象类，其他的具体实现类是继承了这个类的。

`AbstractHttpMessageConverter`提供了一个Map类型的变量`supportedMediaTypes`，用来表示HTTP请求与响应支持的`Content-Type`类型，所以在自定义`HttpMessageConverter`时，需要注意这个变量的赋值情况，否则会出现`'Content-Type' cannot contain wildcard type '*'`这种错误，此时需要正确赋值，这个报错信息在如下代码中：

```Java
/**
 * Set the {@linkplain MediaType media type} of the body
 +-,
 * as specified by the {@code Content-Type} header.
 */
public void setContentType(@Nullable MediaType mediaType) {
	if (mediaType != null) {
		Assert.isTrue(!mediaType.isWildcardType(), "'Content-Type' cannot contain wildcard type '*'");
		Assert.isTrue(!mediaType.isWildcardSubtype(), "'Content-Type' cannot contain wildcard subtype '*'");
		set(CONTENT_TYPE, mediaType.toString());
	}
	else {
		set(CONTENT_TYPE, null);
	}
}
```

注意如果响应时没有指定`Content-Type`，会取`supportedMediaTypes`中的第一个值，所以当程序初始化后，`supportedMediaTypes`里面的值是`*/*`的话，就会报错，这一点还是要注意的。具体赋值方式，可参考源码。

那么`read` 和 `write`究竟是如何实现的呢？这个对于接收数据的不同类型，用的转换器不一样，实现也不一样。假设现在接收的是一个`String`类型的参数，我们来看下`StringHttpMessageConverter`如何处理。

`StringHttpMessageConverter`继承自`AbstractHttpMessageConverter`，所以只需重写`readInternal`方法就可以了，代码如下：

```Java
@Override
protected String readInternal(Class<? extends String> clazz, HttpInputMessage inputMessage) throws IOException {
	Charset charset = getContentTypeCharset(inputMessage.getHeaders().getContentType());
	return StreamUtils.copyToString(inputMessage.getBody(), charset);
}
```

代码也是十分简单，首先去`Content-Type`拿到字符集，如果没有，默认是`StandardCharsets.ISO_8859_1`，这个字符集大家估计都很熟悉了。接着将body里面的数据根据字符集转为字符串，返回即可。

同样的道理，如果返回的是字符串，流程刚好和接收时相反，代码如下，就不说了。

```Java
@Override
protected void writeInternal(String str, HttpOutputMessage outputMessage) throws IOException {
	if (this.writeAcceptCharset) {
		outputMessage.getHeaders().setAcceptCharset(getAcceptedCharsets());
	}
	Charset charset = getContentTypeCharset(outputMessage.getHeaders().getContentType());
	StreamUtils.copy(str, charset, outputMessage.getBody());
}
```

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
