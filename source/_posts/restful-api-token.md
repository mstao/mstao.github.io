---
title: RESTful API风格基于Token的鉴权机制分析(与JWT结合)
tags: [java, RESTful-API, Token, JWT]
author: Mingshan
categories: [Java]
date: 2017-12-19
---

### RESTful API
RESTful API是目前比较成熟的API设计理论，它通过统一的API接口来对外提供服务，这样对其他调用者来说比较友好，更加容易实现前后端分离。具体介绍和如何使用参考这篇文章
[RESTful API介绍与使用](http://mingshan.me/2017/10/01/%E5%88%A9%E7%94%A8SpringMVC%E5%AE%9E%E7%8E%B0RESTful%20API%EF%BC%8C%E5%B9%B6%E4%B8%8ESwagger%E9%9B%86%E6%88%90%E7%94%9F%E6%88%90API%E6%96%87%E6%A1%A3/)

### 遇到的问题
在我们的项目中，仅仅使用RESTful API是远远不够的，因为RESTful API只提供JSON数据，没有了视图层，将视图渲染交给了前端，这就带来了许多问题。比如我们以前在写JSP界面的时候，会把数据放到request里面，把登录的用户信息放在session里面，因为jsp本质上也是servlet，所以可以在界面直接拿到这些数据，现在前后端一分离，怎样对用户的身份进行识别就是首要考虑的问题。

以前我们是通过session来进行对用户的身份验证，由于http协议是无状态的，所以我们需要在服务器端将用户的信息保存下来，这份登录信息会在响应时传递给浏览器，告诉其保存为cookie(Java中的jsessionid),以便下次请求时发送给我们的应用，这样我们的应用就能识别请求来自哪个用户了,这就是传统的基于session认证。

### 使用 Token 进行身份鉴权
现在我们的API可能不只是在浏览器上调用，比如app等也需要调用，这是如果只用session的话就有局限性了，所以我们可以用 **Token** 进行身份鉴权，由于Token我们可以根据自己的需要进行自定义，只要API提供方和调用方约定好如何生成Token和解析token，更加轻量化，扩展性更强。

<!-- more -->

### JWT(JSON Web Token)

JWT 是JSON风格轻量级的授权和身份认证规范，可实现无状态、分布式的Web应用授权，JWT主要由三部分构成，由.进行连接


- Header
- Payload
- Signature

所以完整的JWT如下面形式：

```
xxxxx.yyyyy.zzzzz
```
真实的JWT长这样：

![image](/images/encoded-jwt3.png)

那么header，Payload和Signature分别代表什么呢？
#### header
header主要包含两部分：
- Token的类型，这里是JWT
- 声明加密的算法 通常直接使用 HMAC SHA256

完整的header应该长这样：

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```
然后将头部进行Base64加密，得到第一部分。

#### Payload

第二部分主要是包含声明（Claims），声明是关于实体（通常是用户）和附加元数据的声明，这部分是存放数据的地方，比如用户id什么的，类似下面这样：

```json
{
  "id": "1234567890",
  "name": "mingshan",
  "admin": true
}
```
然后将有效Payload用Base64进行编码，以形成JWT的第二部分。

#### Signature
要创建签名部分，必须要有已经编码的header，编过码的Payload，一个密匙（secret）和加密算法。

例如，如果想使用HMAC SHA256算法，签名将按以下方式创建：

```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```
详细细节请参考JWT官网 [JWT官网介绍](https://jwt.io/introduction/)

### RESTful与JWT结合进行鉴权

上面只是介绍一下RESTful API和JWT，下面我们来考虑怎么实现。

首先是流程，借用JWT官网上的一幅图

![image](/images/jwt-diagram.png)

从图中我们可以总结如下（以Client和Server端为例）：

1. 首先Client发起认证请求，包含用户名和密码，进行登录操作
2. Server端验证用户名和密码，验证通过的话用密匙生成token，并将token存储起来，然后将token返回给Client
3. 此时Client已经验证登录过了，下次进行请求时将token放在header的Authorization中
4. Server端根据规则将token解析出来，拿到subject，从到拿到用户信息，然后通过用户信息去拿已经存储的token，然后将两个token进行比较，从而判断用户是否已登录
5. 将验证信息发送给Client

通过上面的流程分析，我们发现其实并不怎么难。但现在有几个地方我们还没有说

- 怎样标识哪些API需要鉴权，如何优雅地处理？
- token存在哪个地方？
- token的生成与解析如何去做？

下面我们依次来分析。

#### 标识哪些API需要鉴权

标识哪些API需要鉴权，我们需要自定义注解，然后将注解加到需要鉴权的API上面就可以了，这里自定义两个注解，**@Authorization** 和 **@CurrentUser**，**@Authorization**就是标识哪些API需要鉴权，**@CurrentUser**就是将从token获取的用户信息封装到当前user对象里面，主要用在登出处理，下面是代码：

```java
/**
 * @Description:
 * @Author: mingshan
 * @see com.lightblog.authorization.interceptor.AuthorizationInterceptor
 * @Date: Created in 19:23 2017/10/14
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Authorization {
}

```


```java

/**
 * @Description:
 * @Author: mingshan
 * @Date: Created in 19:29 2017/10/14
 */
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface CurrentUser {
}
```

由于我用到SpringMVC，所以我就考虑添加拦截器来处理用户的鉴权请求，Spring为我们提供**了org.springframework.web.servlet.handler.HandlerInterceptorAdapter**这个适配器，继承此类，可以非常方便的实现自己的拦截器。这里主要重写它的预处理方法来处理请求，代码如下：

```java
/**
 * The custom interceptor that checks out the current request has authorization.
 * @Author: mingshan
 * @Date: Created in 19:27 2017/10/14
 */
@Component
public class AuthorizationInterceptor extends HandlerInterceptorAdapter {

    @Autowired
    private TokenManager tokenManager;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                             Object handler) throws Exception {
        // Checks out the annotation of authorization that is method level.
        if (!(handler instanceof HandlerMethod)) {
            return true;
        }

        HandlerMethod handlerMethod = (HandlerMethod)handler;
        Method method = handlerMethod.getMethod();

        // Gets authorization string from request header.
        String authorization = request.getHeader(Constants.AUTHORIZATION);

        // Gets the model of Token from authorization string.
        TokenModel token = tokenManager.getToken(authorization);
        // Checks out the token that is from Redis,
        if (tokenManager.checkToken(token)) {
            // Puts userId into request.
            request.setAttribute(Constants.CURRENT_USER_ID, token.getUserId());
            return true;
        }

        // If verify token failed, and the current method has the annotation of authorization,
        // sets the response code to 401.
        // The 401 code means unauthorized.
        if (method.getAnnotation(Authorization.class) != null) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return false;
        }

        return true;
    }
}
```

在上面的**preHandle**方法里，我们首先得到处理的方法，从http请求的Authorization中拿到token，然后解析token，检测token，如果检测token成功，那么将用户id放到request中，返回true继续执行，如果检测失败，那么返回http状态码401，401意味着未认证，返回false，不继续往下执行了。


#### toekn存储位置

在我这里我是将token存在了Redis里，用户id作为key，token作为value，解析token和检测token主要在RedisTokenManager类中，主要有创建token，删除token，从jwt中获取token以及检测token，比较简单，代码如下：


```java
/**
 * The implement class of Token manager.
 * @Author: mingshan
 * @Date: Created in 23:41 2017/10/13
 */
