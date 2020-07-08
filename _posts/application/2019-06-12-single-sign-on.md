---
layout: post
title: "单点登陆"
date: 2019-06-12 16:58:33
categories: application
---

## 一、简介

**单点登陆**：在多系统中单一位置登录可以实现多系统同时登录的一种技术。常在互联网应用和企业级平台中使用。

**第三方登陆**：在某系统中使用其他系统的用户实现本系统登录的方式。如，在京东中使用微信登录。解决信息孤岛和用户不对等的实现方案。

**单点登陆需要解决的问题**：数据跨域、信息共享和安全。

## 二、跨域解决方案

### 域的概念

​	在应用模型中一个完整的，有独立访问路径的功能集合称为一个域。如：百度称为一个应用或系统。百度有若干的域，如：搜索引擎【www.baidu.com】，百度贴吧【tie.baidu.com】，百度知道【zhidao.baidu.com】，百度地图【map.baidu.com】等。域信息，有时也称为多级域名。域的划分：以IP，端口，域名，主机名为标准，实现划分。

### 跨域概念

客户端请求的时候，请求的服务器，不是同一个IP，端口，域名，主机名的时候，都称为跨域。

如：localhost和127.0.0.1也属于跨域

### Session跨域

​	所谓Session跨域就是摒弃了系统（web容器）提供的Session，而使用自定义的类似Session的机制来保存客户端数据的一种解决方案。如：通过设置cookie的domain来实现cookie的跨域传递。在cookie中传递一个自定义的session_id。这个session_id是客户端的唯一标记。将这个标记作为key，将客户端需要保存的数据作为value，在服务端进行保存。这种机制就是Session的跨域解决。

### 具体逻辑

```xml
<!-- 配置跨域请求 -->
<filter>
  <filter-name>corsFilter</filter-name>
  <filter-class>com.thetransactioncompany.cors.CORSFilter</filter-class>
  <init-param>
    <param-name>cors.allowOrigin</param-name>
    <param-value>*</param-value>
  </init-param>
  <init-param>
    <param-name>cors.supportedMethods</param-name>
    <param-value>GET, POST, HEAD, PUT, DELETE</param-value>
  </init-param>
  <init-param>
    <param-name>cors.supportedHeaders</param-name>
    <param-value>Accept, Origin, X-Requested-With, Content-Type, Last-Modified</param-value>
  </init-param>
  <init-param>
    <param-name>cors.exposedHeaders</param-name>
    <param-value>Set-Cookie</param-value>
  </init-param>
  <init-param>
    <param-name>cors.supportsCredentials</param-name>
    <param-value>true</param-value>
  </init-param>
</filter>
<filter-mapping>
  <filter-name>corsFilter</filter-name>
  <url-pattern>/*</url-pattern>
</filter-mapping>
<!-- 设置token的Servlet -->
<servlet>
  <servlet-name>corssServlet</servlet-name>
  <servlet-class>com.kun.CorssServlet</servlet-class>
</servlet>
<servlet-mapping>
  <servlet-name>corssServlet</servlet-name>
  <url-pattern>/corss</url-pattern>
</servlet-mapping>
```

```java
public class CorssServlet extends HttpServlet {

  @Override
  protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    Cookie[] cookies = req.getCookies();
    PrintWriter writer = resp.getWriter();
    //如果已经存在Cookie则查看Cookie值
    boolean findAuth = false;
    if(cookies != null) {
      for (Cookie cookie : cookies) {
        if(cookie.getName().equals("Auth")){
          findAuth = true;
          writer.println(cookie.getName());
          writer.println(cookie.getValue());
          writer.println(cookie.getMaxAge());
        }
      }
    }

    //生成cookie，注意domain的域，应该用代码智能切分
    //如www.baidu.com,tie.baidu.com均设置为.baidu.com
    if(!findAuth) {
      String token = UUID.randomUUID().toString().replace("-", "");
      Cookie authCookie = new Cookie("Auth", token);
      authCookie.setPath("/");
      authCookie.setDomain(".kun.com");
      resp.addCookie(authCookie);
    }
  }
}
```

## 三、信息共享解决方案

### Spring-session

​	spring-session技术是spring提供的用于处理集群会话共享的解决方案。spring-session技术是将用户session数据保存到三方存储容器中，如：mysql，redis等

​	spring-session技术是解决同域名下的多服务器集群session共享问题的。不能直接解决跨域session共享问题。

需要设置其它选项。

