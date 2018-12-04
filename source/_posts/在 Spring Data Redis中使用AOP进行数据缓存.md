---
title: 在 Spring Data Redis中使用AOP进行数据缓存
tags: [Redis, Spring Data, AOP, 缓存]
author: Mingshan
categories: [Java, Redis]
date: 2017-11-11
---
## 为什么要对数据进行缓存
当我们将数据从数据库中取出来后，如果我们需要再一次进行同样的操作，获取相同的数据，那么再次查询数据库无疑不是很好的方式，这时我们可以考虑来将我们的数据缓存起来，当再次获取相同的数据时，直接从缓存拿就行了。
## 进行缓存需要考虑什么问题
1. 缓存数据存在什么地方？
2. 怎样识别相同的操作(判断两次取的数据相同，生成唯一标识)
3. 数据更新时缓存该如何处理？
4. 缓存数据是否设置过期时间？
5. 如何序列化查询结果？查询结果可能是单个实体对象，也可能是一个List。
6. 代码该写在哪？不能对原有代码有侵入性

针对以上问题，我们下面来慢慢分析解决。

## 采用Redis缓存数据

Redis是一个开源（BSD许可），内存存储的数据结构服务器，可用作数据库，高速缓存和消息队列代理。它支持字符串、哈希表、列表、集合、有序集合，位图，hyperloglogs等数据类型。内置复制、Lua脚本、LRU收回、事务以及不同级别磁盘持久化功能，同时通过Redis Sentinel提供高可用，通过Redis Cluster提供自动分区。

我们在 Spring 中使用 Redis 是通过 Spring Data Redis 提供的 RedisTemplate 来操作Redis

**添加依赖**
```xml
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-redis</artifactId>
    <version>1.4.1.RELEASE</version>
</dependency>
```
spring集合Redis的配置如下：


```xml
<!-- redis 相关配置 -->
<bean id="poolConfig" class="redis.clients.jedis.JedisPoolConfig">
    <property name="maxIdle" value="${redis.maxIdle}" />
    <property name="maxWaitMillis" value="${redis.maxWait}" />
    <property name="testOnBorrow" value="true" />
</bean>

<bean id="jedisConnectionFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory"
  p:host-name="${redis.host}" p:port="${redis.port}" p:pool-config-ref="poolConfig"/>

<!-- redis template definition -->  
<bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate">  
    <!-- 设置 Redis 连接工厂-->
    <property name="connectionFactory" ref="jedisConnectionFactory"/>
    <!-- 设置默认 Serializer ，包含 keySerializer & valueSerializer -->
    <property name="defaultSerializer">
        <bean class="com.alibaba.fastjson.support.spring.GenericFastJsonRedisSerializer"/>
    </property>
    <!-- 单独设置 keySerializer -->
    <property name="keySerializer">
        <bean class="com.alibaba.fastjson.support.spring.GenericFastJsonRedisSerializer"/>
    </property>
    <!-- 单独设置 valueSerializer -->
    <property name="valueSerializer">
        <bean class="com.alibaba.fastjson.support.spring.GenericFastJsonRedisSerializer"/>
    </property>
</bean>
```
<!-- more -->

## 为查询生成唯一标识

由于Redis是以key-value形式存储数据，所以我们要考虑该如何对查询生成唯一的标识呢？首先我们可以想到可以根据sql语句来作为key，但ORM框架我用的是MyBatis，这就不是一个好的方式。其实如果两次查询调用的类名、方法名和参数值相同，我们就可以确定这两次查询结果一定是相同的（在数据没有变动的前提下）。因此，我们可以将这三个元素组合成一个字符串做为key, 就解决了标识问题。
代码如下：

