---
layout: post
title: "spring cloud gateway使用详解"
date: 2021-02-02 16:42:30
categories: Spring Cloud
keywords: "spring cloud gateway"
description: "spring-cloud-gateway使用详解"
---

## 一、Gateway概述

### 微服务网关简介

在微服务架构中，不同的微服务可以有不同的网络地址，各个微服务之间通过互相调用完成用户请求。这样会带来几个问题：

- 客户端多次请求不同的微服务，增加客户端的复杂性
- 认证复杂，每个服务都要进行认证
- 存在跨域请求，比较复杂

于是微服务网关就是在客户端和服务端之间增加一个API网关，所有的外部请求先通过这个微服务网关，它只需跟网关进行交互，而由网关进行各个微服务的调用。这样前端只需请求一个IP地址，也解决的权鉴问题。

总结一下，服务网关大概就是四个功能：**统一接入、流量管控、协议适配、安全维护**

### Gateway简介

spring Cloud Gateway是Spring Cloud推出的第二代网关框架，取代Zuul网关。提供了路由转发、权限校验、限流控制等作用。Spring Cloud Gateway 使用非阻塞 API，支持 WebSockets。

**相关概念：**

- **Route（路由）**：网关的基本构建块。由一个 ID，一个目标 URI，一组断言和一组过滤器定义。如果断言为真，则路由匹配。
- **Predicate（断言）**：这是一个Predicate。输入类型是一个ServerWebExchange。可以使用它来匹配来自 HTTP 请求的任何内容，例如headers或参数。
- **Filter（过滤器）**：这是`GatewayFilter`或`GlobalFilter`的实例，可以使用它修改请求和响应。

<img src="/img/spring-cloud-gateway/gateway架构.png" alt="gateway架构" style="zoom:80%;" />

客户端向Spring Cloud Gateway发出请求。如果Gateway Handler Mapping中找到与请求相匹配的路由，将其发送到Gateway Web Handler。Handler再通过指定的过滤器链来将请求发送到实际的服务执行业务逻辑，然后返回。

## 二、集成Gateway

**第一步：添加依赖**

```xml
<!-- 不可以引入spring-boot-starter-web会有冲突 -->
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

**第二步：配置好注册中心[按照Eureka、zookeeper、nacos客户端配置即可]**

**第三步：编写YAML**

```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          # 是否从注册中心读取服务
          enabled: true
      routes:
          # 服务的ID，唯一即可一般与微服务的service name一致
        - id: cloud-order-service
          # lb表示负载均衡
          uri: lb://cloud-order-service
          predicates:
            # 路径匹配,所有order的请求都转发到cloud-order-service
            - Path=/order/**
```

## 三、Gateway的设置

### Gateway的断言设置

断言设置可以用户延迟发布和信息校验等

```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
      routes:
        - id: cloud-order-service
          uri: lb://cloud-order-service
          predicates:
            - Path=/order/**
            # 日期类断言
            - After=2020-12-11T10:58:41.659+08:00[Asia/Shanghai]
            - Before=2020-06-20T22:46:41.659+08:00[Asia/Shanghai]
            - Between=2020-06-20T22:46:41.659+08:00[Asia/Shanghai]
            # 参数校验，后跟正则表达式
            - Cookie=username,^\w+$
            - Header=username,^\w+$
            - Query=username, \d+
            - Query=password, \d+
            # 请求方式校验
            - Method=GET     
```

### Gateway的过滤器设置

Spring Cloud Gateway的Filter从作用范围可分为另外两种`GatewayFilter`与`GlobalFilter`。Filter可用于权鉴、流控、日志等

**GlobalFilter应用到所有的路由上**

```java
@Component
public class LoginFilter implements GlobalFilter, Ordered {

  /**
    * 执行过滤器中的业务逻辑
    */
  @Override
  public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    System.out.println("执行了自定义的全局过滤器");
    //1.获取请求参数access-token
    String token = exchange.getRequest().getQueryParams().getFirst("access-token");
    //2.判断是否存在
    if(token == null) {
      //3.如果不存在 : 认证失败
      System.out.println("没有登录");
      exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
      return exchange.getResponse().setComplete(); //请求结束
    }
    //4.如果存在,继续执行
    return chain.filter(exchange); //继续向下执行
  }

  /**
    * 指定过滤器的执行顺序 , 返回值越小执行优先级越高
    */
  @Override
  public int getOrder() {
    return Ordered.HIGHEST_PRECEDENCE;
  }
}
```

**GatewayFilter一般使用系统提供的**

例如：Gateway以及微服务上都设置了<a href="#cors">CORS（解决跨域）</a>，如果不做任何配置[请求 -> 网关 -> 微服务]将会有重复的请求头，使用GatewayFilter过滤器可以过滤重复请求头

```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
      routes:
        - id: cloud-order-service
          uri: lb://cloud-order-service
          filters:
            # 表示采用第一个为准
            - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin Origin, RETAIN_FIRST