public class RedisTokenManager implements TokenManager {
    private static final Logger logger = LoggerFactory.getLogger(RedisTokenManager.class);
    private RedisTemplate<Long, String> redisTemplate;

    public void setRedisTemplate(RedisTemplate<Long, String> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    @Override
    public TokenModel creatToken(long userId) {
        User user = new User();
        user.setId(userId);
        String subject = JWTUtil.generalSubject(user);
        String token = JWTUtil.createJWT(userId, subject, Constants.JWT_TTL);
        TokenModel model = new TokenModel(userId, token);
        redisTemplate.boundValueOps(userId).set(token, Constants.TOKEN_EXPIRES_HOUR, TimeUnit.HOURS);
        return model;
    }

    @Override
    public void deleteToken(long userId) {
        redisTemplate.delete(userId);
    }

    @Override
    public boolean checkToken(TokenModel model) {
        if (model == null) {
            return false;
        }
        Object source = redisTemplate.boundValueOps(model.getUserId()).get();
        if (source == null) {
            return false;
        }

        String token = source.toString();
        if ("".equals(token) || !token.equals(model.getToken())) {
            return false;
        }

        redisTemplate.boundValueOps(model.getUserId()).expire(Constants.TOKEN_EXPIRES_HOUR, TimeUnit.HOURS);
        return true;
    }

    @Override
    public TokenModel getToken(String authorization) {
        if (authorization == null || "".equals(authorization)) {
            return null;
        }

        User user = RequestCheck.getUserFromToken(authorization);
        if (user == null) {
            return null;
        }

        long userId = user.getId();

        String token = RequestCheck.extractJwtTokenFromAuthorizationHeader(authorization);
        TokenModel model = new TokenModel(userId, token);
        return model;
    }
}

```

上面这个类用到了如何去从JWT中解析token，这里的**getUserFromToken**方法是从JWT字符中解析token，具体来说先获取Cliams，然后获取subject，再从subject获取用户信息，最后封装成User对象，序列化对象用的是fastjson，并不复杂，代码如下：

```java
/**
 * @Author: mingshan
 * @Date: Created in 12:41 2017/12/16
 */
public class RequestCheck {
    protected static final Logger logger = LoggerFactory.getLogger(RequestCheck.class);