```java
/**
 * 生成缓存需要的key
 * @param clazzName
 * @return 生成的key
 */
private String generateKey(String clazzName, String methodName, Object[] args) {
    StringBuffer sb = new StringBuffer(clazzName);
    sb.append(Constants.ELIMITER);
    sb.append(methodName);
    sb.append(Constants.ELIMITER);
    if (args != null) {
        for (Object arg : args) {
            sb.append(arg);
            sb.append(Constants.ELIMITER);
        }
    }

    // 去除最后一个分隔符
    sb.replace(sb.length() - 1, sb.length(), Constants.ELIMITER);
    return sb.toString();
}
```


## 代码该写在什么地方

由于考虑到不能对原有代码有侵入性，所以我们就要用到AOP了。我们可以把从数据库查询出来的数据映射到实体类，然后将其序列化为json，从缓存中取出来后再进行反序列化，代码如下：

```java
/**
 * 序列化数据
 * @param source
 * @return json字符串
 */
private String serialize(Object source) {
    return JSON.toJSONString(source);
}

/**
 * 反序列化
 * @param source
 * @param clazz
 * @param modelType
 * @return 反序列化的数据
 */
private Object deserialize(String source, Class<?> clazz, Class<?> modelType) {
    // 判断是否为List
    if (List.class.isAssignableFrom(clazz)) {
        return JSON.parseArray(source, modelType);
    }

    // 正常反序列化
    return JSON.parseObject(source, clazz);
}
```
## 具体实现

因为我们要拦截的是Mapper接口方法，因此必须命令spring使用JDK的动态代理而不是cglib的代理。为此，我们需要做以下配置：
```
<!-- 当proxy-target-class为false时使用JDK动态代理 -->
<!-- 为true时使用cglib -->
<!-- cglib无法拦截接口方法 -->
<aop:aspectj-autoproxy proxy-target-class="false" />
```

然后我们定义两个方法级别的自定义注解，其中RedisCache代表该方法需要进行缓存数据，
RedisEvict代表需要清除缓存
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Documented
public @interface RedisCache {
    @SuppressWarnings("rawtypes")
    Class type();
    int expire() default -1;      //缓存多少秒,默认无限期  
}

```


```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface RedisEvict {
    @SuppressWarnings("rawtypes")
    Class type();
}

```

注解使用方式

```java
@Override
@RedisCache(type = User.class)
public User selectByPrimaryKey(Integer id) {
    return userDao.selectByPrimaryKey(id);
}