**第一步：导入依赖**

   ```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>版本号</version>
</dependency>
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>版本号</version>
    <scope>provided</scope>
</dependency>
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
    <version>版本号</version>
</dependency>
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session</artifactId>
    <version>版本号</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
    <version>版本号</version>
</dependency>
   ```

**第二步：web.xml配置**

```xml
<!-- Session设置代理过滤器 -->
<filter>
    <filter-name>springSessionRepositoryFilter</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
</filter>
<filter-mapping>
    <filter-name>springSessionRepositoryFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>

<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath*:/spring-redis.xml</param-value>
</context-param>

<servlet>
    <servlet-name>Dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath*:/spring-mvc.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>Dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

**第三步：spring-mvc，spring-redis配置**

```xml
<!-- spring-mvc配置 -->
<context:component-scan base-package="com.kun.controller" use-default-filters="false" >
    <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller" />
</context:component-scan>
<mvc:annotation-driven/>
<mvc:default-servlet-handler/>
```

```xml
<!-- spring-redis配置 -->
<context:component-scan base-package="com.kun" >
    <context:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller" />
</context:component-scan>

<bean id="poolConfig" class="redis.clients.jedis.JedisPoolConfig">
    <property name="maxIdle" value="300"/>
    <property name="maxWaitMillis" value="1000"/>
    <property name="testOnBorrow" value="true"/>
</bean>

<context:component-scan base-package="org.springframework.web.filter"/>

<!-- 添加RedisHttpSessionConfiguration用于session共享 -->
<bean id="redisHttpSessionConfiguration" class="org.springframework.session.data.redis.config.annotation.web.http.RedisHttpSessionConfiguration">
    <property name="maxInactiveIntervalInSeconds" value="18000" />
    <property name="cookieSerializer" ref="defaultCookieSerializer"/>
</bean>

<!-- 设置Cookie domain的名称，如果不设置将无法跨域 -->
<bean id="defaultCookieSerializer" class="org.springframework.session.web.http.DefaultCookieSerializer">
    <property name="domainName" value=".kun.com"/>
    <property name="cookieName" value="SESSION"/>
    <property name="cookiePath" value="/" />
</bean>

<bean id="jedisConnectionFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
    <property name="hostName" value="192.168.1.158"/>
    <property name="port" value="6379"/>
    <property name="poolConfig" ref="poolConfig"/>
    <property name="usePool" value="true"/>
    <property name="timeout" value="3000"/>
</bean>
```

**Spring-session解决跨域修改web容器方案**

​	默认session存放在当前域下，如访问www.kun.com的session的domain为www.kun.com，访问pps.kun.com的session的domain为pps.kun.com。session因为domain不同所以session并不能跨域。

​	解决方案：在web目录下，创建和WEB-INF同级的META-INF目录，并创建`context.xml`文件，将以下内容写入即可改变session的domain实现跨域访问。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- 设置主域名与子域名sessionid一致 -->
<Context  sessionCookiePath="/" sessionCookieDomain=".kun.com"/>
```

### Nginx Session共享

​	nginx中的ip_hash技术能够将某个ip的请求定向到同一台后端，这样一来这个ip下的某个客户端和某个后端就能建立起稳固的session，ip_hash是在upstream配置中定义的，如下示例

```nginx
# 设置为ip_hash策略，或者自定义hash策略【经过代理的不可设为ip_hash策略】
upstream nginx.app.com
{
    server www.kun.com:8080;
    server pps.kun.com:8080;
    ip_hash;
    # hash $remote_add; 使用自定义hash策略
}

server
{
    listen 80;
    location /
    {
        proxy_pass http://nginx.app.com;
        proxy_set_header Host  $http_host;
        proxy_set_header Cookie $http_cookie;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        client_max_body_size  100m;
    }
}
```

## 四、Token单点登陆机制

### 传统身份认证【基于cookie】

客户端使用用户名还有密码通过了身份验证，不过下回这个客户端再发送请求时候因为HTTP 是一种没有状态的协议，还得再验证一下。

解决的方法是，当用户请求登录的时候，如果没有问题，我们在服务端生成一条记录，这个记录里存储了登录的用户信息，然后把这条记录的ID号返给客户端。客户端收到以后把这个 ID 号存储在 Cookie 里，下次这个用户再向服务端发送请求的时候，可以带着这个 Cookie ，这样服务端会验证一个这个 Cookie 里的信息，看看能不能在服务端这里找到对应的记录，如果可以，说明用户已经通过了身份验证，就把用户请求的数据返回给客户端。

**这种认证中出现的问题是**：