    public static User getUserFromToken(String auth) {
        if (auth == null) {
            return null;
        }

        if (!Constants.TOKEN_PREFIX.equals(auth.substring(0, 7))) {
            return null;
        } else {
            String token = extractJwtTokenFromAuthorizationHeader(auth);
            try {
                Claims claims = JWTUtil.parseJWT(token);
                String subject = claims.getSubject();
                User user = JSONObject.parseObject(subject, User.class);
                return user;
            } catch (Exception e) {
                e.printStackTrace();
                logger.error("解析JWT token 失败！");
            }
        }
        return null;
    }

    /**
     * 获取真实的toekn，去掉‘Bearer ’
     * @param auth
     * @return
     */
    public static String extractJwtTokenFromAuthorizationHeader(String auth) {
        // Replace "Bearer Token" to "Token" directly
        return auth.replaceFirst("[B|b][E|e][A|a][R|r][E|e][R|r] ", "").replace(" ", "");
    }

    public static void main(String[] args) {
        String s = "Bearer 122";
        System.out.print(extractJwtTokenFromAuthorizationHeader(s));

        User user = new User();
        user.setId(1);
        JSONObject jo = new JSONObject();
        jo.put("userId", user.getId());
        System.out.print(jo.toJSONString());
    }
}
```


#### token的生成与解析

这里我们写了一个工具类JWTUtil，用来创建token和解析JWT，先引入jjwt，jjwt封装了操作jwt的常用操作

```xml
<dependency>
  <groupId>io.jsonwebtoken</groupId>
  <artifactId>jjwt</artifactId>
  <version>0.9.0</version>
</dependency>
```

其中header和Payload的编解码用到了**java.util.Base64**的代码， 代码如下：


```java
/**
 * @Author: mingshan
 * @Date: Created in 21:29 2017/12/14
 */
public class JWTUtil {

    /**
     *
     * 编码
     * String asB64 = Base64.getEncoder().encodeToString("some string".getBytes("utf-8"));
     *
     * 解码
     * byte[] asBytes = Base64.getDecoder().decode("c29tZSBzdHJpbmc=");
     * System.out.println(new String(asBytes, "utf-8"));
     */

    /**
     * 获得加密的key
     * @return 加密后的key
     */
    public static SecretKey generateKey() {
        byte[] encodedKey = Base64.getDecoder().decode(Constants.JWT_SECRET);
        SecretKey key = new SecretKeySpec(encodedKey, 0, encodedKey.length, "AES");
        return key;
    }

    /**
     * 签发 JWT
     * @param id
     * @param subject
     * @param ttlMillis
     * @return 生成的JWT token
     */
    public static String createJWT(Long id, String subject, long ttlMillis) {
        SignatureAlgorithm signatureAlgorithm = SignatureAlgorithm.HS256;
        long nowMillis = System.currentTimeMillis();
        Date now = new Date(nowMillis);
        SecretKey secretKey = generateKey();
        JwtBuilder builder = Jwts.builder()
                .setId(id.toString()) // JWT_ID
                .setSubject(subject) // 主题
                .setIssuedAt(now) // 签发时间
                .signWith(signatureAlgorithm, secretKey); // 签名算法以及密匙
        if (ttlMillis >= 0) {
            // 设置过期时间
            long expMillis = nowMillis + ttlMillis;
            Date expDate = new Date(expMillis);
            builder.setExpiration(expDate);
        }
        return builder.compact();
    }

    /**
     * 解密jwt
     * @param jwt
     * @return
     * @throws Exception
     */
    public static Claims parseJWT(String jwt) throws Exception{
        SecretKey key = generateKey();
        Claims claims = Jwts.parser()
                .setSigningKey(key)
                .parseClaimsJws(jwt).getBody();
        return claims;
    }

    /**
     * 生成subject信息
     * @param user
     * @return
     */
    public static String generalSubject(User user){
        JSONObject jo = new JSONObject();
        jo.put("id", user.getId());
        return jo.toJSONString();
    }
}

```

### 完整代码

本博客的完整代码在这里，可以参考一下：
https://github.com/mstao/hnote/tree/master/hnote-bk

### 参考

http://www.scienjus.com/restful-token-authorization/