@Override
@RedisEvict(type = User.class)
public User deleteByPrimaryKey(Integer id) {
    userDao.deleteByPrimaryKey(id);
    User user = new User();
    user.setId(id);
    return user;
}
```

首先对于要进行数据缓存操作，我们先要生成唯一标识key值，然后去Redis查询，判断缓存是否命中
- 如果缓存命中，那么将数据反序列化，将其返回
- 如果缓存未命中，那么去数据库查询数据，然后将数据进行序列化，这是需要判断是否设置了超时时间，如果没有设置，那么默认无限期，如果设置了，那么对数据设置时间。

具体AOP代码如下：


```java
@Aspect
@Component
public class RedisCacheAspect {
    public static final Logger logger = LoggerFactory.getLogger(RedisCacheAspect.class);

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    /**
     * 从Redis获取缓存的数据或者将数据缓存到Redis
     * @param pjp
     * @return 获取到的数据
     * @throws Throwable
     */
    @Around("execution(* com.han.service..*Impl.select*(..))" +
            "|| execution(* com.han.service..*Impl.get*(..))" +
            "|| execution(* com.han.service..*Impl.find*(..))" +
            "|| execution(* com.han.service..*Impl.search*(..))")
    @SuppressWarnings({ "unchecked", "rawtypes" })
    public Object cache(ProceedingJoinPoint pjp) throws Throwable {
        // 生成key
        String key = getKey(pjp);

        if (logger.isDebugEnabled()){
            logger.debug("已生成key = {}" + key);
        }

        // 得到目标方法
        Method targetMethod = getTargetMethod(pjp);
        // 得到被代理的方法上的注解
        Class<?> modelType = targetMethod.getAnnotation(RedisCache.class).type();
        String hashName = modelType.getName();

        // 利用Redis的Hash数据类型（散列）
        HashOperations opsForHash = redisTemplate.opsForHash();
        // 检查redis中是否有缓存
        String value = (String) opsForHash.get(hashName, key);

        // 最终返回结果
        Object result = null;
        // 判断缓存是否命中
        if (value != null) {
            // 缓存命中
            if (logger.isDebugEnabled()) {
                logger.debug("缓存命中, value = {}", value);
            }

            // 得到被代理方法的返回值类型
            Class<?> returnType = ((MethodSignature) pjp.getSignature()).getReturnType();

            // 反序列化从缓存中拿到的json
            result = deserialize(value, returnType, modelType);

            if (logger.isDebugEnabled()) {
                logger.debug("反序列化结果 = {}", result);
            }
        } else {
            // 缓存未命中
            if (logger.isDebugEnabled()) {
                logger.debug("缓存未命中");
            }

            // 跳过缓存,到后端查询数据
            result = pjp.proceed(pjp.getArgs());
            // 序列化查询结果
            String jsonStr = serialize(result);

            // 获取设置的缓存时间
            int timeout = targetMethod.getAnnotation(RedisCache.class).expire();
            // 如果没有设置过期时间,则无限期缓存(默认-1)
            if (timeout <= 0) {
                opsForHash.put(hashName, key, jsonStr);  
            } else {
                final TimeUnit unit = TimeUnit.SECONDS;
                final long rawTimeout = TimeoutUtils.toMillis(timeout, unit);
                // 设置缓存时间  
                redisTemplate.execute(new RedisCallback<Object>() {
                    @Override
                    public Object doInRedis(RedisConnection redisConn) throws DataAccessException {
                        // 配置文件中指定了这是一个String类型的连接
                        // 所以这里向下强制转换一定是安全的
                        StringRedisConnection conn = (StringRedisConnection) redisConn;
                        // 判断hash名是否存在
                        // 如果不存在，创建该hash并设置过期时间
                        if (!conn.exists(hashName)) {
                            conn.hSet(hashName, key, jsonStr);
                            conn.expire(hashName, rawTimeout);
                        } else {
                            conn.hSet(hashName, key, jsonStr);
                        }

                        return null;
                    }
                });
            }
        }

        return result;
    }

    /**
     * 在方法调用前清除缓存，然后调用业务方法
     * @param jp
     * @return 获取到的数据
     * @throws Throwable
     */
    @Around("execution(* com.han.service..*Impl.delete*(..))" +
            "|| execution(* com.han.service..*Impl.remove*(..))")
    @SuppressWarnings("rawtypes")
    public Object evictCache(ProceedingJoinPoint pjp) throws Throwable {
        // 得到目标的方法
        Method targetMethod = getTargetMethod(pjp);

        // 得到被代理的方法上的注解
        Class modelType = targetMethod.getAnnotation(RedisEvict.class).type();

        if (logger.isDebugEnabled()) {
            logger.debug("清空缓存:{}", modelType.getName());
        }

        // 清除对应缓存
        redisTemplate.delete(modelType.getName());

        return pjp.proceed(pjp.getArgs());
    }