```

### Gateway的自定义错误页面

**Gateway是采用webflux开发，所以不能使用spring-webmvc那套@ControllerAdvice自定义错误处理**

```java
/**
 * 实际业务处理
 */
public class JsonErrorWebExceptionHandler extends DefaultErrorWebExceptionHandler {

  public JsonErrorWebExceptionHandler(ErrorAttributes errorAttributes,
                                      ResourceProperties resourceProperties,
                                      ErrorProperties errorProperties,
                                      ApplicationContext applicationContext) {
    super(errorAttributes, resourceProperties, errorProperties, applicationContext);
  }

  @Override
  protected Map<String, Object> getErrorAttributes(ServerRequest request, boolean includeStackTrace) {
    // 这里可以根据异常类型进行定制化逻辑
    Throwable error = super.getError(request);
    Map<String, Object> errorAttributes = new HashMap<>(8);
    errorAttributes.put("message", error.getMessage());
    errorAttributes.put("code", HttpStatus.INTERNAL_SERVER_ERROR.value());
    errorAttributes.put("method", request.methodName());
    errorAttributes.put("path", request.path());
    if(error instanceof ResponseStatusException){
      ResponseStatusException statusException = (ResponseStatusException) error;
      errorAttributes.put("code", statusException.getStatus().value());
    }
    return errorAttributes;
  }

  @Override
  protected RouterFunction<ServerResponse> getRoutingFunction(ErrorAttributes errorAttributes) {
    return RouterFunctions.route(RequestPredicates.all(), this::renderErrorResponse);
  }

  @Override
  protected int getHttpStatus(Map<String, Object> errorAttributes) {
    // 这里其实可以根据errorAttributes里面的属性定制HTTP响应码
    return (int)errorAttributes.get("code");
  }
}



/**
 * 自定义异常处理
 */
@Configuration
@EnableConfigurationProperties({ ServerProperties.class, ResourceProperties.class })
public class ExceptionHandlerConfiguration {

  private final ServerProperties serverProperties;

  private final ApplicationContext applicationContext;

  private final ResourceProperties resourceProperties;

  private final List<ViewResolver> viewResolvers;

  private final ServerCodecConfigurer serverCodecConfigurer;

  public ExceptionHandlerConfiguration(ServerProperties serverProperties, ResourceProperties resourceProperties,
                                       ObjectProvider<List<ViewResolver>> viewResolversProvider, ServerCodecConfigurer serverCodecConfigurer,
                                       ApplicationContext applicationContext) {
    this.serverProperties = serverProperties;
    this.applicationContext = applicationContext;
    this.resourceProperties = resourceProperties;
    this.viewResolvers = viewResolversProvider.getIfAvailable(Collections::emptyList);
    this.serverCodecConfigurer = serverCodecConfigurer;
  }

  // 实例化逻辑错误处理类
  @Bean
  @Order(Ordered.HIGHEST_PRECEDENCE)
  public ErrorWebExceptionHandler errorWebExceptionHandler(ErrorAttributes errorAttributes) {
    JsonErrorWebExceptionHandler exceptionHandler = new JsonErrorWebExceptionHandler(errorAttributes, this.resourceProperties,
                                                                                     this.serverProperties.getError(), this.applicationContext);
    exceptionHandler.setViewResolvers(this.viewResolvers);
    exceptionHandler.setMessageWriters(this.serverCodecConfigurer.getWriters());
    exceptionHandler.setMessageReaders(this.serverCodecConfigurer.getReaders());
    return exceptionHandler;
  }
}
```

### <span id="cors">Gateway设置跨域请求</span>

```java
@Configuration
public class GwCorsFilter {

  @Bean
  public CorsWebFilter corsFilter() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowCredentials(true);   // 允许cookies跨域
    config.addAllowedOrigin("*");       // #允许向该服务器提交请求的URI，*表示全部允许
    config.addAllowedHeader("*");       // #允许访问的头信息,*表示全部
    config.addAllowedMethod("*");       // 允许提交请求的方法类型，*表示全部允许
    config.setMaxAge(18000L);           // 预检请求的缓存时间（秒），即在这个时间段里，对于相同的跨域请求不会再预检了
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource(new PathPatternParser());
    source.registerCorsConfiguration("/**", config);
    return new CorsWebFilter(source);
  }
}
```