- 会话存储：每次认证用户发起请求时，服务器需要去创建一个记录来存储信息。当越来越多的用户发请求时，内存的开销也会不断增加。

- 可扩展性：如果在服务端的内存【不借助redis等】中存储登录信息，伴随而来的是可扩展性问题。如Token需要同步。

- CORS(跨域资源共享)：如果让数据跨多台移动设备上使用时，跨域资源的共享是个问题。在使用Ajax抓取另一个域的资源，就可以会出现禁止请求的情况。

- CSRF(跨站请求伪造)：用户在访问银行网站时，可能被利用其访问其他的网站。

### Token身份认证

使用基于 Token 的身份验证方法，在服务端需要存储用户的登录记录。大致流程如下：

1. 客户端使用用户名、密码请求登录
2. 服务端收到请求，验证用户名、密码
3. 验证成功后，服务端会签发一个Token，再把这个Token发送给客户端
4. 客户端收到 Token 以后可以把它存储起来，比如放在Cookie里或者Local Storage或Session Storage里
5. 客户端每次向服务端请求资源时请求头需要带着服务端签发的Token
6. 服务端收到请求后验证客户端请求里面带着的Token，如果验证成功，就向客户端返回请求的数据

使用Token验证的优势：

无状态、可扩展。在客户端存储的Tokens是无状态的，并且能够被扩展。基于这种无状态和服务端不存储Session信息，负载负载均衡器能够将用户信息从一个服务传到其他服务器上。

安全性。请求中发送token而不再是发送cookie能够防止CSRF(跨站请求伪造)。即使在客户端使用cookie存储token，cookie也仅仅是一个存储机制而不是用于认证。不将信息存储在Session中，让我们少了对session操作。

## 五、JSON Web Token（JWT）机制

​	JWT是一种紧凑且自包含的，用于在多方传递JSON对象的技术。传递的数据可以使用数字签名增加其安全行。可以使用HMAC加密算法或RSA公钥/私钥加密方式。

**紧凑**：数据小，可以通过URL，POST参数，请求头发送。且数据小代表传输速度快。

**自包含**：使用payload数据块记录用户必要且不隐私的数据，可以有效的减少数据库访问次数，提高代码性能。

---

JWT一般用于处理用户身份验证或数据信息交换。

**用户身份验证**：一旦用户登录，每个后续请求都将包含JWT，允许用户访问该令牌允许的路由，服务和资源。单点登录是当今广泛使用JWT的一项功能，因为它的开销很小，并且能够轻松地跨不同域使用。

**数据信息交换**：JWT是一种非常方便的多方传递数据的载体，因为可以保证数据的有效性和安全性。

### JWT数据结构

JSON Web Token由三部分组成，它们之间用圆点(.)连接。这三部分分别是：

- Header

  header由两部分组成：token的类型（“JWT”）和算法名称（比如：HMAC SHA256或者RSA等等），如

  ```json
  {
    "alg": "HS256",
    "type": "JWT"
  }
  ```

- Payload

  包含声明，声明有三种类型: registered, public 和 private。

  - Registered claims : 这里有一组预定义的声明，它们不是强制的。如：iss (发行者), exp (过期时间), sub (主题), aud (受众)等。
  - Public claims : 可以随意定义。一般都会在JWT注册表中增加定义。避免和已注册信息冲突。
  - Private claims : 用于在同意使用它们的各方之间共享信息，并且不是注册的或公开的声明。

  ```json
  {
  	"sub": "user Login",
  	"name": "kun"
  	"admin": true
  }
  ```

- Signature

  签名算法是header中指定的，用于将Header和Payload加密判断是否更改，具体操作步骤为：

  加密算法(base64URLEncoder(header) + "." + base64URLEncoder(payload), secret)

  签名是用于验证消息在传递过程中有没有被更改，并且，对于使用私钥签名的token，它还可以验证JWT的发送方是否为它所称的发送方。

### token保存位置

- webstorage

  webstorage可保存的数据容量为5M分为localStorage和sessionStorage。且只能存储字符串数据。

  - **localStorage**的生命周期是永久的，关闭页面或浏览器之后localStorage中的数据也不会消失。localStorage除非主动删除数据，否则数据永远不会消失。

  - **sessionStorage**是会话相关的本地存储单元，生命周期是在仅在当前会话下有效。sessionStorage引入了一个“浏览器窗口”的概念，sessionStorage是在同源的窗口中始终存在的数据。只要这个浏览器窗口没有关闭，即使刷新页面或者进入同源另一个页面，数据依然存在。但是sessionStorage在关闭了浏览器窗口后就会被销毁。同时独立的打开同一个窗口同一个页面，sessionStorage也是不一样的。