    /**
     * 更新缓存的数据
     * @return 新获取的数据
     */
    @Around("execution(* com.han.service..*Impl.update*(..))" +
            "|| execution(* com.han.service..*Impl.insert*(..))" +
            "|| execution(* com.han.service..*Impl.save*(..))")
    @SuppressWarnings({ "unchecked", "rawtypes" })
    public Object updateCache(ProceedingJoinPoint pjp) throws Throwable {
        // 生成key
        String key = getKey(pjp);

        if (logger.isDebugEnabled()){
            logger.debug("已生成key = {}" + key);
        }

        // 得到目标方法
        Method targetMethod = getTargetMethod(pjp);
        // 得到被代理的方法上的注解
        Class<?> modelType = targetMethod.getAnnotation(RedisCache.class).type();
        String hashName = modelType.getName();

        // 利用Redis的Hash数据类型（散列）
        HashOperations opsForHash = redisTemplate.opsForHash();

        // 跳过缓存,到后端查询数据
        Object result = pjp.proceed(pjp.getArgs());
        // 序列化查询结果
        String jsonStr = serialize(result);
        if (logger.isDebugEnabled()) {
            logger.debug("序列化结果 = {}", result);
        }

        // 获取设置的缓存时间
        int timeout = targetMethod.getAnnotation(RedisCache.class).expire();
        // 如果没有设置过期时间,则无限期缓存(默认-1)
        if (timeout <= 0) {
            opsForHash.put(hashName, key, jsonStr);  
        } else {
            final TimeUnit unit = TimeUnit.SECONDS;
            final long rawTimeout = TimeoutUtils.toMillis(timeout, unit);
            // 设置缓存时间  
            redisTemplate.execute(new RedisCallback<Object>() {
                @Override
                public Object doInRedis(RedisConnection redisConn) throws DataAccessException {
                    // 配置文件中指定了这是一个String类型的连接
                    // 所以这里向下强制转换一定是安全的
                    StringRedisConnection conn = (StringRedisConnection) redisConn;
                    // 判断hash名是否存在
                    // 如果不存在，创建该hash并设置过期时间
                    if (!conn.exists(hashName)) {
                        conn.hSet(hashName, key, jsonStr);
                        conn.expire(hashName, rawTimeout);
                    } else {
                        conn.hSet(hashName, key, jsonStr);
                    }

                    return null;
                }
            });
        }

        return result;
    }

    /**
     * 得到目标方法
     * @param pjp
     * @return 目标方法
     * @throws SecurityException
     * @throws NoSuchMethodException
     */
    private Method getTargetMethod(ProceedingJoinPoint pjp) throws NoSuchMethodException,
                                            SecurityException {
        Signature sig = pjp.getSignature();
        if (!(sig instanceof MethodSignature)) {
            throw new IllegalArgumentException("该注解只能用于方法");
        }
        MethodSignature msig = (MethodSignature) sig;
        Object target = pjp.getTarget();
        Method targetMethod = target.getClass().getMethod(msig.getName(), msig.getParameterTypes());
        return targetMethod;
    }

    /**
     * 通过类名，方法名和参数来获取对应的key
     * @param pjp
     * @return 生成的key
     */
    private String getKey(ProceedingJoinPoint pjp) {
        // 获取类名
        String clazzName = pjp.getTarget().getClass().getName();
        // 获取方法名
        String methodName = pjp.getSignature().getName();
        // 方法参数
        Object[] args = pjp.getArgs();
        // 生成key
        return generateKey(clazzName, methodName, args);
    }

    /**
     * 生成缓存需要的key
     * @param clazzName
     * @return 生成的key
     */
    private String generateKey(String clazzName, String methodName, Object[] args) {
        StringBuffer sb = new StringBuffer(clazzName);
        sb.append(Constants.ELIMITER);
        sb.append(methodName);
        sb.append(Constants.ELIMITER);
        if (args != null) {
            for (Object arg : args) {
                sb.append(arg);
                sb.append(Constants.ELIMITER);
            }
        }

        // 去除最后一个分隔符
        sb.replace(sb.length() - 1, sb.length(), Constants.ELIMITER);
        return sb.toString();
    }

    /**
     * 序列化数据
     * @param source
     * @return json字符串
     */
    private String serialize(Object source) {
        return JSON.toJSONString(source);
    }

    /**
     * 反序列化
     * @param source
     * @param clazz
     * @param modelType
     * @return 反序列化的数据
     */
    private Object deserialize(String source, Class<?> clazz, Class<?> modelType) {
        // 判断是否为List
        if (clazz.isAssignableFrom(List.class)) {
            return JSON.parseArray(source, modelType);
        }

        // 正常反序列化
        return JSON.parseObject(source, clazz);
    }
}

```