- Cookie

  使用Cookie存储，使用请求头发送可以避免CSRF(跨站请求伪造)，且方便跨域请求。

### JWT执行流程

![JWT-process](/img/sso/JWT-process.png)

### JWT应用示例

JWT工具类，用于生成JWT和解析JWT

```java
public class JwtUtil {

  // 生成签名的时候使用的秘钥secret,这个方法本地封装了的，一般可以从本地配置文件中读取
  // 切记这个秘钥不能外露哦。它就是你服务端的私钥，在任何场景都不应该流露出去。
  // 一旦客户端得知这个secret, 那就意味着客户端是可以自我签发jwt了。
  private static final String secret = "com.kun.secret";

  /**
    * 用户登录成功后生成Jwt
    * 使用HS256算法  私匙使用用户密码
    */
  public static String createJWT(long ttlMillis, User user) {
    // 指定签名的时候使用的签名算法，也就是header那部分，jjwt已经将这部分内容封装好了。
    SignatureAlgorithm signatureAlgorithm = SignatureAlgorithm.HS256;

    // 生成JWT的时间
    long nowMillis = System.currentTimeMillis();

    // 创建payload的私有声明（根据特定的业务需要添加，如果要拿这个做验证，一般是需要和jwt的接收方提前沟通好验证方式的）
    Map<String, Object> claims = new HashMap<String, Object>();
    claims.put("userId", user.getId());
    claims.put("userName", user.getUserName());
    claims.put("userAge", user.getUserAge());

    // 生成签发人
    String subject = user.getUserName();

    // 下面就是在为payload添加各种标准声明和私有声明了
    // 这里其实就是new一个JwtBuilder，设置jwt的body
    JwtBuilder builder = Jwts.builder()
      // 如果有私有声明，一定要先设置这个自己创建的私有的声明，这个是给builder的claim赋值
      // 一旦写在标准的声明赋值之后，就是覆盖了那些标准的声明的
      .setClaims(claims)
      // 设置jti(JWT ID)：是JWT的唯一标识，根据业务需要，这个可以设置为一个不重复的值
      // 主要用来作为一次性token,从而回避重放攻击。
      .setId(UUID.randomUUID().toString())
      // iat: jwt的签发时间
      .setIssuedAt(new Date(nowMillis))
      // 代表这个JWT的主体，即它的所有人
      // 这个是一个json格式的字符串，可以存放什么userid，roldid之类的，作为什么用户的唯一标志。
      .setSubject(subject)
      // 设置过期时间
      .setExpiration(new Date(nowMillis + ttlMillis))
      // 设置签名使用的签名算法和签名使用的秘钥
      .signWith(signatureAlgorithm, secret);
    return builder.compact();
  }


  /**
    * Token的解密, parseClaimsJws步骤会校验
    */
  public static Claims parseJWT(String token) {
    // 签名秘钥，和生成的签名的秘钥一模一样
    // 得到DefaultJwtParser
    Claims claims = Jwts.parser()
      //设置签名的秘钥
      .setSigningKey(secret)
      //设置需要解析的jwt
      .parseClaimsJws(token).getBody();
    return claims;
  }
}
```

JWT测试

```java
public class JWTTest {

  private static User user = new User(1001, "kun", "123456", 19);

  // 创建
  @Test
  public void create() {
    String token = JwtUtil.createJWT(60000, user);
    System.out.println(token);
  }

  // 解析&校验
  @Test
  public void parse() {
    String token = "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJrdW4iLCJ1c2VyTmFtZSI6Imt1biIsImV4cCI6MTU2MDMyNzcxOSwidXNlcklkIjoxMDAxLCJpYXQiOjE1NjAzMjc2NTksInVzZXJBZ2UiOjE5LCJqdGkiOiIwNGE1MDQxMi1jY2Q5LTRhYzQtODYwZC02NzcyYTAzMWVjMmEifQ.T1Q7JF6HyJrA1R0XJd5QFpFBAGpepJwPNNOeuFFEbux";
    Claims claims = JwtUtil.parseJWT(token);
    System.out.println(claims.get("userId"));
    System.out.println(claims.get("userName"));
    System.out.println(claims.get("userAge"));
  }
}
```

### JWT单点登陆使用注意点

- 使用Cookie时需要解决跨域问题（设置domain），Cookie只是存储手段，需要使用Http请求头发送
- 服务器需要开启允许跨域请求，否则无法发起跨域请求
- JWT的过期时间需要在每次请求之后进行修改

